# Sandbox Configuration

## Overview

Sandboxes provide **runtime enforcement** of constraints. While guardrails and tool policies define what an agent *should* do, sandboxes define what an agent *can* do at the operating system and runtime level. They are the last line of defense.

The key insight from Anthropic's work on Claude Code security: **the best security comes from making the safe path the default path, not from relying on the agent to always make the right decision.**

---

## Sandbox Dimensions

### 1. File System Restrictions

Control which directories and files the agent can read, write, and execute.

```yaml
filesystem:
  # Base workspace - the root of all allowed operations
  workspace_root: "/path/to/project"

  readable_paths:
    - "${workspace_root}/**"          # Full project tree
    - "/usr/local/lib/**"             # System libraries (read-only)
    - "${HOME}/.config/git/**"        # Git config (read-only)

  writable_paths:
    - "${workspace_root}/src/**"      # Source code
    - "${workspace_root}/tests/**"    # Test files
    - "${workspace_root}/docs/**"     # Documentation
    - "${workspace_root}/output/**"   # Generated outputs
    - "/tmp/agent-*"                  # Temp files (namespaced)

  executable_paths:
    - "/usr/local/bin/*"              # Installed tools
    - "${workspace_root}/node_modules/.bin/*"  # Project tools
    - "${workspace_root}/scripts/*"   # Project scripts

  forbidden_paths:
    - "${HOME}/.ssh/**"               # SSH keys
    - "${HOME}/.aws/**"               # AWS credentials
    - "${HOME}/.config/gcloud/**"     # GCP credentials
    - "**/.env"                       # Environment files (write)
    - "**/.env.*"                     # Environment variants (write)
    - "**/credentials/**"             # Credential directories
    - "**/secrets/**"                 # Secret directories
    - "/etc/**"                       # System config
    - "/var/**"                       # System data
```

### 2. Network Restrictions

Control which hosts, ports, and protocols the agent can access.

```yaml
network:
  # DNS resolution
  allow_dns: true

  # Outbound connections
  allowed_hosts:
    - "github.com"
    - "api.github.com"
    - "raw.githubusercontent.com"
    - "registry.npmjs.org"
    - "pypi.org"
    - "api.notion.com"
    - "www.googleapis.com"
    - "oauth2.googleapis.com"
    - "slack.com"
    - "api.slack.com"
    - "api.anthropic.com"
    - "api.openai.com"
    # Documentation sites
    - "*.readthedocs.io"
    - "docs.github.com"
    - "developer.mozilla.org"
    - "nodejs.org"

  blocked_hosts:
    - "*.onion"                       # Tor hidden services
    - "localhost"                      # Prevent SSRF to local services
    - "127.0.0.1"
    - "0.0.0.0"
    - "169.254.169.254"               # Cloud metadata endpoint
    - "metadata.google.internal"       # GCP metadata

  allowed_ports:
    - 443   # HTTPS
    - 80    # HTTP (documentation sites)
    - 22    # SSH (for git operations only)

  blocked_ports:
    - 3306  # MySQL
    - 5432  # PostgreSQL
    - 6379  # Redis
    - 27017 # MongoDB
    # (block direct database access; use approved APIs instead)

  protocols:
    allow_https: true
    allow_http: true    # For documentation sites; prefer HTTPS
    allow_ssh: true     # For git operations
    allow_ftp: false
    allow_websocket: false  # Unless specifically needed
```

### 3. Execution Restrictions

Control what processes the agent can spawn and how they behave.

```yaml
execution:
  # Process limits
  max_concurrent_processes: 5
  max_process_runtime: "300s"    # 5 minutes per command
  max_output_size: "10MB"        # Per command output

  # Allowed commands (allowlist approach)
  allowed_commands:
    # Version control
    - "git"
    # Package managers
    - "npm"
    - "npx"
    - "yarn"
    - "pnpm"
    - "pip"
    - "pip3"
    - "cargo"
    # Build tools
    - "node"
    - "python"
    - "python3"
    - "tsc"
    - "esbuild"
    - "vite"
    - "webpack"
    # Testing
    - "jest"
    - "vitest"
    - "pytest"
    - "cargo test"
    # Linting & formatting
    - "eslint"
    - "prettier"
    - "black"
    - "ruff"
    - "clippy"
    # Utilities
    - "cat"
    - "head"
    - "tail"
    - "wc"
    - "sort"
    - "uniq"
    - "diff"
    - "find"
    - "grep"
    - "rg"
    - "jq"
    - "ls"
    - "pwd"
    - "echo"
    - "date"
    - "which"
    - "env"     # Read-only environment inspection

  blocked_commands:
    - "sudo"
    - "su"
    - "chmod"   # File permission changes
    - "chown"   # File ownership changes
    - "mount"
    - "umount"
    - "kill"
    - "killall"
    - "pkill"
    - "reboot"
    - "shutdown"
    - "systemctl"
    - "service"
    - "crontab"
    - "at"
    - "nohup"   # Persistent background processes
    - "screen"
    - "tmux"
    - "nc"      # Netcat - arbitrary network connections
    - "nmap"
    - "curl | sh"
    - "wget | sh"
    - "eval"
    - "exec"

  blocked_flags:
    - "--force"      # On destructive commands
    - "--hard"       # git reset --hard
    - "--no-verify"  # Skip git hooks
    - "-rf"          # rm -rf (recursive force delete)
    - "--skip-tests" # Skip test suites
```

### 4. Resource Limits

Prevent runaway processes from consuming excessive resources.

```yaml
resources:
  # Memory
  max_memory_per_process: "512MB"
  max_total_memory: "2GB"

  # CPU
  max_cpu_time_per_process: "300s"
  cpu_priority: "nice 10"           # Lower priority than user processes

  # Disk
  max_disk_write_per_session: "1GB"
  max_file_size: "100MB"
  max_files_created_per_session: 500

  # Network
  max_bandwidth: "10MB/s"
  max_connections: 20
  connection_timeout: "30s"
  request_timeout: "60s"
```

---

## Environment Profiles

Different environments require different constraint levels. Below are three standard profiles.

### Development (Least Restrictive)

For local development where the agent works on a developer's machine.

```yaml
profile: "development"
description: "Local development environment. Agent works alongside developer."

filesystem:
  workspace_root: "${PROJECT_DIR}"
  writable_paths:
    - "${workspace_root}/**"         # Full project write access
    - "/tmp/agent-*"
  forbidden_paths:
    - "${HOME}/.ssh/**"
    - "${HOME}/.aws/**"
    - "**/.env"                      # Still protect env files
    - "**/credentials/**"

network:
  mode: "allowlist_with_exceptions"
  allowed_hosts: ["*"]               # Allow all hosts
  blocked_hosts:
    - "169.254.169.254"              # Cloud metadata (always blocked)
    - "metadata.google.internal"
  allowed_ports: [80, 443, 22, 3000, 5173, 8080]  # Include dev server ports

execution:
  max_concurrent_processes: 10
  max_process_runtime: "600s"        # 10 minutes (builds can be slow)
  allowed_commands: ["*"]            # Allow all commands
  blocked_commands:
    - "sudo"
    - "su"
    - "reboot"
    - "shutdown"

resources:
  max_memory_per_process: "2GB"
  max_total_memory: "4GB"
  max_disk_write_per_session: "5GB"

approval_overrides:
  # In development, these actions are auto-approved
  auto_approve:
    - "npm install"
    - "pip install"
    - "git commit"
    - "running tests"
    - "starting dev server"
  still_require_approval:
    - "git push"
    - "sending emails"
    - "modifying config files"
    - "deleting files"
```

### Staging (Moderate)

For CI/CD pipelines and staging environments where the agent runs tests and prepares deployments.

```yaml
profile: "staging"
description: "CI/CD and staging. Agent runs tests, builds, prepares deployments."

filesystem:
  workspace_root: "${CI_PROJECT_DIR}"
  writable_paths:
    - "${workspace_root}/src/**"
    - "${workspace_root}/tests/**"
    - "${workspace_root}/build/**"
    - "${workspace_root}/dist/**"
    - "/tmp/agent-*"
  forbidden_paths:
    - "${HOME}/.ssh/**"
    - "${HOME}/.aws/**"
    - "**/.env*"
    - "**/credentials/**"
    - "**/secrets/**"
    - "${workspace_root}/.git/config"  # Protect git remote config

network:
  mode: "strict_allowlist"
  allowed_hosts:
    - "github.com"
    - "api.github.com"
    - "registry.npmjs.org"
    - "pypi.org"
    - "api.anthropic.com"
  blocked_hosts:
    - "169.254.169.254"
    - "metadata.google.internal"
    - "localhost"
    - "127.0.0.1"
  allowed_ports: [443, 22]           # HTTPS and SSH only

execution:
  max_concurrent_processes: 5
  max_process_runtime: "300s"
  allowed_commands:
    - "git"
    - "npm"
    - "node"
    - "tsc"
    - "jest"
    - "eslint"
    - "prettier"
  blocked_commands:
    - "sudo"
    - "curl"
    - "wget"
    - "ssh"     # No interactive SSH
    - "kill"

resources:
  max_memory_per_process: "1GB"
  max_total_memory: "2GB"
  max_disk_write_per_session: "2GB"

approval_overrides:
  auto_approve:
    - "running tests"
    - "running linters"
    - "building project"
  still_require_approval:
    - "Everything else"
```

### Production (Most Restrictive)

For production environments. The agent should have minimal access, primarily for monitoring and diagnostics.

```yaml
profile: "production"
description: "Production environment. Read-only with minimal diagnostics."

filesystem:
  workspace_root: "${APP_DIR}"
  writable_paths:
    - "/tmp/agent-diagnostics-*"     # Diagnostic output only
  readable_paths:
    - "${workspace_root}/logs/**"    # Application logs
    - "${workspace_root}/config/**"  # Config files (read-only)
  forbidden_paths:
    - "**/.env*"
    - "**/credentials/**"
    - "**/secrets/**"
    - "**/database/**"
    - "**/backups/**"

network:
  mode: "strict_allowlist"
  allowed_hosts:
    - "api.anthropic.com"            # Only the AI API
  blocked_hosts:
    - "*"                            # Block everything else
  allowed_ports: [443]               # HTTPS only

execution:
  max_concurrent_processes: 2
  max_process_runtime: "60s"
  allowed_commands:
    - "cat"
    - "head"
    - "tail"
    - "grep"
    - "wc"
    - "ls"
    - "pwd"
    - "date"
    - "df"
    - "free"
    - "ps"       # Process inspection only
    - "top -bn1" # Single snapshot
  blocked_commands:
    - "*"         # Block everything not in allowed list

resources:
  max_memory_per_process: "256MB"
  max_total_memory: "512MB"
  max_disk_write_per_session: "100MB"

approval_overrides:
  auto_approve: []                   # Nothing is auto-approved
  still_require_approval:
    - "All actions"
```

---

## Implementing Sandboxes

### Option 1: Claude Code Built-in Sandbox

Claude Code provides built-in sandboxing through its `--sandbox` flag:

```bash
# Read-only mode: no file writes allowed
claude --sandbox read-only

# Workspace-write mode: writes only within workspace
claude --sandbox workspace-write

# Full access mode (dangerous, requires explicit flag)
claude --sandbox danger-full-access
```

Map to profiles:
- **Production** -> `read-only`
- **Staging** -> `workspace-write`
- **Development** -> `workspace-write` (default; avoid `danger-full-access`)

### Option 2: Docker-based Sandbox

For stronger isolation, run the agent in a Docker container:

```dockerfile
FROM node:20-slim

# Create non-root user
RUN useradd -m -s /bin/bash agent

# Set working directory
WORKDIR /workspace

# Copy project files
COPY --chown=agent:agent . .

# Drop capabilities
USER agent

# Resource limits applied at runtime
# docker run --memory=2g --cpus=2 --network=agent-net ...
```

```yaml
# docker-compose.yml
services:
  agent:
    build: .
    user: "agent"
    read_only: true
    tmpfs:
      - /tmp:size=100M
    volumes:
      - ./src:/workspace/src
      - ./tests:/workspace/tests
    networks:
      - agent-net
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: "2.0"

networks:
  agent-net:
    driver: bridge
    internal: false  # Set to true for no internet access
```

### Option 3: macOS Sandbox Profile

For macOS environments (like the current setup):

```scheme
;; agent-sandbox.sb - macOS sandbox profile
(version 1)
(deny default)

;; Allow reading from project workspace
(allow file-read*
  (subpath "/Users/watanabe/project"))

;; Allow writing to specific directories
(allow file-write*
  (subpath "/Users/watanabe/project/src")
  (subpath "/Users/watanabe/project/tests")
  (subpath "/Users/watanabe/project/output")
  (subpath "/tmp/agent"))

;; Allow network to specific hosts
(allow network-outbound
  (remote tcp "github.com:443")
  (remote tcp "api.github.com:443")
  (remote tcp "registry.npmjs.org:443"))

;; Allow process execution
(allow process-exec
  (literal "/usr/local/bin/node")
  (literal "/usr/local/bin/git")
  (literal "/usr/local/bin/npm"))

;; Deny everything else by default
```

---

## Sandbox Selection Guide

| Scenario | Profile | Sandbox Method | Rationale |
|----------|---------|---------------|-----------|
| Interactive development session | Development | Claude Code `workspace-write` | Balance of productivity and safety |
| Automated code review | Staging | Docker (read-only filesystem) | No writes needed for review |
| CI/CD pipeline | Staging | Docker (workspace-write) | Isolated from host, controlled writes |
| Production monitoring | Production | Docker (read-only, no network) | Absolute minimum access |
| Content generation | Development | Claude Code `workspace-write` | Write to output directories only |
| Security audit | Staging | Docker (read-only, full network) | Read access to inspect, network to verify |

## Monitoring and Alerts

Every sandbox should generate logs for:

1. **Denied operations**: Any action blocked by the sandbox. These indicate either a misconfiguration (if the action was legitimate) or a security boundary being tested (if the action was not legitimate).
2. **Near-limit usage**: When resource consumption reaches 80% of limits.
3. **Unusual patterns**: Rapid file creation, large network transfers, repeated denied operations.

```yaml
monitoring:
  log_denied_operations: true
  log_resource_usage: true
  alert_thresholds:
    denied_operations_per_minute: 5     # Alert if agent hits sandbox walls frequently
    memory_usage_percent: 80
    disk_usage_percent: 80
    network_bytes_per_minute: "50MB"
  alert_destination: "stderr"           # Or webhook, Slack, etc.
```
