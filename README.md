# Customer Churn Analytics with Alteryx Designer

> **Project type:** Customer churn analytics, feature engineering, KPI aggregation and reporting preparation  
> **Platform:** Alteryx Designer  
> **Data:** Synthetic practice dataset (117 customer records; five related tables)  
> **Current project status:** Data integration and descriptive churn KPI workflow documented. A trained predictive model has **not** been produced in the workflow shown here.

---

## Table of contents

- [1. Project overview](#1-project-overview)
- [2. Business problem](#2-business-problem)
- [3. Dataset and relationships](#3-dataset-and-relationships)
- [4. End-to-end workflow](#4-end-to-end-workflow)
- [5. Data preparation and quality](#5-data-preparation-and-quality)
- [6. Feature engineering](#6-feature-engineering)
- [7. KPI aggregation branches](#7-kpi-aggregation-branches)
- [8. KPI formulas](#8-kpi-formulas)
- [9. Combining branch outputs](#9-combining-branch-outputs)
- [10. Results and interpretation](#10-results-and-interpretation)
- [11. Predictive modelling: what remains](#11-predictive-modelling-what-remains)
- [12. Reporting and visualisation](#12-reporting-and-visualisation)
- [13. How to reproduce](#13-how-to-reproduce)
- [14. Data governance and modelling cautions](#14-data-governance-and-modelling-cautions)
- [15. Repository structure](#15-repository-structure)
- [16. Future improvements](#16-future-improvements)

---

## 1. Project overview

This project uses Alteryx Designer to build a customer-level analytics workflow from several related source tables. The workflow reads customer, purchase, engagement, support and churn-label data; joins the tables using `customer_id`; creates analytical features; and calculates descriptive churn KPIs across customer segments, acquisition channels and spending groups.

The goal is to demonstrate a practical analytics pipeline:

1. Read multiple source datasets.
2. validate and standardise their fields.
3. Integrate related tables using their customer key.
4. Engineer customer-level features.
5. Aggregate metrics for business analysis.
6. Calculate churn rates and customer-value categories.
7. combine the KPI outputs for review and downstream reporting.

**Important distinction:** The current workflow is a descriptive analytics and KPI-reporting workflow. A churn label is present in the practice data, but the screenshots do not show a fitted Logistic Regression, Random Forest or other trained model, nor model validation metrics. Therefore, this repository does not claim that a predictive model has been trained or deployed.

## 2. Business problem

A business wants to understand customer attrition and identify where retention efforts may be most useful.

The analysis is designed to answer questions such as:

- What proportion of customers are marked as churned?
- Which customer segment has the highest observed churn rate?
- Does observed churn vary by acquisition channel?
- How do average spending, customer lifetime value (CLV) and purchase frequency vary across groups?
- Which groups may deserve further retention investigation?

These are descriptive questions. They identify patterns in the available data; they do not establish causation.

## 3. Dataset and relationships

The practice workbook contains five related data tables. The supplied practice dataset is synthetic and contains 117 customer records. The workbook and separate CSV source files are included under `data/synthetic_sample/` for practice and reproducibility.

| Source table | Main purpose | Key fields |
|---|---|---|
| `Customers` | Customer profile and acquisition details | `customer_id`, `customer_segment`, `customer_country`, `customer_city`, `customer_age_months`, `signup_date`, `acquisition_channel`, `customer_age_band`, `preferred_device`, `customer_status` |
| `Purchase_Metrics` | Customer purchase and monetary behaviour | `purchase_metric_id`, `customer_id`, `total_spend_usd`, `avg_order_value_usd`, `num_purchases`, `days_since_last_purchase`, `customer_lifetime_value_usd`, `last_purchase_date` |
| `Engagement` | Email, social and loyalty engagement | `engagement_id`, `customer_id`, `emails_sent`, `emails_opened`, `emails_clicked`, `email_conversion_rate`, `social_media_engagement_score`, `loyalty_score`, `referral_count` |
| `Customer_Support` | Customer service and satisfaction | `support_id`, `customer_id`, `satisfaction_score`, `satisfaction_level`, `support_ticket_count`, `returns_count`, `support_interactions`, `feedback_score`, `support_channel` |
| `Churn_Labels` | Churn outcome and related label attributes | `churn_id`, `customer_id`, `is_churned`, `churn_flag`, `days_since_churn`, `churn_date`, `churn_reason`, `churn_category`, `churn_risk_score`, `churn_probability`, `churn_revenue_impact_usd`, `customer_health_score` |

### Relationship design

`customer_id` is the customer-level business key. The `Customers` table is treated as the master table and the other tables are matched to it using `customer_id`.

Conceptual relationship:

```text
                         Customers
                             |
              customer_id (master customer key)
                             |
        +--------------------+--------------------+
        |                    |                    |
  Purchase_Metrics       Engagement        Customer_Support
        |                    |                    |
        +--------------------+--------------------+
                             |
                       Churn_Labels
```

In Alteryx, the workflow uses Join tools to combine the differently structured tables horizontally. A Union tool is not appropriate for joining these five different schemas; Union is for vertically appending records with compatible columns.

**Join validation:** Check that each input has the intended grain and that `customer_id` is unique where one-row-per-customer is expected. If a production purchase or support table has multiple rows per customer, aggregate it to customer level before joining. Otherwise, joining multiple one-to-many tables can multiply rows (join fan-out).

## 4. End-to-end workflow

The workflow is organised into the following logical stages:

| Stage | Work performed | Typical Alteryx tools |
|---|---|---|
| Source ingestion | Read the source CSVs or workbook sheets | Input Data |
| Source quality | Inspect fields, types, nulls and duplicate keys | Select, Data Cleansing, Unique, Browse |
| Integration | Join source tables using `customer_id` | Join |
| Post-join checks | Check row counts, key uniqueness and missing joined values | Browse, Unique, Formula, Filter |
| Feature engineering | Create engagement, purchase and support indicators | Formula |
| KPI analysis | Aggregate customers and churn labels by group | Summarize |
| KPI calculations | Calculate churn rates, percentages and categories | Formula |
| Precision formatting | Round selected numeric fields for presentation | Multi-Field Formula |
| Branch integration | Combine KPI result tables with compatible schemas | Join Multiple / Join, Select |
| Review and export | Inspect and save results | Browse, Output Data |

### Overall workflow screenshot

![Overall Alteryx workflow](screenshots/01_overall_workflow.png)

The screenshot represents the current canvas structure. It is a visual record of the workflow, not a substitute for the native `.yxmd` workflow file.

## 5. Data preparation and quality

The source tables have different schemas, so each source should be checked independently before integration.

Recommended checks:

- Confirm field names and data types.
- Convert numeric measures from text to numeric types before calculations.
- Parse dates into Date or DateTime fields.
- Check nulls in keys and important measures.
- Check duplicate primary/business keys.
- Confirm `customer_id` values in child tables match the customer master.
- Compare input and output row counts after each join.
- Review unmatched records rather than silently dropping them.

### Important key-field checks

- `customer_id`: retain in every source used for joining.
- Table-specific identifiers (`purchase_metric_id`, `engagement_id`, `support_id`, `churn_id`): retain only if needed for audit or record-level traceability. They are not normally model features.
- `*Unknown`: this is an Alteryx metadata placeholder in some Select configurations, not a business field to use in analysis. Remove or leave it unchecked when appropriate.

## 6. Feature engineering

The Formula tool is used to create row-level features before the aggregation branches. The screenshots show calculations including:

### Email open rate

```alteryx
IF IsNull([emails_sent]) OR [emails_sent] = 0 THEN 0
ELSE [emails_opened] / [emails_sent]
ENDIF
```

### Email click rate

```alteryx
IF IsNull([emails_sent]) OR [emails_sent] = 0 THEN 0
ELSE [emails_clicked] / [emails_sent]
ENDIF
```

### Purchase frequency per customer tenure

```alteryx
IF IsNull([customer_age_months]) OR [customer_age_months] <= 0 THEN 0
ELSE [num_purchases] / [customer_age_months]
ENDIF
```

### Average monthly spend

```alteryx
IF IsNull([customer_age_months]) OR [customer_age_months] <= 0 THEN 0
ELSE [total_spend_usd] / [customer_age_months]
ENDIF
```

### Support issue rate per month of tenure

```alteryx
IF IsNull([customer_age_months]) OR [customer_age_months] <= 0 THEN 0
ELSE [support_ticket_count] / [customer_age_months]
ENDIF
```

### Recency category

```alteryx
IF IsNull([days_since_last_purchase]) THEN "Unknown"
ELSEIF [days_since_last_purchase] <= 30 THEN "Recent"
ELSEIF [days_since_last_purchase] <= 60 THEN "Moderate"
ELSEIF [days_since_last_purchase] <= 90 THEN "At Risk"
ELSE "Inactive"
ENDIF
```

### Customer tenure category

```alteryx
IF IsNull([customer_age_months]) THEN "Unknown"
ELSEIF [customer_age_months] <= 3 THEN "New Customer"
ELSEIF [customer_age_months] <= 12 THEN "Growing Customer"
ELSEIF [customer_age_months] <= 24 THEN "Established Customer"
ELSE "Long-Term Customer"
ENDIF
```

Feature engineering screenshots:

![Feature engineering formulas - engagement, purchase and support](screenshots/02_feature_engineering_email_purchase_support.png)

![Feature engineering formulas - customer categories](screenshots/03_feature_engineering_customer_categories.png)

**Formula note:** These are customer-level derived indicators. Check the business meaning and units before using them. For example, `customer_age_months` must be positive and measured consistently.

## 7. KPI aggregation branches

After feature engineering, the workflow branches into separate Summarize tools. This is intentional: each branch answers a different business question and can have a different grouping level.

### Branch A — Churn by customer segment

**Business question:** How many customers and churned customers are there in each segment?

Summarize configuration:

| Field | Action | Meaning |
|---|---|---|
| `customer_segment` | Group By | One output row per segment |
| `is_churned` | Sum | Number of churned customers when the field is 0/1 and there is one row per customer |
| `customer_id` | Count Distinct | Number of unique customers |

Then use Formula:

```alteryx
IF [CountDistinct_customer_id] = 0 THEN 0
ELSE [Sum_is_churned] / [CountDistinct_customer_id]
ENDIF
```

Name the result `Segment_churn_rate`.

For percentage:

```alteryx
[Segment_churn_rate] * 100
```

Configuration evidence:

![Segment churn Summarize configuration](screenshots/04_segment_churn_summarize_configuration.png)

![Segment churn formula configuration](screenshots/07_segment_churn_formula_configuration.png)

### Branch B — Churn by acquisition channel

**Business question:** Does observed churn vary across acquisition channels?

Summarize configuration:

| Field | Action | Meaning |
|---|---|---|
| `acquisition_channel` | Group By | One output row per channel |
| `is_churned` | Sum | Churned customer count |
| `customer_id` | Count Distinct | Unique customer count |
| `total_spend_usd` | Average | Average spend for the group |

Formula for channel churn rate:

```alteryx
IF [CountDistinct_customer_id] = 0 THEN 0
ELSE [Sum_is_churned] / [CountDistinct_customer_id]
ENDIF
```

Percentage:

```alteryx
[Channel_Churn_Rate] * 100
```

Configuration evidence:

![Acquisition channel Summarize configuration](screenshots/05_channel_churn_summarize_configuration.png)

![Acquisition channel formula configuration](screenshots/08_channel_churn_formula_configuration.png)

### Branch C — Spending and customer value analysis

**Business question:** How do average spend, average CLV and purchase frequency differ by segment?

Recommended Summarize configuration:

| Field | Action | Output |
|---|---|---|
| `customer_segment` | Group By | `customer_segment` |
| `total_spend_usd` | Average | `Avg_total_spend_usd` |
| `customer_lifetime_value_usd` | Average | `Avg_customer_lifetime_value_usd` |
| `num_purchases` | Average | `Avg_num_purchases` |
| `is_churned` | Sum | `Sum_is_churned` |
| `customer_id` | Count Distinct | `CountDistinct_customer_id` |

The screenshots document the aggregation and the rounding/CLV categorisation steps:

![Spending analysis Summarize configuration](screenshots/06_spending_analysis_summarize_configuration.png)

![Spending category formula configuration](screenshots/09_spending_category_formula_configuration.png)

![Customer lifetime value category formula configuration](screenshots/10_customer_lifetime_value_formula_configuration.png)

## 8. KPI formulas

### Churn rate

For any group:

```alteryx
IF [CountDistinct_customer_id] = 0 THEN 0
ELSE [Sum_is_churned] / [CountDistinct_customer_id]
ENDIF
```

### Churn percentage

```alteryx
[Churn_Rate] * 100
```

Use the appropriate rate field name for the branch, such as `Segment_churn_rate` or `Channel_Churn_Rate`.

### Spending category

The current workflow screenshot shows a threshold-based spending classification. The example expression below uses ordered thresholds; adjust the thresholds to the scale and distribution of the actual data:

```alteryx
IF [Avg_total_spend_usd] >= 5500 THEN "High Spend"
ELSEIF [Avg_total_spend_usd] >= 5000 THEN "Medium Spend"
ELSE "Low Spend"
ENDIF
```

### CLV category

The screenshot shows a CLV category based on average customer lifetime value:

```alteryx
IF [Avg_customer_lifetime_value_usd] >= 11000 THEN "High Value"
ELSEIF [Avg_customer_lifetime_value_usd] >= 8000 THEN "Medium Value"
ELSE "Low Value"
ENDIF
```

Thresholds are illustrative business rules, not statistically validated cut-offs. In a production analysis, define them with business stakeholders or use quantiles/segments based on the data distribution.

### Rounding numeric measures

The Multi-Field Formula tool can apply a consistent rounding expression to multiple selected numeric fields.

For two decimal places:

```alteryx
Round([_CurrentField_], 0.01)
```

If **Copy Output Fields and Add** is enabled, the tool creates copies. A prefix such as `Rounded_` is clearer than `New_`. Do not apply decimal rounding to count fields or binary target fields.

## 9. Combining branch outputs

The KPI branches have different grains and dimensions:

- Segment branch: one row per customer segment.
- Channel branch: one row per acquisition channel.
- Spending branch: one row per customer segment.

These are not naturally the same table. Before combining them, decide what the final structure should be.

**Recommended approach:** Keep separate outputs/tables for segment, channel and spending analysis. This preserves each table's meaning and makes reporting easier.

If a single consolidated table is required, standardise the branches first:

1. Add a common `Analysis_Type` field (for example `Segment`, `Channel`, `Spend`).
2. Add a common `Category` field (segment name, channel name, or segment name).
3. Rename common metrics consistently.
4. Add typed nulls for metrics that do not apply to a branch.
5. Use Union configured by field name only after all three outputs have compatible schemas.

A Join Multiple tool can combine branches only when the join keys and row grain make sense. Do not join unrelated groupings merely to force one wide table. A join between a segment-level table and channel-level table on no meaningful shared key can create misleading combinations or a many-to-many result.

The workflow screenshot shows the KPI branches being brought together for a combined preview. Validate the join keys, row counts and expected output grain carefully:

![KPI branches and final output preview](screenshots/11_kpi_branches_and_join_multiple.png)

## 10. Results and interpretation

The screenshot preview shows five acquisition-channel rows and columns including:

- `acquisition_channel`
- `CountDistinct_customer_id`
- `Sum_is_churned`
- `Avg_total_spend_usd`
- `Channel_Churn_Rate`
- `Channel_churn%`
- `Spend_Category`

The displayed preview includes the following channel-level values:

| Acquisition channel | Unique customers | Churned count | Average total spend | Churn rate | Churn % |
|---|---:|---:|---:|---:|---:|
| Direct | 27 | 2 | 4,462.143 | 0.074 | 7.407 |
| Social Media | 30 | 4 | 5,727.695 | 0.133 | 13.333 |
| Paid Ads | 19 | 2 | 5,946.507 | 0.105 | 10.526 |
| Organic Search | 17 | 3 | 4,319.912 | 0.176 | 17.647 |
| Referral | 24 | 1 | 5,623.888 | 0.042 | 4.167 |

These values are transcribed from the visible Alteryx Browse preview. Small differences may appear if the underlying workflow or dataset is changed.

### Initial descriptive observations

- Organic Search has the highest observed channel churn percentage in this preview (about 17.65%).
- Referral has the lowest observed channel churn percentage in this preview (about 4.17%).
- Social Media has the largest customer count in this preview (30).
- Paid Ads has the highest average spend in this preview (about 5,946.51).

These are descriptive observations from a small synthetic practice dataset. They should not be treated as causal conclusions or as evidence of real customer behaviour.

![Final KPI output preview](screenshots/12_final_kpi_output_preview.png)

![Final workflow overview](screenshots/13_final_workflow_overview.png)

## 11. Predictive modelling: what remains

A genuine churn prediction project requires a model training and evaluation stage. The Alteryx screenshots provided for this repository do not show a trained predictive model. The next stage should be completed only after confirming the available tools/licensing or using an external modelling environment.

Suggested next steps:

1. Define `is_churned` as the target (0 = not churned, 1 = churned).
2. Define a prediction observation date and an outcome window.
3. Use only information available at the prediction date.
4. Exclude target duplicates and post-outcome fields.
5. Split data into train/test sets (prefer a time-based split if dates permit).
6. Train a baseline Logistic Regression model.
7. Evaluate with confusion matrix, precision, recall, F1, ROC-AUC and PR-AUC.
8. Review class imbalance and select a threshold based on business costs.
9. Score unseen customers and create a retention-priority output.

### Potential target leakage

Do not use these fields as predictors if they are derived from or recorded after the churn outcome:

- `churn_flag` (duplicate representation of the target)
- `days_since_churn`
- `churn_date`
- `churn_reason`
- `churn_category`
- `churn_risk_score` (if generated using the outcome or future information)
- `churn_probability` (if generated from an existing model or target information)
- `churn_revenue_impact_usd` (if calculated after churn)
- `customer_health_score` (verify how and when it was calculated)

Use `is_churned` as the target label, not as a feature. Ensure that predictors such as purchase recency, spend and engagement are measured strictly before the prediction date.

## 12. Reporting and visualisation

The aggregated KPI tables can be used to create a management report in Alteryx Reporting tools or a dashboard in Power BI.

Suggested report components:

| Report section | Recommended visual |
|---|---|
| Overall churn | KPI cards or a one-row summary table |
| Churn by segment | Bar chart |
| Churn by acquisition channel | Sorted bar chart |
| Average spend by segment | Column chart |
| Average CLV by segment | Column chart |
| Customer-value category | Stacked bar or summary table |

In Alteryx, a typical static reporting sequence is:

```text
Summarize / Formula outputs
          |
      Table / Chart
          |
     Report Text
          |
        Layout
          |
        Render
          |
     PDF / HTML report
```

Reporting tools produce formatted report output; they do not automatically create an interactive Power BI-style dashboard with slicers.

## 13. How to reproduce

1. Install and open Alteryx Designer.
2. Download or prepare the synthetic source workbook/CSV files.
3. Point each Input Data tool to its source table.
4. Configure the joins on `customer_id`.
5. Add source-level and post-join data quality checks.
6. Configure Formula tools for the row-level features.
7. Create the Summarize branches as documented above.
8. Add Formula tools to calculate churn rates and business categories.
9. Round only presentation metrics where needed.
10. Inspect each branch with Browse.
11. Save branch outputs separately or standardise schemas before Union.
12. Export KPI tables and build a report.

**Reproduction note:** The screenshots are evidence of the user-interface configuration and outputs. The native Alteryx `.yxmd` workflow file was not included in the supplied materials, so this repository cannot execute the workflow by itself. Recreate or add the `.yxmd` file in the repository if you want a fully runnable Alteryx project.

## 14. Data governance and modelling cautions

- The practice data is synthetic and is intended for learning/demo purposes.
- Do not publish real customer personally identifiable information (PII), account details or confidential banking data to a public repository.
- Verify key uniqueness and row grain before joins.
- Validate data types before mathematical operations.
- Avoid using target-derived fields as model predictors.
- Document the prediction date and target window for any future model.
- Report sample sizes alongside churn rates; a high rate from a very small group can be unstable.
- Treat correlations and group differences as descriptive, not causal.

## 15. Repository structure

```text
Customer-Churn-Prediction-Alteryx-GitHub/
│
├── README.md
├── .gitignore
├── LICENSE
├── requirements.txt
├── PROJECT_NOTES.md
├── data/
│   └── synthetic_sample/
│       ├── Customer_Churn_Analysis_Data.xlsx
│       ├── Customers.csv
│       ├── Purchase_Metrics.csv
│       ├── Engagement.csv
│       ├── Customer_Support.csv
│       └── Churn_Labels.csv
└── screenshots/
    ├── 01_overall_workflow.png
    ├── 02_feature_engineering_email_purchase_support.png
    ├── 03_feature_engineering_customer_categories.png
    ├── 04_segment_churn_summarize_configuration.png
    ├── 05_channel_churn_summarize_configuration.png
    ├── 06_spending_analysis_summarize_configuration.png
    ├── 07_segment_churn_formula_configuration.png
    ├── 08_channel_churn_formula_configuration.png
    ├── 09_spending_category_formula_configuration.png
    ├── 10_customer_lifetime_value_formula_configuration.png
    ├── 11_kpi_branches_and_join_multiple.png
    ├── 12_final_kpi_output_preview.png
    └── 13_final_workflow_overview.png
```

## 16. Future improvements

- Add the native Alteryx `.yxmd` workflow.
- Add automated data-quality exception outputs.
- Add a reusable macro for source validation or repeated KPI calculations.
- Add a time-aware churn prediction dataset and train/test split.
- Train and validate Logistic Regression and tree-based models.
- Add feature importance and model explainability.
- Add a retention action framework based on predicted risk and customer value.
- Add a Power BI dashboard or Alteryx PDF report.
- Add automated workflow scheduling and run logging where the Alteryx environment supports it.

---

## Author / portfolio note

This repository documents a hands-on Alteryx customer churn analytics project, covering multi-source integration, data quality, feature engineering, grouped KPI calculations and reporting preparation. It is suitable as a learning and portfolio demonstration. Update this section with your own name, contact details and final project status before publishing.
