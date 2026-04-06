# Agent-Index Marketplace — Roadmap

Current version: 2.0.2
Last updated: 2026-04-05

---

## Current State

v2.0 is the marketplace discovery and collection management collection. Org admins use it to browse, download, and install marketplace collections. The collection operates on a hybrid model: the marketplace directory is fetched from a remote GitHub-hosted JSON file and cached locally with a 24-hour TTL. Collections are downloaded as ZIP files and uploaded to the org's remote filesystem via MCP.

v2.0 runs on the remote filesystem model (MCP-based), consistent with agent-index-core v2.0+. The marketplace directory is authoritative and external; the local cache is a performance optimization only.

### Known Limitations

- **Marketplace directory is GitHub-only.** The directory is fetched from a hardcoded GitHub URL. There is no configuration option to use an alternative directory source or to run a private/internal marketplace. Large enterprises may need to fork and host their own, but there is no built-in support for private instances.

- **Network access is required for first-time setup.** If the GitHub marketplace directory URL is not reachable (e.g., blocked by network policy), the marketplace cannot be initialized and no fallback exists. The user must update their network settings or use an alternative approach. For air-gapped deployments, a bundled directory snapshot or manual registration is not yet supported.

- **Cache TTL is organization-wide, not per-user.** The 24-hour cache is stored at `/shared/marketplace-cache/` and is shared across all members. If one member manually refreshes the cache, all members see the refreshed data. There is no per-member cache override or "always fetch latest" mode for users who want the most recent listings.

- **Download conflict detection is filename-based.** If two collections have the same directory name (e.g., both named `projects`), `download-collection` will detect a conflict and ask the admin to rename locally. There is no namespace isolation or versioning strategy to prevent name collisions.

- **No collection lifecycle tracking in the org.** The marketplace knows what is available and what is installed, but there is no record of when a collection was installed, who installed it, or what changes were made to it since installation (beyond the contents of the collection's own CHANGELOG.md). Org-level audit trails for collection management are not available.

- **Update availability is read-only.** `check-updates` surfaces what updates are available but provides no scheduling or automatic installation. The action steps to actually publish and apply updates live in agent-index-core, not in the marketplace. A user seeing "update available" must switch context to core tasks to proceed.

### Known Bugs

None currently tracked.

---

## Wishlist

### v2.1 — Quality of Life

- **Per-collection download progress tracking.** Currently, `download-collection` blocks until the ZIP is fully uploaded to remote. For large collections, this can take minutes with no feedback. Add progress indicators or enable partial upload and resume.
- **Local collection cache verification.** When listing org collections, verify that all entries in the local manifest have corresponding files on the remote filesystem. Surface warnings if files are missing (e.g., due to manual remote deletions or sync failures).
- **Collection compatibility matrix.** Show in `list-marketplace-collections` which collections are compatible with the current agent-index-core version and which have known dependency issues (e.g., require a specific MCP connector or org role setup).

### v2.2 — Deeper Integration

- **Private marketplace URL configuration.** Allow orgs to configure an alternative marketplace directory URL (internal GitHub, self-hosted, or custom HTTP endpoint) via org-setup. Support both public and private registries.
- **Collection dependency tracking.** When an admin installs a collection that declares dependencies on other collections or capabilities, automatically suggest (or offer to auto-install) those dependencies. Track the dependency graph in org-config.
- **Scheduled collection updates.** New task `schedule-collection-updates` that allows admins to set up automatic update checks on a schedule (e.g., daily or weekly). When a new collection version is detected, publish it for members or automatically push it (with admin config).

### v2.3 — Org Governance

- **Collection approval workflow.** Collections downloaded but not yet installed can be marked as "approved for installation." New members are prompted to install approved collections during onboarding. Admins can manage the approved list without changing the org-config directly.
- **Collection version pinning.** Allow orgs to "pin" a collection to a specific version, preventing auto-upgrades and requiring explicit admin approval to move to a newer version. Useful for stability-critical deployments.
- **Marketplace directory auditing.** Track all marketplace directory refreshes (when, by whom, what changed) in a local audit log. Helps admins understand when new collections became available and when upstream breaking changes were detected.

### v3.0 — Structural Changes (breaking)

- **Multi-registry support.** Support multiple marketplace directories (primary + fallback, or internal + public). Resolve collection names with priority rules (e.g., internal registry first, then public). Removes the single-source-of-truth constraint and enables hybrid public/private collections.
- **Collection marketplace governance.** Move from a flat list to a tagged/categorized marketplace (by domain, skill type, author, stability tier). Add filtering, search, and recommendation logic. Support featured collections, collections by trusted authors, and user ratings.

---

## Design Notes

- **The marketplace is a read-only discovery layer.** It surfaces what is available in the upstream directory but does not modify or curate. Curation (what gets installed, what gets recommended, what gets pinned) is the org admin's responsibility via `download-and-install-collection`, `list-org-collections`, and decisions about what to run `publish-updates` on.

- **Network fetch is synchronous, not backgrounded.** `refresh-marketplace-cache` is a synchronous task that blocks until the fetch completes. This is intentional for simplicity, but means large directories or slow networks can cause noticeable delays. Async caching is a future optimization, not a current priority.

- **No local marketplace modifications are persisted back to the upstream repo.** Orgs cannot contribute improvements (e.g., better descriptions, bug reports, version updates) directly from within their Cowork environment. All communication with collection authors is manual and external.

- **The marketplace directory is a static JSON registry, not a package manager.** It is not responsible for versioning, dependency resolution, or compatibility checking. Those are the collection author's responsibility (via their setup templates and manifests) and the org admin's responsibility (via careful testing before publishing updates).

- **Collections are immutable once installed.** There is no "edit installed collection" workflow. If an admin needs to modify a collection, they must download a fresh copy, modify it locally, and potentially re-register it. Modifications to shared data within an installed collection are outside the scope of the marketplace — those are governed by individual collection workflows.
