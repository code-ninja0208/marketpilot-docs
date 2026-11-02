# ADR 0006: Launch pricing

- Status: accepted (D6); free-tier AI cost cap set in W0-3
- Date: 2026-11-02

| | Free | Starter | Growth | Business |
|--|--|--|--|--|
| Price / month | $0 | $19 | $59 | $149 |
| Businesses | 1 | 1 | 2 | 5 |
| Connected social accounts | 1 | 3 | 8 | 20 |
| AI credits / month | 30 | 200 | 800 | 2,500 |
| Scheduled posts / month | 5 | 30 | 150 | 500 |
| Automatic workflows | — | 1 | 5 | 20 |
| Team seats | 1 | 1 | 3 | 10 |

Source: the latest table in the planning conversation (PDF p.61); it replaces the two earlier
tables (review item A1). Limits are implemented as data in `packages/domain/plans.ts` (W11-1).
