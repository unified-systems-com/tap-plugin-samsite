# Samsite plugin

Samsite is TAP's reference assessment target: a real, deployed website whose AWS
infrastructure, GitHub build/deploy pipeline, and signed `/.well-known/` compliance
artifacts are all pulled onto the grid so Rampart can assess it end to end. It is a
cross-deployment of Sam Aydlette's [samaydlette.com](https://github.com/sam-aydlette/samaydlette.com),
whose Terraform is a FedRAMP 20x compliance overlay — which is what makes it a
genuine 20x target rather than a demo fixture.

**Samsite is a projection/pages plugin.** It owns:

- the landing page that projects the deployment as a Cytoscape graph,
- the compliance pages (KSI scoreboard, OSCAL POA&M, IIW inventory, VDR report),
- the per-type viewer pages,
- the compliance collector that fetches and sigstore-verifies the signed
  `/.well-known/` artifacts,
- the collector schedules.

It does **not** own AWS resource models, AWS edges, or the boto3 collector — those
live in [`aws_core`](https://github.com/notgeorge/tap-plugin-aws-core). It does not own the
KSI catalog either; that is `fedramp_20x_ksi`.

---

## Running samsite against your own deployment

Samsite pulls from a **live** system. There is no bundled snapshot, so the plugin is
only useful pointed at a deployment you control. This section is the preparation
checklist.

### What you need before you start

1. **Your own samsite deployment.** Fork the upstream site repo, deploy it into your
   own AWS account behind your own domain, and let its CI publish the signed
   `/.well-known/` artifacts. The compliance collector fetches five of them —
   `ksi-signal.json`, `vdr-report.json`, `oscal-ssp.json`, `oscal-poam.json`, `iiw.csv`
   — each alongside a `.bundle` sigstore signature. The full list, with what each one
   decomposes into, is declarative in
   `tap_plugin/samsite/collectors/compliance_collector/artifact_manifest.json`.
2. **An AWS principal that can read that account.** Read-only. See below.
3. **A TAP instance** with the `samsite` boot profile's plugin set installed.

### Step 1 — Place the AWS credential

The boto3 collector never reads credential files directly; credentials resolve through
the `tap_cares` secrets subsystem. Drop one envelope under your `TAP_SECRETS_ROOT`:

```
$TAP_SECRETS_ROOT/aws_core/boto_collector.secret.json
```

The path is not free-form. The collector resolves a fixed reference — scope `aws_core`,
key `boto_collector` — and **the filename must match the envelope's `key`**. Scope names
the plugin that *consumes* the credential, not the provider that issued it.

For a deployment in **your own account**, use the `aws_static_access_key` kind:

```json
{
  "scope": "aws_core",
  "key": "boto_collector",
  "kind": "aws_static_access_key",
  "description": "Read-only AWS credentials for the boto3 collector against my samsite deployment.",
  "data": {
    "access_key_id": "<your-access-key-id>",
    "secret_access_key": "<your-secret-access-key>",
    "regions_allowed": ["us-east-1", "us-east-2"]
  }
}
```

`session_token` is accepted if you are using temporary credentials. The `data` block is
strict — `additionalProperties` is false, so anything not in the schema is rejected at
resolution rather than ignored.

If instead you are collecting from an account you do **not** own, use the
`aws_assumed_role` kind and follow `aws_core`'s cross-account handoff kit at
`tap_plugin/aws_core/collectors/boto3_collector/handoff/` — it ships a CloudFormation
template, a Terraform equivalent, and the operator-side principal policy. That path
requires an External ID and never moves long-lived keys across the account boundary.

**Least privilege.** The collector reads ACM, CloudFront, DynamoDB, EventBridge, IAM,
Lambda, CloudWatch Logs, Route 53, S3 and STS. AWS's managed `SecurityAudit` policy
covers this — read-only configuration metadata with no data-plane object reads — and is
what the cross-account role template attaches. Do not give the collector write
permissions; it never writes.

### Step 2 — Scope the regions

Region scope is operator-owned and lives on the secret, for both kinds:

- `regions_allowed` — a non-empty list; regional collection is confined to exactly these.
- `region` — a single region, used when `regions_allowed` is absent.
- **Neither: the run fails visibly.** This is deliberate; there is no implicit default.

The reference deployment spans `us-east-1` and `us-east-2` (CloudFront and ACM
certificates are us-east-1 by nature). Use whatever your own deployment actually spans —
every extra region is collection time you pay on every run.

### Step 3 — Point the compliance collector at your site

`artifact_manifest.json` carries three things that are specific to a deployment:

| Field | Reference value | What yours should be |
| --- | --- | --- |
| `site_base_url` | `https://samsite.unified-systems.com` | your deployed site's origin |
| `verification.github_repository` | `notgeorge/samsite` | the repo whose Actions workflow signs your artifacts |
| `verification.oidc_issuer` | `https://token.actions.githubusercontent.com` | unchanged, if you sign via GitHub Actions |

> **Known limitation.** The manifest is loaded from a path relative to the installed
> module and there is no override — no env var, no boot-profile key, no collector
> config. To point samsite at your own site today you need an **editable** checkout of
> this plugin rather than a pinned git install:
> `scripts/spawn-session.sh <name> samsite --dev-plugins samsite`. Making this
> per-install configuration is tracked as the next change to this plugin.

The signing identity is *not* hardcoded: `sigstore_link.py` parses whatever SAN URI the
verified certificate carries and resolves it against the grid. Change the manifest's
`github_repository` and verification follows your repo.

### Step 4 — Boot, then fire the collectors in order

The `samsite` boot profile seeds the pages and then fires four collectors. **The order
is load-bearing**, and getting it wrong degrades silently rather than loudly:

1. `aws_core:boto3` — the AWS resource graph. Must run **first**: it lands the
   `aws_account` nodes that compliance-boundary membership is synthesized from.
2. `github_core:github_core` — repository and Actions data. Must run **before** the
   compliance collector: `sigstore_link` resolves the signing workflow by content-match,
   and if the workflow node is not on the grid yet those `SIGNED_BY_IDENTITY` edges are
   **silently omitted**. You get a graph that looks fine and is missing its provenance
   edges.
3. `fedramp_20x_ksi:ksi-catalog` — the KSI catalog over public HTTPS. No credentials.
4. `samsite:samsite-compliance` — fetches the five artifacts and verifies their sigstore
   bundles.

If the landing graph renders but signing provenance is missing, re-run in this order
before investigating anything else.

---

## What is still pinned to the reference deployment

Named rather than implied, so you know what you are looking at:

- **`artifact_manifest.json`** — site URL and signing repo (Step 3 above). The one
  functional blocker for pointing samsite elsewhere.
- **`samsite-keystone.grift.json`** — the instance keystone describes the reference
  deployment: its site URLs, its owner, and its upstream. Cosmetic, but it is what an AI
  assistant reads to learn what the instance *is*, so it will describe the reference
  system until you edit it.
- **`landing.grift.json`** — two `description` fields mention the reference AWS account
  id. Prose inside seed data; no functional effect.

The account id is **not** hardcoded anywhere functional. The collector resolves it at
runtime via STS `get_caller_identity`, so it lands whatever account your credentials
actually reach.

One asymmetry worth knowing: the `aws_assumed_role` kind accepts an optional
`expected_account_id`, which fails the run closed if the resolved account is not the one
you declared. The `aws_static_access_key` kind has no such field — on the static path
there is no assert-on-land guard, so double-check which account your keys belong to.

## Handling the credential safely

- **Never commit the envelope.** Secret material does not belong in any repository, and
  publication is one-way — a credential in git history is disclosed for the life of the
  repo even after deletion. Keep it under `TAP_SECRETS_ROOT`, outside the tree.
- **The scanner does not cover all of it.** TAP's credential-shape scanner detects AWS
  *access key ids* (`AKIA`/`ASIA` prefixes). It does **not** detect the *secret access
  key* — that has no distinguishing prefix, and entropy heuristics were measured on this
  codebase at 21 findings and 21 false positives, so they are deliberately off. The
  secret access key rests on the envelope layer and `.gitignore`, not on detection.
- **Rotation is restart-to-rotate.** There is no atomic reload; changing a value
  requires restarting the instance.
- **If your `TAP_SECRETS_ROOT` is a shared directory** (it is, in the multi-session dev
  layout), editing an envelope mutates every live session at once. Point
  `TAP_SECRETS_ROOT` at a private directory for experiments.

---

## Specs

Behavior is specified in `specs/`:

- `spec-samsite-compliance-collector-v0.md` — the collector, its manifest, and verification
- `spec-samsite-compliance-pages-v0.md` — the compliance page set
- `spec-samsite-ksi-scoreboard-v0.md` — the KSI scoreboard panel
- `spec-samsite-vdr-ingestion-health-v0.md` — VDR ingestion health
- `spec-samsite-viewer-pages-v0.md` — the per-type viewer pages
