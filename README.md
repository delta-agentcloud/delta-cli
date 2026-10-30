# delta-cli

Command-line tool for the Delta Agent Cloud, used to manage Delta Sandbox GPU/CPU compute workloads and invoke Science tools.

## Installation

### Option 1: npx one-click install (recommended)

```bash
npx @delta-agentcloud/cli@latest install
```

This command performs, in order:
1. **Global install** of `@delta-agentcloud/cli` (includes the Go binary)
2. **Select AI tool platforms** (multi-select): generic agent dir `~/.agents/skills/` (Codex/Cursor/OpenCode), Claude Code `~/.claude/skills/`, OpenCode native `~/.config/opencode/skills/`, Cursor `~/.cursor/skills/`, Memento-S dev mode `~/memento_s/skills/`
3. **Install AI Skills** into the selected platforms' skills directories
4. **Initialize config** automatically writing the default server address (no manual base_url entry)
5. **Authenticate** `delta-cli auth login` (interactive; skipped when not a TTY)

AI agents can also install directly with:

> Install delta-cli for me: https://github.com/delta-agentcloud/delta-cli

### Uninstall

```bash
npx @delta-agentcloud/cli@latest uninstall
```

Cleans up the global package, AI Skills from all known platforms, and the config file.

### Upgrade

```bash
# Upgrade CLI + AI Skills
delta-cli upgrade

# Only check whether an update is available (do not upgrade)
delta-cli upgrade --check

# Only upgrade the CLI binary
delta-cli upgrade --cli-only

# Only upgrade AI Skills
delta-cli upgrade --skills-only
```

Upgrading the CLI pulls the latest version via `npm install -g @delta-agentcloud/cli@latest`; upgrading Skills reinstalls AI Skills via `npx skills add`.

### Option 2: Download the binary directly

Download the archive for your platform from [GitHub Releases](https://github.com/delta-agentcloud/delta-cli/releases) (below, `latest` points to the newest version):

```bash
# macOS (Apple Silicon)
curl -L -o delta-cli.tar.gz https://github.com/delta-agentcloud/delta-cli/releases/latest/download/delta-cli-darwin-arm64.tar.gz
tar -xzf delta-cli.tar.gz
sudo mv delta-cli-darwin-arm64 /usr/local/bin/delta-cli
rm delta-cli.tar.gz

# Linux (AMD64)
curl -L -o delta-cli.tar.gz https://github.com/delta-agentcloud/delta-cli/releases/latest/download/delta-cli-linux-amd64.tar.gz
tar -xzf delta-cli.tar.gz
chmod +x delta-cli-linux-amd64
sudo mv delta-cli-linux-amd64 /usr/local/bin/delta-cli
rm delta-cli.tar.gz

# Windows (AMD64, PowerShell)
Invoke-WebRequest -Uri https://github.com/delta-agentcloud/delta-cli/releases/latest/download/delta-cli-windows-amd64.zip -OutFile delta-cli.zip
Expand-Archive -Path delta-cli.zip -DestinationPath . -Force
Move-Item -Path delta-cli-windows-amd64.exe -Destination delta-cli.exe
Remove-Item -Path delta-cli.zip
# Put delta-cli.exe in a directory on your PATH
```

### Option 3: One-line install script (works without npm)

The install script **auto-detects the environment**:
- **npm present** → installs the `@delta-agentcloud/cli` npm package from the official npm registry
- **no npm** → downloads the binary directly (PowerShell + curl on Windows; bash + curl + tar on macOS/Linux, both built in), **no Node.js/npm required**

```bash
# Linux / macOS
curl -L https://raw.githubusercontent.com/delta-agentcloud/delta-cli/main/install.sh | bash

# Windows (PowerShell)
Invoke-RestMethod -Uri https://raw.githubusercontent.com/delta-agentcloud/delta-cli/main/install.ps1 | Invoke-Expression
```

The script downloads the binary directly from GitHub Releases and verifies the SHA-256. Use the `DELTA_CLI_VERSION` environment variable to pin a version.

### Option 4: npm

```bash
npm install -g @delta-agentcloud/cli
```

`postinstall` downloads the binary for the current platform:

1. The mirror specified by the `DELTA_CLI_MIRROR` environment variable (if set; must be HTTPS and allow the full-prefix format, e.g. `https://your-mirror`)
2. The GitHub source `https://github.com/delta-agentcloud/delta-cli/releases/download/...` (fallback)

If the download fails during install:

- Run `delta-cli` once; it will automatically retry the download.
- Or set a local archive path before installing:

```bash
DELTA_CLI_ARCHIVE=/path/to/delta-cli-linux-amd64.tar.gz npm install -g @delta-agentcloud/cli
```

Common environment variables:

| Variable | Description |
|----------|-------------|
| `DELTA_CLI_MIRROR` | Download mirror root, e.g. `https://your-mirror`. The script expands it to the full-prefix format (`<mirror>/https://github.com/...`). Must be HTTPS and allowlisted. |
| `DELTA_CLI_MIRROR_ALLOWLIST` | Additional allowed hosts, comma-separated. |
| `DELTA_CLI_ARCHIVE` | Path to a pre-downloaded `.tar.gz` / `.zip` to skip the network download. |
| `DELTA_CLI_SKIP_POSTINSTALL` | Set to `1` to skip the binary download at install time; `delta-cli` retries at runtime. |
| `DELTA_CLI_FATAL_ON_ERROR` | Exit non-zero on install failure (default: warn only; useful for debugging). |
| `DELTA_CLI_DOWNLOAD_TIMEOUT` | Max wait time per download source (ms, default 120000). |

## Quick Start

### 1. Initialize configuration

```bash
delta-cli config init
```

The server address needs no manual entry: it defaults to `https://delta-infra-nacos.laddery.uk` (an existing config keeps its own base_url); the CLI only prompts interactively for a Bearer Token (optional — you can also configure it later with `auth login`).

To point at a self-hosted server, specify it once via flag or environment variable:

```bash
# Via flag
delta-cli config init \
  --base-url http://your-server/api/v1 \
  --token your-token

# Via environment variable (API Key or Token, pick one)
export DELTA_INFRA_BASE_URL=http://your-server/api/v1
export DELTA_INFRA_API_KEY=your-api-key        # or DELTA_INFRA_TOKEN
delta-cli config init
```

The config file is stored at `~/.delta-infra/config.json` (mode 0600).

### 2. Authentication

Two credential types are supported: **API Key** and **Bearer Token** (API Key takes priority).

**Get an API Key**: sign in to the Delta Agent Cloud **web console** and generate a key on the **API Keys** page.

```bash
# API Key (recommended)
delta-cli auth login --apikey <your-api-key>

# Token
delta-cli auth login --token <your-token>

# Interactive login (paste the Key/Token; the CLI detects the format automatically)
delta-cli auth login

# Show auth status
delta-cli auth status
```

You can also provide credentials via environment variables: `DELTA_INFRA_API_KEY=<key>` / `DELTA_INFRA_TOKEN=<token>`.

### 3. Sandbox lifecycle

```bash
# Create a sandbox with a GPU (--max-life defaults to 30 minutes; raise it for long tasks)
# ⚠️ Lifecycle timing (see skills/delta-sandbox/references/lifecycle.md):
#   - Private compute (OpenSandbox): the clock starts when the create request is accepted, and the
#     image pull / readiness time counts towards max-life; the platform reclaims the sandbox at expiry.
#   - Third-party compute (AutoDL/RunPod): the clock starts when the sandbox is ready (started_at),
#     and the server-side cleanup reclaims it at max-life plus up to 10 minutes of grace.
#   - Auto-reclaim (including an early platform reclaim) does NOT write back to OSS; pull the
#     artifacts or call `sandbox finish` explicitly if you need to keep them. `sandbox status`
#     returns expires_at / reclaim_source / termination_reason, and a finished sandbox stays queryable.
# ⚠️ Resource semantics: `--memory` is the container CPU memory TOTAL (one per sandbox); `--gpu-mem` is the memory PER CARD (MiB) —
# `--gpu 2 --gpu-mem 8000` = 8000MiB per card, 16000MiB total (the private OpenSandbox/HAMi backend writes it straight to the HAMi per-card annotation;
# the cloud RunPod backend treats --gpu-mem as total memory ÷ card count). Per-card memory limit is visible via `sandbox resources` (memory_mib).
# ⚠️ Resource semantics: `--shm-size` is the container /dev/shm size (default only 64Mi). PyTorch DataLoader(num_workers>0)/
# multiprocessing, Chromium-based kernels etc. fail with "No space left on device"/"Bus error" when shared memory is insufficient; raise it then.
# The unit is mandatory (4G/512M are normalized to 4Gi/512Mi; a bare number is rejected by the server) and must not exceed --memory.
# `--devices` requests host devices (comma-separated, e.g. kvm → /dev/kvm inside the container, for nested virtualization): a privilege escalation,
# `--ready-timeout` caps how long the CREATE step waits for the sandbox to become ready (seconds, default 600, range 10-1800):
# a cold node fetching a large image layer (e.g. 2.9GB) often takes longer than 60s, so raising it avoids error_code=SandboxReadyTimeout.
# Applies to all providers (private compute / AutoDL / RunPod). Distinct from --max-life (post-create TTL).
# rejected by the server when the allowlist is disabled (never silently ignored). Both apply to private compute only; third-party compute ignores them.
delta-cli sandbox create \
  --image-name image.yangtzeailab.com/opensandbox/pytorch-cuda13:latest \
  --cpu 4 --memory 16Gi --gpu 1 \
  --max-life 120

# For DataLoader multiprocessing / nested virtualization: raise /dev/shm and request /dev/kvm (shm must not exceed memory)
delta-cli sandbox create \
  --image-name image.yangtzeailab.com/opensandbox/pytorch-cuda13:latest \
  --cpu 4 --memory 8Gi --shm-size 4Gi --devices kvm \
  --max-life 120

# Sandbox that must live long and not be auto-cleaned: add --no-auto-cleanup
# (it disables both the platform TTL and the server-side cleanup; only explicit kill/finish destroys it)
delta-cli sandbox create \
  --image-name image.yangtzeailab.com/opensandbox/pytorch-cuda13:latest \
  --cpu 4 --memory 16Gi --gpu 1 \
  --no-auto-cleanup

# Run a command (short tasks run synchronously; use run-bg instead of run for long tasks)
delta-cli sandbox run <sandbox_id> \
  --command "<command>" \
  --timeout 3600

# v1.0.85+ sandbox run uses an SSE real-time stream: stdout carries raw data: frames (no trailing envelope);
# exit_code / log_file / stdout_tail / duration_seconds are in the complete frame
delta-cli sandbox run <sandbox_id> --command "echo '{\"status\":\"ok\",\"result\":\"done\"}'" --timeout 60

# Automation/orchestration: --format json (same as --json-result) prints exactly ONE JSON envelope,
# whose data holds execution_id / finished / exit_code / stdout_tail / stderr_tail /
# stdout_size / stderr_size / log_file / duration_seconds / result_summary.
# The field names match the complete frame; on failure it is still a single object:
# {"ok":false,"error":{...},"data":{...}} and the process exits non-zero.
delta-cli sandbox run <sandbox_id> --command "python run.py" --timeout 600 --format json

# Background task + wait + unified result
delta-cli sandbox run-bg <sandbox_id> --command "python train.py" --timeout 3600 --wait --format json

# Read a result file
delta-cli sandbox read <sandbox_id> --path /workspace/result.json

# Destroy the sandbox (the destroy keeps running server-side; the response carries request_id)
delta-cli sandbox kill <sandbox_id>

# Idempotent create retries + reconciliation afterwards
delta-cli sandbox create --image-name <image name> --idempotency-key retry-001
delta-cli sandbox operation <request_id>
```

## Command list

### Global commands

| Command | Description |
|---------|-------------|
| `delta-cli config init` | Initialize the config file |
| `delta-cli config show` | Show current config (sensitive fields masked) |
| `delta-cli config set <key> <value>` | Set a config value |
| `delta-cli config remove <key>` | Remove a config value (supports `token`, `apikey`) |
| `delta-cli auth login` | Interactive login (detects API Key / Token automatically) |
| `delta-cli auth login --apikey <key>` | API Key authentication (recommended) |
| `delta-cli auth login --token <token>` | Token authentication |
| `delta-cli auth status` | Show auth status |
| `delta-cli upgrade` | Upgrade the CLI and AI Skills to the latest version |
| `delta-cli upgrade --check` | Check whether an update is available |
| `delta-cli upgrade --cli-only` | Only upgrade the CLI binary |
| `delta-cli upgrade --skills-only` | Only upgrade AI Skills |
| `delta-cli usage` | Show current usage and remaining quota |
| `delta-cli --version` | Show version |

### Sandbox commands

> **Tip**: `sandbox list` (list sandbox instances) differs from `sandbox ls <id>` (list a directory); don't confuse them.

> **Paths inside the sandbox**: a path that points *inside* the sandbox (`--path`, the remote part of `write-multiple --entry`, `upload --target`, `pull --source`, `run`/`run-bg --log-file`) is prefixed with the sandbox working directory when it is relative, and absolute paths are used as-is; paths that point at the *local* machine (`write --source`, `upload --source`, `pull --target`, `read --output`) are local and are never rewritten.

| Command | Description |
|---------|-------------|
| `delta-cli sandbox list` | List the current user's sandbox instances |
| `delta-cli sandbox providers` | Show available compute backends |
| `delta-cli sandbox resources [--provider opensandbox\|autodl]` | Show remaining requestable resources (GPU/memory/cores) per backend |
| `delta-cli sandbox images` | Show the available image list (each entry has `image_name`/`tags`/`description`/`supports_shared_workspace`; `supports_shared_workspace=true` means the compute supports a shared workspace (opensandbox/autodl) and can pair with `--shared-workspace`; the actual seed/write-back is decided at create time by probing for rclone inside the image) |
| `delta-cli sandbox recommend --cpu N --memory XGi` | Get a resource configuration recommendation |
| `delta-cli sandbox create --image-name <image name> [--idempotency-key <key>] [--shm-size XGi] [--devices kvm] [--no-auto-cleanup] [--shared-workspace=true\|false]` | Create a sandbox container (`--image` is the legacy alias; **`--idempotency-key`**: retrying with the same key replays the original sandbox instead of creating a second one, and a different request under the same key is rejected; the response carries `request_id`/`operation_status`/`replayed`; `--no-auto-cleanup` disables both the platform TTL and the server-side cleanup, so only an explicit kill/finish destroys it; `--shared-workspace` defaults to true: enables the shared workspace (seed/write-back determined at create time by probing for rclone in the image; skipped when absent, data not persistent across sandboxes); false skips seed/write-back, suited to fast tasks; `--shm-size` raises /dev/shm (default 64Mi, **must carry a Gi/Mi unit** and be ≤ `--memory`, common for PyTorch DataLoader/multiprocessing); `--devices` is a comma-separated list of host device tokens (`kvm` → /dev/kvm, nested virtualization), **a privilege escalation** the server **explicitly rejects** when the allowlist is off; `--shm-size`/`--devices` apply to private compute only, third-party compute ignores them) |
| `delta-cli sandbox extend <id> --max-life N` | **Dynamically extend** a running sandbox's max-life (N must be greater than the current expiry and ≤10080; shortening is not accepted — use `kill`/`finish` for an early reclaim); without `--max-life` it queries the current lifecycle (elapsed/remaining/expiry); `status` output includes a `data.lifecycle` summary (elapsed/remaining/expiry; omitted when the lifecycle API is unavailable) |
| `delta-cli sandbox connect <id>` | Connect to a sandbox |
| `delta-cli sandbox status <id>` | Show sandbox status; returns `expires_at` / `reclaim_source`, and a finished sandbox stays queryable with `termination_reason` / `termination_remark` / `terminated_at` (e.g. `max_life_exceeded` / `platform_gone` / `user_killed` / `user_finished`) |
| `delta-cli sandbox finish <id> [--no-wait]` | Save results and destroy; the destroy is a **server-side task** and the response carries `request_id`/`operation_status` (`--no-wait` returns `pending` immediately and the task continues after a disconnect — reconcile it with `sandbox operation <request_id>`) |
| `delta-cli sandbox kill <id> [--no-wait]` | Destroy a sandbox (without saving results); the response carries `request_id`/`operation_status`, so after an interrupted call or a network error the real terminal state can be queried with `sandbox operation` |
| `delta-cli sandbox operation [request_id] [--sandbox-id <id>] [--operation create\|destroy] [--limit N]` | Query the server-side operation ledger: with a `request_id` it fetches that single operation, otherwise it lists the recent create/destroy records (terminal state, time, reason) for billing and resource reconciliation |
| `delta-cli sandbox run <id> --command "..." [--timeout <sec>] [--format sse\|json] [--json-result] [--summary/--no-summary] [--artifacts]` | Run a command synchronously. By default (`--format sse`) the raw SSE `data:` frames are passed through to stdout with no trailing envelope, and `exit_code`/`log_file`/`stdout_tail`/`duration_seconds` are in the `complete` frame; `--format json` (or `--json-result`) prints exactly **one** JSON envelope whose `data` holds `execution_id`/`finished`/`exit_code`/`stdout_tail`/`stderr_tail`/`stdout_size`/`stderr_size`/`log_file`/`duration_seconds`/`result_summary`, and on failure it is `{ok:false,error,data}` with a non-zero process exit code; `--summary`/`--artifacts` only apply on the legacy-server fallback path. **Exit-code semantics**: a non-zero command exit → `error.type=command_failed`, process exit code **8**; the command finished but its **exit status is unknown** (the stream ended before the `complete` frame, or only an error frame arrived) → `error.type=unknown_execution_state`, process exit code 1 — the CLI **never synthesizes a success** |
| `delta-cli sandbox run-bg <id> --command "..." [--timeout <sec>] [--wait] [--cancel-on-timeout] [--format sse\|json] [--json-result] [--summary/--no-summary] [--artifacts]` | Run a command in the background; without `--wait` it returns `{execution_id, sandbox_id}` immediately (with `--format json`, the unified result object with `finished=false`); with `--wait` it follows via SSE (raw `data:` frames to stdout, no trailing envelope); `--format json` prints exactly one envelope with the same structure as `sandbox run --format json`. **On a wait timeout** → `error.type=timeout` and a **non-zero** process exit code, with `error.details` keeping `sandbox_id`/`execution_id`/`remote_state` (`not_cancelled`/`cancelled`/`cancel_failed`) so the command can be queried or cancelled afterwards; `remote_state=not_cancelled` means only the local wait stopped and **the remote command may still be running**. The remote command is left running by default — add `--cancel-on-timeout` to cancel it (a failed cancel is reported as `cancel_failed`) |
| `delta-cli sandbox logs <id> --execution-id <eid> [--tail N --grep <pattern> --context N --max-bytes N]` | Show background logs; returns `stderr_size + stderr_tail` by default (avoids context blow-up); filter with `--tail/--grep` |
| `delta-cli sandbox cancel <id> --execution-id <eid>` | Interrupt a background command |
| `delta-cli sandbox read <id> --path <path> [--output <local path>] [--tail N] [--grep <pattern>] [--offset N] [--limit N] [--context N] [--max-bytes N] [--parse-json]` | Read a file in the container; `--output <path>` saves it locally; `--tail`/`--grep` filter the output. **A full copy (no `--tail`/`--grep`/`--offset`/`--limit`/`--max-bytes`) is byte-exact**: it goes through the chunked download path (binary-safe, sha256-verified — the envelope reports `byte_exact:true`, `integrity:"sha256"`, `digest_verified`). With a range filter the filter is applied to the **exact bytes**, so the copy is a byte-exact slice of the original (`byte_exact:true`, `integrity:"sha1"`). Printed inline, a text file gives `content` (strictly decoded, `byte_exact:true`) while a **non-UTF-8 file gives no `content` at all** — the exact original bytes are in `content_base64` (base64-decode it) and the envelope is marked `byte_exact:true`. Only a server that does not support that negotiation returns the lossy text (`integrity:"text_decoded"`, with a warning on stderr). An `--output` that names an existing local directory receives the file inside it, under its own name (the way `cp file dir/` works). |
| `delta-cli sandbox pull <id> --source <sandbox path> --target <local path> [--pattern <glob>] [--concurrency N] [--no-resume] [--no-verify]` | Pull files/dirs from the sandbox to local (mirror of `upload`; flag direction reversed: source=remote, target=local). **Directories are always pulled recursively** (`--recursive` is deprecated and no longer needed); the transfer uses the **chunked download** path: binary-safe, end-to-end **sha256** verified (`digest_verified`), **resumable** (a partial `<target>.delta.part` is reused), and **parallel** across files and chunks (`--concurrency`, default 4) — so a large file is not bound by a single command timeout. The server publishes its limits with every response (`limits`): 4 MiB chunks (16 MiB max), 100 GiB max per file, 120 s per chunk, 600 s for the whole-file digest. `--no-resume` ignores a leftover `.delta.part`; `--no-verify` skips the sandbox-side sha256 comparison. **The payload moves as raw bytes** (the server's binary endpoint, so it is not inflated by base64; older servers fall back automatically), and **the chunks must tile the whole file**: a gap or an interrupted transfer is reported as a failure, never as a successful download. When `--target` names an existing local directory, the file is written inside it under its own name (the way `cp file dir/` works) |
| `delta-cli sandbox write <id> --path <path> --source <local path>` | Write a file. **Binary-safe**: the content travels as raw bytes (the server's binary endpoint, with a base64 fallback for older servers) and the CLI declares its `size`/`sha256`, which the server verifies before writing (a mismatch is rejected); the response carries `server_sha256`/`sha256`/`size`/`size_verified`. A `--source` above 32 MiB goes through the streaming upload path instead of being read into memory |
| `delta-cli sandbox write-multiple <id> --entry <remote path>=<local path>` | Write multiple files |
| `delta-cli sandbox ls <id> --path <path>` | List a directory (a path that is a file, or is missing, is rejected with an explicit error instead of a generic provider failure) |
| `delta-cli sandbox stat <id> --path <path>` | File metadata |
| `delta-cli sandbox mv <id> --entry <src=dest>` | Move/rename |
| `delta-cli sandbox replace <id> --path <path> --old <text> --new <text>` | Replace content |
| `delta-cli sandbox chmod <id> --path <path> --mode <octal>` | Change permissions |
| `delta-cli sandbox rm <id> --path <path>` | Delete a file. **Files only**: a directory path is rejected (use `rmdir`), and a path that does not exist is an error rather than a silent success |
| `delta-cli sandbox rmdir <id> --path <dir>` | Recursively delete a directory (cannot be undone). **Directories only**: a file path is rejected and the file is left untouched, and a path that does not exist is an error |
| `delta-cli sandbox mkdir <id> --path <dir>` | Create a directory |
| `delta-cli sandbox search <id> --path <root> --pattern <glob>` | Search files (an unterminated glob such as `[` is rejected instead of silently matching nothing) |
| `delta-cli sandbox working-directory <id>` | Get the sandbox working directory path |
| `delta-cli sandbox upload <id> --source <local dir> --target <sandbox path>` | Upload a directory (tar.gz + auto-extract) |
| `delta-cli sandbox expose <id> [--port N]` | Expose a sandbox service port to the public internet (frp tunnel), default port 8700 |
| `delta-cli sandbox exposes <id>` | List the sandbox's active public port tunnels |
| `delta-cli sandbox dispose <id> --port N` | Recycle a public port tunnel (specify the port) |

### Science tools

> The Science Server is derived from `base_url` (`{base_url}/science_tool`); no separate `science_base_url` config is needed.
> The Sandbox API is likewise derived from `base_url` (`{base_url}/sandbox/api/v1`).

| Command | Description |
|---------|-------------|
| `delta-cli science list` | List all enabled science tools |
| `delta-cli science get <tool_name>` | Show details for a tool |
| `delta-cli science invoke --tool <tool_name> --endpoint <endpoint_name> [--data '{"key":"value"}'] [--params '{"key":"value"}']` | Invoke a tool endpoint |
| `delta-cli science endpoints list <tool_name>` | List all endpoints of a tool |
| `delta-cli science task submit --tool <tool> --operation <operation> --input '<json>' [--wait]` | Submit a durable Science task |
| `delta-cli science task get/list/cancel/invocations/artifacts` | Query, cancel tasks, and read invocation/artifact metadata |
| `delta-cli science file invoke --tool <tool> --endpoint <operation> --file <path>` | Invoke a controlled file/multipart operation |
| `delta-cli science file download --tool <tool> --endpoint <operation> --params '<json>' --output <path>` | Download a raw artifact and compute its SHA-256 |

`tools.name` and `tool_endpoints.name` are the only authoritative names for Science tool calls.
`science list` and `science endpoints list` show the enabled entries from the server database verbatim;
`science invoke` passes `--tool` and `--endpoint` verbatim, with no alias mapping or legacy-name compatibility.

Since v1.1.1 the online catalog includes `qe` (Quantum ESPRESSO DFT, 20 endpoints: task
submit/query/logs/artifact/PP library, etc.) and `materials-design` (durable materials agent
modeling, 7 endpoints: health/capabilities/job-submit(multipart)/job-status/job-cancel, etc.).
See the Skill docs `skills/delta-science/references/qe.md` and
`skills/delta-science/references/materials-design.md`.
The cloud XRD phase-identification, artifact, CIF simulation and refinement contract is in
`skills/delta-science/references/xrd.md`.

## Output format

All commands return a unified JSON envelope:

```json
// success
{"ok":true,"data":{"id":"sb-xxx","status":"running"}}

// error
{"ok":false,"error":{"type":"auth","message":"not authenticated","hint":"run 'delta-cli auth login'","request_id":"create_a1b2c3"}}
```

`error.hint` gives the actionable next step (for a failed create it names the concrete cause - for example, the image was not cached on the target node, so retry or pick a GPU type whose node already has the image), and `error.request_id` identifies the same failure in the server log.

Error types and exit codes:

| Type | Exit Code | Scenario |
|------|-----------|----------|
| validation | 2 | Argument error — includes a file path used where a directory is required (or the reverse) and an invalid glob |
| auth | 3 | Not authenticated |
| permission | 4 | Insufficient permission |
| not_found | 5 | Resource does not exist |
| network | 6 | Network error |
| api | 7 | Server error |
| command_failed | 8 | Command inside the sandbox exited non-zero |
| internal | 10 | Client internal error |
| timeout | 11 | `run-bg --wait` local wait deadline elapsed (the remote command may still be running; see `error.details`) |
| content_integrity_mismatch / content_size_mismatch | 12 | File-transfer verification failed (sha256 or byte-count mismatch) |
| unknown_execution_state | 13 | The command finished but its exit status is unknowable (**never treated as success**) |

## AI Agent Skills

delta-cli ships built-in AI Agent operation manuals (Skills) that help assistants such as Claude Code use the CLI correctly:

```bash
# Install all Skills from the repo
npx skills add delta-agentcloud/delta-cli -y -g

# Install only the unified Science Skill
npx skills add delta-agentcloud/delta-cli -s delta-science
```

Skill files live under `skills/`:
- `delta-shared/` — shared global rules (auth, config, error handling)
- `delta-sandbox/` — Sandbox operation guide (lifecycle, command routing)
- `delta-science/` — the single Science entry point, with the unified Delta CLI wrapper, cross-tool workflows, and
  on-demand references for PubChem, RDKit, pymatgen, XRD, GSAS-II, QE, LAMMPS, Delta-BO, LDM-BO, SynBO,
  AntBO, materials-design, strbo and protein/bioinformatics (`huada_protein_tools`); the protein
  endpoint contracts live under `skills/delta-science/references/protein/`

Hosts such as Memento only need to select `delta-science`. The Skill selects the tool and
operation from a natural-language research goal and loads only the reference for the current
tool, avoiding the planner overhead of many independent Skills. Upgrades automatically remove
the old standalone Science Skills.

### Adding a Science tool or operation

The tool catalog is maintained by the Science Server database. To add a tool, configure a
`tools` record and its `tool_endpoints` records; set `status=false` to disable. The CLI needs
no static table sync or new commands. To let an AI auto-select a new tool and safely assemble
requests, also update the tool-selection table in `skills/delta-science/SKILL.md` and the
corresponding `skills/delta-science/references/<tool>.md`; the references directory holds only
real tool docs, never meta-docs such as catalogs, routing, service maps or cross-tool
workflows. The canonical tool name in the database must exactly match the tool table and its
reference.

## Development

```bash
# Local build
make build

# Run tests
make test

# Lint
make lint

# Cross-platform build (produces platform binaries in bin/)
make release

# Install into the system PATH from source (default /usr/local/bin; macOS / Linux)
make install

# Custom install prefix
make PREFIX=$HOME/.local install

# Windows source install (pick one)
# A: install to GOPATH/bin with go install, from the repository root
go install ./cmd/delta-cli

# B: run the PowerShell script from the repo root to build and add to the user PATH
# .\install-from-source.ps1
# .\install-from-source.ps1 -InstallDir "$env:USERPROFILE\bin"
```

### Daily development flow

After modifying `skills/` or building `delta-cli`, sync to the Memento-S project for it to take effect:

```bash
# 1. Sync the full Skill set via the install wizard
node scripts/install-wizard.js

# 2. Build the Windows version with Docker (when no local Go toolchain is available)
docker run --rm -v "$(pwd):/src" -w /src \
  -e GOPROXY=https://proxy.golang.org,direct \
  golang:1.23 \
  sh -c 'go build -buildvcs=false \
    -ldflags "-X github.com/delta-infra/delta-infra-cli/internal/build.Version=$(git describe --tags --always --dirty 2>/dev/null || echo dev)" \
    -o bin/delta-cli-windows-amd64.exe ./cmd/delta-cli'

# 3. Place delta-cli into Memento-S's node_modules/.bin (replacing the old version)
cp bin/delta-cli-windows-amd64.exe \
   <memento-s>/electron_rebuild/node_modules/.bin/delta-cli

# 4. Restart the Memento-S sidecar to pick up the new binary
#   Reload in the Electron UI, or restart the sidecar process manually
```

> **Note**: after replacing `delta-cli`, if a token is already configured in `~/.delta-infra/config.json`,
> the Sandbox and Science Skills work immediately; otherwise configure it first with `auth login`.

## Build and Release

### Local build

```bash
# Local build
make build

# Run tests
make test

# Lint
make lint

# Cross-platform build (produces platform binaries in bin/)
make release
```

> `make release` only produces platform binaries (e.g. `bin/delta-cli-linux-amd64`); it does not package them. Use `release.sh` for packaging and publishing.

### One-click release (maintainers)

The `release.sh` script at the repo root drives the full release flow:

```bash
export GH_TOKEN=<your-github-pat>
export NPM_TOKEN=<your-npm-token>
./release.sh v1.1.19
```

Script steps:
1. Update the `package.json` version.
2. Cross-compile the 5 platform binaries (prefers Docker + `make release`, falls back to the local Go toolchain when Docker is unavailable).
3. Produce `.tar.gz` / `.zip` archives and SHA-256 checksums in `bin/`, and update the root `checksums.txt`.
4. Create the GitHub Release and upload the binary archives and checksums via `.ci/publish-github-release.js`.
5. Sync the npm scripts, README and all Skills to the public repo `delta-agentcloud/delta-cli`.
6. Run `npm publish --access public` to publish `@delta-agentcloud/cli`.
7. Create the local source commit and Git tag; the source GitLab remote is pushed only when `PUSH_SOURCE=1` is explicitly set.

`gh` CLI and Node.js/npm must be installed, and `GH_TOKEN` and `NPM_TOKEN` configured. See `.gitlab-ci.yml` for the CI flow.

## In-sandbox mirror auto-configuration

At sandbox creation, the server **auto-detects the deployment's network region** (mainland China / overseas) and, when the region is mainland China, injects mirror configuration into the container. No manual configuration is needed — it works on entry.

- **Mainland China (domestic)**: the server injects mirror config so `pip install` / `npm install` / `git clone` / HuggingFace downloads use domestic mirrors.
- **Overseas**: no mirrors are injected; each package manager uses its official source.
- The server probes connectivity to `pypi.org` / `npmjs.org` to decide the region (cached; falls back to overseas on probe failure, i.e. fail-open).

Four categories of configuration are covered:

| Type | Description | Example |
|------|-------------|---------|
| pip index-url | Python package index | `https://pypi.tuna.tsinghua.edu.cn/simple` |
| npm registry | Node package source | `https://registry.npmmirror.com` |
| HF_ENDPOINT | HuggingFace download domain | `https://hf-mirror.com` |
| git insteadOf | GitHub clone proxy | `https://gh-proxy.com/https://github.com` |

The AutoDL compute backend has mirror injection off by default (overseas platform); it takes effect only when operations explicitly enable it.

> **Note the two separate mirror mechanisms**: this in-sandbox mirror config targets the **package managers inside the sandbox** (pip / npm / git / HF). It is unrelated to the CLI's own download mirror `DELTA_CLI_MIRROR` — `DELTA_CLI_MIRROR` is used only to **install the delta-cli binary** and does not affect the sandbox.

## Tech stack

- **Language**: Go 1.23+
- **CLI framework**: [Cobra](https://github.com/spf13/cobra)
- **Config**: JSON file (`~/.delta-infra/config.json`, mode 0600)
- **Output**: JSON envelope (AI-agent friendly, scriptable)
- **Distribution**: GitHub Release (tar.gz/zip) + npm wrapper
- **Architecture**: flat static command tree, Factory dependency injection, lazy initialization

## License

MIT
