# Gamification & Behavioral Finance Strategy

The brief asks for gamification "using psychology" — this document makes
that concrete and specific enough to implement and defend in a viva/demo,
rather than leaving it as a vague aspiration.

## 1. Financial Health Score (0–100)

A single, transparent number — transparent meaning the user can always see
*why* it is what it is, via its component breakdown, rather than treating
it as a black box.

**Suggested component weighting (tune during testing, document any
changes in `decisions.md`):**

| Component | Weight | What it measures |
|---|---|---|
| Savings rate | 30% | (Income − Expenses) / Income over the period, scaled to 0–100 (e.g., a 20% savings rate → full marks; scale down for lower, cap for very high). |
| Budget adherence | 25% | % of envelopes not overspent this period. |
| Emergency fund coverage | 20% | Liquid savings ÷ average monthly essential expenses, scaled (e.g., 3+ months → full marks). |
| On-time bill / subscription payments | 15% | % of tracked recurring bills paid without a late flag. |
| Debt trend | 10% | Whether total debt balance is decreasing period-over-period. |

Store the score **and** its component breakdown in
`financial_health_scores.components` (see `docs/database.md`) so the UI
can render "Your score is 72. Your savings rate is strong (28/30), but
your budget adherence dropped this month (12/25) — 2 envelopes went over."
A score with no visible reasoning teaches the user nothing and invites
distrust; a score with visible components turns it into an actual coaching
tool, which is the entire point of a "proactive advisor" rather than a
passive tracker.

**Compute this on a schedule** (e.g., nightly, via the scheduled-jobs
mechanism in `docs/deployment.md`), not on every page load — it's a
derived, slowly-changing value, not something that needs real-time
recomputation.

## 2. Spending streaks

- **Streak type:** "days with zero discretionary spending," where
  "discretionary" is whatever categories the user has *not* marked as
  essential (a sensible default essential set — rent, utilities,
  groceries, transport to work — with the rest discretionary by default,
  user-adjustable).
- **Reset behavior:** a single discretionary transaction resets the
  current streak to zero but preserves `longest_count` — losing today's
  streak shouldn't erase the record of your best one, which is
  demotivating for no behavioral benefit.
- **Framing matters:** show the streak as an encouraging number ("14-day
  streak!") rather than a shaming one when it resets — avoid guilt-based
  copy like "you broke your streak"; frame it as "new streak started
  today" instead. This is a deliberate choice, not just tone-policing: the
  goal is sustained engagement, and shame-based framing reliably backfires
  on habit-formation apps.

## 3. Color-coded budget nudges

A three-band system per envelope, computed from `spent / allocated`:

| Band | Threshold | Color | UI treatment |
|---|---|---|---|
| Healthy | < 70% spent | Green | Neutral, unobtrusive. |
| Caution | 70–100% spent | Yellow/amber | A gentle inline nudge, not a popup/interrupt. |
| Over | > 100% spent | Red | Visible on the envelope and rolled up into the Safe-to-Spend calculation — but still not a blocking interrupt; the user retains control over their own money. |

Keep thresholds configurable (a constant in one place, not scattered magic
numbers) — different users may reasonably want tighter or looser warning
bands, and Phase 2 could expose this as a setting.

## 4. Badges

A **starter set** (MVP, per `docs/PRD.md`) is enough to demonstrate the
mechanic; the full catalog is Phase 2. Suggested starter badges:

| Badge | Criteria |
|---|---|
| "First Envelope" | Created your first budget envelope. |
| "Zero-Based Rookie" | Fully allocated a month's income across envelopes. |
| "7-Day Streak" | Reached a 7-day discretionary-spending-free streak. |
| "Debt Free" | A tracked debt's balance reaches zero. |
| "Goal Getter" | A short-term goal reaches 100% of its target. |
| "Receipt Wrangler" | Scanned 10 receipts via OCR. |

Store criteria as structured JSON (`badges.criteria`, see
`docs/database.md`) so new badges can be added by data, not by shipping
new code for each one — e.g. `{"type": "streak_reaches", "value": 7}`.

## 5. "Financial Wrapped" (Phase 2)

A shareable monthly/yearly summary, in the spirit of a music-streaming
year-in-review. Suggested slide content: total spent vs. saved, top 3
categories, biggest single splurge, longest streak achieved, a "you're in
the top X% of savers this month" comparison **only if** you have enough
opted-in users to make that statistically meaningful and privacy-safe —
otherwise, compare the user only against their own history ("you spent 12%
less than last month"), which avoids both a cold-start problem and any
cross-user data exposure.

## 6. Why this counts as "using psychology," specifically

Worth stating plainly for the FYP report rather than leaving "gamification"
as an unexamined buzzword — the mechanisms above map to well-established
behavioral-economics and habit-formation concepts:

- **Loss aversion / framing effects:** the color-band system and
  streak-reset copy are designed around how loss-framed vs. gain-framed
  messaging affects behavior differently.
- **Immediate feedback loops:** badges and streaks provide short-cycle
  reinforcement for behavior (saving, budgeting) whose real-world payoff
  is otherwise months or years away — bridging that gap is a large part of
  why gamification is used in personal finance apps at all.
- **Transparency/self-efficacy:** the Financial Health Score's visible
  component breakdown is specifically designed to support a sense of
  control and actionability, rather than presenting an opaque judgment.

Cite general behavioral-economics/gamification literature (e.g., work on
loss aversion, self-determination theory, and gamified habit formation) in
the FYP report's literature review rather than in this file — this
document is the implementation spec, not the academic citation itself.
