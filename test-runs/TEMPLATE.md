# Test run: <YYYY-MM-DD> — <provider> <machine type> (<arch>, <OS>) — Rakazo <version or tag>

<!--
File name: test-runs/YYYY-MM-DD-<provider>-<arch>-<version>.md
One file per run. Fill in every field. Delete these comments before committing.
Never paste secrets, .env values, email addresses, IP addresses, or personal domains.
Sanitise all command output before pasting it.
-->

- **Date (Europe/Berlin):**
- **Tester:**
- **Provider / machine type:**
- **Architecture:** <!-- arm64 or x86_64 -->
- **Operating system:** <!-- for example Ubuntu 24.04 -->
- **Scenario:** <!-- fresh install / upgrade from X to Y / pitfall reproduction / reboot recovery -->

## Versions (paste the actual command output)

```text
$ docker version
<output>

$ docker compose version
<output>
```

- **Rakazo app image:** `ghcr.io/elie222/rakazo/app:<tag>` digest `sha256:<digest>`
- **Rakazo computer image:** `ghcr.io/elie222/rakazo/computer:<tag>` digest `sha256:<digest>`
- **How the digests were read:** <!-- for example: docker compose -f docker-compose.images.yml images, or docker inspect -->

## Commands run

```bash
<the exact commands, in order, with personal values replaced by placeholders>
```

## Timing and resources

- **Install duration:** <!-- measured, with what the timer covered -->
- **Idle memory after install:** <!-- measured, with the command used -->
- **Notes:**

## Service status (sanitised)

```text
$ docker compose -f docker-compose.images.yml ps
<sanitised output — all services and their health status>
```

## Smoke test

- **Test performed:** <!-- for example: sign in, create a bot, one message covering memory + a computer-tool task + a time question, then ask for the remembered item again -->
- **Result:** <!-- pass / fail, and exactly what differed from the expected result -->

## Errors encountered

<!--
For each error: the verbatim error text in code formatting, what caused it,
what fixed it, and whether it is already in the README error index.
If an error is new, propose the index row in the same pull request, with a source.
-->

## Backup / upgrade / rollback (if applicable)

- **Backup taken before the run:** <!-- method and size, no file contents -->
- **Upgrade path:**
- **Rollback attempted:** <!-- yes / no; if yes, the result and the time it took -->

## Conclusion

<!-- One or two sentences: what this run proves, and what it does not prove. -->
