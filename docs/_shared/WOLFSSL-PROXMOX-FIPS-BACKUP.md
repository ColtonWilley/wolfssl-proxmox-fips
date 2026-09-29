# wolfSSL FIPS on Proxmox VE — Backup

Configure and verify backups after completing your storage deployment and Node Configuration and
Recovery guides. For guests and archives from a previous deployment, use the migration guides.

---

## 1. Backup encryption

For standard `vzdump` procedures, see the [Backup and Restore](https://pve.proxmox.com/pve-docs/chapter-vzdump.html)
chapter of the Proxmox VE Administration Guide.

| Path | Archive encryption |
| --- | --- |
| `vzdump` to file-level storage | Depends on the target storage. `vzdump` performs no cryptography of its own. |
| Proxmox Backup Server (PBS) | Depends on the datastore's storage, plus client-side chunk encryption when a key is configured (§3). |

**Important:** An encrypted source does not protect an archive written to an unencrypted target.
Backup output does not warn about this loss of at-rest protection.

---

## 2. Verify the target and restore

The deployment guides do not create backup storage. Check the target's encryption separately from
the source storage.

**Caution:** `local` is unencrypted and accepts backups unless excluded by its content types. It may
be the only target offered by `vzdump` or the web interface. The ZFS/LUKS guides restrict it to
`vztmpl,iso`; the Ceph guide does not. Check available backup storages and each job's target before
use, not only when creating the job.

```bash
pvesm status --content backup                  # -> the storages that accept backups
```

Back up and restore a guest, then verify a file checksum. Use the deployment guide's verification
container or a guest with an operating system and data. The guide's verification VM has an empty disk.

**Inside the source guest**, record a representative file's checksum:

```bash
#   sha256sum <a representative file> > /root/pre-backup.sha256
```

**On the Proxmox node**, back up the guest and identify the newest archive:

```bash
vzdump <vmid> --storage <target> --mode snapshot
pvesm list <target> --vmid <vmid> | awk '/backup/{print $1}' | sort | tail -1    # -> the newest volid
```

Restore with the commands for the guest type below.

**Virtual machine:**

```bash
qmrestore "<volid>" <new-vmid> --storage <guest-storage>
qm start <new-vmid> && sleep 5 && qm status <new-vmid>     # -> status: running
```

**Container:** Use `pct restore`, not `qmrestore`; the wrong command leaves a VM configuration
stub (§4).

```bash
pct restore <new-ctid> "<volid>" --storage <guest-storage>
pct start <new-ctid> && sleep 5 && pct status <new-ctid>   # -> status: running
```

**Inside the restored guest**, verify the checksum:

```bash
#   sha256sum -c /root/pre-backup.sha256       # -> OK
```

**On the Proxmox node**, remove only the test guest you restored, using the matching command:

```bash
qm stop <new-vmid> && qm destroy <new-vmid> --purge        # VM test guest
pct stop <new-ctid> && pct destroy <new-ctid> --purge      # container test guest
```

---

## 3. Proxmox Backup Server

For Proxmox Backup Server installation and datastore administration, see the
[Proxmox Backup Server documentation](https://pbs.proxmox.com/docs/).

With an encryption key configured, the client encrypts chunks with AES-256-GCM before sending them
from the Proxmox VE node.

**Important:** Complete Node Configuration and Recovery guide §2 on every node before adding a
PBS target.

### 3.1 Configure the backup client

Container and host backups use `/usr/bin/proxmox-backup-client`. Install a wrapper on **every node**
to select wolfProvider regardless of the parent process's environment:

```bash
# Save the packaged binary only if it has not already been wrapped:
if [ "$(head -c2 /usr/bin/proxmox-backup-client)" != '#!' ]; then
  cp -a /usr/bin/proxmox-backup-client /usr/bin/proxmox-backup-client.real
fi
cat > /usr/bin/proxmox-backup-client <<'EOF'
#!/bin/bash
export OPENSSL_CONF=/etc/wolfprovider/openssl.cnf
exec -a "$0" /usr/bin/proxmox-backup-client.real "$@"
EOF
chmod +x /usr/bin/proxmox-backup-client
```

**Important:** Unpacking `proxmox-backup-client` replaces the wrapper. Hold it and the other packages
used for backup cryptography:

```bash
apt-mark hold proxmox-backup-client proxmox-backup-file-restore libproxmox-backup-qemu0
```

**Caution:** Holds do not prevent reinstallation. `apt-get install --reinstall` replaces the wrapper
and clears the hold without warning. After package operations affecting these three packages,
reapply the wrapper and hold, then run §3.3.

For single-file restore, repeat the wrapper procedure with `proxmox-file-restore` in place of
`proxmox-backup-client`. It connects directly to PBS and requires `PBS_FINGERPRINT` to be exported;
otherwise it fails with `certificate validation failed`. Verify the wrapper and run the binary:

```bash
head -c2 /usr/bin/proxmox-file-restore; echo             # -> #!
grep -q wolfprovider /usr/bin/proxmox-file-restore && echo "OK: file-restore scoped"
# Run it to detect a missing wrapper target. Use help; this binary has no version subcommand.
proxmox-file-restore help >/dev/null && echo "OK: file-restore runs"
```

**Important:** The backup-client wrapper covers containers and hosts, not VMs. VM backups use
`libproxmox-backup-qemu` in QEMU; VM restores use `/usr/bin/pbs-restore`. Apply and verify their
wrappers in Node Configuration and Recovery guide §2.

### 3.2 Encryption keys

Direct host/container backups with `proxmox-backup-client` use a key file that may have a passphrase.
Guest backups and scheduled jobs use the key in their Proxmox VE storage configuration. These key
models are not interchangeable.

Create passphrase-protected key files with `--kdf pbkdf2`:

```bash
proxmox-backup-client key create <keyfile> --kdf pbkdf2
```

**Important:** wolfProvider does not support the default KDF, scrypt. Creating or opening a scrypt
key fails without fallback, including during restore.

For Proxmox VE storage, generate the key when adding the datastore:

```bash
pvesm add pbs <id> --server <host> --datastore <datastore> \
  --username <user>@<realm> --password <password> \
  --fingerprint <fingerprint> --encryption-key autogen
```

**Important:** Storage keys must not have a passphrase. Unattended jobs cannot prompt for one and
fail without writing a backup (§4).

**Warning:** Copy the [client-side key](https://pbs.proxmox.com/docs/backup-client.html) outside the
cluster immediately. Losing it makes every backup encrypted with it unrestorable.

**Important:** When recovering with a
[master key](https://pbs.proxmox.com/docs/backup-client.html#using-a-master-key-to-store-and-recover-encryption-keys),
pass `--kdf pbkdf2` to `key import-with-master-key`. It encrypts the recovered key under a new
passphrase and otherwise defaults to scrypt, regardless of the original key's KDF.

### 3.3 Verify the client

```bash
# Check the wrapper:
head -c2 /usr/bin/proxmox-backup-client; echo  # -> #!
grep -q wolfprovider /usr/bin/proxmox-backup-client && echo "OK: client scoped"
# Run it to confirm the wrapper's target exists and works:
proxmox-backup-client version                  # -> client version: <version>
# Confirm all three package holds:
apt-mark showhold | grep -c 'proxmox-backup-client\|proxmox-backup-file-restore\|libproxmox-backup-qemu0'   # -> 3
```

Then run the §2 round trip against the Proxmox Backup Server storage.

---

## 4. Troubleshooting

**`vzdump --mode stop` fails with `VM quit/powerdown failed - got timeout`.** A guest without an OS,
including the deployment guide's verification VM, cannot acknowledge the ACPI shutdown. The backup
times out after ten minutes without writing an archive. Use `--mode snapshot`.

**`vzdump --mode stop` is refused with `Cannot execute a backup with stop mode on a HA managed and
enabled Service`.** The Ceph deployment places its guests under high availability. Back them up with
`--mode snapshot`.

**`qmrestore` refuses a container archive with `got unexpected backup format 'pbs-ct'`.** It leaves a
VM configuration that blocks `pct restore` with `VM <new-ctid> already exists`. Remove that stub with
`qm destroy <new-ctid> --purge`, then use `pct restore`.

**An unattended backup cannot load the storage key.** A VM reports
`failed to load decryption key from "/etc/pve/priv/storage/<id>.enc"`; a container reports `no password
input mechanism available`. The key has a passphrase, so both jobs fail without writing a backup.
Replace it with a key without a passphrase (`--encryption-key autogen`, §3.2).

**`proxmox-backup-client benchmark` reports a scrypt error.** The benchmark tests scrypt, which
wolfProvider does not support.
