# Auditing my own notebook with two AI models

The final project of my OpenClassrooms Data Analyst training (Project 13) asked me to audit one of my earlier deliverables with AI: critically, with the method documented, and comparing several tools. I chose my Bottleneck notebook (Project 6, March 2026). This page summarises that audit. The cell numbers refer to the original French notebook.

## Set-up

| | Claude Opus 5 (high effort) | Gemini 3.1 Pro (extended thinking) |
| - | --------------------------- | ---------------------------------- |
| Role | Main tool: audit, technology watch, tests on the data | Control: an independent audit of the same notebook |
| Posture | Internal auditor, familiar with the project: the prompt said it had taken part in the original work | External auditor: the prompt presented the notebook as a student's work |
| Documents | The Project 13 brief, the Project 6 source files, presentation and notebook | The same |

Only the posture changed. The reason: an auditor reviewing work it took part in may be tempted to validate it. The independent audit was there to control that bias, and to test how self-critical Claude would be.

**Prompt given to Claude** (translated): *"For the last project of my Data Analyst training, we are going to revisit an earlier project you took part in, Project 6, an analysis of sales, margins, stock and correlations for the online wine shop Bottleneck. The results were then presented at the defence. The goal is a complete audit of this project, to find areas for improvement and fix any errors."*

**Prompt given to Gemini** (translated): *"You are a senior Data Analyst. Here is a Python notebook produced by a student (source files: an ERP export, a web export and a link table). Run a critical audit: list the weaknesses in method, the doubtful calculations and the reproducibility problems. Rank each finding by severity."*

## Gemini's audit: 10 findings

| Severity | Cell | Issue | Consequence |
| -------- | ---- | ----- | ----------- |
| High | 4 | Absolute path to load the files | The code cannot run on anyone else's computer |
| High | 60 | High sales removed on the imperfect duplicates with `.idxmax()` | Data falsification (confirmation bias), revenue understated |
| High | 49–50 | Arbitrary primary key: SKU 13127-1 renamed 19823 | Breaks the data lineage without business or ERP validation |
| High | 32 | Negative stocks overwritten with 0 | Hides possible logistics or IT anomalies (oversales, sync lags) |
| High | 23 | Negative prices turned into absolute values | Loses financial information: a negative price may be a credit note or a refund |
| Medium | 107 | Stock rotation extrapolated from one month × 12 | Ignores seasonality (year-end holidays for spirits), distorts the stock recommendations |
| Medium | 54–55 | Rows without a SKU but with negative sales removed | Revenue ignores possible returns or cancellations |
| Medium | 91–92 | Outlier test run on the whole catalogue (IQR) | False positives: a grand cru is not an anomaly next to a table wine |
| Low | 82 | Hard-coded IDs (product_id 4594, 5070, 5075 removed by hand) | Will fail or be wrong on next month's data |
| Low | 113 | Implicit, hard-coded VAT rate (`/1.20`) | Wrong if some products carry another rate |

## Claude's audit: 15 findings

| Severity | Cell | Issue | Consequence |
| -------- | ---- | ----- | ----------- |
| High | 107–108 | Coverage calculated as stock / (sales / 12) | Result multiplied by 12: 35.6 months presented instead of 3.0. The diagnosis wrongly flips from a sound stock to a critical overstock |
| High | 107–109 | Infinite coverages (no sale) replaced by 0 | Never-sold products ranked as the best performers: 25 products and €14,959 of stock disappear from the ranking |
| High | 51–55 | Rows removed without any check on negative sales | Two rows with negative sales (−56 and −17) vanish without a trace; on a future export, corrupted rows would be dropped without an alert |
| Medium | 113–115 | A product presented as "sold at a loss" (€12.65 for a €77.48 purchase price) | Probably an entry error presented as a commercial decision: the catalogue sells at about 1.9 times the purchase price, this product at 0.16. €126.50 would bring it back in line |
| Medium | 119–121 | Pearson correlation on very skewed distributions | Relationships understated: price and sales −0.52 instead of −0.71 (Spearman), stock and sales 0.44 instead of 0.76 |
| Medium | 96–113 | Revenue including VAT, margin rate excluding VAT | Indicators that cannot be compared: €23,933 of VAT separates the two bases on October's revenue |
| Medium | 58–60 | Imperfect duplicates settled by hand, keeping the lower value against the median | An unwritten rule that cannot be replayed on a new export |
| Medium | 50, 60, 73, 82 | Corrections hard-coded on IDs (SKU 1366, 14561, 14950, 15346, product_id 4594, 5070, 5075, 13127-1 renamed 19823) | The notebook only works on the October export |
| Medium | 116–118 | Average margin by category not weighted by volume | A product sold twice weighs as much as one sold thirty times |
| Medium | 4 | Absolute path, no dependency file | The notebook cannot run on another machine |
| Medium | 18–21, 26–29, 35–37 | Quality checks are plain printouts, with no threshold and no stop | A degraded export can pass unnoticed, while my own conclusion recommended automating the cleaning |
| Medium | 101–106 | 80/20 analysis stops at the finding (60% of products make 80% of revenue) | No prioritisation, no list of actions |
| Low | 94 | A slice of a dataframe modified (`SettingWithCopyWarning`) | Result not guaranteed by pandas |
| Low | Intro | The OpenClassrooms template text left at the top | Not presentable to a client |
| Low | 122 | Final export as a raw Excel file in the working folder | No summary the business can use |

## Comparing the two audits

| | Value |
| - | ----- |
| Claude's findings | 15 |
| Gemini's findings | 10 |
| Distinct findings | 19 |
| Findings in common | 6 |

**The six common findings**, and the severity each model gave them:

| Finding | Claude | Gemini |
| ------- | ------ | ------ |
| Absolute path (cell 4) | Medium | High |
| Imperfect duplicates settled by hand (58–60) | Medium | High |
| Hard-coded IDs (50, 60, 73, 82) | Medium | Low |
| SKU 13127-1 renamed (50) | Medium | High |
| Implicit VAT rate (113) | Medium | Low |
| Rows without a SKU and negative sales (51–55) | High | Medium |

The models agree on several corrections but not on a single severity. Claude kept "high" for what distorts a result presented to Bottleneck; Gemini gave it to any flaw in data integrity or code engineering.

**Where they read the same cell differently:**

| Type | Finding | Claude | Gemini |
| ---- | ------- | ------ | ------ |
| Interpretation | Negative prices made positive (23) | A legitimate fix of a sign error | Loss of information: possible credit notes or refunds |
| Interpretation | Imperfect duplicates (60) | An unwritten rule | Data falsification, confirmation bias |
| Interpretation | Coverage formula (107) | A calculation error, result multiplied by 12 | An extrapolation that ignores seasonality |
| Severity | The 6 common findings | Scale based on the impact on the client | Scale based on technical integrity |

**Blind spots.** Each model saw what the other missed, but the misses did not weigh the same. Gemini found neither of the two errors that distorted figures presented to the client (coverage multiplied by 12, never-sold products ranked as the best). Claude missed two real quality signals that changed no presented result: the negative stocks set to 0, and the outlier test run across categories.

## Cross-reviews, checked on the data

### Claude on Gemini's audit

| Gemini's finding | Claude's position | What the data says |
| ---------------- | ----------------- | ------------------ |
| Absolute path (high) | Partly shared | Agreement on the substance, not on the severity |
| Imperfect duplicates, "falsification" (high) | Not shared | Each product has two rows in the export, its product page and its image page. The 4 high values all sit on the image rows, the low ones on the product pages: removing the high value kept the product page, the right choice. The high values are far outside the sales distribution, and each looks like the low value with one or two extra digits typed in front. Keeping them would have inflated revenue by €10,068 including VAT |
| SKU renamed (high) | Partly shared | A useless renaming, but applied the same way to both tables (cells 50 and 73): the join was correct and product_id 7247 was kept |
| Negative stocks set to 0 (high) | Partly shared, missing from Claude's first audit | A negative stock signals an oversale or a sync lag; overwriting it deletes that information. Added to the audit, with a lower severity: 2 products (−10 and −1), both already out of stock, and no indicator presented to Bottleneck affected |
| Negative prices made positive (high) | Not shared, the fix was judged legitimate | If these prices have a mistyped sign, their absolute value must follow the catalogue's usual ratio (1.935). Their ratios are 1.94, 1.84 and 1.97: sign errors, not credit notes or refunds |
| Seasonality ignored (medium) | Partly shared | Kept, but the main problem is the calculation error: the sales are already monthly |
| Negative sales without a SKU (medium) | Partly shared | Found independently by both audits, with different severities. Claude rated it high because the mechanism would silently drop any number of rows on a future export. Claude did not share Gemini's reasoning: nothing confirms these are returns, and adding them to revenue would create false data. They must be reported and counted, not added back |
| Unsegmented IQR (medium) | Partly shared, missing from Claude's first audit | Right diagnosis (36 alerts on the 825 ERP prices, no real anomaly), but segmentation is impossible: 4 of the 6 categories have 2 to 14 products. The price ratio finds the 4 anomalies without segmenting |
| Hard-coded IDs (low) | Partly shared | Agreement on the main obstacle to monthly reuse, not on the severity |
| Implicit VAT (low) | Partly shared | Agreement, but the main inconsistency is mixing amounts including and excluding VAT: €23,933 of VAT on October's revenue |

The two findings Claude rejected are the two Gemini rated high, and the data contradicts both. Gemini's severities are also inconsistent with each other: absolute paths and hard-coded IDs are both reproducibility problems, yet Gemini rated the first high and the second low. The second is the more dangerous one: an absolute path fails at once and visibly, while a hard-coded ID lets the code run without an alert and produce a wrong result. In its revised audit, Gemini merged the two under a single medium severity.

**Result: no finding fully shared, 8 partly shared, 2 not shared.**

### Gemini on Claude's audit

| Claude's finding | Gemini's position | Gemini's argument | What the data says |
| ---------------- | ----------------- | ----------------- | ------------------ |
| Coverage multiplied by 12 (high) | Full agreement | "The overstock diagnosis is mathematically wrong" | Confirmed: 35.6 months presented, 3.0 in reality |
| Never-sold products ranked as the best (high) | Full agreement, missing from its own audit | "It artificially puts them at the top of the fast-moving products" | Confirmed: 25 products, €14,959 at purchase price |
| Negative sales without a SKU (high) | Not addressed | | Gemini had raised this finding in its own audit, but did not take a position on Claude's severity |
| Product wrongly "sold at a loss" (medium) | Agreement, missing from its own audit | "Shows excellent business sense" | Confirmed by the purchase / selling price ratio |
| Revenue incl. VAT, margins excl. VAT (medium) | Agreement, partly present in its own audit under the implicit VAT | "Presenting them together makes no business sense" | Confirmed: €23,933 of VAT |
| Unweighted margins (medium) | Full agreement, missing from its own audit | "A classic aggregation error" | **Disproved**: computed on totals (total margin over total purchase cost), the weighted margin never moves more than 1.3 points from the simple average across the six categories, and the ranking of categories does not change. Claude had already downgraded it after checking; Gemini approved it without recalculating. A cross-review without verification can validate an error |
| Imperfect duplicates (medium) | Raised to high | "Forcing the data to fit the median is an unacceptable confirmation bias" | The high values sit on the image rows: the notebook's choice was right |
| Absolute paths (medium) | Disagreement, kept at high | "Code that crashes in production is a blocking failure" | No result is affected, and Gemini itself came back to medium in its revised audit |
| Hard-coded IDs | Rated low | No position | Gemini rated it low, while it is the same reproducibility problem as the absolute paths it rated high |
| Pearson correlation (medium) | No position | Not addressed | |
| Non-blocking quality checks (medium) | No position | Not addressed | |
| 80/20 without recommendations (medium) | No position | Not addressed | |
| pandas warning (low) | No position | Not addressed | |
| OpenClassrooms template text (low) | No position | Not addressed | |
| Raw final export (low) | No position | Not addressed | |

**Result: 5 agreements, 2 disagreements on severity, 8 findings with no position.** Of the 5 findings Gemini approved, 3 were entirely missing from its first audit and 2 only partly present, and one of them (unweighted margins) turned out to be wrong. None of these is a change of mind: they are blind spots filled by the cross-review. Gemini kept two positions the data contradicts, without a new argument.

Each model filled the other's blind spots, but neither was reliable without checking the data.

## Revised audits

After reading Claude's cross-review, Gemini produced a revised audit. It dropped three findings it was still defending in its cross-review (the negative prices, the duplicates and the SKU renaming) without saying so or explaining why. It aligned its severities on Claude's: the absolute paths went from high to medium, the negative stocks from high to low, citing the two products already out of stock. It added the four Claude findings it had approved. But it lost its own contributions, the seasonality and the unsegmented detection, although both diagnoses had been judged right. The second opinion no longer brought a view of its own.

Claude's revision added the negative-stock finding from Gemini, dropped the unweighted-margins finding after checking it, and kept two positions that contradict Gemini, each backed by a calculation.

The two models converge more after revision, but mostly because Gemini aligned on Claude's positions, not because of a shared verification. **Two models that agree can agree for the wrong reasons.**

## A last check on the data

When I prepared this repository, one more look at the data refined a finding both audits shared: of the 25 never-sold products, **22 have no stock at all**. They are not dormant stock but products that could not sell. The €14,959 is held by the 3 others. With the original formula, those 3 got a coverage of 0 and were ranked as the fastest-moving products, while the 22 without stock gave 0 divided by 0, an empty value that dropped out of the average without a warning. The rebuilt notebook now separates the two groups, and the action list asks a different question for each.

The same check corrected one of my own conclusions. My first version deleted 3 products by ID (4594, 5070, 5075), and I first thought the rebuild made that problem disappear. It does not: these 3 products are declared online in the ERP but have no website ID in the link table. They are a real mismatch, now reported as an alert. And the €25 gift card, which the first version deleted, is kept as a product in the rebuilt notebook: 7 sold, €146 of revenue excluding VAT.
