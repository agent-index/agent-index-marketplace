---
name: agent-index-marketplace-collection-setup
type: collection-setup
version: 2.0.0
collection: agent-index-marketplace
description: Org-admin setup for the agent-index-marketplace collection
upgrade_compatible: true
---

## Collection Setup Overview

This sets up the marketplace for your org. It initializes the local marketplace cache so your org can browse and install collections. This takes about one minute.

---

## Prerequisites

- `org-config.json` is readable on the remote filesystem (via `aifs_read`)
- `/shared/marketplace-cache/` directory exists or can be created
- Internet access is available (for the initial cache fetch)

---

## Org-Level Parameters

### Marketplace Cache Configuration

**marketplace_cache_ttl_hours**
- Description: How long the local marketplace directory cache is considered fresh before a refresh is needed
- Applies to: list-marketplace-collections, download-collection, download-and-install-collection
- Interview prompt: "How often should I check for new or updated collections in the marketplace? The default is every 24 hours — this means once a day I'll fetch the latest list from GitHub. You can set it higher for less frequent checks or lower if you want more current information."
- Accepted values: Any positive integer representing hours
- Default: `24`
- Implication of choices: Lower values mean more frequent network requests to GitHub. Higher values mean the list may be slightly stale. 24 hours is appropriate for most orgs.

---

## Setup Completion

1. Write collected parameter values to `collection-setup-responses.md`
2. Create `/shared/marketplace-cache/` if it does not exist
3. Perform initial cache fetch — download `marketplace-directory.json` from the canonical GitHub URL and write to `/shared/marketplace-cache/marketplace-directory.json`
4. Write `/shared/marketplace-cache/cache-metadata.json`:
```json
{
  "source_url": "https://raw.githubusercontent.com/agent-index/marketplace/refs/heads/main/marketplace-directory.json",
  "last_fetched": "{now ISO 8601}",
  "ttl_hours": {ttl_hours},
  "expires_at": "{now + ttl_hours ISO 8601}",
  "directory_version": "{version from fetched directory}"
}
```
5. If initial cache fetch fails (network unreachable): halt and surface:

   > "The marketplace directory can't be reached. Before continuing, please whitelist this URL in your Cowork network settings:
   > `https://raw.githubusercontent.com/agent-index/marketplace/refs/heads/main/marketplace-directory.json`
   >
   > Once that's done, say '@ai:refresh-marketplace-cache' to retry."

   Do not write a cache file. Do not proceed to step 6 until the admin confirms the URL is reachable and the fetch succeeds.

6. Update `org-config.json` installed_collections to include agent-index-marketplace
7. Confirm to admin: "Marketplace is ready. Say '@ai:marketplace' or 'open marketplace' to browse available collections."

---

## Upgrade Behavior

### Preserved Responses
- `marketplace_cache_ttl_hours` — preserved across upgrades

### Reset on Upgrade
None.

### Requires Admin Attention
None unless the canonical marketplace URL changes — which would be noted here with the new URL.

### Migration Notes
- v1.0 → future versions: migration notes will be added here as new versions are published.
