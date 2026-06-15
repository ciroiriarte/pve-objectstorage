# ADR 0001 — Tenant & identity model

- **Status:** Proposed (Phase 0 gate — blocks Phases 2 & 4)
- **Date:** 2026-06-15
- **Deciders:** Ciro Iriarte <ciro.iriarte+software@gmail.com>
- **Supersedes/refines:** AGENTS.md §7 (which floated a bare "tenant → PVE Group")

## Context

Proxmox VE has **no native "tenant" object**. We need a tenancy boundary that:

1. maps cleanly onto a Ceph RGW isolation primitive,
2. is authorized through PVE's existing ACL system (so admins use one mental model),
3. supports both auth modes in AGENTS.md §2.6 (native keys, and OIDC/STS later), and
4. is **upstreamable** — i.e. follows how Proxmox itself adds a new resource type.

What PVE already gives us (verified against `pve-access-control` source):

- ACLs bind `(path, user|group|token, role, propagate)`. Valid paths are a fixed
  whitelist regex in `check_path` (`src/PVE/AccessControl.pm:1283-1318`).
- Privileges and roles are declared in the same file; the SDN feature added
  `SDN.Allocate` / `SDN.Audit` / `SDN.Use` (`AccessControl.pm:1119-1125`) and a
  hierarchy of `/sdn/...` paths (e.g. `/sdn/zones/<zone>`, `/sdn/fabrics/<id>`).
  API handlers gate on them with `check => ['perm', '/sdn/zones/{zone}', ['SDN.Audit']]`
  (e.g. `pve-network` `PVE/API2/Network/SDN/Dns.pm:98`).
- `Pool.Allocate` / `Pool.Audit` (`AccessControl.pm:1143-1149`) back the existing
  `/pool/<id>(/sub){0,2}` resource-grouping container (`check_path:1293`).

What RGW already gives us:

- A native **tenant** namespace: users created as `tenant$user` and buckets scoped
  within the tenant, so bucket names cannot collide across tenants — hard,
  server-side multi-tenancy isolation.

## Decision

**A "tenant" is a first-class object-storage tenant, exposed at a new ACL path
`/objectstorage/<tenant>`, backed 1:1 by an RGW tenant namespace, and authorized
through standard PVE ACLs.** Concretely:

1. **Boundary:** PVE object-storage tenant `<t>` ⇔ **RGW tenant namespace `<t>`**.
   S3 users are `radosgw-admin user create --tenant=<t> --uid=<user>`; buckets live
   inside the tenant namespace. Isolation is enforced by RGW, not by us.

2. **ACL path:** add `/objectstorage` and `/objectstorage/<tenant>` to `check_path`
   (and a `pve-objectstorage-tenant-id` format), mirroring the `/sdn/...` precedent.

3. **Privileges (new), mirroring SDN's verb split:**
   - `S3.Allocate` — create/delete tenants, create/delete RGW daemons & S3 users,
     set quotas (admin / infrastructure).
   - `S3.Audit` — read tenant config, usage, quota (no secrets).
   - `S3.Use` — tenant self-service: view endpoint, **rotate own** access/secret
     keys, view own quota/usage, optionally provision buckets.
   - `S3.KeyManage` — *optional* finer split: manage/revoke keys without full
     `S3.Allocate`. Start folded into `S3.Use`/`S3.Allocate`; promote only if a real
     separation-of-duty need appears.

4. **Membership is an ACL binding, not an equation.** We do **not** hard-equate a
   tenant to one PVE group. Instead admins grant a **group** (preferred), user, or
   API token a **role** containing `S3.*` on `/objectstorage/<tenant>` — exactly how
   every other PVE resource is delegated. A convenience flow in the UI may create a
   matching group and binding in one step, but the data model keeps them separate.

5. **Stock roles:** ship `PVES3Admin` (`S3.Allocate`+`S3.Audit`) and `PVES3User`
   (`S3.Use`+`S3.Audit`) as predefined roles, alongside admins composing custom ones.

6. **Tenant metadata** (name, default placement target, quota, linked group, auth
   mode) is *structured cluster config* → store it as an **observed pmxcfs file**
   (e.g. `objectstorage.cfg`) registered in `PVE/Cluster.pm` + `pmxcfs/status.c`.
   This is the legitimate `cfs_register_file` case (contrast: raw cert blobs in
   AGENTS.md §2.4, which need no registration). **`memdb.c` is not touched.**

7. **Auth-mode mapping (forward-compatible with §2.6):**
   - *Mode A (native):* `S3.Use` lets a member mint/rotate native RGW keys for
     `<t>$<user>`.
   - *Mode B (OIDC/STS, Phase 5):* per-tenant RGW IAM roles; the
     `/objectstorage/<tenant>` ACL governs who may assume them. The PVE group bound
     to the tenant is the unit mapped onto the IAM trust policy.

## Alternatives considered

| Option | Why not (primary) |
|---|---|
| **Tenant = PVE Group** | A group is a *set of users*, not a resource container — no place to hang quota, placement, or an RGW namespace; `/access/groups/<g>` is for managing the group, not a resource root. Kept as the *membership* mechanism, not the tenant identity. |
| **Tenant = PVE Pool** (`/pool/<id>`) | Closest existing primitive and tempting (reuse `Pool.*`, nests 3 levels). Rejected for v1: pools aggregate VMs/CTs/storage, so this conflates object-storage tenancy with VM grouping, forces every S3 tenant to also be a VM pool, and `Pool.Audit/Allocate` can't express the S3 Audit-vs-KeyManage split. May offer *optional* pool↔tenant linkage later as convenience. |
| **Tenant = PVE User** | Too granular; can't share keys/quota/buckets across people; no team ownership. |
| **Tenant = PVE Realm** | A realm is an *authentication source*, not a resource boundary; can't host multiple tenants in one realm, nor one tenant across realms. |

## Consequences

**Positive**
- One ACL mental model; reuses ProxMox's most recent new-resource pattern (SDN), so
  it is the most upstreamable shape.
- Hard isolation delegated to RGW's own tenant namespacing.
- Clean path to OIDC/STS without reworking the tenancy boundary.

**Negative / costs**
- **New fork: `pve-access-control`** — patch `check_path`, register `S3.*` privs and
  the stock roles. This is a previously unlisted patch surface (now added to
  AGENTS.md §§3–4). Forked to `github.com/ciroiriarte/pve-access-control`.
- A small **observed-file registration** in `pve-cluster` for `objectstorage.cfg`
  (no `memdb.c` change; this is the sanctioned `cfs_register_file` path).
- `S3.*` privileges must be threaded through every new API handler with
  `check => ['perm', '/objectstorage/{tenant}', [...]]`.

## Open questions (resolve during Phase 1–2)

1. Does `objectstorage.cfg` live in `priv/` (if it ever holds secrets) or plain
   cluster config (metadata only, keys stay in RGW)? Prefer **metadata-only, plain**
   — keys never leave RGW.
2. Tenant-level quota: enforce via RGW per-user quota summed, or a dedicated RGW
   tenant/bucket quota? Decide once Phase 1 exercises `radosgw-admin` quota.
3. Naming/charset constraints to keep tenant ids valid as both an ACL path segment
   *and* an RGW tenant (intersect both regexes; likely `[A-Za-z0-9._-]+`).
4. Should `S3.KeyManage` ship in v1 or stay folded? Default: folded.

## References

- `pve-access-control` `src/PVE/AccessControl.pm:1119-1125` (SDN privs),
  `:1143-1149` (Pool privs), `:1166` (`$valid_privs`), `:1283-1318` (`check_path`).
- `pve-network` `src/PVE/API2/Network/SDN/Dns.pm:77-98`,
  `Fabrics.pm:32-66` (ACL-gated API pattern).
- AGENTS.md §2.4 (cert storage / pmxcfs), §2.6 (auth modes), §7 (tenant model),
  §10 (upstream cross-check). Ceph RGW multi-tenancy (`tenant$user`).
