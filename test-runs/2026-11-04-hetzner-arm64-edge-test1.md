# Test run: 2026-11-04 — Hetzner CAX21 (arm64, Ubuntu 24.04) — Rakazo edge (test 1: version completion)

- **Date (Europe/Berlin):** 2026-11-04, about 10:12 CET
- **Tester:** DeployButler
- **Provider / machine type:** Hetzner CAX21 (4 vCPU / 8 GB RAM / 80 GB, Nuremberg) — the existing October instance, hostname `<hostname>`
- **Architecture:** arm64 (aarch64)
- **Operating system:** Ubuntu 24.04.4 LTS
- **Scenario:** Test 1 from the field-guide test plan: complete the October run's version record for the still-running instance — image tags and digests, Docker and Compose versions, service status. **Read-only: no configuration was changed, no container was restarted, nothing was pulled or upgraded.** Access used the SSH key that was already authorised on this machine; no keys were added or removed.

## Versions (actual command output)

```text
$ hostname; uname -m; grep PRETTY_NAME /etc/os-release; uptime
<hostname>
aarch64
PRETTY_NAME="Ubuntu 24.04.4 LTS"
up 10 days, 18:34 (at check time), load average: 0.22, 0.20, 0.17

$ docker version
Client: 29.9.2  Server: 29.9.2

$ docker compose version
Docker Compose version v5.6.1
```

- **Rakazo app image:** `ghcr.io/elie222/rakazo/app:edge` digest `sha256:24e460aca4390fd4773c7eacee50b6d5e5bca4af8cdb663729940b2591416638` (image size 5.59 GB, created 9 days before this check)
- **Rakazo computer image:** `ghcr.io/elie222/rakazo/computer:edge` digest `sha256:fa96e9d45fc287f15e02aeaceac5f124c07968ae8171fd7218ec3b322a846c07` (image size 2.38 GB, created 9 days before this check)
- **How the digests were read:** `docker images --digests` and `docker image inspect ... --format '{{json .RepoDigests}}'`; both methods agree

## Commands run

```bash
# All commands were read-only inspection commands over SSH:
hostname; uname -m; cat /etc/os-release; uptime; nproc
docker version; docker compose version
docker compose -f docker-compose.images.yml ps
docker images --digests ghcr.io/elie222/rakazo/app
docker images --digests ghcr.io/elie222/rakazo/computer
docker image inspect ghcr.io/elie222/rakazo/app:edge
docker image inspect ghcr.io/elie222/rakazo/computer:edge
curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:5173
free -h; df -h /
```

## Timing and resources

- **Install duration:** not applicable (no install in this run)
- **Resource snapshot at check time:** memory 2.1 GiB used of 7.5 GiB total; root disk 17 GB used of 75 GB (24%)
- **Notes:** These are point-in-time readings of a live instance with one bot, not idle-after-install figures; the idle figures from the original install are in the 2026-10-24 log

## Service status (sanitised, verbatim status strings)

```text
$ docker compose -f docker-compose.images.yml ps
api         ghcr.io/elie222/rakazo/app:edge    Up 9 days (healthy)
postgres    postgres:16                        Up 10 days (healthy)
supervisor  ghcr.io/elie222/rakazo/app:edge    Up 9 days (healthy)
web         ghcr.io/elie222/rakazo/app:edge    Up 9 days
worker      ghcr.io/elie222/rakazo/app:edge    Up 9 days (healthy)
```

Four services report `(healthy)` explicitly; the web service reports `Up` with no health annotation in its status string, exactly as shown. The web app answered `200` on `http://127.0.0.1:5173` during the same check.

## Smoke test

- **Test performed:** Local HTTP check only (see above). No bot conversation was run in this read-only version-completion pass; the end-to-end smoke test for this instance is recorded in the 2026-10-24 log and its 2026-10-25 upgrade follow-up
- **Result:** Web endpoint 200; all other results as recorded above

## Errors encountered

- None on the server. (Locally, the first SSH attempt used the wrong default identity path and was rejected; retrying with the explicit authorised identity file succeeded. No server-side change was involved or made.)

## Backup / upgrade / rollback (if applicable)

- Not applicable to this run. Database row counts were deliberately not captured here: they are the baseline for the upgrade test (matrix row 3) and will be taken together with that test's mandatory pre-upgrade backup, using the schema current at that time.

## Conclusion

This run completes the version record for the October instance and confirms the maintainers' 2026-11-04 reference data, with one annotation difference: the reference record lists all five containers as healthy, while `docker compose ps` in this run shows web as `Up` without a `(healthy)` annotation. Everything else matches: same hostname, same Docker 29.9.2 / Compose v5.6.1, and identical app and computer `edge` digests. It proves nothing about v0.6.0; upgrading this instance to v0.6.0 is matrix row 3 and has not been run yet.
