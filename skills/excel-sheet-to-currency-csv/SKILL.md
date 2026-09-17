---
name: excel-sheet-to-currency-csv
description: >
  Extracts a user-specified sheet from an uploaded Excel workbook, converts numeric decimal columns into standard USD currency ($#,##0.00) while ignoring ID fields, and outputs solely that transformed sheet as a lightweight CSV file to bypass workbook memory and timeout limits. Use this skill whenever a user asks to "export an Excel sheet to CSV with currency formatting," "convert an Excel tab to currency CSV," "format numbers to dollars and export single sheet," or reports "timeout/runtime errors processing large Excel files."
---

# Excel Single Sheet to Currency CSV Converter

## What this skill is for

This skill extracts and transforms a single user-specified sheet from large Excel workbooks directly into a CSV file with formatted currency columns (`$#,##0.00`). Large multi-tab workbooks frequently exceed session memory or compute timeouts when loaded and written back in full. By isolating only the required sheet and exporting to CSV, this skill delivers fast, memory-efficient transformations that eliminate runtime errors.

## What this skill will NOT do

- This skill does not re-export or re-save the full multi-tab Excel workbook (`.xlsx`/`.xlsm`); it outputs strictly a single `.csv` file.
- This skill does not modify columns whose header contains "ID" (case-insensitive) or whose numeric values are six-digit identifiers.
- This skill does not process unrequested sheets in the workbook.
- This skill does not make speculative assumptions about sheet names if the user fails to provide one or provides a name not present in the workbook.

## Connectors and knowledge sources

This skill is instruction- and code-execution-based and does not require external connectors or document repositories. It executes inside the session environment using Python data processing libraries (`pandas` or `openpyxl` in read-only mode).

## How to do the task

### Step 1 — Verify Sheet Name and Isolate Tab Efficiently
Inspect the workbook to read available sheet names without loading full worksheet payloads into memory (e.g., using `openpyxl.load_workbook(filename, read_only=True, keep_links=False).sheetnames` or `pd.ExcelFile(path).sheet_names`).
- Verify that the user's requested sheet exists in the workbook.
- If missing or not specified, immediately stop and present the list of available sheets to the user.
- Load ONLY the targeted sheet (e.g., via `pd.read_excel(file_path, sheet_name=target_sheet)`).

### Step 2 — Scan Columns for ID Exclusions and Decimals
Examine the columns of the isolated sheet:
1. Header exclusion: Check column names. If a column name contains `ID` (case-insensitive, e.g., "Employee ID", "Vendor_id", "ID_Code"), exclude it from formatting.
2. Value pattern exclusion: Check integer/numeric columns where values are uniformly six digits long (e.g., `100000` to `999999`). Exclude these as system or employee identifiers.
3. Decimal inclusion: Identify columns of numeric float type containing fractional/decimal values.

### Step 3 — Format Decimal Columns to Currency Strings
For each identified currency column, format the values into standard USD currency strings:
- Conversion format: `"${:,.2f}".format(x)` (producing outputs like `$1,250.50`).
- Ensure null/NaN values remain blank rather than printing `$nan`.
- Non-numeric or non-decimal columns must retain their original raw data and representations.

### Step 4 — Export Single Sheet to CSV
Write the transformed single sheet directly to a CSV file (e.g., `[sheet_name]_formatted.csv`) using UTF-8 encoding:
- Set `index=False` during export to prevent writing arbitrary row indices.
- Provide the generated CSV file download link and an execution summary to the user.

## Output format

The response must provide a concise status table followed by the download file:

| Metric / Parameter | Value |
|---|---|
| **Source Workbook** | `[File Name]` |
| **Extracted Sheet** | `[Sheet Name]` |
| **Columns Formatted as Currency** | `[List of formatted columns]` |
| **Excluded Columns (ID / Protected)** | `[List of skipped columns]` |
| **Export Output** | `[Generated CSV File Name]` |

Download: `[File download link / attachment]`

## Failure modes to watch for

- **Memory exhaustion from full workbook load:** Never use `openpyxl.load_workbook()` without `read_only=True` if inspecting sheets, and do not load all sheets at once. Only read the requested tab.
- **Handling NaN/empty cells:** Ensure empty cells do not convert to string literals like `"$nan"` or `"$None"`. Leave them as empty strings in the CSV output.
- **Breaking CSV commas:** When exporting to CSV, formatted currency values contain commas (e.g., `"$1,250.00"`). Ensure standard CSV quoting is maintained (`quoting=csv.QUOTE_MINIMAL` or default pandas CSV writing) so commas within currency strings do not split cells.

## When the input is unclear

- **No file provided:** Prompt the user to upload the Excel workbook.
- **No sheet specified:** Display the sheet names present in the uploaded file and prompt the user to select one.
- **Specified sheet does not match:** If the sheet name is not found, list all available sheet names and request confirmation before running extraction.

## Why this design

Exporting directly to CSV bypasses the high CPU and RAM overhead of parsing, rendering, and re-serializing complex Excel XML trees, styles, and calculation chains across multiple sheets. This guarantees that large datasets complete within execution timeout limits while cleanly isolating the user's required data.
