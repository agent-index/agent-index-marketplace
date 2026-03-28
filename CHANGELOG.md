# Agent-Index Marketplace — Changelog

## [2.0.0] — 2026-03-25

### Changed
- **Breaking:** Migrated from shared-mount-drive model to remote filesystem model. All org-level reads/writes now use `aifs_*` MCP tools instead of direct filesystem access.
- `download-collection` no longer supports git clone — collections are downloaded as ZIP and uploaded to remote filesystem via `aifs_write`
- All tasks that read `org-config.json` or collection directories now use `aifs_read`/`aifs_list` for remote filesystem access
- All "library root" references updated to "remote filesystem" across setup files and task definitions
- `@ai:fs-setup` references updated to `@ai:member-bootstrap`
- `agent_index_min_version` bumped to `2.0.0` — requires agent-index-core v2.0.0 with MCP server support
- Version bumped to 2.0.0 across all task definitions, setup files, and manifests

## [1.0.1] — 2026-03-18

### Changed
- Marketplace directory URL updated to dedicated repo: `agent-index/marketplace`
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
