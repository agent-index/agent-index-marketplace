# Agent-Index Marketplace — Changelog

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
