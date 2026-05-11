# upgrade-collection — Setup template

This task has no member-defined parameters. It's a per-invocation operation, not a configurable capability.

The task accepts these per-invocation flags (passed when the admin runs it):

- `<collection_name>` (required, positional)
- `--to <version>` (optional)
- `--check-upstream` (optional)
- `--dry-run` (optional)

No setup interview runs at install time — this task is admin-callable directly.
