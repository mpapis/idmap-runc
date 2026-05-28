# Changelog

## 1.1.0 — 2026-05-28

- Docker named volumes inherit idmap mapping from `/home` bind mounts in the same container. Fixes a crash mode in dev containers that mount both a project directory (idmapped) and a named volume (e.g. `node_modules`, formerly not idmapped) — the asymmetric UID view between the two paths broke tools like pnpm/npm with SIGKILL mid-install.
- Inheritance is skipped when `/home` mounts have conflicting owner UID/GID, and when no `/home` mount is present (containers using only named volumes are untouched).
- New integration test (test 9) exercising the volume-inheritance path.

## 1.0.0 — 2025-03-24

Initial release.

- OCI runtime wrapper injecting kernel-level ID-mapped mounts on bind mounts from `/home`
- Bidirectional UID/GID swap (host user ↔ container root)
- Per-container opt-out via `IDMAP_SKIP=true`
- Non-root containers (`--user`) pass through unmodified
- Automated installer with safety checks and rollback support
- Integration test suite (8 tests)
