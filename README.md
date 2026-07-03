# Governance Pipeline

**The reference blueprint for how an API governance pipeline should actually be
built — as forkable GitHub Actions, a one-step composite action, and an owned,
grounded starter ruleset.**

This repo exists because of a specific finding. In
[*The State of Spectral in API Pipelines*](https://apievangelist.com), API
Evangelist read **1,005 real public GitHub pipelines** running
[Spectral](https://github.com/stoplightio/spectral) in CI and scored each one
against an eight-point maturity rubric. The ceiling was six. Two repositories
reached it. **Nobody assembled the whole blueprint** — even though every single
piece of it is already shipping somewhere in the corpus. The parts are lying
around the scrapyard; no one has bolted them together.

So this repo bolts them together. It is a **teaching artifact you fork**: copy a
good fragment instead of the bad one you'd otherwise find first.

Part of the [API Commons tools](https://apicommons.org/tools/), alongside
[Spectral Reporter](https://reporter.apicommons.org) and
[API Validator](https://validator.apicommons.org).

---

## The blueprint, one line each

| Piece | Research finding it fixes | Real exemplar |
| --- | --- | --- |
| Gate on **`pull_request`** (+ push for a baseline) | A third of pipelines lint on push-to-main — *after* the merge they meant to prevent | vtex/openapi-schemas |
| **`paths:` filter** to the spec/ruleset | Only 22% run only when the spec actually changed | mongodb/openapi |
| Spectral **pinned by commit SHA** | Of 215 Action users, 101 float on `@latest`; only 14 pin by commit | mongodb/openapi |
| An **owned, grounded ruleset** | 63% run the tool's defaults — the most common way to use a governance tool is to not govern | teamdigitale/dati-semantic-backend (shared national ruleset) |
| A **separate OWASP security job** | Security rules appear in just 14% of pipelines | geobeyond/fastgeoapi |
| A **human-readable report** artifact | Readable report/summary ~7%, SARIF ~3% | geobeyond/fastgeoapi |
| **Sparse blocking** severity | Teams either never tune severity or set `continue-on-error`; almost none land in the disciplined middle | — |
| **Docs link on every rule** | Cryptic red builds with no "why" teach teams to route around governance | teamdigitale (rules grounded in national guidelines) |

Nobody in 1,005 pipelines had all of these at once. This repo does.

---

## Fork this (quickstart)

```bash
# 1. Grab the repo
git clone https://github.com/api-commons/governance-pipeline
cd governance-pipeline

# 2. Try the ruleset on the example specs (OpenAPI 3.x AND Swagger 2.0)
npx @stoplight/spectral-cli lint -r ruleset/.spectral.yaml examples/openapi.yaml          # clean 3.x
npx @stoplight/spectral-cli lint -r ruleset/.spectral.yaml examples/openapi-fail.yaml     # teaches (3.x)
npx @stoplight/spectral-cli lint -r ruleset/.spectral.yaml examples/swagger-2.0.yaml      # clean 2.0
npx @stoplight/spectral-cli lint -r ruleset/.spectral.yaml examples/swagger-2.0-fail.yaml # teaches (2.0)
```

Then, in **your** repo:

1. Copy `starter/api-governance.yml` to `.github/workflows/api-governance.yml`.
2. Copy the `ruleset/` directory in, and **rewrite the rules against your own
   operations** — replace `acme` with your org, point the `documentationUrl`s at
   your handbook. Do not adopt ours unread; an inherited ruleset is the
   anti-pattern.
3. Edit the `paths:` and `file_glob`/`spec` values to match where your spec
   lives.
4. Commit on a branch and open a PR — watch it gate before the merge.

Prefer one step? Use the **composite action** instead of the starter workflow:

```yaml
- uses: actions/checkout@v4
- uses: api-commons/governance-pipeline@v1   # pin by SHA in production
  with:
    spec: "openapi.yaml"
    ruleset: "ruleset/.spectral.yaml"
    fail-severity: error
    security: "true"
```

---

## Why each decision — the long version

This is the educational core. Each subsection ties a design choice to the
finding it fixes and the real team that proved the piece.

### Why gate on the pull request (and also run on push)

A rule is a completely different experience depending on *where* it fires. In
the editor it feels like help; on the PR it feels like a gate; after the merge
it's a ticket. A third of the surveyed pipelines lint on **push to main** — they
run *after* the merge, reporting a decision instead of informing it, at the most
expensive point on the cost gradient. The starter workflow triggers on
`pull_request` so governance is the **last cheap checkpoint before the spec is
in**, and additionally on `push: [main]` so the main branch always carries a
fresh baseline and report. The PR is the gate; the push is the scoreboard.
*(Exemplar: vtex/openapi-schemas lints the changed files and comments on the PR.)*

### Why the `paths:` filter

Only 22% of pipelines path-filter. Without it, the governance job runs on every
commit — including the thousand that never touched the API — which is noise that
trains people to ignore it. With `paths:` scoped to the spec **and the ruleset
itself**, the job runs exactly when the thing it governs changed. That is both
the *intent* (we lint the spec because the spec moved) and the *efficiency* the
rest of the corpus lacks. *(Exemplar: mongodb/openapi.)*

### Why pin Spectral by commit SHA

This is the one with the satisfying irony. A governance tool floating on
`@latest` can change its behavior between one Tuesday and the next — no commit in
your repo, no line in your changelog. **The thing enforcing your rules is itself
ungoverned.** Of the 215 pipelines using the official Action, 101 float on
`@latest` and only **14 pin to a specific commit**. The starter pins
`stoplightio/spectral-action` by full SHA with the version in a comment, and
pins the CLI and every ruleset dependency by exact version. Pinning is what a
team does when it has decided, on purpose, what runs — and made that a reviewable
fact. *(Exemplar: mongodb/openapi.)*

### Why an owned, grounded ruleset

The headline finding: **63% run Spectral with no ruleset of their own.** The
most common way to use a governance tool is to not govern with it. The default
ruleset is a config file that ships with the software — a hodgepodge of atomic
checks assembled by an open-source project that has never seen your API. Turning
it on against a mature API produces a *wall of red*, most of it stylistic, and
teaches the team in one afternoon that governance is noise to route around.

The ruleset in `ruleset/` is the counter-example: ~11 rules, each written on
purpose, each with a prose `description` (the why) and a `documentationUrl` (the
owned page where the why lives and can be argued with), named by a documented
convention, and severity-tuned. Read
[`ruleset/RULESET.md`](./ruleset/RULESET.md) — it is the record of the thinking,
and it models the paper's point that **the identical YAML is a governance
artifact in one repo and an empty gesture in another; the difference is whether
human work exists behind it.** *(Exemplar: teamdigitale/dati-semantic-backend
pulls a shared national ruleset owned by Italy's digital-government team — one
governed place, versioned, with real provenance.)*

### Why a separate OWASP security job

Security rules show up in only 14% of pipelines. Folding a couple of security
checks into the design lint hides them; a **dedicated job** makes security a
first-class signal — you can tell at a glance whether an API failed on *design*
or on *security*. The security job inherits the
[OWASP API Security ruleset](https://github.com/stoplightio/spectral-owasp-ruleset)
— inheriting a security *standard* with real provenance is good reuse, the
opposite of inheriting a style linter's defaults — pinned by exact version, with
one house rule layered on top. *(Exemplar: geobeyond/fastgeoapi runs a separate
OWASP job and writes a readable summary.)*

### Why sparse blocking severity

The naive reading of "a third lint after merge and an eighth run
`continue-on-error`" is *block more*. That is wrong. A pipeline that fails builds
over hint-level style teaches teams to route around it into **shadow APIs** —
undocumented, ungoverned, invisible — which are strictly worse than the merely
inconsistent APIs the blocking was meant to prevent. The discipline is: **gate
the few things that genuinely cannot ship, and let everything else inform without
blocking.** The example ruleset has exactly **three** `error` rules (an owner, a
machine-addressable operationId, a defined auth scheme); everything else is
`warn` or `info`. The blocking set's credibility comes from being short.

### Why a human-readable report

A linter, by nature, only ever tells people what is *wrong* — every encounter
with governance becomes an encounter with your failures. Readable reports appear
in ~7% of pipelines, SARIF in ~3%. The pipeline runs
[`@api-common/spectral-reporter`](https://reporter.apicommons.org) after the lint
to emit a self-contained HTML **governance report** and uploads it as an
artifact (and optionally pushes SARIF to GitHub's Security tab). Paired with the
**positive-twin** rules in the ruleset — rules that fire on what *already*
complies — the report can say *"82% of operations already carry an
operationId"* instead of *"143 violations."* Same findings, a scoreboard with a
trajectory instead of a scolding. That trend line is how a governance program
survives the budget conversation. *(Exemplar: geobeyond/fastgeoapi.)*

---

## What's in this repo

```
governance-pipeline/
├── starter/
│   └── api-governance.yml        Copy-paste workflow, every decision commented
├── action.yml                    One-step composite action (inputs below)
├── ruleset/
│   ├── .spectral.yaml            OWNED design ruleset (13 rules, 3 blocking; OpenAPI 3.x + Swagger 2.0)
│   ├── .spectral-security.yaml   Security ruleset (extends OWASP API Top 10)
│   └── RULESET.md                Naming convention + who/what/when/where/why + 2.0 support
├── examples/
│   ├── openapi.yaml              A clean, well-governed spec (OpenAPI 3.x)
│   ├── openapi-fail.yaml         A half-complying spec that teaches (OpenAPI 3.x)
│   ├── swagger-2.0.yaml          The same clean spec in Swagger 2.0
│   └── swagger-2.0-fail.yaml     The same teaching spec in Swagger 2.0
├── index.html                    Landing page for pipeline.apicommons.org
├── LICENSE                       Apache-2.0
└── README.md                     You are here
```

### The composite action inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `spec` | `openapi.yaml` | Glob/path to the API definition(s) to lint |
| `ruleset` | `ruleset/.spectral.yaml` | Your owned design ruleset |
| `fail-severity` | `error` | Minimum severity that fails the build — keep at `error`, keep blocking rules few |
| `security` | `true` | Run the dedicated OWASP security pass |
| `security-ruleset` | `ruleset/.spectral-security.yaml` | Security ruleset (extends OWASP) |
| `report` | `true` | Render + upload the HTML governance report |
| `report-title` | `API Governance Report` | Title on the HTML report |
| `spectral-version` | `6.16.1` | Pinned Spectral CLI version |
| `owasp-version` | `2.0.1` | Pinned OWASP ruleset version |
| `reporter-version` | `0.2.0` | Pinned Spectral Reporter version |

---

## The pieces this assembles (all real, all pinned)

- **Spectral Action** — `stoplightio/spectral-action@6416fd0` (v0.8.13), SHA-pinned.
- **Spectral CLI** — `@stoplight/spectral-cli@6.16.1`.
- **OWASP ruleset** — `@api-common/spectral-owasp-ruleset@0.1.0`.
- **HTML report** — `@api-common/spectral-reporter@0.2.0` → [reporter.apicommons.org](https://reporter.apicommons.org).

---

## License

[Apache-2.0](./LICENSE) — free and open. A project of
[API Evangelist](https://apievangelist.com), maintained under
[API Commons](https://apicommons.org). The blueprint is a map; API Evangelist
also does the work alongside you — [governance
services](https://apievangelist.com/services/) when you want an owned ruleset
written and grounded against your operations, and severity and rollout tuned so
governance guides instead of alienates.
