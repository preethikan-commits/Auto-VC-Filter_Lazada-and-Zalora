# Voucher Eligibility Automation Tool

Streamlit application for automated voucher code (VC) eligibility processing across **Lazada** and **Zalora** marketplaces for **SG** and **MY** regions.

---

## Project Structure

```
voucher_app/
├── app.py              # Streamlit dashboard UI
├── processor.py        # All business logic & data processing
├── requirements.txt    # Python dependencies
└── README.md           # This file
```

---

## Setup & Run Locally

```bash
pip install -r requirements.txt
streamlit run app.py
```

---

## Deploy to Streamlit Community Cloud

1. Push this folder to a **GitHub repository**.
2. Go to [share.streamlit.io](https://share.streamlit.io) and sign in.
3. Click **New app** → select your repo and branch.
4. Set **Main file path** to `app.py`.
5. Click **Deploy**.

> `requirements.txt` must be present at the repository root.

---

## Input Files

| File | Required | Format | Notes |
|------|----------|--------|-------|
| SC Report | ✅ | `.xlsx` / `.xls` / `.csv` | Seller Center export for selected marketplace |
| Ecom Tracker | ✅ | `.xlsx` / `.xls` / `.csv` | Headers auto-detected (scans for STYLE# row) |
| Content File | ✅ | `.xlsx` / `.xls` / `.csv` | EAN → ALU mapping source |
| AM Exclusion Sheet | ⬜ Optional | `.xlsx` / `.xls` / `.csv` | Region-specific exclusion list |

---

## Dashboard Inputs

| Input | Type | Description |
|-------|------|-------------|
| Select Marketplace | Dropdown | `Lazada` or `Zalora` |
| Region | Dropdown | `SG` or `MY` |
| Price Tier Reference | Text | Column range e.g. `AX-BA` |
| Launch Date Cutoff | Date Picker | Products launched after this date are excluded |
| Voucher Configurations | Dynamic cards | Add multiple vouchers; each evaluated independently |

---

## Key Column Mappings

### ALU Mapping
| Source | Column | Purpose |
|--------|--------|---------|
| SC Report | `SellerSKU` | Matched against Content File EAN |
| Content File | `EAN` | Linked to SellerSKU |
| Content File | `Color No` | Source of ALU value |
| Ecom Tracker | `STYLE#` | Matched to ALU to retrieve product data |

### Ecom Tracker Column Lookups
| Field | Column Name |
|-------|-------------|
| Launch Date | `Launch Dates` |
| Ecom Status (Lazada) | `Lazada` |
| Ecom Status (Zalora) | `Zalora` |

### Price Tier Reference — Positional Mapping

Example reference: `AX-BA`

| Position | Excel Col | Field |
|----------|-----------|-------|
| START | AX | RRP |
| START+1 | AY | SRP |
| START+2 | AZ | DISC % |
| END | BA | Exclusion Remarks |

Column names are resolved **by header name first**, then by position as fallback.

### AM Exclusion Sheet Tabs
| Region | Tab |
|--------|-----|
| SG | `SG VC Exclusions` |
| MY | `MY VC Exclusions` |

---

## Inclusion Keywords — Dropdown

When the Ecom Tracker is uploaded, the app reads all unique non-blank values from the **Exclusion Remarks** column and populates a multiselect dropdown for each voucher.

- Select one or more values per voucher.
- A SKU must **exactly match** at least one selected keyword to pass the filter.
- Matching is **case-sensitive and exact** — `"OPEN FOR ALL"` does not match `"OPEN FOR ALL (10days max)"`.
- Leave the dropdown empty to skip keyword filtering for that voucher.

If the Ecom Tracker is not yet uploaded, a text input is shown as a fallback (comma-separated values).

---

## Voucher Eligibility Validation Logic

Each voucher is evaluated **completely independently**. Filters are applied in order:

| # | Filter | Rule |
|---|--------|------|
| 1 | Launch Date | Must be a valid date ≤ Cutoff Date. Excludes: blank, `00-Jan-1900`, `Past Season`, `TBC`, any non-date text. |
| 2 | Ecom Status | Must be exactly `YES` (marketplace-specific column: `Lazada` or `Zalora`). |
| 3 | RRP | Must be > 16. Excludes: blank, zero, ≤ 16. |
| 4 | SRP | Must be 0 or > 16. Excludes: SRP values between 1 and 16 inclusive. |
| 5 | Inclusion Keyword | Exclusion Remarks column must exactly match a configured keyword. Skip if no keywords selected. |
| 6 | AM Exclusion | ALU must not be excluded for this voucher's percentage. |

**Output:** `Yes-Eligible` if all filters pass. Blank if not eligible.

### SRP Examples
| SRP | Result |
|-----|--------|
| 0 | ✅ Eligible |
| 18 | ✅ Eligible |
| 16 | ❌ Excluded |
| 10 | ❌ Excluded |

---

## AM Exclusion Logic

Exclusion text is parsed case-insensitively with flexible pattern matching. Multiple rules can apply to a single ALU — the most restrictive combination is used.

### Rule 1 — Global Exclusion
Triggered by any of these phrases (case-insensitive):

- `Exclude from all Voucher`
- `Exclude from platform voucher`
- `Exclude`
- `Exclude from VC`
- `Exclude from Voucher`
- `Voucher exclusion`

**Action:** ALU is excluded from **every** voucher.

---

### Rule 2 — Exact Percentage Exclusion
Triggered by patterns like:
- `Exclude from 10% VC`
- `Exclude from 15%`
- `Exclude from 20% Voucher`

**Action:** ALU is excluded only from vouchers whose name contains the **same percentage**.

**Example:**

| Vouchers | AM Exclusion | Result |
|----------|-------------|--------|
| 10% ABC, 15% DEF | Exclude from 10% Voucher | Excluded from 10% ABC; Eligible for 15% DEF |

---

### Rule 3 — Percentage and Above Exclusion
Triggered by patterns like:
- `Exclude from 30% and above`
- `Exclude from 40% above`

**Action:** ALU is excluded from all vouchers with a percentage **≥ threshold**.

**Examples:**

| Vouchers | AM Exclusion | Result |
|----------|-------------|--------|
| 30% NMS, 40% NMS, 45% MS80 | Exclude from 30% and above | All three excluded |
| 30% NMS, 40% NMS, 45% MS80 | Exclude from 40% | Only 40% NMS excluded |
| 30% NMS, 40% NMS, 45% MS80 | Exclude from 40% and above | 40% NMS and 45% MS80 excluded; 30% NMS eligible |

---

## Output

**File name:** `Voucher_Eligibility_Output_{Marketplace}.xlsx`

| Section | Columns |
|---------|---------|
| Original SC Report | All columns from the uploaded SC Report |
| Working Columns | `ALU`, `Launch Date`, `Ecom Status`, `RRP`, `SRP`, `DISC %`, `Exclusion`, `AM Exclude` |
| Voucher Columns | One column per configured voucher — `Yes-Eligible` or blank |

**Excel formatting:**
- 🔵 Dark blue headers = SC Report columns
- 🟢 Dark green headers = Working columns
- 🟣 Purple headers = Voucher columns
- Green cell highlight = `Yes-Eligible`
- Alternating row shading
- Frozen header row
