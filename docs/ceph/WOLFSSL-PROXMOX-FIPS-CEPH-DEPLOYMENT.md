# wolfSSL FIPS for Ceph on Proxmox VE — Deployment

Deploy a new Ceph Tentacle cluster on freshly prepared Proxmox VE nodes. This guide covers the
wolfSSL package selection, encrypted OSDs and encrypted Ceph connections. Use Proxmox's
[Ceph administration chapter](https://pve.proxmox.com/pve-docs/chapter-pveceph.html) for topology,
capacity, pools, monitors and routine administration.

Run commands as root. Replace angle-bracket placeholders with the site's devices, storage IDs
and network values. Read [Cluster planning](../storage-cluster/WOLFSSL-PROXMOX-FIPS-CLUSTER.md#1-plan-the-cluster)
before preparing the first node.

## 1. Prepare the Ceph packages

Use fresh Proxmox VE 9 nodes, with separate OSD disks and no existing Ceph cluster. Install
Proxmox's Tentacle packages on each node **before** adding the wolfSSL repository or installing
the Foundation packages. The following selects the no-subscription repository; use
`enterprise` instead when the nodes have the required subscriptions.

```sh
pveceph install --repository no-subscription --version tentacle
ceph --version
```

Require Tentacle 20.2.4 for this new deployment. Do not initialize Ceph yet. Squid is not part
of this wolfSSL Ceph deployment. Do not run `pveceph install` again after configuring the
wolfSSL repository; select the Ceph metapackage during Foundation instead.

The Ceph and ZFS metapackages conflict. Select Ceph for these nodes; encrypted ZFS on the same
node is outside this guide's wolfCrypt storage coverage. Separate local LUKS volumes are
supported with the additional [LUKS metapackage and configuration](../storage-cluster/WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-LUKS.md).

## 2. Install and activate the wolfSSL integration

Complete [Foundation](../_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md) on every node. Select
`Components: main ceph-tentacle` in the wolfSSL repository and `wolfssl-fips-pve-ceph` in
Foundation's package installation. Include `wolfssl-fips-pve-luks` in that transaction if the
node also needs separate local LUKS volumes. Complete SSH preparation, signing-key enrollment
where required, reboot and verification before continuing.

Use the same channel on all nodes. FIPS Ready is not a FIPS-validated module; the licensed
channel's applicability remains subject to the module Security Policy as described in Foundation.
Check the installed Ceph packages on every node:

```sh
dpkg-query -W ceph-common ceph-mon ceph-mgr ceph-osd ceph-volume wolfssl-fips-pve-ceph
```

Require the wolfSSL Ceph builds (`~wolfcryptfips` in the package versions). The metapackage
supplies the Ceph service environment and compatible cryptsetup/ceph-volume settings. Open a
new login session so that command-line Ceph clients receive the packaged environment.

Before any Ceph daemon exists, the verifier's Ceph-daemon check can fail because no daemon is
running. Corosync is also not yet running. Resolve other applicable failures before continuing;
repeat full verification after cluster creation.

## 3. Create the cluster and encrypted OSDs

First create and verify the new Proxmox cluster using [Cluster, section 2](../storage-cluster/WOLFSSL-PROXMOX-FIPS-CLUSTER.md#2-create-and-verify-the-cluster).
All nodes must have completed sections 1–2 above. Do not initialize Ceph before that cluster
has quorum and its corosync verification passes.

On the first node, initialize Ceph with the planned Ceph network:

```sh
pveceph init --network <ceph-network-cidr>
pveceph mon create
```

The first monitor also creates the first manager; do not create a second manager on that node.
Set the following once before creating OSDs. The temporary `secure crc` setting allows the
initial daemons to establish connections while secure mode is configured; section 5 removes
CRC mode before production use.

```sh
for setting in ms_cluster_mode ms_service_mode ms_client_mode ms_mon_cluster_mode ms_mon_service_mode ms_mon_client_mode; do
    ceph config set global "$setting" 'secure crc'
done
ceph config set global ms_bind_msgr2 true
ceph config set global plugin_crypto_accelerator crypto_openssl
ceph config set global rbd_default_map_options ms_mode=secure
```

Create the additional [monitors](https://pve.proxmox.com/pve-docs/chapter-pveceph.html#pve_ceph_monitors)
and [managers](https://pve.proxmox.com/pve-docs/chapter-pveceph.html#pve_ceph_manager)
required by the planned topology.
Check `ceph mon stat` for the intended monitor count and quorum membership.

**Warning:** OSD creation consumes the selected disk. Confirm that `<osd-disk>` is a dedicated,
empty data disk before running this command on each OSD host:

```sh
pveceph osd create <osd-disk> --encrypted 1
ceph-volume lvm list
```

The packaged ceph-volume selects AES-256-CBC with ESSIV-SHA256 and PBKDF2-SHA256 instead of the
stock encryption defaults. It also selects compatible keyslot parameters. Inspect each new
OSD's block device, and its separate DB/WAL devices if configured; obtain their paths from
`ceph-volume lvm list`:

```sh
cryptsetup luksDump <osd-luks-device>
```

Require `aes-cbc-essiv:sha256`, a 256-bit key and PBKDF2 with SHA-256 for the applicable data
and keyslot metadata. No slot may use Argon2 or exceed the licensed module's 10,000,000-iteration
ceiling. Section 5's verifier checks these storage properties as well.

OSD unlock keys are held in the Ceph monitor store and made available during OSD activation;
this is automatic cluster-managed unlocking, not an operator passphrase prompt at each boot.
Protect the monitor store, Ceph credentials and their recovery copies. Encrypted OSDs do not
encrypt the Proxmox root disk, monitor metadata, node-local backup files or every storage on
the node. Follow [Backup](../_shared/WOLFSSL-PROXMOX-FIPS-BACKUP.md) for an independent backup
and tested restore with retained encryption keys.

A cluster created on this Ceph 20.2.4 build has AES-256 cephx keys from creation; no cephx key
migration is required.

## 4. Configure Ceph storage clients

Create pools and register RBD storage using the
[Proxmox Ceph instructions](https://pve.proxmox.com/pve-docs/chapter-pveceph.html).
All OSDs backing those pools must be encrypted. Encryption does not follow a guest moved to
another, unencrypted storage.

For CephFS, create the filesystem without automatic Proxmox storage registration. Replace
`<cephfs-name>` with the filesystem name and select pool settings using the Proxmox instructions:

```sh
pveceph fs create --name <cephfs-name> --add-storage 0
```

Complete the applicable client configuration below before registering CephFS storage.

The supplied kernel supports encrypted kernel Ceph clients, including `krbd 1`, containers on
RBD, VM TPM-state volumes on RBD, and kernel CephFS mounts. RBD with `krbd 0` uses QEMU's
userspace client through the Foundation wrapper. A TPM-state volume on encrypted RBD is
protected storage; this does not put vTPM cryptographic operations inside wolfCrypt's validated
boundary. Choose the normal Proxmox RBD client setting for the workload.

### 4.1 Kernel CephFS

Before registering a kernel CephFS storage, set every `mon_host` address in the `[global]`
section of `/etc/pve/ceph.conf` to its msgr2 endpoint, including port 3300. Obtain the endpoints
from the monitor map:

```sh
ceph mon dump
nano /etc/pve/ceph.conf
```

For example, use `<monitor-address>:3300`, or `[<ipv6-address>]:3300`. Proxmox shares this file
across the cluster. Register the storage with `ms_mode=secure` in the initial command, because
registration mounts it immediately. Replace `<cephfs-storage>` with the Proxmox storage ID:

```sh
pvesm add cephfs <cephfs-storage> --fs-name <cephfs-name> \
  --content backup,iso,vztmpl,snippets --options ms_mode=secure
```

Include any other required mount options in the same comma-separated option list. Reboot each
node in turn before putting data on these mounts, then verify the mounts as described in section 5.
After adding a monitor later, add port 3300 to its new `mon_host` entry as well.

### 4.2 FUSE CephFS

For a CephFS storage configured with `fuse 1`, install this mount-unit environment on every
client node before its first mount:

```sh
mkdir -p /etc/systemd/system/mnt-pve-.mount.d
cat > /etc/systemd/system/mnt-pve-.mount.d/10-wolfprov.conf <<'UNIT'
[Mount]
Environment=OPENSSL_CONF=/etc/wolfprovider/openssl.cnf
UNIT
systemctl daemon-reload
```

If the storage is already mounted, reboot the nodes one at a time to activate the setting.
The Ceph metapackage does not configure these Proxmox-created mount units.

## 5. Require secure connections and verify

On each node with kernel Ceph clients, inspect the active mounts and RBD maps:

```sh
findmnt -t ceph -o TARGET,OPTIONS
grep -H . /sys/bus/rbd/devices/*/config_info
```

Every configured kernel CephFS mount must appear with `ms_mode=secure`; every active RBD map
must also show `ms_mode=secure`. No RBD map is expected until a workload maps an RBD volume;
a missing glob is not evidence of a successful client connection.

Once all nodes and clients are configured, make secure mode mandatory and disable msgr1:

```sh
for setting in ms_cluster_mode ms_service_mode ms_client_mode ms_mon_cluster_mode ms_mon_service_mode ms_mon_client_mode; do
    ceph config set global "$setting" secure
done
ceph config set global ms_bind_msgr1 false
```

Restart Ceph on one node at a time, waiting for monitor quorum and healthy placement groups
before the next node:

```sh
systemctl restart ceph.target
ceph -s
ceph mon stat
```

Finally, run on every node:

```sh
wolfssl-fips-pve-verify --ceph-cluster
ceph -s
ceph mon stat
```

Resolve every failure and review warnings, including plaintext `local` or `local-lvm` storage
eligibility. Require all intended OSDs `up` and `in`, all placement groups `active+clean`, and
all intended monitors in quorum. The verifier checks provider activation, messenger policy,
OSD encryption and relevant kernel clients; it does not prove every guest volume or backup
uses encrypted storage. Check those destinations before creating or importing guests.

## 6. Maintain the cluster

Use [Node maintenance](../_shared/WOLFSSL-PROXMOX-FIPS-NODE-CONFIGURATION.md) for APT updates
and access recovery, and [Cluster maintenance](../storage-cluster/WOLFSSL-PROXMOX-FIPS-CLUSTER.md#5-maintain-and-restart-the-cluster)
for serial reboots. Installing an update does not activate every running Ceph daemon; complete
the required reboot and re-run section 5 on each node. Wait for clean placement groups before
proceeding to the next node.

For a replacement OSD, follow Proxmox's
[OSD replacement procedure](https://pve.proxmox.com/pve-docs/chapter-pveceph.html#pve_ceph_osd_replace), create the
replacement with `--encrypted 1`, and repeat the header and cluster checks in sections 3 and 5.
For an added node, repeat sections 1–2 before joining it through the Cluster guide. Create its
Ceph services and encrypted OSDs, update kernel CephFS monitor addresses if applicable, and
verify it before assigning guests.

If a monitor is missing from `ceph mon stat`, inspect its service and monitor-map membership
before continuing. If a CephFS mount fails after disabling msgr1, check port 3300 in
`mon_host` and `ms_mode=secure` on that storage before retrying. Do not restore CRC mode as a
permanent workaround.
