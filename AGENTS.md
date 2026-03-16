# Repository Guidelines

## Project Structure & Module Organization
- Cargo workspace with 14 crates in `crates/` (kernel, runtime, api, channels, memory, types, skills, hands, extensions, wire, cli, migrate, desktop, xtask).
- Agent templates live in `agents/` (each folder holds an `agent.toml`). Docs sit in `docs/`, shared assets in `public/`, helper scripts in `scripts/`, and deployment helpers in `deploy/`.
- `packages/whatsapp-gateway` contains the WhatsApp adapter service; `openfang.toml.example` is the base config; `xtask` hosts build automation tasks.

## Build, Test, and Development Commands
- `cargo build --workspace --lib` builds all libraries; `cargo build --release -p openfang-cli` produces the release CLI binary.
- `cargo run --bin openfang -- start` runs the daemon from source; `cargo run -- doctor` verifies local setup.
- `cargo test --workspace` runs the full suite; `cargo test -p <crate>` narrows scope when iterating.
- `cargo clippy --workspace --all-targets -- -D warnings` enforces lint cleanliness.
- `cargo fmt --all -- --check` enforces formatting; run without `--check` to apply fixes.

## Coding Style & Naming Conventions
- Format with rustfmt defaults and keep clippy at zero warnings.
- Public items need `///` docs; avoid `unwrap()` in library code; prefer `thiserror` for error types.
- Naming: types `PascalCase`, functions `snake_case`, constants `SCREAMING_SNAKE_CASE`, crates `openfang-{name}`.
- Reuse workspace dependencies where possible; justify any new dependency in the PR.

## Testing Guidelines
- Tests live alongside crates and in `tests/`; isolate filesystem/network resources (`tempfile::TempDir`, random ports).
- LLM-dependent tests read provider keys from env (e.g., `GROQ_API_KEY`, `ANTHROPIC_API_KEY`) and skip if absent.
- Every feature or fix needs accompanying tests; mirror reported bugs with regression cases.

## Commit & Pull Request Guidelines
- Branch from `main` with names like `feat/<topic>` or `fix/<issue>`.
- Commit messages are imperative and specific: “Add Matrix channel adapter”, “Fix session restore crash”.
- Before opening a PR: run `cargo fmt --all`, `cargo clippy --workspace --all-targets -- -D warnings`, and `cargo test --workspace`.
- PRs should focus on one concern, describe what changed and why, link issues, and add screenshots for UI/dashboard tweaks. CI must pass and a maintainer review is required before merge.

## Security & Configuration Tips
- Never commit secrets; use env vars for API keys. Copy `openfang.toml.example` when creating local configs.
- Keep approval gates and guardrails intact in Hands and channel adapters; do not disable security checks for convenience.
