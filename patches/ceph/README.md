# Ceph patches

These patches target the Ceph source used by Proxmox Ceph **20.2.2-pve1**, packaging commit
`06f6a7ab7187f570d9fc04b7ac9664f1fb6b84b0` in
[Proxmox's Ceph repository](https://git.proxmox.com/?p=ceph.git;a=summary).

These source patches belong to the previous manual-build edition. The current
[Ceph Deployment guide](../../docs/ceph/WOLFSSL-PROXMOX-FIPS-CEPH-DEPLOYMENT.md) uses prebuilt APT
packages and does not require applying them. Refer to repository history for the previous
source-build procedure. Patch contents, licensing and attribution are retained below.

| Patch | Upstream file |
| --- | --- |
| [9997-ceph-volume-fips-luks-parameters.patch](9997-ceph-volume-fips-luks-parameters.patch) | `src/ceph-volume/ceph_volume/util/encryption.py` |
| [9998-ceph-crypto-hmac-evp-mac.patch](9998-ceph-crypto-hmac-evp-mac.patch) | `src/common/ceph_crypto.h` |
| [9999-cephx-cached-evp.patch](9999-cephx-cached-evp.patch) | `src/auth/Crypto.cc` |

## License and attribution

Modifications: Copyright (C) 2026 wolfSSL Inc.

wolfSSL's modifications are licensed under **LGPL-2.1-only OR LGPL-3.0-only**, at your option.
Existing Ceph material retains its applicable upstream licenses and notices. See
[LICENSING.md](../../LICENSING.md), [LGPL-2.1](../../LICENSES/LGPL-2.1.txt), and
[LGPL-3.0](../../LICENSES/LGPL-3.0.txt).

Ceph v20.2.2's [COPYING](https://github.com/ceph/ceph/blob/v20.2.2/COPYING) contains the following
default notice for files without a more specific entry:

```text
Files: *
Copyright: (c) 2004-2010 by Sage Weil <sage@newdream.net>
License: LGPL-2.1 or LGPL-3 (see COPYING-LGPL2.1 and COPYING-LGPL3)
```

The upstream [src/auth/Crypto.cc](https://github.com/ceph/ceph/blob/v20.2.2/src/auth/Crypto.cc)
also carries this file notice, retained by patch `9999`:

```text
Ceph - scalable distributed file system

Copyright (C) 2004-2009 Sage Weil <sage@newdream.net>

This is free software; you can redistribute it and/or
modify it under the terms of the GNU Lesser General Public
License version 2.1, as published by the Free Software
Foundation.  See file COPYING.
```

The upstream [ceph-volume package metadata](https://github.com/ceph/ceph/blob/v20.2.2/src/ceph-volume/setup.py)
additionally identifies its license as `LGPLv2+`. These notices describe the upstream material;
the patch license does not replace or remove existing upstream rights or attribution.
