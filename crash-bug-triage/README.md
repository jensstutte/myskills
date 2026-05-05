# crash-bug-triage

Slash command that triages Firefox crash bugs to determine whether they are still actionable. Works on a single bug, a list of bug numbers, or every open crash bug in a team's components.

## What it does

Given a bug number, a team name, or explicit `Product::Component` names:

1. Resolves the input to a set of crash bugs (single or many).
2. For each bug, spawns a parallel agent that:
   - Fetches the bug, extracts crash signatures, and runs Phase 1 of the
     `crash-analysis` skill to gather Socorro data.
   - Cross-references against current supported Firefox versions, similar
     signatures, Socorro bug associations, and code existence in searchfox.
   - Flags congruence issues (missing `cf_crash_signature`, severity vs.
     volume mismatches, security-relevant patterns like UAF on a public bug).
   - Returns an actionability assessment and a draft Bugzilla comment.
3. For batch runs, writes a per-bug report to
   `/tmp/crash-bug-triage-report.md` with copy-pasteable comment drafts and
   a summary table at the top.

## Prerequisites

- **`socorro-cli`** for crash data queries
- **`searchfox-cli`** for code references and permalinks
- **`mcp__moz` MCP server** for `get_bugzilla_bug` (reading bug metadata,
  comments, history)
- **`curl` + `jq`** for direct Bugzilla REST API queries
- **`crash-analysis` skill** installed at `~/.claude/skills/crash-analysis/`
  — the per-bug agent reads its `SKILL.md`, `crash-patterns.md`, and
  `lessons.md` to drive crash interpretation
- A Bugzilla account is **not** required for read-only triage; everything
  goes through the public REST API and the MCP `get_bugzilla_bug` tool

## Output

- **Single bug**: draft Bugzilla comment printed in the conversation,
  ready to copy-paste.
- **Multiple bugs**: report at `/tmp/crash-bug-triage-report.md` with a
  summary table and one fenced comment per bug, plus cross-bug
  observations (shared signatures, potential duplicates).

## Notes

- Comments are written as **recommendations**, not decisions — phrasing
  like "seems like a candidate for WORKSFORME" rather than "Closing as ...".
- Says "installs" rather than "users" when discussing crash sources.
- For UAF indicators on public bugs, the structured findings include the
  full evidence (poison addresses, CFG violations) but the draft comment
  omits exploitation-relevant detail; the user decides whether to request
  a sec-rating before posting.
