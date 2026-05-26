# Service Unavailability Patterns (内网部署)

This reference documents strategies for handling external service unavailability during multi-agent development, specifically for air-gapped / 内网 environments where Docker Hub, PyPI, or other registries are unreachable.

## Common Scenarios

| Scenario | Symptom | Detection |
|----------|---------|-----------|
| Docker Hub unreachable | `dial tcp 157.240.17.35:443: connectex: connection failed` | `curl https://registry-1.docker.io/v2/` times out |
| Docker Hub via mirror | Pull succeeds via mirror but Docker only uses it for `docker pull` not `docker compose` | Check `docker info` for Registry config |
| PyPI unreachable | `pip install` fails with timeout | `curl -s pypi.org` fails |

## Strategy 1: Built-in Service Fallback

When an upstream service (Dify, Elasticsearch, Redis) is essential for the API surface but unavailable, add a **built-in compatible mock** to the calling service.

### Architecture

```
/dify/{path}  ──►  proxy() / handler()
                     │
                     ├── if upstream available ──►  proxy to real service
                     │
                     └── else ──►  built-in handler()
                                    ├── GET  /dify/health           ──►  status + db check
                                    ├── GET  /dify/v1/models        ──►  from LM Studio
                                    ├── GET  /dify/v1/workflows     ──►  from local DB
                                    ├── POST /dify/v1/chat-messages  ──►  via LM Studio
                                    └── other ──►  404 + compat error
```

### Implementation pattern (FastAPI)

```python
# Module-level flag set at startup
dify_available = False

# In lifespan startup:
async def lifespan(app):
    global dify_available
    dify_available = await check_dify_availability()
    yield

# In proxy endpoint:
@router.api_route("/dify/{path:path}", methods=["GET", "POST", ...])
async def dify_proxy(path: str, request: Request):
    if dify_available:
        try:
            return await proxy_to_real_dify(path, request)
        except Exception:
            logger.warning("Dify proxy failed, falling back")
            return await _handle_builtin_dify(path, request)
    else:
        return await _handle_builtin_dify(path, request)
```

### Requirements for mock endpoints

- Return the **same response shape** as the real service (same JSON field names, status codes)
- Include a `"builtin": true` or `"source": "mock"` marker so callers can distinguish
- Log every fallback invocation so operators know the real service is down
- For chat/completion endpoints, relay through any available local inference (LM Studio, Ollama)

## Strategy 2: Docker Registry Mirror (Windows)

### Config location

```
C:\ProgramData\Docker\config\daemon.json
```

### Content

```json
{
  "registry-mirrors": ["https://docker.m.daocloud.io"],
  "dns": ["8.8.8.8", "8.8.4.4"],
  "ipv6": false,
  "experimental": false
}
```

### Caveats

- Docker Desktop v29+ on Windows uses WSL2 Docker engine — `daemon.json` goes in `C:\ProgramData\Docker\config\` (Windows path), not inside WSL
- The mirror config only affects `docker pull`, not `docker compose`'s implicit pulls
- Docker Desktop may still dial `registry-1.docker.io` directly via `hubproxy.docker.internal:5555`
- Check with `docker system info | grep -i "registry\|mirror\|proxy"`
- After changing daemon.json: restart Docker Desktop from system tray (not just `docker restart`)
- Test with: `docker pull langgenius/dify-api:1.14.2`

### Available mirrors for China

| Mirror | URL | Response |
|--------|-----|----------|
| DaoCloud | `https://docker.m.daocloud.io` | 302 redirect (works) |
| USTC | `https://docker.mirrors.ustc.edu.cn` | May be blocked |
| Tencent | `https://mirror.ccs.tencentyun.com` | 000 (unreachable) |

## Strategy 3: Build from Source Instead of Pull

When `docker pull` fails but you have the source code:

```bash
# Build a docker image from local source
cd /path/to/source
docker build -t langgenius/dify-api:1.14.2 -f api/Dockerfile api/
```

This still fails if the Dockerfile's `FROM python:3.12-slim-bookworm` can't be pulled either. Check locally cached images first:

```bash
docker images | grep <image-name>
```

## Strategy 4: Run Service Directly (No Docker)

For Python services, skip Docker entirely and run on the host:

```bash
cd /path/to/service
pip install -r requirements.txt
flask run --host 0.0.0.0 --port 5001
```

This requires all dependencies (DB, Redis, vector store) to also be host-accessible.

## Diagnosis Checklist

When an upstream service is "unavailable":

- [ ] Check if service process is running: `docker ps`, `netstat -ano | grep :PORT`
- [ ] Check port binding: `curl http://localhost:PORT/health`
- [ ] Check Docker Hub reachability: `curl -m 5 https://registry-1.docker.io/v2/`
- [ ] Check Docker registry mirrors: `docker system info | grep -i registry`
- [ ] For Docker Desktop: check `C:\ProgramData\Docker\config\daemon.json`
- [ ] Check if image is cached locally: `docker images | grep <service>`
- [ ] Check disk space: `docker system df`
