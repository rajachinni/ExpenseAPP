# Expense Tracker — Requirements

**Version:** 0.3 (draft) · **Date:** 2026-11-10
**Source:** derived from the current system in [ARCHITECTURE.md](current-ARCHITECTURE.md) (iOS Shortcut → Google Form → Google Sheet + Apps Script + formulas), plus India findings in [RESEARCH.md §6](RESEARCH.md).
**Changes in 0.3:** owner decisions recorded (§0); priorities adjusted to match.
**Changes in 0.2:** email-alert capture channel, TRAI sender filtering, spam rejection, cross-channel dedupe, UPI Lite handling, template-drift alerts, DPDP-driven data ownership.
**Scope:** approach-neutral. These hold whichever direction is chosen: Sheet + Apps Script, local web app, or native iOS app.

| Priority | Meaning |
|---|---|
| **MUST** | Needed for v1 / parity with today |
| **SHOULD** | Strongly wanted, can follow shortly after v1 |
| **COULD** | Nice to have |

`NEW` = not in today's system; added for multi-user use or to fix a known problem.

---

## 0. Decisions (2026-11-10)

| # | Question | Decision | Effect |
|---|---|---|---|
| Q1 | Friends' platforms | **All iPhone** | Shortcuts SMS capture works for everyone; Android out of scope. |
| Q2 | $99/yr Apple developer account | **No — keep it free** | Rules out a native iOS app. |
| Q3 | How data is seen and edited | **Spreadsheet + web dashboard** | Google Sheet stays the store; a dashboard on top. |
| Q4 | Income | **Auto-capture from credit SMS** | FR-P5 stays MUST; credit rules needed. |
| Q5 | Default status of new records | **Always review** | Every new transaction lands in the review queue (FR-T7 default = Review). |
| Q6 | Axis Bank SMS | **Automatic, with a skip / Ignore button** | Capture filter covers Axis too; FR-R2 needs one-tap Ignore. |
| Q8/Q9 | Email alerts | **No — SMS only** | FR-C11, FR-U7 moved out of v1. |
| Q10 | UPI Lite | **Not used** | FR-C13 out of v1. |
| — | Migration | **Fresh start** from a chosen date; current sheet kept untouched as an archive | FR-X3 import downgraded to COULD. |
| — | Device | **Phone and desktop equally** | Dashboard must be responsive. |
| Q7 | History | **One sheet per year** (as today) | Reports cover one year; a yearly rollover carries settings forward (FR-X6). |

**Resulting direction:** Option A — each user owns a Google Sheet + Apps Script web app (capture endpoint + responsive dashboard), with code shared from one library. No Google Form, no formulas, no server. Still open: Q11 (sender IDs).

---

## 1. Goals

| ID | Goal |
|---|---|
| G1 | Capture bank transaction SMS automatically with no manual typing. |
| G2 | Turn raw SMS into clean records: date, merchant, amount, category. |
| G3 | Show where money goes: by category, month, year, and against budgets. |
| G4 | Few moving parts; no paid database or server; near-zero running cost. |
| G5 | `NEW` Friends can run their own independent copy with their own data. |
| G6 | `NEW` Adding a bank, card, category or report never breaks existing data. |

## 2. Users

| ID | User | Notes |
|---|---|---|
| U1 | Owner (you) | Daily user, defines rules, reviews transactions. |
| U2 | `NEW` Friend | Same capabilities as the owner, on their own data only. Non-technical; setup must not require editing code or formulas. |

---

## 3. Functional requirements

### 3.1 Capture

| ID | Priority | Requirement |
|---|---|---|
| FR-C1 | MUST | Automatically capture incoming SMS matching a configured filter (today: message contains "ICICI Bank"), without user confirmation. |
| FR-C2 | MUST | Allow manual submission of a message (today: Shortcut run by hand for Axis Bank SMS; falls back to "Ask for Text"). |
| FR-C3 | MUST | Raw message text is the only required input (today: one field, `entry.<MESSAGE_FIELD_ID> = Message`). |
| FR-C4 | MUST | Record a received timestamp for every capture. |
| FR-C5 | MUST | Keep a capture log — pending before send, success after (today: two Apple Notes). Failures must be visible. |
| FR-C6 | SHOULD | `NEW` Retry failed sends automatically, or queue them on the device until they succeed. |
| FR-C7 | MUST | `NEW` Ignore duplicates: the same transaction creates one record even if it arrives twice or through several channels (SMS, email). Match on UPI / reference number when present, otherwise on amount + account + time window, plus a fingerprint of the text. |
| FR-C8 | MUST | `NEW` Capture filters configurable per user (which banks / sender keywords trigger capture). v1 covers ICICI **and Axis** automatically (decision Q6). |
| FR-C9 | MUST | `NEW` Only the owner of a dataset can write to it; any capture endpoint requires a per-user secret. |
| FR-C10 | SHOULD | Quick manual entry of cash / non-SMS expenses (today: unused tab "spends added manually"). Raised from COULD because UPI Lite and cash spends produce no per-payment SMS. |
| FR-C11 | OUT (v1) | `NEW` **Email-alert channel:** read bank transaction alert emails from the user's own mailbox (e.g. Gmail) on a schedule and feed them into the same parsing pipeline. Processed emails are marked (e.g. a label) so they are never read twice. Works without an iPhone, so it covers Android users. |
| FR-C12 | SHOULD | `NEW` Record the **sender ID** with each message, and filter by the TRAI category suffix (`-T` transactional, `-S` service, `-P` promotional, `-G` government). Promotional (`-P`) messages are dropped before parsing. Which suffix banks actually use must be confirmed from real messages. |
| FR-C13 | OUT (v1) | `NEW` **UPI Lite:** recognise wallet top-up and consolidated UPI Lite debit messages. Record top-ups as a transfer to a "UPI Lite wallet", not as an expense, and prompt for manual entry of the individual small spends (or accept the consolidated total). |

### 3.2 Parsing

| ID | Priority | Requirement |
|---|---|---|
| FR-P1 | MUST | Extract amount from the SMS (today: `(Rs.?\|INR)` + digits with 2 decimals). |
| FR-P2 | MUST | Extract merchant / payee using an ordered rule list. Each rule: **Name, Contains** (all tokens present), **Start text, End text**, optional **Start regex**, optional **Refine regex** (capture group 1). First match wins. |
| FR-P3 | MUST | Ship the 12 rule types in use today: ICICI CC UPI, MSI Debit, Standing Instruction, ICICI/Axis P2P, UPI Mandate, Standard ("at … on"), Simple UPI, NEFT Credit, ICICI CC SAF, ICICI auto debit, ICICI CC AMZN, Axis Bank CC. |
| FR-P4 | MUST | If no rule matches, mark the record **Manual Review Needed** instead of dropping it. |
| FR-P5 | MUST | `NEW` Detect direction: debit (outflow) vs credit (inflow). Today inflow reports are all 0. |
| FR-P6 | SHOULD | `NEW` Extract account / card suffix (e.g. XX1234) and the transaction date written in the SMS. |
| FR-P7 | MUST | `NEW` Rules are user data, not code: add, edit, reorder and test a rule against a sample SMS without touching code. |
| FR-P8 | SHOULD | `NEW` Card numbers are not hard-coded into rules; a template refers to "my ICICI credit card" and each user maps their own suffix. |
| FR-P9 | SHOULD | `NEW` Re-run parsing on selected / failed records after a rule change (today: `retryFailedRows`). |
| FR-P10 | MUST | `NEW` **Spam / non-transaction rejection** before parsing: OTPs, promotions, balance-only and "payment due" reminders are discarded or tagged, not turned into expense rows. |
| FR-P11 | SHOULD | `NEW` **Template-drift alert:** if messages from a known bank sender stop matching any rule (e.g. several unmatched in a row), notify the user that the bank has probably changed its message format. |
| FR-P12 | SHOULD | `NEW` Extract the UPI / transaction reference number when present (used for dedupe, FR-C7). |
| FR-P13 | COULD | `NEW` Rules can apply to either channel (SMS or email); each rule says which channel(s) it handles. |
| FR-P14 | COULD | `NEW` Seed default bank templates from open-source Indian parsers (e.g. PennyWise AI, kharcha-core, Cashiro), respecting their licences. |

### 3.3 Merchant mapping

| ID | Priority | Requirement |
|---|---|---|
| FR-M1 | MUST | Merchant dictionary: raw identifier → clean display name → default category (today: "Expense Mapping", ~360 entries). |
| FR-M2 | MUST | Add unseen merchants automatically, flagged as new, clean name = raw identifier. |
| FR-M3 | MUST | Fall back to "No Vendor Available" / "Category not available" when unmapped, and surface for review. |
| FR-M4 | SHOULD | `NEW` Mapping a merchant once from review also updates its past uncategorised transactions. |
| FR-M5 | COULD | `NEW` Multiple raw identifiers can be aliases of one merchant. |

### 3.4 Categories and groups

| ID | Priority | Requirement |
|---|---|---|
| FR-K1 | MUST | Two-level taxonomy, Category → Group (see table below). |
| FR-K2 | MUST | `NEW` Categories defined in **exactly one place**; every screen, rule and report reads from it. Fixes today's mismatches (`Mutual Funds`/`MutualFunds`, `Chitfund`/`Chits`, `Gold Physical`/`Gold`, typo `Entertainmet`). |
| FR-K3 | MUST | `NEW` Add, rename or retire a category without breaking records or reports. |
| FR-K4 | MUST | Optional monthly budget limit per category (today: "limit" in Statistics). |
| FR-K5 | SHOULD | Secondary reporting groups for quarterly / yearly analysis with expected % (today: Rent + Maintenance, Food Maintenance, Bike Maintenance, Cosmetics & Toiletries, Memberships, Family, Outside Food, Movies, Travel, Leisure & Hobbies, Savings). |

| Group | Categories |
|---|---|
| Earning | Salary, Dividends, TaxReturns, Others |
| Payments | Household, Groceries, Transport, PersonalCare |
| Savings | MutualFunds, Stocks, Fixed, Recurring, GoldStock, Gold, Chits |
| Expenses | Fitness, Entertainment, Gaming, Food, Travel, Family, Shopping |

### 3.5 Transaction record and manual overrides

| ID | Priority | Requirement |
|---|---|---|
| FR-T1 | MUST | Each transaction stores at least: id, received_at, source (auto/manual), raw message, parsed merchant, display name, amount, direction, category, status, split flag, my share, effective date, description. |
| FR-T2 | MUST | Manual overrides win over automated values (same as today's formulas) — see table below. |
| FR-T3 | MUST | Status values at least: **Valid, RegexIssues, Pending Split**, plus an excluded state (e.g. Ignored / Duplicate / Refund). Only the first three count in reports (same as today's MasterExpenses filter). |
| FR-T4 | MUST | Split transactions: record my share; list all Pending Split items for follow-up. |
| FR-T5 | SHOULD | One transaction can be divided across several categories (today: comma-separated categories fanned out). |
| FR-T6 | MUST | `NEW` Automated and manual fields stored separately, so an edit, sort or new column can't corrupt computed values. |
| FR-T7 | MUST | `NEW` New records default to **Review** (decision Q5). The default is a setting, so it can later change to auto-Valid when merchant and category resolve. |

| Field | Rule |
|---|---|
| Effective date | override date, else received date |
| Display name | manual name, else mapped name |
| Category | manual category, else mapped category |
| Counted amount | my share if split, else full amount |

### 3.6 Review workflow

| ID | Priority | Requirement |
|---|---|---|
| FR-R1 | MUST | Review queue for: Manual Review Needed, unmapped merchant, RegexIssues, Pending Split. |
| FR-R2 | MUST | `NEW` Fast review on phone and desktop: one tap to **confirm**, **ignore** (e.g. unwanted Axis items), change category, or mark as split. |
| FR-R3 | COULD | `NEW` Notification when new items need review. |

### 3.7 Reports and dashboards

| ID | Priority | Requirement |
|---|---|---|
| FR-D1 | MUST | Monthly totals per category for a selected year (today: Yearly Inflow & Outflow, Jan–Dec). |
| FR-D2 | MUST | Group totals per month: Inflow, Invested Savings, Payments, Expenses. |
| FR-D3 | MUST | Monthly percentages: Savings %, Payments %, Expenses %, Unused Balance % of income. |
| FR-D4 | MUST | Budget vs actual per category per month, % used, visual gauge (green < 80%, amber 80–100%, red > 100%). |
| FR-D5 | SHOULD | Quarterly and yearly stats by reporting group, current % vs expected %, yearly income / savings / payments / expenses. |
| FR-D6 | MUST | `NEW` No row limits (today ~1,000 rows). Reports cover one year per sheet (decision Q7). |
| FR-D7 | SHOULD | `NEW` Interactive: filter by month, category, merchant, account; drill from a total down to its transactions. |
| FR-D8 | MUST | `NEW` Reports regenerated from data, never hand-edited, so they can't drift. |

### 3.8 Data, export and migration

| ID | Priority | Requirement |
|---|---|---|
| FR-X1 | MUST | Storage is spreadsheet-like or file-based and free (no paid DB or server). |
| FR-X2 | MUST | Export all data to CSV / Excel at any time. |
| FR-X3 | COULD | One-time import of existing 2026 data (~1,050 transactions, ~360 merchants, 13 rules, budgets). |
| FR-X4 | SHOULD | User-controlled backup and restore (file or cloud drive). |
| FR-X5 | COULD | Import a bank statement CSV to fill gaps / reconcile. |
| FR-X6 | MUST | `NEW` **Yearly rollover:** one action creates next year's sheet with rules, merchants, categories and budgets copied, and the capture link unchanged (or updated in one step). |
| FR-X7 | MUST | `NEW` **Fresh start:** the new system starts on a chosen date; the current sheet, form, script and shortcut keep working untouched until switched off by hand. |

### 3.9 Multi-user `NEW`

| ID | Priority | Requirement |
|---|---|---|
| FR-U1 | MUST | Each user has a fully separate dataset; no one, including the owner, can read another user's transactions. |
| FR-U2 | MUST | New user set up in ≤ 10 minutes with no code or formula editing: install/copy, pick banks, set budgets, import the Shortcut. |
| FR-U3 | MUST | Ships with default categories, groups and bank rule templates that each user can customise. |
| FR-U4 | SHOULD | Fixes and improvements reach all users without redoing setup (central code, per-user data). |
| FR-U5 | COULD | Users can share rule templates for new banks. |
| FR-U6 | MUST | `NEW` Each user's data lives only in **their own** account or device (their Google Drive / Gmail, or their phone / iCloud). The project owner never hosts or can access friends' data (see NFR-6, DPDP). |
| FR-U7 | OUT (v1) | `NEW` Users without an iPhone can still capture automatically via the email-alert channel (FR-C11). |

---

## 4. Non-functional requirements

| ID | Area | Requirement |
|---|---|---|
| NFR-1 | Cost | No recurring running cost (free tiers only). A one-time / yearly distribution fee is a decision, not a default. |
| NFR-2 | Simplicity | At most 2 moving parts (capture client + one app/script). No separate DB, queue or server. |
| NFR-3 | Robustness | Adding a column, category, rule, bank or year breaks nothing. Code addresses fields by name, never column position (today: `e.values[5]`, columns 2 and 3). |
| NFR-4 | Performance | A capture is processed in < 5 s and touches only the new record (today: every submit rescans the whole sheet). |
| NFR-5 | Scale | ≥ 5,000 transactions per user per year, no hard-coded limits. |
| NFR-6 | Privacy & law | Financial data stays in the user's own account or device; no central server holds anyone's transactions. Capture secrets never appear in shared files. Rationale: India's DPDP Act 2023 exempts only personal/domestic processing by an individual, so holding friends' data would likely bring obligations (not legal advice — get advice before any hosted model). |
| NFR-7 | Reliability | No SMS lost silently; every failure is retried or visible. |
| NFR-8 | Maintainability | All logic in version-controlled code in this repo; none hidden in spreadsheet formulas. |
| NFR-9 | Testability | Parsing rules covered by automated tests built from real (anonymised) sample SMS. |
| NFR-10 | Platform | iPhone only (decision Q1); capture via Shortcuts, the only free automatic SMS route on iOS in India. Dashboard is a responsive web page that works on phone and desktop. Android is out of scope. |
| NFR-11 | Freshness | SMS captures appear within 1 minute. |
| NFR-12 | Free-tier limits | Scheduled email polling and script runs stay well within free Google Apps Script daily quotas for a typical user (a few hundred transactions a month). |

---

## 5. Known defects in today's system (must not carry over)

| ID | Defect |
|---|---|
| D1 | Form-submit trigger runs `retryFailedRows` (full-sheet scan) instead of the per-row `processNewFormResponse`. |
| D2 | Retry check compares `"No Vendor available"`, formula writes `"No Vendor Available"` — condition never matches. |
| D3 | `getActiveSheet()` inside a trigger; breaks if tab order changes. |
| D4 | Category names differ between form, formulas and reports. |
| D5 | Fixed ranges (`A2:A1001`, `A$1:A1146`) silently cap the data. |
| D6 | Card numbers hard-coded in the Rules tab. |
| D7 | Credit SMS never categorised as income. |
| D8 | Shortcut has no retry or duplicate protection. |
| D9 | Form still has Category and Split questions the Shortcut no longer sends. |
| D10 | The "contains ICICI Bank" automation also catches promotions and OTPs, which become "Manual Review Needed" noise. |
| D11 | UPI Lite spends and cash are not captured at all. |

---

## 6. Open questions (decide before choosing a direction)

| ID | Question |
|---|---|
| Q1 | ✅ *Decided — see §0.* Do all intended users have iPhones, or is Android needed? |
| Q2 | ✅ *Decided — see §0.* Is a $99/year Apple developer account acceptable (needed to distribute a native iOS app to friends)? |
| Q3 | ✅ *Decided — see §0.* Must users see and edit data as a spreadsheet, or is an app with charts and export enough? |
| Q4 | ✅ *Decided — see §0.* Should income (salary, dividends, refunds) be captured automatically from credit SMS? |
| Q5 | ✅ *Decided — see §0.* Which Expense Status values exist beyond Valid / RegexIssues / Pending Split, and what should new records default to? |
| Q6 | ✅ *Decided — see §0.* Should Axis Bank SMS stay manual by choice, or be automatic with an easy way to skip unwanted ones? |
| Q7 | ✅ *Decided — see §0.* How many years of history must be kept and reported on? |
| Q8 | ✅ *Decided — see §0.* Do ICICI and Axis send email alerts for every transaction type you use (cards, UPI, NEFT, auto-debit)? Decides whether email can be a full channel or only a backup. |
| Q9 | ✅ *Decided — see §0.* Should the email channel be on for you too (backup for missed SMS), or only for friends without iPhones? |
| Q10 | ✅ *Decided — see §0.* Do you use UPI Lite? If yes, is logging the top-up enough, or do you want each small spend? |
| Q11 | Which sender IDs and suffixes do your ICICI / Axis alerts actually come from (check a few SMS)? |
