# Power BI for Business Intelligence — participant files

Files for the two-day training on 12–13 November 2026.

## Download

Click **Code > Download ZIP** at the top of this page, then extract the ZIP. You do not need git.

Extract it to a short path, for example `C:\Training\`. Then the CSV files are in `C:\Training\data\`.

## What is in each folder

| Path | Used in | Content |
|---|---|---|
| `data/` | Day 1, Labs 1–4 | The six raw CSV files: `sales.csv`, `products.csv`, `stores_plan.csv`, `stores.csv`, `customers.csv`, `target.csv` |
| `pbix/Contoso-Before.pbix` | Day 1 morning, Day 2 capstone | The "before" dashboard that we critique and then redesign |
| `measures.md` | Day 2 | The measures and visual settings of the final dashboard, ready to copy |
| `forecast.md` | Day 2 afternoon | The revenue forecast with Python in Power Query: setup, the M and the script, the measures and the chart |
| `lab5/contoso-theme-starter.json` | Day 2, Lab 5 | A starter theme to edit |
| `lab8/slow-measures-10M.dax` | Day 2, Lab 8 | The slow measures to refactor, for `Contoso 10M.pbix` |
| `lab8/LICENSE-SQLBI.txt` | Day 2, Lab 8 | Licence of the SQLBI Contoso data (MIT) |
| `Coursebook_Power_BI_for_Business_Intelligence.pdf` | Both days | The coursebook. The slides name the pages to read for each lab |

## Lab 8 needs one more file: `Contoso 10M.pbix`

It is 483 MB, too large for this repository. Download it from SQLBI before Day 2:

1. Download `pbix-10M.7z` (433 MB): https://github.com/sql-bi/Contoso-Data-Generator-V2-Data/releases/download/ready-to-use-data/pbix-10M.7z
2. Extract it with 7-Zip. The result is `Contoso 10M.pbix`.
3. Open it once in Power BI Desktop to check that it opens. **Do not refresh it**: its data source is not reachable, so the refresh fails.

The file needs about 1 GB of free memory when it is open. Close other Power BI files during Lab 8.

## If you refresh a `.pbix` from `pbix/`

The `.pbix` files already hold the data, so you do not need to refresh them. If you do refresh, first point the model to your CSV folder: **Transform data > Edit parameters > DataFolder**, and enter the path of your `data` folder, for example `C:\Training\data\`.

## Before Day 1

- Power BI Desktop is installed and opens `pbix/Contoso-Before.pbix`.
- The `data` folder holds the six CSV files.
