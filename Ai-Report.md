---
name: report
description: Generate a daily AI productivity report from all handoffs in today's dated directory. Synthesizes work accomplished, estimates time savings across multiple angles (execution speed, rabbit hole avoidance, scripted steps, documentation), and produces a leadership-facing document structured for downstream Jira ticket generation. Use when the user asks to "generate the daily report", "create today's report", "/report", or wants a summary of the day's AI-assisted work.
---

# Daily Productivity Report

Generate a **leadership-facing** AI productivity report from today's handoff documents. The audience is Astrion program managers and leadership — not engineers. Write in plain language. Avoid acronyms without expansion on first use. The report will also be consumed by a Claude instance to generate discrete Jira tickets, so each work item must be clearly delineated.

## Step 1 — Locate today's handoffs

1. Determine today's dated directory: `date +"%d%b%y" | tr '[:lower:]' '[:upper:]'` → e.g. `24JUN26`.
2. List all files in `/opt/core/handoff/DATEDIR/` that match `[0-9][0-9]-*.md` (numbered handoffs only — exclude `report.md` if it already exists).
3. Read every handoff file. Do not skip any.
4. If the directory doesn't exist or contains no numbered handoffs, tell the user and stop — there is nothing to report.

## Step 2 — Extract and synthesize

From each handoff, extract:
- **What was accomplished** (sections 1–2): objective, scope, done items.
- **Design decisions made** (section 3): significant choices and rationale.
- **What's still open** (section 5): blockers, pending decisions.
- **Time & efficiency data** (section 8): session duration, manual baseline, specific time-savers noted.

If section 8 is absent or sparse in a handoff, apply the heuristic table below to estimate based on task type. Label heuristic estimates as `[est.]` in the report. Label explicit estimates from section 8 as `[reported]`.

### Time estimation heuristics (manual baseline, experienced engineer, no AI)

| Task category | Manual time range | Notes |
|---|---|---|
| OCP/SNO install config authoring (agent-based, install-config, agent-config) | 4–8 hrs | YAML authoring, cross-referencing docs, validation iteration |
| Firewall / network troubleshooting | 2–4 hrs | Research, rule identification, test-and-iterate |
| Windows Server install + DC promotion (OCP Virt) | 3–5 hrs | ISO prep, VM config, install, post-config, DNS verification |
| Certificate / PKI setup | 2–3 hrs | Procedure research, config generation, chain validation |
| oc-mirror / disconnected registry config | 2–4 hrs | ImageSet authoring, mirror run, troubleshooting |
| Bash / Python script authoring | 1–3 hrs | Varies by complexity |
| Ansible role / playbook authoring | 2–4 hrs | Per role, includes testing |
| Architecture / design decision (research + evaluate tradeoffs) | 1–3 hrs | |
| Technical runbook / documentation writing | 1–2 hrs | Per document |
| Troubleshooting a failing service (logs, config, restart cycle) | 1–3 hrs | High variance |
| Identity / SSO configuration (Keycloak, AD integration) | 3–6 hrs | |
| GitLab or Jira integration setup | 2–4 hrs | |

**AI-assisted time multiplier:** Divide by 3–5× for config/scripting tasks where Claude generates drafts and iterates. Divide by 2–3× for troubleshooting where Claude narrows the search space. For design review with grill-me, estimate 1–2 hrs saved per session (rabbit holes avoided, decisions sharpened before costly implementation).

## Step 3 — Build the report

Use this structure exactly. The Jira-generation Claude instance will parse section 5 by work item headers.

```markdown
# Daily AI Productivity Report — <DDMMMYY>
**Project:** CORE Lab — bootstrap01 / core01 SNO Deployment
**Prepared for:** Astrion Leadership / Program Management
**Reporting period:** <full date, e.g. 24 June 2026>

---

## Executive Summary

[2–4 sentences. What was accomplished today at a plain-language level. Headline time savings figure. No jargon — spell out all acronyms.]

---

## Work Accomplished Today

[One paragraph or tight bullets per handoff, summarizing what was done and what state things are in now. Plain language. Reference the handoff file that covers each item: e.g. (ref: 01-section5-dc01-windows-install.md).]

---

## Time Efficiency Analysis

### Execution Speed
[For each major task: what was done, estimated manual hours, actual AI-assisted hours (or session duration), and the delta. Cite [est.] or [reported] per figure. Use a table.]

| Task | Manual estimate | AI-assisted | Time saved | Confidence |
|---|---|---|---|---|
| ... | ... | ... | ... | [est.] or [reported] |

**Subtotal — execution speed:** X hours saved

### Rabbit Hole Avoidance
[Describe specific design decisions, architecture reviews, or /grill-me sessions that occurred. For each: what was the decision point, what alternative paths existed, and why following those paths would have cost time. Estimate cost of each avoided detour.]

**Subtotal — rabbit hole avoidance:** X hours saved

### Scripted & Automated Steps
[List scripts, configs, or automation artifacts generated by Claude this session. For each: what it was, what manual effort it replaced, and the time delta.]

**Subtotal — scripted/automated steps:** X hours saved

### Documentation & Runbook Generation
[List handoff documents, runbooks, configs, or other documentation produced with Claude's assistance. Estimate how long each would have taken to write manually.]

**Subtotal — documentation:** X hours saved

---

## Total Estimated Time Savings

| Category | Hours saved |
|---|---|
| Execution speed | X |
| Rabbit hole avoidance | X |
| Scripted / automated steps | X |
| Documentation generation | X |
| **Total** | **X** |

**Confidence level:** [High / Medium / Low — with a one-line explanation of what drives uncertainty]

---

## Open Items & Blockers

[Anything from section 5 of any handoff that is blocking progress or awaiting a decision. Enough context for leadership to understand the impact and who needs to act.]

---

## Section 5 — Discrete Work Items (Jira-Ready)

<!-- This section is parsed by the Jira generation workflow. Each ### header becomes a ticket title. -->

### <Work item title — imperative, concise>
- **Description:** [Plain-language description of the work done or the capability delivered.]
- **Outcome:** [What is now true / working / available as a result.]
- **Time saved:** X hours ([est.] or [reported])
- **Efficiency category:** [Execution Speed | Rabbit Hole Avoidance | Automation | Documentation]
- **Evidence:** [Handoff file reference, specific artifacts produced, or gates passed.]
- **Status:** [Complete | In Progress | Blocked]

### <Next work item>
[...repeat for each discrete item...]

---

## Methodology

Time estimates are derived from a combination of:
- **[reported]** — explicit time notes captured in handoff section 8 by the engineer during the session.
- **[est.]** — heuristic baselines for task categories applied where section 8 data is absent, calibrated to experienced RHEL/OpenShift engineers without AI tooling.

All estimates represent best-effort approximations. Actual savings will vary.

---
*Report generated by Claude Code. Source handoffs: /opt/core/handoff/<DATEDIR>/*
```

## Step 4 — Save the report

1. Write the report to `/tmp/report-draft.md`.
2. Save it:
   ```bash
   DATEDIR=$(date +"%d%b%y" | tr '[:lower:]' '[:upper:]')
   sudo install -d -m 0755 /opt/core/handoff/${DATEDIR}
   sudo install -o root -g root -m 0644 /tmp/report-draft.md /opt/core/handoff/${DATEDIR}/report.md
   rm /tmp/report-draft.md
   ```
3. If `report.md` already exists in the dir, overwrite it — this is idempotent by design.

## Finishing

Tell the user the saved path, the total estimated hours saved, and the count of Jira-ready work items in section 5.
