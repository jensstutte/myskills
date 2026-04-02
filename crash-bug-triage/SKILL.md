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

When sorting by "oldest" or selecting stale bugs, use `last_change_time` to skip bugs with recent meaningful activity (within ~4 months). A bot comment updating keywords or volume stats doesn't count as meaningful — look for human comments with actual analysis. Bugs where a domain expert recently investigated the crash are low-value triage targets; prioritize bugs that have been dormant longer.

Present the list to the user as a table: bug number, summary (truncated), signature count, severity, age, last meaningful human comment date. Then wait for the user to pick bugs (by number) or confirm "all" — unless `all` was already in $ARGUMENTS.

## Pre-flight: Verify crash-analysis skill

Before spawning any triage agents, verify that the crash-analysis skill file exists. Use Glob to find it:

```
~/.claude/skills/crash-analysis/SKILL.md
```

If not found, try `~/.claude/skills/**/SKILL.md` to locate all skill files and look for one named `crash-analysis`. If still not found, ask the user for the path to the crash-analysis skill directory.

Store the resolved absolute path — the per-bug agents will need it to read the Phase 1 instructions.

## Per-Bug Triage

If more than 10 bugs are selected, ask the user to confirm before proceeding. Suggest triaging the first 10 (e.g. oldest, or highest severity) and offer to continue with the rest afterward.

For each selected bug, spawn a separate Agent (subagent_type: "general-purpose") to triage it in parallel. Each agent receives the full per-bug prompt below, with the bug number filled in and `{CRASH_ANALYSIS_SKILL_DIR}` replaced with the resolved absolute path from the pre-flight step.

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
> **Step 3 — Crash analysis (Phase 1 triage)**
> This is the most important step — it provides the crash data that drives the entire assessment.
> Read `{CRASH_ANALYSIS_SKILL_DIR}/SKILL.md` and follow **Phase 1** (steps 1a through 1e) for each signature. Also read `crash-patterns.md` and `lessons.md` in the same directory for pattern recognition and pitfalls.
>
> **Caveat**: Crash pings do not include ESR builds. Check the version facet in Socorro reports before concluding volume is negligible based on pings alone.
> Report both opt-in report counts and crash ping counts — never omit one because the other showed results.
>
> **Step 3b — Supported version check**
> Fetch current supported Firefox versions:
> ```bash
> for ch in release beta nightly esr; do
>   v=$(curl -s "https://whattrainisitnow.com/api/release/schedule/?version=$ch" | jq -r '.version')
>   echo "$ch: $v"
> done
> ```
> Cross-reference the version facet from step 3 against these. Classify crash volume:
> - **Supported**: crashes on current release, beta, nightly, or current ESR (e.g. 140.x ESR) — fully actionable
> - **Previous release**: one or two versions behind current release (e.g. 148 when 149 is current) — still relevant, likely affects current too
> - **Old ESR**: crashes on previous ESR cycle (e.g. ESR 115 when ESR 140 is current) — lower concern, especially if ESR 115 is approaching EOL. Note this in the assessment.
> - **Ancient**: crashes only on versions many releases behind (e.g. Fx 120 when 149 is current) — likely not actionable unless code hasn't changed
>
> If ALL crashes are on old/ancient versions and none on supported versions, this significantly reduces actionability. Conversely, a rare crash that appears on 149 is likely also possible on 150 — don't dismiss low volume on current versions.
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
> **Step 8 — Examine individual crashes for low-volume signatures**
> For signatures with fewer than ~20 crashes in 90 days, fetch several individual crash reports (`socorro-cli crash CRASH_ID`) and examine the actual stacks. Do not rely solely on facet-level metadata (platform, version, crash reason). Check:
> - Do all crashes hit the same call site (same frame #0 / proto_signature), or do they have genuinely different stacks?
> - What is actually being dereferenced at the crash point?
> - Are faulting addresses consistent (same bug) or scattered/patterned (hardware corruption)?
>   - `0xffffffff`, `0xfffffffe` → bitflip-of-NULL patterns
>   - `0xe5e5e5e5e5e5e5e5` → mozjemalloc freed memory (kAllocPoison) — **confirms UAF**
>   - `0x4b4b4b4b4b4b4b4b` → jemalloc poison — **confirms UAF**
>   - Small offsets from poison values (e.g. `0xe5e5e5e5e5e5e5ed`) → UAF with vtable offset
>   - Bit 31 set on otherwise valid user-mode pointer → single-bit flip
>   - Diverse random addresses with consistent crash reason → use-after-free (freed memory reused with different content each time)
>   - Diverse random addresses with diverse crash reasons → hardware corruption
>
> This distinguishes three very different situations:
> - **Bucket signature**: A generic infrastructure function (template wrapper, ref-counting method, QI dispatch) that appears at different positions in different stacks, grouping truly unrelated crashes. These benefit from Socorro prefix bugs.
> - **Canary site**: All crashes hit the exact same code path, but the crash addresses and corruption patterns vary. This means the crash site is the first dereference that exposes earlier corruption — the root causes are different but the manifestation point is the same. These do NOT benefit from prefix bugs (there is no more specific frame above).
> - **Hardware/external corruption**: Diverse crash reasons (EXEC, READ, WRITE, ILLEGAL, PRIV, STACK_COOKIE) at the same code location, third-party modules in proto_signatures, bitflip address patterns. Not a Firefox code bug.
>
> **Step 9 — Generic/bucket signature check**
> If step 8 identified a bucket signature (different stacks landing in a generic function), check https://github.com/mozilla-services/socorro/blob/main/socorro/signature/siglists/prefix_signature_re.txt to see if the function is already listed. If not, recommend filing a Socorro prefix bug. Do NOT recommend closing the crash bug — make it depend on the prefix bug.
>
> **Step 10 — Congruence check (actionable bugs only)**
> Compare bug metadata against actual crash data:
> - **Missing cf_crash_signature**: Bug has the crash keyword but `cf_crash_signature` is empty. If you found active signatures in earlier steps, flag this — the signature field should be populated.
> - **Missing crash keyword**: Bug has `cf_crash_signature` set but no crash keyword. Flag for adding the keyword.
> - **UAF on non-sec bug**: If crash data shows use-after-free indicators (poison addresses, CFG violations, diverse faulting addresses with consistent READ reason) and the bug is NOT security-restricted (i.e. publicly visible), flag this prominently. UAF bugs are potentially exploitable and may need sec-rating and access restriction. Indicators to check:
>   - Faulting addresses matching `0xe5e5e5e5*` (kAllocPoison) or `0x4b4b4b4b*` (jemalloc poison)
>   - Any `FAST_FAIL_GUARD_ICALL_CHECK_FAILURE` in the reason facet (CFG caught a vtable call on freed memory)
>   - Consistent crash reason (ACCESS_VIOLATION_READ) but widely scattered faulting addresses (freed memory reused with different content)
>   - Disassembly showing a vtable dispatch (`call [reg]` or `call [reg+offset]`) at the crash point
> - Platform/OS drift (bug says macOS-only but crashes now on Linux too?)
> - Affected versions (filed against Fx57 but crashes on 147+?)
> - Summary/description accuracy
> - Severity vs volume mismatch (S3/S4 but 1000+ crashes/90d? S1/S2 but 2 crashes/90d?)
>
> Use `socorro-cli search --signature "..." --days 90` with `--facet platform`, `--facet version`, `--facet platform_version`.
>
> **Step 11 — Read comment history thoroughly**
> Read ALL comments in the bug before making your assessment. This is critical — domain experts often leave detailed technical analysis, identify root causes, or explain why earlier spikes happened. Your triage comment should build on and reference their findings (e.g. "As nika noted in comment 10, ...") rather than re-derive conclusions from scratch. If an expert already analyzed the crash mechanism, incorporate their framing. Ignoring existing analysis leads to superficial or incorrect assessments.
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
> - Supported versions: {yes — N crashes on current release/beta/ESR | no — all crashes on EOL versions}
>
> ### Crash Reason Breakdown
> - ACCESS_VIOLATION_READ (58), STATUS_HEAP_CORRUPTION (26), etc.
> - Assessment: single bug / diverse root causes / hardware corruption
>
> ### Proto_signature Clusters
> - cluster 1 (N crashes): brief description of code path
> - cluster 2 (N crashes): brief description
> - scattered (N crashes): no common pattern
>
> ### Third-party Module Involvement
> - igd10iumd64.dll (Intel GPU, N crashes) / none detected
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
> ### Security
> - **UAF on public bug**: yes/no. If yes: list evidence (poison addresses, CFG violations, scattered faulting addresses). Flag that bug may need sec-rating and access restriction.
> - If no UAF indicators found, omit this section.
>
> ### Comment History Notes
> - Comment 5: domain expert said "likely fixed by bug NNNNNN"
>
> ### Assessment
> - Actionable: yes/no
> - Reason: {which criteria matched, or why none did}
> - Recommendation: {close WONTFIX, close INCOMPLETE, update metadata, keep open, etc.}
> - **If UAF detected on public bug**: recommend sec-rating assessment regardless of other actionability determination. A non-actionable UAF (e.g. code removed) is fine to close, but an actionable UAF on a public bug should be flagged urgently.
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
   - Reference and build on existing expert analysis from bug comments (e.g. "As nika noted in comment 10, ...") rather than presenting findings as entirely new
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
