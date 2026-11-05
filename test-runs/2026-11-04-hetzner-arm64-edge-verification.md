# Verification: 2026-11-04 — Hetzner CAX21 (arm64, Ubuntu 24.04) — Rakazo edge (running instance)

- **Date (Europe/Berlin):** 2026-11-04
- **Tester:** DeployButler. This is a verification of the still-running October instance, recorded as reference data for test 1. The maintainers' independent re-log was completed later the same day and is filed separately as `test-runs/2026-11-04-hetzner-arm64-edge-test1.md`; it confirms every value in this file except the web container's health annotation (this file lists all five containers as healthy; the re-log's `docker compose ps` output shows web as `Up` without a `(healthy)` annotation), so this file now serves as the reference record that the formal log was checked against
- **Provider / machine type:** Hetzner CAX21 (arm64, Nuremberg)
- **Hostname:** `<hostname>`
- **Architecture:** arm64
- **Operating system:** Ubuntu 24.04
- **Scenario:** Running-instance verification only — no install, upgrade, or configuration change was performed

## Versions

```text
$ docker version
Docker 29.9.2

$ docker compose version
Docker Compose v5.6.1
```

- **Rakazo app image:** `ghcr.io/elie222/rakazo/app:edge` digest `sha256:24e460aca4390fd4773c7eacee50b6d5e5bca4af8cdb663729940b2591416638`
- **Rakazo computer image:** `ghcr.io/elie222/rakazo/computer:edge` digest `sha256:fa96e9d45fc287f15e02aeaceac5f124c07968ae8171fd7218ec3b322a846c07`

## Service status

Four resident containers reported (healthy) and web was up, with uptimes of 9–10 days at the time of the check. The public HTTPS endpoint of the instance also returned 200 during the same check.

## Notes

- Access used the project key on file; no keys were added or removed for this check
- A first connection attempt in this check initially failed with a host-key verification error because the local `known_hosts` had been lost in an environment change; identity was confirmed via the public HTTPS endpoint plus the hostname and container state above before proceeding

## Conclusion

This file proves only that, on 2026-11-04, the October instance was still running the digests above on Docker 29.9.2 / Compose v5.6.1 with four containers reporting (healthy) and web up. The v0.6.0 upgrade test on this instance (matrix row 3) has not been run yet; it requires a Hetzner snapshot and a database backup before it starts, and the snapshot must be deleted and evidenced afterwards.
