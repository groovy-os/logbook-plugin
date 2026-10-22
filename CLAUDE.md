# logbook-plugin: working rules

## CI cost policy (organisation-wide, 2026-10-22)

- Target **10 total runner-minutes per ordinary PR run**, summed across jobs, including setup. This is a measured target, not a ten-minute wall-clock timeout or permission to skip required checks. Explain broader schema, security, infrastructure or simulation checks in the PR.
- Check changed areas locally; push each completed fix round once. Cancel superseded PR runs. Rerun only failed jobs for a confirmed flake or infrastructure failure; do not dispatch extra full suites without an explicit request.
- Put cheap lint/type checks before expensive suites. Prefer affected tests plus small smoke coverage where selection is proven; first remove shared-state dependencies. Keep migration, permission and tenant-isolation checks before merge. Never silently narrow an untraceable dependency.
- Keep exhaustive regression coverage on the scheduled cadence, skipping only unchanged inputs after a successful scheduled run. Unknown or failed evidence must run. Do not repeat the entire PR suite on every merge; keep quick post-merge checks and production release checks.
- Some cross-feature regressions may be caught on the scheduled run; failed required merge checks still block merging. Measure savings after rollout, including failed and cancelled runs; do not present a one-day extrapolation as an invoice.

This repository currently has no GitHub Actions workflows. Do not add scheduled or paid CI merely to conform to this policy; apply it when CI is introduced.
