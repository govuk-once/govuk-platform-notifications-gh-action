# Slack Notification Actions

A set of composite GitHub Actions that send notifications to Slack via the GOV.UK One notifications hub. Each action targets a specific use case with only the inputs it needs.

## Contents

- [Overview](#overview)
- [Actions](#actions)
  - [release](#release)
  - [security-alerts](#security-alerts)
  - [info](#info)
- [How it works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Examples](#examples)
  - [Release — triggered by a GitHub release](#release--triggered-by-a-github-release)
  - [Security alerts — ZAP scan findings](#security-alerts--zap-scan-findings)
  - [Info — simple informational message](#info--simple-informational-message)
- [Formatting the message field](#formatting-the-message-field)
- [Storing shared configuration](#storing-shared-configuration)
- [IAM permissions](#iam-permissions)
- [Troubleshooting](#troubleshooting)

---

## Overview


| Action | Use case | 
|---|---|
| [`release`](#release) | New version published |
| [`security-alerts`](#security-alerts) | Security scan findings |
| [`info`](#info) | General informational messages |

Reference them as subdirectory actions:

```yaml
uses: govuk-once/govuk-platform-notifications-gh-action/release@main
uses: govuk-once/govuk-platform-notifications-gh-action/security-alerts@main
uses: govuk-once/govuk-platform-notifications-gh-action/info@main
```

---

## Actions

### release

Sends a release notification. Sidebar is always purple.

```yaml
uses: govuk-once/govuk-platform-notifications-gh-action/release@main
```

#### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `title` | **Yes** | — | Short title (e.g. `"my-service v1.5.0"`). |
| `repository` | **Yes** | — | Repository name (e.g. `govuk-once/my-service`). |
| `tag` | **Yes** | — | Version tag (e.g. `v1.5.0`). |
| `message` | **Yes** | — | Body text. Supports Markdown — works well with GitHub's auto-generated release notes. |
| `slack_channel` | **Yes** | — | Target Slack channel (e.g. `#my-channel`). |
| `aws_account_id` | **Yes** | — | AWS account ID where the `notifications-spoke` queue is deployed. |
| `aws_region` | **Yes** | — | AWS region of the spoke SQS queue. |

---

### security-alerts

Sends a security alert notification (e.g. ZAP scan findings).

```yaml
uses: govuk-once/govuk-platform-notifications-gh-action/security-alerts@main
```

#### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `title` | **Yes** | — | Short title (e.g. `"New ZAP security findings"`). |
| `finding_count` | **Yes** | — | Number of findings. |
| `findings` | **Yes** | — | Findings content. |
| `repository` | **Yes** | — | Repository name (e.g. `govuk-once/my-service`). |
| `workflow_url` | **Yes** | — | URL of the GitHub Actions workflow run. |
| `message` | **Yes** | — | Body text. Supports Markdown. |
| `slack_channel` | **Yes** | — | Target Slack channel (e.g. `#my-channel`). |
| `aws_account_id` | **Yes** | — | AWS account ID where the `notifications-spoke` queue is deployed. |
| `aws_region` | **Yes** | — | AWS region of the spoke SQS queue. |

---

### info

Sends a general informational notification. The simplest action — just a title and message.

```yaml
uses: govuk-once/govuk-platform-notifications-gh-action/info@main
```

#### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `title` | **Yes** | — | Short title of the notification. |
| `message` | **Yes** | — | Body text. Supports Markdown. |
| `slack_channel` | **Yes** | — | Target Slack channel (e.g. `#my-channel`). |
| `aws_account_id` | **Yes** | — | AWS account ID where the `notifications-spoke` queue is deployed. |
| `aws_region` | **Yes** | — | AWS region of the spoke SQS queue. |

---

## Adding a new notification type

New actions can be added based on message types:

1. Create a new subdirectory with an `action.yaml` following the existing examples (hardcode the `message_type` in the jq template).
2. Create a matching Slack template in the hub Lambda at [notifications/hub/templates](https://github.com/govuk-once/aws-account-bootstrapping/tree/main/libs/lambdas/notifications/hub/templates).

---

## How it works

```
Your GitHub Actions workflow
  │
  └─ The action builds a JSON payload and sends it to SQS
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

Each action does two things:

1. **Builds a JSON payload** from your inputs using `jq`. The `message_type` is hardcoded — you never set it.
2. **Sends the payload to SQS** using `aws sqs send-message`.

All formatting, templating, and Slack API interaction happens downstream in the hub Lambda.

---

## Prerequisites

1. **Notifications spoke deployed in your AWS account.** Speak to the platform team if it is not set up yet.
2. **AWS credentials configured in your workflow.** Use `aws-actions/configure-aws-credentials` (or equivalent) **before** calling any of these actions.
3. **IAM permissions.** The GitHub Actions role must have `sqs:SendMessage` on the spoke queue. See [IAM permissions](#iam-permissions).
4. **Slack bot invited to the target channel.** Use `/invite @bot-name` in Slack.

---

## Examples

### Release — triggered by a GitHub release

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
        uses: govuk-once/govuk-platform-notifications-gh-action/release@main
        with:
          aws_account_id: ${{ vars.AWS_ACCOUNT_ID }}
          title: "${{ github.event.release.name }}"
          message: "${{ github.event.release.body }}\n\n<${{ github.event.release.html_url }}|View release>"
          repository: ${{ github.repository }}
          tag: ${{ github.event.release.tag_name }}
          slack_channel: "#releases"
```

### Security alerts — ZAP scan findings

```yaml
      - name: Send ZAP notification
        if: steps.new_zap_findings.outputs.count != '0'
        uses: govuk-once/govuk-platform-notifications-gh-action/security-alerts@main
        with:
          aws_account_id: ${{ steps.aws-account.outputs.account-id }}
          title: "New ZAP security findings"
          message: "New findings detected during the authenticated ZAP API scan."
          finding_count: ${{ steps.new_zap_findings.outputs.count }}
          findings: ${{ steps.zap_notification.outputs.findings }}
          repository: ${{ github.repository }}
          workflow_url: "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          slack_channel: "#govuk-once-flex-alerting-dev"
```

### Info — simple informational message

```yaml
      - name: Send info notification
        uses: govuk-once/govuk-platform-notifications-gh-action/info@main
        with:
          aws_account_id: ${{ steps.aws-account.outputs.account-id }}
          title: "New ZAP security findings"
          message: "my message"
          slack_channel: "#govuk-once-flex-alerting-dev"
```

---

## Formatting the message field

The `message` field supports Markdown, which the hub converts to Slack's mrkdwn format:

| Syntax | Result in Slack |
|---|---|
| `\n` | Line break |
| `- item` or `* item` | Bullet point |
| `1. item` | Numbered list item |
| `## Heading` | **Bold heading** |
| `**bold text**` | **Bold text** |
| `[link text](https://url)` | Clickable link |
| `` `code` `` | Inline code |
| `<https://url\|link text>` | Clickable link (Slack native syntax) |

---

## Storing shared configuration

Store `AWS_ACCOUNT_ID` as a GitHub Actions **variable** (not a secret — it is not sensitive):

| Level | Where to set it | When to use |
|---|---|---|
| **Organisation** | GitHub org > Settings > Secrets and variables > Actions > Variables | Same account ID for all repos |
| **Repository** | Repo > Settings > Secrets and variables > Actions > Variables | Per-repo override |
| **Environment** | Repo > Settings > Environments > \<env\> > Variables | Different account per environment |

Reference it as `${{ vars.AWS_ACCOUNT_ID }}`.

---

## IAM permissions

The GitHub Actions role must have permission to send messages to the spoke queue:

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

Replace `<ACCOUNT_ID>` with the AWS account ID where the spoke is deployed.

---

## Troubleshooting

### No message appears in Slack

1. **Check the workflow logs.** The "Send notification" step prints the SQS `SendMessage` response. A `MessageId` confirms the message reached the queue.
2. **Check the spoke DLQ.** Failed messages end up in `notifications-spoke-dlq`:
   ```sh
   aws sqs get-queue-attributes \
     --queue-url "https://sqs.eu-west-2.amazonaws.com/<ACCOUNT_ID>/notifications-spoke-dlq" \
     --attribute-names ApproximateNumberOfMessagesVisible
   ```
3. **Check the hub Lambda logs.** Look in CloudWatch Logs under `/aws/lambda/notifications-poster` in the hub account.
4. **Check the Slack bot is in the channel.** Use `/invite @bot-name` in Slack.

### AccessDenied or AuthorizationError

The GitHub Actions role is missing `sqs:SendMessage` permission. See [IAM permissions](#iam-permissions).

### Empty fields in the notification

If fields like Environment or Region show "unknown", the notifications spoke may not be deployed in your account. Contact the platform team.
