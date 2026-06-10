---
name: refresh-marketplace-cache
type: task
version: 2.4.0
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

Download the marketplace directory from `source_url` **using the Distribution fetch protocol (SHA-pinned) — standards.md § "Distribution fetch protocol"** (replaces the 2.9.0 cache-buster in marketplace 2.11.0; closes bug `20260601-8d20ea22-2`):

1. Resolve the repo's branch head SHA: `GET https://api.github.com/repos/{owner}/{repo}/commits/{branch}` → `.sha` (derive `{owner}/{repo}/{branch}/{path}` from `source_url`).
2. Fetch `https://raw.githubusercontent.com/{owner}/{repo}/{SHA}/{path}` — immutable, cannot be served stale. Record `source: pinned` + the SHA.
3. If SHA resolution fails: fetch `https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}` (record `source: jsdelivr-fallback`); if that also fails, fetch the bare `source_url` (record `source: unpinned`).

Bare branch-form raw URLs MUST NOT be fetched as the primary path: the fetch layer caches them by exact URL and `?t=` cache-busters are **stripped on the raw redirect**, so a refresh right after a release re-caches the *old* catalog with no error. Fallback-sourced (`jsdelivr-fallback`/`unpinned`) results are advisory: Step 4's comparison rules still apply, and a fallback result that looks older or identical must be reported with its source so the admin knows the confidence level.

If fetch succeeds: proceed to Step 4.

If fetch fails:

**First, classify the failure shape** (added in core 3.7.4 to close section D of idea `allowlist-failure-mode-warnings-in-tasks`):

- **Allowlist-blocked signature:** HTTP 403 with empty body and no upstream-server headers, OR connection-refused, OR connection-timeout. The host is in the canonical allowlist (`raw.githubusercontent.com` for this task) but the Cowork network allowlist doesn't permit it.
- **Other network error:** HTTP 5xx, DNS failure, real 403 from the destination host (rate limit, etc.), or any other shape. The host responded; the issue is upstream.

Then check whether a local cache already exists at `/shared/marketplace-cache/marketplace-directory.json`:

- **Cache exists, fetch failed:**
  - In automatic mode: surface a non-blocking notice to the calling task — "Marketplace cache couldn't be refreshed — using cached version." Proceed with the existing cache.
  - In manual mode: surface to the admin: "Couldn't reach the marketplace at `{source_url}`. Your local cache from {last_fetched} is still available. Check your internet connection and try again." Halt.

- **No cache exists, fetch failed (hard stop):**
  - In both automatic and manual mode: surface and halt. Wait for admin confirmation before retrying.
  - **If the allowlist-blocked signature matched:** surface the canonical Allowlist Failure Recognition message (see `agent-index-core/collection-authoring-guide.md` § "Allowlist failure recognition") naming `raw.githubusercontent.com` as the blocked host. Recommend `@ai:verify-network-allowlist` to test all required hosts at once.
  - **Otherwise (real network error):** surface — "Couldn't reach the marketplace at `{source_url}` and no local cache exists. The request reached the host but failed with `{error_detail}`. Check the source URL, GitHub status, or your network connection, then say '@ai:refresh-marketplace-cache' to retry."

---

### Step 4: Compare and Update

Read the existing `/shared/marketplace-cache/marketplace-directory.json` if it exists.

**Newer-than-cache test (content-signal rule, added 2.11.0 — closes the detection half of bug `20260607-8d20ea22-131906-d1rv`):** the fetched directory is newer iff `directory_version` increased, **or** `directory_version` is equal AND `last_updated` is newer AND the content actually differs (compare a hash of the canonicalized JSON). Never key on `directory_version` alone — listing content has shipped under an unchanged `directory_version` and was invisible to version-only comparison.

Compare the fetched directory to the existing cache. Track:
- New collections added since last fetch
- Collections with version updates available
- Collections removed from the marketplace (rare — note but do not remove from local cache automatically)

If the fetched directory is newer (per the rule above), write it to `/shared/marketplace-cache/marketplace-directory.json`.

Update `cache-metadata.json`:
```json
{
  "source_url": "{source_url}",
  "last_fetched": "{now ISO 8601}",
  "ttl_hours": {ttl_hours},
  "expires_at": "{now + ttl_hours ISO 8601}",
  "directory_version": "{version from fetched directory}",
  "fetch_source": "{pinned | jsdelivr-fallback | unpinned}",
  "pinned_sha": "{resolved commit SHA, or null}"
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

In automatic mode this task is invisibl