# Dimensional AGENTS.md

## What is DimOS

The agentic operating system for generalist robotics. `Modules` communicate via typed streams over LCM, ROS2, DDS, or other transports. `Blueprints` compose modules into runnable robot stacks. `Skills` give agents the ability to execute physical on-hardware functions like `grab()`, `follow_object()`, or `jump()`.

---

## Quick Start

```bash
# Install
uv sync --all-extras --no-extra dds

# List all runnable blueprints
dimos list

# --- Go2 quadruped ---
dimos --replay run unitree-go2                  # perception + mapping, replay data
dimos --replay run unitree-go2 --daemon         # same, backgrounded
dimos --replay run unitree-go2-agentic          # + LLM agent (GPT-4o) + skills + MCP server
dimos run unitree-go2-agentic --robot-ip 192.168.123.161  # real Go2 hardware

# --- G1 humanoid ---
dimos --simulation run unitree-g1-agentic-sim   # G1 in MuJoCo sim + agent + skills
dimos run unitree-g1-agentic --robot-ip 192.168.123.161   # real G1 hardware

# --- Inspect & control ---
dimos status
dimos log              # last 50 lines, human-readable
dimos log -f           # follow/tail in real time
dimos agent-send "say hello"
dimos stop             # graceful SIGTERM → SIGKILL
dimos restart          # stop + re-run with same original args
```

### Blueprint quick-reference

| Blueprint | Robot | Hardware | Agent | MCP server | Notes |
|-----------|-------|----------|-------|------------|-------|
| `unitree-go2-agentic` | Go2 | real | via McpClient | ✓ | McpServer live |
| `unitree-g1-agentic-sim` | G1 | sim | GPT-4o (G1 prompt) | — | Full agentic sim, no real robot needed |
| `xarm-perception-agent` | xArm | real | GPT-4o | — | Manipulation + perception + agent |
| `xarm-perception-sim-agent` | xArm | sim | GPT-4o | — | Manipulation + perception + agent, sim |
| `xarm7-planner-coordinator` | xArm7 | real | — | — | Trajectory planner coordinator |
| `teleop-quest-xarm7` | xArm7 | real | — | — | Quest VR teleop |
| `dual-xarm6-planner` | xArm6×2 | real | — | — | Dual-arm motion planner |

Run `dimos list` for the full list.

---

## Tools available to you (MCP)

**MCP only works if the blueprint includes `McpServer`.** All shipped agentic blueprints use `McpServer` + `McpClient`. E.g.: `unitree-go2-agentic`.

```bash
# Start the MCP-enabled blueprint first:
dimos --replay run unitree-go2-agentic --daemon

# Then use MCP tools:
dimos mcp list-tools                                              # all available skills as JSON
dimos mcp call move --arg x=0.5 --arg duration=2.0               # call by key=value args
dimos mcp call move --json-args '{"x": 0.5, "duration": 2.0}'    # call by JSON
dimos mcp status      # PID, module list, skill list
dimos mcp modules     # module → skills mapping

# Send a message to the running agent (works without McpServer too):
dimos agent-send "walk forward 2 meters then wave"
```

The MCP server runs at `http://localhost:9990/mcp` (`GlobalConfig.mcp_port`).

### Adding McpServer to a blueprint

Use **both** `McpServer.blueprint()` and `McpClient.blueprint()`.

```python
from dimos.agents.mcp.mcp_client import McpClient
from dimos.agents.mcp.mcp_server import McpServer

unitree_go2_agentic = autoconnect(
    unitree_go2_spatial,   # robot stack
    McpServer.blueprint(), # HTTP MCP server — exposes all @skill methods on port 9990
    McpClient.blueprint(), # LLM agent — fetches tools from McpServer
    _common_agentic,       # skill containers
)
```

Reference: `dimos/robot/unitree/go2/blueprints/agentic/unitree_go2_agentic.py`

---

## Repo Structure

```
dimos/
├── core/                    # Module system, blueprints, workers, transports
│   ├── module.py            # Module base class, In/Out streams
│   ├── core.py              # @rpc decorator
│   ├── stream.py            # In[T], Out[T], Transport[T]
│   ├── transport.py         # LCM/SHM/ROS/DDS/Jpeg transports
│   ├── coordination/
│   │   ├── blueprints.py           # Blueprint, autoconnect()
│   │   ├── module_coordinator.py   # Deploy + lifecycle orchestration
│   │   ├── python_worker.py        # Forkserver workers + Actor IPC
│   │   └── worker_manager_*.py     # Python / docker worker pools
│   ├── global_config.py     # GlobalConfig (env vars, CLI flags, .env)
│   └── run_registry.py      # Per-run tracking + log paths
├── robot/
│   ├── cli/dimos.py         # CLI entry point (typer)
│   ├── all_blueprints.py    # Auto-generated blueprint registry (DO NOT EDIT MANUALLY)
│   ├── unitree/             # Unitree robot implementations (Go2, G1, B1)
│   │   ├── unitree_skill_container.py  # Go2 @skill methods
│   │   ├── go2/             # Go2 blueprints and connection
│   │   └── g1/              # G1 blueprints, connection, sim, skills
│   └── drone/               # Drone implementations (MAVLink + DJI)
│       ├── connection_module.py        # MAVLink connection
│       ├── camera_module.py            # DJI video stream
│       ├── drone_tracking_module.py    # Visual object tracking
│       └── drone_visual_servoing_controller.py  # Visual servoing
├── agents/
│   ├── agent.py             # Agent module (LangGraph-based)
│   ├── system_prompt.py     # Default Go2 system prompt
│   ├── annotation.py        # @skill decorator
│   ├── mcp/                 # McpServer, McpClient, McpAdapter
│   └── skills/              # NavigationSkillContainer, SpeakSkill, etc.
├── navigation/              # Path planning, frontier exploration
├── perception/              # Object detection, tracking, memory
├── visualization/rerun/     # Rerun bridge
├── msgs/                    # Message types (geometry_msgs, sensor_msgs, nav_msgs)
└── utils/                   # Logging, data loading, CLI tools
docs/
├── usage/modules.md         # ← Module system deep dive
├── usage/blueprints.md      # Blueprint composition guide
├── usage/configuration.md   # GlobalConfig + Configurable pattern
├── development/testing.md   # Fast/slow tests, pytest usage
├── development/dimos_run.md # CLI usage, adding blueprints
└── agents/                  # Agent system documentation
```

---

## Architecture

### Modules

Autonomous subsystems. Communicate via `In[T]`/`Out[T]` typed streams. Run in forkserver worker processes.

```python
from dimos.core.module import Module
from dimos.core.stream import In, Out
from dimos.core.core import rpc
from dimos.msgs.sensor_msgs import Image

class MyModule(Module):
    color_image: In[Image]
    processed: Out[Image]

    @rpc
    def start(self) -> None:
        super().start()
        self.color_image.subscribe(self._process)

    def _process(self, img: Image) -> None:
        self.processed.publish(do_something(img))
```

### Blueprints

Compose modules with `autoconnect()`. Streams auto-connect by `(name, type)` matching.

```python
from dimos.core.coordination.blueprints import autoconnect

my_blueprint = autoconnect(module_a(), module_b(), module_c())
```

To run a blueprint directly from Python:

```python
# build() deploys all modules into forkserver workers and wires streams
# loop() blocks the main thread until stopped (Ctrl-C or SIGTERM)
autoconnect(module_a(), module_b(), module_c()).build().loop()
```

Expose as a module-level variable for `dimos run` to find it. Add to the registry by running `pytest dimos/robot/test_all_blueprints_generation.py`.

### GlobalConfig

Singleton config. Values cascade: defaults → `.env` → env vars → blueprint → CLI flags. Env vars prefixed `DIMOS_`. Key fields: `robot_ip`, `simulation`, `replay`, `viewer`, `n_workers`, `mcp_port`.

### Transports

- **LCMTransport**: Default. Multicast UDP.
- **SHMTransport/pSHMTransport**: Shared memory — use for images and point clouds.
- **pLCMTransport**: Pickled LCM — use for complex Python objects.
- **ROSTransport**: ROS topic bridge — interop with ROS nodes (`dimos/core/transport.py`).
- **DDSTransport**: DDS pub/sub — available when `DDS_AVAILABLE`; install with `uv sync --extra dds` (`dimos/protocol/pubsub/impl/ddspubsub.py`).

---

## CLI Reference

### Global flags

Every `GlobalConfig` field is a CLI flag: `--robot-ip`, `--simulation/--no-simulation`, `--replay/--no-replay`, `--viewer {rerun|rerun-web|foxglove|none}`, `--mcp-port`, `--n-workers`, etc. Flags override `.env` and env vars.

### Core commands

| Command | Description |
|---------|-------------|
| `dimos run <blueprint> [--daemon]` | Start a blueprint |
| `dimos status` | Show running instance (run ID, PID, blueprint, uptime, log path) |
| `dimos stop [--force]` | SIGTERM → SIGKILL after 5s; `--force` = immediate SIGKILL |
| `dimos restart [--force]` | Stop + re-exec with original args |
| `dimos list` | List all non-demo blueprints |
| `dimos show-config` | Print resolved GlobalConfig values |
| `dimos log [-f] [-n N] [--json] [-r <run-id>]` | View per-run logs |
| `dimos mcp list-tools / call / status / modules` | MCP tools (requires McpServer in blueprint) |
| `dimos agent-send "<text>"` | Send text to the running agent via LCM |
| `dimos lcmspy / agentspy / humancli / top` | Debug/diagnostic tools |
| `dimos topic echo <topic> / send <topic> <expr>` | LCM topic pub/sub |
| `dimos rerun-bridge` | Launch Rerun visualization standalone |

Log files: `~/.local/state/dimos/logs/<run-id>/main.jsonl`
Run registry: `~/.local/state/dimos/runs/<run-id>.json`

---

## Agent System

### The `@skill` Decorator

`dimos/agents/annotation.py`. Sets `__rpc__ = True` and `__skill__ = True`.

- `@rpc` alone: callable via RPC, not exposed to LLM
- `@skill`: implies `@rpc` AND exposes method to the LLM as a tool. **Do not stack both.**

#### Schema generation rules

| Rule | What happens if you break it |
|------|------------------------------|
| **Docstring is mandatory** | `ValueError` at startup — module fails to register, all skills disappear |
| **Type-annotate every param** | Missing annotation → no `"type"` in schema — LLM has no type info |
| **Return `str`** | `None` return → agent hears "It has started. You will be updated later." |
| **Full docstring verbatim in `description`** | Keep `Args:` block concise — it appears in every tool-call prompt |

Supported param types: `str`, `int`, `float`, `bool`, `list[str]`, `list[float]`. Avoid complex nested types.

#### Minimal correct skill

```python
from dimos.agents.annotation import skill
from dimos.core.core import rpc
from dimos.core.module import Module

class MySkillContainer(Module):
    @rpc
    def start(self) -> None:
        super().start()

    @rpc
    def stop(self) -> None:
        super().stop()

    @skill
    def move(self, x: float, duration: float = 2.0) -> str:
        """Move the robot forward or backward.

        Args:
            x: Forward velocity in m/s. Positive = forward, negative = backward.
            duration: How long to move in seconds.
        """
        return f"Moving at {x} m/s for {duration}s"

my_skill_container = MySkillContainer.blueprint
```

### System Prompts

| Robot | File | Variable |
|-------|------|----------|
| Go2 (default) | `dimos/agents/system_prompt.py` | `SYSTEM_PROMPT` |
| G1 humanoid | `dimos/robot/unitree/g1/system_prompt.py` | `G1_SYSTEM_PROMPT` |

Pass the robot-specific prompt: `McpClient.blueprint(system_prompt=G1_SYSTEM_PROMPT)`. The default prompt is Go2-specific; using it on G1 causes hallucinated skills.

### RPC Wiring

To call methods on another module, declare a `Spec` Protocol and annotate an attribute with it. The blueprint injects the matching module at build time — fully typed, no strings, fails at build time (not runtime) if no match is found.

```python
# my_module_spec.py
from typing import Protocol
from dimos.spec.utils import Spec

class NavigatorSpec(Spec, Protocol):
    def set_goal(self, goal: PoseStamped) -> bool: ...
    def cancel_goal(self) -> bool: ...

# my_skill_container.py
class MySkillContainer(Module):
    _navigator: NavigatorSpec   # injected by blueprint at build time

    @skill
    def go_to(self, x: float, y: float) -> str:
        """Navigate to a position."""
        self._navigator.set_goal(make_pose(x, y))
        return "Navigating"
```

If multiple modules match the spec, use `.remappings()` to resolve. Source: `dimos/spec/utils.py`, `dimos/core/coordination/blueprints.py`.

### Adding a New Skill

1. Pick the right container (robot-specific or `dimos/agents/skills/`).
2. `@skill` + mandatory docstring + type annotations on all params.
3. If it needs another module's RPC, use the Spec pattern.
4. Return a descriptive `str`.
5. Update the system prompt — add to the `# AVAILABLE SKILLS` section.
6. Expose as `my_container = MySkillContainer.blueprint` and include in the agentic blueprint.

---

## Testing

```bash
# Fast tests (default)
uv run pytest

# Include slow tests (CI)
./bin/pytest-slow

# Single file
uv run pytest dimos/core/test_blueprints.py -v

# Mypy
uv run mypy dimos/
```

`uv run pytest` excludes `slow`, `tool`, and `mujoco` markers. CI (`./bin/pytest-slow`) includes slow, excludes tool and mujoco. See `docs/development/testing.md`.

---

## Pre-commit & Code Style

Pre-commit runs on `git commit`. Includes ruff format/check, license headers, LFS checks.

**Always activate the venv before committing:** `source .venv/bin/activate`

Code style rules:
- Imports at top of file. No inline imports unless circular dependency.
- Use `requests` for HTTP (not `urllib`). Use `Any` (not `object`) for JSON values.
- Prefix manual test scripts with `demo_` to exclude from pytest collection.
- Don't hardcode ports/URLs — use `GlobalConfig` constants.
- Type annotations required. Mypy strict mode.

---

## `all_blueprints.py` is auto-generated

`dimos/robot/all_blueprints.py` is generated by `test_all_blueprints_generation.py`. After adding or renaming blueprints:

```bash
pytest dimos/robot/test_all_blueprints_generation.py
```

CI asserts the file is current — if it's stale, CI fails.

---

## Git Workflow

- Branch prefixes: `feat/`, `fix/`, `refactor/`, `docs/`, `test/`, `chore/`, `perf/`
- **PRs target `dev`** — never push to `main` or `dev` directly
- **Don't force-push** unless after a rebase with conflicts
- **Minimize pushes** — every push triggers CI (~1 hour on self-hosted runners). Batch commits locally, push once.

---

## Further Reading

- Module system: `docs/usage/modules.md`
- Blueprints: `docs/usage/blueprints.md`
- Visualization: `docs/usage/visualization.md`
- Configuration: `docs/usage/configuration.md`
- Testing: `docs/development/testing.md`
- CLI / dimos run: `docs/development/dimos_run.md`
- LFS data: `docs/development/large_file_management.md`
- Agent system: `docs/agents/`

---

## Pipeline (fork @ `jiangtao129/dimos`)

Everything above this section is upstream content; everything below
is fork-only and only applies when developing against
`github.com/jiangtao129/dimos`. Do not assume any of it is true upstream.

### Remotes

This fork lives behind three remotes; keep all three:

- `origin`   — `git@gitlab.topsun:topsun/dimos_huazhijian.git`  (corp GitLab; do not delete)
- `upstream` — `https://github.com/dimensionalOS/dimos.git`     (community upstream)
- `github`   — `git@github.com:jiangtao129/dimos.git`           (this fork; PR target)

PRs target **`jiangtao129/dimos`'s `main`**, not upstream's `dev`. Sync
with upstream via `git fetch upstream && git merge upstream/main` (or
rebase, when there are no shared review threads). Never push to `origin`
directly without confirming the corp GitLab branch policy.

### Ground rules

These are mirrored in `.cursor/rules/00-workflow.mdc` so an editing
agent picks them up automatically.

1. **No direct push to `main`.** Every change goes through a feature
   branch + PR + auto-merge.
2. **Never skip `bash scripts/verify.sh`.** If verify is red, fix the
   code, not the script. Adding `|| true`, weakening assertions, or
   shortening the test path to "make CI green" is a P0 review block.
3. **No `git push --force` to `main`.** `--force-with-lease` is only
   acceptable on the bootstrap PR (covered by an explicit user
   decision) or after a clean rebase that the user has consented to.
4. **No secrets in the repo.** `.env`, OAuth tokens, API keys, robot
   SSH keys, telemetry credentials must stay in `.gitignore`. Pre-commit
   `lfs_check` and `largefiles-check` will not catch secrets — review
   every diff.
5. **One PR, one focused change.** No drive-by refactors, no
   "while-I'm-here" formatting churn outside the changed area.
6. **GitHub-side settings are owned by the human user.** Branch
   protection, Codex auto-review on/off, repo permissions, secrets — the
   agent emits a TODO checklist; it never clicks GitHub UI on the user's
   behalf.

### Verify (single source of truth)

```bash
bash scripts/verify.sh
```

The same script is executed by `.github/workflows/verify.yml` on every
PR; keep them in sync. Verify covers, in order:

1. `uv sync --all-extras --no-extra dds --frozen`
2. `uv run pre-commit run --all-files` (ruff format/check, license
   header, JSON/YAML/TOML, LFS + largefiles, doclinks, commit-msg filter)
3. `uv run pytest -q --maxfail=5` (testpaths=`dimos`, with `slow`,
   `tool`, `mujoco` markers excluded by `pyproject.toml`)

Two environment variables, set automatically by `verify.sh`, make the
test suite runnable in this fork's CI environment:

- `DIMOS_SKIP_MISSING_LFS=1` — when the corp LFS server is unreachable
  (the case from GitHub Actions ubuntu-latest), `_lfs_pull` failures are
  converted to `pytest.skip(...)` via the repo-root `conftest.py`.
- `DIMOS_SKIP_KNOWN_FLAKY=1` — skips a small whitelist of upstream-bug
  flaky tests (currently `TestKinematicSim` LCM-thread leak); see the
  `_KNOWN_FLAKY` list in `conftest.py`.

Both default to unset, so upstream's self-hosted CI is unaffected by
this fork's `conftest.py`.

### Ship (one-shot PR)

The `.cursor/commands/ship.md` slash command (`/ship`) runs the full
sequence: branch → commit → `bash scripts/verify.sh` → push → PR via
REST API → `gh pr merge --auto --squash --delete-branch`. It is the
*only* sanctioned way to land a change in this fork. Read that file
before invoking it.

### Codex review guidelines

The ChatGPT Codex GitHub App is enabled on `jiangtao129/dimos` for PR
auto-review. The bot reacts with a thumbs-up (no issues) or files a
`COMMENTED` review with a P0/P1/P2/P3 breakdown. When writing code that
Codex will review, self-check against the categories below before
opening the PR — Codex is a second pair of eyes, not the first one.

#### P0 — must block the PR

- Control-loop or real-time loop with blocking I/O and no timeout
  (network read, file lock, queue get, IPC roundtrip).
- `cmd_vel`, joint torque, joint velocity, end-effector pose, or any
  hardware command publish that is not clamped by a documented limit
  before going on the wire. A runaway robot is a hardware-cost incident.
- `subprocess.run`, `os.system`, `eval`, or `exec` invocations whose
  arguments come from user-supplied LLM / MCP / `agent-send` text
  without explicit validation. This is command injection.
- API keys, OAuth tokens, SSH keys, or telemetry credentials committed
  to the repo, including `.env` files and "example" placeholders that
  contain real values.
- LCM, DDS, or ROS topic-name or message-schema change without
  matching updates to publishers, subscribers, and serializers.
  Symptoms appear at runtime as silent message-drop, not at import.
- Type hints on `In[T]` / `Out[T]` streams that disagree with the
  generic actually used at runtime — fails inside the `Module` worker
  process and is hard to debug from the parent.
- Edits to `scripts/verify.sh`, `.github/workflows/verify.yml`, or this
  AGENTS.md section that weaken the verify gate (skipping checks,
  swallowing exit codes, adding broad `|| true`).

#### P1 — flag, may not block

- Backwards-incompatible change to a `Module` RPC signature without
  migrating callers in the same PR.
- Change to a `Blueprint`'s `autoconnect()` wiring that silently
  re-routes streams from one transport to another.
- Change to the default transport selection
  (`LCMTransport` / `SHMTransport` / `pSHMTransport` / `pLCMTransport` /
  `DDSTransport` / `ROSTransport`) for any `In[T]` / `Out[T]`.
- Test coverage drop on a code path that previously had tests; or a
  feature added with no test at all.
- New `pyproject.toml` dependency without a one-line justification in
  the PR body.
- Disabling a pre-commit hook globally, or marking a previously enabled
  pytest marker as default-skipped.

#### P2 / P3 — do not flag

- Python typos, docstring typos, comment typos. (Handled by humans
  during normal review.)
- Import ordering, line length, format choices (handled by ruff).
- Style preferences such as f-string vs `.format()`, comprehension vs
  loop, when both are correct.
