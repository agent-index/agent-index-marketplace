# Agent-Index Marketplace — Changelog

## [2.13.0] — 2026-06-25 — Release C: backend distribution (members never fetch from GitHub)

- **check-updates 2.9.0:** the member-facing source of truth for "latest" is `/shared/dist/manifest.json` on the org backend, not `infrastructure_directory_url`. Member-current = installed vs the backend manifest; org-vs-upstream = an admin-only `git` check. Dissolves the `listinglag`/`shasolve` stale-version-check confusion (the org's published manifest is the authority and can't be "behind a stale GitHub listing").
- **download-collection 2.3.0:** sources a collection from the admin's tag-pinned **local clone** (via the `clone-script-generator`), not a `zip_url` GitHub download; uploads to the backend and republishes `/shared/dist/`. The GitHub `zip_url` path survives only as the deprecated fallback.
- Pairs with core 3.18.0; standards.md § "Distribution: backend-first."

## [2.12.0] — 2026-06-16 — Release B: install-collection setupresp guardrail

Release record: core-improvements releases/ms365-adapter/ (Release B). Pairs with core 3.13.0.

### Fixed (install-collection)
- **setupresp guardrail (bug 20260615-8d20ea22-setupresp):** writing `collection-setup-responses.md` with `setup_status: complete` is now a mandatory step that is NOT skipped by an "accept defaults / don't ask me questions" shortcut — a defaults install writes the responses file (empty-but-complete when the collection has no org-level parameters), verified by read-back before the collection is marked installed. Closes the failure mode where a bulk/defaults install left collections with no responses file, which org-setup then treats as a hard block for **every** member trying to install any capability from that collection. Installing several collections "with defaults" must loop install-collection per collection — never bulk-upload + register without responses files.

## [2.11.2] — 2026-06-12 — Deploy Readiness: download-collection tail reconstruction

Release record: core-improvements releases/deploy-readiness/. download-collection: final edge case (pre-existing remote collection directory → surface, overwrite-or-abort, never silent merge) RECONSTRUCTED (reviewed; inline provenance note); sentinel re-stamped over verified-complete content.

## [2.11.1] — 2026-06-10 — repair: tail truncations introduced in 2.11.0

The 2.11.0 release commits contained tail-truncated capability specs — a mount-mediated read-modify-write during version restamping wrote stale truncated views back to disk (FCI-1 class; see bug 20260608-8d20ea22-003039-trunc and release record platform-reliability/build-record.md). 2.11.1 splices the complete pre-release tails back under the 2.11.0 content edits, verified byte-exact against the pre-release endings, and stamps the repaired files with AIFS:FILE-END sentinels. No behavioral changes beyond 2.11.0.

## [2.11.0] — 2026-06-09 — Platform Reliability: SHA-pinned distribution fetches; directory is source of truth

Release record: core-improvements `releases/platform-reliability/`. Closes the marketplace half of bug `20260601-8d20ea22-2` and the detection half of `20260607-8d20ea22-131906-d1rv`.

### Changed
- **check-updates 2.8.0**: Steps 2/2.5/2.6 use the Distribution fetch protocol (SHA-pinned; standards.md) instead of `?t=` cache-busters (stripped on the raw redirect — the 2.9.0 mitigation did not survive it). Fallback-sourced results are labeled and never sufficient to report "✓ up to date."
- **refresh-marketplace-cache 2.4.0**: Step 3 fetches via the SHA-pinned protocol; `cache-metadata.json` records `fetch_source` + `pinned_sha`. Step 4 gains the content-signal newer-than-cache rule (d1rv): newer iff `directory_version` increased, OR equal version AND newer `last_updated` AND content hash differs. No-downgrade guard unchanged.
- **upgrade-collection 1.3.0**: the marketplace directory is the source of truth. Branch-form `zip_url`s are rewritten to the SHA-pinned codeload form before download. A directory-vs-zip version mismatch after a SHA-pinned refresh now HALTS as a listing bug — the previous "proceed with the actual zip version?" bypass is removed (bypassing the directory is how unadvertised content reaches an org).
- **download-collection 2.2.0**: branch-form `zip_url`s rewritten to the SHA-pinned codeload form before download (D5 finding: all marketplace listing `zip_url`s are branch-form, i.e., mutable). `download-and-install-collection` inherits via delegation.

## [2.10.1] — 2026-06-08 — capture folder_id + grant code-dir reader at install (Option B; closes cr01/cr02)

### Fixed

- **download-collection / install-collection capture the collection folder's Drive ID** (`aifs_stat("/{name}")`) into `org-config.json` `installed_collections[].folder_id`. Members address collections by this ID for capability sync (they are not Drive members and cannot resolve `/{name}` by name — bug 20260606-…-db13).
- **install-collection Step 5.5 now grants `all@` reader on the collection code dir `/{name}/`** via the same batched permission-change-helper Accept as the collaborative-acls writers — UNCONDITIONALLY for every installed collection, even those with no `collaborative-acls.json`. Without this, non-admin members cannot read a newly-installed collection to sync its capabilities (the brand-book 2026-06-08 failure, bug 20260608-…-cr01). The Step 5.5 absent-declaration gate no longer skips the whole step.

### Requires Admin Attention

- Requires core 3.10.1 (folder_id schema + id-anchored member reads + Migration 4 backfill for existing collections). Upgrade core first. Existing collections are backfilled by an admin `@ai:update` (core 3.10.1 Migration 4); new installs are correct from the start.

## [2.10.0] — 2026-06-07 — capability-provider registration + survey (companion to core 3.10.0)

### Added

- **install-collection Step 5.7** — validates `provides[]` declarations against capability-type definitions, prompts the admin, registers providers into `org-config.json` `capability_providers` (revision-aware) + logs `provider-register`; surveys `requires[]` with spec-conformant notices (never blocks install).
- **upgrade-collection Step 6.7** — refreshes a registered provider's `operations_available`/`capability_version` when `provides[]` changes; flags provides-but-unregistered collections without auto-registering.
- **check-updates Step 4.7** — read-only Capabilities survey: registered providers, unregistered provides, satisfied/unmet requires.

### Fixed

- **upgrade-collection Step 10 now pairs the published-state snapshot with the update-log write** — closes bug 20260605-8d20ea22-234936-c06b (every upgrade previously caused a false diff in the next publish-updates run).

### Requires Admin Attention

- Requires core 3.10.0 (capability_providers schema + capability types). Upgrade core first.

## [2.9.1] — 2026-06-04 — upgrade-collection provisioning detection

### Added

- **`upgrade-collection` 1.0.0 → 1.1.0 — Step 6.5 "Detect Provisioning Needs".** After the staging-vs-remote diff, if the target version adds or changes `collaborative-acls.json` or `setup/collection-setup.md`, the task sets `provisioning_needed`: the Step 7 plan shows a **PROVISIONING REQUIRED** block (including any "Requires Admin Attention" text from the staged CHANGELOG), and the Step 11 result explicitly marks the upgrade **NOT YET FUNCTIONAL** until the admin runs `@ai:install-collection {name}` (idempotent — re-runs the setup interview preserving answers, creates new shared folders, applies ACL grants via the permission helper). The task must not present such an upgrade as complete and should offer to run install-collection immediately.

### Notes

- Encodes post-release finding **F1** from the strategy 1.0.3 → 1.1.0 upgrade (2026-06-04): file sync + verification passed while `share-strategy` was broken org-wide — the pointer-index folder and its `all@` writer grant were never provisioned, because `upgrade-collection` is file sync only.
- Companion release: core 3.8.1 (`apply-updates` member-state self-heal — finding F3).

---

## [2.9.0] — 2026-06-02 — cache-bust directory fetches

### Fixed

- **`check-updates` 2.5.0 → 2.6.0 and `refresh-marketplace-cache` 2.2.0 → 2.3.0 now append a cache-buster query param (`?t={unix_seconds}`) to every directory/version fetch** (closes bug `20260601-8d20ea22-2`). The fetch layer caches `raw.githubusercontent.com` responses keyed on the exact URL and serves stale bytes long after a push. Symptom: an org running `@ai:check-updates` shortly after a release silently reads pre-release versions and reports "✓ up to date" (no error), and `refresh-marketplace-cache` re-caches the *old* catalog. check-updates applies the buster to `infrastructure_directory_url`, `filesystem_adapter_directory_url`, and fallback `*_version_url` (Steps 2 / 2.5 / 2.6); refresh-marketplace-cache applies it to the marketplace directory fetch. Companion: `agent-index-core` `publish-updates` 3.6.0 applies the same buster to its `--check-upstream` infrastructure + zip fetches.

### Notes

- `check-updates` 2.5.0 → 2.6.0; `refresh-marketplace-cache` 2.2.0 → 2.3.0; `collection.json` 2.8.0 → 2.9.0. Bootstrapping caveat: because this very bug hides new versions, the **first** deployment of 2.9.0 to an org must be pulled with a manual cache-bust (a distinct query param on the directory URL); subsequent fetches self-bust.

---

## [2.8.0] — 2026-05-31 — collaborative-ACL provisioning

### Added

- **`install-collection` 2.0.0 → 2.1.0: Step 5.5 — provision collaborative-folder ACLs.** After writing setup-responses and marking the collection installed, install-collection now checks for a `collaborative-acls.json` at the collection root. If present, it resolves the declared `{param}` placeholders (from `collection-setup-responses.md` + `org-config.json`), filters grants already satisfied (idempotent), composes a single `permission-change-helper` spec, and routes it for admin review + Accept. This is the supported mechanism for granting members write access to a collection's shared collaborative folders (e.g. bug-reports' `bugs/`) under the least-privilege access model — collections never call `aifs_share` directly. Pairs with `agent-index-core` standards.md § "Collaborative Folder ACLs" and bug-reports 1.3.0. Supports bug `20260531-8d20ea22`.

### Notes

- `install-collection` task 2.0.0 → 2.1.0. `collection.json` 2.7.0 → 2.8.0. `install-collection-manifest.json` `collection_version` → 2.8.0; remaining API manifests' `collection_version` reconcile to 2.8.0 on members' next `apply-updates` (manifest-sync subroutine).

---

## [2.7.0] — <RELEASE_DATE> — companion to core 3.7.4 "Closing the Loop"

### Added

- **Allowlist failure-mode detection branches** in 4 network-dependent tasks (implements section D of idea `allowlist-failure-mode-warnings-in-tasks`). When a fetch fails with the allowlist-blocked signature (HTTP 403 + empty body + no upstream headers, OR connection-refused, OR connection-timeout), the task surfaces the canonical Allowlist Failure Recognition message naming the missing host instead of a generic network-error message. Detection heuristic + canonical message format documented in `agent-index-core/collection-authoring-guide.md` (added in core 3.7.4).

  Tasks updated:
  - `refresh-marketplace-cache` 2.1.0 → 2.2.0 (Step 3 no-cache hard-stop path)
  - `download-collection` 2.0.0 → 2.1.0 (Step 4 download/upload path)
  - `download-and-install-collection` 2.0.0 → 2.1.0 (Step 2 — sub-task delegation; pass-through messaging)
  - `check-updates` 2.4.0 → 2.5.0 (Step 2 infrastructure-fetch path; check continues with notice but admin sees specific actionable diagnosis)

### Notes

- All API manifests' `collection_version` bumped 2.6.0 → 2.7.0.
- `install-collection` was originally listed in the scope but does NOT do HTTP fetches (it reads from remote filesystem via `aifs_*` on already-downloaded local + remote state). No edit needed; scope-divergence captured in the 3.7.4 decision record.
- Companion release: agent-index-core 3.7.4 "Closing the Loop" — ships the authoring-guide section, the publish-updates Step 0a allowlist branch (the other GitHub-fetch surface), and three other unrelated work-streams.

---

## [2.6.0] — 2026-05-20 — companion to core 3.7.3 "Install-Layer Reliability"

### Fixed

- **Allowlist message in `refresh-marketplace-cache.md` Step 3** (Step 3 no-cache hard-stop path) and **`setup/collection-setup.md` Step 5** (initial-cache-fetch hard-stop path) extended from the single `raw.githubusercontent.com` host to the full canonical four-host list: `raw.githubusercontent.com`, `github.com`, `api.github.com`, `codeload.github.com`. The previous one-host messages were a known drift relative to the broader allowlist need — `codeload.github.com` was missing from every prose surface; admins who followed the marketplace-task guidance allowlisted only one host and then failed on first collection install. (Companion fix for bug `20260515-8d20ea22`; canonical host list lives in `agent-index-core/templates/network-allowlist.template.json` as of core 3.7.3.) Messages also reference the new `@ai:verify-network-allowlist` task as the canonical way to test coverage across all required hosts.

### Notes

- **Versions:** `agent-index-marketplace` collection 2.5.0 → 2.6.0. `refresh-marketplace-cache` task 2.0.0 → 2.1.0. All API manifests' `collection_version` bumped to 2.6.0. No other task content changes.
- **Coordination with core 3.7.3:** these are pure prose updates that mirror core's canonical host list. The actual reachability-check logic and standalone verify task live in core; the marketplace messages just point admins at the same hosts and tooling.

---

## [2.5.0] — 2026-05-13

### Added

- **`check-updates` 2.3.0 → 2.4.0: contract-version-aware surfacing** (closes idea `contract-version-aware-update-surfacing`). New Step 4a computes `max(requires_contract_version)` across all installed collections and compares against the local adapter's `contract_version`. Three surfacing modes: (a) `installed_contract >= required` — optional passive informational column on the adapter row; (b) `installed_contract < required` — top-of-report BLOCKER naming the driving collection(s) plus the recommended remedy (`@ai:publish-updates --check-upstream`); (c) `installed_contract < directory_contract` but no installed collection requires the higher contract — keep the existing passive opportunity NOTE on the adapter row (no escalation).
- **Lightweight-mode result gains `contract_blockers[]`**, populated by Step 4a so session-start can surface a one-line warning at next session entry.
- Collections may declare an optional top-level `requires_contract_version` field in `collection.json` (semver string). Authoring guidance added to `agent-index-core/collection-authoring-guide.md` § "Declaring adapter contract requirements" — defaults to `"1.0.0"` for collections that don't declare it (the floor every supported adapter meets); set to `"2.0.0"` for any collection that uses contract-2.0 ops (`aifs_share`, `aifs_unshare`, `aifs_get_permissions`, `aifs_search`, `aifs_transfer_ownership`, revision-aware writes).

### Notes

- `check-updates` task version bumped 2.3.0 → 2.4.0. All API manifests' `collection_version` bumped 2.4.0 → 2.5.0.
- Companion releases ship in core 3.7.2 (CHANGELOG documents the multi-repo cleanup patch).

---


## [2.4.0] — 2026-05-11

### Added

- **`check-updates` Step 4.5 — detect capabilities available to install.** New step that runs after the per-capability version comparison (Step 4) and before the report rendering (Step 5). For each entry in `org-config.json` → `installed_collections[]` with `status: "installed"`, enumerates the collection's `collection.json` `api[]` array, cross-references against the member's `member-index.json`, and records any capabilities present in the collection but NOT installed for this member. Result lands in a new `available_capabilities[]` field in the lightweight-mode result and as a new "Available to Install" table in the full report. Mirrors the existing "Available" section in `org-setup`'s management dashboard — so members running `@ai:check-updates` get a complete picture of both *what's out of date* and *what's available to pick up* without needing to consult `@ai:setup` separately.

  Gated on `--show-available` (default: on). Empty case (member has installed every capability from every installed collection) suppresses the section. "What to do" gains a corresponding hint: "Say `@ai:setup` to install any of the {Q} capabilities listed under Available to Install."

  Closes idea `check-updates-show-available-to-install`. Pairs naturally with the disambiguation work landing in core 3.7.0 — admins newly routed to check-updates get the full signal, including what they could install for themselves on this org's footprint.

### Changed

- `check-updates` task version 2.2.1 → 2.3.0. All API manifests' `collection_version` bumped 2.3.0 → 2.4.0.

---


## [2.3.0] — 2026-05-07

### Added

- **New `@ai:upgrade-collection <name>` task** at `api/upgrade-collection.md`. Closes bug `20260502-8d20ea22` — the `download-collection` and `download-and-install-collection` tasks have referenced `@ai:upgrade-collection` since v3.1.0 ("'{collection}' is already installed. To upgrade it, say '@ai:upgrade-collection {name}'.") but no such task existed; admins reading the prompt hit a dead end. After this release, the alias resolves to a real task. Surface: `@ai:upgrade-collection <name> [--to <version>] [--check-upstream] [--dry-run]`. Counterpart to `@ai:publish-updates --check-upstream` (infrastructure) and `@ai:edit-org → Update Adapter Bundle` (filesystem adapters). Workflow: refresh marketplace cache, look up target version (validate against `min_required_version` if defined), download the new zip from the directory's `zip_url`, stage, compute diff against remote, surface plan summary, get admin Y/N, upload changed files via `aifs_write` with LF normalization, delete remote-only files (with the documented preserve list), update `org-config.json` with the new version + upgraded_date, write a `collection-update` CHANGELOG entry to `/shared/updates/update-log.json` for member-side `@ai:apply-updates` consumption, verify post-upload, surface result.

- **Setup-responses preservation as a documented Directive.** During upgrade, `<collection>/setup/collection-setup-responses.md` and any future per-org state files under `<collection>/state/` are explicitly NOT overwritten or deleted by the upgrade flow. They contain answers the org provided during the original `install-collection` setup interview — org-specific data, not collection content. Re-running setup is disruptive and shouldn't be a side effect of upgrading.

### Changed

- All API manifests' `collection_version` bumped 2.2.1 → 2.3.0.
- `collection.json` `api[]` array gains the `upgrade-collection` entry with three trigger phrases: "upgrade collection", "update collection", "refresh collection".

### Migration notes

After this release lands, marketplace collections have a coherent admin upgrade verb. The existing `download-collection` / `download-and-install-collection` halt messages (which point at `@ai:upgrade-collection`) finally resolve to a real task. Pre-2.3.0 admins who hit those messages had to manually fetch + upload + bump org-config — same flow this task now automates.

## [2.2.1] — 2026-05-05

### Changed

- **`check-updates` "what to do" admin guidance updated.** Infrastructure and adapter upgrade rows now point admins at `@ai:publish-updates --check-upstream` (introduced in `agent-index-core` 3.5.0). The new flag handles upstream fetch + local sync + bootstrap regen + CHANGELOG entry in one step, replacing the pre-3.5.0 manual `git pull → @ai:edit-org → @ai:publish-updates` ritual. The pre-3.5.0 path remains as a fallback for admins who want to inspect the bundle before publishing.

## [2.2.0] — 2026-05-05

### Added

- **`check-updates` Step 2.6 — registered binary tools.** For each entry in `infrastructure-directory.json` → `binaries[]`, surfaces locally-installed version vs. org-pinned version vs. registry's `current_version`. Reports `available — not pinned`, `up to date`, `↑ install/update available`, `no published binary for <os>/<arch>`, or `↑ admin must update pin — below required floor`. Companion to core 3.4.0's binary-distribution architecture.

## [2.1.2] — 2026-05-01

### Fixed

- **Bug `20260430-8d20ea22` (capability-version-comparison mismatch):** `check-updates` Step 4 no longer compares per-capability member-index `version` against collection-level `current_version`. The pre-2.1.2 algorithm produced false "upgrade available" rows whenever a collection's `collection.json` `version` advanced faster than its individual `api/{name}.md` files (the normal pattern for collection-level changes — trigger arrays, README polish, dependency manifest tweaks). On a typical install this was producing dozens of false positives, including spurious MAJOR upgrade flags. Step 4 now `aifs_read`s each capability's `.md` file, parses the frontmatter `version`, and compares against the member-index entry's recorded `version` (which is the same thing — set at install/upgrade time by `org-setup`'s install flow).

### Changed

- check-updates task: 2.1.1 → 2.1.2
- Step 5 capability table column header renamed: "Collection Version" → "Latest Version" (the column now reflects the per-capability frontmatter version, not the collection-level version).
- Step 4 algorithm now classifies `PATH_NOT_FOUND` results into "capability removed from collection" vs "collection unavailable" so orphan member-index entries are surfaced explicitly instead of silently failing.
- Step 4 maintains a per-run cache keyed by `/{collection}/api/{name}.md` to avoid duplicate reads within a single workflow run.
- Edge Cases section extended with capability-removed, collection-missing, and missing-frontmatter cases.

### Notes

- This release does **not** modify `apply-updates` or `org-setup`. The `org-setup` "Needs Attention" criterion has the same conceptual error in its prose ("the collection version in the member index" — member-index doesn't store a collection version per capability) and is tracked separately in idea `core-improvements/org-setup-capability-version-comparison-mismatch`.
- The companion broadcast file `infrastructure-directory.json` is bumped to advertise marketplace 2.1.2.
- Three other follow-up ideas filed in `core-improvements` and `developer-collection` cover related work that emerged during this fix's investigation: `per-capability-manifest-vs-md-version-drift` (preflight check), `check-updates-show-available-to-install` (new Available section), and `org-setup-suggest-orphan-cleanup` (consuming the new "removed from collection" signal).

---

## [2.1.1] — 2026-05-01

### Added

- **`check-updates` task v2.1.1 — Step 2.5: Check Filesystem Adapter Version.** New step inserted between infrastructure (Step 2) and collections (Step 3). Reads `filesystem_adapter_directory_url` from `agent-index.json` (with fallback to the canonical raw URL), looks up the org's installed `backend_id` (from `remote_filesystem.backend`) in the directory, and compares `current_version` against the locally installed `mcp-servers/filesystem/adapter.json` `version`. Surfaces drift as a new "Filesystem Adapter" section in the report with the same status semantics as the infrastructure rows. Also surfaces a secondary NOTE on the row when `contract_version` advances (informational only — collection-aware contract gating is tracked separately).
- **`adapter_updates` field in lightweight-mode result.** Session-start consumers now receive an array of pending adapter updates with installed/latest versions and contract levels.
- **New CLI flag `--adapter-only`** to scope `check-updates` to just the filesystem adapter check.

### Fixed

- **Bug `20260501-8d20ea22` (adapter-drift portion):** check-updates is no longer blind to filesystem adapter releases. Concretely: an org running gdrive 2.1.3 against a directory advertising gdrive 2.2.0 now sees an explicit "↑ update available" row instead of silence. The capability-comparison portion of that bug remains tracked under idea `check-updates-capability-version-comparison`.

### Changed

- check-updates task: 2.1.0 → 2.1.1
- About section now describes four layers (added "Filesystem adapter") instead of three.
- Edge Cases extended with adapter-specific failure modes (missing `adapter.json`, custom backends not in the directory, `backend_id` mismatch between `agent-index.json` and `adapter.json`).

### Notes

- This release does **not** modify `apply-updates`; the publish/apply path already understands `adapter-bundle-update` operations. The only change is in the diagnostic path (`check-updates`).
- The companion broadcast file `infrastructure-directory.json` is bumped to advertise marketplace 2.1.1.
- Two related improvements remain queued as ideas in `core-improvements`: `bundle-vs-config-adapter-drift` (local config-vs-bundle drift on a single install) and `contract-version-aware-update-surfacing` (collection-aware contract-blocker surfacing). Neither is required for this fix.

---

## [2.1.0] — 2026-04-30

### Added

- **`check-updates` task v2.1.0:** Step 2 (Check Infrastructure Versions) rewritten to prefer `infrastructure_directory_url` from `agent-index.json` over the deprecated `core_version_url` and `marketplace_version_url`. The new directory file in `agent-index-resource-listings` is the single source of truth for both core and marketplace versions. Fixes the long-standing "core_version_url returns 404" issue caused by the agent-index-core repo being private (the new directory file lives in the public resource-listings repo). The deprecated URLs remain as fallbacks for pre-3.1.1 installs that haven't yet migrated.

### Changed

- check-updates task: 2.0.0 → 2.1.0

### Notes

- The `infrastructure_directory_url` field is added to `agent-index.json` automatically by agent-index-core 3.1.1's apply-updates Phase 1 step 4 (non-destructive field migration). Members upgrading from 3.1.0 or earlier will see the field appear after running `@ai:update`.
- Pre-3.1.1 installs running this version of `check-updates` will fall back to the legacy URLs. Since `core_version_url` is currently 404, those installs will see "unable to check" for core until they apply the 3.1.1 upgrade.

---

## [2.0.4] — 2026-04-19

### Added
- **Natural language trigger phrases in `collection.json`.** API entries now include trigger arrays that map conversational phrases to capabilities, powering the routing layer introduced in agent-index-core 3.0.5. Members can say things like "open marketplace" or "check for updates" instead of using `@ai:` alias syntax. Triggers are customizable per-member via `routing.json`.

---

## [2.0.3] — 2026-04-14

### Changed
- `agent_index_min_version` bumped to `3.0.0` — requires agent-index-core v3.0.0 with on-demand executor
- Documentation updated: "aifs_* MCP tools" references replaced with "aifs_* tools" throughout all API definitions and setup templates

---

## [2.0.2] — 2026-04-02

### Changed
- `check-updates` now checks for pending update instructions (`/shared/updates/latest.json`) and references `@ai:update` in its "What to do" recommendations. The diagnostic relationship between `check-updates` (marketplace) and `apply-updates` (agent-index-core) is documented in the task's About section.

---

## [2.0.1] — 2026-03-31

### Changed
- `list-org-collections` and `download-collection` now attempt automatic re-authentication on auth failures instead of prompting users to say `@ai:member-bootstrap`

---

## [2.0.0] — 2026-03-25

### Changed
- **Breaking:** Migrated from shared-mount-drive model to remote filesystem model. All org-level reads/writes now use `aifs_*` MCP tools instead of direct filesystem access.
- `download-collection` no longer supports git clone — collections are downloaded as ZIP and uploaded to remote filesystem via `aifs_write`
- All tasks that read `org-config.json` or collection directories now use `aifs_read`/`aifs_list` for remote filesystem access
- All "library root" references updated to "remote filesystem" across setup files and task definitions
- `@ai:fs-setup` references updated to `@ai:member-bootstrap`
- `agent_index_min_version` bumped to `2.0.0` — requires agent-index-core v2.0.0 with remote filesystem support
- Version bumped to 2.0.0 across all task definitions, setup files, and manifests

## [1.0.1] — 2026-03-18

### Changed
- Marketplace directory URL updated to dedicated repo: `agent-index/agent-index-resource-listings`
- Removed bundled `marketplace-directory.json` — network access required for first-time setup
- `create-org` now bootstraps `agent-index-marketplace` before opening the marketplace
- First-time setup and cache-miss now surface a whitelist instruction and hard-stop rather than falling back to a bundled file

## [1.0.0] — 2026-03-17

### Added
- Initial release
- `list-marketplace-collections` — browse marketplace with status indicators
- `list-org-collections` — view org's downloaded and installed collections
- `download-collection` — ZIP download with conflict detection
- `install-collection` — org-admin setup interview with alias collision resolution
- `download-and-install-collection` — convenience wrapper for both steps
- `refresh-marketplace-cache` — hybrid cache model with 24-hour TTL
- `marketplace-directory.json` — bundled directory with initial projects collection entry
- `collection-setup.md` — org admin setup for cache TTL configuration
- Setup templates and manifests for all six tasks
