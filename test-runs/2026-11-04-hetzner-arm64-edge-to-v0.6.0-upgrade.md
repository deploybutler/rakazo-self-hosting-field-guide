# Test run: 2026-11-04 — Hetzner CAX21 (arm64, Ubuntu 24.04) — upgrade edge -> v0.6.0 and rollback (test 3)

- **Date (Europe/Berlin):** 2026-11-04
- **Tester:** DeployButler
- **Machine:** the long-running October instance (Hetzner CAX21, arm64, Ubuntu 24.04.4 LTS, Docker 29.9.2 / Compose v5.6.1 — the versions recorded in test 1; the OS and Docker were not changed by this test)
- **Scenario:** Upgrade the existing October `edge` installation, with its real data, to the v0.6.0 published images; verify data and service; then roll back and verify the instance is back in its pre-upgrade state.
- **Safety nets in place before starting:** a Hetzner snapshot of the whole machine (created by the operator in the console the same evening, 7.58 GB, status available — it was **not** needed and never restored), plus the fresh database backup below.

## Core findings

1. **The upgrade itself is uneventful**: one installer run (132 seconds), zero data change.
2. **On a rolling channel, "roll back by tag" is not "roll back to the same build."** Measured evidence, as a pair:
   - Pre-upgrade baseline: app `sha256:24e460aca4390fd4773c7eacee50b6d5e5bca4af8cdb663729940b2591416638`, computer `sha256:fa96e9d45fc287f15e02aeaceac5f124c07968ae8171fd7218ec3b322a846c07`.
   - After setting the tags back to `edge` and re-running the installer the same day: app `sha256:f12fa80e89efe7287d4080edf9635253b1b09df78f30a76f74e70b5137409388`, computer `sha256:1af14870395d54c77d37ae2e697ab43e54ae2f546c723b5f5b0611bb5ffdac8c` — upstream had republished `edge` in the meantime, so the tag rollback landed on a *newer* build than the one upgraded from (the stack was healthy and the data intact, but it was not the pre-upgrade state).
   - **Guide rule that follows: a rollback on the `edge` channel must be pinned by digest** — record the running digests before upgrading; to roll back exactly, pull those digests, retag them, and restore the pre-upgrade database backup. That exact procedure is what the second rollback step below did, and its final state matched the baseline digests verbatim.

## Baseline (pre-upgrade state, measured)

```text
Running images:
  ghcr.io/elie222/rakazo/app:edge
    sha256:24e460aca4390fd4773c7eacee50b6d5e5bca4af8cdb663729940b2591416638
  ghcr.io/elie222/rakazo/computer:edge
    sha256:fa96e9d45fc287f15e02aeaceac5f124c07968ae8171fd7218ec3b322a846c07
Services: api / postgres / supervisor / worker Up (healthy), web Up
  (postgres had been up 11 days; the stack had run since 2026-10-24/25)

Fresh database backup taken before any change (backup procedure: pg_dump
to a file on the server first, then gzip):
  backup-pre-v060-2026-11-04.sql     1,204,905 bytes, 86 CREATE TABLE
  backup-pre-v060-2026-11-04.sql.gz    352,790 bytes, gzip -t OK
The October backup (backup-pre-upgrade-2026-10-25.sql.gz) was not touched.

Baseline row counts (exact SELECT count(*)):
  users=1  bots=4  messages=97  account=1  session=3  computers=1  events=515
```

## Upgrade to v0.6.0

```bash
# in .env: RAKAZO_IMAGE_TAG=edge -> v0.6.0 ; RAKAZO_COMPUTER_IMAGE_TAG=edge -> v0.6.0
bash install-images.sh        # exit 0, 132 seconds (includes image pulls)
```

```text
Running images after upgrade (identical to the test 2 fresh install):
  app      sha256:b8719f7449327aa9f36ad490b84b57602e42d688212a7791a704d42f402f3aca
  computer sha256:0c2800e6e3fa86052b9b07a94d5656abcaa7fa88e228d5abc7c1f89846ed0f85
Services: postgres kept running (not recreated); api / supervisor / worker
  Up (healthy), web Up
GET  http://127.0.0.1:5173/        -> 200
GET  http://127.0.0.1:5173/health  -> 200
GET  https://<public-host>/        -> 200
POST /api/auth/sign-in/email (deliberately wrong password) -> HTTP 401
  {"message":"Invalid email or password","code":"INVALID_EMAIL_OR_PASSWORD"}
  (the maintainers hold no owner credential for this instance, so the auth
  pipeline was probed this way instead of a real sign-in; the response is
  the normal credential rejection, served against the preserved database)
Diagnostics CLI: Revision 6f9b1689d33889a8e2df2e7d20748778343e1f0b
  (same as test 2), database connection ok, 218 migrations in image,
  Bots: 4 (0 archived)
Row counts after upgrade: users=1 bots=4 messages=97 account=1 session=3
  computers=1 events=515 — identical to baseline, no data lost.
```

**Upgrade result: passed.** One-command upgrade via the official installer, about 2 minutes, zero data change.

## Rollback, step 1: tag-level (set the tags back to `edge`)

```bash
# in .env: both tags back to edge
bash install-images.sh        # exit 0, 130 seconds
```

The stack came back healthy (all services up, local and public 200s, auth probe 401, row counts identical to baseline), and the newer `edge` code ran without errors against the database that v0.6.0 had migrated. **But the instance was not back at its pre-upgrade build**: upstream had published new `edge` images during the same day, so the pull resolved to

```text
  app      sha256:f12fa80e89efe7287d4080edf9635253b1b09df78f30a76f74e70b5137409388  (baseline: 24e460ac…)
  computer sha256:1af14870395d54c77d37ae2e697ab43e54ae2f546c723b5f5b0611bb5ffdac8c  (baseline: fa96e9d4…)
```

Lesson recorded for the guide: **`edge` is a moving tag; a tag-level rollback returns you to the channel's current build, not to the build you upgraded from.** Exact rollback needs the baseline digests (and, for the database, the pre-upgrade backup).

## Rollback, step 2: exact (baseline digests + database restore)

```bash
# the baseline images were no longer cached locally; both were still
# retrievable from GHCR by digest:
docker pull ghcr.io/elie222/rakazo/app@sha256:24e460aca4390fd4773c7eacee50b6d5e5bca4af8cdb663729940b2591416638
docker pull ghcr.io/elie222/rakazo/computer@sha256:fa96e9d45fc287f15e02aeaceac5f124c07968ae8171fd7218ec3b322a846c07
docker tag <app digest> ghcr.io/elie222/rakazo/app:edge
docker tag <computer digest> ghcr.io/elie222/rakazo/computer:edge

# services stopped (postgres left running), database dropped and recreated,
# then restored from this run's pre-upgrade backup file:
docker compose -f docker-compose.images.yml stop web api worker supervisor
# DROP DATABASE ... WITH (FORCE); CREATE DATABASE ... (same name and owner)
# psql -v ON_ERROR_STOP=1 < backup-pre-v060-2026-11-04.sql   -> exit 0
docker compose -f docker-compose.images.yml up -d            # 24 seconds
```

One incident, recorded honestly: the first combined attempt at the database step was interrupted in the test environment after the drop/recreate had succeeded but before the restore had run, leaving the stack stopped with an empty database for a few minutes. The restore was then executed as a standalone step from the verified backup file and completed with exit code 0 — which is exactly what the mandatory pre-upgrade backup is for. The Hetzner snapshot was not needed. **Process lesson for every future upgrade run: never combine the database drop and the restore in a single command — run them as separate steps and confirm exit code 0 for each before continuing.**

Final state after the exact rollback:

```text
Running containers' image: sha256:24e460aca439… — the baseline app build
  (computer edge tag also back at sha256:fa96e9d4…)
Services: api / postgres / supervisor / worker Up (healthy), web Up
GET  http://127.0.0.1:5173/        -> 200
GET  http://127.0.0.1:5173/health  -> 200
GET  https://<public-host>/        -> 200
POST /api/auth/sign-in/email (wrong password probe) -> HTTP 401 (as before)
Row counts: users=1 bots=4 messages=97 account=1 session=3 computers=1
  events=515 — identical to baseline.
Diagnostics CLI: not available on this build — the rolled-back image
  predates it (`Cannot find module '/app/apps/api/dist/cli/diagnostics.js'`,
  Node v22.23.2), so health on the old build was verified via container
  status, HTTP checks, the auth probe and the row counts instead.
```

**Rollback result: passed, with the moving-tag caveat above.** The instance is back at its exact pre-upgrade build and database state. The v0.6.0 and newer-edge images remain in the local image cache (unused); the rollback's temporary decompressed dump file was removed, and this run's `backup-pre-v060-2026-11-04.sql.gz` remains on the machine next to the October backup.

## Notes

- The model-provider key on this instance (a 7-day test key created in October) was already expired by design; per the test plan, no bot conversation was attempted in either direction and the key was not touched. Bot-level function on this stack was proven in October (test 1a) and the v0.6.0 stack was verified in test 2.
- The pre-upgrade Hetzner snapshot (`pre-v060-upgrade-2026-11`) was never needed during this test and was deleted on 2026-11-05.
- No credentials, `.env` contents, server addresses or personal data are recorded in this log; the public hostname is the same `sslip.io` name used in the project's earlier records and is abbreviated here as `<public-host>`.
