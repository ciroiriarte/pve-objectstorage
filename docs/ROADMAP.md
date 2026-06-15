# Implementation Roadmap

Phased delivery plan for native S3 Object Storage in Proxmox VE 9.x. Derived from
the verified design in [`../AGENTS.md`](../AGENTS.md) (see §10 for the upstream
code cross-check that grounds these tasks). Read that first; this file is the *how
and in what order*, not the *what*.

## Principles

- **Vertical slices.** Each phase ends with something installable and testable on a
  real PVE 9 + Ceph cluster, not just merged code.
- **Extend, don't greenfield.** `pveceph` already recognises the `radosgw` service
  (`PVE/Ceph/Services.pm:117`) and `rgw` pool application
  (`PVE/API2/Ceph/Pool.pm:308,824`) — build on it.
- **Upstreamable from day one.** Every patch is DCO-signed, scoped, and shaped as a
  `git format-patch` series against the relevant fork (§5 of AGENTS.md). Assume each
  phase may be proposed to `pve-devel`.
- **v1 = RGW only.** The gateway's "PVE API backend" mode is explicitly deferred
  (see Phase 5+), per the §3 scope narrowing.
- **Decisions gate code.** The open items in AGENTS.md §§7–9 block specific phases;
  each is called out as a **Gate** below and must land as a short ADR first.

## Dependency overview

```
Phase 0  Foundations + Tenant ADR
   │
   ├──────────────┬───────────────────────────┐
   ▼              ▼                            ▼
Phase 1        Phase 3 (gateway, Mode A)    (parallelizable after P0)
RGW backend       │
   │              │
   ▼              │
Phase 2  ◄────────┘   (admin UI can consume backend + gateway status)
Admin UI
   │
   ▼
Phase 4  Tenant scope + native auth   ──►  Phase 5  OIDC/STS + Mode B/C HA
   │                                            │
   └───────────────► Phase 6  Hardening + upstreaming ◄──────────┘  (cross-cutting)
```

Phases 1 and 3 are independent after Phase 0 and can run in parallel.

---

## Phase 0 — Foundations & blocking decisions

**Objective:** a reproducible build/test loop and the one decision that unblocks all
user-facing work.

- **Gate (§7) — Tenant/identity ADR. ✅ Drafted:**
  [`docs/adr/0001-tenant-model.md`](adr/0001-tenant-model.md). Tenant = first-class
  object at ACL path `/objectstorage/<tenant>` ⇔ RGW tenant namespace; new `S3.*`
  privileges; membership via ACL group binding. Needs `pve-access-control` patch.
  Move to **Accepted** once reviewed; everything in Phases 2/4 depends on it.
- Stand up a **3-node PVE 9 (Trixie) test cluster** with Ceph (mon/mgr/osd) and the
  `download.proxmox.com/debian/devel` repo enabled.
- Establish the **fork → rebase workflow**: `upstream` remote on each
  `ciroiriarte/*` fork; document the rebase cadence; pick a working branch naming
  scheme (e.g. `objstor/<phase>`).
- **Build tooling**: follow `pve-common/README.dev`; get a no-op build of the two
  greenfield packages (`pve-ceph-radosgw`, `pve-service-gateway`) producing
  installable `.deb`s. Decide local-build vs OBS now.
- CI: lint (`eslint`, `perltidy`/Perl style), `dpkg-buildpackage`, DCO check.

**Exit criteria:** test cluster installs both empty packages cleanly; tenant ADR
merged; one fork demonstrably rebased on upstream and rebuilt.

**Risks:** Ceph version drift on the devel repo; CLA (Harmony) must be on file
before any upstream submission — start it now.

---

## Phase 1 — RGW daemon lifecycle backend  *(package: `pve-ceph-radosgw`)*

**Objective:** "Backend Automation" + "Stateless Processing" from AGENTS.md §2.1 —
create/destroy a working bare RGW (Beast) daemon from the CLI/API.

- New API module `PVE/API2/Ceph/RGW.pm` + register it in the Ceph API index.
- `pveceph rgw create <id>` / `destroy <id>` in `PVE/CLI/pveceph.pm`.
- Backend automation: Ceph client key generation, **realm / zonegroup / zone**
  init, `systemd` unit generation, Beast bound to local/private IP on **:7480**.
- **Pool placement** modelled correctly (§2.5): placement targets / storage classes,
  `index_pool`, `data_pool`, mandatory `data_extra_pool` for EC, CRUSH/EC selection;
  enforce placement immutability at creation.
- Idempotency + clean teardown (no orphaned pools/keys/units).

**Exit criteria:** `pveceph rgw create` yields a daemon serving S3 on :7480;
`aws-cli`/`boto3` can create a bucket and PUT/GET against it; `destroy` removes the
daemon and (optionally) pools; EC data pool works because `data_extra_pool` exists.

**Verify:** integration test driving create → S3 round-trip → destroy on the cluster.

---

## Phase 2 — Admin Web UI  *(fork: `pve-manager`)*

**Objective:** AGENTS.md §2.5 admin scope — **Datacenter → Ceph → Object Storage**.

- New panels under `www/manager6/ceph/` (alongside the existing 13 Ceph components):
  realm health, daemon distribution, per-zone status.
- GUI wrapper around `radosgw-admin`: provision S3 users, generate/rotate keys,
  apply basic quotas.
- **Advanced Storage Constraints** wizard exposing placement targets / storage
  classes (consume Phase 1 backend).
- Patch `www/manager6/Makefile` + API plumbing; never edit generated
  `pvemanagerlib.js`.

**Exit criteria:** an admin creates an RGW instance, a user, and keys entirely from
the UI; health/daemon distribution renders from live cluster state.

**Depends on:** Phase 1 backend.

---

## Phase 3 — Infrastructure / Service Gateway, Mode A HA  *(package: `pve-service-gateway`)*

**Objective:** AGENTS.md §2.2–2.4 — the HA, TLS-terminating S3 front door.
Runs in parallel with Phases 1–2 (only needs a reachable RGW to front, real or stub).

- `systemd`-managed **HAProxy per service**, **VRF-scoped** via `ip vrf exec`;
  **explicit IP binds only** — generator must refuse `*:80`/`*:443`.
- Datacenter-level UI panel adjacent to ACME/SDN (not an invented tree).
- **Certificates:** store key/cert under `/etc/pve/priv/<service>/` as plain files
  (no pmxcfs fork — verified §10); **reuse the existing ACME framework**
  (`PVE/API2/ACME*.pm`); DNS-01 preferred; **single cert-lead** renewal; graceful
  `systemctl reload` on renewal.
- **Mode A** HA: HAProxy + Keepalived (VRRP) failover — the default mode, mirroring
  cephadm's ingress topology.

**Exit criteria:** S3 reachable over TLS on a floating VIP; killing the active node
fails the VIP over with bounded interruption; a forced cert renewal reloads HAProxy
without dropping live connections.

**Gate (§8):** cert-lead election mechanism decided (ADR).

---

## Phase 4 — Tenant scope & native auth  *(forks: `pve-manager` + gateway)*

**Objective:** AGENTS.md §2.5 tenant scope + §2.6 **Mode A (Isolated Native Auth)**.

- Restricted tenant view: S3 endpoint URL, view/rotate access/secret keys, quota
  utilisation, optional basic bucket provisioning.
- Wire the **Phase 0 privileges** into PVE ACLs so views are permission-gated.
- `radosgw-admin user create` provisioning tied to the tenant (PVE group) model.
- **Key revocation** (immediate disable), not just rotation; quota-utilisation
  alerts; basic bucket **versioning/lifecycle** toggles.

**Exit criteria:** a non-admin user in an authorised PVE group self-serves keys and
sees only their quota/usage; revocation immediately blocks S3 access.

**Depends on:** Phase 1 (provisioning) + Phase 0 ADR.

---

## Phase 5 — Federation & advanced HA  *(forks: gateway + `pve-network` aware)*

**Objective:** AGENTS.md §2.6 **Mode B (OIDC/STS)** and §2.3 **Modes B/C**.

- OIDC/STS: create RGW **OIDC provider object**, **IAM roles** + trust policies,
  `AssumeRoleWithWebIdentity` via the STS endpoint; document the Keycloak path and
  how PVE groups map to roles.
- **Mode B** all-active DNS round-robin (ship with the documented health-check
  caveat).
- **Mode C** BGP/ECMP anycast: default to a **dedicated BGP speaker**
  (`bird`/`gobgp`/ExaBGP) for isolation; offer SDN integration via the
  `/etc/frr/frr.conf.local` hook (`Frr.pm:35`) when SDN is present. Opt-in only.

**Exit criteria:** JWT→temporary-S3-credential flow works end to end; anycast VIP
withdraws on local RGW failure on a routed test fabric.

**Depends on:** Phases 3 + 4.

---

## Phase 6 — Operational hardening & upstreaming  *(cross-cutting)*

**Objective:** AGENTS.md §§8–9 plus the packaging/distribution strategy of §3.

- **Observability:** RGW health/latency/5xx + HAProxy stats → PVE Metric Server;
  local-only HAProxy `/metrics`.
- **Metadata backup/restore:** export realm/zone/zonegroup/user/key config.
- **Upgrade path** across Ceph majors; generated **Beast tuning**.
- **Failure modes (§9):** RGW-crash backend drain, VRF-misconfig recovery,
  cert-renewal-failure tolerance (keep serving old cert).
- **Packaging:** finalise OBS pipeline, **continuous rebase** on each upstream
  release, **narrow per-package APT pins** (never `Pin-Priority > 1000`).
- **Upstream submission:** `git format-patch -s` series to `pve-devel`; CLA on file.

**Exit criteria:** monitoring dashboards live; documented DR for RGW metadata; clean
upgrade across one Ceph point release; patch series posted to `pve-devel`.

---

## Cross-cutting (every phase)

- DCO `git commit -s` as `Ciro Iriarte <ciro.iriarte+software@gmail.com>`.
- Keep `CHANGELOG.md` current; minimal reviewable patches against forks.
- Integration tests run on the Phase 0 cluster; no "works on my node" merges.
- Security review on anything touching auth, keys, TLS, or VRF isolation.
