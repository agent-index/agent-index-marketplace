---
name: check-updates-setup
type: setup
version: 2.0.0
collection: agent-index-marketplace
description: Setup interview for check-updates
target: check-updates
target_type: task
upgrade_compatible: true
---

## Setup Overview

This setup configures the update checking task. The defaults work well for most members — the only decisions are how verbose the report is and whether session-start should include update notices.

---

## Pre-Setup Checks

- Member has agent-index-marketplace collection installed → proceed with org setup if not

---

## Parameters

### Member-Overridable Parameters [member-overridable]

**default_report_mode** [member-overridable]
- Description: Whether the update report shows everything or only items needing attention
- Default: full
- Interview prompt: "Update reports can show all items (full) or only items that need attention (quiet). Which do you prefer as the default?"
- Accepted values: `full`, `quiet`

**session_start_update_notices** [member-overridable]
- Description: Whether session-start includes a lightweight update check and surfaces notices about available updates
- Default: true
- Interview prompt: "Would you like to see update-available notices when you start a session? This adds a brief check but keeps you informed."
- Accepted values: `true`, `false`

---

## Setup Completion

1. Write all collected parameter values to `setup-responses.md`
2. Generate the personalized installed instance
3. Write the installed instance to `/members/{member_hash}/tasks/check-updates/`
4. Write manifest.json
5. Register entry in `member-index.json`
6. Confirm completion to member

---

## Upgrade Behavior

### Preserved Responses
All parameters preserved at v1.0.0.

### Reset on Upgrade
None at v1.0.0.

### Requires Member Attention
None at v1.0.0.

### Migration Notes
None at v1.0.0.
