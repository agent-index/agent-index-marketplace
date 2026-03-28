---
name: install-collection-setup
type: setup
version: 2.0.0
collection: agent-index-marketplace
description: Setup for the install-collection task
target: install-collection
target_type: task
upgrade_compatible: true
---

## Setup Overview

This installs the Install Collection task. Run org-admin setup for a downloaded collection.

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
2. Write the installed instance to `/members/{member-id}/tasks/install-collection/`
3. Write `manifest.json`
4. Register entry in `member-index.json` with alias `@ai:install-collection`
5. Confirm to member: "'Install Collection' is installed. Say '@ai:install-collection' followed by a collection name to configure it."

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
