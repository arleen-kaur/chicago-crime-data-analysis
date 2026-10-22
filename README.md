# Chicago Crime Data Analysis

A big-data pipeline and web app that turns the City of Chicago’s crime reports into answers you can actually look at. Raw incident records are cleaned and aggregated with Apache Spark, stored in MySQL, and explored in a React app that charts patterns, maps hotspots, and checks hypotheses with an LLM.

This is a senior design project for **CS 179G (Big Data)** at UC Riverside. The cleaned dataset used in the scaling experiments is about **7.1 million** incident records, covering reports from **2001 through 2025**.

## What you can do with it

Each screen in the app is a question, not a raw table dump. You pick a question, see the aggregated result as a chart or map, and can ask the app whether a stated hypothesis holds.

Examples:

- Does reported crime peak in the evening and overnight, or during the day?
- Do summer months carry more volume than winter?
- Do theft, burglary, and robbery shift around Christmas, Halloween, or Thanksgiving?
- Which community areas are persistent hotspots?
- Is theft and robbery more common downtown than in residential areas?
- Do robberies cluster more around transit or around commercial locations?
- How does theft around airports compare with other places?
- Which crime types were most common during the Great Recession (2007–2009)?
- How has citywide volume changed year over year?

For any of those views, **Generate insight** sends the aggregated rows, the question, and the hypothesis to Gemini. The model returns a short plain-English summary and a status of `supported`, `mixed`, or `not_supported`. It is instructed to use only the rows it is given.

## How it is built

```text
City of Chicago crime CSV
        |
        v
PySpark cleaning  -->  Parquet
        |
        v
PySpark aggregations (time, place, holidays, trends)
        |
        v
MySQL aggregate tables  (Spark JDBC)
        |
        v
Flask REST API
        |
        v
React app  (charts, map, LLM insights)
```

The heavy work stays in Spark. The database and the API only hold summaries, so the website stays responsive even though the source data is millions of rows.

| Layer | Role | Tools |
| --- | --- | --- |
| Processing | Clean, normalize locations, aggregate | Apache Spark (PySpark), Parquet |
| Storage | Hold one table per analysis | MySQL, Spark JDBC |
| API | Serve tables, summary stats, and insights | Flask, PyMySQL, LangChain, Gemini |
| UI | Browse questions, chart results, map community areas | React, Vite, Recharts, Leaflet |

The API only allows a fixed set of table names, so the frontend cannot query arbitrary SQL.

## Analyses

Questions are grouped into four areas. Each one is a MySQL table produced by `analysis/part2_analysis.py` and described for the UI in `frontend/src/constants/tables.js`.

**Time of day.** Hourly counts, plus daytime (8:00–15:59), evening (16:00–23:59), and overnight (00:00–7:59).

**Season and holidays.** Monthly totals, a summer / late-winter / other split, holiday vs. non-holiday, and crime-type comparisons for Christmas, Halloween, and Thanksgiving.

**Place.** Location categories (residential, street, transit, retail, airport, and others), the top crime type in each category, theft share by location, downtown vs. residential theft and robbery, transit vs. commercial robbery, airport theft, crimes at sports locations, and totals by Chicago community area. Community areas are plotted on a Leaflet map.

**Long-term trends.** Yearly totals, and crime type during 2007–2009.

Location strings from the source data are messy (`RESIDENCE PORCH/HALLWAY` vs. `RESIDENCE-PORCH/HALLWAY`, several airport and school variants, and so on). The Spark job normalizes those labels before aggregating so similar places are counted together.

## Scaling

`analysis/experiment_runner.py` reruns the same group-by workload while changing two things: Spark worker count (1, 2, 4, 8) and how much of the cleaned data is used (10% through 100%). At full scale that is 7,078,879 records. Results are written to `analysis/experiment_results.json`, and the charts are in `docs/images/`.

## Repository layout

```text
data-cleaning/     PySpark jobs that clean the raw CSV and write Parquet
analysis/          Aggregations, MySQL loads, and the scaling experiment
backend/           Flask API and the MySQL schema
frontend/          React + Vite app
docs/images/       Worker-count and data-size timing charts
```

The raw CSV and the cleaned Parquet dataset are not in this repo. They are large and were processed on the class cluster.

## Run the web app

The Spark jobs expect the class cluster (`class-142.cs.ucr.edu`) and paths under `/home/cs179g`. The website can run on a laptop once MySQL has the aggregate tables.

**1. Database**

Create the tables from `backend/schema.sql` in a database named `cs179g`. The API connects as `root` on `127.0.0.1` with an empty password.

**2. API**

```bash
cd backend
pip install flask pymysql langchain-google-genai langchain-core
export GEMINI_API_KEY=your_key   # optional; insights return 503 without it
python3 app.py                    # http://127.0.0.1:5001
```

**3. Frontend**

```bash
cd frontend
npm install
npm run dev                       # http://127.0.0.1:5173
```

Vite proxies `/api` to port 5001. `npm run build` writes a production bundle to `frontend/dist`, which Flask can serve directly.

**Pipeline on the cluster**

```bash
spark-submit data-cleaning/test.py
spark-submit --jars ~/libs/mysql-connector-j-9.4.0.jar analysis/part2_analysis.py
```

`data-cleaning/test.py` reads the source CSV, standardizes column names, parses timestamps, casts arrest and domestic flags, drops rows missing date, crime type, or coordinates, removes duplicate case IDs, and writes Parquet. `analysis/part2_analysis.py` reads that Parquet and loads one MySQL table per question.

## Team

Built with Neha Gutapalli, Tanya Carillo, Eric Via, and Jade Someda.
