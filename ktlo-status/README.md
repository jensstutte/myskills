# ktlo-status

Slash command that checks a KTLO Jira ticket against its BugDash maintenance-effectiveness numbers and (optionally) posts the assessment back to Jira as a formatted comment.

## What it does

Given a Jira ticket key (e.g. `FFXP-3811`):

1. Reads the ticket via `acli` to find the Bugzilla components and the ME goal.
2. Builds a BugDash overview URL for those components.
3. Loads BugDash in Firefox via the DevTools MCP and screenshots the Overview tab.
4. Reports the open-defect counts, maintenance trend table, and an assessment vs. the goal.
5. On request, posts the assessment back to Jira as an ADF-formatted comment.

## Prerequisites

- **`acli`** installed and authenticated against your Jira instance
  (used both to read the ticket and to post comments).
- **Firefox Nightly** at `/usr/bin/firefox-nightly` (or edit the path in `SKILL.md`).
- **Firefox DevTools MCP server** registered with Claude Code. The skill calls
  `mcp__firefox-devtools__restart_firefox`, `list_pages`, `take_snapshot`,
  `screenshot_page`, and `evaluate_script`.
- **A persistent Firefox profile at `~/.firefox-mcp-profile`** that is logged
  into BugDash. See setup below.

## Setting up `~/.firefox-mcp-profile` for BugDash

BugDash (`https://bugdash.moz.tools/`) requires Mozilla SSO. The MCP launches
Firefox in a way that does not interactively prompt, so the profile must
already hold a valid BugDash session.

One-time setup:

1. Create an empty profile directory if it doesn't already exist:

   ```sh
   mkdir -p ~/.firefox-mcp-profile
   ```

2. Launch Firefox Nightly **manually** against that profile and complete the
   SSO flow once:

   ```sh
   /usr/bin/firefox-nightly --profile ~/.firefox-mcp-profile --no-remote https://bugdash.moz.tools/
   ```

   - Sign in via Mozilla SSO when prompted.
   - Wait for the BugDash overview to render so the auth handshake completes.
   - Close Firefox cleanly so cookies/storage are flushed to disk.

3. From this point on, the MCP can launch Firefox against the same profile and
   BugDash will load already-authenticated. The skill will not work if this
   profile is missing or the BugDash session has expired — re-run step 2 if
   you start seeing the SSO/login screen in screenshots.

## Bugzilla component name gotchas

A few component names in Jira ticket descriptions don't match Bugzilla
exactly. The skill tolerates these by retrying with the correct name once
seen, but it's worth knowing:

- "Core: Widget: Windows" in tickets → actually `Core: Widget: Win32`
- "Core: Widget: GTK" in tickets → actually `Core: Widget: Gtk`
- "Core: Hardware Abstraction Layer" → actually `Core: Hardware Abstraction Layer (HAL)`

The Bugzilla REST API can be queried for the canonical list:

```sh
curl -s 'https://bugzilla.mozilla.org/rest/product?names=Core&include_fields=components.name,components.team_name' | jq
```

This is also useful for finding all components owned by a given team
(`team_name` field).

## Posting comments

When asked to post the assessment, the skill writes an
[Atlassian Document Format](https://developer.atlassian.com/cloud/jira/platform/apis/document/structure/)
(ADF) JSON document to `/tmp/ktlo-comment.json` and uses:

```sh
acli jira workitem comment create --key <KEY> --body-file /tmp/ktlo-comment.json
```

Plain-text `--body` loses all formatting (newlines, links, tables); always go
through `--body-file` with ADF.
