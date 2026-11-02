# ADR 0002: Launch industry is real estate

- Status: accepted (default from the development plan, D1; revisit before Week 3)
- Date: 2026-11-02

## Decision
The beta launches for **real-estate agents and agencies** (listings → inquiries → viewings).
Sales and dropshipping are built after the pilot as additional industry modules.

## Consequences
- Week 3 onboarding, listings and the fair-housing check are real-estate specific.
- The industry module interface (`packages/industry/<segment>`) must stay generic so the next two
  segments plug in without schema changes to core tables.
