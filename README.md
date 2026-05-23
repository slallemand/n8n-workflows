# n8n-workflows

Collection de workflows [n8n](https://n8n.io) self-hosted, construits autour d'une stack IA locale : Infomaniak AI (embeddings, LLM), pgvector (recherche sémantique) et Home Assistant (domotique).

> Version anglaise : [README.en.md](README.en.md)

---

## Workflows

### [RSS-DailyDigest](RSS-DailyDigest/)

Pipeline de veille technologique quotidienne en deux phases :

1. **Ingestion** — récupère les articles RSS depuis FreshRSS, génère un embedding par article (`bge_multilingual_gemma2`, 3584 dims) et les stocke dans PostgreSQL/pgvector.
2. **Digest** — cherche les articles les plus proches de tes centres d'intérêt par similarité cosinus, et envoie un résumé HTML par mail (LLM Gemma 4 31B).

**Stack** : n8n · FreshRSS · PostgreSQL/pgvector · Infomaniak AI · SMTP

### [NestorAI](NestorAI/)

Assistant IA personnel accessible depuis un groupe **Signal**, capable de répondre à des messages texte et vocaux. Connecté à la domotique et à la météo locale via Home Assistant.

**Stack** : n8n · signal-cli · Infomaniak AI · Whisper · Home Assistant (MCP) · Ollama

---

## Prérequis communs

- **n8n** self-hosted
- **Infomaniak AI** — compte avec un déploiement actif (embeddings + LLM)

Chaque workflow liste ses propres prérequis dans son `README.md`.

---

## Structure

```
n8n-workflows/
├── RSS-DailyDigest/
│   ├── DailyDigest.json     # workflow n8n à importer
│   ├── README.md            # documentation (fr)
│   └── README.en.md         # documentation (en)
├── NestorAI/
│   ├── NestorAI - Maison.json
│   ├── README.md
│   └── README.en.md
└── OLD/                     # archives (workflows non maintenus)
```
