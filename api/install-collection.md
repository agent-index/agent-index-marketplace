---
name: install-collection
type: task
version: 2.5.1
collection: agent-index-marketplace
description: Runs the org-admin setup interview for a downloaded collection, configuring it for the org and making it available for members to install. Provisions any collaborative-folder ACLs the collection declares (collaborative-acls.json) and the members' code-dir read grant at Step 5.5, through the core permission-changes procedure (the admin confirms the exact changes).
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

**Core 4.0.0 helper gate (`gatedownloaded`, Dev Two live test finding 26).** If the org's core (`org-config.agent_index_version`) is 4.0.0 or later, check the collection against the minimum-version table in `/agent-index-core/upgrade/3-to-4.md` § 1 (read through core's read base: gdrive `id:{core folder_id}/upgrade/3-to-4.md`, OneDrive the path), and, for a collection not in that table, search its `api/` for live calls to the retired helper (the patterns `upgrade/3-to-4.md` § 1 lists). If it's below its minimum or has a live call: surface "'{name}' {version} still uses the permission helper that core 4.0.0 removed. Download {minimum} or later (`@ai:download-collection {name}`), then install." Halt. This applies to Reconfigure and Apply access only as well.

If `status: installed`: offer two paths and require an explicit choice:
- **Apply access only (2.4.0)** — re-run only Step 5.5 against the existing setup responses. This is the path for anything pending: `member_read_grant: "pending"`, `collaborative_acls: "pending"`, or provisioning flagged by `upgrade-collection`. The setup interview is not repeated. Go straight to Step 5.5.
- **Reconfigure** — "'{display_name}' is already installed. Reconfiguring re-runs setup and overwrites the existing configuration. Proceed?" Continue only on explicit confirmation. Reconfiguring replaces **configuration** (setup responses), never **data**: seed files a collection's setup creates (manifests, indexes, keys, logs) are written only when absent, and an existing one is kept as is (`reconfigresetsdata`, finding 30). If the collection's own setup says to write a seed file unconditionally, still skip it when the file exists, and say which files were kept.

Record whether `status` was `installed` at the start of this run (`existing_install: true|false`); Step 5.5 step 2a needs it.

Verify the collection directory exists on the remote filesystem via `aifs_exists` and that `collection.json` is readable via `aifs_read`. If not: surface "'{name}' is recorded as downloaded but the directory isn't accessible on the remote filesystem. The files may have been moved or deleted. Try downloading it again with '@ai:download-collection {name}'." Halt.

**On success:** Proceed to Step 2.

---

### Step 2: Check for Existing Setup Responses

Check whether `/{collection-name}/setup/collection-setup-responses.md` already exists.

If it exists and `status` in org-config is `downloaded` (setup started but not completed previously): surface "It looks like setup for '{display_name}' was started but not finished. Would you like to start over or try to continue from where you left off?"

- Continue: load existing responses and resume from the first unanswered parameter
- Start over: proceed fresh, overwriting existing responses

If responses exist and `status` is `installed` (reconfiguration flow): always start fresh — confirmed in Step 1. (The *apply access only* path skips this step.)

**Keep the admin's notes on Reconfigure (finding 79).** Before starting fresh, read the existing file and collect every line that is not part of the canonical shape (frontmatter, `##`/`###` headings, `- **Value:**` lines): comment lines (`#`, `//`, `<!-- … -->`), `> NOTE`-style lines, inline trailing comments after a value, and free prose. Attach each one to the parameter it belongs to — the block it sits in, or the parameter it names or sits directly above; a note tied to no parameter goes with the nearest following parameter. Step 5 writes them back as `- **Note:** {text}` lines (verbatim, one per note) under that parameter's `- **Value:**` line, so nothing the admin wrote is lost. Existing `- **Note:**` lines are carried the same way. Notes for a parameter the new setup no longer has go on the first parameter block, prefixed `(was on {old_param})`.

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
> {if notes were carried from the old file (Step 2)}: Notes kept from your previous setup: {param}: "{note}" (one line each)
>
> Ready to complete installation?

Wait for explicit confirmation.

On confirmation:

1. Write `collection-setup-responses.md` to `/{collection-name}/setup/` on the remote filesystem via `aifs_write` with `setup_status: complete`, in the canonical shape (`### {param}` + `- **Value:**`; core `org-setup.md` step 8). On Reconfigure, add the notes collected in Step 2 as `- **Note:** {text}` lines after each parameter's `- **Value:**` line (finding 79). Readers parse only the `- **Value:**` line, so notes never change a value.
2. Update `org-config.json` on the remote filesystem via `aifs_write`: set `status: installed`; add `installed_date: {today}` **only when the entry has none** — on Reconfigure keep the original install date (finding 80)
   - **Provenance (2.3.0 — multi-marketplace):** never change an existing `marketplace_id`. If the entry has **no `marketplace_id` key** — the sideload path, where the collection's files were placed at `/{name}/` and registered `status: downloaded` without `download-collection` — write `"marketplace_id": null` and tell the admin: "Recorded as sideloaded (no origin catalog). If it belongs to a catalog you subscribe to, set that in '@ai:edit-org' → Manage marketplaces." `install-collection` never reads a catalog and never infers provenance from one.
   - **Source record (channels, 2.22.0):** never change `source_ref`, `source_commit` or `source_tag` here. `download-collection` and `upgrade-collection` write them from the clone-resolution record. A sideloaded entry has none, and none is invented.
3. Write `current-state.md` to task state directory recording completion

**Guardrail — never mark a collection installed without writing its responses file (bug `20260615-8d20ea22-setupresp`).** Step 5 item 1 (writing the responses file) is MANDATORY and is NOT skipped by an "accept defaults / don't ask me questions" shortcut: a defaults install still writes `collection-setup-responses.md` with `setup_status: complete` (and the resolved default/org-mandated values) — an empty-but-complete file when the collection has no org-level parameters. Without it, `org-setup` hard-blocks **every member** from installing any capability from this collection. If you're installing several collections at once "with defaults," loop this task per collection (each writing its responses) — do **not** bulk-upload + register collections without their responses files. After Step 1, verify it landed: `aifs_read("/{collection-name}/setup/collection-setup-responses.md")` must succeed with `setup_status: complete` before proceeding to Step 2.

Then run Step 5.5 (collaborative-ACL provisioning) before the final confirmation below.

Confirm to admin:
> "'{display_name}' is now installed and configured. Members can install its skills and tasks by saying '@ai:setup' in their Cowork session."
> {if alias overrides were set}: "Alias resolutions have been recorded — members will receive the correct aliases when they install."
> {if any collaborative row ended `applied` or `skipped_noop`}: "Collaborative access provisioned: {recipients} now have {roles} on {paths}" (only those rows).
> {if `collaborative_acls` is `"pending"`}: "Shared-folder access is NOT yet provisioned for {paths} — {reason}. Run '@ai:install-collection {name}' → *Apply access only* when ready."
> {if external dependencies}: "Reminder: this collection requires access to {external systems}. Make sure members have the necessary credentials before they try to use it."

---

### Step 5.5: Provision Collaborative ACLs

Some collections need members to write into shared collaborative folders (e.g., bug-reports' `bugs/`, projects' shared project tree). Members are otherwise reader-only on `/shared`. A collection declares the grants it needs in a `collaborative-acls.json` file at its root; this step provisions them. Permission changes go only through the core permission-changes procedure (`/agent-index-core/internal/permission-changes.md`): the admin sees the exact changes, confirms them, and they're applied and verified under the admin's own credentials (standards.md § "Permission changes: list, confirm, apply, verify" and § "Collaborative Folder ACLs").

1. **Check for the declaration.** `aifs_exists("/{collection-name}/collaborative-acls.json")`. If absent: there are no collaborative `/shared` folders to provision — but do NOT skip the step (2.10.1): the unconditional collection code-dir reader grant + folder_id capture described at the end of this step STILL run for every installed collection. Treat `acls[]` as empty and proceed to those.
2. **Read & resolve.** `aifs_read` it; parse `acls[]`.
   - **Legacy alias (2.4.0):** if `acls` is absent but `grants` is present, use `grants[]`, and read each entry's `purpose` as its `rationale`. projects ≤ 4.3.0 and client-intelligence ≤ 2.3.0 shipped with `grants[]`, which earlier versions of this step silently treated as empty.
   - Resolve `{param}` placeholders from the just-written `collection-setup-responses.md` (e.g., `{bug_log_path}`) and from `org-config.json` (`{all_members_group}` ← `remote_filesystem.connection.all_members_group`; `{domain}` if used).
   - **Normalize every resolved path before building rows (finding 81).** A `*_path` value may already end in `/` (e.g. `bug_log_path: /shared/bug-reports/`), so `{bug_log_path}/bugs` would become `/shared/bug-reports//bugs`. After substitution, collapse each run of `/` into one and drop a trailing `/` (keep a lone `/`). Do this for every `resource` and `resource_label`, so the change list, the procedure's access reads and the recorded rows all use the same path. For an `id:` anchor, normalize only the path part after the anchor.
   - If any referenced placeholder cannot be resolved, surface a clear blocker, **skip provisioning** (do not guess a recipient or path), and note it in the confirmation so the admin can fix and re-run.
2a. **Upgrade-ordering guard (2.4.0).** Some collections must not have their collaborative grants applied until an upgrade script's admin half has run: projects 3→4 (gated-idea review) and client-intelligence 1→2 (gated-instance review). Before 2.4.0 this was kept only by accident, because those files used `grants[]`, which this step ignored. The guard is driven by declared data, not by version arithmetic: `upgrade-collection` rewrites `installed_collections[].version` before this step runs, so a version comparison can't see the upgrade.

   - **Declared reviews:** read `requires_upgrade_review` from `collaborative-acls.json` (a list of upgrade-script names such as `["3-to-4"]`; absent = none).
   - **Already confirmed:** org-config `installed_collections[{name}].upgrade_reviews_confirmed` (a list; absent = empty).
   - **Fresh install** (`existing_install: false` from Step 1): there is no pre-existing content for the grants to expose. Add every declared review to `upgrade_reviews_confirmed` and continue.
   - **Existing install, every declared review already confirmed:** continue.
   - **Existing install, any declared review not confirmed:** ask for each one: *"{display_name} requires the admin half of `upgrade/{review}.md` to be finished before its shared-folder grants are applied (it reviews content these grants would expose). Is that done?"*
     - **Yes** → add it to `upgrade_reviews_confirmed` and continue.
     - **Anything else** → leave every `collaborative_acls` row out of the change list, keep the code-dir reader row, set `installed_collections[{name}].collaborative_acls: "pending"`, and say: *"I've left the shared-folder grants out. Finish the upgrade script's admin half, then run '@ai:install-collection {collection-name}' → Apply access only."*

   Never infer that a review is done from file contents. Write the `upgrade_reviews_confirmed` and `collaborative_acls` changes in the same revision-aware org-config write as `member_read_grant` (below). If this step withheld the collaborative rows, `collaborative_acls` stays `"pending"` (never `"granted"`, even though no collaborative row was in the table). Otherwise: when every collaborative row ends `applied` or `skipped_noop`, set `collaborative_acls: "granted"`; when any is unsatisfied, set it to `"pending"`.
3. **Build the change list.** One change per entry: `{op:"share", resource: path, resource_label: path, is_folder: true, recipient, role, inherit (if present), reason: rationale, source: "collaborative_acls"}`. A `restrict: true` entry is the same `share` with `inherit: false`; the procedure treats it as high-impact and confirms it individually. Add the code-dir reader change described below to the **same** list, so the admin sees one table for this collection.
4. **Apply.** Follow `/agent-index-core/internal/permission-changes.md` with that list:
   - `calling_task: "install-collection"`
   - `purpose: "Give members the access {display_name} needs"`
   - `confirmer: "admin"`

   The procedure reads current access and skips anything already in effect. If nothing remains, it reports "No permission changes needed".
5. **Act on the result:**
   - **Every collaborative row `applied` or `skipped_noop`:** record the provisioned grants for the confirmation message.
   - **Any collaborative row not satisfied** (declined, failed, not verified): surface "Collaborative access was NOT fully provisioned — members will not be able to write to {paths} until this is done. Run '@ai:install-collection {collection-name}' → Apply access only to retry," with the procedure's `summary_lines`. Do **not** roll back the installation.
6. Proceed to the code-dir handling below, then the confirmation message.

**Collection code-dir reader grant + folder_id (added in 2.10.1 — Option B id-anchored access; closes cr01/cr02).** Non-admin members are not Drive members; they can read a collection's code dir at `/{name}/` only if it is directly shared with them, and they address it by stored `folder_id` (they cannot resolve `/{name}` by name — bug 20260606-…-db13). So Step 5.5 ALSO does the following, folded into the same flow above:
   - **Capture folder_id:** if `org-config.json` `installed_collections[{name}].folder_id` is absent, `aifs_stat("/{name}")` → write the resolved Drive ID into that entry (revision-aware). (download-collection captures it too; this covers the install-only and reconfigure paths.)
   - **Grant all@ reader on the collection code dir:** include the change `{op:"share", resource: {code_dir}, resource_label: "{display_name} code folder", is_folder: true, recipient: "{all_members_group}", role: "reader", reason: "Members need read to sync this collection's capabilities", source: "task_spec"}` in the SAME list built in step 3. One table and one confirmation cover both the `/shared` collaborative writers AND the code-dir reader. The procedure skips it if `all@` already holds reader. **Without this grant, members cannot sync the collection's capabilities** — this is the brand-book 2026-06-08 failure (cr01). A collection with no `collaborative-acls.json` still gets THIS grant: the code-dir reader is unconditional for every installed collection, independent of whether the collection declares collaborative `/shared` folders.
     - **`{code_dir}` by backend (finding 63).** gdrive: `"id:{folder_id}"` when captured, else `"/{name}"`. OneDrive: `"/{name}"` (path first); use `"id:{drive_id}:{folder_id}"` only if the path fails and both ids are known (`drive_id` from org-config `remote_filesystem.connection`). Never a bare `id:{folder_id}` on OneDrive: it resolves against the admin's own drive (`/me/drive`), so `aifs_get_permissions` returns `PATH_NOT_FOUND` and provisioning blocks (ms_install_10).
   - **Record the result (hard step, C.1.5.0 `collreadgrantmissing`).** The procedure has already re-read the backend.
     - Code-dir row `applied` or `skipped_noop` → set `installed_collections[{name}].member_read_grant: "granted"` in `org-config.json` (revision-aware).
     - Any other state → set `member_read_grant: "pending"` and **hard-surface**: "Members cannot read {collection-name} until the all-members reader grant on its code dir is applied — run '@ai:install-collection {collection-name}' → Apply access only, or it will be offered again on the next '@ai:publish-updates' (Step 6e)."

     Do NOT report the collection as fully installed-and-readable while it is `pending`. This closes the client-intelligence gap where a collection was marked installed but the all-members reader grant was never actually present.

This step is the **only** place install-collection touches permissions, and only through the permission-changes procedure. It is idempotent (the procedure skips what's already in effect) and safe to re-run for backfill on already-installed orgs.

### Step 5.7: Register Capability Providers (added in 2.10.0, requires core 3.10.0)

If the collection's `collection.json` declares `provides[]`:

1. **Validate the declaration.** For each provides entry: resolve the capability type definition — well-known (`/agent-index-core/capability-types/{capability}.json` via `aifs_read`) or collection-custom (`/{collection}/capability-types/{name}.json`). Verify every `required: true` operation in the type is present in the entry's `operations` map, and every `implemented_by` names a member of the collection's `api[]`. On any failure: surface the specific gap ("provides declares `{op}` implemented by `{name}`, which is not in api[]"), **skip registration for that entry**, and continue the install — a broken provides declaration must not block the collection's direct use.
   - **Undefined type (finding 56).** If neither definition exists, don't skip silently. Tell the admin in one line: "`{collection}` says it provides `{capability}`, but this org has no definition for that capability type, so it can't be registered as a provider. The collection itself works normally." Then continue.
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
