# Revenue forecast with Python in Power Query

The question: will 2024 reach its target? Sales stop on 20 April 2024. A Python script in Power Query forecasts April to December 2024, and the forecast goes onto the line chart "revenue against target" from `measures.md` section 7.

Copy one block at a time.

## 1. Check Python

Open a Command Prompt and run:

```
python -c "import sys; print(sys.executable)"
```

The path must not contain `WindowsApps`. A `WindowsApps` path is the Microsoft Store shortcut, and Power BI cannot use it. Install Python from python.org or with the Python Install Manager, then run the command again.

Write down the folder of `python.exe`, for example `C:\Users\<you>\AppData\Local\Python\pythoncore-3.14-64`. You need it in step 2.

Install the packages:

```
python -m pip install pandas matplotlib statsmodels
```

Check that they load:

```
python -c "import pandas, statsmodels, matplotlib; print('ok')"
```

## 2. Power BI settings

**File > Options and settings > Options > Python scripting**: in **Set a Python home directory**, choose **Other**, then select the folder of `python.exe` from step 1. Click OK.

**File > Options and settings > Options > Current file > Privacy**: choose **Ignore the Privacy Levels and potentially improve performance**. Click OK.

Why: the Python step sends data from `sales.csv` to Python, which Power BI treats as another data source, and without this setting the refresh stops with a `Formula.Firewall` error.

## 3. Power Query: the query `Forecast`

In Power Query: **Home > New Source > Blank Query**. Rename the query to `Forecast`. Then **Home > Advanced Editor**, delete the text, paste this block, and click Done.

```m
let
    Source = FactSales,
    // Revenue per order line, the same formula as the Revenue measure
    #"Added Revenue" = Table.AddColumn(Source, "Revenue", each [Quantity] * [UnitPrice] * (1 - [Disc]), type number),
    #"Added MonthStart" = Table.AddColumn(#"Added Revenue", "MonthStart", each Date.StartOfMonth([OrderDate]), type date),
    #"Grouped by MonthStart" = Table.Group(#"Added MonthStart", {"MonthStart"}, {{"Revenue", each List.Sum([Revenue]), type number}}),
    // April 2024 is a partial month (sales stop on 20 April), so it must not train the model
    #"Removed April 2024" = Table.SelectRows(#"Grouped by MonthStart", each [MonthStart] < #date(2024, 4, 1)),
    // Send the date as ISO text (yyyy-mm-dd), so Python reads it the same way on every laptop
    #"MonthStart as text" = Table.TransformColumns(#"Removed April 2024", {{"MonthStart", each Date.ToText(_, [Format = "yyyy-MM-dd"]), type text}})
in
    #"MonthStart as text"
```

Check: the table has 111 rows, from 2015-01-01 to 2024-03-01.

Then **Transform > Run Python script**, paste this script, and click OK. If Power BI asks for permission to run the script, click **Run**.

```python
# 'dataset' comes from Power Query: MonthStart and Revenue, one row per full month
import pandas as pd
from statsmodels.tsa.holtwinters import ExponentialSmoothing

# 1. Read MonthStart as a date, whether it arrives as text or as datetime
df = dataset.copy()
df["MonthStart"] = pd.to_datetime(df["MonthStart"])

# 2. Make a monthly series in date order (MS = the first day of each month)
history = df.sort_values("MonthStart").set_index("MonthStart")["Revenue"].astype(float).asfreq("MS")

# 3. Holt-Winters: a trend that flattens out (damped) plus a 12-month season
model = ExponentialSmoothing(history, trend="add", damped_trend=True, seasonal="add", seasonal_periods=12).fit()

# 4. Forecast the months left in the last year: April to December 2024
months_left = 12 - history.index[-1].month
prediction = model.forecast(months_left)

# 5. Return MonthStart as ISO text (yyyy-mm-dd), so Power Query reads it in any locale
forecast = pd.DataFrame({"MonthStart": prediction.index.strftime("%Y-%m-%d"), "Forecast": prediction.round(0).values})
```

Power BI shows a table of the tables that the script made. In the row `forecast`, click **Table**.

The columns come back as text. Click **fx** in the formula bar and paste this step. Replace `forecast` with the name of your last step if it is different.

```m
= Table.TransformColumnTypes(forecast, {{"MonthStart", type date}, {"Forecast", type number}}, "en-US")
```

Check: 9 rows, from 2024-04-01 to 2024-12-01. Then **Home > Close & Apply**.

### Why this model

We held out 2023, trained on the years before, and compared the forecast with the actual months. We did the same for April to December 2023 (trained to March 2023, as we do now for 2024) and for 2022. The baseline "seasonal naive" repeats the same month of the year before.

| Model | 2023: MAPE per month | 2023: total against actual $31,871,231 | Apr–Dec 2023: MAPE | 2022: MAPE |
|---|---|---|---|---|
| Seasonal naive (same month last year) | 38.9% | $43,426,104 (+36.3%) | 46.9% | 50.9% |
| Holt-Winters, additive season, no trend | 52.5% | $43,224,860 (+35.6%) | 29.9% | 23.0% |
| Holt-Winters, additive season, damped trend | 44.2% | $40,587,279 (+27.3%) | 23.7% | 25.0% |
| Holt-Winters, multiplicative season, damped trend | 54.1% | $41,576,764 (+30.5%) | 24.3% | 27.9% |
| Holt-Winters, multiplicative season, damped trend, last 5 years only | 41.8% | $37,435,897 (+17.5%) | 27.1% | 31.3% |

All models use the full history from 2015, except the last row. MAPE is the mean absolute percentage error of the monthly forecast.

No model forecasts 2023 well: revenue fell 27% from 2022, and no model saw the fall coming. In 2023 the seasonal naive has the lowest monthly error, but it misses the year total by the most. In the other two tests, Holt-Winters with an additive season and a damped trend has about half the error of the seasonal naive. It is also the simplest model that is good in all three tests: one setting for the trend, one for the season, and no choice of time window.

## 4. Model

In Model view, check the relationships of `Forecast`. If Power BI made a relationship to `DimDate[MonthStart]`, delete it.

Drag `Forecast[MonthStart]` onto `DimDate[Date]`. Power BI proposes "One to one", because both columns are unique. Change it to **Many to one (\*:1)**, with **Cross filter direction: Single**. Click OK.

## 5. Measures

Select **FactSales**, then **Home > New measure**, as in `measures.md`. Format: currency with 0 decimals, except `Estimate vs Target %` as percentage with 1 decimal.

```dax
Forecast Revenue = SUM ( Forecast[Forecast] )
```

The full-year estimate is the actual revenue of January to March plus the forecast for April to December. The forecast replaces the partial April actual (1 to 20 April).

```dax
Revenue Estimate =
VAR FirstForecastMonth =
    CALCULATE ( MIN ( Forecast[MonthStart] ), REMOVEFILTERS ( Forecast ) )
RETURN
    CALCULATE ( [Revenue], KEEPFILTERS ( DimDate[Date] < FirstForecastMonth ) )
        + [Forecast Revenue]
```

```dax
Estimate vs Target % = DIVIDE ( [Revenue Estimate], [Target] )
```

Check: with Year = 2024, `Revenue Estimate` is $27,300,307, `Target` is $42,000,000 and `Estimate vs Target %` is 65.0%.

## 6. Visuals

The first page does not change: it stays as `measures.md` section 7 builds it. The forecast goes on a new page.

**New page.** Add a page and name it `Forecast`.

**Line chart: revenue against target, with the forecast.** On the first page, select the line chart "revenue against target" and press Ctrl+C. Go to the page `Forecast` and press Ctrl+V. Make these changes on the copy only:

- Add `Forecast Revenue` to the Y-axis, under `Revenue` and `Target`.
- In the Filters pane of the copy, delete the filter `IsPastSales = True`. The filter hides all dates after 20 April 2024, so it hides the forecast for May to December. Keep the filter Year = 2022, 2023, 2024.

Without the filter, the copy shows the target for all 12 months of 2024, and its `Revenue` line drops in April 2024 because April holds only 1 to 20 April. The forecast line shows the full month of April.

To show the forecast as a dashed line: **Format visual > Lines**, select the series `Forecast Revenue`, then **Line style: Dashed**.

**Card (new): the year estimate.** On the page `Forecast`. Values `Revenue Estimate`, `Target`, `Estimate vs Target %`. Filter on the visual: Year = 2024.

The answer: no. With the forecast, 2024 reaches about 65% of its target, a gap of about $14.7 million.

Forecast per month:

| MonthStart | Forecast |
|---|---|
| 2024-04-01 | $1,279,878 |
| 2024-05-01 | $2,409,018 |
| 2024-06-01 | $2,519,072 |
| 2024-07-01 | $2,020,641 |
| 2024-08-01 | $2,159,390 |
| 2024-09-01 | $2,207,701 |
| 2024-10-01 | $2,246,819 |
| 2024-11-01 | $2,222,361 |
| 2024-12-01 | $2,610,271 |
| Total | $19,675,151 |

Another version of statsmodels can give numbers that differ a little.

## If it fails

- **Python not found**, or "Python home directory is not valid": the home directory in step 2 is wrong or is a `WindowsApps` path. Run the command of step 1 and select the folder that it prints.
- **`No module named pandas`** (or `statsmodels`): the packages are installed for another Python. Run `python -m pip install pandas matplotlib statsmodels` with the same `python.exe` that Power BI uses, then restart Power BI Desktop.
- **`Formula.Firewall`**: set **Ignore the Privacy Levels** in step 2, then click **Refresh preview**.
- **The script times out**: Power BI stops a Python script after 30 minutes. This script takes a few seconds. If it runs long, check that the step before it is `MonthStart as text` with 111 rows, not the full FactSales table.
- **The date column comes back as text**: add the type step after `forecast` (step 3). Its locale `en-US` reads `yyyy-mm-dd` on every laptop.
