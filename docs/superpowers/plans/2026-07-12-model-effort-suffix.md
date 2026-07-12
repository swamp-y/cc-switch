# Model Effort Suffix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Parse a final non-empty model suffix such as `(max)`, remove it from the forwarded model ID, and transparently use its contents as the final reasoning effort.

**Architecture:** Add one borrowing parser in `transform.rs`. Both OpenAI Chat and Responses converters use its parsed model and explicit effort; when no valid suffix exists, they retain PR #5267's existing effort resolution.

**Tech Stack:** Rust, `serde_json`, existing Cargo unit tests.

## Global Constraints

- Explicit suffix effort overrides Claude Code `output_config.effort` and thinking fallback.
- Suffix values are forwarded unchanged without validation or clamping.
- Empty or incomplete suffixes are not parsed.
- No new dependency and no behavior change outside OpenAI conversion.

---

### Task 1: Parse and apply explicit model effort suffixes

**Files:**
- Modify: `src-tauri/src/proxy/providers/transform.rs`
- Modify: `src-tauri/src/proxy/providers/transform_responses.rs`
- Test: inline test modules in both files

**Interfaces:**
- Produces: `parse_model_effort_suffix(model: &str) -> Option<(&str, &str)>`
- Consumes: existing `resolve_reasoning_effort(body: &Value) -> Option<&'static str>` when no suffix exists

- [ ] **Step 1: Write failing parser and conversion tests**

Add tests proving `gpt-5.6-sol(max)` becomes model `gpt-5.6-sol`, that `max` overrides `output_config.effort=xhigh`, that `foo` is forwarded unchanged, and that incomplete/empty suffixes remain unchanged.

- [ ] **Step 2: Run focused tests and verify RED**

Run:

```bash
cargo test --manifest-path src-tauri/Cargo.toml model_effort_suffix -- --nocapture
```

Expected: FAIL because the forwarded model still contains the suffix and the Claude effort still wins.

- [ ] **Step 3: Implement the minimum shared parser and converter wiring**

Add a parser equivalent to:

```rust
pub(crate) fn parse_model_effort_suffix(model: &str) -> Option<(&str, &str)> {
    let open = model.rfind('(')?;
    let effort = model.get(open + 1..model.len().checked_sub(1)?)?;
    (model.ends_with(')') && open > 0 && !effort.is_empty())
        .then_some((&model[..open], effort))
}
```

In each converter, use the parsed base model for the outgoing `model` field and use the parsed effort before calling `resolve_reasoning_effort`.

- [ ] **Step 4: Run focused and existing converter suites and verify GREEN**

Run:

```bash
cargo test --manifest-path src-tauri/Cargo.toml model_effort_suffix -- --nocapture
cargo test --manifest-path src-tauri/Cargo.toml proxy::providers::transform::tests
cargo test --manifest-path src-tauri/Cargo.toml proxy::providers::transform_responses::tests
cargo fmt --manifest-path src-tauri/Cargo.toml -- --check
git diff --check
```

Expected: all commands pass.

- [ ] **Step 5: Commit the implementation**

```bash
git add src-tauri/src/proxy/providers/transform.rs src-tauri/src/proxy/providers/transform_responses.rs
git commit -m "feat(proxy): support model effort suffix overrides"
```

### Task 2: Publish the stacked PR

**Files:**
- No source changes

**Interfaces:**
- Consumes: branch `feat/model-effort-suffix` based on `fix/claude-openai-effort-mapping`
- Produces: a GitHub PR targeting `swamp-y:fix/claude-openai-effort-mapping` and referencing farion1231/cc-switch#5267

- [ ] **Step 1: Push the branch**

```bash
git push -u fork feat/model-effort-suffix
```

- [ ] **Step 2: Create the stacked PR**

Create the PR with base `swamp-y:fix/claude-openai-effort-mapping`, document precedence and transparent unknown-effort forwarding, and link #5267.
