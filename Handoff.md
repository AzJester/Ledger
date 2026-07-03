---
name: handoff
description: Generate a self-contained handoff document that captures the current work session so it can be resumed later by the user, a teammate, or a fresh Claude instance with no prior context. Use when the user asks to "hand off", "write a handoff", "snapshot this session", "capture context before I stop", or wants a resumable record of in-progress work. Also triggers automatically at crucial checkpoints and on unrelated task transitions. The document distills the conversation into objective current status (with concrete artifacts — commit SHAs, file paths, command output), the design and the reasoning behind key decisions, a step-by-step runbook for what's next, open items and pending decisions, and carry-forward guardrails — deliberately excluding secrets and sensitive values. Saved to /opt/core/handoff/<DDMMMYY>/ with a sequential filename.
---

# Handoff

Produce a **self-contained** handoff document: enough that someone with **zero prior context** — the user after a break, a teammate, or a fresh Claude instance — can pick the work up cold and continue without re-deriving anything. The reader cannot see this conversation. Anything not written down is lost.

## When to generate automatically (without being asked)

Two conditions trigger an automatic handoff. Do this proactively — don't wait for the user to ask.

### 1. Crucial checkpoint reached

Generate a handoff when a meaningful, self-contained unit of work completes and is verified. Examples:
- A service, component, or deployment step is fully applied and confirmed working (e.g., Guacamole pods running and accessible, a DNS zone propagated, a certificate issued).
- A significant configuration block is written, applied, and tested.
- A critical bug is fixed and the fix is verified in the running system.
- A migration or data operation completes successfully.
- A section or stage of multi-section work closes (e.g., "Section 1 done, moving to Section 2").

Do **not** generate at minor waypoints: a single file edit, a failed attempt being retried, or an intermediate step that only makes sense once the next step is done. The test: would a stranger picking this up cold have a stable, runnable system to build on? If yes, generate.

### 2. Unrelated task transition

Generate a handoff when the conversation pivots to a topic that is not a natural continuation of the current task. The signal is a domain or system boundary — not just a new subtask. Examples of transitions that warrant a handoff:
- Guacamole deployment → Keycloak configuration
- DNS troubleshooting → Harbor registry setup
- Writing a runbook → debugging a live outage in a different service

Examples that do **not** warrant a handoff (same task, different step):
- Configuring Guacamole networking → configuring Guacamole TLS (same system)
- Writing a deployment manifest → applying it and checking pods (same task)

When you detect a transition, generate the handoff for the **prior** task before beginning the new one. Tell the user: "I'm generating a handoff for [prior task] before we switch to [new task]." Then proceed.

## Operating principle

Write for a competent stranger, not for yourself. Every claim must stand on its own:
- **Objective over narrative.** State what *is true now*, not the story of how you got there. "Auth middleware is wired into `server/app.ts:42` and passing its tests" — not "then I tried X, it failed, so I switched to Y."
- **Concrete over vague.** Cite artifacts: commit SHAs, branch names, file paths with line numbers, exact command invocations, real output snippets, test names, PR/issue numbers, config keys. "Done" means *verified* — say how it was verified.
- **Current over historical.** If a decision was reversed, record only the final state plus a one-line note that the alternative was rejected and why. Don't make the reader replay dead ends.
- **Honest about uncertainty.** Mark what is unverified, assumed, or flaky. A false "this works" is worse than a flagged "this is untested."

## Before writing — gather ground truth

Don't reconstruct status from memory; the conversation may be stale or summarized. Verify against the actual workspace. Run what's relevant (read-only):
- `git status`, `git log --oneline -15`, `git branch --show-current`, `git diff --stat` — real commit SHAs, branch, uncommitted work.
- Existence/state of key files you reference (Read or `ls`), and the latest test/build/lint result if a command can produce it quickly.
- Any task/PR/issue identifiers and their current state.

Prefer captured output over recollection. If you cannot verify a claim, label it `[unverified]` rather than dropping or asserting it.

## Secrets — hard exclusion

**Never** write secrets or sensitive values into the document. This includes: passwords, API keys, tokens, pull secrets, registry/AD/SSO/DB credentials, CA private keys, kubeconfig/kubeadmin contents, SSH private keys, and any specific value the project marks sensitive. Refer to them by **location and purpose only** — e.g. "registry credentials at `/opt/core-secrets/registry-creds.json` (root, 0600)", never the value. When in doubt, name the pointer, not the payload. (On this host the rule is absolute: secrets never leave bootstrap01.)

If the project's `CLAUDE.md` or equivalent defines its own secret-handling or data-egress rules, honor them in addition to this baseline.

## Document structure

Use this skeleton. Drop a section only if it's genuinely empty (say "None" rather than omitting where the absence is meaningful). Keep prose tight; favor scannable lists and tables.

```markdown
# Handoff: <concise title of the work> — <DDMMMYY>

## 1. Objective & scope
What this work is trying to achieve, and the boundary of what's in/out of scope.
One short paragraph. Link the driving ticket/issue/spec if one exists.

## 2. Current status
Objective snapshot of where things stand *right now*.
- **Done & verified:** bullets with concrete artifacts (commit SHA, file:line, how verified).
- **In progress:** what's partially done; exactly where it stands; what's left to finish it.
- **Branch / working tree:** branch name, whether it's pushed, uncommitted changes, key commit SHAs.
- **Environment / build state:** last known test/build/lint result with the command and outcome.

## 3. Design & key decisions
The architecture as it stands, and *why* — the reasoning a stranger would otherwise have to reverse-engineer.
For each significant decision: what was chosen, what was rejected, and the rationale/trade-off.
Use a table when there are several: | Decision | Choice | Rejected alternative | Why |.

## 4. Runbook — what's next
Ordered, executable steps to continue. Each step concrete enough to act on without guessing:
exact files to touch, commands to run, expected result, and the verification/gate that confirms it.
Number them. Note dependencies/ordering. Call out any blocking gate before later steps.

## 5. Open items & pending decisions
Unresolved questions, decisions awaiting the user or a teammate, known bugs, TODOs, risks.
For each: what's blocked on it and any leaning/recommendation. Flag anything `[unverified]`.

## 6. Carry-forward guardrails & constraints
Non-negotiable rules to keep honoring: pinned versions, verification gates, firewall/egress
boundaries, secret-handling rules, conventions, "do not touch" areas. Reference where they're
defined (e.g. project CLAUDE.md / REFERENCE.md) rather than restating values.

## 7. Key references (no secrets)
Files, dirs, dashboards, tickets, docs the next person needs. Pointers and purposes only —
for sensitive files give path + purpose + permissions, never contents.

## 8. Time & efficiency notes
**Fill this in — it feeds the daily /report.** Estimate honestly; ranges are fine.

- **Session duration (approx):** e.g. "~2.5 hours"
- **Manual baseline estimate:** How long would a skilled engineer have taken to accomplish this
  same scope without AI assistance? Think: research, config authoring, trial-and-error debugging,
  doc writing. e.g. "4–6 hours for an experienced RHEL/OCP admin"
- **Key time-savers this session:** Bullet each notable efficiency gain:
  - Config/script generation (what, and rough time saved)
  - Design review / grill-me (what decision was sharpened, what rabbit hole was avoided)
  - Automated troubleshooting (what would have been manual trial-and-error)
  - Documentation generated (what docs, rough time saved)
  - Any other notable efficiency
- **What slowed things down (if anything):** Blockers, dead ends, back-and-forth that added time.
  Honest accounting makes the estimates credible.
```

## Filename & save location

**Save location: `/opt/core/handoff/DDMMMYY/`** where `DDMMMYY` is today's date (e.g. `24JUN26`). Every handoff `.md` goes here. Do not write handoffs to any other path.

1. Determine today's dated directory name: run `date +"%d%b%y" | tr '[:lower:]' '[:upper:]'` to get e.g. `24JUN26`. The full path is `/opt/core/handoff/24JUN26/`.

2. Determine the next sequential number: count existing sequentially-named handoffs in the dir and increment.
   ```bash
   DATEDIR=$(date +"%d%b%y" | tr '[:lower:]' '[:upper:]')
   COUNT=$(ls /opt/core/handoff/${DATEDIR}/[0-9][0-9]-*.md 2>/dev/null | wc -l)
   NEXT=$(printf "%02d" $((COUNT + 1)))
   ```
   The filename is `NN-short-descriptive-slug.md` — e.g. `03-section5-dc01-dns-fix.md`. Use a slug describing the work, not the date (the date is already in the directory name).

3. Create the dated dir if it doesn't exist, then save:
   - Write the document to a temp file with the Write tool (e.g. `/tmp/handoff-draft.md`).
   - `sudo install -d -m 0755 /opt/core/handoff/${DATEDIR}`
   - `sudo install -o root -g root -m 0644 /tmp/handoff-draft.md /opt/core/handoff/${DATEDIR}/${NEXT}-<slug>.md`
   - Remove the temp file.
   - `install` is used (not `tee`/`cat`/`echo`) so the document body never passes through the shell.

4. **Post-save secret scan (do this every time):** grep the saved file for obvious credential patterns (`password`, `BEGIN .*PRIVATE KEY`, `kubeadmin`, `token=`, `api[_-]?key`) and confirm any hits are pointers/filenames, not values. If a real secret slipped in, delete the file and regenerate.

## SBD / ASBD update (do this after saving the handoff, before Kanban)

After saving each handoff, update `/opt/core/docs/SBD-core.md` and `/opt/core/docs/ASBD-core.md`
to incorporate the work captured in the handoff. This is mandatory — the SBD/ASBD are the
authoritative living build record that crosses the diode; handoffs alone are session notes.

**What to update (not exhaustive — use judgment):**

1. **Phase status lines** — change `· IN PROGRESS` / `· PLANNED` to `· VALIDATED YYYY-MM-DD`
   and add the gate pass date when a gate completes.

2. **Phase runbook body** — if the handoff records commands or steps that differ from what the SBD
   says (a workaround, a corrected command, a new script), update the SBD to match the actual
   procedure so a rebuild would succeed.

3. **Design decisions** — if a decision was made or changed (e.g., "rejected Valkey, chose Redis"),
   add or update the decision table in the relevant Phase section.

4. **ASBD automation status table** — update the row for the relevant phase/component:
   - Change "not yet" / "manual" → actual automation status
   - Add new rows for components that didn't exist before (new apps, new scripts)
   - Note gate pass dates

5. **oc-mirror / imageset changes** — if images were added or removed from the imageset, or
   parallelism was tuned, update the `--parallel-images` flag and the imageset list in the SBD Phase 7
   pre-work section.

6. **New top-level sections** — for significant new work streams (a new app, a new phase, a new
   infrastructure component), add a new `## Phase N.x — ...` section to the SBD.

**What NOT to update:**
- Session narrative / what slowed things down — that belongs only in the handoff.
- Unverified steps — only write commands that were confirmed working. Mark `[unverified]` if needed.
- Secrets or secret values — same rules as the handoff itself.

**Note on em-dashes:** The ASBD table rows use Unicode em-dash (`—`) in phase labels.
The Edit tool may fail to match these. Use a Python replacement script when editing ASBD table rows:
```bash
python3 << 'EOF'
with open('/opt/core/docs/ASBD-core.md', 'r', encoding='utf-8') as f:
    content = f.read()
content = content.replace('OLD ROW', 'NEW ROW')
with open('/opt/core/docs/ASBD-core.md', 'w', encoding='utf-8') as f:
    f.write(content)
EOF
```

## Kanban update (do this after saving the handoff)

After the handoff is saved, identify newly opened action items from Section 5, present them to the user for selection, then update `/var/www/core-kanban/kanban.json` with only the items the user approves. This keeps the Kanban board at `http://build.bootstrap01.internal/kanban/` current. The JSON is also used to create Jira tickets on the Astrion network.

### Schema

Each ticket has these fields:

```json
{
  "id": "CORE-YYYYMMDD-short-slug",
  "summary": "one-line ticket title (≤255 chars, Jira-compatible)",
  "description": "fuller detail — files, blockers, next action",
  "issuetype": "Task",
  "priority": "High | Medium | Low",
  "labels": ["label1", "label2"],
  "duedate": "YYYY-MM-DD or null",
  "status": "open | done",
  "created": "YYYY-MM-DD (date first seen)",
  "resolved": "YYYY-MM-DD or null",
  "source_handoff": "DDMMMYY/NN-slug.md"
}
```

`id` is the dedup key across sessions. Use `CORE-YYYYMMDD-short-slug` where the date is the `created` date and the slug is a 2–5 word kebab-case summary of the ticket topic. Once assigned, an `id` never changes.

### Reconciliation steps

1. Read `/var/www/core-kanban/kanban.json` (create an empty structure if it doesn't exist).

2. Extract all open action items from **Section 5** of the handoff you just saved.

3. **Scope awareness**: This handoff covers a specific topic (e.g., MEAT/Azure, or on-prem Phase 7, or Keycloak config). Only reconcile tickets whose labels/content fall within that same scope. Do not close tickets from a different domain just because they don't appear in this handoff's Section 5.

4. **Identify newly opened items** — items in Section 5 that do NOT already appear in kanban.json as an "open" ticket (match on `summary` or `id`). Present these as a numbered list to the user and ask which ones to add to the board. Wait for their response before writing anything to kanban.json. Example format:

   > **New action items from this handoff — which should go on the Kanban board?**
   > 1. [High] Resolve Keycloak SAML assertion mismatch — blocks AD SSO login
   > 2. [Medium] Rotate bootstrap01 registry mirror cert before 2026-07-15 expiry
   > 3. [Low] Document manual iPXE fallback procedure in runbook
   >
   > Reply with the numbers you want added (e.g. "1, 3"), "all", or "none".

5. Add only the user-selected items as new tickets with `status: "open"`, a fresh `id`, and today's date as `created`. Items the user skipped are not added.

6. For each existing "open" ticket **within this handoff's scope** that does NOT appear in Section 5 → set `status: "done"` and `resolved` to today's date in `YYYY-MM-DD` format. (This closure is automatic — no prompt needed.)

7. Update `meta.generated` to the current UTC timestamp (ISO 8601). Set `meta.source_handoff` to the full path of the handoff just saved.

8. Write the updated JSON to a temp file, then install:
   ```bash
   sudo install -o root -g root -m 0644 /tmp/kanban-update.json /var/www/core-kanban/kanban.json
   ```
   Remove the temp file after.

9. Tell the user how many tickets are now open and how many were closed by this handoff. One sentence is enough.

### Priority guidance

- **High**: deadline within 7 days, blocks another task, or on the critical path to a hard commitment (award, delivery, compliance gate).
- **Medium**: important but not immediately blocking; should be done within a few weeks.
- **Low**: nice to have, low urgency, or only needed for a later phase.

### Labels guidance

Use short, consistent kebab-case labels. Common ones: `on-prem`, `phase7`, `MEAT`, `azure`, `GCC-High`, `CMMC`, `BD`, `white-paper`, `mirror`, `quay`, `keycloak`, `iron-bank`, `networking`, `IT-request`, `GPU`, `ARO`, `cloud`, `compliance`, `legal`.

## Finishing

After saving the handoff, tell the user:
1. The full path to the handoff file.
2. A 2–3 line summary of what the handoff covers (status headline + immediate next step).

Then present the numbered list of newly opened action items and ask which to add to the Kanban board (per step 4 above). Wait for the user's reply, apply their selection, and report in one line how many tickets are now open and how many were closed this session.

Do not paste the whole document back into the chat.
