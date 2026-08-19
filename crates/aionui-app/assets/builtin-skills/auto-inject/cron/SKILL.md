---
name: cron
description: Scheduled task management - create, query, update scheduled tasks to automatically execute operations at specified times.
---

# Scheduled Task Skill

Manage scheduled tasks for the current conversation with the bundled
agent-facing config CLI.

## Rules

1. Each conversation can have at most one scheduled task.
2. Always query existing tasks before creating or updating.
3. Do not ask for extra confirmation after the user has already requested the scheduling change.
4. Never pass, inline, export, echo, or set any `AIONUI_...` environment variable.
5. Call the bundled helper through the platform's `AIONUI_HELPER_BIN` environment-variable syntax. Never hardcode its path.
6. On Unix, pass create and update payloads through stdin heredocs. On Windows PowerShell 5.1, pass them with `--json`. Do not write payload JSON files to disk.
7. Put `job_id` in the update JSON payload, not in a command flag.
8. After a successful create or update, send one short final confirmation that a normal user can understand. Include the task name and schedule description. Do not show internal ids such as `cron_...`.
9. If the CLI fails, report the failure from stderr/stdout in normal prose and do not claim the task was created.
10. Do not inspect the skill catalog, MCP catalog, assistant catalog, provider configuration, or credentials before scheduling. Creating a task records the user's requested future instruction; it does not require proving that every future dependency is currently available.
11. Do not speculate that the task is bound to the wrong assistant. The `cron current` helper binds the task to the active conversation and copies its runtime configuration.
12. On success, stop immediately after the short confirmation. Do not add a background summary, internal configuration details, warnings, or suggested follow-up work unless the user explicitly asks for them.

## Workflow

1. Run `"$AIONUI_HELPER_BIN" config cron current list`.
2. If the returned `data` array is empty, create the task with the platform-specific command below.
3. If one task exists and the user wants to change it, update that task with the platform-specific command below.
4. If a task already exists and the user is asking for a different additional task, ask how they want to handle the existing task.
5. Report success or failure from the CLI output in normal prose, following the final confirmation rule above.

Success means the helper exits with code 0 and prints JSON with `"success": true`. The `cron.job-created` event is also emitted so the app can refresh its scheduled-task indicator. Do not infer success from an attempted command or a temporary script file.

## Payload

Create payload:

- `name`: Short descriptive name.
- `schedule`: Standard 5-field cron expression.
- `schedule_description`: Human-readable schedule.
- `message`: Complete, self-contained instruction sent to the AI when the task runs.

Update payload uses the same fields and also requires:

- `job_id`: The id from the existing task returned by the list command.

The `message` must tell the AI exactly what to do when the task fires. It should
not merely restate the user's scheduling request.

| User says                         | Bad message              | Good message                                                                                 |
| --------------------------------- | ------------------------ | -------------------------------------------------------------------------------------------- |
| "Send me hello every day at 10am" | Send me hello            | Reply with exactly: Hello!                                                                   |
| "Remind me to drink water daily"  | Remind me to drink water | Reply with a friendly reminder to drink water.                                               |
| "Summarize AI news every Monday"  | Summarize AI news        | Search for the latest AI news from this week and produce a concise bullet-point summary.     |

## Examples

Examples use English sample text. For real tasks, write the task name,
schedule description, and message in the user's language.

Query:

```bash
"$AIONUI_HELPER_BIN" config cron current list
```

Create:

```bash
"$AIONUI_HELPER_BIN" config cron current create <<'JSON'
{
  "name": "Weekly Meeting Reminder",
  "schedule": "0 9 * * MON",
  "schedule_description": "Every Monday at 9:00 AM",
  "message": "Reply with a short weekly meeting reminder that includes the current date and time."
}
JSON
```

Windows PowerShell 5.1 (use inline JSON; do not use here-strings or temporary files):

```powershell
& $env:AIONUI_HELPER_BIN config cron current create --json '{"name":"每日17:30日报","schedule":"30 17 * * *","schedule_description":"每天17:30","message":"执行玲玲日报技能并生成当天日报。"}'
```

Update:

```bash
"$AIONUI_HELPER_BIN" config cron current update <<'JSON'
{
  "job_id": "cron_123",
  "name": "Daily Summary",
  "schedule": "0 18 * * MON-FRI",
  "schedule_description": "Weekdays at 6:00 PM",
  "message": "Review today's conversation context and produce a concise end-of-day summary."
}
JSON
```

Windows PowerShell 5.1:

```powershell
& $env:AIONUI_HELPER_BIN config cron current update --json '{"job_id":"cron_123","name":"每日17:30日报","schedule":"30 17 * * *","schedule_description":"每天17:30","message":"执行玲玲日报技能并生成当天日报。"}'
```

Multiline message example:

```json
{
  "name": "Daily Summary",
  "schedule": "0 9 * * *",
  "schedule_description": "Every day at 9:00 AM",
  "message": "First paragraph.\nSecond paragraph.\nThird paragraph."
}
```

## Cron Expression

Format: `minute hour day-of-month month day-of-week`.

Example: `0 9 * * MON-FRI` means weekdays at 9:00 AM.

Use only standard cron fields and ranges supported by the backend parser. Do
not use Quartz-style extensions such as `L`, `L-N`, `W`, `LW`, `#`, or `?`.
