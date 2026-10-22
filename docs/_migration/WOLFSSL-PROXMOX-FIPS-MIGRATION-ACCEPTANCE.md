# wolfSSL FIPS for Proxmox VE — Migration acceptance

Use this guide after importing or restoring guests onto a freshly deployed target. Verify
storage placement and guest data before retiring the source. Commands run as root on the
destination node unless labeled otherwise.

## 1. Check the destination

Confirm that every persistent guest volume is on the intended encrypted storage:

```sh
# Virtual machine
qm config <vmid>

# Container
pct config <vmid>
```

Include boot and data disks, EFI variables, TPM state, detached disks and container mount
points. Container bind mounts and passed-through devices require separate review because
a restore's target-storage option does not relocate external data. An encrypted primary
disk does not encrypt other volumes automatically.

Check storage availability and the host configuration:

```sh
pvesm status
wolfssl-fips-pve-verify --login
```

Resolve all failures. For ZFS, check the dataset's encryption root and loaded key using the
[ZFS guide](../storage-cluster/WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-ZFS.md). For LUKS, check the
mapping and mounted filesystem using the
[LUKS guide](../storage-cluster/WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-LUKS.md). For Ceph, check
encrypted OSDs and secure messaging using the [Ceph guide](../ceph/WOLFSSL-PROXMOX-FIPS-CEPH-DEPLOYMENT.md).

## 2. Verify the guest

Keep the source stopped while its replacement is running. Start the destination guest:

```sh
# Virtual machine
qm start <vmid>

# Container
pct start <vmid>
```

Check the application, network access and expected data. Where the source guide recorded a
Linux checksum manifest, run inside the restored guest:

```sh
sha256sum -c /root/migration.sha256
```

Every recorded file must match. For an older backup, compare against that backup's records,
not a checksum captured after the backup was taken.

## 3. Confirm recovery and backup

In a maintenance window, confirm the target's storage unlock and guest startup after a node
reboot, following the selected deployment guide. For a cluster, use the
[Cluster maintenance procedure](../storage-cluster/WOLFSSL-PROXMOX-FIPS-CLUSTER.md) so quorum,
HA and Ceph availability are preserved. Complete the selected guide's unlock checks before starting guests.

Repeat the guest data check after restart. Test a backup and restore of the migrated guest
using the [Backup guide](../_shared/WOLFSSL-PROXMOX-FIPS-BACKUP.md), keeping restored test
copies isolated from the production guest.

## 4. Retire the source

Retain the source, its backup and all required encryption keys until the application owner
accepts the target and its recovery procedure. Remove or retire the source through the
normal Proxmox or VMware administration procedure.

If cutover is abandoned, stop the destination guest before restarting the source. Writes
made after cutover are not copied back automatically; reconcile them before returning the
application to the source.
