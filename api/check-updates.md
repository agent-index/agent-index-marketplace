---
name: check-updates
type: task
version: 2.0.0
collection: agent-index-marketplace
description: Comprehensive update check across infrastructure, installed collections, and member capabilities — shows everything that has a newer version available and what to do about it.
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

The single command that answers "is anything out of date?" It checks three layers of the system in one sweep:

1. **Infrastructure** — is agent-index-core or agent-index-marketplace itself outdated?
2. **Installed collections** — do any marketplace collections have newer versions available?
3. **Member capabilities** — are any of the running member's installed skills and tasks behind their collection's current version?

The result is a clear, prioritized report showing what's current, what has updates available, and what action to take for each. No files are written — this is a read-only diagnostic.

Any org member can run this task, not just admins. Everyone should be able to see their own update status. However, only admins can act on infrastructure and collection-level updates.

**Relationship to the update instruction system:** This task and the `apply-updates` task serve different purposes. `check-updates` is a diagnostic — it scans live version data from GitHub, the marketplace, and the remote filesystem to show the full picture of what is out of date. `apply-updates` is an action — it reads admin-published update instructions and executes them. A member might run `check-updates` to understand their situation, and `@ai:update` to act on it. The two are complementary: `check-updates` can detect version drift that hasn't been published yet (e.g., a new marketplace version the admin hasn't installed), while `apply-updates` only acts on what the admin has explicitly published.

### Inputs

None required. Optionally, the member can request:
- `--infrastructure-only` — skip collection and capability checks, just check core and marketplace
- `--collections-only` — skip infrastructure and capability checks
- `--my-capabilities-only` — skip infrastructure and collection checks, just check the running member's installed skills and tasks
- `--quiet` — show only items that need attention (suppress "up to date" lines)

### Outputs

A formatted update report displayed to the member. No files written.

### Cadence & Triggers

On demand. Invoked when a member or admin wants to know if anything is out of date. Also invoked internally by session-start (in lightweight mode) to generate update-available notices.

---

## Workflow

### Step 1: Read System Configuration

Read `agent-index.json` from its fixed path. Extract:
- `version` — the installed core version
- `core_version_url` — where to check for the latest core version
- `marketplace_version_url` — where to check for the latest marketplace version
- `marketplace_cache_path` — where the marketplace directory cache lives

Read `org-config.json` from the remote filesystem via `aifs_read`. Extract:
- `installed_collections` — the list of collections the org has installed, with their versions and repo URLs
- `agent_index_version` — the core version recorded at org setup time

If `agent-index.json` is not readable: surface error and halt. This file is required.
If `org-config.json` is not readable: proceed with infrastructure checks only; skip collection checks.

**On success:** Proceed to Step 2.

---

### Step 2: Check Infrastructure Versions

**Check agent-index-core:**

Fetch `core_version_url` from agent-index.json. This returns the canonical `collection.json` for agent-index-core from GitHub. Parse the `version` field from the response.

Compare the fetched version against the installed version (from `agent-index-core/collection.json` on the remote filesystem via `aifs_read`).

Record the result:
- If fetched version > local version: `update available` (local → latest)
- If fetched version = local version: `up to date`
- If fetch fails: `unable to check` (note the error; do not block)

**Check agent-index-marketplace:**

Fetch `marketplace_version_url`. Same process — compare fetched version against the `agent-index-marketplace/collection.json` version on the remote filesystem via `aifs_read`.

Record the result with the same categories.

**On success:** Proceed to Step 3.
**On fetch failure for both:** Queue notice that infrastructure checks couldn't complete due to network issues. Proceed to Step 3.

---

### Step 3: Refresh Marketplace Cache and Check Collection Versions

Invoke `run agent-index-marketplace task refresh-marketplace-cache` in automatic mode to ensure the marketplace directory is fresh.

Read the marketplace directory from cache. For each collection in `org-config.json`'s `installed_collections` (excluding `agent-index-core` and `agent-index-marketplace`, which were checked in Step 2):

1. Find the matching entry in the marketplace directory by collection name
2. Compare the `current_version` in the directory against the `version` in `org-config.json`

Record the result for each:
- If directory version > installed version: `update available` (installed → latest)
- If directory version = installed version: `up to date`
- If collection not found in directory (org-authored collection): `org collection — no marketplace tracking`
- If marketplace cache unavailable: `unable to check`

Additionally, for each installed collection on the remote filesystem, read its `collection.json` via `aifs_read` and compare against `org-config.json`'s recorded version. If these differ, it means the remote filesystem was updated but `org-config.json` wasn't updated to match — flag as `version mismatch — remote filesystem differs from org-config`.

**On success:** Proceed to Step 4.

---

### Step 4: Check Member Capability Versions

Read the running member's `member-index.json`. For each installed skill and task:

1. Look up the collection it belongs to
2. Read the collection's `collection.json` from the remote filesystem via `aifs_read` to get its current version
3. Compare the `version` in the member index entry against the collection version

Record the result for each:
- If collection version > member version: `upgrade available` — the collection has been updated but the member's installed instance hasn't been upgraded yet
- If collection version = member version: `current`
- If collection `collection.json` is unreadable: `unable to check`

**On success:** Proceed to Step 5.

---

### Step 5: Present Update Report

**Check update instruction status:** Before compiling the report, read `/shared/updates/latest.json` from the remote filesystem via `aifs_read`. If it exists, compare `latest_id` against the member's `last_applied_update` from `member-index.json`. Record whether update instructions are pending — this influences the "What to do" section of the report.

Compile all results into a prioritized report.

**Full mode (default):**

> **Agent-Index Update Report**
> Checked: {timestamp}
>
> **Infrastructure**
> | Component | Installed | Latest | Status |
> |---|---|---|---|
> | agent-index-core | 1.0.0 | 1.1.0 | ↑ update available |
> | agent-index-marketplace | 1.0.0 | 1.0.0 | ✓ up to date |
>
> **Installed Collections**
> | Collection | Installed | Latest | Status |
> |---|---|---|---|
> | projects | 2.0.0 | 3.0.0 | ↑ update available |
> | strategy | 1.0.0 | 1.0.0 | ✓ up to date |
> | capture | 1.0.0 | 1.0.0 | ✓ up to date |
>
> **Your Installed Capabilities** ({N} total)
> | Capability | Type | Collection | Your Version | Collection Version | Status |
> |---|---|---|---|---|---|
> | create-project | task | projects | 2.0.0 | 3.0.0 | ↑ upgrade available |
> | edit-project | task | projects | 2.0.0 | 3.0.0 | ↑ upgrade available |
> | capture | task | capture | 1.0.0 | 1.0.0 | ✓ current |
> ...
>
> **Summary:** {N} updates available, {M} items up to date, {P} unable to check.
>
> **What to do:**
> {if update instructions are pending (last_applied_update behind latest.json)}: "Your admin has published update instructions. Say '@ai:update' to apply them — this will handle infrastructure, collection, and capability updates in one step."
> {if no update instructions pending but updates detected}:
> {if infrastructure updates}: "Infrastructure updates require an org admin. {if member is admin: Say '@ai:marketplace' to upgrade agent-index-core, then '@ai:publish-updates' to publish instructions for members. | if not admin: Contact your org admin to upgrade infrastructure and publish update instructions.}"
> {if collection updates}: "{if member is admin: Say '@ai:marketplace' to upgrade collections, then '@ai:publish-updates' to publish instructions for members. | if not admin: Contact your org admin to upgrade the {collection} collection and publish update instructions.}"
> {if capability upgrades}: "Say '@ai:update' if update instructions are available, or '@ai:setup' to upgrade your installed capabilities manually."

**Quiet mode** (`--quiet`):

Show only items with updates or issues. If everything is current:
> "Everything is up to date."

**Lightweight mode** (used internally by session-start):

Return a structured result (not displayed) containing:
```json
{
  "infrastructure_updates": [],
  "collection_updates": [{"name": "projects", "installed": "2.0.0", "latest": "3.0.0"}],
  "capability_upgrades": [{"name": "create-project", "collection": "projects", "installed": "2.0.0", "latest": "3.0.0"}],
  "errors": [],
  "everything_current": false
}
```

This structured result is consumed by session-start to generate update-available notices without displaying the full report.

**On success:** Task complete.

---

## Directives

### Behavior

This task is diagnostic — it tells you what's going on, clearly and completely. It never modifies anything. It never triggers an upgrade. It reports and recommends.

Be specific about what the member can do. If they're an admin, give them the exact command. If they're not an admin, tell them to contact their admin. Don't give a member actions they can't take.

When running in lightweight mode (triggered by session-start), be silent. Return the structured result and nothing else. Session-start will handle surfacing any notices.

### Version Comparison

Use semantic version comparison throughout. `1.2.0` > `1.1.9` > `1.1.0` > `1.0.0`. Compare MAJOR first, then MINOR, then PATCH. Do not use string comparison.

When an update involves a MAJOR version change, note this in the report:
> "↑ **major update** available (1.0.0 → 2.0.0) — may require migration"

MAJOR updates have breaking changes and may require running upgrade scripts. MINOR and PATCH updates are non-breaking.

### Constraints

Never modify any file. This task is strictly read-only.

Never trigger an upgrade. Only report and recommend.

Never suppress infrastructure update notifications. Even in quiet mode, if core or marketplace has an update available, show it.

Never show another member's capability status. Only the running member's own installed capabilities.

### Edge Cases

If the org has no installed collections (brand new org): skip the collections section. Only show infrastructure.

If the member has no installed capabilities (brand new member): skip the capabilities section. Show infrastructure and collections if the member is an admin, otherwise show a note: "You haven't installed any capabilities yet. Say '@ai:setup' to get started."

If the remote filesystem is unreachable: report what can be checked locally (capability versions vs. local member-index records) and note that infrastructure, collection, and marketplace version checks require remote filesystem connectivity.

If `org-config.json` records a collection that no longer exists on the remote filesystem: flag it as `missing — directory not found` (consistent with `list-org-collections` behavior).

If the member invokes this task with a specific collection name (e.g., "check updates for projects"): show only that collection's status plus the member's capabilities from that collection. Skip everything else.
