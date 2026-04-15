---
name: download-collection
type: task
version: 2.0.0
collection: agent-index-marketplace
description: Downloads a marketplace collection to the org's remote filesystem. Runs conflict detection before downloading. Downloads as ZIP and uploads to remote via aifs_write.
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

Downloads a collection from the marketplace to the org's remote filesystem via `aifs_write`. This is the first step before installation — the collection files need to be present on the remote filesystem before the setup interview can run.

Downloading does not configure the collection for the org. That is `install-collection`'s job. After downloading, the collection exists on the remote filesystem but is not yet usable by members.

### Inputs

Collection name — provided in the invocation or asked for if not provided.

### Outputs

- `/{collection-name}/` — collection directory created on the remote filesystem root (via `aifs_write`) with all collection files
- `org-config.json` — updated on the remote filesystem with new entry: `status: downloaded`, `install_method: zip`

---

## Workflow

### Step 1: Identify Collection

If the member named a collection in their invocation: use that name.
If not: ask "Which collection would you like to download? Say '@ai:list-marketplace-collections' to see what's available."

Ensure the marketplace cache is fresh: invoke `run agent-index-marketplace task refresh-marketplace-cache` in automatic mode.

Look up the collection name in `/shared/marketplace-cache/marketplace-directory.json`.

If not found: surface "'{name}' wasn't found in the marketplace. Check the name and try again, or say '@ai:list-marketplace-collections' to browse." Halt.

Check `org-config.json` — if this collection is already present with `status: installed`:
Surface: "'{display_name}' is already installed. To upgrade it, say '@ai:upgrade-collection {name}'." Halt.

If present with `status: downloaded`:
Surface: "'{display_name}' has already been downloaded but setup isn't complete. Say '@ai:install-collection {name}' to finish setting it up." Halt.

**On success:** Proceed to Step 2 with the collection's directory entry.

---

### Step 2: Run Conflict Detection

Read all currently installed collections from `org-config.json`. Read their `collection.json` files for category and API member names.

**Category conflict check:**
If any installed collection shares the same `category` as the collection being downloaded:

> "'{display_name}' is a {category} collection, and you already have '{existing_name}' installed in that category. Orgs typically run one {category} system. Would you like to:
> 1. Proceed anyway (install both)
> 2. Cancel this download"

Require explicit choice before proceeding.

**Name collision check:**
Compare the API members listed in the marketplace directory entry against all API members of all installed collections.

If collisions found: surface them now as a preview notice — not a blocker at download time, but the admin needs to know:

> "Heads up: '{display_name}' has {N} task/skill name(s) that overlap with your installed collections: {list of colliding names}. You'll be asked to assign aliases to resolve these during installation."

This is informational only at download time. Aliases are assigned during `install-collection`.

**On success:** Proceed to Step 3.

---

### Step 3: Download Method

In the remote filesystem model, collections are always downloaded as ZIP and uploaded to the remote filesystem. Git clone is not supported because the remote filesystem is accessed via `aifs_*` tools, not as a local mount.

Record `install_method: zip`.

---

### Step 4: Download

1. Download the ZIP from the collection's `zip_url` to a local temporary directory
2. Extract to the temporary directory
3. Rename the extracted folder to `{collection-name}` (GitHub appends `-main` to the folder name)
4. Upload all collection files to `/{collection-name}/` on the remote filesystem via `aifs_write`
5. Clean up local temporary files

If download or upload fails for any reason: surface the specific error. Do not write to `org-config.json`. The remote filesystem should be clean — if a partial directory was created on the remote, remove it via `aifs_delete` before reporting failure.

**On success:** Proceed to Step 5.

---

### Step 5: Verify Download

Confirm the collection uploaded correctly by checking that `collection.json` is readable on the remote filesystem via `aifs_read` at `/{collection-name}/collection.json` and that its `name` field matches the expected collection name.

If verification fails: surface the error, remove the partially uploaded directory from the remote filesystem via `aifs_delete`, do not update `org-config.json`. Halt.

**On success:** Proceed to Step 6.

---

### Step 6: Update org-config.json and Confirm

Add a new entry to `installed_collections` in `org-config.json`:

```json
{
  "name": "{name}",
  "display_name": "{display_name}",
  "version": "{version from collection.json}",
  "downloaded_date": "{today YYYY-MM-DD}",
  "repo_url": "{repo_url}",
  "zip_url": "{zip_url}",
  "install_method": "zip",
  "status": "downloaded"
}
```

Update `last_updated` on `org-config.json`.

Confirm to admin:
> "'{display_name}' v{version} has been downloaded ({install_method}). Say '@ai:install-collection {name}' to configure it for your org, or say '@ai:download-and-install-collection' to do both steps together next time."

---

## Directives

### Behavior

Conflict detection in Step 2 is informational for name collisions and confirmatory for category conflicts. Do not over-alarm the admin — a category conflict warning is appropriate; name collision previews should be matter-of-fact.

The download itself should feel fast and quiet. Progress can be noted ("Downloading...") but don't narrate every sub-step.

Always verify the download before updating `org-config.json`. The org-config entry is the record of truth — only write it when the download is confirmed good.

### Constraints

Never write to `org-config.json` before the download is verified in Step 5.

Never leave a partial or corrupted collection directory on the remote filesystem. If any step after directory creation fails, clean up via `aifs_delete` before reporting the error.

Never download `agent-index-core` or `agent-index-marketplace` through this task — those are managed separately.

### Edge Cases

If the remote filesystem root is not writable: check `aifs_auth_status()`. If `authenticated: false`, attempt automatic re-authentication via `aifs_authenticate` and retry the write. If re-auth fails or the write still fails: surface "The remote filesystem isn't writable. I tried to restore your connection but wasn't able to. Try '@ai:member-bootstrap' to troubleshoot." Halt.

If the collection's `zip_url` or `repo_url` is missing from the marketplace directory entry: surface "The download URL for '{name}' is missing from the marketplace directory. Try refreshing the cache with '@ai:refresh-marketplace-cache' and trying again."

If a directory with the collection name already exists on the remote filesystem but is not in `org-config.json`: surface "A directory named '{name}' already exists on the remote filesystem but isn't recorded in your org config. This may be a leftover from a previous failed download. Would you like to overwrite it or cancel?"
