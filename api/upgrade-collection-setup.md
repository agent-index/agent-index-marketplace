---
name: upgrade-collection-setup
type: setup
version: 1.0.1
collection: agent-index-marketplace
description: Setup for the upgrade-collection task — no member-defined parameters
target: upgrade-collection
target_type: task
upgrade_compatible: true
---

## Setup Overview

This task has no member-defined parameters. It's a per-invocation operation, not a configurable capability.

The task accepts these per-invocation flags (passed when the admin runs it):

- `<collection_name>` (required, positional)
- `--to <version>` (optional)
- `--check-upstream` (optional)
- `--dry-run` (optional)

No setup interview runs at install time — this task is admin-callable directly.

---

## Setup Completion

1. Register entry in `member-index.json` with alias `@ai:upgrade-collection` (admin-only task).
2. Confirm: "Upgrade Collection is installed. Run `@ai:upgrade-collection {name}` to upgrade an installed collection."

---

## Upgrade Behavior

### Preserved Responses
N/A — no member-defined parameters.

### Reset on Upgrade
N/A.

### Requires Member Attention
None.

### Migration Notes
- v1.0.1 (marketplace 2.22.0): added the standard setup frontmatter and Upgrade Behavior section (preflight structural fix); no behaviour change.
