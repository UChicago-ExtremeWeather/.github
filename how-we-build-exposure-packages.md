# How we build exposure packages

This document describes the standard pattern for building exposure data packages. 
Every exposure (hurricanes, tornadoes, heat, wildfire, etc.) follows the same structure 
so the packages are consistent, citable, and maintainable as students come and go.

## Structure of packages

### The two-package pattern

Each exposure is built as **two R packages** and then we will create a stable DOI via
Zenodo which also makes it easier for non-R users to download data. 

| Role | Package name | Contents |
| ---- | ------------ | -------- |
| Data | `{exposure}exposuredata` | datasets only, no functions |
| Code | `{exposure}exposure` | functions that explore, summarize, and map the data |

For example: `tornadoexposuredata` + `tornadoexposure`,
`heatexposuredata` + `heatexposure`.

### Why split them? Why not just use one package? 

1. **Size.** Exposure datasets are large (tens of MB). Keeping data separate means
   updating a function doesn't force everyone to re-download the data.
2. **Citation.** The dataset gets its own version and DOI, so it can be cited
   independently of the code.
3. **Consistency.** Every exposure looks the same, which makes the whole org
   legible and easier to maintain.

The code package declares the data package as a dependency, so installing the code
package makes the data available automatically.

### Naming convention

- Data package: `{exposure}exposuredata`
- Code package: `{exposure}exposure`

**Exception for extending existing work.** If you are extending someone else's
package (as we did with hurricanes, building on Brooke Anderson's
`hurricaneexposuredata`), and their *code* package expects a *data* package with a
specific name, you must keep that same name so their code can find your data. In
that case, the provenance is made clear through attribution and citation rather
than through a different name. For any exposure you originate yourself, there is no
collision, so just use `{exposure}exposuredata` / `{exposure}exposure`. Hoewver, future 
instances of this are unlikely 

### Hosting and distribution

- **All packages live in the `UChicago-ExtremeWeather` organization on GitHub** and
  are public.
- Users install with `devtools::install_github("UChicago-ExtremeWeather/<pkg>")`.
- We do **not** publish to CRAN. GitHub + Zenodo gives us accessibility and a
  citable DOI without the CRAN maintenance burden.

## Getting started

### Templates
Start from the lab templates:

- **Data package:** use the `exposuredata_template` repository
  ("Use this template" on GitHub).
- **Code package:** use the `exposure_template` repository.

Each template contains a README, a pre-filled `DESCRIPTION`, the correct folder
structure, a placeholder `inst/CITATION`, the GPL license, and a **student to-do
checklist** that walks you through every step above.

### Who creates repos

Only organization **owners** can create repositories (this is a deliberate setting
so repos can't be accidentally created or deleted). Ask Kate to create your repo - she will
make a copy of both templates and you'll be given **Write** access to push your
work.

## Citations and finalizing your package

### Zenodo and DOIs (required)

Every package (both data and code) is archived on **Zenodo** to get a permanent
DOI. This makes the work citable and guarantees it persists even if a repo moves.

**Use the shared lab Zenodo account, not personal accounts.** This is essential
for stability: on Zenodo, only the account that *owns* a record can publish new
versions of it, and there is no way to transfer record ownership between accounts
after the fact. If a student publishes under their personal account and then
leaves, the lab cannot update that record. So all archiving goes through one
shared, lab-controlled Zenodo account (tied to a lab email, not a personal one),
which owns every record permanently. Optionally, also create a lab **Community**
on Zenodo to group and brand all the lab's records.

To archive a package:

1. Log into the **shared lab Zenodo account** (not your personal account).
2. Create a GitHub **Release** (e.g. `v0.1.0`) and download the release `.zip`.
3. In the lab account, upload the release files, fill in metadata (title, authors,
   description, license = GPL-2.0+), and associate with the lab community.
4. Publish. Zenodo mints a DOI owned by the lab account.
5. Put the DOI in the package README and `inst/CITATION`.

When data updates later, publish a **new version** of the same Zenodo record (from
the lab account); each version gets its own DOI under one permanent concept DOI.

Zenodo also stores the actual files, so **non-R users** can download the data
without R. Where cross-language use matters, also deposit the data in an open
format (CSV, Parquet, or NetCDF for gridded data) alongside the package.


### Licensing

All exposure packages use **GPL (>= 2)**.

- For packages **derived** from existing GPL work (e.g. anything built on Anderson's
  hurricane packages), GPL is required because GPL is copyleft: derivatives must
  stay GPL-compatible.
- For packages you **originate**, we still use GPL (>= 2) for consistency across the
  org.

Keep the `LICENSE` file in every repo and set `License: GPL (>= 2)` in `DESCRIPTION`.

### Citation and attribution

Every package includes an `inst/CITATION` file so that `citation("<pkg>")` tells
users exactly what to cite.

- Cite **both** the code package and the data package.
- If a package **extends prior work**, or uses data from elswhere, be sure to credit the original authors:
  - add them to `Authors@R` in `DESCRIPTION` with role `ctb` (contributor) or
    `cph` (copyright holder), and
  - include them in `inst/CITATION` so users cite both your extension and the
    original.

Standard citation line for READMEs:

> When using this data, please cite both this package (DOI: ...) and the original
> work by [original authors].

