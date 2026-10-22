# wolfSSL FIPS for Proxmox VE — Encrypted LUKS Storage

Create a new encrypted data volume with the wolfSSL LUKS packages after completing
[Foundation](../_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md). The result is a directory storage
on LUKS2, unlocked after the kernel wolfCrypt module loads and before guests start.

## 1. Requirements and encryption choices

Use a fresh Foundation node with an ext4/LVM root and an unused data disk or partition. Select
`wolfssl-fips-pve-luks` during Foundation and complete its reboot and verification before continuing. Read
[Cluster planning](WOLFSSL-PROXMOX-FIPS-CLUSTER.md) before preparing cluster storage.

Use these settings for new volumes and keyslots:

| Setting | Required value and reason |
| --- | --- |
| Data and keyslot cipher | `aes-cbc-essiv:sha256`, 256-bit key. The stock AES-XTS default is outside the licensed kernel module's supported AES modes. |
| Key derivation | `pbkdf2` with SHA-256. Argon2 is unavailable in the wolfSSL build. |
| PBKDF2 iterations | `1000000` explicitly. Automatic calibration can exceed the licensed module's 10,000,000-iteration ceiling. |
| Recovery passphrase | At least **14 bytes**; use at least 14 ASCII characters to avoid character/byte ambiguity. |

These parameters select the package-supported storage path; they do not establish certification
of the whole volume or system. In particular, PBKDF2 is outside the licensed userspace module's
approved services. Confirm applicability with wolfSSL as described in Foundation.

The boot key stays on the root filesystem. This protects a data disk removed separately;
possession of the root disk and data disk also provides the key. Keep protected off-node copies
of the key and LUKS header, and retain the recovery passphrase. The host root filesystem remains
unencrypted.

Run commands as root. Replace `<data-disk>` with the persistent ID of the unused data disk or
partition. `cryptdata`, `/mnt/secure` and `secure` are example mapping, mountpoint and storage
names; substitute them consistently.

## 2. Verify the LUKS integration

Foundation installs the LUKS metapackage, matching cryptsetup packages and boot unlock service.
Confirm package installation and activation in the current boot before creating a volume:

```bash
dpkg-query -W -f='${binary:Package}\t${db:Status-Status}\t${Version}\n' \
  wolfssl-fips-pve-luks cryptsetup cryptsetup-bin libcryptsetup12
uname -r
cat /proc/sys/crypto/fips_enabled
journalctl -k -b --no-pager | grep -iE 'FIPS 140-3 wolfCrypt.*self-test succeeded'
```

Require every listed package to report `installed`, cryptsetup versions ending in `~wolfprovN`,
a kernel containing `+wolfcrng`, FIPS state `1` and the successful wolfCrypt self-test from this
boot. Keep the metapackage installed for updates and storage verification. If these checks fail,
return to [Foundation verification](../_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md#6-reboot-and-verify).
Do not open the new volume under a stock kernel: dm-crypt selects its crypto backend when the
mapping opens and retains it for the mapping's lifetime.

## 3. Create the volume and boot unlock

**Warning:** `luksFormat` and `mkfs.ext4` destroy data on their targets. Confirm that the selected
device is unused and is not the root disk. A failed format, including rejection of a short
passphrase, can still write device metadata.

Create a random boot key, format LUKS2, and add an independently held recovery passphrase. The
`luksAddKey` command prompts for the new passphrase; do not put it on the command line.

```bash
install -d -m 700 /etc/luks
(umask 077; dd if=/dev/urandom of=/etc/luks/cryptdata.key bs=64 count=1 status=none)
cryptsetup luksFormat --type luks2 --cipher aes-cbc-essiv:sha256 --key-size 256 \
  --keyslot-cipher aes-cbc-essiv:sha256 --keyslot-key-size 256 \
  --pbkdf pbkdf2 --hash sha256 --pbkdf-force-iterations 1000000 \
  --key-file /etc/luks/cryptdata.key /dev/disk/by-id/<data-disk>
cryptsetup luksAddKey --keyslot-cipher aes-cbc-essiv:sha256 --keyslot-key-size 256 \
  --pbkdf pbkdf2 --hash sha256 --pbkdf-force-iterations 1000000 \
  --key-file /etc/luks/cryptdata.key /dev/disk/by-id/<data-disk>
cryptsetup luksDump /dev/disk/by-id/<data-disk>
```

Require version `2`, data and both keyslot ciphers `aes-cbc-essiv:sha256`, 256-bit keys, and PBKDF2
with SHA-256 and 1,000,000 iterations in both slots. Test the recovery passphrase at the prompt:

```bash
cryptsetup open --test-passphrase /dev/disk/by-id/<data-disk>
cryptsetup open --key-file /etc/luks/cryptdata.key /dev/disk/by-id/<data-disk> cryptdata
mkfs.ext4 /dev/mapper/cryptdata
cryptsetup close cryptdata
cryptsetup luksUUID /dev/disk/by-id/<data-disk>
```

Replace `<luks-uuid>` below with that UUID. Append its entry to `/etc/crypttab` once:

```bash
printf '%s\n' 'cryptdata UUID=<luks-uuid> /etc/luks/cryptdata.key luks,noauto' >> /etc/crypttab
```

Keep `noauto`, and do not add this mount to `/etc/fstab`. The package service must open the volume
after wolfCrypt loads. It reads the key from `crypttab`; it cannot prompt for a passphrase.

```bash
mkdir -p /mnt/secure /etc/wolfssl-fips-pve-luks
printf '%s\n' 'MOUNTPOINT=/mnt/secure' > /etc/wolfssl-fips-pve-luks/cryptdata.conf
systemctl enable --now wolfssl-fips-luks@cryptdata.service
findmnt --mountpoint /mnt/secure
(umask 077; cryptsetup luksHeaderBackup /dev/disk/by-id/<data-disk> \
  --header-backup-file /root/cryptdata-header.img)
```

Require `/mnt/secure` to be mounted from `/dev/mapper/cryptdata`. Copy the header and key to
protected recovery storage before placing guest data. A header backup contains keyslots: protect
it as sensitive material and refresh it after keyslot changes.

The service refuses to mount over files written to `/mnt/secure` while the volume was closed.
Its failure blocks **all** automatic guest startup on the node, including guests on other storage.

## 4. Register encrypted Proxmox storage

```bash
mountpoint /mnt/secure || exit 1
pvesm add dir secure --path /mnt/secure --content images,rootdir,backup --is_mountpoint yes
pvesm set local --content iso,vztmpl
```

Use a directory backend on the mounted filesystem. The verifier does not recognize LVM or
LVM-thin on a LUKS volume as an encrypted destination. `is_mountpoint` stops ordinary guest
startup, restore and online migration from using the underlying plaintext directory while the
volume is closed. It does not prevent every offline migration; follow the Cluster guide's
destination checks.

Disable any other unencrypted guest/backup destination on this fresh node. For an unused
installer-created `local-lvm`, use:

```bash
pvesm set local-lvm --disable 1
```

Run that command only if the storage exists; use the actual IDs for other destinations. In a
cluster, restrict `secure` to nodes with the volume using
`pvesm set secure --nodes <node-list>`. A common storage ID does not make the disks shared.

Put container backup staging on the encrypted mount. After confirming the mount, create its
directory and replace any existing `tmpdir` setting:

```bash
mountpoint /mnt/secure || exit 1
install -d -m 1777 /mnt/secure/tmp
touch /etc/vzdump.conf
sed -i '/^[[:space:]]*tmpdir:/d' /etc/vzdump.conf
printf '%s\n' 'tmpdir: /mnt/secure/tmp' >> /etc/vzdump.conf
```

Mode `1777` permits staging for unprivileged containers. Finish destination, encryption and restore
configuration in the [Backup guide](../_shared/WOLFSSL-PROXMOX-FIPS-BACKUP.md) before running jobs.

Choose `qcow2` for VM disks when snapshots are required. Directory storage does not provide
container snapshots, and a VM with raw TPM state cannot be snapshotted there even if its main
disks use qcow2. Use backups for those guests. Other format and backend options follow
[Proxmox directory storage](https://pve.proxmox.com/pve-docs/chapter-pvesm.html#storage_directory).

## 5. Verify data and unlock after reboot

Before placing production data, write a small test file and record its digest on the root filesystem:

```bash
mountpoint /mnt/secure || exit 1
dd if=/dev/urandom of=/mnt/secure/.storage-check bs=1M count=1 status=none
sha256sum /mnt/secure/.storage-check > /root/luks-storage-check.sha256
sync
reboot
```

After reconnecting:

```bash
systemctl is-enabled wolfssl-fips-luks@cryptdata.service
systemctl is-active wolfssl-fips-luks@cryptdata.service
cryptsetup status cryptdata
findmnt --mountpoint /mnt/secure
sha256sum -c /root/luks-storage-check.sha256
pvesm status
systemctl is-active pve-guests
wolfssl-fips-pve-verify --login
```

Require an enabled, active unlock service; an active LUKS2 mapping with the intended cipher and
device; the expected mapper mount; a matching digest; and `secure` **active**. Wait for
`pve-guests` to become active before running the verifier. Resolve every failure and review
warnings, including the cryptsetup backend, kernel crypto and encrypted-storage checks.

Create guests with every volume on `secure`, including EFI, TPM state and container mount points.
Check their storage IDs, verify guest data after their first reboot, and complete a backup/restore
check from the Backup guide. The node verifier alone does not prove all guest or backup placement.

## 6. Resolve unlock failures

```bash
systemctl --failed
journalctl -b -u wolfssl-fips-luks@cryptdata.service --no-pager
```

Restore a missing boot key to the path in `/etc/crypttab`, then start the unlock service. If only
the recovery passphrase is available, open the mapping interactively before starting the service:

```bash
cryptsetup open /dev/disk/by-id/<data-disk> cryptdata
systemctl start wolfssl-fips-luks@cryptdata.service
```

A nonempty mountpoint can contain plaintext written while the volume was closed. Preserve and
inspect it before moving it aside; unlock the volume and copy required data onto the encrypted
mount. Do not delete the contents merely to make unlock succeed.

Follow the shared guide's [storage startup recovery](../_shared/WOLFSSL-PROXMOX-FIPS-NODE-CONFIGURATION.md)
to resume guests and restore boot-key custody before the next reboot. Stopping the unlock service
after normal startup can stop every guest because of the `pve-guests` dependency; schedule that
work as node maintenance.
