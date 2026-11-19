# NSW Property Sales 2016–2025
### Rebuilding a price model from the raw archive, and what the rebuild changed

**Mehmood Ahmed Khan** | Data Analyst & Analytics Engineer

[![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![pandas](https://img.shields.io/badge/pandas-data%20engineering-150458?logo=pandas&logoColor=white)](https://pandas.pydata.org/)
[![scikit-learn](https://img.shields.io/badge/scikit--learn-models-F7931E?logo=scikitlearn&logoColor=white)](https://scikit-learn.org/)

The first version of this project looked finished: a SQL pipeline, 23 charts, a case study and a headline model score. When I went back to the raw government archive to check it, several of its foundations did not hold. This repository documents the rebuild: what was wrong, how it was found, what replaced it, and what the corrected model can and cannot do.

> **Start here.**
> - **Report (23 pages):** [`report/Ten_Years_of_NSW_Property_Sales.pdf`](report/Ten_Years_of_NSW_Property_Sales.pdf)
> - **Presentation:** [`presentation/`](presentation/)
> - **Status:** the analysis and models described here are complete. An interactive map app is planned. The earlier SQL version is kept for the record in [`archive/`](archive/) and is **superseded**; see [`archive/README.md`](archive/README.md) for why it should not be used for results.

---

## At a glance

| | |
|---|---|
| **Source** | NSW Valuer General weekly property-sales extracts: 10 yearly archives, 520 weekly extracts, 66,747 district files, 4 Jan 2016 to 29 Dec 2025 |
| **Raw sale lines** | 2,186,102, of which 153,584 (7.0%) were the same sale re-published in a later extract and were removed |
| **Cleaned rows** | 2,032,518, with **0 imputed values**; 24 automated checks pass |
| **Unit of a sale** | the *dealing*: 1,978,968 dealings (35,377 span two or more lots) |
| **Modelling scope** | 1,899,113 dealings settled January 2016 to late October 2025 (96.0% of all dealings) |
| **Segments modelled** | House, Unit, Vacant land, Acreage residence |
| **Validation design** | train 2016–2023, tune on 2024, report once on 2025 |

### Headline results (2025 test year)

| Segment | Dealings tested | Typical error | Within 10% of the price | R² (log price) | Linear baseline error |
|---|---|---|---|---|---|
| House | 82,992 | **13.4%** | 39.7% | 0.762 | 16.7% |
| Unit | 51,255 | **10.0%** | 49.9% | 0.717 | 18.6% |
| Vacant land | 10,740 | **13.9%** | 39.5% | 0.678 | 25.4% |
| Acreage residence | 3,986 | **24.0%** | 21.3% | 0.546 | 31.2% |

*Typical error is the median absolute percentage error: half of the predictions are within that percentage of the actual price. It is a typical error, not an accuracy rate. R² is on log price and should not be compared across segments, because price spread differs by segment. The test year was looked at more than once during design, so these figures are slightly flattering. Details, baselines and ablations: [`docs/model_evaluation.md`](docs/model_evaluation.md).*

---

## The story: seven assumptions that did not hold

The first version merged the yearly files into one CSV of about 1.7 million rows (2018 to 2025), loaded it into DuckDB and modelled it. It produced maps, charts and a model that explained about 71% of the variation in price (R² of about 0.71). Nothing in it signalled a problem. What raised the doubt was the model: one blended local price carried 35.4% of its feature importance, and a random split can quietly flatter a score. Re-extracting from the raw archive, with row counts checked at every step, showed the following.

| # | First version | What the rebuild found | What changed |
|---|---|---|---|
| 1 | Treated each row as a separate sale | 153,584 lines (7.0%) were the **same sale re-published** in a later weekly extract, and the merged file had lost lines and dates | Extracted from the raw archive; every row accounted for (2,186,102 lines in, 153,584 duplicates out, 2,032,518 kept) |
| 2 | Assumed every area was in square metres | **139,687 rows (6.9%) are recorded in hectares**; the unit is a field in each record | Area unit read from each row; no single-unit assumption |
| 3 | Treated one row as one sale | One sale can span several lots, with the **same total price repeated on every lot** (35,377 dealings) | Analysis at the dealing level; the old price per m² was too high by a median factor of about 2.4 on multi-lot rows |
| 4 | Treated 2023 as a "gap caused by a filing artefact" | The dates had been **lost in extraction** (177,979 dealings); the raw files contain them | 2023 restored; no source year is missing |
| 5 | Used postcode-level LGA and region labels as if they described the locality | Those labels are **identical for every locality in a postcode** in 629 of 634 postcodes; 35% of postcodes span more than 20 km | Maps use locality coordinates; postcode labels carry a risk flag and were tested as model inputs (they added nothing) |
| 6 | Dropped the strata lot number as "mostly blank" (65.84% blank) | **The blank is the information:** it means the property is freehold. Without it, houses and units could not be told apart | Strata lot number kept; it defines the Unit segment |
| 7 | Reported one score from a random split on pooled residential data | A random split lets sales from the same markets and months land on both sides, which makes the test easier than real use | Time-based validation, one row per dealing, separate models per segment, and baselines to beat |

Headline findings from the first version that depended on items 2 and 5 (for example the Metro versus Very Remote price-per-m² ratio and the LGA rankings) are **retired**, not carried forward.

Three further points:
- **The first version's score is shown for history only.** Its R² of about 0.71 came from a different row set, split and labels, so it cannot be compared with these results. It was never proved inflated, and this project does not claim it was. A separate earlier pandas run (R² 0.84) was inflated by duplicated rows.
- **The analysis window moved from 2018–2025 to 2016–2025.** A better machine meant the full decade could be processed in one pass. January 2016 sales are as complete as later months (the first six months run at 84% to 102% of the same months a year later), so the extra two years were kept. The 2016–2017 rows are part of why the first version's 1,697,724 rows and the rebuild's 2,032,518 differ.
- **"House" and "Unit" are proxies.** The raw data has **no dwelling-type field**: nothing says house, townhouse, villa or apartment. A *Unit* here means strata-titled (apartments, but also strata townhouses and villas), and a *House* means a freehold residence on under 2 hectares.

---

## What the pipeline does now

```
NSW Valuer General weekly archives (10 zips, 520 extracts)
        │
        ▼
 1  EXTRACTION AND CLEANING
    parse every sale line; remove re-published copies (keep the latest, record the
    first publication date); read the area unit per row; match places by
    (locality, postcode); no imputation; every rule logs the rows it touches
        │            2,032,518 rows, 24 automated checks
        ▼
 2  SCOPE AND SEGMENTS
    one row per dealing; window, incomplete-tail and price rules as flags with a
    row ledger; segment label per dealing (House, Unit, Acreage residence, Vacant
    land, Farm or rural, Commercial or other)
        │            1,899,113 in-scope dealings
        ▼
 3  GROUP EDA AND MAPS
    the same six images per segment, four time periods each, one dot per locality
        │
        ▼
 4  MODELS
    as-of price features, time-based validation, baselines, ablations, a
    challenge from other boosting libraries
```

Design rules that held throughout: **no imputation; flag, do not filter; no step may multiply rows; every exclusion appears in a ledger with its row count.**

### Scope: what is in and out

| Rule (applied in order) | Dealings removed |
|---|---|
| Settled before 2016 | 19,500 |
| Settled in the incomplete last 60 days | 40,456 |
| Revision pair (same sale, conflicting price) | 193 |
| Lots disagree on one price | 18 |
| Freehold lot with a unit number in the address | 8,204 |
| Bulk sale (5 or more lots) | 1,542 |
| Price under $10,000 | 8,721 |
| Price over $50,000,000 | 1,221 |
| **Remaining** | **1,899,113** |

Nothing is deleted from the cleaned data; these are scope flags. Only the start of the window and the $10,000 floor (for vacant land) move any median price. The other rules protect the models from odd rows without changing the picture. The last 60 days are excluded because sales reach the public files after they settle: for settlements in 2022 to 2024, 99.5% were published within 60 days. The reasoning behind each threshold is in [`docs/pipeline_architecture.md`](docs/pipeline_architecture.md).

### Segments

| Segment | Dealings in scope | Median price, first period → last period |
|---|---|---|
| House | 943,278 | $650k → $1.03M |
| Unit | 600,499 | $670k → $780k |
| Vacant land | 202,293 | $321k → $480k |
| Acreage residence | 51,160 | $675k → $1.17M |
| Farm or rural | 26,700 | $750k → $1.45M |
| Commercial or other | 68,767 | $660k → $920k |
| Unknown | 6,416 | not shown |

*Periods are 2016–19 and 2024–25 (the latter to late October), in nominal dollars, not adjusted for inflation and not like-for-like. The two descriptive segments, Farm or rural and Commercial or other, are not modelled because each mixes very different kinds of property.*

---

## How the models are built, and how they are kept honest

- **Target:** log price at the dealing level, scored back in dollars.
- **Main inputs:** the price level of the locality and of the whole segment **as it stood in earlier months**, location and distance from the Sydney CBD, area, zoning, lot counts, and, where they exist, the **previous sale of the same dwelling** and the **price level of the same building** (strata).
- **Leakage controls:** those price levels use only sales settled in strictly earlier months, never the dealing's own month. This was tested against a brute-force calculation and with a probe that edits one price and checks that nothing in that month or earlier changes; the functions and their tests are in [`code/`](code/). Category lists are learned from the training years only, and `Price_per_sqm` is never an input.
- **Protocol:** fit on 2016–2023 and tune on 2024; refit on 2016–2024 and score 2025. The test year chose nothing: the number of trees and the final model were fixed on 2024.
- **Baselines the model has to beat:** a locality price-level baseline and a linear (Ridge) model with the same inputs. The model beats both in every segment.

### What the results say

- **The model beats both baselines in every segment**, but modestly against the linear model for houses (13.4% against 16.7% typical error). Much of the signal is the neighbourhood's recent price level, area and location.
- **Earlier sales are the biggest single gain.** Adding the previous sale of the same dwelling cuts typical error from 15.2% to 13.4% for houses and from 26.5% to 24.0% for acreage. For units, adding the building's price level as well cuts it from 15.9% to 10.0%. For vacant land the previous sale shows no reliable gain on validation. Only about a third of 2025 test houses (34%) had an earlier sale in the data, so the gain comes from a minority of dealings; for units it was 88%.
- **Geography matters, and distance to the CBD carries most of it for houses and units.** Removing all geography made every segment worse on the 2024 validation year: houses by 1.0 point, units by 2.0, acreage by 2.5 and vacant land by 4.8.
- **Postcode-level labels added nothing** for houses, units and acreage, and made vacant land 0.8 points worse (2024 validation), so the final models leave them out.
- **The model is weakest where data is thin.** Where a locality has no recent price level to learn from, typical error rises to 26.7% for houses and 32.8% for vacant land (2025 test year).
- **A challenge from LightGBM and XGBoost** was run under a rule set before looking: a candidate had to win on validation, win on the test year by more than 0.5 percentage points, and have a bootstrap interval excluding zero. Nothing cleared it in any segment. The best case, a three-model blend for units, gained about 0.3 points and was judged not worth three models to maintain.

---

## What the market looks like (descriptive, not causal)

<p align="center">
  <img src="docs/images/house_map_median_price_sydney.png" width="95%" alt="House median price by locality, Sydney, four periods">
</p>

- **Prices rose unevenly.** From 2016–19 to 2024–25 the median rose 58% for houses, 16% for units, 50% for vacant land and 73% for acreage residences, in nominal dollars. Acreage peaked near $1.19 million in 2022–23 and slipped to about $1.17 million in 2024–25.
- **Prices rose at every distance from the CBD.** The median house price rose by 48% in the 0–10 km band and by 69% in the 160–320 km band. In dollars the gap is largest within 10 km, at about $0.9 million, and smallest beyond 320 km, at about $229 thousand.
- **Prices cluster on round numbers.** Exactly $1,000,000 occurs 4.2 times as often as nearby prices, and exactly $2,000,000 occurs 13 times as often.
- **Sub-$10,000 sales are fading.** They were 0.80% of dealings in 2016 and 0.14% in 2025. Three-quarters of those under $10,000 are vacant land at round amounts, which suggests nominal transfers; the cause of the decline is unknown.
- **Publication got faster.** The median delay between settlement and first publication fell from 19 days (2016) to 7 days (2019 onward).
- **2021 was the busiest year**, with 230,964 dealings. April 2020 dipped, and late 2020 through 2021 surged.

<p align="center">
  <img src="docs/images/house_price_by_distance_remoteness.png" width="95%" alt="House median price by distance and remoteness class">
</p>

---

## Limits, stated plainly

- **No property attributes.** There is nothing on bedrooms, condition, building age or views. That sets a ceiling on the results, and the typical error is a statement about valuing from location, area and price history only. The output is an estimate as of a given month, not a valuation.
- **Proxies, not types.** House and Unit are inferred from strata title and area (see above).
- **Properties never seen before are not reported separately.** 34% of 2025 test houses and 88% of test units were already in the training window.
- **Price history helps only where it exists.** The previous-sale and building inputs need earlier sales, and the data starts in 2016.
- **The price levels are slightly optimistic.** They assume last month's sales are known when valuing, but a few sales take weeks to be published.
- **Scope.** Prices under $10,000 and over $50 million, bulk sales and revision pairs are outside the model, so the results apply to ordinary in-scope sales (96.0% of dealings).
- **Postcode-level labels remain unreliable.** 28.5% of all sales fall in postcodes flagged high-risk. The final models do not use them, and maps are drawn from locality coordinates.
- **There is one test year.** It was looked at more than once during design, and the previous-sale and building inputs were designed after seeing the first run's results, so the 2025 figures are slightly flattering. The challenge with other models covered boosting libraries only.
- **Acreage is the weakest model**, with the fewest test sales (3,986) and thin local markets.
- **The 2020–21 pandemic period is not treated separately.** It is in the data and the models train on it without adjustment; a sensitivity check would be a natural next step.
- **Not tried:** prediction intervals, a repeat-sales price index (a quarter of properties sold more than once) and macro-economic inputs such as the cash rate.

---

## What is in this repository

```
README.md                  this file
report/
  Ten_Years_of_NSW_Property_Sales.pdf   the 23-page report
presentation/              the final presentation
docs/
  data_provenance.md       where the data comes from, what it can and cannot tell you
  pipeline_architecture.md the design, the rules and the evidence behind each decision
  model_evaluation.md      full results, ablations, the challenge and the leakage audit
  images/                  figures used in the documents
code/
  price_history_features.py       the two leakage-critical feature functions
  test_price_history_features.py  15 tests on synthetic data (run: cd code && pytest -q)
archive/                   the superseded first version (notebooks, charts, case-study page)
```

**The full analysis code is not published.** Every figure in this repository was taken from an executed analysis notebook, and the documents describe the method, the rules and the checks in enough detail to reproduce them from the public source data. The one piece that is published is the part most open to doubt: the functions that build price-history inputs without leakage, with tests that need no data ([`code/`](code/)). They show that those functions behave as described; they do not on their own show that the whole pipeline is leak-free (see the audit in [`docs/model_evaluation.md`](docs/model_evaluation.md)). The rest of the code is not public, and I am glad to walk through it on request (contact via LinkedIn below).

---

## Data sources

| Source | Use |
|---|---|
| [NSW Valuer General](https://valuation.property.nsw.gov.au/embed/propertySalesInformation), weekly property sales extracts, published under a Creative Commons Attribution licence | all sales data; attribute NSW Valuer General when reusing |
| [Matthew Proctor, Australian Postcodes](https://matthewproctor.com/australian_postcodes) | locality coordinates and postcode reference |
| ABS ASGS Edition 3 (2021), CC BY 4.0 | state and territory boundaries on the maps |

---

## Earlier work

The first version of this project, a DuckDB SQL pipeline over a merged CSV with 23 charts and a case-study page, is kept in [`archive/`](archive/). It is **superseded**, and its headline findings are retired for the reasons in the table above.

---

## Author
**Mehmood Ahmed Khan**, Data Analyst & Analytics Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Mehmood%20Ahmed-0077B5?logo=linkedin&logoColor=white)](https://www.linkedin.com/in/mehmoood)
[![GitHub](https://img.shields.io/badge/GitHub-Mehmoodkhans-181717?logo=github&logoColor=white)](https://github.com/Mehmoodkhans)
