# Visual templates

Inline HTML widgets for rendering Safeguard data with color, cards, and badges directly in the conversation. Markdown cannot produce a background colour or a badge, so anything that needs visual weight has to be a rendered widget.

These require a visual rendering tool in the session. **If none is available, fall back to the markdown templates in `chat-templates.md`.** Never describe a visual you cannot actually render, and never apologise for its absence. Just render the markdown version.

## Contents

1. [When a widget beats markdown](#1-when-a-widget-beats-markdown)
2. [The severity scale](#2-the-severity-scale)
3. [Posture dashboard](#3-posture-dashboard)
4. [Vulnerability card](#4-vulnerability-card)
5. [Fix queue](#5-fix-queue)
6. [Signal chips](#6-signal-chips)
7. [Rules that keep it native](#7-rules-that-keep-it-native)

---

## 1. When a widget beats markdown

| Situation | Use |
|---|---|
| Posture overview someone will look at and absorb | Widget |
| A single vulnerability being explained | Widget |
| Ranked fix queue with signals per row | Widget |
| A list the user will copy into a ticket | Markdown or fenced block |
| Audit evidence tables | Markdown, because it gets pasted into documents |
| Two or three numbers | Neither, just write the sentence |
| Anything over about 12 rows | Markdown table, since widgets should not scroll |

A widget costs more to produce and cannot be copied out as text. Use it when the visual encoding is doing work, not for decoration.

---

## 2. The severity scale

One scale, used everywhere, never redefined per response. Put it in a compact style block at the top of every widget.

```html
<style>
.sev{display:inline-block;font-size:12px;font-weight:500;padding:2px 8px;border-radius:var(--radius);min-width:36px;text-align:center}
.sev-c{background:#FCEBEB;color:#791F1F}
.sev-h{background:#FAECE7;color:#712B13}
.sev-m{background:#FAEEDA;color:#633806}
.sev-l{background:#F1EFE8;color:#444441}
@media (prefers-color-scheme:dark){
.sev-c{background:#501313;color:#F7C1C1}.sev-h{background:#4A1B0C;color:#F5C4B3}
.sev-m{background:#412402;color:#FAC775}.sev-l{background:#2C2C2A;color:#D3D1C7}}
</style>
```

Critical is red, high is coral, medium is amber, **low is neutral grey and never green**. Green reads as "resolved" or "safe", and a low severity finding is neither.

For solid fills such as a distribution bar, use `#d03b3b` critical, `#ec835a` high, `#fab219` medium, `#b4b2a9` low.

**Colour never carries meaning alone.** Every severity badge shows its number or label. A colourblind reader and a screen reader user both have to get the same information.

---

## 3. Posture dashboard

Metric cards, then a distribution bar with a legend, then project rows, then the single most important action.

```html
<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(140px, 1fr)); gap: 12px; margin-bottom: 1.5rem;">
  <div style="background: var(--surface-1); border-radius: var(--radius); padding: 1rem;">
    <p style="font-size: 13px; color: var(--text-secondary); margin: 0 0 4px;">{label}</p>
    <p style="font-size: 24px; font-weight: 500; margin: 0;">{value}</p>
  </div>
</div>
```

Colour a metric value only when it is the alarming one, using `color: var(--text-danger)`. If every number is coloured, none of them stand out.

Distribution bar with a 2px surface gap between segments:

```html
<div style="display: flex; height: 8px; border-radius: 4px; overflow: hidden; margin-bottom: 8px;">
  <div style="width: {pct}%; background: #d03b3b;"></div>
  <div style="width: 2px; background: var(--surface-0);"></div>
  <div style="width: {pct}%; background: #ec835a;"></div>
</div>
```

Project rows go in one card with hairline dividers, using flex rather than a table so the severity column widths stay fixed:

```html
<div style="background: var(--surface-2); border: 0.5px solid var(--border); border-radius: 12px; overflow: hidden;">
  <div style="display: flex; align-items: center; padding: 12px 1.25rem; border-bottom: 0.5px solid var(--border);">
    <span style="flex: 1; font-size: 14px; font-weight: 500;">{project}</span>
    <span style="width: 46px; text-align: center;"><span class="sev sev-c">{n}</span></span>
    <span style="width: 46px; text-align: center;"><span class="sev sev-h">{n}</span></span>
  </div>
</div>
```

Cap at about eight rows. Beyond that the widget stops being scannable and a markdown table serves better.

---

## 4. Vulnerability card

One bounded object, one raised card. Severity badge and identifier on the top line, signal chips below, then two or three sentences of substance.

```html
<div style="background: var(--surface-2); border: 0.5px solid var(--border); border-radius: 12px; padding: 1rem 1.25rem;">
  <div style="display: flex; align-items: center; gap: 10px; margin-bottom: 10px;">
    <span class="sev sev-h">High</span>
    <span style="font-size: 15px; font-weight: 500;">{CVE ID}</span>
    <span style="font-size: 13px; color: var(--text-secondary); margin-left: auto; font-family: var(--font-mono);">CVSS {score}</span>
  </div>
  <div style="display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 12px;">{chips}</div>
  <p style="font-size: 14px; color: var(--text-secondary); line-height: 1.6; margin: 0;">{what the flaw is and what an attacker gets}</p>
</div>
```

---

## 5. Fix queue

One card per component upgrade, never per CVE. The version transition goes in `var(--font-mono)` so it reads as code.

```html
<div style="display: flex; align-items: baseline; justify-content: space-between; gap: 12px; margin-bottom: 10px;">
  <p style="font-size: 15px; font-weight: 500; margin: 0;">Fix first: {component}</p>
  <span style="font-size: 13px; color: var(--text-secondary); font-family: var(--font-mono);">{current} to {fix version}</span>
</div>
```

Close with one action button that hands the next step back to Claude:

```html
<button onclick="sendPrompt('Show me the full remediation plan for {component} across all affected projects')">Build the remediation plan ↗</button>
```

Use `sendPrompt` only where Claude needs to think. Filtering and sorting belong in JavaScript.

---

## 6. Signal chips

The reachability, KEV, and blast radius signals are what make Safeguard data worth reading. Chips make them scannable.

```html
<span style="display:inline-flex;align-items:center;gap:4px;font-size:12px;padding:3px 9px;border-radius:var(--radius);background:var(--surface-1);color:var(--text-secondary)">
  <i class="ti ti-route" style="font-size:14px" aria-hidden="true"></i>Reachable
</span>
```

| Signal | Icon | Label |
|---|---|---|
| Reachable in call path | `ti-route` | Reachable |
| Known exploited | `ti-flame` | In KEV |
| Fix available | `ti-arrow-up-circle` | Fix available |
| No fix | `ti-ban` | No fix |
| Direct dependency | `ti-package` | Direct |
| Blast radius | `ti-git-branch` | {n} projects |
| Age | `ti-clock` | Open {n} days |
| Policy blocking | `ti-shield-x` | Blocks build |

Give a KEV chip a danger tint (`background: var(--bg-danger); color: var(--text-danger)`). Leave every other chip neutral. Two emphasis levels only, or the emphasis stops meaning anything.

Outline icons only. Never a `-filled` suffix, and never emoji.

---

## 7. Rules that keep it native

**No Safeguard brand colours in chat widgets.** Iris and the marketing palette belong on safeguard.sh, in the product UI, and in exported documents. A widget in someone's conversation should look like it belongs to the conversation, not like an advertisement wedged into it. Use the host CSS variables and the severity scale above. This is a deliberate choice, not an oversight.

**Transparent outer container.** Never set a background or a dark panel on the outermost element. The host provides the page background, and a hardcoded one will be wrong in the other colour mode.

**Every colour works in both modes.** Text uses `var(--text-primary)` and `var(--text-secondary)`. Hardcoded greys are invisible against a near-black background. The test: if the page were near-black, would every element still be readable.

**Two font weights, 400 and 500.** Nothing heavier. Sentence case throughout, including labels and headings.

**Flat.** No gradients, shadows, glows, or rounded corners on single-sided borders.

**Start with a screen reader summary**, since a widget is otherwise opaque to one:

```html
<h2 class="sr-only">{one sentence stating the headline finding}</h2>
```

with `.sr-only{position:absolute;width:1px;height:1px;overflow:hidden;clip:rect(0,0,0,0)}`.

**The prose goes outside the widget.** Explanation, caveats, staleness warnings, and next steps are written as normal response text. The widget holds the visual only. A staleness warning buried inside a graphic is a staleness warning nobody reads.
