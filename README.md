# Sales & Pipeline Analytics Platform

### Lead Sources → CRM Pipeline → Power BI

An end-to-end sales and pipeline analytics solution: one star-schema model and a 2-page Power BI dashboard that shows where leads come from, where they drop off, and what revenue is actually closing.

Companion project: [marketing-performance-intelligence-platform](https://github.com/gunjannkhatri/marketing-performance-intelligence-platform) (ad spend, CPL, CAC). This one picks up where that ends, after the lead lands in the CRM.

---

## Business Problem

Leads come in from many places (Google Ads, Meta Ads, organic search, Instagram, referrals, WhatsApp, email), but the sales side is usually tracked in a CRM export or a spreadsheet nobody has time to analyze.

This project answers:

- How many leads are we getting, and which sources actually turn into revenue?
- Where do leads drop off in the funnel?
- How are we tracking against the monthly revenue and lead targets?
- How healthy is the open pipeline, and which deals are going stale?
- Why are we losing deals?

---

## Architecture

```
CRM / Sheet export ──► Power Query (M) ──► Star Schema ──► Power BI
(leads, stages, owners)   clean + type      Fact + Dims     2-page dashboard
                                                            scheduled refresh
```

In a real setup the CSV files are replaced by a CRM export or API pull (LeadSquared, Zoho, HubSpot) through Power Automate or a SQL view. The model and DAX don't change.

---

## Data Model (Star Schema)

### Dimension Tables

| Table | Purpose |
| --- | --- |
| Dim_Date | Shared calendar (day, week, month) |
| Dim_Source | Lead source and source group (Paid, Organic, Social, Referral, Direct, Email) |
| Dim_Owner | Sales rep |
| Dim_Product | Package / service line |
| Dim_Stage | Ordered pipeline stages with win probability |

### Fact Tables

| Table | Grain |
| --- | --- |
| Fact_Leads | One row per lead |
| Fact_Targets | One row per month (revenue target, lead target) |

### Relationships

- Dim_Date → Fact_Leads on CreatedDate (active, cohort view)
- Dim_Date → Fact_Leads on WonDate (inactive, revenue by close date)
- Dim_Date → Fact_Leads on LostDate (inactive, losses by close date)
- Dim_Source, Dim_Owner, Dim_Product, Dim_Stage → Fact_Leads
- Dim_Date → Fact_Targets on MonthStart

---

## Key DAX Measures

| Measure | Logic |
| --- | --- |
| Win Rate | Won / (Won + Lost) |
| Lead to Won % | Won / Total Leads |
| Revenue | Sum of won deal value, by close date |
| Avg Deal Size | Revenue / Deals Closed Won |
| Avg Sales Cycle | Average days from lead created to won |
| Revenue vs Target % | Revenue / Monthly Target |
| Funnel Leads | Leads that reached at least each stage |
| Weighted Pipeline | Open deal value x stage probability |
| Stalled Deals | Open deals with no activity for 14+ days |

Full DAX code in `/dax/measures.md`

---

## Key Insights (from sample data)

- 1,943 leads produced 201 won deals: about ₹1.47 Cr in revenue, ~₹73K average deal size, ~25-day average sales cycle
- Biggest funnel drop is Contacted → Qualified: only 53% of contacted leads get qualified
- Google Ads brings 25% of leads but 28% of revenue; Instagram brings 13% of leads but only 9% of revenue and has the lowest win rate (8.8%)
- Referral has the highest win rate (13.7%) but only 9% of lead volume, so it is an under-used source
- 80% of lost deals are lost early (no response, budget too low, not a fit), which points to lead quality and follow-up speed, not the closing stage
- Open pipeline is 190 deals worth ~₹1.63 Cr, but weighted value is only ~₹55.7 L, and 24% of open deals (45) have had no activity for 14+ days
- Win rate ranges from 8.6% to 14.7% across the five sales reps

---

## Dashboard

### Page 1: Pipeline Overview

- KPI cards: Leads, Win Rate, Revenue, Avg Deal Size, Avg Sales Cycle
- Monthly revenue vs target
- Pipeline funnel (New Lead → Won) with step conversion %
- Leads and win rate by source
- Sales rep performance table

### Page 2: Deal Health

- Open pipeline by stage (value and weighted value)
- Stalled deals list
- Lost reasons breakdown
- Win rate by product and by source group

Layout details in `/docs/dashboard_layout.md`

![Pipeline Overview](docs/dashboard_pipeline_overview.png)

![Deal Health](docs/dashboard_deal_health.png)

---

## Tools Used

| Tool | Purpose |
| --- | --- |
| Power BI Desktop | Data modeling, DAX, dashboard |
| Power Query (M) | Data transformation |
| DAX | Calculated measures |
| Power Automate | Automated refresh pipeline (production option) |
| CSV / CRM export | Source data |

---

## Files

- `/data/` : Sample CSV files (synthetic data, not real company data)
- `/dax/measures.md` : All DAX measures
- `/powerquery/m_code.md` : Power Query M code for every table
- `/docs/` : Dashboard layout guide and screenshots

---

## How to Use

1. Clone the repo and open Power BI Desktop.
2. Create a `BaseUrl` parameter pointing to the `/data/` folder (see `/powerquery/m_code.md`).
3. Load the 7 tables, set up the relationships listed above, and mark `Dim_Date` as a date table.
4. Paste the measures from `/dax/measures.md` into a `_Measures` table.
5. Build the two pages using `/docs/dashboard_layout.md`.

---

## Note on Data

All data in this repository is synthetically generated for demonstration purposes. It does not represent real company data. Numbers are realistic for a small business generating leads from multiple channels but are not sourced from any real CRM.

---

## Author

**Gunjan**, Data Analyst & Power BI Developer
[LinkedIn](https://www.linkedin.com/in/gunjan-khatri-00b1242ba) | [Portfolio](https://github.com/gunjannkhatri)
