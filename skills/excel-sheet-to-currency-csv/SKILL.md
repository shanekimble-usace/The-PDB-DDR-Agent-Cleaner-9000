---
name: excel-date-standardizer
description: >
  Processes an uploaded file (Excel workbook with a specified sheet, or a direct CSV), identifies date columns using case-insensitive header keywords ('date', 'start', 'finish', 'last modified') and pattern inspection, normalizes mixed date formats (including '4-Sep-26', '9/1/26', 'MM/DD/YYYY', and timestamps) into standard ISO format (YYYY-MM-DD) using US date conventions (MM/DD/YYYY), and exports a single lightweight CSV file. Use this skill whenever a user asks to "standardize mixed date formats in Excel or CSV," "convert dates like 4-Sep-26 to YYYY-MM-DD," "clean schedule or finish dates," or "export standardized date CSV."
---

# Universal Tabular Date Standardizer

## What this skill is for

This skill automates the identification and normalization of inconsistent, mixed-format date columns across tabular datasets. It ingests either an uploaded Excel file (`.xlsx`, `.xlsm`) with a user-specified sheet name or a standalone `.csv` file. It targets date columns by scanning headers for key indicators (`date`, `start`, `finish`, `last modified`) and inspecting cell contents. It reliably handles tricky mixed-string variations within the same column (such as `4-Sep-26` alongside `9/1/26`), resolves 2-digit years, parses ambiguous dates using US conventions (`Month/Day/Year`), converts all valid entries to standard `YYYY-MM-DD`, and writes out a single clean CSV file.

## What this skill will NOT do

- This skill does not output multi-sheet Excel files; it outputs strictly a single `.csv` file.
- This skill does not assume international date convention (`DD/MM/YYYY`) when dates are ambiguous; it strictly defaults to US format (`MM/DD/YYYY` where `9/1/26` means September 1, 2026).
- This skill does not retain time or timezone components; all date values are standardized strictly to `YYYY-MM-DD`.
- This skill does not alter columns that do not match the date header criteria or contain non-date data.
- This skill does not guess the sheet name when an Excel workbook is provided without one.

## Connectors and knowledge sources

This skill is instruction- and code-execution-based and does not require external connectors or document repositories. It processes user-uploaded files locally in the session environment using Python data libraries (`pandas`, `python-dateutil`).

## How to do the task

### Step 1 — Ingest Input Data Efficiently
Inspect the uploaded file extension:

| File Type | Ingestion Protocol |
|---|---|
| **CSV File (`.csv`)** | Load the file directly using `pd.read_csv()`, keeping text columns as strings (`dtype=str` or standard object types). Do not ask for a sheet name. |
| **Excel File (`.xlsx`, `.xlsm`)** | Read sheet names using low-memory inspection (`openpyxl.load_workbook(filename, read_only=True).sheetnames` or `pd.ExcelFile(path).sheet_names`). If the user specified a sheet name, load ONLY that sheet. If missing or invalid, halt and prompt the user with the available sheets. |

### Step 2 — Identify Date Columns via Header Keywords and Content
Evaluate every column in the dataset to select columns for date standardization:

1. **Header Keyword Match (Case-Insensitive):**
   Check if the column name contains any of the following substrings:
   - `date` (e.g., "Due Date", "Order_Date", "DATE")
   - `start` (e.g., "Start Time", "Project Start", "actual_start")
   - `finish` (e.g., "Finish Date", "Target Finish", "finish")
   - `last modified` (e.g., "Last Modified", "last_modified_by_date", "Last Modified Date")
   Any column containing one of these terms is **automatically designated** as a date column.

2. **Automatic Pattern Detection (for other columns):**
   For columns whose headers do not contain those keywords, sample non-empty values. If values resemble dates (e.g., `D-Mon-YY`, `M/D/YY`, `YYYY-MM-DD`, or timestamps), flag the column for conversion. Do not flag pure numeric columns (IDs, counters, currency).

### Step 3 — Robust Cell-by-Cell Mixed Date Parsing
Columns often contain mixed formats (e.g., `4-Sep-26` and `9/1/26` in the same column). Apply robust parsing to every cell in designated date columns:

1. **Clean cell values:** Strip leading/trailing whitespace. If the cell is null, NaN, or an empty string, leave it as an empty string `""`.
2. **Parse with US date convention & 2-digit year support:**
   Use `python-dateutil.parser.parse(val, dayfirst=False)` or a robust fallback function:
   - Handles short-month strings like `4-Sep-26` &rarr; `2026-09-04`.
   - Handles slash dates like `9/1/26` &rarr; `2026-09-01` (interpreting `9` as Month, `1` as Day, `26` as 2026).
   - Handles standard ISO dates and full timestamps (e.g., `2026-09-15 14:30:00` &rarr; `2026-09-15`).
3. **Format string:** Output strictly `YYYY-MM-DD`.
4. **Fallback for unparseable entries:** If an individual cell cannot be parsed into a date (e.g., text like "TBD", "N/A", or "Pending"), preserve the original cell text verbatim so no operational notes are destroyed.

### Step 4 — Export Transformed Dataset to CSV
Export the transformed table to a single CSV file:
- Output filename: `[source_name]_[sheet_name]_standardized_dates.csv` (for Excel) or `[source_name]_standardized_dates.csv` (for CSV).
- Use `index=False`, standard UTF-8 encoding, and standard CSV quoting.
- Present a concise transformation summary table and provide the file download.

## Output format

The final response must provide an execution summary table followed by the download file link:

| Metric / Parameter | Value |
|---|---|
| **Input File** | `[File Name]` |
| **Input Type** | `Excel (Sheet: [Name])` OR `Standalone CSV` |
| **Matched Date Columns** | `[List of columns identified and standardized]` |
| **Trigger Reason** | `[Header Keyword Match / Pattern Detection]` |
| **Output File** | `[Generated CSV Name]` |

Download: `[File download link / attachment]`

## Failure modes to watch for

- **Crashing on mixed formats:** Do not rely on fixed-format parsers like `pd.to_datetime(col, format='%m/%d/%Y')`. Because a single column can have `4-Sep-26` and `9/1/26`, parse row-by-row or use flexible parsing with `dateutil.parser.parse(str(val), dayfirst=False)`.
- **Interpreting 9/1/26 as January 9th:** Always enforce `dayfirst=False` so `9/1/26` is parsed as September 1, 2026.
- **Converting 'TBD' or 'Pending' to NaT:** When a date column contains status notes, do not wipe them out. Keep the raw text if parsing fails.
- **Exporting "NaT" into the CSV:** Ensure empty or null dates output as blank strings `""`, never literal `"NaT"`.

## When the input is unclear

- **No file uploaded:** Prompt the user to upload either an Excel workbook (`.xlsx`, `.xlsm`) or a `.csv` file.
- **Excel file uploaded without sheet name:** List all sheet names present in the workbook and ask the user which one they want to process.
- **Excel sheet name mismatch:** Display the available sheet names found in the workbook and ask the user to verify their selection before proceeding.

## Why this design

In project schedules and construction logs, dates are frequently entered by different teams resulting in mixed representations (`4-Sep-26`, `9/1/26`) within the exact same milestone or completion column. Explicitly targeting schedule headers (`date`, `start`, `finish`, `last modified`) paired with flexible, US-ordered datetime parsing ensures 100% capture of schedule dates without corrupting non-date text entries.
