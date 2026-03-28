---
name: list-marketplace-collections-setup
type: setup
version: 2.0.0
collection: agent-index-marketplace
description: Setup for the list-marketplace-collections task
target: list-marketplace-collections
target_type: task
upgrade_compatible: true
---

## Setup Overview

This installs the List Marketplace Collections task. Browse all available marketplace collections with download and install status.

---

## Pre-Setup Checks

- `org-config.json` is readable on the remote filesystem (via `aifs_read`) → if not: "Org configuration isn't readable. Run '@ai:create-org' first."
- `/shared/marketplace-cache/` exists → if not: "Marketplace cache directory is missing. The marketplace collection may not be fully set up — contact your org admin."

---

## Parameters

### Org-Mandated Parameters [org-mandated]

**marketplace_cache_ttl_hours** [org-mandated]
- Description: Cache TTL for the marketplace directory
- Value source: injected from collection org-setup-responses
- Member visibility: read-only

---

## Setup Completion

1. Write all collected parameter values to `setup-responses.md`
2. Write the installed instance to `/members/{member-id}/tasks/list-marketplace-collections/`
3. Write `manifest.json`
4. Register entry in `member-index.json` with alias `@ai:list-marketplace-collections`
5. Confirm to member: "'List Marketplace Collections' is installed. Say '@ai:marketplace' or '@ai:list-marketplace-collections' to browse available collections."

---

## Upgrade Behavior

### Preserved Responses
All responses are injected from collection setup — preserved automatically when the collection upgrades.

### Reset on Upgrade
None.

### Requires Member Attention
None.

### Migration Notes
- v1.0 → future versions: migration notes will be added here as new versions are published.
