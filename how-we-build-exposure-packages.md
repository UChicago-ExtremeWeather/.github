# How we build exposure packages

This document describes the standard pattern for building exposure data packages.
Every exposure (hurricanes, tornadoes, heat, wildfire, etc.) follows the same
structure so the packages are consistent, citable, and maintainable as students
come and go.

## The model

Each exposure has **one R package** plus **data files hosted on Zenodo**, the latter
of which is only necessary if the processed data is too large to live on GitHub. 

| Role | Name | Where it lives | Contents |
| ---- | ---- | -------------- | -------- |
| Code | `{exposure}exposure` | GitHub repo in the lab org; archived to Zenodo on each release | R functions that explore, summarize, and map the data |
| Data | `{exposure}exposuredata` | A Zenodo record (data files only) | The processed exposure data, as Parquet (and an open gridded format for raster data) |

- For **small** data, processed data can stay on GitHub. This will then be archieved to Zenodo upon
- release to create a stable DOI.
- For **large** data, processed data will be hosted on a Zenodo (e.g. heatexposuredata) and the R
package downloads only the slice a user requests from the Zenodo record and caches
it locally (e.g. heatexposure). This will result in two Zenodo records.

### Why are we doing it this way? 

- **Size.** Processed exposure data can be very large (heat is daily values for
  every ZCTA and county in the country, 1980 to present, which is hundreds of
  millions of rows). That cannot live in a repo or an installed R package. Hosting
  it on Zenodo and fetching slices keeps the package small and installable.
- **Stablitiy.** Zenodo creates a stable DOI and citation for users.  

### Naming convention

- R package (code), and associated Zenodo record: `{exposure}exposure`
- Zenodo data record: `{exposure}exposuredata`

**Exception for extending existing work.** If you are extending someone else's
package (as we did with hurricanes, building on Brooke Anderson's
`hurricaneexposuredata`) and their code expects a data source with a specific name,
keep that name so their code can find your data. Provenance is then handled through
attribution and citation rather than a different name. For any exposure you
originate yourself there is no collision, so use the standard names above. However, future
instances of this are unlikely.

## Data format and storage - more details

### Format: Parquet

Store processed tabular data as **Parquet**, not CSV or `.rda`. Parquet lets the R
package read only the rows it needs (via the `arrow` package) instead of loading
the whole dataset, and it compresses well. For gridded/raster data, also deposit an
open gridded format (NetCDF) alongside the Parquet for cross-language users.

**Sort the data by year, then by geography (ZCTA or county), before writing it.**
This makes year-range and geography queries read contiguous chunks of the file, so
slice reads stay fast even on a single large file.

### How many files

Default to **organizing by spatial level**: for example, ZCTA file and a county file, each
containing all years. If that is too big, you can slice by 10 or 5 year buckets for each spatial
level (e.g. ZCTA_2015-2020).

Only split further (e.g. one file per year per level) if the per-level file gets so
large that forcing a non-R user to download the whole thing is unreasonable, or if
you approach Zenodo's per-record limits. Do not split a single dataset across
multiple Zenodo records just to dodge the size limit; Zenodo treats that as misuse.

### Zenodo size limits (know these before you start)

- Per record: **50 GB and at most 100 files** by default.
- A one-time increase to **200 GB** per record can be requested (file cap stays 100).
- Each account has an additional 150 GB allowance to distribute across records.

Before building anything, estimate your size: write out **one year, both spatial
levels, as compressed Parquet, and measure it**, then multiply by the number of
years. That tells you immediately whether you are a comfortable single-record job or
need to request a quota increase / contact Zenodo.

## Clariying: two Zenodo records 

Each exposure ends up with **two Zenodo records, under the one shared lab Zenodo
account** (excepting when processed data is very small). They are different records, 
and they are created two different ways.

1. **The data record (`{exposure}exposuredata`).** When needed, you can create this **manually**:
   you log into the *lab Zenodo account,* make a new upload, and drag your processed
   Parquet files into it. This record holds the actual data and is what the R
   package downloads slices from. Add a README file. 

3. **The code archive (the `{exposure}exposure` package).** This is created
   **automatically** by Zenodo's GitHub integration: when you cut a Release of the
   package repo on GitHub, Zenodo snapshots the repo and mints a DOI for the code.
   You do not upload anything by hand for this one. Wait to do this until you are done.

## Step-by-step for students

Do these in order. The data record comes **first** because the R package needs to
know where to fetch from.

### Step 1: Ask Kate to create your code repo

- Only org **owners** can create repositories.
- Kate copies the `exposure_template` repo for you and gives you **Write** access to
  push your work.
     - Note that that repo will also include this list of instructions for students so
       you have it on hand and can keep notes. 

### Step 2: Estimate size and decide file layout

- Discuss with Kate.
- If still unsure, you could process one year at both spatial levels, write as
   compressed Parquet, measure it,and project the full size.
- **If the projected processed data is larger than ~1GB, use the Zenodo-hosted
  Parquet pattern below** (true for essentially all exposures except very small
  event datasets like tornadoes, which bundle their data in the package instead).
- Decide your file layout: default is one ZCTA file and one county file. Only go
  per-year if size forces it (see "How many files" above).

### Step 3: Process the data and generate the Parquet files

1. Write the processing code in `data-raw/` (this is part of the repo, separate from
   the user-facing functions). It should load the raw source data, processes it, and writes
   the processed output as Parquet at the appropriate spatial levels (e.g. ZCTA and county). Sort by
   year, then geography, before writing.
2. Run it locally to produce the actual Parquet files. Dump the raw data once you
   have the processed output.
3. Confirm the files match your size estimate from Step 1 and your chosen file
   layout (default: one ZCTA file, one county file).

### Step 4: Create the data record on Zenodo

1. Log into the **shared lab Zenodo account.** Only the owning account can publish
   new versions of a record later, and ownership cannot be easily transferred,
   so personal accounts are not allowed.
3. Create a **new upload**.
4. Upload the processed Parquet files you generated in Step 3 (ZCTA file, county
   file; plus NetCDF if the data is gridded).
5. Fill in metadata: title `{exposure}exposuredata`, authors, description, license
   GPL-2.0+, and associate it with the lab Community.
6. **Publish (or reserve the DOI first if you want it in hand while you finish the
   package).** You will need this DOI/download location in the R package so
   `get_data()` knows where to fetch.
7. Note the DOI and the file download URLs; you will wire the resolved data location
   into the package in Step 5.

### Step 5: Build the access functions and wire them to Zenodo

1. Write the user-facing functions following the template. Every exposure package
   exposes the same interface: `get_data(geography, years)` returns an exposure
   table, plus mapping/summary functions on top.
2. `get_data()` should: check a local cache directory
   (`tools::R_user_dir("{exposure}exposure", "cache")`), download the needed Parquet
   from the Zenodo data record (Step 4) if not already cached, then use `arrow` to
   read **only** the requested geography and years. Repeat calls read from cache.
3. **Add the local data directory to `.gitignore`.** This is a guardrail so you never
   accidentally commit a multi-GB Parquet file into the repo. The data is not
   supposed to be in the repo at all; gitignore just stops `git add .` from sweeping
   it in.
4. Confirm the round trip works: install the package fresh in a clean R session and
   run `get_data()` for a small geography and year range. It should download from
   Zenodo, cache, and return the slice. This is the check that the package and the
   Zenodo record actually talk to each other.

### Step 6: Release the code and get its DOI

1. Make sure Zenodo's GitHub integration is switched on for the repo (under the lab
   Zenodo account's GitHub settings).
2. Cut a **Release** on GitHub (e.g. `v0.1.0`). Zenodo automatically archives the
   repo and mints a DOI for the code package.
3. Put **both** DOIs in the README and `inst/CITATION`: the code-package DOI and the
   data-record DOI.

## How users access the data

Users never touch the data files directly. They install the package and call a
function:

```r
devtools::install_github("UChicago-ExtremeWeather/{exposure}exposure")
library({exposure}exposure)

# downloads only the requested slice from Zenodo on first call, caches locally
dat <- get_data(zcta_list = c(60637, 60615), year_range = 2010:2015)
```

Non-R users go to the `{exposure}exposuredata` Zenodo landing page and download the
Parquet files directly. Parquet opens in Python, QGIS, and many other tools, so this
does not lock anyone into R.

## Hosting and distribution

- The **code package** lives in the `UChicago-ExtremeWeather` organization on GitHub
  and is public. Users install with
  `devtools::install_github("UChicago-ExtremeWeather/{exposure}exposure")`.
- The **data** lives on Zenodo under the shared lab account.
- We do **not** publish to CRAN. GitHub + Zenodo gives us accessibility and citable
  DOIs without the CRAN maintenance burden.

## Getting started: templates

Start from the lab templates:

- **Code package:** use the `exposure_template` repository ("Use this template" on
  GitHub). It contains a README, a pre-filled `DESCRIPTION`, the correct folder
  structure, a `get_data()` stub wired for Zenodo + `arrow` + local caching, a
  placeholder `inst/CITATION`, the GPL license, and a student to-do checklist that
  walks through every step above.

## Licensing

All exposure packages use **GPL (>= 2)**.

- For packages **derived** from existing GPL work (e.g. anything built on Anderson's
  hurricane packages), GPL is required because GPL is copyleft: derivatives must stay
  GPL-compatible.
- For packages you **originate**, we still use GPL (>= 2) for consistency across the
  org.

Keep the `LICENSE` file in every repo and set `License: GPL (>= 2)` in `DESCRIPTION`.

## Citation and attribution

Every package includes an `inst/CITATION` file so `citation("{exposure}exposure")`
tells users exactly what to cite.

- Cite **both** the code package (its DOI) and the data record (its DOI).
- If a package **extends prior work** or uses data from elsewhere, credit the
  original authors:
  - add them to `Authors@R` in `DESCRIPTION` with role `ctb` (contributor) or `cph`
    (copyright holder), and
  - include them in `inst/CITATION` so users cite both your work and the original.

Standard citation line for READMEs:

> When using this data, please cite both this package (DOI: ...), the data record
> (DOI: ...), and, where applicable, the original work by [original authors].
