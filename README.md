# incus-spawn-templates

Community template images and tool definitions for [incus-spawn](https://github.com/Sanne/incus-spawn).

These extend isx's built-in `tpl-minimal` with project-specific environments that include pre-cloned repositories, build tools, and primed dependency caches.

`tpl-dev` and `tpl-java` live here too. They used to ship with isx, which now marks its own copies deprecated and will drop them ([#1195](https://github.com/Sanne/incus-spawn/issues/1195)). The definitions here have the same names and contents, so with this directory on your search path they replace the built-ins without the deprecation warning, and templates you already built stay in sync.

## Setup

Add this directory as a search path in `~/.config/incus-spawn/config.yaml`:

```yaml
searchPaths:
  - /path/to/incus-spawn-templates
```

Then build and branch:

```shell
isx build tpl-quarkus
isx branch tpl-quarkus my-feature
```

## Templates

| Template | Parent | Description                                                          |
|----------|--------|----------------------------------------------------------------------|
| `tpl-dev` | `tpl-minimal` | Podman, GitHub CLI, Starship, tmux                                   |
| `tpl-java` | `tpl-dev` | JDK 25 + Maven                                                       |
| `tpl-quarkus` | `tpl-java` | Quarkus + quickstarts source, Podman, Gradle                         |
| `tpl-hibernate-orm` | `tpl-java` | Hibernate ORM source, Podman, Gradle                                 |
| `tpl-hibernate-reactive` | `tpl-hibernate-orm` | Hibernate Reactive source (includes ORM)                             |
| `tpl-tamboui` | `tpl-java` | Tamboui TUI framework source, Gradle, selected skills                |
| `tpl-incus-spawn` | `tpl-java` | incus-spawn itself — recursive development with Incus and COPR tools |
| `tpl-graalvm-dev` | `tpl-dev` | GraalVM development — `mx`, labsjdk, the `oracle/graal` source |

Most templates inherit the `tpl-java` chain, which provides JDK 25, Maven, Claude Code, GitHub CLI, and tmux.

### Working on GraalVM

`tpl-graalvm-dev` is ready to build on first shell — `mx`, a JVMCI-enabled labsjdk in
`/opt/jdks` (already `JAVA_HOME`), and the `oracle/graal` source, with the shell opening
in `~/graal`:

```shell
isx branch tpl-graalvm-dev graal-work
cd compiler && mx build
```

It hangs off `tpl-dev` rather than `tpl-java` on purpose: graal has to be built against the
labsjdk, and `JAVA_HOME` can only be set once across a template chain.

To build against a different JDK, override the `labsjdk` tool's `jdk_id` parameter (any id
from `mx fetch-jdk --list`) in your own template.

`mx` validates TLS in Python rather than through the JVM truststore, so the `mx fetch-jdk`
step in this build is also a good check that the isx MITM proxy's certificates satisfy
OpenSSL 3.5+ strict verification.

## Tools

| Tool | Description |
|------|-------------|
| `podman` | Podman with Docker socket compatibility, configured for Testcontainers |
| `gradle` | Gradle 9.4.1 (installed to `/opt`, symlinked to PATH) |
| `graalvm` | Oracle GraalVM for JDK 25 (installed to `/opt`, `native-image` on PATH) |
| `mx` | GraalVM's `mx` build tool (cloned to `/opt/mx`, `mx` on PATH) |
| `labsjdk` | JVMCI-enabled labsjdk fetched by `mx` into `/opt/jdks` and set as `JAVA_HOME` (param: `jdk_id`) |
| `jtreg` | JTReg test harness (installed to `/opt`, `jtreg` on PATH) |
| `pinchtab` | [PinchTab](https://github.com/pinchtab/pinchtab) browser control for agents — headless Chromium, the `pinchtab` skill (params: `allowed_domains`, `mcp`) |
| `omp` | [omp](https://omp.sh/) — Pi's fork with a Rust core, installed as a self-contained binary |

### Coding with omp

omp ships one self-contained binary per platform, and the upstream installer
only falls back to these release assets when it cannot find a usable Bun. This
tool takes them directly, so neither Bun nor Node.js enters the image, and pins
the checksums the release publishes in `SHA256SUMS.txt`. The version is pinned
too, and the startup update check is off: a template cannot take a newer build
without a rebuild. `omp update` still works inside an instance when the agent
asks for it.

omp brings no provider of its own. It reads the provider variables the template
already exports, the same ones Pi and the other agents use — `OPENAI_API_KEY`
from `codex`, `GEMINI_API_KEY` from `gemini-cli`, `ANTHROPIC_API_KEY` from a
`claude` template on an API-key account, `ANTHROPIC_OAUTH_TOKEN` from `pi` — so
pair it with the agent tool of whichever account it should spend:

```yaml
name: tpl-omp
parent: tpl-java
tools:
  - codex    # exports OPENAI_API_KEY for omp to spend
  - omp
```

### Giving an agent a browser

Add `pinchtab` to any template to put a real browser behind a token-efficient CLI:

```yaml
tools:
  - pinchtab
```

The agent gets the upstream `pinchtab` skill in the shared skills directory, so
it can open pages, read them as accessibility snapshots rather than raw HTML,
click through flows, and export screenshots or PDFs. Nothing needs to be
started first — `pinchtab nav <url>` launches the control-plane server on
demand. For a server that survives a container restart, run `pinchtab daemon
install` inside the container.

As with any isx skill, Claude Code picks it up automatically through the
`~/.claude/skills` symlink; other agents get a working `pinchtab` CLI but have
to be pointed at `~/.agents/skills/pinchtab/SKILL.md` themselves.

PinchTab ships locked down to locally hosted sites, which makes it useless for
an agent doing research, so this tool opens the allowlist (`allowed_domains`
defaults to `*`) on the isx premise that the container is the blast radius.
Everything else stays at PinchTab's recommended defaults — IDPI strict mode
enforcing, the content guard wrapping page text as untrusted, sensitive
endpoint families off — and `pinchtab security` inside the container reports
the allowlist as the single deviation.

Know what the open allowlist means here specifically. The host proxy injects
real credentials for intercepted domains, and a loaded page's *own JavaScript*
reaches them: a cross-origin `fetch('https://api.github.com/user')` from a
visited page comes back authenticated, because GitHub's API allows every
origin. That is exposure the agent never chose, and it is a property of putting
any browser in a credential-injecting container — not of PinchTab.

`allowed_domains` gates **navigation**, so narrowing it decides which pages get
to run JavaScript at all. It does not contain one that already did: a page
served from localhost still read the GitHub API with the allowlist set to
`localhost` alone. Narrow it to keep hostile pages out, not to declaw them:

```yaml
tools:
  - pinchtab: {allowed_domains: "quarkus.io,*.quarkus.io,github.com"}
```

(`*.quarkus.io` matches subdomains but not the apex, so list both.)

Set `mcp: true` to also register PinchTab's MCP server with Claude Code and
install the `pinchtab-mcp` skill; the default is the CLI path, which works for
any agent that can run a shell command.

## Contributing

To add a new template, create a YAML file in `images/` — see the [incus-spawn documentation](https://github.com/Sanne/incus-spawn) for the image definition format. Custom tools go in `tools/`.
