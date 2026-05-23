# NestorAI — Assistant IA maison via Signal

Nestor est un assistant IA personnel accessible depuis un groupe **Signal**, capable de répondre à des messages texte et des messages vocaux. Il est connecté à la domotique, à la météo locale, et connaît le contexte de la conversation.

> Basé sur [Angie, personal AI assistant with Telegram voice and text](https://n8n.io/workflows/2462-angie-personal-ai-assistant-with-telegram-voice-and-text/) de Derek Cheung. Nestor en reprend l'architecture (gestion vocale/texte, mémoire de session, outils) en remplaçant Telegram par Signal et en ajoutant l'intégration Home Assistant.

---

## Prérequis

### Infrastructure
- **n8n** (self-hosted)
- **signal-cli** avec l'API REST ([signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api))
- **Home Assistant** avec l'API activée et le serveur MCP exposé
- Un compte **Infomaniak AI** (transcription Whisper + LLM)
- **Ollama** (optionnel, utilisé en modèle de fallback)

### Credentials n8n à configurer
| Credential | Type | Usage |
|---|---|---|
| `Signal account` | Signal CLI REST API | Réception et envoi de messages Signal |
| `InfomaniakAI v2 : OpenAI account` | OpenAI API | LLM principal (Gemma 4 31B) |
| `Bearer Auth account for InfomaniakAI` | HTTP Bearer | Transcription audio (Whisper) |
| `Ollama account` | Ollama API | LLM de fallback (Gemma3) |
| `Home Assistant account` | Home Assistant API | Outil météo |
| `HASS_long-living-token` | HTTP Bearer | Accès au serveur MCP Home Assistant |

### Fichier
**`NestorAI - Maison.json`**

---

## Mise en route

### 1. Préparer Signal

- Créer un groupe Signal nommé `NestorAI` (ou tout autre nom — adapter la valeur dans le nœud `AllowList`)
- Inviter le numéro Signal géré par signal-cli dans ce groupe

### 2. Configurer les credentials dans n8n

Créer les credentials suivants avant d'importer le workflow :

| Credential | Type n8n | Valeur |
|---|---|---|
| `Signal account` | Signal CLI REST API | URL de ton instance signal-cli-rest-api |
| `InfomaniakAI v2 : OpenAI account` | OpenAI API | Base URL `https://api.infomaniak.com/1/ai/<product_number>/openai` + clé API |
| `Bearer Auth account for InfomaniakAI` | HTTP Bearer | Token API Infomaniak |
| `Ollama account` | Ollama API | URL de ton instance Ollama |
| `Home Assistant account` | Home Assistant API | URL + token long-lived Home Assistant |
| `HASS_long-living-token` | HTTP Bearer | Token long-lived Home Assistant (pour le serveur MCP) |

> Le `<product_number>` est visible dans la console Infomaniak AI sous le nom de ton déploiement.

### 3. Importer et adapter le workflow

1. Importer `NestorAI - Maison.json` dans n8n
2. Dans le nœud **`AllowList`** : vérifier que la valeur `allow` correspond exactement au nom du groupe Signal
3. Dans le nœud **`HASS MCP`** : remplacer `<hass_url>` par l'URL de ton instance Home Assistant (ex. : `https://homeassistant.local:8123`)
4. Dans le nœud **`Meteo`** : remplacer `weather.<ville>` par l'entité météo de ton Home Assistant (ex. : `weather.maison`)
5. Dans les nœuds **`Speech 2 Text`** et **`GetTranscription`** : remplacer `<product number>` par ton identifiant de produit Infomaniak AI
6. Lier chaque nœud à son credential via l'interface n8n

### 4. Activer le workflow

Activer le workflow dans n8n. Il se déclenche dès qu'un message arrive dans le groupe Signal configuré.

---

## Flux

```
Signal (groupe "NestorAI")
  → GetGroups          — liste les groupes Signal pour retrouver l'ID
  → Marks As Read      — marque le message comme lu (accusé de réception)
  → AllowList          — nom du groupe autorisé ("NestorAI")
  → StoreGroupID       — stocke l'ID du groupe pour la réponse
  → StartTyping        — affiche l'indicateur "en train d'écrire"
  → If1                — vérifie que le message vient bien du groupe autorisé
  → Voice or Text      — extrait le texte et l'ID de session
  → If                 — message vocal (pièce jointe) ou texte ?
      ├─ [vocal] DownloadAttachment → Speech2Text → Wait → GetTranscription → ParseTranscription
      └─ [texte] →
  → Nestor, AI Assistant  — agent LLM avec outils
  → StopTyping
  → Signal             — envoie la réponse dans le groupe
```

---

## Nœuds clés

### Sécurité — AllowList + If1

Seuls les messages provenant du groupe Signal dont le nom correspond à `AllowList` (valeur : `"NestorAI"`) sont traités. Tout message d'un autre groupe ou d'une conversation privée est ignoré silencieusement.

L'ID du groupe est résolu dynamiquement via `GetGroups` au lieu d'être codé en dur, ce qui évite de devoir mettre à jour le workflow si le groupe est recréé.

### Messages vocaux — Speech2Text

Si le message ne contient pas de texte (`messageText` vide), le workflow suppose qu'il s'agit d'un message vocal :

1. `DownloadAttachment` — télécharge la pièce jointe audio depuis signal-cli
2. `Speech2Text` — envoie le fichier à l'API Whisper d'Infomaniak (`POST /audio/transcriptions`)
3. `Wait` (1 s) — laisse le temps à l'API asynchrone de traiter
4. `GetTranscription` — récupère le résultat (retry × 5)
5. `ParseTranscription` — extrait le champ `text` du JSON retourné

Le texte transcrit rejoint ensuite le même pipeline que les messages texte.

### Agent Nestor

LLM principal : **Gemma 4 31B** via Infomaniak AI.
LLM de fallback : **Gemma3** via Ollama local.

**Mémoire** : fenêtre glissante de 10 messages par session (clé de session = numéro Signal de l'expéditeur).

**System prompt** (directives) :
- Filtre les e-mails promotionnels lors des synthèses
- Synthèse mail : expéditeur, date, objet, résumé bref
- Si pas de date précisée → suppose aujourd'hui
- Utilise l'outil Météo pour la météo locale
- Pour le calendrier : filtre les événements non pertinents à la question

**Outils disponibles** :

| Outil | Source | Rôle |
|---|---|---|
| `Meteo` | Home Assistant entity `weather.<ville>` | Météo locale actuelle |
| `HASS MCP` | Home Assistant MCP server (`/api/mcp`) | Contrôle domotique (lumières, volets, capteurs...) |

### Indicateurs de frappe — StartTyping / StopTyping

`StartTyping` est envoyé dès que le groupe autorisé est identifié, avant même le traitement IA. `StopTyping` est envoyé juste avant la réponse finale. L'utilisateur voit ainsi que Nestor a pris en compte le message, même si le LLM prend quelques secondes.

---

## Adaptation

- **Changer le groupe Signal autorisé** : modifier la valeur dans le nœud `AllowList`
- **Ajouter un outil** : brancher un nœud outil (Tool) sur l'entrée `ai_tool` de `Nestor, AI Assistant`
- **Changer le LLM** : modifier le modèle dans `OpenAI Chat Model` (Infomaniak) ou `Ollama Chat Model`
- **Changer la taille de la mémoire** : modifier `contextWindowLength` dans `Simple Memory`
- **Ajouter une entité météo** : modifier le `entityId` dans le nœud `Meteo`
