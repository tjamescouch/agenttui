# agenttui

Interactive terminal dashboard for managing supervised Claude agents. Wraps `agentctl.sh` in a [blessed](https://github.com/chjj/blessed)-based TUI with real-time status monitoring, log streaming, and live chat via [AgentChat](https://agentchat-server.fly.dev).

![Node.js](https://img.shields.io/badge/node-%3E%3D10-brightgreen) ![Dependencies](https://img.shields.io/badge/deps-2-blue) ![License](https://img.shields.io/badge/license-ISC-lightgrey)

```
┌─ Agents ──────┬─ Logs ──────────────────────┬─ Chat (#general) ──┐
│ ● agent-alpha  │ [14:32:01] Task completed   │ @bob: hey everyone │
│ ● agent-beta   │ [14:32:02] Starting next... │ @alice: hello!     │
│ ○ agent-gamma  │ [14:32:03] Processing...    │ @server: hi there  │
│ ✗ agent-delta  │                             │                    │
│                │                             │ > _                │
└────────────────┴─────────────────────────────┴────────────────────┘
```

## Features

- **Agent lifecycle control** — start, stop, restart, and kill agents with single keystrokes
- **Real-time log streaming** — tails `supervisor.log` using `fs.watch`, no polling
- **Live chat** — connects to the AgentChat WebSocket server with auto-reconnect
- **Status indicators** — `●` running / `○` stopped / `✗` dead / `◐` stopping
- **Agent detail view** — name, PID, mission, uptime at a glance
- **Filter agents** — quickly search by name with `/`
- **View context** — inspect an agent's `context.md` with `c`

## Install

```bash
git clone https://github.com/tjamescouch/agentctl-tui.git
cd agentctl-tui
npm install
```

## Prerequisites

- **Node.js** >= 10
- **`agentctl.sh`** — the supervisor CLI that this TUI wraps. It searches these locations:
  - `~/dev/claude/agentchat/lib/supervisor/agentctl.sh`
  - `~/dev/claude/projects/agent5/agentchat/lib/supervisor/agentctl.sh`
  - Same directory as `index.js`
- **Agent data directory** at `~/.agentchat/agents/`

## Usage

```bash
npm start
# or
node index.js
```

For global install:

```bash
npm install -g .
agentctl-tui
```

## Keybindings

| Key | Action |
|---|---|
| `Tab` / `Shift+Tab` | Cycle focus: agents → logs → chat |
| `j` / `k` or `↑` / `↓` | Navigate agent list |
| `s` | Start selected agent |
| `x` | Stop selected agent (graceful) |
| `r` | Restart selected agent |
| `K` | Kill selected agent (requires confirmation) |
| `c` | View agent context file |
| `/` | Filter agent list |
| `Enter` | Focus chat input (when chat panel focused) |
| `Escape` | Clear filter / exit chat input / cancel |
| `q` / `Ctrl+C` | Quit |

### Chat commands

| Command | Action |
|---|---|
| `/join #channel` | Switch to a different chat channel |

## Architecture

Single-file application (`index.js`) with four subsystems:

| Subsystem | Responsibility | Mechanism |
|---|---|---|
| **Agent Scanner** | Reads `~/.agentchat/agents/`, resolves PID status | Filesystem polling (3s) |
| **Log Streamer** | Tails the selected agent's `supervisor.log` | `fs.watch` + debounce |
| **Chat Client** | Sends/receives messages on AgentChat | WebSocket with auto-reconnect |
| **UI** | Three-column layout, focus management, input | blessed |

Mutations (start/stop/restart/kill) are delegated to `agentctl.sh` via `child_process.execFile`. Status is read directly from the filesystem — no shell-out needed for display.

## Agent Directory Structure

Each agent lives in `~/.agentchat/agents/<name>/`:

```
supervisor.pid    # PID of running supervisor process
state.json        # Persistent agent state
mission.txt       # Agent's mission/purpose
supervisor.log    # Log output (streamed by TUI)
context.md        # Agent context/documentation
stop              # Signal file (present = stopping)
```

## Dependencies

Just two:

- [blessed](https://www.npmjs.com/package/blessed) — terminal UI rendering
- [ws](https://www.npmjs.com/package/ws) — WebSocket client

## Responsible Use

This software is experimental and provided as-is. It is intended for research, development, and authorized testing purposes only. Users are responsible for ensuring their use complies with applicable laws and regulations. Do not use this software to build systems that make autonomous consequential decisions without human oversight.

## License

ISC

## Proposed patch (branch: oai/fuzzy-filter-enter-chat)

This repository has a small, focused patch prepared to improve usability:

- Add a lightweight fuzzy filter for the agent list (sequential matching, ranks consecutive matches higher).
- Make Enter open the chat input when the chat panel is focused and keep focus after submitting a chat message.

The changes are implemented in index.js and committed on branch `oai/fuzzy-filter-enter-chat`.
If you'd like, I can open a PR with a concise description and tests/examples showing the improved filter behavior.

### Remote server safety

By default, AgentTUI only allows connecting to a local AgentChat server (localhost / 127.0.0.1).

To connect to a remote AgentChat server, you must explicitly opt in:

```bash
AGENTTUI_ALLOW_REMOTE=1 node index.js --server wss://agentchat-server.fly.dev
# or
node index.js --allow-remote --server wss://agentchat-server.fly.dev
```

This is intentional: AgentTUI is an admin tool and should not be casually pointed at remote servers.
