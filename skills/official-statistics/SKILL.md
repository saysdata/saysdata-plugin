---
name: official-statistics
description: Delivers trustworthy answers from official statistics. Finds, queries, correctly interprets and accurately reports data published by national statistics agencies and international organizations (e.g. Statistics Canada, Eurostat, the OECD) for countries and their provinces, states, regions and cities around the world, through the SaysData tools. Coverage includes economic, financial, social, health, cultural, demographic and environmental data, from GDP and wages to greenhouse-gas emissions and cultural participation. Use it whenever an answer depends on a figure a statistics agency or international organization might publish. That includes questions that never name a source ("how fast is Alberta growing?", "are rents rising faster than wages?"), checking a statistic someone quoted, comparing places or years, and drafting a report, brief or chart that needs current numbers. Every figure comes from a query and is reported with its source and a link to check it.
---

# Official statistics with SaysData

SaysData lets you find, query, interpret and cite thousands of datasets from national and
international agencies. The flow:

1. `search_datasets` for candidates
2. `describe_dataset` for the one that fits. If `describe_dataset` returns truncated lists, use
   `distinct_values` to see all members
3. `execute_sql` as needed to compute every figure you need
4. Read each result's notes and warnings
5. Report the figures with their query links and citations
6. Check your work critically before you declare done

Each section below covers one step: the rule first, then tips.

## 1. Find candidates

Call **search_datasets** with the measure, place and breakdown in plain words. Weigh the
candidates' names, periods, geography and breakdowns against the question. If none exactly fits, do some additional searches to find similar or reasonable proxy measures. If the user asked a question about a specific statistical measure that you could not find, be explicit about not finding it and suggest any viable alternatives you did find.

## 2. Understand the dataset

Call **describe_dataset** before any query. It returns a dataset "brief". Read its dimensions and which member of each is the
total, units and scale, periods, the provider's notes, and how to cite it. Most wrong answers come
from skipping this. Never guess an id: when the brief's list of members is truncated, look the
member up with **distinct_values**.

### If it becomes clear that the dataset won't answer the question
Either describe another candidate that might be a fit, or return to searching.

### If the dataset is still being prepared

Large datasets can take some time to prepare. If it's going to take a while, the response will tell you when to check back. Tell the user. While you wait, you may try another candidate or do additional searches.

## 3. Compute every figure in the query

Every number you report must appear, as reported, in an execute_sql result. That includes single
figures (a total, change, rate, share, average, rank or maximum): return each as its own column.
Never add, divide, average or re-round numbers yourself, however simple.

### A basic query

A call has two parts:

- **Slices** choose the data. Copy the brief's slice template and keep every dimension it lists.
  Choose members by id: `members` for a list, `level` for every place at one geography level
  (the brief names the levels), `childrenOf` for everything directly under one member, and
  `except` to leave some out. Choose periods with `members` or `from`/`to`.
- **SQL** is one SELECT that reads the slices by name. Write column names as the brief gives
  them, in double quotes. Under the hood SaysData uses DuckDB SQL dialect.

Population of each province and territory in 2026, largest first:

```json
{
  "slices": [{
    "name": "pop",
    "dataset": "01423cfe-ec8f-4755-91fb-e87f95b60cdf",
    "dimensions": {
      "GEO_ID": {"level": "2"},
      "Gender_ID": {"members": ["1"]},
      "Age_group_ID": {"members": ["1"]},
      "REF_DATE": {"members": ["2026"]}
    }
  }],
  "sql": "SELECT \"GEO\" AS place, \"VALUE\" AS persons FROM pop ORDER BY persons DESC"
}
```

### Single figures

Canada's population change, 2016 to 2026:

```json
{
  "slices": [{
    "name": "pop",
    "dataset": "01423cfe-ec8f-4755-91fb-e87f95b60cdf",
    "dimensions": {
      "GEO_ID": {"members": ["1"]},
      "Gender_ID": {"members": ["1"]},
      "Age_group_ID": {"members": ["1"]},
      "REF_DATE": {"members": ["2016", "2026"]}
    }
  }],
  "sql": "WITH y AS (SELECT max(\"VALUE\") FILTER (WHERE \"REF_DATE\" = '2016') AS persons_2016, max(\"VALUE\") FILTER (WHERE \"REF_DATE\" = '2026') AS persons_2026 FROM pop) SELECT persons_2016, persons_2026, persons_2026 - persons_2016 AS change_persons, round(100 * (persons_2026 - persons_2016) / persons_2016, 1) AS change_pct FROM y"
}
```

### Percentages

- Multiply by 100 in the query and round once, at the end. Name the column for its unit
  (`share_pct`).
- For a share of a total, divide by the published total member, in its own slice, not by a sum
  of the parts: parts can be suppressed, missing or overlapping.
- A change in a value that is already a rate (an unemployment rate) is in percentage points, not
  percent. Say which.

Each province's and territory's share of Canada's population in 2026:

```json
{
  "slices": [
    {"name": "provinces", "dataset": "01423cfe-ec8f-4755-91fb-e87f95b60cdf",
     "dimensions": {"GEO_ID": {"level": "2"}, "Gender_ID": {"members": ["1"]},
                    "Age_group_ID": {"members": ["1"]}, "REF_DATE": {"members": ["2026"]}}},
    {"name": "canada", "dataset": "01423cfe-ec8f-4755-91fb-e87f95b60cdf",
     "dimensions": {"GEO_ID": {"members": ["1"]}, "Gender_ID": {"members": ["1"]},
                    "Age_group_ID": {"members": ["1"]}, "REF_DATE": {"members": ["2026"]}}}
  ],
  "sql": "SELECT p.\"GEO\" AS place, p.\"VALUE\" AS persons, round(100 * p.\"VALUE\" / c.\"VALUE\", 1) AS share_pct FROM provinces p CROSS JOIN canada c ORDER BY share_pct DESC"
}
```

### Combining datasets

For a per-person figure, a ratio or a gap between two datasets, put each in its own slice and join
them in SQL. execute_sql warns when two datasets are joined on a column of the same name and
share no value of it, but nothing checks a join where only some values match, so:

- Pin every dimension except the join key to one member in each slice, so the join cannot
  multiply rows.
- Never join on numbered member ids, such as Statistics Canada's: each table numbers its own
  members (GEO_ID 14 is Nunavut in one table and the Northwest Territories in another).
- Where both files carry labels, join on the labels. Check with distinct_values that both
  datasets spell them the same, and tell the user about any normalization you needed.
- OECD and Eurostat files hold codes with no label column, and they code some places differently
  (Greece is GRC at the OECD and EL at Eurostat). To join such a file to a dataset that codes or
  labels its members differently, map the codes in SQL and tell the user about the mapping.
  The same code can also name different members (AU is the African Union in some OECD lists and
  Australia in others), so check with distinct_values that each code you join has the same label
  in both datasets.
- Use a FULL JOIN, so a key found in only one file, or a code you did not map, comes back as a
  row with a NULL instead of silently disappearing.
- Name every output column.
- Check both briefs' units and scale, convert in SQL (millions × 1,000,000) before dividing, and
  say what the result is per. Tell the user about any conversions you applied in the query.

GDP per person by province and territory in 2024 (GDP is in millions of dollars):

```json
{
  "slices": [
    {"name": "gdp", "dataset": "af24ac1e-0f67-47b1-b6c1-fd7f3b3bada5",
     "dimensions": {"GEO_ID": {"level": "2"}, "Prices_ID": {"members": ["2"]},
                    "Estimates_ID": {"members": ["38"]}, "REF_DATE": {"members": ["2024"]}}},
    {"name": "pop", "dataset": "01423cfe-ec8f-4755-91fb-e87f95b60cdf",
     "dimensions": {"GEO_ID": {"level": "2"}, "Gender_ID": {"members": ["1"]},
                    "Age_group_ID": {"members": ["1"]}, "REF_DATE": {"members": ["2024"]}}}
  ],
  "sql": "SELECT \"GEO\" AS place, g.\"VALUE\" * 1000000 AS gdp_dollars, p.\"VALUE\" AS persons, round(g.\"VALUE\" * 1000000 / p.\"VALUE\") AS gdp_dollars_per_person FROM gdp g FULL JOIN pop p USING (\"GEO\") ORDER BY gdp_dollars_per_person DESC"
}
```

## 4. Read the results
### Refused queries
SaysData employs a number of automated checks on every execute_sql query to help agents avoid common mistakes. If your query is refused, review the query, fix any issues and try again.

### Review your query
Before anything else, double check your query to ensure you actually asked for what you were intending to request. If there was an error in your query, discard the result, update your query and try again.

### Notes and warnings
Read each result's notes and warnings before its rows. Notes describe what the query read: the dataset, its units and flags, and the members and periods each slice holds. Warnings flag a risk in how the query used them. Consider whether these are relevant to your analysis, interpretation or confidence in the results. If you conclude the results are suitable for use, make sure your answer to the user is clear about limitations to the data you present.

### Reading values

- NULL means not released, never zero. A sum or average over rows holding one is partial; say so.
- Report values in the brief's scale ("millions") or multiply them out in SQL, never both.
- Pass on any flag (preliminary, estimated, revised, terminated) with the number it marks.

## 5. Report
### Citing the results
Always cite the work using the citation in the execute_sql response, verbatim, with its link and licence. You may optionally include "via SaysData.com".

### Showing your work
After using SaysData results in an answer or in producing an artifact, ALWAYS show the user a table of query descriptions and the historyUrl for each execute_sql call you used. This lets the user see how you got each number and is MANDATORY. Describe each query in one sentence: what it was for, including place, period, and any breakdown such as age or sex. The historyUrl opens for the signed-in user who ran it, for 60 days. Example:

```md
| Description | Verification Link |
|---|---|
| Canada's population, July 1 2016 and 2026, with the change in persons and percent (2026 preliminary) | [8cc919f7](https://saysdata.com/history/8cc919f7-7a0e-4521-8db1-5787e59e37a7) |
```

### Cross-dataset comparisons and compatibility
When comparing numbers across datasets always consider whether the definitions are actually compatible (e.g. via `describe_dataset`) even if they are labeled the same. Differing definitions of data with the same label or even different measures that have sufficient similarity can be compared carefully depending on the specific question they are trying to answer. For example, "housing starts" and "residential building permits" are reasonable proxies for some questions, but not for others. This is very context dependent, requires judgement on your part and transparency with the user.

### Keep what the data shows apart from everything else

The rows give values. They do not say why values differ, what caused a change, what comes next,
or anything about periods, places or measures you did not query.
Always explicitly differentiate:

- Data from SaysData
- Editorial content including **Background** (general knowledge, not from SaysData), **Inference** (your reasoning from the data) or **Speculation** (what the data cannot test).

Never mix an editorial statement into a sentence reporting a queried number. When they are in the same paragraph make sure the transition between fact and editorial is explicit. Never give a figure from memory.

## 6. Check your work

Before you say it is done:

- Every number in text and charts matches a result row in value, unit, scale, place and period.
- Every query a number came from is listed with its link, and every dataset is cited.
- Every claim beyond the rows is queried or labelled.
- Every warning has been evaluated for relevance.
- Every comparison is like for like, or the difference is labelled.

Fix the query or the text before you tell the user the work is complete.

## Feedback

If the user wants to give feedback, wants data you could not find or says something about the
data was wrong, send_feedback sends it to the SaysData team: show the user the exact text before
sending, send only with their OK, and leave out personal or confidential details. They can also
email support@saysdata.com.
