# Was the wine shop really sitting on three years of stock?

A stock and sales analysis for Bottleneck, an online wine shop, then an audit of my own notebook with two AI models: two figures I had presented were wrong, and the rebuilt pipeline now checks every export before trusting it.

**Read the case study:** https://axeldrtdata.github.io/works/bottleneck-stock-sales/

## Context

Bottleneck sells wine and spirits online. Its head of sales sent three exports for October (the ERP, the WordPress website and a link table between their product IDs) with a brief in two phases: reconcile the files and find the errors in them, then analyse revenue, the 80/20 rule, price outliers, margins, stock rotation and correlations for the management committee.

I presented that analysis in March 2026 (OpenClassrooms Project 6). In September 2026, the final project of my training (Project 13) asked me to audit one of my own deliverables with AI. This repository holds the rebuilt notebook.

## Key results

| Question | Result |
| -------- | ------ |
| Products sold online | 713, after joining 825 ERP products, 1,513 website rows and the link table (97.3% match rate) |
| Revenue in October | €119,666 excl. VAT, 5,737 items sold (7 of them €25 gift cards) |
| 80/20 rule | 433 products, 60.7% of the catalogue, make 80% of revenue: no star products |
| Stock | €277,225 at purchase price, covering 2.98 months of sales on average (2.40 at the median) |
| Where the capital sleeps | 27 products hold 39.7% of the stock value: 24 cover more than a year of sales, 3 never sold. 23 of them are champagnes, 38% of the whole stock |
| Sold-out products | 23 products sold out after selling 194 bottles, €5,516 excl. VAT in October, to reorder first |
| Price and sales | Spearman ρ = −0.71; stock and sales ρ = 0.76 |
| Price anomalies | 4 entry errors fixed by rule (3 sign errors, 1 price ten times too low), 3 products priced at or below cost reported |

### What the audit changed

I gave Claude and Gemini the same documents, with two postures: Claude as an internal auditor that had helped on the original project, Gemini as an external auditor of "a student's notebook". Claude raised 15 findings, Gemini 10; they shared 6 and disagreed on the severity of all six. Gemini missed both errors behind the figures I had presented. Every disputed point was settled on the data. The full audit is summarised in [`docs/ai-audit.md`](docs/ai-audit.md).

| Indicator | First version (March) | Rebuilt version |
| --------- | --------------------- | --------------- |
| Average stock coverage | 35.62 months: sales already monthly, divided by 12 | 2.98 months |
| Never-sold products | 3 ranked as the fastest-moving (coverage 0), 22 silently dropped | 25 identified, 3 of them holding €14,959 of stock |
| Revenue | €143,424 incl. VAT | €119,666 excl. VAT, the same basis as purchase prices |
| Stock value | €493,872 at selling price incl. VAT | €277,225 at purchase price |
| Product sold at a loss | 1, at −86% | 0: a price entered at €12.65 instead of €126.50 |
| Correlations | Pearson: −0.52 and 0.44 | Spearman: −0.71 and 0.76 |
| Corrections hard-coded on product IDs | 8 | 0 |
| Automated checks | 0 | 10 rules, blocking or alert |

## The quality checks

The notebook validates the three exports with [pandera](https://pandera.readthedocs.io/) before any analysis. Blocking rules stop the processing when a file cannot be used; alert rules report an anomaly and let the month's analysis go ahead. Entry errors are fixed by rule (a sign or scale change that brings the price back to 1.4 to 3 times the purchase price), never by product ID, so the notebook runs on next month's export without changes. The rules and the choices behind them are in [`docs/specification.md`](docs/specification.md).

## Repository structure

```
bottleneck-stock-sales/
├── data/
│   ├── raw/
│   │   ├── erp.xlsx              # ERP export: price, purchase price, stock, online status (825 products)
│   │   ├── web.xlsx              # WordPress export: SKU, sales in October, product details (1,513 rows)
│   │   └── liaison.xlsx          # link table: ERP product_id against website SKU
│   └── processed/                # written by the notebook
│       ├── priority_actions.xlsx # 73 products to act on, sorted by the value at stake
│       └── final_dataset.xlsx
├── docs/
│   ├── ai-audit.md               # the two AI audits, their cross-reviews and what the data settled
│   └── specification.md          # stakeholders, scope, rules, options compared, decisions
├── notebooks/
│   └── bottleneck_stock_sales_analysis.ipynb
├── requirements.txt
└── README.md
```

## How to run it

1. Clone the repository and open the folder in VS Code.
2. Create a virtual environment and install the dependencies:
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```
3. Open `notebooks/bottleneck_stock_sales_analysis.ipynb`, select the `.venv` kernel and run all cells. The notebook reads `data/raw/` with relative paths and writes its two Excel files to `data/processed/`. From a fresh kernel, it runs in under 30 seconds.

## Limits

- One month of data: no seasonality, although October comes right before the year-end peak. A dormant champagne may just be waiting for Christmas.
- VAT is not in the files: 20% is assumed, the standard French rate on wine and spirits, kept in a single constant. The three olive oils normally carry the reduced 5.5% rate.
- The €25 gift card is counted as a product: 7 sold, €146 of revenue excl. VAT.
- The thresholds (1 month to reorder, 12 months for dormant stock, a MAD score of 5 for prices) are conventions to set with the buyer.
- The 109 products the ERP declares offline are not analysed.
- The checks test that the exports are consistent, not that they are true.

## Stack

Python · Pandas · pandera · Plotly · Matplotlib · Claude and Gemini (audit) · VS Code

## Data sources

Three exports from Bottleneck's ERP and website for October, provided by OpenClassrooms for the Data Analyst path. Sales run from 1 to 31 October; stock is the state on 31 October. The files contain no personal data.

## Note on the translation

The original analysis was written in French in March 2026 (Project 6), and the audit and rebuilt notebook in French in September and October 2026 (Project 13), as part of my OpenClassrooms Data Analyst training. The notebook and documentation were translated into English for my portfolio. The source files are in French, so product names and types (`Vin`, `Huile d'olive`) stay in French.

---

Axel Derobert · Data Analyst · Lyon · [Portfolio](https://axeldrtdata.github.io) · [LinkedIn](https://www.linkedin.com/in/axel-derobert-5717463b1/)
