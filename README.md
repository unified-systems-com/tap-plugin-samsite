# Samsite plugin

![The samsite authorization boundary rendered as a live graph: AWS serving and compliance resources inside the boundary, with the GitHub deploy pipeline, Sigstore transparency log, and CISA KEV catalog connected around it](docs/screenshots/samsite-boundary-graph.png)

This is a real deployment's authorization boundary, rendered as a living graph —
collected from the running system, not drawn. Every authorization package has a
boundary diagram; most are drawn once and stale by the next sprint. This one is
generated from what is actually deployed. The red boundary encloses the AWS
resources the collector found in the account — the CloudFront/Route 53/S3 serving
path, the Lambda that runs OPA compliance checks and the IAM role it assumes, the
Terraform state locks from bootstrap. Around it sit the systems your security
program actually depends on but rarely draws: the GitHub repository and Actions
workflow that deploys the site, the Sigstore transparency-log entries attesting
every signed compliance artifact, and CISA's Known Exploited Vulnerabilities
catalog — on the same graph as the inventory it applies to.

The edges are typed facts, not arrows: `ROUTES_TRAFFIC`, `ASSUMES_ROLE`,
`ATTESTED_BY`, `FEDERATES_VIA`. "What talks to what, and who vouches for it" becomes
a query you can run, not an interview you have to schedule.

Samsite is TAP's reference assessment target: a real, deployed website whose AWS
infrastructure, GitHub build/deploy pipeline, and signed `/.well-known/` compliance
artifacts are all pulled onto the grid so Rampart can assess it end to end. It is a
cross-deployment of Sam Aydlette's [samaydlette.com](https://github.com/sam-aydlette/samaydlette.com),
whose Terraform is a [FedRAMP 20x](https://www.fedramp.gov/20x/) compliance overlay —
which is what makes it a genuine 20x target rather than a demo fixture.

**Samsite is a projection/pages plugin.** It owns:

- the landing page that projects the deployment as a Cytoscape graph,
- the compliance pages (KSI scoreboard, OSCAL POA&M, IIW inventory, VDR report),
- the per-type viewer pages,
- the compliance collector that fetches and sigstore-verifies the signed
  `/.well-known/` artifacts,
- the collector schedules.

It does **not** own AWS resource models, AWS edges, or the boto3 collector — those
live in [`aws_core`](https://github.com/unified-systems-com/tap-plugin-aws-core). It does not own the
KSI catalog either; that is `fedramp_20x_ksi`.

**Samsite is TAP's testing / demo configuration.** The `samsite` boot profile is the
full-stack exercise: it installs eleven plugins, seeds their pages and schedules, then
fires four collectors that pull a real deployment onto the grid. If you want to see
everything TAP does in one boot — or verify that a change didn't break the end-to-end
path — this is the profile to stand up.

Because the collectors pull from a **live** system, fully exercising this plugin
requires access to a working samsite deployment — a deployed site publishing signed
`/.well-known/` compliance artifacts, plus read credentials for its AWS account and
its signing repo. The reference deployment is already running on Sam's side; nobody
else is expected to stand all of that up. If you do want to,
[the operator guide](docs/doc-samsite-operator-guide.md) is the complete standup
reference — credentials, region scoping, collector ordering, and the failure modes
that have actually bitten.

---

## What it looks like

Two more screens from the same live instance, alongside the boundary graph above.
Everything in them is collected data — nothing is a mockup, and nothing was typed in
by hand. If your team is staring at FedRAMP 20x and asking "what is continuous,
machine-readable assessment actually supposed to look like?", this is one concrete
answer, small enough to read end to end.

### The 20x scoreboard: 47 KSIs, scored from signed emissions

![The FedRAMP 20x KSI scoreboard: 47 Key Security Indicators rolled up by capability family, each row showing passing / in-progress / accepted / gap status with per-control counts](docs/screenshots/samsite-ksi-scoreboard.png)

[FedRAMP 20x](https://www.fedramp.gov/20x/) replaces the narrative SSP with
[Key Security Indicators](https://github.com/FedRAMP/rules) — and the question
every compliance team is quietly asking is what reporting against them looks like in
practice. Like this: 47 KSIs scored from the deployment's own signed, machine-readable
emissions — the SSP and POA&M fetched from the live site that same day, timestamped at
the top of the page. Rolled up by capability family, each row carries one of four
states, kept honestly distinct: **passing** (validated by evidence), **in-progress**,
**accepted** (a risk someone signed their name to, visible rather than buried), and
**gap**. When an assessor asks how you're doing on `KSI-CNA-MAT`, the answer is a row
reading 14/14 controls with a date — not a week of document archaeology.

### Evidence collection as a system, not a fire drill

![The CARES collector dashboard: five collectors with run states, last-run timestamps, and result summaries, plus the cron schedules that keep them current](docs/screenshots/samsite-collectors.png)

The two screens above are only as good as their inputs, so the inputs are first-class
and inspectable. Five collectors, each with a run state, a timestamp, and a one-line
result you can hold it to: the AWS collector refreshes the resource graph daily; the
FedRAMP KSI catalog re-pulls nightly, so an indicator deprecated upstream shows up as
a diff the next morning instead of a surprise at assessment; and the site's own
compliance artifacts — KSI signal, the VDR vulnerability report, OSCAL SSP and POA&M,
the integrity inventory — are fetched over HTTPS and their Sigstore signatures
**verified before a single node lands on the graph**. Every fact on the previous two
screens has the same answer to "where did this come from, and when."

That loop — machine-readable artifacts, cryptographic provenance, scheduled
collection, continuous scoring — is the thing 20x and modern vulnerability management
are both asking for. Seeing it fully live requires a working samsite deployment to
point at (the reference one is already running); for anyone who wants to build their
own, [the operator guide](docs/doc-samsite-operator-guide.md) covers the entire path.

---

## Specs

Behavior is specified in `specs/`:

- `spec-samsite-compliance-collector-v0.md` — the collector, its manifest, and verification
- `spec-samsite-compliance-pages-v0.md` — the compliance page set
- `spec-samsite-ksi-scoreboard-v0.md` — the KSI scoreboard panel
- `spec-samsite-vdr-ingestion-health-v0.md` — VDR ingestion health
- `spec-samsite-viewer-pages-v0.md` — the per-type viewer pages
