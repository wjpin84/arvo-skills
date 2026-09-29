---
name: arvo-platform
description: Working across the Arvo repositories. Use when adding or changing code in arvo-engine, arvo-ui, arvo-desktop, arvo-engine-api, arvo-extension-api or arvo-adrs, when deciding which crate or repo a change belongs in, when launching the engine or window locally, running the test suites, cutting a release, or writing an ADR.
---

# Working in Arvo

Arvo is an AI-native financial research platform: NautilusTrader executes,
Arvo owns the research loop above it. The differentiator is **evaluation,
not execution**. The rule that orders every change:

> Anything that makes a wrong answer look right outranks anything that adds
> capability.

## The repositories

| Repo | What it is | Licence |
|---|---|---|
| `arvo-engine` | The daemon and the CLI (`arvo-engine session …`), the MCP server, every crate that decides anything | Apache-2.0, LGPL Nautilus linked |
| `arvo-engine-api` | The protos and the generated Rust (`arvo-api`, `arvo-client`) and Python (`arvo-client`) bindings. Submodule at `arvo-engine/contract/` | |
| `arvo-extension-api` | What a data source or signal plugin implements. Submodule at `arvo-engine/extension/` | |
| `arvo-ui` | The window, on gpui (Zed's UI crate); holds the finance chart crate | GPL — Zed workbench crates are GPL |
| `arvo-desktop` | The earlier Tauri window; still built | |
| `arvo-adrs` | One ADR per file, one number sequence across every repo, superseded rather than edited. Research notes under `research/` | |
| `arvo-docs` | MkDocs site. Stay on mkdocs-material; upstream said not to migrate | |

Clone the engine with `--recurse-submodules`; both contracts are submodules
and the build reads them.

## Where a change goes

- **Logic** → `crates/arvo-service` in the engine: the service tier every
  front end shares. The window and the MCP server are clients of it and know
  nothing about how a study is run.
- **A new call** → `arvo-engine-api` first (the proto), then the engine
  implements the server trait, then the client. Never hand-write a twin of a
  generated message.
- **Window shapes** → `arvo-ui`.
- **A Nautilus type** → only `crates/arvo-nautilus` may name one.
- **A decision** → an ADR in `arvo-adrs`, linked by URL. Check the highest
  number there before taking the next.
- The engine is the CLI. There is no separate `arvo` binary by decision
  (2026-09-20): a client binary would hold the same control token the
  window holds.

Say **rule** and **ruleset** in anything a person reads. `strategy` stays in
code, protos, MCP and findings.

## Branches, CI, releases

- Work on the **`release`** branch, not master. A branch push starts no CI;
  a PR or a merge does, and Actions credits are short.
- Public repos have branch protection with admin bypass; private ones cannot.
- A release is a tag: bump `[workspace.package].version` in `Cargo.toml`,
  commit, tag `vX.Y.Z`, push the tag. Engine and desktop CI need
  `ARVO_REPOS_TOKEN` for the private submodules.
- After an engine release, bump `app/arvo-runtime/engine-version` in the
  desktop; `tools/fetch_engine.py` pulls that archive.

## Building and testing

```
cargo test --workspace        # run the whole workspace, never --lib alone:
                              # --lib misses the service tests
cargo build -p arvo-engine
cd contract/python && ARVO_ENGINE=../../target/debug/arvo-engine uv run pytest
```

Engine ≈ 910 tests, desktop ≈ 66. The Python contract tests need a built
engine.

**Never `rustfmt` an existing file.** The tree is not rustfmt-clean and
formatting one file rewrites its child modules too.

## Running it

- The window needs `ARVO_ENGINE` pointing at a built `arvo-engine`.
- Closing the window minimises to the tray; the engine keeps running.
- Before launching, check for an orphaned `arvo-plugin-yahoo`: killing an
  engine hard orphans its plugins.
- On Windows, a process spawning a long-lived child from a piped parent
  must mark its own std handles non-inheritable first, or the child holds
  the client's pipes.
- The MCP server and the window talk to **whatever engine `engine.json`
  names**, which may be an older binary than the source tree: a refusal
  such as "expected one of SMA, ATR, MAX, MIN" from an engine built before
  EMA/RSI/MACD landed is the engine's age, not the rule's error. Compare
  the running process's binary date with `git log` before trusting a
  refusal that contradicts the source.
- The project folder is remembered in `%APPDATA%/com.arvo.desktop/project.json`;
  the engine writes `engine.json` and `control.json` beside it. The MCP
  server starts an engine from `ARVO_ENGINE` when none is running.

## Scope since 2026-09-18

Make it work. No new plugin, provider or extensibility surface; keep
Nautilus contained. Before adding protocol, ask which way the call goes —
the plugin boundary is two call directions with no identity machinery.
