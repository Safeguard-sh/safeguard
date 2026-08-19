---
name: safeguard-triage
description: Turn Safeguard scan results into a ranked, batched fix list using reachability, KEV, EPSS, and fix availability rather than raw CVSS. Use whenever the user asks what to fix first, what actually matters, how to cut a backlog down, whether a CVE is exploitable or reachable in their code, what will break the build or breach an SLA, or wants a sprint plan, remediation plan, or Jira-ready list from Safeguard data. Trigger even without the word "triage". Questions like "what should I fix", "which of these matter", "we have 800 highs and no time", "is this one real", and "what do I tell the team to do this week" all belong here. Read the `safeguard` skill first for the tool rules, then this one for the ranking model.
---

# Safeguard triage

The point of triage is to turn a large severity sorted list into a short ordered list of **actions**. Severity alone is a poor ranking. Safeguard carries the signals that do the work, and they are the reason to use this platform rather than reading a CVE feed.

## The signals, in order of decision weight

| Signal | Field | Why it matters |
|---|---|---|
| Known exploited | `source_data_is_in_kev`, `source_data_exploit_info_is_in_kev` | Attackers are using it now. Overrides everything else. |
| Reachability | `source_data_is_reachable`, `source_data_in_call_path` | Not reachable means the vulnerable code path is never executed. This is the biggest backlog reducer. |
| Fix availability | `source_data_fix_version`, `source_data_priority_factors_fix_availability` | A fix that exists is cheap. A fix that does not exist needs a different response. |
| Exploit maturity | `source_data_exploit_info_exploit_available`, `exploit_maturity` | Public exploit code shortens the window. |
| EPSS | `source_data_epss_score`, `source_data_epss_percentile` | Probability of exploitation in the next 30 days. Use the percentile when talking to non specialists. |
| Directness | `source_data_is_direct`, `source_data_depth_level` | Direct dependencies are yours to fix. Deep transitives may need an upstream bump. |
| Blast radius | `source_data_occurrence_count`, `source_data_affected_project_count` | One fix clearing six projects is worth more than one clearing one. |
| Severity and CVSS | `severity`, `source_data_cvss_score` | The tiebreaker, not the sort key. |

Safeguard also precomputes `source_data_priority_score` (0 to 100), `source_data_priority_level`, and `source_data_priority_recommendation` (for example "URGENT: Schedule upgrade of postcss to 8.5.18 within 7 days"), with the contributing weights exposed in `source_data_priority_factors_*`.

**Use the platform's `priority_score` as the default sort.** It already blends these factors. Recompute manually only when the user asks for a different weighting, and say when you have departed from the platform score so the two are never confused.

## Ranking procedure

1. Pull findings for the project with a small limit. Dedupe on `cve_id + component_name + component_version` per the `safeguard` skill.
2. Sort by `source_data_priority_score` descending.
3. Promote to the top of the list, regardless of score: anything with `is_in_kev` true, and anything with `is_reachable` true **and** a `fix_version` available.
4. Demote, but never delete: `is_reachable` false. Explain that these stay on the list at lower priority rather than disappearing.
5. Stop at the top 10 unless asked for more. A ranked list nobody finishes reading is not a plan.

## Batch by component, not by CVE

This is the single largest effort reduction available and it is easy to miss.

Multiple CVEs usually land on the same component at the same version. A single pinned version of a common package will often carry two or more high severity CVEs whose fix versions differ, for example one fixed in 8.5.12 and another in 8.5.18. These are not two pieces of work. **One upgrade to the highest fix version closes both.**

So the output of triage is a list of **upgrades**, not a list of CVEs:

```
postcss  8.4.31 → 8.5.18    clears 2 CVEs (1 reachable)
lodash   4.17.20 → 4.17.21  clears 1 CVE
```

For each component, take the highest fix version across all its CVEs, then verify that version actually covers the others rather than assuming it does. Where fix versions conflict or a major version jump is involved, flag it as needing a compatibility check instead of presenting it as a clean bump.

## When there is no fix version

`fix_version` empty means an upgrade will not resolve it. Do not present it as an upgrade task. The honest options are:

- Check `source_data_risk_factors` (for example `no_distro_patch`) for why.
- If not reachable, record and monitor.
- If reachable, the response is a compensating control, removing the dependency, or vendoring a patch. Say that plainly rather than producing a task nobody can complete.

## Policy gates and SLAs

Findings carry the gate and policy they were evaluated against: `gate_name`, `gate_stage` (for example Plan & Fix, Monitor), `gate_integration_type` (Jira, SCM), `gate_function_type` (Dev, App Sec), plus `policy_name`, `policy_severity`, `policy_action` (`warn` or `block`), and `policy_threshold`.

When the user asks what will break their build or breach an SLA, filter on `policy_action` rather than on severity. A `warn` and a `block` are different conversations. `source_data_exposure_window_days` tells you how long a finding has been open, which is the number an SLA is actually measured against.

## Framing for the reader

Each finding carries prewritten context: `source_data_why_it_matters`, `source_data_business_impact`, and `source_data_exploit_scenario`. Use them, condensed. They are written for a human and save you from paraphrasing a CVE description badly.

Do not paste `source_data_description` in full. It can run to thousands of words of root cause analysis. Summarize in two sentences and offer the detail if wanted.

## What not to do

- Do not tell someone to fix 800 things. Rank, batch, and cut to a week of work.
- Do not treat unreachable as safe. It is lower priority, not resolved.
- Do not present a CVSS 9.8 that is unreachable, has no exploit, and no fix above a CVSS 7.5 that is reachable, in KEV, and has a one line fix. That inversion is the whole point of the product.
- Do not invent SLA windows. If the tenant's policy thresholds are visible, use them. If not, say the recommendation is generic.
