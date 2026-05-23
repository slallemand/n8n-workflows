# NestorAI — Home AI assistant via Signal

Nestor is a personal AI assistant accessible from a **Signal** group, able to reply to both text and voice messages. It is connected to home automation, local weather, and maintains conversation context.

> Built on top of [Angie, personal AI assistant with Telegram voice and text](https://n8n.io/workflows/2462-angie-personal-ai-assistant-with-telegram-voice-and-text/) by Derek Cheung. Nestor reuses its core architecture (voice/text handling, session memory, tools) while replacing Telegram with Signal and adding Home Assistant integration.

---

## Prerequisites

### Infrastructure
- **n8n** (self-hosted)
- **signal-cli** with the REST API ([signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api))
- **Home Assistant** with the API enabled and the MCP server exposed
- An **Infomaniak AI** account (Whisper transcription + LLM)
- **Ollama** (optional, used as fallback model)

### n8n credentials to configure
| Credential | Type | Purpose |
|---|---|---|
| `Signal account` | Signal CLI REST API | Receiving and sending Signal messages |
| `InfomaniakAI v2 : OpenAI account` | OpenAI API | Primary LLM (Gemma 4 31B) |
| `Bearer Auth account for InfomaniakAI` | HTTP Bearer | Audio transcription (Whisper) |
| `Ollama account` | Ollama API | Fallback LLM (Gemma3) |
| `Home Assistant account` | Home Assistant API | Weather tool |
| `HASS_long-living-token` | HTTP Bearer | Home Assistant MCP server access |

### File
**`NestorAI - Maison.json`**

---

## Getting started

### 1. Prepare Signal

- Create a Signal group named `NestorAI` (or any name — update the value in the `AllowList` node accordingly)
- Invite the Signal number managed by signal-cli into that group

### 2. Configure credentials in n8n

Create the following credentials before importing the workflow:

| Credential | n8n type | Value |
|---|---|---|
| `Signal account` | Signal CLI REST API | URL of your signal-cli-rest-api instance |
| `InfomaniakAI v2 : OpenAI account` | OpenAI API | Base URL `https://api.infomaniak.com/1/ai/<product_number>/openai` + API key |
| `Bearer Auth account for InfomaniakAI` | HTTP Bearer | Infomaniak API token |
| `Ollama account` | Ollama API | URL of your Ollama instance |
| `Home Assistant account` | Home Assistant API | URL + long-lived token of your Home Assistant |
| `HASS_long-living-token` | HTTP Bearer | Home Assistant long-lived token (for the MCP server) |

> The `<product_number>` is visible in the Infomaniak AI console under your deployment name.

### 3. Import and configure the workflow

1. Import `NestorAI - Maison.json` into n8n
2. In the **`AllowList`** node: make sure the `allow` value matches the exact name of your Signal group
3. In the **`HASS MCP`** node: replace `<hass_url>` with your Home Assistant URL (e.g. `https://homeassistant.local:8123`)
4. In the **`Meteo`** node: replace `weather.<ville>` with your Home Assistant weather entity (e.g. `weather.home`)
5. In the **`Speech 2 Text`** and **`GetTranscription`** nodes: replace `<product number>` with your Infomaniak AI product ID
6. Link each node to its credential via the n8n interface

### 4. Activate the workflow

Activate the workflow in n8n. It triggers whenever a message arrives in the configured Signal group.

---

## Flow

```
Signal (group "NestorAI")
  → GetGroups          — lists Signal groups to resolve the group ID
  → Marks As Read      — marks the message as read (read receipt)
  → AllowList          — allowed group name ("NestorAI")
  → StoreGroupID       — stores the group ID for the reply
  → StartTyping        — shows the "typing" indicator
  → If1                — checks the message comes from the allowed group
  → Voice or Text      — extracts text and session ID
  → If                 — voice message (attachment) or text?
      ├─ [voice] DownloadAttachment → Speech2Text → Wait → GetTranscription → ParseTranscription
      └─ [text] →
  → Nestor, AI Assistant  — LLM agent with tools
  → StopTyping
  → Signal             — sends the reply to the group
```

---

## Key nodes

### Security — AllowList + If1

Only messages from the Signal group whose name matches `AllowList` (value: `"NestorAI"`) are processed. Messages from any other group or private conversation are silently ignored.

The group ID is resolved dynamically via `GetGroups` rather than hardcoded, so the workflow doesn't need updating if the group is recreated.

### Voice messages — Speech2Text

If the message contains no text (`messageText` is empty), the workflow assumes it is a voice message:

1. `DownloadAttachment` — downloads the audio attachment from signal-cli
2. `Speech2Text` — sends the file to Infomaniak's Whisper API (`POST /audio/transcriptions`)
3. `Wait` (1 s) — allows the async API time to process
4. `GetTranscription` — fetches the result (retry × 5)
5. `ParseTranscription` — extracts the `text` field from the returned JSON

The transcribed text then joins the same pipeline as text messages.

### Nestor agent

Primary LLM: **Gemma 4 31B** via Infomaniak AI.
Fallback LLM: **Gemma3** via local Ollama.

**Memory**: sliding window of 10 messages per session (session key = sender's Signal number).

**System prompt directives**:
- Filter promotional emails when summarising
- Email summary format: sender, date, subject, brief summary
- If no date specified → assume today
- Use the Weather tool for local weather
- For calendar questions: filter out irrelevant events

**Available tools**:

| Tool | Source | Purpose |
|---|---|---|
| `Meteo` | Home Assistant entity `weather.<ville>` | Current local weather |
| `HASS MCP` | Home Assistant MCP server (`/api/mcp`) | Home automation control (lights, shutters, sensors...) |

### Typing indicators — StartTyping / StopTyping

`StartTyping` is sent as soon as the allowed group is identified, before any AI processing begins. `StopTyping` is sent just before the final reply. The user can see that Nestor has acknowledged the message even while the LLM takes a few seconds to respond.

---

## Customisation

- **Change the allowed Signal group**: edit the value in the `AllowList` node
- **Add a tool**: wire a Tool node to the `ai_tool` input of `Nestor, AI Assistant`
- **Change the LLM**: update the model in `OpenAI Chat Model` (Infomaniak) or `Ollama Chat Model`
- **Change memory size**: update `contextWindowLength` in `Simple Memory`
- **Change the weather entity**: update the `entityId` in the `Meteo` node
