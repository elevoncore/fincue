# User Guide

*This document is written for the person using the app, not the developer
— it doubles as a first draft of the in-app help content. Screenshots
referenced below should be added to `docs/assets/` once the UI exists.*

## Getting started

1. **Create your account** and set a strong password. On first login,
   you'll be asked to set up **biometric lock** (Face ID, Touch ID, or
   Windows Hello) — this adds a quick check whenever you open the app,
   even if your phone or laptop is already unlocked.
2. **Add your first account.** This can be a bank account, a cash wallet,
   or a credit card — you're just telling Fincue what you want to
   track, not linking your real bank (that's an optional, later step).
3. **Log a transaction.** Tap "Add," enter an amount, pick a category, and
   save — or use the camera icon to scan a receipt instead (see below).

## Adding transactions

- **Manual entry** works offline — if you don't have signal, your entry is
  saved on your device and syncs automatically once you're back online.
  You'll see a small "pending" indicator until it syncs.
- **Scan a receipt:** tap the camera icon, photograph the receipt flat and
  well-lit, and Fincue will read the merchant, date, and total for
  you. **Always double-check the pre-filled fields before saving** — OCR
  (the technology that reads the receipt) isn't perfect, especially on
  crumpled or faded receipts.
- **Split a receipt:** if you bought groceries and a household item on one
  receipt, tap "Split" and divide the total across categories.
- **Foreign currency:** if a transaction is in a currency different from
  the account's, enter the amount in the original currency —
  Fincue converts it using the day's exchange rate automatically.
  You can also type in your own rate if you already know what you were
  charged.
- **Tags:** add free-text tags (like `#business-trip` or `#gift`) to any
  transaction for your own filtering later — these are separate from
  categories and are entirely up to you.

## Budgeting with envelopes

Fincue uses **zero-based budgeting**: every dollar you earn gets
assigned a job (an "envelope") at the start of the period. If you don't
spend everything in an envelope, you can choose to let the leftover amount
**roll over** into next month, or reset to zero.

Your **Safe-to-Spend** number on the home screen already accounts for
upcoming bills you know about — it's meant to answer "can I afford this
right now?" more honestly than your raw account balance can.

## Getting help from the assistant

Tap the assistant icon and ask a plain-language question, like:

> "How much did I spend on dining out last weekend?"
> "Am I on track for my Bali trip goal?"

The assistant currently understands questions about your spending,
budgets, and goal progress — if it doesn't understand a question, it will
say so rather than guessing.

## Goals and debt payoff

- **Short-term goals** (like a vacation) show progress as a simple bar;
  **retirement projections** show a long-range estimate that accounts for
  compound growth and inflation — treat this as an illustrative estimate,
  not financial advice.
- **Round-up savings:** turn this on to automatically sweep the "spare
  change" from each purchase (e.g., a $4.60 coffee rounds up $0.40) into a
  goal of your choice.
- **Debt payoff simulator:** if you have more than one debt, compare
  **Avalanche** (pay off the highest-interest debt first — saves the most
  money) against **Snowball** (pay off the smallest balance first — often
  easier to stay motivated with) side by side before choosing.
- If a month is tight, Fincue may suggest **pausing** a long-term
  goal's contribution temporarily rather than missing a bill — you always
  choose whether to accept the suggestion.

## Your Financial Health Score

A single number from 0–100 summarizing your habits — tap it to see exactly
which components (savings rate, budget adherence, and others — see
`docs/gamification-strategy.md`) make it up, rather than treating it as an
opaque score you can't act on.

## Streaks and badges

Your **spending streak** counts consecutive days without discretionary
(non-essential) spending. Budget bars shift from green to yellow to red as
you approach a limit, so you can see it coming rather than being surprised
by a red screen after the fact. Badges unlock for milestones like hitting
a savings goal or paying off a debt.

## Reports

- The **Sankey diagram** shows your money flowing from income into each
  spending category — a quick visual gut-check on where things are
  actually going.
- **Tax export** lets you download transactions tagged as tax-deductible
  as a CSV for your accountant.
- **Financial Wrapped** gives you a shareable monthly/yearly summary of
  your habits, in the spirit of a music-streaming year-in-review.

## Your data

You can **export your entire financial history** at any time from
Settings → Export Data. This requires re-authenticating (a security
measure, since it's your complete financial history) and gives you a
downloadable copy in a standard format.

## Privacy note

Your questions to the assistant may be processed by a third-party AI
service to understand what you're asking — but your actual financial
figures (balances, transaction amounts, account details) never leave
Fincue's own database to do so. See `docs/security.md` if you want
the technical detail.
