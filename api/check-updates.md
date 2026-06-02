---
name: check-updates
type: task
version: 2.6.0
collection: agent-index-marketplace
description: Comprehensive update check across infrastructure, the filesystem adapter, installed collections, and member capabilities — shows everything that has a newer version available and what to do about it.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks:
    - refresh-marketplace-cache
external_dependencies: []
reads_from: null
writes_to: null
---

## About This Task

The single command that answers "is anything out of date?" It checks four layers of the system in one sweep:

1. **Infrastructure** — is agent-index-core or agent-index-marketplace itself outdated?
2. **Filesystem adapter** — is the bundled filesystem adapter (gdrive / onedrive / s3 / etc.) behind its published version, and does it satisfy the contract level the directory advertises?
3. **Installed collections** — do any marketplace collections have newer versions available?
4. **Member capabilities** — are any of the running member's installed skills and tasks behind their collection's current version?

The result is a clear, prioritized report showing what's current, what has updates available, and what action to take for each. No files are written — this is a read-only diagnostic.

Any org member can run this task, not just admins. Everyone should be able to see their own update status. However, only admins can act on infrastructure and collection-level updates.

**Relationship to the update instruction system:** This task and the `apply-updates` task serve different purposes. `check-updates` is a diagnostic — it scans live version data from GitHub, the marketplace, and the remote filesystem to show the full picture of what is out of date. `apply-updates` is an action — it reads admin-published update instructions and executes them. A member might run `check-updates` to understand their situation, and `@ai:update` to act on it. The two are complementary: `check-updates` can detect version drift that hasn't been published yet (e.g., a new marketplace version the admin hasn't installed), while `apply-updates` only acts on what the admin has explicitly published.

### Inputs

None required. Optionally, the member can request:
- `--infrastructure-only` — skip adapter, collection, and capability checks, just check core and marketplace
- `--adapter-only` — skip everything except the filesystem adapter check
- `--collections-only` — skip infrastructure, adapter, and capability checks
- `--my-capabilities-only` — skip infrastructure, adapter, and collection checks, just check the running member's installed skills and tasks
- `--quiet` — show only items that need attention (suppress "up to date" lines)

### Outputs

A formatted update report displayed to the member. No files written.

### Cadence & Triggers

On demand. Invoked when a member or admin wants to know if anything is out of date. Also invoked internally by session-start (in lightweight mode) to generate update-available notices.

---

## Workflow

### Step 1: Read System Configuration

Read `agent-index.json` from its fixed path. Extract:
- `version` — the installed core version
- `infrastructure_directory_url` — single source of truth for the latest core + marketplace versions (added in agent-index-core 3.1.1; preferred when present)
- `core_version_url` — fallback URL for the canonical core `collection.json` (deprecated as of 3.1.1; still consulted when `infrastructure_directory_url` is absent or unreachable)
- `marketplace_version_url` — fallback URL for the canonical marketplace `collection.json` (deprecated, same fallback semantics as `core_version_url`)
- `filesystem_adapter_directory_url` — single source of truth for the latest filesystem adapter versions across all backends (used by Step 2.5)
- `remote_filesystem.backend` — the org's installed adapter `backend_id` (e.g., `"gdrive"`, `"onedrive"`, `"s3"`); used by Step 2.5 to find the matching directory entry
- `marketplace_cache_path` — where the marketplace directory cache lives

Also read local `mcp-servers/filesystem/adapter.json` (the bundled adapter manifest packaged with the install). Extract:
- `version` — the installed adapter version
- `contract_version` — the contract level the installed adapter implements (may be absent on pre-2.0 adapters)
- `backend_id` — the adapter's own backend identifier (cross-check against `remote_filesystem.backend`; surface a warning if they differ)

If `mcp-servers/filesystem/adapter.json` is not readable, Step 2.5 will record `unable to check (no installed adapter manifest)` and continue.

Read `org-config.json` from the remote filesystem via `aifs_read`. Extract:
- `installed_collections` — the list of collections the org has installed, with their versions and repo URLs
- `agent_index_version` — the core version recorded at org setup time

If `agent-index.json` is not readable: surface error and halt. This file is required.
If `org-config.json` is not readable: proceed with infrastructure and adapter checks only; skip collection checks.

**On success:** Proceed to Step 2.

---

### Step 2: Check Infrastructure Versions

The 3.1.1 + flow uses `infrastructure_directory_url` to discover both core and marketplace versions in a single fetch. Prior to 3.1.1, two separate canonical URLs were consulted; those remain as fallbacks.

> **Cache-buster (required for ALL directory/version fetches in this task — Steps 2, 2.5, 2.6; closes bug `20260601-8d20ea22-2`):** the fetch layer caches `raw.githubusercontent.com` responses keyed on the exact URL and serves stale bytes for a long time, so a directory fetch right after a release silently returns pre-release versions and the task wrongly reports "up to date." Before every directory/version GET, append a unique cache-buster query param — e.g. `?t={current unix epoch seconds}` (use `&t=…` if the URL already has a query string). This forces a fresh fetch. Apply it to `infrastructure_directory_url`, `filesystem_adapter_directory_url`, and any fallback `*_version_url`.

**Primary path (3.1.1+):**

If `infrastructure_directory_url` is set in `agent-index.json`:

1. Fetch the URL **with the cache-buster appended** (see above). It returns a JSON object with shape:
   ```json
   {
     "directory_version": "...",
     "last_updated": "...",
     "infrastructure": [
       { "name": "agent-index-core", "current_version": "X.Y.Z", ... },
       { "name": "agent-index-marketplace", "current_version": "X.Y.Z", ... }
     ]
   }
   ```
2. Find the entry where `name` is `agent-index-core`. Compare its `current_version` against the local `agent-index-core/collection.json` `version` (read via `aifs_read("/agent-index-core/collection.json")`).
3. Find the entry where `name` is `agent-index-marketplace`. Compare its `current_version` against the local `agent-index-marketplace/collection.json` `version`.
4. If the fetch succeeded but an expected entry is missing, record that piece as `unable to check (entry not found in directory)` and continue.

Record both results:
- If directory version > local version: `update available` (local → latest)
- If directory version = local version: `up to date`
- If directory version < local version: `local ahead of directory` (this is an unusual state — local install is on a version that hasn't been broadcast yet; surface as a NOTE for the admin)

**Fallback path (pre-3.1.1 installs OR primary fetch failed):**

If `infrastructure_directory_url` is absent OR the fetch fails:

1. Fall back to `core_version_url` for the core check. Fetch returns the canonical `collection.json` for agent-index-core from GitHub. Parse the `version` field.
2. Fall back to `marketplace_version_url` for the marketplace check.
3. Same comparison and result categories as the primary path.

If a fallback URL also fails, record that piece as `unable to check (network or 404)` and continue to Step 3 with a notice.

**Detect allowlist-blocked failures** (added in core 3.7.4 / marketplace 2.7.0 to close section D of idea `allowlist-failure-mode-warnings-in-tasks`): if the fetch failure shape matches the allowlist-blocked signature (HTTP 403 with empty body and no upstream-server headers, OR connection-refused, OR connection-timeout against `raw.githubusercontent.com`), do not record the result as a generic "network or 404." Instead, surface the canonical Allowlist Failure Recognition message (see `agent-index-core/collection-authoring-guide.md` § "Allowlist failure recognition") naming `raw.githubusercontent.com` as the blocked host. Recommend `@ai:verify-network-allowlist` to test all required hosts at once. The check itself still continues (so adapter and other components can be checked even if infra checks are blocked), but the admin sees the specific actionable diagnosis rather than a generic notice.

**Pre-3.1.1 installs note:** the agent-index-core repo is private, so `core_version_url` will 404 for installs that haven't yet upgraded to 3.1.1. The fix is to upgrade — once 3.1.1 lands, `infrastructure_directory_url` is migrated onto the local `agent-index.json` automatically (see apply-updates Phase 1 step 4). Until then, surface the 404 plainly so the admin understands why the check is blind.

**On success:** Proceed to Step 2.5.
**On all infrastructure fetches failing:** Queue notice that infrastructure checks couldn't complete due to network issues. Proceed to Step 2.5.

---

### Step 2.5: Check Filesystem Adapter Version

The bundled filesystem adapter (gdrive, onedrive, s3, etc.) carries its own version and contract level. This step compares the installed bundle against the published `filesystem-adapter-directory.json` so admins can see when a newer adapter — including bug-fix releases — is available. Mirrors the Step 2 pattern for `infrastructure_directory_url`.

This step is added in v2.1.1 to fix the adapter-drift portion of bug `20260501-8d20ea22`. Before 2.1.1 this drift class was silent.

**Determine the directory URL:**

- If `filesystem_adapter_directory_url` is set in `agent-index.json`, use it.
- Otherwise, fall back to the canonical raw URL in `agent-index-resource-listings`:
  `https://raw.githubusercontent.com/agent-index/agent-index-resource-listings/refs/heads/main/filesystem-adapter-directory.json`
  (same fallback shape as the `infrastructure_directory_url` fix shipped in 2.1.0.)

**Determine the installed backend:**

- Use `remote_filesystem.backend` from `agent-index.json`. If absent, fall back to `remote_filesystem.exec.adapter`.
- If neither is set, record `no backend configured — skipping adapter check` and proceed to Step 3.

**Fetch and compare:**

1. Fetch the directory URL. It returns a JSON object with shape:
   ```json
   {
     "directory_version": "...",
     "last_updated": "...",
     "adapters": [
       {
         "backend_id": "gdrive",
         "current_version": "X.Y.Z",
         "contract_version": "A.B.C",
         "...": "..."
       }
     ]
   }
   ```
2. Find the entry whose `backend_id` equals the org's installed backend.
   - **Not found:** record `org adapter — no directory tracking` (the org may be running a custom or private adapter). Skip the drift check; proceed to Step 3.
3. Compare `entry.current_version` against the locally read `mcp-servers/filesystem/adapter.json` `version`:
   - directory > local → `↑ update available`
   - directory == local → `✓ up to date`
   - directory < local → NOTE: `local ahead of directory` (this is unusual; the local install is on a version that hasn't been broadcast yet — surface for admin awareness)
4. Compare `entry.contract_version` against the locally read `adapter.json` `contract_version`:
   - If `installed_contract` is missing **or** numerically less than `entry.contract_version`, record a SECONDARY NOTE on the same adapter row: `contract upgrade available (X.Y.Z → A.B.C) — new adapter ops unlocked by upgrade`. This is informational; the primary status pill is set by step 3.

4a. **Compute collection contract requirements and escalate contract gaps that block installed capabilities** (added in v2.5.0; closes idea `contract-version-aware-update-surfacing`). The basic contract-gap signal in step 4 is informational — it tells the admin "an upgrade is available" but doesn't tell them whether anything actually depends on it. v2.5.0 wires the gap to the org's installed-collection footprint to produce a stronger signal when the contract gap is a hard blocker.

   1. For each entry in `org-config.json` → `installed_collections[]` with `status: "installed"`, read the remote `aifs_read("/{name}/collection.json")` (reuse the Step 3 cache).
   2. Check each collection's `collection.json` for an optional top-level `requires_contract_version` field (semver string, e.g. `"2.0.0"`). Collections that don't declare it are treated as `"1.0.0"` (the floor — every supported adapter implements contract 1.0.0 ops).
   3. Compute `required_contract = max(requires_contract_version)` across all installed collections. Track which collection(s) drove the max so we can name them in the report.
   4. Compare `installed_contract` (from local `adapter.json`) against `required_contract`:
      - **`installed_contract >= required_contract`** — no blocker. Optionally render a passive informational column on the adapter row: `contract: {installed_contract} ✓ meets org's required minimum`.
      - **`installed_contract < required_contract`** — render a **BLOCKER** as the first section of the report, above the Infrastructure table:

        > **⚠ Filesystem adapter contract below required minimum**
        >
        > Your installed adapter is at contract `{installed_contract}`. Installed collection(s) require contract `{required_contract}`: {comma-separated list of collections driving the max}.
        >
        > Capabilities in those collections will fail at runtime until the adapter is upgraded. Run `@ai:publish-updates --check-upstream` (admin) to fetch a contract-`{required_contract}+` adapter bundle. The current contract gap is a hard prerequisite — not just a nice-to-have upgrade.
      - **`installed_contract < entry.contract_version` BUT `installed_contract >= required_contract`** — contract upgrade is available upstream but no installed collection depends on it yet. Keep the existing SECONDARY NOTE on the adapter row from step 4 (informational opportunity); do NOT escalate to a blocker.
   5. Record the result in the lightweight-mode `contract_blockers[]` field so session-start can surface a one-line warning at next session entry: `[{ "required_contract": "...", "installed_contract": "...", "driving_collections": [...] }]` if blocked, empty otherwise.

   **Notes:**
   - `requires_contract_version` is opt-in for collection authors. The default-to-`"1.0.0"` baseline means existing collections without the field continue to render with no blocker (the floor everyone meets).
   - The `requires_contract_version` field is per-collection, not per-API-member, for now. A future iteration could move it down to individual capabilities (since only some tasks within a collection may use contract-2.0 ops), but the per-collection grain is sufficient to surface the blocker class today.
   - Collection authors who use contract-2.0 adapter ops (`aifs_share`, `aifs_unshare`, `aifs_get_permissions`, `aifs_search`, `aifs_transfer_ownership`, revision-aware writes) MUST declare `requires_contract_version: "2.0.0"` in their `collection.json`. See `agent-index-core/collection-authoring-guide.md` § "Declaring adapter contract requirements" for details.
5. If `mcp-servers/filesystem/adapter.json` is missing locally, record `unable to check (no installed adapter manifest)` and proceed.
6. If the directory fetch fails (network, 404, etc.), record `unable to check (network or 404)` and proceed.

**On success:** Proceed to Step 2.6.
**On directory fetch failure:** Queue notice that adapter check couldn't complete; proceed to Step 2.6 without aborting the workflow.

---

### Step 2.6: Check Registered Binary Tools (added in core 3.4.0)

The org may declare native binary tools via `org-config.json` → `binaries{}`. Each entry pins a target version and policy. The `infrastructure-directory.json` published by the central agent-index registry declares the latest `current_version` and `min_required_version` for each binary. This step compares the org-pinned version against (a) the directory's published current/min and (b) the locally-installed version, and surfaces the deltas.

For each entry in `infrastructure-directory.json` → `binaries[]`:

1. Look up the binary in `org-config.json` → `binaries{}`. If not present, record:
   `available — not pinned (admin can run @ai:pin-binary-version <name>)`. Do not flag as needing action.
2. If present, resolve the **target version** per the org's `policy`:
   - `pinned` → `org_entry.version` exactly.
   - `min` → max of `org_entry.version` and the locally-installed version.
   - `latest` → directory's `current_version`.
3. Validate the target against the directory's `min_required_version`. If `target < min_required_version`, record:
   `↑ admin must update pin — org pin <version> below required floor <min>`.
4. Read the locally-installed version from the path declared by the directory's `version_file` (e.g. `mcp-servers/permission-helper-go/version.txt`). If file missing, record `not installed`.
5. Compare local to target:
   - `local == target` → `up to date`.
   - `local != target` (or not installed) → `↑ install/update available (local <local> → target <target>)`. Surface this whether the change is upgrade or downgrade — both are intentional under "pinned" policy.
6. If the directory has no `platforms[]` entry matching the host's `os`/`arch`, surface:
   `↑ no published binary for <os>/<arch> — ask admin to upload`.

If the directory fetch fails (network, 404), record `unable to check (network or 404)` for binaries and proceed.

**On success:** Proceed to Step 3.

---

### Step 3: Refresh Marketplace Cache and Check Collection Versions

Invoke `run agent-index-marketplace task refresh-marketplace-cache` in automatic mode to ensure the marketplace directory is fresh.

Read the marketplace directory from cache. For each collection in `org-config.json`'s `installed_collections` (excluding `agent-index-core` and `agent-index-marketplace`, which were checked in Step 2):

1. Find the matching entry in the marketplace directory by collection name
2. Compare the `current_version` in the directory against the `version` in `org-config.json`

Record the result for each:
- If directory version > installed version: `update available` (installed → latest)
- If directory version = installed version: `up to date`
- If collection not found in directory (org-authored collection): `org collection — no marketplace tracking`
- If marketplace cache unavailable: `unable to check`

Additionally, for each installed collection on the remote filesystem, read its `collection.json` via `aifs_read` and compare against `org-config.json`'s recorded version. If these differ, it means the remote filesystem was updated but `org-config.json` wasn't updated to match — flag as `version mismatch — remote filesystem differs from org-config`.

**On success:** Proceed to Step 4.

---

### Step 4: Check Member Capability Versions

This step compares each member-installed capability against the version frontmatter of the corresponding capability file on the remote filesystem — **not** against the collection-level `current_version`. Capability files version independently of their collection (collection-level changes like trigger array updates, README polish, or dependency manifest tweaks bump `collection.json` `version` without touching most `api/{name}.md` files), so comparing per-capability member-index versions against collection-level versions produces false "upgrade available" results.

This step was rewritten in v2.1.2 to fix bug `20260430-8d20ea22`. The pre-2.1.2 algorithm — compare member-index per-capability `version` against `collection.json` `current_version` — was structurally wrong: `member-index.json` stores the capability's `.md` frontmatter version (set at install/upgrade time by `org-setup`'s install flow which reads `aifs_read("/{collection}/api/{name}.md")`), not the collection-level version.

**Algorithm:**

Read the running member's `member-index.json`. Build a single combined list from `installed.skills` + `installed.tasks`. Maintain a per-run cache keyed by `/{collection}/api/{name}.md` to avoid re-reading the same file twice within one workflow run.

For each entry:

1. Let `member_version = entry.version`, `path = "/{entry.collection}/api/{entry.name}.md"`.
2. If `path` is in the cache, use the cached result. Otherwise `aifs_read(path)` and cache the response (success or error).
3. Classify the result:
   - **Read succeeded, frontmatter parsed, `version` field present:**
     Compare `remote_version` vs `member_version` using semver:
     - `remote > member` → `upgrade available` (record severity: MAJOR / MINOR / PATCH)
     - `remote == member` → `current`
     - `remote < member` → NOTE: `local ahead of remote` (unusual; the member's install is on a version not yet broadcast — surface for awareness, not an error)
   - **Read succeeded, but frontmatter has no `version` field** (malformed file):
     `unable to check (no version in frontmatter)` — surface the path; do not abort the workflow.
   - **`PATH_NOT_FOUND` and the collection directory exists** (`aifs_exists("/{collection}/")` returns true):
     `capability removed from collection` — the file is gone but the collection is still present. The member's entry is now an orphan. Hint: "Say `@ai:setup` and ask to remove `{name}` — it's no longer in the {collection} collection." (See follow-up idea `org-setup-suggest-orphan-cleanup` for the consuming surface.)
   - **`PATH_NOT_FOUND` and the collection directory is also missing:**
     `collection unavailable` — the entire collection is gone from the remote. This is already surfaced by Step 3 as `missing — directory not found`, so suppress per-capability rows here to avoid double-flagging; just note the collection name once.
   - **Other read failure (auth, network, etc.):**
     `unable to check (read failed)` — surface the path and the underlying error class.
4. Continue to the next entry. Never abort the workflow on a single capability's failure.

Record the result for each capability with the comparison values that were actually used (member version, remote version, status, and severity if applicable).

**On success:** Proceed to Step 4.5.

---

### Step 4.5: Detect Capabilities Available to Install (added in v2.3.0)

This step surfaces a fifth class of signal: **capabilities present in the org's installed collections but not yet installed for the running member**. The org might have rolled out a new task in a collection upgrade (e.g., `projects` 3.0.5 → 3.0.6 added a new `archive-project-v2` task) and the member never picked it up. Without this step, the member sees the collection-level "update available" signal but no enumeration of *what* is newly available to them.

This step is gated on `--show-available` (default: on; suppress with `--no-show-available` for noise-averse callers).

**Algorithm:**

For each entry in `org-config.json` → `installed_collections[]` with `status: "installed"`:

1. Read the collection's `collection.json` from remote: `aifs_read("/{collection}/collection.json")`. This was already done in Step 3 — reuse the cached parse.
2. Enumerate the `api[]` array. Each entry is either a bare string (capability name) or an object with `name` and `triggers[]`. Normalize to a list of names.
3. Cross-reference against the running member's `member-index.json`: filter to capability names NOT present in `installed.skills[]` or `installed.tasks[]` for this collection.
4. For each available-but-uninstalled capability:
   - Read `aifs_read("/{collection}/api/{name}.md")` and parse the frontmatter to get `version` and `type` (skill or task) — reuse the Step 4 cache if it's already there.
   - If the file is missing or malformed: skip it (cap was orphaned at the collection level — not eligible to surface as "install this").
   - Record `{ name, type, collection, latest_version }`.

After the loop, the result is a flat list `available_capabilities[]` containing all the install-able items across all installed collections the member hasn't picked up.

**Empty case.** If `available_capabilities[]` is empty (the member has everything from every installed collection), suppress the section in the report rendering — no need to surface an empty table.

**On success:** Proceed to Step 5.

---

### Step 5: Present Update Report

**Check update instruction status:** Before compiling the report, read `/shared/updates/latest.json` from the remote filesystem via `aifs_read`. If it exists, compare `latest_id` against the member's `last_applied_update` from `member-index.json`. Record whether update instructions are pending — this influences the "What to do" section of the report.

Compile all results into a prioritized report.

**Full mode (default):**

> **Agent-Index Update Report**
> Checked: {timestamp}
>
> **Infrastructure**
> | Component | Installed | Latest | Status |
> |---|---|---|---|
> | agent-index-core | 1.0.0 | 1.1.0 | ↑ update available |
> | agent-index-marketplace | 1.0.0 | 1.0.0 | ✓ up to date |
>
> **Filesystem Adapter**
> | Adapter | Backend | Installed | Latest | Contract | Status |
> |---|---|---|---|---|---|
> | agent-index-filesystem-gdrive | gdrive | 2.1.3 | 2.2.0 | 1.0.0 → 2.0.0 | ↑ update available |
>
> **Installed Collections**
> | Collection | Installed | Latest | Status |
> |---|---|---|---|
> | projects | 2.0.0 | 3.0.0 | ↑ update available |
> | strategy | 1.0.0 | 1.0.0 | ✓ up to date |
> | capture | 1.0.0 | 1.0.0 | ✓ up to date |
>
> **Your Installed Capabilities** ({N} total)
> | Capability | Type | Collection | Your Version | Latest Version | Status |
> |---|---|---|---|---|---|
> | create-project | task | projects | 2.0.0 | 3.0.0 | ↑ upgrade available |
> | edit-project | task | projects | 2.0.0 | 2.0.0 | ✓ current |
> | capture | task | capture | 1.0.0 | 1.0.0 | ✓ current |
> | old-task | task | projects | 1.0.0 | — | × removed from collection |
> ...
>
> *"Latest Version" is the `version` field in the capability file's frontmatter on the remote filesystem (`/{collection}/api/{name}.md`), not the collection-level `current_version`. Capability files version independently of their collection.*
>
> **Available to Install** ({N} capabilities) — *omitted if zero*
> | Capability | Type | Collection | Latest Version |
> |---|---|---|---|
> | archive-project-v2 | task | projects | 1.0.0 |
> | revisions | task | capture | 1.1.0 |
> | manage-recipients | task | email-triage | 1.0.0 |
> ...
>
> *Capabilities present in your org's installed collections that you haven't installed yet. Say `@ai:setup` to install any of these.*
>
> **Summary:** {N} updates available, {M} items up to date, {P} unable to check, {Q} capabilities available to install.
>
> **What to do:**
> {if update instructions are pending (last_applied_update behind latest.json)}: "Your admin has published update instructions. Say '@ai:update' to apply them — this will handle infrastructure, collection, and capability updates in one step."
> {if no update instructions pending but updates detected}:
> {if infrastructure updates (core/marketplace) AND member is admin}: "Infrastructure has upstream updates. Say '@ai:publish-updates --check-upstream' to fetch the new versions from GitHub, sync to remote, regenerate the bootstrap zip if needed, and broadcast to members — all in one step." (Replaces the pre-3.5.0 manual `git pull → @ai:edit-org → @ai:publish-updates` ritual; the new flag in publish-updates Step 0a does the fetch automatically.)
> {if infrastructure updates AND member is NOT admin}: "Infrastructure updates require an org admin. Contact your org admin to publish the upstream updates."
> {if adapter update available AND member is admin}: "Filesystem adapter has an update available. Say '@ai:publish-updates --check-upstream' — the new flow detects adapter changes via the prerequisite-detection step and regenerates the bootstrap zip automatically. (The pre-3.5.0 path was '@ai:edit-org → Update Adapter Bundle, then @ai:publish-updates' — still works as a fallback if you want to inspect the bundle before publishing.)"
> {if adapter update available AND member is NOT admin}: "Contact your org admin to refresh the filesystem adapter bundle."
> {if collection updates}: "{if member is admin: Say '@ai:marketplace' to upgrade collections, then '@ai:publish-updates' to publish instructions for members. | if not admin: Contact your org admin to upgrade the {collection} collection and publish update instructions.}"
> {if capability upgrades}: "Say '@ai:update' if update instructions are available, or '@ai:setup' to upgrade your installed capabilities manually."
> {if available_capabilities is non-empty}: "Say '@ai:setup' to install any of the {Q} capabilities listed under 'Available to Install'."

**Quiet mode** (`--quiet`):

Show only items with updates or issues. If everything is current:
> "Everything is up to date."

**Lightweight mode** (used internally by session-start):

Return a structured result (not displayed) containing:
```json
{
  "infrastructure_updates": [],
  "adapter_updates": [
    {
      "backend_id": "gdrive",
      "installed": "2.1.3",
      "latest": "2.2.0",
      "contract_installed": "1.0.0",
      "contract_latest": "2.0.0"
    }
  ],
  "collection_updates": [{"name": "projects", "installed": "2.0.0", "latest": "3.0.0"}],
  "capability_upgrades": [{"name": "create-project", "collection": "projects", "installed": "2.0.0", "latest": "3.0.0"}],
  "available_capabilities": [{"name": "archive-project-v2", "type": "task", "collection": "projects", "latest_version": "1.0.0"}],
  "contract_blockers": [],
  "errors": [],
  "everything_current": false
}
```

`adapter_updates` is `[]` when the adapter is up to date or when the directory could not be reached. The `contract_installed` field may be null on pre-2.0 adapters that didn't declare a contract version.

This structured result is consumed by session-start to generate update-available notices without displaying the full report.

**On success:** Task complete.

---

## Directives

### Behavior

This task is diagnostic — it tells you what's going on, clearly and completely. It never modifies anything. It never triggers an upgrade. It reports and recommends.

Be specific about what the member can do. If they're an admin, give them the exact command. If they're not an admin, tell them to contact their admin. Don't give a member actions they can't take.

When running in lightweight mode (triggered by session-start), be silent. Return the structured result and nothing else. Session-start will handle surfacing any notices.

### Version Comparison

Use semantic version comparison throughout. `1.2.0` > `1.1.9` > `1.1.0` > `1.0.0`. Compare MAJOR first, then MINOR, then PATCH. Do not use string comparison.

When an update involves a MAJOR version change, note this in the report:
> "↑ **major update** available (1.0.0 → 2.0.0) — may require migration"

MAJOR updates have breaking changes and may require running upgrade scripts. MINOR and PATCH updates are non-breaking.

### Constraints

Never modify any file. This task is strictly read-only.

Never trigger an upgrade. Only report and recommend.

Never suppress infrastructure update notifications. Even in quiet mode, if core or marketplace has an update available, show it.

Never show another member's capability status. Only the running member's own installed capabilities.

### Edge Cases

If the org has no installed collections (brand new org): skip the collections section. Only show infrastructure and adapter.

If `mcp-servers/filesystem/adapter.json` is missing locally: render the Filesystem Adapter row as `unable to check (no installed adapter manifest)`. Do not abort the workflow — this can happen on partially-bootstrapped installs.

If `remote_filesystem.backend` is set to a `backend_id` not present in the directory (custom or private adapter): render the row as `org adapter — no directory tracking`. Do not flag as an error.

If `mcp-servers/filesystem/adapter.json` `backend_id` differs from `remote_filesystem.backend` in `agent-index.json`: render the adapter row as `backend mismatch — agent-index.json says {x}, adapter.json says {y}`. Surface the mismatch as a NOTE; do not silently pick one over the other.

If the member has no installed capabilities (brand new member): skip the capabilities section. Show infrastructure and collections if the member is an admin, otherwise show a note: "You haven't installed any capabilities yet. Say '@ai:setup' to get started."

If a capability's `api/{name}.md` is missing on the remote (`PATH_NOT_FOUND`) but the collection directory still exists: render the capability row as `× removed from collection` with no "Latest Version" value. Hint: "Say `@ai:setup` and ask to remove this — it's no longer in the {collection} collection."

If a capability's `api/{name}.md` is missing on the remote and the collection directory is also missing: do not render per-capability rows for that collection's orphaned member-index entries — the collection-level "missing — directory not found" row from Step 3 already surfaces the situation; per-capability noise would be redundant. Show a single suppressed-row count instead: "(N capabilities from {collection} suppressed — collection directory not found)".

If a capability's `api/{name}.md` exists but its frontmatter has no `version` field: render as `unable to check (no version in frontmatter)` with the file path. Do not abort the workflow.

If the remote filesystem is unreachable: report what can be checked locally (capability versions vs. local member-index records) and note that infrastructure, collection, and marketplace version checks require remote filesystem connectivity.

If `org-config.json` records a collection that no longer exists on the remote filesystem: flag it as `missing — directory not found` (consistent with `list-org-collections` behavior).

If the member invokes this task with a specific collection name (e.g., "check updates for projects"): show only that collection's status plus the member's capabilities from that collection. Skip everything else.
