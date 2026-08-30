---
name: tellius-run-and-schedule-a-mission
description: Execute a saved Tellius Kaiya Mission headlessly, resume it correctly when it pauses mid-run, and schedule recurring delivery to email, Slack or Teams — including the destructive operations to guard.
api: Tellius MCP Server
generated: '2026-08-30'
method: generated
source: https://help.tellius.com/kaiya/tellius-mcp-server
operations:
  - tellius_list_workflows
  - tellius_search_workflows
  - tellius_get_workflow
  - tellius_run_workflow
  - tellius_resume_workflow
  - tellius_create_schedule
  - tellius_list_schedules
  - tellius_get_schedule
  - tellius_update_schedule
  - tellius_list_slack_channels
  - tellius_list_teams_channels
---

# Run and schedule a Kaiya Mission

A Kaiya Mission is a saved multi-step workflow that pursues an objective — monitoring
data, investigating changes, and delivering briefings. Both live entirely on the MCP
surface: there is no REST API for workflows or schedules at all.

## Find and inspect

1. `tellius_list_workflows` for a paginated list, or `tellius_search_workflows` to search
   by name or description.
2. `tellius_get_workflow` for the full step detail before you run anything.

## Run it

Call `tellius_run_workflow` with the workflow id and any inputs it needs.

**A Mission can pause for input at two different points, and the tool you call to
continue depends on which one paused it.** This is the single most common integration
error on this surface:

- **A question asked before the run starts** comes from the Mission's own defined inputs.
  Answer it by calling `tellius_run_workflow` again with the input filled in.
- **A question asked while the Mission is already running** comes from a step that
  requests input mid-execution. This pause carries a **`conversation_id`**. When you see
  one, call `tellius_resume_workflow` with that same `conversation_id`.

Calling `tellius_run_workflow` again to answer a mid-run question starts a brand-new run
instead of continuing the paused one, so the same question comes back and the Mission
never completes. **The presence of a `conversation_id` in the clarification is the signal
to switch tools.**

## Schedule recurring delivery

1. If delivering to Slack or Teams, look up channel ids first with
   `tellius_list_slack_channels` or `tellius_list_teams_channels`. Do not guess a channel
   id.
2. Call `tellius_create_schedule` with the name, the cadence, what to run (a saved
   workflow or a stored ad-hoc question), and where to send it. Creating the schedule
   records it; Tellius runs it on its cadence from then on.
3. `tellius_list_schedules` returns schedules scoped to the calling user.
   `tellius_get_schedule` returns one schedule's full delivery configuration.

## Reversibility — read this before any write

Tellius documents **no reversal operation and no reversal window** for these surfaces.

- `tellius_delete_workflow` and `tellius_delete_schedule` are marked **Destructive** in
  Tellius' own reference, described as **"Permanent delete."** There is no undo, no
  trash, no restore window, and no REST path to reach the object either.
- Require an explicit human confirmation before calling either one. An agent should never
  call them as a step inside a larger plan.
- `tellius_update_schedule` **can** pause and resume a schedule, and from 6.3.2 a
  schedule can be disabled and held from executing until it is enabled again. Prefer
  disabling to deleting whenever the intent is "stop this for now".
- There is no idempotency key. If `tellius_create_schedule` or `tellius_create_workflow`
  times out, list first and check whether it landed before retrying — a blind retry
  creates a duplicate.

## Authorization note

Everything runs as a specific Tellius user, with Business View permissions and row-level
security applied. But the OAuth token carries a single `mcp` scope covering all 25 tools,
including the destructive ones — the token itself draws no least-privilege boundary, so
the guard has to be yours.
