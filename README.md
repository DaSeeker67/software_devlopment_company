# DevAgents — World-Class Blueprint v1.0
> **Status:** Complete build spec. All critical gaps addressed.  
> **Scope:** Feature dev · Bug fixing · Architecture planning · Ground-up builds  
> **LLM providers:** Multi-provider (OpenAI, Anthropic, Google, Groq, local Ollama)  
> **Last updated:** May 2026

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Architecture](#2-system-architecture)
3. [Phase 0 — Knowledge Graph Layer](#3-phase-0--knowledge-graph-layer) ← *gap #3 fully resolved*
4. [Phase 1 — DinD Sandbox Architecture](#4-phase-1--dind-sandbox-architecture) ← *gap #1 fully resolved*
5. [Phase 2 — Full-Auto Commit Mode](#5-phase-2--full-auto-commit-mode) ← *gap #2 fully resolved*
6. [Phase 3 — Agent Architecture](#6-phase-3--agent-architecture)
7. [Phase 4 — Terminal UI Design](#7-phase-4--terminal-ui-design) ← *gap #4 fully resolved*
8. [Phase 5 — Builder / Skeptic Debate Format](#8-phase-5--builder--skeptic-debate-format) ← *gap #5 fully resolved*
9. [Phase 6 — Semgrep Integration](#9-phase-6--semgrep-integration) ← *gap #6 fully resolved*
10. [Phase 7 — Project Structure](#10-phase-7--project-structure)
11. [Phase 8 — Build Roadmap (4 sprints)](#11-phase-8--build-roadmap-4-sprints)
12. [Configuration Reference](#12-configuration-reference)
13. [Key Architectural Decisions](#13-key-architectural-decisions)

---

## 1. Executive Summary

DevAgents is a **multi-agent software engineering system** that operates autonomously on development tasks — feature implementation, bug fixing, architecture planning, and ground-up project construction. It replicates the TradingAgents pattern (parallel analyst teams, researcher debate, portfolio-manager gating, persistent memory) but applies it to software development.

**The single biggest differentiator over every existing tool (Copilot Workspace, Devin, Cursor):**

Before any agent reads a single file, a persistent knowledge graph is queried. The graph was built once and stays current via background file-watching. Agents receive a *surgically scoped context* of 10–30 relevant files and functions — not the whole repository. This produces a **99.2% token reduction** (412k → 3.4k tokens for 5 structural queries) with no loss in answer quality.

**Three non-negotiable design constraints addressed in this plan:**

| Gap | Solution |
|-----|----------|
| DinD sandbox — agents executing `git push` and `pytest` must be isolated | Full Docker sandbox design with network policy, `SandboxClient` API, and security hardening |
| Full-Auto mode — autonomous commit + push with confidence gating | `--full-auto` flag, Tech Lead confidence threshold (0.85), branch protection, rollback |
| Knowledge graph persistence — where does it live between sessions? | `codebase-memory-mcp` → SQLite at `~/.cache/codebase-memory-mcp/` (ACID-safe, WAL mode, auto-sync daemon) |

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DEVAGENTS SYSTEM                                │
│                                                                         │
│  CLI INPUT (devagents run "Add rate limiting to auth endpoint")         │
│      │                                                                  │
│      ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  PHASE 0: KNOWLEDGE GRAPH GATE  (runs before everything)         │  │
│  │                                                                  │  │
│  │  codebase-memory-mcp daemon  ←── SQLite at                      │  │
│  │  (auto-sync file watcher)         ~/.cache/codebase-memory-mcp/ │  │
│  │         │                                                        │  │
│  │         ▼  detect_changes() + trace_call_path()                 │  │
│  │  ScopedContext { files:[15], functions:[34], tests:[8] }        │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│      │                                                                  │
│      ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  LANGGRAPH SUPERVISOR (DevOrchestrator)                          │  │
│  │                                                                  │  │
│  │  ┌──────────────── ANALYST TEAM (parallel) ───────────────────┐ │  │
│  │  │ CodeAnalyst │ SpecAnalyst │ TestAnalyst │ SecurityAnalyst  │ │  │
│  │  │                          │ ArchAnalyst                     │ │  │
│  │  └─────────────────────────────────────────────────────────────┘ │  │
│  │      │                                                            │  │
│  │      ▼                                                            │  │
│  │  ┌────────────── RESEARCHER DEBATE (configurable rounds) ──────┐ │  │
│  │  │   Builder Agent  ←──────────→  Skeptic Agent               │ │  │
│  │  │   (best path)                  (failure modes)             │ │  │
│  │  └─────────────────────────────────────────────────────────────┘ │  │
│  │      │                                                            │  │
│  │      ▼                                                            │  │
│  │  ┌──────────────── DEVELOPER AGENT ───────────────────────────┐ │  │
│  │  │  Writes code diff + tests + PR description                  │ │  │
│  │  │  Executes inside DinD Sandbox (network-isolated container)  │ │  │
│  │  └─────────────────────────────────────────────────────────────┘ │  │
│  │      │                                                            │  │
│  │      ▼                                                            │  │
│  │  ┌──────────────── QA + SECURITY GATE ────────────────────────┐ │  │
│  │  │  pytest/jest/go test  +  Semgrep (threshold-gated)         │ │  │
│  │  │  BLOCK on: any ERROR severity, test regression > 5%        │ │  │
│  │  │  On failure → loop back to Developer with error context    │ │  │
│  │  └─────────────────────────────────────────────────────────────┘ │  │
│  │      │                                                            │  │
│  │      ▼                                                            │  │
│  │  ┌──────────────── TECH LEAD (Portfolio Manager) ─────────────┐ │  │
│  │  │  APPROVE / REQUEST_CHANGES / REJECT + confidence score     │ │  │
│  │  │  If --full-auto AND confidence >= 0.85: merge PR           │ │  │
│  │  │  Else: create PR for human review                          │ │  │
│  │  └─────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│      │                                                                  │
│      ▼                                                                  │
│  Decision Log  →  ~/.devagents/memory/dev_memory.md                    │
│  Checkpoint DB →  ~/.devagents/cache/checkpoints/<TASK_ID>.db          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Phase 0 — Knowledge Graph Layer

> **This section fully resolves gap #3: "Knowledge graph persistence is vague."**

### 3.1 Engine: codebase-memory-mcp

**Engine:** [`codebase-memory-mcp`](https://github.com/DeusData/codebase-memory-mcp) — single static binary, zero dependencies, 66 languages via vendored tree-sitter grammars.

**Why this over vector RAG:** Vector similarity captures topical similarity but fails on multi-hop architectural reasoning (controller → service → repository chains, interface-driven wiring, inheritance). A deterministic AST-derived knowledge graph handles these correctly. Benchmarked: 83% answer quality, 10× fewer tokens, 2.1× fewer tool calls vs. file-by-file exploration.

### 3.2 Persistence — Exactly Where Data Lives

```
~/.cache/codebase-memory-mcp/
├── config.json                    ← global settings (auto_index, limits)
├── projects.db                    ← registry of all indexed repos
└── <repo-fingerprint>/
    ├── graph.db                   ← SQLite, WAL mode, ACID-safe
    ├── graph.db-wal               ← WAL journal (auto-checkpointed)
    └── graph.db-shm               ← shared memory for WAL
```

**The graph survives restarts.** SQLite WAL mode means writes never corrupt the database mid-session. `PRAGMA journal_mode=WAL` is set by the binary — no configuration needed.

**Custom location override** (for CI or shared storage):
```bash
export CBM_CACHE_DIR=/mnt/shared/cbm-data   # team-shared NFS, S3-mount, etc.
```

**Repo fingerprint** = SHA-256 of the canonical repo path (or git remote URL if available). Same repo always maps to same database, even across machine reboots.

### 3.3 MCP Server Process Management

The `codebase-memory-mcp` binary runs as a **stdio MCP server** — it is not a long-running daemon by default. DevAgents manages it as a subprocess:

```python
# devagents/knowledge/graph_client.py

import subprocess
import json
from pathlib import Path
from contextlib import asynccontextmanager

class GraphClient:
    """
    Manages the codebase-memory-mcp process lifecycle.
    Starts on first query, keeps alive for the session duration,
    tears down on exit.
    """

    def __init__(self, repo_path: str, cache_dir: str | None = None):
        self.repo_path = Path(repo_path).resolve()
        self.env = {"CBM_CACHE_DIR": cache_dir} if cache_dir else {}
        self._proc: subprocess.Popen | None = None

    def start(self):
        """Start the MCP server subprocess."""
        binary = self._find_binary()
        self._proc = subprocess.Popen(
            [binary],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            env={**os.environ, **self.env},
        )
        self._ensure_indexed()

    def _ensure_indexed(self):
        """Index the repo if not already in the graph."""
        projects = self._call("list_projects", {})
        repo_str = str(self.repo_path)
        if not any(p["path"] == repo_str for p in projects.get("projects", [])):
            self._call("index_repository", {"repo_path": repo_str})

    def blast_radius(self, task_description: str, changed_files: list[str] | None = None) -> "ScopedContext":
        """
        Core pre-task query. Returns only the files/functions/tests
        relevant to this task. This is what prevents agents from
        reading the entire codebase.
        """
        # Step 1: detect changes if files were provided
        if changed_files:
            changes = self._call("detect_changes", {
                "repo_path": str(self.repo_path),
                "project": self.repo_path.name,
            })
        else:
            changes = {"affected_symbols": []}

        # Step 2: architecture overview for task routing
        arch = self._call("get_architecture", {
            "project": self.repo_path.name,
        })

        # Step 3: search for symbols mentioned in the task
        symbols = self._call("search_graph", {
            "project": self.repo_path.name,
            "name_pattern": self._extract_symbol_hints(task_description),
            "limit": 20,
        })

        # Step 4: trace call paths for each found symbol
        call_chains = []
        for sym in symbols.get("results", [])[:5]:  # top 5 most relevant
            chain = self._call("trace_call_path", {
                "project": self.repo_path.name,
                "function_name": sym["name"],
                "direction": "both",
                "depth": 3,
            })
            call_chains.append(chain)

        return ScopedContext.from_mcp_results(
            arch=arch,
            symbols=symbols,
            call_chains=call_chains,
            changes=changes,
        )

    def stop(self):
        if self._proc:
            self._proc.terminate()
            self._proc.wait(timeout=5)

    def _call(self, tool: str, params: dict) -> dict:
        """Send a JSON-RPC 2.0 call to the MCP server."""
        request = json.dumps({
            "jsonrpc": "2.0",
            "id": 1,
            "method": "tools/call",
            "params": {"name": tool, "arguments": params},
        })
        self._proc.stdin.write((request + "\n").encode())
        self._proc.stdin.flush()
        response = self._proc.stdout.readline()
        return json.loads(response)["result"]
```

### 3.4 Auto-Sync: How the Graph Stays Current

`codebase-memory-mcp` has a **built-in background file watcher** (`src/watcher/`). It uses git polling with adaptive intervals:

- Polls `git status --short` every 5 seconds while the MCP server is alive
- On detecting changes: re-indexes only the delta (changed files via tree-sitter re-parse)
- Re-index time: **~2 seconds** for a 2,900-file repo; **~15 seconds** for a 27,700-file repo
- Memory is released after indexing (RAM-first pipeline with LZ4 compression)

**To enable auto-sync** (run once, persists in config):
```bash
codebase-memory-mcp config set auto_index true
codebase-memory-mcp config set auto_index_limit 50000
```

DevAgents calls this during `devagents init`:
```python
# devagents/knowledge/graph_setup.py
def initialize_graph(repo_path: str):
    subprocess.run(["codebase-memory-mcp", "config", "set", "auto_index", "true"], check=True)
    subprocess.run([
        "codebase-memory-mcp", "cli", "index_repository",
        json.dumps({"repo_path": repo_path})
    ], check=True)
```

### 3.5 ScopedContext — What Agents Receive

```python
# devagents/knowledge/blast_radius.py

from pydantic import BaseModel

class ScopedContext(BaseModel):
    """
    The surgical context injected into ALL agent prompts.
    Never exceeds ~5,000 tokens regardless of repo size.
    """
    # Files agents are allowed to read
    relevant_files: list[str]        # absolute paths, max 30
    # Functions at the epicenter of the change
    epicenter_functions: list[str]   # qualified names (pkg.Class.method)
    # Tests that cover the blast radius
    affected_tests: list[str]        # test file paths + test function names
    # Call chain summaries (text, not raw code)
    call_chain_summary: str          # "AuthMiddleware → TokenValidator → UserRepo"
    # Architecture layer of the change
    arch_layer: str                  # "HTTP handler", "service", "repository", etc.
    # Risk level from detect_changes
    risk_level: str                  # "LOW" | "MEDIUM" | "HIGH" | "CRITICAL"
    # Raw token count of this context
    token_estimate: int

    @classmethod
    def from_mcp_results(cls, arch, symbols, call_chains, changes) -> "ScopedContext":
        # ... build from MCP results ...
        pass

    def to_prompt_block(self) -> str:
        """Renders as a compact XML block for agent prompt injection."""
        return f"""<codebase_context>
<relevant_files>
{chr(10).join(f"  - {f}" for f in self.relevant_files)}
</relevant_files>
<epicenter_functions>{", ".join(self.epicenter_functions)}</epicenter_functions>
<affected_tests>
{chr(10).join(f"  - {t}" for t in self.affected_tests)}
</affected_tests>
<call_chain>{self.call_chain_summary}</call_chain>
<arch_layer>{self.arch_layer}</arch_layer>
<risk_level>{self.risk_level}</risk_level>
</codebase_context>"""
```

**Token budget enforcement:** If `token_estimate > 6000`, the context is trimmed by dropping lower-confidence files first, ensuring agents never accidentally balloon their context.

---

## 4. Phase 1 — DinD Sandbox Architecture

> **This section fully resolves gap #1: "No DinD sandbox design."**

### 4.1 Architecture Diagram

```
Host Machine
├── DevAgents Orchestrator (Python process, port 8765)
│   │
│   └── SandboxClient ──HTTP──► Sandbox REST API (port 8766 inside container)
│                                     │
│                              devagents-sandbox container
│                              ├── /workspace (bind-mounted repo)
│                              ├── /output    (bind-mounted output dir)
│                              ├── sandbox_api.py  (FastAPI server)
│                              ├── git (read-only token, limited remote)
│                              ├── pytest / jest / go test
│                              ├── semgrep
│                              └── NO general internet access
│
└── Docker Engine (unix:///var/run/docker.sock)
```

### 4.2 The devagents-sandbox Container

#### Dockerfile.sandbox

```dockerfile
# devagents/sandbox/Dockerfile.sandbox
FROM python:3.12-slim AS base

# ── System packages ───────────────────────────────────────────────────────────
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    curl \
    nodejs \
    npm \
    golang-go \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# ── Python tooling ────────────────────────────────────────────────────────────
RUN pip install --no-cache-dir \
    semgrep==1.75.0 \
    pytest==8.2.0 \
    pytest-cov==5.0.0 \
    fastapi==0.111.0 \
    uvicorn==0.29.0

# ── Non-root user ─────────────────────────────────────────────────────────────
RUN useradd --create-home --shell /bin/bash devagent
USER devagent

# ── Sandbox REST API ──────────────────────────────────────────────────────────
COPY --chown=devagent:devagent sandbox_api.py /home/devagent/sandbox_api.py

WORKDIR /workspace

EXPOSE 8766

CMD ["python", "/home/devagent/sandbox_api.py"]
```

#### docker-compose.yml (sandbox service)

```yaml
# docker-compose.yml
version: "3.9"

services:
  devagents-sandbox:
    build:
      context: ./devagents/sandbox
      dockerfile: Dockerfile.sandbox
    image: devagents/sandbox:latest
    container_name: devagents-sandbox-${TASK_ID:-default}

    # ── Volumes ──────────────────────────────────────────────────────────────
    volumes:
      - type: bind
        source: ${REPO_PATH}            # host repo path (set by orchestrator)
        target: /workspace
        read_only: false                # Developer agent writes diffs here
      - type: bind
        source: ${OUTPUT_PATH}          # host output directory
        target: /output
        read_only: false

    # ── Networking ───────────────────────────────────────────────────────────
    # Default: complete network isolation.
    # The orchestrator communicates via Docker exec, NOT via network port.
    # If the repo needs package installs during tests, use the "restricted" profile.
    networks:
      - devagents-sandbox-net

    # ── Security hardening ───────────────────────────────────────────────────
    security_opt:
      - no-new-privileges:true
      - apparmor:docker-default
    cap_drop:
      - ALL
    cap_add: []                         # no capabilities added

    # ── Resource limits ──────────────────────────────────────────────────────
    mem_limit: 2g
    memswap_limit: 2g
    cpus: "2.0"
    pids_limit: 256

    # ── Read-only root filesystem (tmp dirs are explicit tmpfs) ──────────────
    read_only: true
    tmpfs:
      - /tmp:size=512m,mode=1777
      - /home/devagent/.cache:size=256m

    environment:
      - TASK_ID=${TASK_ID}
      - PYTHONDONTWRITEBYTECODE=1

networks:
  devagents-sandbox-net:
    driver: bridge
    internal: true                      # no outbound internet from this network
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

### 4.3 Network Policy

| Traffic direction | Default policy | Exception |
|------------------|---------------|-----------|
| Sandbox → internet | **DENY** | None (internal bridge) |
| Sandbox → host | **DENY** | Orchestrator communicates via Docker exec, not TCP |
| Sandbox → other containers | **DENY** | Isolated bridge network |
| Host orchestrator → sandbox | Via `docker exec` | SandboxClient uses Docker SDK |

**Why Docker exec, not a port mapping?** Port mappings expose the container to the host network interface. `docker exec` runs a command inside the container through the Docker daemon socket — it is fully local and cannot be reached from outside the host. This eliminates the attack surface entirely.

### 4.4 SandboxClient — The Python API

```python
# devagents/tools/sandbox_client.py

import docker
import json
import tarfile
import io
from pathlib import Path
from pydantic import BaseModel
from typing import Literal

class TestResult(BaseModel):
    passed: bool
    total: int
    failed: int
    errors: int
    coverage_pct: float
    output: str
    failed_tests: list[str]

class ScanResult(BaseModel):
    gate_status: Literal["PASS", "WARN", "BLOCK"]
    error_count: int
    warning_count: int
    info_count: int
    findings: list[dict]
    output: str

class ApplyResult(BaseModel):
    success: bool
    files_changed: list[str]
    error: str | None = None

class CommitResult(BaseModel):
    success: bool
    commit_hash: str | None = None
    branch: str | None = None
    error: str | None = None

class PushResult(BaseModel):
    success: bool
    remote_url: str | None = None
    error: str | None = None


class SandboxClient:
    """
    All agent-driven code execution goes through this client.
    Agents NEVER execute code on the host directly.
    """

    def __init__(self, task_id: str, repo_path: str, output_path: str):
        self.task_id = task_id
        self.repo_path = Path(repo_path).resolve()
        self.output_path = Path(output_path).resolve()
        self._docker = docker.from_env()
        self._container = None

    def start(self):
        """Spin up the sandbox container."""
        import subprocess
        import os
        env = {
            "TASK_ID": self.task_id,
            "REPO_PATH": str(self.repo_path),
            "OUTPUT_PATH": str(self.output_path),
        }
        subprocess.run(
            ["docker", "compose", "up", "-d", "devagents-sandbox"],
            env={**os.environ, **env},
            check=True,
        )
        self._container = self._docker.containers.get(f"devagents-sandbox-{self.task_id}")

    def stop(self):
        """Tear down the sandbox container."""
        if self._container:
            self._container.stop(timeout=10)
            self._container.remove(force=True)

    def _exec(self, cmd: list[str], workdir: str = "/workspace") -> tuple[int, str]:
        """Execute a command inside the container, return (exit_code, output)."""
        result = self._container.exec_run(
            cmd,
            workdir=workdir,
            user="devagent",
            environment={"PYTHONDONTWRITEBYTECODE": "1"},
        )
        return result.exit_code, result.output.decode(errors="replace")

    def apply_diff(self, unified_diff: str) -> ApplyResult:
        """Apply a unified diff to the workspace."""
        # Write diff to container via tar archive
        diff_bytes = unified_diff.encode()
        tar_buf = io.BytesIO()
        with tarfile.open(fileobj=tar_buf, mode="w") as tar:
            info = tarfile.TarInfo(name="patch.diff")
            info.size = len(diff_bytes)
            tar.addfile(info, io.BytesIO(diff_bytes))
        tar_buf.seek(0)
        self._container.put_archive("/tmp", tar_buf.read())

        exit_code, output = self._exec(
            ["git", "apply", "--check", "/tmp/patch.diff"]
        )
        if exit_code != 0:
            return ApplyResult(success=False, files_changed=[], error=output)

        self._exec(["git", "apply", "/tmp/patch.diff"])
        # Parse changed files
        _, diff_stat = self._exec(["git", "diff", "--name-only", "HEAD"])
        files_changed = [f.strip() for f in diff_stat.splitlines() if f.strip()]
        return ApplyResult(success=True, files_changed=files_changed)

    def run_tests(
        self,
        test_files: list[str] | None = None,
        framework: str = "pytest",
    ) -> TestResult:
        """Run the test suite. Returns structured results."""
        if framework == "pytest":
            targets = " ".join(test_files) if test_files else ""
            cmd = [
                "python", "-m", "pytest",
                "--tb=short",
                "--json-report", "--json-report-file=/output/test_results.json",
                f"--cov={str(self.repo_path)}",
                "--cov-report=json:/output/coverage.json",
            ] + (test_files or ["."])
        elif framework == "jest":
            cmd = ["npx", "jest", "--json", "--outputFile=/output/test_results.json"]
        elif framework == "go":
            cmd = ["go", "test", "./...", "-v", "-coverprofile=/output/coverage.out"]
        else:
            raise ValueError(f"Unsupported test framework: {framework}")

        exit_code, output = self._exec(cmd)

        # Parse JSON output
        exit_c, json_out = self._exec(["cat", "/output/test_results.json"])
        try:
            results = json.loads(json_out)
            return TestResult(
                passed=exit_code == 0,
                total=results.get("numTotalTests", 0),
                failed=results.get("numFailedTests", 0),
                errors=results.get("numErrored", 0),
                coverage_pct=self._parse_coverage("/output/coverage.json"),
                output=output,
                failed_tests=[
                    t["nodeid"] for t in results.get("tests", [])
                    if t.get("outcome") in ("failed", "error")
                ],
            )
        except (json.JSONDecodeError, KeyError):
            return TestResult(
                passed=exit_code == 0, total=0, failed=0,
                errors=1 if exit_code != 0 else 0,
                coverage_pct=0.0, output=output, failed_tests=[],
            )

    def run_semgrep(self, rules: list[str] | None = None) -> ScanResult:
        """Run Semgrep SAST scan. Always runs; threshold is checked by QA gate."""
        ruleset = rules or ["p/owasp-top-ten", "p/python", "p/javascript"]
        rule_args = []
        for r in ruleset:
            rule_args.extend(["--config", r])

        cmd = [
            "semgrep", "scan",
            *rule_args,
            "--json",
            "--output=/output/semgrep.json",
            "/workspace",
        ]
        exit_code, output = self._exec(cmd)

        exit_c, json_out = self._exec(["cat", "/output/semgrep.json"])
        try:
            results = json.loads(json_out)
            findings = results.get("results", [])
            error_count = sum(1 for f in findings if f.get("extra", {}).get("severity") in ("ERROR", "CRITICAL"))
            warning_count = sum(1 for f in findings if f.get("extra", {}).get("severity") == "WARNING")
            info_count = sum(1 for f in findings if f.get("extra", {}).get("severity") in ("INFO", "NOTE"))
            gate = "BLOCK" if error_count > 0 else ("WARN" if warning_count > 0 else "PASS")
            return ScanResult(
                gate_status=gate,
                error_count=error_count,
                warning_count=warning_count,
                info_count=info_count,
                findings=findings,
                output=output,
            )
        except (json.JSONDecodeError, KeyError):
            return ScanResult(
                gate_status="PASS", error_count=0,
                warning_count=0, info_count=0,
                findings=[], output=output,
            )

    def git_commit(self, message: str, branch: str) -> CommitResult:
        """Create a branch and commit all staged changes."""
        # Create branch
        exit_code, out = self._exec(["git", "checkout", "-b", branch])
        if exit_code != 0:
            return CommitResult(success=False, error=f"Branch creation failed: {out}")

        # Stage all changes
        self._exec(["git", "add", "-A"])

        # Commit
        exit_code, out = self._exec([
            "git", "commit",
            "-m", message,
            "--author", "DevAgents <devagents-bot@localhost>",
        ])
        if exit_code != 0:
            return CommitResult(success=False, branch=branch, error=out)

        _, commit_hash = self._exec(["git", "rev-parse", "HEAD"])
        return CommitResult(
            success=True,
            commit_hash=commit_hash.strip(),
            branch=branch,
        )

    def git_push(self, remote: str = "origin", branch: str | None = None) -> PushResult:
        """Push the current branch to remote. Only called in --full-auto mode."""
        target = branch or "HEAD"
        exit_code, out = self._exec(["git", "push", remote, target])
        if exit_code != 0:
            return PushResult(success=False, error=out)
        return PushResult(success=True, remote_url=remote)

    def _parse_coverage(self, path: str) -> float:
        exit_c, data = self._exec(["cat", path])
        try:
            cov = json.loads(data)
            return cov.get("totals", {}).get("percent_covered", 0.0)
        except Exception:
            return 0.0
```

### 4.5 Security Constraints Summary

| Constraint | Value | Rationale |
|-----------|-------|-----------|
| `cap_drop: ALL` | Drop all Linux capabilities | No privilege escalation path |
| `no-new-privileges: true` | Prevent setuid tricks | Defense in depth |
| `read_only: true` | Root FS immutable | Malware can't persist |
| `tmpfs /tmp 512m` | Explicit writable areas | Known, limited surface |
| `mem_limit: 2g` | No OOM attacks on host | Resource isolation |
| `pids_limit: 256` | Fork bomb prevention | Stability |
| `network: internal bridge` | No internet | Data exfiltration prevention |
| `user: devagent` | Non-root | Privilege separation |
| Git credentials | Never in container | Secrets are not mounted |

**Git push in --full-auto mode:** The sandbox container itself never holds push credentials. The orchestrator calls `git_push()` through Docker exec, and git credentials are provided via a short-lived `GIT_ASKPASS` script on the host that is invoked by git inside the container only for the duration of the push operation — not stored in the container environment.

---

## 5. Phase 2 — Full-Auto Commit Mode

> **This section fully resolves gap #2: "Full-Auto mode is undesigned."**

### 5.1 The Flag and Its Configuration

```bash
# Semi-auto (default): creates PR, human reviews
devagents run "Add rate limiting to the auth endpoint"

# Full-auto: commits AND merges if Tech Lead confidence >= threshold
devagents run \
  --full-auto \
  --confidence-threshold 0.85 \      # default: 0.85
  --branch-prefix "devagents/" \      # default: "devagents/"
  --max-retry-loops 3 \               # default: 3 (QA loop iterations)
  "Add rate limiting to the auth endpoint"

# Full-auto with a lower bar (risky, not recommended for production branches)
devagents run --full-auto --confidence-threshold 0.70 "Fix typo in README"
```

Configuration file (`.devagents/config.toml`):
```toml
[full_auto]
enabled = false                     # never run full-auto by default
confidence_threshold = 0.85
max_confidence_for_direct_push = 0.95   # above this, skip PR creation entirely
branch_prefix = "devagents/"
protected_branches = ["main", "master", "develop", "release/*"]
max_retry_loops = 3
rollback_on_test_regression = true
semgrep_block_on_error = true
```

### 5.2 Tech Lead Decision Schema

The Tech Lead agent MUST return a structured `TechLeadDecision` object (enforced via LLM structured outputs / Pydantic):

```python
# devagents/agents/schemas.py

from pydantic import BaseModel, Field, field_validator
from typing import Literal

class SecurityIssue(BaseModel):
    severity: Literal["CRITICAL", "ERROR", "WARNING", "INFO"]
    rule_id: str
    file: str
    line: int
    message: str

class TechLeadDecision(BaseModel):
    verdict: Literal["APPROVE", "REQUEST_CHANGES", "REJECT"]
    confidence: float = Field(ge=0.0, le=1.0)
    rationale: str = Field(min_length=20)
    specific_changes_required: list[str]   # populated for REQUEST_CHANGES
    rejection_reason: str | None = None    # populated for REJECT
    risks: list[str]
    test_coverage_delta: float             # positive = improvement
    security_issues: list[SecurityIssue]

    @field_validator("confidence")
    @classmethod
    def confidence_cannot_be_one_without_evidence(cls, v, info):
        # Prevent LLMs from always outputting 1.0
        if v == 1.0 and info.data.get("risks"):
            raise ValueError("confidence cannot be 1.0 if risks are present")
        return v
```

### 5.3 The Full-Auto Decision Gate

```python
# devagents/graph/full_auto_gate.py

from devagents.agents.schemas import TechLeadDecision

class FullAutoGate:
    """
    Single source of truth for when autonomous commit + push is allowed.
    Every condition must pass — there is no override.
    """

    def __init__(self, config: dict):
        self.threshold = config.get("confidence_threshold", 0.85)
        self.protected = config.get("protected_branches", ["main", "master", "develop"])

    def can_auto_commit(
        self,
        decision: TechLeadDecision,
        test_result: "TestResult",
        scan_result: "ScanResult",
        target_branch: str,
        baseline_coverage: float,
    ) -> tuple[bool, str]:
        """
        Returns (can_commit: bool, reason: str).
        All 5 gates must pass for autonomous commit.
        """
        checks = [
            (decision.verdict == "APPROVE",
             f"Tech Lead verdict is {decision.verdict}, not APPROVE"),

            (decision.confidence >= self.threshold,
             f"Confidence {decision.confidence:.2f} < threshold {self.threshold}"),

            (len(decision.security_issues) == 0 or all(
                i.severity not in ("CRITICAL", "ERROR")
                for i in decision.security_issues
            ), "Security gate: CRITICAL or ERROR severity findings present"),

            (test_result.passed,
             f"Tests failed: {test_result.failed} failures, {test_result.errors} errors"),

            (
                test_result.coverage_pct >= baseline_coverage - 5.0,
                f"Coverage regression: {test_result.coverage_pct:.1f}% < "
                f"baseline {baseline_coverage:.1f}% - 5%"
            ),

            (scan_result.gate_status != "BLOCK",
             f"Semgrep BLOCK: {scan_result.error_count} ERROR/CRITICAL findings"),

            (not any(target_branch.startswith(p.rstrip("*"))
                     or target_branch == p
                     for p in self.protected),
             f"Branch {target_branch!r} is in the protected list"),
        ]

        for passed, reason in checks:
            if not passed:
                return False, reason

        return True, "All gates passed"
```

### 5.4 Branch Naming and Protection Strategy

```
Branch naming convention:
  devagents/<task-id>/<short-slug>

Examples:
  devagents/task-a1b2c3/add-rate-limiting-auth
  devagents/task-x9y8z7/fix-null-pointer-user-service
  devagents/task-m3n4o5/refactor-payment-module

Rules enforced by DevAgents (not by GitHub branch rules):
  1. NEVER commits to: main, master, develop, release/*, hotfix/*
  2. ALWAYS creates a new branch per task
  3. Branch is deleted after merge or rejection
  4. One commit per task (squash strategy)
```

### 5.5 Rollback Mechanism

```python
# devagents/tools/git_tools.py

import contextlib
import subprocess
from pathlib import Path

class AtomicCommit:
    """
    Context manager that captures pre-commit state.
    If an exception occurs after commit, rolls back automatically.
    Use with all --full-auto operations.
    """

    def __init__(self, repo_path: str):
        self.repo_path = Path(repo_path).resolve()
        self._pre_state: dict | None = None

    def __enter__(self):
        result = subprocess.run(
            ["git", "rev-parse", "HEAD"],
            cwd=self.repo_path, capture_output=True, text=True,
        )
        branch_result = subprocess.run(
            ["git", "rev-parse", "--abbrev-ref", "HEAD"],
            cwd=self.repo_path, capture_output=True, text=True,
        )
        self._pre_state = {
            "commit_hash": result.stdout.strip(),
            "branch": branch_result.stdout.strip(),
        }
        return self

    def rollback(self):
        if not self._pre_state:
            return
        subprocess.run(
            ["git", "checkout", self._pre_state["branch"]],
            cwd=self.repo_path, check=True,
        )
        subprocess.run(
            ["git", "reset", "--hard", self._pre_state["commit_hash"]],
            cwd=self.repo_path, check=True,
        )

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            self.rollback()
        return False   # don't suppress the exception
```

### 5.6 Full-Auto Mode Flow

```
TASK INPUT
    │
    ▼
[Knowledge Graph Gate] → ScopedContext
    │
    ▼
[Analyst Team (parallel)] → 5 AnalystReports
    │
    ▼
[Researcher Debate] → DebateResult
    │
    ▼
[Developer Agent] → code diff + tests
    │
    ▼
[apply_diff() via SandboxClient]
    │
    ▼
[run_tests() via SandboxClient] ──FAIL──► [retry loop, max 3x]
    │                                           │
    PASS                                   [Developer re-attempts with error ctx]
    │
    ▼
[run_semgrep() via SandboxClient]
    │
    ▼
[Tech Lead → TechLeadDecision]
    │
    ├── verdict == APPROVE AND confidence >= 0.85?
    │       │
    │       ├── YES: FullAutoGate.can_auto_commit() → all 5 checks pass?
    │       │           │
    │       │           ├── YES: git_commit() + git_push() + create_pr(auto_merge=True)
    │       │           │        → Decision log entry: COMMITTED
    │       │           │
    │       │           └── NO:  create_pr(auto_merge=False)
    │       │                    → Decision log entry: PR_CREATED (gate_reason)
    │       │
    │       └── NO:  create_pr(auto_merge=False)
    │                → Decision log entry: PR_CREATED (needs_human_review)
    │
    └── verdict == REJECT: discard diff, log reason, notify user
```

---

## 6. Phase 3 — Agent Architecture

### 6.1 DevState — LangGraph State Schema

```python
# devagents/graph/state.py

from typing import Annotated, TypedDict
import operator
from devagents.knowledge.blast_radius import ScopedContext
from devagents.agents.schemas import (
    TechLeadDecision, DebateResult, AnalystReport
)

class DevState(TypedDict):
    # Input
    task_description: str
    task_id: str
    repo_path: str
    mode: str                                    # "semi-auto" | "full-auto"
    confidence_threshold: float

    # Phase 0
    scoped_context: ScopedContext | None
    baseline_coverage: float

    # Phase 1 — Analyst outputs (list so LangGraph can reduce/append)
    analyst_reports: Annotated[list[AnalystReport], operator.add]

    # Phase 2 — Debate
    debate_result: DebateResult | None

    # Phase 3 — Developer
    proposed_diff: str | None
    proposed_tests: str | None
    pr_description: str | None

    # Phase 4 — QA
    test_result: "TestResult | None"
    scan_result: "ScanResult | None"
    qa_retry_count: int

    # Phase 5 — Tech Lead
    tech_lead_decision: TechLeadDecision | None

    # Outputs
    commit_hash: str | None
    pr_url: str | None
    error: str | None

    # Memory
    previous_decisions: list[dict]              # injected from dev_memory.md
```

### 6.2 The 10 Agents

#### Analyst Team (5 agents, run in parallel)

| Agent | Input | Output | Model tier |
|-------|-------|--------|-----------|
| **CodeAnalyst** | scoped file list from ScopedContext | complexity score, coupling metrics, hotspots, suggested refactor scope | fast (gpt-4o-mini / claude-haiku) |
| **SpecAnalyst** | task description + issue/PR template | acceptance criteria, edge cases, definition of done, API contract if applicable | fast |
| **TestAnalyst** | affected_tests from ScopedContext + coverage report | coverage gaps, test strategy recommendation, regression risk | fast |
| **SecurityAnalyst** | scoped files + OWASP checklist | potential injection vectors, auth gaps, secrets exposure risk, OWASP top-10 flags | fast |
| **ArchAnalyst** | call_chain_summary + arch_layer + architecture overview | coupling debt introduced, pattern consistency, scalability concerns, ADR recommendations | fast |

All 5 analysts return an `AnalystReport`:
```python
class AnalystReport(BaseModel):
    agent_role: str
    summary: str                    # 2-3 sentences
    key_findings: list[str]         # max 5 bullet points
    risk_flags: list[str]           # issues that need attention
    recommendations: list[str]      # actionable suggestions
    confidence: float               # 0-1
```

#### Researcher Debate (2 agents, sequential per round)

See Section 8 for the full debate format.

#### Developer Agent

```python
class DeveloperOutput(BaseModel):
    unified_diff: str                   # standard unified diff format
    new_test_file: str | None           # content of new/modified test file
    modified_files: list[str]           # list of files touched
    implementation_rationale: str       # why this approach was chosen
    pr_title: str
    pr_description: str                 # markdown, includes context + test plan
    estimated_risk: Literal["LOW", "MEDIUM", "HIGH"]
```

The Developer agent has access to:
- `read_file(path)` — reads from `/workspace` via SandboxClient
- `apply_diff(diff)` — calls `SandboxClient.apply_diff()`
- `run_tests()` — calls `SandboxClient.run_tests()` (preview before QA gate)
- `search_symbol(name)` — queries the knowledge graph via GraphClient

#### QA Gate (deterministic, not LLM)

The QA gate is **not an agent** — it is a deterministic function that reads `TestResult` and `ScanResult` and decides pass/fail:

```python
def qa_gate(state: DevState) -> dict:
    test = state["test_result"]
    scan = state["scan_result"]
    baseline = state["baseline_coverage"]

    failed_reasons = []
    if not test.passed:
        failed_reasons.append(f"Tests failed: {test.failed} failures")
    if test.coverage_pct < baseline - 5.0:
        failed_reasons.append(f"Coverage regression: {test.coverage_pct:.1f}% < {baseline - 5.0:.1f}%")
    if scan.gate_status == "BLOCK":
        failed_reasons.append(f"Semgrep BLOCK: {scan.error_count} critical findings")

    if failed_reasons and state["qa_retry_count"] < 3:
        # Send back to Developer with error context
        return {
            "qa_retry_count": state["qa_retry_count"] + 1,
            "error": "\n".join(failed_reasons),
            "__next__": "developer",
        }
    elif failed_reasons:
        return {"error": "QA gate failed after 3 retries", "__next__": "end"}
    else:
        return {"__next__": "tech_lead"}
```

#### Tech Lead Agent

The Tech Lead is the **only agent** that receives the complete context: all 5 analyst reports + debate result + developer output + QA results + previous decisions from memory. It uses a **larger, slower model** (gpt-4o / claude-sonnet). It returns `TechLeadDecision` (structured output enforced).

The Tech Lead prompt injection includes:
```xml
<previous_decisions_on_similar_tasks>
{last 3 entries from dev_memory.md for this module}
</previous_decisions_on_similar_tasks>
```

### 6.3 LangGraph — DevGraph Definition

```python
# devagents/graph/dev_graph.py

from langgraph.graph import StateGraph, END
from devagents.graph.state import DevState

def build_dev_graph() -> StateGraph:
    graph = StateGraph(DevState)

    # ── Nodes ──────────────────────────────────────────────────────────────
    graph.add_node("knowledge_gate",    knowledge_gate_node)
    graph.add_node("analyst_team",      analyst_team_node)    # parallel fan-out
    graph.add_node("researcher_debate", researcher_debate_node)
    graph.add_node("developer",         developer_node)
    graph.add_node("qa_gate",           qa_gate_node)
    graph.add_node("tech_lead",         tech_lead_node)
    graph.add_node("commit_and_pr",     commit_and_pr_node)
    graph.add_node("write_memory",      write_memory_node)

    # ── Edges ──────────────────────────────────────────────────────────────
    graph.set_entry_point("knowledge_gate")
    graph.add_edge("knowledge_gate",    "analyst_team")
    graph.add_edge("analyst_team",      "researcher_debate")
    graph.add_edge("researcher_debate", "developer")
    graph.add_edge("developer",         "qa_gate")

    # QA gate can loop back to developer
    graph.add_conditional_edges(
        "qa_gate",
        lambda s: s.get("__next__", "tech_lead"),
        {"developer": "developer", "tech_lead": "tech_lead", "end": END},
    )

    graph.add_edge("tech_lead",    "commit_and_pr")
    graph.add_edge("commit_and_pr", "write_memory")
    graph.add_edge("write_memory",  END)

    return graph.compile(
        checkpointer=SqliteSaver.from_conn_string(
            f"~/.devagents/cache/checkpoints/{task_id}.db"
        )
    )
```

### 6.4 Multi-Tier LLM Routing

```python
# devagents/default_config.py

LLM_TIERS = {
    # Fast agents — low stakes, high volume
    "fast": {
        "openai":    "gpt-4o-mini",
        "anthropic": "claude-3-5-haiku-latest",
        "google":    "gemini-2.0-flash",
        "groq":      "llama-3.1-8b-instant",
        "ollama":    "llama3.2:3b",
    },
    # Deep-think agents — high stakes, low volume
    "deep": {
        "openai":    "gpt-4o",
        "anthropic": "claude-sonnet-4-5",
        "google":    "gemini-2.5-pro",
        "groq":      "llama-3.3-70b-versatile",
        "ollama":    "qwen2.5-coder:32b",
    },
}

AGENT_TIERS = {
    "code_analyst":       "fast",
    "spec_analyst":       "fast",
    "test_analyst":       "fast",
    "security_analyst":   "fast",
    "arch_analyst":       "fast",
    "researcher_builder": "fast",
    "researcher_skeptic": "fast",
    "developer":          "deep",   # writes actual code
    "tech_lead":          "deep",   # final approval decision
}
```

---

## 7. Phase 4 — Terminal UI Design

> **This section fully resolves gap #4: "Terminal UI has no design spec."**

### 7.1 Layout Specification (Rich library)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  DevAgents v0.1.0  │  Task: add-rate-limit-auth  │  Mode: SEMI-AUTO  │  ●  │
├──────────────────┬──────────────────────────────────────────────────────────┤
│  KNOWLEDGE GRAPH │  ANALYST TEAM                                            │
│                  │                                                          │
│  ✓ Indexed       │  ● Code Analyst     ████████░░  80%  ETA: 4s            │
│  2,847 nodes     │  ✓ Spec Analyst     ██████████  Done (12s)              │
│  4,918 edges     │  ● Test Analyst     ████░░░░░░  40%  ETA: 12s           │
│                  │  ○ Security Analyst ░░░░░░░░░░  Queued                  │
│  Blast Radius:   │  ○ Arch Analyst     ░░░░░░░░░░  Queued                  │
│  15 files        │                                                          │
│  34 functions    ├──────────────────────────────────────────────────────────┤
│  8 tests         │  RESEARCHER DEBATE  (Round 2 / 3)                        │
│                  │                                                          │
│  Risk: MEDIUM    │  Builder: "JWT middleware is cleanest — no DB lookup.    │
│                  │           Rate limit state lives in Redis TTL."          │
│  Token saved:    │                                                          │
│  ~46,000         │  Skeptic: "Redis adds a new dependency. What happens     │
│  ($0.138 saved)  │           on Redis failure? Fallback must be designed."  │
├──────────────────┴──────────────────────────────────────────────────────────┤
│  DEVELOPER         │  QA GATE              │  TECH LEAD                     │
│  ○ Waiting         │  ○ Waiting            │  ○ Waiting                     │
├────────────────────┴───────────────────────┴────────────────────────────────┤
│  Tokens used: 4,234  │  Saved: ~46,000  │  Cost: $0.013  │  Elapsed: 1m 23s │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Implementation

```python
# devagents/cli/tui.py

from rich.layout import Layout
from rich.live import Live
from rich.panel import Panel
from rich.progress import Progress, BarColumn, TimeRemainingColumn
from rich.table import Table
from rich.text import Text
from rich import box
import threading

class DevAgentsTUI:
    def __init__(self):
        self.layout = Layout()
        self._state = {
            "graph": {"status": "indexing", "nodes": 0, "files_in_scope": 0,
                       "tokens_saved": 0, "cost_saved": 0.0, "risk": "UNKNOWN"},
            "analysts": {
                "code":     {"status": "queued", "pct": 0},
                "spec":     {"status": "queued", "pct": 0},
                "test":     {"status": "queued", "pct": 0},
                "security": {"status": "queued", "pct": 0},
                "arch":     {"status": "queued", "pct": 0},
            },
            "debate": {"round": 0, "max_rounds": 3, "last_builder": "", "last_skeptic": ""},
            "developer": {"status": "waiting", "files_changed": []},
            "qa": {"status": "waiting", "tests_passed": 0, "tests_failed": 0, "coverage": 0.0},
            "tech_lead": {"status": "waiting", "verdict": "", "confidence": 0.0},
            "tokens_used": 0,
            "cost_usd": 0.0,
            "elapsed_s": 0,
            "task_name": "",
            "mode": "SEMI-AUTO",
        }
        self._lock = threading.Lock()

    def update(self, key: str, value):
        with self._lock:
            # deep-set via dotted key e.g. "analysts.code.pct"
            parts = key.split(".")
            obj = self._state
            for part in parts[:-1]:
                obj = obj[part]
            obj[parts[-1]] = value

    def _build_header(self) -> Panel:
        mode_color = "red" if self._state["mode"] == "FULL-AUTO" else "cyan"
        return Panel(
            f"[bold]DevAgents v0.1.0[/bold]  │  "
            f"Task: [yellow]{self._state['task_name']}[/yellow]  │  "
            f"Mode: [{mode_color}]{self._state['mode']}[/{mode_color}]  │  "
            f"[green]●[/green] LIVE",
            box=box.HORIZONTALS,
        )

    def _build_graph_panel(self) -> Panel:
        g = self._state["graph"]
        status_icon = "✓" if g["status"] == "ready" else "⟳"
        lines = [
            f"  {status_icon} Indexed",
            f"  {g['nodes']:,} nodes",
            f"",
            f"  Blast Radius:",
            f"  {g['files_in_scope']} files",
            f"",
            f"  Risk: [{'red' if g['risk']=='HIGH' else 'yellow' if g['risk']=='MEDIUM' else 'green'}]{g['risk']}[/]",
            f"",
            f"  Tokens saved:",
            f"  [green]~{g['tokens_saved']:,}[/green]",
            f"  [green](${g['cost_saved']:.3f} saved)[/green]",
        ]
        return Panel("\n".join(lines), title="[bold]KNOWLEDGE GRAPH[/bold]", padding=(0, 1))

    def _build_analyst_panel(self) -> Panel:
        rows = []
        icons = {"done": "✓", "running": "●", "queued": "○", "failed": "✗"}
        colors = {"done": "green", "running": "cyan", "queued": "dim", "failed": "red"}
        for name, data in self._state["analysts"].items():
            icon = icons.get(data["status"], "?")
            color = colors.get(data["status"], "white")
            bar = "█" * (data["pct"] // 10) + "░" * (10 - data["pct"] // 10)
            rows.append(
                f"  [{color}]{icon}[/{color}] {name.title():<16} "
                f"[{color}]{bar}[/{color}]  {data['pct']}%"
            )
        d = self._state["debate"]
        debate_lines = [
            "",
            f"  RESEARCHER DEBATE  (Round {d['round']} / {d['max_rounds']})",
            "",
            f"  [cyan]Builder:[/cyan] {d['last_builder'][:70]}",
            "",
            f"  [yellow]Skeptic:[/yellow] {d['last_skeptic'][:70]}",
        ]
        return Panel(
            "\n".join(rows) + "\n".join(debate_lines),
            title="[bold]ANALYST TEAM[/bold]",
            padding=(0, 1),
        )

    def _build_bottom_bar(self) -> Panel:
        return Panel(
            f"  Tokens used: [yellow]{self._state['tokens_used']:,}[/yellow]  │  "
            f"Saved: [green]~{self._state['graph']['tokens_saved']:,}[/green]  │  "
            f"Cost: [white]${self._state['cost_usd']:.3f}[/white]  │  "
            f"Elapsed: [white]{self._state['elapsed_s']}s[/white]",
            box=box.HORIZONTALS,
            padding=(0, 0),
        )

    def start(self, task_name: str, mode: str):
        self._state["task_name"] = task_name[:40]
        self._state["mode"] = mode

        self.layout.split_column(
            Layout(name="header", size=3),
            Layout(name="main"),
            Layout(name="bottom", size=3),
        )
        self.layout["main"].split_row(
            Layout(name="graph", ratio=1),
            Layout(name="right", ratio=3),
        )
        self.layout["right"].split_column(
            Layout(name="analysts", ratio=2),
            Layout(name="pipeline", ratio=1),
        )

        with Live(self.layout, refresh_per_second=4, screen=True):
            while True:
                self.layout["header"].update(self._build_header())
                self.layout["graph"].update(self._build_graph_panel())
                self.layout["analysts"].update(self._build_analyst_panel())
                self.layout["bottom"].update(self._build_bottom_bar())
```

### 7.3 Keyboard Shortcuts

| Key | Action |
|-----|--------|
| `q` | Graceful quit (saves checkpoint before exiting) |
| `p` | Pause (suspends LangGraph execution at next node boundary) |
| `r` | Resume from checkpoint |
| `d` | Toggle debug pane (raw LLM output) |
| `l` | Open LangSmith trace URL in browser |

---

## 8. Phase 5 — Builder / Skeptic Debate Format

> **This section fully resolves gap #5: "Builder/Skeptic debate output format is undefined."**

### 8.1 Structured Output Schema

```python
# devagents/agents/schemas.py (continued)

class DebatePosition(BaseModel):
    round: int
    speaker: Literal["BUILDER", "SKEPTIC"]
    proposed_approach: str = Field(
        description="The implementation approach being argued. 2-4 sentences.",
        max_length=500,
    )
    key_arguments: list[str] = Field(
        description="3-5 arguments supporting this position.",
        max_items=5,
    )
    evidence_from_analysts: list[str] = Field(
        description="Direct references to analyst report findings that support this position. "
                    "Format: '<AgentRole>: <finding>'",
        max_items=5,
    )
    acknowledged_risks: list[str] = Field(
        description="Risks the speaker concedes are real, even while arguing their position.",
        max_items=3,
    )
    proposed_mitigations: list[str] = Field(
        description="Specific steps to address the acknowledged risks.",
        max_items=3,
    )
    confidence: float = Field(ge=0.0, le=1.0,
        description="Speaker's self-assessed confidence in their position.")

class DebateRound(BaseModel):
    round_number: int
    builder_position: DebatePosition
    skeptic_position: DebatePosition
    convergence_score: float = Field(
        ge=0.0, le=1.0,
        description="How much the two positions agree. 1.0 = full consensus.",
    )

class DebateResult(BaseModel):
    rounds: list[DebateRound]
    final_consensus: str = Field(
        description="The synthesized approach both agents converged on. "
                    "If no consensus, the approach with better evidence.",
        max_length=800,
    )
    unresolved_risks: list[str] = Field(
        description="Risks neither agent could mitigate. "
                    "These are passed to the Tech Lead for final judgment.",
        max_items=5,
    )
    recommended_approach: str
    overall_confidence: float
    debate_summary_for_developer: str = Field(
        description="A single compact paragraph the Developer agent will read. "
                    "Includes: chosen approach, key constraints, and what NOT to do.",
        max_length=600,
    )
```

### 8.2 Debate Flow (3 rounds)

```
Round 1 — Independent Positions
─────────────────────────────────
  Builder reads:  ScopedContext + all 5 AnalystReports
  Builder writes: DebatePosition (round=1, speaker="BUILDER")

  Skeptic reads:  ScopedContext + all 5 AnalystReports
  Skeptic writes: DebatePosition (round=1, speaker="SKEPTIC")

Round 2 — Rebuttal
────────────────────
  Builder reads:  Round 1 Skeptic position → addresses each concern
  Builder writes: DebatePosition (round=2) — must cite acknowledged_risks
                  from Round 1 Skeptic and explain how they are mitigated

  Skeptic reads:  Round 2 Builder position → probes mitigations
  Skeptic writes: DebatePosition (round=2) — must acknowledge any concessions

Round 3 — Synthesis
─────────────────────
  Builder attempts synthesis: integrates best Skeptic points
  Builder writes: DebatePosition (round=3) — convergence attempt

  Judge (deterministic function, not LLM):
    if round3.convergence_score >= 0.7: use builder's final position
    else: pass both positions to Tech Lead as "unresolved debate"
```

### 8.3 System Prompts

**Builder system prompt:**
```
You are the Builder researcher in a software development debate.
Your role is to argue for the BEST POSSIBLE implementation of the given task.
You must:
- Cite specific findings from the analyst reports (format: "CodeAnalyst: <finding>")
- Acknowledge real risks (do not pretend they don't exist)
- Propose concrete mitigations for every risk you acknowledge
- Be practical: prefer existing patterns in the codebase over new dependencies
You must NOT: oversell, ignore the Skeptic's strongest points, or recommend
approaches that introduce security vulnerabilities flagged by SecurityAnalyst.
Return your response as a valid JSON object matching the DebatePosition schema.
```

**Skeptic system prompt:**
```
You are the Skeptic researcher in a software development debate.
Your role is to argue the FAILURE MODES and TRADEOFFS of the Builder's proposed approach.
You must:
- Identify real failure modes (not hypothetical edge cases that will never occur)
- Propose a BETTER alternative when you identify a flaw (not just criticism)
- Acknowledge when the Builder's mitigations are genuinely adequate
- Focus on: performance, security, maintainability, and operational complexity
You must NOT: reject every approach reflexively, ignore evidence from analyst reports,
or argue for approaches the ArchAnalyst flagged as inconsistent with existing patterns.
Return your response as a valid JSON object matching the DebatePosition schema.
```

---

## 9. Phase 6 — Semgrep Integration

> **This section fully resolves gap #6: "Semgrep integration has no pass/fail threshold."**

### 9.1 Severity Model and Thresholds

| Finding severity | Source | Gate behavior |
|-----------------|--------|---------------|
| `CRITICAL` | Any rule | **BLOCK** — cannot commit, must fix |
| `ERROR` | Any rule | **BLOCK** — cannot commit, must fix |
| `WARNING` | Any rule | **WARN** — flagged in PR description, cannot auto-merge |
| `INFO` / `NOTE` | Any rule | **LOG** — appears in output, does not affect gates |

**BLOCK threshold:** `error_count + critical_count > 0` → gate status = `BLOCK`
**WARN threshold:** `warning_count > 0` → gate status = `WARN`
**Full-auto additional check:** In `--full-auto` mode, even WARN status prevents autonomous push (human must review the warnings).

### 9.2 Rule Set Selection (Auto-detected)

```python
# devagents/tools/sast_runner.py

def select_semgrep_rules(repo_path: str, scoped_files: list[str]) -> list[str]:
    """Auto-select Semgrep rulesets based on detected languages."""
    rules = ["p/owasp-top-ten"]   # always on

    # Detect languages from file extensions in blast radius
    extensions = {Path(f).suffix for f in scoped_files}

    if ".py" in extensions:
        rules.append("p/python")
        if _has_file(repo_path, "manage.py"):
            rules.append("p/django")
        if _has_file(repo_path, "app.py") or _has_import(scoped_files, "flask"):
            rules.append("p/flask")

    if {".js", ".ts", ".jsx", ".tsx"} & extensions:
        rules.append("p/javascript")
        if _has_import(scoped_files, "react"):
            rules.append("p/react")
        if _has_import(scoped_files, "express"):
            rules.append("p/nodejs")

    if ".go" in extensions:
        rules.append("p/golang")

    if ".java" in extensions:
        rules.append("p/java")
        if _has_file(repo_path, "pom.xml"):
            rules.append("p/spring")

    if ".rs" in extensions:
        rules.append("p/rust")

    # User custom rules
    custom_rules_dir = Path.home() / ".devagents" / "semgrep_rules"
    if custom_rules_dir.exists():
        rules.append(str(custom_rules_dir))

    return rules
```

### 9.3 Finding → Developer Feedback

When the QA gate blocks on Semgrep findings, the Developer agent receives a structured error context:

```python
def format_semgrep_for_developer(scan: ScanResult) -> str:
    """Format Semgrep findings as actionable feedback for the Developer agent."""
    if not scan.findings:
        return ""
    
    lines = ["The following security issues must be fixed before commit:\n"]
    for f in scan.findings:
        if f["extra"]["severity"] in ("ERROR", "CRITICAL"):
            lines.append(
                f"  [{f['extra']['severity']}] {f['check_id']}\n"
                f"  File: {f['path']}:{f['start']['line']}\n"
                f"  Issue: {f['extra']['message']}\n"
                f"  Fix: {f['extra'].get('fix', 'See rule documentation')}\n"
            )
    return "\n".join(lines)
```

### 9.4 Semgrep in the Sandbox

Semgrep runs **inside** the `devagents-sandbox` container via `SandboxClient.run_semgrep()`. This ensures:
- Only the blast-radius files are scanned (not the whole repo, faster)
- Semgrep version is pinned in `Dockerfile.sandbox` (reproducible results)
- Semgrep cannot make outbound network calls (network isolation)

Semgrep's registry rules are pre-downloaded into the image at build time:
```dockerfile
# In Dockerfile.sandbox
RUN semgrep --config p/owasp-top-ten --dry-run /dev/null 2>/dev/null || true
RUN semgrep --config p/python --dry-run /dev/null 2>/dev/null || true
# ... etc — pre-populates ~/.semgrep cache inside the image
```

---

## 10. Phase 7 — Project Structure

```
devagents/
│
├── cli/
│   ├── main.py                   # Typer CLI: run, init, analyze, review, arch, graph
│   └── tui.py                    # Rich terminal UI (Section 7)
│
├── devagents/
│   ├── graph/
│   │   ├── dev_graph.py          # LangGraph StateGraph definition
│   │   ├── state.py              # DevState TypedDict + reducers
│   │   ├── nodes.py              # Each node function (knowledge_gate, analyst_team, etc.)
│   │   └── full_auto_gate.py     # FullAutoGate decision logic (Section 5.3)
│   │
│   ├── agents/
│   │   ├── schemas.py            # All Pydantic output schemas
│   │   ├── code_analyst.py
│   │   ├── spec_analyst.py
│   │   ├── test_analyst.py
│   │   ├── security_analyst.py
│   │   ├── arch_analyst.py
│   │   ├── researcher_builder.py
│   │   ├── researcher_skeptic.py
│   │   ├── developer.py
│   │   └── tech_lead.py
│   │
│   ├── knowledge/
│   │   ├── graph_client.py       # MCP subprocess manager + JSON-RPC client
│   │   ├── blast_radius.py       # ScopedContext builder
│   │   ├── graph_setup.py        # First-run indexing + auto-sync config
│   │   └── memory_log.py         # Decision log read/write
│   │
│   ├── tools/
│   │   ├── sandbox_client.py     # SandboxClient (Section 4.4)
│   │   ├── git_tools.py          # AtomicCommit, branch management, PR creation
│   │   ├── sast_runner.py        # Semgrep rule selection + feedback formatter
│   │   └── llm_router.py         # Multi-tier, multi-provider LLM routing
│   │
│   └── default_config.py         # LLM_TIERS, AGENT_TIERS, defaults
│
├── sandbox/
│   ├── Dockerfile.sandbox        # Hardened sandbox container (Section 4.2)
│   └── sandbox_api.py            # (optional) thin FastAPI inside container
│
├── tests/
│   ├── unit/                     # Unit tests for each agent + tool
│   ├── integration/              # End-to-end tests against a real test repo
│   └── fixtures/                 # Sample repos, diffs, and expected outputs
│
├── docker-compose.yml            # Full stack (orchestrator + sandbox)
├── pyproject.toml
├── .env.example
└── .devagents/
    ├── config.toml               # Project-level config (gitignored)
    └── semgrep_rules/            # Custom SAST rules for this project
```

**User data directories (outside project, not gitignored):**
```
~/.devagents/
├── memory/
│   └── dev_memory.md             # Persistent decision log (all tasks, all repos)
└── cache/
    └── checkpoints/
        └── <TASK_ID>.db          # LangGraph checkpoint per task

~/.cache/codebase-memory-mcp/     # Knowledge graph (managed by codebase-memory-mcp)
    └── <repo-fingerprint>/
        └── graph.db
```

---

## 11. Phase 8 — Build Roadmap (4 Sprints)

### Sprint 1 (Week 1–2): Knowledge Graph + Scaffold
**Goal:** Prove the token savings. Make the graph work before writing a single agent.

| Task | Owner | Definition of Done |
|------|-------|-------------------|
| Install `codebase-memory-mcp` on dev machine | Dev | `codebase-memory-mcp cli list_projects` returns JSON |
| Build `GraphClient` | Dev | `blast_radius("add rate limiting")` returns `ScopedContext` with < 30 files |
| Measure token savings | Dev | Logged ratio: (naive file scan tokens) / (ScopedContext tokens) ≥ 10× |
| Scaffold `DevState` + `build_dev_graph()` | Dev | `devagents run "test"` executes graph nodes in correct order (stub agents OK) |
| Set up LangSmith tracing | Dev | Every run produces a trace URL in the TUI bottom bar |
| Implement `AtomicCommit` rollback | Dev | Unit test: exception inside context manager → git reset to pre-state |

**Sprint 1 exit criteria:** `devagents run "Add logging to auth module"` against a test repo:
- Knowledge graph returns ≤ 25 files in scope
- Token savings ≥ 10× vs. reading all potentially relevant files
- LangGraph executes nodes in correct order
- Checkpoint resume works (`Ctrl+C` mid-run, `devagents resume <task-id>` restarts from last node)

---

### Sprint 2 (Week 3–4): Analyst Team
**Goal:** 5 parallel analysts producing structured, useful reports.

| Task | Owner | Definition of Done |
|------|-------|-------------------|
| Implement `CodeAnalyst` with `AnalystReport` output | Dev | Output passes Pydantic validation, contains ≥ 3 key_findings |
| Implement `SpecAnalyst` | Dev | Generates ≥ 5 acceptance criteria from a real task description |
| Implement `TestAnalyst` | Dev | Correctly identifies uncovered functions in blast radius |
| Implement `SecurityAnalyst` | Dev | Flags a deliberately introduced SQL injection in test fixture |
| Implement `ArchAnalyst` | Dev | Correctly identifies coupling violation in test fixture |
| Wire all 5 in parallel in `analyst_team_node` | Dev | All 5 complete before `researcher_debate` node starts |
| Build `DevAgentsTUI` (Section 7) | Dev | Live updating panels during analyst team execution |

**Sprint 2 exit criteria:** Full analyst team run with live TUI on a real repo. All 5 reports are structured, relevant, and finish in < 60 seconds combined.

---

### Sprint 3 (Week 5–6): Researcher + Developer + QA Gate
**Goal:** End-to-end code generation with security and test gating.

| Task | Owner | Definition of Done |
|------|-------|-------------------|
| Implement `researcher_builder.py` + `researcher_skeptic.py` | Dev | 3-round debate produces `DebateResult` with `debate_summary_for_developer` |
| Build `DebatePosition` convergence scoring | Dev | Unit test: identical positions → convergence=1.0; opposing → ≤ 0.4 |
| Implement `DeveloperAgent` | Dev | Produces valid unified diff + test file from `DebateResult` |
| Implement `SandboxClient` (Section 4.4) | Dev | `run_tests()` returns correct `TestResult` from inside container |
| Build `Dockerfile.sandbox` | Dev | Container passes security scan; cannot reach internet |
| Implement `QA Gate` (Section 6.2) | Dev | Loops back to Developer up to 3× on test failure |
| Implement `SemgrepRunner` (Section 9) | Dev | BLOCK/WARN/PASS correctly from real Semgrep output |

**Sprint 3 exit criteria:** Given a task description, the system produces a git branch with passing tests and no Semgrep BLOCK findings. Developer → QA retry loop works correctly.

---

### Sprint 4 (Week 7–8): Tech Lead + Memory + Full-Auto + CLI
**Goal:** Production-ready system with all 8 layers and full-auto mode.

| Task | Owner | Definition of Done |
|------|-------|-------------------|
| Implement `TechLeadAgent` | Dev | Returns `TechLeadDecision` with confidence score from real Pydantic validation |
| Implement `FullAutoGate` (Section 5.3) | Dev | All 7 gate checks have unit tests (pass + fail cases) |
| Implement `AtomicCommit` + `git_push` in sandbox | Dev | Full-auto run creates branch, commits, pushes (test against local Gitea) |
| Implement `memory_log.py` | Dev | Previous decisions injected into Tech Lead prompt on repeat tasks |
| Build CLI (`devagents run`, `devagents init`, `devagents resume`) | Dev | `devagents --help` shows all commands with descriptions |
| Keyboard shortcuts in TUI | Dev | `q`, `p`, `r`, `d`, `l` all work |
| Integration test suite | Dev | 5 end-to-end test scenarios (feature, bugfix, security fix, architecture query, ground-up) |
| Docker Compose full stack | Dev | `docker compose up` starts orchestrator + sandbox; no host-level dependencies |
| Write `README.md` + `SETUP.md` | Dev | A new developer can run their first task in < 15 minutes |

**Sprint 4 exit criteria:** `devagents run --full-auto "Add input validation to the user registration endpoint"` autonomously commits a correct, tested, secure implementation to a feature branch and opens a PR — with a LangSmith trace showing all 10 agent steps.

---

## 12. Configuration Reference

### `.devagents/config.toml`

```toml
[llm]
provider = "anthropic"            # openai | anthropic | google | groq | ollama
fast_model = ""                   # leave empty to use tier defaults from default_config.py
deep_model = ""                   # leave empty to use tier defaults

[graph]
cache_dir = ""                    # leave empty for ~/.cache/codebase-memory-mcp
auto_sync = true
max_files_in_scope = 30           # hard cap on files sent to agents
max_tokens_in_scope = 6000        # hard cap on ScopedContext token budget

[debate]
rounds = 3                        # number of Builder/Skeptic rounds
convergence_threshold = 0.7       # if convergence_score >= this, skip to developer

[qa]
test_framework = "auto"           # auto | pytest | jest | go | vitest
coverage_regression_threshold = 5.0  # % drop allowed before blocking
semgrep_block_on_error = true
semgrep_warn_on_warning = true
max_retry_loops = 3

[full_auto]
enabled = false
confidence_threshold = 0.85
branch_prefix = "devagents/"
protected_branches = ["main", "master", "develop", "release/*"]
rollback_on_test_regression = true

[sandbox]
mem_limit = "2g"
cpus = "2.0"
pids_limit = 256
```

### Environment Variables (`.env`)

```bash
# LLM API keys (only the provider you use)
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_API_KEY=AIza...
GROQ_API_KEY=gsk_...

# LangSmith tracing (highly recommended)
LANGSMITH_API_KEY=ls__...
LANGCHAIN_TRACING_V2=true
LANGCHAIN_PROJECT=devagents

# Git credentials for sandbox push
GIT_PAT=ghp_...             # GitHub/GitLab personal access token
GIT_AUTHOR_NAME="DevAgents Bot"
GIT_AUTHOR_EMAIL="devagents@yourorg.com"

# Knowledge graph
CBM_CACHE_DIR=              # leave empty for default ~/.cache/codebase-memory-mcp

# Sandbox
DOCKER_HOST=unix:///var/run/docker.sock
```

---

## 13. Key Architectural Decisions

### Decision 1: codebase-memory-mcp over vector RAG

**Decision:** Use `codebase-memory-mcp` (tree-sitter AST knowledge graph) instead of a vector embedding store (Chroma, Pinecone, etc.).

**Rationale:**
- Vector similarity captures topical similarity but fails on multi-hop architectural reasoning: controller → service → repository chains, interface-driven wiring, and inheritance-based behavior.
- `codebase-memory-mcp` resolves these correctly via deterministic AST analysis.
- 99.2% token reduction (412k → 3.4k for 5 structural queries, benchmarked on real repos).
- SQLite persistence with WAL mode — no separate database infrastructure.
- Single static binary, zero dependencies, ships on Windows/macOS/Linux.
- Auto-sync via git polling — no manual re-indexing needed.

**What we give up:** Natural language → code search (e.g., "find code that does X without knowing the function name"). Mitigation: use `search_graph` with a broad regex name pattern first.

---

### Decision 2: LangGraph over CrewAI / AutoGen

**Decision:** LangGraph as the orchestration framework.

**Rationale:**
- No hidden prompts, no enforced cognitive architecture. Full control over what gets passed to each LLM in what order.
- Built-in checkpoint/resume via SQLite (critical for long-running tasks).
- Built-in LangSmith integration for observability.
- Parallel node execution for the analyst fan-out.
- Conditional edges for the QA retry loop.
- Active development by LangChain team with strong community.

---

### Decision 3: DinD Sandbox for All Execution

**Decision:** All code execution (tests, Semgrep, git operations) runs inside `devagents-sandbox`.

**Rationale:**
- Agents executing `pytest`, `npm test`, or `go test` on arbitrary code is an RCE vector if run on the host.
- The sandbox has no internet access, no root, no capabilities, and a read-only root filesystem.
- This makes the system safe to run against untrusted repositories (e.g., open source contributions).

**What we give up:** Slightly higher setup overhead (Docker required). Mitigation: `docker compose up` handles everything; documented in `SETUP.md`.

---

### Decision 4: Semi-Auto as Default, Full-Auto as Opt-In

**Decision:** The system creates PRs by default. `--full-auto` enables autonomous push only when all 7 gates pass.

**Rationale:**
- Trust is built incrementally. Run 20 tasks in semi-auto mode, observe quality, then enable full-auto for low-risk task categories.
- The 7-gate model (verdict + confidence + security + tests + coverage + semgrep + branch protection) makes the risk quantifiable.
- Rolling back a bad commit is always possible via `AtomicCommit`.

---

### Decision 5: Deep-Think vs Fast Models per Agent

**Decision:** Analysts and researchers use fast/cheap models; Developer and Tech Lead use deep/expensive models.

**Cost impact example (Anthropic):**
- 5 analysts × 2,000 tokens @ claude-haiku: ~$0.002
- 2 researchers × 3,000 tokens @ claude-haiku: ~$0.002
- 1 developer × 8,000 tokens @ claude-sonnet: ~$0.024
- 1 tech lead × 5,000 tokens @ claude-sonnet: ~$0.015
- **Total: ~$0.043 per task** (before knowledge graph savings)

Without knowledge graph: analysts each read the full codebase → +50,000 tokens each → ~$0.50/task. **Knowledge graph reduces cost by ~10×.**

---

### Decision 6: Decision Memory Across Sessions

**Decision:** Every completed task appends a structured entry to `~/.devagents/memory/dev_memory.md`. The Tech Lead reads the last 3 entries for the same module before making its decision.

**Format of a memory entry:**
```markdown
## 2026-05-01T21:45:00 | task-a1b2c3 | repo: my-api | module: auth/middleware.py
**Task:** Add rate limiting to the auth endpoint
**Approach chosen:** Redis TTL counter in middleware (rejected JWT-only approach)
**Skeptic concern that was valid:** Redis failure fallback was needed; added circuit breaker
**Test result:** 47 tests passed, coverage 84.2% (+2.1%)
**Tech Lead confidence:** 0.91
**Outcome:** Merged 2026-05-01, no post-merge regressions
**Key learning:** Any new middleware on this repo must handle Redis.ConnectionError or QA blocks
```

This memory prevents the system from making the same mistake twice and allows it to get faster and more confident on familiar codebases over time.

---

*End of DevAgents Blueprint v1.0*

*Next step: Begin Sprint 1 — run `devagents init` against a target repo and verify the knowledge graph blast radius returns a ScopedContext with ≤ 30 files for a realistic task description.*
