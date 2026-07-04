# 14 — Deploy the MQTT forensic tap (IRSA path)

Stand up `mqtt-forensic-tap` (Go) on the `safer-ops-prod` EKS cluster, in the
`saferhomes-workers` namespace, capturing ground-truth MQTT frames into the Mongo
time-series collection `tbl_device_events`. Design:
`docs/investigations/2026-06-30-device-events-journal-design.md`.

**Preconditions**
- `tbl_device_events` time-series collection exists (done 2026-06-30; see applied-changes).
- `AWS_PROFILE=sensorsyn-mfa`; `kubectl` context `safer-ops-prod`; Docker + Go available on
  the build host.
- Reachability already satisfied: pod→EMQX `1883` (VPC CIDR rule) and pod→Atlas (shared NAT
  EIP) need no SG/allowlist change.

---

## Step 1 — Apply the IAM + ECR Terraform (ELEVATED — named-resource ack)

Creates `aws_ecr_repository.mqtt_forensic_tap` and `aws_iam_role.saferhomes_forensic_tap`
(+ policy/attachment/lifecycle). New resources only; nothing existing is modified.

```sh
cd code/infra/terraform/environments/prod
git mv saferhomes_forensic_tap.tf.proposed saferhomes_forensic_tap.tf
terraform fmt && terraform init && terraform plan   # review: 5 to add, 0 to change, 0 to destroy
# apply is run externally by the operator after the named ack
terraform apply
terraform output saferhomes_forensic_tap_role_arn
terraform output -raw mqtt_forensic_tap_ecr_url
```

Ack to apply: **"go — SaferHomesForensicTapRole + mqtt-forensic-tap ECR"**. Record in
`docs/applied-changes.md`.

## Step 2 — Build & push the image

```sh
cd code/mqtt-forensic-tap
go mod tidy            # first time: generate go.sum
ECR=$(terraform -chdir=../infra/terraform/environments/prod output -raw mqtt_forensic_tap_ecr_url)
aws ecr get-login-password --region ap-southeast-2 \
  | docker login --username AWS --password-stdin "${ECR%/*}"
docker build -t "$ECR:0.1.0" .
docker push "$ECR:0.1.0"
```

## Step 3 — Fill the manifest placeholders

From the Step-1 outputs, replace `<ACCOUNT_ID>` in `deploy/k8s/serviceaccount.yaml` and
`deploy/k8s/secretstore.yaml` (role ARN), and set the StatefulSet image to `$ECR:0.1.0`
(`deploy/k8s/statefulset.yaml`, or a `kustomization.yaml` `images:` entry).

## Step 4 — Deploy

First deploy direct (fastest to validate), then move to GitOps:

```sh
kubectl apply -k deploy/k8s
```

GitOps alternative: commit the manifests and add a Flux `Kustomization` CR under
`clusters/safer-ops-prod/` pointing at this path.

## Step 5 — Verify

```sh
kubectl -n saferhomes-workers get externalsecret,secret,statefulset,pod
kubectl -n saferhomes-workers describe externalsecret mqtt-forensic-tap-env   # want: SecretSynced
kubectl -n saferhomes-workers logs sts/mqtt-forensic-tap | \
  grep -E 'mongo connected|subscribed|stats'
```

Expect a `subscribed` line granting QoS 2 on `sg/sas/resp/#` + `sg/sas/cmd/#`, then `stats`
lines with `captured`/`flushed` climbing and a non-zero `qos2`. Confirm rows landing
(read-only):

```sh
AWS_PROFILE=sensorsyn-mfa SENSOR_API_INSTANCE_ID=i-... \
  python3 scripts/diag/prod-read.py --collection tbl_device_events --op count --filter '{}'
# spot-check direction + qos split:
... --op aggregate --pipeline '[{"$group":{"_id":{"direction":"$direction","qos":"$qos"},"n":{"$sum":1}}}]'
```

Success = count climbing, both `command` and `report` directions present, `qos:2` rows
present (confirms the QoS-2 subscription is capturing the critical frames).

## Rollback

- Workload: `kubectl delete -k deploy/k8s` (or scale the StatefulSet to 0).
- Infra: `terraform destroy -target=...` the role + repo, or `git mv` the file back to
  `.tf.proposed` and apply.
- The `tbl_device_events` collection is independent — leave it (drop only if abandoning).

## Notes

- Single-replica StatefulSet is intentional: stable pod name → stable MQTT clientId →
  `CleanSession=false` resumes the queued QoS-2 session. HA = raise replicas **and** set
  `TAP_SHARED_GROUP` (our own `$share` group, never `sensorGroup`).
- If `ExternalSecret` sync fails with a token error, the ESO ClusterRole likely lacks
  `create serviceaccounts/token` for our SA — grant it (usually already cluster-wide).
- This is read-only against prod data; the tap only inserts to its own collection.
