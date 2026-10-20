# 2023 Travel Bookings: Data Screening, Cleaning & Transformation

A Python (pandas) project that screens, cleans and enriches a raw export of 2023 travel-package bookings, producing an analysis-ready dataset.

---

## 1. Project Overview

| | |
|---|---|
| **Goal** | Turn the raw 2023 bookings export into a consistent, analysis-ready table |
| **Scope** | Data screening → data cleaning → light feature engineering |
| **Tooling** | Python 3, pandas, Jupyter Notebook |
| **Records** | 3,867 bookings (no rows added or dropped) |
| **Period** | Bookings made 1 Jan – 31 Dec 2023; travel dates run from Jan 2023 to Jun 2025 |

---

## 2. Repository Contents

```
Bookings/
├── clean/
│   └── bookings_cleaned_python_2023.csv   # CLEANED output for 2023 (15 columns)
├── notebook/
│   └── analysis.ipynb                     # Screening, cleaning and transformation code
├── raw/
│   ├── Bookings_2023 (1).csv              # RAW 2023 input (12 columns), documented here
│   ├── Bookings_2024 (1).csv              # RAW 2024 input (not yet cleaned)
│   └── Bookings_2025 (1).csv              # RAW 2025 input (not yet cleaned)
└── README.md
```

| Folder | Purpose |
|---|---|
| `raw/` | Original exports, kept untouched. Never edit these files. |
| `notebook/` | The Jupyter notebook that reads from `raw/` and writes to `clean/`. |
| `clean/` | Analysis-ready outputs. |

> **Scope note:** only the 2023 file has been screened and cleaned so far. The 2024 and 2025 raw files are in the repo but are not covered by this documentation yet.

---

## 3. Data Dictionary

### Raw file: `raw/Bookings_2023 (1).csv`

| Column | Type | Description |
|---|---|---|
| `BookingID` | text | Unique booking reference, format `BK-2023-NNNNN` |
| `BookingDate` | date | Date the booking was made |
| `TravelStartDate` | date | Date the trip starts |
| `CustomerID` | text | Customer reference, format `CNNNNN` (2,946 unique customers) |
| `PackageID` | text | Travel package (`PK01`–`PK17`) |
| `AgentID` | text | Sales agent (`AG01`–`AG13`); blank if no agent |
| `Adults` | int | Number of adults |
| `Children` | int | Number of children (0–3) |
| `UnitPriceUSD` | int | Price per traveller in USD |
| `DiscountPct` | float | Discount tier: 0, 0.05, 0.10 or 0.15 |
| `Status` | text | Booking outcome |
| `PaymentMethod` | text | Card, Bank Transfer, PayPal, M-Pesa |

### Cleaned file: `clean/bookings_cleaned_python_2023.csv`

Same as above, with these differences:

| Column | Change |
|---|---|
| `Adults` | **Renamed** to `AdultCount` |
| `BookingDate`, `TravelStartDate` | **Reformatted** from `YYYY-MM-DD` to `DD/MM/YYYY` |
| `Status` | **Standardised** to `Completed` / `Cancelled` |
| `HeadCount` | **New:** `AdultCount + Children` |
| `TotalPrice` | **New:** `HeadCount × UnitPriceUSD` |
| `ActualPaidPrice` | **New:** price after discount (see Section 5 and the known issue in Section 7) |

---

## 4. Data Screening Findings

Screening was done on the raw file before any changes were made.

| Check | Result |
|---|---|
| Shape | 3,867 rows × 12 columns |
| Duplicate rows / duplicate `BookingID` | None |
| ID formats (`BookingID`, `CustomerID`) | All valid |
| Date parsing | All dates valid; no travel date earlier than its booking date |
| **`Status` inconsistencies** | **9 spellings** for what are really 2 outcomes: `Completed`, `completed`, `COMPLETED`, `Completed ` (trailing space), `Cancelled`, `Canceled`, `cancelled`, `CANCELLED`, `Cancelled ` |
| **Missing `AgentID`** | 1,100 rows (28.4%) |
| **Missing `DiscountPct`** | 120 rows (3.1%) |
| `PaymentMethod` | Clean: 4 consistent values |
| Package pricing | Each package has up to 5 different unit prices across the year (likely seasonal or tiered pricing) |

---

## 5. Data Cleaning & Transformation Steps

1. **Load** the raw CSV with pandas and inspect shape, dtypes, nulls and unique values.
2. **Standardise `Status`**: strip whitespace, lowercase, then map `canceled`/`cancelled` → `Cancelled` and `completed` → `Completed`. Result: 3,507 Completed, 360 Cancelled.
3. **Rename** `Adults` → `AdultCount` for clarity next to `Children`.
4. **Reformat dates** to `DD/MM/YYYY`.
5. **Engineer features:**
   - `HeadCount = AdultCount + Children`
   - `TotalPrice = HeadCount × UnitPriceUSD`
   - `ActualPaidPrice = TotalPrice × (1 − DiscountPct / 100)`
6. **Handle missing values:** `AgentID` and `DiscountPct` were intentionally **left as null** rather than imputed, because the true values are unknown. Consequently `ActualPaidPrice` is null for the 120 rows with no discount value.
7. **Export** to `clean/bookings_cleaned_python_2023.csv`.

> The steps above were reconstructed by comparing the raw and cleaned files. If your notebook differs in any detail, please adjust this section to match.

---

## 6. Validation Checks on the Cleaned Output

| Check | Result |
|---|---|
| Row count matches raw (3,867) | ✅ |
| `Status` has exactly 2 values | ✅ |
| `HeadCount == AdultCount + Children` for every row | ✅ |
| `TotalPrice == HeadCount × UnitPriceUSD` for every row | ✅ |
| No duplicate `BookingID` | ✅ |
| `PaymentMethod` unchanged | ✅ |
| Nulls remaining: `AgentID` 1,100 · `DiscountPct` 120 · `ActualPaidPrice` 120 | ⚠️ By design |

---

## 7. Known Issues & Recommendations

1. **Discount formula looks wrong (please verify).** `DiscountPct` holds fractions (0.15 = 15%), but `ActualPaidPrice` is computed as `TotalPrice × (1 − DiscountPct/100)`, so a "15%" discount only reduces the price by 0.15%. For example, booking `BK-2023-00005` is priced at 420 and recorded as 419.37 instead of 357.00. If the intended discount is 15%, the formula should be `TotalPrice × (1 − DiscountPct)`. This affects 1,760 rows and overstates paid revenue (about $35.9M recorded versus a lower figure once corrected).
2. **Cancelled bookings carry a paid price.** Cancelled rows still have an `ActualPaidPrice` (about $3.0M in total). Decide whether this should be zero or a refund-adjusted amount before doing revenue analysis.
3. **Dates are stored as `DD/MM/YYYY` text.** For analysis, parse them back with `pd.to_datetime(..., dayfirst=True)`, or keep ISO format in future exports to avoid day/month ambiguity.
4. **Missing agents (28%).** Treat as "Direct / No agent" in reporting, but confirm with the business that blank means no agent.
5. **Missing discounts (3%).** Confirm whether blank means "no discount" (0) before imputing.

---

## 8. How to Run

```bash
# 1. Install dependencies
pip install pandas jupyter

# 2. Launch the notebook from the project root
jupyter notebook notebook/analysis.ipynb
```

Run all cells in order. The notebook reads `raw/Bookings_2023 (1).csv` and writes `clean/bookings_cleaned_python_2023.csv`. Because the notebook lives in `notebook/`, file paths inside it should be relative to that folder:

```python
RAW_PATH   = "../raw/Bookings_2023 (1).csv"
CLEAN_PATH = "../clean/bookings_cleaned_python_2023.csv"
```

Quick load of the cleaned data:

```python
import pandas as pd

df = pd.read_csv("clean/bookings_cleaned_python_2023.csv")
df["BookingDate"] = pd.to_datetime(df["BookingDate"], dayfirst=True)
df["TravelStartDate"] = pd.to_datetime(df["TravelStartDate"], dayfirst=True)
```

---

## 9. Possible Next Steps

- Fix and re-run the discount calculation (Section 7, item 1)
- Apply the same screening and cleaning to the 2024 and 2025 raw files, then combine the years
- Add `LeadTimeDays` (travel start − booking date) and `BookingMonth`
- Analyse cancellation rate by package, agent and payment method
- Revenue and seasonality dashboard

---

*Author: add your name · Last updated: October 2026*
