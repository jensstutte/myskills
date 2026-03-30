---
name: crash-bug-triage
description: Detect stale/unactionable crash bugs for a team or component. Use when asked to triage stale crash bugs, review crash bugs in a component, or run crash bug triage.
argument-hint: "<bug-number> | <team-name-or-component-names> [all]"
allowed-tools:
  - Agent
  - Bash(socorro-cli:*)
  - Bash(searchfox-cli:*)
  - Bash(curl:*)
  - Bash(jq:*)
  - Read
  - Grep
  - Glob
  - WebFetch
  - mcp__moz__get_bugzilla_bug
---

# Crash Bug Triage — Detecting Stale/Unactionable Bugs

Triage crash bugs for: $ARGUMENTS

## Argument Parsing

$ARGUMENTS is one of:
1. **A bug number** (all-digits) — triage that single bug
2. **A team name or explicit component names** — fetch crash bugs from those components, then either:
   - Present the list and let the user pick which to triage, OR
   - If `all` is appended to the arguments, triage every bug found

## Resolving Team Names to Components

If $ARGUMENTS is a team name (e.g. "OS Integration") rather than explicit `Product::Component` names, fetch the components from Bugzilla:

```bash
curl -s 'https://bugzilla.mozilla.org/rest/product?type=accessible&include_fields=name,components.name,components.team_name,components.is_active' \
  | jq -r --arg team "TEAM_NAME" '[.products[] | .name as $prod | .components[] | select(.is_active) | select(.team_name == $team) | $prod + "::" + .name] | .[]'
```

Use case-sensitive matching first; if no results, try case-insensitive.

## Fetching Crash Bugs from Components

Query Bugzilla for open bugs that have a crash keyword OR a non-empty crash signature (using OR avoids missing bugs where only one was set):

```bash
curl -s 'https://bugzilla.mozilla.org/rest/bug?resolution=---&j_top=OR&f1=keywords&o1=anywords&v1=crash&f2=cf_crash_signature&o2=isnotempty&include_fields=id,summary,cf_crash_signature,severity,creation_time,last_change_time&product=PRODUCT&component=COMPONENT&limit=100'
```

Run one query per product/component pair. Collect all results, deduplicate by bug ID.

Present the list to the user as a table: bug number, summary (truncated), signature count, severity, age. Then wait for the user to pick bugs (by number) or confirm "all" — unless `all` was already in $ARGUMENTS.

## Per-Bug Triage

If more than 10 bugs are selected, ask the user to confirm before proceeding. Suggest triaging the first 10 (e.g. oldest, or highest severity) and offer to continue with the rest afterward.

For each selected bug, spawn a separate Agent (subagent_type: "general-purpose") to triage it in parallel. Each agent receives the full per-bug prompt below, with the bug number filled in.

### Per-Bug Agent Prompt

> You are triaging Firefox crash bug {BUG_NUMBER} to determine if it is still actionable.
> Use `socorro-cli`, `searchfox-cli`, and the `mcp__moz__get_bugzilla_bug` tool.
> Read `--help` for any CLI tool before first use.
>
> **Step 1 — Fetch the bug**
> Use `mcp__moz__get_bugzilla_bug` to get the bug, including its `cf_crash_signature`, severity, summary, comments, and metadata.
>
> **Step 2 — Extract signatures**
> Parse crash signature(s) from `cf_crash_signature`. If empty:
> - Examine bug comments and attachments for crash stacks, crash report URLs (crash-stats.mozilla.org/report/index/...), or Socorro search links. Extract function names and crash IDs from these.
> - Use partial matches (`socorro-cli search --signature "~FunctionName"`) and crash pings (`socorro-cli crash-pings --signature "~FunctionName"`) to find matching signatures.
> - Query the Socorro Bugs API (`socorro-cli bugs --bug-id {BUG_NUMBER}`) to find signatures already associated with this bug.
> - If a crash ID is found, fetch it with `socorro-cli crash CRASH_ID` to get the full signature.
>
> **Step 3 — Check crash volume (90 days)**
> For each signature, you MUST check BOTH data sources — crash reports (opt-in) and crash pings (opt-out, representative). Neither alone gives the full picture: reports have detail but under-count, pings are representative but lack ESR and detail.
>
> Crash reports:
> ```bash
> socorro-cli search --signature "SIGNATURE" --days 90
> ```
>
> Crash pings (always run this, even if reports show volume):
> ```bash
> total=0; for i in $(seq 0 89); do d=$(date -d "today - ${i} days" +%Y-%m-%d); result=$(socorro-cli crash-pings --signature "SIG" --date "$d" 2>&1); count=$(echo "$result" | grep -oP '\((\d+) pings?\)' | grep -oP '\d+'); if [ -n "$count" ] && [ "$count" -gt 0 ]; then echo "$d: $count pings"; total=$((total + count)); fi; done; echo "Total: $total"
> ```
>
> **Caveat**: Crash pings do not include ESR builds. Check the version facet in Socorro reports before concluding volume is negligible based on pings alone.
> Report both numbers in the output — never omit one because the other showed results.
>
> **Step 4 — Similar signatures**
> If no hits, search for similar signatures in Socorro.
>
> **Step 5 — Socorro bug associations**
> ```bash
> socorro-cli bugs --signature "SIGNATURE"
> ```
>
> **Step 6 — Check code existence**
> Check if stack frames reference current code via searchfox. If not found, use `git log -S 'FunctionName' -- path/to/file` to find what happened.
>
> **Step 7 — Related intermittents**
> Search Bugzilla for intermittent bugs mentioning the same signature.
>
> **Step 8 — Generic signature check**
> If the signature is active, check if it groups unrelated crashes:
> - Top frame is a templated wrapper, ref-counting method, or infrastructure function
> - Crashes come from many different callers/subsystems
>
> If so, check https://github.com/mozilla-services/socorro/blob/main/socorro/signature/siglists/prefix_signature_re.txt to see if the function is already listed. If not, recommend filing a Socorro prefix bug. Do NOT recommend closing the crash bug — make it depend on the prefix bug.
>
> **Step 9 — Congruence check (actionable bugs only)**
> Compare bug metadata against actual crash data:
> - **Missing cf_crash_signature**: Bug has the crash keyword but `cf_crash_signature` is empty. If you found active signatures in earlier steps, flag this — the signature field should be populated.
> - **Missing crash keyword**: Bug has `cf_crash_signature` set but no crash keyword. Flag for adding the keyword.
> - Platform/OS drift (bug says macOS-only but crashes now on Linux too?)
> - Affected versions (filed against Fx57 but crashes on 147+?)
> - Summary/description accuracy
> - Severity vs volume mismatch (S3/S4 but 1000+ crashes/90d? S1/S2 but 2 crashes/90d?)
>
> Use `socorro-cli search --signature "..." --days 90` with `--facet platform`, `--facet version`, `--facet platform_version`.
>
> **Step 10 — Read comment history**
> Always read the bug's comments before deciding. Domain experts may have left analysis or context.
>
> **Actionability criteria** — a bug is actionable if ANY of:
> 1. Active crashes on supported versions
> 2. Similar signatures exist (bug just needs signature updates)
> 3. Stack frames reference code that still exists
> 4. Related intermittent test bugs hit the same signature
> 5. Socorro bug associations show other open bugs for overlapping signatures
>
> NOT actionable if NONE of the above.
>
> **Verification** — before returning:
> - Verify all bug cross-references by fetching cited bugs and confirming summaries match
> - Generate searchfox URLs with `searchfox-cli --permalink` (never construct by hand)
> - Verify descriptions of cited bugs accurately reflect their actual summary
>
> **Output** — return structured findings in this format:
>
> ```
> ## Bug {NUMBER}: {SUMMARY}
>
> ### Signatures
> - `sig1` — {N} reports/90d, {N} pings/90d
> - `sig2` — 0 reports, 0 pings
>
> ### Volume Breakdown
> - Platforms: Windows 78%, Linux 15%, macOS 7%
> - Versions: 136 (40%), 137 (55%), ESR 128 (5%)
>
> ### Code Status
> - `SomeClass::Method` — still exists ({searchfox permalink})
> - `OldClass::Removed` — deleted in bug 1234567
>
> ### Socorro Associations
> - bug 111111 (FIXED) — same signature
> - bug 222222 (OPEN) — overlapping signature
>
> ### Generic Signature
> - Not generic / Generic: top frame is `Release`, recommend prefix bug
>
> ### Congruence
> - Missing cf_crash_signature — has keyword but no signature; found active sig `OtherFunc`
> - Missing crash keyword — has signature but no keyword
> - Severity S4 but 342 crashes/90d — mismatch
> - Bug says macOS-only, now mostly Windows
>
> ### Comment History Notes
> - Comment 5: domain expert said "likely fixed by bug NNNNNN"
>
> ### Assessment
> - Actionable: yes/no
> - Reason: {which criteria matched, or why none did}
> - Recommendation: {close WONTFIX, close INCOMPLETE, update metadata, keep open, etc.}
> ```

## Collecting Results and Writing Comments

After all agents complete:

1. Review all structured findings. Look for cross-bug patterns:
   - Bugs sharing the same signature or Socorro associations
   - Bugs that duplicate each other
   - Signatures that appear across multiple bugs

2. Group results:
   - **Not actionable** — bugs recommended for closure
   - **Actionable with notes** — bugs that should stay open but need metadata updates
   - **Actionable, no changes needed** — briefly list (no comment needed)

3. For each bug that needs a comment (not-actionable and actionable-with-notes), craft a **bugzilla comment** from the agent's structured findings. Format rules:
   - Use markdown links to searchfox for source references (permalink URLs from agent findings)
   - Use `bug NNNNNN` for bugzilla cross-references (auto-linked by bugzilla)
   - Tone: factual, slightly hedging where evidence isn't conclusive ("seems", "appears")
   - Say "installs" not "users" when referring to crash sources
   - Include: crash volume (90d) with version/platform breakdown, Socorro bug associations and status, code existence with searchfox permalinks, congruence notes if actionable, recommendation

### Output

**Single bug**: present the draft bugzilla comment directly in the conversation (inside a code block, ready to copy-paste).

**Multiple bugs**: write a report to `/tmp/crash-bug-triage-report.md` containing:
- A summary table at the top: bug number, summary, assessment (actionable/not), recommendation
- Cross-bug observations (shared signatures, potential duplicates)
- One section per bug with:
  - Key findings (brief)
  - Draft bugzilla comment (inside a fenced code block, ready to copy-paste)

Tell the user the report path and print only the summary table in the conversation. The user can review the full report in their editor.

## Data Sources

- Bugzilla REST API, `mcp__moz__get_bugzilla_bug`, socorro-cli (search, crash-pings, bugs), searchfox-cli
- Socorro prefix list: https://github.com/mozilla-services/socorro/blob/main/socorro/signature/siglists/prefix_signature_re.txt
- Supported versions: https://whattrainisitnow.com/api/release/schedule/?version=N
