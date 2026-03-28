---
name: list-org-collections
type: task
version: 2.0.0
collection: agent-index-marketplace
description: Shows all collections the org has downloaded or created, with install status for each. Does not show marketplace collections that haven't been downloaded.
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

The org's collection view. Unlike `list-marketplace-collections`, this task shows only what the org actually has on the remote filesystem — downloaded marketplace collections (installed or not) and org-authored collections. It reads from the remote filesystem via `aifs_read` and `aifs_list` but does not require a marketplace cache check.

This is the admin's operational view: what do we have, what state is it in, does anything need attention?

### Inputs

None required.

### Outputs

A formatted display of org collections. No files written.

---

## Workflow

### Step 1: Read org-config.json

Read `org-config.json` from the remote filesystem via `aifs_read`. Extract `installed_collections`.

If `org-config.json` cannot be read: surface "Org configuration isn't readable. Try '@ai:member-bootstrap' to check remote filesystem connectivity." Halt.

---

### Step 2: Scan Remote Filesystem Root

List all directories at the remote filesystem root via `aifs_list`. For each directory, attempt to read its `collection.json` via `aifs_read`.

Cross-reference with `org-config.json`:

- **In `org-config.json` and on remote filesystem:** confirmed present
- **In `org-config.json` but not on remote filesystem:** orphaned record — collection was recorded but directory is missing
- **On remote filesystem with valid `collection.json` but not in `org-config.json`:** unregistered collection — present but not recorded

Skip `agent-index-core` and `agent-index-marketplace` — these are infrastructure, not user collections.

---

### Step 3: Categorize Collections

Split collections into two groups:

**Marketplace collections** — those with a `marketplace_url` in their `collection.json` or a matching entry in `org-config.json` with a `repo_url`. These were downloaded from the marketplace.

**Org collections** — those without a marketplace URL. These were authored by the org.

Within each group, determine status per collection:

| Condition | Status |
|---|---|
| `status: installed` in org-config, version matches filesystem | `installed` |
| `status: installed` but version in org-config differs from filesystem | `version mismatch — may need attention` |
| `status: downloaded` | `downloaded — setup not complete` |
| On filesystem but not in org-config | `unregistered` |
| In org-config but not on remote filesystem | `missing — directory not found` |

---

### Step 4: Display

> **Your Org's Collections**
>
> **Marketplace Collections**
> ✓ projects v1.0.0 — installed (zip)
>   4 tasks: create-project, edit-project, archive-project, unarchive-project
>
> ⬇ greenhouse-replacement v1.3.0 — downloaded, setup not complete
>   Say '@ai:install-collection greenhouse-replacement' to complete setup.
>
> **Org Collections**
> ✓ acme-corp-hr-ops v2.0.0 — installed
>   3 tasks, 2 skills
>
> {if any unregistered}: ⚠ unregistered-collection — present on remote filesystem but not recorded in org config. Say '@ai:edit-org' to investigate.
>
> {if any missing}: ⚠ some-collection — recorded in org config but directory not found. The collection may have been deleted manually.

After the list, offer relevant actions based on what is shown:
- If any downloaded but not installed: "Say '@ai:install-collection {name}' to complete setup."
- If any missing: "Contact your org admin to investigate missing collections."
- If nothing installed yet: "Say '@ai:list-marketplace-collections' to browse what's available."

---

## Directives

### Behavior

This is an operational status view. Be precise about what each status means and what the admin should do about it. Anomalous states (unregistered, missing, version mismatch) should stand out — don't bury them at the bottom.

This task reads from the remote filesystem but never triggers a marketplace cache refresh. If the admin wants to see what's available in the marketplace, direct them to `list-marketplace-collections`.

### Constraints

Never modify `org-config.json` or any collection directory from within this task. It is read-only.

Never show infrastructure collections (`agent-index-core`, `agent-index-marketplace`) in this list.

### Edge Cases

If `org-config.json` exists but `installed_collections` is an empty array and no non-infrastructure collection directories are found: surface "Your org hasn't downloaded any collections yet. Say '@ai:list-marketplace-collections' to browse what's available."

If a collection directory exists and has a `collection.json` but the JSON is malformed: list it as `present but unreadable — collection.json may be corrupted` and suggest the admin investigate.
