# 2026-07-01 — createfilezip* Lambda decommission

## Trigger

Reviewing AWS end-of-support notices (2026-06-30 session): 5 Lambda functions
flagged on `nodejs16.x` (deprecated runtime, well past AWS's EOL cutoff).
Investigated whether to bump the runtime or retire the functions.

## What they are

`createfilezipprod` / `-dev` / `-sandbox` / `-qa` / `-uat` — one per
environment, all `index.handler`, invoked only via a Lambda **Function URL**
(IAM-auth, no API Gateway, no S3/EventBridge trigger). Purpose from source: take
a JSON payload of S3 object keys + new file names, stream them into a zip via
`archiver`, upload the zip back to the same bucket with `ACL: public-read`, and
return a CloudFront link. Looks like a prototype for a "download selected
files/photos as a zip" feature that was never wired into any frontend.

`prod` / `dev` / `sandbox` / `qa` run **identical code** (own bundled
`aws-sdk@2.1692.0` + `archiver@7.0.1`, `engines.node: ">=14"`, no dependency on
the Lambda runtime's built-in SDK — a runtime bump would have been a clean
metadata-only change with no code edits needed).

`uat` is different: it is the **unmodified "Hello from Lambda!" template stub**
(174-byte `index.js`, never implemented) and its environment variables contain
a **live plaintext AWS access key + secret pair** (`AWS_S3_ACCESS_KEY` /
`AWS_S3_SECRET_KEY`, not referenced by any real logic since the handler never
reads them). Values were not re-printed in this doc or in chat beyond the one
accidental echo during initial discovery.

## Evidence: unused, not just old

Only `createfilezipprod` is Terraform-managed
(`code/infra/terraform/environments/prod/generated_lambda.tf`,
`aws_lambda_function.createfilezip_prod`, plus a dedicated security group
`aws_security_group.createfilezipprod` / `sg-029fc2f964f80006f` in
`networking.tf:599` and `locals.sg_createfilezipprod_id` in `locals.tf:34`).
`dev`/`sandbox`/`qa`/`uat` are console-created, present only as CloudWatch-alarm
string references in the sandbox/qa/dev Terraform.

No caller of any of the 5 Function URLs was found anywhere in the checked-out
repos (`sensor-alarm-backend`, `safer-ops`, `sensor-angular`, mobile apps).

Real last-invocation timestamps, read from each function's CloudWatch Logs log
group (`describe-log-streams --order-by LastEventTime`, more reliable than the
default 180-day metrics window):

| Function | Last actual invocation |
|---|---|
| `createfilezipprod` | 2025-01-30 (~17 months idle) |
| `createfilezipdev` | **never** — log group never existed |
| `createfilezipsandbox` | 2025-04-04 (~15 months idle) |
| `createfilezipqa` | **never** — log group never existed |
| `createfilezipuat` | 2024-04-15 (~2+ years idle, template stub) |

CloudTrail (management events only; the account trail
`sensor-global-cloudtrail-events` has no data-event selectors, so `Invoke`
calls were never logged either way — the log-group check above is the ground
truth) shows the **only** entity that has ever touched these functions'
configuration is IAM user `peresada1@gmail.com` (this account), via three
console `UpdateFunctionConfiguration` saves on 2026-04-21 with empty request
diffs (`vpcConfig: {subnetIds:[], securityGroupIds:[]}`, `environment: {}}` —
consistent with opening the console configuration tab and clicking Save with
no actual change, not a redeploy). No external CI/CD, no other AWS principal,
no cross-account access involved.

Each function has its **own dedicated IAM role** (not shared) — deleting is
clean, no other resource depends on these roles. 4 of the 5 roles carry
`AmazonS3FullAccess` (account-wide, not bucket-scoped) attached to an unused
function — an extra reason to remove rather than just leave dormant.

## Decision

**Decommission all 5.** Idle 15-17 months (or never invoked at all), no
caller anywhere in the codebase, no owner other than console clicks, one of
them holding a live leaked credential pair for no functional reason. Keeping
them "just in case" only keeps the leaked-key exposure and the broad
S3-full-access roles alive. Code is archived below/in git history if the
zip-download feature is ever revived properly.

## Archived source (createfilezipprod — same code as dev/sandbox/qa)

`package.json`:
```json
{
  "name": "lambda_zip_qa_test",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "archiver": "^7.0.1",
    "aws-sdk": "^2.1692.0",
    "stream": "^0.0.3"
  }
}
```

`index.js`: takes `{ fileNames: [{s3FileName, newFileName}], folderName,
outputFilePrefix }`, streams each source object from `AWS_S3_BUCKET` into an
`archiver` zip via `s3.getObject().createReadStream()`, uploads the result
back to the same bucket as `<outputFilePrefix>_<random10>_<epochMs>.zip` with
`ACL: public-read`, returns `{ fileLink, missingFiles, errors }`. Full source
preserved in git history of this repo's Lambda deploy artifacts and in the
CloudTrail/console record; not reproduced verbatim here since it adds no
information beyond this description.

`createfilezipuat/index.js` (full file, for completeness — this is the entire
contents, the AWS default template):
```js
exports.handler = async (event) => {
  // TODO implement
  const response = {
    statusCode: 200,
    body: JSON.stringify('Hello from Lambda!'),
  };
  return response;
};
```

## Resources removed (filled in after execution)

- [x] `createfilezipprod` — function + Function URL + role
      `createfilezipprod-role-q9k4qdpz` deleted. Security group
      `sg-029fc2f964f80006f` **pending** — 3 ENIs still `in-use` at write
      time (Lambda VPC-ENI release lag); needs its own named `go` (elevated:
      security-group mutation) once they clear.
- [x] `createfilezipdev` — function, Function URL, role
      `createfilezipdev-role-fcjekd1v`
- [x] `createfilezipsandbox` — function, Function URL, role
      `createfilezipsandbox-role-pd8hyue6`
- [x] `createfilezipqa` — function, Function URL, role
      `createfilezipqa-role-wnjvr790`
- [x] `createfilezipuat` — function, Function URL, role `createfilezipuat`
      (also removed the leaked `AWS_S3_ACCESS_KEY`/`AWS_S3_SECRET_KEY` pair)
- [x] Terraform reconciliation (prod only): removed
      `aws_lambda_function.createfilezip_prod` from `generated_lambda.tf` and
      `aws_security_group.createfilezipprod` from `networking.tf` and
      `sg_createfilezipprod_id` from `locals.tf`; `terraform state rm
      aws_lambda_function.createfilezip_prod` done. **`terraform state rm
      aws_security_group.createfilezipprod` + final clean `terraform plan`
      still pending the SG deletion above.**
- [ ] CloudWatch alarms referencing these function names in
      dev/sandbox/qa Terraform (string references only, will go stale —
      confirm whether to remove or leave as no-op)
- [x] Recorded in `docs/applied-changes.md`

Deletion is irreversible (Function URL domains are randomly generated and
won't be recoverable even by recreating an identically-named function) —
executed only after per-function approval, one `go` each.
