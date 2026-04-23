# f0335
# Normalise & Merge Crossref Authors

A Python script that reads the structured author metadata produced by [f0334](https://github.com/data-community-of-practice/f0334), deduplicates author appearances across publications, and outputs a single JSON file of researcher entities with merged name variants, publication lists, and deduplicated affiliations.

Entries that cannot be auto-merged with confidence are flagged as `similar_to` links for manual review.

## Pipeline context

```
f0334  →  Crossref_AuthorMetadata.json
f0335  →  Normalised_Authors.json
```

`f0334` fetches one author record per appearance per DOI. The same researcher can appear dozens of times across publications, with slightly different name forms (`"Jane L. Doe"`, `"J.L. Doe"`, `"Jane Doe"`). This script collapses those appearances into one researcher entity per person.

## How it works

The script runs five sequential steps:

### Step A — Fix surname-first entries
Detects records where given and family name are swapped (e.g. `given="Doe"`, `family="Jane L."`) and corrects the order before any matching begins. Only high-confidence cases are fixed: those where the `family` field contains a period but the `given` field does not.

### Step B — ORCID-based merge
Any two appearances that share the same ORCID are merged unconditionally. This is the most reliable merge signal.

### Step C — Normalised-name merge
Appearances without ORCID are merged if their **merge key** is identical. The merge key is constructed by:
- Stripping periods (`"J."` → `"j"`)
- Normalising whitespace
- Expanding compressed initials (`"JL"` → `"j l"`)
- Keeping full words intact

This means `"Jane L. Doe"` and `"Jane L Doe"` auto-merge, and `"J.L. Doe"` and `"JL Doe"` auto-merge — but `"Jane L. Doe"` and `"J.L. Doe"` do **not** auto-merge (initial-to-full is only flagged, not assumed). Non-ORCID appearances whose merge key matches an existing ORCID cluster are absorbed into that cluster.

### Step D — Build researcher nodes
Each cluster of merged appearances is consolidated into one researcher object. The most complete name form (longest given name) is chosen as the canonical name. Affiliations across all appearances are deduplicated using token-based Jaccard similarity (threshold: 0.80), keeping the longest version and filling in any missing ROR or place data from variants.

### Step E — Flag similar pairs
Researchers that were not auto-merged but may still be the same person are linked via `similar_to`. Cases flagged include:

| Case | Example |
|------|---------|
| Initial vs full name | `"J.L. Doe"` ~ `"Jane L. Doe"` |
| Missing name parts | `"Jane Doe"` ~ `"Jane L. Doe"` |
| Spelling variant (≤2 edits) | `"John Mathew Smith"` ~ `"John Matthew Smith"` |
| Hyphenated vs space-separated | `"McCarthy-Jones"` ~ `"McCarthy Jones"` |
| Surname-first swap | `"Doe Jane"` ~ `"Jane Doe"` |

## Output

One file: **`Normalised_Authors.json`** — a JSON array of researcher objects.

### Researcher object

```json
{
  "id": "3f2a1b4c-...",
  "given": "Jane Louise",
  "family": "Doe",
  "full_name": "Jane Louise Doe",
  "orcid": "0000-0001-2345-6789",
  "name_variants": ["J.L. Doe", "Jane Doe", "Jane L. Doe", "Jane Louise Doe"],
  "publications": ["10.1111/aaa.001", "10.1111/aaa.007"],
  "affiliations": [
    {
      "name": "Department of Science, Example University",
      "ror": "https://ror.org/0153tk833",
      "place": "Melbourne, Australia"
    }
  ],
  "similar_to": [
    {
      "id": "9e8d7c6b-...",
      "full_name": "J. Doe",
      "orcid": null
    }
  ],
  "merge_confidence": "orcid"
}
```

### Fields

| Field | Description |
|-------|-------------|
| `id` | UUID assigned to this researcher entity. |
| `given` | Best available given name (longest form seen). |
| `family` | Family name. |
| `full_name` | `given + family` combined. |
| `orcid` | ORCID identifier, or `null`. |
| `name_variants` | All distinct name strings seen across appearances, sorted. |
| `publications` | Deduplicated list of DOIs this researcher appears on, sorted. |
| `affiliations` | Deduplicated affiliation objects. Each has `name`; optionally `ror` and `place`. |
| `similar_to` | List of other researchers flagged as possibly the same person (for manual review). Each entry has `id`, `full_name`, and `orcid`. |
| `merge_confidence` | `"orcid"` if the cluster was anchored by an ORCID match; `"name"` otherwise. |

## Requirements

- Python 3.7+
- No external libraries (standard library only)

## Usage

```bash
python f0335.py
```

By default the script looks for `Crossref_AuthorMetadata.json` in the current directory, then in the script's own directory.

Specify paths explicitly:

```bash
python f0335.py path/to/Crossref_AuthorMetadata.json --output path/to/Normalised_Authors.json
```

### From Spyder or Jupyter

```python
!python "E:\your\folder\f0335.py"
```

## Console output

```
Input:  /path/to/Crossref_AuthorMetadata.json
Output: /path/to/Normalised_Authors.json

DOIs in input: 412
Total author appearances: 3847

Step A: Fixing surname-first entries...
  Fixed: 3

Step B: Merging by ORCID...
  ORCID clusters: 891

Step C: Merging non-ORCID by normalised name...
  Total clusters: 2104

Step D: Building researcher nodes...
  Researchers: 2104

Step E: Finding similar pairs...
  Similar pairs: 47

============================================================
NORMALISATION SUMMARY
============================================================
Input appearances:          3847
Deduplicated researchers:   2104
  With ORCID:               891
  Without ORCID:            1213
  Multi-publication:        318
  Merged name variants:     204
  Flagged similar:          89 (47 pairs)

Top merged researchers:
  Jane Louise Doe [ORCID: 0000-0001-2345-6789]
    Variants: ['J.L. Doe', 'Jane Doe', 'Jane L. Doe', 'Jane Louise Doe']
    Pubs: 6
  ...

Similar pairs (review manually):
  "J. Doe"  ~  "Jane Doe"
  ...

Saved: /path/to/Normalised_Authors.json
```

## Reviewing similar pairs

The `similar_to` field lists researcher IDs and names that could not be automatically merged. To decide whether to merge them:

1. Compare `name_variants` of both entries — do the name forms overlap in a way that suggests one person?
2. Check `affiliations` — do they share an institution?
3. Check `publications` — do they co-appear on papers, or appear in different research areas?

Manual merges are outside the scope of this script and should be handled in a downstream step.

## Limitations

- **False positives (over-merging)**: Two different researchers with the same family name and identical initials will be merged. Review `name_variants` and `publications` to catch this.
- **False negatives (under-merging)**: Significantly different spellings or transliterations (e.g. `"Müller"` vs `"Mueller"`) are not matched and will appear as `similar_to` candidates at best.
- **ORCID coverage**: Not all Crossref records include ORCID. The majority of merges rely on name normalisation, which is inherently heuristic.

## License

This script is provided as-is for research and data management purposes.
