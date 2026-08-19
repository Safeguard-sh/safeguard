# Safeguard

Companion skills for the Safeguard security connector. They teach Claude how to read Safeguard scan data correctly, decide what is worth fixing, and present the answer in a form a person can act on.

Built and maintained by [Safeguard.sh Inc](https://safeguard.sh).

## About Safeguard

Safeguard is an AI-native software supply chain and application security platform. The `.sh` stands for Self-Healing: where most tools scan and alert, Griffin AI traces which vulnerabilities are actually reachable and authors the fix.

The platform covers SBOM lifecycle management, SCA, SAST, DAST, zero-day discovery, third-party risk management, and the Gold Registry of pre-vetted zero-CVE components, with policy gates that enforce standards in CI.

| | |
|---|---|
| Website | [safeguard.sh](https://safeguard.sh) |
| Connector in the Claude directory | [claude.ai/directory/connectors/safeguard](https://claude.ai/directory/connectors/safeguard) |
| Documentation | [docs.safeguard.sh](https://docs.safeguard.sh) |
| Academy, free courses and certification | [academy.safeguard.sh](https://academy.safeguard.sh) |
| Blog | [safeguard.sh/resources/blog](https://safeguard.sh/resources/blog) |
| About the company | [safeguard.sh/company/about](https://safeguard.sh/company/about) |
| GitHub | [github.com/Safeguard-sh](https://github.com/Safeguard-sh) |
| X | [@safeguard_sh](https://x.com/safeguard_sh) |

## Why the skills matter

The connector returns raw platform data. Three properties of that data will produce confidently wrong answers if Claude reads it naively:

**Findings are duplicated by design.** A finding is one row per vulnerability, per component occurrence, per policy gate. A project reporting 640 findings may hold around 39 distinct vulnerabilities. Without a deduplication rule, a user is told they have sixteen times more work than they do.

**Counts do not reconcile.** Severity buckets and the platform total routinely disagree on the same record, and totals cap at 10,000.

**Fields mean different things per scan type.** A DAST project reports `risk_level: "Unknown"` and `risk_score: 0` while carrying hundreds of high severity issues. Read literally, the highest volume assets look like the safest ones.

These skills encode the handling for all three, so the numbers a user sees are the numbers that are true.

## What is included

| Skill | Purpose |
|---|---|
| `safeguard` | Entry point. Tool map, data model, and the accuracy rules. Everything else depends on it. |
| `safeguard-triage` | Ranks and batches remediation using reachability, KEV, EPSS, and fix availability rather than raw CVSS. Outputs component upgrades, not CVE lists. |
| `safeguard-reporting` | Presentation. Layouts for engineer, AppSec lead, executive, and auditor, plus two template libraries: markdown output and rendered visual widgets. |
| `safeguard-at-scale` | Portfolio work across dozens or hundreds of projects without exhausting context. |

Claude loads these on its own when the subject matches. No commands to memorise.

## Requirements

A Safeguard account, and the Safeguard connector authorised.

The Safeguard connector is listed in the Claude connector directory at [claude.ai/directory/connectors/safeguard](https://claude.ai/directory/connectors/safeguard). If it is already connected there, these skills attach to that existing connection and nothing further is needed.

This plugin also bundles the connector definition, so installing the plugin on its own makes both the connector and the skills available in one step.

Sign up at [safeguard.sh](https://safeguard.sh).

## Scope

The connector is read only. It surfaces results that already exist in a Safeguard tenant. It cannot start a scan, change a finding's triage state, open a ticket, or raise a fix pull request. Those actions live in the Safeguard web app, CLI, and CI integration. The skills instruct Claude to say so plainly rather than approximating them.

## What it looks like in use

Ask in plain language:

- "What should we fix first in the payments repo?"
- "Is that CVE anywhere in our estate?"
- "Summarise our security posture for the board."
- "Which repositories have not been scanned recently?"
- "Turn the top three into Jira tickets."

Claude reaches for the right skill, calls the connector, applies the accuracy rules, and formats for whoever is reading.

## Support

Documentation is at [docs.safeguard.sh](https://docs.safeguard.sh). For product questions, evaluations, or issues with these skills, contact the team through [safeguard.sh](https://safeguard.sh).

---

© Safeguard.sh Inc. Safeguard and Griffin are trademarks of Safeguard.sh Inc.
