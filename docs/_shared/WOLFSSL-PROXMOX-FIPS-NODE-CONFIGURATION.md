# wolfSSL FIPS on Proxmox VE — Node Configuration and Recovery

Configure wolfProvider for management TLS and guest processes after completing your storage
deployment guide and acceptance checks. Recovery and rollback procedures follow in §4–§5. This guide
applies to clustered and standalone nodes.

---

## 1. Management TLS (`pveproxy`)

Configure `pveproxy` to use wolfProvider for the web interface and API on port 8006:

```bash
mkdir -p /etc/systemd/system/pveproxy.service.d
printf '[Service]\nEnvironment="OPENSSL_CONF=/etc/wolfprovider/openssl.cnf"\n' \
  > /etc/systemd/system/pveproxy.service.d/10-wolfprov.conf
```

**Important:** Leave `pvedaemon` on system OpenSSL. Its login-ticket digest is unsupported by
wolfProvider; routing it through the provider causes web and API logins to fail with `401`.
Only management TLS is configured here, not login-ticket cryptography.

**Note:** `pvesh` runs the API in its own unscoped process and can succeed when logins are broken. Use the
`curl` login check in §3.

Restrict TLS 1.2 and 1.3 to AES-GCM in `/etc/default/pveproxy`, using the settings documented in
[pveproxy(8)](https://pve.proxmox.com/pve-docs/pveproxy.8.html):

```bash
cat > /etc/default/pveproxy <<'EOF'
CIPHERS="ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256"
CIPHERSUITES="TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256"
HONOR_CIPHER_ORDER="1"
EOF
```

```bash
systemctl daemon-reload && systemctl restart pveproxy spiceproxy   # both read /etc/default/pveproxy
```

---

## 2. Guest processes (QEMU)

Configure the guest's QEMU process for:

- **Ceph storage:** QEMU handles the client side of msgr2 for guest storage I/O.
- **Proxmox Backup Server (PBS):** QEMU encrypts backup chunks and handles the server connection.

On ZFS or LUKS without PBS, guest I/O is encrypted in the kernel and the wrapper is optional.
Install it if PBS may be added later; setting the environment variable is harmless when unused.

`pvedaemon` starts guests but must remain unscoped (§1) and does not read `/etc/environment`.
Use wrappers to set the provider configuration for the guest and restore processes:

```bash
# Remove the kvm symlink before writing; otherwise QEMU is overwritten and execs itself in a loop.
# Leave an existing wrapper in place.
if [ "$(head -c2 /usr/bin/kvm)" != '#!' ]; then
  rm -f /usr/bin/kvm
  cat > /usr/bin/kvm <<'EOF'
#!/bin/bash
export OPENSSL_CONF=/etc/wolfprovider/openssl.cnf
# Preserve argv[0]=kvm for KVM acceleration; otherwise QEMU uses software emulation and cpu: host fails.
exec -a "$0" /usr/bin/qemu-system-x86_64 "$@"
EOF
  chmod +x /usr/bin/kvm
fi

# PBS VM restores decrypt chunks in pbs-restore, which needs its own wrapper:
if [ "$(head -c2 /usr/bin/pbs-restore)" != '#!' ]; then
  cp -a /usr/bin/pbs-restore /usr/bin/pbs-restore.real
  cat > /usr/bin/pbs-restore <<'EOF'
#!/bin/bash
export OPENSSL_CONF=/etc/wolfprovider/openssl.cnf
exec -a "$0" /usr/bin/pbs-restore.real "$@"
EOF
  chmod +x /usr/bin/pbs-restore
fi
```

**Important:** Running guests retain their original environment. Stop and start them after installing
the wrapper; a reboot or migration is also sufficient. Install it on every node before moving guests:
the destination node's `/usr/bin/kvm` determines their environment, and a missing wrapper causes no
reported error.

**Caution:** Unpacking `pve-qemu-kvm` replaces both wrappers with the stock files. Hold the package
and reapply the wrappers after an upgrade:

```bash
apt-mark hold pve-qemu-kvm
```

A hold prevents version changes, not reinstallation. `apt-get install --reinstall` replaces the
wrappers and clears the hold without warning. Reapply both wrappers and the hold afterward.

**Note:** Run this section on Ceph nodes too. It preserves the `kvm` wrapper already installed by
the deployment guide and adds the `pbs-restore` wrapper.

Check a guest started after wrapper installation. The environment can be checked while idle;
wolfProvider appears in the process maps only after the first OpenSSL use.

**Important:** On Ceph, start the guest through the web interface or HA. A root-shell start inherits
`OPENSSL_CONF` from `/etc/environment` and can pass this check without a working wrapper.

```bash
head -c2 /usr/bin/kvm; echo                              # -> #!
apt-mark showhold | grep -c pve-qemu-kvm                 # -> 1
QPID=$(qm status <vmid> --verbose | awk '/^pid:/{print $2}')
readlink /proc/$QPID/exe                                 # -> .../qemu-system-x86_64
tr '\0' '\n' < /proc/$QPID/environ | grep OPENSSL_CONF   # -> the provider configuration
# Back up to PBS, not file-level storage, and check the maps after the job finishes:
vzdump <vmid> --storage <proxmox-backup-server-storage> --mode snapshot
grep -c libwolfprov /proc/$QPID/maps                     # -> non-zero
```

**Note:** On Ceph, the msgr2 session loads the provider at guest startup; a running guest must have a
non-zero map count. On PBS, a zero count can mean the guest predates the wrapper, the provider has not
loaded yet, or the backup used file-level storage, which performs no cryptography in QEMU. On small
guests, loading can occur after `vzdump` reports completion; check once the job finishes. The provider
stays mapped after use.

---

## 3. Verify the management interface

```bash
# Check the pveproxy environment:
systemctl show pveproxy -p Environment | grep -o 'OPENSSL_CONF=[^ ]*'

# Check HTTPS and a login through pveproxy -> pvedaemon. Enter the password at the prompt.
curl -sk -o /dev/null -w '%{http_code}\n' https://127.0.0.1:8006/       # -> 200
read -rs -p "root@pam password: " RPW; echo
curl -sk -d "username=root@pam&password=$RPW" https://127.0.0.1:8006/api2/json/access/ticket \
  | grep -o '"ticket":"PVE:[^"]*' | head -c 40      # -> a PVE:root@pam:... ticket

systemctl is-active pvedaemon pveproxy
```

**Important:** Confirm that a login ticket is issued, not just that HTTPS responds. If `pvedaemon`
was also scoped, remove `/etc/systemd/system/pvedaemon.service.d/` and restart both services.

---

## 4. Recover administrative access

An ed25519 administrative key is rejected by the SSH FIPS configuration. An incomplete userspace
installation can also break SSH: `sshd` loads the provider on each connection and can fail with
`startup pipe … reset`.

**Restore SSH and web access**

Log in on the node console and temporarily remove the provider settings:

```bash
mkdir -p /root/wolfprov-off
for f in ssh pveproxy proxmox-backup-proxy corosync; do
  d=/etc/systemd/system/$f.service.d/10-wolfprov.conf
  if [ -f "$d" ]; then mv "$d" /root/wolfprov-off/$f.conf; echo "moved aside: $f"
  else echo "not scoped on this node: $f"; fi
done
# For an ed25519-key lockout, also move aside the server-side SSH algorithm restriction.
# The client-side restriction remains in place for outbound SSH.
[ -f /etc/ssh/sshd_config.d/10-wolfprov-fips.conf ] \
  && mv /etc/ssh/sshd_config.d/10-wolfprov-fips.conf /root/wolfprov-off/sshd-fips.conf
systemctl daemon-reload
systemctl restart ssh          # SSH returns, on the system OpenSSL
systemctl restart pveproxy     # restart to apply the change to the running process
```

**Restore guest and backup tools**

Broken userspace also prevents wrapped QEMU processes from starting. Restore the stock binaries:

```bash
if [ "$(head -c2 /usr/bin/kvm)" = '#!' ]; then
  rm -f /usr/bin/kvm && ln -s qemu-system-x86_64 /usr/bin/kvm
fi
if [ "$(head -c2 /usr/bin/pbs-restore)" = '#!' ]; then
  if [ -f /usr/bin/pbs-restore.real ]; then
    mv /usr/bin/pbs-restore.real /usr/bin/pbs-restore
  else
    # The wrapper's target is missing. Reinstall from the reachable Proxmox package repository.
    # Reinstallation clears the hold; §2 reapplies it.
    apt-get install -y --reinstall pve-qemu-kvm
  fi
else
  rm -f /usr/bin/pbs-restore.real     # a package operation already restored the packaged binary
fi
```

Restore the Backup guide's wrapped binaries from their saved copies, or reinstall their packages if
those copies are missing:

```bash
for b in proxmox-backup-client proxmox-file-restore; do
  [ "$(head -c2 /usr/bin/$b)" = '#!' ] || continue
  if [ -f /usr/bin/$b.real ]; then mv /usr/bin/$b.real /usr/bin/$b
  # Ask dpkg for the owning package; its name differs from the binary's:
  else apt-get install -y --reinstall "$(dpkg -S /usr/bin/$b | cut -d: -f1)"; fi
done
proxmox-backup-client version                    # -> client version: <version>
proxmox-file-restore help >/dev/null             # -> rc 0; this binary has no `version` subcommand
```

**Restore wolfProvider configuration**

Correct the cause first: replace the ed25519 key with ECDSA or RSA, or rebuild userspace to a staging
prefix and verify it with `openssl list -providers` and `sshd -t` before installation.

Guests started during recovery do not use wolfProvider, even if health checks pass. Restore every
wrapper and restart the guests:

```bash
# Reapply §2 for kvm and pbs-restore, and Backup §3.1 for proxmox-backup-client and proxmox-file-restore.
# Then restart every running guest under the restored wrappers:
qm list; pct list                      # restart every guest that is running
```

**Important:** On clustered nodes, an unloadable provider prevents corosync from starting and the
node leaves the cluster (Cluster guide §3.2). Removing its drop-in allows stock cryptography during
recovery. Restore the drop-in and restart corosync in a maintenance window to resume using wolfProvider.

Restore the saved drop-ins and SSH restriction:

```bash
for f in ssh pveproxy proxmox-backup-proxy corosync; do
  [ -f /root/wolfprov-off/$f.conf ] || continue
  mkdir -p /etc/systemd/system/$f.service.d
  mv /root/wolfprov-off/$f.conf /etc/systemd/system/$f.service.d/10-wolfprov.conf
done
[ -f /root/wolfprov-off/sshd-fips.conf ] && \
  mv /root/wolfprov-off/sshd-fips.conf /etc/ssh/sshd_config.d/10-wolfprov-fips.conf
systemctl daemon-reload
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf /usr/sbin/sshd -t && echo "sshd OK"   # must be rc=0
systemctl restart ssh pveproxy
```

The disk, kernel, and encrypted storage remain unchanged throughout recovery.

---

## 5. Revert kernel FIPS enforcement

Keep a disk or boot-environment snapshot where possible. Revert only the kernel boot setting to
`fips=0`; the custom kernel, provider configuration, and encrypted storage remain deployed.

```bash
sed -i 's/fips=1/fips=0/' /etc/default/grub
# Confirm the edit; sed succeeds even without a match. If fips= is in a grub.d drop-in, edit it there.
grep -r 'fips=' /etc/default/grub /etc/default/grub.d/ 2>/dev/null   # -> fips=0, and no fips=1 anywhere
update-grub
reboot
```

After boot:

```bash
grep -o 'fips=[01]' /proc/cmdline      # -> fips=0
uname -r                               # -> still the CRNG kernel
cat /proc/sys/crypto/fips_enabled      # -> still 1: module loading sets it, so it cannot confirm the revert
# Test the kernel crypto API directly; openssl checks userspace, not kernel enforcement.
# Expect enforcement OFF. INCONCLUSIVE means the test could not run and the revert is unconfirmed.
python3 - <<'EOF'
import socket
def bind(alg):
    try:
        socket.socket(socket.AF_ALG, socket.SOCK_SEQPACKET, 0).bind(("hash", alg))
        return True
    except OSError:
        return False
if not bind("sha256"):
    print("INCONCLUSIVE: the kernel crypto user API is unavailable on this node")
elif bind("md5"):
    print("kernel FIPS enforcement OFF (kernel served md5)")
else:
    print("kernel FIPS enforcement ON (kernel refused md5)")
EOF
curl -sk -o /dev/null -w '%{http_code}\n' https://127.0.0.1:8006/       # -> 200
pvesm status                           # -> the encrypted storage, once unlocked below
qm list                                # -> guests start as usual
```

Unlock storage as required by the deployment guide: a passphrase-model ZFS pool returns locked;
a key-file LUKS volume is opened by its unit. Start guests, then restore enforcement when ready:

```bash
sed -i 's/fips=0/fips=1/' /etc/default/grub
grep -r 'fips=' /etc/default/grub /etc/default/grub.d/ 2>/dev/null   # -> fips=1, and no fips=0 anywhere
update-grub
reboot                                 # the deployment guide's acceptance checks pass again after boot
```

**Note:** Changing `fips=` does not require re-signing the kernel.

---

## Next

Backups are covered in the accompanying Backup guide.
