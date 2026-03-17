# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Replication package for Moretti (2021) "The Effect of High-Tech Clusters on the Productivity of Top Inventors" (AER). All code is Stata, converted from `.do` to `.txt` for AI readability. The purpose of this repo is to audit the code for errors — the README notes ten major errors to find.

The paper and appendix are in `readings/`. Moretti's original replication README is `code/README.pdf`.

## Code Format

All `.txt` files in `code/` are Stata `.do` files renamed for easy uploading. When reading them, treat as Stata syntax. The original code was written for Stata 13.

## Pipeline Architecture

`code/MAIN.txt` is the master script. It sets `global main` to a root path and sequentially `include`s all other scripts. The pipeline has three phases:

**1. Data Preparation** (scripts reference `$main/data/` and `$main/data2/`):
- `create_COMETS_Patent_ExtractForEnrico.txt` — reads/reformats COMETS patent data
- `read_apat.txt` — reads NBER patent database (`apat63_99.txt`)
- `read1.txt` — reads BEA Input-Output Tables
- `read.txt` — reads county characteristics (`tva1.dta`) and BEA crosswalk
- `data_3.txt` — **key file**: merges sources into the main working dataset
- `data_4.txt` — creates alternative dataset for Appendix Table A7

**2. Derived Variables** (scripts reference `$main/data2/`):
- `iv_new.txt` — creates instrumental variables (Kodak shock)
- `citations_received.txt`, `citations_made.txt` — citation measures
- `density_team.txt`, `density14_W.txt`, `density_14_3_reshaped.txt` — cluster density measures

**3. Analysis** (regression scripts produce tables/figures):

| Script(s) | Output |
|---|---|
| `reg3.txt`, `reg27.txt` | Table 2 (cols 1-4), Table 4 (cols 1-4), Fig 2 (bottom), Fig 3 |
| `reg22.txt` | Table 2 (col 5), Table 4 (col 5) |
| `reg.txt` | Table 3, Table 6 (rows 1-8), Table 8 (top), Fig 5, Fig A1 |
| `reg2.txt` | Table 1, 10, 11, 12 (col 1), Table A1, Fig 1, Fig 4 |
| `reg11.txt`, `reg13.txt` | Table 5 (IV results; F-stats in log marked "XXX FTEST") |
| `reg20.txt` | Table 6 (cols 9-10), Table 7, Table 8 (bottom), Tables A3, A5, A8 |
| `reg23.txt` | Fig 6 |
| `reg29.txt` | Table 12 (col 2) |
| `reg5.txt` | Table A2 |
| `reg36.txt` | Table A4 |
| `reg31.txt`, `reg30.txt` | Table A6 |
| `reg4.txt`, `reg34.txt` | Table A7 |
| `kodak_stock.txt` | Fig 2 (top panel) |

## Output Locations

Output directories are `tables/`, `tables2/`, `tables3/`, `tables4/`, `tables11/`, `tables20/`, `tables22/`, `tables23/`, `tables30/`, `tables31/`, `tables34/`, `tables36/`. Directory names do **not** correspond to table numbers — see `code/README.pdf` pages 9-12 for the exact mapping. Tables are in `.tex` and `.txt` formats. Some results (Tables 11, 12, A2) are printed to `Main.log` and marked with "XXX" tags.

## Key Data Sources

All data are publicly available:
- **COMETS** (primary): patent_inventors.dta, patent_zd_cats.dta, patent_assignees.dta, patent_cite_counts.dta, patent_citations.dta
- **NBER Patent Database**: apat63_99.txt
- **BEA Input-Output Tables**: Use_SUT_Framework_2007_2012_DET.csv
- **NAICS-USPC Crosswalk**: uspc_Class_to_naics07_6.txt
- **BEA Economic Areas**: bea_codes.dta
- **County Characteristics**: tva1.dta
- **FIPS-BEA Crosswalk**: bea_county_crosswalk.txt
- **Kodak Stock Prices**: kodak_stock.csv

## Required Stata Packages

estout, reghdfe, ftools, ivreghdfe, ivreg2, ranktest

## Auditing Notes

When reviewing code for errors, pay attention to: variable construction logic, merge operations, sample restrictions, regression specifications vs. what the paper describes, and whether output matches the claimed table/figure content.
