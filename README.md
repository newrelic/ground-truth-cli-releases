# Ground Truth CLI 

Talk to **Ground Truth** — New Relic's SRE agent — from your terminal.

```console
$ groundtruth ask "Is checkout-service healthy right now?"

**checkout-service is healthy right now.** Both instances show stable performance with no
active alerts.

- **Throughput:** 5,505 requests/min (stable)
- **Response time:** 10.3 ms average
- **Error rate:** 0%
```

---

## What

Ground Truth CLI is a terminal client for New Relic's Ground Truth agent gateway and NerdGraph API. It
lets you ask the **Ground Truth** SRE agent questions about your services, run NRQL/entity
queries directly, and run a bounded, concurrent **investigate** collection (golden signals,
relationships, tracked changes, incidents, log signals) that Ground Truth then interprets — all
without leaving a shell or opening a browser. Every command supports a stable `--json`
envelope and distinct exit codes, so it doubles as a scriptable building block for runbooks
and CI, not just an interactive chat.

**Acronyms:** *AG-UI* is the streaming agent-event protocol (see
[docs.ag-ui.com](https://docs.ag-ui.com/concepts/events)) the gateway speaks over
Server-Sent Events. *NerdGraph* is New Relic's GraphQL API. *NRQL* is New Relic's SQL-like
query language. *SRE* is Site Reliability Engineer.

## Why

On-call engineers need answers fast, from wherever they already are — a terminal, a
runbook step, a CI job — not a New Relic browser tab. This CLI exists so that talking to
Ground Truth and querying telemetry can be scripted and composed like any other Unix tool:
piped into `jq`, checked for an exit code, run from a script with no interactive session.

It wraps the same AG-UI gateway and NerdGraph API the web product uses, so it isn't a
second implementation of "talk to Ground Truth" — it's a different, terminal-native front end
onto the same backend. Its responsibilities have grown since the first version: it started
as `ask`/chat only, then grew `nrql`/`entity search` for direct queries that don't need the
agent, and `investigate`/`bundle`/`groundtruth web` for collecting and reviewing evidence
offline (including a pannable, interactive view and a local browser handoff), independent of
whether Ground Truth itself is asked to interpret it.

### Common troubleshooting

**Start here:** run `groundtruth`, type `/doctor`. It checks your PATH, region and profile,
account, credential, and terminal, and prints a row per check.

| Symptom | Fix |
|---|---|
| `command not found: groundtruth` | The install directory isn't on your PATH. The install script puts the binary in `~/.local/bin` (macOS/Linux) or `%LOCALAPPDATA%\ground-truth-cli\bin` (Windows) — add that directory to your PATH, or open a new terminal. See [Local Setup](#local-setup). |
| `no API key configured` | Run `groundtruth init`. |
| `profile "prod" not found (available: production)` | Typo — use a name from `groundtruth profile list`. This is a hard error on purpose, so a mistake can't run against the wrong environment. |
| `the agent returned HTTP 404` | Your key isn't valid for this environment. Check which one you're on with `groundtruth profile list`. |
| `could not reach Ground Truth for …` / requests hang | Check your network connection, then run `groundtruth doctor` to confirm the profile's region matches your New Relic account. |
| `an account ID is required` | `nrql` needs one: `groundtruth nrql "..." --account-id 1234567`. |
| A long investigation or query stops with a timeout | The CLI's own limits are generous: `investigate --timeout` defaults to `1h` and `0` removes it, and each NerdGraph request may take up to 10 minutes. Server-side limits still apply (NRQL's own query timeout, gateway idle timeouts), so a very wide `--since` window can still fail on the server. Narrow the window, or press Ctrl+C to stop a run at any time. |

**Anything else** — capture what the service actually sent:

```sh
groundtruth --debug ask "is checkout healthy?"
cat groundtruth-debug.log
```

Your API key is redacted in that file, and it's written owner-only. It does contain your
questions and the telemetry that came back, so read it before pasting it into a ticket.

## Local Setup


Install a prebuilt binary directly from GitHub Releases:

```sh
curl -fsSL https://github.com/newrelic/ground-truth-cli-releases/releases/latest/download/install.sh | sh
```

On Windows (PowerShell):

```powershell
irm https://github.com/newrelic/ground-truth-cli-releases/releases/latest/download/install.ps1 | iex
```

Check it worked:

```sh
groundtruth --version
```

**Run it locally** — `groundtruth init` writes `~/.groundtruth-cli/config.toml` for you:

```console
$ groundtruth init

No configuration found at /Users/you/.groundtruth-cli/config.toml
Let's set one up. Press Ctrl+C to cancel.

Profile name [default]: account1

Which environment?
  1) us       production, US data centre
  2) eu       production, EU data centre
  3) jp       production, JP data centre
Choice: 1

New Relic account ID: 1234567

Authentication method:
  1) OAuth Login       sign in with your browser
  2) API Key           paste a New Relic user API key
Choice: 1

Opening your browser to sign in. If it doesn't open, visit:
https://oauth2.service.newrelic.com/oauth2/auth?...
Signed in.

Authenticated as you@newrelic.com

Wrote /Users/you/.groundtruth-cli/config.toml (owner-only)
Profile "account1" is now the default (region us)
```

Choosing API Key instead prompts for a New Relic user API key (starts with `NRAK`) and
verifies it against the region you picked before saving. Either way the key is never
shown as you type it, and the file is written owner-only (`0600`). You never type a
URL — picking a region sets the agent, NerdGraph and OAuth endpoints for you. That
last one is why the region matters even for a browser sign-in: an access token is
only valid in the environment that issued it, so a profile's region decides where it
is minted as well as where it is sent.


### Just the credential

`init` is full first-time setup (region, account ID, credential, all in one prompt).
Once a profile already has those and you just need to (re-)authenticate, `auth login`
is quicker — it only ever touches the credential (and covers OAuth and API keys):

```sh
# OAuth 2.0 Authorization Code + PKCE — opens a browser
groundtruth auth login

# A New Relic API key instead, inline or prompted
groundtruth auth login --api-key=NRAK-...
groundtruth auth login --api-key            # prompts (masked), or reads --stdin

groundtruth auth status                     # what's configured, and the token's state
groundtruth auth logout                     # revoke and clear the credential
```

With more than one profile configured, `auth login` asks which one to sign in to,
listing each profile's region beside its name:

```
Which profile do you want to sign in to?
 * 1) prod-us1          region us
   2) prod-us2          region us
   3) a new profile

* = current default (prod-us1)
Choice:
```

Name one directly with `groundtruth --profile <name> auth login` (or `GROUNDTRUTH_PROFILE`)
to skip the question; a non-interactive shell skips it too and uses the default
profile, so scripts and CI are unaffected. `auth login` never changes an existing
profile's region — that's `init`'s job — so `--region` applies only when the profile
is brand new.

Secrets go to the OS keychain by default (macOS Keychain / Windows Credential Manager /
Freedesktop Secret Service); set `credential_storage = "file"` in a profile to store
them as plaintext in `config.toml` instead. If the keychain backend is ever unavailable
on a given machine, saving falls back to the plaintext file automatically, with a
warning. An OAuth access token near expiry is refreshed automatically the next time a
command needs it — no need to re-run `login` just because a token expired.

---

## Dependencies

- **Ground Truth agent gateway** — the endpoint for the profile's region (`us`, `eu`, `jp`). Every
  `ask`/chat turn and `investigate`'s interpretation step call it over AG-UI/SSE.
- **New Relic NerdGraph API** — backs `nrql`, `entity search`, and `investigate`'s evidence
  collection, independent of the agent gateway.
- **A New Relic credential** — a user API key (`NRAK-…`) or an OAuth token, per profile in
  `~/.groundtruth-cli/config.toml`.

A profile's `region` (`us`, `eu` or `jp`) derives both endpoints above, so no
hostname is ever typed by hand.

## Using this Service

**Chat in your terminal:**

```sh
groundtruth
```

Type a question, press Enter. `Esc` or `Ctrl+C` to quit. Press `/` for commands —
`/nrql <query>` and `/entity <term>` query your data without leaving the chat (both take
`--debug` to log the raw request; `/entity` also takes `--limit` and `--raw`; `/nrql` uses the
account for the session, set at startup with `groundtruth --account-id 1234567` or in your
profile), `/doctor` checks your setup, `/debug` logs raw events.

**Ask one question and exit** — for scripts and runbooks:

```sh
groundtruth ask "Why is checkout-service slow?"
```

**Ground the question in a knowledge connector** with `--rag` (repeatable), or attach a
ground-truth query with `--nrql` — its rows are reported alongside the answer:

```sh
groundtruth ask "Summarize the incident runbook" --rag <connector-id>
groundtruth ask "How many errors in the last hour?" --nrql "SELECT count(*) FROM TransactionError SINCE 1 hour ago"
```

**Query your data directly**, no agent involved:

```sh
groundtruth nrql "SELECT count(*) FROM Transaction SINCE 1 hour ago"
```

**Find a service's GUID:**

```sh
groundtruth entity search checkout
```

**Investigate a service** — collect its golden signals, relationships, tracked changes,
incidents, and log signals, then have New Relic AI read them:

```sh
groundtruth investigate checkout-service
```

```console
GOLDEN SIGNALS                                    30 intervals · NerdGraph NRQL
  ● Error Rate          ▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅▅  0.00% → 0.00%  Δ +0.00pp
  ● P95 ms              ▆▄▄▃▄▃▂▅▂▅▃▄█▃▆▅▃▄▃▅▆▁▃▄  3.40ms → 2.78ms  Δ −18.1%
  ◉ Throughput Per Min… █▄█▄█▄█▄█▄█▄█▄█▄█▄█▄█▄█▁  86.0/min → 69.0/min  Δ −19.8%
                        20:37              21:07
```

Add `--evidence-only` to skip the AI turn and just look at the evidence, `--data` for the
exact numbers behind each chart, `--rag <connector-id>` to ground the interpretation in a
knowledge connector, `--interactive` for a pannable terminal view over the collected range,
and `--save` to keep the snapshot. The interpretation is always labeled separately from the
evidence it was based on.

**Switch environments:**

```sh
groundtruth profile use production
```

Every later command uses that environment. To see what you have:

```sh
groundtruth profile list
```

```console
  NAME        REGION   ACCOUNT  CREDENTIAL
  production  us       9876543  set
* production  us       1234567  set

* = active   ·   /Users/you/.groundtruth-cli/config.toml
```

Add another environment with `groundtruth init` and give it a new profile name.

**Commands:**

| Command | What it does |
|---|---|
| `groundtruth` | Chat session in the terminal |
| `groundtruth ask "<question>"` | Ask one question, print the answer, exit |
| `groundtruth nrql "<query>"` | Run an NRQL query directly |
| `groundtruth entity search <term>` | Find entities and their GUIDs |
| `groundtruth investigate <entity>` | Collect golden signals, relationships, changes, incidents, and logs for an entity, then ask New Relic AI to interpret them |
| `groundtruth bundle <list\|view\|report\|compare\|import\|export>` | Inspect, compare, and share saved evidence bundles, offline |
| `groundtruth web` | Serve the browser evidence/chat UI on loopback |
| `groundtruth init` | Create or update a profile |
| `groundtruth doctor` | Check your setup: PATH, region and profile, account, credential, terminal |
| `groundtruth auth login` | (Re-)authenticate an existing profile — just the credential |
| `groundtruth auth status` | Show what's configured and the token's state |
| `groundtruth auth logout` | Revoke and clear the credential |
| `groundtruth profile list` | Show configured profiles |
| `groundtruth profile use <name>` | Switch the active profile |
| `groundtruth schema` | Machine-readable description of every command |

Run `groundtruth <command> --help` for the flags on any command.

**Flags you'll actually use:**

| Flag | Purpose |
|---|---|
| `--json` | Emit the structured JSON envelope instead of text |
| `-v, --verbose` | Show the agent's steps and tool calls |
| `--thread <id>` | (`ask`) Continue an earlier conversation |
| `--rag <connector-id>` | (`ask`, `investigate`) Attach a RAG knowledge connector (repeatable) |
| `--nrql "<query>"` | (`ask`) Also run this query and report its rows as ground truth |
| `--limit <n>` | (`entity search`) Maximum results |
| `--since <window>` | (`investigate`) Evidence window, e.g. `30m`, `2h`, `7d` (default `30m`) |
| `--evidence-only` | (`investigate`) Collect evidence without asking New Relic AI to interpret it |
| `--data` | (`investigate`) Print the exact per-interval numbers beneath the charts |
| `--interactive` | (`investigate`) Open a pannable terminal view over the collected evidence, no AI call |
| `--timeout <duration>` | (`investigate`) Deadline for collection *and* interpretation together (default `1h`; `0` for no client deadline) |
| `--save` | (`investigate`) Save the evidence and interpretation locally |
| `--open` | (`investigate`) Save, then open the snapshot in the browser evidence view |
| `--profile <name>` | Use a different profile for one command |
| `--debug` | Log raw requests and events to `groundtruth-debug.log` (works with `ask`, `nrql`, and `entity search`) |

**Scripting:**

Add `--json` to any command for a stable envelope: `{ok, data, error, hint, meta}`.

```sh
groundtruth --json ask "Is checkout healthy?" | jq -r .data.answer

groundtruth --json entity search checkout | jq -r '.data.entities[].guid'
```

Exit codes:

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | The agent or an upstream service failed |
| `2` | Usage error, or the service was unreachable |
| `3` | Configuration problem |
| `6` | The command ran, but its result is partial — a signal that couldn't be collected, evidence trimmed to a budget, or an interpretation that never arrived. The evidence it *did* gather is still in `data`. Treat it as a gap to inspect, not as proof that everything is fine. For `ask`, it also means the agent stopped to wait for you (an approval or more context): `meta.pendingInput` holds its requests, and you reply with `groundtruth ask --thread <meta.threadId> "..."`, or answer an approval with `groundtruth ask --thread <meta.threadId> --approve` (or `--deny`, optionally with `--note "..."`). Read-only mode refuses `--approve` and `--deny`. With `ask --events`, each moment of the run (`run_started`, `text`, `approval_request`, `run_finished` and the rest) is a JSON line on stdout as it happens, and the envelope is the last line. |
| `130` | Interrupted with Ctrl+C |

```sh
if groundtruth --json ask "any incidents?" > result.json; then
  jq -r .data.answer result.json
else
  jq -r '.error.message, .hint' result.json
fi
```

`groundtruth schema` describes every command as JSON, generated from the live command tree —
safe for an AI agent to read and drive.

## FAQ

**Start here:** run `groundtruth doctor`. It checks your PATH, region and profile, account,
credential, and terminal, and prints a row per check. It exits 3 when a check fails, so a
script can gate on setup being complete:

```sh
groundtruth doctor || echo "fix the failing checks above"
```

The report names regions and profiles only — never a credential, never a service hostname —
so it's safe to paste into a ticket. `--json` gives the same rows as an envelope. The same
checks are available as `/doctor` inside a chat session.

**Why does `nrql` need an account ID but `entity search` doesn't?** NRQL is always
account-scoped; entity search isn't. See [Using this Service](#using-this-service).

**Where do I report a bug or ask for a feature?** Open a [GitHub issue](https://github.com/newrelic/ground-truth-cli-releases/issues).

**Something's broken and none of the above explains it** — see
[Common troubleshooting](#common-troubleshooting), and if that doesn't resolve it, capture a
`--debug` log before filing an issue.
