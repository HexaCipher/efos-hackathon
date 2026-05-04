# ask

<p align="center"><b>AI agent for your terminal. Everything stays local.</b></p>
<p align="center">Written in Go. Runs everywhere. Full control over your data.</p>

<p align="center">
  <a href="./assets/demo-smooth.gif">
    <img src="./assets/demo-small.gif" alt="ask demo" width="900" />
  </a>
  <br/>
  <sub>Click to see full demo</sub>
</p>

A no-nonsense CLI for talking to LLMs with actual superpowers. Chat in REPL mode or fire one-off questions. Optional agent mode lets the AI run shell commands, edit files, make HTTP calls, and manage local todos/memory—all with approval gates you control.

## Features

- **REPL chat** with slash commands for config on the fly
- **One-shot mode** for quick questions (pipes stdin too)
- **Streaming markdown** rendering as you type
- **Agent mode** with tool calling (shell, file ops, HTTP, clipboard, todos, memory)
- **Approval gates** per action (bypass with `--yolo` if you're in a hurry)
- **Chat history** persisted in SQLite
- **Vector memory** store for long-term context (add/view/update/delete)
- **Named lists/todos** stored locally (accessible to the agent)
- **Shell completions** for bash/zsh/fish
- **Custom system prompts** (load from file)

## Requirements

Go 1.25+ and a Gemini API key.

**On Linux/macOS:** Set it in your environment:

```bash
export GEMINI_API_KEY="your_key_here"
```

**On Windows:** Use the built-in Environment Variable manager:
1. Press **Win + R**, type `sysdm.cpl`, and press Enter.
2. Go to the **Advanced** tab and click **Environment Variables...**.
3. Under **User variables**, click **New...**.
4. Set **Variable name** to `GEMINI_API_KEY` and **Variable value** to your key.
5. Click OK on all windows and restart your terminal.

## Environment Variables

The program reads these environment variables at runtime:

| Variable                       | Required for             | Notes                                                        |
| ------------------------------ | ------------------------ | ------------------------------------------------------------ |
| `GEMINI_API_KEY`               | Core CLI and agent mode  | Required to talk to the Gemini API.                          |
| `ASKCLI_SERVER_KEY`            | Remote server mode       | Server-side: API key that clients must provide to authenticate. Client-side: used as fallback if `ASKCLI_CLIENT_KEY` not set. |
| `ASKCLI_CLIENT_KEY`            | Remote client mode       | Client-side: API key to authenticate with a remote server (alternative to `ASKCLI_SERVER_KEY`). |
| `TELEGRAM_BOT_TOKEN`           | Telegram background mode | Required when running `ask --background=true`.               |
| `AGENT_MAIL_API_KEY`           | AgentMail tool           | Required for the `mail` tool.                                |
| `INBOX_NAME`                   | AgentMail tool           | The inbox name used by the `mail` tool.                      |
| `ELEVEN_LABS_API_KEY`          | TTS tool                 | Required for `text_to_speech_file`.                          |
| `DISPLAY` or `WAYLAND_DISPLAY` | Clipboard tool           | Needed when using clipboard features in a graphical session. |
| `PORT`                         | Server mode              | Port to run the server on (default: 3000).                   |

If you only use the local CLI, `GEMINI_API_KEY` is the only required variable.

## Quick Start

```bash
git clone https://github.com/zephex/go-ask.git
cd go-ask
go build -o ask
./ask "Your question here"
```

Or install globally:

```bash
sudo mv ask /usr/local/bin/
```

## Usage

**One-shot mode:**

```bash
ask "What is a goroutine?"
ask --model exp "Analyze this architecture"
cat main.go | ask "Explain this code"
```

**Chat mode:**

```bash
ask --chat
# or
ask chat
```

**Agent mode (enable tool calling):**

```bash
ask --chat --agent
```

Auto-approve tool actions (use with caution):

```bash
ask --chat --agent --yolo
```

## Model Aliases

Quick names for common models:

- `free` – `gemma-4-26b-a4b-it` (default, fast)
- `cheap` – `gemini-3.1-flash-lite-preview` (ultra-light)
- `exp` – `gemini-3-flash-preview` (more capable)

Or pass any full model name.

## Reasoning Control

Dial up the thinking time (higher = slower, more accurate):

- `HIGH` – deep reasoning
- `MED` / `MEDIUM` / `MID`
- `LOW`
- `MIN` / `MINIMAL` – fast, lightweight

## Common Flags

```
--chat              Start REPL mode
--agent             Enable tool calling
--yolo              Auto-approve all actions
--stream            Stream markdown as it renders (default: on)
--system <file>     Load custom system prompt
--model <alias>     Pick a model
--reason <level>    Set reasoning level
--clear             Nuke chat history on startup
--connect <url>     Connect to a remote ask server (e.g. http://host:3000)
--server-key <key>  API key for remote server authentication (overrides env vars)
--background        Run as background Telegram bot
```

## Remote Server

Run `ask` as a server that other clients can connect to. The server processes requests and returns responses via HTTP.

**Server setup:**

1. Set the API key that clients will need to provide:
   ```bash
   export ASKCLI_SERVER_KEY="your-secret-key-here"
   export GEMINI_API_KEY="your-gemini-key"
   ```

2. Start the server (runs on port 3000 by default, or set `PORT` env):
   ```bash
   ask --background=true
   # or directly (without Telegram):
   go run . 2>/dev/null &
   # The server listens on /ask (authenticated) and /health (no auth)
   ```

**Client usage:**

Connect to the remote server from another machine or terminal using `--connect`:

- **One-shot query:**
  ```bash
  ask --connect http://server:3000 --server-key YOUR_KEY "your question"
  ```

- **Interactive chat:**
  ```bash
  ask --connect http://server:3000 --server-key YOUR_KEY --chat
  ```

- **Using env vars (on client):**
  ```bash
  export ASKCLI_CLIENT_KEY="your-secret-key-here"
  ask --connect http://server:3000 --chat
  ```

**Notes:**

- The `--server-key` flag overrides environment variables (`ASKCLI_CLIENT_KEY`, `ASKCLI_SERVER_KEY`).
- Server validates the `x-askcli-api-key` header on each request.
- The server shares the same SQLite database and vector memory across all clients.
- Remote clients do not support streaming `--connect` (server-side only for now, YOLO mode is set to `true` in server letting agent perform any tool calls.).

## Chat Mode (REPL)

Drop into an interactive session with slash commands for everything:

```bash
ask --chat
```

**Available commands:**

- `/help` – show this list
- `/status` – what model/settings are active
- `/model <name>` – switch models on the fly
- `/reason <level>` – adjust reasoning (HIGH/MED/LOW/MIN)
- `/stream on|off` – toggle streaming output
- `/agent on|off` – enable/disable tool calling
- `/yolo on|off` – auto-approve tools
- `/pwd` – print working directory
- `/cd <path>` – change directory for tool commands
- `/history [n]` – show last n messages
- `/clear` – wipe current conversation
- `/memories` – open the memory manager
- `/exit` or `/quit` – leave

## Memory (Vector Store)

Store facts locally and let the AI access them across chats. Useful for storing coding patterns, project context, or anything you want the agent to remember.

**Access:**

- CLI: `ask memories` (list), `ask memories manage` (interactive editor)
- Agent tools: `memory_view`, `memory_add`, `memory_update`, `memory_delete`

**Manager commands:**

- `l` / `list` – show all
- `d <n>` / `del <n>` – delete entry n
- `da` / `delall` – nuke everything
- `q` / `quit` – exit manager

### How It Works

**Storage:** Chromem persistent DB in `~/db`. Each memory gets a stable hash-based ID.

**Management:** Explicit (for now). Memories don't auto-inject into every prompt. You manage them via CLI or the agent tools. Automatic extraction/saving is disabled by design—keep it simple.
