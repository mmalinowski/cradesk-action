# CRA readiness report - GitHub Action

Turns a CycloneDX or SPDX software bill of materials (SBOM) into a **Cyber Resilience Act (CRA)
readiness report** in your job summary: what your SBOM does and does not tell you, which CRA
class your product falls into, and what the 11 September 2026 reporting obligation requires you
to have in place.

Free, no account required - the report is computed in the job and no data leaves the runner
unless you opt into uploading to a CRA Desk panel (see below).

```yaml
- uses: actions/checkout@v4

# Any SBOM generator works; the action does not generate one.
- run: npx @cyclonedx/cdxgen -o sbom.json .

- uses: mmalinowski/cradesk-action@v1
  with:
    sbom-path: sbom.json
    github-token: ${{ secrets.GITHUB_TOKEN }} # optional: also comment on the PR
```

The action reads an SBOM and never generates one; `cdxgen` above is one option. The
[SBOM generation guide](https://cradesk.eu/sbom) has a generate step for every ecosystem the
vulnerability watch covers - Maven, Gradle, Python, Go, .NET, Rust, Ruby, PHP, Elixir and Alpine
images among them.

## What the report contains

- **SBOM** - how many components carry a package URL (purl), a version and a licence, and
  whether dependency relationships are recorded.
- **Coverage** - how many components the vulnerability watch can match, by ecosystem, and what
  falls outside its sources. [Watched ecosystems and sources](https://cradesk.eu/coverage).
- **CRA scope** - product class and conformity route, from your `cradesk.yml` answers. The same
  questions run in the [browser classifier](https://cradesk.eu/classifier).
- **Readiness checklist** - the checks behind the Article 14 reporting obligation, each with its
  legal source. [The checklist, explained](https://cradesk.eu/readiness).

Background on the obligation itself:
[Cyber Resilience Act reporting started 11 September 2026](https://cradesk.eu/articles/cra-reporting-starts-sept-11).

## Inputs

| Input | Default | Description |
|---|---|---|
| `sbom-path` | - | CycloneDX or SPDX **JSON**. Comma-separated list allowed, and one `*` in the final segment (`build/*.cdx.json`). Omit it and the report explains how to generate one. |
| `config-path` | `cradesk.yml` | Answers the classifier questions and the checklist items CI cannot know. |
| `report-path` | - | Also write the report to this file. GitHub Actions already has a job summary; this exists for the GitLab component, which has no equivalent surface. |
| `comment` | `true` | Post the report as a PR comment (needs `github-token` and `pull-requests: write`). |
| `github-token` | - | Used only for the PR comment. Without it: job summary only. |
| `token` | - | CRA Desk ingest token (panel's Tokens page). Presence of this input is what turns on upload. |
| `api-url` | - | Base URL of your CRA Desk panel. No default yet - without it, upload is skipped and the report says so. |
| `product-id` | - | Product ID from the panel. Needed for upload to succeed once `token` is set. |
| `product-version` | - | Label for this upload (e.g. a release tag), shown in the product's SBOM history. |

## Uploading to a CRA Desk panel

Optional, and off unless you set `token`. Uploads the same SBOM the report was built from,
gzip-compressed, to your panel's ingest endpoint - continuous CVE/KEV watch over the stored
inventory is the panel's job, not this action's.

```yaml
- uses: mmalinowski/cradesk-action@v1
  with:
    sbom-path: sbom.json
    token: ${{ secrets.CRADESK_TOKEN }}
    api-url: https://cradesk.eu
    product-id: ${{ vars.CRADESK_PRODUCT_ID }}
    product-version: ${{ github.ref_name }}
```

A missing `api-url`, a rejected token, or an unreachable panel never fails the build - the
"Upload" section of the report says what happened instead. An SBOM matching a previous upload
is reported as already ingested, not as an error - re-running the same build twice is a normal
case, not a mistake.

The action never fails your build. A missing SBOM, an unreadable file or a refused comment are
reported, not thrown.

<!-- cradesk-config:start -->
## `cradesk.yml`

Answers what a scan cannot derive from an SBOM: what the product *is*, and which organisational
duties you already have in place. The file is optional and every part of it is optional - without
one, the report still scores the four SBOM checks, reports the CRA class as undetermined, and
links the browser classifier.

Put it in the repository root, or point the config-path input (see Inputs above) somewhere else.
The default is `cradesk.yml`.

Two things to know before writing one:

- **A file that fails to parse is ignored, exactly as if it were absent.** A typo in an enum
  value, a wrong indent, an unknown field - the whole file is discarded and the report reads as
  though you never wrote it. After adding or editing one, check that the report's Classification
  section actually names your class.
- **Nothing is ever assumed answered.** An omitted checklist key is reported as unanswered, never
  as met. Omit a key rather than guessing at it - an unanswered duty is what the report exists to
  surface.

### Full example

```yaml
# Free text. Names the product in the report header. Optional.
product: Acme Firewall

# All four fields are required if this block is present. Omit the whole block and the report
# says the class is undetermined instead.
classification:
  categorySlug: firewalls-ids-ips
  distribution: commercial
  execution: user-device
  otherUnionLaw: none

# Checklist items only you can answer. Booleans; omit what you do not know.
answers:
  product-scope-known: true
  vulnerability-monitoring: true
  incident-owner: true
  csirt-coordinator-known: false
  srp-account: false
  cvd-policy: true
  vulnerability-contact: true
  user-notification-path: true
  conformity-route-known: true
```

### `classification.categorySlug`

Where the product sits in the CRA's own annexes. Use `none` if no category fits - absence from
the annexes is itself a verdict (default class, self-assessment), not a gap. Use `unknown` if you
have not decided yet; the report then says so rather than picking for you.

Important, class I (Annex III):
`identity-and-privileged-access-management`, `browsers`, `password-managers`, `malware-detection`,
`vpn`, `network-management-systems`, `siem`, `boot-managers`, `pki-and-certificate-issuance`,
`network-interfaces`, `operating-systems`, `routers-modems-switches`,
`microprocessors-with-security-functions`, `microcontrollers-with-security-functions`,
`asic-fpga-with-security-functions`, `smart-home-virtual-assistants`,
`smart-home-security-products`, `internet-connected-toys`, `personal-wearables`

Important, class II (Annex III):
`hypervisors-and-container-runtimes`, `firewalls-ids-ips`, `tamper-resistant-microprocessors`,
`tamper-resistant-microcontrollers`

Critical (Annex IV):
`hardware-devices-with-security-boxes`, `smart-meter-gateways`, `smartcards-and-secure-elements`

### `classification.distribution`

| Value | Meaning |
|---|---|
| `commercial` | Sold, licensed, or otherwise supplied in the course of a commercial activity. |
| `commercial-foss` | Open source, but monetised - a paid product, support, or a hosted service around it. In scope, with a lighter conformity route available. |
| `non-commercial-foss` | Open source developed outside any commercial activity. |
| `steward` | An open-source software steward under the CRA's own definition, not a manufacturer. |

Free and open source does not by itself put a product outside the CRA. What matters is whether it
is supplied in the course of a commercial activity - a free tool distributed to sell something
else is `commercial-foss`.

### `classification.execution`

| Value | Meaning |
|---|---|
| `user-device` | Runs on the user's own machine or hardware. |
| `browser-only-service` | Reached only through a browser, with no component the user installs. |
| `remote-part-required-for-function` | Installed software whose function depends on a remote part you operate. |

### `classification.otherUnionLaw`

Sector legislation that displaces or overlays the CRA for this product. One of `none`,
`medical-device`, `motor-vehicle`, `civil-aviation`, `marine-equipment`, `national-security`.

### `answers`

A map of checklist slug to `true` or `false`. Nine items can be answered here:

| Slug | Question it answers |
|---|---|
| `product-scope-known` | The product's CRA scope and class are established. |
| `vulnerability-monitoring` | Components are monitored against vulnerability sources. |
| `incident-owner` | A named person owns the 24-hour early warning. |
| `csirt-coordinator-known` | The CSIRT designated as coordinator is identified. |
| `srp-account` | The route into the ENISA Single Reporting Platform is arranged. |
| `cvd-policy` | A coordinated vulnerability disclosure policy is published. |
| `vulnerability-contact` | A contact address for vulnerability reports is discoverable. |
| `user-notification-path` | There is a way to notify affected users. |
| `conformity-route-known` | The 2027 conformity route is known. |

Four further checks are read from the SBOM itself and ignore anything you put here:
`sbom-machine-readable`, `component-identifiers`, `component-versions`, `dependency-graph`.
<!-- cradesk-config:end -->

## What it checks

Four checks come from the SBOM itself: that it is machine-readable at all (Annex I Part II(1)),
that components carry package identifiers and versions, and whether dependency relationships are
recorded. The rest are the duties a tool cannot discharge for you - vulnerability monitoring, a
named owner for the 24-hour early warning, the CSIRT designated as coordinator for your main
establishment, access to ENISA's Single Reporting Platform, a coordinated vulnerability
disclosure policy, a contact address for reports, and a way to notify affected users
(Article 14(8)). Unanswered items are reported as unanswered, never as met.

Every line cites the provision it comes from. Where a check rests on practice rather than on an
operative provision, the report says so.

## Formats

CycloneDX 1.5, 1.6 and 1.7, SPDX 2.3 - JSON only. XML and SPDX tag-value are detected and explained
rather than mis-parsed.

## What ships here

`index.cjs` is a bundle: the rules, the SBOM parsers and the two npm dependencies (`zod`,
`yaml`) are inlined, so a runner executes one file with no install step. It is not minified -
you can read and diff it. Third-party licence texts travel with it in
`THIRD-PARTY-LICENSES.md`, generated from what the bundler actually included.

The report you generate is yours: no licence condition attaches to the output, and it is meant
to go into your compliance file.

## Licence

Apache-2.0 - see [LICENSE](LICENSE) and [NOTICE](NOTICE). You can read the code, fork it, and
run it in a private pipeline without asking. The classification rules and readiness checklist
are derived from EU legal acts published on EUR-Lex; only the EU's own published texts are
authentic.

The hosted CRA Desk product (the classifier at cradesk, the panel, the reporting workflow) is
**not** covered by this licence - it has its own terms of service.

## This is not legal advice

The report is a compliance aid. The obligations rest with the manufacturer; vulnerability-source
coverage is declared, never complete. Every report is stamped with the tool version and the
checklist version, and with the classification rules version whenever it carries a verdict.

---

Built by [CRA Desk](https://cradesk.eu). The classification rules live in the
CRA Desk monorepo and are versioned with citations; this repository is the packaged Action.
