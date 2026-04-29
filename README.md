# ohio-brine-form15-extraction

AI-assisted workflow for converting Ohio Department of Natural Resources (ODNR) Surface Application Annual Report Form 15 PDFs into a structured, road-matched dataset of oilfield brine road spreading records.

## Overview

This repository contains a single Google Colab notebook (`oilfield_brine_workflow_cleaned_public.ipynb`) that documents the full processing pipeline used in the study:

> *From public records to public health access: An AI-assisted workflow for mapping oilfield brine surface application to support community engagement in Ohio.* 

The notebook is organized into modular stages, each controlled by a toggle in a `RUN_STEPS` dictionary so stages can be run independently during development or re-runs.

## Pipeline stages

| # | Stage | Purpose |
|---|---|---|
| 1 | **Setup & configuration** | Mount Google Drive, set paths, define config dictionaries |
| 2 | **PDF → images** | Rasterize multi-page Form 15 PDFs at 300 DPI |
| 3 | **Image enhancement** | Upscale (2×), grayscale, sharpen, and adaptive-threshold for OCR |
| 4 | **OCR** | Transcribe page images using **Qwen2.5-VL-7B-Instruct** (4-bit quantized); optional Nanonets branch |
| 5 | **Text → table parsing** | Parse OCR text into structured JSON records using an LLM via **LiteLLM** |
| 6 | **CSV merge** | Combine per-county/per-batch parsed CSVs into one dataset |
| 7 | **Cleaning & road matching** | Normalize road names and fuzzy-match to a **Census TIGER** master road list |

Output: a cleaned CSV of Form 15 spreading records joined to TIGER road references, ready for GIS mapping.

## Output schema

The parsing step returns one record per logical row in the Form 15 report, with the following fields:

| Field | Description |
|---|---|
| `source_file` | Filename of the source PDF |
| `report_year` | Reporting year of the Form 15 |
| `record_type` | `SPREADING` (Section II) or `SOURCE_WELL` (Section IV) |
| `operator_name` | Operator or hauler name |
| `source_well_name` | Well name (source records) |
| `source_api` | API / permit number (source records) |
| `app_date` | Date of application |
| `app_county` | County of application |
| `app_township` | Township of application |
| `app_road` | Road description as extracted |
| `app_volume_bbl` | Volume applied in barrels (1 BBL = 42 U.S. gallons) |
| `notes` | Free-text notes captured from the record |

After the cleaning and matching step, additional columns are added: `Matched_Township`, `Matched_Road`, `Township_Confidence`, `Match_Confidence`, and `Match_Note`.

## Running the notebook on Google Colab

The notebook is written to run on **Google Colab** with a **T4 GPU** (Colab Pro recommended for longer runs; L4 or A100 also supported).

**1. Open the notebook in Colab**

Upload `oilfield_brine_workflow_cleaned_public.ipynb` to Colab, or add an "Open in Colab" badge to the top of the notebook.
**2. Set the runtime to GPU**

`Runtime → Change runtime type → T4 GPU`

**3. Mount Google Drive and set project paths**

The notebook expects the following structure under your Google Drive project folder:

```
<PROJECT_FOLDER>/
└── Workflow/
    ├── RawData/<COUNTY_OR_BATCH_NAME>/   # input Form 15 PDFs
    ├── 1_Images/                         # page images
    ├── 2_ImagesEnhanced/                 # preprocessed images
    ├── 3_Extraction/                     # OCR text output
    ├── 4_DataParsing/                    # parsed CSVs (per county/batch)
    ├── 5_MergedData/                     # merged CSV across batches
    ├── 6_DataCleaning/                   # TIGER master road list
    └── 7_FinalData/                      # final cleaned, road-matched CSV
```

Update the `PROJECT_ROOT` and the placeholder paths in the config block at the top of the notebook (`<PROJECT_FOLDER>`, `<COUNTY_OR_BATCH_NAME>`, `<IMAGE_SUBFOLDER>`, etc.) to match your setup.

**4. Provide API credentials**

Add your API key through Colab's Secrets panel (🔑 in the left sidebar):

- `OPENAI_API_KEY` — for the LLM parsing step (called through LiteLLM)
- `NANONETS_API_KEY` — only if running the optional Nanonets OCR branch

**5. Provide the TIGER master road list**

Download Census TIGER/Line road shapefiles for Ohio, extract road names into the compound-key CSV format expected by `MATCHING_CONFIG`, and place the file at `6_DataCleaning/Tiger_roads_compound_key.csv`. Expected columns: `County_Name`, `Township_Name`, `FULLNAME`.

**6. Toggle stages and run**

Set the stages you want to run to `True` in the `RUN_STEPS` dictionary:

```python
RUN_STEPS = {
    "convert_pdfs": True,
    "enhance_images": True,
    "run_nanonets": False,
    "run_qwen": True,
    "parse_text": True,
    "merge_csvs": True,
    "clean_and_match": True,
}
```

Then run the final cell, which calls `main()`.

## Dependencies

The first cell of the notebook installs everything needed. Core packages:

- `pdf2image` + `poppler-utils` — PDF rasterization
- `transformers`, `accelerate`, `bitsandbytes`, `qwen-vl-utils` — Qwen2.5-VL inference
- `opencv-python-headless` — image enhancement
- `litellm` — unified LLM client for the parsing step
- `thefuzz` — fuzzy string matching for road name cleaning
- `pandas`, `openpyxl`, `python-dotenv` — data handling

## Data availability

Form 15 records are publicly available upon request from ODNR. A companion archive of the raw Form 15 PDFs used in this study (2010–2024) and the public-facing interactive map will be linked here once published:

- Raw Form 15 PDF archive: *(link TBD)*
- Public interactive map: *(link TBD)*

## Limitations

- **Source documents vary substantially.** Form 15 records are completed by many different local administrators and include handwritten entries, poor-quality scans, typed forms, and custom tables. OCR quality depends heavily on the source document.
- **Crossroad information is frequently missing**, so extracted records typically capture the main road only. Downstream mapping will overestimate road length when the full main road is mapped in place of a specific segment.
- **AI-derived data is not error-free.** Low-confidence road matches (below the configured threshold) should be reviewed manually against the original Form 15 records.
- **LLM costs and privacy.** The parsing step calls a hosted LLM through LiteLLM. Form 15 contains no personally identifiable information, but consider endpoint and data handling carefully before adapting this notebook to other document types.

## Citation

If you use this notebook, please cite:

```bibtex
@article{ohio_brine_form15_2026,
  title   = {From public records to public health access: An AI-assisted workflow for mapping oilfield brine surface application to support community engagement in Ohio},
  author  = {<Authors>},
  journal = {<Journal>},
  year    = {2026},
  note    = {Manuscript in preparation}
}
```

## License

MIT License — see [LICENSE](LICENSE) for details.

## Acknowledgments

Developed at **The Ohio State University**. We thank the Ohio Department of Natural Resources for providing Form 15 records, and acknowledge the open-source communities behind Qwen2.5-VL, LiteLLM, thefuzz, and the U.S. Census TIGER/Line program.
