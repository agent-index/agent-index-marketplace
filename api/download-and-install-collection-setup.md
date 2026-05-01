---
name: download-and-install-collection-setup
type: setup
version: 2.0.0
collection: agent-index-marketplace
description: Setup for the download-and-install-collection task
target: download-and-install-collection
target_type: task
upgrade_compatible: true
---

## Setup Overview

This installs the Download and Install Collection task. Download and install a marketplace collection in one flow.

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
2. Write the installed instance to `/members/{member-id}/tasks/download-and-install-collection/`
3. Write `manifest.json`
4. Register entry in `member-index.json` with alias `@ai:download-and-install-collection`
5. Confirm to member: "'Download and Install Collection' is installed. Say '@ai:download-and-install-collection' followed by a collection name to add it to your org."

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
