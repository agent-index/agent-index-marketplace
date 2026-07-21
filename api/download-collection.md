---
name: download-collection
type: task
version: 2.6.0
collection: agent-index-marketplace
description: Downloads a marketplace collection to the org's remote filesystem. Runs conflict detection before downloading. Sources the collection from the admin's tag-pinned LOCAL GIT CLONE (Release-C backend-first; never a GitHub web fetch) and uploads to remote via aifs_write_batch (single-process bulk upload; chunked per-file fallback only when the adapter lacks the batch op).
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

Downloads a collection from the marketplace to the org's remote filesystem via `aifs_write_batch` (one-process bulk upload). This is the first step before installation — the collection files need to be present on the remote filesystem before the setup interview can run.

Downloading does not configure the collection for the org. That is `install-collection`'s job. After downloading, the collection exists on the remote filesystem but is not yet usable by members.

### Inputs

Collection name — provided in the invocation or asked for if not provided.

### Outputs

- `/{collection-name}/` — collection directory created on the remote filesystem root (via `aifs_write`) with all collection files
- `org-config.json` — updated on the remote filesystem with new entry: `status: downloaded`, `install_method: git-clone`

---

## Workflow

### Step 1: Identify Collection

If the member named a collection in their invocation: use that name.
If not: ask "Which collection would you like to download? Say '@ai:list-marketplace-collections' to see what's available."

**Locate the catalog (self-distributing orgs: read the LOCAL clone, never the web — `mktcatalogwebfetch`).** This org is self-distributing: the authoritative marketplace catalog is the admin's local `agent-index-resource-listings` clone (`marketplace-directory.json`), kept current by the clone scripts (git) — NOT the public directory. **Read `marketplace-directory.json` from the local `resource-listings` clone; do NOT invoke `refresh-marketplace-cache` (which web-fetches the public directory) and do NOT WebSearch / raw-fetch the directory.** This is the same rule `publish-updates` M1 already enforces for version/directory discovery (`adminupstreamstale`). Only an org that genuinely *consumes* from the public marketplace directory uses `refresh-marketplace-cache`'s SHA-pinned web fetch. (Bug `mktcatalogwebfetch`: the add-collection flow web-fetched the catalog first and hard-failed when the web was correctly blocked; the admin had to steer it back to the local clone.)

Look up the collection name in the local `resource-listings` clone's `marketplace-directory.json` (or, for a public-directory-consuming org, the refreshed `/shared/marketplace-cache/marketplace-directory.json`).

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

### Step 3: Source Method (Release-C: backend-first, from a local git clone)

**Collections are sourced from the admin's tag-pinned LOCAL GIT CLONE, then uploaded to the remote via `aifs_*`. Never fetch collection source from `github.com`/`codeload.github.com` on the web.** (The remote filesystem is written via `aifs_*` — that is the *upload* channel; it is not the *source*. The source is always a local clone, per standards.md § "Distribution: backend-first".)

Determine the local source clone:

1. If the admin already has a tag-pinned local clone of `{collection-name}` (collections selected at create-org are cloned during bring-up), use it. Confirm it's checked out to the org's adopted tag.
2. **If there is no local clone** (e.g. the collection was not selected at create-org — this is the common `download-collection` case), do **not** fall to the web. Emit a clone **manifest** via the `clone-script-generator` subroutine and have the admin run the committed `agent-index-core/lib/clone/clone-repos` script (which reads it) to clone `{collection-name}` at the org's adopted version, then use that clone. **DO NOT author a bespoke `clone-{collection-name}.ps1`/`.sh` (or copy an existing clone-*.ps1) -- that is the `clonescripttagassumption` regression; the ONLY sanctioned path is the data manifest + the COMMITTED `lib/clone/clone-repos`. If a bespoke clone script would be written, STOP and emit the manifest instead.**

Record `install_method: git-clone` and the resolved source tag.

The `zip_url`/web path survives **only** as the deprecated fallback for a not-yet-migrated org; if it is ever used, emit the standards.md deprecation warning. It is never the default and never used when a clone can be produced.

---

### Step 4: Download

1. **Release C — source the collection from the admin's LOCAL CLONE, not a GitHub download.** Adding a collection is a `git clone`/`pull` via the committed `agent-index-core/lib/clone/clone-repos` script driven by a `clone-script-generator` manifest (the admin runs the committed script). Read the collection files from the local `{collection-name}` clone (checked out to the org's adopted tag). **Do not fetch `zip_url` from GitHub** — that path survives only as the deprecated fallback for a not-yet-migrated org (standards.md § "Distribution: backend-first"; emit the deprecation warning if used).
2. Upload all collection files from the local clone to `/{collection-name}/` on the remote filesystem **via `aifs_write_batch`, in fixed conservative sub-batches of 10 files per call** (`bulkuploadserial` / `instcollnobatch`, gdrive adapter 2.9.0+ / onedrive 2.6.0+). The batch op removes the per-file process-spawn overhead of a naive `aifs_write` loop, but the TOTAL work inside a single `aifs_write_batch` call still runs within one ~45-second per-invocation window — so an unbounded batch (the whole tree, or "two big chunks") still times out. Partition the in-scope file list into fixed sub-batches of **10 files (the default; state this number explicitly to the admin)** and issue **one `aifs_write_batch` call per sub-batch** (each call is one process). For each sub-batch, build an `entries` array of `{path, content_file}` (prefer the `content_file` form over inline `content` for every file — inline is capped by the shell argument length, ~128KB) and pass it to `aifs_write_batch`; it ensures the unique parent folders once per call (duplicate-parent-safe) and runs each file's durable read-back verify inside that one process. **After each sub-batch call, confirm the sub-batch landed** — rely on the batch op's own read-back, or do a per-sub-batch `aifs_exists` check on that group's files — before starting the next sub-batch. **This upload is resume-safe and idempotent: a re-run skips files already present on the remote, so a partial upload (some sub-batches landed, then a timeout) can be safely re-run and it picks up where it left off.** **Do not upload the whole tree, or "two big chunks," in one batch call.**

   **This supersedes the C.1.4.3 "single big batch" behavior** (which built ONE `entries` array over all files and only chunked "if very large"). Motivation (`instcollnobatch`): a 66-file client-intelligence collection uploaded as the whole tree / two large chunks STILL overran the ~45-second window — only ~20 files landed before timeout — and completed only after being manually re-chunked to ~11 files per batch. The fixed small sub-batch removes the need for that manual re-chunking.

   **Check the adapter advertises the batch op before choosing this path.** Read the adapter's `supported_operations` (from `adapter.json` / the aifs capabilities the adapter reports) and confirm `writeBatch` is listed. If `writeBatch` IS advertised, use `aifs_write_batch` per sub-batch as above. If it is NOT advertised (an older deployed adapter that predates the batch op), fall back to per-file `aifs_write` — but issue the writes in the **same fixed sub-batches of 10 files**, verifying each sub-batch landed (and skipping files already present) before starting the next group, NOT one giant sequential loop over all files. Chunking the fallback this way keeps a large collection from timing out mid-upload the way the un-chunked loop did (`instcollnobatch`).

   **Deployment note:** if the adapter reports no batch op (`writeBatch` absent from `supported_operations`) despite being on a version that *should* advertise it (>= gdrive 2.10.0, or the onedrive equivalent), that is a deployment/packaging issue — a stale or mis-built adapter bundle — and should be checked separately (e.g. via `@ai:verify-workspace-policy` or by re-staging the adapter). Do not treat it as normal: the chunked fallback keeps the upload working, but an adapter on that version should be advertising the op.
3. Re-publish `/shared/dist/` (manifest + any changed directories) per the `backend-distribution` subroutine, so the org's version authority reflects the new collection.

If download or upload fails: classify the failure shape first (added in core 3.7.4 to close section D of idea `allowlist-failure-mode-warnings-in-tasks`):

- **Allowlist-blocked signature on the download** (HTTP 403 with empty body and no upstream-server headers, OR connection-refused, OR connection-timeout against `codeload.github.com` or `github.com`): surface the canonical Allowlist Failure Recognition message (see `agent-index-core/collection-authoring-guide.md` § "Allowlist failure recognition") naming `codeload.github.com` as the blocked host (or `github.com` if that's where the request was directed). Recommend `@ai:verify-network-allowlist` to test all required hosts at once. Halt without writing to `org-config.json`.
- **Other download/upload failure** (real HTTP error from the destination, DNS failure, partial-write failure, etc.): surface the specific error. The remote filesystem should be clean — if a partial directory was created on the remote, remove it via `aifs_delete` before reporting failure. Do not write to `org-config.json`.

**On success:** Proceed to Step 5.

---

### Step 5: Verify Download

Confirm the collection uploaded correctly by checking that `collection.json` is readable on the remote filesystem via `aifs_read` at `/{collection-name}/collection.json` and that its `name` field matches the expected collection name.

If verification fails: surface the error, remove the partially uploaded directory from the remote filesystem via `aifs_delete`, do not update `org-config.json`. Halt.

**On success:** Proceed to Step 6.

---

### Step 6: Update org-config.json and Confirm

**Capture the collection folder's Drive ID (added in 2.10.1, Option B id-anchored access).** After the collection files are on the remote, `aifs_stat("/{name}")` to resolve the collection code dir's Drive object ID. This `folder_id` is how members (who are not Drive members and cannot resolve `/{name}` by name — bug 20260606-…-db13) address the collection for capability sync. If the stat fails, note it and proceed with `folder_id` omitted (apply-updates Migration 4 will backfill it later).

Add a new entry to `installed_collections` in `org-config.json`:

```json
{
  "name": "{name}",
  "display_name": "{display_name}",
  "version": "{version from collection.json}",
  "folder_id": "{Drive ID from aifs_stat('/{name}') — omit if stat failed}",
  "downloaded_date": "{today YYYY-MM-DD}",
  "repo_url": "{repo_url}",
  "source_tag": "{the tag the local clone was checked out to}",
  "install_method": "git-clone",
  "status": "downloaded"
}
```

(`zip_url` is retained in the marketplace directory entry only as the deprecated fallback; it is no longer written as the `install_method` here.)

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

If the collection directory already exists on the remote (an interrupted prior download, or a collection removed from org-config but not from the filesystem): surface what is there and offer to overwrite with the fresh download or abort. Never silently merge a new download over a partial prior upload.

<!-- RECONSTRUCTED 2026-06-10: final edge-case completion of a tail lost to truncation (bug 20260608-8d20ea22-003039-trunc); reviewed and approved by Bill. -->

<!-- AIFS:FILE-END -->
