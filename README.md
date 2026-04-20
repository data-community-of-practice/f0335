# f0335
BDBSF Extract unique co-authors

# Unique Author Extractor

A Python script that extracts unique authors from the output of [Crossref Author Fetch](./README.md), normalises name variants, and produces a clean Excel file mapping each unique author to their associated DOIs.

## The problem

Crossref records the same author in different formats across publications. A single researcher might appear as:

- John J. Smith
- John J Smith
- John Smith
- JJ Smith
- Smith John
- Smith, John J.

This script recognises all of these as the same person and merges them into one entry.

## How name matching works

The algorithm uses a multi-step approach:

1. **Clean**: Strip periods, extra whitespace, and normalise hyphens.
2. **Parse**: Detect comma format ("Rossell, Susan") and flip to standard order. For non-comma names, try both "Given Family" and "Family Given" interpretations.
3. **Expand initials**: Concatenated initials like "SL" are split into individual letters for matching.
4. **Match**: Two names are considered the same person if they share a family name AND their given-name tokens are compatible — meaning one set is a subset of the other, with initials matching by first character (e.g., "S" matches "Susan").
5. **Merge**: The longest (most complete) name form is kept as the canonical version. A second merge pass catches clusters that only become matchable after initial expansion.

## Requirements

- Python 3.7+
- Library: `openpyxl`

```bash
pip install openpyxl
```

## Input file format

The script expects an Excel file with at minimum these two columns in the header row:

| Column | Description |
|---|---|
| `Publication_DOI` | The DOI for each publication |
| `Crossref_Authors` | Semicolon-separated author names (as produced by `crossref_author_fetch.py`) |

Rows where `Crossref_Authors` starts with `[DOI not found]`, `[No authors listed]`, or `[Error` are skipped automatically.

## Usage

### From a terminal / command prompt

```bash
python extract_unique_authors.py input.xlsx
```

Optionally specify an output filename:

```bash
python extract_unique_authors.py input.xlsx custom_output.xlsx
```

### From Spyder (IPython console)

```python
!python "E:\your\folder\extract_unique_authors.py" "E:\your\folder\input.xlsx"
```

### From a Jupyter notebook

```python
!python extract_unique_authors.py input.xlsx
```

## Output

The script produces an Excel file named `<input_name>_unique_authors.xlsx` with four columns:

| Column | Description |
|---|---|
| `Author_Name` | The canonical (most complete) form of the author's name |
| `Name_Variants` | Other name forms that were merged into this entry, semicolon-separated. Use this column to audit merges. |
| `DOI_Count` | Number of unique DOIs associated with this author |
| `DOIs` | All associated DOIs, semicolon-separated |

### Example output

| Author_Name | Name_Variants | DOI_Count | DOIs |
|---|---|---|---|
| John J. Smith | Smith John; Smith, John; JJ Smith; John J Smith; John Smith | 6 | 10.1111/aaa.001; 10.1111/aaa.002; ... |
| Jane A. Doe | Jane Doe | 3 | 10.1111/aaa.001; 10.1111/aaa.004; ... |

The output file includes a frozen header row and auto-filters for easy sorting and searching.

## Typical workflow

This script is designed to run after `crossref_author_fetch.py`:

```
1. Start with:  Publications_with_high_confidence.xlsx
                  (has Publication_DOI column)

2. Run:         python crossref_author_fetch.py Publications_with_high_confidence.xlsx
   Produces:    Publications_with_high_confidence_with_crossref_authors.xlsx
                  (adds Crossref_Authors column)

3. Run:         python extract_unique_authors.py Publications_with_high_confidence_with_crossref_authors.xlsx
   Produces:    Publications_with_high_confidence_with_crossref_authors_unique_authors.xlsx
                  (one row per unique author with their DOIs)
```

## Limitations and caveats

**False positives**: Purely algorithmic name matching cannot distinguish between two different researchers who share the same family name and compatible initials (e.g., two different people both named "J. Smith"). Always review the `Name_Variants` column to catch incorrect merges.

**False negatives**: The script may fail to merge names with significantly different spellings, transliterations, or major typos that go beyond formatting differences. For example, "Müller" vs "Mueller" would not be matched.

**Family name assumption**: The script assumes the family name is a single token (the last word in standard order). Multi-word family names without a comma (e.g., "Van Rheenen") are handled by trying both orderings, but unusual structures may occasionally be misinterpreted.

## Troubleshooting

| Problem | Solution |
|---|---|
| `Could not find 'Crossref_Authors' column` | The input file must have a column with this exact header. Run `crossref_author_fetch.py` first. |
| `Could not find 'Publication_DOI' column` | The input file must have a column named exactly `Publication_DOI`. |
| Unexpected merges in output | Check the `Name_Variants` column. If two different people were merged, they share a family name and compatible initials — this is a known limitation. |
| Missing authors in output | Authors from rows with error messages (`[DOI not found]`, etc.) are skipped. Check the input file for these. |

## License

This script is provided as-is for research and data management purposes.
