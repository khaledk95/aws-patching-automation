# AWS Automated EC2 Patching Solution

A single, self-contained CloudFormation template that automates patching of EC2 instances, grouped by a `PatchGroup` tag, using AWS Step Functions, Lambda, Systems Manager, EventBridge Scheduler, SNS and S3.

The workflow supports two modes:

- **`install`** — takes a pre-patch AMI backup of every target instance, waits for the AMIs to become available, then runs `AWS-RunPatchBaseline` with `Operation=Install` and `RebootIfNeeded`.
- **`scan`** — no AMIs, no reboot. Runs `AWS-RunPatchBaseline` with `Operation=Scan`, then builds a patch compliance report and uploads it to S3.

Everything is deployed by one file: [`AWSPatchingAutomation.yaml`](AWSPatchingAutomation.yaml). All Lambda code is inline in the template — there is nothing to build, package or upload.

---

## Table of contents

- [Architecture](#architecture)
- [What the template creates](#what-the-template-creates)
- [Parameters](#parameters)
- [Prerequisites](#prerequisites)
- [Deployment](#deployment)
- [Post-deployment steps](#post-deployment-steps)
- [Usage](#usage)
- [Workflow in detail](#workflow-in-detail)
- [Notifications](#notifications)
- [Compliance reports](#compliance-reports)
- [IAM and security](#iam-and-security)
- [Monitoring and logs](#monitoring-and-logs)
- [Operational notes and limitations](#operational-notes-and-limitations)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)

---

## Architecture

```
EventBridge Scheduler (up to 5 schedules, one per PatchGroup)
          │  Input: {"detail":{"PatchGroup":"<group>","PatchMode":"install"|"scan"}}
          ▼
   Step Functions state machine
          │
          ├── DeterminePatchMode (Choice)
          │
   ┌──────┴───────────────────────────┐
   │ install                          │ scan
   ▼                                  ▼
CreateAMIs ─────────┐          GetInstancesForScan
  (ec2:CreateImage) │            (discover only)
   │                │                 │
CheckAMIStatus ◄────┘          RunScan (SSM Scan, NoReboot)
   │ poll 30s                         │
RunPatching                    CheckScanStatus
  (SSM Install,                       │ poll 60s
   RebootIfNeeded)              GeneratePatchReport ──► S3 bucket
   │                                  │
CheckPatchingStatus                   │
   │ poll 60s                         │
   └──────────────┬───────────────────┘
                  ▼
            SNS topic ──► email
```

Services used:

| Service | Role |
| --- | --- |
| **Step Functions** | Orchestrates the whole workflow, including polling loops and error handling |
| **Lambda** (6 functions) | Discovers instances, creates AMIs, sends SSM commands, polls status, builds reports, sends notifications |
| **Systems Manager** | Runs `AWS-RunPatchBaseline` on the instances and supplies patch compliance data |
| **EventBridge Scheduler** | Optional recurring triggers, one schedule per patch group |
| **SNS** | Email notifications at each milestone |
| **S3** | Stores patch compliance reports (scan mode) |
| **CloudWatch Logs** | Execution and function logs, 30-day retention |

---

## What the template creates

**S3 bucket** — `<ReportBucketNamePrefix>-missing-patches-report`, AES256 server-side encryption, all public access blocked.

**Step Functions state machine** — `<StackName>-step-function`, logging level `ALL` with execution data, and a 6-hour execution timeout (`TimeoutSeconds: 21600`).

**Six Lambda functions** (`python3.14`, 128 MB, code inline in the template):

| Function | Purpose | Timeout |
| --- | --- | --- |
| `<StackName>-create-ami-function` | Discovers tagged instances; creates pre-patch AMIs; also serves the `checkStatus` and `discover` operations | 300 s |
| `<StackName>-run-patching-function` | Sends `AWS-RunPatchBaseline` with `Operation=Install`, `RebootIfNeeded` | 300 s |
| `<StackName>-run-scan-function` | Sends `AWS-RunPatchBaseline` with `Operation=Scan`, `NoReboot` | 300 s |
| `<StackName>-check-status-function` | Polls SSM command invocation status per instance | 300 s |
| `<StackName>-patch-compliance-report-function` | Collects non-compliant patches and writes the report to S3 | 180 s |
| `<StackName>-send-notification-function` | Formats and publishes messages to SNS | 60 s |

**SNS topic** — `<StackName>-alerts`, encrypted with the AWS managed key `alias/aws/sns`, with an email subscription for `EmailAddress`.

**Three IAM roles** — one for Step Functions, one shared by all Lambda functions, one for EventBridge Scheduler. Role names are explicit, so deployment requires the `CAPABILITY_NAMED_IAM` capability.

**Seven CloudWatch log groups** — declared explicitly so that 30-day retention is enforced rather than left unlimited.

**Up to five EventBridge schedules** — one per non-blank `PatchGroupN` parameter. Each is created **disabled**, with a placeholder date and a placeholder patch mode, and must be edited after deployment.

---

## Parameters

| Parameter | Required | Description |
| --- | --- | --- |
| `EmailAddress` | Yes | Address subscribed to the SNS topic for all notifications. Must be confirmed by email before any alert is delivered. |
| `ReportBucketNamePrefix` | Yes | Prefix for the report bucket. The bucket is created as `<prefix>-missing-patches-report`. Must be lowercase letters, numbers and hyphens, starting and ending alphanumeric (2–40 characters). |
| `PatchGroup1` … `PatchGroup5` | No | `PatchGroup` tag values. One EventBridge schedule is created per non-blank slot. Leave blank to skip. |

Patch group slots are independent: filling in a new blank slot and updating the stack adds a schedule without modifying schedules you have already configured, because CloudFormation does not touch resources whose template definition is unchanged.

> **Note on bucket naming:** S3 bucket names are globally unique, and the IAM role names in this template are account-wide. Deploying the same stack name twice in the same account (for example in two regions) will therefore collide. Use a distinct stack name and bucket prefix per deployment.

---

## Prerequisites

1. **SSM Agent** installed and running on every target instance. It is preinstalled on current Amazon Linux, Ubuntu and Windows Server AMIs.
2. **Instance profile** with the `AmazonSSMManagedInstanceCore` managed policy attached, so the instance registers as an SSM managed node.
3. **Network path to Systems Manager** — either outbound internet access via NAT/IGW, or VPC endpoints for `ssm`, `ssmmessages` and `ec2messages`.
4. **`PatchGroup` tag** on every instance you want this solution to manage.
5. **A patch baseline associated with each patch group.** *This template does not create or register any patch baseline.* Without an explicit association, Systems Manager applies the default baseline for the operating system. See [Custom patch baselines](#custom-patch-baselines).
6. Permission to create named IAM roles in the target account.

### Tagging instances

Console: select the instances, then **Actions → Instance settings → Manage tags**.

CLI:

```bash
aws ec2 create-tags --resources i-1234567890abcdef0 i-0fedcba9876543210 --tags Key=PatchGroup,Value=production-web
```

Only instances in the **`running`** state are discovered.

---

## Deployment

### AWS Console

1. Download `AWSPatchingAutomation.yaml`.
2. Open **CloudFormation → Create stack → With new resources (standard)**.
3. Choose **Upload a template file**, select the YAML, and continue.
4. Set the stack name and parameters (see [Parameters](#parameters)).
5. On the review page, tick **"I acknowledge that AWS CloudFormation might create IAM resources with custom names."**
6. Create the stack.

### AWS CLI

```bash
aws cloudformation deploy --template-file AWSPatchingAutomation.yaml --stack-name automated-patching --capabilities CAPABILITY_NAMED_IAM --parameter-overrides EmailAddress=you@example.com ReportBucketNamePrefix=mycompany-prod PatchGroup1=production-web PatchGroup2=production-db
```

`CAPABILITY_NAMED_IAM` is required because the IAM roles use explicit names.

### Stack outputs

| Output | Description |
| --- | --- |
| `StepFunctionArn` | ARN of the state machine, needed for on-demand runs |
| `SNSTopicArn` | ARN of the notification topic |
| `S3BucketName` | Report bucket name |
| `ScheduleArn1` … `ScheduleArn5` | ARN of each created schedule |
| `DirectInstallTriggerCommand` | Ready-to-run CLI command for an install run |
| `DirectScanTriggerCommand` | Ready-to-run CLI command for a scan run |
| `ScheduleSetupNote`, `PatchModeNote`, `DynamicPatchGroupNote` | Reminders about required post-deployment configuration |

---

## Post-deployment steps

Both steps are mandatory — the solution does nothing useful until they are done.

### 1. Confirm the SNS subscription

Open the **"AWS Notification - Subscription Confirmation"** email and click the confirmation link. Until you do, no notifications are delivered. Check spam if it has not arrived.

### 2. Configure each schedule

Every schedule is deliberately created in a non-functional state so that nothing can patch anything before you have reviewed it:

- **State:** `DISABLED`
- **Schedule expression:** `at(2090-09-09T05:09:00)` — a placeholder
- **Timezone:** `Asia/Dubai`
- **Target input:** `{"detail":{"PatchGroup":"<your group>","PatchMode":"YOUR_PATCH_MODE_VALUE"}}`

For each schedule, in **EventBridge → Scheduler → Schedules**:

1. Select the schedule named `<StackName>-<PatchGroup>-schedule` and choose **Edit**.
2. Replace the placeholder expression with a real recurring one, for example `cron(0 2 ? * SUN *)` for 02:00 every Sunday.
3. Set the timezone you actually want (the template defaults to `Asia/Dubai`).
4. In the target input, replace `YOUR_PATCH_MODE_VALUE` with **`install`** or **`scan`** — lowercase.
5. Set **State** to **Enabled** and save.

> **Important:** `PatchMode` is compared case-sensitively against the exact string `scan`. Any other value — including `Scan`, `SCAN`, or the unedited `YOUR_PATCH_MODE_VALUE` — takes the **install** path, which creates AMIs and reboots instances. Double-check this field before enabling a schedule.

Each schedule uses a 2-minute flexible time window and does not retry on failure (`MaximumRetryAttempts: 0`).

---

## Usage

### On-demand run

Install mode — creates AMI backups, patches, reboots if needed:

```bash
aws stepfunctions start-execution --state-machine-arn <StepFunctionArn> --input '{"detail":{"PatchGroup":"production-web","PatchMode":"install"}}'
```

Scan mode — no AMIs, no reboot, produces a compliance report:

```bash
aws stepfunctions start-execution --state-machine-arn <StepFunctionArn> --input '{"detail":{"PatchGroup":"production-web","PatchMode":"scan"}}'
```

### Input format

```json
{
  "detail": {
    "PatchGroup": "production-web",
    "PatchMode": "scan"
  }
}
```

Both fields should always be supplied. `PatchGroup` must match the tag value exactly. Always include `PatchMode` explicitly rather than relying on a default.

### Recommended first run

Run a **scan** against a non-production patch group first. It makes no changes and no reboots, and the resulting report confirms that instances are discoverable, SSM-managed and reporting compliance data correctly.

---

## Workflow in detail

The state machine begins at `DeterminePatchMode`, which routes on `$.detail.PatchMode`.

### Install path

| State | Behaviour |
| --- | --- |
| `CreateAMIs` | Finds running instances tagged with the patch group (paginated). For each, calls `ec2:CreateImage` with `NoReboot=true`, naming the AMI `<instance-name>-<instance-id>-AMI-BEFORE-PATCHING-<timestamp>`. Tags both image and snapshots with `PrePatchBackup`, `CreatedBy`, `PatchGroup`, `SourceInstanceId`, `SourceInstanceName`. |
| `CheckInstancesFound` | If no instances matched, sends an informational email and ends. |
| `CheckAMIStatus` → `WaitForAMIs` | Polls AMI state every 30 seconds. States `failed`, `error`, `invalid` and `deregistered` are treated as terminal. |
| `SendAMIFailureNotification` | If any AMI failed, **patching is aborted** and an error email is sent. |
| `SendPrePatchingNotification` | All AMIs available — sends the list of instances and AMIs. |
| `RunPatching` | Sends `AWS-RunPatchBaseline` with `Operation=Install`, `RebootOption=RebootIfNeeded`, `TimeoutSeconds=3600`. |
| `CheckPatchingStatus` → `WaitForPatchingToComplete` | Polls command status every 60 seconds until success or failure. |
| `SendSuccessNotification` / `SendFailureNotification` | Terminal email with per-instance status detail. |

Instances are patched only **after** every AMI has reached `available`, so a usable rollback point exists before any package changes.

### Scan path

| State | Behaviour |
| --- | --- |
| `GetInstancesForScan` | Reuses the AMI function with `operation: discover` — finds instances, creates **no** AMIs. |
| `CheckScanInstancesFound` | Informational email and exit if nothing matched. |
| `RunScan` | Sends `AWS-RunPatchBaseline` with `Operation=Scan`, `RebootOption=NoReboot`. |
| `CheckScanStatus` → `WaitForScanToComplete` | Polls every 60 seconds. |
| `GeneratePatchReport` | Queries `ssm:ListComplianceItems` for `NON_COMPLIANT` patches across up to 10 instances in parallel, formats a text report, uploads it to S3. |
| `SendScanSuccessNotification` / `SendScanFailureNotification` | Terminal email, including the report location on success. |

### Error handling

- Every task retries `Lambda.ServiceException`, `Lambda.AWSLambdaException`, `Lambda.SdkClientException`, `Lambda.TooManyRequestsException` and `States.TaskFailed` up to 3 times with a 5-second base interval and 2× backoff.
- Unhandled errors are caught and routed to `SendExecutionFailedNotification`, which references only fields guaranteed to exist, so it works regardless of which task failed.
- SSM's eventual consistency is handled explicitly: an `InvocationDoesNotExist` error immediately after `SendCommand` is reported as in-progress so the workflow re-polls instead of declaring a false failure.
- Patch commands target instances by **tag**, not by an explicit instance list, which avoids the 50-instance cap that applies to `InstanceIds`.

---

## Notifications

| Subject | When |
| --- | --- |
| `<Stack> - No Instances Found for Patch` | Install run matched no running tagged instances |
| `<Stack> - AMI Creation Failed` | A pre-patch AMI entered a failed state; patching aborted |
| `<Stack> - AMIs Created Successfully` | All backups ready, patching about to start |
| `<Stack> - Patching Completed Successfully` | All instances patched |
| `<Stack> - Patching Failed` | One or more instances failed to patch |
| `<Stack> - No Instances Found for Scan` | Scan run matched no running tagged instances |
| `<Stack> - Scan Completed Successfully` | Scan finished; includes report location |
| `<Stack> - Scan Failed` | Scan failed on one or more instances |
| `<Stack> - Patching Workflow Error` | Unexpected error after retries were exhausted |

Each message includes a timestamp, status, patch group, the affected instances (name and ID), any AMIs created, and — in scan mode — the report location. Timestamps are rendered in **UTC+4**, hardcoded in the template.

---

## Compliance reports

Scan runs write two objects per execution:

```
s3://<bucket>/reports/<PatchGroup>/<PatchGroup>-patch-compliance-report-<YYYYMMDD-HHMMSS>.txt
s3://<bucket>/reports/<PatchGroup>/<PatchGroup>-latest-patch-compliance-report.txt
```

The `-latest-` object is overwritten each run, which makes it convenient for dashboards and alerting.

The report contains a summary (instances scanned, instances with missing patches, total missing patches) followed by per-instance detail sorted by number of missing patches, with patches ordered by severity and CVE references listed where Systems Manager supplies them.

> **The link in the notification email is an S3 console URL, not a pre-signed URL.** The object stays private; the recipient must be signed in to the account and hold `s3:GetObject` on the bucket to open it. Reports are not publicly accessible.

Install runs do **not** generate a report — use scan mode for compliance reporting.

Fetch the latest report directly:

```bash
aws s3 cp s3://<bucket>/reports/production-web/production-web-latest-patch-compliance-report.txt -
```

---

## IAM and security

Security controls built into the template:

- **`ssm:SendCommand` is scoped to the `AWS-RunPatchBaseline` document only.** The roles cannot run arbitrary documents such as `AWS-RunShellScript`, so a compromise of the automation cannot be trivially turned into remote code execution.
- **`ssm:SendCommand` is additionally restricted to instances carrying a `PatchGroup` tag**, via a `ssm:resourceTag/PatchGroup` condition. Untagged instances are out of reach.
- **S3 bucket** blocks all public access and encrypts objects at rest with AES256.
- **SNS topic** is encrypted with the AWS managed key `alias/aws/sns`.
- **Log retention** is capped at 30 days on all seven log groups.
- **Reports are never made public** — notifications carry a console link, not a pre-signed URL.

Points worth reviewing against your own standards:

- `ec2:CreateImage`, `ec2:CreateTags`, `ec2:DescribeInstances` and `ec2:DescribeImages` are granted on `*`, since AMI IDs cannot be known in advance. `ec2:CreateTags` on `*` permits tagging any resource in the account.
- The Step Functions role carries EC2, SSM and SNS permissions it does not strictly need, because all real work is performed by the Lambda functions. It can safely be reduced to `lambda:InvokeFunction` plus the log-delivery permissions.
- IAM role names are explicit, so the stack is effectively one-per-account-per-name.

---

## Monitoring and logs

| Log group | Contents |
| --- | --- |
| `/aws/vendedlogs/states/<Stack>-step-function` | Full state machine execution history, including input/output |
| `/aws/lambda/<Stack>-create-ami-function` | AMI discovery and creation |
| `/aws/lambda/<Stack>-run-patching-function` | Install command dispatch |
| `/aws/lambda/<Stack>-run-scan-function` | Scan command dispatch |
| `/aws/lambda/<Stack>-check-status-function` | Per-instance command polling |
| `/aws/lambda/<Stack>-patch-compliance-report-function` | Report generation and upload |
| `/aws/lambda/<Stack>-send-notification-function` | Notification publishing |

The Step Functions console graph view is the fastest way to see where an execution is, which state it is looping in, and the exact input and output of each step.

---

## Operational notes and limitations

Things to know before running this in production:

- **`PatchMode` is case-sensitive.** Only the exact lowercase string `scan` selects scan mode. Anything else patches and reboots. Always set it explicitly in schedule inputs and on-demand calls.
- **Executions are capped at 6 hours.** The cap exists so a stuck polling loop cannot run for the Step Functions maximum of one year. If an execution is terminated by this timeout, **no notification is sent** — the timeout is not caught. Check the Step Functions console if an expected email never arrives.
- **Instances that are not SSM-managed will loop.** If a tagged, running instance never registers an SSM command invocation, status polling keeps returning in-progress until the 6-hour timeout. Confirm SSM registration before adding instances to a patch group.
- **Pre-patch AMIs are crash-consistent, not application-consistent.** `NoReboot=true` means the filesystem is captured without quiescing. For databases and other stateful workloads, take an application-level backup as well.
- **AMIs and snapshots are never deleted.** Every install run creates one AMI per instance and they accumulate, along with their EBS snapshot storage costs. Filter on the `PrePatchBackup=true` tag and add a separate lifecycle process — for example an Amazon Data Lifecycle Manager policy or a scheduled cleanup Lambda.
- **Reports accumulate too.** The bucket has no lifecycle rule; add an expiration rule if you want old reports aged out.
- **Instance targeting happens twice.** Instances are discovered up front by tag, but the SSM command also targets by tag at send time. An instance tagged in between is patched but not included in status polling or notifications.
- **Retries are not idempotent.** A partial failure in `CreateAMIs` followed by a retry can create duplicate AMIs; a retry of `RunPatching` can issue the patch command twice.
- **Report generation is bounded by a 180-second Lambda timeout** with 10 parallel workers. Very large patch groups may need a longer timeout or a batched approach.
- **No patch baseline is created.** Instances fall back to the default baseline for their OS unless you register one explicitly.
- **Timestamps are hardcoded to UTC+4** in AMI names, report filenames and email bodies, and schedules default to the `Asia/Dubai` timezone. Change these if you operate elsewhere.
- **Reboots are unconditional in install mode.** `RebootIfNeeded` reboots whenever a patch requires it, with no draining, load-balancer deregistration or maintenance-window coordination. Build that around the schedule if you need it.

---

## Troubleshooting

**No instances found**
Confirm the `PatchGroup` tag value matches the input exactly — it is case-sensitive — and that the instances are in the `running` state.

```bash
aws ec2 describe-instances --filters "Name=tag:PatchGroup,Values=production-web" "Name=instance-state-name,Values=running" --query "Reservations[].Instances[].[InstanceId,State.Name]" --output table
```

**Execution loops in `CheckPatchingStatus` or `CheckScanStatus`**
Usually an instance that is tagged and running but not registered with Systems Manager. Verify registration:

```bash
aws ssm describe-instance-information --query "InstanceInformationList[].[InstanceId,PingStatus,PlatformName]" --output table
```

An instance missing from this list is not manageable by SSM. Check the instance profile, SSM Agent status and network path to the SSM endpoints.

**Patching fails on some instances**
Inspect the per-instance detail in the failure email, then the command output:

```bash
aws ssm list-command-invocations --command-id <command-id> --details --query "CommandInvocations[].[InstanceId,Status,StatusDetails]" --output table
```

**No email notifications**
Confirm the SNS subscription is `Confirmed`:

```bash
aws sns list-subscriptions-by-topic --topic-arn <SNSTopicArn> --query "Subscriptions[].[Endpoint,SubscriptionArn]" --output table
```

A `SubscriptionArn` of `PendingConfirmation` means the link has not been clicked yet.

**Execution ended with no email at all**
Likely the 6-hour timeout, which is not caught. Open the execution in the Step Functions console and check the status and the last state reached.

**Report is empty or missing patches**
Compliance data comes from Systems Manager and only exists after a scan has run against the instance:

```bash
aws ssm list-compliance-items --resource-ids i-1234567890abcdef0 --resource-types ManagedInstance --filters "Key=ComplianceType,Values=Patch"
```

**Stack creation fails with an IAM capability error**
Add `--capabilities CAPABILITY_NAMED_IAM`, or tick the acknowledgment box in the console.

**Stack creation fails on the S3 bucket**
Bucket names are globally unique and must be lowercase. Choose a different `ReportBucketNamePrefix`.

---

## Custom patch baselines

The template does not manage baselines. To control exactly which patches are approved, create a baseline and register it with your patch group:

```bash
aws ssm create-patch-baseline --name "Custom-Production-Baseline" --description "Critical and important security updates, 7-day soak" --approval-rules 'PatchRules=[{PatchFilterGroup={PatchFilters=[{Key=CLASSIFICATION,Values=[SecurityUpdates]},{Key=SEVERITY,Values=[Critical,Important]}]},ApproveAfterDays=7}]'
```

```bash
aws ssm register-patch-baseline-for-patch-group --baseline-id pb-0123456789abcdef0 --patch-group "production-web"
```

The `--patch-group` value must match the `PatchGroup` tag value used by this solution.

---

## Adding another patch group

1. Set the next blank `PatchGroupN` parameter to the new tag value.
2. Update the stack.
3. Configure the new schedule as described in [Post-deployment steps](#post-deployment-steps).

Existing schedules are untouched, because CloudFormation only updates resources whose template definition has changed. For more than five groups, trigger the state machine directly or add further `PatchGroupN` parameters, conditions, schedules and outputs following the existing pattern.

---

## Cleanup

The report bucket must be emptied before the stack can be deleted.

```bash
aws s3 rm s3://<ReportBucketNamePrefix>-missing-patches-report --recursive
```

```bash
aws cloudformation delete-stack --stack-name automated-patching
```

Deleting the stack does **not** remove the AMIs or EBS snapshots created by install runs. List them and deregister any you no longer need:

```bash
aws ec2 describe-images --owners self --filters "Name=tag:PrePatchBackup,Values=true" --query "Images[].[ImageId,Name,CreationDate]" --output table
```

---

## License

No license file is currently included in this repository. Add one if you intend others to reuse this template.
