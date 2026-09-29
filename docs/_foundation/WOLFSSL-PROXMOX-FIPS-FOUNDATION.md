# wolfSSL FIPS on Proxmox VE — Shared Build Foundation

Build the shared kernel and userspace artifacts for wolfSSL FIPS on Proxmox VE once, then follow
the ZFS, LUKS, or Ceph deployment guide to distribute them to nodes.

This guide produces:

1. **CRNG FIPS kernel** — a custom Proxmox kernel `.deb` and matching headers, with the wolfCrypt
   CRNG patch, FIPS support, and a retained module-signing key.
2. **wolfSSL and wolfProvider** — `libwolfssl.so` (FIPS wolfCrypt) and `libwolfprov.so` (the OpenSSL
   provider), built as a matched pair and installed system-wide. The deployment guides configure
   services and tools to use the provider.

Run every command in order, as root, on the builder: a fresh PVE 9.2 node. The artifacts are written
to `/root`. Replace each `<placeholder>` with the real value.

**Note:** The kernel build consumes approximately 60 GB under `/root`; allow 80 GB free. It may take
one to four hours.

**Important:** A default installation assigns `/` roughly a quarter of the system disk (the installer's
[`maxroot`](https://pve.proxmox.com/pve-docs/chapter-pve-installation.html#advanced_lvm_options)) and
the rest to the `local-lvm` thin pool. Set a larger root at install time, or remove the thin pool and
extend `/`. The ZFS deployment removes `local-lvm`; the LUKS deployment does not use it.

---

## Version pins

| Component | Pin |
|---|---|
| Proxmox VE | **9.2** (Debian trixie, OpenSSL 3.5) |
| Kernel | **`proxmox-kernel-7.0.12-1+wolfcrng1-pve`** (pve-kernel commit `85c2f14`) |
| CRNG kernel patch | In the licensed FIPS source bundle: `linuxkm/patches/7.0/WOLFSSL_LINUXKM_HAVE_GET_RANDOM_CALLBACKS-7v0.patch` |

---

## Deployment requirements

### Minimum secret length

Use at least **14 characters** for every storage passphrase and backup key password; wolfCrypt FIPS
module versions differ in whether they reject shorter secrets.

### Administrative SSH key

wolfCrypt FIPS v5 does not provide ed25519 or curve25519. The administrative SSH key must be **ECDSA**
(P-256, P-384, or P-521) or **RSA**; an ed25519 key causes a lockout when the deployment guides
restrict `sshd` to FIPS algorithms.

Create the key on the administrative workstation, not the node, and authorize it for the node's
`root` account:

```bash
ssh-keygen -t ecdsa -b 384 -f ~/.ssh/id_pve_fips -C "pve-fips-admin"   # run on your workstation
ssh-copy-id -i ~/.ssh/id_pve_fips.pub root@<node-ip>                   # authorize it on the node
# Without ssh-copy-id, append the contents of ~/.ssh/id_pve_fips.pub to the node's
# /root/.ssh/authorized_keys. Use -i ~/.ssh/id_pve_fips for subsequent SSH connections.
```

### Kernel FIPS mode

After a successful power-on self-test, the in-kernel `libwolfssl.ko` sets
`/proc/sys/crypto/fips_enabled` from 0 to 1 even when the node boots with `fips=0`. The deployment
guides later set `fips=1` on the kernel command line, which additionally enforces FIPS from early
boot and runs the kernel's own FIPS crypto self-tests.

### Signing keys

- **Kernel modules.** The wolfSSL modules must carry a signature the custom kernel trusts.
- **The kernel image.** A Secure Boot node requires a signed image from §2.6.

One key, generated on the builder (§2.4), serves both. Retain the private key securely for future
rebuilds; do not distribute it to deployment nodes.

### UEFI Secure Boot

These guides use the signed Proxmox GRUB and shim path. Configure and verify that path as described in
Proxmox's [Secure Boot documentation](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#sysboot_secure_boot)
before enrolling a Machine Owner Key (MOK).

Check each target node:

```bash
mokutil --sb-state
```

For `SecureBoot enabled`, sign the kernel in §2.6 and complete the deployment guide's MOK step.
For disabled or unsupported Secure Boot, or legacy BIOS, use the unsigned kernel `.deb` from §2.5.

A site using custom firmware `PK`/`KEK`/`db` keys must follow its own policy and Proxmox's
[custom `db` procedure](https://pve.proxmox.com/wiki/Secure_Boot_Setup#Setup_instructions_for_db_key_variant)
instead of the MOK step.

**Note:** A TPM is not required for this Secure Boot path; TPM measured boot is outside the scope of
these guides.

---

## 1. Prerequisites

### 1.1 A standard Proxmox VE 9.2 node

Use a default Proxmox VE 9.2 installation with network access and a root shell. No guests, storage
pools, or cluster configuration are required. For installation, follow
[Installing Proxmox VE](https://pve.proxmox.com/pve-docs/chapter-pve-installation.html)
([ISO download](https://www.proxmox.com/en/downloads)).

### 1.2 Repositories

Configure the Proxmox VE and Debian repositories per
[Package Repositories](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#sysadmin_package_repositories).
Either the enterprise or no-subscription repository supplies the required packages.

Bring the node current before building:

```bash
apt update && apt full-upgrade -y
```

### 1.3 Toolchain and Rust

Install the build tools and use `rustup stable` for Rust; Trixie's packaged `rustc` is too old for
the kernel build:

```bash
apt install -y git build-essential devscripts equivs dh-make flex bison \
  libelf-dev bc patch autoconf automake libtool pkg-config libssl-dev \
  dh-python dh-sequence-sphinxdoc curl unzip patchelf quilt
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y --default-toolchain stable
. "$HOME/.cargo/env"
rustup component add rust-src rustfmt clippy
```

### 1.4 wolfSSL FIPS source bundle

Obtain the licensed wolfSSL FIPS source bundle from wolfSSL, not a public wolfSSL clone. Copy it
to the builder and unpack it to `/root/wolfssl-fips-src`:

```bash
# Rename the unpacked directory to the path used throughout these guides:
cd /root && unzip -q <your-bundle>.zip
mv wolfssl-*-fips-*/ /root/wolfssl-fips-src   # trailing / — matches the directory, not the .zip
ls /root/wolfssl-fips-src/wolfcrypt/src/fips.c /root/wolfssl-fips-src/fips-hash.sh   # both must exist
# Get the kernel patch
cp /root/wolfssl-fips-src/linuxkm/patches/7.0/WOLFSSL_LINUXKM_HAVE_GET_RANDOM_CALLBACKS-7v0.patch /root/
```

---

## 2. The CRNG kernel

This kernel differs from stock in four ways:

- FIPS support and self-tests: `CONFIG_CRYPTO_FIPS=y` and `CONFIG_CRYPTO_SELFTESTS=y`.
- The wolfCrypt CRNG (`get_random_callbacks`) patch.
- A distinct local version, `+wolfcrng1`, to prevent replacement by a stock kernel.
- A retained module-signing key (§2.4), required to load the wolfSSL modules with `fips=1`.

**Note:** Every backend (ZFS, LUKS, Ceph) builds this exact kernel. It is pinned to the 7.0.12 line
because the wolfZFS module requires the OpenZFS 2.4.2 that ships with it.

### 2.1 Fetch kernel sources

```bash
git clone https://git.proxmox.com/git/pve-kernel.git /root/pve-kernel
cd /root/pve-kernel
git checkout 85c2f14                                  # 7.0.12-1 (no tag exists; pin the commit) — all backends
git -c submodule.fetchJobs=4 submodule update --init --recursive   # full — never --depth 1
git submodule status   # expect zfsonlinux 069198f9…, ubuntu-kernel d873103e… (gitlinks pinned by 85c2f14)
```

**Important:** Do not pass `--depth 1` to the submodules: `zfsonlinux` and `ubuntu-kernel` are pinned
to non-tip commits a shallow fetch can miss, producing an incomplete ZFS tree.

### 2.2 Set the local version

```bash
cd /root/pve-kernel
sed -i 's/^KREL_EXTRA=.*/KREL_EXTRA=+wolfcrng1/' Makefile
grep -n '^KREL_EXTRA' Makefile          # -> KREL_EXTRA=+wolfcrng1
```

**Important:** Set `KREL_EXTRA` before `make build-dir-fresh` (§2.3), which writes the version into
`debian/control`, `env.mk`, and `EXTRAVERSION`. Changing it afterward produces a mismatched kernel.

### 2.3 Generate the build tree, apply the CRNG patch, enable FIPS

```bash
cd /root/pve-kernel
. "$HOME/.cargo/env"                     # restore Rust on PATH in a fresh shell
make build-dir-fresh
BD=$(ls -d /root/pve-kernel/proxmox-kernel-*/ | head -1); BD=${BD%/}
KSRC="$BD/ubuntu-kernel"                 # build tree for this guide and the backend build, not the submodule
# Save paths and the Rust environment for later build stages and new sessions.
printf 'BD=%q\nKSRC=%q\n. "$HOME/.cargo/env"\n' "$BD" "$KSRC" > /root/wolf-buildenv.sh
echo "build dir=$BD  KSRC=$KSRC"

# CRNG patch — in the build tree:
cd "$KSRC"
patch -p1 < /root/WOLFSSL_LINUXKM_HAVE_GET_RANDOM_CALLBACKS-7v0.patch
grep -c WOLFSSL_LINUXKM_HAVE_GET_RANDOM_CALLBACKS drivers/char/random.c   # -> 9

# Use config fragments: the packaged build regenerates .config and ignores `make menuconfig` changes.
cd "$BD"
printf '\n-e CONFIG_CRYPTO_FIPS\n-e CONFIG_CRYPTO_SELFTESTS\n' >> debian/rules.d/config-common.opts
tail -2 debian/rules.d/config-common.opts
```

**Note:** While the build tree is generated, `build-dir-fresh` reports `dpkg-buildapi: error: cannot
read debian/control` and `/bin/sh: 1: test: Illegal number:`, twice. The step completes; no action is
required.

### 2.4 Pin a persistent module-signing key

The pve-kernel build deletes its generated signing key. Create one outside the build tree and use
its **absolute path** so it remains available to sign the wolfSSL modules. The build embeds its
certificate in the kernel; modules signed with this key are trusted with or without Secure Boot:

```bash
source /root/wolf-buildenv.sh
cat > /root/kernel-signing.genkey <<'GK'
[ req ]
default_bits = 4096
distinguished_name = req_dn
prompt = no
string_mask = utf8only
x509_extensions = ext
[ req_dn ]
CN = Proxmox wolfCrypt-FIPS module signing key
[ ext ]
basicConstraints=critical,CA:FALSE
keyUsage=digitalSignature
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid
GK
openssl req -new -nodes -utf8 -sha256 -days 3650 -batch -x509 \
    -config /root/kernel-signing.genkey -outform PEM -out /root/ksk-cert.pem -keyout /root/ksk-priv.pem
cat /root/ksk-priv.pem /root/ksk-cert.pem > /root/kernel-signing-key.pem   # PEM: private key + cert
openssl x509 -in /root/ksk-cert.pem -outform DER -out /root/kernel-signing-key.x509
chmod 600 /root/kernel-signing-key.pem; rm -f /root/ksk-priv.pem /root/ksk-cert.pem
# Set the absolute key path in the build's config fragment:
echo '--set-str CONFIG_MODULE_SIG_KEY /root/kernel-signing-key.pem' >> "$BD/debian/rules.d/config-common.opts"
# Seed certs/signing_key.pem: the build needs it for module signing and to derive signing_key.x509,
# but does not generate it when CONFIG_MODULE_SIG_KEY points to an external key.
cp /root/kernel-signing-key.pem "$KSRC/certs/signing_key.pem"
# Allow post-sign cleanup when the copy is absent:
sed -i 's#rm \./ubuntu-kernel/certs/signing_key\.pem#rm -f ./ubuntu-kernel/certs/signing_key.pem#' "$BD/debian/rules"
echo "signing key staged: certs/signing_key.pem = your key, protected from regeneration"
```

### 2.5 Build the kernel `.deb`

```bash
source /root/wolf-buildenv.sh            # restore paths and Rust environment
cd /root/pve-kernel
mk-build-deps -i -r -t "apt-get -y" "$BD/debian/control"
cd "$BD"
dpkg-buildpackage -b -uc -us
# Packages: /root/pve-kernel/. The retained key stays at /root/kernel-signing-key.pem.
```

**Important:** This build stops at `make[1]: *** [debian/rules:266: fwcheck] Error 1`. Remove the stale
check file and resume without cleaning:

```bash
source /root/wolf-buildenv.sh
cd "$BD"
rm "$BD/fwlist-previous-amd64"
dpkg-buildpackage -b -uc -us -nc
```

If instead a config or a patch was changed, re-run `make build-dir-fresh` (§2.3) and re-apply
§2.3–§2.4 before rebuilding: `debian/rules clean` is a no-op here, and a bare `dpkg-buildpackage`
re-run skips compilation and silently repackages the stale build.

Verify the signing key, FIPS configuration, package version, and module contents:

```bash
grep '^CONFIG_MODULE_SIG_KEY=' "$KSRC/.config"      # -> "/root/kernel-signing-key.pem", not certs/…
grep '^CONFIG_CRYPTO_FIPS='   "$KSRC/.config"       # -> =y
openssl x509 -in /root/kernel-signing-key.x509 -noout -ext subjectKeyIdentifier
#   ^ the key the installed kernel trusts in .builtin_trusted_keys
ls /root/pve-kernel/*.deb | grep wolfcrng1
dpkg-deb -f /root/pve-kernel/proxmox-kernel-*+wolfcrng1-pve_*.deb Package
dpkg-deb -c /root/pve-kernel/proxmox-kernel-*+wolfcrng1-pve_*.deb | grep -c lib/modules/   # -> big number; no error = clean
```

### 2.6 Sign the kernel image for Secure Boot

For Secure Boot deployments, sign the image inside the `.deb` on the builder. Do not install the
node-side signing hook from Proxmox's
[custom-kernel procedure](https://pve.proxmox.com/wiki/Secure_Boot_Setup#Signing_a_custom_kernel).

```bash
apt-get install -y sbsigntool
source /root/wolf-buildenv.sh
V=7.0.12-1+wolfcrng1-pve
# Convert the §2.4 DER certificate to PEM for sbsign:
openssl x509 -inform DER -in /root/kernel-signing-key.x509 -out /root/kernel-signing-key.crt
# unpack the kernel .deb, sign the image in place, and repack:
rm -rf /root/signdeb && mkdir -p /root/signed-debs
dpkg-deb -R /root/pve-kernel/proxmox-kernel-${V}_*.deb /root/signdeb
sbsign --key /root/kernel-signing-key.pem --cert /root/kernel-signing-key.crt \
       --output /root/signdeb/boot/vmlinuz-$V.signed /root/signdeb/boot/vmlinuz-$V
mv /root/signdeb/boot/vmlinuz-$V.signed /root/signdeb/boot/vmlinuz-$V
# Refresh the signed kernel's checksum; otherwise debsums reports it as modified on every node.
cd /root/signdeb
grep -v " boot/vmlinuz-$V\$" DEBIAN/md5sums > /tmp/md5sums.new
md5sum boot/vmlinuz-$V >> /tmp/md5sums.new
mv /tmp/md5sums.new DEBIAN/md5sums
cd /
dpkg-deb -b /root/signdeb /root/signed-debs/
```

Confirm the image is signed by the retained key:

```bash
sbverify --cert /root/kernel-signing-key.crt /root/signdeb/boot/vmlinuz-$V
#   -> Signature verification OK
# --cert rejects an unsigned image or a different certificate.
# --list can exit 0 for an unsigned image; check the output, not just the exit status:
sbverify --list /root/signdeb/boot/vmlinuz-$V
#   -> signature 1
#   -> image signature issuers:
#   ->  - /CN=<the subject of the §2.4 certificate>
ls -l /root/signed-debs/                               # the signed kernel .deb
```

Distribute the signed `.deb`, the unchanged headers package, and the public certificate
`/root/kernel-signing-key.x509`. Keep `/root/kernel-signing-key.pem` on the builder.

For MOK replacement or revocation, follow Proxmox's
[Secure Boot documentation](https://pve.proxmox.com/wiki/Secure_Boot_Setup).

---

## 3. Build wolfSSL and wolfProvider

Build the FIPS `libwolfssl.so` first, then wolfProvider against it and the **system** OpenSSL to
match the ABI used by Proxmox services:

```bash
# Clone before installing wolfSSL under /opt/wolfProvider/wolfssl-install to avoid a directory conflict:
cd /opt
git clone https://github.com/wolfSSL/wolfProvider.git
cd wolfProvider && git checkout v1.2.1

# Use your licensed bundle's FIPS configuration (e.g. ready or v5.2.4) for <your-bundle-config>.
# wolfProvider requires --enable-opensslcoexist and both -D flags.
# -DWOLFSSL_OLD_OID_SUM sets the OID scheme checked in §3.1.
cd /root/wolfssl-fips-src
./configure --prefix=/opt/wolfProvider/wolfssl-install --enable-fips=<your-bundle-config> \
    --enable-opensslcoexist CFLAGS="-DWOLFSSL_OLD_OID_SUM -DWOLFSSL_DH_EXTRA"
make -j"$(nproc)"
./fips-hash.sh                  # set the required in-core FIPS integrity hash
make -j"$(nproc)" && make install

# Build wolfProvider against system OpenSSL (/usr) and the FIPS wolfSSL:
cd /opt/wolfProvider
./autogen.sh
./configure --with-openssl=/usr --with-wolfssl=/opt/wolfProvider/wolfssl-install \
    CFLAGS="-DWOLFSSL_OLD_OID_SUM -DWOLFSSL_DH_EXTRA"
make -j"$(nproc)"
```

Install both libraries in system paths so services can load them without shell environment variables:

```bash
# Install the provider at the path used by the deployment guides:
install -Dm755 /opt/wolfProvider/.libs/libwolfprov.so \
               /usr/lib/x86_64-linux-gnu/ossl-modules/libwolfprov.so
# First install: cp -a is safe. For later replacement on a running system, use install/mv instead.
cp -a /opt/wolfProvider/wolfssl-install/lib/libwolfssl.so* /usr/local/lib/
ldconfig
# Remove the build-tree RUNPATH so libwolfssl resolves from /usr/local/lib even if /opt is deleted:
patchelf --remove-rpath /usr/lib/x86_64-linux-gnu/ossl-modules/libwolfprov.so
```

Verify the module resolves under `/usr/local/lib` with no shell environment variables:

```bash
env -i LD_LIBRARY_PATH= ldd /usr/lib/x86_64-linux-gnu/ossl-modules/libwolfprov.so | grep -F /usr/local/lib
```

### 3.1 Verify wolfProvider

The power-on self-test (POST) detects a mismatched FIPS bundle configuration.

```bash
cat > /tmp/wp.cnf <<'EOF'
openssl_conf = openssl_init
[openssl_init]
providers = provider_sect
[provider_sect]
wolfprov = wolfprov_sect
[wolfprov_sect]
module = /usr/lib/x86_64-linux-gnu/ossl-modules/libwolfprov.so
activate = 1
EOF
export OPENSSL_CONF=/tmp/wp.cnf
openssl list -providers | grep -A2 -i wolf      # -> "wolfSSL Provider FIPS ... status: active"  (POST passed)
echo hello | openssl dgst -sha256               # digest works
openssl genrsa -out /tmp/wp.key 2048 && openssl pkey -in /tmp/wp.key -noout && echo "RSA OK (OID scheme)"
# SHA-1 signing must fail:
openssl dgst -sha1 -sign /tmp/wp.key /tmp/wp.cnf >/dev/null 2>&1 && echo "FAIL: SHA-1 allowed" || echo "SHA-1 blocked (FIPS OK)"
unset OPENSSL_CONF; rm -f /tmp/wp.key /tmp/wp.cnf
```

### 3.2 Configure wolfProvider

Installation alone does not route a process through wolfProvider. Set its `OPENSSL_CONF` to the
file below. The deployment guides apply this setting through systemd drop-ins or on the command line.

```bash
mkdir -p /etc/wolfprovider
cat > /etc/wolfprovider/openssl.cnf <<'EOF'
openssl_conf = openssl_init
[openssl_init]
providers = provider_sect
alg_section = algorithm_sect
ssl_conf = ssl_sect
[provider_sect]
wolfprov = wolfprov_sect
[wolfprov_sect]
module = /usr/lib/x86_64-linux-gnu/ossl-modules/libwolfprov.so
activate = 1
[algorithm_sect]
default_properties = provider=wolfprov
[ssl_sect]
system_default = system_default_sect
[system_default_sect]
# Pin NIST curves: X25519 is not an approved KAS and FIPS clients fail its keyshare.
Groups = P-256:P-384:P-521
# Pin AES-GCM server-side: the provider advertises non-approved ChaCha20-Poly1305
# and will serve it if offered.
CipherSuites = TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256
EOF
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf openssl list -providers | grep -qi 'wolfSSL Provider FIPS' \
  && echo "wolfProvider routing configuration OK"
```

**Important:** Keep `default_properties = provider=wolfprov` mandatory: do not add `?`. This prevents
fallback to the default provider, including explicit in-code fetches.

---

## Continue in a deployment guide

The builder now holds the shared artifacts:

```
libwolfssl.so* + libwolfprov.so                    (the userspace FIPS matched pair — §3)
/etc/wolfprovider/openssl.cnf                      (the wolfProvider routing configuration — §3.2)
proxmox-kernel-7.0.12-1+wolfcrng1-pve_*.deb        (the CRNG FIPS kernel — §2)
proxmox-headers-7.0.12-1+wolfcrng1-pve_*.deb       (matching headers — §2)
/root/kernel-signing-key.pem  (+ .x509)            (the retained signing key and public certificate)
```

Keep the private key on the builder and distribute only the files the deployment guide identifies. For
Secure Boot, the signed kernel from §2.6 replaces the kernel `.deb` above:

```
/root/signed-debs/proxmox-kernel-7.0.12-1+wolfcrng1-pve_*.deb   (the signed CRNG FIPS kernel — §2.6)
/root/kernel-signing-key.x509                                   (the certificate enrolled on each node)
```

Retain the kernel **build tree** (`$BD` / `$KSRC`) and the unpacked FIPS source at
`/root/wolfssl-fips-src`; the backend build compiles its in-kernel `libwolfssl.ko` from that source
against that tree.

**Continue in the deployment guide for your storage architecture** — Ceph (distributed / shared
storage), or single-node **ZFS** or **LUKS** — then the **Node Configuration and Recovery** and
**Backup** guides.
