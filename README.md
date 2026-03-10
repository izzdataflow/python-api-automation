# 📈 Python API Automation — Crypto Price Tracker (CoinMarketCap)

## Preface

This project builds an automated **cryptocurrency data pipeline** using the CoinMarketCap API. It fetches live price and percent-change data for the top cryptocurrencies, logs every API pull with a timestamp into a CSV file, and visualizes price trends over time using Seaborn.

The steps follow a natural build order — connect to the API, normalize the JSON response into a DataFrame, wrap everything into a reusable function that saves to CSV, run it on a schedule, then explore and visualize the collected data.

---

## 📋 Table of Contents

1. [Connect to CoinMarketCap API](#1-connect-to-coinmarketcap-api)
2. [Configure Pandas Display Settings](#2-configure-pandas-display-settings)
3. [Normalize JSON Response into DataFrame](#3-normalize-json-response-into-dataframe)
4. [Wrap Everything into a Reusable Function](#4-wrap-everything-into-a-reusable-function)
5. [Run the API on a Schedule](#5-run-the-api-on-a-schedule)
6. [Read the Saved CSV](#6-read-the-saved-csv)
7. [Fix Scientific Notation Display](#7-fix-scientific-notation-display)
8. [Group by Coin & Average Percent Changes](#8-group-by-coin--average-percent-changes)
9. [Stack Wide DataFrame to Long Format](#9-stack-wide-dataframe-to-long-format)
10. [Check the Data Type](#10-check-the-data-type)
11. [Convert Series to DataFrame](#11-convert-series-to-dataframe)
12. [Count Rows](#12-count-rows)
13. [Reset the Index](#13-reset-the-index)
14. [Rename Column](#14-rename-column)
15. [Simplify Percent Change Labels](#15-simplify-percent-change-labels)
16. [Import Visualization Libraries](#16-import-visualization-libraries)
17. [Plot All Coin Trends — Categorical Point Chart](#17-plot-all-coin-trends--categorical-point-chart)
18. [Filter for a Single Coin](#18-filter-for-a-single-coin)
19. [Plot Bitcoin Price Over Time — Line Chart](#19-plot-bitcoin-price-over-time--line-chart)

---

## 1. Connect to CoinMarketCap API

> Set up a session with your API key and query parameters, then send a GET request to the CoinMarketCap listings endpoint. Network errors are caught and printed without crashing the script.

```python
from requests import Request, Session
from requests.exceptions import ConnectionError, Timeout, TooManyRedirects
import json

url = 'https://pro-api.coinmarketcap.com/v1/cryptocurrency/listings/latest'
parameters = {
  'start':'1',
  'limit':'5000',
  'convert':'USD'
}
headers = {
  'Accepts': 'application/json',
  'X-CMC_PRO_API_KEY': '50e05fb86518430cb517fa8aab3290a6',
}

session = Session()
session.headers.update(headers)

try:
  response = session.get(url, params=parameters)
  data = json.loads(response.text)
  print(data)
except (ConnectionError, Timeout, TooManyRedirects) as e:
  print(e)
```

> 💡 `json.loads(response.text)` converts the raw JSON string into a Python dictionary so you can navigate it like `data['data']`.

> ⚠️ Keep your `X-CMC_PRO_API_KEY` private — never commit it to a public repo. Store it in an environment variable instead.

---

## 2. Configure Pandas Display Settings

> Set display options so all columns are visible and rows are kept at a manageable number before working with the DataFrame.

```python
import pandas as pd
pd.set_option('display.max_columns', None)
pd.set_option('display.max_rows', 10)
```

| Option | Effect |
|---|---|
| `display.max_columns` = `None` | Show every column — no truncation |
| `display.max_rows` = `10` | Show 10 rows before collapsing |

---

## 3. Normalize JSON Response into DataFrame

> The API returns nested JSON. `pd.json_normalize()` flattens it into a clean tabular DataFrame. A timestamp column is added to record when each pull happened.

```python
df = pd.json_normalize(data['data'])
df['timestamp'] = pd.to_datetime('now')

df
```

> 💡 `pd.to_datetime('now')` stamps every row with the current date and time — essential for tracking price changes across multiple runs.

---

## 4. Wrap Everything into a Reusable Function

> Combine the API call, normalization, timestamping, and CSV export into one `api_runner(df)` function. Accepts the current DataFrame, appends new data, saves to CSV, and returns the updated DataFrame.

```python
df = pd.DataFrame()

def api_runner(df):
  from requests import Request, Session
  from requests.exceptions import ConnectionError, Timeout, TooManyRedirects
  import json

  url = 'https://pro-api.coinmarketcap.com/v1/cryptocurrency/listings/latest'
  parameters = {
    'start':'1',
    'limit':'15',
    'convert':'USD'
  }
  headers = {
    'Accepts': 'application/json',
    'X-CMC_PRO_API_KEY': '50e05fb86518430cb517fa8aab3290a6',
  }

  session = Session()
  session.headers.update(headers)

  try:
    response = session.get(url, params=parameters)
    data = json.loads(response.text)
    print(data)
  except (ConnectionError, Timeout, TooManyRedirects) as e:
    print(e)

  df2 = pd.json_normalize(data['data'])
  df2['timestamp'] = pd.to_datetime('now')

  df = pd.concat([df, df2], ignore_index=True)

  if not os.path.isfile(r'C:\Users\ACER\API.csv'):
    df.to_csv(r'C:\Users\ACER\API.csv', index=False)
  else:
    df.to_csv(r'C:\Users\ACER\API.csv', mode='a', header=False, index=False)

  #Then to read in the file:
  #df = pd.read_csv(r'C:\Users\ACER\API.csv')
```

> 💡 The `if/else` CSV block writes headers only on the very first run, then appends without headers on every run after — so column names never get duplicated mid-file.

> 💡 `pd.concat([df, df2], ignore_index=True)` is the correct modern replacement for the deprecated `.append()` method removed in Pandas 2.0.

---

## 5. Run the API on a Schedule

> Loop `api_runner()` 5 times with a 30-second sleep between each call. The return value is reassigned to `df` each run so the DataFrame grows correctly. No `exit()` — that would kill the Jupyter kernel.

```python
import os
from time import time
from time import sleep

for i in range(5):
    df = api_runner(df)
    print('API Runner Completed')
    sleep(30) #sleep for 1 minute
print('Done.')
```

> 💡 `df = api_runner(df)` — reassigning the return value is essential. Without it, the `df` outside the function never updates and all accumulated data is lost.

> 💡 Increase `range(5)` and `sleep(30)` to collect over a longer period. For example: `range(1440)` with `sleep(60)` runs every minute for 24 hours.

---

## 6. Read the Saved CSV

> Load the accumulated CSV into a DataFrame after all runs complete to inspect the full collected dataset.

```python
df = pd.read_csv(r'C:\Users\ACER\API.csv')
df
```

---

## 7. Fix Scientific Notation Display

> Large or very small numbers like crypto prices display in scientific notation by default. This forces Pandas to show them as regular decimals to 5 decimal places.

```python
#I want to see number in this case and not scientific notation
pd.set_option('display.float_format', lambda x: '%.5f' % x)
```

**Before:** `1.23457e+04`
**After:** `12345.67800`

---

## 8. Group by Coin & Average Percent Changes

> Group all rows by coin name and compute the mean percent change across six time windows: 1h, 24h, 7d, 30d, 60d, and 90d. This gives a single summary row per coin.

```python
#Look at coin trends overtime
df3 = df.groupby('name', sort=False)[['quote.USD.percent_change_1h', 
      'quote.USD.percent_change_24h', 'quote.USD.percent_change_7d', 
      'quote.USD.percent_change_30d', 'quote.USD.percent_change_60d', 
      'quote.USD.percent_change_90d']].mean()
```

> 💡 `sort=False` preserves the original coin ranking order (by market cap) rather than sorting alphabetically.

---

## 9. Stack Wide DataFrame to Long Format

> `stack()` pivots the six time-window columns into rows — converting wide-format data into long-format. This restructuring is required for Seaborn to treat each time window as a category on the x-axis.

```python
df4 = df3.stack()
df4
```

**Before (wide):** one column per time window per coin.
**After (long/stacked):** one row per coin-per-time-window combination.

---

## 10. Check the Data Type

> After stacking, the result becomes a Pandas `Series` instead of a `DataFrame`. Confirm the type before converting it.

```python
type(df4)
# <class 'pandas.core.series.Series'>
```

---

## 11. Convert Series to DataFrame

> Turn the stacked Series into a proper DataFrame with a named `values` column so it can be used with Seaborn.

```python
df5 = df4.to_frame(name='values')
df5
```

---

## 12. Count Rows

> Verify the total number of rows. With 15 coins × 6 time windows = 90 rows expected.

```python
df5.count()
```

---

## 13. Reset the Index

> After stacking, the DataFrame carries a multi-level index (coin name + column name). `reset_index()` flattens it back to a clean integer-based index and moves the old index levels into regular columns — making the DataFrame ready for renaming and plotting.

```python
#Because of how it's structured above we need to set an index. I don't want to pass a column as an index for this dataframe
#So I'm going to create a range and pass that as the dataframe. You can make this more dynamic, but I'm just going to hard code it

index = pd.Index(range(90))

#Set the above DataFrame index object as the index
#df6 = df5.set_index(index)
#df6

# If it only has the index and values try doing reset_index like "df5.reset_index()"
df6 = df5.reset_index()
df6
```

> 💡 `reset_index()` is used here instead of `set_index(index)` because it automatically promotes the multi-level index into columns (`name`, `level_1`) which are needed for the rename and replace steps that follow.

---

## 14. Rename Column

> The stacked level column comes out named `level_1` by default. Rename it to the more descriptive `percent_change` for clarity in charts and analysis.

```python
#Change the column name
df7 = df6.rename(columns={'level_1': 'percent_change'})
df7
```

---

## 15. Simplify Percent Change Labels

> Replace the long API column name strings in `percent_change` with short readable labels so the chart x-axis is clean and human-friendly.

```python
#Look at coin trends overtime
df7['percent_change'] = df7['percent_change'].replace(['quote.USD.percent_change_1h', 'quote.USD.percent_change_24h', 
    'quote.USD.percent_change_7d', 'quote.USD.percent_change_30d', 
    'quote.USD.percent_change_60d', 'quote.USD.percent_change_90d'],
    ['1h', '24h', '7d', '30d', '60d', '90d'])
df7
```

**Before:** `quote.USD.percent_change_7d`
**After:** `7d`

---

## 16. Import Visualization Libraries

> Import Seaborn and Matplotlib before creating any charts.

```python
import seaborn as sns
import matplotlib.pyplot as plt
```

---

## 17. Plot All Coin Trends — Categorical Point Chart

> Visualize how each coin's average percent change compares across all time windows. Each coin gets its own color via `hue`, and each time window sits on the x-axis.

```python
sns.catplot(x='percent_change', y='values', hue='name', data=df7, kind='point')
```

**Reads as:** for each time window (1h, 24h, 7d...) on the x-axis, show the average percent change for every coin — color-coded by coin name.

---

## 18. Filter for a Single Coin

> Narrow the full DataFrame down to just Bitcoin rows with only the columns needed for the price-over-time chart.

```python
# Now to do something much simpler
# we are going to create a dataframe with the columns we want
df10 = df[['name', 'quote.USD.price', 'timestamp']]
df10 = df10.query("name == 'Bitcoin'")
df10
```

> 💡 `.query("name == 'Bitcoin'")` is equivalent to `df10[df10['name'] == 'Bitcoin']` — both work, `.query()` is more readable for string conditions.

---

## 19. Plot Bitcoin Price Over Time — Line Chart

> Plot Bitcoin's USD price across every API pull timestamp to visualize how the price moved during the collection period.

```python
sns.set_theme(style="darkgrid")
sns.lineplot(x='timestamp', y='quote.USD.price', data=df10)
```

---

## 💡 Full Pipeline at a Glance

```
Connect to CoinMarketCap API
    ↓
Normalize JSON → flatten into DataFrame
    ↓
Add timestamp column to each row
    ↓
Wrap in api_runner(df) function
    ↓
Loop every 30 seconds × 5 runs → save each run to CSV
    ↓
Read CSV → inspect full dataset
    ↓
Group by coin → average % change across 6 time windows
    ↓
Stack (wide → long) → convert to DataFrame → reset index
    ↓
Rename column → simplify labels
    ↓
Plot: catplot (all coins) or lineplot (single coin)
```

---

## 📊 Chart Types Used

| Chart | Function | Use |
|---|---|---|
| Categorical point plot | `sns.catplot(kind='point')` | Compare all coins across time windows |
| Line chart | `sns.lineplot()` | Track one coin's price over time |

---

> 💬 *Never hardcode API keys in scripts you share or push to GitHub. Use `os.environ.get('CMC_API_KEY')` and store the key in a `.env` file listed in `.gitignore`.*
