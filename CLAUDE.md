# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Rig is a Rust library (v0.41.0, MIT license) for building scalable, modular, and ergonomic LLM-powered applications. It provides provider-agnostic traits for AI model interaction with 20+ model providers and 10+ vector stores under unified interfaces. Built by Playgrounds: https://github.com/0xPlaygrounds/rig

## Key Files

- `AGENTS.md` -- Repository-specific engineering rules for reading, editing, testing, and documenting code. Read before making changes.
- `CONTRIBUTING.md` -- Contributor-facing policy, PR expectations, accountability guidance.
- `tests/README.md` -- Test targets, cassette commands, record/replay doctrine.
- `Cargo.toml` -- Workspace root: features, lint rules (clippy deny on unwrap/expect/todo/unimplemented/panic), profile release LTO.
- `src/lib.rs` -- Public facade; re-exports all modules behind feature gates.
- `rust-toolchain.toml` -- Rust 1.94.0 with wasm32 target.
- `.pre-commit-config.yaml` -- pre-commit hooks: trailing-whitespace, end-of-file-fixer, cargo fmt/clippy, commitizen.
- `.github/workflows/ci.yaml` / `.github/workflows/cd.yaml` -- CI/CD pipelines.

## Architecture

**Monorepo with ~30 crates:**

| Layer | Path | Purpose |
|-------|------|---------|
| Facade | `.` (root) | Re-exports everything under `use rig::prelude::*`; feature-gated companion crates |
| Core | `crates/rig-core` | Provider-neutral contracts: `CompletionModel`, `EmbeddingModel`, `VectorStoreIndex`, `Tool` traits; built-in providers; portable tools; memory/vector-store traits; loaders |
| Agent runtime | `crates/rig-agent` | Classic agent orchestration: builder, prompt/streaming traits, typed hooks, contextual tools, extraction, serializable `AgentRun` state machine |
| Derive macros | `crates/rig-derive` | `#[rig_tool]` derive macro |
| Providers | `crates/rig-*` | 25+ provider implementations (OpenAI, Anthropic, Gemini gRPC, Vertex AI, Bedrock, Groq, Cohere, Mistral, Ollama, Together, Perplexity, DeepSeek, HuggingFace, Azure, etc.) |
| Vector stores | `crates/rig-*` | 20+ vector-store integrations (LanceDB, Milvus, MongoDB, Neo4j, PostgreSQL, Qdrant, SQLite, SurrealDB, ScyllaDB, S3Vectors, Cloudflare Vectorize, HelixDB) |
| Extras | `crates/rig-candle`, `crates/rig-fastembed`, `crates/rig-memory` | Local CPU inference (Candle), FastEmbed embeddings, memory policies |
| Examples | `examples/*` | 60+ example packages demonstrating agentic workflows, RAG, tool use, streaming, OTEL tracing, etc. |

**Core traits:** `CompletionModel`, `EmbeddingModel`, `VectorStoreIndex`, `Tool`. Use these — not parallel abstractions. Configurable public types follow the builder pattern (e.g., `client.agent(model).preamble(...).tool(t).temperature(0.8).build()`).

**Feature gating:** Every provider/vector-store is opt-in via Cargo features (`features = ["lancedb", "fastembed"]`). Check `Cargo.toml` and `src/lib.rs` before documenting or changing exposed integrations.

**WASM-first:** Targets `wasm32-unknown-unknown`. Use `WasmCompatSend`/`WasmCompatSync` instead of raw `Send`/`Sync`; `WasmBoxedFuture` for boxed futures. Platform-specific error type bounds for boxed `dyn Error`.

## Commands

### Build & Format
```bash
cargo fmt
cargo clippy --all-targets --all-features
cargo doc --workspace --no-deps
```

### Run Tests (default members only — no external services)
```bash
cargo test                              # default-members + root crate
cargo test -p rig                       # core tests
cargo test -p rig --all-features        # all root features
cargo test -p rig --test core           # provider-agnostic core tests
cargo test -p rig --all-features --test integrations  # integration tests (with feature flags)
```

### Run targeted crate tests
```bash
cargo test -p rig-core                  # core unit tests
cargo test -p rig-agent                 # agent runtime tests
cargo test -p rig-derive                # trybuild + dependency-rename fixtures (requires special toolchain)
```

### Cassette provider tests (replay — no API keys needed)
```bash
cargo test -p rig --all-features --test openai openai::cassette -- --nocapture --test-threads=1
cargo test -p rig --all-features --test anthropic anthropic::cassette -- --nocapture --test-threads=1
cargo test -p rig --all-features --test gemini gemini::cassette -- --nocapture --test-threads=1
# ... see tests/README.md for full list

# Run single cassette test:
cargo test -p rig --all-features --test gemini streaming_tools_smoke -- --nocapture --test-threads=1
```

### Record cassettes (requires provider API keys)
```bash
RIG_PROVIDER_TEST_MODE=record \
  cargo test -p rig --all-features --test openai openai::cassette -- --nocapture --test-threads=1
```

### Live provider tests (ignored by default, requires credentials)
```bash
cargo test -p rig --all-features --test openrouter -- --ignored --nocapture --test-threads=1
```

### Run examples
```bash
cargo run -p <example-package>          # e.g., cargo run -p simple-chat
cargo run -p <example-package> --release # release builds
```

## Engineering Rules (from AGENTS.md)

- **Read before changing** — study existing implementation first. Keep changes scoped to the request.
- **Prefer existing Rig traits, builders, modules** over new abstractions. No TODOs/stubs/speculative APIs.
- **Never commit/push/open PRs** unless explicitly asked.
- **Error handling:** Use `thiserror` enums (never `String`). No `unwrap()`/`expect()` on fallible ops. Prefer `?`.
- **Clippy rules forbid:** `dbg!`, `expect_used`, `unwrap_used`, `todo`, `unimplemented`, `panic`, `panic_in_result_fn`, `await_holding_lock/refcell_ref`, `indexing_slicing`, `db_macro`.
- **Provider changes:** Study closest existing provider (`crates/rig-core/src/providers/openai/`). Include: extension/builder types, capabilities declaration, `from_env`/`from_val`, model constants, request/response conversion, streaming support, error preservation, telemetry spans. Never add fields not in the real API.
- **Cassette regression tests:** New provider behavior gets a cassette-backed test. Cassette files under `tests/cassettes/<provider>/`. Review for no API keys/tokens. Assert on request boundaries (mock 404 = request-shape regression).
- **Vector stores in companion crates.** Implement both `top_n` and `top_n_ids`. Return `VectorStoreError`, not string errors.
- **Agent hooks:** Per-run lifecycle observers. Streaming and non-streaming surfaces share the same driver (`drive_agent`). Register observe-only hooks before steering hooks (stop actions short-circuit).
