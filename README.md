Project Wiki
A self-checking knowledge base for Claude Projects. Version 2.4.

Project Wiki is a Claude Skill that builds and maintains a lightweight, compiled index over the files in a Claude Project. It keeps a consistent mental model across separate conversations, cuts token waste by reading small orientation files before pulling heavy source, and reports its own health at the start of every session so the index never silently drifts out of sync.
It is designed around one hard constraint: project files are read-only. The only way to change one is to delete it and re-upload by hand, and a re-upload does not always re-index immediately. So every wiki update has a real manual cost, and the entire design minimizes how often, and how many, files must be swapped.

Why this exists
Without a compiled index, a fresh conversation starts cold. Claude either re-reads every source file (expensive, and not always possible within context) or works from a partial picture. A wiki fixes that, but a naive wiki has the opposite failure mode: it goes stale the moment source changes and nobody notices until an answer is wrong.

Project Wiki moves the integrity check to read time. Instead of “remember to review the wiki,” the wiki reports its own health the instant a session begins. A healthy project produces almost no output. A drifting one says exactly what is off and how to fix it.

What is new in 2.4
The headline change makes the skill safe to distribute. Invariant reconciliation (the part that verifies a count in the wiki still matches its source of truth) used to be hardcoded into the audit script, which meant the script carried the original author’s private filenames and structure. In 2.4 those specs are declarative: each invariant declares its own source, sheet, column, and match pattern as data inside the project’s state file. The shipped script holds zero project-specific detail and works for any project unchanged.

Also new:
	•	CSV and TSV reconciliation with no extra dependency (spreadsheet sources still use openpyxl when present).
	•	Duplicate state-block detection. If more than one state block exists, the check prefers wiki_state.md and warns instead of silently binding to the wrong file.
	•	Unreadable-vs-missing distinction, so a broken wiki file can no longer pass as healthy.
	•	pending_count cross-check against the actual length of the pending log.
	•	Actionable output, including the consolidation steps to run when sync debt is detected.
	•	--json output for piping the verdict into other automation.
	
Installation
The skill is distributed as a single archive, 2.4 wiki.zip. Download and unzip it, then pick the path that matches where you run Claude. The archive contains SKILL.md, the two scripts, and the packaged project-wiki.skill.
	1.	Upload the packaged skill. Install project-wiki.skill (inside the unzipped archive) through the Skills interface in your Claude surface (see the Anthropic docs for the exact upload step on your plan). Custom skills are account-level, so once installed it applies to every project.
	2.	Filesystem (Claude Code). Put the unzipped SKILL.md and both scripts in a folder named project-wiki/ where Claude Code discovers skills. They load together.
	3.	Single project, no skill. If you only want it in one project, add SKILL.md and the two scripts to that project’s knowledge directly.
This is an update to an existing skill, not a new one. The name and directory are unchanged, so installing over a prior version replaces it in place rather than creating a parallel entry.
How it works

Wiki layout
A wiki is a five-file stable corpus plus one small operational ledger. The stable files move only when project substance moves. Routine churn is routed to the ledger, so most upkeep is a one-file swap.
