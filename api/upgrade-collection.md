---
name: upgrade-collection
type: task
version: 1.0.0
collection: agent-index-marketplace
description: Upgrade an already-installed marketplace collection to a newer version. Fetches new files from the registry-declared zip_url, uploads to remote, updates org-config.json, writes a CHANGELOG entry, and preserves per-org setup-responses. Closes the @ai:upgrade-collection alias that download-collection and download-and-install-collection have referenced since v3.1.0.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks:
    - refresh-marketplace-cache
external_dependencies: []
reads_from: null
writes_to: "/org-config.json, /<collection>/, /shared/updates/"
---

## About This Task

Upgrades an already-installed marketplace collection to a newer version. Counterpart to `@ai:publish-updates --check-upstream` (which handles infrastructure: agent-index-core, agent-index-marketplace) and `@ai:edit-org` Step 5 (which handles filesystem adapter bundles). After this task lands, every category of installed agent-index resource has a coherent admin-side upgrade verb:

| Category | Upgrade verb |
|---|---|
| Infrastructure (`agent-index-core`, `agent-index-marketplace`) | `@ai:publish-updates --check-upstream` |
| Filesystem adapter (`agent-index-filesystem-gdrive`, etc.) | `@ai:edit-org → Update Adapter Bundle` |
| Marketplace collection (`projects`, `developer`, `email-triage`, etc.) | **`@ai:upgrade-collection <name>`** ← this task |

Closes bug `20260502-8d20ea22` — the `download-collection` and `download-and-install-collection` tasks have halted with "say `@ai:upgrade-collection {name}`" since at least v3.1.0, but no such task existed. Now it does.

### Inputs

- `<collection_name>` (required, positional) — name of an installed marketplace collection.
- `--to <version>` (optional) — target a specific version. Default: latest in `marketplace-directory.json`.
- `--check-upstream` (optional) — refresh marketplace cache before reading the directory. Default: skip if cache is fresh (per `refresh-marketplace-cache`'s freshness rule).
- `--dry-run` (optional) — show the upgrade plan without applying.

### Outputs

- `/<collection>/` — uploaded with new version's files (overwriting existing) on the remote filesystem.
- `/org-config.json` — `installed_collections[<name>].version` updated to the new version, `installed_collections[<name>].upgraded_date` set.
- `/shared/updates/update-log.json` — new `collection-update` entry written for members to consume via `@ai:apply-updates`.

### Cadence & Triggers

On demand. Org admins say `@ai:upgrade-collection <name>` (or natural-language equivalents like "upgrade developer", "update the projects collection", "refresh email-triage to latest").

---

## Workflow

### Step 1: Verify Admin and Identify Collection

Resolve the running member's `member_hash` (SHA-256 of lowercase email, first 16 hex chars). Read `org-config.json` from remote via `aifs_read("/org-config.json")`. If the member is not in `admins[].member_hash`, surface:

> "Only org admins can upgrade collections. The current admins are: {admin list}. Contact one of them, or run `@ai:check-updates` to see what's available."

Halt.

If `<collection_name>` was not provided in the invocation: ask "Which collection would you like to upgrade?" and list the installed marketplace collections from `org-config.json` → `installed_collections[]`.

Look up the collection in `installed_collections[]`. If not present, surface:

> "`{name}` is not installed. To install it, run `@ai:download-and-install-collection {name}`. To see installed collections, say `@ai:list-org-collections`."

Halt.

### Step 2: Refresh Marketplace Cache (Conditional)

If `--check-upstream` was passed OR the marketplace cache is older than 1 hour, invoke `run agent-index-marketplace task refresh-marketplace-cache` in automatic mode. Otherwise use the cached `marketplace-directory.json` at `/shared/marketplace-cache/marketplace-directory.json`.

If the cache cannot be loaded after refresh (network or registry issue): surface the error and halt.

### Step 3: Look Up Target Version

Find `<collection_name>` in `marketplace-directory.json` `collections[]`. If not present:

> "`{name}` is no longer in the marketplace directory. The collection may have been deprecated by its author. Your installed copy continues to work, but no upgrades are available. To remove it, manually edit `org-config.json` (a future `@ai:remove-collection` task is planned but not yet implemented)."

Halt.

Resolve target version:

- If `--to <version>` was provided: validate that `<version>` exists in the directory's `current_version` field OR in any `version_history[]` array (if the directory schema includes one in the future). If unknown, surface "Unknown version `{version}` for `{name}`. The directory's current_version is `{x}`. Pass `--to {x}` or omit `--to` for latest."
- Otherwise: use the directory's `current_version`.

If `target_version <= installed_version`: surface "`{name}` is already at version `{installed}` (target was `{target}`). Nothing to do." Halt cleanly.

If the directory entry has a `min_required_version` field and `target_version < min_required_version`: surface "Cannot upgrade `{name}` to `{target}` — directory says minimum allowed version is `{min}`. Try `--to {min}` or omit `--to`." Halt.

### Step 4: Detect Migration Needs

If the upgrade crosses a MAJOR version boundary (e.g., installed 1.x → target 2.x), a migration script may be required. Check for `aifs_read("/<collection>/upgrade/{from_major}-to-{target_major}.md")` after Step 5's download — if such a file exists, it gets surfaced to the admin during Step 7's confirmation as a "may require attention" note. Same-MAJOR upgrades (1.2.3 → 1.3.0) skip this concern.

For multi-MAJOR jumps (1.x → 3.x), chain migration scripts: 1-to-2 then 2-to-3. The chained scripts get presented as a sequence in the confirmation summary.

### Step 5: Download and Stage

Download the collection's zip from the marketplace directory's `zip_url` to a local staging directory under `<project_dir>/.agent-index/staging/upgrade-{collection}-{ISO-timestamp}/`. Extract the zip in place.

Verify the extracted tree contains `collection.json` with `version` matching `target_version`. If mismatch (e.g., the GitHub `main` branch has a different version than the directory advertises), surface the discrepancy and ask "Proceed with the actual zip version `{actual}`, or halt?"

### Step 6: Compute Diff Against Remote

For each file in the staged tree, compare against the corresponding remote file at `/<collection>/<relative_path>`:

- `local_only` — file exists in staged tree, not at remote → upload
- `differs` — file exists in both, content differs → upload (overwrite)
- `synced` — content matches → no-op
- `remote_only` — file exists at remote but not in staged tree → flag (typically these are the per-org `setup/collection-setup-responses.md` and similar)

**Preserve list (do NOT overwrite or delete):**
- `setup/collection-setup-responses.md` — per-org setup answers from `install-collection`'s setup interview. Org-specific state, not collection content.
- Any file under `/<collection>/state/` if such a path is present at remote — collection-managed state, not part of the upstream artifact.

The staging-vs-remote diff result is the "upgrade plan." If counts are `upload: 0, delete: 0, synced: N`: surface "`{name}` is already at the target version's content; nothing to upload." Halt cleanly.

### Step 7: Present Plan and Confirm

Surface the plan summary to the admin:

```
Upgrade plan for {collection}:
  current version: {installed_version}
  target version:  {target_version}
  source URL:      {zip_url}

Files to upload:    {N}
Files to delete:    {M}  (excluding preserve-list)
Files preserved:    {P}  (e.g., setup/collection-setup-responses.md)
Files synced:       {S}

{if migration script present}
Migration:
  This is a MAJOR-version upgrade. The collection ships an upgrade script at
  /<collection>/upgrade/{from_major}-to-{target_major}.md that may need attention
  during member-side @ai:apply-updates. Members will see the script's instructions
  during their upgrade.

Proceed with upgrade? [Y/N]
```

If `--dry-run`: halt here, do not proceed.

On `N`: surface "Upgrade cancelled. No changes made." Halt cleanly.

### Step 8: Upload to Remote

For each `local_only` and `differs` file in the diff, upload via `aifs_write("/<collection>/<relative_path>", <content>)`. Apply LF normalization (per `apply-updates` Phase 1 step 6 convention) for text-shaped files (`.md`, `.json`, `.html`, `.yaml`, `.yml`, `.sh`, `.js`).

For each `remote_only` file NOT in the preserve list: delete via `aifs_delete("/<collection>/<relative_path>")`.

Surface a one-line confirmation per file: `✓ upload <path>` / `✓ delete <path>`.

If any single file write fails: surface the error and halt. The remote is left in a partial state — the next `@ai:upgrade-collection` re-run will continue from where the previous one left off (since the diff will reflect the partial-upload state).

### Step 9: Update org-config.json

Read `org-config.json`, mutate:

- `installed_collections[<name>].version ← <target_version>`
- `installed_collections[<name>].upgraded_date ← <today YYYY-MM-DD>`
- `installed_collections[<name>].upgraded_by ← <admin's member_hash>`

Write back via `aifs_write("/org-config.json", ...)`.

### Step 10: Write CHANGELOG Entry

Append a new entry to `/shared/updates/update-log.json`'s `entries[]` array:

```json
{
  "id": "<next sequential id>",
  "type": "collection-update",
  "collection": "<name>",
  "from_version": "<installed_version>",
  "target_version": "<target_version>",
  "has_migration": <true if migration script present>,
  "published_date": "<today ISO>",
  "published_by": "<admin's member_hash>",
  "summary": "<read collection.json's description field, or fall back to default>"
}
```

Update `/shared/updates/latest.json` `latest_id` to the new entry's id. Members consume this entry via `@ai:apply-updates` (Phase 4 collection-update handler).

### Step 11: Verify and Surface Result

Verify post-upload by reading `/<collection>/collection.json` from remote — confirm `version` matches `target_version`. If mismatch, surface a warning: "Verification failed: `/<collection>/collection.json` reports `{actual}` but expected `{target}`. The upgrade may have been only partially applied. Review the upload log above and consider re-running `@ai:upgrade-collection {name}` to retry."

On success surface:

```
✓ Upgraded {name} from {installed_version} to {target_version}
  Files: {N} uploaded, {M} deleted, {P} preserved
  Members will pick up the upgrade on their next @ai:apply-updates
{if migration script: "  Members will see the migration script's instructions during their upgrade."}
```

### Step 12: Cleanup

Delete the staging directory `<project_dir>/.agent-index/staging/upgrade-{collection}-{timestamp}/` if all upload steps succeeded. Leave it in place if any step failed (for debugging).

---

## Failure Modes

| Failure | Recovery |
|---|---|
| Marketplace cache unreachable | Surface and halt. Admin retries with `--check-upstream` after fixing connectivity. |
| Zip download fails | Skip-or-halt prompt. Admin retries. |
| Zip is corrupt or extract fails | Halt. Staging dir is left for debugging. |
| Single `aifs_write` fails partway through Step 8 | Surface error, halt. org-config NOT yet updated (Step 9). Re-running diffs against the partial-upload state and continues from where it left off. |
| Verification mismatch in Step 11 | Surface warning. Suggest re-run. Org-config has already been updated in Step 9 (since uploads succeeded), so the next run will see no diff if the partial-upload state matches the target — verification confirms cleanly on retry. |

## Directives

### Preserve setup-responses

The per-org setup-responses file at `/<collection>/setup/collection-setup-responses.md` (and any future per-org state files under `/<collection>/state/`) MUST NOT be overwritten or deleted by this task. They contain answers the org provided during the original `install-collection` setup interview — org-specific data, not collection content. Re-running setup is disruptive and shouldn't be a side effect of upgrading.

### Never call aifs_share / aifs_unshare / aifs_transfer_ownership directly

This task doesn't modify permissions. If a future enhancement requires permission changes (e.g., a collection upgrade needs to grant additional read access on shared collection files), the change MUST go through the `permission-change-helper` skill per `agent-index-core/standards.md`.

### Don't invoke this task from inside another task

`upgrade-collection` is admin-facing. If a future task needs to programmatically trigger an upgrade (e.g., a "bulk upgrade everything" admin task), it should compose with this task at the agent-orchestration layer (the agent invokes both in sequence with confirmations between), not call into `upgrade-collection`'s internals.
