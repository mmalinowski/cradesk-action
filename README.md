# CRA readiness report - GitHub Action

Turns a CycloneDX or SPDX SBOM into a **Cyber Resilience Act readiness report** in your job
summary: what your SBOM does and does not tell you, which CRA class your product falls into,
and what the 11 September 2026 reporting obligation requires you to have in place.

Free, no account, no data leaves the runner - the report is computed in the job.

```yaml
- uses: actions/checkout@v4

# Any SBOM generator works; the action does not generate one.
- run: npx @cyclonedx/cdxgen -o sbom.json .

- uses: mmalinowski/cradesk-action@v0
  with:
    sbom-path: sbom.json
    github-token: ${{ secrets.GITHUB_TOKEN }} # optional: also comment on the PR
```

## Inputs

| Input | Default | Description |
|---|---|---|
| `sbom-path` | - | CycloneDX or SPDX **JSON**. Comma-separated list allowed, and one `*` in the final segment (`build/*.cdx.json`). Omit it and the report explains how to generate one. |
| `config-path` | `cradesk.yml` | Answers the classifier questions and the checklist items CI cannot know. |
| `comment` | `true` | Post the report as a PR comment (needs `github-token` and `pull-requests: write`). |
| `github-token` | - | Used only for the PR comment. Without it: job summary only. |

The action never fails your build. A missing SBOM, an unreadable file or a refused comment are
reported, not thrown.

## `cradesk.yml`

Everything is optional. Without a `classification` block the report says the class is
undetermined and links the browser classifier.

```yaml
product: Acme Firewall

classification:
  # A category from Annex III / Annex IV, or `none` (not listed) or `unknown`.
  categorySlug: firewalls-ids-ips
  # commercial | commercial-foss | non-commercial-foss | steward
  distribution: commercial
  # user-device | browser-only-service | remote-part-required-for-function
  execution: user-device
  # none | medical-device | motor-vehicle | civil-aviation | marine-equipment | national-security
  otherUnionLaw: none

# Checklist items only you can answer.
answers:
  srp-account: true
  cvd-policy: true
  vulnerability-contact: true
  incident-owner: false
```

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
coverage is declared, never complete. Rules version and tool version are stamped on every
report.

---

Built by [CRA Desk](https://cradesk.dev.cloudsoft.com.pl). The classification rules live in the
CRA Desk monorepo and are versioned with citations; this repository is the packaged Action.
