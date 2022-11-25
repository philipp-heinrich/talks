### Optimise your GCP Data Platform for Cost and Performance
##### Philipp Heinrich 

> My Name is Philipp and I'm a Senior Cloud Data Architect at doit. In my day to day job I'm helping our customers succeeding in the Cloud - especially Google Cloud. 
---

/assets/agenda.png
size: contain
> Most often we talk about Cost & Performance Optimisations. They influence each other, and most often you can't have both.

---
### AGENDA
		[ ] How expensive are your Dashboards? 📊
		[ ] Speed up your dashboard queries 🚀 
		[ ] Slots, slots, slots. 🎰

> Todays talk is about three very specific methods: 
> This is **not** a general introduction into cost or performance optimisation. This is not about: "setup logging & monitoring, or write less queries, or don't use SELECT * in BigQuery
> I just picked three, that are not that obvious: Looker, BI Engine & Slot Scaling 

---

### How expensive are your Dashboards? 📊

> Wouldn't it be cool, to see how much every Dashboard actually costs you? Or how much query costs are associated with each Dashboard Development? 

> Especially, when you use BigQuery and are on the on-demand pricing model, you could easily sky-rocket your cloud bill with running queries in your BI Tool of choice. 

--- 
#### High level Architecture
/assets/dashboard_costs2.png
size: contain
y: center

> When talking about Looker, and I mean Looker and not Looker Studio, the data-studio replacement, every Job / every Query is logged in the System Activity Tables. The PK is called history_id and we can join a bunch of other tables against this history table 
> everything that is initiated by Looker and run on BigQuery gets annotated with some metadata:  History_ID + Instance ID 
> Can be found in Billing Export or Data Access Logs

---
### System Activity Data 
	- built into Looker
	- logging of all Queries 
	- last 90 days
	- a lot (!) of usable fields <br> (runtime, caching, branches, users, ...)
	- cannot be accessed programmatically 🤯 
	- Must be exported via <br> a Look + GCS + BQ

### Billing Data
	+ good for on-demand
	+ probably the easiest to set up
	+ consumption is always 0$ <br>when using flat-rate pricing 😪
 
### Data Access Logs
	+ good for flat-rate  
	+ setup a Log Sink & <br> export to BigQuery
	+ Data Protection?!
	+ BigQuery Lens via DoiT CMP ✅
--- 

```sql
CREATE TABLE looker_lens_us.agg AS
SELECT
    looker_data.*,
    (SELECT CAST(value AS INT64) FROM UNNEST(billing_export.labels) as lbl WHERE lbl.key = "looker-context-history_id" LIMIT 1) AS billing_export_history_id,
    billing_export.invoice.month  AS billing_export_invoice__month,
    billing_export.project.name  AS billing_export_project__name,
    billing_export.billing_account_id  AS billing_export_billing_account_id,
    billing_export.currency  AS billing_export_currency,
    billing_export.cost AS billing_export_total_cost,
    RAND()*10 as fake_costs
FROM `philipph-playground.looker_lens_us.lens_20221122` AS looker_data
LEFT JOIN `doitintl-cmp-gcp-data.gcp_billing_*` AS billing_export 
        ON looker_data.ID_6 = ((SELECT CAST(value AS INT64) FROM UNNEST(billing_export.labels) as label WHERE label.key = "looker-context-history_id" LIMIT 1))
WHERE DATE(billing_export.export_time) >= "2022-01-01"
```
---
#### Example
/assets/looker_cost.png
size: contain

---

### Speed up your Dashboard Queries 🚀

---

#### What is BI Engine
	- fast, **in-memory** analysis service
	- own query engine
	- not using on-demand / flat-rate pricing
	- support for: Console: ODBC/JDBC, API + client library
	- supported by Looker + Looker Studio + Tableau ..
	- only works with high-level analytical queries

/assets/bi_engine.png
size: contain
x: right
y: center

---

#### It's not great for....
	- Transformative Queries (ETL)
	- Joins
	- Large tables (~5GB / Million Rows)
	- UDFs
	- Analytic functions
	- External Tables 
	  
	*cloud.google.com/bigquery/docs/bi-engine-optimized-sql*

---

/assets/bi_engine_pricing.png
size: contain

/assets/high_costs.png
size: contain

---

/assets/job_results.png
size: contain
 
/assets/metric explorer.png
size: contain

/assets/bi_engine_dashboard.png
size: contain

---

### Slots, slots, slots. 🎰
> This is one of the most asked questions. 
> Slots are complex.
> Should I use slots? 
> What does it costs? 
> Does it make sense? 
> And I always answer with: It's complex and it depends

---
### Basics. 
	**on-demand**: 
	2000 slots on demand per project - pay-as-go $5 per TB 
	**flat-rate:**
	- 100 slots for $2000 per month *or*
	- 100 slots for $4 per hour
---
/assets/confused.gif

---	
### Example 1
	Your BI Tools is consuming on average 300 slots during working hours.

--- 	

### Monthly reservation 
```
	300*2000$ = 6000$ or 
	300*1700$ = 5100$ 
```

	*Tipp: Make a dedicated GCP Project for Analysts. Buy Slots at Org Level. Non-allocated slots can be redistributed to other projects automatically, but Analysts getting higher Priority*

---

### Example 2
	You have a heavy ETL batch job, that needs to run once a month to calculate invoice sums. 

---
### On-demand reservation 
/assets/flex-slots.png
background: false
size: contain
y: bottom

---
### RECAP
		[x] How expensive are your Dashboards? 📊
		[x] Speed up your dashboard queries 🚀 
		[x] Slots, slots, slots. 🎰