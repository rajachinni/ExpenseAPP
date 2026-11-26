# Prior Art — What Others Have Built

Research date: 2026-11-06. Sources are Reddit (r/shortcuts, r/ynab, r/swift), TechnoFino, blogs, app listings and GitHub. Mostly anecdotal; nothing below was installed or tested.

## TL;DR

1. **People build the same thing, but cruder.** The closest match is an r/shortcuts / TechnoFino "automatic expense tracker for India": Message automations on "spent" / "debited" → Shortcut → a CSV-style Apple Note. The author calls it "a bit clumsy", and it often misses the merchant. Your system (rules engine, merchant mapping, reports) is already further along.
2. **Shortcut → Apps Script `doPost` → Sheet is the standard free bridge.** Several guides use exactly the Option A design. There is no native Google Sheets action in Shortcuts.
3. **Native iOS apps with an App Intent exist** (Walletry, WalletPal, PayShare). Walletry even exposes an App Intent for *tracking transactions via SMS*. That validates Option C, but there are reports of flaky automation permissions.
4. **Open-source Indian bank-SMS parsers exist** (PennyWise AI covers 54 institutions including ICICI and Axis; Cashiro covers 20+). Their regex patterns could seed your rule templates instead of writing every rule by hand.
5. **iOS gives apps no SMS inbox access.** A Shortcuts Message automation is the only way to capture SMS automatically on iPhone. The no-SMS alternative is India's Account Aggregator framework (e.g. Fold), which requires linking bank accounts.

## 1. Same idea: SMS → Shortcut → log

| Project | How it works | Takeaways for us |
|---|---|---|
| r/shortcuts "Sharing an automatic expense tracker for India" / TechnoFino cross-post | Two Message automations ("spent", "debited"), Run Immediately → shortcut parses → appends CSV line to an Apple Note | Same trigger as yours. Fully native, no Sheet; merchant extraction weak. Author planned a manual-entry mode — friends asked for it. |
| r/ynab "Fully automated transaction entry with iOS" (2023) | Shortcut grabs bank SMS and posts it to the YNAB API | Same capture pattern; the destination is a budgeting app's API. |
| r/shortcuts Apple Pay → Excel/Numbers threads | Wallet "Transaction" trigger → append to Numbers/Excel | Parsing the passed transaction data is a common pain point. |

## 2. Bridge pattern: Shortcut → Apps Script web app → Google Sheet

| Source | Notes |
|---|---|
| TechTiff — "I Taught My iPhone to Track My Expenses" | Apps Script `doPost` template + Shortcut using *Get Contents of URL*; web app deployed with access "Anyone". |
| fassist finance-tracker docs | `doPost` only saves the payload and returns in about 1 s; a **time-based trigger** processes it later. **Gotcha: creating a new deployment changes the URL and breaks every Shortcut — always edit the existing deployment.** |
| getfinny — "Log expenses to Google Sheets with a Shortcut (2026)" | Confirms there is no native Sheets action; the options are a free Apps Script web app or a paid webhook (Zapier). |
| r/shortcuts "Connecting Google Sheets & Shortcuts" | Community Q&A on the same bridge. |

**Implications for Option A:** design it as fast capture (append the raw SMS) followed by a separate processing step. Use a fixed deployment URL. Protect the endpoint with a token, because access must be "Anyone".

## 3. Native iOS apps (Option C territory)

| App | Relevant feature |
|---|---|
| Walletry | App Intents + Shortcuts support, Apple Pay tracking, **App Intent for tracking transactions via SMS** |
| WalletPal | Shortcuts passes Apple Pay transaction details into the app; no bank linking |
| PayShare | Wallet capture via a one-time Shortcuts automation |
| r/swift "I built an automatic expense tracking app" (2024) | Indie developer building the same idea |

Risks reported:
- Apple Developer Forums: "Always Allow" doesn't persist for the Wallet *After Payment* trigger with App Intents.
- Another thread: "Shortcut automation + AppIntents is flaky sometimes".
- The Wallet *Transaction* trigger only fires for **tap-to-pay**, not online payments or UPI, so it is of limited use in India. Bank SMS remains the main signal.

## 4. Open-source parsers (reuse candidates)

| Project | Platform | Coverage | Notes |
|---|---|---|---|
| [PennyWise AI](https://github.com/sarim2000/pennywiseai-tracker) | Android (F-Droid) | 54 Indian institutions incl. HDFC, SBI, ICICI, Axis | Pattern-based parsing on device, optional AI; CSV/PDF export. Check licence before reusing patterns. |
| Cashiro | Android (F-Droid) | 20+ Indian banks & payment apps | GPL-3.0, regex parsing, also parses PDF statements. GPL affects reuse in a closed app. |

These could provide the default **bank rule templates** (FR-P3, FR-P8, FR-U3) for friends' banks you don't use yourself.

## 5. The non-SMS route

- iOS gives third-party apps no access to the SMS inbox. Indian iPhone trackers either use Shortcuts forwarding, statement import, or the **Account Aggregator** framework.
- **Fold** (Account Aggregator) is the most-cited automatic option for iPhone users in India. It needs no SMS, but you must link your bank accounts.
- One App Store listing (iLekka) claims automatic SMS tracking on iPhone. That conflicts with the platform restriction, so treat the claim with suspicion.

## What this means for our direction

| Question | Evidence says |
|---|---|
| Is SMS + Shortcuts viable long-term? | Yes. It's the only automatic route on iPhone short of Account Aggregator, and many people use it. |
| Option A (Sheet + Apps Script) | Proven pattern with free tooling. Watch the deployment-URL gotcha and token security. |
| Option C (native app + App Intent) | Proven by shipping apps. Costs $99/yr to distribute, and there are some automation-permission bugs. |
| Our differentiator | The rules engine + merchant mapping + review queue. Most community builds stop at "log the raw amount". |
| Shortcut to faster coverage for friends | Seed rule templates from PennyWise / Cashiro patterns (mind the licences). |

## 6. India-specific context

### 6.1 What Indian builders are doing

| Project | Approach | Notes |
|---|---|---|
| r/shortcuts / TechnoFino tracker | iOS Message automation → Shortcut → Apple Note CSV | Author: "couldn't find any reliable apps that automatically track spending in India". |
| [kharcha-core](https://docs.rs/crate/kharcha-core/0.1.0) (Rust crate) | Rule-based core: SMS parsing, **spam rejection**, categorisation, money math, **dedupe by UPI reference + content hash + time window** | Good reference design for our parser / dedupe (FR-C7). |
| DEV.to "Automate your UPI transaction tracking — 60 lines of Python" | Per-bank regex for HDFC, Axis, SBI, ICICI over an SMS export | Author warns: **regex breaks whenever a bank changes its template.** |
| AuditMint, Trakio (Android) | Read SMS + notifications on device; Trakio asks the user to confirm each item | Confirm-each-item matches our review queue idea. |
| HisabKarle, Ruply | Paste SMS or screenshots; AI extracts amount / merchant / date | AI as a fallback when rules fail. |
| [cc-transaction-trackers](https://github.com/hariks789/cc-transaction-trackers), Medium HDFC UPI tutorial, [Biyete](https://dev.to/clsource/biyete-parse-bank-emails-and-send-it-to-google-sheets-4c13) | **Gmail + Apps Script**: parse bank alert *emails* into Google Sheets on a time trigger, with Gmail labels to mark processed mail | No phone involved. HDFC / Amex / IndusInd examples; ICICI and Axis would need their own patterns. |

### 6.2 Commercial apps in India

| App | How it gets data | iPhone? |
|---|---|---|
| Walnut / Axio, Money View, AuditMint, Mera Kharcha | Read SMS inbox | **Android only** — iOS doesn't allow it |
| Fold (Rainmatter-backed), INDmoney, Fi | **Account Aggregator** (RBI consent framework, since Sept 2021) | Yes, but you link bank accounts; budgeting is secondary for INDmoney |
| Slice AI, others | Read Gmail bank alerts | Yes |
| Finnie and similar | Manual / AI-assisted entry | Yes |

- **AA ecosystem is still settling:** PhonePe surrendered its AA licence in Feb 2025 after failing to onboard enough partners. Reports in late 2024 still noted gaps in bank coverage. Building on AA ourselves would need regulated partners, so it isn't realistic for a hobby project.
- Most "best tracker" lists are vendor blogs. Consensus: on iPhone, automatic tracking means Shortcuts forwarding, email alerts, or AA.

### 6.3 Indian SMS / payments quirks that affect design

| Quirk | Impact | What to do |
|---|---|---|
| **TRAI sender suffixes (from May 2025):** headers end in `-T` (transactional), `-S` (service), `-P` (promotional), `-G` (govt), e.g. `BP-HDFCBK-S` | "Message contains ICICI Bank" also catches promos and OTPs → noise / "Manual Review Needed" rows | Filter or tag by sender header; drop `-P` messages. Sources disagree on whether bank alerts are `-T` or `-S`, so check against your own SMS. |
| **UPI Lite** (on-device wallet, small payments) | No per-payment SMS — the bank sends **one consolidated SMS** at intervals; only top-ups appear in statements | Small daily spends (tea, auto) are invisible to SMS tracking. Options: log the top-up as a "wallet" expense, or allow quick manual entry. |
| **Bank template changes** | Regex rules silently stop matching | Rules as data (FR-P7), alert when a sender's match rate drops, test suite of real samples (NFR-9). |
| **Same transaction, multiple alerts** (SMS + email + UPI app notification) | Double counting if several channels are combined | Dedupe on UPI ref / amount + time window (as kharcha-core does). |
| **Promos and OTPs from bank senders** | Junk rows | Spam rejection step before parsing. |

### 6.4 Email alerts — a channel we hadn't considered

Every Indian bank also emails transaction alerts. A Gmail-triggered Apps Script (time-based, every 5–15 min) can parse them into the same Sheet:
- **Pros:** no Shortcut and no phone dependency, so it **works for Android friends too**; fully server-side and free; it's the user's own Google account.
- **Cons:** delayed by the trigger interval; not every bank or account sends emails for every transaction (UPI especially); needs Gmail read permission; a separate parser per bank's email format.
- Could run **alongside** SMS capture, with dedupe across both.

### 6.5 Privacy and law (DPDP Act 2023)

- The Act **does not apply to personal data processed by an individual for personal or domestic purposes.** Your own tracker is fine.
- Hosting **friends'** financial data on something you run is likely outside that exemption. The final DPDP Rules were notified in Nov 2025.
- **Design implication:** each friend's data should live in *their own* Google account or device, never in anything you host. Options A (per-user Sheet + script) and C (on-device + their iCloud) both satisfy this; a central server you run would not. This isn't legal advice — get a lawyer's view before hosting others' data.

### 6.6 India takeaways for the decision

1. iPhone + Shortcuts SMS forwarding is the **only free, automatic, no-bank-linking route on iOS in India** — your current approach is the right base.
2. Add a **Gmail alert channel** as an option. It makes the system work for Android friends without building an Android app.
3. Use **TRAI sender suffixes and spam rejection** to cut noise before parsing.
4. Plan for **UPI Lite blind spots** and **template drift**.
5. Keep every user's data in their own account (DPDP) — rules out a central hosted backend.

## 7. Possible solutions (for review)

All options must fit the decisions in [REQUIREMENTS.md §0](REQUIREMENTS.md): zero running cost, all-iPhone users, each person's data in their own account, spreadsheet + dashboard preferred, few moving parts.

Two separate choices are being made:
- **Platform** — where data lives and where the logic and UI run (7.1).
- **Data source** — how transactions get in (7.2).

### 7.1 Platform options

#### P1. Google Sheets + Apps Script web app  ⭐ current front-runner
```
SMS → iOS automation → Shortcut ──POST + token──► Apps Script web app (user's Google account)
                                                   ├─ parse (Rules) → Transactions tab
                                                   └─ serves dashboard (HTML) for phone + desktop
```
- **Cost:**
  | Item | Free allowance | If outgrown |
  |---|---|---|
  | Google account, Sheets, Apps Script | ₹0; 15 GB Drive storage shared with Gmail/Photos (the sheet needs a few MB) | Google One storage, if Drive is full for other reasons |
  | Script runtime | 6 min per execution; **90 min/day total trigger runtime** (consumer accounts) | Google Workspace (paid per user) raises it to 6 h/day — not needed at a few hundred SMS/month |
  | Triggers / URL fetch | 20 time-driven triggers per script; 20,000 URL fetches/day | — |
  | Web-app requests | No separate published quota; covered by runtime limits | — |
  | Build effort (one-time) | Medium: script + HTML dashboard + shared library | — |
- **Friend setup:** copy the template sheet → run "Setup" → deploy the web app → paste the URL + token into the Shortcut. About 10 min.
- **Pros:**
  - Proven community pattern.
  - Spreadsheet and dashboard both included.
  - Shared code library pushes fixes to every friend.
  - Data stays in each user's own Drive.
- **Cons:**
  - Web app must allow "Anyone", so the endpoint needs a secret token.
  - A new deployment changes the URL (always edit the existing one).
  - Apps Script quotas and cold starts (~1–3 s).
  - Depends on Google.
- **Other drawbacks:**
  - **Google "unverified app" warning** when each friend authorises the script. It's a scary screen for non-technical users; avoiding it needs Google OAuth verification.
  - **Library versioning:** friends' copies pin a library version, so fixes reach them only after a version bump (or when using the risky "HEAD"/development mode).
  - **Leaked token = junk rows.** Anyone with the URL + token can post; tokens need rotating and the endpoint needs basic validation.
  - **Occasional Google errors or timeouts:** the Shortcut needs a retry, or the SMS is lost (only the Apple Note log keeps it).
  - The dashboard runs in Google's sandboxed iframe, with a banner ("created by a Google Apps Script user"), and is slower than a normal web app.
  - Google states quotas "may change at any time without notice".
- **Fits decisions:** ✅ all.

#### P2. Scriptable + iCloud Drive (all on the phone)
```
SMS → iOS automation → Shortcut → Scriptable script (JavaScript, on iPhone)
                                   ├─ parse (rules.json) → transactions.csv/json in iCloud Drive
                                   └─ home-screen widgets / in-app report; Numbers for spreadsheet view
```
- **Cost:**
  | Item | Free allowance | If outgrown |
  |---|---|---|
  | Scriptable app | ₹0 (free app; optional tip) | — |
  | iCloud storage | 5 GB free; data is a few hundred KB/year | iCloud+ only if the user's iCloud is already full |
  | Build effort (one-time) | Medium–high: parser + widgets + review UI built in Scriptable's UI kit | — |
- **Friend setup:** install Scriptable, import the script + Shortcut. About 5 min.
- **Pros:**
  - No internet endpoint, token or Google account. Works offline.
  - Data in each user's iCloud — the strongest privacy story.
  - Instant.
- **Cons:**
  - No web dashboard on a laptop; the Mac only gets Numbers / Finder.
  - Widget and report UI must be hand-built.
  - Rule editing is a JSON file or a simple in-app form.
  - Code updates are per device (re-import the script).
- **Other drawbacks:**
  - **Single-developer app:** if Scriptable stops being updated for a new iOS version, the whole system breaks. There's no alternative runtime with the same features.
  - **iCloud sync quirks:** a file that hasn't downloaded yet (a placeholder) or edits from two devices can cause failed reads or conflicts. The code must handle "download first".
  - Scripts started from Shortcuts in the background have iOS time and memory limits. Heavy work (reports) must run when the app is open.
  - No Windows/web access at all; the Mac only sees raw files (or Numbers).
  - Backup and restore are the user's job (iCloud only); deleting the folder loses data.
  - Debugging on a phone is slow for the developer.
- **Fits decisions:** ⚠️ partial — "spreadsheet + dashboard on desktop" is weak.

#### P3. Notion (free plan)
```
SMS → Shortcut (parses with regex) ──Notion API + user token──► Notion databases (Transactions, Merchants, Categories)
                                                                 └─ Notion views / charts as dashboard
```
- **Cost:**
  | Item | Free allowance | If outgrown |
  |---|---|---|
  | Notion personal plan | ₹0 | Plus plan (paid per member/month) for longer page history, more charts and features |
  | Notion API | Free; **about 3 requests/second** per integration (HTTP 429 above) | — |
  | Build effort (one-time) | Low–medium: templates + Shortcut parsing logic | — |
- **Friend setup:** duplicate the Notion template, create an integration token, paste it into the Shortcut. About 10 min.
- **Pros:**
  - Polished UI on phone and desktop.
  - Easy manual edits, filters and charts.
- **Cons:**
  - No server-side code, so parsing has to happen inside the Shortcut (regex in Shortcuts is clumsy and hard to maintain).
  - Notion formulas and rollups are limited for reports.
  - Depends on Notion; API rate limits apply.
- **Other drawbacks:**
  - **Rules live inside each friend's Shortcut.** Any parsing fix means everyone re-imports the Shortcut, and their customisations are lost.
  - Budget-vs-actual, monthly %, and split logic are hard to express in Notion formulas and rollups.
  - Free-plan limits on charts and page history; features can move between plans.
  - Weak offline support; Notion is slow with thousands of rows.
  - Notion token stored in the Shortcut (same leak risk as P1's token).
  - Export is per-database CSV; no single clean backup.
- **Fits decisions:** ⚠️ partial — no spreadsheet; parsing logic lands in the Shortcut.

#### P4. Cloudflare Workers + D1 (free tier)
```
SMS → Shortcut ──POST──► Worker API ──► D1 (SQLite) ; Worker also serves dashboard
```
- **Cost:**
  | Item | Free allowance | If outgrown |
  |---|---|---|
  | Workers | **100,000 requests/day** | Workers Paid plan (small monthly fee, ~$5) |
  | D1 database | **5 GB storage; 5M rows read/day; 100k rows written/day**. Since 1 Sep 2026, queries **fail** when the daily limits are exceeded (resets 00:00 UTC) | Workers Paid usage pricing |
  | Domain | Free `*.workers.dev` subdomain | Own domain (~₹800–1,500/yr) is optional |
  | Build effort (one-time) | High: API + auth + dashboard + migrations | — |
- **Pros:** a real database and real code, fast, full control.
- **Cons:**
  - Either **you host everyone's data** (DPDP concern — see 6.5), or each friend deploys their own Cloudflare account (too technical).
  - No spreadsheet.
- **Other drawbacks:**
  - **You run a service:** uptime, security patches, backups and login are your responsibility, and friends depend on you.
  - **Authentication must be built** (or Cloudflare Access configured), plus per-user tokens for the Shortcut.
  - A breach or bug exposes everyone's financial data at once.
  - Free-plan CPU time per request is small; heavy report queries must be efficient.
  - Spreadsheet export must be built.
- **Fits decisions:** ❌ conflicts with "data in user's own account" unless deployed per user.

#### P5. Supabase / Firebase (free tier)
- Hosted database + sign-in + API; proper multi-user.
- **Cost:**
  | Item | Free allowance | If outgrown |
  |---|---|---|
  | Supabase Free | ~500 MB database, ~5 GB egress, 2 active projects; **pauses after 7 days with no database requests** | Pro ~$25/month per project (no pausing) |
  | Firebase Spark | Firestore 1 GiB stored, **50k reads / 20k writes per day** | Blaze pay-as-you-go (**needs a card**; required for Cloud Functions, i.e. server-side parsing) |
  | Build effort (one-time) | High: auth + API + dashboard | — |
- **Cons:**
  - Central data you control (DPDP).
  - Supabase pauses idle free projects.
  - Free-tier limits can change.
- **Other drawbacks:**
  - Firebase server-side code (Cloud Functions) needs the card-linked Blaze plan, so "free" only holds if parsing runs on the client.
  - A paused Supabase project rejects captures until someone restores it, so SMS could be lost during a holiday.
  - Same "you run a service" burden as P4 (security, backups, support).
- **Fits decisions:** ❌ same issue as P4.

#### P6. Airtable (free plan)
- **Cost:** Free plan: **1,000 records per base**, **1,000 API calls per month per workspace**, 100 automation runs/month, 5 requests/second. Paid plans (Team and up) are priced per user per month.
- **Drawbacks:**
  - The API cap alone fails: ~1,000 transactions/year plus reads exceeds 1,000 calls/month quickly.
  - The record cap is reached in about a year.
- **Rejected.**

#### P7. Phone-only web app (PWA, data in browser storage)
- Static site (GitHub Pages) storing data in the browser.
- The Shortcut has to *open Safari* with the SMS on every message, which is disruptive.
- Data is lost if browser storage is cleared. **Not recommended** for automatic capture.
- **Cost:** ₹0 (GitHub Pages / Cloudflare Pages hosting).
- **Other drawbacks:**
  - Safari can evict script-written storage for sites not used for 7 days (home-screen apps are treated better, but it's still not a database).
  - No sync between phone and laptop.
  - Backups are manual exports.

#### P8. Native iOS app (SwiftUI + App Intent + iCloud)
- Best UX: silent capture via App Intent, widgets.
- **Needs $99/yr to distribute to friends**, which conflicts with decision Q2. **Parked** unless Q2 changes.
- **Cost:**
  | Item | Amount |
  |---|---|
  | Apple Developer Program | **$99/year (~₹8,300)**, needed for TestFlight / App Store |
  | Mac with Xcode | Required for building (you likely have one) |
  | CloudKit (iCloud sync) | Free within Apple's per-user quotas |
  | Build effort (one-time) | High: SwiftUI app + App Intent + reports |
- **Other drawbacks:**
  - TestFlight builds expire after 90 days; App Store review is needed for wider sharing.
  - Reported flakiness with Shortcuts "Always Allow" permissions for App Intents.
  - Yearly iOS updates need app maintenance.
  - Without the paid account, sideloaded builds expire every 7 days.

#### Drawbacks common to every option
- **Depends on the iOS Shortcuts Message automation.** If Apple changes or restricts it, automatic capture stops for all options.
- **Bank message formats drift**, so rules need upkeep whatever the platform.
- **The Shortcut has no built-in retry or outbox.** A capture can fail silently unless we design a retry (FR-C6).
- **Your own time** is the real cost: build plus ongoing rule maintenance for each bank your friends use.

### 7.2 Data-source options

| # | Source | Freshness | Merchant quality | Coverage | Effort for user | Notes |
|---|---|---|---|---|---|---|
| S1 | **Bank SMS via Shortcuts** (current) | Instant | Good when a rule matches; misses on new formats | Misses UPI Lite and any SMS the automation skips | None after setup | Only free automatic route on iPhone. |
| S2 | **Bank alert emails** (Gmail + Apps Script) | 5–15 min | Similar to SMS | Varies; some UPI flows send no email | None after setup | Only works with P1. Decided "No" for v1. |
| S3 | **Monthly statements** (credit card + savings, PDF/CSV upload) | Monthly (savings can be downloaded any time) | **Good** — card lines carry the merchant (e.g. `SWIGGY BANGALORE`); savings lines carry the UPI narration (`UPI/<ref>/<payee>/…`) | **Complete** — includes fees, interest, cashback, UPI Lite, missed SMS | Download + upload monthly; PDFs are password-protected | Per-bank parser; PDF layouts change; no live budget alerts. |
| S4 | **Account Aggregator** (Fold etc.) | Near real-time | Good | Complete for linked accounts | Link bank accounts | Building on AA needs a regulated licence or partner — not feasible for us. Users can *use* Fold separately. |
| S5 | **Manual entry** | Instant | User-typed | Cash, UPI Lite, anything else | Per transaction | Needed anyway as a fallback (FR-C10). |

**About statements (S3):** merchants are *not* lost. Statements usually name the merchant or payee as well as or better than SMS. The real drawbacks are:
1. Monthly delay, so no in-month budget tracking.
2. PDF unlocking and table parsing per bank, which is fragile.
3. A manual monthly step.

Their strength is completeness and authoritative amounts.

**Suggested combination:** S1 as the live source + S5 for gaps + S3 later as a **monthly reconciliation**. Upload the statement, auto-match it to captured transactions, and flag missing or mismatched items. This maps to FR-X5 (COULD) and needs no change to the v1 platform choice.

### 7.3 Comparison matrix

| | P1 Sheets + Apps Script | P2 Scriptable + iCloud | P3 Notion | P4 CF Workers + D1 | P8 Native iOS |
|---|---|---|---|---|---|
| Running cost | Free | Free | Free | Free tier | $99/yr |
| Paid upgrade if outgrown | Workspace (unlikely) | None needed | Notion Plus | Workers Paid ~$5/mo | — |
| Hard free-tier risk | 90 min/day trigger runtime | iOS background limits | ~3 req/s API | Queries fail over daily row limits | — |
| Build effort | Medium | Medium–high | Low–medium | High | High |
| Biggest other drawback | Unverified-app warning, token leak | Single-dev app, no desktop | Rules inside Shortcut | You host everyone's data | $99/yr, 90-day TestFlight |
| Data in user's own account | ✅ Google Drive | ✅ iCloud | ✅ Notion | ❌ unless per-user deploy | ✅ iCloud |
| Spreadsheet view | ✅ native | ⚠️ via Numbers | ❌ | ❌ | ❌ (export) |
| Dashboard phone + desktop | ✅ web app | ⚠️ phone widgets only | ✅ | ✅ | ⚠️ phone only |
| Silent SMS capture | ✅ | ✅ | ✅ | ✅ | ✅ |
| Where parsing runs | Server-side script | On-device JS | Inside Shortcut ⚠️ | Server-side | On-device Swift |
| Rule editing UX | Sheet tab / dashboard form | JSON / simple form | Notion table (parsing still in Shortcut) | Dashboard form | App screen |
| Friend setup | ~10 min | ~5 min | ~10 min | Hard | App install |
| Fixes reach everyone | ✅ shared library | ❌ re-import | ❌ Shortcut re-import | ✅ | ✅ app update |
| Works offline | ❌ | ✅ | ❌ | ❌ | ✅ |
| Vendor dependency | Google | Apple + Scriptable dev | Notion | Cloudflare | Apple |
| Moving parts | 2 (Shortcut, script) | 2 (Shortcut, Scriptable) | 2 (Shortcut, Notion) | 2–3 | 2 |
| Statement reconciliation later | ✅ easy (upload to Drive) | ⚠️ PDF parsing on-device | ❌ hard | ✅ | ✅ |

### 7.4 Current recommendation

1. **P1 (Google Sheets + Apps Script web app) with S1 (SMS) + S5 (manual entry)** for v1. It's the only option that meets every decision.
2. **Keep P2 (Scriptable + iCloud) as the fallback** if Google dependency, endpoint security, or Apps Script quotas become a problem.
3. **Add S3 (statement reconciliation) in phase 2.**

### 7.5 Review checklist (owner + co-developer)

Mark ✅ / ❌ / ❓ and add comments.

| # | Check | Owner | Co-dev | Comments |
|---|---|---|---|---|
| 1 | Agree the platform choice is P1, with P2 as fallback | | | |
| 2 | OK with depending on Google (Sheets + Apps Script) for v1 | | | |
| 3 | OK with a public web-app URL protected by a per-user secret token | | | |
| 4 | Friend setup of ~10 min (copy sheet, deploy, paste URL) is acceptable | | | |
| 5 | Dashboard served from Apps Script (HTML) is good enough on phone + desktop | | | |
| 6 | Statements as phase-2 reconciliation, not a v1 source | | | |
| 7 | Account Aggregator (S4) and native iOS app (P8) parked | | | |
| 8 | Anyone wants to prototype P2 (Scriptable) as a comparison spike? | | | |
| 9 | Any other platform we missed? | | | |

## Sources

- [r/shortcuts — Sharing an automatic expense tracker for India](https://www.reddit.com/r/shortcuts/comments/1ko40wz/sharing_an_automatic_expense_tracker_for_india/)
- [TechnoFino — Expense tracker for iOS/Mac using Shortcuts app](https://technofino.in/community/threads/expense-tracker-for-ios-mac-using-shortcuts-app.40920/)
- [r/ynab — Fully automated transaction entry with iOS](https://www.reddit.com/r/ynab/comments/12vu40s/solved_fully_automated_transaction_entry_with_ios/)
- [r/shortcuts — Tutorial: automatically add Apple Pay transactions](https://www.reddit.com/r/shortcuts/comments/17dqih9/tutorial_how_to_automatically_add_apple_pay/)
- [r/shortcuts — Apple Pay to Excel spreadsheet](https://www.reddit.com/r/shortcuts/comments/1e6ecoz/apple_pay_to_excel_spreadsheet/)
- [r/shortcuts — Connecting Google Sheets & Shortcuts](https://www.reddit.com/r/shortcuts/comments/1fq0usz/connecting_google_sheets_shortcuts/)
- [r/swift — I built an automatic expense tracking app](https://www.reddit.com/r/swift/comments/1gcoh4e/update_i_built_an_automatic_expense_tracking_app/)
- [TechTiff — I Taught My iPhone to Track My Expenses](https://techtiff.substack.com/p/apple-wallet-expense-tracker-shortcuts)
- [fassist — finance tracker docs](https://fassist.pages.dev/finance-tracker-docs)
- [getfinny — Log expenses to Google Sheets with a Shortcut (2026)](https://getfinny.app/blog/log-expenses-google-sheets-shortcut-2026)
- [getfinny — Best expense tracker apps in India 2026](https://getfinny.app/blog/best-expense-tracker-apps-india-2026)
- [getfinny — SMS expense tracking apps](https://getfinny.app/blog/sms-expense-tracking-app)
- [Walletry on the App Store](https://apps.apple.com/app/id6458391876)
- [WalletPal](https://walletpalapp.github.io/)
- [PayShare on the App Store](https://apps.apple.com/app/id6757592649)
- [Apple Developer Forums — "Always Allow" not persisting for Wallet trigger](https://developer.apple.com/forums/thread/850369)
- [Apple Community — Shortcut automation + AppIntents is flaky](https://discussions-jp-prz.apple.com/thread/256123920)
- [PennyWise AI on GitHub](https://github.com/sarim2000/pennywiseai-tracker) · [F-Droid](https://f-droid.org/zh_Hans/packages/com.pennywiseai.tracker)
- [Cashiro on F-Droid](https://f-droid.org/it/packages/com.ritesh.cashiro)

**India-specific**
- [kharcha-core (docs.rs)](https://docs.rs/crate/kharcha-core/0.1.0)
- [DEV — Automate your UPI transaction tracking in 60 lines of Python](https://dev.to/automate-archit/automate-your-upi-transaction-tracking-60-lines-of-python-free-works-offline-5gcm)
- [AuditMint on Product Hunt](https://www.producthunt.com/p/auditmint-auto-expense-tracker/auditmint) · [Trakio](https://www.producthunt.com/p/trakio-2/trakio-3) · [HisabKarle](https://webmail.hunted.space/product/hisabkarle) · [Ruply](https://www.producthunt.com/products/ruply/makers)
- [cc-transaction-trackers (GitHub)](https://github.com/hariks789/cc-transaction-trackers)
- [Medium — Automatically log your expenses using Google Apps Script](https://threetweetthread.medium.com/automatically-log-your-expenses-in-spreadsheet-using-google-app-scripts-4f487362db40)
- [gsmart — Parse Gmail into Google Sheets](https://blog.gsmart.in/?p=1091)
- [Biyete — parse bank emails to Google Sheets](https://dev.to/clsource/biyete-parse-bank-emails-and-send-it-to-google-sheets-4c13)
- [Zerodha Rainmatter — Introducing Fold](https://zerodha.com/z-connect/rainmatter/introducing-fold)
- [YourStory — PhonePe exits Account Aggregator business (Feb 2025)](https://yourstory.com/2025/02/phonepe-exits-account-aggregator-business-embracing-partnership-aa)
- [Daily Guardian — Account Aggregator system faces gaps](https://www.magzter.com/en/stories/newspaper/The-Daily-Guardian/ACCOUNT-AGGREGATOR-SYSTEM-FACES-GAPS)
- [Storyboard18 — TRAI's new SMS header format explained](https://www.storyboard18.com/how-it-works/why-your-sms-now-looks-like-a-code-trais-new-message-format-explained-70270.htm)
- [MSG91 — Suffix in SMS sender ID](https://msg91.com/help/dlt-registration-in-india/suffix-in-sms-sender-id)
- [India TV — SMS suffixes S, G, P, T explained](https://www.indiatvnews.com/technology/news/understanding-sms-suffixes-what-s-g-p-and-t-mean-and-how-they-help-you-avoid-scams-2025-08-29-1005732)
- [NPCI circular OC-138 — UPI Lite](https://www.npci.org.in/PDF/npci/upi/circular/2022/UPI-OC-138-Introduction-of-On-Device-wallet-UPI-Lite-for-Small-Value-Transactions.pdf)
- [PayU — What is UPI Lite](https://payu.in/blog/what-is-upi-lite/)
- [Axis Bank — What is UPI Lite](https://www.axis.bank.in/blogs/payments/what-is-upi-lite)
- [kharcha-platform — expense tracking on iPhone in India](https://kharcha-platform.web.app/pages/blog_8.html)
- [getfinny — Best expense tracker apps in India 2026](https://getfinny.app/blog/best-expense-tracker-apps-india-2026)
- [AZB & Partners — DPDP Act 2023](https://www.azbpartners.com/?p=47234)
- [EY — DPDP Act 2023 and Rules 2025](https://www.ey.com/content/dam/ey-unified-site/ey-com/en-in/pdf/2025/11/ey-india-dpdp-act-2023-and-rules-2025-pov.pdf)

**Platform options & costs (§7)** — limits checked 2026-11-10 from the official pages below where available; some (Supabase, Airtable, Notion plan details) come from third-party summaries. Vendors change limits without notice, so re-check before building.
- [Apps Script quotas (official)](https://developers.google.com/apps-script/guides/services/quotas)
- [Cloudflare D1 pricing (official)](https://developers.cloudflare.com/d1/platform/pricing/) · [D1 free-tier enforcement changelog, Sep 2026](https://developers.cloudflare.com/changelog/post/2026-09-01-d1-free-tier-limit-enforcement/)
- [Notion API request limits (official)](https://developers.notion.com/reference/request-limits)
- [Firebase pricing plans (official)](https://firebase.google.com/docs/projects/billing/firebase-pricing-plans)
- [Airtable plans (official support)](https://support.airtable.com/docs/airtable-plans.md) · [Supabase free tier limits 2026 (summary)](https://automationatlas.io/answers/supabase-free-tier-limits-2026/)
- [Scriptable](https://scriptable.app) · [Notion API](https://developers.notion.com) · [Cloudflare Workers / D1 pricing](https://developers.cloudflare.com/d1/platform/pricing/) · [Supabase pricing](https://supabase.com/pricing) · [Airtable pricing](https://airtable.com/pricing) · [Apps Script quotas](https://developers.google.com/apps-script/guides/services/quotas)
