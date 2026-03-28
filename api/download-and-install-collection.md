---
name: download-and-install-collection
type: task
version: 2.0.0
collection: agent-index-marketplace
description: Convenience wrapper that downloads a marketplace collection and immediately runs its installation setup in a single flow.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks:
    - download-collection
    - install-collection
external_dependencies: []
reads_from: null
writes_to: null
---

## About This Task

The standard path for adding a new collection to an org. Combines `download-collection` and `install-collection` into a single continuous flow — the admin names a collection and Claude handles everything from conflict detection through setup completion without requiring a separate invocation between steps.

### Inputs

Collection name — provided in invocation or asked for.

### Outputs

Same as `download-collection` + `install-collection` combined:
- `/{collection-name}/` — collection directory on the remote filesystem root
- `/{collection-name}/setup/collection-setup-responses.md`
- `org-config.json` — updated with `status: installed`

---

## Workflow

### Step 1: Identify Collection

If the member named a collection in their invocation: use that name.
If not: ask "Which collection would you like to add? Say '@ai:list-marketplace-collections' to browse what's available."

Check `org-config.json`. If the collection is already `installed`: surface "'{display_name}' is already installed." Halt.

If already `downloaded`: surface "'{display_name}' has been downloaded but setup isn't complete. Picking up from the install step." Skip Step 2 and proceed directly to Step 3.

**On success with a new collection:** Proceed to Step 2.

---

### Step 2: Download

Run the full download flow from `run agent-index-marketplace task download-collection` for this collection.

If download fails: halt. Do not proceed to installation.

**On success:** Proceed to Step 3.

---

### Step 3: Install

Run the full installation flow from `run agent-index-marketplace task install-collection` for this collection.

The transition between download and install should feel seamless — no break in the conversation, no re-introduction. Simply continue:

> "'{display_name}' is downloaded. Now let's configure it for your org."

Then proceed with the setup interview.

---

## Directives

### Behavior

This task is a wrapper — its entire job is to sequence `download-collection` and `install-collection` cleanly. Keep the transition between the two invisible to the admin. The experience should feel like one continuous guided process.

If the admin invoked this but the collection was already downloaded, skip straight to installation without re-downloading. State clearly that you're picking up from the install step so the admin isn't confused.

### Constraints

Do not re-implement the logic of `download-collection` or `install-collection` within this task. Invoke them. All error handling, conflict detection, alias resolution, and setup interview logic lives in those tasks.

### Edge Cases

If the collection is already fully installed: do not re-download or re-install silently. Surface the status and offer relevant alternatives (upgrade, reconfigure).
