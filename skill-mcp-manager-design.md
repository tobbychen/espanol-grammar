# Skill & MCP Server Manager — Implementation Design

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Manager CLI / API                          │
│  skill-manager.py                                              │
│  ├── load-skill <name>@<version>                              │
│  ├── load-mcp <name>@<version>                                │
│  ├── list-skills                                              │
│  ├── list-mcps                                                │
│  └── update --all                                             │
└──────────────────────┬──────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ SkillStore  │  │  MCPRegistry │  │  AgentBridge │
│ ~/.skills/  │  │ ~/.mcps/    │  │  context injection │
└─────────────┘  └─────────────┘  └─────────────┘
        │              │              │
        ▼              ▼              ▼
  Git Repo         Git Repo      Agent Runtime
  skills-registry  mcps-registry
```

---

## Directory Structure

```
~/.skill-manager/
├── config.yaml                    # Global configuration
├── registry/                       # Local registry cache
│   ├── skills/
│   │   ├── index.json              # All available skills
│   │   └── es-translator/
│   │       ├── 1.2.0/
│   │       │   ├── SKILL.md
│   │       │   └── metadata.json
│   │       └── 1.3.0/
│   │           └── SKILL.md
│   └── mcps/
│       ├── index.json
│       ├── github-mcp/
│       │   ├── 1.2.0/
│       │   │   ├── manifest.json
│       │   │   └── dist/server.js
│       │   └── 1.3.0/
│       │       └── dist/server.js
│       └── filesystem-mcp/
│           └── 1.1.0/
├── processes/                      # Running MCP server processes
│   ├── github-mcp-1.2.0.pid
│   └── filesystem-mcp-1.1.0.pid
└── logs/
    └── mcp-github-mcp-1.2.0.log
```

---

## Core Components

### 1. Configuration Schema (config.yaml)

```yaml
registry:
  skills:
    url: git@github.com:your-org/skills-registry.git
    branch: main
    auth: ${GIT_SSH_KEY}  # or github token

  mcps:
    url: git@github.com:your-org/mcps-registry.git
    branch: main
    auth: ${GIT_SSH_KEY}

storage:
  base_dir: ~/.skill-manager

cache:
  skills_ttl: 24h       # How long before checking for updates
  mcps_ttl: 1h

runtime:
  mcp_stdio_timeout: 30s
  skill_context_max_tokens: 8000

security:
  verify_signatures: true
  allowed_skill_authors:
    - your-org
  allowed_mcp_sources:
    - your-org
```

### 2. Skill Manifest (skills/index.json)

```json
{
  "skills": [
    {
      "name": "es-translator",
      "version": "1.3.0",
      "description": "Translates English to Spanish with grammar analysis",
      "author": "your-org",
      "triggers": ["translate to spanish", "traducir", "en español"],
      "runtime": "claude-code",
      "entrypoint": "SKILL.md",
      "dependencies": [],
      "signature": "sha256:abc123...",
      "git_ref": "v1.3.0"
    },
    {
      "name": "code-reviewer", 
      "version": "2.0.1",
      "description": "Performs automated code review",
      "author": "your-org",
      "triggers": ["review code", "code review"],
      "runtime": "claude-code",
      "entrypoint": "SKILL.md",
      "dependencies": ["github-mcp"],
      "signature": "sha256:def456..."
    }
  ],
  "updated_at": "2025-05-08T10:00:00Z"
}
```

### 3. MCP Manifest (mcps/index.json)

```json
{
  "servers": [
    {
      "name": "github-mcp",
      "version": "1.2.0",
      "description": "GitHub integration for code, PRs, issues",
      "author": "your-org",
      "runtime": "node",
      "entrypoint": "dist/index.js",
      "command": "node",
      "args": ["dist/index.js"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      },
      "capabilities": ["search_code", "get_issues", "create_pr"],
      "dependencies": [],
      "build_required": true,
      "build_command": "npm install && npm run build",
      "signature": "sha256:ghi789..."
    },
    {
      "name": "filesystem-mcp",
      "version": "1.1.0",
      "description": "File system operations",
      "runtime": "python",
      "entrypoint": "server.py",
      "command": "python3",
      "args": ["server.py", "--port", "{port}"],
      "env": {},
      "capabilities": ["read_file", "write_file", "list_dir"],
      "build_required": false,
      "signature": "sha256:jkl012..."
    }
  ]
}
```

---

## Core Classes

### SkillManager

```python
# skill_manager/skill_loader.py

import hashlib
import json
import subprocess
from dataclasses import dataclass
from datetime import datetime, timedelta
from pathlib import Path
from typing import Optional
import yaml

@dataclass
class Skill:
    name: str
    version: str
    content: str
    metadata: dict
    trigger_patterns: list[str]

class SkillManager:
    def __init__(self, config_path: str = "~/.skill-manager/config.yaml"):
        self.config = self._load_config(config_path)
        self.base_dir = Path(self.config["storage"]["base_dir"]).expanduser()
        self.registry_dir = self.base_dir / "registry" / "skills"
        self.cache_dir = self.base_dir / "cache"
        
    def _load_config(self, path: str) -> dict:
        # Load and interpolate env vars
        content = Path(path).expanduser().read_text()
        # Replace ${VAR} with env vars
        import re
        def replacer(m):
            import os
            return os.environ.get(m.group(1), m.group(0))
        content = re.sub(r'\$\{([^}]+)\}', replacer, content)
        return yaml.safe_load(content)
    
    def list_skills(self) -> list[dict]:
        """List all available skills from registry."""
        index_path = self.registry_dir / "index.json"
        if not index_path.exists():
            self._sync_registry("skills")
        return json.loads(index_path.read_text())["skills"]
    
    def load_skill(self, name: str, version: str = "latest") -> Skill:
        """
        Load a skill by name and version.
        Version can be: "latest", "1.2.0", "^1.0", "~1.2"
        """
        # Ensure registry is up to date
        self._sync_if_stale("skills")
        
        # Resolve version
        resolved = self._resolve_version("skills", name, version)
        
        # Check cache
        skill_dir = self.cache_dir / "skills" / name / resolved
        skill_path = skill_dir / "SKILL.md"
        metadata_path = skill_dir / "metadata.json"
        
        if not skill_dir.exists():
            self._fetch_skill(name, resolved)
        
        # Verify signature if enabled
        if self.config.get("security", {}).get("verify_signatures"):
            self._verify_signature(name, resolved)
        
        return Skill(
            name=name,
            version=resolved,
            content=skill_path.read_text(),
            metadata=json.loads(metadata_path.read_text()),
            trigger_patterns=json.loads(metadata_path.read_text()).get("triggers", [])
        )
    
    def _sync_if_stale(self, registry_type: str) -> None:
        """Check if registry needs refresh based on TTL."""
        index_path = self.registry_dir.parent.parent / registry_type / "index.json"
        if not index_path.exists():
            self._sync_registry(registry_type)
            return
            
        last_modified = datetime.fromtimestamp(index_path.stat().st_mtime)
        ttl = self.config["cache"].get(f"{registry_type}_ttl", "24h")
        # Parse TTL and check
        
    def _sync_registry(self, registry_type: str) -> None:
        """Clone/pull the registry repo."""
        url = self.config["registry"][registry_type.rstrip('s')]["url"]
        target = self.registry_dir.parent.parent / registry_type
        
        if target.exists():
            subprocess.run(["git", "pull", "origin", "main"], cwd=target, check=True)
        else:
            subprocess.run(["git", "clone", url, str(target)], check=True)
    
    def _resolve_version(self, registry_type: str, name: str, version: str) -> str:
        """Resolve semantic version spec to actual version string."""
        index = self.list_skills() if registry_type == "skills" else self.list_mcps()
        entries = [e for e in index if e["name"] == name]
        if not entries:
            raise ValueError(f"Unknown {registry_type.rstrip('s')}: {name}")
        
        if version == "latest":
            # Return highest semver
            return sorted(entries, key=lambda x: x["version"])[-1]["version"]
        
        # Handle semver ranges: ^1.0, ~1.2, >=1.0
        # Use packaging.version for proper semver handling
        from packaging.version import Version, InvalidVersion
        try:
            target = Version(version)
            for entry in entries:
                try:
                    if Version(entry["version"]) == target:
                        return entry["version"]
                except InvalidVersion:
                    continue
        except InvalidVersion:
            pass
        
        return version  # Return as-is if not semver
    
    def _fetch_skill(self, name: str, version: str) -> None:
        """Clone the skill repo and checkout the specific version."""
        skill_dir = self.cache_dir / "skills" / name / version
        skill_dir.parent.mkdir(parents=True, exist_ok=True)
        
        # Clone with depth=1 and checkout specific tag
        subprocess.run([
            "git", "clone", "--depth", "1", 
            f"--branch", f"v{version}",
            self.config["registry"]["skills"]["url"],
            str(skill_dir)
        ], check=True)
    
    def _verify_signature(self, name: str, version: str) -> bool:
        """Verify skill signature against trusted keys."""
        # Implementation: compute SHA256 of SKILL.md, compare with manifest
        pass
    
    def trigger_skill(self, user_input: str) -> Optional[Skill]:
        """Auto-detect which skill to use based on user input patterns."""
        for skill in self.list_skills():
            for pattern in skill.get("triggers", []):
                if pattern.lower() in user_input.lower():
                    return self.load_skill(skill["name"], skill["version"])
        return None
```

### MCPManager

```python
# skill_manager/mcp_loader.py

import subprocess
import json
import time
import signal
import os
from dataclasses import dataclass
from pathlib import Path
from typing import Optional
import yaml

@dataclass
class MCPServer:
    name: str
    version: str
    manifest: dict
    
class MCPManager:
    def __init__(self, config_path: str = "~/.skill-manager/config.yaml"):
        self.config = self._load_config(config_path)
        self.base_dir = Path(self.config["storage"]["base_dir"]).expanduser()
        self.cache_dir = self.base_dir / "cache" / "mcps"
        self.processes_dir = self.base_dir / "processes"
        self.processes: dict[str, subprocess.Popen] = {}
        
    def list_mcps(self) -> list[dict]:
        """List all available MCP servers."""
        index_path = self.base_dir / "registry" / "mcps" / "index.json"
        if not index_path.exists():
            self._sync_registry("mcps")
        return json.loads(index_path.read_text())["servers"]
    
    def load_mcp(self, name: str, version: str = "latest") -> MCPServer:
        """Load and start an MCP server."""
        # Resolve version
        resolved = self._resolve_version(name, version)
        
        # Build server if needed
        server_dir = self._ensure_server(name, resolved)
        
        # Spawn server process
        process = self._spawn_server(name, resolved, server_dir)
        
        # Perform handshake
        self._mcp_handshake(process)
        
        return MCPServer(name=name, version=resolved, manifest=self._load_manifest(name, resolved))
    
    def _ensure_server(self, name: str, version: str) -> Path:
        """Ensure server is built and ready."""
        server_dir = self.cache_dir / name / version
        
        if not server_dir.exists():
            self._fetch_and_build(name, version, server_dir)
        else:
            # Check if build is complete
            manifest = json.loads((server_dir / "manifest.json").read_text())
            if manifest.get("build_required") and not (server_dir / "dist").exists():
                self._build(name, version, server_dir)
        
        return server_dir
    
    def _fetch_and_build(self, name: str, version: str, target: Path) -> None:
        """Clone repo and build the MCP server."""
        target.mkdir(parents=True, exist_ok=True)
        
        # Clone specific version
        url = self.config["registry"]["mcps"]["url"]
        subprocess.run([
            "git", "clone", "--depth", "1",
            "--branch", f"v{version}",
            url, str(target)
        ], check=True)
        
        # Build if required
        manifest = self._load_manifest_raw(target / "manifest.json")
        if manifest.get("build_required"):
            self._build(name, version, target)
    
    def _build(self, name: str, version: str, server_dir: Path) -> None:
        """Run build command for MCP server."""
        manifest = self._load_manifest_raw(server_dir / "manifest.json")
        build_cmd = manifest.get("build_command", "npm install && npm run build")
        
        # Handle env vars
        env = os.environ.copy()
        for key, val in manifest.get("env", {}).items():
            if val.startswith("${") and val.endswith("}"):
                env_var = val[2:-1]
                env[key] = os.environ.get(env_var, "")
        
        subprocess.run(
            build_cmd,
            shell=True,
            cwd=server_dir,
            env=env,
            check=True
        )
    
    def _spawn_server(self, name: str, version: str, server_dir: Path) -> subprocess.Popen:
        """Spawn the MCP server as a subprocess."""
        manifest = self._load_manifest_raw(server_dir / "manifest.json")
        
        # Prepare args with env interpolation
        args = []
        for arg in manifest.get("args", []):
            if arg.startswith("${") and arg.endswith("}"):
                env_var = arg[2:-1]
                args.append(os.environ.get(env_var, ""))
            else:
                args.append(arg)
        
        # Prepare env
        env = os.environ.copy()
        for key, val in manifest.get("env", {}).items():
            if val.startswith("${") and val.endswith("}"):
                env_var = val[2:-1]
                env[key] = os.environ.get(env_var, "")
            else:
                env[key] = val
        
        # Build command
        cmd = [manifest["command"]] + args
        
        # Spawn process
        process = subprocess.Popen(
            cmd,
            cwd=server_dir,
            env=env,
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True
        )
        
        # Store PID
        pid_file = self.processes_dir / f"{name}-{version}.pid"
        pid_file.write_text(str(process.pid))
        
        self.processes[f"{name}@{version}"] = process
        return process
    
    def _mcp_handshake(self, process: subprocess.Popen) -> None:
        """Perform MCP JSON-RPC initialization handshake."""
        init_request = {
            "jsonrpc": "2.0",
            "id": 1,
            "method": "initialize",
            "params": {
                "protocolVersion": "2024-11-05",
                "capabilities": {},
                "clientInfo": {
                    "name": "skill-manager",
                    "version": "1.0.0"
                }
            }
        }
        
        # Send init
        process.stdin.write(json.dumps(init_request) + "\n")
        process.stdin.flush()
        
        # Wait for response
        import select
        start = time.time()
        while time.time() - start < self.config["runtime"]["mcp_stdio_timeout"]:
            if select.select([process.stdout], [], [], 0.1)[0]:
                line = process.stdout.readline()
                response = json.loads(line)
                if response.get("id") == 1:
                    return  # Handshake complete
        
        raise TimeoutError("MCP handshake timeout")
    
    def stop_mcp(self, name: str, version: str) -> None:
        """Stop a running MCP server."""
        key = f"{name}@{version}"
        if key in self.processes:
            process = self.processes[key]
            process.terminate()
            try:
                process.wait(timeout=5)
            except subprocess.TimeoutExpired:
                process.kill()
            del self.processes[key]
        
        # Clean up PID file
        pid_file = self.processes_dir / f"{name}-{version}.pid"
        if pid_file.exists():
            pid_file.unlink()
    
    def _resolve_version(self, name: str, version: str) -> str:
        """Resolve version spec to actual version."""
        # Same logic as SkillManager
        pass
    
    def _sync_registry(self, registry_type: str) -> None:
        """Sync registry repo."""
        # Similar to SkillManager
        pass
```

### Combined Manager (Facade)

```python
# skill_manager/manager.py

from .skill_loader import SkillManager, Skill
from .mcp_loader import MCPManager, MCPServer

class Manager:
    """Combined facade for both skills and MCPs."""
    
    def __init__(self, config_path: str = "~/.skill-manager/config.yaml"):
        self.skills = SkillManager(config_path)
        self.mcps = MCPManager(config_path)
        self.active_skills: list[Skill] = []
        self.active_mcps: dict[str, MCPServer] = {}
    
    def load(self, name: str, version: str = "latest") -> Skill | MCPServer:
        """Load either a skill or MCP by name."""
        # Try skill first
        try:
            skill = self.skills.load_skill(name, version)
            self.active_skills.append(skill)
            return skill
        except ValueError:
            pass
        
        # Try MCP
        try:
            mcp = self.mcps.load_mcp(name, version)
            self.active_mcps[f"{name}@{version}"] = mcp
            return mcp
        except ValueError:
            pass
        
        raise ValueError(f"Unknown skill or MCP: {name}")
    
    def trigger(self, user_input: str) -> Optional[Skill]:
        """Auto-detect and load skill based on input."""
        skill = self.skills.trigger_skill(user_input)
        if skill:
            self.active_skills.append(skill)
        return skill
    
    def get_active_context(self) -> dict:
        """Get all active skills and MCPs for agent context."""
        return {
            "skills": [
                {
                    "name": s.name,
                    "version": s.version,
                    "content": s.content
                }
                for s in self.active_skills
            ],
            "mcps": {
                name: {
                    "version": m.version,
                    "capabilities": m.manifest.get("capabilities", [])
                }
                for name, m in self.active_mcps.items()
            }
        }
    
    def cleanup(self) -> None:
        """Stop all MCP servers and clear skills."""
        for key in list(self.active_mcps.keys()):
            name, version = key.split("@")
            self.mcps.stop_mcp(name, version)
        self.active_skills.clear()
        self.active_mcps.clear()
```

---

## Agent Integration

### Agent Bridge (for Claude Code or similar)

```python
# agent_bridge.py

class AgentBridge:
    """
    Bridge between Manager and Agent runtime.
    Injects skills into context and connects MCP servers.
    """
    
    def __init__(self, manager: Manager):
        self.manager = manager
    
    def inject_into_session(self, session_context: dict) -> dict:
        """
        Take agent session context and inject:
        1. Active skills as system prompts
        2. MCP server connections
        """
        active = self.manager.get_active_context()
        
        # Inject skills as system prompt additions
        skill_prompts = []
        for skill in active["skills"]:
            skill_prompts.append(
                f"\n\n--- BEGIN SKILL: {skill['name']} v{skill['version']} ---\n"
                f"{skill['content']}"
                f"\n--- END SKILL ---"
            )
        
        if skill_prompts:
            session_context["system_prompt"] = (
                session_context.get("system_prompt", "") + 
                "\n".join(skill_prompts)
            )
        
        # Inject MCP tool definitions
        session_context["mcp_tools"] = self._build_mcp_tools(active["mcps"])
        
        return session_context
    
    def _build_mcp_tools(self, mcps: dict) -> list[dict]:
        """Convert MCP capabilities to agent tool format."""
        tools = []
        for key, mcp in mcps.items():
            for cap in mcp.get("capabilities", []):
                tools.append({
                    "name": f"{mcp['name']}_{cap}",
                    "description": f"Use {mcp['name']} capability: {cap}",
                    "input_schema": {...}  # MCP specific
                })
        return tools
    
    def handle_tool_call(self, tool_name: str, tool_input: dict) -> dict:
        """Route tool call to appropriate MCP server."""
        # Parse tool name to find MCP + capability
        # Send JSON-RPC to appropriate subprocess
        pass
```

---

## CLI Interface

```bash
# skill-manager CLI

# List all available
skill-manager list skills
skill-manager list mcps

# Load specific
skill-manager load es-translator@1.2.0
skill-manager load github-mcp@latest

# Auto-trigger from input
skill-manager run "translate this to spanish" --input "hello world"

# Update
skill-manager update --all
skill-manager update skills
skill-manager update mcps

# Remove from cache
skill-manager cache clean --all
skill-manager cache clean es-translator@1.2.0

# Status
skill-manager status
```

---

## Version Resolution

Supports semver ranges:

| Spec | Meaning |
|------|---------|
| `1.2.0` | Exact version |
| `latest` | Highest version |
| `^1.2.0` | Compatible (>=1.2.0, <2.0.0) |
| `~1.2.0` | Patch compatible (>=1.2.0, <1.3.0) |
| `>=1.0` | Minimum version |

---

## Security Model

```yaml
# config.yaml security section
security:
  verify_signatures: true
  
  # Only allow skills from these authors
  allowed_skill_authors:
    - your-org
  
  # Only allow MCPs from these sources  
  allowed_mcp_sources:
    - your-org
    - modelcontextprotocol  # official MCP servers
  
  # Network restrictions for MCP servers
  mcp_network_allowed: false  # true = allow MCP servers to make network calls
  
  # Filesystem restrictions for MCP servers
  mcp_fs_root: /allowed/path  # Limit filesystem access to this path
```

---

## Next Steps / Future Enhancements

| Feature | Priority | Notes |
|---------|----------|-------|
| **Dependency resolution** | High | Skill needs MCP → auto-load |
| **Hot reload** | Medium | Update skill/MCP without restart |
| **Observability** | Medium | Logs, metrics, traces |
| **UI dashboard** | Low | Web UI to manage |
| **Multi-tenant** | Low | Teams with different permissions |

---

## Files to Create

```
skill-manager/
├── pyproject.toml
├── setup.py
├── skill_manager/
│   ├── __init__.py
│   ├── config.py
│   ├── skill_loader.py
│   ├── mcp_loader.py
│   ├── manager.py
│   ├── agent_bridge.py
│   ├── utils.py
│   └── exceptions.py
├── cli/
│   ├── __init__.py
│   └── main.py
├── examples/
│   ├── config.yaml.example
│   └── skills/
│       └── es-translator/
│           ├── SKILL.md
│           └── metadata.json
├── tests/
│   ├── test_skill_loader.py
│   └── test_mcp_loader.py
└── README.md
```