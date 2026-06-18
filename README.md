# lightning-rider-plugins

A [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
holding thin, skill-only plugins that drive lightning-rider-999's self-hosted
command-line tools.

Each plugin is **thin** on purpose: it ships a skill (an agent contract) and
nothing else — no bundled binary, no copied operation catalog. A skill
pre-approves its tool's binary, checks that the binary is installed (and points
at the tool's own installer if it is not), then discovers the live operation
surface from the binary itself (`<tool> catalog`, `<tool> --help`) rather than a
snapshot that drifts as the tool evolves.

## Plugins

| Plugin     | Drives        | Tool repo                              | Nature                                                                 |
|------------|---------------|----------------------------------------|------------------------------------------------------------------------|
| `stashbox` | `stashbox` CLI | [`lightning-rider-999/go-stashbox`](https://github.com/lightning-rider-999/go-stashbox) | **Read-only** — query a stash-box GraphQL instance (StashDB or any other). |
| `stash`    | `stash` CLI    | [`lightning-rider-999/go-stash`](https://github.com/lightning-rider-999/go-stash) | Read **and** write — query and operate a self-hosted Stash media server. |

## Install

Add the marketplace, then install whichever plugin(s) you want:

```shell
# From a local checkout (testing):
/plugin marketplace add ./lightning-rider-plugins

# From GitHub once published:
/plugin marketplace add lightning-rider-999/lightning-rider-plugins

/plugin install stashbox@lightning-rider-plugins
/plugin install stash@lightning-rider-plugins
```

The skill in each plugin checks for its CLI binary on first use and, if it is
missing, points you at the tool's own installer. The plugins do not bundle the
binaries.

### Installing the CLIs the plugins drive

Each CLI installs via `go install`, a checksum-verifying `install.sh`, or
**Homebrew** — pick whichever you like; the skill just needs the binary on your
PATH. As of **v0.1.1**, both CLIs are available through Homebrew (macOS / Linux)
from the
[`lightning-rider-999/homebrew-tap`](https://github.com/lightning-rider-999/homebrew-tap):

```sh
brew install lightning-rider-999/tap/stashbox   # the stashbox CLI
brew install lightning-rider-999/tap/stash       # the stash CLI
```

Full install options (Go, `install.sh`, manual download) live in each tool's
README: [`go-stashbox`](https://github.com/lightning-rider-999/go-stashbox) and
[`go-stash`](https://github.com/lightning-rider-999/go-stash).

## Related projects

Sibling repositories in the `lightning-rider-999` family:

- [`lightning-rider-999/go-stashbox`](https://github.com/lightning-rider-999/go-stashbox)
  — the read-only `stashbox` CLI and Go client/library for a stash-box GraphQL
  instance (StashDB or any other). Installable via Homebrew:
  `brew install lightning-rider-999/tap/stashbox` (v0.1.1+).
- [`lightning-rider-999/go-stash`](https://github.com/lightning-rider-999/go-stash)
  — the read/write `stash` CLI and Go client/library for a self-hosted Stash
  media server. Installable via Homebrew:
  `brew install lightning-rider-999/tap/stash` (v0.1.1+).
- [`lightning-rider-999/homebrew-tap`](https://github.com/lightning-rider-999/homebrew-tap)
  — the Homebrew tap that packages both CLIs (`brew install
  lightning-rider-999/tap/{stashbox,stash}`).

## Configuration the tools expect

These are read by the CLIs the skills drive (the skills never print the API
keys):

- **`stashbox`** — `STASHBOX_URL` (required, no default; the base stash-box UI
  URL) and `STASHBOX_API_KEY` (sent in the `ApiKey` header; optional for an
  instance that allows unauthenticated read access).
- **`stash`** — `STASHAPP_URL` (the base Stash UI URL) and `STASHAPP_API_KEY`
  (sent in the `ApiKey` header; optional for an instance with auth disabled).

## Validate

```shell
claude plugin validate .                       # the marketplace manifest
claude plugin validate ./plugins/stashbox      # a plugin + its skill frontmatter
claude plugin validate ./plugins/stash
```

## License

[MIT](LICENSE).
