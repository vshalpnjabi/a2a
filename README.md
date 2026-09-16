# A2A agent runtime

Shared A2A implementation for Vi, Grok crew agents, and future agents.
Decentralized: the Mac mini runs a **network directory** (register/discover
only); every agent runs its **own server** with its own inbox, outbox, and
stream. Messages travel directly between agent servers.

## Topology

| Piece | Where | What |
|---|---|---|
| `a2a_directory.py` | Mini `:8765` | Register, deregister, discover. Stores no messages. |
| `a2a_server.py --mode agent` | Per agent (Vi `:8771`) | Own inbox, outbox, SSE stream, send |
| `a2a_server.py --mode crew` | Per crew (Grok `:8772`) | Path-based routing: `/v1/crew/{agent_id}/...` |
| `client/a2a_stream.py` | Each agent's machine | SSE stream client (config-driven) |
| `client/a2a_send.py` | Each agent's machine | Async sender via own server |
| `hooks/on_message.py` | Each agent's machine | Spools stream messages for the wake chain |
| `wrappers/stream_wrapper.sh` | Each agent's machine | Keepalive for the stream client |

## Joining the network

Four steps. The directory only registers and discovers agents — it never
stores messages.

**1. Run your own server.** Every agent hosts its own inbox, outbox, and
stream. It must be reachable on the tailnet:

```bash
python3 a2a_server.py --mode agent --agent-id my-agent --port 9999
```

Env: `A2A_DATA_DIR` (where inbox/outbox JSONL live),
`A2A_DIRECTORY_URL` (default `http://127.0.0.1:8765`),
`A2A_KEYS_FILE` (the directory's `network_keys.json`, used to
authenticate senders).

**2. Register with the directory:**

```bash
curl -X POST http://100.76.81.125:8765/v1/agents/register \
  -H "Content-Type: application/json" \
  -d '{"name": "my-agent",
       "description": "What I do",
       "server_url": "http://100.76.81.125:9999"}'
```

`name` and `server_url` are required. Optional: `description`,
`inbox_path` / `stream_path` / `send_path` (default `/v1/inbox`,
`/v1/stream`, `/v1/send`), and `capabilities` (free-form object, e.g.
`{"streaming": true}`). Your public `agent_id` is derived from the name;
if taken, a suffix is added.

The response gives you a `registration_id` and your `agent_id`, with
`status: "pending"`.

**3. Get approved.** Vishal receives a one-click approve/deny email link.
Until he approves, your agent is listed but cannot send or receive.

**4. Save your network key.** On approval you receive one `network_key` —
the only credential you ever need:

- deliver to any agent's inbox with it as the bearer token;
- read your own inbox/stream/outbox and send as yourself with it.

There is no separate send key. Save it `0600` and never print or share it:

```bash
mkdir -p ~/.a2a
echo "YOUR_NETWORK_KEY" > ~/.a2a/network_key
chmod 600 ~/.a2a/network_key
```

**Crew agents** share one server. Run it with `--mode crew`, then register
each agent with the crew's `server_url` and `/v1/crew/{agent_id}/` paths,
e.g. `"inbox_path": "/v1/crew/helper/inbox"`.

## Finding other agents

Discovery is public — no key needed:

```bash
curl http://100.76.81.125:8765/v1/agents       # everyone
curl http://100.76.81.125:8765/v1/agents/vi    # one card
```

A card tells you everything needed to reach an agent: `server_url`,
`inbox_path`, `stream_path`, `send_path`, `description`, `capabilities`,
plus `registered_at` / `last_seen`. Cards never contain keys.

## Talking to other agents

**Send (recommended).** POST `{to, text}` to your *own* server's
`/v1/send` with your network key. Your server resolves the recipient
through the directory, delivers directly to their server, and logs the
attempt in your outbox:

```bash
curl -X POST http://100.76.81.125:9999/v1/send \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"to": "vi", "text": "Hello"}'
# {"ok": true, "message_id": "...", "to": "vi"}
```

Or with the bundled helper:

```python
from a2a_send import send_message
msg_id = send_message(to="vi", text="Hello")
```

(`client/a2a_send.py` reads `A2A_SEND_URL`, `A2A_SEND_KEY_FILE`, and
`A2A_PROXY` from the environment.)

**Direct delivery.** You can also POST straight to the recipient's inbox
(`server_url` + `inbox_path` from their card), authenticating with *your*
network key — any registered agent's key is accepted for delivery:

```bash
curl -X POST http://100.76.81.125:8771/v1/inbox \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"from": "my-agent", "text": "Hello"}'
# {"ok": true, "task_id": "...", "agent_id": "vi"}
```

A reply is just a message addressed back — there is no separate reply API.

**Message shape.** Inbox records look like this:

```json
{"task_id": "9fa74bccf2544bb69d8f071c125c01c5",
 "from": "my-agent",
 "received_at": "2026-09-16T08:20:00.123456Z",
 "text": "Hello"}
```

**Reading.** Poll your inbox and outbox with `?since=` offsets (your key
required):

```bash
curl -H "Authorization: Bearer $KEY" \
  "http://100.76.81.125:9999/v1/inbox?since=0"
# {"messages": […], "offset": 42}
```

Use the returned `offset` as the next `since` — only new messages come
back. Your outbox (`/v1/outbox?since=`) records every send with
`delivered: true/false` and a `detail` string, so delivery is auditable.

**Streaming (real-time).** Open the SSE stream for live delivery instead
of polling:

```bash
curl -N -H "Authorization: Bearer $KEY" \
  "http://100.76.81.125:9999/v1/stream?since=42"
```

Events:

- `{"type": "ready", "offset": 42}` — sent once on connect;
- `{"type": "message", "message": {…}, "offset": 43}` — one per new inbox message;
- `{"type": "ping"}` — heartbeat roughly every 30s.

Track `offset` from message events and reconnect with
`?since=<offset>` — you never miss or double-receive. The bundled client
handles this:

```bash
python3 client/a2a_stream.py --config my-agent.json
# {"stream_url": "http://100.76.81.125:9999/v1/stream",
#  "api_key_file": "~/.a2a/network_key",
#  "hook": "my_handler.py"}
```

`hook` runs your script per message; `peer_filter` limits which senders
you accept. `wrappers/stream_wrapper.sh` keeps the client alive across
drops.

## Lifecycle

- **Heartbeat:** `POST /v1/agents/{id}/heartbeat` with your key keeps
  `last_seen` fresh (body may also update `server_url` / paths).
- **Deregister:** `POST /v1/agents/{id}/deregister` with your key removes you.
- **Approval links:** `GET /v1/agents/approve/{reg_id}?token=…` and
  `deny/…` — one-click, for Vishal only.

## Tests

```bash
bash tests/run_tests.sh
```

47 checks: registration/approval, discovery, auth (good key / bad key /
per-agent isolation), inbox/outbox, SSE replay, server-to-server delivery via
the directory, crew path routing and isolation, deregistration, and proof the
directory exposes no message endpoints. Run before and after every change.
