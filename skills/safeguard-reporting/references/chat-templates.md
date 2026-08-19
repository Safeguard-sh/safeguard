# Chat rendering templates

Fill-in templates for presenting Safeguard data inside a chat response. Replace `{tokens}`. Delete any row, column, or line you do not have real data for. Never leave a token unfilled and never invent a value to fill one.

## Contents

1. [Chat rendering rules](#1-chat-rendering-rules)
2. [Estate overview](#2-estate-overview)
3. [Single project](#3-single-project)
4. [Ranked fix list](#4-ranked-fix-list)
5. [Single vulnerability card](#5-single-vulnerability-card)
6. [Blast radius answer](#6-blast-radius-answer)
7. [Executive summary](#7-executive-summary)
8. [Weekly digest](#8-weekly-digest)
9. [Coverage and freshness](#9-coverage-and-freshness)
10. [Audit evidence](#10-audit-evidence)
11. [Ticket and PR blocks](#11-ticket-and-pr-blocks)
12. [Empty, partial, and error states](#12-empty-partial-and-error-states)

---

## 1. Chat rendering rules

**Markdown tables for reading. Fenced code blocks only for copying.**

A fenced block does not wrap, so on a phone it becomes a horizontal scroll bar. Use markdown pipe tables for anything the user reads, and reserve fenced blocks for text that gets pasted somewhere else: ticket bodies, commit messages, CLI commands, config.

**Six columns is the ceiling.** Chat is narrow. If a table needs more, split it into two tables or move detail into the row beneath.

**Right align numbers** with `--:` in the separator row. Severity counts scan far faster in a column that lines up.

**Truncate long identifiers.** Project names like `https://pentest-ground.com:4280/` blow out a column. Show `pentest-ground.com:4280` and keep the full value only if the user needs to act on it.

**The facts strip.** For a single object, a `·` separated line carries five or six attributes in one line without a table:

```
High · CVSS 7.5 · EPSS 0.6% (46th) · Reachable · Fix available
```

**One bold lead line per section.** The reader should get the answer before the evidence.

**Charts are rarely worth it in chat.** Severity distribution across 20+ projects or a trend over time can justify one. Four numbers never do. If the user wants something to circulate, offer a file instead.

---

## 2. Estate overview

Split by provider. The columns mean different things per provider and must not share a table.

```markdown
**{n} projects scanned · {crit} critical · {high} high open**
Scans from {earliest date} to {latest date}.

### Code repositories

| Project | Crit | High | Med | Low | Components |
|---|--:|--:|--:|--:|--:|
| {name} | {c} | {h} | {m} | {l} | {count} |

### Web targets (DAST, no component inventory)

| Target | Crit | High | Med | Low | Last scan |
|---|--:|--:|--:|--:|---|
| {host} | {c} | {h} | {m} | {l} | {date} |

{staleness or exclusion note, if any}
```

Sort by critical descending, then high. Do not put a Components column on DAST rows.

---

## 3. Single project

```markdown
**{project name}** · {provider} · last scanned {date}

| | Critical | High | Medium | Low |
|---|--:|--:|--:|--:|
| Open | {c} | {h} | {m} | {l} |

{count} components · {n} distinct vulnerabilities across {m} components

**Worth your attention**
{one or two sentences naming the single most important thing and why}
```

If the platform total disagrees with the severity buckets, add:

```markdown
> The platform reports {total} total against buckets summing to {sum}.
> The breakdown above is the more reliable figure.
```

---

## 4. Ranked fix list

The default deliverable. One row per **component upgrade**, never per CVE.

```markdown
**{n} upgrades clear {m} vulnerabilities**

| # | Component | Upgrade to | Clears | Reachable | Also in |
|--:|---|---|--:|---|--:|
| 1 | {name} {version} | {fix version} | {n} CVEs | Yes | {k} projects |
```

When the reasoning matters more than the density, use the list form instead:

```markdown
**1. {component} {current} to {fix version}**
Clears {n} CVEs, {k} of them reachable. Direct dependency.
Present in {m} other projects.

**2. {component} {current} to {fix version}**
Clears {n} CVE. Not reachable, so this can wait for the next dependency sweep.
```

Cut at ten. A list nobody finishes is not a plan. If there are more, say how many are below the line.

---

## 5. Single vulnerability card

```markdown
**{CVE ID}: {short human title}**
{Severity} · CVSS {score} · EPSS {pct}% ({percentile}th) · {KEV status} · {reachability}

Affects **{component} {version}** across {n} projects, {m} occurrences.
{Fixed in X | No fix available}.

{Two or three sentences on what the flaw actually is and what an attacker gets.}

**Why it matters here**
{one sentence grounded in this tenant, not generic CVE text}
```

Never paste `source_data_description` in full. It can run past several thousand words. Summarize, then offer the detail.

---

## 6. Blast radius answer

For "is this CVE anywhere in our estate".

```markdown
**Yes. {CVE} is in {n} projects.**

{component} {version} · {m} total occurrences · fixed in {version}

| Project | Occurrences | Reachable |
|---|--:|---|
| {name} | {n} | {Yes/No} |

One upgrade to {fix version} clears all of them.
```

If the answer is no, say so in one line and state what you checked, so the negative is trustworthy:

```markdown
**No. {CVE} does not appear in any of your {n} scanned projects.**
Checked against the full vulnerability index as of {date}.
```

---

## 7. Executive summary

No CVE IDs. No component names. No CVSS. One number, its direction, and the action.

```markdown
Across {n} scanned projects there {is/are} **{c} critical and {h} high severity
{issue/issues}** open. {The critical is/The criticals are} in {direct dependencies
with fixes available / X}, scheduled {timeframe}. {One sentence on the largest
concentration of risk and who owns it.}

{One sentence on coverage: what fraction of the estate this represents.}
```

If no prior snapshot exists, do not imply a trend. Write "point in time" explicitly.

---

## 8. Weekly digest

```markdown
**Week of {date}**

| | This week |
|---|--:|
| New critical | {n} |
| New high | {n} |
| Closed | {n} |
| Scans gone stale | {n} |

**Needs a decision**
- {item and the decision required}

**Fix queue**
{the top three from the ranked fix list, one line each}
```

The connector is stateless and cannot compute a delta on its own. Either you have a saved snapshot from last run, or the "This week" column is unavailable and you say so rather than estimating.

---

## 9. Coverage and freshness

Coverage is the number dashboards hide. Lead with it whenever posture is the question.

```markdown
**{n} of {m} known repositories are scanned ({pct}%).**

| Status | Projects |
|---|--:|
| Scanned in last 14 days | {n} |
| Stale (over 14 days) | {n} |
| Scan incomplete | {n} |
| Never scanned | {n} |

{List the stale and never-scanned projects by name if under about ten.}
```

If the true repository count is unknown, ask for it rather than reporting the scanned set as if it were the whole estate.

---

## 10. Audit evidence

Every claim carries a date and a scope.

```markdown
**Scope**: {n} projects, scanned {method}, covering {what}
**Period**: {start} to {end}
**Excluded**: {anything omitted, and why}

| Project | Method | Last scan | Status | Critical | High |
|---|---|---|---|--:|--:|
| {name} | {provider} | {date} | {complete/partial} | {c} | {h} |

**Policy status**
{gate and policy names with their action and current pass/fail}
```

Never present a partial scan as complete. If `assessment_progress` is under 100, that project's row says so.

---

## 11. Ticket and PR blocks

These get copied, so use a fenced block.

```
Title: Upgrade {component} {current} to {fix version}

Clears {n} vulnerabilities:
  {CVE-1}  {severity}  CVSS {score}  {reachable/not reachable}
  {CVE-2}  {severity}  CVSS {score}  {reachable/not reachable}

Component: {name} {version}
Dependency: {direct/transitive, depth N}
Blast radius: {m} projects, {k} occurrences
Policy: {policy name} ({action}), open {n} days

Action: bump to {fix version} and run the test suite.
```

Keep the block plain. No markdown formatting inside, since it lands in a tracker that may not render it.

---

## 12. Empty, partial, and error states

These are most of what goes wrong in practice, and a vague answer here costs more trust than a wrong number.

**No findings returned**

```markdown
No open findings returned for {project}. Two things worth confirming before
treating that as clean: the scan completed on {date}, and this project is a
{provider} scan, which {does/does not} produce component findings.
```

Retry with `status: "all"` before reporting zero. The status aliases do not always match stored values.

**Project not found by name**

Name filters are exact match only. Say what you did:

```markdown
No exact name match for "{query}". Closest in your estate: {a}, {b}, {c}.
Which one did you mean?
```

**Capped counts**

```markdown
{Project} reports 10,000+ issues. That figure is capped at the platform
limit, so the true count is higher. The severity breakdown below is accurate.
```

**Stale data**

Lead with it. Do not bury it in a footnote.

```markdown
> Last scan {date}, {n} days ago. These numbers may not reflect current code.
```

**Suspected test projects**

```markdown
Excluded from the totals above: {names}. These look like test or demo
targets rather than production assets. Say the word and I will include them.
```
