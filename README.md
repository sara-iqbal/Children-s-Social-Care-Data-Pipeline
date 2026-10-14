# Children's Social Care Data Pipeline

**From messy case records to a statutory return and a performance scorecard, built end to end on synthetic data.**

[Live dashboard](https://<your-username>.github.io/<repository-name>/) · [Notebook](childrens_social_care_pipeline.ipynb) · [Open in Colab](https://colab.research.google.com/github/<your-username>/<repository-name>/blob/main/childrens_social_care_pipeline.ipynb)

![Assessment timeliness by month](images/assessment_timeliness.png)

## The problem

Every week, social workers in England record what is happening for thousands of children: referrals, assessments, child protection plans, children coming into care and moving between placements. All of that lives in a case management system such as Liquidlogic.

Somebody then has to turn those records into something useful. That's the job of a council's intelligence and performance team, and it comes down to three questions:

1. **Can we trust the data?** Records are typed in by busy people. Dates get swapped, dropdowns get skipped, and old codes linger. Errors need to be found and sent back to the right team to fix.
2. **Can we meet our statutory duties?** Every local authority sends the Department for Education an annual return called the **SSDA903**, covering every child looked after during the year. It has a strict format and strict validation rules.
3. **Are children getting help on time?** Managers need a monthly view of a few key measures, such as whether assessments finish within 45 working days or whether children in care are moving placement too often.

I wanted to show I understand this work, not just that I can write Python. The obvious obstacle is that real child-level social care data is, quite rightly, never made public. So I built the whole thing myself: a realistic synthetic dataset, and the full analyst workflow on top of it.

## How I approached it

The project runs as one Google Colab notebook in four stages.

**1. Generate the data.** Python (pandas and NumPy) creates linked tables for two fictional borough services, following a child's journey through the system: referral, assessment, child protection plan, care, and leaving care. I deliberately built realistic behaviour into it. One assessment team is under pressure, timeliness drops after the summer holidays, and a minority of children in care move placement many times. A fixed random seed means anyone who runs the notebook gets exactly the same numbers.

**2. Load it into SQL and validate it.** The tables go into a SQLite database, mirroring how an analyst queries a reporting warehouse. Ten validation rules, written in SQL and modelled on the kinds of checks the DfE applies, produce an exceptions report broken down by rule, team and worker. Because I planted the errors myself, I could test my own rules honestly. Every planted error was caught.

**3. Build the statutory return.** The notebook produces the three core SSDA903 tables (Header, Episodes and OC3 for care leavers) in the DfE layout, with `dd/mm/yyyy` dates and DfE codes for legal status, placement type and reasons for leaving care. Children whose errors would block submission are flagged rather than quietly dropped, so they can be fixed before the deadline.

**4. Report performance.** Eight standard children's social care KPIs are calculated with targets and red, amber and green ratings, for each borough and combined. The results are exported as tidy CSV files for a Power BI report and as JSON for the web dashboard on this repository's GitHub Pages site.

One design decision I'm particularly pleased with: each KPI rule (for example, "was this assessment completed within 45 working days, excluding weekends and bank holidays?") is calculated **once**, in Python, and exported as a 1/0 flag. The Power BI measures then become simple sums and divisions, so the notebook, Power BI and the web dashboard can never disagree about a definition.

## The data

Everything here is synthetic. No real child, family or worker is represented.

| Table | Rows | What it represents |
| :- | -: | :- |
| Children | 5,505 | Date of birth, sex, DfE ethnicity code, UPN, borough |
| Referrals | 6,494 | Referral date, source (police, schools, health and so on) and outcome |
| Assessments | 4,900 | Single assessments with start and end dates, team and allocated worker |
| Child protection plans | 793 | Category of abuse or neglect, start and end dates, including historic plans |
| Care episodes | 758 | 264 children in care, with legal status, placement and reason for each change |
| Care leavers | 115 | Young people aged 19 to 21: in touch, activity and accommodation |

Around 2 to 5 per cent of records contain planted errors: blank ethnicity, assessments ending before they start, missing workers, dates of birth after a referral, invalid legal status codes, overlapping care episodes and duplicate referrals. The errors are concentrated in one team and two individual workers, because that's how recording problems tend to appear in practice.

The code lists follow the DfE's SSDA903 guidance but are simplified. The real guide runs to over a hundred pages.

## Results

Reporting year 2025-26, both boroughs combined. Targets are illustrative, loosely informed by recent England averages.

| KPI | Result | Target | Rating |
| :- | -: | -: | :- |
| Assessments completed within 45 working days | 81.8% | 85% | Amber |
| Re-referrals within 12 months | 16.6% | 20% | Green |
| CP plans ended that lasted 2+ years | 2.2% | 4% | Green |
| CP plans that were a repeat plan | 23.8% | 22% | Amber |
| Children in care with 3+ placements in the year | 12.6% | 10% | Amber |
| Long-term placement stability | 76.7% | 70% | Green |
| Care leavers in education, employment or training | 54.8% | 55% | Amber |
| Care leavers in suitable accommodation | 87.0% | 88% | Amber |

What stands out:

* **One team explains most of the timeliness gap.** Kingston Assessment Team B completed 66.0% of assessments on time, against 86.1% for the other three teams. Kingston as a whole is rated red (77.5%), while Richmond meets the target (85.6%). A borough-level average would have hidden where the problem actually sits.
* **The same team has the most recording errors.** It logs 3.9 errors per 100 assessments, against 1.1 to 2.3 elsewhere. Slow assessments and poor recording appearing together usually point to capacity, not carelessness.
* **Timeliness dipped in autumn.** Assessments completed in October 2025 had the lowest on-time rate (75.0%), and most of the late ones had started in July, when staff leave peaks.
* **The return isn't ready to submit yet.** The validation found 372 exceptions, 168 of them errors. 25 of the 190 children in the SSDA903 return have at least one blocking error that needs fixing first.

![Recording errors by team](images/dq_by_team.png)

## What I learned, and how I'd interpret this

**Definitions matter more than code.** Getting "45 working days" right meant excluding weekends and bank holidays, deciding whether to count from the referral or the assessment start, and deciding what to do with records that have impossible dates. Each choice moves the headline figure. In a real team, I'd want those definitions written down and agreed before anyone builds a chart.

**Rates, not counts.** Comparing raw error counts would have penalised bigger teams. Errors per 100 assessments and referrals per 10,000 children make fair comparisons possible.

**Small numbers need care.** The long-term stability measure rests on 43 children, and one team's monthly timeliness figure rests on about 30 assessments. A single case can move these by several points, so I always show the denominator and read trends rather than individual months.

**Data quality is a relationship, not a report.** The exceptions list is only useful if it reaches the right team manager in a form they can act on, record by record, and if they understand why it matters for the child and for the return. That conversation is as important as the SQL.

**A red rating is a question, not a verdict.** Team B's figures tell a manager where to look. They don't explain why. Vacancies, sickness, a run of complex cases or a new starter could all produce the same pattern. The data starts the conversation; practitioners finish it.

## Limitations

* The data is synthetic, so the "findings" are patterns I designed. The project demonstrates **method**, not facts about any real borough.
* DfE code lists and validation rules are simplified.
* Targets are illustrative. Real benchmarking would use published DfE figures and statistical neighbours.
* Timeliness is measured from the assessment start date. Statutory guidance counts from the point of referral, and local definitions vary.

## Run it yourself

1. Open the notebook in Google Colab using the link at the top of this page.
2. Select **Runtime > Run all**. Nothing needs installing.
3. The final cell downloads a zip containing the SQLite database, the SSDA903 files, the Power BI CSVs and the dashboard data.

To rebuild the Power BI report, follow `PowerBI_Build_Guide.docx`, which walks through the data model, every DAX measure and each report page.

## What's in this repository

| File | Purpose |
| :- | :- |
| `childrens_social_care_pipeline.ipynb` | The full pipeline, with commentary |
| `index.html` | Interactive dashboard, published with GitHub Pages |
| `PowerBI_Build_Guide.docx` | Step-by-step Power BI build instructions |
| `images/` | Charts used in this README |

## Tools

Python (pandas, NumPy, Matplotlib), SQL (SQLite), Power BI and DAX, Chart.js, Google Colab, GitHub Pages.

*Built by Sara Iqbal (MSc Data Science) as a portfolio project for data analyst roles in children's services.*
