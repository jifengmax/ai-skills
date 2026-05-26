# MCP SDK Version Compatibility: FastMCP(version) Parameter

## The Issue

When MCP SDK (`mcp` package) version is **>=1.0.0 and <2.0.0** (specifically 1.27.x), `FastMCP()` does NOT accept a `version` keyword argument:

```python
from mcp.server.fastmcp import FastMCP

# This FAILS on mcp 1.27.x:
mcp = FastMCP("my-server", version="1.0.0")
# TypeError: FastMCP.__init__() got an unexpected keyword argument 'version'

# This works on ALL versions:
mcp = FastMCP("my-server")
```

## How to Check Installed Version

```bash
pip show mcp 2>/dev/null | grep Version
# or
python -c "import mcp; print(mcp.__version__)"
```

## How to Fix

Remove the `version=` parameter from ALL `FastMCP()` calls in your server files. The server name is sufficient:

```python
# Before (broken on 1.27.x):
mcp = FastMCP("mcp-calculator", version="1.0.0")

# After (works everywhere):
mcp = FastMCP("mcp-calculator")
```

## Scope

This affects all 4 custom MCP plugins in the oneteam project:
- `mcp_plugins/mcp-calculator/server.py`
- `mcp_plugins/mcp-police-db/server.py`
- `mcp_plugins/mcp-map-geo/server.py`
- `mcp_plugins/mcp-notify/server.py`

## Why This Happens

Newer versions of the `mcp` SDK (1.27.1 in this environment) may remove or change `FastMCP` constructor parameters. The exact git history of the `fastmcp` subpackage explains it — `version` was either not implemented yet or was added in a later patch. The safest approach is to always omit `version` from `FastMCP()` unless you've validated it works on the target SDK version.

## Diagnosis Script

```python
import mcp
from mcp.server.fastmcp import FastMCP
try:
    mcp = FastMCP("test", version="1.0.0")
    print("version parameter SUPPORTED")
except TypeError as e:
    print(f"version parameter NOT supported: {e}")
    mcp = FastMCP("test")
    print("fallback works")
```
