---
name: stashbox
description: >-
  Query a stash-box GraphQL instance (StashDB or any other stash-box server)
  through the read-only `stashbox` CLI — scenes, performers, studios, tags,
  sites, edits, drafts, notifications, and their fingerprints. Use whenever the
  task is to look up, search, or report stash-box / StashDB metadata. The CLI
  is read-only: it has no mutations and cannot modify the server.
allowed-tools: Bash(stashbox:*)
---

# stashbox — read-only stash-box GraphQL client

`stashbox` is a command-line client for a [stash-box](https://github.com/stashapp/stash-box)
GraphQL instance ([StashDB](https://stashdb.org) or any other stash-box
server). It exposes every stash-box **query** as a resource-and-verb command
(`stashbox scene get`, `stashbox performer query`) and prints JSON by default.

This skill is the agent contract for the tool. It does **not** bundle the
binary or a copy of the operation catalog (either would drift from the real
tool). Instead: confirm the binary is installed, then ask the binary itself
what it can do.

## Read-only by construction

The `stashbox` CLI is **read-only**. The generated surface is query-only — the
GraphQL Mutation root is stripped before code generation — so there are no
mutations, no async jobs, no uploads, and no destructive-confirmation gate.
Nothing you run through this tool can change the server. Do not look for a
"write" or "update" command; there is none.

## BINARY-CHECK — do this first, every time

Before any other step, verify the binary is on `PATH`:

```sh
command -v stashbox
```

If that prints a path, continue. **If it prints nothing / exits non-zero, the
binary is not installed — stop and install it, do not improvise.** The tool
lives in `lightning-rider-999/go-stashbox`; install it with its own installer:

```sh
# Homebrew (macOS / Linux), from the lightning-rider-999 tap:
brew install lightning-rider-999/tap/stashbox

# Recommended for a Go toolchain:
go install github.com/lightning-rider-999/go-stashbox/cmd/stashbox@latest

# Linux/macOS without Go (detects OS/arch, verifies the sha256 checksum):
curl -sSL https://raw.githubusercontent.com/lightning-rider-999/go-stashbox/main/install.sh | sh
```

`install.sh` honours `INSTALL_DIR=...` (target dir, default `/usr/local/bin`)
and `VERSION=vX.Y.Z` (pin a release). After installing, re-run the
`command -v stashbox` check before proceeding.

## Configuration — required and secret

The CLI reads two environment variables (each overridable per-invocation by a
flag, `--url` / `--api-key`):

- **`STASHBOX_URL` — required, no default.** The base UI URL of the stash-box
  instance, e.g. `https://stashdb.org`. GraphQL is served at `<url>/graphql`;
  the URL is normalised, so the base is enough. The client errors if neither
  `STASHBOX_URL` nor `--url` is set — there is no built-in default instance.
- **`STASHBOX_API_KEY` — secret.** Sent verbatim in the `ApiKey` HTTP header;
  found on your user page in the stash-box instance. Optional for an instance
  that allows unauthenticated read access.

Key handling rules:
- **Never print, echo, log, or paste the API key** — not in commands, not in
  output, not in summaries. Treat it as a credential.
- Prefer the `STASHBOX_API_KEY` environment variable over the `--api-key` flag:
  a flag value is visible in the process listing to other users and lands in
  shell history.
- Assume `STASHBOX_URL` (and `STASHBOX_API_KEY` if needed) are already exported
  in the environment. If `STASHBOX_URL` is unset, ask the user for the instance
  URL rather than guessing one.

## Be a well-behaved guest

StashDB is a shared, community-run public service, not a private endpoint.
Respect rate limits, keep concurrency bounded, never hammer it, and don't add
retries that mask a failure. Query what you need and no more.

## Discover the live operation surface

The command set is generated from the server's schema, so **discover it from
the binary, never from memory**. The embedded `catalog` subcommand needs no
network connection:

```sh
# The full machine-facing catalog: every operation's field, kind, arguments,
# return type, exit codes, and the $defs type dictionary.
stashbox catalog

# Just one operation's entry (inputs + return type), e.g.:
stashbox catalog QueryScenes
stashbox catalog FindPerformer
```

`stashbox --help` lists the resource groups (`scene`, `performer`, `studio`,
`tag`, `tag-category`, `site`, `site-category`, `edit`, `draft`, `notification`,
`user`, `config`, `mod-audit`, `misc`, …); each group's `--help` lists its verbs,
and a verb's `--help` lists its convenience flags. Treat the binary's own list as
authoritative — it can grow with the server schema.

## How invocations are shaped

- Output is JSON by default; `-o table` gives a compact tabular view.
- Operation variables come from `--input` — a JSON file path, or `-` for stdin
  — forwarded as raw JSON, which preserves the present / absent / null
  distinction.
- Convenience flags fill an operation's scalar arguments without writing JSON
  (e.g. `--id`, `--term`, `--limit`). Check a verb's `--help` for which it has.
- On failure the CLI writes a single-line JSON error envelope to stderr and
  exits with a frozen taxonomy code (ok, usage, auth, transport, validation,
  server-fault, not-found).

## Example invocations

```sh
# Ask the server for its build identity (a cheap connectivity/version check).
stashbox misc version

# Look up one performer by stash-box UUID via the --id convenience flag.
stashbox performer get --id 2f5d7e1a-0000-4000-8000-000000000000

# Search performers by term, capped, as a compact table.
stashbox performer search --term "asa akira" --limit 10 -o table

# Search scenes by term.
stashbox scene search --term "office" --limit 5

# Query scenes with a full filter, variables read from a JSON file.
stashbox scene query --input scene-query.json

# Same, piping variables in on stdin.
echo '{"input":{"per_page":5}}' | stashbox scene query --input -

# Look up one scene by UUID.
stashbox scene get --id 9c1b3d44-0000-4000-8000-000000000000

# Inspect an operation's inputs and return type without touching the server.
stashbox catalog QueryScenes
```

Confirm the exact verb names and flags with `stashbox <resource> --help` and
`stashbox <resource> <verb> --help` — the surface is generated, so the binary
is the source of truth, not this list.
