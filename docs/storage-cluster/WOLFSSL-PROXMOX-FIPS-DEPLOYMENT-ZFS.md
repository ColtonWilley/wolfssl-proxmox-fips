# wolfSSL FIPS for Proxmox VE — Encrypted ZFS Storage

Create new encrypted data storage with wolfZFS after completing
[Foundation](../_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md). The result is an encrypted
ZFS dataset for guest volumes, with file-based boot unlock ordered before guest startup.

## 1. Requirements and encryption choices

Use a fresh Foundation node with an ext4/LVM root and unused data disks. Foundation must have
installed **both** `wolfssl-fips-pve-apt` and `wolfssl-fips-pve-apt-zfs` before replacing packages,
selected `wolfssl-fips-pve-zfs`, and completed its reboot and verification. Do not install the
ZFS and Ceph storage metapackages together; they conflict. Read
[Cluster planning](WOLFSSL-PROXMOX-FIPS-CLUSTER.md) before preparing cluster storage.

This procedure uses AES-256-GCM and a 32-byte raw key. wolfZFS routes dataset encryption through
the kernel wolfCrypt module. A raw key avoids ZFS passphrase derivation: ZFS fixes the PBKDF2 salt
at 64 bits; do not treat that passphrase derivation as an approved service.
Foundation explains the distinction between FIPS Ready and licensed modules and their applicability.

The key remains on the root filesystem so the data unlocks unattended. This protects a data disk
removed separately; possession of the root disk and data disks also provides the key. Keep a
protected recovery copy off the node before storing guest data. Do not put the only key copy on
the dataset it unlocks. This procedure does not encrypt the host root filesystem.

Run commands as root. `datatank`, `datatank/secure`, `/datatank/secure`, `zsecure` and `zbackup`
are example pool, dataset, mount and storage names; substitute consistently. Replace `<data-disk>`
with the persistent ID of an unused data disk. Choose pool redundancy and devices using the
[Proxmox ZFS administration guide](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#chapter_zfs).

## 2. Verify wolfZFS activation

Foundation installs `wolfssl-fips-pve-zfs` with the common packages. The metapackage supplies matching
ZFS tools and kernel modules; the current APT release uses OpenZFS **2.4.4**, including the
licensed channel. Installation does not replace an already loaded ZFS module. Complete
Foundation's reboot before creating encrypted storage.

Check the installed packages, running module and current boot log:

```bash
dpkg-query -W -f='${binary:Package}\t${db:Status-Status}\t${Version}\n' \
  wolfssl-fips-pve-apt wolfssl-fips-pve-apt-zfs wolfssl-fips-pve-zfs zfsutils-linux
zfs version
cat /proc/sys/crypto/fips_enabled
journalctl -k -b --no-pager | grep -i 'zfs using libwolfssl'
```

Require every listed package to report `installed`, matching ZFS userspace/kernel releases,
FIPS state `1` and the `zfs using libwolfssl` message from this boot. Keep the metapackage and
both pin packages installed for updates. If these checks fail, stop here and use
[Node configuration and maintenance](../_shared/WOLFSSL-PROXMOX-FIPS-NODE-CONFIGURATION.md).

## 3. Create the encrypted dataset and boot unlock

**Warning:** Pool creation writes to the selected disks. Confirm their identities and that no
required data remains. The example below is a single-disk pool; adapt its device layout to the
redundancy selected in §1.

```bash
zpool create -O mountpoint=none datatank /dev/disk/by-id/<data-disk>
install -d -m 700 /etc/zfs/keys
(umask 077; dd if=/dev/urandom of=/etc/zfs/keys/datatank-secure.key bs=32 count=1 status=none)
zfs create -o encryption=aes-256-gcm -o keyformat=raw \
  -o keylocation=file:///etc/zfs/keys/datatank-secure.key \
  -o mountpoint=/datatank/secure -o overlay=off datatank/secure
systemctl enable --now "wolfzfs-unlock@$(systemd-escape datatank/secure).service"
```

Keep the pool root at `mountpoint=none` and give the encrypted dataset its own mountpoint. This
prevents a later pool-root mount from hiding the unlocked dataset. `overlay=off` refuses to mount
on a nonempty directory, exposing files accidentally written there while the dataset was locked.

Enable one package unlock service for each encryption root. Descendants inheriting this root's
key do not need separate instances. The service loads the file key and mounts the dataset before
`pve-guests`; a failed unlock prevents **all** automatic guest startup on the node.

## 4. Register encrypted Proxmox storage

Point the ZFS backend at the encrypted dataset, so new VM zvols and container datasets inherit its
encryption. Register a directory on that dataset for local backup archives:

```bash
mountpoint /datatank/secure || exit 1
pvesm add zfspool zsecure --pool datatank/secure --content images,rootdir
mkdir -p /datatank/secure/backup
pvesm add dir zbackup --path /datatank/secure/backup --content backup \
  --is_mountpoint /datatank/secure || exit 1
pvesm set local --content iso,vztmpl
```

Continue only if `mountpoint` confirms the encrypted mount. The directory storage's
`is_mountpoint` must name `/datatank/secure`, even when its storage path is below it. This prevents
ordinary storage use of the underlying root directory when the dataset is locked. It does not
protect every offline migration; follow the Cluster guide's destination checks.

Disable any other unencrypted guest/backup destination on this fresh node. For an unused
installer-created `local-lvm`, use:

```bash
pvesm set local-lvm --disable 1
```

Run that command only if the storage exists; disable other unencrypted destinations by their
actual IDs. Configure backups and encrypted staging with the
[Backup guide](../_shared/WOLFSSL-PROXMOX-FIPS-BACKUP.md) before running backup jobs. Storage on the
same pool is not an independent recovery copy.

For clusters, restrict each storage to nodes with the corresponding encrypted dataset, using
`pvesm set <storage-id> --nodes <node-list>`; these are local pools, not shared storage. Routine
backend options are documented in [Proxmox ZFS storage](https://pve.proxmox.com/pve-docs/chapter-pvesm.html#storage_zfspool).

## 5. Verify data and unlock after reboot

Before placing production data, write a small test file and keep its expected digest on the root
filesystem. Confirm the dataset is mounted before writing:

```bash
mountpoint /datatank/secure || exit 1
dd if=/dev/urandom of=/datatank/secure/.storage-check bs=1M count=1 status=none
sha256sum /datatank/secure/.storage-check > /root/zfs-storage-check.sha256
sync
reboot
```

After reconnecting:

```bash
systemctl is-enabled "wolfzfs-unlock@$(systemd-escape datatank/secure).service"
systemctl is-active "wolfzfs-unlock@$(systemd-escape datatank/secure).service"
zpool status datatank
zfs get encryption,keyformat,encryptionroot,keystatus,mounted,overlay datatank/secure
findmnt --mountpoint /datatank/secure
sha256sum -c /root/zfs-storage-check.sha256
pvesm status
systemctl is-active pve-guests
wolfssl-fips-pve-verify --login
```

Require an enabled, active unlock service; a healthy pool; `aes-256-gcm`, `raw`, encryption root
`datatank/secure`, key `available`, mounted `yes` and overlay `off`; the expected ZFS mount; a
matching digest; and both storage IDs **active**. Wait for `pve-guests` to become active before
running the verifier. Resolve every failure and review warnings.

Create guests with all volumes on `zsecure`, including EFI, TPM state and container mount points.
Check their storage IDs and `zfs get -r encryption,encryptionroot datatank/secure`; new guest
volumes must inherit the encrypted root. Verify guest data after their first reboot and complete
a backup/restore check from the Backup guide. The node verifier does not establish coverage of
every guest volume or backup destination.

## 6. Resolve unlock failures

```bash
systemctl --failed
journalctl -b -u "wolfzfs-unlock@$(systemd-escape datatank/secure).service" --no-pager
```

Restore a missing key from the protected copy to the configured `keylocation`. A nonempty
mountpoint may contain plaintext written while storage was locked: preserve and inspect those
files before moving them aside, then unlock and copy required data onto the encrypted mount.
Do not bypass this check by enabling `overlay`.

After correcting the cause, start the unlock service from §3 and follow the shared guide's
[storage startup recovery](../_shared/WOLFSSL-PROXMOX-FIPS-NODE-CONFIGURATION.md) before restarting
guests. Stopping an unlock service after normal startup can stop every guest because of the
`pve-guests` dependency; schedule that work as node maintenance.
