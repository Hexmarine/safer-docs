# Vanta Recommendation

- Date: 2026-04-09
- Scope: recommend whether to keep, replace, or renegotiate Vanta for AWS account `747293622182`
- Related systems: Vanta, AWS IAM, AWS Cost Explorer, AWS Config, Security Hub, Inspector, compliance operations, vendor management

## Objective

Turn the Vanta research into an actionable commercial and operational recommendation for this Australian smoke alarm monitoring AWS account.

## Sources Used

- Repository files:
  - `docs/investigations/2026-04-09-vanta-research.md`
  - `docs/investigations/2026-04-09-cost-quick-wins.md`
  - `docs/investigations/2026-04-09-ap-southeast-2-architecture-baseline.md`
- AWS commands or consoles:
  - IAM role and policy inspection for `vanta-auditor`
  - Cost Explorer `Vanta` charge history and usage type review
  - AWS Organizations `describe-organization`
  - AWS Config, Security Hub, and Inspector status checks
- External references:
  - Vanta AWS overview
  - Vanta Terms / MSA
  - Vanta Data Processing Addendum
  - OAIC APP 8 guidance
  - Drata product overview
  - Secureframe Comply overview
  - Thoropass overview

## Recommendation

- Recommended path: `renegotiate and right-size Vanta now; do not replace it immediately`
- Secondary path: `evaluate replacement vendors only if renewal is near, scope is clearly oversized, or Vanta ownership/use is weak`
- Not recommended today: `rip-and-replace Vanta immediately`

## Why This Is The Best Fit

- Vanta is active and in use:
  - the `vanta-auditor` role was used on `2026-04-08`
  - the account has supporting security/compliance services enabled, including Config, Security Hub, and Inspector
- The `6250 USD` charge looks contractual rather than operational:
  - usage type is `Global-SoftwareUsage-Contracts`
  - the charge appeared as a single posting in March 2026 within the checked billing window
- This AWS account is a member account, not the Organizations management account:
  - that makes it plausible that Vanta scope could be reduced or reviewed per account, but it does not prove easy removal
- The account supports a production-like Australian smoke alarm workload:
  - compliance and audit continuity are more likely to matter than in a low-risk dev-only account
- There is not yet enough evidence that:
  - Vanta is unused
  - its scope is obviously excessive
  - another vendor would reduce total cost and migration risk enough to justify immediate switch

## Decision Table

| Option | Recommendation | Why |
| --- | --- | --- |
| Keep as-is | No | too expensive to leave unchallenged without checking scope, owner, and renewal terms |
| Renegotiate / right-size | Yes | lowest operational risk and most likely to produce savings without breaking audit/compliance continuity |
| Replace with another platform | Conditional | worth evaluating only if the contract is near renewal, features are underused, or better pricing/scope fit is available |
| Remove and go manual | No | high audit and operational burden unless compliance needs are minimal, which is not supported by current evidence |

## Alternative Assessment

### Drata

- Best fit if the organization wants:
  - broad trust management
  - integrated third-party risk
  - customer assurance workflows
  - ongoing continuous compliance
- Likely the strongest direct substitute for Vanta if the business wants similar platform breadth.

### Secureframe

- Best fit if the organization wants:
  - strong compliance automation
  - broad framework support
  - a modular, automation-first product
  - a platform marketed around reducing ongoing operational compliance costs
- Likely a good fit if the business values automation depth more than a broader trust-management layer.

### Thoropass

- Best fit if the organization wants:
  - compliance automation plus audit delivery under one vendor
  - more auditor-services coupling
  - less internal coordination between tooling and external auditors
- More compelling if the pain is audit execution rather than tool functionality alone.

## Public Pricing Comparison

- Exact dollar-for-dollar comparison is not available from official public sources for most of these vendors.
- The only concrete price observed in this investigation is the AWS-billed `Vanta` posting of `6250 USD` in March 2026.
- That `6250 USD` amount should not automatically be treated as monthly recurring price:
  - it appeared only once in the checked billing window
  - it used the contract-style usage type `Global-SoftwareUsage-Contracts`
  - this is most consistent with annual, prepaid, installment, or private-offer style billing

| Vendor | Public pricing transparency | Publicly visible pricing signal | What that means for comparison |
| --- | --- | --- | --- |
| Vanta | medium | official pricing page shows plan tiers `Essentials`, `Plus`, `Professional`, `Enterprise` but no public dollar amounts | feature packaging is visible, but exact price requires quote or contract review |
| Drata | medium | official plans page shows `Foundation`, `Advanced`, `Enterprise`; `Foundation` includes up to `50 FTEs` and `1` pre-mapped framework, but no public dollar amounts | packaging is clearer than many peers, but price still requires sales scoping |
| Secureframe | medium | official pricing page shows `Fundamentals`, `Complete`, `Defense`; both commercial packages show `1` compliance framework and quote-only pricing | useful for scope comparison, not for direct price benchmarking |
| Thoropass | low to medium | official site says pricing is quote-based and depends on audit scope and framework complexity; claims `25-50%` savings vs traditional audit firms and quote turnaround in `24 hours` | strongest if comparing bundled audit-plus-platform cost, weakest for clean software-only comparison |

## Practical Pricing Interpretation

- If the observed `6250 USD` Vanta charge is annual for this account’s current scope, it is not obviously out of family with compliance-platform spend based on how these vendors package products and gate pricing behind quotes.
- If the same `6250 USD` were monthly recurring, it would be much harder to justify without a very broad scope or enterprise GRC usage.
- The biggest problem is not that Vanta is uniquely opaque; it is that the whole comparison set is mostly quote-based.
- Because of that, the most reliable commercial comparison is not list price but an apples-to-apples RFP or renewal quote using the same scope:
  - number of AWS accounts
  - number of employees or FTEs
  - number of frameworks
  - need for trust center
  - need for questionnaires, vendor risk, and access reviews
  - whether audit services are bundled

## Price Comparison Bottom Line

- Public evidence does not prove Vanta is overpriced versus Drata, Secureframe, or Thoropass.
- Public evidence does prove:
  - Vanta is quote-based despite showing clear plan tiers
  - Drata and Secureframe are also quote-based
  - Thoropass compares differently because it bundles audit services more aggressively
- Commercially, the strongest next step is:
  - get the current Vanta contract scope
  - request renewal or downsell pricing
  - request matched quotes from Drata and Secureframe
  - request a bundled quote from Thoropass only if you want to compare software-plus-audit together

## Legal And Commercial Implications

- This is not legal advice. It is a practical risk summary for legal, procurement, security, and platform stakeholders.
- Renewal risk:
  - Vanta’s public terms say subscriptions renew per the Order Form; if the Order Form is silent, the default is one year with automatic one-year renewals unless notice is given at least 30 days before term end.
  - Immediate next step should be to locate the Order Form or reseller paperwork and confirm renewal date, notice window, and whether the March 2026 posting reflects annual billing.
- Privacy and cross-border risk:
  - Vanta’s DPA says Vanta acts as a processor for customer data used in the service and may transfer personal data outside the UK, EEA, or Switzerland, including to the United States, with contractual safeguards.
  - For an Australian operator, OAIC APP 8 means overseas disclosures of personal information can leave the Australian entity accountable for downstream mishandling in some circumstances.
  - If Vanta contains employee, device, security, or vendor-risk data tied to Australian operations, legal/privacy review should confirm the organization is comfortable with the disclosure pathway and vendor controls.
- Audit continuity risk:
  - Replacing Vanta can disrupt control mappings, evidence continuity, and auditor workflows even if the AWS integration itself is technically simple to remove.
  - If there is an active or upcoming audit, immediate replacement is especially risky.
- Data exit risk:
  - Vanta’s DPA says data will be returned or deleted at the customer’s choice following completion of services unless retention is legally required.
  - Before any migration or termination, the organization should confirm what evidence, exports, mappings, and historical records must be preserved.

## Practical Recommendation By Time Horizon

### Next 2 weeks

- Identify the internal owner of Vanta.
- Find the Order Form, AWS Marketplace paperwork, or reseller agreement.
- Confirm:
  - service term
  - renewal date
  - notice deadline
  - whether `6250 USD` is annual, installment, or private-offer billing
- Ask Vanta or the reseller for a scope breakdown:
  - modules purchased
  - number of monitored assets/accounts
  - any add-on products

### Next 30 days

- Decide whether the account is actually in scope for all currently enabled Vanta monitoring.
- Confirm whether the AWS member account should remain monitored individually, or whether scope can be reduced to only in-scope production systems.
- Review whether Config, Security Hub, and Inspector are compliance-required, security-policy-required, or just historically enabled.

### 60 to 90 days before renewal

- If the contract looks oversized:
  - pursue renegotiation first
  - ask for reduced scope, lower tiering, or a different packaging model
- Only run a vendor bake-off if:
  - renewal is near
  - Vanta feature usage is weak
  - better-fit pricing appears plausible
  - there is no audit timing conflict

## Recommendation Summary

- Best current business choice: `keep Vanta in place short term, but actively renegotiate or right-size it`
- Best replacement trigger: `clear evidence that the organization is overpaying for unused scope and has time to migrate safely before renewal`
- Best immediate action: `contract and ownership review`, not technical removal

## Open Questions

- Who owns Vanta internally: security, compliance, finance, or an external consultant?
- Is the March 2026 `6250 USD` posting annual billing, a private offer drawdown, or some other contract event?
- What frameworks or customer requirements actually depend on Vanta today?
- Is there an active audit window that would make vendor change risky in the near term?
