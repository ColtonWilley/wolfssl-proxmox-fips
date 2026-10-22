# wolfSSL FIPS for Proxmox VE — Importing guests from VMware vSphere

Import guests onto a fresh wolfSSL-configured Proxmox target using the standard Proxmox
ESXi importer. The import procedure is unchanged; encrypted destination selection and
supported guest settings are the wolfSSL-specific requirements.

## 1. Prepare the source and destination

Complete [Installation](../_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md) and the selected
storage guide. Unlock the destination storage before importing.

Follow Proxmox's [VM preparation and ESXi import guidance](https://pve.proxmox.com/wiki/Migrate_to_Proxmox_VE#Automatic_Import_of_Full_VM)
for source access, guest drivers, firmware and network adjustments. In particular:

- Disks encrypted by a VMware storage policy must be decrypted before import. Retain access
  to the source key provider until the source is retired.
- The importer cannot read vSAN-backed disks; move them to supported source storage first.
- Shut down the source VM before importing and retain it for recovery.

Before shutdown, record application acceptance checks or checksums of representative data.
For a Linux guest, run inside the source guest:

```sh
sha256sum <data-file> > /root/migration.sha256
```

Keep these records for comparison after import. This does not change any encryption within
the guest; retain its own disk-encryption and recovery credentials separately.

## 2. Import onto encrypted storage

In the Proxmox web interface:

1. Add the ESXi import source under **Datacenter → Storage → Add → ESXi**. Enter the source
   credentials through the interface and verify the server identity.
2. Select the source storage on the destination node, choose the VM, and select **Import**.
3. Set the target storage to the encrypted storage configured by the destination guide.
   Check the mapping for every disk and any separately allocated EFI or TPM state volume.
4. Complete the import without starting a second copy of the source VM.

Do not select SPICE/QXL display, SPICE audio or SPICE USB redirection. The wolfSSL-configured
host does not support SPICE. Use the standard VGA/noVNC console or another supported display.

The APT installation supplies the system OpenSSL provider and required wrappers. Web-interface
imports do not require a separate provider configuration. `qemu-img` need not load wolfProvider
when it only copies local disk data; destination storage supplies the encryption. The ESXi
connection uses the importer's TLS path.

## 3. Verify before cutover

Review the imported disk placement and guest settings before starting the VM:

```sh
qm config <new-vmid>
wolfssl-fips-pve-verify
```

Make the driver, firmware and network adjustments described in Proxmox's migration guidance.
A virtual TPM does not place guest TPM cryptography inside wolfCrypt's validated boundary.

Continue with [Migration acceptance](WOLFSSL-PROXMOX-FIPS-MIGRATION-ACCEPTANCE.md).
