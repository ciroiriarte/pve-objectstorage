# Changelog

All notable changes to **pve-objectstorage** are documented here.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Maintained by Ciro Iriarte <ciro.iriarte+software@gmail.com>.

## [Unreleased]

### Added
- Initial repository scaffolding for native S3 Object Storage support in
  Proxmox VE 9.x.
- `AGENTS.md` design & contributor guide (Native RADOS Gateway / S3 + Unified
  Infrastructure Proxy proposal), symlinked as `CLAUDE.md`.
- AGPLv3 `LICENSE` to match Proxmox VE upstream and enable upstream contribution.
- Open-design sections (§7 tenant/identity model, §8 operational concerns,
  §9 failure modes) capturing decisions to resolve before implementation.

### Changed
- Fact-checked the spec against current upstream `pve-manager`/`pve-cluster`/
  `pve-storage` trees (tri-model review) and corrected:
  - Removed the unnecessary `pmxcfs` C fork — cert/key blobs go under
    `/etc/pve/priv/<service>/` via plain file ops (existing storage-secret pattern);
    `memdb.c` was the wrong target regardless.
  - Patch surfaces updated to `www/manager6/*.js` + `PVE/CLI/pveceph.pm` +
    `PVE/API2/...` (not the generated `pvemanagerlib.js`).
  - `ip vrf exec` described as VRF-scoped (not network-namespace isolation).
  - RGW pools modelled as placement targets / storage classes incl. mandatory
    `data_extra_pool` for EC; bucket placement is immutable after creation.
  - ACME: DNS-01 preferred (HTTP-01 still supported); added cert-lead renewal.
  - Versioning: continuous rebase + narrow per-package pins; dropped the unsound
    `+rgwN` / `Pin-Priority: 1001` claim.
  - Infrastructure Proxy UI relocated to Datacenter level (adjacent to ACME/SDN).

[Unreleased]: https://github.com/ciroiriarte/pve-objectstorage/commits/main
