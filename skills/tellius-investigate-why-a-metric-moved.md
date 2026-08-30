---
name: tellius-investigate-why-a-metric-moved
description: Run a multi-step agentic root cause investigation in Tellius with tellius_deep_insight, handling its long runtime and concurrency constraints correctly.
api: Tellius MCP Server
generated: '2026-08-30'
method: generated
source: https://help.tellius.com/kaiya/tellius-mcp-server
operations:
  - tellius_list_business_views
  - tellius_get_business_view_details
  - tellius_deep_insight
  - tellius_send_feedback
---

# Investigate why a metric moved

`tellius_ask` answers a question with a single query. `tellius_deep_insight` runs a
multi-step agentic investigation: it plans, runs several steps, and validates its own
findings. Use it for questions about causes and drivers, where one aggregation would not
answer the question — "why did revenue decline in the Northeast", not "what was revenue
in the Northeast".

## Steps

1. **Resolve the Business View** with `tellius_list_business_views`, and confirm it holds
   the metric and the dimensions the investigation will need with
   `tellius_get_business_view_details`. A driver analysis can only rank dimensions the
   Business View actually models.
2. **Call `tellius_deep_insight`** with the same request shape as `tellius_ask`:
   `question`, `business_view_ids`, `include_raw: true`. The response has the same shape
   too — summary, data table, SQL.
3. **Optionally record quality** with `tellius_send_feedback` on the response.

## The two constraints that break integrations

- **Set a generous client timeout.** A deep insight can run for a few minutes on a large
  Business View. A timeout tuned for a simple query will cut it off before it finishes,
  and you will read the cutoff as a failure rather than as your own limit.
- **Do not fire many deep insights at once.** Long-running calls hold resources for their
  whole duration. Queue them rather than launching a batch in parallel, or fast queries
  issued at the same time will be left waiting behind them.

Tellius states no numeric concurrency limit and returns no rate-limit headers, so the
queue discipline is yours to enforce. Note also that concurrent-job capacity is a
**tier** property: the Premium tier caps AutoML and data pipeline jobs at 5 concurrent
each.

## Reading the result

Treat the investigation output the same way as any `tellius_ask` answer: check you got
data rather than a clarifying question, check the returned SQL matches the intent, and
sanity-check the magnitudes before presenting ranked drivers to anyone.

## What this does not do

Deep insight is read-oriented analysis. It does not create a saved artifact you can
return to later — for that, build a Mission (see
`tellius-run-and-schedule-a-mission`).
