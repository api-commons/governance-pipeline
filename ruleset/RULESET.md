# The ruleset, and why it is owned

This directory holds an **example** governance ruleset. It exists to model an
argument from [*The State of Spectral in API
Pipelines*](https://apievangelist.com): a ruleset is a governance artifact only
if a human wrote it on purpose, can explain every rule, and owns where it lives.
The identical YAML can be governance in one repo and an empty gesture in
another — the difference is entirely whether the work behind it was done.

So this is not `spectral:oas`, and it is not a template someone pasted in. It is
a small, deliberately-written set of rules, and this file is the record of the
thinking behind them. **Replace `acme` with your organization and rewrite these
rules against your own operations — do not adopt ours unread.** A ruleset you
inherited without argument is exactly the anti-pattern the paper is about.

## The naming convention

Every rule id follows one pattern, so a reader can place any finding instantly:

```
<org>-<category>-<subject>-<constraint>
   |       |         |          |
 acme   security  operation  declared
```

- **`<org>`** — the owning organization prefix (`acme`). It signals at a glance
  that this is *our* rule, not a tool default. When a finding says
  `acme-…` everyone knows who to argue with.
- **`<category>`** — the design area the rule governs: `info`, `ops`,
  `naming`, `schema`, `responses`, `security`.
- **`<subject>`** — what the rule is about (`operationid`, `contact`,
  `path`, `property`).
- **`<constraint>`** — the check being made (`required`, `meaningful`,
  `kebab-case`, `present`).

> The canonical API Commons rule-id convention is **Spec / Version / Property /
> Semantics / Severity** (`<spec>-<version>-<property>-<semantics>-<severity>`),
> as defined in the [Spectral Ruleset Studio](https://studio.apicommons.org) and
> the *Governance of APIs* naming-convention chapter. The `<org>`-prefixed scheme
> here is a domain-scoped variant of it — it leads with the owning org so a
> finding announces *whose* rule it is.

Positive twins reuse the same id with a `-present`/positive constraint so it is
obvious they are the mirror of a negative rule.

**Format twins** append `-oas2` / `-oas3` to the id (e.g.
`acme-security-scheme-defined-oas2`). They are the same rule expressed against
the two document structures — see "Swagger 2.0 support" below.

## The grounding (who / what / when / where / why)

A rule that cannot answer these five questions is not grounded, and an
ungrounded rule is one nobody can defend when a developer pushes back — which is
how rulesets quietly die. Every rule here answers them:

- **Who** owns it — the `<org>` prefix and a named team behind the handbook.
- **What** it checks — the `given`/`then` and the human-readable `message`.
- **When/where** it fires — in CI, on the pull request, path-filtered to the
  spec and this ruleset (see `starter/api-governance.yml`). Severity is matched
  to that location: a few `error` rules gate the build; the rest inform.
- **Why** it exists — the `description` on every rule states the reason in
  prose, and the `documentationUrl` points to the owned page where that reason
  lives and can be argued with. In this example those URLs are
  `https://handbook.acme.example/api/rules/<id>` placeholders; in a real repo
  they resolve to your API handbook. **A rule without a `documentationUrl` is a
  cryptic red build; a rule with one is a teachable moment.**

## Severity discipline

The rules are tuned, not uniform. This is the single most important property of
the set and the thing the surveyed corpus got most wrong (teams either ran the
defaults' built-in severities untouched, or gave up and set
`continue-on-error`).

| Severity | Per document | Role |
| --- | --- | --- |
| `error` | 3 | **Blocks the build.** Only things that genuinely cannot ship: an owner (`info-contact`), an addressable name for machines (`ops-operationid`), and a defined auth scheme (`security-scheme-defined`). |
| `warn` | 6 | Informs without blocking — quality and consistency guidance the team improves over time. |
| `info` | 2 | **Positive twins** — fire on what *already* complies so a report can show "82% already have an operationId" instead of only deficits. |

Counts are "per document" — 3/6/2 fire against any single spec. The file
actually declares **13** rules, because two of them (`security-scheme-defined`
and `responses-problem-shape`) ship as `-oas2` / `-oas3` twins where the 2.0 and
3.x structures differ; only the twin matching the document's format ever runs.

The blocking set is short *on purpose*. Its credibility comes from being few:
a pipeline that fails builds over hint-level style teaches teams to route around
governance into shadow APIs. Gate the few things that truly can't ship; annotate
everything else.

## Swagger 2.0 support

The design ruleset governs **both** Swagger/OpenAPI **2.0** and OpenAPI **3.x**
with parity. Spectral auto-detects a document's format (`swagger: "2.0"` →
`oas2`; `openapi: 3.x` → `oas3`) and runs a rule only when the document's format
is listed in that rule's `formats`. The file-level `formats: [oas2, oas3]` makes
that the default; individual rules narrow it when they must.

Three porting strategies are used, in priority order:

1. **Format-agnostic (one rule, no `formats`).** When the same `given`/`then`
   is valid in both versions, the rule inherits `[oas2, oas3]` and fires on
   whichever applies. Most rules are here — `info.contact`, `operationId`,
   summaries/descriptions, path kebab-case, the positive twins.
2. **Multipath `given` (one rule, no `formats`).** When only the *location*
   differs, the `given` lists both paths and the single rule matches whichever
   exists. `acme-schema-property-described` walks both
   `$.components.schemas[*].properties[*]` (3.x) and
   `$.definitions[*].properties[*]` (2.0).
3. **Format-tagged twins (`-oas2` / `-oas3`).** When the *structure* genuinely
   differs, the rule ships as a matched pair, each tagged with its `formats`, so
   a 3.x-shaped check can never misfire on a 2.0 document (and vice-versa):
   - `acme-security-scheme-defined-oas3` (`$.components.securitySchemes`) ↔
     `acme-security-scheme-defined-oas2` (root `$.securityDefinitions`).
   - `acme-responses-problem-shape-oas3` (schema inside the per-media-type
     `content` map) ↔ `acme-responses-problem-shape-oas2` (schema directly on
     the response — 2.0 media types live in `produces`, not per-response).

The `.spectral-security.yaml` house rule (`acme-security-operation-declared`) is
format-agnostic and carries an explicit `formats: [oas2, oas3]` to override the
inherited OWASP ruleset's oas3 default, so it governs 2.0 operations too.

Key 3.x → 2.0 structural map: `servers[].url` → `host` + `basePath` +
`schemes`; `components.schemas` → `definitions`; `components.securitySchemes` →
`securityDefinitions`; a request body → an `in: body` parameter's `schema`;
per-response `content['mt'].schema` → the response's own `schema` plus operation
/root `consumes`/`produces`.

## Files

- **`.spectral.yaml`** — the owned API **design** ruleset (13 rules; 11 fire on
  any one document — 2 concepts ship as oas2/oas3 twins).
- **`.spectral-security.yaml`** — the **security** ruleset, run in a separate CI
  job. It inherits the [OWASP API Security ruleset for
  Spectral](https://github.com/stoplightio/spectral-owasp-ruleset) — inheriting
  a security *standard* with real provenance is good reuse — and adds one house
  rule on top.

## Try it

```bash
# Lint a spec against the design ruleset
npx @stoplight/spectral-cli lint -r ruleset/.spectral.yaml examples/openapi.yaml

# See the positive vs. negative framing on a spec that half-complies
npx @stoplight/spectral-cli lint -r ruleset/.spectral.yaml examples/openapi-fail.yaml

# The SAME two specs in Swagger 2.0 — identical findings, proving 2.0 parity
npx @stoplight/spectral-cli lint -r ruleset/.spectral.yaml examples/swagger-2.0.yaml
npx @stoplight/spectral-cli lint -r ruleset/.spectral.yaml examples/swagger-2.0-fail.yaml

# Run the security layer
npx @stoplight/spectral-cli lint -r ruleset/.spectral-security.yaml examples/openapi.yaml
```
