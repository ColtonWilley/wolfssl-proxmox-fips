# wolfSSL FIPS on Proxmox VE — Cluster

Configure and operate a two-node cluster with local ZFS or LUKS encrypted storage. This guide covers
the wolfSSL FIPS changes to Proxmox's
[Cluster Manager](https://pve.proxmox.com/pve-docs/chapter-pvecm.html) procedures: corosync routing,
encrypted guest placement, backup, and recovery. Use standard Proxmox procedures for unchanged steps.

Complete each node's deployment using
**[Foundation](../_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md)**, the deployment guide for its
storage backend — **[ZFS](WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-ZFS.md)** or
**[LUKS](WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-LUKS.md)** — and the
**[Node Configuration and Recovery Guide](../_shared/WOLFSSL-PROXMOX-FIPS-NODE-CONFIGURATION.md)**.

Each node owns its storage; disks are not shared. For shared storage, follow the Ceph deployment guide
instead. Replication (§5) and replica failover (§7.3) require ZFS.

---

## 1. Cryptographic components

| Component | Where it runs | Configuration |
|---|---|---|
| corosync / knet — membership and pmxcfs sync | corosync → libcrypto | Requires both `crypto_model: openssl` and a node-local provider drop-in (§3.2). |
| Guest migration and ZFS replication | OpenSSH tunnel | Receiving `sshd` uses wolfProvider (ZFS §11, LUKS §9); sending `ssh` uses stock OpenSSL (§4.1). |
| Backup and restore | userspace | Container/host: Backup guide §3.1 wrapper. VM: QEMU and `pbs-restore` wrappers in Node Configuration and Recovery §2. See §6. |
| Data at rest | in-kernel wolfCrypt | Each node has its own key. Guests are re-encrypted on the destination (ZFS §8, LUKS §6). |
| Cluster CA and node certificates | stock OpenSSL | Not provider-routed (§8.1). |

Leave management authentication on system OpenSSL; routing `pvedaemon` breaks cluster login (§2.1).

---

## 2. Requirements

Complete this section on **both** nodes before forming the cluster.

### 2.1 Verify both nodes

Each node must pass its deployment checks (ZFS §13, LUKS §10) and use the same kernel and wolfSSL
components, pinned in the deployment guide's §2.3.

```bash
grep -o 'fips=[01]' /proc/cmdline         # -> fips=1
cat /proc/sys/crypto/fips_enabled         # -> 1
uname -r                                  # identical on both nodes
```

**Important:** Do not scope `pvedaemon` to the provider. `pvecm add` authenticates against the first
node's API, and a scoped `pvedaemon` rejects the login ticket with `401`, failing the join.

```bash
grep -rlq OPENSSL_CONF /etc/systemd/system/pvedaemon.service.d/ 2>/dev/null \
  && echo "FAIL: pvedaemon is scoped — the join will 401" || echo "pvedaemon unscoped"
```

### 2.2 Storage identity across nodes

Use the same encrypted storage ID and path on both nodes, each with its own volume and key.

**Important:** `pvecm add` replaces the joining node's `storage.cfg`. A differing storage ID or path
is silently lost. Confirm both nodes agree before joining (§3.4).

```bash
pvesm status
```

On **ZFS**, read the encryption properties of the dataset:

```bash
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs get -H -o name,property,value encryption,encryptionroot <pool>/<dataset>
```

On **LUKS**, confirm the mount is the encrypted mapping, and record the header UUID of each node's
volume:

```bash
findmnt -n -o SOURCE /mnt/secure       # -> /dev/mapper/cryptdata
cryptsetup luksUUID <disk>             # record per node; the two must differ
```

**Note:** `blkid` against `/dev/mapper/cryptdata` reports the filesystem UUID inside the mapping, not
the LUKS header. Read the header UUID from the disk.

### 2.3 SSH keys

**Important:** The deployment's SSH configuration excludes ed25519 and curve25519. Confirm RSA or
ECDSA host and administrative keys; an ed25519-only host key breaks migration.

```bash
grep -E '^HostKey ' /etc/ssh/sshd_config.d/*.conf   # on each node (trailing space: not HostKeyAlgorithms)
ssh-keygen -lf ~/.ssh/<your-key>.pub                # on your workstation — must not report ED25519
```

`pvecm add` establishes node-to-node trust in pmxcfs; do not copy root keys manually. Use standard
cluster networking and name resolution.

### 2.4 The joining node must have no guests

`pvecm add` requires an empty node. Destroy the deployment guide's verification guests (ZFS §9,
LUKS §8), which carry the same ids on either backend:

```bash
qm destroy 700 --purge; pct destroy 710 --purge    # the verification guests
qm list; pct list                                  # both empty on the joining node
```

**Caution:** Move or back up any non-test guests before destroying them. Earlier deployment steps
have not backed them up.

**Caution:** Joining replaces `/etc/pve/priv`. If encrypted PBS storage is configured, copy
`/etc/pve/priv/storage/<id>.enc` off-node first; losing it makes earlier backups unrestorable (§6).
This path is absent if no encrypted PBS storage is configured.

### 2.5 Conventions

Run everything as **root**. The **first node** creates the cluster; the **joining node** runs
`pvecm add`. Values in `<angle brackets>` are yours to supply.

On ZFS, retain the `OPENSSL_CONF` prefix (ZFS §8.1); omitting it silently derives keys with stock
OpenSSL. LUKS `cryptsetup` needs no prefix: provider routing is compiled in (LUKS §4).

Placeholders:

| Placeholder | Value |
|---|---|
| `<cluster-name>` | Cluster name (§3.1) |
| `<first-node>` / `<first-node-ip>` | Node running `pvecm create` |
| `<joining-node>` / `<joining-node-ip>` | Node running `pvecm add` |
| `<pool>/<dataset>` | ZFS dataset, e.g. `datatank/secure` (ZFS §8.2) |
| `<disk>` | LUKS data disk (LUKS §6) |
| `<encrypted-storage>` | Storage ID: e.g. `zdata` (ZFS §9) or `secure` (LUKS §8) |
| `<other-encrypted-storage>` | Second encrypted storage on the same node (§4.4) |
| `<vmid>` / `<ctid>` | Guest IDs (§4) |
| `<id>` | Backup storage ID (§6) |

---

## 3. Create the cluster

Form the cluster only after both nodes boot with `fips=1`.

### 3.1 Create the cluster on the first node

```bash
pvecm create <cluster-name> --link0 <first-node-ip>
```

### 3.2 Route corosync through wolfCrypt

**Important:** Cluster creation defaults to NSS. Set `crypto_model: openssl` and install the provider
drop-in before adding the second node. The model setting alone uses stock OpenSSL; cluster health
does not confirm provider routing.

Edit `corosync.conf` using
[Proxmox's documented procedure](https://pve.proxmox.com/pve-docs/chapter-pvecm.html#pvecm_edit_corosync_conf)
— copy, edit, increment `config_version`, move into place. Inside `totem { }`:

```
  crypto_model: openssl      # the default, nss, is outside the boundary
```

```bash
cp /etc/pve/corosync.conf /etc/pve/corosync.conf.new
# Edit the copy: add `crypto_model: openssl` inside totem { } and increment config_version by one.
# Confirm both edits before replacing the file; moving an unedited copy silently leaves NSS active:
grep -E 'config_version|crypto_model' /etc/pve/corosync.conf.new
mv /etc/pve/corosync.conf.new /etc/pve/corosync.conf
grep -E 'config_version|crypto_model' /etc/pve/corosync.conf                # the move took
# Read the latest reconfiguration, not the exit status. This line is absent without a live model
# change; after restart, use the library-load line and process maps below:
journalctl -u corosync -b | grep 'kronosnet crypto reconfigured' | tail -1   # -> openssl/aes256/sha256
```

Install the node-local provider drop-in using the configuration from Foundation §3.2:

```bash
mkdir -p /etc/systemd/system/corosync.service.d
printf '[Service]\nEnvironment=OPENSSL_CONF=/etc/wolfprovider/openssl.cnf\n' \
  > /etc/systemd/system/corosync.service.d/10-wolfprov.conf
systemctl daemon-reload
systemctl restart corosync
```

Check both the process mappings and the journal:

```bash
COPID=$(pidof corosync)
grep -q libwolfprov.so /proc/$COPID/maps && grep -q libwolfssl.so /proc/$COPID/maps \
  && echo "corosync routed" || echo "FAIL: corosync is not routed to wolfProvider"
journalctl -u corosync -b | grep 'crypto_openssl.so has been loaded' | tail -1   # the model this start loaded
```

After a live model change, old libraries remain mapped; use the latest `reconfigured` line to identify
the active model. After a restart, require both provider mappings and the `crypto_openssl.so` load line.

**Note:** `crypto_model` is replicated. The provider drop-in is not: install it on every joining or
rejoining node (§7.5).

**Caution:** If wolfProvider cannot load, corosync refuses to start; it does not fall back to stock
crypto. In a two-node cluster, this also leaves the peer inquorate. Restart corosync only in a
maintenance window with console access on a live cluster.

### 3.3 Prepare the joining node

Install the drop-in before joining so corosync starts with wolfProvider:

```bash
# on the joining node, still standalone — corosync is not running, so nothing to restart:
mkdir -p /etc/systemd/system/corosync.service.d
printf '[Service]\nEnvironment=OPENSSL_CONF=/etc/wolfprovider/openssl.cnf\n' \
  > /etc/systemd/system/corosync.service.d/10-wolfprov.conf
systemctl daemon-reload
```

### 3.4 Add the joining node

```bash
# on the first node — the fingerprint to check against the join prompt:
openssl x509 -in /etc/pve/nodes/<first-node>/pve-ssl.pem -noout -fingerprint -sha256
# on the joining node:
pvecm add <first-node-ip> --link0 <joining-node-ip>
```

**Important:** A refused join can still exit 0. Confirm membership before continuing:

```bash
pvecm status          # -> the cluster name and both nodes; an error here means the join did not happen
```

The join replaces `storage.cfg` and root's `authorized_keys`. Confirm inherited storage with
`pvesm status`; matching IDs should work unchanged (§2.2). Re-add differing or node-local storage with
`--nodes <node>`. Correct inherited per-node backup storage in §6 (§9.1 describes the error).

Restore your administrative key if the join removed it. Check before appending;
`pvecm updatecerts` does not remove duplicates:

```bash
grep -qxF '<your-admin-public-key>' /etc/pve/priv/authorized_keys \
  || echo '<your-admin-public-key>' >> /etc/pve/priv/authorized_keys
```

**Caution:** `pvecm add` leaves `/etc/corosync/authkey` world-readable, exposing the membership key
without affecting cluster checks. Restrict it after every join:

```bash
stat -c '%n mode=%a' /etc/corosync/authkey     # -> mode=644 on a node that has just joined
chmod 400 /etc/corosync/authkey
stat -c '%n mode=%a' /etc/corosync/authkey     # -> mode=400
```

Then confirm the joining node's corosync came up already routed:

```bash
COPID=$(pidof corosync)
grep -q libwolfprov.so /proc/$COPID/maps && echo "corosync routed" \
  || echo "FAIL: the §3.3 drop-in was missed — apply it and restart corosync in a maintenance window"
```

### 3.5 Verify the cluster

Run on **both** nodes.

```bash
# 1. quorum
pvecm status | grep -E 'Quorate|Total votes|Expected votes'
# 2. corosync routed — both readings, per §3.2
COPID=$(pidof corosync)
grep -q libwolfprov.so /proc/$COPID/maps && grep -q libwolfssl.so /proc/$COPID/maps \
  && echo "$(hostname): corosync routed" || echo "$(hostname): FAIL"
journalctl -u corosync -b | grep 'kronosnet crypto reconfigured' | tail -1   # -> openssl/… or empty
# 3. login succeeds under FIPS — probe the real daemon path, not pvesh
read -rs -p "root@pam password: " RPW; echo
curl -sk -d "username=root@pam&password=$RPW" https://127.0.0.1:8006/api2/json/access/ticket \
  | grep -q '"ticket":"PVE:' && echo "login OK" || echo "FAIL: login rejected"
# 4. Test root SSH with PVE's known-hosts file and alias; bare ssh can fail on a healthy cluster (§9.2):
OTHER=<other-node>
ssh -o BatchMode=yes -o HostKeyAlias=$OTHER \
    -o UserKnownHostsFile=/etc/pve/nodes/$OTHER/ssh_known_hosts \
    -o GlobalKnownHostsFile=none root@<other-node-ip> hostname
# 5. certificates consistent
pvecm updatecerts
```

---

## 4. Moving guests

Verify guest movement in both directions.

**Caution:** Unlock target storage first. ZFS refuses a locked target; an offline LUKS migration can
succeed onto the plaintext root (§4.5, §9.4).

### 4.1 The migration channel

Keep the default `secure` migration policy: it carries guest disks and live RAM through SSH.
The deployment guides configure the receiving `sshd` for wolfProvider and restrict SSH algorithms
(ZFS §11, LUKS §9). The sending `ssh` client still uses stock OpenSSL, inherited from the unscoped
`pvedaemon`/`pvestatd`; it is not provider-routed.

Check the migration policy:

```bash
grep -Eq '^migration:.*\binsecure\b' /etc/pve/datacenter.cfg \
  && echo "FAIL: migration policy is insecure" || echo "migration policy is secure"
```

**Warning:** Do not enable `migration: insecure` for performance; it sends guest RAM in plaintext
over TCP.

During live migration, confirm the transfer uses SSH, not ports 60000/60001:

```bash
ss -tnp | grep -E ':22 |:60000|:60001'
```

### 4.2 Virtual machines

Run each migration from the node that currently owns the guest (§9.3).

```bash
# offline, with the guest stopped. On the origin node:
qm migrate <vmid> <target-node>
# then, on the target node, to bring it back:
qm migrate <vmid> <origin-node>
```

Because the guest disk is local, live migration requires `--with-local-disks`; without it, the command aborts:

```bash
# with the guest running. On the origin node:
qm migrate <vmid> <target-node> --online --with-local-disks
# then, on the target node, to bring it back:
qm migrate <vmid> <origin-node> --online --with-local-disks
```

Confirm on the target after each move that the disk landed under **that node's** key.

On **ZFS**:

```bash
# on the TARGET node:
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs get -H -o name,property,value encryption,encryptionroot <pool>/<dataset>/vm-<vmid>-disk-0
#   -> one row per property: encryption aes-256-gcm, encryptionroot <pool>/<dataset>
```

On **LUKS**, require exactly `/dev/mapper/cryptdata`. Matching `/dev/mapper` alone also accepts the
unencrypted root (`/dev/mapper/pve-root`):

```bash
# on the TARGET node:
findmnt -n -o SOURCE -T /mnt/secure/images/<vmid>/vm-<vmid>-disk-0.raw   # -> /dev/mapper/cryptdata
cryptsetup status cryptdata                                              # -> LUKS2, aes-cbc-essiv:sha256
```

**Important:** Storage names match across nodes and do not identify their keys. Record each node's
ZFS encryption-root `keyguid` or LUKS header UUID; the values must differ:

```bash
# ZFS — keyguid is per-dataset, so read it from the encryption ROOT, not the guest's disk:
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs get -H -o value keyguid <pool>/<dataset>
# LUKS — the header UUID of the node's volume:
cryptsetup luksUUID <disk>
```

### 4.3 Containers

```bash
# stopped container — on the origin node, then the return leg on the target:
pct migrate <ctid> <target-node>
pct migrate <ctid> <origin-node>
# running container:
pct migrate <ctid> <target-node> --restart
pct migrate <ctid> <origin-node> --restart
```

Verify the destination as in §4.2. The container volume is `subvol-<ctid>-disk-0` on ZFS or
`/mnt/secure/images/<ctid>/vm-<ctid>-disk-0.raw` on LUKS.

### 4.4 Moving a disk between encrypted pools on one node

Applies only where a node carries a second encrypted storage: a further encrypted child dataset with
its **own** key (ZFS Deployment Guide §8.2), registered per that guide's §9 with `--nodes <node>` so it
stays node-local.

```bash
qm move-disk <vmid> <disk> <other-encrypted-storage> --delete 1
```

The disk re-encrypts under the destination pool's key.

**Caution:** If the source volume is held open, the copy and the configuration change succeed and the
command still exits zero, but the source delete fails and old-key ciphertext remains. Confirm the
source volume is gone and destroy it by hand if it is not.

### 4.5 Target storage requirements

**Caution:** Migration and replication rely on destination encryption. An unencrypted target accepts
plaintext without error, and moving the disk away does not erase it. Encrypt every possible guest
destination; never point `--targetstorage` at unencrypted storage.

**Caution:** Verify unlock before migration. Locked ZFS storage can report `active`; check `keystatus`
instead. An offline migration to locked LUKS writes to the plaintext root and removes the source,
leaving only the unencrypted copy (§9.4).

---

## 5. Replication

Replication requires ZFS. On LUKS, `pvesr` refuses VM and CT jobs with
`missing replicate feature on volume '<storage>:<vmid>/vm-<vmid>-disk-0.raw'`.

`pvesr` sends plaintext over the same SSH tunnel, and the target dataset re-encrypts under its own key.
Creating and scheduling jobs is standard Proxmox VE
([Storage Replication](https://pve.proxmox.com/pve-docs/chapter-pvesr.html)); verify the landed volume:

```bash
pvesr create-local-job <vmid>-0 <target-node> --schedule '<schedule>'
pvesr run --id <vmid>-0
# on the target:
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs get -H -o name,value encryption,encryptionroot <pool>/<dataset>/vm-<vmid>-disk-0
```

**Caution:** `pvesr delete --force` removes the job but leaves the replica volume and its stale
`@__replicate__` snapshot orphaned on the target, **and a matching snapshot on the source**. Use plain
`pvesr delete`; where `--force` has already been used, remove both orphans by hand.

---

## 6. Cluster backup and restore

Taking, scheduling and restoring backups is standard Proxmox VE
([Backup and Restore](https://pve.proxmox.com/pve-docs/chapter-vzdump.html)); the Backup guide states
what protects an archive.

**Important:** These steps require a PBS server reachable from both nodes, configured separately from
the deployment guides. A node-local backup cannot be restored from the peer; `pvesm list <id>` there
exits 0 with no archives.

**Important:** After joining, `storage.cfg` and `/etc/pve/priv` are shared. Replace per-node backup
definitions with one reachable server and one key; `localhost` resolves differently on each node.
Retain displaced keys for earlier backups (§2.4).

```bash
# 1. save the surviving key first — pvesm remove deletes it:
cp /etc/pve/priv/storage/<id>.enc /root/<id>.enc.saved
# 2. re-add one cluster-wide storage by stable address, reusing that key:
pvesm remove <id>
pvesm add pbs <id> --server <backup-server> --datastore <datastore> \
    --username <user>@<realm> --password <password> \
    --fingerprint <fingerprint> --nodes <node1>,<node2> --encryption-key /root/<id>.enc.saved
```

**Caution:** Changing the server address requires removing and re-adding the storage. Save its key
first: `pvesm remove` deletes `/etc/pve/priv/storage/<id>.enc`, making backups unrestorable if no copy
remains. Supply the username's realm and a password; otherwise re-add fails with `401 Unauthorized`
after the key has been removed.

Before continuing, install the Backup guide §3.1 `proxmox-backup-client` wrapper on every node for
container and host backups. The deployment guides do not install it.

Verify backup and restore across nodes:

```bash
# leg A — back up on one node:
vzdump <vmid> --storage <id> --mode snapshot
# and restore on the other:
pvesm list <id>
qmrestore "<id>:backup/vm/<vmid>/<timestamp>" <new-vmid> --storage <encrypted-storage>
```

Confirm the restored disk landed under the target node's key, as in §4.2, then repeat in the opposite
direction. Both legs must succeed.

Chunk encryption and TLS use the client-side provider. These guides do not configure the backup
server's internal cryptography.

**Important:** VM backups use QEMU through `libproxmox-backup-qemu`; restores use `pbs-restore`.
Install both wrappers from Node Configuration and Recovery §2 on every node. The backup-client wrapper
does not cover them. Check a running VM during a snapshot-mode backup; stop mode provides only a
short-lived process:

```bash
# Get the guest PID from Proxmox, not a process-name match that could select the checking shell:
QPID=$(qm status <vmid> --verbose | awk '/^pid:/{print $2}')
readlink /proc/$QPID/exe                                 # -> .../qemu-system-x86_64
tr '\0' '\n' < /proc/$QPID/environ | grep OPENSSL_CONF   # -> the provider configuration
grep -c libwolfprov /proc/$QPID/maps                     # -> non-zero
```

The provider can load on first TLS or chunk-encryption use. This checks process routing, not which
operation loaded it. For a map count of `0`, see §9.5.

---

## 7. Recovery and lifecycle

### 7.1 A node boots with its encrypted storage locked

Corosync and pmxcfs run from the unencrypted root, so quorum can form while storage is locked and
guests remain stopped.

On **ZFS** with a passphrase, unlock after boot:

```bash
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs load-key <pool>/<dataset>
systemctl start wolfzfs-unlock            # the unlock unit from ZFS Deployment Guide §10
pvesh create /nodes/$(hostname)/startall  # starts only guests with onboot=1 (ZFS Deployment Guide §10.2)
qm list; pct list                         # confirm the guests you expected actually started
```

On **LUKS**, the local keyfile normally unlocks storage at boot. If `secure` is inactive,
`/mnt/secure` is not mounted, and guest startup reports `Dependency failed`, correct the unlock failure
(LUKS §12.3), then start the service. `systemctl reset-failed` is not required:

```bash
systemctl start luks-unlock.service
mountpoint /mnt/secure
findmnt -n -o SOURCE /mnt/secure          # -> /dev/mapper/cryptdata
pvesh create /nodes/$(hostname)/startall  # starts only guests with onboot=1
qm list; pct list                         # confirm the guests you expected actually started
```

**Important:** `startall` can return a task ID and exit 0 without starting guests: it covers only
`onboot=1`. Start others with `qm start` / `pct start`, or set `qm set <vmid> --onboot 1`.

### 7.2 Quorum loss on a two-node cluster

Recover with `pvecm expected 1` per Proxmox's
[Cluster Manager](https://pve.proxmox.com/pve-docs/chapter-pvecm.html) procedure. A survivor that has
just rebooted may also hold locked storage and be unable to start guests until it is unlocked (§7.1).

### 7.3 Failing over to a replica

**Note:** Applies to ZFS only; it depends on the replication in §5.

Follow the replication chapter's
[migration in case of error](https://pve.proxmox.com/pve-docs/chapter-pvesr.html) procedure with the
FIPS-specific steps below.

**Caution:** This is a data-loss operation. Confirm before forcing quorum:

- **The failed node is powered off or fenced, not partitioned.** A two-node cluster has no automatic
  fencing; starting the guest while the other copy still runs causes a split brain.
- **The replica's currency is acceptable.** The guest boots from the last completed snapshot.
- **The survivor's storage is unlocked** (§9.4).

```bash
ping -c1 -W1 <failed-node> && { echo "ABORT: the node still answers"; exit 1; }
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs get -H -o value keystatus <pool>/<dataset>      # must be available
# Restore quorum before checking replication or moving the config; pmxcfs is read-only without it:
pvecm expected 1
pvesr status                                          # header only: the failed node owns the job
# Read the replica snapshot's timestamp to determine the recovery point:
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs list -H -t snapshot -o name,creation -r <pool>/<dataset> | grep __replicate_<vmid>-
mv /etc/pve/nodes/<failed-node>/qemu-server/<vmid>.conf /etc/pve/nodes/$(hostname)/qemu-server/<vmid>.conf
qm start <vmid>
#   `cluster not ready - no quorum?` here -> §9.6; wait a few seconds and re-run.
```

Once the failed node returns, delete the stale replication job without `--force` (§5), then migrate
the guest home. Deletion needs the peer reachable; otherwise `remove_job full` retries indefinitely
with an increasing `FailCount`.

Wait for background removal to finish, normally within a few minutes of the node returning. Until
then, migration reports `refusing to migrate replicated VM whose replication job is marked for removal`.

### 7.4 Cold start

Stop guests before node shutdown, or ensure they respond to ACPI shutdown. Unresponsive guests,
including the deployment verification VMs, make `pve-guests.service` wait for the shutdown timeout.

After boot, quorum and `crypto_model` are restored automatically. Confirm provider routing:

```bash
grep -q libwolfprov /proc/$(pidof corosync)/maps && echo "corosync still routed"
```

LUKS and ZFS keyfile nodes unlock and start `onboot=1` guests automatically. ZFS passphrase nodes
require manual unlock before guest startup, migration, or restore (§7.1, §9.4; key custody: ZFS §10).

### 7.5 Removing and rejoining a node

Use Proxmox's
[node removal](https://pve.proxmox.com/pve-docs/chapter-pvecm.html#pvecm_remove_node) and
[Separate a Node Without Reinstalling](https://pve.proxmox.com/pve-docs/chapter-pvecm.html#pvecm_separate_node_without_reinstall)
procedures with the two-node adjustment below. Separate and rejoin in place to preserve the FIPS
deployment.

**Caution:** Removing a running node from a two-node cluster loses quorum mid-operation. The
`corosync.conf` update fails and leaves a stale lock (§9.9). Corosync also fails on the targeted node;
restart it there after restoring quorum.

**Power the node off first.** While both nodes are active, `pvecm expected 1` is refused with
`CS_ERR_INVALID_PARAM`. Lower the expectation only after shutdown:

```bash
# 1. Empty the node before removal; deleting guests after separation leaves stale cluster configs
#    that a rejoin restores without their disks:
qm list; pct list                                 # both empty
# 2. shut that node down and confirm it is gone:
shutdown -h now                                   # on the node being removed
ping -c1 -W1 <node-to-remove> && echo "still up — wait"   # from the survivor
# 3. On the survivor, restore quorum and wait for the PVE layer before removing the node:
pvecm expected 1
sleep 10
pvecm delnode <node-to-remove>
grep -E '^\s+name: |config_version' /etc/pve/corosync.conf   # only the surviving node should remain
# 4. Remove the departed node's directory; otherwise rejoining can restore stale guest configs:
ls /etc/pve/nodes/
rm -rf /etc/pve/nodes/<node-to-remove>
```

`Could not kill node` is harmless (§9.7). For `cluster not ready`, wait and retry; removal has not
completed (§9.6).

Keep the departed node's key in `/etc/pve/priv/authorized_keys` for rejoining. For permanent removal,
delete its line; `pvecm delnode` does not remove it.

**Caution:** Have console or password access before separation. Stopping `pve-cluster` unmounts
`/etc/pve`, breaking the `/root/.ssh/authorized_keys` symlink and key-based root logins until the
service starts again.

Power the removed node on. Its stale two-node configuration leaves it inquorate with `/etc/pve`
read-only. Enter local mode before cleanup to avoid `Permission denied`:

```bash
# on the separated node
systemctl stop pve-cluster
systemctl stop corosync
pmxcfs -l                                         # local mode, so /etc/pve is writable
rm -f /etc/pve/corosync.conf
rm -rf /etc/corosync/*
rm -f /var/lib/corosync/*
killall pmxcfs
systemctl start pve-cluster
```

`forcing local mode (although corosync.conf exists)` is expected from `pmxcfs -l`.

Clear the other node's guest configs from the local pmxcfs copy; they otherwise block rejoining (§9.8):

```bash
# on the separated node, BEFORE `pvecm add`:
qm list; pct list                                 # both empty — this node genuinely holds no guests
find /etc/pve/nodes -name '*.conf'                # but the OTHER node's configs are still here
rm -rf /etc/pve/nodes/<other-node>
find /etc/pve/nodes -name '*.conf'                # must now print nothing
```

**Caution:** A refused `pvecm add` can exit 0. Confirm the current `Config Version` and `Quorate: Yes`
with `pvecm status` on the rejoining node.

**Important:** Re-apply the node-local drop-in (§3.3) before rejoining, then repeat §3.4's process-map
check and `chmod 400 /etc/corosync/authkey`. Inheriting `crypto_model` without the drop-in silently
uses stock OpenSSL.

---

## 8. Scope and limitations

### 8.1 Cluster PKI runs on stock OpenSSL

PVE generates CA and node certificates using SHA-256 with RSA on stock OpenSSL. Kernel `fips=1` does
not route those operations through wolfProvider.

### 8.2 Out of scope

| Item | Why |
|---|---|
| QDevice / qnetd | Quorum-vote TLS uses NSS, not wolfProvider. Use a third node for a tiebreaker. |
| HA manager | Requires three or more nodes and shared storage; no additional crypto beyond pmxcfs, corosync, and the local watchdog. |
| More than two nodes | Same routing configuration; each additional node needs its own drop-in and check (§3.3). |
| Redundant corosync links | All links use the cluster-wide `crypto_model`. |
| Cross-cluster `remote-migrate` | Uses API-token and websocket TLS paths, not intra-cluster SSH. |
| `migration: insecure` | Sends guest RAM in plaintext (§4.1). |
| Certificate expiry and rotation at scale | `pvecm updatecerts` uses stock OpenSSL (§8.1). |
| LUKS replication and replica failover | `pvesr` requires ZFS (§5, §7.3). |
| Shared encrypted storage | These ZFS pools and LUKS mappings are single-host storage. Use the Ceph deployment guide for shared storage. |

---

## 9. Troubleshooting

### 9.1 An inherited backup storage reports `error fetching datastores`

After a join, a per-node Proxmox Backup Server storage inherited from the first node reports
`error fetching datastores` on the joining node: its address resolves to the wrong host from there.
Redefine the storage cluster-wide as described in §6.

### 9.2 A bare `ssh` between nodes fails host-key verification

Use §3.5's `HostKeyAlias` and `UserKnownHostsFile` options for PVE's node-to-node trust. Bare
`ssh <other-node>` does not use them. A stale `/root/.ssh/known_hosts` entry can instead report
`REMOTE HOST IDENTIFICATION HAS CHANGED`, possibly naming a pre-deployment ED25519 key; remove the
stale entry.

### 9.3 `qm migrate` refuses: `target is local node`

A return leg issued on the origin node is refused with
`400 Parameter verification failed. target: target is local node.` Each leg runs on the node that
currently owns the guest; issue the return leg on the target. `pct migrate` behaves the same.

### 9.4 A migration, restore, or replication onto a locked target

On **ZFS**, migration or restore to a locked encryption root fails with exit status 255. Look for
the initial error:

```
cannot receive new filesystem stream: inherited key must be loaded
cannot open '<pool>/<dataset>/vm-<vmid>-disk-0': dataset does not exist
command 'zfs recv -F -x encryption -- <pool>/<dataset>/vm-<vmid>-disk-0' failed: exit code 1
```

The unloaded key is the cause; missing-dataset errors follow because receive failed. The final
`migration aborted` line omits the cause. A container migration may also report
`ERROR: found stale volume copy '<storage>:subvol-<vmid>-disk-0' on node '<target>'` even though the
destination was never created.

Live migration instead reports `zfs error: cannot create '<pool>/<dataset>/vm-<vmid>-disk-0': encryption
root's key is not loaded or provided`, then `ERROR: online migrate failure`. Unlock the target:

```bash
# on the target node:
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  zfs get -H -o value keystatus <pool>/<dataset>        # unavailable = locked, this is the cause
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs load-key <pool>/<dataset>
mountpoint -q /<pool>/<dataset> || \
  OPENSSL_CONF=/etc/wolfprovider/openssl.cnf zfs mount <pool>/<dataset>
```

A passphrase-model node comes up locked by design and cannot receive migrations or restores until an
operator unlocks it; a keyfile-model node unlocks at boot (ZFS Deployment Guide §10).

On **LUKS**, the outcome depends on the operation. An online migration, a `qmrestore`, and a migration
*out of* a locked node are refused with exit status 255, naming the storage:

```
unable to activate storage 'secure' - directory is expected to be a mount point but is not mounted: '/mnt/secure'
```

Offline `qm migrate` or `pct migrate` instead writes to `/mnt/secure` on the unencrypted root, reports
`migration finished successfully`, exits 0, and removes the source. Only the plaintext copy remains;
unlocking hides it beneath the mount while the guest configuration still references it.

Confirm which filesystem holds the disk, and recover the guest before starting it:

```bash
# on the target node — /dev/mapper/pve-root here means the disk is on the unencrypted root:
findmnt -n -o SOURCE -T /mnt/secure/images/<vmid>/vm-<vmid>-disk-0.raw
# the mount would otherwise hide the copy:
systemctl stop luks-unlock.service
mkdir -p /root/luks-residue
mv /mnt/secure/images/<vmid> /root/luks-residue/
systemctl start luks-unlock.service
mv /root/luks-residue/<vmid> /mnt/secure/images/
findmnt -n -o SOURCE -T /mnt/secure/images/<vmid>/vm-<vmid>-disk-0.raw   # -> /dev/mapper/cryptdata
rmdir /root/luks-residue
```

**Caution:** The second move copies data to the encrypted volume and unlinks the plaintext original.
Unlinking does not erase its blocks. Treat the root filesystem as having held that data.

### 9.5 The map count reads `0` on a guest being backed up

If the guest started before the Node Configuration and Recovery guide §2 wrapper was installed and is
still running unscoped, stop and restart or migrate it, then repeat the check. If its environment
contains `OPENSSL_CONF`, the provider may load only after a small backup reports completion; repeat
the map count after the backup.

### 9.6 `cluster not ready - no quorum?`

After `pvecm expected`, the PVE layer can lag corosync by a few seconds even while `pvecm status`
reports `Quorate: Yes`. Wait, then retry. A refused `pvecm delnode` has not removed the node:
`corosync.conf` still lists it and `config_version` is unchanged.

### 9.7 `Could not kill node (error = CS_ERR_NOT_EXIST)`

This message is expected and harmless when `pvecm delnode` runs against a node that is already
powered off.

### 9.8 The rejoin is refused, naming guests this node does not hold

Without the §7.5 separation and cleanup, `pvecm add` on the rejoining node is refused:

```
detected the following error(s):
* authentication key '/etc/corosync/authkey' already exists
* cluster config '/etc/pve/corosync.conf' already exists
* this host already contains virtual guests
* corosync is already running, is this node already in a cluster?!
Check if node may join a cluster failed!
```

§7.5's separation clears the key, cluster config, and running-corosync errors. Removing the other
node's local config copy clears the guest error; the precheck counts those configs even when
`qm list` and `pct list` are empty.

### 9.9 A `pvecm delnode` retry is blocked by a stale lock

Removing a running node can leave a stale cfs lock. Restore quorum before removing it; otherwise
pmxcfs is read-only and removal fails with `Permission denied`:

```bash
pvecm expected 1
rmdir /etc/pve/priv/lock/file-corosync_conf
```
