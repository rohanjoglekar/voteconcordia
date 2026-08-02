# Design decisions

Product and UI calls, with the reasoning. Some of these cost me features I
wanted; I think each trade was right.

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
most consumer finance UIs. Then I measured what the curve actually drew:
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

**Recorded, not interpolated.** The account-value chart is built from actual
observations written every 10 minutes, and a failed read leaves a gap rather
than a made-up point. Early history is sparse (the recorder is newer than the
club), so gaps between real observations are bridged by estimates shaped by
the club's own performance index and pinned to real recordings at both ends —
and the payload flags every estimated point as an estimate, so the UI can
render it differently. An estimate never overrides a recording, and as real
sessions accumulate the estimates retire day by day. I even removed the 1D
range rather than fake it: the dense recording was younger than a day, and a
1D chart with two real points is a lie with an x-axis.

**Say what the number is.** When a stock page shows the previous close's
range because today hasn't traded yet, the label reads "Last session," not
"Day range." Small thing; it's the difference between a chart you can trust
and one you have to second-guess.

## The demo is the real codebase

I needed something public for people who will never be invited to a private
club. The standard move is a marketing page with screenshots. Instead the
demo at demo.voteconcordia.com runs the production application — same
executor, same voting engine, same onboarding — against a deterministic
simulated brokerage, in an isolated instance with its own database and keys.
Anyone can sign up and use the actual product. The one thing the demo can't
simulate is a track record, so it doesn't try: the club history and
performance shown there are the real club's, mirrored one-way through the
file boundary described in [ARCHITECTURE.md](ARCHITECTURE.md). Fake fixtures
would have invented a record; an empty history would have hidden the product.
Mirroring the real one was the option that neither lied nor hid.

## Guided onboarding, not a settings page

Joining a real-money club is consequential, so onboarding is a wizard with
explicit stages — the club's rules, a suitability check, the stake commitment,
then the brokerage OAuth connect — rather than a form with defaults. The stake
step is the one I care most about: a member commits a specific slice of their
account and the system promises that only that slice ever trades. Rebalance
budgets are computed from the committed stake, not from available buying
power, so the club can never quietly grow into money a member didn't put on
the table. Participation is one switch, pausable any time, and account
deletion is self-serve.

## One build, three hosts

The member site, the admin console, and the demo are one Next.js build; the
hostname picks the shell. This keeps the admin console from drifting behind
the member app (they can't — they ship together) and means the demo is
provably running the same frontend the club uses. Since browser storage is
per-origin, each host's sessions are isolated by the platform itself: an admin
token can't leak into the member site, and a demo login can't collide with a
real one. The admin host swaps the entire shell for a role-gated console
before anything renders, and the demo host adjusts branding so nobody
mistakes fake money for real.

## Derived, not duplicated

A recurring pattern across the codebase: presentation state is computed from
trading records, never written back over them. Order rows keep their raw
review/place/error payloads forever, and anything the UI needs — cycle
status, member health, performance — is derived at read time. The one
exception proves the rule: the reconcile pass may rewrite an order's
*quantity*, but only to replace an estimate with the broker's actual fill,
moving the record closer to truth, not further from it.
