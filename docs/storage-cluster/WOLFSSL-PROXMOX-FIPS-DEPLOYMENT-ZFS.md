# wolfSSL FIPS on Proxmox VE — ZFS Native Encryption Deployment

Deploy ZFS native encryption using the in-kernel wolfCrypt FIPS module after completing the
**Shared Build Foundation** guide. This guide configures the encrypted pool, SSH, and boot-time FIPS
enforcement, then verifies the deployment.

Use this guide for single-node local storage. For distributed or shared storage, use the Ceph
deployment guide; for shared-nothing clusters, use the Cluster guide.

---

## 1. Cryptographic components

| Layer | Where it runs | wolfCrypt-FIPS component |
|---|---|---|
| Bulk data encryption (AES-256-GCM) | kernel, ZFS crypto layer | in-kernel `libwolfssl.ko`, linked into `zfs.ko` |
| Dataset key unwrap (HKDF) and IV generation | kernel, ZFS crypto layer | in-kernel `libwolfssl.ko` |
| Dataset key entropy | kernel RNG | wolfCrypt FIPS DRBG, registered as the system `stdrng` |
| Passphrase-to-wrapping-key derivation (PBKDF2), passphrase model only | userspace, ZFS tools | wolfProvider, when the invoking process is routed to it (§8.1) |
| SSH, inbound (`sshd`) | userspace OpenSSL | wolfProvider, with the algorithm sets restricted (§11) |
| SSH, outbound (`ssh`) | userspace OpenSSL | System OpenSSL, not wolfProvider. Algorithm restrictions in §11; `root` override in §15.3. |

**Note:** Management TLS — the web interface and the API on `pveproxy` — is covered in the **Node
Configuration and Recovery** guide, which follows this one.

---

## 2. Requirements

### 2.1 The node

Use a fresh **Proxmox VE 9.2** installation with an **ext4/LVM root**, not a ZFS root, and a **second
disk** for the encrypted pool. Keeping the custom modules and data pool out of the boot path lets the
node boot if a module fails. Confirm SSH access with an ECDSA or RSA administrative key (Foundation
requirements).

**Note:** For Secure Boot, complete Foundation §2.6 before §5, then use §5.2. Otherwise use §5.1.

### 2.2 The shared foundation

Complete the **Shared Build Foundation** guide in full before starting here.

### 2.3 Version pins

| Component | Version | Source pin |
|---|---|---|
| Proxmox VE | **9.2.x** (Debian 13 "Trixie"), **ext4/LVM root** | — |
| Kernel | **proxmox-kernel-7.0.12-1+wolfcrng1-pve** | built in the Foundation guide from pve-kernel commit `85c2f14` |
| OpenZFS | **2.4.2** (`zfs-2.4.2-pve1`) | ships as the `submodules/zfsonlinux` gitlink of kernel commit `85c2f14` |
| wolfZFS patch | **osp master** | `zfs/patches/proxmox/zfs-2.4.2-pve1_wolfzfs.patch`; optionally pin merge commit `35a7dea` |

**Important:** Use the Foundation kernel because the wolfZFS patch supports only OpenZFS 2.4.2 on the
7.0.12 kernel line.

### 2.4 Conventions

Run every command as **root**. Replace `<placeholders>` with your values; enter version pins verbatim.

---

## 3. Build the in-kernel FIPS `libwolfssl.ko`

Build against the Foundation kernel **build tree** using `/root/wolfssl-fips-src`. Register only
`stdrng-default`: `zfs.ko` calls wolfCrypt directly for encryption, not through the kernel crypto API.

```bash
source /root/wolf-buildenv.sh            # restore $KSRC
cd /root/wolfssl-fips-src
make clean >/dev/null 2>&1 || true
./configure --enable-fips=<your-bundle-config> --enable-linuxkm \
    --enable-linuxkm-lkcapi-register=stdrng-default --with-linux-source="$KSRC"
# objtool must be disabled for the FIPS boundary assembly; the target bakes the in-core integrity hash:
make -j"$(nproc)" KBUILD_EXTRA_FLAGS=FORCE_GLOBAL_OBJTOOL_OFF=1 module-with-matching-fips-hash || true
ls -l linuxkm/libwolfssl.ko                          # must exist; §6 signs it regardless of auto-sign outcome
modinfo linuxkm/libwolfssl.ko | grep vermagic        # must read 7.0.12-1+wolfcrng1-pve, not the running stock kernel
```

**Important:** Do not strip `libwolfssl.ko`. Stripping changes its in-core integrity hash and breaks
the power-on self-test.

---

## 4. Build `zfs.ko` and `spl.ko` against the FIPS wolfSSL

Compile both modules against the kernel **build tree**, not the running kernel.

```bash
git clone https://github.com/wolfSSL/osp.git /root/osp

source /root/wolf-buildenv.sh                                  # restore $BD/$KSRC
# Copy the Proxmox-prepared pkg-zfs tree matching zfs-2.4.2-pve1:
cp -a "$BD/modules/pkg-zfs" /root/wolfzfs-build
cd /root/wolfzfs-build
rm -rf .git; make distclean 2>/dev/null || true
# Apply the Proxmox patch for zfs-2.4.2-pve1:
patch -p1 < /root/osp/zfs/patches/proxmox/zfs-2.4.2-pve1_wolfzfs.patch   # routes ZFS ICP crypto -> wolfCrypt

./autogen.sh                                                   # MANDATORY after the wolfZFS patch
./configure --with-config=kernel --with-linux="$KSRC" --with-linux-obj="$KSRC" \
            --with-wolfssl-source=/root/wolfssl-fips-src       # the FIPS source, not non-FIPS
make -j"$(nproc)"
ls -l module/spl.ko module/zfs.ko                              # both must exist before continuing to §5
```

---

## 5. Install and boot the FIPS kernel

Boot with `fips=0` while configuring storage and SSH; §12 enables `fips=1`.

Determine whether the node boots with UEFI Secure Boot, then run **either §5.1 or §5.2 — not both**:

```bash
mokutil --sb-state
#   SecureBoot enabled -> §5.2
#   otherwise          -> §5.1
```

### 5.1 Install the kernel — Secure Boot disabled

The kernel package is the one built in the Foundation guide (§2.5), installed as built.

```bash
# the build-deps metapackage conflicts with the headers package; removing it leaves the toolchain in place:
apt-get remove -y proxmox-kernel-7.0-build-deps 2>/dev/null || true
dpkg -i /root/pve-kernel/proxmox-kernel-*+wolfcrng1-pve_*.deb \
        /root/pve-kernel/proxmox-headers-*+wolfcrng1-pve_*.deb
```

Continue at §5.3.

### 5.2 Install the kernel and enroll the certificate — Secure Boot enabled

The kernel package is the **signed** one produced in the Foundation guide (§2.6), from
`/root/signed-debs/`. The headers package is the one built alongside it and is installed unchanged.

```bash
# the build-deps metapackage conflicts with the headers package; removing it leaves the toolchain in place:
apt-get remove -y proxmox-kernel-7.0-build-deps 2>/dev/null || true
dpkg -i /root/signed-debs/proxmox-kernel-*+wolfcrng1-pve_*.deb \
        /root/pve-kernel/proxmox-headers-*+wolfcrng1-pve_*.deb
```

Queue the Foundation certificate for MOK enrollment before rebooting:

```bash
# `Already in kernel trusted keyring. Skip` -> nothing further is needed; see Troubleshooting (§15.4):
mokutil --timeout -1                                   # do not time the enrollment prompt out
mokutil --import /root/kernel-signing-key.x509         # choose a one-time password
mokutil --list-new | grep -i -A2 wolfCrypt
```

Open the node console for the next reboot and complete Proxmox's
[MOK enrollment procedure](https://pve.proxmox.com/wiki/Secure_Boot_Setup#Enrolling_the_MOK). A node
that stops at `error: bad shim signature` is recovered in §15.1.

### 5.3 Pin the FIPS kernel as the boot default

Set the kernel command line and make the FIPS kernel the boot default (**GRUB / ext4-LVM root**):

```bash
# add fips=0 to the kernel cmdline (idempotent; §12 sets this to 1):
grep -q 'fips=' /etc/default/grub || \
  sed -i 's/^\(GRUB_CMDLINE_LINUX_DEFAULT="[^"]*\)"/\1 fips=0"/' /etc/default/grub
grep GRUB_CMDLINE_LINUX_DEFAULT /etc/default/grub      # confirm it now contains fips=0
update-grub
# pin the FIPS kernel as the boot default (the tool regenerates the GRUB configuration itself):
proxmox-boot-tool kernel pin 7.0.12-1+wolfcrng1-pve
proxmox-boot-tool kernel list                          # -> "Pinned kernel:" names the FIPS kernel
```

### 5.4 Reboot and confirm

On a Secure Boot node, keep the console open and complete §5.2's enrollment during this reboot.

```bash
reboot
# after reboot:
uname -r                                               # -> 7.0.12-1+wolfcrng1-pve
cat /proc/sys/crypto/fips_enabled                      # -> 0 until the wolfCrypt module loads in §6
grep -E 'CONFIG_CRYPTO_FIPS|CONFIG_CRYPTO_SELFTESTS' /boot/config-$(uname -r)   # both =y
mokutil --sb-state                                     # Secure Boot node -> SecureBoot enabled
```

**Important:** Keep the FIPS kernel [pinned](https://pve.proxmox.com/wiki/Host_Bootloader). Without the
pin, GRUB selects a stock kernel. Unlike a standard Proxmox deployment, this pin must remain; it also
prevents automatic selection of newer kernels with security fixes.

---

## 6. Sign and install the modules

Sign the three out-of-tree modules with the retained key used for the kernel's packaged modules:

```bash
VER=$(uname -r)          # now the custom kernel — you rebooted in §5
# Confirm all three modules exist before signing:
for f in /root/wolfssl-fips-src/linuxkm/libwolfssl.ko \
         /root/wolfzfs-build/module/spl.ko /root/wolfzfs-build/module/zfs.ko; do
  [ -f "$f" ] || { echo "MISSING: $f — complete §3 and §4 before this section"; exit 1; }
done
# Use sign-file from the installed headers; $KSRC does not survive reboot:
SIGN=/lib/modules/$VER/build/scripts/sign-file; KEY=/root/kernel-signing-key.pem; CERT=/root/kernel-signing-key.x509
mkdir -p /lib/modules/$VER/updates
# libwolfssl.ko UNSTRIPPED (FIPS in-core hash); zfs/spl may be stripped to keep the initramfs lean:
cp /root/wolfssl-fips-src/linuxkm/libwolfssl.ko /lib/modules/$VER/updates/
cp /root/wolfzfs-build/module/spl.ko /root/wolfzfs-build/module/zfs.ko /lib/modules/$VER/updates/
strip --strip-debug /lib/modules/$VER/updates/spl.ko /lib/modules/$VER/updates/zfs.ko
for m in libwolfssl spl zfs; do "$SIGN" sha512 "$KEY" "$CERT" /lib/modules/$VER/updates/$m.ko; done
depmod "$VER"
```

Load the module at boot — otherwise only `spl` loads and the pool does not import:

```bash
cat > /etc/modules-load.d/wolfzfs.conf <<'EOF'
# load the wolf zfs.ko at boot (pulls in spl and libwolfssl)
zfs
EOF
```

**Important:** Hold `zfsutils-linux` so an upgrade cannot move userspace beyond the 2.4.x release
series tolerated by the pinned OpenZFS 2.4.2 modules:

```bash
apt-mark hold zfsutils-linux
```

---

## 7. Verify the loaded modules

Proxmox loads stock ZFS at boot. Installing replacement files and running `modprobe zfs` does not
replace an already loaded module. Unload stock ZFS, then load wolfZFS before creating the pool:

```bash
VER=$(uname -r)
systemctl stop zfs-zed zfs.target 2>/dev/null
modprobe -r zfs spl                             # unload stock
modprobe zfs                                    # loads updates/zfs.ko (depmod priority) -> pulls in libwolfssl + spl
```

Confirm the result:

```bash
# 1. wolfCrypt sets fips_enabled on load even at fips=0; §12 enables boot-time enforcement:
[ "$(cat /proc/sys/crypto/fips_enabled)" = 1 ] || { echo "FAIL: fips_enabled != 1"; exit 1; }
# 2. wolfCrypt module passed its power-on self-test:
dmesg | grep -qiE 'FIPS 140-3 wolfCrypt.*(startup )?self-test succeeded' || { echo "FAIL: no wolfCrypt POST line"; exit 1; }
# 3. No unsigned-module taint:
[ $(( $(cat /proc/sys/kernel/tainted) & 8192 )) -eq 0 ] || { echo "FAIL: unsigned-module taint (8192) set"; exit 1; }
# 4. The three modules are loaded:
for m in zfs spl libwolfssl; do lsmod | grep -q "^$m " || { echo "FAIL: module '$m' not loaded"; exit 1; }; done
# 5. ZFS is using wolfCrypt:
dmesg | grep -qi 'zfs using libwolfssl' || { echo "FAIL: zfs is NOT using libwolfssl (stock ZFS crypto?)"; exit 1; }
echo "wolfZFS foundation OK"
lsmod | grep -E '^(zfs|spl|libwolfssl)'
dmesg | grep -iE 'zfs using libwolfssl|wolfCrypt: .* algorithm|registered as systemwide default stdrng'
modinfo zfs | grep -E 'signer|sig_hashalgo'     # signer = your kernel signing key; sig_hashalgo sha512
```

**Important:** Stop if any check fails. If `libwolfssl` or the `zfs using libwolfssl` banner is absent,
repeat the unload/load commands and checks before writing data.

---

## 8. Create the encrypted pool and dataset

### 8.1 Route the ZFS userspace tools through wolfProvider

ZFS tools derive passphrase wrapping keys in userspace. Set `OPENSSL_CONF` on each invocation to use
the provider configuration from Foundation §3.2:

```bash
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs <subcommand> …
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zpool <subcommand> …
```

Use this prefix for subsequent `zfs` and `zpool` operations as well.

Confirm the provider is active before creating the dataset:

```bash
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf openssl list -providers | grep -qi 'wolfSSL Provider FIPS' \
  && echo "ZFS key operations will route to wolfCrypt"
```

**Important:** Omitting the prefix silently derives the wrapping key with stock OpenSSL. A later
routed unlock does not correct an unrouted `zfs create`; use a routed `zfs change-key` to re-wrap the
master key without rewriting data (§14.2).

### 8.2 Create the pool and the encrypted dataset

**Important:** Leave the unencrypted pool root unmounted. Store data only on the encrypted child.

```bash
# Identify the blank data disk, not the root disk. Use by-id for a disk or by-partuuid for a partition:
ls -l /dev/disk/by-id/ /dev/disk/by-partuuid/ 2>/dev/null ; lsblk
# Destructive — wipes the target. mountpoint=none means the pool root never mounts (no plaintext target):
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zpool create -f -o ashift=12 -O mountpoint=none datatank /dev/disk/by-id/<your-data-disk>
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zpool set feature@encryption=enabled datatank
```

Choose **one** unlock model. Passphrases require an operator at each boot and leave no unlock key on
disk. A keyfile permits unattended unlock but stores the key on the node.

**(a) Passphrase — attended unlock:**

```bash
# `encryption=on` == aes-256-gcm; explicit child mountpoint keeps the parent unmounted.
# Passphrase minimum is 14 characters. Always target the encrypted child, never the pool root.
# The OPENSSL_CONF prefix routes the passphrase derivation through wolfProvider (§8.1) — do not omit it:
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs create -o encryption=on -o keyformat=passphrase -o mountpoint=/datatank/secure datatank/secure
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs get encryption,keystatus,keylocation,mounted datatank/secure
#   -> encryption = aes-256-gcm ; keystatus = available ; keylocation = prompt ; mounted = yes
```

**(b) Raw keyfile — unattended unlock:**

```bash
mkdir -p /etc/zfs/keys && chmod 700 /etc/zfs/keys
dd if=/dev/urandom of=/etc/zfs/keys/datatank-secure.key bs=32 count=1
chmod 600 /etc/zfs/keys/datatank-secure.key
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs create -o encryption=aes-256-gcm -o keyformat=raw \
    -o keylocation=file:///etc/zfs/keys/datatank-secure.key -o mountpoint=/datatank/secure datatank/secure
```

**Caution:** A keyfile stored on the unencrypted root lets the node unlock itself, but a stolen disk
together with that key yields plaintext. Where at-rest protection must hold against physical theft with
unattended unlock, keep the key off this machine's root, on removable media or a network key server.

**Caution:** Choose the custody model before placing data on the dataset. Switching custody models
re-creates the dataset and therefore destroys its contents.

---

## 9. Register the encrypted dataset as PVE storage

```bash
# always the encrypted child, never bare `datatank`:
pvesm add zfspool zdata --pool datatank/secure
# Remove the default plaintext guest storage so a disk can never land unencrypted.
# Migrate or audit any existing guests off local-lvm first — this removes the storage entry:
pvesm remove local-lvm 2>/dev/null || true
pvesm set local --content vztmpl,iso          # keep 'local' for templates/ISOs only, no guest disks
pvesm status | grep -E 'zdata|local'
```

Only `zdata` should accept guest disks (`pvesm status --content images`); the pool root remains unmounted.

Create a VM and CT on `zdata` to verify encrypted guest storage. IDs 700 and 710 are examples; use free
IDs and retain the guests for §13.
Container templates are managed with `pveam`, per
[Container Images](https://pve.proxmox.com/pve-docs/chapter-pct.html#pct_container_images):

```bash
qm create 700 --name fipstest --memory 512 --scsihw virtio-scsi-single \
    --scsi0 zdata:1 --net0 virtio,bridge=vmbr0
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs get -H -o name,property,value encryption,encryptionroot datatank/secure/vm-700-disk-0
#   -> encryption = aes-256-gcm ; encryptionroot = datatank/secure (the guest disk is encrypted)
qm start 700 && qm status 700               # running — off the encrypted zvol
qm stop 700
# Containers: the rootfs lands as an encrypted child dataset:
pveam update && pveam available --section system | grep debian   # current template name (revs over time)
pveam download local debian-13-standard_13.1-2_amd64.tar.zst     # substitute the name you just saw
pct create 710 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
    --hostname fipsct --memory 512 --rootfs zdata:2 --net0 name=eth0,bridge=vmbr0,ip=dhcp \
    --password <a-container-root-password> --unprivileged 1 --features nesting=1
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs get -H -o name,property,value encryption datatank/secure/subvol-710-disk-0   # -> aes-256-gcm
pct start 710 && pct status 710             # running
pct stop 710
```

---

## 10. Boot-time unlock service

Proxmox recommends unlocking encrypted datasets manually after boot, or with a custom unit that
passes the key material to `zfs load-key`; this section creates that unit
([Encrypted ZFS Datasets](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#zfs_encryption)).

**Caution:** While locked, the mountpoint is a directory on the unencrypted root. Writes there remain
in plaintext, hidden when the dataset mounts; unmount it to remove them. The service below blocks
guest startup until the key is loaded and the dataset mounts.

A `oneshot` service cannot answer a passphrase prompt:

- **Passphrase:** the service fails at boot and guests stay stopped until manual unlock (§10.2).
- **Keyfile:** uncomment `ExecStartPre` for unattended unlock. A missing keyfile causes
  `Dependency failed for pve-guests`; guests stay stopped until the key is restored.

### 10.1 Create and enable the unit

Create the unit file `/etc/systemd/system/wolfzfs-unlock.service` with the contents below:

```bash
cat > /etc/systemd/system/wolfzfs-unlock.service <<'EOF'
[Unit]
Description=Mount the wolfZFS encrypted data pool (fail-closed)
DefaultDependencies=no
After=zfs-import.target
Before=pve-guests.service zfs-mount.service
Requires=zfs-import.target
[Service]
Type=oneshot
RemainAfterExit=yes
# Required: systemd does not inherit the operator's OPENSSL_CONF (§8.1).
Environment="OPENSSL_CONF=/etc/wolfprovider/openssl.cnf"
# KEYFILE model only — uncomment this line to unlock unattended:
#ExecStartPre=/bin/sh -c '[ "$(/sbin/zfs get -H -o value keystatus datatank/secure)" = available ] || /sbin/zfs load-key datatank/secure'
# Mount, tolerating a dataset that PVE storage activation has already mounted (§9):
ExecStart=/bin/sh -c '/sbin/zfs mount datatank/secure || /usr/bin/mountpoint -q /datatank/secure'
[Install]
WantedBy=multi-user.target
EOF
```

Enable the service and add a `Requires=` dependency to block guest startup if unlock fails.
`Before=` alone only sets ordering:

```bash
mkdir -p /etc/systemd/system/pve-guests.service.d
printf '[Unit]\nRequires=wolfzfs-unlock.service\n' > /etc/systemd/system/pve-guests.service.d/10-require-wolfzfs.conf
systemctl daemon-reload && systemctl enable wolfzfs-unlock.service
# Verify Requires directly, not with list-dependencies --reverse:
systemctl show pve-guests.service -p Requires --value | tr ' ' '\n' | grep wolfzfs-unlock || \
  { echo "FAIL: dependency edge missing"; exit 1; }
```

### 10.2 Unlock after boot — passphrase model

```bash
# Skip load-key if the key is already loaded:
[ "$(zfs get -H -o value keystatus datatank/secure)" = available ] || \
  OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs load-key datatank/secure   # enter the passphrase
systemctl start wolfzfs-unlock
pvesh create /nodes/$(hostname)/startall
```

**Important:** `startall` can return a task ID and exit 0 without starting guests: it covers only
`onboot=1`. Start others with `qm start <vmid>` / `pct start <ctid>`, or set the flag.

---

## 11. SSH

Restrict both SSH endpoints to the algorithms below and serve only RSA or ECDSA host keys. OpenSSH
implements curve25519, ed25519, and chacha20-poly1305 internally; they do not use the provider and are
not supported by wolfCrypt FIPS v5.

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

**Note:** A per-user `~/.ssh/config` outranks this file, and a Proxmox VE installation writes one for
`root`; see §15.3.

Scope `sshd` to the provider, validate the configuration, and only then restart the service:

```bash
mkdir -p /etc/systemd/system/ssh.service.d
printf '[Service]\nEnvironment="OPENSSL_CONF=/etc/wolfprovider/openssl.cnf"\n' \
  > /etc/systemd/system/ssh.service.d/10-wolfprov.conf
systemctl daemon-reload
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf /usr/sbin/sshd -t && echo "sshd OK"   # must be rc=0
systemctl restart ssh
# open a second ssh session with your ECDSA or RSA key and confirm it succeeds before closing this one.
```

**Warning:** This rejects ed25519 administrative keys. Use the Foundation ECDSA or RSA key, run
`sshd -t` before restarting, and confirm a new login before closing the current session. The client
`Host *` block applies to all destinations. For lockout recovery, see **Node Configuration and Recovery**.

**Note:** An ed25519-only host key breaks negotiation for `pvecm add`, `qm migrate`, and PVE HA.

---

## 12. Enable FIPS enforcement at boot

Set `fips=1` now:

```bash
sed -i 's/fips=0/fips=1/' /etc/default/grub && update-grub
reboot
```

**Note:** Changing `fips=` does not require re-signing the kernel.

**Note:** With passphrase unlock, `wolfzfs-unlock` fails at boot and guests stay stopped until the
dataset is unlocked. §13.2 begins with the unlock procedure.

---

## 13. Verify the deployment

After the §12 reboot, verify every result before placing production data on the node. Some checks
only print failures; they do not stop execution.

### 13.1 Kernel, modules, and userspace

Kernel, modules, and RNG:

```bash
cat /proc/sys/crypto/fips_enabled                       # -> 1
grep -o 'fips=[01]' /proc/cmdline                       # -> fips=1; fips_enabled alone cannot confirm this
uname -r                                                # -> 7.0.12-1+wolfcrng1-pve
journalctl -k -b | grep -m1 'FIPS 140-3 wolfCrypt-fips' # -> ...startup self-test succeeded
# no unsigned-module taint (bit 13 = 8192):
test $(( $(cat /proc/sys/kernel/tainted) & 8192 )) -eq 0 && echo "no unsigned-module taint"
# the three wolf modules are resident:
for m in zfs spl libwolfssl; do lsmod | grep -q "^$m " || echo "FAIL: module '$m' not loaded"; done
# the FIPS DRBG is the system stdrng (the driver name carries 'fips-140-3'):
grep -A6 wolfcrypt /proc/crypto | grep -E 'driver|priority|selftest'
#   driver: sha2-256-drbg-nopr-wolfentropy-wolfcrypt-fips-140-3-with-global-replace  priority:100000  selftest:passed
```

Userspace provider and SSH:

```bash
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf openssl list -providers | grep -i 'wolfSSL Provider FIPS'
echo x | OPENSSL_CONF=/etc/wolfprovider/openssl.cnf openssl dgst -md5 \
  && echo "BAD: MD5 served" || echo "OK: MD5 refused"
# sshd is scoped to the provider and serving:
systemctl show ssh -p Environment | grep -o 'OPENSSL_CONF=[^ ]*'
systemctl is-active ssh                                 # -> active
```

### 13.2 Encrypted storage and recovery

Unlock the dataset and verify encryption and module routing:

```bash
zfs get -H -o value keystatus datatank/secure           # -> unavailable, until the key is loaded
# Use wolfProvider for passphrase derivation; skip load-key if the boot service already loaded it:
[ "$(zfs get -H -o value keystatus datatank/secure)" = available ] || \
  OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs load-key datatank/secure   # enter the passphrase
systemctl start wolfzfs-unlock                          # the §10 unit mounts the dataset
systemctl is-active wolfzfs-unlock                      # -> active
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs get -H -o name,value encryption,mounted datatank/secure   # -> aes-256-gcm ; yes
journalctl -k -b | grep 'zfs using libwolfssl'          # ZFS is routed to wolfCrypt on this boot
```

Confirm the dataset is inaccessible while locked and returns identical data after unlocking:

```bash
dd if=/dev/urandom of=/datatank/secure/test.bin bs=1M count=64 status=none && sync
sha256sum /datatank/secure/test.bin                     # note this hash
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs unmount datatank/secure
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs unload-key datatank/secure   # the dataset is now locked
# the locked half — the key is gone and the dataset is not mounted:
zfs get -H -o value keystatus datatank/secure           # -> unavailable
mountpoint -q /datatank/secure && echo "BAD: still mounted" || echo "OK: locked and unmounted"
# Reload, mount, and compare; pvestatd may have mounted the dataset already (§15.2):
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs load-key datatank/secure     # re-enter the passphrase
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs mount datatank/secure || mountpoint -q /datatank/secure
sha256sum /datatank/secure/test.bin                     # must equal the first hash, byte for byte
rm -f /datatank/secure/test.bin
```

Finally, confirm that the guests created in §9 still run off the encrypted storage under `fips=1`:

```bash
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs get -H -o name,value encryption datatank/secure/vm-700-disk-0 datatank/secure/subvol-710-disk-0
#   -> both aes-256-gcm
qm start 700 && sleep 5 && qm status 700                # -> status: running
pct start 710 && pct exec 710 -- ls /                   # -> lists the container's root filesystem
qm stop 700; pct stop 710
```

### 13.3 Expected state

| Surface | Expected under `fips=1` |
|---|---|
| Kernel | cmdline `fips=1`; `fips_enabled=1`; `FIPS 140-3 wolfCrypt-fips v7.0.0 startup self-test succeeded` |
| Kernel RNG | `stdrng` driver name contains `fips-140-3`, priority 100000, selftest passed |
| Modules | `libwolfssl`, `spl`, and `zfs` loaded; no unsigned-module taint |
| Data at rest | `datatank/secure` reports `aes-256-gcm`; `zfs using libwolfssl` present this boot; the dataset unlocks after reboot and the round trip returns the identical hash |
| Guest storage | guest disks and subvols under `datatank/secure` report `aes-256-gcm`; guests start from them |
| Userspace | `wolfSSL Provider FIPS` active; MD5 refused |
| SSH | `sshd` scoped to the provider; ECDSA and RSA host keys and user keys only |

---

## 14. ZFS administration

### 14.1 Snapshot and rollback

Use the standard `snapshot`, `rollback`, and `delsnapshot` commands in
[`qm`](https://pve.proxmox.com/pve-docs/qm.1.html) and [`pct`](https://pve.proxmox.com/pve-docs/pct.1.html).
Rollback stops a running guest. Prefix dataset-level `zfs snapshot` and `zfs rollback` with
`OPENSSL_CONF` (§8.1).

**Caution:** Dataset rollback rewinds all files in that dataset, including `vzdump` archives or a PBS
datastore stored there. Child zvols and subvols are separate datasets and are unaffected. Use
`qm rollback` or `pct rollback` for guest state.

### 14.2 Key rotation

`zfs change-key` re-wraps the master key under a new passphrase or keyfile. Data is not rewritten, and
the old secret stops working immediately.

```bash
# Every key operation below carries the §8.1 prefix so the new wrapping key is derived by wolfCrypt:
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs change-key datatank/secure            # keylocation=prompt: enter the new passphrase
#   (keyfile model: add -o keylocation=file:///root/new.key -o keyformat=passphrase <ds>)
# Confirm the new passphrase unlocks the dataset:
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs unmount datatank/secure
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs unload-key datatank/secure
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs load-key datatank/secure   # enter the NEW passphrase
# pvestatd may mount the dataset first (§15.2) — accept either outcome:
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs mount datatank/secure || mountpoint -q /datatank/secure
ls /datatank/secure                                                      # data intact
```

---

## 15. Troubleshooting

### 15.1 `error: bad shim signature` at boot

A Secure Boot node whose boot default is the custom kernel stops at GRUB with this
error when the signing certificate is not trusted — the MOK enrollment was missed, or its pending
request was discarded. Boot a stock Proxmox kernel from GRUB **Advanced options**, confirm Foundation
§2.6 produced the signed image, then repeat the §5.2 enrollment and reboot.

### 15.2 `zfs mount` reports `filesystem already mounted`

This message is harmless: after the key is loaded, `pvestatd` may mount `datatank/secure` before the
guide's mount command runs.

### 15.3 Outbound `ssh` from `root` still offers chacha20-poly1305

Proxmox installs `/root/.ssh/config` with a `Ciphers` list including `chacha20-poly1305@openssh.com`.
It overrides §11's client cipher list for `root`, including Proxmox operations; the key-exchange, MAC,
and host-key restrictions still apply. Check the effective settings and adjust the per-user file if
required:

```bash
ssh -G <host> | grep -E '^(ciphers|kexalgorithms|macs) '   # as root
```

### 15.4 `mokutil --import` reports `Already in kernel trusted keyring. Skip`

The certificate is already trusted; no enrollment is queued. The `mokutil --list-new` pipeline then
matches nothing and exits non-zero. No action is needed.

### 15.5 Guests do not start after boot; `pve-guests.service` refuses a manual start

`wolfzfs-unlock.service` failed — the key was not loaded, or `datatank/secure` did not mount — and the
§10.1 `Requires=` drop-in then fails `pve-guests.service` closed. Its refusal of a manual start with
`may be requested by dependency only` is by design. Correct the unlock (§10.2), then start the guests
with `qm start <vmid>` / `pct start <ctid>`.

### 15.6 `modprobe -r zfs spl` reports `FATAL: Module zfs is in use.`

A pool is imported, so the stock module cannot be unloaded and the `modprobe zfs` that follows does
nothing. Reboot instead — `updates/zfs.ko` outranks the stock module by depmod priority and the pool
re-imports under wolfZFS — then run §7's checks.

---

## 16. Next steps

Continue with **Node Configuration and Recovery** to configure management TLS and cipher suites on
`pveproxy`, which still uses system OpenSSL. That guide also covers access recovery and reverting to
`fips=0`.

The guides that follow ask for these values:

| Value | This deployment |
|---|---|
| PVE storage id | `zdata` |
| encrypted dataset | `datatank/secure` |
| mounted at | `/datatank/secure` |
