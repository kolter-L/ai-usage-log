# DEMO.md

Demo script — NEWITY lender comparison tool. ~5 minutes. Requirement numbers are `SPEC.md`.

## 1. What the tool does

- A rep is on a live call. Four borrower answers in, they need lender names, rates and turnaround times *now* — not after a callback.
- In: loan amount, business type, years in business, credit tier. Out: three ranked sections — **Likely fits**, **Near misses**, **Closest available** — and a copyable summary for the CRM.
- Every row is in exactly one section (F7), so nothing is double-counted, and Closest available is always populated (F11): the tool never answers a live call with a blank screen.
- Compliance shape is deliberate: never "qualified / approved / eligible" (F20), rates always a **range** (F17), the non-binding disclaimer on screen *and* inside every copy (F19).
- Static React app, no backend. Rates are one ops-owned CSV fetched at runtime (F25) — ops overwrites the file, the rep reloads, no developer in the loop.

## 2. Live walkthrough

1. Load → header reads "Rates as of 2/8/2026" (F23), taken from the newest `last_updated` in the sheet. Loading and error states are explicit (F26, F27).
2. Enter **$250,000 / Other / 3 yrs / Excellent**. "Other" matches only programs open to `All` business types (F4) → 19 likely fits, top 5 shown, "Show all 19" beneath.
3. Default sort is lowest starting rate: five 504 loans, 5.5%–7.8% — and 48–82 day turnarounds. The right answer, and the wrong answer for a borrower who needs money this month.
4. Toggle **Fastest turnaround** (F18): BlueRidge Express goes from off-screen to first at 5 days. That is the pitch — one click reframes the whole list mid-sentence.
5. **Copy top 3** (F21), paste straight into the CRM note:

```
Likely fits — rates as of 2/8/2026

1. BlueRidge Lending Partners — SBA Express
   Rate: 11.7% – 13.2%
   Loan range: $25,000 – $500,000
   Max term: 120 mo
   SBA guarantee: 50%
   Turnaround: 5 days
   Collateral: No
   Notes: Owner must have 20%+ equity stake
   Updated: 1/29/2026

2. Heritage Credit Union — SBA Express
   Rate: 10.3% – 11.4%
   Loan range: $10,000 – $500,000
   Max term: 84 mo
   SBA guarantee: 50%
   Turnaround: 10 days
   Collateral: No
   Updated: 1/23/2026

3. Liberty SBA Lending — SBA Express
   Rate: 11.5% – 12.9%
   Loan range: $25,000 – $500,000
   Max term: 120 mo
   SBA guarantee: 50%
   Turnaround: 10 days
   Collateral: No
   Notes: Requires 10% owner injection
   Updated: 1/20/2026

Likely fit — subject to full eligibility review. Not a pre-approval.
```

6. Read that block out: it is in **turnaround order**, because the copy inherits the view — what the rep pasted is what they just talked through. Ranges only, never a figure. Two kinds of date: the sheet's (F23) and each lender's own (F22). `Notes:` appears only where the program has requirements. First and last lines bracket it so that, stripped of the app, it still cannot read as a pre-approval.
7. Change years to **1** → programs needing 2–3 years drop into Near miss with red inline chips, no hover: `⚠ Needs 2 yrs in business — borrower has 1` (F13). The rep can now say *what* is missing instead of "no".
8. Enter **$6,000,000** → no matches, and a blocker headline rather than an empty area (F14, F15): "the highest in our network is $5,500,000". Closest available still shows the nearest five.
- Demo note: staleness badges (F22) are relative to the system clock, so on a machine set later than March 2026 every row badges stale.

## 3. Scope decisions

- **All of F1–F28 is built.** The scoping work was in resolving what the requirements left open, and in refusing what they did not ask for.
- **Two ambiguities I closed in the spec, before writing code.** (a) Section membership is *exclusive* — Closest available is everything not already placed, or programs appear twice. (b) Near-miss rows all have zero hard misses, so they sort by miss count ascending, tie-broken on `interest_rate_min` — explainable to a rep, and no weight tuning.
- **Two hard criteria, three soft, four display-only** (F4–F6). `min_credit_score` is the tempting one: it is in the data and it is *not* a filter — credit tier is the filter, and a score cutoff would silently delete programs a rep could still place. Tier is a floor, not an equality (F5).
- **Three deviations from the spec text, all deliberate.** Six business types, not seven — the CSV holds six concrete types plus `All`. No upper bound on the loan amount field — capping it would hide F15's blocker headline, which is the most useful answer to an oversized ask. CSV lives at `public/lender_programs.csv` and is gitignored: ops data, not source.
- **Bounded by design.** Match shows 5 with "Show all N"; Near miss and Closest available cap at 5 with no expander (F9, F10). A rep on a call does not scroll 49 rows.
- **N9 flipped from the original NFRs**: CSV bundled at build → fetched at runtime. That single change is what makes a rate update an ops task instead of a deploy.
- **Cut deliberately**: no separate compare flow (F16 makes the aligned table the comparison), no persistence or URL state, no auth, no router, no component library, no state library, no runtime validation library, no cross-browser or accessibility audit beyond labelled controls.
- **One honest gap in the cut**: `min_credit_score` and `max_existing_debt_ratio` are parsed and typed but rendered nowhere. F17's column list is the authority and seven columns is what stays scannable — but F6 calls them display-only, so arguably they belong in the expanded row.

## 4. How I'd test it

- **Two layers, no middle.** 79 Vitest unit tests over the pure layers (`domain/`, `data/`, `services/`), and 17 Playwright scenarios in `TEST_PLAN.md` that drive a real browser end to end. No jsdom component layer — it answers a third question neither of those asks.
- **The unit layer exists because `domain/` is pure.** `matchPrograms(criteria, programs, sort)` takes `today` as a parameter, so the 30-day staleness boundary and F12's weights are tested without mocking a clock or a DOM.
- **E2E runs against the production build** — `playwright.config.ts` builds and serves with `vite preview` on localhost, which is the same static file serving ops deploys, and satisfies N4 so the clipboard under test is the real one.
- **The interesting cases are failure and data, not the happy path**: missing CSV, header short a column, every row rejected, and 49 good rows + 3 bad (which must load anyway, warn in the console, and match over the survivors). Fixture swap on disk, exactly what ops does.
- **Scale is measured, not asserted.** 1000 rows: load 69–83 ms, expand 124–133 ms, re-sort 131–140 ms. Recorded and printed; no wall-clock threshold to fail flakily on a shared runner.
- **F20 needed a judgement call in the tests.** Four real rows read "Franchise must be SBA-approved", so a naive banned-word scan fails on the data. The scan excludes lender requirement text and the mandated disclaimer, and a companion test asserts every on-screen "approved" is inside those exclusions — so app copy can never quietly widen it.
- **Kept manual, honestly**: that the chip reads red while the card stays neutral, that three programs are genuinely scannable side by side, and tone beyond a three-word list — the actual compliance risk is a phrasing that implies approval without using the words.
- **Known test debt**: the exact 30-vs-31-day staleness unit test is still unwritten, and the e2e suite is not wired into CI.

## 5. What I'd build next

- **The backend, first.** `loadPrograms.ts` is the only CSV-aware file and V2 is a body swap to `fetch('/api/programs')`; `domain/` lifts to the server verbatim, because it imports no React, no fetch and no clock. That boundary was the point of the layering.
- **What the server buys**, in order: validation at ingest, so a bad row is rejected when ops uploads it rather than dropped silently into a rep's console; versioned rate sheets, so "what did we quote on the 8th" is answerable; and a real `logRecommendation` endpoint (F24 is a `console.debug` stub today) so F12's closeness weights can be tuned against what reps actually send instead of my guesses.
- **Then the two product misses**: results that run live as the rep types instead of behind a submit button, and a bad-row banner so a shorter list is never silent.
- **Then**: shareable call state in the URL, staleness enforced server-side against a per-lender freshness SLA rather than the browser clock, and a responsive/print pass.

## 6. Self-assessment

- **I over-invested in planning.** Spec and architecture took a disproportionate slice of a ~3 hour budget. It bought real rework savings — the layering never moved once — but 30 minutes would have bought most of that, and I would have hit the real CSV sooner.
- **Right stack, thin familiarity.** TS/React/Vite is the correct pick here and I argued it on merits, but I do not know it deeply. I substituted the type system and unit tests for experience I do not have. That held up for correctness; it shows in the pace and in the CSS.
- **The submit button is the clearest product miss** — a call-speed tool with a gate in front of results. Cheap to fix, and I should have caught it by picturing the call rather than the form.
- **I trusted a verification command that verified nothing**: the root `tsconfig.json` is a solution file, so `tsc --noEmit` exits 0 no matter what is broken. The real check is `npm run typecheck` (`tsc -b`). Caught late.
- **Written, not operationalized**: the e2e suite depends on hand-swapping a gitignored fixture and runs nowhere but my laptop.
