# shutdownkill-triage

Slash command that triages content-process `IPCError-browser | ShutDownKill` crashes (meta bug 1279293) by correlating Socorro proto_signatures with BHR (Background Hang Reporter) hang data on Nightly.

## What it does

For a given time window (default: 7 days):

1. Pulls ShutDownKill proto_signatures from the Socorro SuperSearch API,
   facetted on `proto_signature` for Firefox Nightly.
2. Strips boilerplate frames (OS primitives, event loop, IPC dispatch,
   allocator internals, etc.) from each proto_signature to identify the
   "interesting function" — the application-level call that says what the
   content process was actually doing when killed.
3. Groups proto_signatures by interesting function and sums kill counts.
4. Loads BHR child-process hang aggregates and builds a per-function hang
   index weighted by usage hours.
5. Correlates each interesting function against the BHR index to see
   whether it also hangs during normal browsing.
6. Looks for existing bugs depending on bug 1279293 or carrying matching
   `[bhr:...]` whiteboard tags.
7. Outputs a ranked summary table; on request, drafts a Bugzilla comment
   for any specific entry with searchfox permalinks and a recommendation.

## Prerequisites

- **`socorro-cli`** and **`searchfox-cli`**
- **`curl` + `jq`** for the Socorro SuperSearch API and the BHR JSON aggregate
- **Python 3** for parsing the BHR JSON (~677K samples, ~5 seconds)
- No auth required — Socorro SuperSearch, the BHR aggregate JSON, and the
  Bugzilla REST queries used here are all public

## Data sources

- Socorro SuperSearch API
  (`https://crash-stats.mozilla.org/api/SuperSearch/`)
- BHR aggregate
  (`https://analysis-output.telemetry.mozilla.org/bhr/data/hang_aggregates/hangs_child_current.json`)
- Bugzilla REST API for bugs depending on 1279293 and `[bhr:...]` whiteboard
- Searchfox via `searchfox-cli --permalink` for code references
- Companion dashboard: <https://fqueze.github.io/hang-stats/child/>
  (source: <https://github.com/fqueze/hang-stats>)

## Notes

- ShutDownKill crashes are Nightly-only; proto_signatures symbolicate best
  there.
- ~59% of ShutDownKills are "idle at shutdown" — content processes waiting
  for work when killed. These are reported separately and aren't
  individually actionable.
- A function appearing in both ShutDownKill and BHR means the hang happens
  during normal browsing too — fixing it improves both shutdown speed and
  general responsiveness. A function only in ShutDownKill is
  shutdown-specific.
- BHR aggregate represents one Nightly build date (the most recent
  available); historical data uses
  `hangs_child_{YYYY-MM-DD}.json` instead of `current`.
