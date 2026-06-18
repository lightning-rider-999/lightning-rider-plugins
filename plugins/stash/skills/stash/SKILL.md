---
name: stash
description: >-
  Query and operate a self-hosted Stash media server through the `stash` CLI —
  scenes, performers, studios, tags, galleries, images, groups, files, plus
  library operations like metadata scans and scrapes. Use whenever the task is
  to read or change data on a Stash instance. Unlike a read-only client, this
  CLI can mutate the server, so destructive operations gate behind an explicit
  confirmation flag.
allowed-tools: Bash(stash:*)
---

# stash — Stash GraphQL CLI

`stash` is an agent-first command-line client for a self-hosted
[Stash](https://github.com/stashapp/stash) media server's GraphQL API. It
exposes every Stash operation as a resource-and-verb command
(`stash scene list`, `stash metadata scan`) and prints JSON by default.

This skill is the agent contract for the tool. It does **not** bundle the
binary or a copy of the operation catalog / AGENTS reference (either would drift
from the real tool). Instead: confirm the binary is installed, then ask the
binary itself what it can do.

## Not read-only — it can change the server

Unlike a read-only client, `stash` exposes the full Stash GraphQL surface,
including mutations: create / update / bulk-update / destroy verbs across
resources, and library operations such as `metadata scan` and `scrape`.

- **Job-returning mutations take `--wait`** to block on the job's outcome
  rather than returning the job id and exiting.
- **Destructive operations require `--yes-i-understand`** — the CLI refuses to
  run them otherwise. Do not pass this flag unless the user has explicitly
  asked for the destructive action; prefer a read (`list` / `get`) to confirm
  the target first.

When the task is only to look something up, use the read verbs (`list`, `get`)
and never a mutating one.

## BINARY-CHECK — do this first, every time

Before any other step, verify the binary is on `PATH`:

```sh
command -v stash
```

If that prints a path, continue. **If it prints nothing / exits non-zero, the
binary is not installed — stop and install it, do not improvise.** The tool
lives in `lightning-rider-999/go-stash`; install it with its own installer:

```sh
# Homebrew (macOS / Linux), from the lightning-rider-999 tap:
brew install lightning-rider-999/tap/stash

# Recommended for a Go toolchain:
go install github.com/lightning-rider-999/go-stash/cmd/stash@latest

# Linux/macOS without Go (detects OS/arch, verifies the sha256 checksum):
curl -sSL https://raw.githubusercontent.com/lightning-rider-999/go-stash/main/install.sh | sh
```

`install.sh` honours `INSTALL_DIR=...` (target dir, default `/usr/local/bin`)
and `VERSION=vX.Y.Z` (pin a release). After installing, re-run the
`command -v stash` check before proceeding.

## Configuration — required and secret

The CLI reads two environment variables (each overridable per-invocation by a
flag, `--url` / `--api-key`):

- **`STASHAPP_URL` — the Stash instance.** The base UI URL, e.g.
  `http://stash.local:9999`. GraphQL is served at `<url>/graphql`; the URL is
  normalised, so the base is enough. Provide it via the env var or `--url`;
  there is no built-in default instance. If it is unset, ask the user for the
  instance URL rather than guessing one.
- **`STASHAPP_API_KEY` — secret.** Sent verbatim in the `ApiKey` HTTP header
  (and in the subscription `connection_init` payload). Optional for an instance
  with authentication disabled.

Key handling rules:
- **Never print, echo, log, or paste the API key** — not in commands, not in
  output, not in summaries. Treat it as a credential.
- Prefer the `STASHAPP_API_KEY` environment variable over the `--api-key` flag:
  a flag value is visible in the process listing to other users and lands in
  shell history.
- Assume `STASHAPP_URL` (and `STASHAPP_API_KEY` if needed) are already exported
  in the environment.

## Discover the live operation surface

The command set is generated from the server's schema, so **discover it from
the binary, never from memory**. The embedded `catalog` subcommand needs no
network connection:

```sh
# The full machine-facing catalog: every operation's field, kind, arguments,
# inputs, enums, return type, and exit codes.
stash catalog

# Just one operation's entry, e.g.:
stash catalog FindScenes
```

`stash --help` lists the resource groups (`scene`, `scene-marker`, `performer`,
`studio`, `tag`, `gallery`, `gallery-chapter`, `image`, `group`, `movie`,
`file`, `folder`, `job`, `log`, `metadata`, `scan`, `scrape`, `config`,
`saved-filter`, `stash-box`, `misc`, …); each group's `--help` lists its verbs,
and a verb's `--help` lists its convenience flags. Treat the binary's own list
as authoritative — it can grow with the server schema. The repo's
`docs/AGENTS.md` is the full machine contract (exit codes, error envelope,
input model, the `--wait` re-attach flow, the partial-update three-state rule)
if you have a checkout.

## How invocations are shaped

- Output is JSON by default; `-o ndjson|table|yaml` selects another format.
- Operation variables come from `--input` — a JSON file path, or `-` for stdin
  — forwarded as raw JSON, which preserves the present / absent / null
  distinction that partial-update mutations depend on.
- Convenience flags fill scalar arguments without writing JSON (e.g. `--id`).
  Check a verb's `--help` for which it has.
- On failure the CLI writes a single-line JSON error envelope to stderr and
  exits with a frozen taxonomy code.

## Example invocations

```sh
# List the first page of scenes as newline-delimited JSON.
stash scene list -o ndjson

# Fetch one scene by id.
stash scene get --id 42

# Inspect an operation's inputs, enums, and exit codes without a server.
stash catalog FindScenes

# Kick off a library metadata scan and block until the job finishes.
stash metadata scan --wait

# A mutation with variables from a JSON file (partial-update three-state rules
# apply — present / absent / null are distinct).
stash scene update --input scene-update.json

# A destructive operation — only with the user's explicit go-ahead. The id
# rides in the input (SceneDestroyInput!); destroy has no --id convenience flag.
echo '{"input":{"id":"42"}}' | stash scene destroy --input - --yes-i-understand
```

Confirm the exact verb names and flags with `stash <resource> --help` and
`stash <resource> <verb> --help` — the surface is generated, so the binary is
the source of truth, not this list.
