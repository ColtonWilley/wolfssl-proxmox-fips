# wolfSSL FIPS for Ceph on Proxmox VE — Deployment Guide

Deploy a Proxmox VE 9 cluster with hyper-converged Ceph (Tentacle 20.2.2), using wolfCrypt FIPS for
Ceph encryption and authentication, corosync, and dashboard TLS.

Run the commands in order: on the builder first (§4–§5), then on every node (§6 onward).
Replace each `<placeholder>` with the real value. Blocks that span machines identify where to run
each command.

---

## 1. Scope and requirements

The deployment routes these components through wolfCrypt FIPS:

| Surface | What it protects | wolfCrypt routing |
|---|---|---|
| **msgr2 secure wire** | OSD↔OSD replication, recovery/backfill, mon traffic, and the client half of guest storage I/O inside the guest's QEMU process | wolfProvider (AES-128-GCM), for the daemons and the guest processes scoped in §6.3 |
| **cephx auth** | cluster authentication tickets + key material | Ceph source patch → wolfProvider (AES-128-CBC, HMAC) |
| **key/nonce RNG** | the msgr2 session key + all cephx key material | CRNG kernel patch (getrandom → wolfCrypt FIPS-DRBG) |
| **OSD data at rest** | BlueStore data on each OSD device | in-kernel `libwolfssl.ko` LKCAPI (dm-crypt `aes-cbc-essiv:sha256` bulk cipher + ESSIV salt-hash) |
| **OSD keyslot cryptography** | the LUKS keyslot that protects each OSD volume key | userspace wolfProvider, through the patched `cryptsetup` (§5.2) |
| **corosync membership** | quorum + `/etc/pve` sync | `crypto_model: openssl` → wolfProvider |
| **mgr dashboard / REST TLS** | the Ceph dashboard's HTTPS transport (§6.6) | wolfProvider (TLS 1.3 + AES-GCM) |

**Note:** `/etc/environment` routes `ceph` and `rbd` through wolfProvider in PAM sessions, including
interactive logins and remote commands (§6.3). Calls from services without a PAM session, such as
`pvestatd`, use system OpenSSL for cluster state. Guest data I/O uses the QEMU processes configured
in §6.3.

The Proxmox VE management interface is configured separately in the Node Configuration and Recovery
guide.

### 1.1 Requirements

Meet the standard Ceph
[preconditions](https://pve.proxmox.com/pve-docs/chapter-pveceph.html): node count, network sizing, and
a blank data device per node. Provision these and the guest network bridge for §7.2 before §6.

Three prerequisites are specific to this deployment:

- **A licensed wolfSSL FIPS bundle.** Use its FIPS build configuration.
- **The three Ceph patches** — `9997-ceph-volume-fips-luks-parameters.patch`,
  `9998-ceph-crypto-hmac-evp-mac.patch` and `9999-cephx-cached-evp.patch`, included in this
  repository's [patches/ceph/ directory](../../patches/ceph/). §5.1 applies them to the Proxmox
  Ceph source.
- **A build machine**, on which §4 and §5 are performed.

**Note:** For Secure Boot, complete Foundation §2.6 and use §6.2 on every node.

---

## 2. Architecture

The deployment uses five components:

1. **CRNG kernel** (`proxmox-kernel-7.0.12-1+wolfcrng1-pve`) — routes `getrandom()` and
   `/dev/urandom` to the wolfCrypt FIPS DRBG. It includes `CONFIG_CRYPTO_FIPS` and a retained
   module-signing key.
2. **In-kernel `libwolfssl.ko`** — registers wolfCrypt with the kernel crypto API for OSD dm-crypt.
3. **wolfProvider and its matched `libwolfssl`** — route userspace OpenSSL calls through wolfCrypt,
   configured per process with `OPENSSL_CONF`.
4. **The two cephx patches** — replace low-level calls that bypass the provider interface.
5. **`fips=1` on the kernel command line** — the module runs its power-on self-tests, and module
   signing keeps unsigned modules out of the kernel.

**Note:** On this kernel, `fips_enabled` becomes `1` at module load even with `fips=0`. Check the boot
flag and the FIPS userspace provider separately in §9; the runtime flag alone is insufficient.

---

## 3. Version pins

Use Proxmox VE 9 on Debian trixie; no particular point release is required.

| Component | Pin |
|---|---|
| Ceph | **Tentacle 20.2.2-pve1** — `git.proxmox.com/git/ceph.git` commit `06f6a7ab` (+ the three wolfcrypt patches) |
| wolfSSL FIPS, wolfProvider, CRNG kernel and patch | See `WOLFSSL-PROXMOX-FIPS-FOUNDATION.md` |
| Ceph patches | `9997-ceph-volume-fips-luks-parameters.patch` (`src/ceph-volume/ceph_volume/util/encryption.py`), `9998-ceph-crypto-hmac-evp-mac.patch` (`src/common/ceph_crypto.h`), `9999-cephx-cached-evp.patch` (`src/auth/Crypto.cc`) |
| cryptsetup | the distribution's own version, rebuilt with the osp routing patch as **`<version>+wolfprov1`** (§5.2) |

---

## 4. Build the foundation and Ceph module

Run §4–§5 once on a Debian trixie builder, such as a fresh PVE 9 node, with **~210 GB free**:
~90 GB for the kernel tree and ~120 GB for Ceph. §5.3 collects the artifacts for installation in §6.

### 4.1 Build the shared foundation

Complete `WOLFSSL-PROXMOX-FIPS-FOUNDATION.md` on the builder before continuing.

Retain the kernel **build tree** (`$BD`/`$KSRC` in `/root/wolf-buildenv.sh`) and
`/root/wolfssl-fips-src` through §4.2. §5.1 removes the build tree to free disk space. Keep **`quilt`**
installed through §5.1.

Place the three Ceph patches (§1.1) in `/root/` on the builder and confirm all three are present:

```bash
[ "$(ls /root/999[789]-*.patch 2>/dev/null | wc -l)" = 3 ] \
  && echo "OK: all three Ceph patches present" \
  || { echo "FATAL: expected 3 patches in /root/, found $(ls /root/999[789]-*.patch 2>/dev/null | wc -l)"; exit 1; }
```

### 4.2 In-kernel `libwolfssl.ko` for OSD at-rest

Use the FIPS configuration from your licensed bundle's instructions for `<your-bundle-config>`.

```bash
VER=7.0.12-1+wolfcrng1-pve
# Install the kernel headers on the builder:
apt-get remove -y proxmox-kernel-7.0-build-deps 2>/dev/null || true
dpkg -i /root/pve-kernel/proxmox-headers-${VER}_*.deb
cd /root/wolfssl-fips-src                                  # FIPS source from Foundation
make clean >/dev/null 2>&1 || true
./configure --enable-fips=<your-bundle-config> --enable-linuxkm \
    --enable-linuxkm-lkcapi-register='cbc(aes),sha2,stdrng-default' \
    --with-linux-source=/usr/src/linux-headers-${VER}
make -j"$(nproc)" KBUILD_EXTRA_FLAGS=FORCE_GLOBAL_OBJTOOL_OFF=1 module-with-matching-fips-hash
# Sign the unsigned .ko with the retained key, not .ko.signed; re-signing double-appends the signature.
/lib/modules/${VER}/build/scripts/sign-file sha512 \
    /root/kernel-signing-key.pem /root/kernel-signing-key.x509 linuxkm/libwolfssl.ko
cp linuxkm/libwolfssl.ko /root/libwolfssl-ceph.ko         # at-rest module for distribution in §5.3
```

**Note:** The builder needs the headers at `/usr/src/linux-headers-VER` and `/lib/modules/VER/build`
to configure and sign the module. Remove the conflicting build-deps metapackage before installing them.

**Important:** Build this module with the register set above. Do not reuse a module from another
build: without `cbc(aes)`, OSD dm-crypt will not use wolfCrypt.

---

## 5. Build the packages

### 5.1 Ceph packages

Append the three wolfcrypt patches to Proxmox's quilt series and rebuild with the unchanged toolchain.
Ceph uses wolfProvider at runtime; it does not need to link against libwolfssl.

**Caution:** Allow **at least 120 GB free**. The completed build occupies about 81 GB, with higher
usage during package staging. After completing §4.2, remove the kernel build tree to free space:

```bash
df -h /root                                  # want 120 GB or more
rm -rf /root/pve-kernel/proxmox-kernel-*/    # ~40 GB build tree; the kernel .debs beside it are kept
```

```bash
cd /root
# no tags are published, so Ceph 20.2.2-pve1 is pinned by commit:
git clone https://git.proxmox.com/git/ceph.git proxmox-ceph
cd proxmox-ceph && git checkout 06f6a7ab7187f570d9fc04b7ac9664f1fb6b84b0 && cd /root
head -1 proxmox-ceph/changelog.Debian                  # -> ceph (20.2.2-pve1) trixie
# Copy and confirm all three patches before appending them:
cp 999[789]-*.patch proxmox-ceph/patches/
[ "$(ls proxmox-ceph/patches/999[789]-*.patch | wc -l)" = 3 ] || { echo "FATAL: not all three patches copied"; exit 1; }
echo 9997-ceph-volume-fips-luks-parameters.patch >> proxmox-ceph/patches/series   # last three,
echo 9998-ceph-crypto-hmac-evp-mac.patch         >> proxmox-ceph/patches/series   # in this order
echo 9999-cephx-cached-evp.patch                 >> proxmox-ceph/patches/series
cd proxmox-ceph/ceph && mk-build-deps -i -r -t 'apt-get -y --no-install-recommends' debian/control
cd /root/proxmox-ceph
# Set an identity to prevent dch waiting at `Press RETURN`:
export DEBEMAIL="${DEBEMAIL:-root@$(hostname -f)}" DEBFULLNAME="${DEBFULLNAME:-FIPS builder}"
dch --changelog changelog.Debian --local '+wolfcryptfips' -D trixie --force-distribution \
    "cephx AES/HMAC to EVP; FIPS LUKS parameters for encrypted OSDs"   # top-level changelog, not ceph/debian/
DEB_BUILD_OPTIONS="parallel=8 noautodbgsym" make deb   # raise parallel to suit the builder — see below
```

**Important:** The final `lintian` target reports stock Ceph packaging errors (`E:`), causing
`make deb` to end with `Error 2` after the packages are built. Confirm the artifacts below:

```bash
ls /root/proxmox-ceph/*20.2.2-pve1+wolfcryptfips1*.deb | wc -l     # -> 81
dpkg-deb -f /root/proxmox-ceph/ceph-common_*.deb Version           # -> 20.2.2-pve1+wolfcryptfips1
# Package counts and versions do not confirm patch contents. Check patch 9997 in the built package:
dpkg-deb --fsys-tarfile /root/proxmox-ceph/ceph-volume_*_all.deb \
  | tar xO ./usr/lib/python3/dist-packages/ceph_volume/util/encryption.py \
  | grep -q "aes-cbc-essiv:sha256" \
  && echo "OK: ceph-volume carries the FIPS LUKS parameters" \
  || { echo "FATAL: the build did not apply patch 9997"; exit 1; }
```

**Caution:** Limit `parallel` to roughly one job per 3 GB of RAM to avoid OOM kills. Do not resume a
killed build with `dpkg-buildpackage -nc`; see Troubleshooting (§11).

**Note:** Retain `.git`; `make-dist` needs it for `CEPH_GIT_VER`. Without it, deployment later fails
with a misleading version error (§11).

### 5.2 `cryptsetup` against wolfProvider

`ceph-volume` uses `cryptsetup` for encrypted OSDs. Its private OpenSSL context ignores `OPENSSL_CONF`,
so rebuild with the
[wolfSSL/osp](https://github.com/wolfSSL/osp/tree/master/wolfProvider/libcryptsetup) routing patch to
use wolfProvider for OSD keyslot cryptography.

The build needs about 1 GB and may be run before or after §5.1.

```bash
# Enable deb-src for apt-get source; fresh trixie nodes have only Types: deb.
grep -q '^Types: deb deb-src' /etc/apt/sources.list.d/debian.sources \
  || sed -i 's/^Types: deb$/Types: deb deb-src/' /etc/apt/sources.list.d/debian.sources
apt-get update
mkdir -p /root/cs-build && cd /root/cs-build && apt-get source cryptsetup
cd cryptsetup-*/
apt-get build-dep -y cryptsetup
curl -fsSLO https://raw.githubusercontent.com/wolfSSL/osp/master/wolfProvider/libcryptsetup/libcryptsetup-wolfprov-routing.patch
patch -p1 < libcryptsetup-wolfprov-routing.patch
DEBEMAIL="you@example.com" DEBFULLNAME="wolfSSL FIPS" dch -l +wolfprov -D trixie --nomultimaint \
    "Route libcryptsetup OpenSSL backend through wolfProvider (FIPS)."
# nocheck: the upstream self-tests use non-FIPS vectors and would need wolfprov installed at build time.
DEB_BUILD_OPTIONS=nocheck dpkg-buildpackage -b -uc -us
cd /root/cs-build
ls libcryptsetup12_*wolfprov1_*.deb cryptsetup_*wolfprov1_*.deb cryptsetup-bin_*wolfprov1_*.deb
```

**Note:** Use the routing patch, not osp's larger patch for `cryptsetup` 2.6.1.

**Important:** wolfProvider does not support Argon2. The patched `cryptsetup` rejects LUKS2's default
Argon2id KDF. Patch `9997` (§5.1) makes encrypted-OSD provisioning request PBKDF2 explicitly.

### 5.3 Assemble and distribute artifacts

Collect the artifacts and serve the Ceph packages through a **local apt repository** so their
dependencies resolve:

```bash
# On the builder, collect the §4–§5 artifacts:
A=/root/artifacts; VER=7.0.12-1+wolfcrng1-pve; mkdir -p "$A"
cp -a /usr/local/lib/libwolfssl.so*                         "$A"/   # FIPS libwolfssl   (Foundation §3)
cp    /usr/lib/x86_64-linux-gnu/ossl-modules/libwolfprov.so "$A"/   # wolfProvider      (Foundation §3)
cp    /etc/wolfprovider/openssl.cnf                         "$A"/   # provider routing config (Foundation §3.2)
cp    /root/pve-kernel/proxmox-kernel-${VER}_*.deb          "$A"/   # CRNG kernel       (§4.1)
cp    /root/pve-kernel/proxmox-headers-${VER}_*.deb         "$A"/   # matching headers  (§4.1)
cp    /root/libwolfssl-ceph.ko                              "$A"/   # in-kernel at-rest module (§4.2)
mkdir -p "$A"/cryptsetup && cp /root/cs-build/*wolfprov1_*.deb "$A"/cryptsetup/   # patched cryptsetup (§5.2)
# Include the Secure Boot artifacts when Foundation §2.6 was run:
if ls /root/signed-debs/proxmox-kernel-${VER}_*.deb >/dev/null 2>&1; then
  mkdir -p "$A"/secureboot
  cp /root/signed-debs/proxmox-kernel-${VER}_*.deb "$A"/secureboot/
  cp /root/kernel-signing-key.x509 "$A"/secureboot/
fi
# the Ceph .deb set, served as a local apt repo with an uncompressed index:
mkdir -p "$A"/ceph-repo && cp /root/proxmox-ceph/*.deb "$A"/ceph-repo/
cd "$A"/ceph-repo && dpkg-scanpackages -m . > Packages && gzip -9 -k -f Packages   # both plain + .gz
```

**Note:** Include the uncompressed `Packages` index. Trixie `apt` treats a flat repository with only
`Packages.gz` as empty and installs stock Ceph from the Proxmox repository instead.

**Distribute a complete copy of `$A` to `/root/artifacts` on every node**, including the `cryptsetup/`
and `ceph-repo/` subdirectories. Confirm the copy against the builder:

```bash
ls /root/artifacts/ceph-repo/*.deb | wc -l    # must match the builder
find /root/artifacts -type f | wc -l          # must match the builder
```

---

## 6. Deploy the cluster

Install with `fips=0`, create the cluster, then set `fips=1` and verify in §9.

| Subsection | Where to run it | Complete first |
| --- | --- | --- |
| 6.1 Prepare a node | Every node, identical | §5.3 delivered to this node |
| 6.2 Enroll the signing certificate | Every **Secure Boot** node | 6.1 on this node |
| 6.3 Reboot and configure the node | Every node, identical | 6.1 on this node, and 6.2 on a Secure Boot node |
| 6.4 Bootstrap the first node | The first node only | 6.3 **on this node** |
| 6.5 Bring up each remaining node | Each remaining node, identical | 6.3 on this node, and 6.4 |
| 6.6 Configure dashboard TLS | Once, from any node | 6.5 on every node |

Each node reboots at the start of §6.3; reconnect to finish §6.3 before continuing to §6.4 or §6.5.

**Important:** Complete §6.1–§6.3 before creating or joining the cluster. Cluster creation generates
the corosync authkey and SSH trust once; later configuration does not regenerate them. A cluster
created before these steps must be torn down and rebuilt, even if the §9 runtime checks pass.

Nodes may complete §6.1–§6.3 in parallel.

### 6.1 Prepare a node

Run this subsection **on every node**, including the first.

Configure this node's Proxmox VE and Debian repositories for package dependencies; see
[Package Repositories](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#sysadmin_package_repositories).

**Caution:** `apt` may install stock Ceph from the Proxmox repository. The checks below confirm the
patched package versions and patch `9997` before continuing.

```bash
VER=7.0.12-1+wolfcrng1-pve; A=/root/artifacts      # your distributed-artifacts dir
# 1. Install the wolfProvider pair and configuration files:
rm -f /usr/local/lib/libwolfssl.so*                # clear any stray system wolfssl first
cp -a $A/libwolfssl.so*  /usr/local/lib/           # library + soname symlinks
install -Dm0755 $A/libwolfprov.so /usr/lib/x86_64-linux-gnu/ossl-modules/libwolfprov.so
ldconfig
install -Dm0644 $A/openssl.cnf /etc/wolfprovider/openssl.cnf
# 2. Install the kernel and headers, using the signed image if supplied:
apt-get remove -y proxmox-kernel-7.0-build-deps 2>/dev/null || true
if ls $A/secureboot/proxmox-kernel-${VER}_*.deb >/dev/null 2>&1; then
  dpkg -i $A/secureboot/proxmox-kernel-${VER}_*.deb $A/proxmox-headers-${VER}_*.deb
else
  dpkg -i $A/proxmox-kernel-${VER}_*.deb $A/proxmox-headers-${VER}_*.deb
fi
# 3. Install the at-rest module and load it at boot:
install -m0644 $A/libwolfssl-ceph.ko /lib/modules/${VER}/kernel/libwolfssl.ko
depmod -a "$VER"; echo libwolfssl > /etc/modules-load.d/wolfssl.conf
# 4. Install Ceph from the local repository:
#    Allow apt to read the repository under /root:
echo 'APT::Sandbox::User "root";' > /etc/apt/apt.conf.d/99ceph-repo-sandbox
#    Add the distributed repository:
echo "deb [trusted=yes] file:$A/ceph-repo ./" > /etc/apt/sources.list.d/wolfceph.list
apt-get update
apt-get install -y ceph ceph-mds ceph-volume ceph-mgr-dashboard
#    Verify the patched package versions:
dpkg -l ceph-common ceph-volume | grep -c '+wolfcryptfips' \
  | grep -qx 2 && echo "OK: patched FIPS Ceph installed" \
  || { echo "FATAL: STOCK Ceph installed, not the +wolfcryptfips build — the local repo did not win"; exit 1; }
#    Confirm patch 9997 in the installed file:
grep -q "aes-cbc-essiv:sha256" /usr/lib/python3/dist-packages/ceph_volume/util/encryption.py \
  && echo "OK: ceph-volume carries the FIPS LUKS parameters" \
  || { echo "FATAL: ceph-volume is missing patch 9997"; exit 1; }
# 5. Install patched cryptsetup AFTER Ceph, which depends on stock cryptsetup-bin.
# dpkg -i clears package holds; apply the hold after this and any later dpkg -i of these packages.
dpkg -i $A/cryptsetup/libcryptsetup12_*wolfprov1_*.deb $A/cryptsetup/cryptsetup_*wolfprov1_*.deb \
        $A/cryptsetup/cryptsetup-bin_*wolfprov1_*.deb
apt-mark hold cryptsetup cryptsetup-bin libcryptsetup12    # a later apt run must not revert to stock
dpkg-query -W -f='${Version}\n' cryptsetup-bin | grep -q '+wolfprov' \
  && echo "OK: patched cryptsetup installed" \
  || { echo "FATAL: stock cryptsetup on disk, not the +wolfprov1 build"; exit 1; }
# 6. Boot the CRNG kernel with fips=0 until §9:
grep -q 'fips=' /etc/default/grub || sed -i 's/^\(GRUB_CMDLINE_LINUX_DEFAULT="[^"]*\)"/\1 fips=0"/' /etc/default/grub
update-grub
# Pin the CRNG kernel; this also regenerates GRUB configuration:
proxmox-boot-tool kernel pin "$VER"
proxmox-boot-tool kernel list         # -> "Pinned kernel:" names the CRNG kernel
```

**Important:** Retain this kernel [pin](https://pve.proxmox.com/wiki/Host_Bootloader). Proxmox normally
recommends removing it to receive kernel security updates, but this deployment requires the custom
kernel; a stock kernel does not provide the wolfCrypt kernel cryptography.

Determine whether this node boots with UEFI Secure Boot:

```bash
mokutil --sb-state
#   SecureBoot enabled -> §6.2
#   otherwise          -> §6.3
```

### 6.2 Enroll the signing certificate (Secure Boot)

Queue the signing certificate for MOK enrollment before the §6.3 reboot:

```bash
# `Already in kernel trusted keyring. Skip` needs no further action (§11).
mokutil --timeout -1                                                  # do not time the prompt out
mokutil --import /root/artifacts/secureboot/kernel-signing-key.x509   # choose a one-time password
mokutil --list-new | grep -i -A2 wolfCrypt
```

Open the node console for the §6.3 reboot and complete Proxmox's
[MOK enrollment procedure](https://pve.proxmox.com/wiki/Secure_Boot_Setup#Enrolling_the_MOK).

**Caution:** Attend the console during the §6.3 reboot. An unanswered enrollment prompt discards the
request; `mokutil --list-new` then returns empty output with status 0, and the custom kernel fails
with `error: bad shim signature` (§11). `mokutil --timeout -1` disables the prompt's timeout.

### 6.3 Reboot and configure the node

Run this subsection **on every node**, including the first. On a Secure Boot node, keep the console open
and complete §6.2's enrollment during this reboot.

```bash
reboot
```

After the reboot, confirm the kernel and the at-rest cipher before configuring the node:

```bash
uname -r                              # -> 7.0.12-1+wolfcrng1-pve
grep -q cbc-aes-wolfcrypt-fips /proc/crypto && echo "at-rest cipher registered"
```

Configure wolfProvider for Ceph daemons, corosync, and the CLI:

```bash
# Ceph daemons and corosync are not yet running; no restart is needed.
for u in ceph-mon@ ceph-mgr@ ceph-osd@ ceph-mds@ corosync ; do
  mkdir -p /etc/systemd/system/${u}.service.d
  printf '[Service]\nEnvironment=OPENSSL_CONF=/etc/wolfprovider/openssl.cnf\n' \
    > /etc/systemd/system/${u}.service.d/wolfprov.conf
done
systemctl daemon-reload
# CLI tools (ceph, rbd) in PAM sessions:
grep -qxF 'OPENSSL_CONF=/etc/wolfprovider/openssl.cnf' /etc/environment \
  || echo 'OPENSSL_CONF=/etc/wolfprovider/openssl.cnf' >> /etc/environment
```

**Caution:** Apply the next block in the same session; do not log out first. The `/etc/environment`
setting makes OpenSSH's defaults fail with `ssh_dispatch_run_fatal: … invalid argument`, breaking
SSH, `pvecm add`, `qm migrate`, and PVE HA until the algorithms below are configured.

```bash
FIPS_SSH='KexAlgorithms ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group14-sha256,diffie-hellman-group16-sha512
Ciphers aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-256,hmac-sha2-512,hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
HostKeyAlgorithms rsa-sha2-512,rsa-sha2-256,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521
PubkeyAcceptedAlgorithms rsa-sha2-512,rsa-sha2-256,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521'
printf '%s\n' "$FIPS_SSH" | tee /etc/ssh/ssh_config.d/10-wolfprov-fips.conf > /etc/ssh/sshd_config.d/10-wolfprov-fips.conf
# generate ecdsa/rsa host keys if this node has only an ed25519 one:
[ -f /etc/ssh/ssh_host_ecdsa_key ] || ssh-keygen -t ecdsa -b 256  -f /etc/ssh/ssh_host_ecdsa_key -N ''
[ -f /etc/ssh/ssh_host_rsa_key ]   || ssh-keygen -t rsa   -b 3072 -f /etc/ssh/ssh_host_rsa_key   -N ''
systemctl restart ssh
```

**Important:** A node with only an ed25519 host key cannot join or accept a cluster join; SSH fails
before authentication.

`pvedaemon` does not inherit `/etc/environment`. Install a QEMU wrapper to route guest storage I/O
through wolfProvider:

```bash
# Remove the kvm symlink first; writing through it overwrites QEMU and causes an exec loop.
rm -f /usr/bin/kvm
cat > /usr/bin/kvm <<'EOF'
#!/bin/bash
export OPENSSL_CONF=/etc/wolfprovider/openssl.cnf
# Preserve argv[0]=kvm to enable KVM; without it, QEMU uses software emulation and cpu: host fails.
exec -a "$0" /usr/bin/qemu-system-x86_64 "$@"
EOF
chmod +x /usr/bin/kvm
```

**Important:** Install the wrapper on every node, including HA failover targets. A guest started on
a node without it uses system OpenSSL for storage I/O. §9 checks the wrapper on every node.

**Caution:** Upgrading `pve-qemu-kvm` replaces the wrapper with the stock symlink. Hold the package
in §8. Reinstalling the same version also replaces the wrapper and clears the hold, even when held.
After either operation, restore the wrapper and hold, then run the §9 checks.

**Note:** PBS restores also use `/usr/bin/pbs-restore`, which this wrapper does not cover. Complete
Node Configuration and Recovery guide §2 before performing a PBS restore.

Continue in §6.4 if this is the first node, or §6.5 if it is not.

### 6.4 Bootstrap the first node

Run this subsection **on the first node only**, in one session, once §6.3 has completed on it.

Create the cluster:

```bash
pvecm create <cluster-name>
```

Switch corosync from NSS to OpenSSL. In `/etc/pve/corosync.conf`, set `crypto_model: openssl`, keep
`crypto_cipher: aes256` and `crypto_hash: sha256`, and increment `config_version` in `totem`:

```
totem {
  version: 2
  cluster_name: <cluster-name>
  config_version: 2          # increment this on every edit
  crypto_model: openssl      # replaces the default NSS backend
  crypto_cipher: aes256
  crypto_hash: sha256
  interface { ... }
}
```

**Caution:** A syntax error in `corosync.conf` can break quorum. Edit it while the first node is the
only member; joining nodes inherit the setting.

Restart corosync to pick up the backend and the §6.3 drop-in:

```bash
systemctl restart corosync
pvecm status | grep Quorate                       # -> Yes
grep -q libwolfprov /proc/$(pidof corosync)/maps && echo "corosync mapped to wolfProvider"
```

**Important:** Both `crypto_model: openssl` and the §6.3 drop-in are required. Without the drop-in,
corosync uses stock OpenSSL.

Initialize Ceph and place the first monitor:

```bash
pveceph init --network <ceph-public-network-cidr>
pveceph mon create
```

**Note:** `--network` sets Ceph's **public network**, in which every node must already hold an address
or `pveceph mon create` fails on that node. It is not the corosync link, and not the optional
`--cluster-network`; see the [Proxmox Ceph chapter](https://pve.proxmox.com/pve-docs/chapter-pveceph.html).

After creating the monitor, set cluster-wide msgr2 secure mode and select the OpenSSL backend:

```bash
for kv in "global ms_cluster_mode secure" "global ms_service_mode secure" \
          "global ms_client_mode secure"  "mon ms_mon_cluster_mode secure" \
          "mon ms_mon_service_mode secure" "mon ms_mon_client_mode secure" \
          "global ms_bind_msgr2 true" "global ms_bind_msgr1 false" \
          "global plugin_crypto_accelerator crypto_openssl" ; do ceph config set $kv ; done
```

**Important:** Apply these settings before creating any further daemon or OSD on any node. Daemons
read them at startup. `plugin_crypto_accelerator crypto_openssl` selects the provider instead of
Ceph's built-in crypto; an OSD created earlier transfers its key without wolfProvider.

Place this node's manager and its encrypted OSDs:

```bash
pveceph mgr create
pveceph osd create /dev/<disk> --encrypted    # repeat per blank, unpartitioned data device; not the OS disk
# Verify each with luksDump (§9).
```

**Note:** Patch `9997` selects the required cipher and KDF; no extra `pveceph` arguments are needed.
Verify the resulting OSD header in §9.

**Note:** `HEALTH_WARN` is expected until other nodes supply OSDs for replication (§6.5).
§9 requires `HEALTH_OK`.

### 6.5 Bring up each remaining node

Run this subsection **on each remaining node**, once §6.3 has completed on that node and §6.4 on the
first node.

Join the cluster, running the command **on the joining node**:

```bash
pvecm add <first-node-ip>
```

**Important:** Install the §6.3 drop-in before joining. Corosync starts immediately with the inherited
cluster configuration; without the drop-in it uses system OpenSSL until reconfigured and restarted.

Place this node's monitor, its manager, and its encrypted OSDs:

```bash
pveceph mon create
pveceph mgr create                            # on each node that will host a manager (two or more for HA)
pveceph osd create /dev/<disk> --encrypted    # per blank data device on this node
```

**Note:** [Three monitors are needed for quorum](https://pve.proxmox.com/pve-docs/chapter-pveceph.html);
expect an even count until the third node completes this subsection.

Once every node has completed this subsection, confirm corosync routing across the membership:

```bash
grep -q libwolfprov /proc/$(pidof corosync)/maps \
  && ! grep -qi libnss /proc/$(pidof corosync)/maps \
  && echo "corosync on wolfProvider, no NSS"    # run on each member
```

### 6.6 Configure dashboard TLS

Run once from any node, except for the restart: run that on the node hosting the **active manager**.

```bash
# Generate the certificate before §9; key generation is unavailable with fips=1.
openssl req -x509 -newkey rsa:2048 -nodes -keyout dashboard.key -out dashboard.crt \
    -days 3650 -subj "/CN=$(hostname -f)/O=your-org"
ceph config set mgr mgr/dashboard/ssl true
ceph config-key set mgr/dashboard/crt -i dashboard.crt
ceph config-key set mgr/dashboard/key -i dashboard.key
ceph mgr module enable dashboard
# the dashboard does not bind TLS until the active manager restarts:
ceph mgr stat                              # reports the active manager's name (usually a short hostname)
systemctl restart ceph-mgr@<active-mgr>    # run this on that node
ceph mgr services        # must now show "dashboard": "https://…:8443/"
```

**Note:** The dashboard follows the active manager on failover. Probe `<active-mgr>:8443`, not a
fixed address.

---

## 7. High availability and guest virtual machines

Every step runs **once, from any node**; the QEMU scoping each node needs was installed in §6.3. The
guest network bridge is the §1.1 precondition.

### 7.1 RBD storage

Create a pool for guest disks:

```bash
pveceph pool create <pool-name> --pg_autoscale_mode on
```

**Note:** The default `warn` autoscale mode leaves the cluster at `HEALTH_WARN`, which §9 and §10 do not
accept. For an existing pool, run `ceph osd pool set <pool-name> pg_autoscale_mode on`.

Then define the pool as Proxmox storage, with `krbd` set explicitly rather than inherited:

```bash
pvesm add rbd <storage-id> --pool <pool-name> --content images --krbd 0
pvesm status | grep <storage-id>          # active on every node
```

**Important:** Use `krbd 0` to select QEMU's userspace librbd client. Kernel RBD does not use
wolfProvider and is not supported. Set the option explicitly rather than using
`pveceph pool create --add_storages`, which leaves it at its default.

**Note:** This storage holds disk images only. Do not add `rootdir` — containers on RBD
[use krbd regardless of the `krbd` option](https://pve.proxmox.com/pve-docs/chapter-pvesm.html#storage_rbd_config).
Backup archives need a file-level storage, which this deployment does not create — see the Backup guide.

### 7.2 Guest high availability

Create the guest on the §7.1 storage and the §1.1 bridge, as on any Proxmox cluster. Then place it under
[high availability](https://pve.proxmox.com/pve-docs/chapter-ha-manager.html):

```bash
ha-manager add vm:<vmid> --state started
ha-manager status                       # fencing armed via softdog watchdog
```

Confirm that the running guest's QEMU is scoped to the provider:

```bash
grep -q libwolfprov /proc/$(cat /var/run/qemu-server/<vmid>.pid)/maps \
  && echo "guest QEMU mapped to wolfProvider"
```

---

## 8. Hold the FIPS packages

Hold the packages on **every node** to prevent replacement by stock packages during upgrades.
Use `apt-mark hold`; current trixie `apt` ignores an `/etc/apt/preferences.d` version pin.

```bash
apt-mark hold ceph ceph-base ceph-common ceph-mds ceph-mgr ceph-mgr-dashboard ceph-mon ceph-osd \
  ceph-volume libcephfs2 librados2 librbd1 python3-ceph python3-cephfs python3-rados python3-rbd
apt-mark showhold | grep -q ceph-common && echo "FIPS Ceph held against point-release override"
# Confirm the cryptsetup hold from §6.1 step 5:
apt-mark showhold | grep -q cryptsetup-bin && echo "patched cryptsetup held"
# Hold pve-qemu-kvm to preserve the /usr/bin/kvm wrapper:
apt-mark hold pve-qemu-kvm
apt-mark showhold | grep -q pve-qemu-kvm && echo "kvm wrapper protected from upgrade"
```

---

## 9. Verify the deployment

Set `fips=1` and reboot **one node at a time**; rebooting all nodes together causes a cluster outage.
Set `noout` first to prevent rebalancing during the reboots:

```bash
ceph osd set noout                                      # cluster-wide; run once
# then, on one node at a time:
sed -i 's/fips=0/fips=1/' /etc/default/grub && update-grub && reboot
# wait for the node to rejoin and the cluster to return to HEALTH_OK (ceph -s) before the next node
ceph osd unset noout                                    # once every node is back and HEALTH_OK
```

**Note:** Secure Boot reboots are unattended after §6.2 is complete.

Run the following checks on **every node**. Confirm each expected result and stop if any check
fails; not all checks stop execution automatically.

**Kernel and RNG**

```bash
cat /proc/sys/crypto/fips_enabled            # -> 1
grep -o 'fips=[01]' /proc/cmdline            # -> fips=1; fips_enabled alone does not confirm the boot flag
uname -r                                     # -> 7.0.12-1+wolfcrng1-pve
dmesg | grep -iE 'wolfCrypt.*startup self-test succeeded|registered as systemwide default stdrng'
lsmod | grep '^libwolfssl'                   # in-kernel module loaded (at-rest + stdrng)
test $(( $(cat /proc/sys/kernel/tainted) & 8192 )) -eq 0 && echo "no unsigned-module taint"
```

**Secure Boot — enabled nodes only**

```bash
mokutil --sb-state                           # -> SecureBoot enabled
```

**Userspace provider**

The FIPS provider must be active and refuse MD5:

```bash
openssl list -providers | grep -i -A2 'wolfSSL Provider FIPS'   # -> status: active
echo x | openssl dgst -md5 && echo "BAD: MD5 served" || echo "OK: MD5 refused (FIPS)"
```

**Ceph packages and daemons**

```bash
dpkg -l ceph-common ceph-volume | grep -c '+wolfcryptfips'   # -> 2, else STOCK Ceph on this node
# Confirm patch 9997 in the installed ceph-volume:
grep -q "aes-cbc-essiv:sha256" /usr/lib/python3/dist-packages/ceph_volume/util/encryption.py \
  || { echo "FATAL: ceph-volume carries the version string but not patch 9997"; exit 1; }

# Each running Ceph daemon must have OPENSSL_CONF set and wolfProvider mapped:
for u in $(systemctl list-units --state=running --no-legend 'ceph-*@*' | awk '{print $1}'); do
  pid=$(systemctl show -p MainPID --value "$u")
  tr '\0' '\n' </proc/$pid/environ | grep -q OPENSSL_CONF && grep -q libwolfprov /proc/$pid/maps \
    && echo "$u OK" || { echo "FAIL: $u is not scoped to the provider"; exit 1; }
done
```

**QEMU wrapper and package hold**

```bash
test ! -L /usr/bin/kvm && grep -q 'exec -a "$0"' /usr/bin/kvm && echo "OK: kvm wrapper intact" \
  || { echo "FAIL: /usr/bin/kvm is the stock symlink — guests are not scoped"; exit 1; }
apt-mark showhold | grep -qx pve-qemu-kvm && echo "OK: pve-qemu-kvm held" \
  || { echo "FAIL: pve-qemu-kvm is not held — an upgrade will un-scope every guest"; exit 1; }
```

**OSD cipher**

Use the `[block]` path from `ceph-volume lvm list` for `<osd-dev>`, plus `[db]`/`[wal]` if configured.
These are LVM logical volumes: the `lvm` layer in `lsblk` with a `crypt` child, not the raw disk or
the opened `crypt` mapping. Confirm both the cipher and PBKDF in each header:

```bash
cryptsetup luksDump <osd-dev> | grep -iE 'aes-cbc-essiv:sha256|PBKDF:.*pbkdf2'   # both present
grep cbc-aes-wolfcrypt-fips-140-3 /proc/crypto                                   # wolfcrypt cbc winning
```

**OSD keyslot cryptography**

For `<osd-crypt-name>`, use the opened `crypt` mapping below the LVM volume in `lsblk`, not
`<osd-dev>`. Unsetting `OPENSSL_CONF` checks that routing is compiled into `cryptsetup`:

```bash
dpkg-query -W -f='${Version}\n' cryptsetup-bin | grep -q '+wolfprov' && echo "OK: patched cryptsetup"
env -u OPENSSL_CONF cryptsetup --debug status <osd-crypt-name> | grep -i 'Crypto backend'
#   -> Crypto backend (OpenSSL 3.x ... [libwolfprov][legacy][threads][argon2]) initialized
# [argon2] denotes cryptsetup's compiled-in support, not wolfProvider support.
```

**Corosync**

```bash
grep -q libwolfprov /proc/$(pidof corosync)/maps && ! grep -qi 'libnss' /proc/$(pidof corosync)/maps \
  && echo "corosync on wolfProvider, no NSS"
```

Run the remaining checks **once, from any node**.

**msgr2 secure transport**

```bash
ceph --debug-ms 1 health 2>&1 >/dev/null | grep -c ' secure :'   # > 0
ceph --debug-ms 1 health 2>&1 >/dev/null | grep -c ' crc :' || true   # = 0 (grep -c exits 1 on none)
```

**Dashboard TLS**

Identify the active manager with `ceph mgr services`. Use the stock client (`env -u OPENSSL_CONF`)
for these probes; a FIPS-only client aborts the handshake on its own side.

```bash
env -u OPENSSL_CONF openssl s_client -connect <active-mgr>:8443 -tls1_3 </dev/null 2>/dev/null \
  | grep 'Cipher is TLS_AES'    # -> TLS_AES_256_GCM_SHA384
# A ChaCha-only client must be refused; AES success alone does not confirm the cipher restriction.
env -u OPENSSL_CONF openssl s_client -connect <active-mgr>:8443 -tls1_3 \
  -ciphersuites TLS_CHACHA20_POLY1305_SHA256 </dev/null 2>/dev/null \
  | grep -q 'Cipher is TLS_CHACHA20' && echo "BAD: ChaCha-only client served" || echo "OK: ChaCha-only client refused"
```

### 9.1 Continue

After all checks pass, continue with:

- **Node Configuration and Recovery** — configure `pveproxy` (still on system OpenSSL), administrative
  access recovery, and rollback.
- **Backup** — configure and verify backups.

---

## 10. Maintenance

Follow the Proxmox VE Administration Guide except for the differences below. These procedures retain
`fips_enabled=1` and msgr2 secure for backfill and recovery.

### 10.1 Replace a failed OSD disk

Remove the failed OSD using Proxmox's
[procedure](https://pve.proxmox.com/pve-docs/chapter-pveceph.html) — `ceph osd out`, `ceph osd
ok-to-stop`, `pveceph stop`, `pveceph osd destroy --cleanup`. Then create an **encrypted** replacement
on the node whose disk failed:

```bash
# The replacement uses wolfCrypt cbc(aes) and a fresh volume key from the CRNG DRBG.
pveceph osd create /dev/<disk> --encrypted
ceph -s                                        # degraded -> backfill -> HEALTH_OK
```

**Important:** The replacement requires patched `ceph-volume` and `cryptsetup`. Stock packages
create OSDs with aes-xts and argon2id without reporting an error. Verify the packages and new header:

```bash
# Check patch 9997 in the installed file:
grep -q "aes-cbc-essiv:sha256" /usr/lib/python3/dist-packages/ceph_volume/util/encryption.py \
  || { echo "FATAL: ceph-volume is not carrying the FIPS LUKS parameters (patch 9997)"; exit 1; }
dpkg-query -W -f='${Version}\n' cryptsetup-bin | grep -q '+wolfprov' && echo "OK: patched cryptsetup"
# <osd-dev> is the new OSD's LVM logical volume, from `ceph-volume lvm list`: the [block] path, plus
# [db]/[wal] if configured.
# Require both the cipher and PBKDF:
cryptsetup luksDump <osd-dev> | grep -qi 'aes-cbc-essiv:sha256' \
  && cryptsetup luksDump <osd-dev> | grep -qiE 'PBKDF:[[:space:]]*pbkdf2' \
  && echo "OK: OSD header inside the boundary" \
  || { echo "FATAL: OSD header is not aes-cbc-essiv:sha256 + pbkdf2"; exit 1; }
```

### 10.2 Add a node

Distribute the artifacts (§5.3), run §6.1 to §6.3 — with §6.2 on a Secure Boot node — then §6.5.

**Important:** Check the monitor count after adding or re-adding a node. `HEALTH_OK` does not detect
a monitor missing from the monmap; `mon@-1(probing)` indicates this condition (§11).

```bash
ceph mon stat                                  # -> the expected number of mons, all in quorum
```

Run the §9 checks on the new node before placing workload on it.

### 10.3 Remove a node

Follow the Proxmox VE Administration Guide's
[node-removal procedure](https://pve.proxmox.com/pve-docs/chapter-pvecm.html). First destroy the node's
Ceph services: monitor, manager, and OSDs. Power off the node; do not restart it with its old cluster
configuration. Reinstall before rejoining through §10.2.

### 10.4 Deploy a same-version rebuild

For a same-version rebuild, `apt-get install` reports `0 upgraded, 0 newly installed` and leaves
the old files in place. Reinstallation is required.

Replace the repository contents, then on each node in turn:

```bash
apt-get update                                 # required: the indices changed under an unchanged version
apt-get install --reinstall -y ceph-volume     # and any other package the rebuild corrected
grep -q "aes-cbc-essiv:sha256" /usr/lib/python3/dist-packages/ceph_volume/util/encryption.py \
  || { echo "FATAL: the corrected package did not land"; exit 1; }
```

**Important:** Verify file contents, not the unchanged version. Run `apt-get update` first;
otherwise installation fails with `Hash Sum mismatch`.

### 10.5 Upgrade Ceph

A Ceph point-release requires rebuilding only the Ceph packages, not the CRNG kernel, wolfProvider
pair, or `libwolfssl.ko`. Confirm all three patches apply to the new source with `quilt push -a`.
Update any failing patch; do not omit it. A package can build and run without the required patches.

```bash
# Rebuild for the new release (§5.1), distribute (§5.3), then run on one node at a time:
apt-mark unhold ceph ceph-base ceph-common ceph-mds ceph-mgr ceph-mgr-dashboard ceph-mon ceph-osd \
  ceph-volume libcephfs2 librados2 librbd1 python3-ceph python3-cephfs python3-rados python3-rbd  # the full §8 set
apt-get install -y ceph ceph-mds ceph-volume ceph-mgr-dashboard        # pulls the new +wolfcryptfips build
apt-mark hold   ceph ceph-base ceph-common ceph-mds ceph-mgr ceph-mgr-dashboard ceph-mon ceph-osd \
  ceph-volume libcephfs2 librados2 librbd1 python3-ceph python3-cephfs python3-rados python3-rbd
# re-check after each node before moving on:
dpkg -l ceph-common ceph-volume | grep -c '+wolfcryptfips'            # -> 2
grep -q "aes-cbc-essiv:sha256" /usr/lib/python3/dist-packages/ceph_volume/util/encryption.py \
  || { echo "FATAL: the rebuild dropped patch 9997"; exit 1; }
ceph -s                                                              # HEALTH_OK before the next node
```

**Note:** Ceph packages do not own `/etc/wolfprovider/openssl.cnf` or the
`ceph-*@.service.d/wolfprov.conf` drop-ins, so Ceph upgrades preserve them.

**Important:** Upgrading `pve-qemu-kvm` restores stock `/usr/bin/kvm` and `/usr/bin/pbs-restore`
and clears the hold (§6.3). Run `apt-mark unhold pve-qemu-kvm`, upgrade, restore **both** wrappers
(§6.3 and Node Configuration and Recovery guide §2), reapply the hold, and run the §9 wrapper check.

---

## 11. Troubleshooting

**`pveceph init` or `pveceph mon create` fails with `Ceph Nautilus required — run 'pveceph install'`** —
Ceph was built without `.git`, leaving `CEPH_GIT_VER` empty and the version as `20.2.2 () tentacle`.
Rebuild §5.1 from a checkout that retains `.git`.

**`collect2: ld returned 1` after resuming a killed Ceph build** — an interrupted build can leave
corrupt objects that `dpkg-buildpackage -nc` reuses. Remove `obj-x86_64-linux-gnu` and rebuild.

**Dependency-resolution failure during §6.1 step 4** — the `ceph-repo/` copy may be incomplete.
Repeat the §5.3 count checks against the builder and re-copy.

**`mokutil --import` reports `Already in kernel trusted keyring. Skip`** — the node already trusts the
certificate; no action is needed. The subsequent `mokutil --list-new` pipeline returns non-zero
because there is no pending enrollment.

**The node stops at GRUB with `error: bad shim signature`** — the boot default is the custom kernel but
its certificate is not enrolled: the enrollment prompt was discarded, or Foundation §2.6 was skipped.
Boot a stock Proxmox kernel from GRUB **Advanced options**, confirm Foundation §2.6, and repeat the
§6.2 enrollment.

**`cryptsetup` fails during `pveceph osd create --encrypted`** — patched `cryptsetup` fails if the
wolfProvider pair is missing or unreadable; it does not fall back to stock OpenSSL. Check §6.1 step 1
on that node.

**`pveceph osd destroy --cleanup` prints `pvremove … exit code 5`** — if the destroy otherwise
succeeds, the message is harmless.

**A monitor shows as `mon@-1(probing)` after a node is added** — the monitor is outside the monmap, and
`ceph -s` still reports `HEALTH_OK`. Add it with `ceph mon add <name> <addr>` and confirm with
`ceph mon stat`.
