---
name: safeguard-reporting
description: How to present Safeguard security data so it is readable and decision-ready, with libraries of copy-ready markdown templates and rendered visual widgets (colored severity badges, posture dashboards, vulnerability cards), layouts for the four audiences (engineer, AppSec lead, executive, auditor), and the fields that must never be shown. Use whenever Safeguard results are being shown to a person or written into a report, summary, update, ticket, deck, email, or dashboard, and whenever the user asks for a security report, posture summary, board update, audit evidence, or "make this readable". Trigger even without the word "report". Requests like "summarize the scan", "write this up for my VP", "what do I tell the auditor", "turn this into a ticket", and "show me our posture" all belong here. Safeguard tool responses are raw denormalized JSON with internal IDs and thousand-word descriptions, so dumping them unformatted is never the right answer.
---

# Presenting Safeguard data

Raw connector output is machine shaped: around 150 fields per finding, several fields repeated under different names, internal identifiers throughout, and embedded markdown descriptions running to thousands of words. None of it is presentable as returned. Always transform.

## Pick the audience first

Ask if it is unclear. The same scan produces four different documents.

| Audience | Wants | Leads with | Never include |
|---|---|---|---|
| **Engineer** | What to change in the code | Component, current version, target version | Business impact prose, risk scores |
| **AppSec lead** | Where to spend the team's week | Ranked list with reachability and blast radius | Full CVE descriptions |
| **Executive** | Direction of travel and exposure | Trend, top three risks, one number | CVE IDs, component names, CVSS |
| **Auditor** | Coverage, evidence, dates | Scan dates, scope, policy status | Anything undated or unsourced |

## Never show these

Suppress unless explicitly asked: `org_id`, `tenant_id`, `product_id`, `component_id`, `gate_id`, `policy_id`, `sbom_id`, finding UUIDs, `source_type`, `active`, `version` (the row version, not the component version), and the `organization` / `orgName` / `organizationName` triplicate.

Never render the `source_data_` prefix in output. It is a serialization artifact. Present `source_data_cvss_score` as "CVSS", not as its field name.

`source_data_component_paths` is redacted to `***`. Do not present it and do not imply file level knowledge you do not have.

## Layouts

**Two template libraries live in `references/`. Read the relevant one whenever you are about to render Safeguard data.**

- `chat-templates.md` for markdown output: tables, cards in text, ticket blocks, error states.
- `visual-templates.md` for rendered widgets: colored severity badges, posture dashboards, vulnerability cards, signal chips. Use these when the visual encoding does real work, and fall back to markdown when no visual renderer is available in the session.

`chat-templates.md` It has fill-in templates for the estate overview, single project, ranked fix list, vulnerability card, blast radius answer, executive summary, weekly digest, coverage report, audit evidence, ticket blocks, and the empty and error states.

The two rules that govern all of them:

**Markdown tables for reading. Fenced code blocks only for copying.** A fenced block does not wrap, so on a phone it becomes a horizontal scroll bar. Reserve fenced blocks for text that gets pasted elsewhere, such as ticket bodies and commands. Everything the user reads in the conversation is a real markdown table.

**Six columns is the ceiling**, numbers right aligned with `--:`, and long identifiers truncated. Chat is narrow and often mobile.

A worked example, the estate overview, split by provider because the columns mean different things per provider:

| Project | Crit | High | Med | Low | Components |
|---|--:|--:|--:|--:|--:|
| payments-api | 1 | 4 | 5 | 10 | 204 |
| internal-docs | 0 | 18 | 10 | 18 | 314 |
| web-frontend | 0 | 4 | 4 | 53 | 134 |

DAST targets go in a separate table with no Components column, since it is always 0 there and reads as "no dependencies" rather than "not applicable".

## Numbers

- Thousands separators on anything above 999.
- Never present `total_vulnerabilities` of exactly 10000 as exact. It is capped. Write "10,000+".
- When severity buckets and the platform total disagree, show the buckets and footnote the difference. Do not silently pick one.
- Round CVSS to one decimal. Present EPSS as a percentage with the percentile alongside, since the raw score is meaningless to most readers.
- Deduped counts only. "47 distinct vulnerabilities across 12 components", never a raw finding row count.

## Severity styling

Order critical, high, medium, low, always in that direction. Be consistent within a response. If using color or symbols, use them once as a severity indicator and nowhere else. Do not decorate a security report with emoji beyond that, and never in auditor or executive output.

## Tables, prose, and visuals

Use a table when there are more than three items with the same shape. Use prose for a single finding, because a one row table is harder to read than a sentence.

A chart is worth it for severity distribution across a portfolio or open counts over time. It is not worth it for four numbers. When the user wants something they will circulate, offer a file. When they are reading it in the conversation, keep it inline.

## Freshness

Lead with a staleness note rather than burying it. A clean report from a scan three months old is worse than no report, because it will be believed.

```
Note: last scan 12 June 2026, 69 days ago. These numbers may not reflect
current code.
```
