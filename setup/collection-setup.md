---
name: agent-index-marketplace-collection-setup
type: collection-setup
version: 2.1.0
collection: agent-index-marketplace
description: Org-admin setup for the agent-index-marketplace collection
upgrade_compatible: true
---

## Collection Setup Overview

This sets up the marketplace for your org. It confirms the org's catalog subscriptions are readable so your org can browse and install collections. This takes about one minute. (2.1.0: no cache is created and nothing is fetched from the web — catalogs are read from the admin's local clones; `standards.md` § "Marketplaces".)

---

## Prerequisites

- `org-config.json` is readable on the remote filesystem (via `aifs_read`)
- The public catalog clone exists under the install root (`agent-index-resource-listings/`, created by the infra clone during create-org)

---

## Org-Level Parameters

### Marketplace Cache Configuration

**marketplace_cache_ttl_hours**
- Description: **No longer used as of 2.1.0** — retained so existing setup responses stay valid (removing a parameter is a breaking setup change). Catalogs are read from local clones and have no TTL. Accept the default.
- Applies to: list-marketplace-collections, download-collection, download-and-install-collection
- Interview prompt: "How often should I check for new or updated collections in the marketplace? The default is every 24 hours — this means once a day I'll fetch the latest list from GitHub. You can set it higher for less frequent checks or lower if you want more current information."
- Accepted values: Any positive integer representing hours
- Default: `24`
- Implication of choices: Lower values mean more frequent network requests to GitHub. Higher values mean the list may be slightly stale. 24 hours is appropriate for most orgs.

---

## Setup Completion

1. Write collected parameter values to `collection-setup-responses.md`
2. **Verify catalogs are readable:** follow `/internal/resolve-marketplaces.md` (admin-side). On a new org this reads the single public subscription seeded by create-org from the `agent-index-resource-listings` clone. If it returns `error` with `source_missing`, halt and surface: "The public catalog clone isn't in your install folder yet. Run the committed clone-repos script (the infra clone from create-org), then re-run this setup." Do not create `/shared/marketplace-cache/` and do not fetch anything from the web.
3. (Removed in 2.1.0: cache fetch, `cache-metadata.json`, and the network-allowlist halt for the directory fetch. Network reachability for clones is the clone script's concern, and `@ai:verify-network-allowlist` remains available.)
4. Update `org-config.json` installed_collections to include agent-index-marketplace, with `"marketplace_id": "agent-index-public"`
5. Confirm to admin: "Marketplace is ready. Say '@ai:marketplace' or 'open marketplace' to browse available collections."

---

## Upgrade Behavior

### Preserved Responses
- `marketplace_cache_ttl_hours` — preserved across upgrades (unused since 2.1.0)

### Reset on Upgrade
None.

### Requires Admin Attention
None unless the canonical marketplace URL changes — which would be noted here with the new URL.

### Requires Member Attention
None. Members use the marketplace list automatically; no individual action needed for upgrades.

### Migration Notes
- 2.0.x → 2.1.0: no action. An existing `/shared/marketplace-cache/` is left in place and ignored; an admin may delete it manually.
- v1.0 → future versions: migration notes will be added here as new versions are published.
