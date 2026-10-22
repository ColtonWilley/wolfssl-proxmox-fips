# wolfSSL FIPS for Proxmox VE — Node Maintenance and Recovery

Use this guide after [Installation](../_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md)
and the selected storage guide. Commands run as root on the Proxmox node unless labeled
otherwise. Keep console access available during updates and recovery.

## 1. Update the installed packages

Plan a maintenance window and confirm that backups and encryption keys are available.
On a cluster, follow the [Cluster guide](../storage-cluster/WOLFSSL-PROXMOX-FIPS-CLUSTER.md)
and update one node at a time, preserving quorum and storage availability.

Refresh the preferences before the general upgrade. APT reads them when each command starts;
installing new preferences in the same transaction as the other packages is too late.

```sh
apt update
apt install wolfssl-fips-pve-apt
```

On encrypted ZFS nodes, replace the install command above with:

```sh
apt install wolfssl-fips-pve-apt wolfssl-fips-pve-apt-zfs
```

Then review and apply the upgrade:

```sh
apt full-upgrade --allow-downgrades
```

`--allow-downgrades` permits compatible wolfSSL rebuilds whose versions sort below the
upstream versions. Use this command instead of `apt upgrade`. Do not accept unexpected
removal of the integration or storage packages. In particular, refreshing both ZFS
preferences first keeps the userspace tools and kernel modules on the same OpenZFS release.

**Important:** Upstream security fixes for pinned components become available on these nodes
only after wolfSSL publishes compatible rebuilds. A newly installed stock Proxmox kernel
does not replace the wolfSSL kernel pinned for boot. Review held packages and verifier
warnings, and contact wolfSSL when a required security update is awaiting a rebuild; do not
remove the preferences to bypass the dependency constraints.

```sh
apt list --upgradable
```

Package installation does not replace every running process. Evacuate or shut down guests
using the cluster's normal maintenance procedure, then reboot:

```sh
reboot
```

## 2. Verify activation

After reboot, wait for `pve-guests` to finish starting the configured guests:

```sh
uname -r
systemctl is-active pve-guests
wolfssl-fips-pve-verify --login
```

The kernel must be the intended `+wolfcrng` release and `pve-guests` must be `active`.
The verifier prompts for the root password to check management login. Resolve every `FAIL`
and assess each warning before returning the node to service. Open a fresh SSH connection
with the ECDSA or RSA key configured during Installation.

Repeat the selected storage guide's unlock, placement and data checks. On a cluster,
confirm `pvecm status` reports `Quorate: Yes`; on Ceph, also require the health and OSD
conditions in its deployment guide before advancing to another node. A verifier pass does
not prove that every backup destination, restored guest or application is correctly configured.

The packaged wrappers cover QEMU, PBS restore and backup clients. They survive ordinary
package updates through dpkg diversions; do not recreate the old manual wrappers or package
holds. If a VM was started while wrappers were disabled during recovery, shut it down fully
and start it again. A reboot inside the guest does not replace its QEMU process.

## 3. Retain a working kernel

Keep the previous wolfSSL kernel as a boot fallback. The preference package retains its
matching wolfCrypt module and, on encrypted ZFS nodes, its wolfZFS modules for as long as
Proxmox retains that kernel. Review `apt autoremove` proposals as a set: kernel and matching
modules must be removed together. Do not remove the running kernel or the last working
fallback.

To boot an installed previous wolfSSL kernel once, replace `<kernel-version>` with its full
version from the installed package list:

```sh
dpkg -l 'proxmox-kernel-*+wolfcrng*'
proxmox-boot-tool kernel pin <kernel-version> --next-boot
reboot
```

The verifier reports the temporary kernel selection. The following reboot returns to the
normal pinned kernel. Use this for diagnosis; a previous release may lack newer security fixes.
For general boot and retention administration, see
[Proxmox boot management](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#sysboot).

## 4. Recover administrative access

### 4.1 SSH key rejection

An Ed25519 key cannot authenticate under the packaged SSH policy. At the node console,
authorize an ECDSA or RSA public key as described in
[Installation, section 2](../_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md#2-prepare-ssh-access).
Test a fresh connection with that key before closing the console. If the server configuration
was changed, validate it before restarting SSH:

```sh
sshd -t && systemctl restart ssh
```

### 4.2 Provider cannot load

A missing or damaged provider can disable SSH, management login, corosync and storage
unlock. Diagnose at the console before restarting services:

```sh
openssl dgst -sha256 /dev/null
echo $?
journalctl -b SYSLOG_FACILITY=10 --grep='provider error'
```

A working digest prints `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`
and exits `0`. An empty `openssl list -providers` listing is a failure even if it exits `0`.

If `libwolfprov.so` was moved, restore that file. If it was deleted, reinstall the provider
from a cached package or download it on a working node using the same wolfSSL repository
channel, then transfer it through trusted removable or virtual media. APT downloads can
fail while the provider is unavailable; `dpkg` can install the local package.

On the working node:

```sh
apt download wolfprovider-fips
```

At the affected node's console, replace `<provider-package.deb>` with the transferred or
cached package matching the installed library:

```sh
dpkg -i <provider-package.deb>
test -s /etc/wolfprovider/openssl.cnf
```

If the configuration check fails, or the provider still fails to load, stop and obtain
wolfSSL support for package recovery. Removing service drop-ins alone cannot restore
stock OpenSSL: the installed library selects wolfProvider by default.

Once the configuration file exists and the digest check succeeds again, APT can repair the
shared packages. Reinstall provider, library and config in the same transaction; the APT
hook requires config to be installed or included in that transaction:

```sh
apt install --reinstall --allow-downgrades wolfprovider-fips libwolfssl-fips wolfssl-fips-pve-config
test -s /etc/wolfprovider/openssl.cnf
```

Stop if the transaction or configuration check fails. Verify both provider content and a
cryptographic operation:

```sh
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf openssl list -providers
OPENSSL_CONF=/etc/wolfprovider/openssl.cnf openssl dgst -sha256 /dev/null
echo $?
sshd -t
```

Require `name: wolfSSL Provider FIPS`, `status: active`, the digest above, exit status `0`,
and no SSH configuration error. Shut down guests from inside their operating systems if
Proxmox cannot control them, then reboot the node and complete section 2. Confirm encrypted
storage has unlocked before restarting affected guests. Recover cluster nodes serially,
checking quorum before continuing to another node.

This procedure repairs the installed wolfSSL configuration. Returning permanently to stock
Proxmox requires reinstallation; package removal is not a supported rollback procedure.

### 4.3 Resume guests after an unlock failure

A failed storage-unlock dependency blocks automatic startup of all guests on the node,
including guests on other storage. Identify the failure:

```sh
systemctl --failed
journalctl -b -u 'wolfzfs-unlock@*' -u 'wolfssl-fips-luks@*'
```

Restore the missing key or correct the mount failure using the selected storage guide,
start its unlock service, and confirm the intended encrypted backing mount. Then start
guests configured for boot:

```sh
pvesh create /nodes/$(hostname)/startall
```

Wait for that task to complete; start any other required guests individually with `qm start`
or `pct start`. Do not use `systemctl start pve-guests`: the unit only permits dependency
activation. It remains inactive until the next boot, so section 2's wait for `active` does
not apply to this recovery session. Run the verifier once guest startup finishes; its
warning about inactive `pve-guests` is expected until the next successful boot.

**Important:** An inactive `pve-guests` will not stop guests during the next shutdown.
Before the next reboot, explicitly stop them and wait for the task to complete:

```sh
pvesh create /nodes/$(hostname)/stopall
qm list
pct list
```

Confirm the guests are stopped before rebooting. After a successful boot, repeat section 2
and the storage guide's unlock checks.
