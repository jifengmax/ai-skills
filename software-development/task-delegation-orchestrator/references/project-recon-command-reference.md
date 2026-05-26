# Project Reconnaissance Command Reference

Quick probe commands for understanding an unknown codebase before task decomposition. Run these during Phase 1 Step 2.

## Python Projects

```bash
# Top-level structure
ls -la
find . -maxdepth 2 -type f -name "*.py" | head -20

# Dependencies
cat requirements.txt
cat setup.py 2>/dev/null || cat pyproject.toml 2>/dev/null

# Project size metrics
find . -name "*.py" -not -path "./venv/*" -not -path "./.venv/*" | wc -l
find . -name "*.py" -not -path "./venv/*" -exec wc -l {} + | tail -1

# Database / init scripts
find . -name "init_db*" -o -name "migration*" -o -name "schema*"
cat init_db.py | head -50

# Deployment config
cat Dockerfile 2>/dev/null | head -20
cat docker-compose.yml 2>/dev/null | head -30
cat nginx.conf 2>/dev/null | head -20

# Git history
git log --oneline -10 2>/dev/null

# Frontend
find . -name "*.html" -type f
find . -name "*.js" -type f -not -path "./node_modules/*" | head -10
find . -name "package.json" -type f

# Design documents
find . -name "*.md" -type f -not -path "./node_modules/*" -not -path "./venv/*" | head -20
```

## Node.js / JS Projects

```bash
ls -la
cat package.json | head -30
cat tsconfig.json 2>/dev/null
find . -name "*.ts" -type f | wc -l
find . -name "*.tsx" -type f | wc -l
git log --oneline -10 2>/dev/null
```

## Go Projects

```bash
ls -la
cat go.mod
find . -name "*.go" -type f | wc -l
find . -name "main.go" -o -name "cmd/*"
git log --oneline -10 2>/dev/null
```

## Rust Projects

```bash
ls -la
cat Cargo.toml
find . -name "*.rs" -type f | wc -l
git log --oneline -10 2>/dev/null
```

## General Probes (any language)

```bash
# Documentation clues
find . \( -name "README.md" -o -name "*.md" -o -name "*.pdf" -o -name "overview.md" \) -not -path "./node_modules/*"

# Binary/files
ls *.sh *.ps1 *.bat 2>/dev/null
find . -maxdepth 1 -type f -name "*.yaml" -o -name "*.yml" -o -name "*.toml" -o -name "*.ini" -o -name "*.env*" | head -10

# Nginx
find . -name "nginx*" -o -name "*.conf" | head -5

# MCP plugins
find . -path "*/mcp_*" -type f -name "server.py" 2>/dev/null | head -10

# Skill files
find . -path "*/skills/*" -type f -name "*.md" | head -20

# Running services
curl -s --max-time 3 http://localhost:8000/health 2>/dev/null || echo "port 8000: no response"
curl -s --max-time 3 http://localhost:1234/v1/models 2>/dev/null || echo "port 1234: no response"
curl -s --max-time 3 http://localhost:5001/health 2>/dev/null || echo "port 5001: no response"
curl -s --max-time 3 http://localhost/v1/health 2>/dev/null || echo "port 80: no response"

# Docker status
docker info 2>/dev/null | head -3 || echo "Docker not running"
docker ps 2>/dev/null | head -10 || echo "Docker not available"

# venv detection
find . -maxdepth 3 -name "venv" -type d 2>/dev/null
find . -maxdepth 3 -name ".venv" -type d 2>/dev/null
```

## Quick Codebase Summary (run after probes)

```bash
# Give a one-line summary per major directory
echo "=== Directory Overview ==="
for d in */; do
    count=$(find "$d" -type f 2>/dev/null | wc -l)
    echo "  $d: $count files"
done
```