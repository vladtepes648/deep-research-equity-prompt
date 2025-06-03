# Deep Research Equity Prompt (v 4.0)

## Purpose

Provide a totally self-contained specification so an LLM can create a sell-side-style equity research report in strict JSON. Nothing is referenced externally; every rule, table, and example is written out below.

---

# 1 Parameters

## 1.1 Required

```yaml
# --- USER MUST FILL THESE ---
COMPANY_NAME : "Alphabet Inc."      # e.g., "Apple Inc."
TICKER       : "GOOGL"              # e.g., "AAPL"
EXCHANGE     : "NASDAQ"             # e.g., "NASDAQ"

# --- Dates define the analysis window ---
START_DATE   : "2024-05-29"         # (YYYY-MM-DD)
END_DATE     : "2025-05-29"         # (YYYY-MM-DD)

# --- Reporting Currency ---
CURRENCY     : "USD"                # e.g., USD, EUR, GBP
````

## 1.2 Optional

```yaml
PEER_LIST            : "MSFT, AAPL, AMZN, META, NVDA"
SECTOR               : "Communication Services"    # e.g., "Information Technology"
STRATEGY             : "default"                   # e.g., high_growth, income
MARKET_PHASE         : "normal"                    # normal | recession | boom
BENCHMARK_INDEX      : "NDX"                       # e.g., SPX, DAX, IXIC
TARGET_HORIZON_MONTHS: 12
FX_SOURCE            : "ECB"                       # e.g., FRED, ECB
PRICE_FREQ           : "daily"                     # daily | weekly
NEWS_LOOKBACK_DAYS   : 365
TOP_N_NEWS           : 50
DETAIL_LEVEL         : "detailed"                  # detailed | compact
MAX_TOKENS_RESPONSE  : 3500
```

## 1.3 Quick Parameter Guide

| Field                   | Values / Format                                        | Why it matters                        |
| ----------------------- | ------------------------------------------------------ | ------------------------------------- |
| START\_DATE / END\_DATE | YYYY MM DD                                             | Data window for analysis              |
| PEER\_LIST              | List of tickers                                        | Peer comps & valuation                |
| SECTOR                  | GICS sector (or mappable general industry)             | Infers default STRATEGY, Peer ID      |
| STRATEGY                | default, high\_growth, income, value, high\_volatility | Adjusts module weights                |
| MARKET\_PHASE           | normal, recession, boom                                | Risk & sentiment emphasis             |
| BENCHMARK\_INDEX        | Index ticker                                           | Beta reference                        |
| FX\_SOURCE              | e.g., FRED, ECB                                        | Specifies provider for currency rates |
| PRICE\_FREQ             | daily / weekly                                         | Influences historical price analysis  |
| NEWS\_LOOKBACK\_DAYS    | ≥1                                                     | Sentiment horizon                     |
| TOP\_N\_NEWS            | 1–100                                                  | Token usage limiter                   |
| MAX\_TOKENS\_RESPONSE   | ≥1000                                                  | Prevents truncation                   |

---

# 2 System Instructions

## 2.1 Role & Language

Act as a sell-side analyst AI. Output **American English only**.

## 2.2 Data Window

Analyze data between `START_DATE` and `END_DATE` only.
Earlier history may be used solely for derived metrics (YoY, moving averages, etc.).

## 2.3 Source Strategy & Data Reliability

The Tool is expected to access and use the most current available dynamic data (prices, news, consensus) relevant to the `END_DATE` for its analysis.

**Strict date adherence:** All inputs must reflect information available as of `END_DATE`.

**Most recent official data first:** Verify dividends, M\&A, buybacks, capital structure vs the latest filings/press releases.

**Source Language and Document Processing:**
The AI MUST attempt to process official company disclosures (annual/quarterly reports, filings) for primary financial data, even if these documents are not in English, using its internal translation capabilities.

**Priority of Sources:**

1. Audited official company disclosures (English versions preferred if available and equally comprehensive; otherwise, other languages with translation are primary).
2. Unaudited official company communications (e.g., press releases, investor presentations; English preferred, then other languages with translation).
3. Reputable third-party financial data aggregators, only if primary sources (1 & 2) are insufficient, inaccessible, or translation yields unreliable results after reasonable attempts.

**Transparency Requirements:**

* If data is derived from a translated non-English primary source, the original language MUST be stated (e.g., in KPI notes or the `sources` array).
* Significant reliance on translated non-English primary sources, or use of unaudited data, MUST be noted as a factor in `uncertainty_assessment_details` (e.g., under `driver_category: "Data Quality"`).
* Always specify if data is from an unaudited source.
* Source differentiation: Quant KPIs from structured portals; qualitative summaries from recent news, earnings calls, management commentary.

**Primary Data Sources:**
List primary data sources (e.g., financial data providers, specific company filings if directly referenced for unique key data, major news outlets for sentiment drivers) in the `sources[]` array at the end of the report. It is not required to cite every individual fact. For each source entry, note if it is generally known to be behind a paywall (e.g., by setting `restricted = true`).

**Data Availability for KPIs:**
If critical data for a specific KPI cannot be found from any reputable source after a reasonable search, the KPI's value in `module_details.kpis` should be set to `null`, and a `note` field within that KPI object must explain the missing data:

```json
{ "metric": "PE_Ratio_Fwd", "value": null, "unit": "x", "note": "Forward consensus estimates unavailable" }
```

The AI should still attempt to complete the rest of the report.
Significant data gaps should also be mentioned in the relevant module's summary.

## 2.4 Preferred Data Sources

* Annual/Quarterly Reports, 8-K/ad hoc filings, Investor Relations sites, SEC (via EDGAR database)/ESMA/BaFin.
* Reuters, Bloomberg Free, Yahoo Finance, Google Finance, TradingView, MacroTrends, Finviz.
* Moody’s, S\&P, Fitch public summaries.
* General news outlets & vetted finance blogs.

```

```markdown
## 2.5 Valuation Methodology

Choose one primary method:

1. **Peer Comparables** (default).  
   If `PEER_LIST` is empty or null, the AI should attempt to identify a relevant peer group of up to 10 companies based on the target company's profile.  
   If `SECTOR` is provided, it should be strongly considered. If `SECTOR` is not provided, the AI should attempt to infer the GICS sector and use that, or use other methods like business description similarity.  
   If fewer than 3 peers can be reliably identified, state this and skip peer-related components like `Peer_Comparison_Summary`.

**Criteria for reliable identification:**

- Operate in a similar GICS sub-industry or comparable business model.
- Publicly traded with readily available financials covering at least the last two fiscal years.
- Sufficient market data exists (market cap, stock price, consensus estimates if available).
- The AI can access and process their key financial data and valuation multiples.

2. **DCF** — only if cash flows are reasonably stable and predictable.

**Suitability indicators:**

- History of positive FCF generation (≥3 of last 5 years).
- FCF not excessively volatile relative to revenue or assets.
- Discernible business model for forecasting (not major turnaround or highly speculative).
- Reliable forward guidance or analyst consensus for key drivers.

3. **DDM** — only for mature companies with consistent dividends.

**Suitability indicators:**

- Uninterrupted dividends for 5–7 consecutive years.
- Stated dividend policy or clear pattern of commitment.
- Sustainable payout ratio (unless high-payout industry like Utilities/REITs).

**Justify the choice** in `valuation_details.methodology_justification` (3–4 sentences).  
Blend only if each method stands alone and yields complementary insight. Base-case PT must match the Base scenario (Appendix B).

## 2.6 Module Scoring Logic

Score **A–H** on **0–10**.  
Drivers: peer relative, historical trend, absolute level, strategy fit, qualitative context.

**Normalize KPIs** (z-scores or min-max against peers or historical range) when required. Typically necessary for:

- Comparing growth/profitability/efficiency metrics across peers.
- Trend analysis of a company's own performance.
- Valuation ratios (`P/E_Ratio_LTM`, `EV_EBITDA_LTM`, etc.) are compared as absolute values against peers and history.

## 2.7 Weight Calibration

1. Start with baseline weights (as defined in § 4.1).
2. Apply `STRATEGY` deltas (as defined in § 4.2).
3. Clamp any resulting module weights to 0 if they have become negative.
4. Re-normalize the final set of active module weights so that they sum to 100.

## 2.8 Overall Score & Rating

```

overall\_score = Σ(module\_score × weight) / Σ(weights of scored modules)

````

| Rating       | Score |
|--------------|-------|
| Strong Buy   | > 8.0 |
| Buy          | 6.5 – 8.0 |
| Hold         | 4.5 – 6.4 |
| Sell         | 3.0 – 4.4 |
| Strong Sell  | < 3.0 |

## 2.9 Uncertainty Assessment

Assess uncertainty as: **Very Low** | **Low** | **Medium** | **High** | **Very High**

Based on:

- Data Availability & Quality
- Market Phase & Volatility
- Scenario Spread
- Company-Specific Factors

If uncertainty ≥ Medium, the disclaimer (or `cautionary_note`) must advise extra caution.

## 2.10 Internal Consistency Checks

Before emitting JSON, strictly ensure:

- Weights sum = 100
- Math matches
- No `NaN`
- News not empty unless justified
- All arrays and objects are correctly formatted and populated according to schema requirements.

**On failure:** Output structured error JSON.

---

# 3 Output Schemas

## 3.0 Central ENUMs

```text
STRATEGY      default | high_growth | income | value | high_volatility
MARKET_PHASE  normal | recession | boom
impact        positive | neutral | negative | mixed
uncertainty   Very Low | Low | Medium | High | Very High
````

## 3.1 COMPACT Schema (`DETAIL_LEVEL = "compact"`)

```json
{
 "schema_version": "4.0",
 "$schema": "$schema": "TO_BE_DEFINED",
 "meta": {
   "company": "string",
   "ticker": "string",
   "analysis_period": {"start": "YYYY-MM-DD", "end": "YYYY-MM-DD"},
   "report_date": "YYYY-MM-DD",
   "module_weights": {"A":0.0,"B":0.0,"C":0.0,"D":0.0,"E":0.0,"F":0.0,"G":0.0,"H":0.0},
   "self_check_passed": true
 },
 "executive_summary": {
   "bullets": ["string"],
   "rating": "Strong Buy|Buy|Hold|Sell|Strong Sell",
   "price_target": {"value": 0.0, "currency": "string", "horizon_months": 0},
   "uncertainty_rating": "Very Low|Low|Medium|High|Very High",
   "key_uncertainty_factors_summary": "string|null",
   "cautionary_note": "string|null"
 },
 "module_scores": {"A":0.0,"B":0.0,"C":0.0,"D":0.0,"E":0.0,"F":0.0,"G":0.0,"H":0.0},
 "overall_score": 0.0,
 "disclaimer": "string"
}
```

---

```

```markdown
## 3.2 DETAILED Schema (`DETAIL_LEVEL = "detailed"`)

```json
{
 "schema_version": "3.9",
 "$schema": "https://github.com/your-org/dre-3.9-detailed.schema.json",
 "meta": { /* identical to compact */ },
 "executive_summary": {
   "bullets": ["string"],
   "rating": "Strong Buy|Buy|Hold|Sell|Strong Sell",
   "price_target": {"value": 0.0, "currency": "string", "horizon_months": 0},
   "cautionary_note": "string|null"
 },
 "valuation_details": {
   "primary_method_used": "Peer Comparables|DCF|DDM|Blend",
   "methodology_justification": "string",
   "key_assumptions": ["string"]
 },
 "uncertainty_assessment_details": {
   "overall_uncertainty_rating": "Very Low|Low|Medium|High|Very High",
   "key_drivers_of_uncertainty": [
     {
       "driver_category": "Data Availability|Data Quality|Market Conditions|Company Specific|Model Limitations",
       "description": "string",
       "impacted_modules_or_outputs": ["string"]
     }
   ],
   "mitigation_or_consideration": "string|null"
 },
 "qualitative_analysis": {
   "summary": "string",
   "news": [
     {
       "headline": "string",
       "date": "YYYY-MM-DD",
       "impact": "positive|neutral|negative|mixed",
       "summary": "string",
       "cluster": "string",
       "cluster_method": "embedding|heuristic",
       "cluster_rationale": "string|null"
     }
   ]
 },
 "module_details": [
   {
     "id": "A"|"B"|"C"|"D"|"E"|"F"|"G"|"H",
     "name": "string",
     "data_date": "YYYY-MM-DD"|null,
     "kpis": [
       {
         "metric": "string",
         "value": 0.0|"string"|{},
         "unit": "string"|null,
         "period": "string",
         "comparison": {
           "peer_median": 0.0|null,
           "historical_avg": 0.0|null
         },
         "definition": "string",
         "note": "string|null"
       }
     ],
     "score": 0.0|null,
     "summary": "string",
     "partial_history": true|false|null
   }
 ],
 "scenario_analysis": {
   "optimistic": {
     "assumptions": "string",
     "price_target": {
       "value": 0.0,
       "currency": "string",
       "horizon_months": 0
     }
   },
   "base": {
     "assumptions": "string",
     "price_target": {
       "value": 0.0,
       "currency": "string",
       "horizon_months": 0
     }
   },
   "pessimistic": {
     "assumptions": "string",
     "price_target": {
       "value": 0.0,
       "currency": "string",
       "horizon_months": 0
     }
   }
 },
 "sensitivity": [
   {
     "metric": "string",
     "shock_applied": "string",
     "qualitative_impact_on_overall_score": "High|Medium|Low|Negligible",
     "qualitative_impact_on_price_target": "High|Medium|Low|Negligible",
     "justification_of_impact": "string"
   }
 ],
 "sources": [
   {
     "title": "string",
     "date": "YYYY-MM-DD"|null,
     "url": "string"|null,
     "restricted": false|true
   }
 ],
 "disclaimer": "string"
}
```

---

# 4 Module Weights & Presets

## 4.1 Baseline Weights

| Module | A  | B  | C  | D  | E  | F | G | H | Total |
| ------ | -- | -- | -- | -- | -- | - | - | - | ----- |
| Weight | 35 | 15 | 15 | 10 | 10 | 5 | 5 | 5 | 100   |

## 4.2 Strategy-Based Weight Deltas

| STRATEGY         | ΔA | ΔB | ΔC | ΔD | ΔE | ΔF | ΔG | ΔH | Rationale         |
| ---------------- | -- | -- | -- | -- | -- | -- | -- | -- | ----------------- |
| high\_growth     | 0  | 0  | 0  | -5 | 0  | 0  | 0  | +5 | Emphasize R\&D    |
| income           | 0  | 0  | 0  | +5 | 0  | -2 | 0  | 0  | Dividend focus    |
| value            | 0  | 0  | +5 | -5 | -5 | 0  | 0  | 0  | Valuation metrics |
| high\_volatility | 0  | 0  | 0  | 0  | +5 | +5 | 0  | 0  | Risk & sentiment  |

*After applying deltas: clamp negatives → re-normalize to 100.*

---

```

```markdown
# 5 Appendix A — FULL KPI Dictionary

(Below are all KPI definitions for Modules A–H. `{CURRENCY}` refers to the reporting currency specified in Parameters. Percentages are decimals (e.g., 0.15 = 15%). Each KPI should include a note if data is missing or needs explanation (§ 2.3).)

---

## Module A — Financials & Operating Performance

| KPI              | Definition                          | Unit        | Typical Period |
|------------------|-------------------------------------|-------------|----------------|
| Revenue          | Total top-line revenue               | {CURRENCY}  | FY, Q          |
| Revenue_Growth   | Year-over-year % change in revenue   | %           | FY, Q          |
| EBITDA_Margin    | EBITDA / Revenue                     | %           | FY, Q          |
| EPS              | Diluted EPS (adjusted if disclosed)  | {CURRENCY}  | FY, Q          |
| Free_Cash_Flow   | CFO – CapEx                          | {CURRENCY}  | FY, Q          |
| Segment_Revenue  | Revenue by reported segments         | {CURRENCY}, %| FY            |

---

## Module B — Leverage & Cash Flow Quality

| KPI              | Definition                          | Unit |
|------------------|-------------------------------------|------|
| NetDebt_EBITDA   | Net Debt / EBITDA                    | x    |
| Debt_Equity      | Total Debt / Shareholders’ Equity    | x    |
| FCF_Margin       | Free Cash Flow / Revenue             | %    |
| Share_Buybacks   | Value of shares repurchased          | {CURRENCY} |

---

## Module C — Valuation

| KPI              | Definition                          | Unit |
|------------------|-------------------------------------|------|
| PE_Ratio_LTM     | Price / LTM EPS                      | x    |
| PE_Ratio_Fwd     | Price / Fwd 12-mo EPS (consensus)    | x    |
| EV_EBITDA_LTM    | EV / LTM EBITDA                      | x    |
| EV_EBITDA_Fwd    | EV / Fwd EBITDA (consensus)          | x    |
| PB_Ratio         | Price / Book Value per share         | x    |
| PEG_Ratio        | PE Ratio / EPS growth rate           | x    |

---

## Module D — Dividend Policy

| KPI                   | Definition                                                          | Unit  |
|-----------------------|---------------------------------------------------------------------|-------|
| Dividend_Yield        | Annual DPS / current price                                           | %     |
| Payout_Ratio          | Annualised DPS / Annualised Net Income per share (LTM). <br> If no dividend: 0.0. <br> If NI = 0: null (note "Undefined"). <br> If NI < 0: null (note "N/A"). <br> Consider FCF-based ratio. | %     |
| Dividend_Stability    | Consecutive years without a cut                                      | years |
| Dividend_Growth_Rate  | CAGR of dividends (3–5 yrs)                                          | %     |

---

## Module E — Market Sentiment & Insider Activity

| KPI                   | Definition                                                          | Unit  |
|-----------------------|---------------------------------------------------------------------|-------|
| Insider_Trading       | Net insider open-market purchases – sales (last 6 m). <br> Qualitative signal should accompany value. | {CURRENCY} |
| Short_Interest        | Shares sold short / free float                                       | %     |
| Analyst_Rating_Avg    | Mean consensus rating. Include scale source/description notes.       | numeric |
| News_Sentiment_Score  | Aggregated news sentiment score (-1 to +1) & label.                  | score & label |

---

## Module F — Market Risk & Volatility

| KPI            | Definition                          | Unit |
|----------------|-------------------------------------|------|
| Beta_30d       | Beta vs {BENCHMARK_INDEX} (30 daily returns) | x    |
| Volatility_30d | Annualised stdev of daily returns (30 d)    | %    |

---

## Module G — Macro / Industry Context

| KPI                   | Definition                              | Unit |
|-----------------------|-----------------------------------------|------|
| Inflation_Sensitivity | Qualitative/quantitative EBIT margin impact. | % or qualitative |
| Rate_Sensitivity      | Qualitative/quantitative EBIT margin impact. | % or qualitative |
| Regulatory_Risk       | Qualitative risk level (Low/Medium/High).   | text |

---

## Module H — Extended Metrics & ESG

| KPI                   | Definition                              | Unit |
|-----------------------|-----------------------------------------|------|
| Capex_Ratio           | CapEx / Revenue                          | %    |
| Effective_Tax_Rate    | Income Tax / EBT                         | %    |
| R&D_Ratio             | R&D / Revenue                            | %    |
| ESG_Score             | ESG rating or summary per § 2.3          | varies |
| Peer_Comparison_Summary | Textual ranking vs peers                | text  |

---

# 6 Appendix B — Scenario & Sensitivity Definitions

## 6.1 Scenario Table

Default: use midpoint of ranges unless justified otherwise.

| Scenario    | Revenue CAGR vs Base | EBITDA/Margin Shift | Typical Context                            |
|-------------|---------------------|--------------------|--------------------------------------------|
| Optimistic  | +5 pp to +10 pp      | +2 pp to +5 pp     | Strong demand, flawless execution          |
| Base        | best estimate        | best estimate      | Continuation of current trends             |
| Pessimistic | –5 pp to –15 pp      | –2 pp to –5 pp     | Recession, competitive pressure            |

---

## 6.2 Sensitivity Analysis Rules

Qualitatively assess impact (**High / Medium / Low / Negligible**)  
on `overall_score` & `price_target` for each shock.

| Driver                    | Shock Applied                |
|---------------------------|------------------------------|
| Revenue Growth Rate       | ±1 pp                        |
| EBITDA / Operating Margin | ±100 bps                     |
| WACC (if DCF used)        | ±50 bps                      |
| Terminal Growth Rate / Exit Multiple | ±0.25 pp / ±0.5×    |
| Key Peer Multiple         | ±1.0×                        |

---

# 7 Appendix C — Mini JSON Example (≤50 lines)

```json
{
 "schema_version": "3.9",
 "meta": {
   "company": "ASML Holding N.V.",
   "ticker": "ASML",
   "analysis_period": {"start": "2024-05-01", "end": "2025-05-21"},
   "report_date": "2025-05-22",
   "module_weights": {"A":35,"B":15,"C":15,"D":10,"E":10,"F":5,"G":5,"H":5},
   "self_check_passed": true
 },
 "executive_summary": {
   "bullets": ["EUV demand remains robust; backlog >2 years"],
   "rating": "Buy",
   "price_target": {"value": 1060, "currency": "EUR", "horizon_months": 12},
   "cautionary_note": "Medium uncertainty due to geopolitical factors impacting supply chain; conclusions require careful consideration."
 },
 "uncertainty_assessment_details": {
   "overall_uncertainty_rating": "Medium",
   "key_drivers_of_uncertainty": [
     {
       "driver_category": "Market Conditions",
       "description": "Geopolitical tensions affecting semiconductor supply chain.",
       "impacted_modules_or_outputs": ["G", "Price Target"]
     }
   ],
   "mitigation_or_consideration": "Scenarios account for a range of supply impacts."
 },
 "module_scores": {"A":8.5,"B":7.0,"C":6.8,"D":3.0,"E":6.5,"F":4.5,"G":6.0,"H":7.5},
 "overall_score": 6.90
}
````

---

# 8 License / Disclaimer

{{DISCLAIMER}}
This report is non-independent research. It is not personalised investment advice and does not replace professional guidance. Financial models involve assumptions and are subject to uncertainty; actual results may differ materially. Past performance is not indicative of future results. If uncertainty is assessed as Medium, High, or Very High, the conclusions herein should be treated with increased caution due to specific factors outlined in the report.

---

# Author

[GitHub: vladtepes648](https://github.com/vladtepes648)

---

```
