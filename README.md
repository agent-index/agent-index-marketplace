# Agent-Index Marketplace

The marketplace collection for agent-index. Provides org admins with the tools to discover, download, install, and manage collections from the agent-index marketplace.

---

## What's Included

- **list-marketplace-collections** — Browse all available marketplace collections with status indicators (available, downloaded, installed, update available)
- **list-org-collections** — See what your org has downloaded and installed, including org-authored collections
- **download-collection** — Download a marketplace collection to your org's remote filesystem (ZIP download, uploaded via `aifs_*` tools)
- **install-collection** — Run the org-admin setup interview for a downloaded collection
- **download-and-install-collection** — Download and install in a single flow (recommended for most installs)
- **refresh-marketplace-cache** — Fetch the latest marketplace directory from GitHub
- **check-updates** — Comprehensive update check across infrastructure, installed collections, and member capabilities

**Note:** The update *action* tasks (`publish-updates` and `apply-updates`) live in `agent-index-core`, not in this marketplace collection. The marketplace provides `check-updates` as a diagnostic — it shows what is out of date. To actually distribute and apply updates, admins use `@ai:publish-updates` and members use `@ai:update` (both from agent-index-core).

---

## How It Works

The marketplace directory is hosted in a dedicated GitHub repo at:
```
https://raw.githubusercontent.com/agent-index/agent-index-resource-listings/refs/heads/main/marketplace-directory.json
```

A local cache is kept at `/shared/marketplace-cache/` with a 24-hour TTL (configurable). Any task that reads the cache checks whether it's stale and refreshes automatically if needed. Network access to the marketplace directory URL is required for first-time setup — if the URL is blocked, Cowork's network settings must be updated to whitelist it before the marketplace can be used.

Collections are downloaded as a ZIP and uploaded to the org's remote filesystem via `aifs_write`. The remote filesystem is accessed through `aifs_*` tools running in exec mode — all org-level data (collection directories, org-config, marketplace cache) lives on the remote filesystem while member data stays local.

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

**Getting the latest marketplace listings:**
```
@ai:refresh-marketplace-cache
```

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
