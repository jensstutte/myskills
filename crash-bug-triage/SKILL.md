---
name: crash-bug-triage
description: Detect stale/unactionable crash bugs in a component group. Use when asked to triage stale crash bugs, review crash bugs in a component, or run crash bug triage.
argument-hint: "[component group or Bugzilla component names]"
allowed-tools:
  - Bash(socorro-cli:*)
  - Bash(searchfox-cli:*)
  - Bash(curl:*)
  - Read
  - Grep
  - Glob
  - WebFetch
---

# Crash Bug Triage — Detecting Stale/Unactionable Bugs

Triage crash bugs for: $ARGUMENTS

## Resolving Component Groups

If $ARGUMENTS is a team name (e.g. "OS Integration") rather than explicit component names, fetch the components from Bugzilla:

```bash
curl -s 'https://bugzilla.mozilla.org/rest/product?type=enterable&include_fields=name,components.name,components.team_name,components.is_active' \
  | jq -r --arg team "TEAM_NAME" '[.products[] | .name as $prod | .components[] | select(.is_active) | select(.team_name == $team) | $prod + "::" + .name] | .[]'
```

This uses the `team_name` field on Bugzilla components. Use case-sensitive matching first; if no results, try case-insensitive.

## Goal

Find open crash bugs in the specified components that are likely no longer actionable, so they can be closed or deprioritized.

## Actionability Criteria

A crash bug is likely **actionable** if ANY of:
1. **Active crashes**: The signature receives crash reports or pings for supported versions (check Socorro)
2. **Similar signatures**: Socorro has reports/pings for similar-enough signatures — bug just needs signature updates
3. **Current code**: The stack frames in the bug reference code that still exists in searchfox
4. **Related intermittents**: Even with no Socorro data, there are intermittent test bugs hitting the same signature
5. **Related open bugs**: Socorro's bug associations show other open bugs tracking the same or overlapping signatures

A crash bug is likely **NOT actionable** if NONE of the above is true.

## Generic Signature Check

If a signature is active but groups unrelated crashes from different subsystems, it may be too generic. Signs:
- Top frame is a templated wrapper, ref-counting method, or infrastructure function (e.g. `RunnableMethodImpl<T>::Run`, `Release`, `arena_dalloc`)
- Crashes under that signature come from many different callers/subsystems (check frame 1+ across several reports)

When detected, check https://github.com/mozilla-services/socorro/blob/main/socorro/signature/siglists/prefix_signature_re.txt to see if the function is already listed. If not, recommend filing a bug on Socorro to add it as a signature prefix. **Do not recommend closing the crash bug** — make it depend on the Socorro prefix bug.

## Congruence Check (for actionable bugs)

Compare bug metadata against actual current crash data:
- **Platform/OS**: Bug says macOS-only but crashes now happen on Linux too?
- **Affected versions**: Bug was filed against Firefox 57 but crashes are now on 147+?
- **Summary/description accuracy**: Does the description still match current crashes?
- **Severity vs volume**: Bug is S3/S4 but signature has 1000+ crashes/90d? Or S1/S2 but 2 crashes/90d?

Use `socorro-cli search --signature "..." --days 90` with `--facet platform`, `--facet version`, and `--facet platform_version` to get actual distribution.

## Steps Per Bug

1. Extract crash signature(s) from `cf_crash_signature` field. If empty, find function names from bug description/comments, use partial matches (`socorro-cli search --signature "~FunctionName"`) and crash pings (`socorro-cli crash-pings --signature "~FunctionName"`), and query the Socorro Bugs API.

2. Query Socorro for recent volume on supported versions — check both crash reports and crash pings:
   ```bash
   socorro-cli search --signature "SIGNATURE" --days 90
   ```
   For crash pings (opt-out, representative), loop over 90 days:
   ```bash
   total=0; for i in $(seq 0 89); do d=$(date -d "today - ${i} days" +%Y-%m-%d); result=$(socorro-cli crash-pings --signature "SIG" --date "$d" 2>&1); count=$(echo "$result" | grep -oP '\((\d+) pings?\)' | grep -oP '\d+'); if [ -n "$count" ] && [ "$count" -gt 0 ]; then echo "$d: $count pings"; total=$((total + count)); fi; done; echo "Total: $total"
   ```
   **Caveat**: Crash pings do not include ESR builds. Check the version facet in Socorro reports before concluding volume is negligible based on pings alone.

3. If no hits, search for similar signatures in Socorro.

4. Check Socorro bug associations:
   ```bash
   curl -s 'https://crash-stats.mozilla.org/api/Bugs/?signatures=SIGNATURE'
   ```

5. Check if stack frames reference current code via searchfox. If not found, use `git log -S 'FunctionName' -- path/to/file` to find what happened to the code.

6. Search Bugzilla for intermittent bugs mentioning the same signature.

7. Generic signature check (see above).

8. Congruence check (see above).

## Bug Comment Review

Always read the bug's comment history before recommending a disposition. Domain experts may have left analysis or context that changes the picture.

## Verification

Before presenting results:
- Verify all bug number cross-references by fetching the cited bug and confirming its summary matches
- Verify searchfox URLs using `searchfox-cli --permalink` (never construct URLs by hand)
- Verify descriptions of cited bugs accurately reflect their actual summary

## Output Format

Present findings as a **bugzilla comment**: raw markdown source inside a code block, ready to copy-paste.
- Use markdown links to searchfox for source references (permalink URLs from `searchfox-cli --permalink`)
- Use `bug NNNNNN` for bugzilla cross-references (auto-linked by bugzilla)
- Tone: factual, slightly hedging where evidence isn't conclusive ("seems", "appears")
- Say "installs" not "users" when referring to crash sources
- Include:
  - Current crash volume (90 days) with version and platform breakdown
  - Socorro bug associations and their status
  - Whether the code still exists in the current tree (with searchfox permalink links)
  - If actionable: congruence notes (platform/severity/description drift)
  - Recommendation (close as WONTFIX/INCOMPLETE, update metadata, keep open, etc.)

## Data Sources

- Bugzilla REST API, socorro-cli (search, crash-pings), searchfox-cli
- Socorro Bugs API: `curl 'https://crash-stats.mozilla.org/api/Bugs/?signatures=<sig>'`
- Socorro prefix list: https://github.com/mozilla-services/socorro/blob/main/socorro/signature/siglists/prefix_signature_re.txt
- Refresh supported versions from https://whattrainisitnow.com/api/release/schedule/?version=N
