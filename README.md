# Calculating-Support-and-Resistance-levels-in-Bank-Nifty
a technical, step-by-step plan to calculate Support and Resistance levels using Bank Nifty option chain data on Databricks, along with an explanation of why Databricks is ideal for this task.


🧠 What Are Support and Resistance in Option Chain Terms?
Support and resistance levels are inferred using:

High Open Interest (OI):

High Put OI ⇒ Support level (buyers expect price to stay above)

High Call OI ⇒ Resistance level (sellers expect price to stay below)

Change in OI:

Rising OI + Rising Price ⇒ Strengthening support/resistance

OI unwinding ⇒ Weakening level

🏗️ Why Use Databricks?
Value Add	Why It Matters
Delta Lake	Versioned, fast, scalable option chain snapshots
Structured Streaming	Ingest live option chain data with mini-batch streaming
Databricks SQL	Easy to write analytical queries over bronze/silver/gold data
Notebooks	Combine ingestion, logic, modeling, and visualization
MLflow	Train and deploy ML models (e.g., predictive support/resistance)
Unity Catalog	Fine-grained access control and data lineage

📘 Step-by-Step Plan
✅ Step 1: Ingest Option Chain Data
Set up a Databricks Job (or Workflow) that runs every 5 minutes to fetch option chain data from NSE API.

python
Copy
Edit
import requests
import pandas as pd

url = "https://www.nseindia.com/api/option-chain-indices?symbol=BANKNIFTY"
headers = {"User-Agent": "Mozilla/5.0"}
res = requests.get(url, headers=headers).json()

df = pd.json_normalize(res['records']['data'])
df = df.explode("CE").explode("PE")
Save to Bronze Layer:

python
Copy
Edit
(spark.createDataFrame(df)
 .write.format("delta")
 .mode("append")
 .save("/mnt/bronze/banknifty_optionchain"))
✅ Step 2: Clean and Normalize (Silver Layer)
Parse essential columns:

python
Copy
Edit
df_silver = df.selectExpr(
    "CE.strikePrice as strike",
    "CE.expiryDate as expiry",
    "CE.openInterest as call_OI",
    "PE.openInterest as put_OI",
    "CE.changeinOpenInterest as call_OI_change",
    "PE.changeinOpenInterest as put_OI_change"
)
Write to Delta:

python
Copy
Edit
df_silver.write.format("delta").mode("overwrite").save("/mnt/silver/banknifty_oi")
✅ Step 3: Calculate Support and Resistance Levels
In a Gold Table, compute:

sql
Copy
Edit
-- Create a view from Silver
CREATE OR REPLACE TEMP VIEW options_cleaned AS
SELECT strike, expiry, call_OI, put_OI
FROM delta.`/mnt/silver/banknifty_oi`
WHERE expiry = '2025-05-30' -- nearest expiry
Now, calculate:

sql
Copy
Edit
-- Support level (max Put OI)
SELECT strike AS support_strike
FROM options_cleaned
ORDER BY put_OI DESC
LIMIT 1;

-- Resistance level (max Call OI)
SELECT strike AS resistance_strike
FROM options_cleaned
ORDER BY call_OI DESC
LIMIT 1;
You can enrich this further:

sql
Copy
Edit
SELECT strike,
       call_OI,
       put_OI,
       CASE WHEN put_OI > call_OI THEN 'Support Bias' ELSE 'Resistance Bias' END as level_type
FROM options_cleaned
ORDER BY strike;
✅ Step 4: Visualize OI Distribution
Use Plotly or Databricks SQL Dashboard:

python
Copy
Edit
import plotly.graph_objects as go

fig = go.Figure()
fig.add_trace(go.Bar(x=df['strike'], y=df['call_OI'], name='Call OI'))
fig.add_trace(go.Bar(x=df['strike'], y=df['put_OI'], name='Put OI'))
fig.update_layout(barmode='group', title='Bank Nifty Option Chain OI')
fig.show()
✅ Step 5: Automate with Databricks Jobs + Alerts
Use Databricks Workflows to run this every N minutes.

Write current support/resistance to a gold table.

Set alerts: if support/resistance levels change, send a Slack/Email.

⚙️ Advanced (Optional) Steps
✅ Predictive Support/Resistance (ML)
Train a classifier to predict whether a strike will become a key level based on:

Delta, IV, OI change, previous support/resistance positions.

Log with MLflow:

python
Copy
Edit
with mlflow.start_run():
    mlflow.sklearn.log_model(model, "support_resistance_predictor")
📊 What You'll Have at the End
Output	Description
bronze_banknifty_optionchain	Raw option chain snapshots
silver_banknifty_oi	Cleaned and structured OI table
gold_banknifty_levels	Computed support/resistance per expiry
Dashboards	OI bar charts, level history over time
ML model (optional)	Predictive level classification

💡 Real-World Applications
Live support/resistance tracker on a dashboard

Alerts for strikes becoming new key levels

Use in automated trading strategies or signal generation


