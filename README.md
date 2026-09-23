# Slack Notification Action

A composite GitHub Action that sends deployment and release notifications to Slack via the GOV.UK One notifications hub.

## Contents

- [Overview](#overview)
- [How it works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Inputs](#inputs)
- [Templates](#templates)
  - [Deployment](#deployment)
  - [Release](#release)
- [Examples](#examples)
  - [Deployment — notify on success and failure](#deployment--notify-on-success-and-failure)
  - [Deployment — with a description](#deployment--with-a-description)
  - [Release — triggered by a GitHub release](#release--triggered-by-a-github-release)
  - [Release — manual test via workflow_dispatch](#release--manual-test-via-workflow_dispatch)
- [Formatting the description field](#formatting-the-description-field)
- [Storing shared configuration](#storing-shared-configuration)
- [IAM permissions](#iam-permissions)
- [Troubleshooting](#troubleshooting)

---

## Overview

This action sends a JSON message to an SQS queue in your AWS account. A central hub service picks up the message, applies a Slack template based on the `message_type`, and posts a formatted notification to the specified Slack channel.

Two message types are supported:

| Type | Sidebar colour | Icon | Use case |
|---|---|---|---|
| `deployment` | Green (success) or Red (failure) | :rocket: | CI/CD pipeline outcomes |
| `release` | Purple | :package: | New version published |

---

## How it works

```
Your GitHub Actions workflow
  │
  └─ This action builds a JSON payload and sends it to SQS
       │
       └─ SQS queue (notifications-spoke) in your AWS account
            │
            └─ EventBridge Pipe adds provenance metadata
                 (account name, region, timestamp)
                 │
                 └─ Forwarded to the central notifications hub
                      │
                      └─ Lambda formats and posts to Slack
```

The action itself does two things:

1. **Builds a JSON payload** from the inputs you provide, using `jq` to safely handle special characters.
2. **Sends the payload to SQS** using `aws sqs send-message`, constructing the queue URL from `aws_account_id` and `aws_region`.

All formatting, templating, and Slack API interaction happens downstream in the hub Lambda — the action is a thin client that puts the right message on the queue.

---

## Prerequisites

Before using this action, ensure the following are in place:

1. **Notifications spoke deployed in your AWS account.**
   The spoke creates the `notifications-spoke` SQS queue that this action sends to. Speak to the platform team if it is not set up yet.

2. **AWS credentials configured in your workflow.**
   Use `aws-actions/configure-aws-credentials` (or equivalent) in a step **before** calling this action. The action uses whatever credentials are already in the runner environment.

3. **IAM permissions.**
   The GitHub Actions role must have `sqs:SendMessage` on the spoke queue. See [IAM permissions](#iam-permissions) for the policy.

4. **Slack bot invited to the target channel.**
   The hub's Slack bot must be a member of any channel you want to post to.

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `message` | **Yes** | — | Short title shown in bold at the top of the notification. |
| `message_type` | **Yes** | — | Template to use: `deployment` or `release`. |
| `aws_account_id` | **Yes** | — | AWS account ID where the `notifications-spoke` queue is deployed. |
| `status` | No | `""` | Deployment outcome: `Success` or `Failed`. Controls the sidebar colour and status icon for deployment notifications. |
| `description` | No | `""` | Body text shown below the title. Supports Markdown formatting — see [Formatting the description field](#formatting-the-description-field). |
| `slack_channel` | No | `""` | Target Slack channel (e.g. `#my-channel`). If omitted, the hub posts to its configured default channel. |
| `repository` | No | `""` | Repository name shown in the notification. Used with the `release` template. |
| `tag` | No | `""` | Version tag shown in the notification. Used with the `release` template. |
| `aws_region` | No | `eu-west-2` | AWS region of the spoke SQS queue. Only change this if your spoke is in a different region. |

---

## Templates

### Deployment

Use `message_type: deployment` for CI/CD pipeline notifications. The sidebar colour and status icon are automatically derived from the `status` input.

**What the notification looks like:**

```
┌──────────────────────────────────────────────────────────┐
│ 🟩 (or 🟥)                                               │
│ 🚀 Deployment: my-service abc1234                        │
│                                                          │
│ Deployed to production by github-actions.                │
│                                                          │
│ Status              Environment                          │
│ ✅ Success           myproject-production                │
│                                                          │
│ Region                                                   │
│ eu-west-2                                                │
│                                                          │
│ myproject-production | eu-west-2 | 123456789012 | ...    │
└──────────────────────────────────────────────────────────┘
```

**Automatic colour and icon behaviour:**

| `status` value | Sidebar colour | Status icon |
|---|---|---|
| Contains `success` (case-insensitive) | Green | :white_check_mark: |
| Contains `fail` (case-insensitive) | Red | :x: |
| Anything else | Blue | :white_check_mark: |

The title icon is always :rocket: for deployments.

**Fields displayed:** Status, Environment, Region.

---

### Release

Use `message_type: release` for notifications about new versions. The sidebar is always purple, and the title icon is always :package:.

**What the notification looks like:**

```
┌──────────────────────────────────────────────────────────┐
│ 🟪                                                       │
│ 📦 Release: my-service v1.5.0                            │
│                                                          │
│ *Features*                                               │
│ • New login flow (abc1234)                               │
│                                                          │
│ *Bug Fixes*                                              │
│ • Fixed timeout on /health (def5678)                     │
│                                                          │
│ View release                                             │
│                                                          │
│ Repository            Version                            │
│ govuk-once/my-svc     v1.5.0                             │
│                                                          │
│ myproject-production | eu-west-2 | 123456789012 | ...    │
└──────────────────────────────────────────────────────────┘
```

**Fields displayed:** Repository, Version.

> **Tip:** GitHub's **Auto-generate release notes** feature produces Markdown with headings and bullet lists. The hub automatically converts this to Slack's mrkdwn format, so the release notes render correctly without any extra work.

---

## Examples

### Deployment — notify on success and failure

Add these two steps at the end of your deployment job. GitHub evaluates `if: success()` and `if: failure()` against the outcome of previous steps.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      # ... your deployment steps ...

      - name: Configure AWS credentials
        if: always()
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/<your-role>
          aws-region: eu-west-2

      - name: Notify deployment success
        if: success()
        uses: govuk-once/infra-temp/.github/actions/slack-notification@main
        with:
          aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
          message_type: deployment
          message: "my-service ${{ github.sha }}"
          status: Success
          slack_channel: "#deployments"

      - name: Notify deployment failure
        if: failure()
        uses: govuk-once/infra-temp/.github/actions/slack-notification@main
        with:
          aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
          message_type: deployment
          message: "my-service ${{ github.sha }}"
          status: Failed
          slack_channel: "#deployments"
```

### Deployment — with a description

Include additional context in the notification body:

```yaml
      - name: Notify deployment success
        if: success()
        uses: govuk-once/infra-temp/.github/actions/slack-notification@main
        with:
          aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
          message_type: deployment
          message: "my-service ${{ github.sha }}"
          status: Success
          description: "Deployed to ${{ inputs.environment }} by ${{ github.actor }}."
          slack_channel: "#deployments"

      - name: Notify deployment failure
        if: failure()
        uses: govuk-once/infra-temp/.github/actions/slack-notification@main
        with:
          aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
          message_type: deployment
          message: "my-service ${{ github.sha }}"
          status: Failed
          description: "Deployment to ${{ inputs.environment }} failed.\n\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View run>"
          slack_channel: "#deployments"
```

### Release — triggered by a GitHub release

Trigger the workflow on `release: published`. The release name, body, tag, and URL are all available from `github.event.release`.

```yaml
name: Release notification

on:
  release:
    types: [published]

jobs:
  notify:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/<your-role>
          aws-region: eu-west-2

      - name: Send release notification
        uses: govuk-once/infra-temp/.github/actions/slack-notification@main
        with:
          aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
          message_type: release
          message: "${{ github.event.release.name }}"
          description: "${{ github.event.release.body }}\n\n<${{ github.event.release.html_url }}|View release>"
          repository: ${{ github.repository }}
          tag: ${{ github.event.release.tag_name }}
          slack_channel: "#releases"
```

### Release — manual test via workflow_dispatch

Add `workflow_dispatch` inputs to test the notification without creating a real release:

```yaml
name: Release notification

on:
  release:
    types: [published]
  workflow_dispatch:
    inputs:
      release_name:
        description: "Release name (e.g. my-service v1.2.3)"
        required: true
      release_body:
        description: "Release notes (supports markdown)"
        required: false
        default: "## Features\n- Test feature"
      release_url:
        description: "Release URL"
        required: false
        default: "https://github.com"

jobs:
  notify:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/<your-role>
          aws-region: eu-west-2

      - name: Send release notification
        uses: govuk-once/infra-temp/.github/actions/slack-notification@main
        with:
          aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
          message_type: release
          message: "${{ github.event.release.name || inputs.release_name }}"
          description: "${{ github.event.release.body || inputs.release_body }}\n\n<${{ github.event.release.html_url || inputs.release_url }}|View release>"
          repository: ${{ github.repository }}
          tag: "${{ github.event.release.tag_name || inputs.release_name }}"
          slack_channel: "#releases"
```

Run it from the **Actions** tab in GitHub, selecting "Run workflow" and filling in the test inputs.

---

## Formatting the description field

The `description` field supports Markdown syntax, which the hub automatically converts to Slack's mrkdwn format:

| Syntax | Result in Slack |
|---|---|
| `\n` | Line break |
| `- item` or `* item` | • Bullet point |
| `1. item` | Numbered list item |
| `## Heading` | **Bold heading** |
| `**bold text**` | **Bold text** |
| `[link text](https://url)` | Clickable link |
| `` `code` `` | Inline code |
| `<https://url\|link text>` | Clickable link (Slack native syntax) |

**Example:**

```yaml
description: "Deployment complete.\n\n**Changes included:**\n- Updated authentication flow\n- Fixed timeout on /health endpoint\n\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View workflow run>"
```

---

## Storing shared configuration

Rather than hardcoding the AWS account ID in every workflow, store it as a GitHub Actions **variable** (not a secret — it's not sensitive):

| Level | Where to set it | When to use |
|---|---|---|
| **Organisation** | GitHub org → Settings → Secrets and variables → Actions → Variables | Same account ID for all repos |
| **Repository** | Repo → Settings → Secrets and variables → Actions → Variables | Per-repo override |
| **Environment** | Repo → Settings → Environments → \<env\> → Variables | Different account per environment (dev/staging/prod) |

Set a variable named `AWS_ACCOUNT_ID` with the 12-digit account number, then reference it in workflows as `${{ vars.AWS_ACCOUNT_ID }}`.

If your deployment workflows already target different environments, use environment-scoped variables so the correct account ID is selected automatically:

```yaml
jobs:
  deploy:
    environment: production   # selects this environment's variables
    steps:
      - uses: govuk-once/infra-temp/.github/actions/slack-notification@main
        with:
          aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}   # resolves to the production account ID
          ...
```

---

## IAM permissions

The GitHub Actions role configured with `aws-actions/configure-aws-credentials` must have permission to send messages to the spoke queue:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:eu-west-2:<ACCOUNT_ID>:notifications-spoke"
    }
  ]
}
```

Replace `<ACCOUNT_ID>` with the AWS account ID where the spoke is deployed. If you use multiple regions, add an entry for each.

---

## Troubleshooting

### No message appears in Slack

1. **Check the workflow logs.** The "Send notification" step prints the SQS `SendMessage` response. A `MessageId` in the output confirms the message reached the queue.
2. **Check the spoke DLQ.** If the EventBridge Pipe or forwarding rule fails, the message ends up in the `notifications-spoke-dlq` queue:
   ```sh
   aws sqs get-queue-attributes \
     --queue-url "https://sqs.eu-west-2.amazonaws.com/<ACCOUNT_ID>/notifications-spoke-dlq" \
     --attribute-names ApproximateNumberOfMessagesVisible
   ```
3. **Check the hub Lambda logs.** The hub Lambda logs every message it processes. Look for errors in CloudWatch Logs under `/aws/lambda/notifications-poster` in the hub account.
4. **Check the Slack bot is in the channel.** The bot must be invited to the target channel — use `/invite @bot-name` in Slack.

### AccessDenied or AuthorizationError

The GitHub Actions role is missing `sqs:SendMessage` permission. See [IAM permissions](#iam-permissions).

### Empty fields in the notification

If fields like Environment or Region show "unknown", the notifications spoke may not be deployed in your account (the spoke wraps each message with account metadata). Contact the platform team.

### The message field is empty

If `message` resolves to an empty string (e.g. `${{ github.event.release.name }}` on a `workflow_dispatch` trigger), the hub Lambda skips the message entirely. Use a fallback:
```yaml
message: "${{ github.event.release.name || inputs.release_name }}"
```
