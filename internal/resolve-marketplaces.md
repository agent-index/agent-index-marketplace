# Internal subroutine: resolve-marketplaces

**Added in:** agent-index-marketplace 2.20.0 (requires agent-index-core 3.30.0). **Collision rules revised in 2.21.0** (core 3.31.0): unique names across catalogs; namespaces are optional reservations.
**Normative model:** `agent-index-core/standards.md` § "Marketplaces: catalogs, subscriptions, provenance"
**Used by:** `list-marketplace-collections`, `download-collection`, `check-updates`, `upgrade-collection`. (`list-org-collections` groups by provenance from `org-config.json` alone and reads no catalog.)

This is the **only** place a marketplace task obtains catalog entries. Callers follow these steps and consume the result; they do not read a `marketplace-directory.json` themselves, and they **never** read `/shared/marketplace-cache/` (decommissioned — no writer since marketplace 2.17.0 `mktcatalogwebfetch`).

Not an API member. Referenced from task files as: *"Follow `/internal/resolve-marketplaces.md`."*

---

## Who can run it

Catalogs are **admin-only** — they live in the admin's local clones under the install root, and members never need them (members read what the admin published to `/shared/dist/`).

If the running member's `member_hash` is not in `org-config.json` → `admins[]`: do not attempt any catalog read. Return `{ "status": "not_admin" }`. The caller surfaces: *"The marketplace catalog is available to org admins. To see what your org has installed, say 'list our collections'."*

---

## Step 1: Load subscriptions

Read `org-config.json` via its id anchor (`aifs_read("id:{org_config_id}")`, id from local `agent-index.json` → `remote_filesystem.connection.org_config_id`).

- If `marketplaces[]` is present: use it.
- If absent (pre-3.30.0 org not yet back-filled by `publish-updates` 6g): **synthesise** the single legacy subscription exactly as `standards.md` § "Legacy orgs" defines it — `id: "agent-index-public"`, `enabled: true`, `namespace: null`, `skip_if_unavailable: false`, source `clone` at `agent-index-resource-listings` if `<install_root>/agent-index-resource-listings/marketplace-directory.json` exists, else `url` from `agent-index.json` → `marketplace_directory_url`. Mark the result `synthesised: true`. Do not write it — that is `publish-updates` 6g's job.

Partition into **enabled** and **disabled**. Disabled subscriptions are never read, but their ids and display names are returned so callers can label collections whose origin is disabled.

---

## Step 2: Read each enabled catalog

`<install_root>` is the directory containing `agent-index.json`.

**`clone`:**

1. Validate `source.ref`: it must be a **relative** path with no `..` segment and no leading `/`, drive letter, or `/sessions/` prefix. An absolute ref is a stored-path defect (`appspathsandboxleak` class) — treat the source as unavailable with error `absolute_ref` and tell the admin to fix it in `@ai:edit-org` → Manage marketplaces.
2. Resolve `<install_root>/<ref>/marketplace-directory.json` and read it with the native Read tool.
3. **Verify the trust anchor without running git:** read `<install_root>/<ref>/.git/config` as a file and extract `[remote "origin"] url`. Normalise both it and `trust_anchor.git_url` (strip a trailing `.git` and trailing `/`, lowercase the host). Mismatch → unavailable, error `trust_anchor_mismatch`, naming both URLs. **Never run `git` from the sandbox for this** — agent-side git over the mount is limited to read-only `log`/`show`/`status`/`diff` (CONTRIBUTING), and a file read needs none of it.
4. If the clone directory or file is missing → unavailable, error `source_missing`. The remedy is the committed `lib/clone/clone-repos` script run natively by the admin; say so. **Never author a clone script.**

**`url`:** permitted only for the synthesised legacy subscription of a not-yet-migrated org. Fetch via the SHA-pinned Distribution fetch protocol (`standards.md`), emit the standards.md deprecation warning. **Never `WebSearch`** for a catalog, a version, or a release (`adminupstreamstale`).

**`backend`:** reserved, not implemented in v1 → unavailable, error `source_kind_unsupported`.

Parse the JSON. Invalid JSON → unavailable, error `unparseable`.

---

## Step 3: Verify catalog identity

For each catalog read in Step 2:

- **`marketplace_id`.** If the catalog declares one, it must equal the subscription `id`. If the catalog declares none, it is accepted **only** when the subscription `id` is `agent-index-public` (legacy public catalog, read as `marketplace_id: "agent-index-public"`, `namespace: null`). Otherwise → unavailable, error `identity_mismatch` / `identity_missing`.
- **`namespace`.** Catalog `namespace` must equal the subscription's `namespace` (the value recorded at subscribe). A catalog that changed its namespace after subscription → unavailable, error `namespace_changed`; the admin re-subscribes deliberately.
- (Removed in 2.21.0: the 3.30.0 rule that every non-public catalog must declare a namespace. `namespace: null` is valid for any catalog.)

---

## Step 4: Check reservations and unique names across all catalogs (revised in 2.21.0)

Let `R` = the non-null namespaces across all **readable** catalogs from Step 3.

1. **Overlap (catalog-level).** For any two `a`, `b` in `R`: if `a + "-"` is a prefix of `b + "-"` or vice versa → **both** catalogs unavailable, error `namespace_overlap`. This is a configuration error in the catalogs themselves, so it is not scoped down to entries.
2. **Intrusion (entry-level).** An entry in catalog X whose `name` starts with `n + "-"`, where `n` is reserved by a *different* catalog Y, is **excluded** from X's entries and recorded in `conflicts[]` as `{ "kind": "namespace_intrusion", "name", "catalog": X, "reserved_by": Y }`. Y keeps the prefix; the rest of X stays usable.
3. **Duplicate names (entry-level).** After step 2, any `name` offered by more than one catalog is a conflict: keep **every** copy in `entries` but mark each with `"conflict": ["<other catalog ids>"]`, and record `{ "kind": "duplicate_name", "name", "catalogs": [...] }` in `conflicts[]`. No copy is preferred over another.

(2.20.0's "own entries must start with the catalog's namespace" check is removed. A catalog's own entries may use any names.)

Conflicts never make a catalog unavailable and never trigger the Step 5 failure policy — they are always reported alongside an otherwise-usable result.

---

## Step 5: Apply failure policy

For each unavailable catalog:

- `skip_if_unavailable: false` (the default) → **abort the whole resolve**. Return `{ "status": "error", "errors": [...] }` naming every failing catalog, its error code, and the remedy. The caller surfaces this and stops; it **must not** present a partial catalog as if complete.
- `skip_if_unavailable: true` → drop that catalog and record a `skipped` notice. The caller shows the notice at the top of its output.

---

## Step 6: Return

```json
{
  "status": "ok",
  "synthesised": false,
  "sources": [
    { "id": "agent-index-public", "display_name": "Agent Index Marketplace", "namespace": null,
      "state": "ok", "directory_version": "1.26.0", "last_updated": "2026-09-23", "entry_count": 10 },
    { "id": "…", "display_name": "…", "namespace": "cx", "state": "skipped", "error": "source_missing" }
  ],
  "disabled": [ { "id": "…", "display_name": "…" } ],
  "conflicts": [ { "kind": "duplicate_name", "name": "…", "catalogs": ["…", "…"] } ],
  "entries": [
    { "marketplace_id": "agent-index-public", "marketplace_display_name": "Agent Index Marketplace",
      "entry": { "name": "projects", "current_version": "4.3.0", "…": "…" } }
  ]
}
```

`entries` is the merged list across every `ok` catalog, each entry tagged with the catalog it came from; an entry involved in a duplicate-name conflict also carries `conflict: [other catalog ids]`. Intruding entries are not in `entries` — only in `conflicts[]`. `display_name` is the subscription's local label (falls back to the catalog's).

---

## Lookup by name (for callers resolving one collection)

Two lookups, used for different purposes:

- **New install** (`download-collection`): match `name` across all `entries`. Exactly one non-conflicted match → use it. A match marked `conflict` → **refuse**: "'{name}' is offered by more than one catalog you subscribe to ({display names}). Resolve it in one of the catalogs, or disable one subscription, before installing." **There is no priority or precedence rule** — never pick one (`standards.md` § "Collisions"). A name found only in `conflicts[]` as an intrusion → refuse, naming the reserving catalog.
- **Origin lookup** (`check-updates`, `upgrade-collection`): match `name` **only among entries whose `marketplace_id` equals the installed collection's origin**. A conflict marker on that entry does not block it — the origin is already fixed by provenance, so another catalog's same-named entry cannot redirect an update — but callers surface it as a warning.

No match → the caller reports "not in any subscribed catalog" and, if the name is in `org-config.json` `installed_collections[]` with `marketplace_id: null`, labels it sideloaded.

---

## Constraints

- Admin-only. Read-only. Writes nothing, anywhere.
- Never reads `/shared/marketplace-cache/`. Never `WebSearch`. Never runs git. Never authors a clone script.
- Never reads a disabled subscription's catalog.
- Never returns a partial catalog without an explicit `skipped` notice the caller must display.

<!-- AIFS:FILE-END -->
