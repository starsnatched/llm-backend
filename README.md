# OS-Agent

`OS-Agent` is a modular toolkit for building autonomous AI assistants powered by [Ollama](https://ollama.com) models. It wraps model interactions with persistent memory, a Linux VM for tool execution and convenient session management. A Discord bot implementation is included.

## Requirements

- Python 3.10+
- Docker installed and accessible to the current user

## Installation

```bash
pip install -r requirements.txt
```

Some Python packages may require system dependencies which should be installed in your Docker image or host system.

## Usage

### Basic Example

```python
import asyncio
import agent

async def main():
    async with agent.TeamChatSession(user="demo", session="test") as chat:
        async for part in chat.chat_stream("Hello there!"):
            print(part)

asyncio.run(main())
```

`TeamChatSession` manages a conversation, stores history in SQLite and executes commands in a per-user Linux container. Mini-agents can be spawned during the chat using the `spawn_agent` tool.

### Command Line Demo

Run the sample script which starts a short conversation:

```bash
python run.py
```

### Discord Bot

Create a `.env` file containing your bot token:

```bash
DISCORD_TOKEN="your-token"
```

Start the bot with:

```bash
python -m bot
```

Uploads sent to the bot are stored in `/data` inside the user's VM. Audio files
are transcribed locally and the transcript is uploaded alongside the original
file. No notification is sent to the LLM for these uploads. Use `!exec <command>` to
run shell commands manually. Administrators can stop the bot with `!shutdown`.

Interactive prompts from the VM appear as regular messages. Simply reply in the
channel to forward your response to the shell. If the VM requests input again,
send another message and it will be forwarded automatically.

### WebSocket Server

Launch a persistent WebSocket service to stream responses and VM notifications:

```bash
python -m agent --host 0.0.0.0 --port 8765
```

The old entry point `python -m agent.server` continues to work as well.

Clients should connect via the WebSocket protocol and can specify the user,
session and optional think parameters using query parameters:
`ws://HOST:PORT/?user=<name>&session=<id>&think=<true|false>`.

Here is a minimal client that keeps the connection open, sends user input and
prints all output from the agent including asynchronous notifications:

```python
import asyncio
from contextlib import suppress
import websockets


async def chat():
    uri = "ws://localhost:8765/?user=demo&session=ws&think=false"
    async with websockets.connect(uri) as ws:
        print("Connected. Press Ctrl+C to exit.")

        async def receiver():
            try:
                async for msg in ws:
                    print(msg, end="", flush=True)
            except websockets.ConnectionClosed:
                pass

        recv_task = asyncio.create_task(receiver())
        try:
            loop = asyncio.get_running_loop()
            while True:
                prompt = await loop.run_in_executor(None, input, "> ")
                await ws.send(prompt)
        except KeyboardInterrupt:
            pass
        finally:
            recv_task.cancel()
            with suppress(asyncio.CancelledError):
                await recv_task


asyncio.run(chat())
```


JSON payloads may also be sent to access other API functions. Each message must
include a ``command`` field and optional ``args`` mapping:

```json
{"command": "list_dir", "args": {"path": "/data"}}
```

Responses for non-streaming commands are JSON encoded. Streaming commands such
as ``team_chat`` send text fragments incrementally.

Interactive commands in the VM may request additional input. When a prompt is
detected—typically a line ending with ``?`` or ``:``—the server sends a JSON
message of the form ``{"stdin_request": "<text>"}`` so clients can respond via
the ``vm_input`` command.



## API Overview

The :mod:`agent` package exposes a simple async API:

- `TeamChatSession` – manage conversations
- `agent.team_chat(prompt, user, session)` – convenience wrapper returning a text stream
- `agent.upload_document(path, user, session)` – place a local file in the VM
- `agent.upload_data(data, filename, user, session)` – upload raw bytes
- `agent.vm_execute(command, user)` – run a command directly
- `agent.vm_execute_stream(command, user)` – stream output from a command
- `agent.vm_send_input(data, user)` – send additional input to the VM shell

Utility helpers exist for listing, reading and writing files as well as editing persistent memory.

### Persistent Memory

User data is stored as JSON and automatically appended to the system prompt.
Memory fields can be modified from a chat using the `manage_memory` tool or programmatically:

```python
import agent
agent.edit_protected_memory("demo", "api_key", "secret")

# or modify memory directly on a session
async with agent.TeamChatSession(user="demo") as chat:
    await chat.edit_memory("api_key", "secret", protected=True)
```

### Notifications

Queue background notifications for an agent using ``send_notification`` or via
an active session:

```python
import agent

# Send a notification without opening a chat session
agent.send_notification("Report ready", user="demo")

async with agent.TeamChatSession(user="demo") as chat:
    await chat.send_notification("Session starting")
    async for part in chat.chat_stream("hello"):
        print(part)
```

## Configuration

Behaviour can be tuned through environment variables:

| Variable | Description |
| --- | --- |
| `OLLAMA_MODEL` | Model name used by Ollama (default `qwen2.5`) |
| `OLLAMA_HOST` | URL of the Ollama server |
| `UPLOAD_DIR` | Directory where uploaded files are stored |
| `DB_PATH` | SQLite database file |
| `VM_IMAGE` | Docker image for the user VM |
| `VM_CONTAINER_TEMPLATE` | Format string for container names (`chat-vm-{user}`) |
| `VM_STATE_DIR` | Host directory for persistent VM state |
| `PERSIST_VMS` | Keep VMs running between sessions (`1` by default) |
| `LOG_LEVEL` | Logging verbosity |

Adjust these variables in your environment or `.env` file.

## Docker VM

Each user receives a dedicated Docker container. Files uploaded through the API are mounted at `/data` in the container and persist according to `VM_STATE_DIR`. Commands are executed exclusively inside this container using `execute_terminal` which streams output back to the model. Local command execution is not supported, so a running VM must be active whenever tools are used.


## TODO

Add multimodal agentic flow.

## License

This project is licensed under the Apache 2.0 License. See [LICENSE](LICENSE) for details.
