# Upstream reference docs

Vendor-provided and original SensorGlobal documents (mostly pre-JV). The binaries
(`.docx`/`.pdf`/`.doc`) are the **authoritative** copies; they are not grep-able in
the repo, so plain-text conversions live in [`extracted/`](extracted/) for search
and linking. Re-extract from the binary if they ever diverge.

- **Top-level docs:** Technical Brief, System Description (SOC 2, Rev 6), Architecture
  Document, Ubiquitous Language, Information Security Overview, DevOps (MSP/on-call),
  Company Tools.
- **`Technical Documents/`:** firmware spec, event types, domains & hosting, device
  types, feature flags, KORE SIM, infra hardening, git branching, asset naming, dev
  code of conduct, beta test plan, installation process.

## Reviews / extractions

- The 12 `Technical Documents/` PDFs are cross-referenced in
  [`../investigations/2026-04-12-upstream-tech-docs-review.md`](../investigations/2026-04-12-upstream-tech-docs-review.md).
- The 7 top-level docs + Installation Process are reviewed in
  [`../investigations/2026-05-29-upstream-platform-docs-extraction.md`](../investigations/2026-05-29-upstream-platform-docs-extraction.md)
  (with staleness flags and a "needs confirmation" list).
- The firmware/MQTT protocol has a curated Markdown extraction at
  [`../../code/safer-ops/docs/investigations/firmware-protocol-spec.md`](../../code/safer-ops/docs/investigations/firmware-protocol-spec.md)
  — the canonical version; the `.txt` here is the raw dump.

> ⚠️ These are original SensorGlobal docs written before the Safer Homes JV. Named
> people, vendors, and "always/mandatory" claims reflect that period — confirm before
> relying. See the extraction note's **Needs confirmation** section.
