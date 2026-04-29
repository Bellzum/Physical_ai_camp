# PhysicalAIcamp Voice Agent

Web and IoT-ready voice agent demo built from Agora's Python Conversational AI quickstart.

This workspace uses `agent-quickstart-python/` for the runnable app. It contains a Next.js web frontend and a Python FastAPI backend. The browser or an IoT device joins an Agora RTC channel, then the backend starts an Agora Conversational AI agent in that same channel. Audio travels through Agora RTC; the backend only creates tokens and controls the agent lifecycle.

## What This Includes

- Web frontend: `agent-quickstart-python/web-client/`
- Python backend: `agent-quickstart-python/server-python/`
- Agora Conversational AI managed provider pipeline:
  - STT: Deepgram `nova-3`
  - LLM: OpenAI `gpt-4o-mini`
  - TTS: MiniMax `speech_2_6_turbo`
- RTC plus RTM token generation from `AGORA_APP_ID` and `AGORA_APP_CERTIFICATE`
- IoT device protocol support through the Python backend
- Optional ngrok tunnel for testing from external devices

## Prerequisites

- Bun
- Python 3.8+
- Agora CLI: `agoraio-cli`
- Agora project named `PhysicalAIcamp` with RTC, RTM, and Conversational AI enabled
- Agora App ID and Primary Certificate
- Optional: ngrok

## Configure Agora

From the workspace root:

```bash
cd agent-quickstart-python
agora project use PhysicalAIcamp
agora project env write server-python/.env.local --with-secrets
agora project doctor
```

The backend reads:

```text
agent-quickstart-python/server-python/.env.local
```

Required values:

```bash
AGORA_APP_ID=your_agora_app_id
AGORA_APP_CERTIFICATE=your_primary_certificate
PORT=8000
```

Do not expose the Primary Certificate to the browser or device.

## Install

```bash
cd agent-quickstart-python
bun install
```

## Run Locally

```bash
cd agent-quickstart-python
bun run dev
```

Local URLs:

- Web frontend: http://localhost:3000
- Python backend: http://localhost:8000
- Backend API docs: http://localhost:8000/docs

In local mode, the Next.js frontend calls `/api/*`, and those route handlers proxy to the Python backend at `http://localhost:8000`.

## Web Test Flow

1. Open http://localhost:3000.
2. Allow microphone access.
3. Click `Try it now`.
4. Speak to the agent.
5. Confirm you hear the agent response and see transcript/state updates.

## Ngrok

Expose the web app:

```bash
ngrok http 3000
```

Expose the Python backend for IoT devices:

```bash
ngrok http 8000
```

Use the port 8000 ngrok URL as the IoT device backend base URL. Use the port 3000 ngrok URL only for the web frontend.

## Backend API

### Get Config

```http
GET /get_config
```

Example:

```bash
curl "http://localhost:8000/get_config"
```

Optional query parameters:

- `channel`: reuse a channel name
- `uid`: reuse a numeric user RTC UID

Example response:

```json
{
  "code": 0,
  "data": {
    "app_id": "your_agora_app_id",
    "token": "007...",
    "uid": "4321",
    "channel_name": "device-001-session",
    "agent_uid": "58888506"
  },
  "msg": "success"
}
```

### Start Agent

```http
POST /v2/startAgent
Content-Type: application/json
```

Standard request:

```bash
curl -X POST "http://localhost:8000/v2/startAgent" \
  -H "Content-Type: application/json" \
  -d '{"channelName":"device-001-session","rtcUid":58888506,"userUid":4321}'
```

IoT G.722 request:

```bash
curl -X POST "http://localhost:8000/v2/startAgent" \
  -H "Content-Type: application/json" \
  -d '{"channelName":"device-001-session","rtcUid":58888506,"userUid":4321,"parameters":{"output_audio_codec":"g722"}}'
```

Save the returned `data.agent_id`.

### Stop Agent

```bash
curl -X POST "http://localhost:8000/v2/stopAgent" \
  -H "Content-Type: application/json" \
  -d '{"agentId":"agent-runtime-id"}'
```

## IoT Device Flow

1. Call `GET /get_config`.
2. Read `data.app_id`, `data.token`, `data.channel_name`, `data.uid`, and `data.agent_uid`.
3. Join Agora RTC with:
   - App ID: `data.app_id`
   - Channel: `data.channel_name`
   - Token: `data.token`
   - Local RTC UID: numeric value of `data.uid`
4. Start the agent with `/v2/startAgent`.
5. Send microphone audio through Agora RTC.
6. Listen for agent audio from the same RTC channel.
7. Stop the agent with `/v2/stopAgent`.
8. Leave RTC and release device audio resources.

The `userUid` sent to `/v2/startAgent` must match the UID the device used to join RTC.

## Token Renewal

Tokens expire after 3600 seconds. Renew before expiry by calling `/get_config` with the same channel and UID:

```bash
curl "http://localhost:8000/get_config?channel=device-001-session&uid=4321"
```

Pass the returned `data.token` to the Agora RTC token renewal API. Do not change channel or UID during an active session.

## Verification

```bash
cd agent-quickstart-python
bun run doctor:local
bun run verify:backend
bun run verify:local:fastapi
bun run verify:web:proxy
```

Full local verification:

```bash
cd agent-quickstart-python
bun run verify:local
```

## Useful Commands

```bash
cd agent-quickstart-python

bun run dev                    # Start frontend and backend
bun run backend                # Start Python backend only
bun run frontend               # Start Next frontend only
bun run doctor:local           # Check local env
bun run verify:backend         # Compile-check Python backend
bun run verify:local           # Full local verification
bun run clean                  # Remove generated dependency/build artifacts
```

## Troubleshooting

| Problem | Check |
| --- | --- |
| Backend env missing | Run `agora project env write server-python/.env.local --with-secrets` inside `agent-quickstart-python/` |
| Agora auth errors | Confirm `AGORA_APP_ID` and `AGORA_APP_CERTIFICATE` in `server-python/.env.local` |
| Project not ready | Run `agora project doctor`; enable RTC, RTM, and ConvoAI if needed |
| Frontend cannot reach backend | Confirm `bun run dev` is running and backend is on port 8000 |
| Device cannot reach backend | Use an ngrok tunnel to port 8000 |
| Web app works but IoT does not | Confirm the device joins RTC with the same `uid` and `channel_name` sent to `/v2/startAgent` |
| Agent starts but no audio returns | Confirm the device sends audio through Agora RTC and requests `output_audio_codec: "g722"` if required |

## License

See `agent-quickstart-python/LICENSE`.
