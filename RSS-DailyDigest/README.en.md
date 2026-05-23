# Daily Digest — RSS → pgvector → Email

An n8n workflow forming a complete tech monitoring pipeline:
1. **Ingestion**: fetches RSS articles, generates embeddings and stores them in PostgreSQL/pgvector
2. **Digest**: finds the articles closest to your interests and sends an HTML summary by email

The two phases run back-to-back automatically: ingestion triggers the digest once articles are inserted.

---

## Prerequisites

### Infrastructure
- **n8n** (self-hosted or cloud)
- **PostgreSQL** with the [pgvector](https://github.com/pgvector/pgvector) extension ≥ 0.5.0
- **FreshRSS** with the GReader API enabled
- An **Infomaniak AI** account (or any OpenAI-compatible API for embeddings)

### pgvector table

The table is created automatically on the first run by the `Create and Update DB` node (see Phase 1). Reference schema:

```sql
CREATE TABLE IF NOT EXISTS rss_articles (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    text        TEXT NOT NULL,
    metadata    JSONB DEFAULT '{}',
    embedding   VECTOR(3584),
    inserted_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index to create once the table has data (> 100 rows)
-- BGE-Multilingual-Gemma2 produces 3584 dims, which exceeds the ivfflat limit (2000)
-- Use halfvec if pgvector >= 0.7.0, otherwise a sequential scan is fast enough
-- for a few hundred articles:
-- CREATE INDEX rss_articles_embedding_idx
--     ON rss_articles USING hnsw ((embedding::halfvec(3584)) halfvec_cosine_ops)
--     WITH (m = 16, ef_construction = 64);
```

### n8n credentials to configure
| Credential | Type | Purpose |
|---|---|---|
| `pgvector-rss` | PostgreSQL | Connection to the pgvector database |
| `Bearer Auth account for InfomaniakAI` | HTTP Bearer | Infomaniak embeddings API |
| `InfomaniakAI v2 : OpenAI account` | OpenAI API | LLM for the digest (OpenAI-compatible) |
| `SMTP account` | SMTP | Sending the digest by email |

---

## Getting started

### 1. Prepare FreshRSS

- Enable the GReader API: **Settings → Authentication → API** → check "Allow API access"
- Create at least two subscription categories (e.g. `ActuIT` and `Domotique`) and add your RSS feeds to them

### 2. Configure credentials in n8n

Create the following credentials before importing the workflow:

| Credential | n8n type | Value |
|---|---|---|
| `pgvector-rss` | PostgreSQL | Host, port, database name, username and password of your PostgreSQL + pgvector instance |
| `Bearer Auth account for InfomaniakAI` | HTTP Bearer | Infomaniak API token |
| `InfomaniakAI v2 : OpenAI account` | OpenAI API | Base URL `https://api.infomaniak.com/2/ai/<product_number>/openai/v1` + API key |
| `SMTP account` | SMTP | Your mail server settings (host, port, credentials) |

> The `<product_number>` is visible in the Infomaniak AI console under your deployment name.

### 3. Import and configure the workflow

1. Import `DailyDigest.json` into n8n
2. In the **`FreshRSSLogin`** node: fill in `Email` and `Passwd` with your FreshRSS credentials, and replace `<freshrss_url>` with your instance URL (e.g. `https://rss.example.com`)
3. In the **`FreshRSSGetItems-<first label>`** and **`FreshRSSGetItems-<second label>`** nodes: replace `<freshrss_url>` and the label names with your FreshRSS categories (e.g. `ActuIT`, `Domotique`)
4. In the **`GetEmbedding`** and **`HTTP Request`** (interest embedding) nodes: replace `<product number>` with your Infomaniak AI product ID
5. In the **`Send an Email`** node: set the `from` address (e.g. `NestorAI <nestorai@example.com>`) and `to` (your recipient address)
6. In the **`Définir Centres d'Intérêt`** node: customise the comma-separated list of topics to match your interests
7. Link each node to its credential via the n8n interface

### 4. Activate the workflow

Activate the workflow in n8n. It runs every day at 7 AM (Europe/Paris timezone), ingests FreshRSS articles from the past 24 hours, and sends the digest by email.

> **First run**: the `rss_articles` table is created automatically. Trigger the workflow manually once to verify connectivity before letting the scheduler take over.

---

## File

**`DailyDigest.json`** — contains both phases in a single n8n workflow.

---

## Phase 1 — RSS Ingestion → Embeddings → pgvector

### Purpose
Runs every day at 7 AM. Fetches articles published in the last 24 hours from FreshRSS, generates an embedding for each one, and inserts it into pgvector while avoiding duplicates.

### Flow

```
Schedule Trigger (7 AM)
  → Create and Update DB   — creates the table if missing, purges articles older than 60 days
  → FreshRSSLogin          — GReader API authentication (email/password)
  → GetToken               — extracts the Auth= token from the response
  → FreshRSSGetItems-ActuIT     ┐  parallel requests per category
  → FreshRSSGetItems-Domotique  ┘  (n=100 and n=20, last 24h)
  → Merge                  — merges both streams
  → Normalize              — HTML cleanup, language detection, document formatting
  → Dedup                  — in-memory deduplication by guid/url
  → GetEmbedding           — embeddings API call (bge_multilingual_gemma2)
  → PrepareInsert          — builds the INSERT SQL with database-level dedup
  → InsertArticle          — executes the INSERT into rss_articles
      ↓
  [Phase 2 starts automatically]
```

### Key nodes

**Init**
An `Execute Query` node that runs first on every execution:
```sql
CREATE TABLE IF NOT EXISTS rss_articles (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    text        TEXT NOT NULL,
    metadata    JSONB DEFAULT '{}',
    embedding   VECTOR(3584),
    inserted_at TIMESTAMPTZ DEFAULT NOW()
);
ALTER TABLE rss_articles ADD COLUMN IF NOT EXISTS inserted_at TIMESTAMPTZ DEFAULT NOW();
DELETE FROM rss_articles WHERE inserted_at < NOW() - INTERVAL '60 days';
```
Fully idempotent: creates the table if it does not exist, adds `inserted_at` on an existing installation, and purges articles older than 60 days to keep the database at a stable size.

**FreshRSSLogin / GetToken**
Uses the FreshRSS GReader API (`/api/greader.php/accounts/ClientLogin`). The response contains a line `Auth=<token>` which is then used in request headers as `GoogleLogin auth=<token>`.

**FreshRSSGetItems-ActuIT / Domotique**
Endpoint: `/reader/api/0/stream/contents/user/-/label/<category>`
The `ot` parameter is set to the Unix timestamp of 24 hours ago to fetch only recent articles. Adapt the URL and the `n` parameter (max number of articles) to match your FreshRSS categories.

**Normalize**
- Strips HTML from article content (`contentSnippet`, `summary`, `content`)
- Detects language by counting stopwords (French vs English)
- Produces the document format used for storage:
```json
{
  "pageContent": "Title\n\nCleaned summary",
  "metadata": {
    "id": "guid or title",
    "title": "...",
    "url": "...",
    "source": "Feed name",
    "published_at": 1778787780,
    "language": "fr"
  }
}
```
> `published_at` is a Unix timestamp in seconds (GReader format).

**Dedup** (in-memory)
Filters duplicates within the current batch by `metadata.id` or `metadata.url`. Database-level deduplication is handled in `PrepareInsert` via `WHERE NOT EXISTS`.

**GetEmbedding**
`POST https://api.infomaniak.com/2/ai/<id>/openai/v1/embeddings`
Model: `bge_multilingual_gemma2` (3584 dimensions, multilingual FR/EN).
One API call per article — sequential processing.

**PrepareInsert**
Retrieves the original article from `$('Dedup').all()[i]` (index-based correlation relying on sequential execution order), formats the vector `[x,y,z,...]` and builds the SQL:
```sql
INSERT INTO rss_articles (text, metadata, embedding)
SELECT '...', '...'::jsonb, '[...]'::vector
WHERE NOT EXISTS (
  SELECT 1 FROM rss_articles WHERE metadata->>'id' = '...'
)
```

---

## Phase 2 — Daily Digest

### Purpose
Starts automatically after ingestion (via `InsertArticle`). Takes a list of interest topics, embeds each one separately, finds the closest articles in pgvector, and sends an HTML digest by email.

> A `Schedule Trigger1` node (disabled) is kept as a fallback to run the digest independently from ingestion.

### Flow

```
[InsertArticle]
  → Définir Centres d'Intérêt  — comma-separated list of interest topics
  → SplitInterests             — splits into individual items (1 per interest)
  → HTTP Request               — embeds each interest (bge_multilingual_gemma2)
  → BuildSearchQuery           — builds 1 SQL query per interest
  → Postgres                   — executes each query (N × 5 results)
  → AggregateResults           — deduplicates, sorts, keeps top 10
  → AI Agent - Résumé          — generates an HTML digest (Gemma 4 31B)
  → Send an Email              — sends the digest via SMTP
```

### Key nodes

**Définir Centres d'Intérêt** (Define Interests)
A `Set` node containing a comma-separated string of topics:
```
Intelligence Artificielle, voiture électrique, tesla, domotique, home-assistant, Red Hat, OpenShift, DevSecOps, devops, platform engineering, software supply chain
```
Edit this value to match your own monitoring topics.

**SplitInterests**
Splits the string into individual items. Each interest triggers a separate embedding call and a separate SQL query — significantly better than a single "averaged" vector when interests are diverse.

**HTTP Request** (interest embedding)
Same model and API as ingestion (`bge_multilingual_gemma2`). Using the same model at both stages — ingestion and search — is mandatory for meaningful similarity scores.

**BuildSearchQuery**
Builds a cosine similarity SQL query (`<=>`) for each interest:
```sql
SELECT id, text AS contenu, metadata->>'title' AS titre, metadata->>'url' AS lien,
       metadata->>'published_at' AS date_publication, metadata->>'language' AS language,
       (embedding <=> '[...]'::vector)
         * CASE WHEN metadata->>'language' = 'fr' THEN 0.75 ELSE 1.0 END AS distance
FROM rss_articles
WHERE inserted_at > NOW() - INTERVAL '25 hours'
ORDER BY distance ASC
LIMIT 5
```
Key points:
- `<=>` = cosine distance (not `<->` which is Euclidean — wrong for text embeddings)
- The `0.75` multiplier on French articles compensates for any model bias toward English
- The `25 hours` filter on `inserted_at` ensures the digest only includes articles ingested during today's run — no duplicates between two consecutive digests (25h absorbs clock drift and slightly delayed runs)
- `LIMIT 5` per interest × N interests = large candidate pool before deduplication

**AggregateResults**
Receives all Postgres results (N interests × 5 articles). Deduplicates by `id` keeping the best distance score (most relevant match across all queries), sorts, and returns the top 10 as a single item `{ articles: [...] }`.

**AI Agent - Résumé**
LLM (Gemma 4 31B via Infomaniak) that receives the 10 articles and generates:
- A 2-sentence summary per article, **in French** (even for English articles)
- A clickable link to the article
- A final recommendation
- IT news articles first, home automation articles last
- Output format: **HTML** ready to use in an email body (no `<html>`/`<body>` wrapper tags)

**Send an Email**
Native n8n SMTP node, configured with `from`, `to`, `subject` and the HTML body: `{{ $json.output }}`.

### Customisation
- **Change interests**: edit the `Définir Centres d'Intérêt` node
- **Change number of articles**: update `LIMIT 5` in `BuildSearchQuery` and `.slice(0, 10)` in `AggregateResults`
- **Change the digest time window**: update `INTERVAL '25 hours'` in `BuildSearchQuery`
- **Change the retention period**: update `INTERVAL '60 days'` in the `Init` node
- **Change the LLM**: update the model in the `OpenAI Chat Model` node
- **Run the digest standalone**: enable `Schedule Trigger1` and remove the `InsertArticle → Définir Centres d'Intérêt` connection
- **Add a FreshRSS category**: duplicate a `FreshRSSGetItems-*` node and add an input to the `Merge` node

---

## Embedding model

Both phases use `bge_multilingual_gemma2` (BAAI/BGE-Multilingual-Gemma2):
- **9B parameters**, based on Gemma 2
- **3584 dimensions**
- Natively multilingual (FR, EN, and many others)
- Significantly better than `all-MiniLM-L12-v2` (English-only, 384 dims) for mixed FR/EN monitoring

> The model **must be identical** between ingestion and search. Switching models requires truncating and re-populating the `rss_articles` table.
