# Audit Report: Moretti (2021) "The Effect of High-Tech Clusters on the Productivity of Top Inventors"

**Paper:** American Economic Review, 111(10): 3328–3375
**Replication package:** OpenICPSR 140662
**Audit date:** 2026-03-17
**Scope:** All 31 Stata scripts in `code/`, the published paper, and the online appendix

---

## 1. Definite Code Bugs

### 1.1 Variable name error in `data_4.txt` (6-month section, ~line 200)
The code creates `month1` but then references `month` (which does not exist). This would cause a Stata runtime error, making the entire 6-month panel non-functional.
```stata
g month1 = substr(app_date,6,2)
g tmp = 1 if month=="01" | month=="02"   * ERROR: should be month1
```

### 1.2 Variable name error in `reg23.txt` (~line 155)
`nn` is referenced but only `nn1` exists (created on line 148). This causes a runtime error in the movers analysis for Figure 6. The line is functionally redundant (line 161 filters on `nn1==1` anyway), so Figure 6 may still be producible if the error is caught interactively, but the script cannot run end-to-end.
```stata
egen nn1 = total(move), by(inventor)
...
replace move_year1 = . if nn>1    * ERROR: should be nn1
```

### 1.3 `org_new` dropped then referenced in `reg4.txt` and `reg34.txt`
In the sub-annual and multi-year regressions (1-month, 2-month, 3-month, 6-month, 2-year, 3-year panels for Appendix Table A7), the code does:
```stata
egen org_new2 = group(org_new)
drop org_new
...
reghdfe y x, absorb(... org_new ...)   * ERROR: org_new no longer exists
```
This affects all sub-annual regressions in both `reg4.txt` (~lines 314–320) and `reg34.txt` (~lines 28–187). These regressions may fail at runtime or silently rely on Stata abbreviation matching `org_new` to `org_new2`, which is undocumented behavior in `reghdfe`.

### 1.4 Reversed sign in `Dx_m3` lag construction (`reg11.txt`, `reg13.txt`, `reg36.txt`, all ~line 147)
The third lag of the change in cluster size is computed backwards:
```stata
Dx_m1 = x[_n-1] - x[_n-2]    * correct: change from t-2 to t-1
Dx_m2 = x[_n-2] - x[_n-3]    * correct: change from t-3 to t-2
Dx_m3 = x[_n-3] - x[_n-2]    * WRONG:  should be x[_n-3] - x[_n-4]
```
`Dx_m3` equals `-Dx_m2`, not the change from t-4 to t-3. This affects the IV first-difference specifications in Table 5 and Appendix Table A4 wherever the third lag is included.

### 1.5 `cluster_class_year` duplicated in `absorb()` (`reg4.txt`, `reg34.txt`)
The variable `cluster_class_year` appears twice in the `absorb()` list in all regressions in these files. `reghdfe` likely handles this silently, but it indicates copy-paste errors and raises the question of whether the intended second variable was something else.

---

## 2. Dangerous Stata Practices

### 2.1 `merge m:m` used in multiple critical scripts
Stata's documentation explicitly warns that `merge m:m` does not produce a proper cross-join; it matches observations sequentially within groups, producing unpredictable results when both files have multiple observations per key. This affects:

| File | Line | Merge key | Impact |
|------|------|-----------|--------|
| `create_COMETS_Patent_ExtractForEnrico.txt` | ~60 | `patent_id` (assignees) | If patents have multiple assignees, pairings are incomplete |
| `create_COMETS_Patent_ExtractForEnrico.txt` | ~64 | `patent_id` (ZD categories) | If patents map to multiple fields, results are unpredictable |
| `data_3.txt` | ~47 | `patent` (NBER merge) | Should be `m:1` if NBER data is unique on patent |
| `citations_received.txt` | ~16, ~36 | citation pairs | Citation shares may be miscounted |
| `citations_made.txt` | ~17, ~38 | citation pairs | Citation counts may be inflated |

The `m:m` merges are particularly dangerous in the citations files, where both the citing and cited patent may appear multiple times (multi-inventor patents). These merges directly affect Table 6 (citations received), Table 7 (citations made), and Appendix Table A5 (cross-field spillovers).

### 2.2 Pervasive reliance on Stata variable-name abbreviation
Throughout the codebase, variables are referenced by partial names (e.g., `bea` when the actual variable is `bea_code`, `zd` when it is `zd2`, `inventor` when it is `inventor_id`, `patent` when it is `patent_id`). This works only if no other variable shares the same prefix. It is fragile and makes the code difficult to audit, since the actual variable being referenced depends on the dataset state at runtime. Key instances:
- `data_3.txt` lines 287–333: `bea` → `bea_code`, `zd` → `zd2`
- `data_3.txt` line 47: `patent` → `patent_id`
- `data_4.txt` lines 49–50: `bea` and `zd` in merge keys
- `reg20.txt` line 389: `inventor` in merge key

---

## 3. Variable Construction Issues

### 3.1 Inventor identification based on name strings (`data_3.txt` ~line 27)
```stata
egen inventor_id = group(inventor)
```
Inventor IDs are created from name strings. In a dataset of hundreds of thousands of inventors, name collisions (different people with the same name) are virtually certain. Homonymous inventors are merged into one, inflating their patent counts, corrupting the "star inventor" classification (top 10% by lifetime patents), and biasing cluster size measures. The COMETS database contains proper inventor identifiers that should be used instead.

### 3.2 Cluster size is a national share, not a count (`data_3.txt` ~lines 89–93)
The paper describes cluster size as "the number of other inventors" in a city-field-year. The code computes:
```stata
density14 = (count - 1) / (total - 1)
```
This is a *share* of national field activity, not a raw count. Because the regressions use field × year fixed effects, `log(share) = log(count-1) - log(total-1)`, and the denominator is absorbed. So the regression coefficient is numerically equivalent to using `log(count-1)`. However, summary statistics and any computation outside the regression (e.g., counterfactuals in Tables 10–12) use the share, not the count, which could be a source of error.

### 3.3 Unweighted county averaging in `read.txt` (~line 19)
County-level characteristics (housing values, income, demographic shares) are collapsed to BEA economic areas using unweighted means:
```stata
collapse vhouse90 medfaminc90 trade90 agr90 const9 manuf90 nonwhite90 (sum) pop emp, by(bea)
```
Variables before `(sum)` default to `(mean)`. Averaging percentages across counties without population weighting gives equal weight to a rural county of 5,000 and an urban county of 5,000,000. This affects the Rochester weighted matching estimates in Table 2, column 5.

### 3.4 BEA filtering is career-wide, ZD filtering is year-specific (`data_3.txt` ~lines 56–68)
Inventors appearing in >3 BEA areas across their *entire career* are dropped, but inventors in >3 research fields are only dropped within a *single year*. An inventor in 2 cities per year for 2 different years (4 total) is dropped entirely, even though they had at most 2 cities in any year. The BEA filter should match the ZD filter by using `by(inventor year)`.

### 3.5 Team-adjusted density uses mean team size, not distinct co-authors (`density_team.txt` ~line 91)
```stata
density14_team = (count - tt) / (total - tt)
```
`tt` is the mean team size across an inventor's patents, not the number of distinct co-inventors. An inventor with 3 patents (team sizes 2, 3, 1) gets `tt = 2`, but the actual number of unique teammates to subtract could be anywhere from 0 to 4. This affects Appendix Table A8, column 6.

### 3.6 Quality-weighted density can produce division by zero (`density14_W.txt` ~line 111)
```stata
density14_W1 = (count - K) / (total - K)
```
When `total = K` (e.g., the only inventor above a quality threshold in a field-year), this divides by zero. Commented-out lines suggest this was noticed but not fixed.

---

## 4. Econometric Concerns

### 4.1 Singleton handling inconsistency across specifications
The baseline regressions in `reg.txt` use `keepsin` (keeping singleton groups), but the lead/lag regressions in the same file (~line 287) and the movers regressions in `reg23.txt` (~line 208) do not. This creates different effective samples between the static and dynamic specifications, making direct comparison of coefficients across Tables 3, 5, and Figures 5–6 problematic.

### 4.2 Missing city × year fixed effects in the preferred specification
The paper's equation (1) lists `d_ct` (city × year effects) as a control. Table 3 confirms city × year FE are included in column 7. The coefficient drops from 0.0923 (col 6, with inventor FE but no city × year) to 0.0545 (col 7). This ~41% reduction suggests substantial confounding from time-varying city-level shocks. The preferred column 8 (with firm FE) restores it to 0.0676 — but city × year FE absorb much of the identifying variation when included. The paper's main results may overstate the pure agglomeration effect if city × year shocks that affect both cluster size and productivity are not fully controlled for.

### 4.3 No city × year effects in Appendix Table A2 field-specific regressions
Footnote 18 (p. 3349) notes these models "do not include city × year effects to avoid multicollinearity." Field-specific regressions with only within-field variation cannot separately identify city × year from field × year when there is only one field. But this means the field-specific elasticities (0.076 to 0.262) are estimated without controlling for time-varying city-level confounders, making them upward-biased relative to the main specification.

### 4.4 Reflection problem in cluster size
Cluster size `S_{-ifct}` is the number of *other* inventors in the same city-field-year. But each inventor's productivity (the dependent variable) contributes to the cluster that other inventors are "treated" by. This is the classic reflection problem (Manski 1993). While inventor and firm FE help, they do not fully resolve simultaneity. The IV strategy addresses this for the firm-network instrument, but the main OLS results (Table 3) remain susceptible.

### 4.5 IV construction issues (`iv_new.txt`)
- The DD differencing (~line 67) sorts by `(zd, org_new, year)` without `bea`, mixing city-level DD values across cities for multi-city firms when computing lags.
- `org_new` vs `org_new2` inconsistency (~lines 36 vs 105): the collapse and the focal-firm subtraction may use different firm identifiers.
- The normalization denominator sums firm-level DD1 across all firms in a field-year (~line 88), which is not the same as the total national change in the field described in the paper's formula.

### 4.6 Citations transformation uses extreme constant (`reg.txt` ~lines 67–68)
```stata
y2 = log(citations + 0.00001)
```
Adding 0.00001 instead of 1 creates an extreme gap: zero-citation observations get y2 ≈ −11.5, while one-citation observations get y2 ≈ 0. This inflates the apparent variation in the dependent variable for Table 6 and may distort coefficient estimates and standard errors relative to the standard `log(x+1)` transformation.

### 4.7 NBER patent data coverage gap (`read_apat.txt`)
The NBER patent file (`apat63_99.txt`) covers only 1963–1999, but the sample spans 1971–2007. Patents from 2000–2007 have no NBER variables (technology category, subcategory, generality, originality). These are used for sample stratification and robustness checks. The paper does not discuss this coverage gap or how missing values for 2000–2007 patents are handled.

### 4.8 Counterfactual calculation simplification (`reg2.txt` ~line 271, `reg29.txt` ~lines 197–204)
The aggregate counterfactual (Tables 10–12) uses:
```stata
g alpha = 0.067
productivity_change = mean^alpha - Den_bea_zd^alpha
```
This assumes `y = S^α`, treating the productivity function as a simple power law. The actual model is `log(y) = α·log(S) + [FE + ε]`, so `y = S^α · exp(FE + ε)`. The heterogeneous multiplicative constant (inventor, firm, city-field effects) is ignored. This means the counterfactual implicitly assumes all inventors within a field are identical except for their cluster's size.

---

## 5. Paper–Code Discrepancies

### 5.1 Table 3 column 8 claims city × year FE but code (`reg.txt` ~line 136) confirms this
The paper says column 8 includes "Year, City, Field, Class, City × Field, City × Class, Field × Year, Class × Year, Inventor, City × Year, and Firm effects." The code matches. No discrepancy — noted for verification completeness.

### 5.2 Table A6 baseline comment is misleading (`reg31.txt` ~line 152)
The code comment says "This is the baseline. It should be equal to the main table with no interpolation." But the dependent variable is `asinh(number)`, not `log(number)`. The baseline with `asinh` will not reproduce the main table's `log` results.

### 5.3 Hardcoded elasticity in counterfactuals
`reg2.txt` line 271: `g alpha = 0.067`. `reg29.txt` uses quartile-specific alphas. These are hardcoded rather than programmatically extracted from regression output. If the data or specification changes, these values must be manually updated. The 0.067 value matches Table 3, column 8 (0.0676 rounded), so it is correct as published but creates a fragile dependency.

### 5.4 Output file overwriting (`reg20.txt` ~lines 305 vs 424)
The file `tables20/table7` is written to twice with different content — first with lifecycle-control results, then with cross-field spillover results. The second write overwrites the first.

---

## 6. Reproducibility and Data Integrity Issues

### 6.1 `ssc install binscatter` inside analysis script (`reg2.txt` ~line 203)
Running `ssc install` mid-script installs whatever version is current at runtime. If the package updates with different defaults, Figure 4 could change. Package versions should be pinned.

### 6.2 Blank country treated as US (`create_COMETS_Patent_ExtractForEnrico.txt` ~line 51)
```stata
keep if country=="US" | country==""
```
Patents with blank country fields are assumed to be US. These could be foreign inventors, introducing measurement error in BEA assignment.

### 6.3 No deduplication checks
Neither `read_apat.txt` nor the COMETS extraction script checks for duplicate patent IDs before merging. Combined with the `m:m` merge behavior, silent duplicates could propagate through the entire pipeline.

### 6.4 Relative paths in `reg4.txt` (~lines 233, 243)
Some merge commands use `data2/density14_3` (relative path) instead of `$main/data2/density14_3`. If the working directory differs from `$main`, these merges silently fail.

### 6.5 Temporary file deletion uses wrong path (`read.txt` ~line 22)
```stata
!rm $main/tmp.dta
```
But the file was saved to the current working directory with `save tmp, replace`. If the working directory ≠ `$main`, the temp file is not deleted (or the wrong file is deleted).

---

## 7. Econometric Design Concerns

### 7.1 Star inventor sample selection
Focusing on the top 10% of patenters by lifetime count creates survivorship bias: inventors are classified as "stars" using their complete 1971–2007 record, but the regressions treat this as predetermined. An inventor who was highly productive *because* they were in a large cluster gets classified as a star partly due to the treatment. The paper acknowledges this is "arbitrary" (p. 3335) and shows robustness to alternative cutoffs (Table 9), but the fundamental endogeneity of the star classification remains.

### 7.2 Selection on observability
Inventors are only observed when they patent. The paper acknowledges this creates sample selection (Section IC, p. 3338) and provides interpolation robustness checks (Appendix Table A6). However, the interpolation assigns zero patents for gap years, which could attenuate or inflate the elasticity depending on whether missing years are random or correlated with cluster size changes.

### 7.3 Movers design may conflate firm and location effects
Figure 6 uses movers to identify the cluster size effect. But inventors who move cities may simultaneously change firms. If firm quality covaries with cluster size (better firms locate in larger clusters), the movers estimate captures both agglomeration and firm-quality effects. The paper includes firm FE in the baseline (Table 3, col 8), but the movers analysis in `reg23.txt` does not appear to separately control for firm changes.

### 7.4 Rochester DiD: parallel trends not formally tested
The Rochester case study (Table 2) assumes non-Kodak Rochester inventors would have followed the same productivity trend as non-Kodak inventors elsewhere absent Kodak's decline. Figure 3 shows roughly flat pre-1996 trends, but no formal pre-trend test is reported. The decline could reflect Rochester-specific shocks beyond Kodak (e.g., Xerox also declined in Rochester during this period, noted in footnote 11).

---

## Summary

| Category | Count | Most Critical Items |
|----------|-------|---------------------|
| Definite bugs | 5 | `month1`/`month` in data_4; `nn`/`nn1` in reg23; reversed `Dx_m3`; dropped `org_new` in reg4/reg34 |
| Dangerous `m:m` merges | 5 instances | COMETS extract, data_3, citations_received, citations_made |
| Variable construction issues | 6 | Name-string inventor IDs; unweighted county averages; inconsistent BEA/ZD filtering |
| Econometric concerns | 8 | Reflection problem; IV construction errors; singleton inconsistency; extreme citation transform |
| Paper–code discrepancies | 4 | Misleading comments; hardcoded elasticities; output overwriting |
| Reproducibility issues | 5 | No dedup checks; relative paths; `ssc install` in script |
| Design concerns | 4 | Star classification endogeneity; selection on observability; parallel trends |

The most consequential issues are: (1) inventor identification by name string, which could systematically bias the star inventor sample and cluster size measures; (2) the widespread use of `m:m` merges, which produce unpredictable results in Stata; (3) the reversed `Dx_m3` lag, which affects the IV specifications; and (4) the IV construction logic in `iv_new.txt`, which has sort-order and variable-identity inconsistencies that could produce an incorrectly computed instrument.
