# Model Effort Suffix Design

## Goal

Allow an OpenAI model mapping to pin its final reasoning effort by appending a non-empty parenthesized suffix to the model ID, for example `gpt-5.6-sol(max)`.

## Request transformation

At the shared Anthropic-to-OpenAI transformation boundary, parse the final parenthesized suffix once:

- `gpt-5.6-sol(max)` becomes model `gpt-5.6-sol` and reasoning effort `max`.
- `gpt-5.6-sol(foo)` becomes model `gpt-5.6-sol` and reasoning effort `foo`.
- The suffix value is forwarded unchanged. CC Switch does not validate or clamp it, so future upstream effort values work without a CC Switch release and unsupported values produce an upstream API error.
- An explicit model suffix overrides Claude Code `output_config.effort` and the adaptive-thinking fallback.
- Without a suffix, the GPT-generation capability mapping introduced by PR #5267 remains unchanged.
- Empty or incomplete suffixes such as `gpt-5.6-sol()` and `gpt-5.6-sol(max` are not parsed and the original model ID is preserved.

## Scope

Use one parser shared by both OpenAI conversion paths. Do not change provider configuration UI, non-OpenAI protocols, Codex model catalogs, or workflow orchestration.

## Verification

Tests must prove:

1. The suffix is removed from the forwarded model ID.
2. The suffix overrides a conflicting Claude Code effort.
3. Unknown suffix values are forwarded unchanged.
4. Model-specific clamping does not alter an explicit suffix.
5. Requests without a valid suffix retain PR #5267 behavior.
