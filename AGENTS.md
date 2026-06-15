# pve-objectstorage — Agent & Contributor Guide

> Single source of truth for humans and AI agents working in this repository.
> `CLAUDE.md` is a symlink to this file.

## 1. What this project is

`pve-objectstorage` implements **native S3 Object Storage support in Proxmox VE 9.x**
by integrating the **Ceph RADOS Gateway (RGW / S3)** and a new **Unified
Infrastructure Proxy** subsystem directly into the Proxmox management stack.

The goal: enterprise-grade, highly-available S3 object storage and API access
managed directly from the Proxmox UI, using the *native* `pveceph` stack, native
Linux VRFs, and `systemd`-managed HAProxy — **without** Kubernetes, MetalLB, or
ad-hoc scripts.

### Maintainer / ownership

- **Maintainer:** Ciro Iriarte (GitHub [`ciroiriarte`](https://github.com/ciroiriarte))
- **Contact / commit identity:** `ciro.iriarte+software@gmail.com`
- **Account email:** `ciro.iriarte@gmail.com`
- All commits, changelog entries, `Signed-off-by` trailers and authorship MUST use
  `Ciro Iriarte <ciro.iriarte+software@gmail.com>`.

## 2. Feature scope (from the proposal)

### 2.1 RadosGW native service management (`pveceph` integration)
- RGW daemons run natively on the hypervisor as bare-metal `systemd` services using
  the embedded **Beast** web server.
- **Backend automation:** Proxmox handles Ceph client key generation,
  realm/zonegroup initialization, and `systemd` unit file generation.
- **Stateless processing:** Bare RGW (Beast) daemons bind to local/private IPs
  (port **7480**) and act purely as stateless HTTP→Ceph translators; all stateful
  TCP/TLS processing is offloaded to the Infrastructure Proxy.

### 2.2 Unified Infrastructure Proxy (decoupled from SDN)
- New UI panel at the **Datacenter** level, adjacent to the existing ACME / SDN
  configuration (not an invented `System → Network` sub-tree — that does not match
  current PVE information architecture).
- Decoupled from tenant-focused SDN overlays to prevent accidental disruptions.
- **VRF-scoped execution:** proxies rely on native Linux VRFs via `ifupdown2`;
  HAProxy runs in a **VRF context** using `ip vrf exec`. Note: `ip vrf exec` binds
  sockets into a VRF via `SO_BINDTODEVICE`-style mechanisms — it is **not** a
  network namespace, and has different semantics for listening sockets and DNS
  resolution. Do not describe it as namespace isolation.
- **Supported backends:** Ceph RadosGW (object storage) or the Proxmox VE API
  (management).
- **Port conflict resolution:** HAProxy MUST NEVER use wildcard binds (`*:80` /
  `*:443`); it binds to explicitly defined IPs. Multiple proxies may share a single
  VRF (e.g. `vrf-public`) provided the UI enforces distinct IP addresses when
  sharing a port.
- **Microservice instancing:** Proxmox spawns separate lightweight `systemd`
  workers per service for strict blast-radius isolation.

### 2.3 High availability & routing architecture
- **Mode A — Single Active Ingress (failover):** prioritizes connection stability.
  HAProxy + Keepalived deployed via PVE; HAProxy terminates SSL and proxies to
  local RGW/API.
- **Mode B — All-Active Load Sharing:** maximizes throughput. External DNS
  round-robin resolves the S3/API endpoint to multiple PVE node IPs where HAProxy
  instances listen locally; requires external DNS health checks to drop dead nodes
  from the A-record pool. ⚠️ Caveat: most DNS servers (e.g. stock BIND) do **not**
  health-check A-records, so dead nodes keep serving failures. Treat this mode as
  best-effort unless paired with a health-aware DNS/global LB.
- **Mode C — Anycast (advanced, opt-in):** announce a single S3 VIP from every RGW
  node via BGP/EVPN (PVE SDN already ships FRR). More robust than DNS-RR and faster
  to converge than VRRP failover. ⚠️ This **reintroduces an SDN/FRR dependency**,
  which conflicts with the "decoupled from SDN" principle above, so it must remain
  explicitly opt-in for sites already running a routed fabric — never the default.

Topology note: Mode A (HAProxy + Keepalived) mirrors Ceph's own `cephadm` RGW
ingress service by design. Copy that proven ingress configuration logic; do **not**
adopt `cephadm` as a second control plane beside `pveceph`.

### 2.4 Cluster-centric service certificates
- **Global storage (`pmxcfs`):** service certificate/key material stored in the
  replicated cluster filesystem under `/etc/pve/priv/<service>/`, using **plain
  file operations** — the same pattern PVE already uses for storage secrets
  (`/etc/pve/priv/storage/<id>.pw`). This needs **no `pmxcfs` C patch**: arbitrary
  private subpaths under `/etc/pve/priv/` are already permitted. (A `cfs_register_file`
  observed-file entry is only needed for *structured* `cfs_read_file/cfs_write_file`
  config, which raw cert blobs do not require.)
- **Reuse existing ACME plumbing:** integrate with the existing `pve-manager` ACME
  account/plugin framework and surface these service certs in the existing
  Datacenter **Certificates** UI rather than inventing a parallel cert system.
- **DNS-01 preferred:** the UI recommends/defaults to existing DNS-01 ACME plugins
  for shared/wildcard/floating endpoints, where they avoid ingress routing tricks.
  HTTP-01 is **not impossible** (PVE ships a standalone plugin, and HAProxy can
  route `/.well-known/acme-challenge/`), so it remains a supported fallback — do not
  claim HTTP-01 "cannot route."
- **Single cert-lead renewal:** elect one node as renewal lead (or renew once then
  propagate via `pmxcfs`) to avoid all nodes hitting ACME/CA rate limits or DNS
  provider locks simultaneously.
- **Automated soft reloads:** a cron renewal triggers `systemctl reload` on mapped
  HAProxy instances, updating certs without dropping active TCP connections.

### 2.5 UI/CLI scope boundaries (admin vs tenant)
Proxmox acts as the infrastructure **control plane**, not an object-storage file browser.

**Administrator scope:**
- CLI (`pveceph`): `pveceph rgw create <id>` / `pveceph rgw destroy <id>` manage
  daemon lifecycles.
- Web UI: **Datacenter → Ceph → Object Storage** shows global realm health and
  daemon distribution; GUI wrapper around `radosgw-admin` to provision S3 users,
  generate keys, apply basic quotas.
- **Advanced Storage Constraints** (creation wizard): allows physical pool
  separation to avoid IOPS contention. This must be modelled as Ceph **placement
  targets / storage classes** on the zone/zonegroup — *not* as two hardcoded CRUSH
  dropdowns — because RGW bucket placement is defined there and is **immutable after
  bucket creation**:
  - *Index pool:* CRUSH rule for the bucket index pool (best practice: fast
    NVMe/SSD rules for metadata/omap queries).
  - *Data pool:* CRUSH rule (e.g. `hdd-only`) or Erasure Coded profile (e.g.
    `ec-4-2-profile`) for object data.
  - *Data-extra pool:* a **replicated** pool for multipart-upload metadata and omap.
    This is **mandatory when the data pool is EC** (EC cannot hold omap), and the
    wizard must allocate it automatically — a common omission that breaks EC RGW.
  - The wizard should expose named **placement targets** and **storage classes**
    (e.g. `STANDARD`, `COLD`) rather than implying a single fixed pool layout.

**End-user / tenant scope:**
- Proxmox Web UI (tenant view): highly restricted — view S3 endpoint URL,
  view/rotate access/secret keys, view quota utilization, optionally provision
  basic buckets.
- External S3 CLI: all object-level data manipulation (PUT/GET/DELETE, multipart,
  ACLs, CORS) is delegated to standard S3 clients (AWS CLI, boto3).

### 2.6 Tenant authentication modes
PVE session-based auth cannot fulfill S3 SigV4. Two control-plane provisioning modes:
- **Mode A — Isolated Native Auth (default):** hypervisor is air-gapped from tenant
  data. Proxmox runs `radosgw-admin user create` to generate standalone S3
  access/secret keys, separate from PVE login credentials.
- **Mode B — Identity Federation (OIDC/STS):** clients trade a JWT for temporary S3
  credentials via RGW's STS endpoint (`AssumeRoleWithWebIdentity`) — enterprise SSO
  without putting PVE in the data path. This is more than "RGW trusts an OIDC
  provider"; the control plane must provision: an **RGW OIDC provider object** for
  the issuer (e.g. Keycloak), **IAM roles** with trust policies referencing the
  provider, and the per-role **permission policies**. Spec must define how PVE maps
  its tenant/group model onto these roles.

## 3. Packaging, patching & distribution strategy

This cannot be 100% greenfield due to Proxmox's monolithic API/UI structures.

- **Greenfield packages:** two net-new `.deb` packages —
  - `pve-ceph-radosgw` — backend Perl logic, dependency mapping, `systemd` automation.
  - `pve-service-gateway` (working name; `pve-infra-proxy` is too generic) —
    Infrastructure Proxy backend logic and `systemd` automation.
- **Stock overrides (forks/patches) — real surfaces:**
  - `pve-manager`: add API endpoints under `PVE/API2/...`, RGW lifecycle CLI in
    **`PVE/CLI/pveceph.pm`** (not only `Ceph.pm`), and UI views in the modular
    **`www/manager6/*.js`** tree (**not** `pvemanagerlib.js`, which is a generated
    concatenation), plus the corresponding `Makefile` entries.
  - `pve-cluster`: **no C patch expected.** Cert/key blobs live under
    `/etc/pve/priv/<service>/` via plain file ops (see §2.4). Only *if* a structured
    observed config file is genuinely required, patch the observed-file registries
    (`PVE/Cluster.pm` + `pmxcfs/status.c`) — **never `memdb.c`**, which is the
    in-memory DB, not the whitelist.
- **Versioning & safety:** the `+` suffix does **not** reliably win — `9.0.1-1+rgw1`
  sorts *before* the next upstream `9.0.2-1`, so stock updates will overtake it.
  The only durable strategy is to **continuously rebase the downstream packages onto
  each upstream release**. Use **narrow, per-package APT pins** targeting only the
  affected packages from the downstream repo. Avoid `Pin-Priority > 1000`, which APT
  treats as "allow downgrades" and is actively dangerous; a pin in the
  `500 < priority ≤ 1000` range is sufficient to prefer the downstream build without
  enabling downgrades.

## 4. Upstream sources (git.proxmox.com)

Mirror/reference these upstream repos; do **not** vendor blindly — track patches.

| Upstream repo | Clone URL | Relevance |
|---|---|---|
| `pve-manager` | `https://git.proxmox.com/git/pve-manager.git` | API (`PVE/API2/...`), CLI (`PVE/CLI/pveceph.pm`), modular Web UI (`www/manager6/*.js`) |
| `pve-cluster` | `https://git.proxmox.com/git/pve-cluster.git` | `pmxcfs` cluster FS, `/etc/pve` secret storage (`/etc/pve/priv/...`); observed-file registry in `PVE/Cluster.pm` + `status.c` (not `memdb.c`) |
| `ceph` | `https://git.proxmox.com/git/ceph.git` | Proxmox Ceph packaging (RGW/Beast, `radosgw-admin`) |
| `pve-storage` | `https://git.proxmox.com/git/pve-storage.git` | Storage plugin library |
| `librados2-perl` | `https://git.proxmox.com/git/librados2-perl.git` | Perl bindings for librados |
| `pve-common` | `https://git.proxmox.com/git/pve-common.git` | `README.dev` build instructions, shared tooling |

Read-only clone form: `git clone git://git.proxmox.com/git/<repo>.git`

## 5. Development guidelines (from pve.proxmox.com/wiki/Developer_Documentation)

### 5.1 Coding standards
- **Perl:** follow the Proxmox Perl Style Guide.
- **JavaScript:** ExtJS **7.0.0** for UI; lint with `eslint`.
- **Rust:** `cargo fmt` (default rustfmt); avoid compiler warnings; do **not**
  mass-fix `cargo clippy` lints.
- **C:** matches existing `pmxcfs` style when patching `pve-cluster`.
- **Docs:** line length < 100 chars (80 preferred); follow the Technical Writing
  Style Guide.

### 5.2 Commit message conventions
- Imperative mood ("Fix bug", not "Fixed bug").
- Lines ≤ 72 chars (URLs and git trailers exempt).
- Bug-fix commits start with `fix #1234: summary` (bugzilla.proxmox.com).
- Subsystem tags where appropriate, e.g. `ui: ceph: add object storage panel`.
- Always sign off: `git commit -s` with real name and email
  (`Ciro Iriarte <ciro.iriarte+software@gmail.com>`).
- Causally-ordered trailers (`Suggested-by` before `Signed-off-by`,
  `Tested-by` after).

### 5.3 Submitting patches upstream
- Discuss on the **pve-devel** mailing list: `pve-devel@lists.proxmox.com`.
- Generate patches:
  ```
  git format-patch -s -o my-patches/ \
    --subject-prefix="PATCH [subsystem]" master..my_branch --cover-letter
  ```
- Submit **only** via `git send-email` (preserves formatting).
- Updated series: resend the whole series with `-v2`/`-v3`, document changes in the
  cover letter and commit summaries.
- Archive: https://lists.proxmox.com/pipermail/pve-devel/ · Public inbox:
  lore.proxmox.com · Bug tracker: https://bugzilla.proxmox.com

### 5.4 Contributor requirements
- A signed **Harmony CLA** (individual or entity) must reach `office@proxmox.com`
  before code is accepted upstream.
- Only code under the repo's license (primarily **AGPLv3**) is included — this repo
  is licensed AGPLv3 to match.

### 5.5 Dev environment
- PVE 9 (Debian 13 / Trixie): add a DEB822 `proxmox.sources` file pointing to
  `http://download.proxmox.com/debian/devel/` suite `trixie`.
- PVE 8 (Debian 12): `deb http://download.proxmox.com/debian/devel/ bookworm main`.
- Build instructions live in `pve-common/README.dev`.

## 6. Repository conventions for agents

- **Identity:** never commit as any other name/email than
  `Ciro Iriarte <ciro.iriarte+software@gmail.com>`. The repo's local git config is
  already set; do not override it.
- **Always** `git commit -s` (sign-off / DCO is mandatory for Proxmox).
- Keep `CHANGELOG.md` updated using the [Keep a Changelog](https://keepachangelog.com)
  format; attribute the project to the `+software` identity.
- Respect upstream patch hygiene: when modifying forked `pve-manager`/`pve-cluster`
  code, keep changes as minimal, reviewable patches (think `git format-patch`), not
  rewrites.
- Do not introduce Kubernetes/MetalLB or heavy orchestrators — native `systemd` +
  VRF + HAProxy only, per the design.
- HAProxy config generators MUST refuse wildcard binds (`*:80`/`*:443`).
- This file (`AGENTS.md`) is authoritative; `CLAUDE.md` symlinks to it. Update both
  by editing `AGENTS.md` only.
- **v1 scope is RGW only.** Defer fronting the PVE API through the same gateway —
  it doubles the security blast radius for little functional gain.

## 7. Tenant & identity model (open — must be specified before build)

PVE has **no native "tenant" object**, so the admin/tenant split in §2.5–2.6 is
undefined until this is pinned down:
- **Map a tenant → PVE Group** (most flexible: multiple users share one S3
  identity, quota, and key set). Alternatives — Realm or single User — are weaker.
- Define a new PVE privilege set, e.g. `S3.Allocate`, `S3.Audit`, `S3.KeyManage`,
  applied via the standard ACL system, so tenant views are permission-gated rather
  than a bolt-on portal.
- The OIDC/STS mode (§2.6) must define how this group model maps onto RGW IAM roles.

Required tenant workflows the current spec omits:
- **Key revocation** (immediate disable on leak), not just rotation.
- **Bucket lifecycle**: versioning + lifecycle (transition/expiration) toggles, or
  the tenant scope is too thin and forces users back to the CLI for basic hygiene.
- **Quota-utilization alerts** and a per-tenant usage view; admin "top consumers".

## 8. Operational concerns (open — must be specified before build)

- **Observability:** surface RGW health / latency / 5xx and HAProxy stats. Feed the
  existing PVE **Metric Server** (InfluxDB/Graphite); optionally expose a local-only
  HAProxy `/metrics` endpoint. The spec is currently silent on monitoring.
- **Metadata backup/restore:** object *data* lives in Ceph, but the **realm / zone /
  zonegroup / user / key** metadata needs an explicit export+restore plan — losing
  it makes the key↔user mapping unrecoverable even with intact data pools.
- **Upgrade path:** define how `pve-ceph-radosgw` handles Ceph major-version
  transitions and RGW config-schema changes across PVE 9.x point releases.
- **Beast tuning:** generate sane thread/buffer settings for high-concurrency S3
  rather than shipping Beast defaults.

## 9. Failure modes to design for (open)

- **RGW daemon crash** → HAProxy health-check must drain that backend; define check
  type (S3 `GET /` vs. TCP) and thresholds.
- **VRF misconfiguration** → detection and safe recovery; never leave a half-bound
  listener.
- **Cert renewal failure** → alert + keep serving the old cert until it actually
  expires; never hard-fail the listener on a renewal error.
- **All-active split assumptions** → see §2.3 Mode B caveat; prefer Mode C anycast
  where real health-based withdrawal is required.

---

> **Review status:** §§2–4 above were fact-checked on 2026-06-14 against the current
> upstream `pve-manager` / `pve-cluster` / `pve-storage` trees and official Ceph,
> Proxmox, Debian and Linux-VRF docs (tri-model review). §§7–9 capture open design
> decisions surfaced by that review and must be resolved before implementation.
