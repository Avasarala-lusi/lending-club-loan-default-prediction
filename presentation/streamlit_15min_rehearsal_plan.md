# 15-Minute Streamlit Rehearsal Plan

**Team:** 404 Team Not Found  
**Presentation tool:** Streamlit app  
**Important timing rule:** Q&A is **not** counted in the 15 minutes.

Run the app from the project root:

```powershell
streamlit run src/app/streamlit_app.py
```

This version uses simple speaking language for ESL delivery. Keep the pace
calm. Short sentences are your friend. The model can be complicated; your
talk does not need to sound complicated.

---

## Speaker Split

| Speaker | Responsibility | Time |
|---|---|---:|
| Junwei Zhang | Welcome, Phase 1, Insights + Next | 4:30 |
| Lusi Zhang | Phase 2 Pipeline, Phase 2 Models | 5:00 |
| Aayush Nepal | Phase 3, MCP, Predict a Loan demo | 5:30 |
| Q&A | Everyone | Extra time, not counted |

This is fair. Lusi has the deepest model section. Aayush has the live demo,
which needs calm timing. Junwei opens and closes the story.

---

## 15-Minute Timing

| Time | Tab | Speaker | Duration |
|---:|---|---|---:|
| 0:00-1:00 | Welcome | Junwei Zhang | 1:00 |
| 1:00-2:45 | Phase 1: Data | Junwei Zhang | 1:45 |
| 2:45-4:45 | Phase 2: Pipeline | Lusi Zhang | 2:00 |
| 4:45-7:45 | Phase 2: Models | Lusi Zhang | 3:00 |
| 7:45-10:00 | Phase 3: Spark + Macro | Aayush Nepal | 2:15 |
| 10:00-11:15 | Phase 3: MCP | Aayush Nepal | 1:15 |
| 11:15-13:15 | Predict a Loan | Aayush Nepal | 2:00 |
| 13:15-15:00 | Insights + Next | Junwei Zhang | 1:45 |

After 15:00, open the Q&A tab and take questions.

---

## Run Of Show

- Move through the tabs from left to right.
- Do not explain every chart. Pick one main message per tab.
- Speak slowly. It is better to be clear than fast.
- For the live demo, use one preset. Do not type many values live.
- If something loads slowly, smile and say: "This is the real model, so it is
  doing real work. Tiny dramatic pause."

---

## Oral Script

### 1. Junwei Zhang - Welcome, 1:00

Hi everyone. We are 404 Team Not Found. Our project is **Lending Club Loan
Default Prediction**.

Our question is simple: when someone applies for a loan, can we estimate the
chance that this loan will default?

We built the project from end to end. First, we cleaned the Lending Club data.
Then we trained machine learning models. After that, we rebuilt the data
pipeline in Spark and Delta Lake. We also added macroeconomic data from FRED.
Finally, we made the model usable in two ways: this Streamlit app, and an MCP
tool for conversational prediction.

Our final XGBoost model gets about **0.726 AUC-ROC** on the untouched 2017 test
set. That test set has more than **314,000 loans**.

So this is not just a notebook. It is a small end-to-end ML system. A very
serious system, even though our team name sounds like we disappeared.

**Transition:** First, I will explain the data decisions.

### 2. Junwei Zhang - Phase 1: Data, 1:45

For Phase 1, we made three important choices: target, time range, and split.

Our target is `charged_off`. If `charged_off` is 1, the loan defaulted. If it
is 0, the loan was fully paid.

We used loans from **2014 to 2017**. Data before 2014 is more sparse. Data
after 2017 may not have enough time to mature. So this range is a safer choice.

We also used a time-based split. Training and validation come from 2014 to
2016. The full year 2017 is the test set. We did not use a random split,
because random splitting can mix the future with the past. That would make the
model look better than it really is. We wanted no time travel. Time travel is
cool in movies, not in model evaluation.

The class balance is also important. About **21%** of loans are defaults. That
means most loans are not defaults. We kept this natural imbalance and handled
it with model choices and threshold tuning.

The biggest data decision was leakage. We dropped **21 post-loan columns**.
These include payment history, recoveries, settlement fields, and last FICO
values. Those columns are only known after the loan outcome happens. If we use
them, the model is basically cheating.

So Phase 1 gave us a clean and honest dataset.

**Transition:** Now Lusi will show how we turned this data into model features.

### 3. Lusi Zhang - Phase 2: Pipeline, 2:00

Phase 2 is where we turn the cleaned loan table into model-ready features.

The key rule is shown at the top of this slide: the preprocessing pipeline is
**fit on train only**, then applied to validation and test. This is important
because preprocessing can also leak information if we are not careful.

Our input has **49 cleaned columns**, and the output has **110 features**. The
pipeline has eight stages.

First, `DropCorrelated` removes 13 highly correlated columns. For example,
`fico_range_high` is basically the same as `fico_range_low`, so we keep only
one signal.

Second, `DateExtractor` changes date columns into useful numbers, like issue
year, issue month, and credit history length.

Third, `FeatureConstructor` creates new features, such as loan-to-income,
monthly-payment-to-income, and delinquency rate.

Fourth, `OutlierCapper` fixes extreme values. For example, `dti` had a raw max
of 999, which is more like a data-entry problem than a real borrower. Unless
someone's debt-to-income ratio is from another planet, we should cap it.

Then we add missing-value indicators, join FRED unemployment rate with
`MacroJoiner`, merge rare categories into "other", and finally route each
column to the right transformation.

So the main point is: this is not just cleaning. It is a reproducible feature
factory. Same input logic, same output features, every time.

**Transition:** After the pipeline, we tested different models.

### 4. Lusi Zhang - Phase 2: Models, 3:00

For modeling, all four models use the same preprocessing pipeline from the last
slide. So the comparison is fair.

We used Logistic Regression as a baseline, then tested Decision Tree,
HistGradientBoosting, and XGBoost. For the tuned models, we used Optuna with 50
trials each, and logged the runs in MLflow.

The validation chart shows that XGBoost was the best model. Its validation
AUC-ROC is **0.7238**, HistGradientBoosting is very
close at **0.7233**, but it took much longer to fit. XGBoost gave us strong
performance and useful feature contribution tools, so we chose XGBoost.

Then we tested it on the full untouched 2017 test set.
The final results are **0.7260 AUC-ROC**, **0.4103 AUC-PR**, and **0.3279 KS**.
At the Youden threshold, recall on defaults is **67.3%**.

The threshold tuner is important because the model gives a probability, but the
business still needs a cutoff. At threshold **0.50**, recall is about **66.5%**,
precision is about **34.4%**, and about **40.6%** of the book is flagged.

If we move the threshold very high, like **0.85**, precision becomes higher,
around **61.8%**, but recall drops to only **4.4%**. That means we catch very
few defaults. So a high threshold looks strict, but it may miss too many risky
loans.

Finally, the precision-recall curve compares our model to the class prior. The
average precision is about **0.410**, while the default rate is about **21%**.
So the model roughly doubles the random baseline.

The short message is: XGBoost wins, but the threshold is a business choice.
This slider is small, but it has big power. 

**Transition:** Next, Aayush will show the Spark, macro, and deployment parts.

### 5. Aayush Nepal - Phase 3: Spark + Macro, 2:15

In Phase 3, we rebuilt the data pipeline in Databricks using Spark and Delta
Lake.

We used the medallion architecture. Bronze stores the raw Lending Club data.
Silver stores cleaned and type-fixed data. Gold stores the train, validation,
and test tables.

This follows the same logic as our local pipeline, but it is more scalable.
Instead of only working as local scripts, the data is organized like a real data
engineering workflow.

We also added more macroeconomic data from FRED. We used national unemployment,
the federal funds rate, CPI, inflation, real interest rate, interest-rate
spread, and state unemployment for high-volume states.

The question was: do these macro features improve prediction?

The honest answer is: not really for validation AUC. The logistic regression
with loan features only got about **0.7136**. With extra FRED features, it got
about **0.7131**. That is slightly lower.

But this work still matters. The macro data gives useful insights. Default risk
changes with unemployment. Fed rate regimes affect weaker sub-grades more. And
state unemployment can show local economic risk.

So Phase 3 is valuable because it gives us scale and explanation, not just a
better number. Sometimes the honest result is, "We learned something, but the
metric did not clap for us."

**Transition:** Then we made the model usable through MCP.

### 6. Aayush Nepal - Phase 3: MCP, 1:15

This tab shows our MCP deployment.

MCP means Model Context Protocol. It lets an AI assistant call our model as a
tool.

Our MCP server loads the same preprocessing pipeline and the same XGBoost model
used by this Streamlit app. The user can describe a loan in plain English. The
tool builds the input row, runs the preprocessing pipeline, gets the model
prediction, and returns default probability, risk tier, and recommendation.

The important point is consistency. The Streamlit app and the MCP server use
the same model logic. We are not showing a fake demo model. It is the same
pipeline, just with a different interface.

This means analysts can use the visual app, and other users can ask questions
in natural language.

**Transition:** Now I will show the live scoring app.

### 7. Aayush Nepal - Predict a Loan, 2:00

This tab is the live model demo.

I will use one preset and click **Predict default risk**. The gauge shows the
predicted probability of default. The app also gives a risk tier and a simple
recommendation: approve, review, or decline.

On the right side, we show why the model made this prediction. Red bars push
toward default. Green bars push toward repayment.

This is useful because the model is not just giving a number. It also gives a
reason. That helps a human trust the output more.

We can also change values like FICO score, interest rate, loan term, or
sub-grade. The prediction updates. So this becomes a what-if tool.

There is also a compare mode. We can compare two loans side by side, like a
safe A-grade borrower and a risky E-grade borrower. This makes the model easier
to explain. It is also more fun than staring at a CSV file, which is a low bar,
but still important.

**Transition:** Junwei will close with the main insights and next steps.

### 8. Junwei Zhang - Insights + Next, 1:45

The first main insight is that Lending Club's own sub-grade is very strong.
Default rate goes from about **6% for A1** to about **45% for G5**.

Our model improves on that by using more information: credit history, income,
term, utilization, home ownership, and macro context.

The feature importance also shows that `sub_grade` is the strongest feature.
But other features still matter. For example, 60-month loans are usually riskier
than 36-month loans. Renting also tends to push risk higher compared with
mortgage ownership, when other things are similar.

We also want to be honest about limitations. We did not do a fairness audit.
The model is trained on 2014 to 2017 data, so we did not test it on COVID-era
loan behavior. Also, the threshold should be tuned for real business costs
before deployment.

For next steps, we would build a FastAPI scoring service, add fairness testing,
create adverse-action reason codes, retrain the model regularly, and test more
macro stress scenarios.

So our final answer is yes: we built a model that estimates loan default risk,
explains the drivers, and works in both a visual app and a conversational tool.

Thank you.

---

## Q&A, Extra Time

Q&A is outside the 15-minute presentation time.

Open the Q&A tab after the 15-minute talk. The likely-questions section in the
app can be used as a cheat sheet.

Simple backup answers:

- **Why XGBoost?** It performed best and gives useful feature contributions.
- **Why did macro features not improve AUC?** The model already has strong loan
  features, and the extra macro features may overlap with existing signals.
- **Can this be deployed?** Yes. The MCP server already shows the loading and
  prediction pattern. A FastAPI service would be a natural next step.
- **Why no fairness audit?** It was out of scope, but it is one of the most
  important next steps.
- **How should the threshold be chosen?** It should match business costs, not
  just one fixed academic metric.

If you do not know an answer, say: "Good question. We did not test that yet,
but it would be a good next step." This is much safer than inventing something.

---

## Rehearsal Checklist

- [ ] Run the app before the presentation.
- [ ] Open the Welcome tab first.
- [ ] Pre-load the Predict a Loan tab once.
- [ ] Test one preset in Predict a Loan.
- [ ] Test the threshold slider once.
- [ ] Practice once with a timer.
- [ ] Keep the main talk at 15 minutes.
- [ ] Do Q&A after the 15-minute talk.
