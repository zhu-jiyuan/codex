# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> Most repo-specific guidance already lives in `AGENTS.md` at the repo root. Read it first — it is loaded by `codex` itself and assumed to be in force. This file captures the parts most useful to a future Claude Code instance landing fresh, plus the cross-references needed to navigate the workspace.

## What this project is

This is the **OpenAI Codex** monorepo — a local-first coding agent that runs as a TUI, a headless CLI, an IDE backend, or an MCP server. The Rust implementation under `codex-rs/` is the maintained, default surface; the small npm package at `codex-cli/` is just a distribution shim that ships the prebuilt Rust binaries. Python (`sdk/python/`) and TypeScript (`sdk/typescript/`) SDKs talk to the Rust app-server.

Repo top level:

```
codex-rs/         Cargo workspace, ~110 crates — primary code
codex-cli/        npm package wrapper around prebuilt Rust binaries (@openai/codex)
sdk/python/       Python SDK (uv-managed, ruff-formatted)
sdk/typescript/   TypeScript SDK (tsup-built, jest-tested)
docs/             user-facing CLI docs (config, sandbox, install, etc.)
scripts/          repo-level scripts (lint helpers, packaging)
tools/            workspace tooling (e.g. argument-comment-lint dylint)
justfile          common command runner — `set working-directory := "codex-rs"`
```

## Build & test toolchain

- **Rust 1.93.0** (`codex-rs/rust-toolchain.toml`), **edition 2024** for all workspace crates.
- **Two build systems live in parallel**: Cargo for day-to-day dev; Bazel for reproducible release builds and RBE-backed CI. Dependency changes must keep both in sync (see "Cross-build hygiene" below).
- **Node 22+, pnpm 10.33+** (`package.json` engines). Repo-root `package.json` only owns Prettier-formatting of JSON/MD/YML and a schema-write helper; pnpm workspace covers `codex-cli`, `codex-rs/responses-api-proxy/npm`, and `sdk/typescript`.
- **Python SDK** uses **uv** with the project at `sdk/python/` (the justfile's `fmt` recipe runs `uv run --frozen --project ../sdk/python ...`).

## Commands

All `just` recipes default to running inside `codex-rs/` (`set working-directory := "codex-rs"`). Run `just` from the repo root, **not** from inside `codex-rs/`, unless using a recipe marked `[no-cd]`.

```bash
# Run codex locally
just codex                                     # `cargo run --bin codex --`
just c "explain this repo"                     # alias for codex
just exec PROMPT                               # `codex exec` non-interactive mode
just tui-with-exec-server                      # tui pointed at a local exec-server

# Format & lint Rust (and Python SDK)
just fmt                                       # cargo fmt + ruff for sdk/python — run after every Rust change
just fix -p <crate>                            # cargo clippy --fix --tests — scope with -p, run before finalizing
just clippy -p <crate>                         # clippy without --fix

# Tests
just test                                      # cargo nextest run --no-fail-fast — needs cargo-nextest
cargo test -p codex-tui                        # single crate; preferred for routine local runs
cargo test -p codex-tui -- snapshot_name       # single test
# Avoid --all-features for local runs (blows up target/ disk usage); use only for full feature coverage.

# Bazel
just bazel-codex -- <args>                     # `bazel run //codex-rs/cli:codex` with cwd preserved
just bazel-test                                # bazel test //... (excludes argument-comment-lint)
just bazel-clippy                              # clippy via Bazel
just bazel-lock-update                         # refresh MODULE.bazel.lock after Cargo.toml/Cargo.lock changes
just bazel-lock-check                          # verify lockfile is in sync (CI runs this)
just build-for-release                         # bazel build //codex-rs/cli:release_binaries --config=remote

# Schema regeneration (run when shapes change)
just write-config-schema                       # after ConfigToml changes — updates codex-rs/core/config.schema.json
just write-app-server-schema                   # after app-server v2 API changes
just write-app-server-schema --experimental    # for experimental API fixtures
just write-hooks-schema                        # after codex-hooks schema changes (also pnpm write-hooks-schema)

# Lints with their own setup
just argument-comment-lint                     # Bazel-powered argument-comment dylint; slow first run, <15s after
cargo insta pending-snapshots -p codex-tui     # list pending UI snapshot updates
cargo insta show -p codex-tui path/file.snap.new
cargo insta accept -p codex-tui                # accept all pending snapshots in the crate

# Repo-root JS/MD/YAML formatting
pnpm format                                    # prettier --check
pnpm format:fix                                # prettier --write
```

`cargo` workspace commands run from `codex-rs/`. If you spawn long-running Rust builds (e.g. `just fix`, `just test`), let them finish — the lock contention makes them slow but they are not stuck. Do not kill by PID.

## High-level architecture

### `codex-rs/` workspace shape

110+ crates, all prefixed `codex-` (the crate in `core/` is `codex-core`, the crate in `tui/` is `codex-tui`, etc.). Internal deps go through `[workspace.dependencies]` in `codex-rs/Cargo.toml` so paths stay in one place.

Top-of-the-stack crates a Claude Code instance is most likely to touch:

| Crate | Role |
| --- | --- |
| `cli/` | The `codex` multitool binary. `src/main.rs` wires every subcommand: tui (default), `exec`, `mcp`/`mcp-server`, `app`, `cloud`, `responses-api-proxy`, `login`, `sandbox`, `debug`, etc. |
| `tui/` | Ratatui-based fullscreen UI. Default surface when `codex` is run with no subcommand. **Heavy do-not-grow zone** — see "Module-size discipline" below. |
| `exec/` | Headless non-interactive runner (`codex exec PROMPT`). Reads stdin, prints to stdout, exits. |
| `core/` | Business logic — model client, conversation/thread state, tool dispatch, MCP integration, sandboxing glue. **Resist adding code here** (see below). |
| `protocol/` | Wire types shared between core, app-server, exec, tui. |
| `app-server/` + `app-server-protocol/` + `app-server-daemon/` + `app-server-client/` + `app-server-transport/` | The app-server is the IDE-facing protocol layer. All new API surface goes in v2 (see "App-server v2 API rules"). |
| `mcp-server/` + `codex-mcp/` + `rmcp-client/` | MCP server-mode for Codex, plus the client used to connect Codex to external MCP servers. Mutations to MCP tools/tool-calls should go through `codex-rs/codex-mcp/src/mcp_connection_manager.rs` rather than plumbing through new layers. |
| `state/` | SQLite-backed state DB. `just log` tails its logs. |
| `rollout/` + `rollout-trace/` | Session rollout persistence + replay (`replay_bundle`, reduced-state files). |
| `sandboxing/`, `linux-sandbox/`, `bwrap/`, `landlock/`, `execpolicy/`, `process-hardening/` | Sandbox enforcement (Seatbelt on macOS, Landlock + bubblewrap on Linux, dedicated `windows-sandbox-rs` on Windows). |
| `hooks/`, `core-skills/`, `core-plugins/`, `skills/`, `plugin/` | Lifecycle hooks, agent skills, plugin runtime. |
| `external-agent-sessions/`, `cloud-tasks/`, `cloud-tasks-client/` | Remote/cloud agent integrations. |
| `responses-api-proxy/` | Local proxy for the Responses API. |
| `agent-graph-store/`, `agent-identity/`, `thread-store/`, `message-history/` | Persistence layers for agent/thread/message data. |
| `feedback/`, `analytics/`, `otel/` | Telemetry. |
| `utils/*` | Small focused utility crates — cargo-bin, json-to-toml, output-truncation, fuzzy-match, template, etc. |

The `cli` crate's `src/main.rs` is the most useful map: it imports from roughly every top-level subcommand crate and shows the wiring.

### `codex-core` is a bloat hazard

From `AGENTS.md`: **resist adding code to `codex-core`**. It is already the largest crate. When you reach for it, first check whether an existing peer crate (e.g. `connectors/`, `context/`, `hooks/`, `model-provider/`) is a better home — or whether the work warrants a new workspace crate. Push back on PRs that grow `codex-core` unnecessarily.

### Module-size discipline (especially in `tui`)

- Target Rust modules under **500 LoC** (excluding tests). At ~800 LoC, prefer adding a new module over extending the existing one — unless there is a strong documented reason.
- Tests, type docs, and invariants should travel with the code they describe when you split a module.
- **High-touch files to specifically avoid growing** (per `AGENTS.md`):
  - `codex-rs/tui/src/app.rs`
  - `codex-rs/tui/src/bottom_pane/chat_composer.rs`
  - `codex-rs/tui/src/bottom_pane/footer.rs`
  - `codex-rs/tui/src/chatwidget.rs` — do not add standalone methods unless trivial; keep it focused on orchestration
  - `codex-rs/tui/src/bottom_pane/mod.rs`

## Rust style (deltas from idiomatic defaults)

The workspace `[workspace.lints.clippy]` block in `codex-rs/Cargo.toml` denies a long list of footguns (`unwrap_used`, `expect_used`, `redundant_clone`, `uninlined_format_args`, `await_holding_lock`, etc.). Local clippy overrides live in `codex-rs/clippy.toml` — notably the `disallowed-methods` list that forbids `Color::Rgb`/`Color::Indexed` and the ratatui `.white()`/`.black()`/`.yellow()` Stylize helpers (use ANSI colors per the style guide instead). `allow-{expect,unwrap}-in-tests = true` so test code may use those.

- **Inline format args** — `format!("x={x}")`, not `format!("x={}", x)`. Enforced by clippy.
- **Collapse nested `if`** when clippy's `collapsible_if` would fire.
- **Method references over closures** — `.map(Foo::bar)` not `.map(|x| Foo::bar(x))`.
- **No `bool`/ambiguous `Option` positional params.** Prefer enums, named methods, newtypes. If you genuinely can't refactor and need to pass an opaque literal positionally, use `argument_comment_lint` format: `foo(/*enabled=*/true, /*timeout=*/None)`. Param name must match the callee signature exactly. String/char literals are exempt unless the comment adds clarity. Run `just argument-comment-lint` (slow first run; CI catches it anyway).
- **Exhaustive `match`** — avoid wildcard arms when you can enumerate.
- **No `#[async_trait]` and no `#[allow(async_fn_in_trait)]`.** Use RPITIT with explicit `Send` bounds: `fn foo(&self, ...) -> impl std::future::Future<Output = T> + Send;`. Implementations may still write `async fn foo(...)` as long as they satisfy the contract.
- **Tests**: `pretty_assertions::assert_eq` over `assert_eq!`. Prefer deep object equality over field-by-field. Do not mutate process env in tests — pass env-derived flags/dependencies from above.
- **Doc comments on new traits.** Explain the role and how implementations are expected to use it.
- **Private modules + explicit `pub` API.** Prefer narrow re-exports.
- **No single-use helper methods.** If a helper is referenced once, inline it.
- **Spawning workspace binaries in tests**: use `codex_utils_cargo_bin::cargo_bin("...")` (works under both Cargo and Bazel runfiles), not `assert_cmd::Command::cargo_bin` or `escargot`. For fixtures, use `codex_utils_cargo_bin::find_resource!` instead of `env!("CARGO_MANIFEST_DIR")`.

## TUI conventions (ratatui)

See `codex-rs/tui/styles.md` for the canonical style guide. Highlights:

- Use Stylize helpers: `"text".into()` for plain spans, `"text".dim()`, `.bold()`, `.cyan()`, `.green()`, `.magenta()`, `.red()`, `.italic()`, `.underlined()`. Prefer these over manual `Span::styled` + `Style`.
- Computed styles may use `Span::styled` or `Span::from(text).set_style(style)` — both are acceptable when the style is runtime-computed.
- **Color discipline**: avoid `.white()`/`.black()` (use default fg or `dim`/`bold`); avoid `.yellow()` and `.blue()`. Cyan = user input/selection/status, green = success/additions, red = errors/deletions, magenta = Codex itself. The default terminal theme is usually the right answer.
- Chain Stylize calls: `url.cyan().underlined()`.
- Lines: `vec![...].into()` when the target type is obvious; `Line::from(vec![...])` when inference would otherwise force a type annotation. Don't churn between equivalent forms.
- Wrap plain strings with `textwrap::wrap`. Wrap ratatui `Line`s with the helpers in `codex-rs/tui/src/wrapping.rs` (`word_wrap_line`, `word_wrap_lines`). Prefer `initial_indent`/`subsequent_indent` from `RtOptions` over custom indent logic. Prefix line lists with `prefix_lines` from `line_utils`.
- **Snapshot tests are required for any user-visible UI change.** Use `insta`; review `.snap.new` files via `cargo insta show -p codex-tui ...` before `cargo insta accept`. Install with `cargo install --locked cargo-insta` if missing.

## App-server v2 API rules (`codex-rs/app-server-protocol/`)

- All active API work goes in v2 (`src/protocol/v2.rs` + `src/protocol/common.rs`). Do not extend v1.
- Naming: `*Params` (requests), `*Response`, `*Notification`. RPC methods are `<resource>/<method>` with **singular** resource (`thread/read`, `app/list`).
- Wire is camelCase: `#[serde(rename_all = "camelCase")]` on v2 types. Exception: config RPC payloads mirror `config.toml` and stay snake_case.
- All v2 types must set `#[ts(export_to = "v2/")]` so generated TypeScript lands in the correct namespace. Keep Serde and ts-rs renames aligned (`#[serde(rename = "...")]` + `#[ts(rename = "...")]`).
- Discriminated unions use explicit tagging in both: `#[serde(tag = "type", ...)]` and `#[ts(tag = "type", ...)]`.
- **Never** use `#[serde(skip_serializing_if = "Option::is_none")]` on v2 payloads. Narrow exception: client→server requests that intentionally have no params may write `params: #[ts(type = "undefined")] #[serde(skip_serializing_if = "Option::is_none")] Option<()>`.
- Client→server `*Params` fields: every optional field annotated with `#[ts(optional = nullable)]`. Optional collections must be `Option<Vec<...>>` + `#[ts(optional = nullable)]` — **don't** use `#[serde(default)]` to model optional collections. Booleans that default to false use `#[serde(default, skip_serializing_if = "std::ops::Not::not")] pub field: bool` instead of `Option<bool>`.
- IDs at the API boundary: plain `String` (do UUID parsing internally if needed). Timestamps: `i64` unix seconds, named `*_at` (`created_at`, `updated_at`, `resets_at`).
- New list methods default to cursor pagination: `cursor: Option<String>` + `limit: Option<u32>` on request; `data: Vec<...>` + `next_cursor: Option<String>` on response.
- Experimental surface uses `#[experimental("method/or/field")]` + `ExperimentalApi` derive when needed, and `inspect_params: true` in `common.rs` for partial experimental methods. Don't pad with boilerplate tests asserting marker presence on individual fields — rely on schema generation and behavioral coverage.
- After any v2 shape change: run `just write-app-server-schema` (and `--experimental` if applicable). Update `app-server/README.md` when behavior changes. Validate with `cargo test -p codex-app-server-protocol`.

## Integration test conventions

For end-to-end Codex tests, use `core_test_support::responses`:

- `mount_sse_once(&server, sse(vec![...]))` returns a `ResponseMock`. Hold onto it.
- Assert request bodies with `ResponseMock::single_request()` (one POST) or `ResponseMock::requests()` (all).
- `ResponsesRequest` has helpers: `body_json`, `input`, `function_call_output`, `custom_tool_call_output`, `call_output`, `header`, `path`, `query_param`. Use them instead of raw JSON poking.
- Build SSE with the `ev_*` constructors (`ev_response_created`, `ev_function_call`, `ev_completed`, ...) and `sse(...)`.
- Prefer `wait_for_event` over `wait_for_event_with_timeout`. Prefer `mount_sse_once` over `mount_sse_once_match` / `mount_sse_sequence`.

## Sandboxing — touch with extreme care

**Never add or modify code referencing `CODEX_SANDBOX_NETWORK_DISABLED_ENV_VAR` or `CODEX_SANDBOX_ENV_VAR`.** These are sandbox-detection hooks for tests that know they cannot run inside the sandbox:

- `CODEX_SANDBOX_NETWORK_DISABLED=1` is set whenever the agent's `shell` tool runs. Tests check it to early-exit.
- `CODEX_SANDBOX=seatbelt` is set on processes spawned through `/usr/bin/sandbox-exec`. Integration tests that need to run Seatbelt themselves can't run *under* Seatbelt, so they early-exit when this is set.

Platform sandbox notes (full detail in `codex-rs/core/README.md`):

- **macOS**: Seatbelt via `/usr/bin/sandbox-exec`. Workspace-write keeps `.git`, the resolved `gitdir:` target, and `.codex` read-only.
- **Linux**: bubblewrap (`bwrap`) + landlock. Prefers system `bwrap` on `PATH` (falls back to bundled `codex-resources/bwrap`); switches to a no-`--argv0` compat path on older `bwrap`. WSL2 works; WSL1 is rejected (no user namespaces).
- **Windows**: dedicated `windows-sandbox-rs` crate; only pulled in `cfg(target_os = "windows")`.

## Cross-build hygiene (Cargo ↔ Bazel)

- After any `Cargo.toml`/`Cargo.lock` change: `just bazel-lock-update` from repo root, then include the `MODULE.bazel.lock` update in the same change. Verify with `just bazel-lock-check`.
- Bazel does **not** automatically expose source-tree files at Rust compile time. If you add `include_str!`, `include_bytes!`, `sqlx::migrate!`, or any other build-time file or directory read, update the crate's `BUILD.bazel` (`compile_data`, `build_script_data`, or `data` for tests). Cargo will be silent; Bazel will fail.
- Release builds: `bazel build //codex-rs/cli:release_binaries --config=remote` (see `just build-for-release`).

## Config & schema regeneration

If you change `ConfigToml` or any of its nested types, run `just write-config-schema` and commit the updated `codex-rs/core/config.schema.json` in the same change. Same pattern for hooks (`just write-hooks-schema` / `pnpm write-hooks-schema`) and the app-server protocol (`just write-app-server-schema`).

## Docs

- `docs/contributing.md` — external contributions are by invitation only; PRs from uninvited contributors are closed without review. (This matters for review tone, not the work itself.)
- `docs/install.md` — Rust toolchain bootstrap, `cargo install --locked just` / `cargo-nextest`, `RUST_LOG` defaults (`codex_core=info,codex_tui=info,codex_rmcp_client=info` for TUI; logs at `~/.codex/log/codex-tui.log`; override with `-c log_dir=...`).
- `docs/config.md` — links to the canonical config docs; only the lifecycle-hooks note lives in-repo.
- `docs/sandbox.md`, `docs/exec.md`, `docs/execpolicy.md`, `docs/skills.md`, `docs/slash_commands.md`, `docs/agents_md.md` — user-facing reference; not contributor documentation.
- **Do not** add general product or user-facing documentation under `docs/`. The official Codex docs live elsewhere. The exception is `app-server/README.md`, which is the source of truth for the app-server protocol.

## After-change checklist

1. `just fmt` — always.
2. `just fix -p <crate>` (scope by `-p` to avoid workspace-wide clippy). Only run unscoped `just fix` if you touched shared crates.
3. `cargo test -p <crate>` for the changed crate. If you touched `common`, `core`, or `protocol`, follow up with `just test` (full nextest run) — but ask the user before kicking off the full suite.
4. If `ConfigToml`, hooks schema, or app-server protocol shapes changed: regenerate the schema fixture and commit it.
5. If `Cargo.toml`/`Cargo.lock` changed: `just bazel-lock-update` + verify with `just bazel-lock-check`.
6. If UI changed: update or add `insta` snapshots (`cargo test -p codex-tui` then `cargo insta accept -p codex-tui` after review).
7. Do **not** re-run tests after `just fix` or `just fmt` — they don't change behavior.
