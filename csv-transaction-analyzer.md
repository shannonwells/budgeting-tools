# Transaction Subscription, Donation & Spending Analyzer

You are analyzing a CSV (or pasted transaction data) to:
1. Surface all **subscriptions**, **recurring charges**, and **donations** with totals and frequency
2. **Categorize all spending** into a structured taxonomy
3. Produce a **month-by-month HTML spending chart** broken down by category

The goal is a clear, complete picture of committed recurring spending AND overall spending patterns,
so the user can make informed decisions about what to keep, cancel, or cut back.

---

## Step 0: Clarify Source Before Proceeding

**Always ask which account or card this CSV is from** if the user hasn't said.

Knowing the source matters because:
- Checking account CSVs typically show bill payments, transfers, and direct debits — but NOT the
  underlying credit card spending that gets paid in a lump sum
- Credit card CSVs (e.g., AMEX, Visa, Mastercard) show the actual merchant-level purchases
- Mixing them without labeling leads to double-counting (e.g., an AMEX payment appearing in both)

**If the user provides a CSV without specifying the source, ask:**
> "Before I dive in — which account or card is this CSV from? (e.g., Chase checking, AMEX credit card,
> Schwab brokerage, etc.) This helps me categorize correctly and avoid double-counting if you're
> combining multiple files."

If the user provides multiple CSVs in one go, label each section of the report with its source.
If a CSV appears to be a checking account AND contains a large credit card payment (e.g., "AMEX PAYMENT"
or "CITI PAYMENT"), note: "This looks like your checking account. The $X,XXX credit card payment here
means some of your actual spending may live in a separate card CSV — would you like to add that too?"

---

## Step 1: Load and Parse the CSV

Accept input in any of these forms:
- A file path the user provides → read via `fs.read`
- Text pasted directly into the chat
- An uploaded file artifact → read its content

**Flexible column detection** — auto-detect column names using these common patterns:

| Field | Common column names |
|---|---|
| Date | `Date`, `Transaction Date`, `Posted Date`, `Posting Date`, `Settlement Date` |
| Description | `Description`, `Merchant`, `Payee`, `Name`, `Memo`, `Narrative`, `Details` |
| Amount | `Amount`, `Debit`, `Credit`, `Transaction Amount`, `Charge`, `Payment` |
| Category | `Category`, `Type`, `Transaction Type` (use as a hint, not authoritative) |

**Debit/credit convention:** Some CSVs use positive = debit (charge), others negative = debit.
Infer from context (most charges positive → positive-is-debit). If ambiguous, ask before proceeding.

**Robust date parsing:** Accept MM/DD/YYYY, YYYY-MM-DD, MM/DD/YY, M/D/YY, "Jan 15 2024",
"15-Jan-2024", Unix timestamps, and similar. Raise an error only if no dates can be parsed at all.

**Skip non-transaction rows:** Ignore header rows, footer totals, balance rows, blank lines, and any
row missing a valid date or amount.

**Error handling — stop and ask if:**
- The file is empty or has fewer than 2 parseable rows
- No date column can be identified with confidence
- No amount column can be identified with confidence
- Every row appears to be a transfer, payroll deposit, or refund with no purchases

If the CSV structure is unclear, briefly describe what you see (columns, row count, date range, sample
rows) and ask the user to confirm column mappings. A wrong mapping produces a meaningless report.

---

## Step 2: Categorize ALL Spending Transactions

Assign every outgoing transaction to exactly one category. Skip income (payroll, dividends, refunds,
transfers in). This categorization is used ONLY for the spending chart — it is separate from the
subscription/donation deep-dive in Steps 3–4.

### Core Categories (always present)

**🏠 Home Costs**
*The essential costs of keeping the roof over your head — distinct from utilities.*
- Mortgage or rent payments
- Home/renters insurance (e.g., **Travelers** = home insurance, State Farm, Allstate, Lemonade)
- HOA dues
- Home maintenance & repair (plumber, electrician, pest control, cleaning services, handyman)
- Property taxes (if appearing as transactions)
- Home security monitoring

**🏥 Long-Term Care & Life Insurance**
*Insurance premiums that are not home or auto related — keep these separate from Home Costs.*
- Long-term care (LTC) insurance premiums (e.g., **New York Life LTC**, Genworth, Mutual of Omaha)
- Life insurance premiums (e.g., New York Life, Northwestern Mutual, MetLife, Prudential)
- Disability insurance premiums
- Umbrella / personal liability insurance (if not bundled with home)

> **Note on New York Life:** The two recurring monthly charges from New York Life
> (`NYLLTCPREM NYLIFE ADMINISTR`) are **Long-Term Care Insurance** premiums — categorize them here,
> not in Home Costs.

**⚡ Utilities**
*Essential services that flow into the home or enable connectivity.*
- Electricity, gas, water (PG&E, ConEd, SoCal Gas, etc.)
- Internet service (Comcast, AT&T Fiber, Sonic, etc.)
- Mobile/wireless phone plans (AT&T, Verizon, T-Mobile, Mint, Visible, etc.)
- Cable/satellite TV (if billed as a utility, not a streaming subscription)
- Trash/garbage collection (e.g., Sunset Scavenger, Republic Services, Recology)
- Water delivery services

**📦 Subscriptions**
*(Same list as Step 3 — streaming, software, SaaS, memberships, AI services, etc.)*
Any service billed on a recurring schedule:
- Streaming & entertainment: Netflix, Spotify, Hulu, Apple TV+, YouTube Premium, Disney+, Max, etc.
- Software & SaaS: Adobe, Microsoft 365, Dropbox, iCloud, Google One, Notion, Figma, etc.
- Memberships: Amazon Prime, Costco, AAA, gym memberships, Instacart+, etc.
- News & media: NYT, Substack, Patreon, Medium, etc.
- AI services: ChatGPT Plus, Claude Pro, GitHub Copilot, Midjourney, etc.
- Privacy/security tools: Privacy.com, NordVPN, 1Password, etc.

**🎁 Donations**
*Charitable giving, nonprofits, crowdfunding pledges.*
- Recognizable nonprofits: Red Cross, ACLU, NPR, PBS, Wikipedia, UNICEF, etc.
- Patreon pledges to creators (if clearly a donation/support, not a product)
- Anything with "donation", "pledge", "contribute", "charity" in description

**💸 Other Discretionary**
*The catch-all for everything else that doesn't fit the above.*
- Restaurants, food delivery, groceries
- Retail shopping, clothing, Amazon purchases (non-subscription)
- Travel, transportation, gas
- Healthcare out-of-pocket costs (doctor, dentist, pharmacy)
- Entertainment, events, activities
- Personal care

**💰 Transfers / Savings** *(shown in chart but not counted as expenses)*
- Large transfers to investment/savings accounts (Morgan Stanley, Fidelity, Schwab, Vanguard)
- Credit card payments from checking (AMEX PAYMENT, CITI PAYMENT, etc.)
- Zelle or inter-account transfers to self

### Dynamic Category Detection (Important!)

After categorizing, scan the "Other Discretionary" bucket for patterns. If you find **3 or more
transactions** from the same merchant type or spending category, suggest it as a named sub-category.

Examples of dynamic categories to watch for:
- **🍽️ Dining & Restaurants** — if 3+ restaurant/DoorDash/Uber Eats charges appear
- **🛒 Groceries** — if 3+ grocery store charges appear
- **💊 Healthcare** — if 3+ medical/pharmacy charges appear
- **✈️ Travel** — if 3+ airline/hotel/Airbnb charges appear
- **🚗 Transportation** — if 3+ rideshare/gas/parking charges appear
- **🏥 Auto Insurance** — if auto insurance premiums appear (keep separate from home/LTC insurance)

When suggesting a new category, explain *why* (e.g., "You had 6 DoorDash/Uber Eats charges totaling
$142 — I've broken this out as 'Dining & Delivery' since it's a meaningful pattern.").

---

## Step 3: Identify Subscriptions and Donations (Deep Dive)

### Subscriptions / Recurring Charges

A charge is a subscription if it recurs from the same merchant more than once, OR if it appears only
once but the merchant name/description strongly implies a subscription product. Include:

- **Streaming & entertainment**: Netflix, Spotify, Hulu, Apple TV+, YouTube Premium, Disney+, Max,
  Peacock, Paramount+, Tidal, SiriusXM, Crunchyroll, etc.
- **Software & SaaS**: Adobe Creative Cloud, Microsoft 365, Dropbox, iCloud, Google One, GitHub,
  Notion, Figma, LastPass, 1Password, NordVPN, etc.
- **Memberships**: Amazon Prime, Costco, Sam's Club, AAA, gym memberships, BJ's, Instacart+, etc.
- **News & media**: NYT, Washington Post, The Atlantic, Substack, Patreon, Medium, etc.
- **Cloud & storage**: AWS, Google Cloud, Cloudflare, Backblaze, DigitalOcean, etc.
- **Phone / Internet / Cable**: if billed in a regular, fixed monthly amount
- **Insurance premiums**: if billed monthly or annually at a consistent amount
- **AI services**: ChatGPT Plus, Claude Pro, GitHub Copilot, Midjourney, etc.
- **Privacy tools**: Privacy.com subscriptions, VPN services, etc.
- **Any description containing**: "subscription", "membership", "renewal", "annual fee", "monthly fee",
  "recurring", "subscribe", "plan", "premium"

### Donations

Flag payments as donations if:
- The organization name contains: Foundation, Fund, Society, Charity, Nonprofit, 501c, Relief, Alliance,
  Coalition, Association, Institute (when clearly nonprofit)
- Recognizable nonprofits: Red Cross, ACLU, Planned Parenthood, NPR, PBS, Wikipedia/Wikimedia,
  UNICEF, Doctors Without Borders, Sierra Club, EFF, etc.
- The description contains: "donation", "gift", "pledge", "contribute", "support", "sustaining member"

### Merchant Normalization

Group transactions by merchant — critical for accurate totals:
- Strip noise: asterisks, trailing location codes, reference numbers, partial dates, store numbers
  - `NETFLIX.COM`, `NETFLIX INC`, `NETFLIX*`, `NETFLIX 123456` → **Netflix**
  - `AMZN PRIME*AB12CD`, `AMAZON PRIME MEMBERSHIP` → **Amazon Prime**
  - `WEB ONLINE PGANDE ID5940742640` → **PG&E**
  - `NYLLTCPREM NYLIFE ADMINISTR` → **New York Life (LTC Insurance)**
  - `WEB_PAY SUNSETSCAVENGER` → **Sunset Scavenger (Trash)**
  - `PAYMENT ATT ID...` → **AT&T**
  - `Privacycom PwP [MERCHANT]` → **Privacy.com → [Merchant]** (pass-through card)
  - `TRVL INS`, `TRAVELERS`, `TRAVELERS HOME` → **Travelers (Home Insurance)**
- Use fuzzy matching: if two merchant names share 80%+ of characters and appear at similar intervals
  with similar amounts, treat as the same merchant

### Frequency Classification

Analyze the gap between consecutive charges for each merchant:

| Pattern | Label |
|---|---|
| ~30 days (±10 days) | **Monthly** |
| ~90 days (±14 days) | **Quarterly** |
| ~180 days (±21 days) | **Semi-annual** |
| ~365 days (±30 days) | **Annual** |
| Single charge, name implies subscription | **Annual (one charge in data)** |
| Single charge, ambiguous | **Unknown — one charge** |
| Doesn't fit any pattern | **Irregular** |

For merchants with only 2 charges, append "— limited data" to the frequency label.

---

## Step 4: Detect Notable Patterns

While computing totals, actively look for and flag:
- **Upcoming annual renewals**: Annual charge last occurred ~11–12 months ago → flag as "renewal likely due soon"
- **Duplicate charges**: Same merchant, same amount, within 2 days → flag as possible billing error
- **Overlapping services**: e.g., both Hulu and YouTube Premium → potential redundancy
- **Price increases**: Monthly charge amount jumped → flag old price, new price, and date of change
- **Very high spend**: Any subscription or donation exceeding $500/year → call out in Observations
- **Privacy.com pass-through cards**: Note the actual underlying merchant when visible in the description
- **Credit card payment in checking CSV**: Flag it and suggest adding the card CSV for a complete picture

---

## Step 5: Calculate Totals

For each subscription or donation:

| Metric | How to calculate |
|---|---|
| **Total spent** | Sum of all charges in the dataset |
| **Typical per-period amount** | Median of all charge amounts (more robust than mean) |
| **Number of charges** | Count of included transactions |
| **First charge** | Earliest transaction date → report as Month YYYY |
| **Most recent charge** | Latest transaction date → report as Month YYYY |
| **Annualized cost** | Monthly × 12, Quarterly × 4, Semi-annual × 2, Annual × 1 |

---

## Step 6: Build the Month-by-Month Spending Chart

This is a key deliverable. Produce a **self-contained HTML file** with an interactive bar chart.

### Chart requirements

1. **X-axis**: Calendar months covered by the dataset (e.g., Jan 2026, Feb 2026, Mar 2026)
2. **Y-axis**: Dollar amount spent
3. **Bars**: Stacked bars, one color per category, ordered bottom-to-top:
   - 🏠 Home Costs (deep blue `#1e3a5f`)
   - 🏥 Long-Term Care & Life Insurance (violet `#7c3aed`)
   - ⚡ Utilities (teal `#0d9488`)
   - 📦 Subscriptions (indigo `#6366f1`)
   - 🎁 Donations (rose `#e11d48`)
   - 💰 Transfers/Savings (slate `#64748b`) — if present, shown as a separate reference bar or toggled off by default
   - Any dynamic categories (use auto-assigned colors from a readable palette)
   - 💸 Other Discretionary (gray `#94a3b8`) — always on top
4. **Hover tooltips**: On hover, show the category name and dollar amount for that month's segment
5. **Legend**: Color-coded, clickable to toggle categories on/off
6. **Summary table below chart**: Month × Category grid showing dollar amounts, with row and column totals
7. **Style**: Clean, modern, responsive. White background, rounded cards, subtle shadows.

### Transfers/Savings display

Transfers (CC payments, investment transfers) are large and will visually dwarf real spending.
Show them in the chart but make them **toggled off by default** in the legend, so the spending
categories are readable on first load. Add a note: "💡 Transfers/Savings hidden by default — click
in legend to show."

### HTML chart implementation

Use Chart.js (load from CDN) for the stacked bar chart. Structure the HTML like this:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Monthly Spending by Category</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4/dist/chart.umd.min.js"></script>
  <style>/* clean modern styles */</style>
</head>
<body>
  <div class="header">
    <h1>Monthly Spending by Category</h1>
    <p class="subtitle">Source: [filename / account label] · [date range] · [N transactions analyzed]</p>
  </div>
  <div class="chart-card">
    <canvas id="spendingChart"></canvas>
  </div>
  <div class="summary-table">
    <!-- Month × Category table with totals -->
  </div>
  <script>
    const data = { /* inline the computed monthly data */ };
    // Chart.js stacked bar chart initialization
  </script>
</body>
</html>
```

Inline all computed data directly into the JavaScript — do not rely on external data files.

**Always include the account/card source** in the subtitle (e.g., "Source: AMEX Credit Card · Jan–Apr 2026 · 87 transactions").

Save this HTML file using `fs.write` with `artifact: true` and provide the download link.

---

## Step 7: Output the Full Report

Produce the Markdown report AND the HTML chart. Present in this order:

### 7a. Markdown Report

```
# 💳 Transaction Analysis — [Account/Card Source]

**Source:** [account label or "not specified — see note"]
**Transactions analyzed:** [row count]
**Date range:** [earliest date] – [latest date]

---

## 📊 Monthly Spending by Category

[Link to HTML chart artifact]

A breakdown of all outgoing spend by month, organized into these categories:
[list the categories found, with brief note on any dynamic ones added]

---

## 📦 Subscriptions & Recurring Charges

| Service | Frequency | Per Period | Total Spent | # Charges | Started | Est. Annual |
|---|---|---|---|---|---|---|
| Netflix | Monthly | $15.49 | $185.88 | 12 | Jan 2024 | $185.88 |

**Subscription subtotal (dataset):** $X,XXX.XX
**Estimated annual subscription spend:** $X,XXX.XX/yr

---

## 🎁 Donations

| Organization | Frequency | Per Period | Total Spent | # Charges | Started | Est. Annual |
|---|---|---|---|---|---|---|
| Wikipedia | Annual | $25.00 | $25.00 | 1 (annual) | Nov 2023 | $25.00 |

**Donation subtotal (dataset):** $XXX.XX
**Estimated annual donation spend:** $XXX.XX/yr

---

## 📊 Annual Cost Summary

| Category | Est. Annual Spend |
|---|---|
| Subscriptions & Recurring | $X,XXX.XX |
| Donations | $XXX.XX |
| **Total committed spend/year** | **$X,XXX.XX** |

---

## 💡 Observations

- [Flag upcoming renewals, price increases, duplicates, overlapping services, high-spend items]
- [Note any Privacy.com pass-through charges and what merchant they route to]
- [Note any dynamic categories detected and why]
- Keep to 4–7 bullets. Specific and actionable.

---

## ❓ Unclassified / Needs Your Review

| Merchant | Amount | Date | Why Flagged |
|---|---|---|---|
```

### 7b. Formatting rules

- Sort subscriptions by **estimated annual cost descending**
- Sort donations by **total spent descending**
- Use `$` with 2 decimal places throughout
- If frequency is "Irregular" or "Unknown", write "—" in Est. Annual
- Use "—" in table cells rather than leaving them blank
- Keep Observations tight — specific bullets, not generic financial advice
- **Always present the HTML chart link prominently** — it's the most visual and useful output
- **Always include the account/card source** in the report header. If unknown, note it prominently.

---

## Edge Cases & Judgment Calls

| Situation | How to handle |
|---|---|
| **Source not specified** | Ask before proceeding (see Step 0) |
| **One-time charges** | Exclude from subscriptions unless name implies subscription. Include in spending chart. |
| **Amazon purchases** | Only include in Subscriptions if: Prime, Kindle Unlimited, Subscribe & Save, Music, Audible. Generic AMZN → Other Discretionary |
| **Refunds/credits** | Note in Observations; adjust total spent for affected merchant |
| **Variable amounts** | Note "variable" in Per Period; use total spent as-is; show range (e.g., "$8–$15") |
| **Short date range (<2 months)** | Add prominent note: "⚠️ Dataset covers less than 2 months — annual charges may be missing" |
| **Privacy.com charges** | The description often shows the real merchant (e.g., "Privacycom PwP DIGITALOCEA" → DigitalOcean). Surface the real merchant; note it routes through Privacy.com |
| **Investment transfers** | Large transfers to Fidelity, Schwab, Morgan Stanley, Vanguard → categorize as "Transfers/Savings" in chart, NOT as expenses. Toggle off by default in chart. |
| **Credit card payment in checking CSV** | Flag in Observations: "This looks like a checking account. The $X credit card payment means your actual card spending may be in a separate CSV." |
| **Transfers between own accounts** | Exclude from all spending (Zelle to self, inter-account transfers) |
| **Payroll deposits** | Exclude (income) |
| **Dividend credits** | Exclude (income) |
| **Tax payments, fees** | Exclude from recurring analysis; include in Other Discretionary in chart |
| **Duplicate rows in CSV** | Deduplicate exact matches (same date + merchant + amount); note in Observations |
| **Multi-currency** | Flag currency; don't convert unless asked |
| **New York Life premiums** | Two monthly charges (~$187 + ~$214) → **Long-Term Care Insurance** category, not Home Costs |
| **Travelers charges** | → **Home Costs** (home insurance) |

**When uncertain:** lean toward including in Unclassified rather than silently excluding.
A false negative (missing a real subscription) is worse than a false positive flagged for review.

---

## What NOT to Include (Subscription/Donation Tables Only)

Exclude from the subscription and donation tables (but DO include in the spending chart):
- One-time retail purchases, restaurants, groceries, gas stations
- ATM withdrawals, bank fees (unless recurring)
- Transfers, payments to self, inter-account moves
- Payroll deposits, tax refunds, reimbursements

---

## Final Step

After presenting the report and chart, offer:
> "Would you like me to:
> - **Drill into any category** (e.g., show me all Utilities charges, or break down Other Discretionary)?
> - **Export a PDF version** of the subscription/donation summary?
> - **Calculate what you'd save** if you cancelled specific subscriptions?
> - **Add more months** — paste in another CSV and I'll merge the data?
> - **Combine multiple accounts** — add your checking or another card CSV for a complete picture?"