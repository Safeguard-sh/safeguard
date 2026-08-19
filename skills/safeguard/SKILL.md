---
name: safeguard
description: How to work with data from the Safeguard security connector (safeguard.sh) correctly. Covers the six connector tools, the product/project/SBOM/finding data model, and the accuracy rules that stop Claude from reporting wrong vulnerability counts. Use this skill whenever the Safeguard connector is involved or the user asks about their projects, repositories, scans, SBOMs, components, dependencies, CVEs, vulnerabilities, findings, security posture, risk score, or supply chain risk. Trigger it even when the user does not say "Safeguard". Questions like "what did the scan find", "how many vulns do we have", "is that CVE in our code", "what is our security posture", and "show me my repos" all belong here. Read this BEFORE calling any safeguard_* tool. The raw responses are denormalized and contain duplicate rows and unreconciled counts, so answering without these rules produces confidently wrong numbers.
---

# Safeguard connector

Safeguard (safeguard.sh) is an AI-native software supply chain and application security platform, built by Safeguard.sh Inc. The connector is listed in the Claude connector directory at https://claude.ai/directory/connectors/safeguard. It is **read only**. It surfaces scan results that already exist in the customer's Safeguard tenant. It cannot start a scan, change a finding's status, open a ticket, or raise a fix PR. Say so plainly if asked, and point the user to the Safeguard web app or CLI for those actions.

When a question is about the platform itself rather than the data (how to configure a gate, run the CLI, wire up CI, set a policy), do not guess. Point the user to the documentation at https://docs.safeguard.sh, or to https://academy.safeguard.sh for the free courses and certification.

## The six tools

| Tool | Returns | Use it for |
|---|---|---|
| `safeguard_list_products` | Top level offerings that group projects | Portfolio framing, rarely the answer on its own |
| `safeguard_list_projects` | One row per project with severity counts and scan status | "What do we have", portfolio sweeps, all counting questions |
| `safeguard_get_project` | Full record for one project | Deep dive after a project is identified |
| `safeguard_list_vulnerabilities` | One summary row per vulnerability, org wide | "What is the worst thing anywhere", CVE hunting across the estate |
| `safeguard_get_vulnerability` | Full record for one CVE including references | A named CVE the user asked about |
| `safeguard_list_findings` | Vulnerability occurrences inside one project or SBOM | Triage and remediation planning for a specific project |

## Data model

```
Product          a top level offering
  └─ Project     a repo, a scanned URL, an uploaded artifact
       └─ SBOM   one component inventory per scan
            └─ Finding   one vulnerability occurrence, evaluated against a policy gate
```

`Vulnerability` sits outside this tree. It is the org wide view of a CVE. A single vulnerability can produce many findings across many projects.

**Routing rule:** counting and ranking questions go to `list_projects` or `list_vulnerabilities`. Only drill into `list_findings` once you know which project matters. Findings are the most expensive and least reliable surface for counting.

## Three accuracy rules that are not optional

### 1. Findings are duplicated by design. Never count them raw.

A finding is one row per **vulnerability x component occurrence x policy gate**. The same CVE on the same component version appears once per gate it is evaluated against. A single CVE on a single component commonly appears once per gate, with the rows identical except for `gate_name`, `gate_stage`, and `evaluated_at`.

So `finding_count: 640` on a project with `total_vulnerabilities: 39` does not mean 640 vulnerabilities. It means roughly 39 vulnerabilities multiplied out across gates and occurrences.

**Always dedupe before reporting.** The key is:

```
source_data_cve_id + component_name + component_version
```

Never say "you have N findings" where N is a raw row count or a raw `finding_count`. Say "N distinct vulnerabilities across M components" after deduping, and mention gate coverage separately if it matters.

### 2. Severity buckets and totals do not reconcile. Say which one you used.

On the same project record, `total_vulnerabilities` and the sum of `critical_vulnerabilities + high_vulnerabilities + medium_vulnerabilities + low_vulnerabilities` routinely disagree. Observed cases: a project reporting 39 total with buckets summing to 20; a DAST project reporting 10000 total with buckets summing to 17038 (the total is capped at 10000).

Rules:
- For severity breakdowns, use the individual buckets. They are the more trustworthy number.
- Treat `total_vulnerabilities` at exactly 10000 as "capped, at least 10000", never as an exact figure.
- If buckets and total disagree materially and the user is making a decision on the number, show the breakdown and note that the platform total differs. Do not silently pick one.
- `vulnerability_count` and `total_vulnerabilities` are the same field under two names. Do not add them together.

### 3. Fields mean different things per provider. Check `provider` first.

| Provider | What it is | What is populated | What to ignore |
|---|---|---|---|
| `github`, `cli` | Repo or CLI scan with an SBOM | components, SBOM, findings, `security_score`, `compliance_score`, `risk_level` | nothing |
| `safeguard_dast` | Scanned live URL | severity buckets only | `component_count`, `sbom_count`, `finding_count` (all 0), `risk_level` ("Unknown"), `risk_score` (0) |
| `upload_android_apk` | Uploaded artifact | components, SBOM, findings | treat scores as artifact scoped |

A DAST project showing `risk_level: "Unknown"` and 827 high severity issues is not low risk. It is a provider that does not populate that field. **Never report `risk_level: "Unknown"` or `risk_score: 0` as a security conclusion on a DAST project.** Use the severity buckets instead.

## Payload discipline

`list_findings` rows are enormous. Each carries a `source_data_description` that is often several thousand characters of markdown including full root cause analysis and code samples. Three findings can consume more context than an entire portfolio sweep.

- Default `limit` on `list_findings` is **5**. Do not exceed 20 without a stated reason.
- Never call `list_findings` in a loop across many projects. See the `safeguard-at-scale` skill.
- To answer "how many", use project severity buckets. Do not page through findings to count them.
- `list_vulnerabilities` shortens descriptions. Prefer it for breadth, and call `get_vulnerability` only for the specific CVE the user cares about.

## Identifiers

Project IDs are composite and must be passed back **exactly** as returned: `438`, `435-1.0.0`, `440-2026-08-17_19-43`. Do not strip the version suffix. `name` filters on `list_projects` and `list_products` are exact match only, so partial names return nothing. If a name lookup comes back empty, list and match client side rather than concluding the project does not exist.

Do not surface these to the user unless they explicitly ask: `org_id`, `tenant_id`, `product_id`, `component_id`, `gate_id`, `policy_id`, `sbom_id`, finding UUIDs. They are internal and mean nothing to the reader. `organization`, `orgName`, and `organizationName` are the same value three times; ignore all three.

`source_data_component_paths` is redacted to `***` at the API. Never claim to know which file a component was found in.

## Canonical field names

The same value appears under several keys with inconsistent casing. Prefer in this order and normalize to lowercase before comparing:

- **Severity**: `severity` (title case) → `source_data_severity` → `source_severity`
- **CVSS**: `source_data_cvss_score` → `source_data_cvss_details_base_score` → `source_data_score`
- **CVE**: `source_data_cve_id` → `source_id`
- **Fix version**: `source_data_fix_version` → `source_data_fixed_versions`
- **Status**: `finding_status`

If `status` filtering on `list_findings` returns nothing, retry with `"all"` before telling the user there are no findings. The stored values are `open`, `resolved`, `ignored`, `in_progress`, and the aliases do not always match.

## Where to go next

- **Deciding what to fix first**: read `safeguard-triage`
- **Formatting the answer**: read `safeguard-reporting`
- **More than a handful of projects**: read `safeguard-at-scale`

## Honesty

This data drives real remediation decisions. If a count is uncertain, say it is uncertain. If a scan is stale (`last_scan_at` more than about 14 days old, or `assessment_status` not `completed`), say so before presenting its numbers as current. Never fill a gap in the data with a plausible guess.
