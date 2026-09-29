# wolfSSL FIPS on Proxmox VE — Migration Acceptance

Use this guide after either migration guide to verify disk placement, guest checksums, and recovery
after a node restart. Guest encryption and driver or firmware conversion are outside its scope.

Do not retire the source until every check here passes.

---

## 1. Verify the migrated guest

### 1.1 Check disk placement

```bash
# virtual machine:
qm config <vmid>  | grep -E '^(scsi|virtio|sata|ide|efidisk|tpmstate|unused)[0-9]+:'
# container:
pct config <vmid> | grep -E '^(rootfs|mp[0-9]+|unused[0-9]+):'
#   -> every line names the FIPS storage
```

Check detached (`unusedN`) volumes as well. Attach and move any on the old storage, or remove them
deliberately.

Place virtual TPM state according to the destination:

| Destination | `tpmstate0` |
| --- | --- |
| ZFS or LUKS | On the encrypted storage, with every other volume |
| Ceph | On a local dm-crypt storage. Not on the RBD pool |

**Important:** on Ceph, `tpmstate0` on the RBD pool causes startup to fail with `rbd: sysfs write failed`.
Boot, EFI, and data disks are unaffected.

Prepare the local dm-crypt storage before moving TPM state. Standard `local` and `local-lvm` storage
are unencrypted. Use `aes-cbc-essiv:sha256` with `--pbkdf pbkdf2`, and register it with
`--is_mountpoint 1` to prevent writes to the unencrypted root filesystem while the storage is locked.

```bash
qm move-disk <vmid> tpmstate0 <local-dm-crypt-storage>
```

**Note:** node-local TPM state prevents the guest from failing over to another node.

### 1.2 Start the guest

Stop the source guest first to avoid identity and network conflicts. `vzdump --mode stop` restarts
the source when the backup completes:

```bash
# on the source node, where the source is a Proxmox VE deployment:
qm stop <source-vmid>
```

**Note:** on Ceph, start a guest with new TPM state from the web interface or API. `qm start` from a
root shell configured to use wolfProvider fails to create that state with `TASK ERROR`. Existing TPM
state and ZFS or LUKS destinations are unaffected.

For other guests, use `qm start` below. After either start method, confirm that the guest is running:

```bash
# on the destination node:
qm start <vmid>
qm status <vmid>                        # -> status: running
```

### 1.3 Verify guest checksums

Inside the guest, check the hashes recorded before migration:

```bash
# in the guest, as root:
sha256sum -c /root/pre-migration.sha256         # -> OK
```

For an existing Proxmox Backup Server snapshot, use the verification records from when it was taken
and the same absolute paths. A newly created migration stamp will not be present.

---

## 2. Verify recovery after a node restart

Reboot the node and follow its recovery procedure. Confirm that the guest returns and its checksums
match.

Record the boot ID before restarting:

```bash
cat /proc/sys/kernel/random/boot_id     # record this value
systemctl reboot                        # on the node the migrated guest is running on
```

When the node answers again, confirm it restarted and came back in FIPS mode:

```bash
cat /proc/sys/kernel/random/boot_id     # -> must differ from the recorded value
cat /proc/sys/crypto/fips_enabled       # -> 1
grep -o 'fips=[01]' /proc/cmdline       # -> fips=1 (check this explicitly: fips_enabled
                                        #    reads 1 even with fips=0, because the
                                        #    wolfCrypt module enables it on load)
```

**Important:** if the boot ID is unchanged, the node has not restarted. Resolve this before continuing.

Follow the matching subsection:

| Destination | On return | Recovery |
| --- | --- | --- |
| ZFS, passphrase model | Storage locked; guest has not started | `zfs load-key`, then `systemctl restart wolfzfs-unlock` |
| ZFS, keyfile model | Storage already unlocked; guest has already restarted | None, unless the unlock unit failed |
| LUKS | Storage already unlocked; guest has already restarted | None, unless the unlock unit failed |
| Ceph | Storage already unlocked; guest has already restarted | None |

### 2.1 ZFS with passphrase unlock

With passphrase unlock, storage remains locked and guests are stopped. For keyfile unlock, use §2.2.

```bash
qm status <vmid>                        # -> status: stopped
```

Confirm that the locked guest filesystem cannot be read. Expect no output and `rc=2`:

```bash
blkid -c /dev/null /dev/zvol/<pool>/<dataset>/vm-<vmid>-disk-0-part1 ; echo "rc=$?"
#   -> no output, rc=2.  With the key loaded the same command names the guest's filesystem.
```

Keep `-c /dev/null` to bypass cached filesystem information.

Check wolfProvider before unlocking. Keep the `OPENSSL_CONF` prefix on the unlock commands:

```bash
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf openssl list -providers | grep -i 'wolfSSL Provider FIPS'
```

Load the key using either the interactive prompt:

```bash
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs load-key <pool>/<dataset>
```

or stdin, for example from a secrets manager:

```bash
printf '%s\n' "$PASSPHRASE" | OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs load-key <pool>/<dataset>
```

Loading the key leaves the dataset unmounted. Restart the unlock service to mount it, then confirm
the mount and key state:

```bash
systemctl restart wolfzfs-unlock
zfs list -o name,mounted,keystatus <pool>/<dataset>    # -> yes / available
```

Then start the guests and confirm the data again:

```bash
pvesh create /nodes/$(hostname)/startall
qm status <vmid>                        # -> status: running
# in the guest, as root:
#   sha256sum -c /root/pre-migration.sha256     # -> OK
```

**Important:** `startall` starts only guests with `onboot=1`. If the flag is unset, set it or start the
guest explicitly; a successful task exit alone does not confirm startup.

### 2.2 LUKS, Ceph, or ZFS with keyfile unlock

Storage unlocks automatically, and `pve-guests.service` starts guests with `onboot=1`.

The node may answer SSH before guest startup finishes. Wait for `pve-guests` to become active before
checking the guest:

```bash
cat /proc/sys/kernel/random/boot_id     # -> must differ from the value recorded in §2
systemctl is-active pve-guests          # wait for -> active
pvesm status | grep <fips-storage>      # -> the FIPS storage, active
qm status <vmid>                        # -> status: running
# in the guest, as root:
#   sha256sum -c /root/pre-migration.sha256     # -> OK
```

If the guest remains stopped, check `onboot=1` and the `startall` task log. Determine the cause before
starting the guest manually.

If a LUKS volume remains closed, its unlock unit has failed. `systemctl start luks-unlock.service`
reopens it and mounts `/mnt/secure`.

---

## 3. Retire the source

Retain the source for rollback until every check passes, then retire it under your retention policy.

**Caution:** do not run the source and migrated guest on the same network. For a Proxmox source,
disable autostart as soon as the migrated guest is in service:

```bash
# on the source node, where the source is a Proxmox VE deployment:
qm set <vmid> --onboot 0
```

If site policy requires a decrypted VMware source to remain encrypted until disposal, restore its
encryption storage policy or retain it on protected storage.

**From Proxmox VE.** Deleting a volume frees space but does not erase its contents.

**Caution:** destroy the source only after acceptance passes and it is no longer needed for rollback.

```bash
# on the source node, once every check above has passed:
qm destroy <vmid> --purge 1                     # pct destroy <vmid> --purge 1, for a container

# confirm nothing remains of the guest on the source storage:
pvesm list <source-storage> --vmid <vmid>       # -> no volumes
```

**From VMware vSphere.** Remove the ESXi registration from Proxmox, then dispose of the source VM
under your vSphere retention policy:

```bash
pvesm remove <esxi-src>
pvesm status | grep <esxi-src>          # -> no output
```

**Note:** ESXi storage ignores `--vmid` in `pvesm list` and lists every VM on the host. Use the status
check above to confirm removal of the registration.

Erase or destroy source media under your media-sanitisation policy.

---

## 4. Rollback

Before the source is retired, roll back by stopping the migrated guest, then starting the source.
Only one copy may run.

**Important:** `qm remote-migrate` leaves the source guest locked. If `qm start` reports
`VM is locked (migrate)`, clear the lock before starting:

```bash
# on the source node:
qm unlock <vmid>
qm start <vmid>
```

After the source is retired, restore from backup using the Backup guide.
