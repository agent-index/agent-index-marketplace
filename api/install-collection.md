---
name: install-collection
type: task
version: 2.2.1
collection: agent-index-marketplace
description: Runs the org-admin setup interview for a downloaded collection, configuring it for the org and making it available for members to install. Provisions any collaborative-folder ACLs the collection declares (collaborative-acls.json) via permission-change-helper at Step 5.5.
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

**Guardrail — never mark a collection installed without writing its responses file (bug `20260615-8d20ea22-setupresp`).** Step 1 is MANDATORY and is NOT skipped by an "accept defaults / don't ask me questions" shortcut: a defaults install still writes `collection-setup-responses.md` with `setup_status: complete` (and the resolved default/org-mandated values) — an empty-but-complete file when the collection has no org-level parameters. Without it, `org-setup` hard-blocks **every member** from installing any capability from this collection. If you're installing several collections at once "with defaults," loop this task per collection (each writing its responses) — do **not** bulk-upload + register collections without their responses files. After Step 1, verify it landed: `aifs_read("/{collection-name}/setup/collection-setup-responses.md")` must succeed with `setup_status: complete` before proceeding to Step 2.

Then run Step 5.5 (collaborative-ACL provisioning) before the final confirmation below.

Confirm to admin:
> "'{display_name}' is now installed and configured. Members can install its skills and tasks by saying '@ai:setup' in their Cowork session."
> {if alias overrides were set}: "Alias resolutions have been recorded — members will receive the correct aliases when they install."
> {if collaborative ACLs were provisioned}: "Collaborative access provisioned: {recipients} now have {roles} on {paths}."
> {if external dependencies}: "Reminder: this collection requires access to {external systems}. Make sure members have the necessary credentials before they try to use it."

---

### Step 5.5: Provision Collaborative ACLs

Some collections need members to write into shared collaborative folders (e.g., bug-reports' `bugs/`, projects' shared project tree). Members are otherwise reader-only on `/shared`. A collection declares the grants it needs in a `collaborative-acls.json` file at its root; this step provisions them. Permission changes are **never** applied with `aifs_share` directly — they go through the `permission-change-helper` skill, which the admin reviews and Accepts (per standards.md § "Permission-Modifying Operations" and § "Collaborative Folder ACLs").

1. **Check for the declaration.** `aifs_exists("/{collection-name}/collaborative-acls.json")`. If absent: there are no collaborative `/shared` folders to provision — but do NOT skip the step (2.10.1): the unconditional collection code-dir reader grant + folder_id capture described at the end of this step STILL run for every installed collection. Treat `acls[]` as empty and proceed to those.
2. **Read & resolve.** `aifs_read` it; parse `acls[]`. Resolve `{param}` placeholders from the just-written `collection-setup-responses.md` (e.g., `{bug_log_path}`) and from `org-config.json` (`{all_members_group}` ← `remote_filesystem.connection.all_members_group`; `{domain}` if used). If any referenced placeholder cannot be resolved, surface a clear blocker, **skip provisioning** (do not guess a recipient or path), and note it in the confirmation so the admin can fix and re-run.
3. **Capture current state & filter no-ops.** For each acl entry, `aifs_get_permissions(path)`. Drop entries already satisfied:
   - `inherit: true` grant → drop if `recipient` already has ≥ `role` on `path` (direct or via the same group).
   - `inherit: false` + `restrict: true` → drop if the inherited grant being removed is already absent.
   If nothing remains: surface "Collaborative access already provisioned (no changes needed)." and proceed.
4. **Compose one spec.** Build a single `permission-change-helper` spec (`version: "1.0"`, or `"1.1"` if any entry uses `inherit`) whose `operations[]` are the remaining entries mapped to `{op:"share", resource, recipient, role[, inherit]}`, each with its `before` populated from step 3. **Build this spec with the committed `build-permission-spec` CLI** (see `permission-change-helper` Step 2.5): emit the ops-array as data and run the CLI -- it enforces the op name, email/UPN recipient form, required role, and the canonical `<project_dir>/outputs/` path, and prints the `spec_path`/`link_path` to use. Do not hand-author the spec JSON. (The CLI passes `inherit`/`restrict` through.)
5. **Invoke `permission-change-helper`.** The admin reviews the single batched page and clicks Accept; the apply runs under the admin's own OAuth token. (Requires the admin to hold organizer/owner on the target folders — required anyway for any `inherit:false` op, and the helper binary must be `permission-helper-go ≥ 0.3.0` for `inherit:false`.)
6. **Branch on the returned outcome:**
   - `applied` (or `partial_failure` with the critical grants applied): the helper has already verified post-state. Record the provisioned grants for the confirmation message.
   - `rejected` / `page_closed` / `timed_out`: surface "Collaborative access was NOT provisioned — members will not be able to write to {paths} until this is done. Re-run '@ai:install-collection {collection-name}' (reconfigure) to retry." Do **not** roll back the installation.
   - `binary_not_found`: direct the admin to '@ai:update' (install/upgrade the helper binary), then retry this step.
7. Proceed to the confirmation message.

**Collection code-dir reader grant + folder_id (added in 2.10.1 — Option B id-anchored access; closes cr01/cr02).** Non-admin members are not Drive members; they can read a collection's code dir at `/{name}/` only if it is directly shared with them, and they address it by stored `folder_id` (they cannot resolve `/{name}` by name — bug 20260606-…-db13). So Step 5.5 ALSO does the following, folded into the same flow above:
   - **Capture folder_id:** if `org-config.json` `installed_collections[{name}].folder_id` is absent, `aifs_stat("/{name}")` → write the resolved Drive ID into that entry (revision-aware). (download-collection captures it too; this covers the install-only and reconfigure paths.)
   - **Grant all@ reader on the collection code dir:** include a `share` op (recipient `{all_members_group}`, role `reader`, resource `id:{folder_id}` if captured, else `/{name}`) in the SAME spec composed in step 4 — one batched Accept covers both the `/shared` collaborative writers AND the code-dir reader. Subject to the same step-3 no-op filter (skip if `all@` already holds reader). **Without this grant, members cannot sync the collection's capabilities** — this is the brand-book 2026-06-08 failure (cr01). A collection with no `collaborative-acls.json` still gets THIS grant: the code-dir reader is unconditional for every installed collection, independent of whether the collection declares collaborative `/shared` folders.
   - **Verify the grant landed (hard step, C.1.5.0 `collreadgrantmissing`).** After the helper outcome, `aifs_get_permissions("id:{folder_id}")` and confirm `{all_members_group}` holds `reader` on the code dir. If it does NOT (the grant Accept was `rejected`/`timed_out`, or was filtered as a no-op in error): record `installed_collections[{name}].member_read_grant: "pending"` in `org-config.json` (revision-aware) and **hard-surface**: "Members cannot read {collection-name} until the all-members reader grant on its code dir is Accepted -- re-run '@ai:install-collection {collection-name}' (reconfigure), or it will be auto-healed on the next '@ai:publish-updates' (Step 6e, which now reconciles collection code-dir grants)." On confirmed reader, set `member_read_grant: "granted"`. Do NOT report the collection as fully installed-and-readable while it is `pending`. This closes the client-intelligence gap where a collection was marked installed but the all-members reader grant was never actually present.

This step is the **only** place install-collection touches permissions, and only via `permission-change-helper`. It is idempotent (step 3 filter) and safe to re-run for backfill on already-installed orgs.

### Step 5.7: Register Capability Providers (added in 2.10.0, requires core 3.10.0)

If the collection's `collection.json` declares `provides[]`:

1. **Validate the declaration.** For each provides entry: resolve the capability type definition — well-known (`/agent-index-core/capability-types/{capability}.json` via `aifs_read`) or collection-custom (`/{collection}/capability-types/{name}.json`). Verify every `required: true` operation in the type is present in the entry's `operations` map, and every `implemented_by` names a member of the collection's `api[]`. On any failure: surface the specific gap ("provides declares `{op}` implemented by `{name}`, which is not in api[]"), **skip registration for that entry**, and continue the install — a broken provides declaration must not block the collection's direct use.
2. **Prompt the admin.** "`{collection}` provides the `{capability}` capability ({N} operations). Register it as a `{capability}` provider for this org?" Include any `provider_config` values the collection's setup interview produced for the registry (e.g., brand-book's `brand_usage`).
3. **On accept:** revision-aware update of `org-config.json` (`aifs_stat` → `if_revision` → retry on `REVISION_CONFLICT`): create `capability_providers` if absent; append `{provider_collection, capability_version, registered_date, registered_by: {admin member_hash}, operations_available, provider_config}` to `capability_providers.{capability}.providers`. If other providers already exist for the type, inform: "registered alongside {existing_list}." Then append a `provider-register` op to the update log.
4. **On decline:** no registry write, no log op. Note in the confirmation: "Not registered as a provider — consumers will not discover it. Re-run '@ai:install-collection {collection}' (reconfigure) to register later."

If the collection declares `requires[]`: for each entry, check `capability_providers` for a registered provider implementing all `required_operations`. Surface per spec — `required: true` unmet → WARNING; `required: false` unmet → INFO ("these features will be disabled"). **Never block installation.**

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
