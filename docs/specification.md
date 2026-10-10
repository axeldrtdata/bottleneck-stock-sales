# Specification: a monthly pipeline that checks before it counts

Written after the AI audit (see [`ai-audit.md`](ai-audit.md)), before rebuilding the notebook. The figures were updated after a last check on the data, described at the end of the audit page.

## Goal

Turn the three raw exports into reproducible financial indicators, the same from one month to the next, that **stop when the incoming data is degraded** and end in a list of business actions. An alert and a stop are better than a wrong result produced automatically.

| Stakeholder | Decides | Needs |
| ----------- | ------- | ----- |
| Purchasing and stock manager | Order, clear or drop a product | A prioritised list, with the amounts at stake |
| Management | Where the margin is made, and how much capital sleeps | Financial indicators that are stable from one month to the next |
| E-commerce manager | Whether the online catalogue is consistent | Price anomalies and ERP / website mismatches |

## Scope

- **Input**: the three exports of one month, 825 products, 713 of them sold online.
- **In scope**: quality control, stock indicators, margins, price anomalies and the segmentation of the catalogue.
- **Out of scope**: customer data (GDPR), sales forecasting, and fixing the duplicates inside the ERP itself.

## Constraints

- A single month of exports: no seasonality, no XYZ classification.
- VAT is not in the files: 20% is assumed.
- Each product appears twice in the website export, so a filter must run before any uniqueness check.
- Fewer than 2,000 rows per file: no need for performance tools.

## Assumptions

| Assumption | Basis | Risk if wrong |
| ---------- | ----- | ------------- |
| `total_sales` covers October only | The client's brief: sales run from 1 to 31 October | Coverage would need another calculation |
| VAT at 20% | The standard French rate on wine and spirits, absent from the files | Margins shifted, but fixable in one place |
| Negative prices are sign errors | Ratios of 1.94, 1.84 and 1.97 once the sign is removed, against 1.93 for the catalogue | Limited: the 3 products are not sold online, so they are outside every indicator presented |
| The product "sold at a loss" (€12.65) has a misplaced decimal | A ratio of 0.16 against 1.93; at €126.50 it becomes 1.63, close to the catalogue | A real price would be overstated |
| The purchase / selling price ratio is stable enough to serve as a reference | Median of 1.93; a standard deviation of at most 0.09 within any category; almost every product sits between 1.4 and 3 | A low-margin range would raise false alerts |

## Options compared

The audit raised needs; for each, several options were compared, then tested on the real data before choosing.

| Need | Options compared | Chosen | Why |
| ---- | ---------------- | ------ | --- |
| Measure stock coverage | 1. The original formula, stock / (sales / 12) · 2. Stock / monthly sales · 3. Stock × purchase price | 2, with 3 alongside | The sales are already monthly: dividing by 12 multiplied the result by 12 (35.6 months instead of 2.98). The stock value answers another question, how much money sleeps, and both are useful |
| Amounts including or excluding VAT | 1. All including VAT · 2. All excluding VAT | 2 | The first version mixed revenue including VAT and margins excluding VAT. Purchase prices are excluding VAT, so this convention wins |
| Correlation measure | 1. Pearson (the original matrix) · 2. Spearman | Spearman | Prices range from a few euros to €188 excl. VAT, with a median of €19.67. Pearson is pulled by the extremes and understates the relationships (−0.52 and 0.44 against −0.71 and 0.76) |
| Products with no sales | 1. Replace with 0 (the original) · 2. Exclude them · 3. Isolate them in their own category | 3, sorted by the value at stake | At 0 they look like the best of the catalogue; excluded, they disappear, while they are 25 products and €14,959 of stock |
| Automate the quality checks | 1. pandas assertions · 2. pandera · 3. Great Expectations | pandera | The 8 rules raised by the audit were written in both candidates, then run on a copy of the export with 4 anomalies injected on purpose. Both caught all 4. pandera guarantees the check runs and took 39 lines against 57. Great Expectations was set aside as oversized for 825 rows a month. Two rules were added after this test, on negative sales and on counting the rows rejected by the join, bringing the total to 10 |
| Detect price anomalies | 1. IQR · 2. Robust z-score (MAD) · 3. Isolation Forest, each tested on the price and on the purchase / selling price ratio | MAD on the ratio, threshold 5 | On the price, no method finds a real anomaly. On the ratio, all three find the 4 anomalies, but the IQR flags 45 products in all against 7 for the MAD. Isolation Forest only flags 5 when its contamination is set knowing the number of anomalies; on a future export where that number is unknown, it would always flag the same count |
| Move from findings to action | 1. ABC classification · 2. k-means · 3. Coverage × stock value matrix | The matrix, with business rules | Class A of an ABC would hold 60% of the catalogue. k-means gives a silhouette of 0.33 and mostly finds price ranges. The catalogue is too even for both |

## The ten rules

| Rule | Checks | Level |
| ---- | ------ | ----- |
| R1 | `product_id` (ERP) and `sku` (website) are unique | Blocking |
| R2 | Every price is filled in and positive, once explainable entry errors are fixed | Blocking |
| R3 | Every purchase price is above zero | Blocking |
| R4 | The selling / purchase price ratio stays within 5 robust z-scores of the catalogue | Alert |
| R5 | Stock is filled in; negative stock is reported, then set to zero for the indicators | Alert |
| R6 | `stock_status` only takes `instock` or `outofstock`, and matches the quantity | Blocking / alert |
| R7 | `onsale_web` only takes 0 or 1 | Blocking |
| R8 | At least 95% of the linked products are found in the website export | Blocking |
| R9 | Sales left after the filters are never negative (October's two negative sales had no SKU: they are reported as an alert and counted as rejected) | Blocking |
| R10 | Every rejected row is counted and reported | Report |

## Decisions

| Kept | Why |
| ---- | --- |
| pandera | The check cannot be forgotten, and the schema reads like a specification |
| Two levels: blocking and alert | Processing stops only if a file cannot be used. Occasional anomalies, such as mistyped prices, raise an alert without preventing the month's analysis |
| Fix by rule, not by ID | The notebook runs on the next export without being rewritten |
| MAD on the price ratio, threshold 5 | No need to know the anomaly rate in advance. The threshold sets the distance at which a price is abnormal, not the number of products to flag, so that number varies from month to month with the data |
| A separate "never sold" category | A product with no sales has no coverage: with no history, it cannot be calculated. It gets its own category, split into products with and without stock, sorted by the value at stake |
| Monthly coverage plus stock value | The first version only measured a duration, and it was wrong. Coverage says in how many months the stock will sell; stock value says how much money sleeps in it. Two products can have the same coverage and tie up very different sums: the amount sets the priority |
| Deduplication: keep the `post_type == "product"` row | The aberrant values are all on the image rows |

## Options set aside

| Set aside | Why | Reconsider if |
| --------- | --- | ------------- |
| Great Expectations | Too heavy for 825 rows a month | Bottleneck scales up, with automated exports, a higher frequency, and quality reports the business reads without the analyst |
| Isolation Forest | Needs the anomaly rate set in advance | Detection across several variables becomes necessary |
| k-means | Weak clusters; 3 of the 4 only rediscover price ranges | More history becomes available |
| ABC classification | Class A would hold 60% of the catalogue | Sales concentrate on a minority of products |
| Smoothed average | Impossible on one month | More months of exports are available |
| Detection by category | 4 of the 6 categories have between 2 and 14 products, too few to define what is normal within a group | The volumes become large enough |
| Weighted margins | Computed on totals, less than 1.3 points from the simple average across the 6 categories: right in theory, but it changes no decision | Volumes become very unbalanced within a category |
| Polars, DuckDB | No gain at this volume | The files grow to tens of thousands of rows |

## Acceptance criteria

| Criterion | Target | Status |
| --------- | ------ | ------ |
| The validation rules run on every load | 10 out of 10 | Met |
| A degraded export is rejected | The 4 anomalies injected into a copy of the export are caught | Met: section 3.7 of the notebook |
| No row rejected without a trace | 0 | Met |
| ERP / website match rate | Above 95% | Met: 97.3% |
| Every amount on the same basis | 100% excluding VAT | Met |
| Product IDs hard-coded | 0 | Met |
| Full run from a fresh kernel | No error, under 2 minutes | Met: about 11 seconds for the French notebook, about 26 seconds for this English version, which also exports its charts as images |

## Coverage results

| Coverage | Products | Stock value | Share |
| -------- | -------- | ----------- | ----- |
| Sold out | 23 | €0 | 0% |
| Up to 1 month | 27 | €2,501 | 0.9% |
| 1 to 3 months | 461 | €101,899 | 36.8% |
| 3 to 6 months | 146 | €43,517 | 15.7% |
| 6 to 12 months | 7 | €19,337 | 7.0% |
| Over 12 months | 24 | €95,012 | 34.3% |
| Never sold, with stock | 3 | €14,959 | 5.4% |
| Never sold, no stock | 22 | €0 | 0% |

27 products hold 39.7% of the stock value. The notebook outputs a list of 73 products that need a decision, sorted by the value at stake: suspend restocking (24), clear stock that never sold (3), check whether a product with neither sales nor stock is discontinued (22), or reorder (24). The 24 orders are the 23 sold-out products and one product covering less than a month of sales; the 26 other products in the "up to 1 month" band still hold the stock they need for the coming month. The 23 sold-out products sold 194 bottles in the month, €5,516 excl. VAT: until they are restocked, those are lost sales.

The coverage thresholds (1 month to reorder, 12 months for dormant stock) are conventions. In a real application, they would be set with the purchasing manager based on supplier lead times: with a one-month lead time, a product covering exactly one month should already be on order. And with no history, coverage assumes November will look like October. A product classed as dormant may simply be seasonal; a year of exports would let the sales be smoothed.

## User training

A session of under an hour for the purchasing and stock manager and the e-commerce manager.

| Module | Length | Content |
| ------ | ------ | ------- |
| What the tool guarantees, and what it does not | 10 min | It checks that the exports are consistent; it does not fix the ERP |
| Reading the indicators | 15 min | Coverage, stock value, margin excl. VAT, and why a product that never sold has no coverage |
| Reacting to an alert | 15 min | The families of alerts, and whom to contact |
| Limits | 5 min | Assumed VAT, a single month of exports, adjustable thresholds |

| If you see | It means | Action |
| ---------- | -------- | ------ |
| Coverage over 12 months | More than a year of stock | Suspend restocking |
| Coverage under 1 month | Risk of running out | Reorder |
| Never sold, with stock | No sales during the month | Check the product page, consider clearing the stock |
| Never sold, no stock | Nothing to sell | Check whether the product is discontinued |
| Sold out | The product sold all its stock | Reorder quickly: every day out of stock is lost sales |
| Price anomaly | A price far from the rest of the catalogue | Check the purchase / selling price ratio, then the entry in the ERP |
| Negative stock | Oversale or ERP / website lag | Check the sales concerned and the synchronisation |
| The processing stops | The export cannot be used | Send the error to the analyst, and never reuse last month's results instead |

## Project plan

| Step | Content | Time |
| ---- | ------- | ---- |
| Framing | Audits and cross-reviews, technology watch, this specification | 4 h |
| Technical base | Relative paths, libraries, reference values (VAT rate, MAD threshold of 5, 95% match rate) | 2 h |
| Quality control | The 10 rules | 3 h |
| Indicators | Corrections and definitions | 3 h |
| Analysis | Anomalies, matrix and action list | 2 h |
| Reporting | Narrative, limits and proofreading | 2 h |
| Documentation | A single logbook covering the breakdown, progress and decisions | 5 h |

| Risk | Probability | Impact | Response |
| ---- | ----------- | ------ | -------- |
| A badly calibrated threshold raises too many alerts | Medium | High | Threshold set on a stable plateau of the October data; review after the next month |
| The export format changes | Medium | High | Validation stops instead of producing a wrong result |
| The VAT assumption is wrong | Low | Medium | The rate sits in a single constant; add constants for special cases |
| pandera changes | Low | Medium | Versions pinned in `requirements.txt` |
| An unrepresentative month distorts coverage | High | Medium | Limit stated; a longer history requested |
