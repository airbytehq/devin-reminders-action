# Devin Reminders Action

A reusable GitHub Action for scheduling, listing, cancelling, and firing reminders for [Devin.ai](https://devin.ai) sessions. Uses GitHub Actions artifacts for storage and the Devin API to deliver reminders.

## Features

- Schedule reminders for existing Devin sessions (`put`)
- List all pending reminders and filter due items (`list`)
- Cancel pending reminders by GUID (`cancel`)
- Fire due reminders and clean up automatically (`cron`)
- Optional Slack notifications (opt-in via `slack-channel` input)
- Timezone-aware display for notification messages

## Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `action` | Action to perform: `put`, `list`, `cancel`, or `cron` | Yes | |
| `remind-at` | ISO 8601 timestamp with timezone offset for when the reminder fires. Must be in the future and no more than 3 days ahead. Required for `put`. | No | |
| `reminder-message` | Message to deliver when the reminder fires. Required for `put`. | No | |
| `agent-session-url` | Devin session URL to ping when the reminder fires. Required for `put` and `cancel`. | No | |
| `slack-users-cc` | Comma or newline-delimited list of Slack user tags to CC on notifications (e.g. `<@U12345>, <@U67890>`). | No | |
| `devin-token`| Devin API token. | Yes | |
| `slack-channel` | Slack channel name for notifications. Leave empty to skip Slack. | No | |
| `slack-token` | Slack bot token. Only needed if `slack-channel` is set. | No | |
| `reminder-timezone` | Timezone for displaying times in notifications. Accepts IANA names (e.g. `America/Los_Angeles`) or UTC offsets. Does not affect parsing of `remind-at`. | No | `UTC` |
| `cancel-guids` | JSON array of reminder GUIDs to cancel. Required for `cancel`. | No | |
| `lock-mode` | Controls artifact-based locking to prevent race conditions. `auto` locks on `put`, `cancel`, and `cron`, `none` disables locking, `always` locks on all actions including `list`. | No | `auto` |
| `undeliverable-cc` | Slack user/group tags to CC when a reminder is dropped because its session is unreachable (HTTP 403/404/410), e.g. `<!subteam^S12345>`. Only used when `slack-channel` is set. | No | |

## Outputs

| Name | Description |
|------|-------------|
| `reminders-json` | JSON array of all current reminders |
| `due-json` | JSON array of reminders that are currently due |
| `due-count` | Number of reminders currently due |
| `due-guids` | Newline-delimited list of GUIDs for due reminders |
| `total-count` | Total number of reminders in the list |
| `item-guid` | GUID of the newly added reminder (only for `put`) |
| `popped-count` | Number of reminders removed after cron firing (fired + undeliverable) |
| `failed-count` | Number of reminders that failed to fire and were kept for retry (only for `cron`) |
| `undeliverable-count` | Number of reminders dropped because the target session returned HTTP 403/404/410 (only for `cron`) |
| `cancelled-count` | Number of reminders cancelled (only for `cancel`) |

## Usage

### Schedule a Reminder

```yaml
- uses: airbytehq/devin-reminders-action@v1
  with:
    action: put
    remind-at: "2026-02-20T17:00:00-08:00"
    reminder-message: "Check on the deployment status"
    agent-session-url: "https://app.devin.ai/sessions/abc123"
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
    slack-token: ${{ secrets.SLACK_BOT_TOKEN }}
    slack-channel: devin-reminders
    reminder-timezone: America/Los_Angeles
```

### List Reminders

```yaml
- uses: airbytehq/devin-reminders-action@v1
  id: reminders
  with:
    action: list
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}

- run: echo "Due: ${{ steps.reminders.outputs.due-count }} / Total: ${{ steps.reminders.outputs.total-count }}"
```

### Cancel Reminders

Cancel by GUIDs for a session (requires `agent-session-url` + `cancel-guids`):

```yaml
- uses: airbytehq/devin-reminders-action@v1
  with:
    action: cancel
    agent-session-url: "https://app.devin.ai/sessions/abc123"
    cancel-guids: '["abc-123", "def-456"]'
    devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
    slack-token: ${{ secrets.SLACK_BOT_TOKEN }}
    slack-channel: devin-reminders
```

### Full Workflow (Cron + Manual)

```yaml
name: Devin Reminders Workflow
run-name: "Devin Reminders (${{ inputs.action || 'cron' }})"

on:
  schedule:
    - cron: "*/30 * * * *"

  workflow_dispatch:
    inputs:
      action:
        description: "Action to perform: put, list, cancel, or cron"
        required: true
        type: choice
        options:
          - cron
          - list
          - put
          - cancel
      reminder_message:
        description: "Reminder message to deliver (required for 'put')."
        required: false
        type: string
      remind_at:
        description: >
          ISO 8601 timestamp with timezone offset for when the reminder fires
          (required for 'put'). Must be in the future and no more than 3 days ahead.
          Example: 2026-02-20T17:00:00-08:00
        required: false
        type: string
      agent_session_url:
        description: "Devin session URL (required for 'put' and 'cancel')."
        required: false
        type: string
      slack_users_cc:
        description: >
          Comma-delimited list of Slack user tags to CC on notifications.
          Example: '<@U12345>, <@U67890>'
        required: false
        type: string
      cancel_guids:
        description: "JSON array of reminder GUIDs to cancel (required for 'cancel')."
        required: false
        type: string

jobs:
  list-reminders:
    name: List Reminders
    runs-on: ubuntu-latest
    if: ${{ inputs.action == 'list' }}
    permissions:
      contents: read
      actions: read
    steps:
      - name: Execute reminder action
        uses: airbytehq/devin-reminders-action@v0.4.0
        with:
          action: 'list'
          reminder-timezone: America/Los_Angeles
          devin-token: ${{ secrets.DEVIN_AI_API_KEY }}

  create-new-reminder:
    name: Create New Reminder
    runs-on: ubuntu-latest
    if: ${{ inputs.action == 'put' }}
    permissions:
      contents: read
      actions: write
    steps:
      - name: Execute reminder action
        uses: airbytehq/devin-reminders-action@v0.4.0
        with:
          action: 'put'
          lock-mode: auto
          reminder-timezone: America/Los_Angeles
          devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
          slack-token: ${{ secrets.SLACK_BOT_TOKEN }}
          slack-channel: devin-reminders
          remind-at: ${{ inputs.remind_at }}
          reminder-message: ${{ inputs.reminder_message }}
          agent-session-url: ${{ inputs.agent_session_url }}
          slack-users-cc: ${{ inputs.slack_users_cc }}

  cancel-reminders:
    name: Cancel Reminders
    runs-on: ubuntu-latest
    if: ${{ inputs.action == 'cancel' }}
    permissions:
      contents: read
      actions: write
    steps:
      - name: Execute reminder action
        uses: airbytehq/devin-reminders-action@v0.4.0
        with:
          action: 'cancel'
          lock-mode: auto
          reminder-timezone: America/Los_Angeles
          devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
          slack-token: ${{ secrets.SLACK_BOT_TOKEN }}
          slack-channel: devin-reminders
          agent-session-url: ${{ inputs.agent_session_url }}
          cancel-guids: ${{ inputs.cancel_guids }}

  process-reminders-due:
    name: Process Reminders Due
    runs-on: ubuntu-latest
    if: ${{ github.event_name == 'schedule' || inputs.action == 'cron' }}
    permissions:
      contents: read
      actions: write
    steps:
      - name: Execute reminder action
        uses: airbytehq/devin-reminders-action@v0.4.0
        with:
          action: 'cron'
          lock-mode: auto
          reminder-timezone: America/Los_Angeles
          devin-token: ${{ secrets.DEVIN_AI_API_KEY }}
          slack-token: ${{ secrets.SLACK_BOT_TOKEN }}
          slack-channel: devin-reminders
```

## How It Works

1. **`put`** schedules a reminder by appending it to a JSON artifact
2. **`list`** reads the artifact, filters due reminders, and outputs counts and JSON
3. **`cancel`** removes reminders matching the given GUIDs for the given session URL from the artifact
4. **`cron`** reads the artifact, fires each due reminder via the Devin API, pops successful ones, and uploads the updated artifact

## Storage Model

Reminders are stored as a JSON array in a GitHub Actions artifact (`devin-reminders-list`). The artifact is persisted across workflow runs using `actions/download-artifact@v4` with a `run-id` lookup via the GitHub API, and updated via `actions/upload-artifact@v4` with `overwrite: true`. Old artifacts expire via the 4-day retention policy.

> **Important:** All reminder actions (`put`, `list`, `cancel`, `cron`) must be defined in a **single workflow file**. GitHub Actions only allows workflows to upload artifacts to their own workflow run, so splitting actions across multiple workflow files would prevent them from sharing the same artifact.

## Dependencies

- [`actions/download-artifact@v4`](https://github.com/actions/download-artifact) — cross-run artifact download
- [`actions/upload-artifact@v4`](https://github.com/actions/upload-artifact) — artifact upload
- [`slackapi/slack-github-action@v2`](https://github.com/slackapi/slack-github-action) — Slack notifications (optional)
- [`guibranco/github-artifact-lock-action@v3.0.14`](https://github.com/guibranco/github-artifact-lock-action) — artifact-based mutex locking to prevent race conditions on concurrent `put`/`cron` runs

## License

This project is licensed under the terms of the [MIT License](LICENSE).
