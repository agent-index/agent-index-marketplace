---
name: list-marketplace-collections
type: task
version: 2.0.0
collection: agent-index-marketplace
description: Shows all collections available in the agent-index marketplace, with download and install status for each.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks:
    - refresh-marketplace-cache
external_dependencies: []
reads_from: null
writes_to: null
---

## About This Task

The marketplace catalog view. Shows every collection available in the marketplace, grouped by category, with a clear status indicator for each — whether it's new to the org, already downloaded, or fully installed.

This is typically the starting point when an org admin wants to add new capabilities to their org.

### Inputs

None required. Optional filter by category or search term if the member provides one.

### Outputs

A formatted display of available collections. No files written.

---

## Workflow

### Step 1: Ensure Fresh Cache

Invoke `run agent-index-marketplace task refresh-marketplace-cache` in automatic mode.

Proceed with whatever cache state is available after the refresh attempt.

---

### Step 2: Read Installed Collections State

Read `org-config.json` from the remote filesystem via `aifs_read`. Extract the `installed_collections` array.

Build a lookup map: collection name → `{version, status, install_method}`.

This tells us which collections are `downloaded` (present on the remote filesystem, not yet set up) vs `installed` (downloaded and setup complete) vs not present at all.

---

### Step 3: Read and Enrich Marketplace Directory

Read `/shared/marketplace-cache/marketplace-directory.json`.

For each entry in the directory, determine its status relative to this org:

| Condition | Status Label |
|---|---|
| Not in `org-config.json` | `available` |
| In `org-config.json` with `status: downloaded` | `downloaded — not installed` |
| In `org-config.json` with `status: installed`, version matches current | `installed` |
| In `org-config.json` with `status: installed`, version behind current | `installed — update available` |

---

### Step 4: Apply Filter (If Provided)

If the member provided a category filter or search term in their invocation, apply it now. Filter by `category` for category filters. Filter by name, description, or tags for search terms (case-insensitive substring match).

If no filter provided: show all collections.

---

### Step 5: Display

Present the catalog grouped by category. Within each category, sort featured collections first, then alphabetically.

Format:

> **Marketplace Collections**
> Cache last updated: {last_fetched} {if stale: — "say '@ai:refresh-marketplace-cache' to update"}
>
> **Project Management**
> ✓ Projects v1.0.0 — installed
>   Create, manage, and archive projects across your org.
>
> **HRIS**
> ↓ BambooHR Replacement v2.1.0 — available
>   Full HR management including employee records, time-off, and onboarding.
>
> **ATS**
> ⬇ Greenhouse Replacement v1.3.0 — downloaded, not installed
>   Applicant tracking and recruiting workflow management.

Status icons:
- `✓` — installed (current version)
- `↑` — installed, update available
- `⬇` — downloaded, not installed
- `↓` — available, not downloaded

After the list, offer actions:
> "Say '@ai:download-and-install-collection' followed by a collection name to add it, or ask me about any collection for more details."

---

## Directives

### Behavior

If the member asks for details about a specific collection before downloading: provide the full description, list of included API skills and tasks (from the directory entry if available), license, author, and external dependencies. Give the admin what they need to make an informed decision.

If the cache is stale but a refresh failed: display the list with a clear notice at the top that the information may be out of date and when it was last refreshed. Never refuse to display because the cache is stale.

If the marketplace directory is empty or unreadable: surface clearly — "The marketplace directory isn't available right now. Try '@ai:refresh-marketplace-cache' to fetch it."

### Constraints

Never display `agent-index-core` or `agent-index-marketplace` in this list — infrastructure collections are not user-installable through the marketplace.

Never show collections that don't meet the minimum agent-index version requirement for this org's current version.

### Edge Cases

If a collection is in `org-config.json` but not in the marketplace directory (removed from marketplace or org-authored): include it in `list-org-collections` output only, not here.

If the org has no collections installed yet: show the full catalog with a helpful prompt at the top: "Your org hasn't installed any collections yet. Here's what's available:"
