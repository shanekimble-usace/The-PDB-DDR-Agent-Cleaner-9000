---
name: excel-prefix-stripper
description: >
  Processes an uploaded file (either a multi-tab Excel workbook or a single CSV file), scans all cell values across the dataset for the pattern 'E########-' (an uppercase 'E' followed by exactly 8 numeric digits and a hyphen), strips that leading prefix from matching cells while preserving all subsequent content, and exports a single lightweight CSV file. Use this skill whenever a user asks to "strip E prefixes from cells," "remove E########- from spreadsheet values," "clean project code prefixes from table data," or "remove leading tracking codes from Excel or CSV."
---

# Universal Tabular Prefix Stripper

## What this skill is for

This skill automates cleaning system-generated tracking prefixes from tabular data. It accepts either an uploaded Excel file (`.xlsx`, `.xlsm`) with a user-specified sheet name or a standalone `.csv` file. It scans cell values across all columns for leading prefixes matching `^E\d{8}-` (an uppercase 'E' followed by exactly eight numeric digits and a hyphen, e.g., `E12345678-Dam Repair`), strips the prefix, keeps the remainder of the cell string intact (e.g., `Dam Repair`), and writes the cleaned sheet directly to a single CSV file.

## What this skill will NOT do

- This skill does not modify column header names; it operates strictly on cell values inside the table.
- This skill does not strip lowercase `e` prefixes (e.g., `e12345678-`) or prefixes with more or fewer than eight numeric digits (e.g., `E1234567-` or `E123456789-`).
- This skill does not alter cells that do not begin with the exact `^E\d{8}-` pattern.
- This skill does not re-export full multi-tab Excel workbooks; it outputs strictly a single `.csv` file to avoid timeout and memory constraints.
- This skill does not guess the sheet name when an Excel file is provided without one.

## Connectors and knowledge sources

This skill is instruction- and code-execution-based and does not require external connectors or document repositories. It processes user-uploaded files locally in the session environment using Python data libraries (`pandas`, `re`).

## How to do the task

### Step 1 — Ingest Input Data Efficiently
Inspect the uploaded file extension:

| File Type | Ingestion Protocol |
|---|---|
| **CSV File (`.csv`)** | Load the file directly using `pd.read_csv(dtype=str, keep_default_na=False)`. All columns are read as text to ensure leading zeros and string values are preserved. Do not prompt for a sheet name. |
| **Excel File (`.xlsx`, `.xlsm`)** | Read sheet names using low-memory inspection (`openpyxl.load_workbook(filename, read_only=True).sheetnames` or `pd.ExcelFile(path).sheet_names`). If the user specified a sheet name, load ONLY that sheet with all cells as text (`dtype=str`). If missing or invalid, stop and present the available sheets to the user. |

### Step 2 — Scan and Strip the Leading Prefix
Apply regular expression pattern matching and replacement across all cell values:

1. **Target Regular Expression Pattern:**
   Use the compiled regex: `r'^E\d{8}-'`
   - `^` asserts position at the start of the cell string.
   - `E` matches the literal uppercase letter 'E'.
   - `\d{8}` matches exactly eight consecutive decimal digits (0–9).
   - `-` matches the literal hyphen character.

2. **Transformation Logic:**
   For each cell in every column:
   - Check if the value is non-empty and starts with the target pattern.
   - If a match is found, strip the prefix and retain the remainder: `re.sub(r'^E\d{8}-', '', str(val))`.
   - If no match is found, retain the original cell value unchanged.
   - Keep empty cells, nulls, and blanks intact.

3. **Tracking Updates:**
   Record which columns contained matching prefixes and the total count of modified cells for the final report.

### Step 3 — Export Cleaned Dataset to CSV
Export the transformed table to a single CSV file:
- Output filename: `[source_name]_[sheet_name]_stripped.csv` (for Excel) or `[source_name]_stripped.csv` (for CSV).
- Use `index=False`, standard UTF-8 encoding, and standard CSV quoting.
- Present a concise transformation summary table and provide the file download.

## Output format

The final response must provide an execution summary table followed by the download file link:

| Metric / Parameter | Value |
|---|---|
| **Input File** | `[File Name]` |
| **Input Type** | `Excel (Sheet: [Name])` OR `Standalone CSV` |
| **Target Regex Pattern** | `^E\d{8}-` |
| **Columns Containing Cleaned Values** | `[List of modified columns]` |
| **Total Cells Updated** | `[Count of modified cells]` |
| **Output File** | `[Generated CSV Name]` |

Download: `[File download link / attachment]`

## Failure modes to watch for

- **Modifying column headers:** Ensure the regex substitution applies solely to cell content rows (`df.applymap()` or `df.map()`), leaving `df.columns` untouched.
- **Accidental mid-string replacements:** Ensure the regex begins with `^` so that occurrences in the middle or end of a string (e.g., `"Note regarding E12345678-task"`) are NOT stripped. Only leading prefixes at the start of the cell are removed.
- **Dropping leading zeros in other columns:** Reading the dataset with `dtype=str` guarantees that other identifier columns (e.g., zip codes, 6-digit IDs) do not lose leading zeros during ingestion or export.
- **Prompting for a sheet name on CSV files:** Never prompt the user for a sheet name when processing a CSV file.

## When the input is unclear

- **No file uploaded:** Prompt the user to upload either an Excel workbook (`.xlsx`, `.xlsm`) or a `.csv` file.
- **Excel file uploaded without sheet name:** List all sheet names present in the workbook and ask the user which one they want to process.
- **Excel sheet name mismatch:** Display the available sheet names found in the workbook and ask the user to verify their selection before proceeding.

## Why this design

Project management and financial software often prepend internal database keys (such as `E00124850-`) to task titles, descriptions, and deliverable names upon data export. These codes clutter human-readable reports and downstream presentation charts. Using strict `^E\d{8}-` regex scoping ensures precise removal of system tags without touching genuine project text or column headers.
