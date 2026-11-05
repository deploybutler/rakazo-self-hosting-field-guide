# Test run: 2026-10-24 — Hetzner CAX21 (arm64, Ubuntu 24.04) — Rakazo edge

- **Date (Europe/Berlin):** 2026-10-24 (upgrade follow-up on 2026-10-25 is summarised at the end)
- **Tester:** DeployButler
- **Provider / machine type:** Hetzner CAX21 (4 vCPU / 8 GB RAM / 80 GB, Nuremberg)
- **Architecture:** arm64 (aarch64)
- **Operating system:** Ubuntu 24.04
- **Scenario:** Fresh install, HTTPS via Caddy, first-bot smoke test, backup and rollback

## Versions

```text
$ docker version
Docker 29.9.2

$ docker compose version
Docker Compose v5.6.1
```

(Docker was installed with the official `get.docker.com` script; installation took about 70 seconds.)

- **Rakazo app image:** `ghcr.io/elie222/rakazo/app:edge` — digest at first install is recorded in the original run notes in truncated form only (`0ac276b2...`), so no full digest is claimed here for the first-install build
- **Rakazo computer image:** `ghcr.io/elie222/rakazo/computer:edge` — first-install digest likewise truncated in the original run notes (`303fcb91...`)
- **How the digests were read:** the original run notes, 2026-10-24

## Commands run

```bash
curl -fsSL https://get.docker.com | sh
mkdir -p ~/rakazo && cd ~/rakazo
curl -fsSLO https://raw.githubusercontent.com/elie222/rakazo/main/infra/compose/install-images.sh
bash install-images.sh --prepare-only   # generate .env first
# .env edited: public origins set to the test HTTPS address, host set accordingly
bash install-images.sh                  # full install
docker compose -f docker-compose.images.yml ps
```

HTTPS was terminated with Caddy 2.11.7 on the host, reverse-proxying to `127.0.0.1:5173`, using a temporary `sslip.io` address for the test instance. Personal addresses are omitted from this log on purpose.

## Timing and resources

- **Install duration:** about 137 seconds for the image pull and stack start after Docker was installed; about 66 seconds to add Caddy and reach a public HTTPS 200
- **Idle memory after install:** about 1.1 GB of 7.5 GiB for the stack alone; about 1.5 GB with one bot computer running
- **Notes:** End to end, from a prepared server to a publicly reachable instance, the human-plus-machine time was about 25 minutes including checks

## Service status (sanitised)

Summary of the installer's wait-loop report at the end of the install:

```text
postgres    healthy
api         healthy
worker      healthy
supervisor  healthy
web         healthy
```

The web app answered `200` on `http://127.0.0.1:5173` locally (page title: Rakazo) and `200` over the public HTTPS address with a valid certificate. Bot computer containers are started on demand and are not part of the resident list.

## Smoke test

- **Test performed:** After owner registration, one message to the first bot covering three tasks: (1) remember a code word, (2) use the computer tool to list the home folder and create a file named `hello-test.txt` containing the current date, (3) report the current time in Berlin. Afterwards the bot was asked for the code word again
- **Result:** Pass. Memory answered correctly on the spot and again after later operations; the file was created inside the bot's own sandbox volume (verified on the server: the file exists only under that bot's sandbox directory, not in the host's home directories — sandbox isolation confirmed); the reported Berlin time was correct

## Errors encountered

- `Email not verified` at first owner sign-in. Cause: the default deployment had no SMTP configured, so the verification email was never sent. Fix used in this run (stopgap): an operator marked the owner's email as verified directly in Postgres (one row updated), after which sign-in worked. The durable fixes are to configure email delivery, or to use the server-side owner bootstrap for installations without email — both are now covered in the README error index with upstream sources.
- Public signup was open between going public and owner registration. After the owner account existed, signups were closed (`SIGNUPS_ENABLED=false` plus an allowlist entry for the owner) and the web app was verified to still return 200 and to reject new signups. Treat owner registration and signup lockdown as back-to-back steps.

## Backup / upgrade / rollback

- **Backup taken:** `pg_dump` of the running database, saved to a file first and then gzipped (streaming `pg_dump` through an SSH pipe into gzip failed silently twice in this run; writing the dump to a file first was reliable). Sizes: about 40 KB for the empty baseline, about 54 KB with one bot and its data. The dump was checked to contain the smoke-test code word (2 occurrences)
- **Rollback attempted:** Yes. Web, API, worker, supervisor and the bot computer were stopped, the database was dropped and recreated, the backup was restored, and the installer was re-run. Service restart took about 1 minute after a restore of about 54 seconds. After the rollback: user count unchanged (1), the code word was still present in the restored database (2 occurrences), the sandbox file was intact (it lives on the application-data volume, independent of the database), local and public HTTPS both returned 200, and the owner confirmed in the UI that the bot still answered the code word correctly

## Upgrade follow-up (2026-10-25)

- A genuine cross-version `edge` upgrade was tested the next day: backup first (`pg_dump` to file, then gzip; 84 tables), then `docker compose -f docker-compose.images.yml pull` (about 2.5 minutes) and `up -d`. Postgres was not recreated; the other four services were recreated and reported healthy
- Digests after that upgrade: app `sha256:24e460ac...` and computer `sha256:fa96e9d4...` in truncated form in the original run notes; the full digests for the still-running build are recorded in `test-runs/2026-11-04-hetzner-arm64-edge-verification.md`
- Row counts were unchanged by the upgrade (users=1, bots=1, messages=5), the public HTTPS endpoint returned 200, and the owner verified in the UI that the bot's memory survived the upgrade

## Conclusion

This run proves a fresh `edge` install, HTTPS exposure, a first-bot smoke test, database backup/restore, rollback, and one genuine `edge`-to-newer-`edge` upgrade on an arm64 CAX21 under Ubuntu 24.04. It does not prove anything about the v0.6.0 release tag or about x86 machines; those are covered by matrix rows 2–4.
