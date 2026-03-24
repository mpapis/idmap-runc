# Changelog

## 1.0.0 — 2025-03-24

Initial release.

- OCI runtime wrapper injecting kernel-level ID-mapped mounts on bind mounts from `/home`
- Bidirectional UID/GID swap (host user ↔ container root)
- Per-container opt-out via `IDMAP_SKIP=true`
- Non-root containers (`--user`) pass through unmodified
- Automated installer with safety checks and rollback support
- Integration test suite (8 tests)
