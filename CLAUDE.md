# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file interactive HTML tax estimator for a **married filing jointly (MFJ)** investor with
no state income tax whose income is primarily qualified dividends and long-term capital gains.
The tool visualizes how capital gains and ordinary income interact with the preferential LTCG/QD
tax brackets and helps identify the maximum tax-free capital gain harvest limit.

**File:** `index.html` — all HTML, CSS, and JS in one file. No build step, no dependencies
beyond a Google Fonts import. Open directly in a browser.

## Development

```bash
open index.html          # open in default browser (macOS)
python3 -m http.server   # optional local server if needed for font loading
```

There is no build, lint, or test toolchain. Edit `index.html` and reload the browser.

## Development Rules
- All displayed values must be derived from `calcTax()` — never recompute tax logic inline in the UI layer.
- After any change to `calcTax()`, verify that the bracket bar, headroom badge, metrics grid, breakdown panel, and callout are all consistent.
- After any structural change, update CLAUDE.md to reflect it before committing.

---

## 2026 Tax Parameters (all confirmed unless noted)

```javascript
STD_DED = 32200; // MFJ standard deduction — confirmed
THRESH_0 = 98900; // Top of 0% LTCG bracket (taxable income) — confirmed
THRESH_15 = 613700; // Top of 15% LTCG bracket (taxable income) — confirmed
NIIT_THRESHOLD = 250000; // MAGI threshold for 3.8% NIIT (MFJ) — not inflation-adjusted
```

### Ordinary Income Brackets — confirmed

| Rate | Taxable Income Ceiling |
| ---- | ---------------------- |
| 10%  | $24,800                |
| 12%  | $100,800               |
| 22%  | $211,400               |
| 24%  | $403,550               |
| 32%  | $512,450               |
| 35%  | $768,700               |
| 37%  | unlimited              |

---

## Core Tax Logic

### The Stacking Rule (critical — easy to get wrong)

Per the **IRS Qualified Dividends and Capital Gain Tax Worksheet**:

1. Compute `totalTaxable = max(0, gross - STD_DED)`
2. `investTaxable = max(0, min(divs + cg, totalTaxable))` — investment income is capped at taxable income, clamped to ≥ 0
3. `taxableOI = totalTaxable - investTaxable` — ordinary income gets whatever is left

This means **the standard deduction falls against ordinary income first**, because investment
income anchors to the top of the stack. A common mistake is computing `taxableOI = min(OI, totalTaxable)`, which wrongly assigns ordinary income before the deduction is absorbed. With `OI = $30,000` and `STD_DED = $32,200`, the correct `taxableOI` is `$0` — not `$30,000`.

### LTCG Rate Application

After establishing `taxableOI`:

```
zeroHeadroom = max(0, THRESH_0 - taxableOI)
inv_at0      = min(investTaxable, zeroHeadroom)
inv_at15     = max(0, min(investTaxable, THRESH_15 - taxableOI) - zeroHeadroom)
inv_at20     = max(0, investTaxable - max(0, THRESH_15 - taxableOI))
```

### Capital Losses

The LTCG slider runs from **−$3,000** (IRS annual loss deduction limit) to $300,000.
A net loss reduces `gross` income, which reduces `totalTaxable`. `investTaxable` and NIIT base
are both clamped to `≥ 0` so a loss can't make investment income negative.

### NIIT

```javascript
nii = max(0, divs + cg); // net investment income, can't be negative
niitBase = max(0, min(nii, gross - NIIT_THRESHOLD)); // only the NII portion above the MAGI floor
niitTax = niitBase * 0.038;
```

---

## UI Components

### Two Sliders

| Slider             | Color                      | Range          | Notes                                                         |
| ------------------ | -------------------------- | -------------- | ------------------------------------------------------------- |
| Ordinary Income    | Orange `--orange: #e07b39` | $0 – $300k     | Wages, pension, RMDs, SS, interest, non-QD, short-term gains  |
| Investment Income  | Green `--green: #3fb950`   | −$3k – $600k   | QD + LTCG combined; negative = net capital loss; starts $60k  |

Both sliders use `step="100"`. Each has a `.slider-sources` description line and a color-coded rate label on the right.
The OI slider also renders a `#oiMarginalBadge` pill below it showing the marginal ordinary rate.

### Bracket Track Bar

Shows **taxable income only** (after the standard deduction — deduction is not rendered as a block).
Segments: orange (ordinary), green (QD/LTCG at 0%), ghost/empty (unused 0% headroom), yellow (15%), red (20%).
A `#headroomBadge` pill in the panel header shows exact remaining 0% headroom in dollars, turning
yellow when < $20k.

### Metrics Grid (6 cards)

Ordinary Tax · Invest. at 0% · Invest. at 15% · Invest. at 20% · NIIT · Total Tax

### Chart

Canvas line chart of **total tax vs. capital gains** at the current OI and dividend levels.

- X-axis: −$3k to $600k. Tick marks at $0, $100k … $600k.
- A faint dashed vertical marks the $0 boundary (loss/gain line).
- Blue dashed vertical + dot marks the current CG slider position.
- Dashed orange line shows the flat ordinary income tax floor.
- 61 data points (`POINTS = 61`).

X-position mapping: `xp = v => pad.left + ((v - minX) / (maxX - minX)) * cw` where `minX = -3000`.

Dot index: `cIdx = round(((currentInvest - (-3000)) / (600000 - (-3000))) * 60)`

### Breakdown Panel

- Rows for standard deduction (`#bStdDed`, dynamic), gross income, taxable OI, headroom, investment income tiers, NIIT, total.
- `#bOITiers` container: dynamically rendered tier rows (`$X @ Y% → $Z`) from `getOIBracketTiers()`.
  Hidden when OI = 0.

---

## Helper Functions

```
getMarginalRate(taxableOI)      → rate (0.10–0.37) or null if taxableOI = 0
getOIBracketTiers(taxableOI)    → [{floor, amount, rate, tax}, ...]  only filled tiers
calcOrdinaryTax(taxableOI)      → total ordinary tax (number)
calcTax(oi, invest)             → full result object (see below); invest = QD + LTCG combined
fmt(n)                          → "$1,234" or "−$3,000" for negatives
```

### calcTax() return object

```javascript
{
  (gross,
    totalTaxable,
    taxableOI,
    ordTax,
    zeroHeadroom,
    investTaxable,
    inv_at0,
    inv_at15,
    inv_at20,
    tax15,
    tax20,
    niitBase,
    niitTax,
    total);
}
```

---

## Design Decisions

- **Dark theme only.** CSS variables in `:root`. Background `#0d1117`, surface `#161b22`.
- **No browser storage, no external APIs, no build tooling.** Fully self-contained.
- **Deduction not shown in bracket bar.** The bar represents taxable income only; deduction is
  noted as plain text in the panel subtitle. This avoids a large inert blue chunk eating visual space.
- **Unused 0% space is rendered** as a faint ghost segment in the bracket bar so the planning
  headroom is visually obvious.
- **QD and LTCG are combined into one Investment Income slider** because they're taxed identically.
  The slider represents total preferential-rate income; range −$3k to $600k, step $100.
- **Negative CG display** uses a proper Unicode minus sign (`−`) not a hyphen.
- **fmt() handles negatives** via `'−$' + Math.round(-n).toLocaleString()`.

---

## What This Tool Does NOT Model

- Alternative Minimum Tax (AMT)
- Self-employment tax
- State income taxes
- Social Security taxation phase-in (85% inclusion assumed — user should fold taxable SS into Ordinary Income)
- Capital loss carryforwards beyond the current year's $3,000 limit
- Medicare IRMAA surcharges
- Roth conversion interactions
- Depreciation recapture (Section 1250 / 25% rate)
