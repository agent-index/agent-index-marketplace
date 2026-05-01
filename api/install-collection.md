---
name: install-collection
type: task
version: 2.0.0
collection: agent-index-marketplace
description: Runs the org-admin setup interview for a downloaded collection, configuring it for the org and making it available for members to install.
stateful: true
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

Installation is the second step after downloading. It runs the collection's `collection-setup.md` interview with the org admin, collecting org-level configuration — the values that become org-mandated parameters for all members who install from this collection.

After installation the collection is fully configured and its skills and tasks are available for members to install via `@ai:setup`.

### Inputs

Collection name — provided in invocation or asked for.

### Outputs

- `/{collection-name}/setup/collection-setup-responses.md` — org admin's configuration responses
- `org-config.json` — collection entry updated from `status: downloaded` to `status: installed`

---

## Workflow

### Step 1: Identify and Validate Collection

If the member named a collection in their invocation: use that name.
If not: ask "Which collection would you like to install? Say '@ai:list-org-collections' to see downloaded collections ready for setup."

Check `org-config.json` for the collection entry.

If not found in org-config at all: surface "'{name}' hasn't been downloaded yet. Say '@ai:download-collection {name}' to download it first." Halt.

If `status: installed`: surface "'{display_name}' is already installed. To reconfigure it, you'll need to re-run setup — this will overwrite the existing configuration. Would you like to proceed?" Require explicit confirmation before continuing.

Verify the collection directory exists on the remote filesystem via `aifs_exists` and that `collection.json` is readable via `aifs_read`. If not: surface "'{name}' is recorded as downloaded but the directory isn't accessible on the remote filesystem. The files may have been moved or deleted. Try downloading it again with '@ai:download-collection {name}'." Halt.

**On success:** Proceed to Step 2.

---

### Step 2: Check for Existing Setup Responses

Check whether `/{collection-name}/setup/collection-setup-responses.md` already exists.

If it exists and `status` in org-config is `downloaded` (setup started but not completed previously): surface "It looks like setup for '{display_name}' was started but not finished. Would you like to start over or try to continue from where you left off?"

- Continue: load existing responses and resume from the first unanswered parameter
- Start over: proceed fresh, overwriting existing responses

If responses exist and `status` is `installed` (reconfiguration flow): always start fresh — confirmed in Step 1.

---

### Step 3: Resolve Alias Collisions

Before running the setup interview, check for name collisions that were flagged at download time.

Read the collection's `collection.json` `api` array via `aifs_read`. Compare against all entries in the running member's local `member-index.json` and any other members' indexes accessible via `aifs_list`/`aifs_read` at `/shared/members/`.

For each collision found:

> "'{collection-name}' includes a task/skill named '{name}', which is already used by '{other-collection}'. Please choose an alias for one of them:
> - Keep '{existing alias}' for '{other-collection}:{name}' and assign a new alias for '{collection-name}:{name}'
> - Or assign new aliases to both"

Collect all collision resolutions before proceeding. These resolved aliases will be written into member indexes when members install from this collection.

Store resolved aliases in `collection-setup-responses.md` under an `alias_overrides` section.

If no collisions: proceed to Step 4 directly.

---

### Step 4: Run Collection Setup Interview

Read `/{collection-name}/setup/collection-setup.md` from the remote filesystem via `aifs_read`.

Conduct the setup interview as defined in that file. Inject:
- The collection's `display_name` and `description` as context for Claude
- Any alias overrides resolved in Step 3

Collect all org-level parameter values. These become `[org-mandated]` values for all member-level setup interviews.

Track progress — if the session is interrupted mid-interview, record what has been collected so far in a partial `collection-setup-responses.md` with a `setup_status: incomplete` marker.

**On completion of all parameters:** Proceed to Step 5.

---

### Step 5: Confirm and Write

Present a summary of all configured values:

> **Setup Summary — {display_name}**
> {for each parameter}: {parameter name}: {value}
> {if alias overrides}: Alias resolutions: {list}
>
> Ready to complete installation?

Wait for explicit confirmation.

On confirmation:

1. Write `collection-setup-responses.md` to `/{collection-name}/setup/` on the remote filesystem via `aifs_write` with `setup_status: complete`
2. Update `org-config.json` on the remote filesystem via `aifs_write`: set `status: installed`, add `installed_date: {today}`
3. Write `current-state.md` to task state directory recording completion

Confirm to admin:
> "'{display_name}' is now installed and configured. Members can install its skills and tasks by saying '@ai:setup' in their Cowork session."
> {if alias overrides were set}: "Alias resolutions have been recorded — members will receive the correct aliases when they install."
> {if external dependencies}: "Reminder: this collection requires access to {external systems}. Make sure members have the necessary credentials before they try to use it."

---

## Directives

### Behavior

The setup interview is a conversation with an org admin, not a form. Run it conversationally following the structure in `collection-setup.md`. Explain what each parameter controls and the implications of choices — admins may not be deeply technical.

Progress tracking matters here. The setup interview for a complex collection could involve many parameters across multiple concerns. If the session is long, the admin should be able to pick up where they left off.

After writing, always surface external dependency reminders if any exist. Admins need to know what access to arrange for their team.

### State Management

Write partial progress to `collection-setup-responses.md` after each parameter group is completed — not just at the end. This enables resumption if the session is interrupted.

### Constraints

Never update `org-config.json` to `status: installed` until Step 5 confirmation and all writes are complete.

Never run setup for a collection that hasn't been downloaded — the collection files must be present on the remote filesystem.

Alias collision resolution in Step 3 must be completed before the setup interview begins. Unresolved collisions would result in ambiguous member installations.

### Edge Cases

If `collection-setup.md` does not exist in the collection's `/setup/` directory: surface "'{display_name}' is missing its setup file — the collection may be incomplete or corrupted. Contact the collection author or try re-downloading." Halt.

If the setup interview produces no parameters (collection has no org-level configuration): write an empty `collection-setup-responses.md` with `setup_status: complete` and proceed directly to marking as installed. Not all collections require org-level configuration.

If the admin abandons the interview partway through and the session ends: the partial `collection-setup-responses.md` with `setup_status: incomplete` remains. On the next invocation, detect this and offer to resume.
