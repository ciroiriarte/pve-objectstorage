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
- New UI panel under **Datacenter → System → Network**.
- Decoupled from tenant-focused SDN overlays to prevent accidental disruptions.
- **Underlay VRF execution:** proxies rely on native Linux VRFs via `ifupdown2`;
  HAProxy runs inside isolated namespaces using `ip vrf exec`.
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
  from the A-record pool.

### 2.4 Cluster-centric service certificates
- **Global storage (`pmxcfs`):** service certificates stored in the replicated
  cluster filesystem at `/etc/pve/priv/custom-certs/`.
- **DNS-01 enforcement:** UI strictly enforces existing DNS-01 ACME plugins
  (HTTP-01 cannot reliably route through floating VIPs or round-robin DNS).
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
  separation to avoid IOPS contention —
  - *Index Pool Rule:* CRUSH rule for `.rgw.buckets.index` (best practice: fast
    NVMe/SSD rules for metadata queries).
  - *Data Pool Rule / Profile:* CRUSH rule for `.rgw.buckets.data` (e.g. `hdd-only`)
    or an Erasure Coded profile (e.g. `ec-4-2-profile`).

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
- **Mode B — Identity Federation (OIDC/STS):** RGW daemon trusts an external OIDC
  provider (e.g. Keycloak). Clients trade a JWT for temporary S3 credentials via
  the STS endpoint — enterprise SSO without putting PVE in the data path.

## 3. Packaging, patching & distribution strategy

This cannot be 100% greenfield due to Proxmox's monolithic API/UI structures.

- **Greenfield packages:** two net-new `.deb` packages —
  - `pve-ceph-radosgw` — backend Perl logic, dependency mapping, `systemd` automation.
  - `pve-infra-proxy` — Infrastructure Proxy backend logic and `systemd` automation.
- **Stock overrides (forks/patches):**
  - `pve-manager`: patch to inject new API routing endpoints into `Ceph.pm` and new
    UI views into `pvemanagerlib.js`.
  - `pve-cluster`: patch the `pmxcfs` C source (`status.c`, `memdb.c`) to whitelist
    the new `/etc/pve/priv/custom-certs/` directory.
- **Versioning & safety:** OBS builds use the `+` symbol in version strings (e.g.
  `9.0.1-1+rgw1`) to reliably override stock Proxmox packages. Test environments
  must use APT pinning (`Pin-Priority: 1001`) to prevent upstream security updates
  from blindly overwriting custom packages before the OBS pipeline can sync.

## 4. Upstream sources (git.proxmox.com)

Mirror/reference these upstream repos; do **not** vendor blindly — track patches.

| Upstream repo | Clone URL | Relevance |
|---|---|---|
| `pve-manager` | `https://git.proxmox.com/git/pve-manager.git` | API (`PVE/API2/Ceph/`, `Ceph.pm`), CLI (`pveceph`), Web UI (`pvemanagerlib.js`) |
| `pve-cluster` | `https://git.proxmox.com/git/pve-cluster.git` | `pmxcfs` cluster FS (`status.c`, `memdb.c`), `/etc/pve` |
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
