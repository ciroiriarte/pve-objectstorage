# pve-objectstorage

Native **S3 Object Storage** support for **Proxmox VE 9.x** — integrating the Ceph
RADOS Gateway (RGW / S3) and a new Unified Infrastructure Proxy directly into the
Proxmox management stack, "the Proxmox Way": native `pveceph`, Linux VRFs, and
`systemd`-managed HAProxy. No Kubernetes, no MetalLB, no hacky scripts.

## Highlights

- **Native RadosGW management** via `pveceph rgw create|destroy <id>` (Beast, port 7480).
- **Unified Infrastructure Proxy** (Datacenter → System → Network) for SSL
  termination, buffering and port management, executed inside Linux VRFs.
- **High availability:** Single Active Ingress (HAProxy + Keepalived) or All-Active
  load sharing via external DNS round-robin with health checks.
- **Cluster-centric certificates** in `pmxcfs` (`/etc/pve/priv/custom-certs/`) with
  DNS-01 ACME enforcement and graceful soft reloads.
- **Admin & tenant scopes** with a GUI wrapper around `radosgw-admin`.
- **Tenant auth:** Isolated Native Auth (default) or OIDC/STS Identity Federation.

## Packaging

- Greenfield `.deb`s: `pve-ceph-radosgw`, `pve-infra-proxy`.
- Patches against `pve-manager` (`Ceph.pm`, `pvemanagerlib.js`) and `pve-cluster`
  (`pmxcfs`: `status.c`, `memdb.c`).
- OBS version strings use `+` overrides (e.g. `9.0.1-1+rgw1`) with APT pinning
  `Pin-Priority: 1001`.

## Documentation

See **[AGENTS.md](AGENTS.md)** for the full design, upstream sources, and the
Proxmox developer guidelines this project follows, and
**[docs/ROADMAP.md](docs/ROADMAP.md)** for the phased implementation plan.

## License

[AGPLv3](LICENSE) — matching Proxmox VE upstream.

## Maintainer

Ciro Iriarte — `ciro.iriarte+software@gmail.com` —
[github.com/ciroiriarte](https://github.com/ciroiriarte)
