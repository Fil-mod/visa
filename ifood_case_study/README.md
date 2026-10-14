# iFood Marketing Case Study — solution

| File | What it is |
|---|---|
| `iFood_Case_Study_Solution.ipynb` | Full solution (Q1–Q8), executed, with outputs and charts. For every question: *Approach* → code → **Result & what it means**. Starts with an executive TL;DR. |
| `iFood_Case_Study_Solution.html` | The same notebook rendered as a standalone HTML page (open in any browser). |
| `PRZEWODNIK_PL.md` | Polish study guide: approach / what was done / result / meaning for every question, a metrics glossary and likely interview questions. |
| `ifood_df.csv` | The input data (unchanged). |

## Reproduce

```bash
pip install pandas numpy scipy scikit-learn matplotlib seaborn jupyter
jupyter nbconvert --to notebook --execute --inplace iFood_Case_Study_Solution.ipynb
```

- Only the libraries allowed by the brief are used (pandas, numpy, matplotlib/seaborn, scipy, scikit-learn), plus the standard-library `sqlite3` for Q7.
- Fixed `RANDOM_STATE = 42`. A full run takes ~8–10 minutes, mostly the nested cross-validation in Q5.
- Executed with pandas 3.0 and scikit-learn 1.9. The code also handles scikit-learn < 1.8, where `penalty='elasticnet'` is required.
- The hand-written statistical helpers in Q4 (logistic regression, robust OLS, Wilson / Newcombe CIs, Holm, CMH) were checked against `statsmodels` and agree to machine precision.
