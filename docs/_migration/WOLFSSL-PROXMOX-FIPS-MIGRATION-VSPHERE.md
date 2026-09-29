# wolfSSL FIPS on Proxmox VE — Migration from VMware vSphere

Use this guide to import guests from VMware vSphere to encrypted storage configured by the FIPS
deployment guides. It uses Proxmox's ESXi import integration and follows the
[Migrate to Proxmox VE](https://pve.proxmox.com/wiki/Migrate_to_Proxmox_VE) procedure. For backups and
restores of guests already on the FIPS deployment, use the Backup guide.

---

## 1. Before you begin

### 1.1 What you need

- A FIPS destination with capacity for the guests. Imported disks retain their virtual size.
- Network access from the Proxmox node to each source ESXi host's API.
- An ESXi account with privileges to enumerate the inventory and read the datastores.
- A reachable key provider for encrypted sources (§3.5).
- An inventory of each source VM's disks, firmware, and guest OS.
- Source disks on non-vSAN storage. [vSAN-backed disks cannot be imported](https://pve.proxmox.com/wiki/Migrate_to_Proxmox_VE);
  move them to another datastore first.

### 1.2 Conventions

- Commands run as `root` on the Proxmox node unless noted.
- `<fips-storage>` is the encrypted Proxmox storage built by your deployment guide.
- `<esxi-src>` is the ESXi source registered as a Proxmox storage in §3.6.
- `<bridge>` is the network bridge on the destination node that carries the guest network.

---

## 2. How the import works

Prepare encrypted sources in vCenter (§3.5). Register each individual ESXi host with Proxmox;
importing through vCenter is slower.

Power off the source VM for the import. The import leaves the source unchanged; retain it for rollback
until the imported guest is verified.

---

## 3. Migrate one guest

Repeat this section for each guest; the source is powered off only for its own window.

### 3.1 Inventory the source

Record each VM's disks, firmware type, guest OS, and network configuration.

**Important:** check whether an encryption storage policy protects the source. If so, decrypt it before
importing (§3.5). The importer lists encrypted VMs without identifying their encryption; see §5.1 for
the resulting error.

### 3.2 Record the verification checksum

Inside the running source guest, record checksums for the Migration Acceptance guide:

```bash
# in the guest, as root:
date -Ins > /root/pre-migration.stamp
sha256sum /root/pre-migration.stamp <a representative data file> > /root/pre-migration.sha256
sync
```

Use absolute paths and include a file unique to this migration, such as the stamp above, to help
detect an import from an older disk.

### 3.3 Remove VMware Tools

Remove VMware Tools while the guest is still on VMware, following the instructions for its OS in
[Migrate to Proxmox VE](https://pve.proxmox.com/wiki/Migrate_to_Proxmox_VE).

### 3.4 Power the source off

Shut the guest down from inside — VMware Tools is already removed — and confirm it is off.

### 3.5 Decrypt an encrypted source

Skip this section if §3.1 found the source unencrypted.

Remove the encryption storage policy using your standard vSphere procedure:

- Remove the policy from the **VM home and every disk**. Decrypting only the disks permits import but
  leaves the VM home associated with the key provider.
- The source must be powered off, and the key provider that holds its keys reachable.
- Decryption renames disk files. Use the updated paths from the manifest in §3.6.

Register or refresh the ESXi source (§3.6) after the policy change.

### 3.6 Register or refresh the ESXi source

Register the source after shutdown and any decryption. Registration caches the disk filenames and
power state in a manifest:

```bash
pvesm add esxi <esxi-src> --server <ESXi-HOST> --username <ADMIN-USER> --password <PASSWORD> \
      --skip-cert-verification 1
pvesm status | grep <esxi-src>          # -> <esxi-src>  esxi  active
```

**Note:** on first registration, `pvesm add` returns before the manifest is ready. The status check may
block for a few minutes or print nothing; repeat it until the storage is listed. Removing and re-adding
the storage restarts the cache build.

If the source was already registered before §3.4 or §3.5, refresh it:

```bash
pvesm remove <esxi-src>
pvesm add esxi <esxi-src> --server <ESXi-HOST> --username <ADMIN-USER> --password <PASSWORD> \
      --skip-cert-verification 1
```

Read the disk path and power state from the manifest, and confirm the source is off before importing:

```bash
cat /run/pve/import/esxi/<esxi-src>/manifest.json    # -> "power": "poweredOff", and each disk path
find /run/pve/import/esxi/<esxi-src>/mnt -maxdepth 4
#   -> /run/pve/import/esxi/<esxi-src>/mnt/ha-datacenter/<datastore>/<vm>/<disk>.vmdk
```

**Important:** use each disk's path from the manifest. Filenames can differ from the VM name after
consolidation or decryption, or for multi-disk guests. An older disk can import successfully without
warning. For `Input/output error`, see §5.2.

### 3.7 Import onto the FIPS storage

Unlock the destination storage per its deployment guide before importing. For locked-storage errors,
see §5.3.

On a **Ceph destination**, import from the command line with `OPENSSL_CONF` as shown below to use
wolfProvider for the Ceph connection. Web-interface imports do not inherit the provider configuration.

**Important:** do not configure `pvedaemon` to use wolfProvider. This breaks web-interface and API
logins; see Node Configuration and Recovery guide §1.

**On a Ceph destination only**, check wolfProvider from a second shell while the import is transferring:

```bash
grep -c libwolfprov /proc/$(pgrep -x qemu-img | head -1)/maps     # -> non-zero
```

A `0` during transfer indicates a failure. For concurrent disk imports, check each `qemu-img` process.

Create the guest and import each disk onto the FIPS storage, using the manifest paths from §3.6:

```bash
pvesh get /cluster/nextid                 # a free <vmid>, if you do not already have one assigned
qm create <vmid> --name <name> --memory <MB> --cores <n> --scsihw virtio-scsi-single \
      --net0 virtio,bridge=<bridge> --bios <seabios|ovmf>
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  qm set <vmid> --scsi0 <fips-storage>:0,import-from=<esxi-src>:ha-datacenter/<datastore>/<vm>/<disk>.vmdk
qm set <vmid> --boot order=scsi0
qm set <vmid> --onboot 1
qm config <vmid>                        # -> every disk on <fips-storage>; a boot entry is set
```

For each additional disk the source carries, repeat the import line with the next controller index:

```bash
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf \
  qm set <vmid> --scsi1 <fips-storage>:0,import-from=<esxi-src>:ha-datacenter/<datastore>/<vm>/<disk2>.vmdk
```

**Important:** keep both the boot-order and `onboot` commands above. `qm create` defaults to
`boot: order=net0`; the recovery procedure requires `onboot=1` to start the guest.

### 3.8 Adjust the imported guest

Follow [Migrate to Proxmox VE](https://pve.proxmox.com/wiki/Migrate_to_Proxmox_VE) for guest adjustments:
Windows VirtIO drivers, the OVMF boot entry, the QEMU guest agent, pinned interface names, and DHCP
reservations for the new MAC address.

---

## 4. Verify and retire

Continue in the [Migration Acceptance guide](WOLFSSL-PROXMOX-FIPS-MIGRATION-ACCEPTANCE.md) to verify
the guest's storage, data, and recovery after a node restart before retiring the source.

---

## 5. Troubleshooting

### 5.1 Import fails with `output file is smaller than input file`

The source is encrypted. No volume remains after the failed import, only the empty guest configuration.
Remove the encryption storage policy (§3.5), refresh the registration (§3.6), and re-import.

### 5.2 Import fails with `Could not read image for determining its format: Input/output error`

The manifest is stale — the disk was renamed, consolidated, or decrypted after registration — or the
source is still powered on. Confirm the source is off, refresh the registration (§3.6), and take the
path from the new manifest.

### 5.3 The destination storage is locked

An import onto locked ZFS storage fails with `encryption root's key is not loaded or provided` and
creates no partial volume. `qm start` blocks while holding the guest configuration lock; subsequent
commands may report `got timeout`. Do not force-kill the task. Unlock the storage per the deployment
guide; the blocked task completes once the key is loaded.
