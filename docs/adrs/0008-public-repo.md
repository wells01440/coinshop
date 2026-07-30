# 0008. Public repository; data and secrets never in git

Date: 2026-07-29 · Status: accepted

Context: four AI surfaces need read access; per-platform private-repo
auth is three plumbing jobs and Grok has none; the code is a workflow
engine with nothing sensitive in it.

Decision: wells01440/coinshop is public. Inventory data (SQLite),
coin photos, env files, and keys are gitignored and live only on the
tailnet. gitleaks guards commits.

Consequences: zero-friction multi-surface access; trust signal;
discipline required on every commit. (R31)
