---
name: safeguard-at-scale
description: Work a whole Safeguard estate of dozens or hundreds of projects without exhausting context or missing what matters, using portfolio sweeps, blast radius lookups, and sampling instead of per-project loops. Use whenever the question spans more than a few projects, including fleet-wide or org-wide posture, "which repos are worst", "is this CVE anywhere in our estate", rollout and onboarding of new repos, recurring weekly or monthly security reviews, or comparing teams and business units. Trigger even without the word "scale". Phrases like "across all our repos", "everywhere", "org-wide", "which teams are behind", and "set up a weekly check" all belong here. The naive approach of calling list_findings once per project will exhaust context long before it finishes and produce inflated counts, so read this before any multi-project work.
---

# Safeguard at scale

A mid sized estate is dozens of projects. Findings carry roughly 150 fields each with descriptions running to thousands of words. Iterating findings across every project is not slow, it is impossible. The estate has to be worked top down.

## The cost model

| Call | Rough weight | Safe use |
|---|---|---|
| `list_projects` (limit 20) | Light | Free to sweep the whole estate |
| `list_vulnerabilities` (limit 25) | Moderate, descriptions shortened | The org wide workhorse |
| `get_project` | Moderate | One project at a time |
| `list_findings` (limit 5) | Heavy | Only after a project is chosen |
| `get_vulnerability` | Heavy | Only for a named CVE |

Everything follows from this: **counts and rankings come from projects and vulnerabilities. Findings are for drilling, never for surveying.**

## Portfolio sweep

The default answer to any estate wide question.

1. `list_projects` with `limit: 20`, paging with `offset` until the returned count is exhausted. The header states the total, so you know how many pages to expect.
2. Split by `provider`. Repo and CLI scans, DAST targets, and uploaded artifacts are not comparable and must not be summed into one table.
3. Rank on the severity buckets, not on `risk_score` or `risk_level`. Those are unpopulated for DAST and will silently rank real risk as "Unknown".
4. Flag stale and incomplete scans: `assessment_status` not `completed`, `assessment_progress` under 100, or `last_scan_at` more than about 14 days old. A project that has not been scanned is a bigger problem than a project with findings, and it is invisible if you only look at counts.
5. Drill into the top three to five projects only.

Do not open every project. If asked to, explain the cost and propose the sweep plus targeted drilling instead.

## Blast radius without a loop

To answer "is this CVE anywhere in our estate", do not iterate projects. Each finding and vulnerability record already carries the answer:

- `source_data_affected_project_count`: how many projects contain it
- `source_data_affected_component_count`: how many distinct components
- `source_data_occurrence_count`: total occurrences across the estate

One `list_vulnerabilities` call with a severity or `cvss_min` filter, or one `get_vulnerability` on the named CVE, gives estate wide reach in a single call. Use `list_vulnerabilities` with `cvss_min` to pull the worst of everything without touching a single project.

Note that `ecosystem` filtering on `list_vulnerabilities` is lossy: most records do not carry an ecosystem, so filtering to `npm` returns far fewer results than actually exist for npm. Leave it as `all` unless the user specifically wants only classified records, and say so if you use it.

## Finding shared components

The highest leverage work in a large estate is the component that appears everywhere. One upgrade across 20 repos beats 20 unrelated fixes.

Sort deduped findings by `occurrence_count` and `affected_project_count` descending. Anything in more than about a quarter of projects is a platform level fix and should be routed to whoever owns the base image, shared library, or template, rather than filed against each team separately.

## Recurring reviews

**Weekly**: portfolio sweep, new criticals and highs since last week, any scan that has gone stale, any policy gate now in `block`.

**Monthly**: severity trend by business unit or provider, top shared components, projects never scanned or dormant, coverage gaps against the repo inventory.

The connector is read only and stateless, so it cannot tell you what changed since last week on its own. Comparison requires a previous snapshot. Either save the sweep output each run so the next run has a baseline, or say clearly that the numbers are point in time and no delta is available.

## Onboarding and coverage

Coverage is the thing dashboards hide. A tenant with 38 clean projects and 120 unscanned repos is not in good shape.

When asked about posture, check whether the project list matches the real repo inventory. If a source control connector is also available, compare the two lists and report the gap. If not, ask the user for the expected count. Report coverage as an explicit fraction rather than implying the scanned set is the whole estate.

## Test and placeholder projects

Real tenants accumulate scans of demo targets and test uploads. Names like `testing`, `demo`, or public vulnerable test sites such as `vulnweb.com` and `pentest-ground.com` are not production assets and will badly distort any aggregate.

Flag suspected test projects, exclude them from aggregates, and state that you have done so with the list. Do not delete them from the picture silently, and do not assume: ask if unsure.

## Where the connector stops

Read only, one tenant, no writes. For scheduling scans, changing triage state, enforcing gates in CI, opening tickets, or raising fix PRs, the answer is the Safeguard web app, CLI, or CI integration. Say so directly instead of approximating those actions in conversation.
