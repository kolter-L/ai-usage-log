# REQUIREMENTS.md

NEWITY Product Engineer Assessment — B: Lender Comparison Tool

## 1. The problem in plain English

**Domain crash course (no lending background needed):**

| Term | What it means |
|---|---|
| SBA | U.S. Small Business Administration. It doesn't lend money — it *guarantees* part of a bank's loan, so the bank risks less and lends to riskier businesses. |
| Program type | A flavor of SBA loan. `7(a) Standard` = big, general purpose. `7(a) Small Loan` = smaller version. `SBA Express` = fast, less paperwork, smaller guarantee. `504 Loan` = real estate/equipment, long terms, cheapest rates. `Community Advantage` = for weaker-credit/underserved borrowers, highest rates. |
| Lender | The actual bank/credit union. Each lender offers several programs, with their own rates and rules. |
| Credit tier | Rough bucket of borrower creditworthiness: excellent / good / fair. |
| Owner injection | Down payment from the business owner. |
| Collateral | Assets pledged to secure the loan. |
| Turnaround days | How long from application to funding. |

**The actual pain:** a sales rep is on the phone with a small business owner *right now*. To answer "what do I qualify for?" they alt-tab between PDFs, a wiki, and a rate spreadsheet. It takes minutes, it's error-prone, and new reps take weeks to get fluent.

**What you're building:** a browser tool where a rep types 4 facts (loan amount, business type, years in business, credit tier) and instantly sees the matching lender programs, ranked, comparable side by side.

Think of it as **a filter + compare view over one CSV**. That's the whole core. Everything else is optional.

---

## 2. Users

| User | Count | Need |
|---|---|---|
| Sales rep | 8 | Answer in seconds, mid-call, zero training |
| Sales manager | 2 | See which programs reps recommend most |

Non-technical. "Fast and obvious. No training manual."

---

## 3. Functional requirements

These are split by what the brief *states* vs. what it merely *wishes for*. The assessment explicitly says you can't build it all — that's the test.

**Core (the tool is pointless without these)**

- Input: loan amount, business type, years in business, credit tier
- Match those inputs against the 49 lender-program rows
- Show matching programs with rate range, term, guarantee %, turnaround
- Show *why* something matched or didn't (rep has to explain it out loud to the borrower)

**Requested by the team (pick a subset)**

| Feature | Effort | Value to a rep on a call |
|---|---|---|
| Filter by borrower criteria | Low | Highest — this is the product |
| Rate & term details | Low | High |
| Eligibility requirements at a glance | Low | High — reps must state conditions |
| Side-by-side comparison of 2–3 programs | Medium | High |
| Save/share a comparison with the borrower | High (needs persistence/export) | Medium — happens after the call |
| Flag when lender rates are updated | Medium (needs "last seen" state) | Low during a call |
| Manager view: most-recommended programs | High (needs logging + storage) | Low for V1, only 2 users |

**A strong V1 = the core + comparison + visible eligibility notes.** Cut sharing, manager analytics, and staleness flagging, and explain why in the demo.

**Near-miss handling is the sleeper feature.** Real calls aren't clean yes/no. "You're at 18 months; at 24 months these three open up" is the most useful thing the tool can say. Cheap to build, big product-thinking signal.

---

## 4. Non-functional requirements

| Requirement | Target | Why |
|---|---|---|
| Latency | Sub-second, feels instant | Borrower is on the phone. 49 rows = filter in memory, no server round-trip needed |
| Runs in a browser | Local is fine, no deploy required | Stated in brief |
| Learnability | Usable with zero training | Sales team, not engineers |
| Scale | 10 users, 49 rows | Effectively zero — don't over-engineer a backend |
| Setup | Reviewer runs it in <5 min from README | Grading criterion |
| Data refresh | Rates change monthly; terms quarterly | Needs *some* story for swapping the CSV |
| Tech stack | Free choice | No constraint |
| Integrations | None | Standalone |

**Notable:** no auth requirement, no uptime requirement, no compliance/PII requirement stated. Borrower details are typed in but nothing says they're stored — keeping them in memory only is a defensible, easy call.

---

## 5. The data

49 rows × 17 columns. 15 lenders × 2–4 programs each. 5 program types. No duplicate lender+program pairs. Dates span Jan 1 – Feb 8, 2026.

| Column | Type | Notes |
|---|---|---|
| `lender_name` | text | 15 distinct |
| `program_type` | text | 5: SBA Express (11), 7(a) Small Loan (11), 7(a) Standard (11), 504 Loan (9), Community Advantage (7) |
| `min_loan_amount` / `max_loan_amount` | int | $10k–$250k min; $250k–$5.5M max |
| `min_credit_score` | int | 600–700 |
| `credit_tier_required` | text | **Only "Good" (42) and "Fair" (7). No "Excellent."** |
| `min_years_in_business` | int | 1–3 |
| `interest_rate_min` / `_max` | float | 5.5%–14.5%. 504 cheapest, Community Advantage priciest |
| `max_term_months` | int | 84–300 |
| `sba_guarantee_pct` | int | 40–85 |
| `eligible_business_types` | text | Single value per row: All (19), Restaurant/Food Service (8), Construction (8), Manufacturing (5), Healthcare (4), Professional Services (3), Retail (2) |
| `requires_collateral` | text | Yes (15) / No (16) / **Varies (18)** |
| `max_existing_debt_ratio` | float | 0.40–0.55, **9 nulls** |
| `turnaround_days` | int | 5–82 |
| `special_requirements` | text | 10 distinct phrases, **15 nulls** |
| `last_updated` | text | `M/D/YYYY` strings, not ISO |

**Data quality issues you will hit:**

1. **`credit_tier_required` has no "Excellent."** Reps enter excellent/good/fair. Must be treated as a **minimum bar** (Excellent ≥ Good ≥ Fair), not an equality match. Equality matching would return zero rows for your best borrowers.
2. **Boundary collision at 640.** Fair tops out at 640; Good starts at 640. Pick `>=` or `>` deliberately and document it.
3. **Nulls are ambiguous.** Is a null `max_existing_debt_ratio` "no limit" or "unknown"? Is null `special_requirements` "none"? Both are almost certainly "none/no limit," but say so out loud.
4. **No revenue column.** The team said reps "sometimes" have revenue — you can't filter on it. Don't collect it.
5. **No debt-ratio input from reps.** The team listed 4 inputs; debt ratio isn't one. So `max_existing_debt_ratio` is display-only unless you add a 5th optional field.
6. **`requires_collateral: "Varies"`** isn't filterable — it's a display string.
7. **`eligible_business_types` is single-valued today** but the name is plural. Real data will likely be comma-separated. Parse defensively.
8. **`last_updated` is a US-format string.** Parse explicitly; don't rely on `new Date()` inference.
9. **"25–30 programs" vs 49 rows.** The team counts by *lender-program pair* loosely; the CSV has 49. Word your UI carefully — "program" is ambiguous between `program_type` (5) and a lender's offering (49).

---

## 6. Gaps — decisions you have to make

The brief says explicitly: *"make smart calls with incomplete information and explain your reasoning."* These are the calls.

| # | Open question | Options | Lean |
|---|---|---|---|
| 1 | Hard filter or ranked results? | Exclude non-matches / show all with score / show matches + near-misses | Matches first, near-misses in a separate collapsed section with the reason |
| 2 | How to rank matches? | Lowest rate / fastest turnaround / highest loan ceiling | Rate ascending, with a sort toggle for turnaround |
| 3 | Credit tier vs. credit score input? | Tier dropdown / exact score / both | Tier dropdown (reps have "rough tier"), map tier→score range internally |
| 4 | What does "Excellent" match? | Everything / nothing | Everything. Treat tier as a floor |
| 5 | Business type not in the list? | "Other" option / free text / omit | "Other" → matches only rows with `All` |
| 6 | What does "rates updated" mean? | Changed since last view / older than 30 days / changed vs. previous CSV | Staleness badge if `last_updated` > 30 days before today. Cheapest defensible version |
| 7 | "Today" for staleness — real clock or data-relative? | System date / max date in CSV | System date, but note the data is a Feb 2026 snapshot so everything may show stale |
| 8 | Share a comparison — how? | Print/PDF / copy-to-clipboard summary / shareable URL | Cut it, or do copy-to-clipboard text if time allows. A URL needs hosting |
| 9 | Manager analytics — storage? | localStorage / backend / cut | Cut for V1. 2 users, needs a real backend to be meaningful |
| 10 | CSV loading | Bundled at build / fetched at runtime / user uploads | Fetched at runtime so ops can swap the file without a rebuild |
| 11 | Is borrower data stored? | Yes / in-memory only | In-memory only. Sidesteps every privacy question |
| 12 | Desktop-only or responsive? | — | Desktop-first; reps are at a desk on a headset |

---

## 7. What you're actually graded on

Easy to miss: **the app is one of three deliverables**, and arguably not the most heavily weighted one.

| # | Deliverable | Notes |
|---|---|---|
| 1 | Working code + README | Runnable locally in under 5 min |
| 2 | Demo video, 5–10 min | Aimed at *sales reps*, not engineers. Must cover: what it does, scope decisions, QA plan, honest self-assessment, what's next |
| 3 | AI usage log | Called "the most important deliverable." Must span understanding → scoping → building → testing → docs |

Stated evaluation criteria: AI as a workflow, scope control, full-lifecycle thinking, product thinking, engineering judgment, communication. **Nothing about visual polish or stack choice.**

Budget: 3–4 hours, hard cap at 4. Roughly 2.5–3h build, 20 min demo, 30 min AI log. Overbuilding is explicitly penalized.
