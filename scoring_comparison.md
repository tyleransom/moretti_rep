# Claude vs GPT Scoring Comparison (Wiebe's 10 Issues)

Scores use Wiebe's grading rubric. "Caught" = flagged the key problem.

| # | Issue | Wiebe's Key Finding | Claude | GPT5.2 | GPT5.4 | Refine |
|---|-------|---------------------|--------|--------|--------|--------|
| 1 | Fig 5 DL model | `_n` leads/lags wrong on unbalanced panel | **8/10** | 4/10 | 8/10 | 0/10 |
| 2 | Fig 6 event study | Time-varying cluster size for B_0; no interaction term | **5/10** | 5/10 | 6/10 | 6/10 |
| 3 | Table 5 IV | No sort by city before first-difference | **10/10** | 10/10 | 10/10 | 10/10 |
| 4 | Table 6 citations | `log(y+0.00001)` and per-coauthor calculation error | **8/10** | 7/10 | 4/10 | 1/10 |
| 5 | Table 8 heterogeneity | Omitted quartile indicators in Panel A | **0/10** | 1/10 | 10/10 | 1/10 |
| 6 | Table A6 interpolation | 2-year gap only fills second year; movers excluded | **5/10** | 7/10 | 2/10 | 0/10 |
| 7 | Table A7 time unit | Extensive margin confusion; cluster size not redefined | **3/10** | 3/10 | 3/10 | 0/10 |
| 8 | Table A8 cluster quality | Average vs modal cluster size; no coauthor adjustment | **4/10** | 6/10 | 1/10 | 0/10 |
| 9 | Table A8 team size | Team size is mediator not confounder; double adjustment | **4/10** | 5/10 | 5/10 | 5/10 |
| 10 | Unreproducible cleaning | `m:m` merges; no unique inventor ID | **10/10** | 10/10 | 10/10 | 0/10 |
| | **Average** | | **5.7/10** | 5.8/10 | 5.9/10 | 2.3/10 |

## Claude Score Rationale

- **Issue 1 (8):** Flagged `_n` on unbalanced panel (§2.1, §4.1) and unbalanced panel in IV sort (§4.5); missed DL assumptions critique.
- **Issue 2 (5):** Caught time-varying cluster size for B_0 (§4.6 `log(y+0.00001)` context); flagged `nn`/`nn1` bug in `reg23.txt` (§1.2); missed interaction-term omission.
- **Issue 3 (10):** Explicitly flagged IV sort-order bug in `iv_new.txt` (§4.5, first bullet).
- **Issue 4 (8):** Flagged `log(y+0.00001)` (§4.6) and `m:m` merges in citations files (§2.1); missed per-coauthor error.
- **Issue 5 (0):** Did not flag omitted quartile indicators in Table 8 Panel A.
- **Issue 6 (5):** Did not explicitly flag 2-year gap fill error or mover exclusion (only general interpolation issues noted).
- **Issue 7 (3):** Flagged cluster size not redefined at new time unit (§3.2 density discussion); missed extensive margin confusion.
- **Issue 8 (4):** Flagged `m:m` merges affecting Table A8 (§2.1); missed average-vs-modal and per-coauthor errors specifically.
- **Issue 9 (4):** Did not flag team size as mediator/bad control or double-adjustment (fractional patents + control).
- **Issue 10 (10):** Comprehensively flagged `m:m` merges (§2.1), name-string inventor IDs (§3.1), and unreproducibility.

## Takeaway

Claude performs comparably to GPT5.2/5.4 on average (~5.7 vs 5.8–5.9). Strengths: IV sorting bug, `m:m` merges, `log(y+0.00001)`, unreproducibility. Weaknesses: omitted quartile indicators (Issue 5), team size double-adjustment (Issue 9), interpolation details (Issue 6).
