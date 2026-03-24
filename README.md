# idmap-runc

OCI runtime wrapper that fixes Docker bind mount file ownership on Linux.

Files created by containers on bind-mounted host directories are owned by **your user** instead of root — no Dockerfile changes, no compose overrides, no runtime performance penalty.

> **Warning:** Untested with gosu, su-exec, or other in-container privilege-dropping tools. The bidirectional UID swap exchanges UID 0 and the host UID — software that drops from root to a specific UID may produce unexpected file ownership. Use `IDMAP_SKIP=true` to opt out per container.

## Problem

Docker on Linux runs container processes as root (UID 0). Files created inside containers on bind-mounted host directories are owned by root on the host:

```
$ docker run --rm -v ~/projects/myapp:/app alpine touch /app/newfile
$ ls -la ~/projects/myapp/newfile
-rw-r--r-- 1 root root 0 ...    # Can't edit without sudo chown
```

macOS Docker Desktop doesn't have this problem (VirtioFS translates ownership transparently).

## Solution

A thin shell script sits between Docker and runc. It intercepts container creation, injects kernel-level UID mapping on bind mounts from your project directories, then execs the real runc. The host filesystem is unaffected.

```
Docker daemon → containerd → containerd-shim
  └→ idmap-runc (intercepts "create")
       ├─ Reads OCI config.json
       ├─ Injects idmap + UID/GID swap on matching bind mounts
       └─ exec /usr/bin/runc (real runc with modified spec)
```

## How It Works

### Bidirectional UID Swap

The wrapper injects two mapping entries per mount — a **swap** between container root and your host user. UID/GID are auto-detected from the mount source owner via `stat()`, so no hardcoding is needed. Example for a host user with UID/GID 1000:

```json
{
  "options": ["rbind", "rprivate", "rw", "idmap"],
  "uidMappings": [
    { "containerID": 0, "hostID": 1000, "size": 1 },
    { "containerID": 1000, "hostID": 0, "size": 1 }
  ],
  "gidMappings": [
    { "containerID": 0, "hostID": 1000, "size": 1 },
    { "containerID": 1000, "hostID": 0, "size": 1 }
  ]
}
```

**Why both directions are required:** A one-directional mapping (0→1000 only) causes `EOVERFLOW` ("Value too large for data type") — the kernel can't resolve existing UID-1000 files on disk without a reverse entry. The swap is the minimal mapping that both resolves existing files and remaps new file ownership.

| Scenario | Container sees | Host sees |
|----------|---------------|-----------|
| Container creates file as root | `root:root` | `youruser:youruser` |
| Host user creates/edits file | `root:root` | `youruser:youruser` |
| Container reads host file | `root:root` | unchanged |
| Container appends to host file | `root:root` | `youruser:youruser` (ownership preserved) |

> **Note:** When a mount entry has `uidMappings`/`gidMappings`, runc creates a temporary throwaway user namespace to apply `mount_setattr(MOUNT_ATTR_IDMAP)`, then passes the mapped mount fd to the container. The container itself does **not** need to be in a user namespace.

## Requirements

| Requirement | Minimum | Notes |
|-------------|---------|-------|
| Linux kernel | 5.12+ | `mount_setattr(MOUNT_ATTR_IDMAP)` — zero-overhead UID translation at the VFS layer |
| runc | 1.2.0+ | OCI spec v1.2 mount-level mappings |
| Filesystem | ext4, xfs, btrfs, tmpfs, overlayfs (5.19+), FUSE (6.12+) | Must support idmap; **NFS does not** |
| jq | Any version | JSON manipulation |
| Docker | Any with custom runtime support | `daemon.json` runtimes config |

### Verified On

- openSUSE Leap 16.0, kernel 6.12.0, btrfs
- runc 1.3.4, Docker 28.5.1-ce
- SELinux enforcing (`/home/*/projects` labeled `container_file_t`)

## Installation

Run the install script (requires sudo):

```bash
sudo ./install
```

This will:
1. Copy `idmap-runc` to `/usr/local/bin/`
2. Create the log file at `/var/log/idmap-runc.log`
3. Install logrotate config
4. Register `idmap-runc` as the default Docker runtime in `/etc/docker/daemon.json`
5. Restart Docker and recreate running containers

Options:

| Flag | Description |
|------|-------------|
| `--no-default` | Register runtime but don't make it the default |
| `--no-recreate` | Show running containers instead of recreating them |
| `--uninstall` | Remove idmap-runc binary, clean daemon.json, restart Docker |

To override the detected runc path: `RUNC_PATH=/path/to/runc sudo ./install`

To install **without** making it the default runtime:

```bash
sudo ./install --no-default
```

Then opt in per-service via a compose override or `docker run --runtime=idmap-runc`:

```yaml
# docker-compose.override.yml
services:
  app:
    runtime: idmap-runc
```

### Uninstall

```bash
sudo ./install --uninstall
```

This removes the binary, cleans the runtime entry from `daemon.json`, removes the logrotate config, and restarts Docker. The log file is preserved.

### Manual Installation

<details>
<summary>Step-by-step instructions</summary>

#### 1. Install the wrapper

```bash
sudo cp idmap-runc /usr/local/bin/idmap-runc
sudo chmod +x /usr/local/bin/idmap-runc
sudo touch /var/log/idmap-runc.log
sudo chmod 640 /var/log/idmap-runc.log
sudo chgrp docker /var/log/idmap-runc.log
```

#### 2. Register as Docker runtime

Add to `/etc/docker/daemon.json`:

```json
{
  "default-runtime": "idmap-runc",
  "runtimes": {
    "idmap-runc": {
      "path": "/usr/local/bin/idmap-runc"
    }
  }
}
```

To use without making it default, omit `"default-runtime"` and opt in per-service via `runtime: idmap-runc` in a compose override or `docker run --runtime=idmap-runc`.

```bash
sudo systemctl restart docker
```

</details>

### Verify

```bash
# Container creates file — should be owned by your user on host
docker run --rm -v ~/projects/test:/app alpine sh -c \
  'touch /app/test-file && ls -la /app/test-file'
ls -la ~/projects/test/test-file
# Expected: youruser:youruser

# Host creates file — container sees it as root
touch ~/projects/test/host-file
docker run --rm -v ~/projects/test:/app alpine \
  ls -la /app/host-file
# Expected: root:root inside container
```

## Configuration

These variables are hardcoded at the top of the `idmap-runc` script. Edit the source before running `./install`, or `/usr/local/bin/idmap-runc` after installation:

| Variable | Default | Description |
|----------|---------|-------------|
| `REAL_RUNC` | `/usr/bin/runc` | Path to the real runc binary |
| `LOG_FILE` | `/var/log/idmap-runc.log` | Log file path |
| `IDMAP_PREFIXES` | `("/home")` | Array of source path prefixes to apply idmap on |

UID/GID are auto-detected from the mount source owner via `stat()` — no configuration needed.

## Safety Checks

1. **Only root containers** — skips idmap when `process.user.uid != 0` (non-root containers already create files as the correct user)
2. **Only matching bind mounts** — named volumes and mounts from outside configured paths are never touched
3. **Opt-out per container** — set `IDMAP_SKIP=true` (also accepts `TRUE`, `True`, `1`):
   ```yaml
   environment:
     - IDMAP_SKIP=true
   ```
4. **Graceful fallback** — if jq fails, the original config.json is preserved and runc runs normally
5. **Root-owned mounts skipped** — bind mounts where UID or GID is 0 are not remapped

> **Note:** Only UID 0 and the host user's UID are mapped. Processes running as other UIDs (e.g., `nobody`) will get `EOVERFLOW` on idmapped mounts. This affects containers that use intermediate UIDs via `USER` or `gosu`/`su-exec`.

## Linting

```bash
shellcheck idmap-runc install test
```

## Testing

Run the integration test suite (requires `idmap-runc` registered as a Docker runtime):

```bash
./test
```

To test with an explicit runtime flag:

```bash
./test --runtime=idmap-runc
```

The test suite covers:

| # | Test | Verifies |
|---|------|----------|
| 1 | Container-created file ownership | File created by container root is owned by host user on host |
| 2 | Container sees own file as root | Bidirectional mapping works — container sees `0:0` |
| 3 | Host file seen as root in container | Existing host files map to root inside container |
| 4 | Container appends to host file | Ownership preserved after container writes to host file |
| 5a | Named volume unaffected | Named volumes (non-bind mounts) are not remapped |
| 5b | Non-home bind mount unaffected | Bind mounts outside `IDMAP_PREFIXES` are not remapped |
| 6 | IDMAP_SKIP opt-out | `IDMAP_SKIP=true` env var disables injection |
| 7 | Non-root container bypass | `--user` containers skip idmap (not needed) |
| 8 | Subdirectory creation | Directories and nested files get correct ownership |

## Debugging

```bash
# Watch wrapper activity
tail -f /var/log/idmap-runc.log

# Capture full OCI config for inspection (add to wrapper before exec):
# cp "$BUNDLE/config.json" /tmp/last-idmap-config.json

# Check SELinux denials
sudo ausearch -m avc -ts recent | grep mount
```

| Symptom | Cause | Fix |
|---------|-------|-----|
| Files still root-owned | Mount source didn't match `IDMAP_PREFIXES` | Check log, verify path prefix |
| Container fails to start | runc < 1.2.0 or kernel < 5.12 | Upgrade runc/kernel |
| `EPERM` | SELinux blocking mount_setattr | Check audit log |
| "no such runtime" | daemon.json not reloaded | `sudo systemctl restart docker` |

## Comparison With Alternatives

| Aspect | idmap-runc (default) | idmap-runc (opt-in) | Docker userns-remap | user: + fixuid | bindfs |
|--------|---------------------|--------------------|--------------------|----------------|--------|
| Performance | Zero runtime overhead | Zero runtime overhead | Chown on pull; 2x storage | Zero overhead | FUSE (~4x metadata) |
| Compose changes | None | Override file | None (daemon-wide) | Override file + Dockerfile | Override file |
| Existing images | No impact | No impact | Must re-pull all | No impact | No impact |
| Granularity | Per-mount | Per-service + per-mount | Daemon-wide | Per-container | Per-project |
| Privileged containers | Works | Works | Auto-bypassed | Works | Works |

> **Note:** The idmap-runc wrapper adds ~50-100ms startup overhead per container creation for JSON processing. This is negligible for long-running containers but measurable in rapid-cycling scenarios (CI, batch jobs).

## Future

Docker issue [docker/roadmap#398](https://github.com/docker/roadmap/issues/398) requests native bind mount UID/GID control. When Docker adds `--mount type=bind,idmap=...`, this wrapper becomes unnecessary.

## References

- [OCI Runtime Spec v1.2 mount mappings](https://github.com/opencontainers/runtime-spec/blob/main/config.md)
- [runc idmapped mounts](https://github.com/opencontainers/runc/issues/2821)
- [Linux kernel idmappings](https://docs.kernel.org/filesystems/idmappings.html)
- [mount_setattr(2)](https://man7.org/linux/man-pages/man2/mount_setattr.2.html)
- [NVIDIA Container Runtime](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/arch-overview.html) (same wrapper pattern)
- [Docker roadmap #398](https://github.com/docker/roadmap/issues/398)
