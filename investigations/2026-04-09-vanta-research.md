# Vanta Research

- Date: 2026-04-09
- Scope: identify what Vanta is, how it is connected to this AWS account, and whether it appears tied to smoke alarm monitoring operations in Australia
- Related systems: IAM, AWS Cost Explorer, AWS Config, Security Hub, Inspector, third-party compliance tooling

## Objective

Determine whether the `Vanta` billing line reflects an infrastructure component used by the smoke alarm monitoring workload, or a separate compliance and security vendor integration applied to this AWS account.

## Sources Used

- Repository files:
  - `docs/investigations/2026-04-09-cost-quick-wins.md`
  - `docs/investigations/2026-04-09-ap-southeast-2-architecture-baseline.md`
- AWS commands or consoles:
  - `aws ce get-cost-and-usage` for March 2026 grouped by `SERVICE`
  - `aws ce get-cost-and-usage` for March 2026 grouped by `USAGE_TYPE` for `Vanta`
  - `aws iam list-roles`
  - `aws iam list-policies --scope Local`
  - `aws iam get-role --role-name vanta-auditor`
  - `aws iam list-attached-role-policies --role-name vanta-auditor`
  - `aws iam list-role-policies --role-name vanta-auditor`
  - `aws iam get-policy --policy-arn arn:aws:iam::747293622182:policy/VantaAdditionalPermissions`
  - `aws iam get-policy-version --policy-arn arn:aws:iam::747293622182:policy/VantaAdditionalPermissions --version-id v1`
  - `aws configservice describe-configuration-recorders`
  - `aws configservice describe-configuration-recorder-status`
  - `aws securityhub describe-hub`
  - `aws inspector2 batch-get-account-status --account-ids 747293622182`
  - `aws organizations describe-organization`
  - `aws ce get-cost-and-usage --time-period Start=2025-10-01,End=2026-04-10 --granularity MONTHLY --metrics UnblendedCost --filter '{"Dimensions":{"Key":"SERVICE","Values":["Vanta"]}}' --group-by Type=DIMENSION,Key=USAGE_TYPE`
- External references:
  - Vanta AWS overview page
  - Vanta Help Center documentation for AWS integrations
  - Vanta Terms of Service
  - Vanta Data Processing Addendum
  - OAIC guidance on APP 8 cross-border disclosures
  - Drata product overview
  - Secureframe product overview
  - Thoropass product overview

## Findings

- `Vanta` is not part of the smoke alarm monitoring application stack itself.
- In this AWS account, `Vanta` appears to be a third-party compliance and security tooling integration that continuously audits the account.
- The March 2026 billing line for `Vanta` was `6250 USD`, with usage type `Global-SoftwareUsage-Contracts`, which looks like a contract-style vendor charge rather than metered AWS infrastructure usage.
- Within the available billing window checked (`2025-10-01` through `2026-04-10`), the Vanta charge appeared only in the period `2026-03-01` to `2026-04-01`.
- April 1-9, 2026 month-to-date showed `0 USD` for Vanta.
- This pattern suggests a non-monthly posting, likely an annual, prepaid, or otherwise contracted billing event. This is an inference from billing shape, not a confirmed contract term.
- This account has an active Vanta IAM integration:
  - role: `arn:aws:iam::747293622182:role/vanta-auditor`
  - created: `2023-03-08T01:55:11Z`
  - last used: `2026-04-08T22:02:49Z`
  - last used region: `ap-southeast-2`
- The Vanta role trust policy allows assumption by AWS account `956993596390` with a required `sts:ExternalId`, which is a standard third-party auditor integration pattern.
- The role has two attached managed policies:
  - AWS managed `SecurityAudit`
  - local `VantaAdditionalPermissions`
- The role has no inline policies.
- The local `VantaAdditionalPermissions` policy is minimal and currently contains only a small deny list for:
  - `datapipeline:EvaluateExpression`
  - `datapipeline:QueryObjects`
  - `rds:DownloadDBLogFilePortion`
- The minimal custom policy aligns with Vanta’s documented AWS integration pattern, where `SecurityAudit` is required and `VantaAdditionalPermissions` varies depending on enabled products.
- This AWS account is not the AWS Organizations management account:
  - organization ID: `o-l2ps3sa79r`
  - management account ID: `872515272604`
  - management account email: `AwsresellerenduserAU238@ingrammicro.com`
- That means the current `vanta-auditor` role in account `747293622182` is not, by itself, proof that Vanta is monitoring the whole AWS Organization through this account. It is most consistent with an account-level integration for this specific member account.

## What Vanta Is Likely Doing Here

- Based on AWS evidence and Vanta’s documentation, Vanta is most likely being used for:
  - continuous compliance evidence collection
  - security posture monitoring
  - audit-readiness checks across AWS resources
- This is consistent with the account also having these services enabled:
  - AWS Config recording all supported resource types continuously
  - Security Hub enabled since `2023-09-13`
  - Inspector enabled for EC2, ECR, Lambda, and Lambda code
- These AWS-native security services may support Vanta workflows or the broader compliance program, but this note does not prove Vanta alone enabled them.

## Relationship To The Smoke Alarm Monitoring Account

- The AWS account is clearly used for the smoke alarm monitoring platform in Australia, but the Vanta evidence indicates a governance/compliance overlay on that account, not a workload runtime dependency.
- In practical terms:
  - the smoke alarm system can be hosted in this account
  - Vanta can separately observe the account for compliance and audit purposes
- Nothing found so far suggests Vanta is involved in alarm delivery, device telemetry, application traffic, or production workload execution.

## Billing Cadence Interpretation

- The observable charge pattern does not look like ordinary monthly seat or usage billing.
- Because the only visible posting in the available billing window is the single `6250 USD` March 2026 event, the most likely explanations are:
  - annual or periodic contracted billing
  - AWS Marketplace private offer or contract posting
  - reseller-mediated or enterprise contract settlement that lands in AWS billing on a non-monthly cadence
- I could not confirm the exact contract mechanics from AWS alone.
- The failed wider `2025-01-01` to `2026-04-10` Cost Explorer query indicates AWS historical billing is not enabled here beyond 14 months, so older cadence could not be validated from this session.

## Alternatives

- The closest replacement category is another continuous compliance / trust management platform rather than an infrastructure tool.
- Credible alternatives, based on official vendor positioning, include:
  - `Drata`: positioned as an "Agentic Trust Management Platform" with continuous compliance, integrated third-party risk, trust center, and enterprise GRC features
  - `Secureframe`: positioned around automated security and compliance with support for major frameworks including SOC 2, ISO 27001, HIPAA, PCI DSS, GDPR, and NIST
  - `Thoropass`: positioned as an end-to-end audit and compliance platform with both automation and audit delivery under one vendor
- Practical alternative strategies:
  - keep Vanta but reduce scope to only in-scope production systems or only required modules
  - replace Vanta with another automation vendor if the organization wants a stronger GRC, vendor-risk, or auditor-services mix
  - replace software-led automation with a lighter-weight manual plus auditor-led process if compliance needs are limited and engineering overhead is acceptable

## Alternative Fit Notes

- `Drata` is likely the closest alternative if the organization wants a broad trust-management platform with ongoing compliance automation and integrated risk workflows.
- `Secureframe` is likely strongest if the priority is compliance automation breadth with strong framework coverage and a more modular automation-first approach.
- `Thoropass` is likely strongest if the organization wants tighter coupling between compliance tooling and external audit execution.
- A manual or lighter tooling approach is only realistic if:
  - required frameworks are limited
  - customer security review volume is low
  - the business can tolerate more manual evidence collection and audit preparation

## Legal And Commercial Implications

- This is not legal advice. These are risk areas to review with legal, procurement, security, and the business owner.
- Contract term risk:
  - Vanta’s public terms say subscriptions renew as specified in the Order Form, and if the Order Form does not specify otherwise, the default term is one year with automatic one-year renewals unless either party gives at least 30 days written notice before the current term ends.
  - If this account’s `6250 USD` posting reflects an annual or contract event, notice windows may matter immediately.
- Audit continuity risk:
  - Vanta’s AWS Organization migration guidance says deleting existing AWS credentials in Vanta erases assigned owners and descriptions, and Vanta advises against doing this in the middle of an audit.
  - Replacing Vanta without a transition period could disrupt evidence continuity, auditor access, or ongoing control mappings.
- Privacy and data-processing risk:
  - Vanta’s DPA says Vanta processes customer information to provide the service and, following completion of services, returns or deletes personal data at the customer’s choice unless law requires retention.
  - Vanta’s DPA explicitly addresses GDPR, UK GDPR, Swiss data protection, and US state privacy laws, but public text reviewed here does not specifically call out Australian privacy law.
  - For an Australian smoke alarm business, any personal information disclosed to an overseas recipient may trigger APP 8 cross-border disclosure obligations. OAIC guidance says an APP entity must take reasonable steps to ensure the overseas recipient does not breach the APPs and can remain accountable for the recipient’s mishandling in some cases.
- Data residency and scope risk:
  - If Vanta ingests personnel, device, security, or vendor-risk data beyond what is strictly needed, replacing or downsizing it may require export, retention, deletion, and subprocessor review planning.
- Customer and regulatory commitment risk:
  - If sales contracts, audits, or customer questionnaires rely on Vanta-based evidence, trust center content, or continuous monitoring, removal may create downstream contractual or assurance issues even if the engineering change is simple.

## Migration Considerations

- Before switching vendors, confirm:
  - whether there is an active audit or upcoming renewal date
  - what evidence, mappings, and reviewer workflows live only in Vanta
  - whether a replacement platform can import or recreate those artifacts cleanly
  - whether customer commitments or compliance frameworks rely on continuous history rather than point-in-time snapshots
- A safer migration sequence would usually be:
  - confirm contract notice dates
  - export evidence and reporting artifacts
  - stand up replacement tooling in parallel
  - validate control coverage and auditor acceptance
  - only then retire the Vanta integration

## Risks Or Constraints

- The `Vanta` cost appears large, but it is probably governed by vendor contract terms rather than resource tuning inside AWS.
- Some AWS-side security service costs such as Config, Security Hub, and Inspector may be adjacent to the compliance program, but their necessity should be evaluated carefully before attributing them to Vanta.
- The presence of a recently used `vanta-auditor` role indicates the integration is still active and likely relied upon by someone in the organization.

## Cost Optimization Opportunities

- Treat the `6250 USD` Vanta line item as a vendor/procurement review, not an infrastructure optimization.
- Validate who owns the Vanta contract and which compliance frameworks or audits depend on it.
- Review whether the enabled Vanta product set matches actual needs, because Vanta’s own documentation says additional enabled products can expand permissions and may incur additional AWS-side costs.
- Separately review AWS Config, Security Hub, and Inspector usage to confirm they remain required and appropriately scoped, but do not assume they can be reduced without compliance impact.

## Recommended Next Steps

- Identify the business owner of the Vanta contract and the compliance frameworks it supports.
- Confirm whether Vanta is billed through AWS Marketplace or another AWS contract channel tied to `Global-SoftwareUsage-Contracts`.
- Check the Vanta Order Form or reseller paperwork immediately for renewal dates and notice requirements.
- Review Vanta’s enabled AWS integration products in the Vanta admin interface to see whether optional features beyond the base AWS integration are turned on.
- Decide whether the right comparison set is:
  - Vanta right-sizing
  - Vanta replacement with another automation vendor
  - tooling reduction plus more manual compliance operations
- Decide whether this account truly needs continuous Vanta monitoring, or whether the scope can be reduced to only in-scope production systems.

## Open Questions

- Who in the organization owns the Vanta relationship and renewal decision?
- Which audits, certifications, or customer requirements depend on this Vanta integration?
- Is the `6250 USD` charge monthly, annual, or tied to a private offer billing schedule?
- Are Config, Security Hub, and Inspector being operated because of Vanta requirements, broader security policy, or both?
- Is Vanta used only for this AWS member account, or is there a separate organization-level integration in the management account or another security account?
