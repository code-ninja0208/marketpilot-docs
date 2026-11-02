# ADR 0004: One AI provider behind an adapter

- Status: accepted (D3); model choice deferred to Week 5
- Date: 2026-11-02

## Decision
All model calls go through `packages/ai` (`generateStructured(schema, prompt, opts)`). Business
code never imports a provider SDK. The provider and model are chosen in Week 5 by running the
eval suite against candidates.
