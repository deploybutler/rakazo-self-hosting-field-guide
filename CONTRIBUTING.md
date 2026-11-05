# Contributing

Thank you for helping keep this field guide accurate. The value of this repository is that every claim can be checked, so the rules below are strict on purpose.

## What we accept

- A new test run, filed as `test-runs/YYYY-MM-DD-<provider>-<arch>-<version>.md` using `test-runs/TEMPLATE.md`
- A correction to the compatibility matrix, backed by a test-run log in the same pull request
- A new row in the error index, with a source link (an upstream issue or pull request, an upstream documentation page, or a test-run log in this repository)
- A fix to an existing error row when a test run shows the documented cause or fix is wrong — in that case the test run wins and the row must link to it

## Rules for test runs

1. One file per run. Use the template and fill in every field: date (Europe/Berlin), provider and machine type, architecture, operating system, `docker version` output, `docker compose version` output, the Rakazo image tag **and** digest for both the app and computer images, the commands you ran, the sanitised `docker compose ps` output, and the smoke test result.
2. Record what actually happened. If a step failed, write down the failure and what you did next. A failed run with an honest log is more useful than a clean summary.
3. Time and resource figures (install duration, idle memory) must come from that run's own measurements. Do not copy numbers from another run.
4. If a planned machine type is unavailable, say so in the log and either file the row as TODO or record the substitute machine type honestly. Never present an x86 result as an arm64 result, or the other way around.

## Rules for the error index

- The error text must be copied verbatim and shown in code formatting.
- Every row needs: the cause, the fix, the status (for upstream issues: open or closed, with the date you checked), and a source link that returns HTTP 200.
- If you cannot find a source, do not add the row. If a behaviour is documented but you have not reproduced it, mark the row `TODO: verify` instead of presenting it as confirmed.

## Redaction rules (no exceptions)

Never commit any of the following, in logs, screenshots, or any other file:

- Passwords, API keys, tokens, or any `.env` values
- Email addresses (your own or anyone else's)
- Server IP addresses or personal domains
- Payment details or account identifiers

Before opening a pull request, run the checks from the acceptance list yourself:

```bash
# No CJK characters anywhere in the working tree:
grep -rnP '[\x{4e00}-\x{9fff}]' --exclude-dir=.git .

# No obvious secrets or IP addresses in test runs — review every hit by hand:
grep -rniE 'api[_-]?key|token|password|secret|[0-9]{1,3}(\.[0-9]{1,3}){3}' test-runs/
```

Sanitise `docker compose ps` and command output before pasting: replace addresses, hostnames you own, and account names with neutral placeholders.

## Language and commits

- Everything in this repository is in English, including file names, code comments, and commit messages.
- Keep commit messages short and factual, for example `Add test run 2026-11-04 hetzner arm64 v0.6.0` or `Mark websockify row TODO: source not located`.

## Community conduct

When you discuss this guide in upstream issues, Reddit, Discord, or other community spaces, do not post prices or paid-service links there. Link to the relevant test run or error row instead, so the discussion stays about the technical facts.
