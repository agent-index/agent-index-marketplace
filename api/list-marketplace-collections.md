---
name: list-marketplace-collections
type: task
version: 2.1.1
collection: agent-index-marketplace
description: Shows all collections available across the org's subscribed marketplace catalogs, grouped by catalog, with download and install status for each.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies: []
reads_from: null
writes_to: null
---

## About This Task

The marketplace catalog view. Shows every collection available across the catalogs this org subscribes to, grouped by catalog and then by category, with a clear status indicator for each — whether it's new to the org, already downloaded, or fully installed.

This is typically the starting point when an org admin wants to add new capabilities to their org. Catalogs are admin-only (core 3.30.0): they live in the admin's local clones.

### Inputs

None required. Optional filter by category or search term if the member provides one.

### Outputs

A formatted display of available collections. No files written.

---

## Workflow

### Step 1: Resolve Subscribed Catalogs

Follow `/internal/resolve-marketplaces.md` (added in 2.20.0). It reads every **enabled** subscription in `org-config.json` → `marketplaces[]` (or the synthesised legacy public subscription on a pre-3.30.0 org), verifies catalog identity and namespaces, and returns merged, source-tagged entries.

- `status: "not_admin"` → surface the resolver's admin-only message and halt.
- `status: "error"` → surface every named error and its remedy, and **halt without listing anything**. Do not fall back to another source, to `/shared/marketplace-cache/`, or to the web.
- `status: "ok"` → proceed. Keep `sources[]` (for the header and any `skipped` notices) and `disabled[]`.

This task no longer invokes `refresh-marketplace-cache` and never reads `/shared/marketplace-cache/` (decommissioned; `mktcatalogwebfetch`). For the public catalog on a clone-publishing org, freshness comes from the admin's clone refresh (the committed `lib/clone/clone-repos` script), not from a cache TTL.

---

### Step 2: Read Installed Collections State

From the `org-config.json` already read by the resolver, extract `installed_collections[]`.

Build a lookup map: collection name → `{version, status, marketplace_id}`.

This tells us which collections are `downloaded` (present on the remote filesystem, not yet set up) vs `installed` (downloaded and setup complete) vs not present at all, and which catalog each came from.

---

### Step 3: Enrich Catalog Entries

For each resolver entry, determine its status relative to this org:

| Condition | Status Label |
|---|---|
| Not in `org-config.json` | `available` |
| In `org-config.json` with `status: downloaded` | `downloaded — not installed` |
| In `org-config.json` with `status: installed`, version matches this entry's `current_version` | `installed` |
| In `org-config.json` with `status: installed`, version behind this entry's `current_version` | `installed — update available` |

Compare versions **only against the entry from the collection's own origin catalog** (`installed_collections[].marketplace_id`). With unique names enforced a name appears in at most one catalog unless it is marked `conflict`, so this is normally the same entry; if the installed entry's `marketplace_id` differs from the catalog the entry was found in, show `installed from {origin display name}` and do not claim an update.

---

### Step 4: Apply Filter (If Provided)

If the member provided a category filter or search term in their invocation, apply it now. Filter by `category` for category filters. Filter by name, description, or tags for search terms (case-insensitive substring match).

If no filter provided: show all collections.

---

### Step 5: Display

Present one section **per catalog**, in `sources[]` order with `agent-index-public` last — so a small private catalog is never buried beneath a large public one. Within a catalog, group by category; within a category, featured first, then alphabetical. Omit a catalog section that has no entries after filtering.

If only one catalog is subscribed, omit the per-catalog heading — the output must match pre-2.20.0 exactly except for the header line (backwards compatibility).

Format:

> **Marketplace Collections**
> Catalogs: {display_name} v{directory_version} ({last_updated}) · …
> {for each skipped source: "⚠ {display_name} couldn't be read ({error}) — skipped because it is set to skip when unavailable."}
>
> **CX Studio Catalog** · namespace `cx`
>
> **Client Delivery**
> ↓ CX Studio v3.0.4 — available
>   Client-facing experience design studio.
>
> **Agent Index Marketplace**
>
> **Project Management**
> ✓ Projects v4.3.0 — installed
>   Create, manage, and archive projects across your org.

Status icons:
- `✓` — installed (current version)
- `↑` — installed, update available
- `⬇` — downloaded, not installed
- `↓` — available, not downloaded

**Conflicts (2.21.0).** An entry marked `conflict` is shown in each catalog's section with `⚠` and "also offered by {other display names} — resolve before installing". If `conflicts[]` has any `namespace_intrusion`, add one line per intrusion after the list: "{catalog} lists `{name}`, but `{ns}-` is reserved by {reserving catalog} — not shown." Conflicts never hide the rest of a catalog.

If `disabled[]` is non-empty, add one line after the list: "Not shown: {display names} (disabled — '@ai:edit-org' → Manage marketplaces to re-enable)."

After the list, offer actions:
> "Say '@ai:download-and-install-collection' followed by a collection name to add it, or ask me about any collection for more details."

---

## Directives

### Behavior

If the member asks for details about a specific collection before downloading: provide the full description, list of included API skills and tasks (from the directory entry if available), license, author, and external dependencies. Give the admin what they need to make an informed decision.

Show each catalog's `directory_version` and `last_updated` in the header so the admin can judge freshness. If the public catalog looks behind what they expect, the remedy is refreshing the clones with the committed `lib/clone/clone-repos` script — not a web fetch.

If a catalog is unreadable and not set to skip: the resolver aborts; surface its named errors and remedies and list nothing. A partial catalog must never be presented as complete.

### Constraints

Never display `agent-index-core` or `agent-index-marketplace` in this list — infrastructure collections are not user-installable through the marketplace.

Never show collections that don't meet the minimum agent-index version requirement for this org's current version.

### Edge Cases

If a collection is in `org-config.json` but not in any subscribed catalog (sideloaded, `marketplace_id: null`, or its catalog is disabled/unsubscribed): include it in `list-org-collections` output only, not here.

If the org has no collections installed yet: show the full catalog with a helpful prompt at the top: "Your org hasn't installed any collections yet. Here's what's available:"

<!-- AIFS:FILE-END -->
