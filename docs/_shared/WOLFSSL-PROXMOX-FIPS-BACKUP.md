# wolfSSL FIPS for Proxmox VE — Backup and Restore

Complete [Installation](../_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md) and the selected
storage guide on every backup and restore node first. Installation supplies the rebuilt PBS
clients and their wrappers. Commands run as root on the Proxmox node unless labeled otherwise.
For scheduling, retention, guest consistency and backup exclusions, use the official
[Proxmox Backup and Restore chapter](https://pve.proxmox.com/pve-docs/chapter-vzdump.html).

## 1. Choose the protection for backups

| Destination | Protection supplied by this deployment |
| --- | --- |
| File-level storage (`vzdump` VMA or tar archive) | Encryption of the destination filesystem or block device. The archive itself is not encrypted by `vzdump`. Copying it outside that storage removes this protection. |
| PBS with a client encryption key | Client-side AES-256-GCM encryption of backup data before upload. The rebuilt clients also route chunk SHA-256 operations through wolfProvider. |
| PBS without a client encryption key | No client-side encryption; any at-rest protection depends on the server's datastore storage. |

An encrypted source disk does not encrypt its backup automatically. PBS client encryption
does not cover all backup metadata or local staging. PBS server-side verification and the
server's underlying storage are separate from the Proxmox VE client configuration described here.

## 2. Protect the destination and temporary files

Check each job's target, any `dumpdir` override and the default settings:

```sh
pvesm status --content backup
pvesh get /cluster/backup --output-format json-pretty
cat /etc/vzdump.conf
```

Keep file archives on a destination whose encryption and unlock behavior have been verified
using the selected storage guide. Do not use the unencrypted `local` directory for backups.
The verifier does not inspect every job's `dumpdir`; such an override can bypass the selected
storage. Restrict storage content types and correct jobs before running them.

For a directory destination, configure its actual encrypted mount as a prerequisite. Replace
`<backup-storage>` with the storage ID and `<encrypted-mount>` with its mounted filesystem:

```sh
pvesm set <backup-storage> --is_mountpoint <encrypted-mount>
findmnt --mountpoint <encrypted-mount>
```

Confirm that the displayed source is the intended encrypted dataset or mapping. The storage
flag is an additional check; verify the live backing mount before each backup or restore.
Do not create directories or write data beneath an absent encrypted mount.

Container backups can stage plaintext files locally, including when the final target is PBS.
Use an encrypted filesystem for `tmpdir` on every node that runs these jobs. With that
filesystem already unlocked and mounted:

```sh
mountpoint -q <encrypted-mount> && install -d -m 1777 <encrypted-mount>/backup-tmp
```

Set the existing `tmpdir` entry in `/etc/vzdump.conf`, or add it if absent:

```text
tmpdir: <encrypted-mount>/backup-tmp
```

Mode `1777` permits unprivileged-container backup staging. Reserve capacity for both the
archive and any container staging copy. Check the configured location before starting jobs:

```sh
grep '^tmpdir:' /etc/vzdump.conf
findmnt -T <encrypted-mount>/backup-tmp
```

Require the intended encrypted source, not the root filesystem. Apply the backend guide's
boot/unlock dependencies and ensure the mount is available before scheduled backups run.
Moving the setting does not remove plaintext left by earlier backups.

## 3. Configure PBS encryption and retain its keys

### 3.1 Proxmox VE storage keys

Use an existing PBS datastore and account authorized for the required backup and restore
operations. Obtain the server certificate fingerprint through a trusted channel. Replace
`<pbs-storage>`, `<pbs-host>`, `<datastore>`, `<user@realm>` and `<fingerprint>` below with
those values. The final `--password` prompts interactively:

```sh
pvesm add pbs <pbs-storage> --server <pbs-host> --datastore <datastore> \
  --username <user@realm> --fingerprint <fingerprint> --encryption-key autogen --password
```

`autogen` creates a key without a passphrase, as required by unattended Proxmox VE jobs.
A passphrase-protected storage key cannot be opened by those jobs even though `pvesm` accepts
it. See [PBS storage configuration](https://pve.proxmox.com/pve-docs/pvesm.1.html#storage_pbs)
for account and datastore options.

**Warning:** Preserve `/etc/pve/priv/storage/<pbs-storage>.enc` in protected storage outside
the cluster immediately. The server cannot recover encrypted data without the corresponding
client key. Removing the storage definition deletes this file; joining a node to a cluster
replaces its local storage keys with the cluster's keys.

Verify the off-cluster copy and retain each old key while backups encrypted with it remain.
Test recovery with that retained key, including on a replacement node. For an existing
backup set, import its original key when configuring storage; generating a new key cannot
recover old data:

```sh
pvesm set <pbs-storage> --encryption-key <protected-original-key-file>
```

### 3.2 Standalone backup-client keys

For direct `proxmox-backup-client` backups, a key file may be protected by an interactive
passphrase. Create it with PBKDF2 and a passphrase of at least **14 bytes**; non-ASCII
characters can occupy multiple bytes:

```sh
proxmox-backup-client key create <protected-key-file> --kdf pbkdf2
```

The default scrypt KDF is unsupported by wolfProvider. Existing scrypt keys must be
re-protected with PBKDF2 using a stock client on a separate system before use on these nodes;
retain the same encryption key so existing backups remain recoverable. The licensed library
also limits PBKDF2 to 10,000,000 iterations.

PBS master-key recovery uses RSA operations outside the wolfCrypt validated boundary.
If used under your recovery policy, specify `--kdf pbkdf2` when importing the recovered key,
and use a passphrase of at least 14 bytes. The
[PBS client documentation](https://pbs.proxmox.com/docs/backup-client.html#using-a-master-key-to-store-and-recover-encryption-keys)
covers the master-key procedure. Retain a protected off-cluster copy of standalone keys too.

## 4. Test backup and restore

Run this round trip for each guest type and destination used in production. Use guests with
an operating system and known data. Choose unused restore IDs and an isolated test network
before starting copies, to avoid duplicate guest identities. Confirm the restore storage and
staging mount are unlocked and encrypted as in section 2.

Inside each source guest, record a representative file's SHA-256 digest and retain a copy
outside the guest for comparison:

```sh
sha256sum <known-file>
```

On the node, replace `<vmid>` and `<ctid>` with the source IDs and `<backup-storage>` with
the file-level or encrypted PBS destination:

```sh
vzdump <vmid> <ctid> --storage <backup-storage> --mode snapshot
pvesm list <backup-storage> --vmid <vmid>
pvesm list <backup-storage> --vmid <ctid>
```

Require successful jobs. For PBS, also confirm the job log identifies encryption. Select
the exact volume IDs created by this run; do not select an older backup merely because it
appears in the list. Container snapshot support depends on the source backend; staging may
be needed if the job falls back to suspend mode.

Restore the VM and container with their respective tools. Replace `<vm-backup-volume-id>`
and `<ct-backup-volume-id>` with those IDs, and `<guest-storage>` with the verified encrypted
restore destination:

```sh
qmrestore '<vm-backup-volume-id>' <new-vmid> --storage <guest-storage> --unique 1
pct restore <new-ctid> '<ct-backup-volume-id>' --storage <guest-storage>
qm config <new-vmid>
pct config <new-ctid>
```

Check all restored disks and container mount points against the intended storage and the
backup's contents. After isolating the restored guests' networking, start them:

```sh
qm start <new-vmid>
pct start <new-ctid>
```

Inside each restored guest, repeat `sha256sum <known-file>` and compare it with the externally
retained digest. Confirm the guest boots and its required application data is present.
A successful task alone does not establish a usable restore. Preserve the tested key and
backup until the recovery requirements are satisfied.

## 5. Diagnose encryption failures

If a PBS job reports `failed to load decryption key` or `no password input mechanism available`,
check whether its storage key has a passphrase. Supply the original key without passphrase
protection; keep the original key and all copies until restoration is verified. Do not replace
it with a newly generated key to repair access to existing backups.

`Algorithm (SCRYPT : 0)` indicates an incompatible key KDF; use section 3.2. The client's
benchmark also tests scrypt and is not an acceptance check for this deployment.

For provider or wrapper failures, use
[Node Maintenance and Recovery](WOLFSSL-PROXMOX-FIPS-NODE-CONFIGURATION.md).
After recovery, fully stop and start affected VMs and repeat the backup/restore round trip.
