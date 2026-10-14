# iFood Marketing Case Study — solution

| File | What it is |
|---|---|
| `iFood_Case_Study_Solution.ipynb` | Full solution (Q1–Q8), executed, with outputs and charts. For every question: *Approach* → code → **Result & what it means**. Starts with an executive TL;DR and a **decision log** (every important choice, the alternatives and the reason); ends with **Appendix A**, which checks the assumptions of every method used and explains why each was chosen. |
| `iFood_Case_Study_Solution.html` | The same notebook rendered as a standalone HTML page (open in any browser). |
| `PRESENTATION_SCRIPT_EN.md` | Ready-to-read ~20-minute presentation script (first person): methodology, every decision with its alternatives, results, plus "If they ask…" boxes, a Q&A appendix and a numbers cheat-sheet. |
| `PRZEWODNIK_PL.md` | Polish study guide: approach / what was done / result / meaning for every question, all decisions with their alternatives, why each method is valid here, a metrics glossary and likely interview questions. |
| `ifood_df.csv` | The input data (unchanged). |

## Reproduce

```bash
pip install pandas numpy scipy scikit-learn matplotlib seaborn jupyter
jupyter nbconvert --to notebook --execute --inplace iFood_Case_Study_Solution.ipynb
```

- Only the libraries allowed by the brief are used (pandas, numpy, matplotlib/seaborn, scipy, scikit-learn), plus the standard-library `sqlite3` for Q7.
- Fixed `RANDOM_STATE = 42`. A full run takes ~10–15 minutes, mostly the nested cross-validation and the robustness checks in Q5.
- Executed with pandas 3.0 and scikit-learn 1.9. The code also handles scikit-learn < 1.8, where `penalty='elasticnet'` is required.
- The hand-written statistical helpers in Q4 (logistic regression, robust OLS, Wilson / Newcombe CIs, Holm, CMH) were checked against `statsmodels` and agree to machine precision.
