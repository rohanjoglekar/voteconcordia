# Design decisions

This document collects Concordia's product and UI decisions and the reasoning
behind them. It exists so the trade-offs — costs, chart honesty, what a member
is actually committing to — stay attached to the choices they produced instead
of living only in commit history.

## Rebalance, not liquidate

The naive executor sells everything and buys the new basket every cycle. It's
simpler to write and it's wrong: it churns positions that didn't change,
multiplies spread costs, and turns every cycle into a taxable event for names
the club still believes in. Concordia diffs instead. Names leaving the basket
are sold, overweights are trimmed, entrants and underweights are topped up,
and a holdover at target weight generates no order at all. The sell and buy
legs run on consecutive trading days because members hold cash accounts —
sells need T+1 settlement before the proceeds can buy. That split is also what
created the stale-basket hazard described in [SAFETY.md](SAFETY.md); the
refusal there is the price of the diff design, and worth it.

## Honest charts

Three rules, all learned the hard way.

**No smoothing.** The price charts originally used Catmull-Rom splines like
most consumer finance UIs. Then came a measurement of what the curve actually
drew:
against a series bounded 0..100, control points reached 116.7, and replayed
against real payloads the rendered path exceeded the data's true high/low on
13 of 15 symbol/range pairs. XOM's 3M range of [131.08, 148.67] rendered as
[130.08, 149.24] — a high $0.57 above anything that traded, on a chart that
pins labeled guide lines to the data's actual min and max. The curve visibly
crossed its own labels. The comment now guarding that code:

```tsx
// web/components/PriceChart.tsx
/** Straight segments through every close — the only shape that cannot lie.
 *  ...
 *  A price chart must never draw a price that did not happen, so there is no
 *  smoothing at all — the vertices ARE the closes. (Robinhood and Yahoo draw
 *  it this way for the same reason.) If softening is ever wanted, it must be
 *  monotone/shape-preserving, never Catmull-Rom. */
```

**Real observations.** The account-value chart is built from actual
readings written every 10 minutes, and a failed read leaves a gap rather
than a made-up point. Early history is sparse (the recorder is newer than the
club), so gaps between real observations are bridged by estimates shaped by
the club's own performance index and pinned to real recordings at both ends,
and the payload flags every estimated point as an estimate, so the UI can
render it differently. An estimate never overrides a recording, and as real
sessions accumulate the estimates retire day by day. The 1D
range was removed outright rather than faked: the dense recording was younger
than a day, and a
1D chart with two real points is a lie with an x-axis.

**Say what the number is.** When a stock page shows the previous close's
range because today hasn't traded yet, the label reads "Last session," not
"Day range." Small thing; it's the difference between a chart you can trust
and one you have to second-guess.

## The demo is the real codebase

A private club still needs something public for people who will never be
invited. The standard move is a marketing page with screenshots. Instead the
demo at demo.voteconcordia.com runs the production application against a
deterministic simulated brokerage, in an isolated instance with its own
database and keys, so anyone can sign up and use the actual product. The one
thing the demo can't simulate is a track record, so it shows the real club's,
mirrored one-way through the file boundary described in
[ARCHITECTURE.md](ARCHITECTURE.md) — which also covers why both fake fixtures
and a direct read-only connection to the production database were ruled out.

## The onboarding wizard

Joining a real-money club is consequential, so onboarding is a wizard with
explicit stages — the club's rules, a suitability check, the stake commitment,
then the brokerage OAuth connect — rather than a form with defaults. The stake
step carries the most weight: a member commits a specific slice of their
account and the system promises that only that slice ever trades. Rebalance
budgets are computed from the committed stake, not from available buying
power, so the club can never quietly grow into money a member didn't put on
the table. Participation is one switch, pausable any time, and account
deletion is self-serve.

## One build, three hosts

The member site, the admin console, and the demo are one Next.js build; the
hostname picks the shell. This keeps the admin console from drifting behind
the member app (they can't; they ship together) and means the demo is
provably running the same frontend the club uses. Since browser storage is
per-origin, each host's sessions are isolated by the platform itself: an admin
token can't leak into the member site, and a demo login can't collide with a
real one. The admin host swaps the entire shell for a role-gated console
before anything renders, and the demo host adjusts branding so nobody
mistakes fake money for real.

## Presentation state never overwrites trading records

A recurring pattern across the codebase: anything the UI needs is computed
from trading records, never written back over them. Order rows keep their raw
review/place/error payloads forever, and cycle status, member health, and
performance are all derived at read time. The one exception: the reconcile
pass may rewrite an order's *quantity*, and only to replace an estimate with
the broker's actual fill.
