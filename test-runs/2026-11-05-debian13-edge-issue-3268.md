# Test run: 2026-11-05 — Debian 13 (x86_64) — edge fresh install and issue #3268 reproduction attempt (test 5)

- **Date (Europe/Berlin):** 2026-11-05
- **Tester:** DeployButler
- **Provider / machine type:** Hetzner CPX22 (2 vCPU AMD / 4 GB RAM / 80 GB SSD), Falkenstein data center (throwaway test machine)
- **Architecture:** x86_64 (the matrix row left architecture open; this run's variables are the operating system and the `edge` channel)
- **Operating system:** Debian GNU/Linux 13 (trixie) — the reporter's OS in upstream issue #3268
- **Docker:** 29.9.3 (client and server), Docker Compose v5.7.0 — installed fresh via the official `get.docker.com` script, which took **18 seconds**. The reporter's environment was "Docker 29.x on Debian 13"; this matches.
- **Rakazo release / channel:** `edge` (deliberately not pinned — the reproduction target is the reporter's channel)
  - app `edge` digest: `sha256:1904612350a25baf4ec365e69aa906bb52d453d765e63acd72190e383258c8cc`
  - computer `edge` digest: `sha256:cffc2fc52a7a953103bf5db5cc5e9a2c6e74ede35a31daaaafda70fb567cdf6d`
  - diagnostics revision `f2c3d7dc51fd88cbb9a1e59f3577599ad6fd3d34`, 225 migrations in image
- **Scenario:** Fresh install in production shape (Caddy reverse proxy + public HTTPS), then walk upstream issue #3268's own reproduction steps: create a bot with a team computer, open the computer screen, and watch for the websockify 404.

## Upstream issue state, re-checked before testing (2026-11-05)

Issue [#3268](https://github.com/elie222/rakazo/issues/3268) was read again on the day: still **open**, no comments, no linked fix. The task therefore stayed a reproduction attempt against the current `edge` build named above. (The reporter's `edge` was "latest as of 2026-11-04"; `edge` is a rolling tag — our test 3 run already caught it being republished once on 2026-11-04 — so the reporter's exact build cannot be re-pulled or identified. Everything below is scoped to the digests above.)

## Install and production configuration

```bash
curl -fsSL https://get.docker.com | sh        # 18 s, timed separately
mkdir -p ~/rakazo && cd ~/rakazo
curl -fsSLO https://raw.githubusercontent.com/elie222/rakazo/main/infra/compose/install-images.sh
bash install-images.sh --prepare-only         # tags left at edge on purpose
bash install-images.sh                          # INSTALL_EXIT=0, 107 seconds
```

- Stack came up clean (installer exit 0; `api`/`postgres`/`supervisor`/`worker` healthy, `web` plain `Up` — the same display pattern as every earlier run). Idle footprint at install: host 1.7 GiB of 3.7 GiB used; the five containers together about 1.09 GiB. Local web and `/health` both 200.
- Production shape, same as the earlier runs: Caddy v2.11.9 as TLS terminator for a free sslip.io hostname, `BETTER_AUTH_URL` / `WEB_ORIGIN` / `API_URL` set to the public https origin and `RAKAZO_HOST` to the bare host, installer re-run (exit 0, 24 seconds), public HTTPS 200 verified on the machine and from a second network.
- A test owner account was created with the documented `bootstrap-owner` CLI and verified in the database; sign-in over public HTTPS returned 200. (Account address and credentials are not recorded here; the account exists only on this throwaway machine.)

## Getting to a bot: the model-connection gate, and a field finding

The reporter's steps need a bot, and a fresh Rakazo install gates everything behind onboarding step 1, "Connect a model": there is no skip, and navigating straight to the app bounces back to the model form. No external model account or API key was available to this test (using the operator's personal keys was out of scope), so we ran a small model **on the test machine itself** — Ollama 0.40.14 serving `qwen2.5:0.5b` (397 MB), bound to the Docker-internal bridge address only (see the operator note at the end of this log) — and tried to connect it as an OpenAI-compatible endpoint. That produced a finding worth recording on its own:

1. Submitting the private (RFC1918) endpoint was rejected deterministically, four attempts in a row, with the verbatim error: **"Private model endpoints require operator authorization."**
2. The mechanism is documented in upstream `docs/self-host-models.md` (read on the day): literal private-IP and loopback model endpoints are an operator-level decision. Two settings are involved:
   - `RAKAZO_LOCAL_MODELS` + `RAKAZO_LOCAL_MODELS_URL` register an operator-run local endpoint; after setting them and re-running the installer (35 seconds), the provider list gained a "Local (Ollama / LM Studio)" entry. (Its form still asks for an API-key value, which we did not need to test — the second setting below unblocked the standard form.)
   - `MCP_ALLOW_PRIVATE_ENDPOINT=true` (API and worker) is the documented switch that allows private endpoints for user-connected OpenAI-compatible providers; the doc states no app account bypasses that policy. After setting it and re-running the installer (25 seconds; diagnostics then showed the flag on), the original form passed: model discovery answered "Found 1 model.", and Continue succeeded with the API-key field left empty.
3. Onboarding then completed with no further steps: the first bot ("Chief") was auto-created with a team computer, greeted us, and answered a test message through the local model (about 60 seconds per reply on 2 vCPU — the toy model is a vehicle for this test, not a performance statement).

## The reproduction attempt (issue #3268 steps, walked once, end to end)

1. Bot with a team computer: **yes** — the auto-created bot's panel shows a team computer, initially "Sleeping", with a "Wake up" control. Server side, waking it started a dedicated sandbox container from the `computer:edge` image.
2. Open the computer screen: clicked "Wake up"; the panel showed "Starting live desktop…", and after about 60 seconds **a live desktop rendered** in the "Bot screen" view — a Chromium window on a search homepage, taskbar and clock visible. **No 404, no "Could not load the desktop", no error of any kind appeared.**
3. Server-side corroboration for the same window (11:22–11:23 CET): the API log shows the screen-target requests (`POST /api/internal/screen-target`) completing with **HTTP 200** throughout, and a scan of the api/supervisor/web logs for the whole session window found **zero 404 responses**.

**Result: not reproduced on this build.** Following the issue's own steps on Debian 13 / Docker 29.9.3 / today's `edge` behind a reverse proxy, the screen connects and the desktop is live.

### The paired probe: the token layer behaves exactly as the issue describes

For contrast, the sandbox's websockify endpoint was also probed **directly** on the host loopback, bypassing both Caddy and the app's screen proxy:

```text
GET http://127.0.0.1:<sandbox-port>/            -> 200 (noVNC directory listing)
GET http://127.0.0.1:<sandbox-port>/websockify  -> 404 (request carried no token)
```

So the mechanism in the issue is real at the websockify layer: a request without a valid token does get a 404. The difference on this build is that the app's own screen path — the one the issue says drops the token — delivered a working session end to end. Whether upstream fixed the proxy in the day since the report, or the failure depends on some detail of the reporter's setup that ours does not share (their proxy product is not named in the issue; ours is Caddy), cannot be told from here: the issue is still open and uncommented, and we do not claim an upstream fix — only that **on the digests recorded above, the reported failure did not occur**.

## Resource record (4 GB machine, everything running)

With the stack, the local model and the woken sandbox all up: host memory 2.6 GiB of 3.7 GiB used, 1.1 GiB available; the sandbox container used 716 MiB. No out-of-memory events in the kernel log and no OOM-killed containers at any point in this run.

## Operator note: a mistake of ours during this run

The local model server (Ollama) was first started bound to all network interfaces, which made its port reachable from the public internet for roughly ten minutes. It held no credentials and no user data — only a small public test model — and it was rebound to the Docker-internal bridge address as soon as the requirement was flagged by the maintainers; public unreachability was then verified from outside, and the API container's access over the internal address was re-verified. We record it here because it happened, not because it changed any result.

## Notes

- The machine's address, the test account's credentials, `.env` contents and personal data are not recorded here. The machine was left running for the maintainers' acceptance and deleted only on their instruction; deletion evidence is kept with the project records, not in this repository.
- The screen view was walked once, successfully. A single success does not prove the reporter's failure impossible on other builds or proxy setups — it scopes the claim to this build, this proxy, and this walk, which is what the log records.
