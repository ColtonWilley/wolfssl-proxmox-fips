# wolfSSL FIPS on Proxmox VE — LUKS / dm-crypt Deployment

Deploy LUKS/dm-crypt storage after completing the **Shared Build Foundation** guide. Sector encryption
uses the in-kernel wolfCrypt FIPS module; keyslot cryptography uses wolfProvider. This guide configures
the storage, SSH, and boot-time FIPS enforcement, then verifies the deployment.

This guide covers local guest disks only. Use the Ceph guide for distributed or shared storage with
on-wire cryptography; for shared-nothing clusters, use the Cluster guide.

---

## 1. Cryptographic components

dm-crypt binds wolfCrypt because the in-kernel module registers `cbc(aes)` ahead of the kernel's
native AES implementations.

| Layer | Where it runs | wolfCrypt-FIPS component |
|---|---|---|
| Bulk sector encryption (`aes-cbc-essiv:sha256`) | kernel, dm-crypt | in-kernel `libwolfssl.ko` (kernel crypto API `cbc(aes)`) |
| Volume-key entropy (master key) | kernel RNG | wolfCrypt FIPS DRBG, registered as the system `stdrng` |
| Keyslot KDF (PBKDF2), keyslot AES, and digests | userspace, libcryptsetup | wolfProvider, through the patched cryptsetup (§4) |
| SSH key exchange, host keys, ciphers, and MACs, inbound | userspace, `sshd` | wolfProvider (§9) |
| SSH, outbound (`ssh`) | userspace | System OpenSSL, not wolfProvider. Algorithm restrictions in §9; `root` override in §12.5. |

**Note:** Management TLS — the web interface and the API on `pveproxy` — is covered in the **Node
Configuration and Recovery** guide, which follows this one.

**Important:** Use `--cipher aes-cbc-essiv:sha256` and `--pbkdf pbkdf2` on every `luksFormat`, and
`--pbkdf pbkdf2` on every `luksAddKey`. The LUKS2 defaults are unsuitable: Argon2id fails with
`Not compatible PBKDF options.`, but `aes-xts-plain64` silently uses native kernel AES instead of
wolfCrypt. Check the cipher with `luksDump` in §6.

---

## 2. Requirements

### 2.1 The node

Use a fresh **Proxmox VE 9.2** installation with an **ext4/LVM root** and a **second disk** for the
encrypted volume. Keeping the volume out of the boot path and initramfs lets the node boot if its
crypto module fails. Confirm SSH access with an ECDSA or RSA administrative key (Foundation requirements).

**Note:** For Secure Boot, complete Foundation §2.6 before §5, then use §5.2 and §5.4. Otherwise use
§5.1. Every node reboots and confirms in §5.5.

### 2.2 The shared foundation

Complete the **Shared Build Foundation** guide in full before starting here. Retain on the builder the
unpacked wolfSSL FIPS source, the installed wolfProvider matched pair, the CRNG FIPS kernel `.deb` with
matching headers, the module-signing key, and the kernel build tree this guide compiles against.

### 2.3 Version pins

| Component | Version | Source pin |
|---|---|---|
| Proxmox VE | **9.2.x** (Debian 13 "Trixie"), **ext4/LVM root** | — |
| Kernel | **proxmox-kernel-7.0.12-1+wolfcrng1-pve** | built in the Foundation guide from pve-kernel commit `85c2f14` |
| wolfSSL (FIPS) | **5.9.1**, built from your licensed wolfSSL FIPS source bundle | the FIPS source bundle obtained from wolfSSL |
| wolfProvider | **v1.2.1** | tag `v1.2.1` |
| cryptsetup | Debian **2.7.5-2** plus the wolfProvider routing change → built as **`2.7.5-2+wolfprov1`** | §4 |

### 2.4 Conventions

Run every command as **root**. Replace `<placeholders>` with your values; enter version pins verbatim.
Configuration blocks show the contents of the named file.

---

## 3. Build the in-kernel FIPS `libwolfssl.ko`

Register `cbc(aes),sha2,stdrng-default` for dm-crypt's cipher and hash and the system DRBG.

```bash
source /root/wolf-buildenv.sh
cd /root/wolfssl-fips-src && make clean >/dev/null 2>&1
./configure --enable-fips=<your-bundle-config> --enable-linuxkm \
    --enable-linuxkm-lkcapi-register='cbc(aes),sha2,stdrng-default' --with-linux-source="$KSRC"
# Disable objtool for the FIPS boundary assembly and generate the in-core integrity hash.
# Use libwolfssl.ko, not the optional .signed output; §5.3 signs it with the retained key.
make -j"$(nproc)" KBUILD_EXTRA_FLAGS=FORCE_GLOBAL_OBJTOOL_OFF=1 module-with-matching-fips-hash || true
ls -l linuxkm/libwolfssl.ko                          # MUST exist (do NOT strip it)
```

**Important:** Do not `strip` this module. Stripping changes the sections over which the in-core FIPS
integrity hash is computed and breaks the power-on self-test.

---

## 4. Build cryptsetup against wolfProvider

libcryptsetup's OpenSSL backend loads the default provider. The change below makes it load
`libwolfprov` instead, which routes the keyslot PBKDF2, the keyslot AES, and the digests through
wolfCrypt.

```bash
# Enable deb-src for apt-get source:
grep -q '^Types: deb deb-src' /etc/apt/sources.list.d/debian.sources \
  || sed -i 's/^Types: deb$/Types: deb deb-src/' /etc/apt/sources.list.d/debian.sources
apt-get update
# Rebuild from a clean tree so dch produces +wolfprov1 and the check cannot match stale packages:
rm -rf /root/cs-build
mkdir -p /root/cs-build && cd /root/cs-build && apt-get source cryptsetup && cd cryptsetup-2.7.5
apt-get build-dep -y cryptsetup
# Route the OpenSSL 3.x backend to wolfProvider:
F=lib/crypto_backend/crypto_openssl.c
sed -i 's/OSSL_PROVIDER_try_load(ossl_ctx, "default", 0)/OSSL_PROVIDER_try_load(ossl_ctx, "libwolfprov", 0)/' "$F"
sed -i 's|ossl_legacy = OSSL_PROVIDER_try_load(ossl_ctx, "legacy", 0);|ossl_legacy = ossl_default;|' "$F"
sed -i 's/ossl_default ? "\[default\]" : ""/ossl_default ? "[libwolfprov]" : ""/' "$F"
grep -n '"libwolfprov"' "$F"                          # confirm the edit
DEBEMAIL="you@example.com" DEBFULLNAME="wolfSSL FIPS" dch -l +wolfprov -D trixie --nomultimaint \
    "Route libcryptsetup OpenSSL backend through wolfProvider (FIPS)."
# nocheck: the upstream self-tests use non-FIPS vectors and would need wolfprov installed at build time.
DEB_BUILD_OPTIONS=nocheck dpkg-buildpackage -b -uc -us
ls /root/cs-build/libcryptsetup12_*wolfprov1_*.deb /root/cs-build/cryptsetup*_*wolfprov1_*.deb
```

---

## 5. Install and boot the FIPS kernel

Boot with `fips=0` while provisioning storage; §10 sets `fips=1`. Loading the wolfCrypt module already
enables kernel FIPS mode and supplies `cbc(aes)` for provisioning.

Which kernel package to install depends on whether the node boots with UEFI Secure Boot:

```bash
mokutil --sb-state
#   SecureBoot enabled -> §5.2
#   otherwise          -> §5.1
```

### 5.1 Install the kernel — Secure Boot disabled

```bash
source /root/wolf-buildenv.sh
VER=7.0.12-1+wolfcrng1-pve
# the build-deps metapackage conflicts with the headers pkg -> remove it first:
apt-get remove -y proxmox-kernel-7.0-build-deps 2>/dev/null || true
dpkg -i /root/pve-kernel/proxmox-kernel-${VER}_*.deb /root/pve-kernel/proxmox-headers-${VER}_*.deb
```

Continue at §5.3.

### 5.2 Install the kernel — Secure Boot enabled

The kernel package is the **signed** one produced in the Foundation guide (§2.6), from
`/root/signed-debs/`. The headers package is installed unchanged.

```bash
source /root/wolf-buildenv.sh
VER=7.0.12-1+wolfcrng1-pve
# the build-deps metapackage conflicts with the headers pkg -> remove it first:
apt-get remove -y proxmox-kernel-7.0-build-deps 2>/dev/null || true
dpkg -i /root/signed-debs/proxmox-kernel-${VER}_*.deb /root/pve-kernel/proxmox-headers-${VER}_*.deb
```

### 5.3 Install the module and cryptsetup, and pin the boot default

```bash
source /root/wolf-buildenv.sh
VER=7.0.12-1+wolfcrng1-pve
# Install the unstripped module, sign it, and load it at boot before the LUKS unlock unit:
mkdir -p /lib/modules/$VER/updates
cp /root/wolfssl-fips-src/linuxkm/libwolfssl.ko /lib/modules/$VER/updates/libwolfssl.ko
SIGN=/lib/modules/$VER/build/scripts/sign-file
"$SIGN" sha512 /root/kernel-signing-key.pem /root/kernel-signing-key.x509 /lib/modules/$VER/updates/libwolfssl.ko
depmod "$VER"
echo libwolfssl > /etc/modules-load.d/wolfcrypt-lkcapi.conf
# Install patched cryptsetup; it requires wolfProvider installed by the Foundation guide:
dpkg -i /root/cs-build/libcryptsetup12_*wolfprov1_*.deb /root/cs-build/cryptsetup_*wolfprov1_*.deb \
        /root/cs-build/cryptsetup-bin_*wolfprov1_*.deb
# add fips=0 to the kernel cmdline (idempotent; §10 sets this to 1):
grep -q 'fips=' /etc/default/grub || \
  sed -i 's/^\(GRUB_CMDLINE_LINUX_DEFAULT="[^"]*\)"/\1 fips=0"/' /etc/default/grub
grep GRUB_CMDLINE_LINUX_DEFAULT /etc/default/grub    # confirm it now contains fips=0
update-grub
# pin the FIPS kernel as the boot default (the tool regenerates the GRUB configuration itself):
proxmox-boot-tool kernel pin "$VER"
proxmox-boot-tool kernel list                         # -> "Pinned kernel:" names the FIPS kernel
```

**Important:** Keep the FIPS kernel [pinned](https://pve.proxmox.com/wiki/Host_Bootloader). Unlike a
standard Proxmox deployment, unpinning would allow a stock kernel to replace it. The pin also prevents
automatic selection of newer kernels with security fixes.

### 5.4 Enroll the signing certificate — Secure Boot enabled

Skip this section where §5.1 applied. Queue the Foundation certificate for MOK enrollment before
rebooting:

```bash
# `Already in kernel trusted keyring. Skip` -> nothing further is needed; see Troubleshooting (§12.1):
mokutil --timeout -1                                  # do not time the enrollment prompt out
mokutil --import /root/kernel-signing-key.x509        # choose a one-time password
mokutil --list-new | grep -i -A2 wolfCrypt
```

Open the node console for the next reboot and complete Proxmox's
[MOK enrollment procedure](https://pve.proxmox.com/wiki/Secure_Boot_Setup#Enrolling_the_MOK).

**Caution:** Attend the console. Without `mokutil --timeout -1`, an unanswered enrollment prompt
silently discards the request; `mokutil --list-new` is then empty but still exits 0. The pinned kernel
cannot boot without enrollment: GRUB reports `error: bad shim signature`, and the node remains
unreachable over the network. Recovery: §12.2.

### 5.5 Reboot and confirm

On a Secure Boot node, keep the console open and complete §5.4's enrollment during this reboot.

```bash
reboot
```

After reboot, confirm the module is loaded and wolfCrypt `cbc(aes)` has the highest priority.
Check the displayed results as well as explicit failures; not every command stops the block.

```bash
uname -r                                  # 7.0.12-1+wolfcrng1-pve
mokutil --sb-state                        # Secure Boot node -> SecureBoot enabled
cat /proc/sys/crypto/fips_enabled                                             # 1 (the wolfCrypt module enables it on load)
grep -o 'fips=[01]' /proc/cmdline                                             # fips=0 — set to 1 in §10
# Check unsigned-module taint bit 13 (8192), not the whole bitmask:
[ $(( $(cat /proc/sys/kernel/tainted) & 8192 )) -eq 0 ] || { echo "FAIL: unsigned-module taint (8192) set"; exit 1; }
# Confirm wolfCrypt cbc(aes) is registered and wins by priority, not /proc/crypto file order:
awk '/^name +: cbc\(aes\)$/{b=1} b&&/^driver/{d=$3} b&&/^priority/{print d,$3;b=0}' /proc/crypto \
  | grep -q 'cbc-aes-wolfcrypt-fips-140-3' || { echo "FAIL: wolfcrypt cbc(aes) not registered"; exit 1; }
awk '/^name +: cbc\(aes\)$/{b=1} b&&/^driver/{d=$3} b&&/^priority/{print d,$3;b=0}' /proc/crypto
#   -> cbc-aes-wolfcrypt-fips-140-3 100000   (beats cbc-aes-aesni 400)
# Check wolfProvider with an explicit configuration; the system default does not activate it:
cat > /tmp/wpcheck.cnf <<'EOF'
openssl_conf = openssl_init
[openssl_init]
providers = provider_sect
[provider_sect]
wolfprov = wolfprov_sect
[wolfprov_sect]
module = /usr/lib/x86_64-linux-gnu/ossl-modules/libwolfprov.so
activate = 1
EOF
OPENSSL_CONF=/tmp/wpcheck.cnf openssl list -providers | grep -qi 'wolfSSL Provider FIPS' \
  || { echo "FAIL: wolfProvider not active"; rm -f /tmp/wpcheck.cnf; exit 1; }
rm -f /tmp/wpcheck.cnf
dmesg | grep -iE 'FIPS 140-3 wolfCrypt|fips_enabled from 0 to 1|self-tests for cbc-aes-wolfcrypt'  # POST OK + cbc(aes) self-test passed
# Expect the cbc(aes) self-test here. The ESSIV self-test appears only after a volume opens (§6).
echo "LUKS wolfCrypt foundation OK"
```

---

## 6. Provision the encrypted volume

The service in §7 uses a **key file** for unattended unlock. Interactive passphrase unlock through
`systemd-cryptsetup` is not covered here.

**Important:** The key file resides on the unencrypted root filesystem. This protects a separately
removed data disk, not a stolen node: possession of both disks provides the ciphertext and its key.

**Caution:** `luksFormat` destroys existing data on the target disk. Confirm `DISK` identifies the data
disk, not the root disk.

```bash
mkdir -p /etc/luks && ( umask 077; head -c 512 /dev/urandom > /etc/luks/cryptdata.key )
chmod 400 /etc/luks/cryptdata.key
# use YOUR data disk — prefer by-id, or by-partuuid for a partition:
DISK=/dev/disk/by-id/<your-data-disk>
cryptsetup luksFormat --type luks2 --cipher aes-cbc-essiv:sha256 --key-size 256 \
  --pbkdf pbkdf2 --hash sha256 --pbkdf-force-iterations 600000 \
  --key-file /etc/luks/cryptdata.key --batch-mode "$DISK"
# --cipher sets the keyslot cipher as well; the LUKS2 default would be served outside the module.
# confirm the header before building on it:
cryptsetup luksDump "$DISK" | grep -iE 'Version|Cipher:|Cipher key|PBKDF|Hash|^\s+cipher:'
#   -> Version: 2 ; Cipher: aes-cbc-essiv:sha256 ; 256 bits ; PBKDF: pbkdf2 ; Hash: sha256
#   -> and the keyslot's own `cipher: aes-cbc-essiv:sha256`
UUID=$(blkid -s UUID -o value "$DISK"); echo "$UUID"
```

**`/etc/crypttab`** (`noauto` — the unit in §7 opens the volume):

```
cryptdata  UUID=<paste $UUID>  /etc/luks/cryptdata.key  luks,noauto
```

Open the volume, create a filesystem, mount it, then tear it down cleanly:

```bash
cryptsetup open --type luks2 --key-file /etc/luks/cryptdata.key "$DISK" cryptdata
mkfs.ext4 -q /dev/mapper/cryptdata
mkdir -p /mnt/secure && mount /dev/mapper/cryptdata /mnt/secure
mkdir -p /mnt/secure/{images,dump,private,tmp}; chmod 1777 /mnt/secure/tmp   # 1777, not 700 — see §8
# unmount BEFORE close (closing a mounted mapping fails 'busy' and leaves it open, breaking the unit below):
umount /mnt/secure; cryptsetup close cryptdata
```

---

## 7. Boot-time unlock service

**`/etc/systemd/system/luks-unlock.service`** (substitute your disk's by-id path in `ConditionPathExists`
and in the `open` line):

```ini
[Unit]
Description=Open + mount the LUKS encrypted data disk (fail-closed)
DefaultDependencies=no
After=systemd-modules-load.service cryptsetup.target local-fs.target
Wants=systemd-modules-load.service
Before=pve-guests.service pvescheduler.service
ConditionPathExists=/dev/disk/by-id/<your-data-disk>

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/cryptsetup open --type luks2 --key-file /etc/luks/cryptdata.key /dev/disk/by-id/<your-data-disk> cryptdata
ExecStart=/bin/mount /dev/mapper/cryptdata /mnt/secure
ExecStartPost=/bin/mountpoint -q /mnt/secure
ExecStop=/bin/umount /mnt/secure
ExecStop=/sbin/cryptsetup close cryptdata

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now luks-unlock.service
mountpoint /mnt/secure || { echo "FAIL: /mnt/secure not mounted"; exit 1; }
```

Add a `Requires=` dependency so guests cannot start if unlock fails. `Before=` alone only sets ordering.
Create the drop-in directory before writing the file:

```bash
mkdir -p /etc/systemd/system/pve-guests.service.d
printf '[Unit]\nRequires=luks-unlock.service\n' > /etc/systemd/system/pve-guests.service.d/10-require-luks.conf
systemctl daemon-reload
systemctl show pve-guests.service -p Requires | tr ' ' '\n' | grep luks-unlock   # confirm the edge took
```

**Important:** Load `libwolfssl.ko` before `cryptsetup open`; otherwise ESSIV binds native `cbc(aes)`
for the mapping's lifetime. `After=systemd-modules-load.service` and the module-load file in §5
provide this ordering.

---

## 8. Register the encrypted volume as PVE storage

```bash
# Require a mounted volume for storage activation:
pvesm add dir secure --path /mnt/secure --content images,rootdir,backup --is_mountpoint 1
# remove the default PLAINTEXT guest storage so a disk can never land unencrypted:
pvesm remove local-lvm 2>/dev/null || true
pvesm set local --content vztmpl,iso           # keep 'local' for templates/ISOs only
# vzdump stages to /var/tmp on the unencrypted root when the backup target is a Proxmox Backup Server.
# A running container is backed up in suspend mode, which rsyncs its whole filesystem through staging:
grep -q '^tmpdir:' /etc/vzdump.conf || echo 'tmpdir: /mnt/secure/tmp' >> /etc/vzdump.conf
pvesm status | grep -E 'secure|local'
```

**Caution:** `is_mountpoint 1` blocks guest startup and restore while locked, but not writes to the
path. An offline migration can succeed onto the unencrypted root at `/mnt/secure`; unlocking then
hides that copy beneath the mount. Confirm `secure` is active before moving a guest to the node.

**Important:** Keep `/mnt/secure/tmp` at mode **1777**. An unprivileged container's backup client cannot
traverse a root-owned mode-700 staging directory (§12.4).

Confirm that a real guest lands on the encrypted mount by creating a **VM** and a **CT** on `secure`.
VMID 700 and CTID 710 are examples; any free IDs work:

```bash
qm create 700 --name fipstest --memory 512 --scsihw virtio-scsi-single \
    --scsi0 secure:1 --net0 virtio,bridge=vmbr0
ls -l /mnt/secure/images/700/                # the guest disk image lives on the encrypted mount
qm start 700 && qm status 700               # running
qm stop 700
```

For the container, obtain a template as described in
[Container Images](https://pve.proxmox.com/pve-docs/chapter-pct.html#pct_container_images),
substituting the current template name below:

```bash
pct create 710 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
    --hostname fipsct --memory 512 --rootfs secure:2 --net0 name=eth0,bridge=vmbr0,ip=dhcp \
    --password <a-container-root-password> --unprivileged 1 --features nesting=1
pct start 710 && pct status 710             # running
pct stop 710
```

---

## 9. SSH

Restrict both SSH endpoints to the algorithms below before routing `sshd` through wolfProvider.
OpenSSH implements curve25519, ed25519, and chacha20 internally; they do not use the provider and are
not supported by wolfCrypt FIPS v5. An ed25519-only host key breaks negotiation and Proxmox SSH operations.

Configure the server side:

```bash
cat > /etc/ssh/sshd_config.d/10-wolfprov-fips.conf <<'EOF'
# FIPS-approved SSH algorithms only. No curve25519/ed25519/chacha20 (OpenSSH-internal => not
# provider-routed, not FIPS). HostKey = rsa and ecdsa only (an ed25519 host key would break FIPS sshd).
KexAlgorithms ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group14-sha256,diffie-hellman-group16-sha512
Ciphers aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-256,hmac-sha2-512,hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
HostKeyAlgorithms rsa-sha2-512,rsa-sha2-256,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521
PubkeyAcceptedAlgorithms rsa-sha2-512,rsa-sha2-256,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
EOF
# both host keys must exist, or sshd has no usable identity to present (idempotent):
[ -f /etc/ssh/ssh_host_ecdsa_key ] || ssh-keygen -t ecdsa -b 256  -f /etc/ssh/ssh_host_ecdsa_key -N ''
[ -f /etc/ssh/ssh_host_rsa_key ]   || ssh-keygen -t rsa   -b 3072 -f /etc/ssh/ssh_host_rsa_key   -N ''
```

Configure the client side:

```bash
cat > /etc/ssh/ssh_config.d/10-wolfprov-fips.conf <<'EOF'
Host *
    KexAlgorithms ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group14-sha256,diffie-hellman-group16-sha512
    Ciphers aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
    MACs hmac-sha2-256,hmac-sha2-512,hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
    HostKeyAlgorithms rsa-sha2-512,rsa-sha2-256,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521
    PubkeyAcceptedAlgorithms rsa-sha2-512,rsa-sha2-256,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521
EOF
```

Scope `sshd` to the provider, and validate the configuration before restarting:

```bash
# the routing configuration created in the Foundation guide (§3.2 of that guide):
test -f /etc/wolfprovider/openssl.cnf || { echo "FAIL: complete the Foundation guide first"; exit 1; }
mkdir -p /etc/systemd/system/ssh.service.d
printf '[Service]\nEnvironment="OPENSSL_CONF=/etc/wolfprovider/openssl.cnf"\n' \
  > /etc/systemd/system/ssh.service.d/10-wolfprov.conf
systemctl daemon-reload
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf /usr/sbin/sshd -t && echo "sshd OK"   # must be rc=0
systemctl restart ssh
# open a new ssh session with your ECDSA or RSA key to confirm before closing this one.
```

**Warning:** Lockout risk. An ed25519 administrative key is rejected from this point on; use the ECDSA
or RSA key prepared in the Foundation guide, and confirm a fresh login before closing the current
session. The client `Host *` block restricts outbound SSH to every destination. Always run `sshd -t`
before restarting. Recovery from a lockout is described in the Node Configuration and Recovery guide.

---

## 10. Enable and verify boot-time FIPS enforcement

Set `fips=1` now:

```bash
sed -i 's/fips=0/fips=1/' /etc/default/grub && update-grub
reboot
```

**Note:** Changing `fips=` does not require re-signing the kernel.

After reboot, verify each result before placing production data on the node.

Kernel, module self-test, and RNG:

```bash
cat /proc/sys/crypto/fips_enabled                       # -> 1
grep -o 'fips=[01]' /proc/cmdline                       # -> fips=1; fips_enabled alone cannot confirm this
uname -r                                                # -> 7.0.12-1+wolfcrng1-pve
journalctl -k -b | grep -m1 'FIPS 140-3 wolfCrypt-fips' # -> ... startup self-test succeeded
# Check unsigned-module taint bit 13 (8192):
[ $(( $(cat /proc/sys/kernel/tainted) & 8192 )) -eq 0 ] \
  && echo "no unsigned-module taint" || { echo "FAIL: unsigned-module taint (8192) set"; exit 1; }
# the FIPS DRBG is the system stdrng (the driver name carries 'fips-140-3'):
grep -A6 wolfcrypt /proc/crypto | grep -E 'driver|priority|selftest'
#   driver: sha2-256-drbg-nopr-wolfentropy-wolfcrypt-fips-140-3-with-global-replace  priority:100000  selftest:passed

```

Userspace provider and SSH:

```bash
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf openssl list -providers | grep -i 'wolfSSL Provider FIPS'
echo x | OPENSSL_CONF=/etc/wolfprovider/openssl.cnf openssl dgst -md5 >/dev/null 2>&1 \
  && echo "BAD: MD5 served" || echo "OK: MD5 refused (FIPS)"

# sshd uses the provider (§9):
systemctl show ssh -p Environment | grep -o 'OPENSSL_CONF=[^ ]*'
systemctl is-active ssh                                 # -> active
```

Storage encryption and availability:

```bash
DISK=/dev/disk/by-id/<your-data-disk>
# wolfCrypt cbc(aes) must win by priority, not file order; ESSIV must pass its self-test:
awk '/^name +: cbc\(aes\)$/{b=1} b&&/^driver/{d=$3} b&&/^priority/{print d,$3;b=0}' /proc/crypto
#   -> cbc-aes-wolfcrypt-fips-140-3 100000   (beats cbc-aes-aesni 400)
dmesg | grep -iE 'essiv.*passed'
# Check the header's cipher and KDF:
cryptsetup luksDump "$DISK" | grep -iE 'Version|Cipher:|Cipher key|PBKDF|Hash'
#   -> LUKS2 ; Cipher: aes-cbc-essiv:sha256 ; 256 bits ; PBKDF: pbkdf2 ; Hash: sha256
# Confirm compiled-in routing without OPENSSL_CONF. Stock cryptsetup prints [default], not libwolfprov:
env -u OPENSSL_CONF cryptsetup --debug status cryptdata | grep -i 'Crypto backend' | grep -q libwolfprov \
  && echo "cryptsetup routed" || { echo "FAIL: cryptsetup is not built against wolfProvider"; exit 1; }
# Confirm boot-time unlock and active storage:
systemctl is-active luks-unlock.service                 # -> active
mountpoint /mnt/secure                                  # -> is a mountpoint
# fails closed on an inactive row AND on a row that is missing entirely:
pvesm status | awk '$1=="secure"{f=1; print; if($3!="active") b=1} END{exit (f && !b ? 0 : 1)}'
```

Under enforcement, verify that data remains byte-identical after locking and reopening the volume.

```bash
dd if=/dev/urandom of=/mnt/secure/test.bin bs=1M count=64 status=none && sync
sha256sum /mnt/secure/test.bin                 # note this hash
umount /mnt/secure; cryptsetup close cryptdata # volume LOCKED
cryptsetup open --type luks2 --key-file /etc/luks/cryptdata.key "$DISK" cryptdata
mount /dev/mapper/cryptdata /mnt/secure
sha256sum /mnt/secure/test.bin && rm -f /mnt/secure/test.bin   # MUST equal the first hash
```

Expected state:

| Surface | Expected under `fips=1` |
|---|---|
| Kernel | cmdline `fips=1`; `fips_enabled=1`; `FIPS 140-3 wolfCrypt-fips … startup self-test succeeded`; no unsigned-module taint |
| Kernel RNG | `stdrng` driver name contains `fips-140-3`, priority 100000, selftest passed |
| Data at rest | `cbc-aes-wolfcrypt-fips-140-3` wins at priority 100000; LUKS2 / `aes-cbc-essiv:sha256` / pbkdf2; cryptsetup crypto backend `[libwolfprov]`; `/mnt/secure` round-trips byte-identical across lock and unlock |
| Storage availability | `luks-unlock.service` active after boot; `/mnt/secure` mounted; PVE storage `secure` active |
| Userspace | `wolfSSL Provider FIPS` active; a non-approved digest is refused |
| SSH | `sshd` scoped to the provider; ECDSA and RSA only, ed25519 refused |

---

## 11. LUKS administration

### 11.1 Snapshots

LUKS/dm-crypt has no native snapshots, and §8 registers a directory storage, on which
Proxmox VE allocates guest disks in `raw` format — `qm snapshot` / `pct snapshot` then refuse with
`snapshot feature is not available`. Allocate guest disks in `qcow2` format for Proxmox snapshots on
directory storage (see the [Storage chapter](https://pve.proxmox.com/pve-docs/chapter-pvesm.html)), or
take snapshots at the filesystem or LVM layer.

### 11.2 Keyslot rotation

Add and test the new key before removing the old keyslot. Replace the key file used by the unlock
service and `crypttab` only after the test succeeds.

**Important:** Pass `--pbkdf pbkdf2` on every keyslot operation. Without it LUKS2 falls back to
Argon2id, which wolfProvider does not provide.

```bash
DISK=/dev/disk/by-id/<your-data-disk>
# 1. Generate and add the new key, authorized by the current key:
( umask 077; head -c 512 /dev/urandom > /etc/luks/cryptdata.key.new ); chmod 400 /etc/luks/cryptdata.key.new
cryptsetup luksAddKey --pbkdf pbkdf2 --hash sha256 --key-file /etc/luks/cryptdata.key "$DISK" /etc/luks/cryptdata.key.new
# 2. Test the new key without opening a mapping. Do not remove the old key unless this exits 0:
cryptsetup open --test-passphrase --key-file /etc/luks/cryptdata.key.new "$DISK" && echo "new key opens OK"
# 3. Remove the old keyslot. Alternatively, identify it with luksDump and use:
#    cryptsetup luksKillSlot "$DISK" <old-slot#>
cryptsetup luksRemoveKey --key-file /etc/luks/cryptdata.key "$DISK"
# 4. Replace the key file used by crypttab and luks-unlock.service:
mv /etc/luks/cryptdata.key.new /etc/luks/cryptdata.key
cryptsetup luksDump "$DISK" | grep -iE 'Keyslots|PBKDF|Hash'
```

After rotation, the old key is rejected (rc=2), the new key passes `--test-passphrase` (rc=0), and the
unlock service uses the replacement file.

---

## 12. Troubleshooting

### 12.1 `mokutil --import` reports `Already in kernel trusted keyring. Skip`

The certificate is already trusted; no enrollment is queued. The `mokutil --list-new` pipeline then
matches nothing and exits non-zero. No action is needed.

### 12.2 `error: bad shim signature` at boot

The MOK enrollment did not complete, and Secure Boot refuses the custom kernel pinned as the boot
default. From the GRUB menu, boot a stock Proxmox kernel under **Advanced options**, confirm the
signed kernel was built per Foundation §2.6, and repeat the §5.4 enrollment. Where the pending request
was discarded by an unanswered prompt, `mokutil --list-new` is empty: re-issue `mokutil --import` and
reboot again, with the console attended.

### 12.3 Guests do not start after boot; `pve-guests.service` refuses a manual start

`luks-unlock.service` failed — the volume did not open, or `/mnt/secure` did not mount — and the §7
`Requires=` drop-in then fails `pve-guests.service` closed. Its refusal of a manual start with
`may be requested by dependency only` is by design. Correct the unlock
(`systemctl start luks-unlock.service`), confirm `mountpoint /mnt/secure`, then start the guests
individually with `qm start <vmid>` / `pct start <ctid>`.

### 12.4 A container backup fails with `unable to access …/pct.conf - Permission denied`

The backup staging directory `/mnt/secure/tmp` has lost mode `1777`. The error names a file the job
stages, not the cause: an unprivileged container's backup client runs in a user namespace whose root
maps to uid 100000 and cannot traverse a root-owned mode-700 staging directory. Restore the mode with
`chmod 1777 /mnt/secure/tmp` and re-run the backup.

### 12.5 Outbound `ssh` from `root` still offers chacha20-poly1305

Proxmox installs `/root/.ssh/config` with a `Ciphers` list including `chacha20-poly1305@openssh.com`.
It overrides §9's client cipher list for `root`, including Proxmox operations; the key-exchange, MAC,
and host-key restrictions still apply. Check the effective settings and adjust the per-user file if
required:

```bash
ssh -G <host> | grep -E '^(ciphers|kexalgorithms|macs) '   # as root
```

---

## 13. Next steps

Continue with **Node Configuration and Recovery** to configure management TLS on port 8006, which
still uses system OpenSSL. That guide also covers access recovery and reverting the deployment.

Set the storage variables used by the following guides; repeat in each new shell session:

```bash
STORE=secure ; SECUREPATH=/mnt/secure
echo "STORE=$STORE  SECUREPATH=$SECUREPATH"   # must print these values, not empty
```
