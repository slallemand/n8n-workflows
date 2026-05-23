# n8n-workflows

A collection of self-hosted [n8n](https://n8n.io) workflows built around a local AI stack: Infomaniak AI (embeddings, LLM), pgvector (semantic search), and Home Assistant (home automation).

> French version: [README.md](README.md)

---

## Workflows

### [RSS-DailyDigest](RSS-DailyDigest/)

A two-phase daily tech-news digest pipeline:

1. **Ingestion** — fetches RSS articles from FreshRSS, generates a per-article embedding (`bge_multilingual_gemma2`, 3584 dims), and stores them in PostgreSQL/pgvector.
2. **Digest** — finds the articles closest to your interests using cosine similarity, then sends an HTML summary by email (Gemma 4 31B LLM).

**Stack**: n8n · FreshRSS · PostgreSQL/pgvector · Infomaniak AI · SMTP

### [NestorAI](NestorAI/)

A personal AI assistant accessible from a **Signal** group, able to handle both text and voice messages. Connected to home automation and local weather via Home Assistant.

**Stack**: n8n · signal-cli · Infomaniak AI · Whisper · Home Assistant (MCP) · Ollama

---

## Common requirements

- **n8n** self-hosted
- **Infomaniak AI** — account with an active deployment (embeddings + LLM)

Each workflow lists its own requirements in its `README.en.md`.

---

## Structure

```
n8n-workflows/
├── RSS-DailyDigest/
│   ├── DailyDigest.json     # n8n workflow to import
│   ├── README.md            # documentation (fr)
│   └── README.en.md         # documentation (en)
├── NestorAI/
│   ├── NestorAI - Maison.json
│   ├── README.md
│   └── README.en.md
└── OLD/                     # archives (unmaintained workflows)
```
