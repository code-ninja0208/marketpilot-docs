# ADR 0004: One AI provider behind an adapter — Anthropic Claude

- Status: accepted (D3 2026-11-02; provider chosen 2026-11-03)
- Date: 2026-11-03

## Decision
All model calls go through `@marketpilot/ai` (`generateStructured(provider, schema, prompt, opts)`).
Business code never imports a provider SDK; it depends on the `AiProvider` interface only.

- **Provider: Anthropic Claude** (owner's choice), through the official `@anthropic-ai/sdk`.
- **Model:** `claude-opus-5-5`, effort `medium`; configurable per deployment with `AI_MODEL` and
  `AI_EFFORT` on the worker. Thinking stays on (adaptive); effort is the only depth control.
- **Structured output:** `output_config.format` with the JSON Schema generated from the Zod
  contract (`GenerationOutputSchema`), then Zod validation plus business validation on our side.
- **Refusals:** server-side fallbacks (`fallbacks: "default"`, beta `server-side-fallback-2026-07-01`)
  retry a safety-classifier decline on another model inside the same call. A remaining refusal
  fails the generation with `AI_REFUSED` (no retry, credits released).
- **Retries:** the SDK retries 429/5xx twice; the queue retries provider outages 3 times with
  backoff; validation problems are sent back to the model up to 3 calls per job.
- **No key yet:** the worker defaults to `AI_PROVIDER=demo`, which writes placeholder posts
  marked `[Demo]` from the facts, so every flow works and costs nothing. The key is set only in
  the server's `.env` (`ANTHROPIC_API_KEY`), never in git or the browser.

## Guardrails (independent of the model)
- Prompts take approved, currently valid facts only; user and listing text is placed in escaped
  data sections and declared untrusted (SEC-04).
- Every answer is validated: cited fact ids exist; no number that is not in a fact or in the
  business's own text; platform length/hashtag limits; brand restricted words/claims;
  industry compliance (fair housing for real estate). Failing content is never shown.

## Cost
Credits (one per post) are reserved before the job and settled afterwards (ADR 0006 limits).
Token usage and dollar cost are stored per generation and per usage event, so cost per plan can
be compared with revenue (risk: AI cost above plan revenue).

## Follow-up
Run `pnpm --filter @marketpilot/ai eval` with the real key before the pilot; if Claude Sonnet 5.5
or Haiku 5.5 pass the golden set, the owner may choose one for lower cost (an `AI_MODEL` change).
