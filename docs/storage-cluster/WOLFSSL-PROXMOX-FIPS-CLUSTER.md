# wolfSSL FIPS for Proxmox VE — Cluster Configuration

Use this guide for new clusters made entirely of freshly prepared wolfSSL nodes, and for their
ordinary operation. Read section 1 before installing the first node. Run commands as root;
replace angle-bracket placeholders with site-specific values.

## 1. Plan the cluster

Use the [Proxmox Cluster Manager](https://pve.proxmox.com/pve-docs/chapter-pvecm.html) for network,
quorum and membership planning. Keep the normal UDP corosync transport. The wolfSSL integration
blocks SCTP module loading; do not configure `knet_transport: sctp`. A QDevice uses NSS rather
than this corosync OpenSSL path; use a third Proxmox node when a tie-breaking vote is required
within this deployment model.

Prepare nodes in this order:

1. For Ceph nodes, install stock Tentacle first as described in [Ceph, section 1](../ceph/WOLFSSL-PROXMOX-FIPS-CEPH-DEPLOYMENT.md#1-prepare-the-ceph-packages).
2. Complete [Foundation](../_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md), including the selected
   storage metapackage, SSH key preparation, reboot and verification, on every node. Use the
   same wolfSSL channel throughout the cluster. FIPS Ready is not a FIPS-validated module;
   Foundation describes the licensed module's scope.
3. Complete local [ZFS](WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-ZFS.md) or
   [LUKS](WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-LUKS.md) configuration where selected. Ceph nodes
   instead complete [Ceph, section 2](../ceph/WOLFSSL-PROXMOX-FIPS-CEPH-DEPLOYMENT.md#2-install-and-activate-the-wolfssl-integration)
   and wait until after section 2 below to initialize Ceph.
4. Create the Proxmox cluster, configure corosync and verify every member as below. Only then
   initialize Ceph or place production guests on the cluster.

Do not join stock nodes or convert an existing stock cluster in place. A new member must
complete the same preparation before joining. Returning a node to stock requires reinstallation.

Keep joining nodes free of guests. A join replaces their `/etc/pve`, including storage
configuration and PBS encryption keys, with the cluster's copy. Before joining, retain any
backup encryption keys outside the cluster and plan one cluster-wide storage definition for
each shared service. Limit node-local storage definitions to the nodes that actually provide
them. A `localhost` PBS address refers to a different server on each node.

## 2. Create and verify the cluster

Follow Proxmox's [create and join procedure](https://pve.proxmox.com/pve-docs/chapter-pvecm.html)
with the network options chosen for the site. On the first node only, create the cluster:

```sh
pvecm create <cluster-name>
```

Before joining other nodes, configure the OpenSSL crypto model. The generated configuration
defaults to NSS, which can prevent corosync from starting. Without quorum, `/etc/pve` is not
writable, so first edit the local copy as described in Proxmox's
[configuration recovery procedure](https://pve.proxmox.com/pve-docs/chapter-pvecm.html#pvecm_edit_corosync_conf):

```sh
cp /etc/corosync/corosync.conf /etc/corosync/corosync.conf.new
nano /etc/corosync/corosync.conf.new
```

In the existing `totem` section, set these entries and increment the existing `config_version`
by one. Preserve the other cluster settings.

```text
crypto_model: openssl
secauth: on
crypto_cipher: aes256
crypto_hash: sha256
```

Publish the local file and restart corosync:

```sh
mv /etc/corosync/corosync.conf.new /etc/corosync/corosync.conf
systemctl restart corosync
pvecm status
pvecm nodes
```

Require `Quorate: Yes` and only the first node as a member. Then publish the same configuration
to the shared filesystem before any joins:

```sh
cp /etc/corosync/corosync.conf /etc/pve/corosync.conf.new
mv /etc/pve/corosync.conf.new /etc/pve/corosync.conf
cmp /etc/corosync/corosync.conf /etc/pve/corosync.conf
```

Require `cmp` to report no differences and exit zero. Foundation supplies the node-local
service configuration; joining nodes receive the shared corosync configuration.
The OpenSSL model is required even on FIPS Ready nodes that have wolfSSL-backed NSS packages.
Do not select the NSS model.

On each joining node, run:

```sh
pvecm add <existing-member-address>
```

The join prompts for authentication. After each join, check membership on every current member;
a refused `pvecm add` can return zero, so command exit status alone is insufficient:

```sh
pvecm status
pvecm nodes
```

Require `Quorate: Yes` and the intended members before continuing.

Reboot one node at a time, waiting for it to return to quorum before rebooting another. This
activates the service configuration and corrects the corosync authkey permissions on joined
nodes. Keep console access available: a provider-load failure prevents corosync from starting.

```sh
reboot
```

Reconnect to each returned node and run:

```sh
pvecm status
wolfssl-fips-pve-verify
```

Require the corosync check to pass, including provider use, AES-256/SHA-256 settings and key
permissions. Quorum alone does not prove that corosync uses wolfCrypt. Resolve applicable
failures before continuing; on Ceph-prepared nodes, the no-running-Ceph-daemon check remains
expected only until [Ceph initialization](../ceph/WOLFSSL-PROXMOX-FIPS-CEPH-DEPLOYMENT.md#3-create-the-cluster-and-encrypted-osds).

After a join, review `/etc/pve/storage.cfg` and its node restrictions. The joining node must
have the actual encrypted storage and unlock configuration required by every storage assigned
to it; inheriting the storage definition does not create or unlock its volumes.

## 3. Migrate guests only to available encrypted storage

Keep Proxmox's migration type `secure`. If setting it in `/etc/pve/datacenter.cfg`, preserve
any selected migration network:

```text
migration: secure
```

Secure migration transports memory and local disks over SSH. Foundation provides the compatible
SSH implementation and keys. Do not select `insecure`, which can send guest memory unencrypted.
Use Proxmox's [migration procedure](https://pve.proxmox.com/pve-docs/chapter-pvecm.html#_guest_migration)
for the workload, including `--with-local-disks` when live-migrating a VM with local disks.
Containers use restart migration. Check every guest volume, including EFI and TPM-state storage,
and any explicit target-storage mapping.

Before migration, restore or replication, check the receiving storage on the **target node**:

| Target | Required check |
| --- | --- |
| ZFS | `zfs get encryption,encryptionroot,keystatus <pool/dataset>` must show the intended encrypted root and `available`. An `active` Proxmox storage status alone does not prove the key is loaded. |
| LUKS | `cryptsetup status <mapper-name>` must show the intended active mapping; `findmnt --mountpoint <mountpoint>` must show that mapping as the filesystem source. |
| Ceph | Complete the Ceph guide's OSD, secure-client and cluster-health checks. |

**Warning:** An offline VM or container migration to a closed LUKS directory can write the disk
onto the unencrypted root filesystem, report success and remove the source. The storage's
`is_mountpoint` safeguard does not prevent this offline path. Confirm the target mount and
mapping immediately before the operation. Abort if either is missing. Unlocking later can
hide the plaintext copy beneath the mount; use the LUKS guide's recovery procedure before
starting the guest.

After the move, confirm all guest disks landed on the intended encrypted target and verify the
guest's data. Encryption of the source does not ensure encryption at the destination.

## 4. Replicate and back up encrypted guests

Use [Proxmox storage replication](https://pve.proxmox.com/pve-docs/chapter-pvesr.html) for local
ZFS storage. LUKS directory storage does not support `pvesr`. Replication uses the SSH transport;
the receiving ZFS dataset encrypts the data under its own encryption root. Apply section 3's
unlock check before replication and check the received volume:

```sh
zfs get encryption,encryptionroot,keystatus <target-pool/dataset/guest-volume>
```

For backups and restores, follow [Backup](../_shared/WOLFSSL-PROXMOX-FIPS-BACKUP.md). Use an
encrypted destination or PBS client-side encryption as that guide requires. Keep the PBS
address reachable from every participating node and retain encryption keys outside `/etc/pve`.
Removing a PBS storage definition deletes its local encryption-key file; preserve the key
before replacing that definition. Test a restore from another member onto its encrypted
storage before relying on a cluster-wide backup policy.

## 5. Maintain and restart the cluster

Follow [Node maintenance](../_shared/WOLFSSL-PROXMOX-FIPS-NODE-CONFIGURATION.md) for APT updates.
Update and reboot one node at a time. Evacuate guests only to targets that pass section 3,
and wait for quorum and verification to recover before continuing. Keep console access for
provider or boot recovery. Restarting services or rebooting is required to activate new
libraries; installed package versions alone do not establish activation.

For Ceph, require clean placement groups before maintenance and use Ceph's stop checks before
rebooting an OSD/monitor node:

```sh
ceph -s
ceph osd ok-to-stop $(ceph osd ls-tree $(hostname))
ceph mon ok-to-stop $(hostname)
```

Run only the checks for roles hosted on that node, and proceed only when each applicable
check succeeds. After reboot, require quorum, all expected OSDs up/in, clean placement groups
and `wolfssl-fips-pve-verify --ceph-cluster` before the next member.

For a full shutdown, follow Proxmox's
[Ceph HCI shutdown sequence](https://pve.proxmox.com/pve-docs/chapter-pveceph.html#pveceph_shutdown)
when using Ceph. Before powering off any node, set each HA resource to `ignored` and wait until
`ha-manager status` shows every node's LRM `idle`; otherwise an armed HA watchdog can reset a
node when quorum is lost. For each configured resource:

```sh
ha-manager set <resource-id> --state ignored
ha-manager status
```

After a cold start, quorum can return while local encrypted storage is still locked. Complete
the ZFS or LUKS guide's unlock and mount checks before starting guests or accepting migrations.
Ceph OSDs unlock through their cluster-managed keys. Restore each HA resource's intended state
only after storage and cluster verification succeed; for a resource intended to run:

```sh
ha-manager set <resource-id> --state started
```
