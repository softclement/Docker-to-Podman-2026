# Docker Isn't Dead. So Why Are More Teams Looking at Podman?

> Understanding the 2026 shift toward daemonless and rootless container management — and what it means for database workloads.

## The Analogy: One Building Superintendent vs. Individual Room Keys

Docker's traditional architecture works like an apartment building with a **building superintendent** (the Docker daemon, `dockerd`) who holds a master key. Every tenant (container) goes through the super to unlock their door, get water, adjust the thermostat. It's convenient, centralized, and battle-tested — and Docker does offer rootless mode and other hardening options if you configure them.

Podman skips the superintendent entirely. There is **no daemon** — each tenant gets their own key that only opens their own door. Containers can run rootless under the invoking user, and this is Podman's default model rather than an opt-in add-on.

The more accurate framing:

> Docker's traditional architecture relies on a privileged daemon by default. Podman is daemonless by design, with rootless containers as a first-class capability. Podman can still run rootful when needed.

## Is Docker Being Replaced?

Not really. The two solve the same problem with different architectural trade-offs, and each has genuine strengths.

| Point | Detail |
|---|---|
| Kubernetes runtime | Kubernetes removed the built-in `dockershim` integration in v1.24, shifting toward CRI-compatible runtimes like containerd and CRI-O. Docker images still run fine in Kubernetes — the image format and the runtime are separate concerns. |
| Distro integration | Podman is strongly integrated into the RHEL/Fedora ecosystem and is also available across major Linux distributions. |
| Licensing | Docker Desktop introduced subscription requirements for certain larger organizations. Podman offers an open-source alternative without that commercial licensing model. This is a Docker *Desktop* distinction — Docker Engine is a separate consideration. |
| Compliance | In regulated industries (finance, healthcare, government), audits increasingly ask why a container runtime needs root by default. Podman's architecture addresses that out of the box. |

## Who's Exploring This Migration

- Enterprises standardizing on RHEL/Fedora, where Podman is the default runtime
- Platform engineering teams reassessing Docker Desktop license costs at scale
- DevOps/CI-CD teams tightening security posture ahead of audits
- Self-hosters and homelab users who want systemd-native container management without daemon overhead

## What Actually Changes When You Migrate

CLI commands map almost 1:1:

```bash
docker ps        →  podman ps
docker images    →  podman images
docker run ...   →  podman run ...
docker save ...  →  podman load ...   # or: skopeo copy between them
```

```text
Docker                          Podman
  |                               |
  |-- docker images               |-- podman images
  |-- docker run                  |-- podman run
  |-- docker compose              |-- podman-compose / Quadlet
  |-- Docker volumes              |-- Podman volumes
  `-- dockerd (root daemon)       `-- daemonless (rootless by default)
```

What does **not** translate 1:1 — and is where most real migration effort goes:

- Rootless UID/GID mapping
- Volume ownership and permissions
- Privileged port binding (ports below 1024)
- Networking (Netavark/Aardvark-DNS vs. Docker's bridge network)
- Docker Compose → Podman Quadlet (systemd units)
- `docker.sock` dependencies in CI/CD pipelines

## The DBA Angle: What About Database Containers?

Most Docker-vs-Podman comparisons stop at the CLI and the daemon model. For a DBA, the interesting question starts one layer down: what happens to the database itself?

Take a PostgreSQL container currently running under Docker. Migrated to rootless Podman, PostgreSQL itself mostly should not care — it is still just a process reading and writing to a data directory. But a few areas need hands-on verification rather than assumption.

**What I expect to investigate:**

- Volume ownership and UID/GID mapping between Docker's root-mapped storage and rootless Podman
- Port binding behavior (PostgreSQL's default 5432 is unprivileged, but worth confirming end to end)
- Backup and restore continuity — whether `pg_dump`/`pg_restore` workflows built around `docker exec` carry over cleanly to `podman exec`
- Container lifecycle and restart behavior when converting a `--restart=always` Docker setup to a Podman Quadlet systemd unit

This is a checklist of unknowns, not a set of conclusions. The PoC below will replace each of these with what was actually observed.

## For Freshers

You do not need to pick a side today. But being comfortable with both `docker` and `podman` CLIs, and understanding why daemonless/rootless architecture matters, is becoming a baseline skill rather than a nice-to-have.


## Closing Thought

Docker and Podman are not really competing for the same crown — they are making different architectural trade-offs. Understanding those trade-offs matters more than picking a winner.

---

*Feedback and real-world migration experiences welcome — open an issue or connect on [LinkedIn](https://linkedin.com/in/mariyanclement/).*
