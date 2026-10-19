# Ceph authentication migration assessment

Verified 2026-10-19 on Neptune. This is a partial remediation; the remaining
compatibility warnings have not been resolved.

## Installed stack

- OpenShift 4.22.18; all three nodes run RHCOS kernel 5.14.0-687.55.1.el9_8.x86_64.
- ODF and Rook operators: 4.22.6-rhodf, installed and current CSVs match.
- Red Hat Ceph build: 20.2.1-405.el9cp, release 9.1.2. Use vendor build/errata
  information when evaluating backports; upstream version comparisons alone
  are insufficient.
- Ceph-CSI reports release-4.22, commit eac588b8e003a2d5fc5618161d6ffb5d560fb78b.

## Verified migration and action taken

`ceph auth dump-keys --format=json` reports AES256K for monitors, OSDs,
MDS daemons, managers, admin, crash collector, Ceph exporter, and RGW.
All reported rotating service keys also use AES256K. Daemon generation is 2.
Neither AUTH_INSECURE_SERVICE_KEY_TYPE nor AUTH_INSECURE_SERVICE_TICKETS
is present.

Removed the obsolete sticky mute for AUTH_INSECURE_ROTATING_SERVICE_KEY_TYPE
using `ceph health unmute AUTH_INSECURE_ROTATING_SERVICE_KEY_TYPE`.
The check remains absent after unmuting. No keys, client secrets, mounted
volumes, daemon generations, or allowed cipher settings were changed.

## Remaining warnings

Three existing sticky mutes remain:

- AUTH_INSECURE_CLIENT_KEY_TYPE: ten AES client entities: four legacy CSI
  identities, four storage-consumer CSI identities, the OCS metrics exporter,
  and the RBD mirror peer.
- AUTH_INSECURE_KEYS_ALLOWED: legacy AES authentication is still allowed.
- AUTH_INSECURE_KEYS_CREATABLE: creation of legacy AES keys is still allowed.

These warnings are real compatibility exceptions, not evidence of an
unfinished core daemon rotation. Ceph health is HEALTH_OK with these three
warnings muted; do not describe authentication migration as fully complete.

ODF's release-4.22 operator source explicitly maintains both allowed cipher
types. It requests AES for newly created storage-consumer CSI clients and
metrics-exporter clients, and for new cluster mirror-peer keys. Its
StorageCluster override currently forwards daemon key generation only;
exposing other fields in a CRD does not mean that the operator honors them.
Direct edits to operator-owned desired state are therefore not an established
ODF remediation path.

Upstream Rook requires Ceph-CSI 3.17.1+ and kernel AES256K support for CSI
migration (upstream Linux 7.0+, or 7.2+ in FIPS mode). The installed RHEL kernel
version alone cannot establish whether a backport supplies this capability.
Red Hat must confirm the supported combination and procedure before production
CSI credentials are changed or legacy cipher support is removed.

## Red Hat support request information

Ask Red Hat to confirm:

1. AES256K RBD and CephFS kernel-client support in the installed RHCOS kernel
   and release-4.22 Ceph-CSI build, including relevant errata/backports.
2. The supported ODF workflow for migrating storage-consumer CSI clients,
   legacy CSI identities, the metrics exporter, and the unused mirror-peer
   identity. Block-pool mirroring is not enabled on this cluster.
3. Whether a newer supported ODF/OpenShift build is required. The installed
   ODF subscription currently offers 4.22.6-rhodf; OpenShift offers no update.
4. The supported control for restricting allowed ciphers after all client
   migrations are complete. The installed build exposes
   `mon_auth_emergency_allowed_ciphers`, rather than the upstream
   `auth_allowed_ciphers` configuration option.

Do not manually rotate/delete production CSI identities, remove prior keys,
force service-ticket invalidation, replace Red Hat Ceph images with upstream
images, or set AES256K-only authentication to make the health checks disappear.
Those changes can prevent volume mounts or be reverted by the operators.

Once vendor prerequisites are confirmed, first verify provisioning, mounting,
read/write, and cleanup on disposable RBD and CephFS volumes. Use the approved
operator-driven rotation with prior credentials retained; migrate existing
mounts using the approved maintenance procedure before retiring prior keys.
Only remove remaining mutes and legacy cipher support after the complete
client inventory has migrated and application I/O is verified.

## Sources

- [Red Hat CVE-2025-30156 advisory](https://access.redhat.com/security/cve/CVE-2025-30156)
- [Rook CephX migration and compatibility guidance](https://rook.io/docs/rook/latest-release/Storage-Configuration/Advanced/cephx-key-rotation/)
- [ODF release-4.22 operator: cluster policy](https://github.com/red-hat-storage/ocs-operator/blob/c6ce334edc6d0f4467b7ef3a7259410f4280e1b4/internal/controller/storagecluster/cephcluster.go#L430)
- [ODF release-4.22 operator: storage-consumer CSI policy](https://github.com/red-hat-storage/ocs-operator/blob/c6ce334edc6d0f4467b7ef3a7259410f4280e1b4/internal/controller/storageconsumer/storageconsumer_controller.go#L589)
- [ODF release-4.22 operator: metrics-client policy](https://github.com/red-hat-storage/ocs-operator/blob/c6ce334edc6d0f4467b7ef3a7259410f4280e1b4/internal/controller/storagecluster/exporter.go#L1044)
