# Privacy-Minimized Product Analytics

![Python](https://img.shields.io/badge/Python-3.10+-blue)
![SQL](https://img.shields.io/badge/SQL-DuckDB-yellow)
![Platform](https://img.shields.io/badge/Run%20on-Google%20Colab-orange)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)
![Cost](https://img.shields.io/badge/Cost-Free-success)

A small analytics pipeline that answers one question, "do users finish the export workflow?", while collecting as little as possible: a field allowlist, server-side consent, deduplication, small-group hiding and raw-data deletion.

## Table of Contents
1. [Problem](#problem)
2. [What I Built](#what-i-built)
3. [Results](#results)
4. [Retention and Deletion Drill](#retention-and-deletion-drill)
5. [Acceptance Tests](#acceptance-tests)
6. [What This Design Does NOT Guarantee](#what-this-design-does-not-guarantee)
7. [Tech Stack](#tech-stack)
8. [How to Run](#how-to-run)
9. [Limitations](#limitations)
10. [About Me](#about-me)

## Problem
A small product team wants to know one thing: what share of sessions that start an export also finish it. They do not want to collect every click, identity and raw input to answer that. So the pipeline accepts only a fixed list of fields, checks consent on the server, removes duplicates and deletes raw data on a schedule.

## What I Built
- **100 fake sessions** with rotating test tokens (no real IPs, names, emails or free text), made with a known answer
- **An event contract decided before any data came in**:
  - Events: `export_start`, `export_success`, `export_fail`. "Abandoned" is not an event, it is a start with no end after 30 minutes
  - Allowed fields: `event_id`, `session_token`, `event_name`, `event_time`, `app_version`, `step_count`
  - Forbidden fields: `email`, `ip`, `free_text`, `user_name`, `phone`. If one is present, the **whole event is rejected**
  - Unknown fields are stripped, payloads over 300 bytes are rejected
  - Consent comes from a server table, never from the event itself
- **Ingestion** that rejects, strips and dedupes, and logs only the event id and the reason for every rejection (not the content)
- **A funnel in DuckDB** with the metric definition printed next to the result, late and lost events counted, and small groups hidden
- **Retention and deletion drills**
- **Acceptance tests** checked against the known funnel from the data generator

## Results
**Delivery:** 184 payloads arrived. 169 accepted, 7 duplicate deliveries ignored, 8 rejected, 1 unknown field stripped.

| Rejected | Count |
|---|---|
| No consent | 4 |
| Forbidden field: email | 1 |
| Forbidden field: ip | 1 |
| Forbidden field: free_text | 1 |
| Payload too large | 1 |

**Metric definition:** completion rate = export_success sessions / sessions started at least 30 minutes before the cut-off (consented only, deduplicated by `event_id`, in-progress sessions excluded).

The same query at three moments, on 98 consented sessions:

| | Started | Success | Fail | Abandoned | In progress | Completion rate |
|---|---|---|---|---|---|---|
| Truth (from the generator) | 98 | 60 | 14 | 22 | 2 | 0.625 |
| As of cut-off | 98 | 51 | 14 | 31 | 2 | 0.531 |
| After late events arrived | 98 | 56 | 14 | 26 | 2 | 0.583 |

What the table says:
- **Late events change the answer.** 5 success events arrived 1 to 3 days late. At the cut-off they looked like abandoned sessions (success 51), and after they arrived the count rose to 56
- **Lost events can not be fixed.** 4 success events were never delivered. They still look abandoned, so the final 56 is below the truth of 60 and the completion rate is a **lower bound**
- A dashboard should say when it was computed

**Small groups:** the dashboard breakdown by app version hides any group under 5 sessions.

| App version | Sessions | Success |
|---|---|---|
| 1.0 | 37 | 26 |
| 1.1 | 47 | 22 |
| 2.0 | 11 | 6 |
| 2.0-beta | `<5` | hidden (the raw group had 3 sessions) |

## Retention and Deletion Drill
Today is 2026-09-20 and raw events are kept for 14 days.

| Step | Result |
|---|---|
| Raw events before retention delete | 169 |
| After the delete | 55 (oldest remaining: 2026-09-06 08:03) |
| Daily aggregate saved before deleting | 21 rows, counting 169 events |
| Deletion request for one session token | 2 raw rows removed, 0 left |
| Daily aggregate after the deletion request | unchanged (old totals still there) |

That last row is on purpose. Deleting raw rows does not change an aggregate that was already saved.

## Acceptance Tests
| Test | Result |
|---|---|
| Forbidden fields and the oversize payload are rejected with a reason, the unknown field is stripped | PASS |
| Repeated delivery does not inflate the denominator (7 duplicates ignored, 98 started sessions, truth 98) | PASS |
| Events from sessions without consent never reach the events table (0) | PASS |
| Final success count matches the generator once lost events are counted (56 = 60 - 4) | PASS |
| Old raw rows are removed by retention, and a deleted token has 0 rows left | PASS |

## What This Design Does NOT Guarantee
- It is **not anonymous** and it is **not differential privacy**. Removing names does not stop re-identification, for example by joining time, app version and a rare behaviour
- Rotating tokens help, but one token still links all the events of a session
- Saved aggregates keep their numbers after raw data is deleted
- Consent and the allowlist are only as good as the server that enforces them. A formal privacy claim needs expert review

## Tech Stack
| Area | Tools |
|---|---|
| Language | Python |
| SQL engine | DuckDB |
| Data | pandas |
| Charts | matplotlib |
| Environment | Google Colab |

## How to Run
1. Open `Privacy_Minimized_Product_Analytics.ipynb` in Google Colab
2. Click **Runtime > Run all**
3. No dataset or API key is needed. All data is generated inside the notebook (seed 42)

Or locally: `pip install -r requirements.txt` and open the notebook in Jupyter.

## Limitations
- All sessions are fake, with a known answer. The numbers show that the logic works, not what real users do
- Only one question and 3 event names are covered
- The completion rate is a lower bound as long as events can be lost
- Retention here is a demo inside one notebook, not a scheduled job

## About Me
**Akshat Kesharwani** - Data Analyst / Data Scientist (fresher)

- Portfolio: https://akshatkesharwani-info.github.io/
- GitHub: https://github.com/akshatkesharwani-info

Idea source: "Top 50 Projects to Build in 2026" (Yasir Arafat Sharfi). The build, data and code here are my own.
