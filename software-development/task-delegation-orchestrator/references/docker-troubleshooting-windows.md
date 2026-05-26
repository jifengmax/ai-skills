# Docker Desktop Troubleshooting (Windows)

Common issues when starting Dify via Docker Compose on Windows.

## Issue: "Docker Desktop is unable to start"

### Symptoms
```
$ docker compose up -d
unable to get image '...': Error response from daemon: Docker Desktop is unable to start

$ docker info
Server:
ERROR: Error response from daemon: Docker Desktop is unable to start
errors pretty printing info
```

Note: Docker Desktop **processes** may appear in Task Manager / `tasklist` even when the daemon is not ready. This is a false positive — do not trust process-level checks.

### Diagnostic sequence

```bash
# 1. Check daemon readiness (the real check)
docker info 2>&1 | grep -i "Server\|error\|running\|Engine"

# Expected output when healthy:
#   Server:
#   ......

# Expected output when broken:
#   Server:
#   ERROR: Error response from daemon: Docker Desktop is unable to start

# 2. Check Docker process existence (unreliable indicator)
tasklist 2>/dev/null | grep -i docker
```

### Resolution steps (in order)

1. **Wait** — Docker Desktop takes 30-90 seconds to fully initialize after Windows login. Try `docker info` every 15 seconds.

2. **Restart Docker Desktop** — Right-click the Docker Desktop icon in the system tray (bottom-right) → **Restart**. Wait 60s, then retry `docker info`.

3. **Restart Docker Desktop service** — If tray icon isn't responsive:
   ```powershell
   # Run as Administrator
   net stop com.docker.service
   net start com.docker.service
   ```
   Then launch Docker Desktop from Start Menu.

4. **Reset Docker Desktop** — Open Docker Desktop → Troubleshoot → Reset to factory defaults. This preserves images but clears containers and settings.

5. **Check WSL2** — Docker Desktop on Windows requires WSL2:
   ```powershell
   wsl -l -v
   ```
   If no distributions show, run `wsl --install -d ubuntu`.

### Issue: Docker overlayfs filesystem corruption on Windows

### Symptoms
```
docker compose up -d
```
Fails midway through image pulling with:
```
failed to extract layer (...tar+gzip sha256:...) to overlayfs as "...":
write /var/lib/desktop-containerd/daemon/io.containerd.snapshotter.v1.overlayfs/snapshots/.../fs/...: input/output error
```

The same error persists across retries (`docker compose pull && docker compose up -d`) and affects `docker system prune -a` as well:
```
Error response from daemon: write /var/lib/desktop-containerd/.../meta.db: input/output error
```

### Root cause
The Docker Desktop Linux VM's overlayfs metadata or cached layer data has suffered filesystem corruption. This is **not** a disk-space issue — the actual filesystem on the virtual disk backing Docker's Linux engine has I/O errors.

### Resolution
The only reliable fix is a full Docker Desktop reset:
1. Right-click Docker Desktop system tray icon → **Troubleshoot** → **Reset to factory defaults**
2. This clears the VM disk, all layer caches, and builds a fresh overlayfs
3. After reset, Docker Desktop restarts automatically (may take 1-2 minutes)
4. Re-run `docker compose up -d`

### Alternative: bypass Docker entirely
If Docker Desktop continues to fail after reset, run Dify via pip in a Python virtualenv instead:
```bash
cd /d/dify-main/api
python -m venv venv
source venv/Scripts/activate
pip install -r requirements.txt
flask run --host 0.0.0.0 --port 5001
```
Note: pip mode lacks the full Docker compose orchestration (no nginx, no worker queue, no plugin daemon) — suitable for testing the API layer only.

## Prevention
- Do NOT check via `tasklist` — Docker processes start immediately on boot but the engine may still be initializing for minutes.
