---
name: shutdownkill-triage
description: Triage content process ShutDownKill crashes by correlating Socorro proto_signatures with BHR hang data. Use when asked to triage shutdownkill signatures, review shutdown kills, or investigate IPCError-browser ShutDownKill crashes.
argument-hint: "[time window, e.g. 'last 14 days' — default: 7 days]"
allowed-tools:
  - Bash(socorro-cli:*)
  - Bash(searchfox-cli:*)
  - Bash(curl:*)
  - Read
  - Grep
  - Glob
  - WebFetch
---

# ShutDownKill Triage

Time window: $ARGUMENTS (default: 7 days)

## Goal

Triage `IPCError-browser | ShutDownKill` content process crashes (meta bug 1279293) by identifying what the content process was actually doing when killed, correlating with BHR (Background Hang Reporter) data to gauge hang severity during normal browsing, and checking for existing bugs.

## Data Sources

1. **Socorro SuperSearch API** -- `proto_signature` facet for `=IPCError-browser | ShutDownKill` on nightly
2. **BHR child data** -- `https://analysis-output.telemetry.mozilla.org/bhr/data/hang_aggregates/hangs_child_current.json` (nightly content process hangs, weighted by usage hours)
3. **Bugzilla REST API** -- bugs that depend on bug 1279293, bugs with `[bhr:...]` whiteboard tags
4. **searchfox-cli** -- verify code references, get permalink URLs
5. **Dashboard** -- https://fqueze.github.io/hang-stats/child/ (source: https://github.com/fqueze/hang-stats)

## Step 1: Fetch ShutDownKill proto_signatures from Socorro

```bash
DAYS_AGO=$(date -d "7 days ago" +%Y-%m-%d)
curl -s "https://crash-stats.mozilla.org/api/SuperSearch/?signature=%3DIPCError-browser+%7C+ShutDownKill&_facets=proto_signature&_facets_size=50&_results_number=0&date=%3E%3D${DAYS_AGO}&product=Firefox&release_channel=nightly" \
  | jq '.facets.proto_signature'
```

Each entry has `term` (the full proto_signature) and `count`.

## Step 2: Extract "interesting function" from each proto_signature

Proto_signatures are `|`-separated frame lists, top of stack first. Strip boilerplate to find the application-level function that identifies what was actually happening.

**Boilerplate frame prefixes to skip** (reading top-to-bottom):
- **OS primitives**: `Zw`, `Nt`, `Rtl`, `KiFast`, `pthread_`, `__psynch_`, `epoll_wait`, `kevent`, `mach_msg`, `SleepConditionVariable`, `GetSystemTimes`, `NtQuery`, `CreateRemoteThread`, `CreateThread`, `CloseHandle`, `UnmapViewOfFile`, `ReleaseSemaphore`, `ReleaseMutex`
- **Rust atomics/primitives** (too generic as leaf): `core::sync::atomic::`, `core::ptr::`, `core::ops::function::`, `core::ffi::c_str::`, `core::core_arch::`, `core::mem::`, `servo_arc::Arc`, `servo_arc::impl$`, `alloc::vec::`, `alloc::raw_vec::`, `hashbrown::`, `smallvec::`, `std::collections::`, `atomic_refcell::`
- **C generic**: `strlen`, `memcpy`, `memset`, `memmove`
- **Low-level Mozilla**: `mozilla::detail::ConditionVariableImpl::`, `mozilla::detail::MutexImpl::`, `mozilla::OffTheBooksCondVar::`, `mozilla::OffTheBooksMutex::`, `mozilla::Mutex::`, `mozilla::Monitor::`, `mozilla::detail::BaseAutoLock`, `mozilla::MonitorAutoLock::`, `mozilla::RefPtrTraits`, `nsCOMPtr<`
- **Event loop / message pump**: `nsThread::ProcessNextEvent`, `NS_ProcessNextEvent`, `mozilla::ipc::MessagePump::Run`, `mozilla::ipc::MessagePumpForChildProcess::Run`, `MessageLoop::Run`, `MessageLoop::RunInternal`, `MessageLoop::RunHandler`, `nsBaseAppShell::Run`, `nsAppShell::Run`, `XRE_RunAppShell`, `XRE_InitChildProcess`, `mozilla::BootstrapImpl::`, `NS_internal_main`, `wmain`, `invoke_main`, `__scrt_common_main`, `BaseThreadInitThunk`, `RtlUserThreadStart`, `_RtlUserThreadStart`, `__RtlUserThreadStart`, `_start`
- **IPC dispatch**: `mozilla::ipc::MessageChannel::Dispatch`, `mozilla::ipc::MessageChannel::RunMessage`, `mozilla::ipc::MessageChannel::MessageTask::Run`
- **Task controller**: `mozilla::TaskController::` (all variants)
- **Runnable dispatch**: `mozilla::RunnableTask::Run`, `mozilla::detail::RunnableFunction`, `mozilla::RunnableMethodImpl`
- **SpinEventLoop**: `mozilla::SpinEventLoopUntil`, `nsThreadSyncDispatch::`, `NS_DispatchAndSpinEventLoopUntilComplete`
- **PContent dispatch**: `mozilla::dom::PContentChild::OnMessageReceived`
- **BHR / CPU monitoring**: `mozilla::BackgroundHangThread::`, `mozilla::BackgroundHangManager::`, `mozilla::BackgroundHangMonitor::`, `mozilla::CPUUsageWatcher::`, `mozilla::GetGlobalCPUStats`
- **Allocator internals**: `arena_t::`, `AllocPageInfo::`, `PHC::`, `MozJemalloc::`, `MozJemallocPHC::`
- **Thread creation**: `CreateThreadStub`, `beginthreadex`
- **Exact matches**: `start`, `main`, `poll`, `kevent`, `epoll_wait`, `strlen`, `(unresolved)`, any `0x...` addresses

The first non-boilerplate function is the **interesting function**. Also capture the next 2-3 non-boilerplate frames as **context**.

**Special case**: if ALL frames are boilerplate (idle event loop), classify as "idle at shutdown" -- not actionable.

## Step 3: Group by interesting function

Multiple proto_signatures may share the same interesting function but differ in OS/platform frames. Group them and sum crash counts.

## Step 4: Fetch BHR child data and build per-function hang index

```bash
curl -s 'https://analysis-output.telemetry.mozilla.org/bhr/data/hang_aggregates/hangs_child_current.json' -o /tmp/bhr_child.json
```

Then build a function-level hang index with Python (~5 seconds for ~677K samples):

```python
import json
with open('/tmp/bhr_child.json') as f:
    data = json.load(f)
t = data['threads'][0]
day = t['dates'][0]
date = day['date']
usage_hours = data['usageHoursByDate'][date]

func_names = {i: t['stringArray'][t['funcTable']['name'][i]]
              for i in range(t['funcTable']['length'])}

func_duration = {}  # func_name -> total_duration_ms
func_count = {}     # func_name -> total_count

stack_func = t['stackTable']['func']
stack_prefix = t['stackTable']['prefix']

for sid in range(len(day['sampleHangMs'])):
    dur = day['sampleHangMs'][sid] * usage_hours
    cnt = day['sampleHangCount'][sid] * usage_hours
    visited = set()
    stack = t['sampleTable']['stack'][sid]
    while stack is not None:
        fid = stack_func[stack]
        if fid not in visited:
            visited.add(fid)
            name = func_names[fid]
            func_duration[name] = func_duration.get(name, 0) + dur
            func_count[name] = func_count.get(name, 0) + cnt
        stack = stack_prefix[stack]
```

## Step 5: Correlate ShutDownKill signatures with BHR

For each interesting function from step 3, search the BHR function index. Matching strategy:
- **Exact match**: function name appears verbatim in BHR
- **Prefix match**: for templated names (`Foo<T>` in proto_sig), match BHR functions starting with `Foo<`
- **Context fallback**: if the interesting function has no BHR match, try context frames

**Important**: filter out overly generic matches -- if a BHR function accounts for >50% of total hang time, it's too ubiquitous to be meaningful.

Report for each entry:
- BHR hang duration (seconds)
- BHR hang count
- BHR % of total content-process hang time

## Step 6: Check existing bugs

```bash
# Bugs that depend on bug 1279293 (use blocks= to find bugs whose blocks field includes 1279293)
curl -s 'https://bugzilla.mozilla.org/rest/bug?blocks=1279293&include_fields=id,summary,status,resolution,whiteboard&limit=200'
```

Also fetch BHR-tagged bugs:
```bash
curl -s 'https://bugzilla.mozilla.org/rest/bug?include_fields=id,summary,status,whiteboard&bug_status=UNCONFIRMED&bug_status=NEW&bug_status=ASSIGNED&bug_status=REOPENED&f1=classification&field0-0-0=status_whiteboard&o1=notequals&type0-0-0=substring&v1=Graveyard&value0-0-0=%5Bbhr%3A'
```

Match bugs against interesting functions by:
- Substring match on bug summary
- `[bhr:signature]` whiteboard tag match

## Step 7: Output -- summary table

Ranked by ShutDownKill count (descending):

```
  # | Kill# | BHR Dur(s) |   BHR % | BHR Count | Function + Context
----+-------+------------+---------+-----------+---------
  1 |   312 |          - |       - |         - | mozilla::ChildProfilerController::Shutdown...
    |       |            |         |           |   -> GrabShutdownProfileAndShutdown -> ContentChild::ShutdownInternal
    |       |            |         |           |   bug 1622110 [NEW]
```

Entries classified as "idle at shutdown" listed separately with a note.

## Step 8: Drill-down -- bugzilla comment (on request)

When asked about a specific entry, produce a bugzilla-ready comment with:
- Proto_signature(s) and nightly crash count
- BHR correlation data
- Searchfox permalink links to the interesting function (via `searchfox-cli --permalink`)
- Related existing bugs with status
- Recommendation: file new bug (as dependency of bug 1279293), update existing, or not actionable
- Tone: factual, say "installs" not "users"

## Notes

- BHR data is from a single nightly build date (the most recent available). It represents content process main thread hangs across all nightly installs for that day.
- ShutDownKill crashes are nightly-only. Proto_signatures are best symbolicated on nightly.
- A function in both ShutDownKill and BHR means the hang occurs during normal browsing too -- fixing it benefits both shutdown speed and general responsiveness.
- A function only in ShutDownKill (no BHR match) may be shutdown-specific -- still worth investigating but impact is limited to shutdown.
- BHR data URL: `https://analysis-output.telemetry.mozilla.org/bhr/data/hang_aggregates/hangs_child_{date}.json` (use `current` for latest)
- ~59% of ShutDownKill crashes are "idle at shutdown" -- content processes waiting for work when killed. These are not individually actionable.
