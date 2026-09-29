# wolfSSL FIPS for Proxmox VE

Build and deployment guides for using wolfCrypt FIPS cryptography with Proxmox VE. The guides cover
Ceph distributed storage, local ZFS or LUKS storage, clustering, backup, and migration from existing
Proxmox VE or VMware environments.

This repository contains ten guides and three Ceph source patches. Obtain the licensed wolfSSL FIPS
source bundle separately from wolfSSL; the guides identify the other upstream sources and patches.
You build, sign, and install the components and retain the private signing key on the builder. No
binaries or licensed FIPS source bundles are included.

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

Start with [Shared Build Foundation](docs/_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md). It builds
the shared kernel and userspace artifacts on the builder. Follow the version pins and prerequisites
in that guide and the selected deployment guide.

1. Complete **Foundation**, then **Ceph**, **ZFS**, or **LUKS Deployment**.
2. Continue with [Node Configuration and Recovery](docs/_shared/WOLFSSL-PROXMOX-FIPS-NODE-CONFIGURATION.md)
   and [Backup](docs/_shared/WOLFSSL-PROXMOX-FIPS-BACKUP.md).
3. For several nodes using local storage, continue with the
   [Cluster guide](docs/storage-cluster/WOLFSSL-PROXMOX-FIPS-CLUSTER.md).
4. To bring existing guests onto the deployed target, follow
   [Migration from Proxmox VE](docs/_migration/WOLFSSL-PROXMOX-FIPS-MIGRATION-PVE.md) or
   [Migration from vSphere](docs/_migration/WOLFSSL-PROXMOX-FIPS-MIGRATION-VSPHERE.md), then
   [Migration Acceptance](docs/_migration/WOLFSSL-PROXMOX-FIPS-MIGRATION-ACCEPTANCE.md).

Each deployment guide includes its own verification procedure. Migration starts with an already
deployed FIPS target; the guides do not convert an existing deployment in place.

The guides follow the Proxmox VE Administration Guide's structure. Standard Proxmox procedures are
referenced where applicable; steps changed by the wolfSSL FIPS integration are documented directly.

## Guides

| Guide | Purpose |
| --- | --- |
| [Foundation](docs/_foundation/WOLFSSL-PROXMOX-FIPS-FOUNDATION.md) | Build the CRNG kernel, retain its signing key, and build the matched wolfSSL and wolfProvider libraries. |
| [Ceph Deployment](docs/ceph/WOLFSSL-PROXMOX-FIPS-CEPH-DEPLOYMENT.md) | Build and deploy Ceph with encrypted OSDs and provider routing for messenger and authentication cryptography. |
| [ZFS Deployment](docs/storage-cluster/WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-ZFS.md) | Deploy local storage with ZFS native encryption. |
| [LUKS Deployment](docs/storage-cluster/WOLFSSL-PROXMOX-FIPS-DEPLOYMENT-LUKS.md) | Deploy local storage with LUKS/dm-crypt encryption. |
| [Cluster](docs/storage-cluster/WOLFSSL-PROXMOX-FIPS-CLUSTER.md) | Configure clustering, migration, HA, and recovery for nodes with local storage; replication requires ZFS. |
| [Node Configuration and Recovery](docs/_shared/WOLFSSL-PROXMOX-FIPS-NODE-CONFIGURATION.md) | Configure management TLS, recover administrative access, and revert. |
| [Backup](docs/_shared/WOLFSSL-PROXMOX-FIPS-BACKUP.md) | Configure and verify backup and restore with vzdump or Proxmox Backup Server. |
| [Migration from Proxmox VE](docs/_migration/WOLFSSL-PROXMOX-FIPS-MIGRATION-PVE.md) | Move guests from an existing Proxmox deployment or its Proxmox Backup Server datastore. |
| [Migration from vSphere](docs/_migration/WOLFSSL-PROXMOX-FIPS-MIGRATION-VSPHERE.md) | Import guests from VMware vSphere or ESXi. |
| [Migration Acceptance](docs/_migration/WOLFSSL-PROXMOX-FIPS-MIGRATION-ACCEPTANCE.md) | Verify migrated guests and recovery before retiring the source. |

## Ceph patches

The patches in [patches/ceph/](patches/ceph/) apply to the Ceph source used by Proxmox Ceph
**20.2.2-pve1**, packaging commit `06f6a7ab7187f570d9fc04b7ac9664f1fb6b84b0`. Apply them in the order
below, following the Ceph Deployment guide's build procedure.

| Patch | Purpose |
| --- | --- |
| [9997-ceph-volume-fips-luks-parameters.patch](patches/ceph/9997-ceph-volume-fips-luks-parameters.patch) | Select the cipher, key size, and PBKDF2 parameters when provisioning encrypted OSDs. |
| [9998-ceph-crypto-hmac-evp-mac.patch](patches/ceph/9998-ceph-crypto-hmac-evp-mac.patch) | Route HMAC through the OpenSSL 3 provider interface. |
| [9999-cephx-cached-evp.patch](patches/ceph/9999-cephx-cached-evp.patch) | Route the cephx authentication cipher through the OpenSSL provider interface. |

See the [patch notes](patches/ceph/README.md) for upstream provenance and licensing.

## Scope

The guides describe data-at-rest encryption, backup encryption, and the in-transit surfaces of each
deployment. Management authentication is outside the cryptographic claim. Each guide specifies its
scope, limitations, and the checks used to verify the configuration.

Every FIPS claim is conditional on the customer's licensed wolfSSL FIPS bundle, built and operated
according to its Security Policy. This repository does not grant a FIPS certification to Proxmox VE
or to a deployment. The customer selects the bundle's configuration and verifies the resulting system
in their own environment.

## License

The original documentation and examples are licensed under **GPL-3.0-or-later**. The Ceph patches
carry separate LGPL terms and upstream notices. See [LICENSING.md](LICENSING.md) for the scope and
included license texts.

## Support

Contact [support@wolfssl.com](mailto:support@wolfssl.com) with the guide name and section number.
