# idmap-runc

Shell script project — OCI runtime wrapper that fixes Docker bind mount file ownership via kernel ID-mapped mounts. See README.md for full documentation.

## Scripts

- `idmap-runc` — the runtime wrapper (intercepts runc "create", injects uidMappings/gidMappings into OCI config.json)
- `install` — installer with preflight checks, daemon.json setup, container recreation (`sudo ./install`)
- `test` — integration tests requiring Docker with idmap-runc registered (`./test`)

## Development

- Lint: `shellcheck idmap-runc install test`
- Test: `sudo ./install && ./test`
- No build step — plain bash scripts

## Key design decisions

- Wrapper pattern (like NVIDIA runtime): intercept → modify spec → exec real runc
- Only maps bind mounts from `/home` (configurable via `IDMAP_PREFIXES`)
- Bidirectional UID swap (host ↔ container root), not a range shift
- All spec manipulation via `jq` with `--argjson` (safe interpolation, no injection)
