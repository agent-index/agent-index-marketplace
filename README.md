# Agent-Index Marketplace

The marketplace collection for agent-index. Provides org admins with the tools to discover, download, install, and manage collections from the agent-index marketplace.

---

## What's Included

- **list-marketplace-collections** — Browse all available marketplace collections with status indicators (available, downloaded, installed, update available)
- **list-org-collections** — See what your org has downloaded and installed, including org-authored collections
- **download-collection** — Download a marketplace collection to your org's remote filesystem (ZIP download, uploaded via `aifs_*` tools)
- **install-collection** — Run the org-admin setup interview for a downloaded collection
- **download-and-install-collection** — Download and install in a single flow (recommended for most installs)
- **refresh-marketplace-cache** — *Deprecated (2.20.0).* Legacy web-fetched cache for a not-yet-migrated org only
- **check-updates** — Comprehensive update check across infrastructure, installed collections, and member capabilities

**Note:** The update *action* tasks (`publish-updates` and `apply-updates`) live in `agent-index-core`, not in this marketplace collection. The marketplace provides `check-updates` as a diagnostic — it shows what is out of date. To actually distribute and apply updates, admins use `@ai:publish-updates` and members use `@ai:update` (both from agent-index-core).

---

## How It Works

An org can subscribe to **more than one marketplace catalog** (2.20.0, with core 3.30.0). Each catalog is a repo containing a `marketplace-directory.json` that declares its own `marketplace_id`, `display_name` and reserved `namespace`. The org's subscriptions live in `org-config.json` → `marketplaces[]` and are managed with `@ai:edit-org` → Manage marketplaces. Every new org subscribes to the public Agent Index catalog (`agent-index-resource-listings`).

Catalogs are **admin-only** and are read from the admin's local clones under the install root — never fetched from the web, and there is no cache or TTL. Every marketplace task reads them through one internal resolver (`internal/resolve-marketplaces.md`), which checks catalog identity and namespaces and fails loudly if a subscribed catalog can't be read. Each installed collection records the catalog it came from (`installed_collections[].marketplace_id`), so update checks compare it against its own catalog. Full model: `agent-index-core/standards.md` § "Marketplaces".

`/shared/marketplace-cache/` is decommissioned; nothing reads it.

Collections are sourced from the admin's tag-pinned local clone (Release C; a ZIP download survives only as a deprecated fallback) and uploaded to the org's remote filesystem via `aifs_write_batch`. The remote filesystem is accessed through `aifs_*` tools running in exec mode — all org-level data (collection directories, org-config) lives on the remote filesystem while member data stays local.

---

## Typical Workflow

**Adding a new collection:**
```
@ai:download-and-install-collection projects
```

**Browsing what's available:**
```
@ai:marketplace
```
or
```
@ai:list-marketplace-collections
```

**Seeing what your org has installed:**
```
@ai:list-org-collections
```

**Getting the latest marketplace listings:** refresh your clones with the committed `agent-index-core/lib/clone/clone-repos` script (catalogs are read from local clones).

**Adding another marketplace catalog:**
```
@ai:edit-org
```
then choose "Manage marketplace subscriptions".

**Checking for updates across the system (diagnostic):**
```
@ai:check-updates
```

**Applying published updates (from agent-index-core):**
```
@ai:update
```

---

## For Collection Authors

To get your collection listed in the marketplace, see the submission process in `agent-index-core/standards.md`.

---

## Version History

See CHANGELOG.md.
