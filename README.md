# wolfSSL FIPS for Proxmox VE

Deployment and maintenance guides for installing the wolfSSL integration from prebuilt APT
packages on fresh Proxmox VE nodes. The guides cover Ceph distributed storage, local ZFS or
LUKS storage, clustering, backup, and migration from existing Proxmox VE or VMware environments.
No source build or local patching is required for the APT deployment.

**Note:** The APT deployment model is still being evaluated for production readiness, and issues
may remain. Send feedback and relevant diagnostic logs using the [support instructions](#support).

VMware import instructions were reviewed against Proxmox guidance but were not rerun in the
latest validation pass.

## Choose a deployment

Choose by where guest disks are stored. The encryption mechanism follows from that decision.

| Storage architecture | Deployment guide | Encryption |
| --- | --- | --- |
| Distributed storage shared across nodes | [Ceph Deployment](docs/ceph/WOLFSSL-PROXMOX-FIPS-CEPH-DEPLOYMENT.md) | dm-crypt on each OSD, plus Ceph messenger and authentication cryptography |
| Local disks on a single node | [ZFS Deployment](docs/storage-cluster/WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-ZFS.md) or [LUKS Deployment](docs/storage-cluster/WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-LUKS.md) | ZFS native encryption or dm-crypt beneath the filesystem |
| Several nodes with local storage | ZFS or LUKS Deployment, then [Cluster](docs/storage-cluster/WOLFSSL-PROXMOX-FIPS-CLUSTER.md) | Per-node storage encryption, plus cluster membership and guest-movement cryptography; replication requires ZFS |

Clustering nodes with local storage is shared-nothing clustering, distinct from Ceph's shared storage.
dm-crypt is used by both the local LUKS deployment and Ceph's OSDs, but they are separate deployment
paths.

## Reading order

Choose the storage architecture before installing packages. Ceph and wolfZFS use different
kernel-module profiles and their deployment metapackages cannot be installed together. Ceph
nodes can also use the LUKS package for separate local encrypted storage.

1. For a multi-node deployment, read [Cluster planning](docs/storage-cluster/WOLFSSL-PROXMOX-FIPS-CLUSTER.md#1-plan-the-cluster)
   first. For Ceph, complete its node-preparation steps before the wolfSSL installation.
2. Complete [Foundation](docs/_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md), including repository
   selection, package installation and reboot, on each node.
3. Configure the selected storage. For Ceph, form the new all-FIPS Proxmox cluster before
   initializing Ceph.
4. Configure and test [Backup](docs/_shared/WOLFSSL-PROXMOX-FIPS-BACKUP.md). Use
   [Node Configuration and Recovery](docs/_shared/WOLFSSL-PROXMOX-FIPS-NODE-CONFIGURATION.md)
   for supported updates, verification and access recovery.
5. For local-storage clusters, use the [Cluster guide](docs/storage-cluster/WOLFSSL-PROXMOX-FIPS-CLUSTER.md)
   for migration, replication and HA requirements.

To bring existing guests onto a freshly deployed target, follow
[Migration from Proxmox VE](docs/_migration/WOLFSSL-PROXMOX-FIPS-MIGRATION-PVE.md) or
[Migration from vSphere](docs/_migration/WOLFSSL-PROXMOX-FIPS-MIGRATION-VSPHERE.md), then
[Migration Acceptance](docs/_migration/WOLFSSL-PROXMOX-FIPS-MIGRATION-ACCEPTANCE.md).

The guides document the wolfSSL-specific changes to standard Proxmox administration.
Storage encryption does not automatically encrypt the root filesystem, every guest volume,
or an external backup destination.

## Guides

| Guide | Purpose |
| --- | --- |
| [Foundation](docs/_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md) | Configure repository preferences and signing-key trust, install the shared packages and selected storage profile, then reboot and verify. |
| [Ceph Deployment](docs/ceph/WOLFSSL-PROXMOX-FIPS-CEPH-DEPLOYMENT.md) | Deploy Ceph from APT packages with encrypted OSDs and secure messenger and authentication cryptography. |
| [ZFS Deployment](docs/storage-cluster/WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-ZFS.md) | Deploy local storage with ZFS native encryption. |
| [LUKS Deployment](docs/storage-cluster/WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-LUKS.md) | Deploy local storage with LUKS/dm-crypt encryption. |
| [Cluster](docs/storage-cluster/WOLFSSL-PROXMOX-FIPS-CLUSTER.md) | Create and operate a new all-FIPS cluster, including guest migration and HA requirements; replication requires ZFS. |
| [Node Configuration and Recovery](docs/_shared/WOLFSSL-PROXMOX-FIPS-NODE-CONFIGURATION.md) | Apply supported package updates, verify activation, and recover administrative access and guest startup. |
| [Backup](docs/_shared/WOLFSSL-PROXMOX-FIPS-BACKUP.md) | Configure and verify backup and restore with vzdump or Proxmox Backup Server. |
| [Migration from Proxmox VE](docs/_migration/WOLFSSL-PROXMOX-FIPS-MIGRATION-PVE.md) | Move guests from an existing Proxmox deployment or its Proxmox Backup Server datastore. |
| [Migration from vSphere](docs/_migration/WOLFSSL-PROXMOX-FIPS-MIGRATION-VSPHERE.md) | Import guests from VMware vSphere or ESXi. |
| [Migration Acceptance](docs/_migration/WOLFSSL-PROXMOX-FIPS-MIGRATION-ACCEPTANCE.md) | Verify migrated guests and recovery before retiring the source. |

## Ceph patches

The files in [patches/ceph/](patches/ceph/) are retained from the previous manual-build
edition. They target the Ceph source used by Proxmox Ceph **20.2.2-pve1**, packaging commit
`06f6a7ab7187f570d9fc04b7ac9664f1fb6b84b0`. The current APT deployment supplies the integration
through packages; do not apply these source patches as part of that procedure.

| Patch | Purpose |
| --- | --- |
| [9997-ceph-volume-fips-luks-parameters.patch](patches/ceph/9997-ceph-volume-fips-luks-parameters.patch) | Select the cipher, key size, and PBKDF2 parameters when provisioning encrypted OSDs. |
| [9998-ceph-crypto-hmac-evp-mac.patch](patches/ceph/9998-ceph-crypto-hmac-evp-mac.patch) | Route HMAC through the OpenSSL 3 provider interface. |
| [9999-cephx-cached-evp.patch](patches/ceph/9999-cephx-cached-evp.patch) | Route the cephx authentication cipher through the OpenSSL provider interface. |

See the [patch notes](patches/ceph/README.md) for upstream provenance and licensing.

## Scope

This edition covers fresh Proxmox nodes and new clusters made entirely of wolfSSL-configured
nodes. Existing production or manual-build installations are not converted in place, mixed
stock/wolfSSL clusters are unsupported, and returning a node to stock requires reinstallation.
Guests can be migrated onto freshly deployed targets using the migration guides.

The public **FIPS Ready** channel is for evaluation and is not a FIPS-validated module. The
**licensed FIPS v5** channel supplies the licensed modules. Confirm applicable module versions,
approved services and operating-environment requirements with wolfSSL against the Security Policy.
The packages do not validate Proxmox or the entire operating system. Guest operating systems are
unchanged, and virtual TPM cryptography remains outside the wolfCrypt validated boundary.
[Foundation](docs/_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md) describes the deployment's
cryptographic scope and repository-verification exceptions.

## License

The original documentation and examples are licensed under **GPL-3.0-or-later**. The Ceph patches
carry separate LGPL terms and upstream notices. See [LICENSING.md](LICENSING.md) for the scope and
included license texts.

## Support

Send feedback and issue reports to [support@wolfssl.com](mailto:support@wolfssl.com). Include the
guide name and section, a brief description of the problem, `pveversion -v`,
`wolfssl-fips-pve-verify` output, and relevant service or task logs. Remove passwords, tokens,
private keys and other sensitive information before sharing.
