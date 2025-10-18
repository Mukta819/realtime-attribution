# realtime-attribution


---

## 🧩 **dbt models**

### `stg_events.sql`
```sql
SELECT
  event_timestamp AS event_ts,
  user_pseudo_id AS user_id,
  traffic_source.source AS source,
  traffic_source.medium AS medium,
  traffic_source.name AS campaign,
  event_name,
  event_params
FROM
  `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
WHERE event_name IN ('page_view', 'purchase');


----- int_sessions.sql

SELECT
  user_id,
  session_id,
  MIN(event_ts) AS session_start,
  MAX(event_ts) AS session_end
FROM {{ ref('stg_events') }}
GROUP BY user_id, session_id;


--- mart_attribution_first.sql

SELECT
  user_id,
  campaign,
  FIRST_VALUE(source) OVER (
    PARTITION BY user_id ORDER BY event_ts ASC
  ) AS first_source
FROM {{ ref('stg_events') }};


----- mart_attribution_last.sql

SELECT
  user_id,
  campaign,
  LAST_VALUE(source) OVER (
    PARTITION BY user_id ORDER BY event_ts ASC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS last_source
FROM {{ ref('stg_events') }};


---schema.yml

version: 2

models:
  - name: stg_events
    tests:
      - unique:
          column_name: event_ts
      - not_null:
          column_name: user_id

--stream_events.py

import json, time, random, uuid
from google.cloud import bigquery

client = bigquery.Client()

table_id = "your_project.ga4_stream.streamed_events"

events = ["page_view", "add_to_cart", "purchase"]
sources = ["google", "facebook", "email", "direct"]

while True:
    event = {
        "event_id": str(uuid.uuid4()),
        "user_id": random.randint(1000, 5000),
        "source": random.choice(sources),
        "event_name": random.choice(events),
        "timestamp": time.time()
    }
    errors = client.insert_rows_json(table_id, [event])
    print("Inserted:", event)
    time.sleep(3)


---Streamlit Dashboard (dashboard/app.py)


import streamlit as st
from google.cloud import bigquery
import pandas as pd

st.title("📈 Real-time Attribution Dashboard")

client = bigquery.Client()
query = """
SELECT source, COUNT(*) as total_events
FROM `your_project.ga4_stream.streamed_events`
GROUP BY source
ORDER BY total_events DESC
"""
data = client.query(query).to_dataframe()

st.bar_chart(data, x='source', y='total_events')

st.write("Live Events:")
events = client.query("SELECT * FROM `your_project.ga4_stream.streamed_events` ORDER BY timestamp DESC LIMIT 10").to_dataframe()
st.dataframe(events)








