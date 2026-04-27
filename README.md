# FEC Linkage
> **Note:** This repository contains project documentation only. Code and data are available upon request for academic and research purposes. Please contact me via [LinkedIn]([your-linkedin-url](https://www.linkedin.com/in/alireza-mahmoodzadeh-70763564/)) or email.


**FEC Linkage** is a Python-based pipeline for linking individuals from corporate datasets (USPTO inventors, ExecuComp executives, BoardEx directors) to FEC political contribution records. This project was designed and implemented end-to-end by the author, including data cleaning logic, matching heuristics, quality filters, and diagnostics.

The pipeline follows a modular workflow:
1. **Raw Data Construction** – extracts and combines source files into standardized raw datasets
2. **Preprocessing** – cleans names, organizations, and creates matching variables
3. **Name Matching** – identifies candidate pairs based on personal name similarity
4. **Company Matching** – validates matches using employer/organization name comparison
5. **Output Generation** – produces final linkage files and statistics reports

---

## Project Structure

```
FEC-Linkage-Codes/
├── README.md                   # Project documentation
├── config.yaml                 # Configuration file for paths and parameters
├── run_all.py                  # Pipeline entry point (command-line script)
│
├── src/                        # Source code
│   ├── preprocessing/          # Data preparation modules
│   │   ├── BoardEx_alias_prep.py       # Build unConsolidated BoardEx dataset for ExecuComp alias matching
│   │   ├── FEC_raw.py                  # Construct FEC raw dataset from yearly files
│   │   ├── FEC_preprocessing.py        # Clean FEC contributor names and employers
│   │   ├── USPTO_raw.py                # Construct USPTO raw dataset from patent files
│   │   ├── USPTO_preprocessing.py      # Clean USPTO inventor names and organizations
│   │   ├── BoardEx_raw.py              # Construct BoardEx raw dataset
│   │   ├── BoardEx_preprocessing.py    # Clean BoardEx director names and companies
│   │   ├── ExecuComp_raw.py            # Construct ExecuComp raw dataset
│   │   ├── ExecuComp_preprocessing.py  # Clean ExecuComp executive names and companies
│   │   └── Orbis_auxiliary.py          # Build subsidiary-parent mapping from Orbis data
│   │
│   ├── matching/               # Entity matching modules
│   │   ├── USPTO_FEC_name_matching.py      # USPTO–FEC name matching
│   │   ├── USPTO_FEC_company_matching.py   # USPTO–FEC company matching
│   │   ├── BoardEx_FEC_name_matching.py    # BoardEx–FEC name matching
│   │   ├── BoardEx_FEC_company_matching.py # BoardEx–FEC company matching
│   │   ├── ExecuComp_FEC_name_matching.py  # ExecuComp–FEC name matching
│   │   └── ExecuComp_FEC_company_matching.py # ExecuComp–FEC company matching
│   │
│   └── output/                 # Output generation modules
│       ├── Final_linkage_outputs.py    # Write final filtered linkage CSV files
│       └── FEC_linkage_statistics.py   # Generate statistics Excel report
│
├── data/                       # Data storage
│   ├── raw/                    # Raw input files (user-provided)
│   │   ├── FEC/                    # FEC contribution files (indivXX.zip)
│   │   ├── USPTO/                  # USPTO patent files (granted + applications)
│   │   ├── BoardEx/                # BoardEx profile and employment files
│   │   ├── ExecuComp/              # ExecuComp executive files
│   │   └── Orbis/                  # Orbis subsidiary-parent mapping
│   ├── intermediate/           # Intermediate processing files
│   │   ├── FEC/
│   │   ├── USPTO/
│   │   ├── BoardEx/
│   │   ├── ExecuComp/
│   │   └── Orbis - Auxiliary/
│   └── output/                 # Linkage output files
│       ├── USPTO_g_FFC_linkage_raw.csv     # USPTO granted patents–FEC linkage
│       ├── USPTO_pg_FFC_linkage_raw.csv    # USPTO patent applications–FEC linkage
│       ├── BoardEx_FFC_linkage_raw.csv     # BoardEx–FEC linkage
│       └── ExecuComp_FFC_linkage_raw.csv   # ExecuComp–FEC linkage
│
└── reports/                    # Reports and documentation
    ├── Codebook.xlsx                   # Variable definitions for output files
    ├── FEC Linkage Stats.xlsx          # Match statistics by dataset
    └── FEC_Linkage_Stats_README.md     # Methodology documentation for statistics
```

---

## Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/Mahmoodzadeh90/FEC-Linkage-Codes.git
   cd FEC-Linkage-Codes
   ```

2. Install dependencies:
   ```bash
   pip install pandas pyyaml openpyxl python-docx
   ```

3. Update `config.yaml` with your local paths.

---

## Usage

### Run the complete pipeline
```bash
python run_all.py
```

### Run specific stages
```bash
python run_all.py --stages raw,preprocessing
python run_all.py --stages statistics
```

### Run specific datasets
```bash
python run_all.py --datasets FEC,USPTO
python run_all.py --datasets BoardEx --stages name_matching,company_matching
```

### Other options
```bash
python run_all.py --list          # Show available datasets and stages
python run_all.py --dry-run       # Show execution plan without running
python run_all.py --skip ExecuComp  # Run everything except ExecuComp
```

---

## Configuration

All paths and parameters are controlled via `config.yaml`. Key settings include:

- `base_path`: Root directory for all data files
- `datasets`: Input/output paths and parameters for each dataset
- `reports`: Output path for statistics and reports

---

## Outputs

### Linkage Files
Final linkage files are saved to `data/output/` as CSV files. Each row represents a matched pair between a corporate individual and an FEC contribution record.

### Statistics Report
`reports/FEC Linkage Stats.xlsx` contains match counts organized by:
- **Columns**: Name-matching method (exact, alias, abbreviation)
- **Rows**: Company-matching method (exact, dictionary, Orbis, substring, prefix)

See `reports/FEC_Linkage_Stats_README.md` for detailed methodology.

### Codebook
`reports/Codebook.xlsx` contains variable definitions for all output files.
---

## Limitations & Usage Guidance

### Data Quality Notes

#### Multi-Affiliation Duplicates (Expected Behavior)
The linkage files contain intentional duplicates when one individual holds multiple positions:
- **BoardEx**: A director serving on multiple boards will have separate rows for each board position matched to the same donation
- **ExecuComp**: An executive at multiple companies will appear multiple times
- **USPTO**: An inventor with patents at multiple assignees will have multiple records

**Recommendation**: Deduplicate by `(PersonID, SUB_ID)` when counting unique person-donation pairs.

| Dataset | Person Identifier | Company Identifier |
|---------|------------------|-------------------|
| BoardEx | `DirectorID` | `CompanyID` |
| ExecuComp | `co_per_rol` | (embedded in identifier) |
| USPTO | `Ali_identifier` | (embedded in identifier) |

#### False Positive Rate
Approximately 1-2% of duplicate SUB_IDs may match different individuals with similar names. This occurs when:
- Common names (Smith, Johnson, Jones) match multiple people
- Short FEC employer names provide weak verification

**Recommendation**: Use `exact_match=1` filter for highest-precision subset.

#### FEC Employer Name Quality
The FEC employer field is self-reported and sometimes contains incomplete data. The pipeline filters out matches where `FEC_org` is ≤2 characters unless `exact_match=1`, but some low-quality employer names remain.

#### USPTO Company Name Variations
The same inventor may appear with multiple `Ali_identifier` values due to company name variations in USPTO data (e.g., "cognex" vs "cognextechnologyinvestment"). This creates redundant records rather than false positives.

### Match Confidence Tiers

Matches are identified through multiple methods with varying confidence levels:

| Match Type | Description | Confidence |
|------------|-------------|------------|
| `exact_match=1` | Exact company name match | Highest |
| `exact_match_dic=1` | Match after dictionary expansion | High |
| `Orb_dyn_assignment=1` | Matched via Orbis subsidiary mapping | High |
| `sub_match_*=1` | Substring match (one name contains the other) | Medium |
| `3STR_match=1` | First 3 characters match | Lower |

### Recommended Filters

**High Precision** (fewer matches, higher confidence):
```python
df_high_precision = df[df['exact_match'] == 1]
```

**Balanced** (default output):
```python
# Use the output files as-is
```

**Research-Specific**:
```python
# Deduplicate for unique person-donation pairs
df_unique = df.drop_duplicates(subset=['DirectorID', 'SUB_ID'])  # BoardEx example
```

### Quality Reports

The pipeline generates two diagnostic reports in `reports/`:

| Report | Description |
|--------|-------------|
| `FEC Linkage Stats.xlsx` | Match counts by name and company matching method |
| `FEC Linkage Diagnostics.xlsx` | Quality metrics, duplicate analysis, and false positive examples |

Review `FEC Linkage Diagnostics.xlsx` before using the data to understand the quality characteristics of each dataset.


## Access

This project is part of ongoing PhD research. The full codebase and dataset are available upon request for academic and research purposes. Please reach out via [LinkedIn]([your-linkedin-url](https://www.linkedin.com/in/alireza-mahmoodzadeh-70763564/)) or email to request access.
