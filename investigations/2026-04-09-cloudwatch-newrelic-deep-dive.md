# CloudWatch And New Relic Cost Deep Dive

- Date: 2026-04-09
- Scope: CloudWatch cost drivers, New Relic metric export path, alarm footprint, and log-retention opportunities
- Related systems: CloudWatch Metrics, CloudWatch Metric Streams, Kinesis Data Firehose, CloudWatch Logs, New Relic, `ap-southeast-2`

## Objective

Understand why CloudWatch costs are elevated, determine how much of that spend is tied to the New Relic integration, and identify the safest high-value cost reduction opportunities without weakening critical observability by accident.

## Sources Used

- Repository files:
  - `AGENTS.md`
  - `docs/investigations/2026-04-09-cost-quick-wins.md`
  - `docs/investigations/2026-04-09-ap-southeast-2-architecture-baseline.md`
- AWS commands or consoles:
  - `aws ce get-cost-and-usage --time-period Start=2026-03-01,End=2026-04-01 --granularity MONTHLY --metrics UnblendedCost --filter '{"Dimensions":{"Key":"SERVICE","Values":["AmazonCloudWatch"]}}' --group-by Type=DIMENSION,Key=USAGE_TYPE`
  - `aws ce get-cost-and-usage --time-period Start=2026-03-01,End=2026-04-01 --granularity MONTHLY --metrics UnblendedCost --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Kinesis Firehose"]}}' --group-by Type=DIMENSION,Key=USAGE_TYPE`
  - `aws ce get-cost-and-usage --time-period Start=2026-04-01,End=2026-04-10 --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE`
  - `aws cloudwatch list-metric-streams --region ap-southeast-2`
  - `aws cloudwatch get-metric-stream --region ap-southeast-2 --name NewRelic-Metric-Stream`
  - `aws cloudwatch describe-alarms --region ap-southeast-2`
  - `aws logs describe-log-groups --region ap-southeast-2`
  - `aws firehose describe-delivery-stream --region ap-southeast-2 --delivery-stream-name NewRelic-Delivery-Stream`
- External references: none

## Findings

- March 2026 CloudWatch spend was `1159.77 USD`.
- April 1-9, 2026 month-to-date CloudWatch spend was `257.56 USD`, which suggests CloudWatch remains a live current cost issue rather than a one-off spike.
- The biggest March 2026 CloudWatch cost drivers were:
  - `APS2-CW:MetricStreamUsage = 579.74 USD`
  - `APS2-CW:MetricMonitorUsage = 255.80 USD`
  - `APS2-DataProcessing-Bytes = 171.64 USD`
  - `APS2-VendedLog-Bytes = 58.11 USD`
  - `APS2-CW:AlarmMonitorUsage = 28.21 USD`
  - `APS2-CW:GMD-Metrics = 27.32 USD`
  - `APS2-TimedStorage-ByteHrs = 18.98 USD`
- The highest CloudWatch cost lever is metric streaming and downstream processing, not alarm counts or basic log storage alone.

## New Relic Export Path

- There is exactly one running metric stream in `ap-southeast-2`:
  - `Name`: `NewRelic-Metric-Stream`
  - `Arn`: `arn:aws:cloudwatch:ap-southeast-2:747293622182:metric-stream/NewRelic-Metric-Stream`
  - `State`: `running`
  - `OutputFormat`: `opentelemetry0.7`
  - `FirehoseArn`: `arn:aws:firehose:ap-southeast-2:747293622182:deliverystream/NewRelic-Delivery-Stream`
- The metric stream was created on `2023-05-09` and has not been updated since that date.
- `IncludeLinkedAccountsMetrics` is `false`, so the stream is not aggregating from linked accounts.
- No include or exclude filters were returned by `get-metric-stream`, so the safest current inference is that the stream scope is broad unless a later command proves metric filtering exists elsewhere.

### Firehose delivery details

- The delivery stream is `NewRelic-Delivery-Stream`.
- Firehose sends data directly to:
  - `https://aws-api.newrelic.com/cloudwatch-metrics/v1`
- Delivery stream type is `DirectPut`.
- Buffering is small and frequent:
  - `1 MB`
  - `60 seconds`
- Failed records are backed up to:
  - `s3://sensor-newrelic-firehose-17c83850`
- CloudWatch logging for the Firehose destination is disabled.
- March 2026 Firehose cost was very small:
  - `APS2-BilledBytes = 1.72 USD`
  - `APS2-DataTransfer-Out-Bytes = 0.92 USD`
- This means Firehose is not the cost problem. The expensive part is the CloudWatch metric stream and metric-processing footprint upstream of Firehose.

## Alarm Footprint

- There are `290` metric alarms and `0` composite alarms in `ap-southeast-2`.
- Alarm namespace distribution is:
  - `AWS/EC2`: `107`
  - `AWS/ApplicationELB`: `62`
  - `AWS/Lambda`: `40`
  - `AWS/RDS`: `36`
  - `CWAgent`: `34`
  - `AWS/ElastiCache`: `8`
  - `AWS/SNS`: `3`
- The alarm set is dominated by environment-repeated EC2, ALB, Lambda, RDS, and CWAgent patterns rather than a few unusually expensive alarm constructs.
- Alarm cost exists, but the much larger `MetricMonitorUsage` line suggests the main opportunity is rationalizing what is monitored and exported, not simply deleting alarms indiscriminately.

### Observed monitoring patterns

- Many alarms are duplicated across `prod`, `qa`, `dev`, `sandbox`, and `uat`.
- Repeated alarm families include:
  - EC2 CPU and status checks
  - ALB response time, unhealthy hosts, and `5XX`-style checks
  - RDS CPU, free memory, free storage, read IOPS, and queue depth
  - CWAgent memory and disk alerts for host-level monitoring
  - Lambda error alarms
- This supports the inference that multi-environment duplication is a meaningful contributor to monitoring volume.

## Log Footprint

- There are `143` CloudWatch log groups in `ap-southeast-2`.
- `70` log groups have no explicit retention policy.
- Total stored bytes reported across log groups are `631,539,878,493`, roughly `588 GiB`.
- The largest retained log groups all currently show `365` days retention:
  - `smoke-vpc-flow-log`: `133,475,776,399` bytes
  - `smoke-api-prod-pm2-out-log`: `133,088,150,884` bytes
  - `aws-waf-logs-prod`: `63,055,604,565` bytes
  - `smoke-qa-api-out`: `59,159,836,020` bytes
  - `vpc-flow-log-development`: `53,262,752,778` bytes
  - `qa-vpc-flow-logs`: `49,789,185,161` bytes
  - `sandbox-vpc-flow-log`: `37,603,435,408` bytes
  - `smoke-dev-api-out`: `29,600,259,744` bytes
  - `smoke-api-sandbox-pm2-out-log`: `19,796,689,712` bytes
  - `vpc-uat-log-group-name`: `18,583,786,282` bytes
- Logs matter, but current March 2026 billing shows log ingestion and storage are still materially smaller than metric stream and metric monitor costs.

## Cost Interpretation

### What is most likely driving cost

- The strongest cost signal is broad metrics export to New Relic combined with a large monitored metric footprint inside CloudWatch.
- `MetricStreamUsage + DataProcessing-Bytes` alone account for `751.38 USD` in March 2026.
- `MetricMonitorUsage` adds another `255.80 USD`, which suggests many monitored series, frequent metric collection, or duplicated monitoring patterns across environments.

### What is not the primary driver

- Firehose is not a major cost center.
- Alarm count is notable but not the first optimization target.
- Log retention is a real secondary opportunity, but not the biggest one based on March 2026 service-cost breakdown.

## Ranked Opportunities

### 1. Narrow New Relic metric export scope

- Expected impact: high
- Likely savings band: `300-700 USD` per month
- Evidence:
  - one active New Relic metric stream
  - `579.74 USD` metric stream cost
  - `171.64 USD` CloudWatch data processing cost
- Safe next validation:
  - inspect the exact metric namespaces and dimensions currently exported
  - confirm which metrics New Relic dashboards or alerts actually require
  - identify whether host-level `CWAgent`, ALB, RDS, and non-prod environment metrics all need to leave AWS

### 2. Rationalize monitored metric volume across non-prod environments

- Expected impact: medium to high
- Likely savings band: `100-300 USD` per month
- Evidence:
  - `255.80 USD` metric monitor cost
  - strong environment duplication in EC2, ALB, Lambda, RDS, and CWAgent alarms
- Safe next validation:
  - group alarms and monitored resources by environment
  - identify which non-prod environments are still always-on and operationally important
  - check whether `qa`, `sandbox`, and `uat` all need full production-like monitoring density

### 3. Reduce oversized log retention on high-volume groups

- Expected impact: medium
- Likely savings band: `25-100 USD` per month initially, potentially more over time
- Evidence:
  - `58.11 USD` vended log bytes
  - `18.98 USD` timed storage
  - multiple 365-day retained high-volume groups
- Safe next validation:
  - classify large groups into compliance-required, operationally useful, and likely over-retained
  - focus first on VPC flow logs, PM2 stdout/error logs, WAF logs, and slow query logs

## Risks Or Constraints

- Reducing exported metrics without dependency mapping could break New Relic dashboards, alerting, or SLO views.
- Reducing monitored metric volume across non-prod may create blind spots during release testing or incident simulation.
- Shortening log retention may conflict with compliance or forensic requirements, especially for WAF and VPC flow logs.
- Some duplicated alarms may be generated by automation or deployment patterns, which means cleanup may need process changes rather than one-time deletion.

## Cost Optimization Opportunities

- The best next move is not “delete alarms” or “turn off logs” broadly.
- The best next move is to inspect what `NewRelic-Metric-Stream` is exporting and compare that against what teams actually use.
- After stream scope, the next highest-value review is whether all non-prod environments need the same density of CloudWatch metrics and alarms.
- Log retention should be reviewed after export scope and metric-monitor density because it is probably a secondary savings lever.

## Recommended Next Steps

- Pull metric stream statistics and namespace coverage for `NewRelic-Metric-Stream`.
- Inspect the `NewRelic-Delivery-Stream` S3 failed-record backup only if there is evidence of delivery failures or malformed payloads.
- Build a namespace-by-environment map of alarms and monitored resources to identify duplication.
- Review retention requirements for the top 10 log groups by stored bytes.
- Identify the owner of the New Relic integration before any recommendation becomes an action plan.

## Open Questions

- Which teams, dashboards, and alerts still depend on the full breadth of the current New Relic metric stream?
- Are `CWAgent` host metrics from all non-prod environments actually used outside of incident response?
- Are the large PM2 application logs intentionally retained for a full year, or is that just inherited default practice?
- Which high-volume log groups are compliance-driven versus convenience-driven?
