---
name: refresh-marketplace-cache
type: task
version: 2.0.0
collection: agent-index-marketplace
description: Fetches the latest marketplace directory from GitHub and updates the local cache. Run automatically when the cache is stale, or manually at any time.
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

The marketplace directory — the list of all available collections — is hosted on GitHub as the canonical source of truth. The local cache in `/shared/marketplace-cache/` is a copy of that directory, kept fresh by this task.

This task runs in two modes: automatic (triggered silently by other marketplace tasks when the cache is stale) and manual (invoked directly by an admin who wants the latest list immediately).

In automatic mode it is silent — it refreshes and returns control to whatever task triggered it. In manual mode it reports what changed.

### Inputs

None required. Reads `cache-metadata.json` to determine the source URL and current state.

### Outputs

- `/shared/marketplace-cache/marketplace-directory.json` — updated with latest directory
- `/shared/marketplace-cache/cache-metadata.json` — updated with new fetch timestamp and expiry

---

## Workflow

### Step 1: Read Cache Metadata

Read `/shared/marketplace-cache/cache-metadata.json`.

If the file does not exist: treat as expired — proceed with fetch using the default source URL from `agent-index.json` (`marketplace_directory_url`).

Extract: `source_url`, `expires_at`, `ttl_hours`.

Determine invocation mode:
- If triggered by another task passing a `silent: true` flag: automatic mode
- If invoked directly by a member: manual mode

---

### Step 2: Check Expiry (Automatic Mode Only)

In automatic mode only: compare `expires_at` to current time.

If cache is still fresh: return immediately without fetching. Signal to the calling task that the cache is valid.

If cache is expired or metadata is missing: proceed to Step 3.

In manual mode: always proceed to Step 3 regardless of expiry.

---

### Step 3: Fetch Latest Directory

Download the marketplace directory from `source_url`.

If fetch succeeds: proceed to Step 4.

If fetch fails (network error, GitHub unavailable):

Check whether a local cache already exists at `/shared/marketplace-cache/marketplace-directory.json`:

- **Cache exists, fetch failed:**
  - In automatic mode: surface a non-blocking notice to the calling task — "Marketplace cache couldn't be refreshed — using cached version." Proceed with the existing cache.
  - In manual mode: surface to the admin: "Couldn't reach the marketplace at `{source_url}`. Your local cache from {last_fetched} is still available. Check your internet connection and try again." Halt.

- **No cache exists, fetch failed (hard stop):**
  - In both automatic and manual mode: surface the following and halt. Do not proceed. Wait for admin confirmation before retrying:

    > "The marketplace directory can't be reached and no local cache exists. Before continuing, please whitelist this URL in your Cowork network settings:
    > `https://raw.githubusercontent.com/agent-index/agent-index-resource-listings/refs/heads/main/marketplace-directory.json`
    >
    > Once that's done, say '@ai:refresh-marketplace-cache' to retry."

---

### Step 4: Compare and Update

Read the existing `/shared/marketplace-cache/marketplace-directory.json` if it exists.

Compare the fetched directory to the existing cache. Track:
- New collections added since last fetch
- Collections with version updates available
- Collections removed from the marketplace (rare — note but do not remove from local cache automatically)

Write the fetched directory to `/shared/marketplace-cache/marketplace-directory.json`.

Update `cache-metadata.json`:
```json
{
  "source_url": "{source_url}",
  "last_fetched": "{now ISO 8601}",
  "ttl_hours": {ttl_hours},
  "expires_at": "{now + ttl_hours ISO 8601}",
  "directory_version": "{version from fetched directory}"
}
```

---

### Step 5: Report (Manual Mode Only)

In automatic mode: return silently.

In manual mode, report what changed:

> "Marketplace directory updated."
> {if new collections}: "New collections available: {list}"
> {if version updates}: "Updates available for installed collections: {list with current vs new version}"
> {if no changes}: "No changes since last fetch on {last_fetched}."

---

## Directives

### Behavior

In automatic mode this task is invisible. It does its job and gets out of the way. Never surface anything to the member in automatic mode unless the fetch fails — and even then, surface it as a non-blocking notice, not an error.

In manual mode be informative but brief. The admin invoked this to get current information — give them a clear summary of what changed.

### Constraints

Never remove entries from the local cache based on marketplace changes. If a collection disappears from the marketplace directory, note it in manual mode output but leave the local cache entry intact. The org may have downloaded and installed that collection — removing it from the cache would break list operations.

Never fail in automatic mode. A failed refresh in automatic mode degrades gracefully to using the existing cache. The calling task must always get a response it can continue with.

Always update `cache-metadata.json` atomically with the directory file — write both or neither.

### Edge Cases

If the fetched directory has a lower `directory_version` than the current cache: this should not happen but if it does, do not downgrade the cache. Surface a warning in manual mode and keep the current cache.

If `/shared/marketplace-cache/` does not exist: create it before writing. Surface a notice in manual mode that the cache directory was created.

If the fetched JSON is not valid: treat as a fetch failure and use the existing cache.
