# skills
Sharing Claude skills with the community.
--
name: project-wiki
version: 2.0

# Project Wiki v2 · Self-Checking Knowledge Base for Claude Projects

## What changed from v1

v1 put all integrity work at write time ("update the wiki at the end of any session that changed source"). Write time is the least reliable moment: there is no end-of-conversation hook, so the update only happened when the user prompted. Wikis drifted silently across many sessions.

v2 moves integrity to **read time**, the one trigger that fires automatically every session (the user always sends a first message, and the protocol is read-wiki-first). The check is **deterministic** (manifest + script), **tiered** (quiet when healthy, loud only on structural breaks), and **debt-bounded** (forced consolidation before drift can compound). Capture is decoupled from reconciliation: a cheap one-line pending log records what changed, and a periodic consolidation applies it.

There is no background process between chats. "Automatic" here means: bound to the read-time trigger that always fires, so drift surfaces on the next first message without the user asking.

## Purpose

Reduce token waste and keep a consistent mental model across conversations by keeping a compiled wiki between Claude and raw source files. Read lightweight wiki files first, pull source only when needed, and never let the wiki drift silently out of sync with the project.

## File types in a wiki'd project

Distinguish three kinds of file. The health check treats them differently.

| Type | What it is | Examples |
|---|---|---|
| **Wiki files** (exactly 5) | The orientation layer. Always read first. | `_index.md`, `_concepts.md`, `[domain]-tracker.md`, `[entities].md`, `continuity-tracker.md` |
| **Compiled-source files** | Long-form derived content the wiki summarizes but does not replace. Cataloged in `_index`, not part of the 5. | a signal-library narrative, a metrics narrative |
| **Raw source files** | The primary artifacts (docs, decks, workbooks, HTML). | `.docx`, `.pptx`, `.xlsx`, `.html`, images |

## The 5 Wiki Files

Name them for the domain; the roles are fixed.

| File | Role |
|---|---|
| `_index.md` | Master file registry, topic lookup, **and the Wiki State block** (see below). |
| `_concepts.md` | Concept registry: every important term, entity, or structural element, grouped and cross-referenced. |
| `[domain]-tracker.md` | Domain tracker: items with consistent attributes in table form. |
| `[entities].md` | Quick-reference profiles for people, systems, companies, characters. |
| `continuity-tracker.md` | Locked decisions, resolved and open conflicts, cross-file threads, **the pending-sync log**, and the session log. |

---

## The Wiki State Block (new in v2)

`_index.md` carries one fenced, machine-parseable block. This is the source of truth the health check diffs against. Keep it current as part of every consolidation.

````
```wiki-state
last_sync: YYYY-MM-DD
wiki_files: _index.md, _concepts.md, [domain]-tracker.md, [entities].md, continuity-tracker.md
compiled_source: file-a.md, file-b.md
expected_project_files: <full list of every file that should be in project knowledge>
invariants: key=value; key=value      # e.g. signal_classes=263; org_metrics=153
pending_count: 0
```
````

- **expected_project_files** is the manifest. The check diffs the actual project directory against it. A removed file (like a vanished `entities.md`) or an added-but-uncataloged file is detected deterministically.
- **invariants** are counts or facts that must reconcile to a source of truth (usually workbook IDs). The check recomputes them when the source is present.
- **pending_count** mirrors the pending-sync log length. Drives the sync-debt threshold.

---

## Session-Start Health Check (new in v2 · runs every session)

This is the first action in any wiki'd project, before any requested work. It replaces "review the wiki when asked" with "the wiki reports its own health on entry."

### Procedure

1. Read `_index.md` and parse the Wiki State block.
2. Run the audit (script below, or its logic by hand if code execution is unavailable):
   - **File diff:** actual project files vs `expected_project_files`. Flag additions and removals.
   - **Wiki completeness:** all 5 wiki files present.
   - **Reference integrity:** every wiki-internal file reference resolves to a file that exists.
   - **Invariant reconciliation:** recompute each invariant from its source when present (e.g. count IDs in the workbook) and compare.
   - **Sync debt:** read `pending_count` and `last_sync`.
3. Emit one verdict and act per tier.

### Verdict tiers

| Verdict | Meaning | Action |
|---|---|---|
| **GREEN** | In sync. No file diff, all wiki files present, references resolve, invariants match, pending_count 0. | One quiet line at most ("Wiki in sync, last reviewed [date]"), then proceed with the user's request. Do not nag. |
| **YELLOW** | Minor drift. Uncataloged files, a nonzero pending_count below threshold, or an invariant off by a small amount. | One line naming the drift, offer to reconcile, then proceed with the request unless the user redirects. |
| **RED** | Structural break. A wiki file or a referenced file is missing, or an invariant is wrong in a way that is leaking into work. | Flag prominently before proceeding. Orientation is compromised; recommend fixing first. |

### Quiet-when-healthy rule

The check must be cheap and silent on GREEN. The user should feel it only when it matters. Loudness scales with severity, never with routine.

---

## Pending-Sync Log (new in v2 · the cheap capture)

Lives as an append-only section in `continuity-tracker.md`. The reason drift compounds is that each session's changes are not recorded until a full refresh. Recording a delta is far cheaper than reconciling the wiki, so it actually happens.

### When to append

At the end of any session that did any of the following (mechanical conditions, not judgment):

- Created, modified, or removed a source or compiled-source file.
- Wrote a deliverable to outputs.
- Made or reversed a LOCKED decision or resolved a CONFLICT.
- Changed a count, name, or invariant.

### Format

```
## Pending Sync (un-applied)
- [YYYY-MM-DD] <what changed> → touches: <wiki files> [outputs: <files needing upload>]
```

One line per change. Increment `pending_count` in the Wiki State block.

---

## Sync-Debt Threshold (new in v2 · the compounding guard)

The guarantee that many sessions never pile up again.

- At session-start health check, if `pending_count >= 3` OR the verdict is RED, **proactively recommend a consolidation** before doing the requested work. State it once, let the user decide.
- A consolidation applies the pending log to the wiki, reconciles the manifest and invariants, runs the de-dup pass, resets `pending_count` to 0, and stamps `last_sync`.
- Tune the threshold to the project's velocity. Three is a sensible default; high-velocity projects may want two.

---

## Outputs-to-Project Gap (new in v2 · the real leak)

In most projects, deliverables are written to an outputs area and must be manually uploaded to project knowledge before the wiki can see them. This is the single biggest silent-drift source.

At the end of any session that wrote to outputs:

1. List the artifacts produced.
2. State plainly that they need uploading to project knowledge for the wiki to track them.
3. Log them in the pending-sync log with an `outputs:` tag.

Never assume an output file is in project knowledge next session. Until uploaded, it is invisible to the check.

---

## Maintenance

### Incremental (every changing session, cheap)

1. Append to the pending-sync log (above). This is the minimum and it is non-optional.
2. If the change is small and self-contained (a single new entity, one renamed item), apply it to the affected wiki file now and clear that pending line.
3. Run the de-dup pass on any file you touched.

### Consolidation / full refresh (periodic or on RED / debt threshold)

Trigger when the user says the wiki is stale, `pending_count` hits the threshold, the health check returns RED, or more than ~30% of source has been rewritten.

1. Re-read all source files.
2. Regenerate the affected wiki files (full-file regeneration over diffs).
3. Apply and clear the pending-sync log.
4. Run the de-dup pass across all 5 files.
5. Update the Wiki State block: refresh `expected_project_files`, reconcile `invariants` from source, set `pending_count: 0`, stamp `last_sync`.
6. Regenerate the relationship map.
7. Add a session-log entry summarizing what changed.

---

## The De-Duplication Pass (unchanged intent, bound to a checklist)

After any update, sweep all 5 files and run this checklist. It is the single most common skipped step; treat it as a gate, not a virtue.

- [ ] Merge duplicate entries; one canonical home, cross-referenced from the others.
- [ ] Reconcile contradictions to the latest version.
- [ ] Collapse overlapping tracker rows.
- [ ] Prune entries whose backing source was removed or rewritten.
- [ ] Confirm every cross-file reference still resolves.

---

## Relationship Map

When the wiki reaches meaningful complexity (15+ entities, multiple trackers, dozens of cross-references), or on request, generate a **mermaid diagram**: nodes are entities/concepts/tracker items, edges are cross-references/ownership/evolution, clusters group by wiki file, color-coded by origin. It surfaces orphaned entries, over-connected nodes, dependency chains, and onboarding structure. Regenerate after a full refresh or major restructuring.

---

## Embedded Audit Script

Drop this in the container and run it at session start (`python3 wiki_healthcheck.py /mnt/project`). It is deterministic and dependency-light. The workbook recount runs only if `openpyxl` and the workbook are present. Adjust paths and invariant keys per project.

```python
#!/usr/bin/env python3
import os, re, sys, glob

PROJ = sys.argv[1] if len(sys.argv) > 1 else "/mnt/project"
WIKI = {"_index.md", "_concepts.md", "continuity-tracker.md"}  # + domain tracker + entities, resolved from state

def read(p):
    try:
        return open(p, encoding="utf-8", errors="ignore").read()
    except Exception:
        return ""

def parse_state(idx_text):
    m = re.search(r"```wiki-state(.*?)```", idx_text, re.S)
    state = {}
    if not m:
        return state
    for line in m.group(1).strip().splitlines():
        if ":" in line:
            k, v = line.split(":", 1)
            state[k.strip()] = v.strip()
    return state

def listfmt(s):
    return [x.strip() for x in re.split(r"[,\n]", s) if x.strip()] if s else []

idx = read(os.path.join(PROJ, "_index.md"))
state = parse_state(idx)
actual = {os.path.basename(p) for p in glob.glob(os.path.join(PROJ, "*"))}
expected = set(listfmt(state.get("expected_project_files", "")))
wiki_files = set(listfmt(state.get("wiki_files", "")))

problems_red, problems_yellow = [], []

# Wiki completeness
missing_wiki = sorted(f for f in wiki_files if f not in actual)
if missing_wiki:
    problems_red.append(f"Missing wiki file(s): {', '.join(missing_wiki)}")

# File diff
removed = sorted(expected - actual)
added = sorted(actual - expected - wiki_files)
if removed:
    problems_red.append(f"Manifest files absent from project: {', '.join(removed)}")
if added:
    problems_yellow.append(f"Uncataloged file(s) in project: {', '.join(added)}")

# Reference integrity: every *.md filename referenced in a wiki file must exist
for wf in wiki_files:
    txt = read(os.path.join(PROJ, wf))
    for ref in set(re.findall(r"`([\w\-\.]+\.md)`", txt)):
        if ref in wiki_files and ref not in actual:
            problems_red.append(f"{wf} references missing wiki file {ref}")

# Invariant reconciliation (signal/metric ID recount if workbook present)
inv = {}
for kv in re.split(r"[;\n]", state.get("invariants", "")):
    if "=" in kv:
        k, v = kv.split("=", 1)
        inv[k.strip()] = v.strip()
try:
    import openpyxl
    for fn, sheet, key in [
        ("GTM_OS_Signal_Library_Metrics.xlsx", "All Signals", "signal_classes"),
        ("Org_Capability_Metric_Library_v2_0.xlsx", "All Metrics", "org_metrics"),
    ]:
        p = os.path.join(PROJ, fn)
        if key in inv and os.path.exists(p):
            wb = openpyxl.load_workbook(p, data_only=True)
            if sheet in wb.sheetnames:
                rows = list(wb[sheet].iter_rows(values_only=True))
                ids = [r[0] for r in rows[1:] if r and r[0] and re.match(r"^[A-Z]+-L?\d", str(r[0]))]
                real = len(ids)
                if str(real) != str(inv[key]):
                    problems_red.append(f"Invariant {key}={inv[key]} but source has {real}")
except ImportError:
    problems_yellow.append("openpyxl absent: invariants not recomputed from source")

# Sync debt
try:
    pending = int(state.get("pending_count", "0"))
except ValueError:
    pending = 0
if pending >= 3:
    problems_yellow.append(f"Sync debt: {pending} pending entries (>= threshold) → recommend consolidation")

verdict = "RED" if problems_red else ("YELLOW" if problems_yellow else "GREEN")
print(f"WIKI HEALTH: {verdict}  (last_sync {state.get('last_sync','?')}, pending {state.get('pending_count','?')})")
for p in problems_red:
    print("  RED   ·", p)
for p in problems_yellow:
    print("  YELLOW·", p)
if verdict == "GREEN":
    print("  In sync. Proceed.")
```

---

## Domain Adaptation

| Domain | Tracker | Entities | Continuity focus |
|---|---|---|---|
| TV/Film | episode-tracker | characters | timeline, arcs, planted threads |
| Software | feature-tracker | architecture-entities | API contracts, migration state |
| Research | paper-tracker | concept-definitions | findings chain, methodology |
| Business | workstream-tracker | stakeholders | decisions, dependencies, blockers |
| Book/Novel | chapter-tracker | characters | timeline, foreshadowing |

Pattern: index → concepts → domain tracker → entities → continuity.

---

## Constraints

- Wiki files are orientation aids, not replacements for source. Verify against source before changing source.
- The session-start health check is non-optional in a wiki'd project. It is the first action, before requested work.
- The de-dup pass is a gate, not a virtue. Skipping it is the top failure mode.
- The pending-sync append is the minimum capture every changing session. Skipping it is the second failure mode.
- Quiet when GREEN. Loudness scales with severity, never with routine.
- No background execution exists. Bind integrity to the read-time trigger that always fires.

---

## Changelog

- **v2.0** · Read-time health check with GREEN/YELLOW/RED tiers; machine-checkable Wiki State block (manifest + invariants); embedded audit script; pending-sync log for cheap per-session capture; sync-debt threshold forcing bounded consolidation; outputs-to-project gap handling; reference-integrity and invariant-reconciliation checks; file-type distinction (wiki vs compiled-source vs raw). De-dup and incremental update converted from soft guidance to checklists and mechanical triggers.
- **v1.0** · Five-file structure, read-wiki-first protocol, incremental and full-refresh maintenance, de-dup pass, relationship map, compounding-ROI model.
