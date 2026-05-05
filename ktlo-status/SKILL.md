---
name: ktlo-status
description: Check KTLO Jira ticket status by reading the ticket's Bugzilla components and maintenance effectiveness goal, then fetching actual numbers from BugDash. Use when asked to check KTLO status, maintenance effectiveness, or bugdash numbers for a Jira ticket.
argument-hint: "<Jira ticket key, e.g. FFXP-3811>"
allowed-tools:
  - Bash(acli:*)
  - Bash(sleep:*)
  - Read
  - mcp__firefox-devtools__restart_firefox
  - mcp__firefox-devtools__list_pages
  - mcp__firefox-devtools__take_snapshot
  - mcp__firefox-devtools__screenshot_page
  - mcp__firefox-devtools__navigate_page
  - mcp__firefox-devtools__evaluate_script
---

# KTLO Maintenance Effectiveness Status Check

Check status for: $ARGUMENTS

## Step 1: Read the Jira Ticket

```bash
acli jira workitem view <TICKET_KEY>
```

Extract from the ticket:
- **Bugzilla component(s)**: listed in the description (e.g. `"Toolkit: Crash Reporting"`)
- **Maintenance effectiveness goal**: typically stated as a percentage over a time period (e.g. ">=100% / 3 months")
- **Status**, **Assignee**, and any other relevant context

If the ticket doesn't mention Bugzilla components or a maintenance effectiveness goal, report that and stop.

## Step 2: Build the BugDash URL

Construct the BugDash overview URL from the Bugzilla components. Components use the format `Product:Component` with URL encoding:

```
https://bugdash.moz.tools/?component=<Product>%3A<Component>&...#tab.overview
```

Examples:
- `Toolkit::Crash Reporting` → `component=Toolkit%3ACrash+Reporting`
- `Core::Widget: Win32` → `component=Core%3AWidget%3A+Win32`

Multiple components are joined with `&component=...` for each.

## Step 3: Load BugDash in Firefox

Use the Firefox DevTools MCP with the persistent profile:

```
restart_firefox:
  firefoxPath: /usr/bin/firefox-nightly
  profilePath: /home/jens/.firefox-mcp-profile
  startUrl: <constructed URL>
```

Then call `list_pages` to trigger the launch, wait ~8 seconds for JS to render, and take a snapshot:

```
take_snapshot:
  includeText: true
  maxLines: 200
```

If the snapshot doesn't capture the maintenance trend data clearly, fall back to `screenshot_page` and read the image.

## Step 4: Extract the Numbers

From the snapshot/screenshot, extract:
- **Open Defects by Severity**: All, S1, S2, S3, S4, Unseveritied
- **Maintenance Trend** for each period (1 week, 4 weeks, 12 weeks):
  - Maintenance Effectiveness percentage
  - Burn Down
  - Opened: All, S1, S2, S3, S4, Un
  - Closed: All, S1, S2, S3, S4, Un

## Step 5: Report Status

Present the findings as a concise status report:

1. **Ticket summary**: what the KTLO ticket asks for
2. **Goal**: the maintenance effectiveness target and period
3. **Current numbers**: the maintenance trend table
4. **Assessment**: is the goal being met? If not, by how much and what would it take to close the gap?
5. **Notable patterns**: e.g. bugs piling up in a specific severity, burn down trajectory

Match the assessment period to the goal period (e.g. if goal says "3 months", use the 12-week row).

## Posting Comments to Jira

When asked to post the assessment as a Jira comment, write the comment as an ADF (Atlassian Document Format) JSON file and post it with `acli jira workitem comment create --key <KEY> --body-file /tmp/ktlo-comment.json`.

Plain text `--body` loses all formatting (newlines, links). Always use ADF via `--body-file`.

ADF template (use `table`, `tableRow`, `tableHeader`, `tableCell` nodes for tabular data — plain text with pipes does not render well):
```json
{
  "version": 1,
  "type": "doc",
  "content": [
    {
      "type": "heading",
      "attrs": {"level": 3},
      "content": [{"type": "text", "text": "Heading here"}]
    },
    {
      "type": "paragraph",
      "content": [
        {"type": "text", "text": "Plain text here"}
      ]
    },
    {
      "type": "paragraph",
      "content": [
        {"type": "text", "text": "Link label: "},
        {"type": "text", "text": "link text", "marks": [{"type": "link", "attrs": {"href": "https://example.com"}}]}
      ]
    },
    {
      "type": "table",
      "content": [
        {
          "type": "tableRow",
          "content": [
            {"type": "tableHeader", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "Col 1"}]}]},
            {"type": "tableHeader", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "Col 2"}]}]}
          ]
        },
        {
          "type": "tableRow",
          "content": [
            {"type": "tableCell", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "val 1"}]}]},
            {"type": "tableCell", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "val 2"}]}]}
          ]
        }
      ]
    }
  ]
}
```

Use `hardBreak` nodes for line breaks within a paragraph (not `\n` in text nodes).
