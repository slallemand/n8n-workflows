# Daily Digest — RSS → pgvector → Mail

Un workflow n8n qui forme un pipeline complet de veille technologique :
1. **Ingestion** : récupère les articles RSS, les embedde et les stocke dans PostgreSQL/pgvector
2. **Digest** : cherche les articles les plus proches de tes centres d'intérêt et envoie un résumé HTML par mail

Les deux phases s'enchaînent automatiquement : l'ingestion déclenche le digest une fois les articles insérés.

---

## Prérequis

### Infrastructure
- **n8n** (self-hosted ou cloud)
- **PostgreSQL** avec l'extension [pgvector](https://github.com/pgvector/pgvector) ≥ 0.5.0
- **FreshRSS** avec l'API GReader activée
- Un compte **Infomaniak AI** (ou toute API compatible OpenAI pour les embeddings)

### Table pgvector

La table est créée automatiquement au premier run par le nœud `Create and Update DB` (voir Phase 1). Schéma de référence :

```sql
CREATE TABLE IF NOT EXISTS rss_articles (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    text        TEXT NOT NULL,
    metadata    JSONB DEFAULT '{}',
    embedding   VECTOR(3584),
    inserted_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index à créer une fois la table peuplée (> 100 lignes)
-- BGE-Multilingual-Gemma2 produit 3584 dims, hors de la limite ivfflat (2000)
-- Utiliser halfvec si pgvector >= 0.7.0, sinon le scan séquentiel est suffisant
-- pour quelques centaines d'articles :
-- CREATE INDEX rss_articles_embedding_idx
--     ON rss_articles USING hnsw ((embedding::halfvec(3584)) halfvec_cosine_ops)
--     WITH (m = 16, ef_construction = 64);
```

### Credentials n8n à configurer
| Credential | Type | Usage |
|---|---|---|
| `pgvector-rss` | PostgreSQL | Connexion à la base pgvector |
| `Bearer Auth account for InfomaniakAI` | HTTP Bearer | API embeddings Infomaniak |
| `InfomaniakAI v2 : OpenAI account` | OpenAI API | LLM pour le résumé (compatible OpenAI) |
| `SMTP account` | SMTP | Envoi du digest par mail |

---

## Mise en route

### 1. Préparer FreshRSS

- Activer l'API GReader : **Paramètres → Authentification → API** → cocher "Autoriser l'accès par API"
- Créer au moins deux catégories d'abonnements (ex. : `ActuIT` et `Domotique`) et y ajouter tes flux RSS

### 2. Configurer les credentials dans n8n

Créer les credentials suivants avant d'importer le workflow :

| Credential | Type n8n | Valeur |
|---|---|---|
| `pgvector-rss` | PostgreSQL | Hôte, port, base de données, utilisateur et mot de passe de ta base PostgreSQL + pgvector |
| `Bearer Auth account for InfomaniakAI` | HTTP Bearer | Token API Infomaniak |
| `InfomaniakAI v2 : OpenAI account` | OpenAI API | Base URL `https://api.infomaniak.com/2/ai/<product_number>/openai/v1` + clé API |
| `SMTP account` | SMTP | Paramètres de ton serveur mail (hôte, port, identifiants) |

> Le `<product_number>` est visible dans la console Infomaniak AI sous le nom de ton déploiement.

### 3. Importer et adapter le workflow

1. Importer `DailyDigest.json` dans n8n
2. Dans le nœud **`FreshRSSLogin`** : renseigner `Email` et `Passwd` avec tes identifiants FreshRSS, et remplacer `<freshrss_url>` par l'URL de ton instance (ex. : `https://rss.example.com`)
3. Dans les nœuds **`FreshRSSGetItems-<first label>`** et **`FreshRSSGetItems-<second label>`** : remplacer `<freshrss_url>` et les noms de labels par tes catégories FreshRSS (ex. : `ActuIT`, `Domotique`)
4. Dans les nœuds **`GetEmbedding`** et **`HTTP Request`** (embedding des intérêts) : remplacer `<product number>` par ton identifiant de produit Infomaniak AI
5. Dans le nœud **`Send an Email`** : renseigner l'adresse `from` (ex. : `NestorAI <nestorai@example.com>`) et `to` (ton adresse de réception)
6. Dans le nœud **`Définir Centres d'Intérêt`** : personnaliser la liste de thèmes selon tes intérêts
7. Lier chaque nœud à son credential via l'interface n8n

### 4. Activer le workflow

Activer le workflow dans n8n. Il se déclenche chaque jour à 7h (fuseau Europe/Paris), ingère les articles FreshRSS des 24 dernières heures, puis envoie le digest par mail.

> **Premier démarrage** : la table `rss_articles` est créée automatiquement lors du premier run. Lance le workflow manuellement une première fois pour vérifier la connectivité avant de laisser le planificateur prendre le relais.

---

## Fichier

**`DailyDigest.json`** — contient les deux phases dans un seul workflow n8n.

---

## Phase 1 — Ingestion RSS → Embeddings → pgvector

### Rôle
Tourne chaque jour à 7h. Récupère les articles publiés dans les dernières 24h depuis FreshRSS, génère un embedding pour chacun, et l'insère dans pgvector en évitant les doublons.

### Flux

```
Schedule Trigger (7h)
  → Create and Update DB   — crée la table si inexistante, purge les articles > 60 jours
  → FreshRSSLogin          — authentification GReader API (email/mot de passe)
  → GetToken               — extrait le token Auth= de la réponse
  → FreshRSSGetItems-ActuIT     ┐  requêtes parallèles par catégorie
  → FreshRSSGetItems-Domotique  ┘  (n=100 et n=20, dernières 24h)
  → Merge                  — fusionne les deux flux
  → Normalize              — nettoyage HTML, détection de langue, format document
  → Dedup                  — déduplique en mémoire par guid/url
  → GetEmbedding           — appel API embeddings (bge_multilingual_gemma2)
  → PrepareInsert          — construit le SQL INSERT avec dédup base de données
  → InsertArticle          — exécute l'INSERT dans rss_articles
      ↓
  [Phase 2 démarre automatiquement]
```

### Nœuds clés

**Init**
Nœud `Execute Query` exécuté en premier à chaque run :
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
Idempotent : crée la table si elle n'existe pas, ajoute `inserted_at` sur une installation existante, et purge les articles de plus de 60 jours pour maintenir la base à taille constante.

**FreshRSSLogin / GetToken**
Utilise l'API GReader de FreshRSS (`/api/greader.php/accounts/ClientLogin`). La réponse contient une ligne `Auth=<token>` utilisée ensuite dans les headers `GoogleLogin auth=<token>`.

**FreshRSSGetItems-ActuIT / Domotique**
Endpoint : `/reader/api/0/stream/contents/user/-/label/<categorie>`
Paramètre `ot` = timestamp Unix de il y a 24h pour ne récupérer que les articles récents. Adapter l'URL et le paramètre `n` (nombre max d'articles) selon tes catégories FreshRSS.

**Normalize**
- Nettoie le HTML du contenu (`contentSnippet`, `summary`, `content`)
- Détecte la langue par comptage de mots-outils (français vs anglais)
- Produit le format attendu par pgvector :
```json
{
  "pageContent": "Titre\n\nRésumé nettoyé",
  "metadata": {
    "id": "guid ou titre",
    "title": "...",
    "url": "...",
    "source": "Nom du flux RSS",
    "published_at": 1778787780,
    "language": "fr"
  }
}
```
> `published_at` est un timestamp Unix en secondes (format GReader).

**Dedup** (en mémoire)
Filtre les doublons dans le batch courant par `metadata.id` ou `metadata.url`. La déduplication en base se fait dans `PrepareInsert` via `WHERE NOT EXISTS`.

**GetEmbedding**
`POST https://api.infomaniak.com/2/ai/<id>/openai/v1/embeddings`
Modèle : `bge_multilingual_gemma2` (3584 dimensions, multilingue FR/EN).
Un appel par article — traitement séquentiel.

**PrepareInsert**
Récupère l'article original depuis `$('Dedup').all()[i]` (corrélation par index d'ordre d'exécution), formate le vecteur `[x,y,z,...]` et construit le SQL :
```sql
INSERT INTO rss_articles (text, metadata, embedding)
SELECT '...', '...'::jsonb, '[...]'::vector
WHERE NOT EXISTS (
  SELECT 1 FROM rss_articles WHERE metadata->>'id' = '...'
)
```

---

## Phase 2 — Digest quotidien

### Rôle
Démarre automatiquement après l'ingestion (via `InsertArticle`). Prend une liste de centres d'intérêt, embed chacun séparément, cherche les articles les plus proches dans pgvector, et envoie un résumé HTML par mail.

> Un nœud `Schedule Trigger1` (désactivé) est conservé comme déclencheur de secours pour lancer le digest indépendamment de l'ingestion.

### Flux

```
[InsertArticle]
  → Définir Centres d'Intérêt  — liste de thèmes séparés par des virgules
  → SplitInterests             — sépare en items individuels (1 par intérêt)
  → HTTP Request               — embed chaque intérêt (bge_multilingual_gemma2)
  → BuildSearchQuery           — construit 1 requête SQL par intérêt
  → Postgres                   — exécute chaque requête (N × 5 résultats)
  → AggregateResults           — déduplique, trie, garde le top 10
  → AI Agent - Résumé          — génère un résumé HTML (Gemma 4 31B)
  → Send an Email              — envoie le digest par SMTP
```

### Nœuds clés

**Définir Centres d'Intérêt**
Nœud `Set` contenant une chaîne de thèmes séparés par des virgules :
```
Intelligence Artificielle, voiture électrique, tesla, domotique, home-assistant, Red Hat, OpenShift, DevSecOps, devops, platform engineering, software supply chain
```
Modifier cette valeur pour adapter la veille à tes sujets.

**SplitInterests**
Découpe la chaîne en items individuels. Chaque intérêt déclenchera un appel embedding séparé et une requête SQL séparée — bien meilleur qu'un seul vecteur "moyen" pour des intérêts disparates.

**HTTP Request** (embedding des intérêts)
Même modèle et même API que l'ingestion (`bge_multilingual_gemma2`). Il est impératif d'utiliser le même modèle aux deux étapes — ingestion et recherche.

**BuildSearchQuery**
Construit une requête SQL de recherche par similarité cosinus (`<=>`) pour chaque intérêt :
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
Points importants :
- `<=>` = distance cosinus (et non `<->` qui est euclidienne — incorrect pour les embeddings de texte)
- Le multiplicateur `0.75` sur les articles français compense un éventuel biais du modèle vers l'anglais
- Le filtre `25 hours` sur `inserted_at` garantit que le digest ne contient que les articles ingérés lors du run du jour — pas de doublons entre deux digests consécutifs (les 25h absorbent les décalages d'horloge)
- `LIMIT 5` par intérêt × N intérêts = pool de candidats large avant déduplication

**AggregateResults**
Reçoit tous les résultats Postgres (N intérêts × 5 articles). Déduplique par `id` en gardant la meilleure distance (article le plus pertinent toutes requêtes confondues), trie, et retourne le top 10 sous forme d'un seul item `{ articles: [...] }`.

**AI Agent - Résumé**
LLM (Gemma 4 31B via Infomaniak) qui reçoit les 10 articles et génère :
- Un résumé en 2 phrases par article, **en français** (même pour les articles anglais)
- Un lien cliquable vers l'article
- Une recommandation finale
- Les articles Actu IT d'abord, puis les articles domotique
- Format de sortie : **HTML** prêt à l'emploi dans un mail (sans balises `<html>`/`<body>`)

**Send an Email**
Nœud SMTP natif n8n, configuré avec `from`, `to`, `subject` et le body HTML : `{{ $json.output }}`.

### Adaptation
- **Changer les intérêts** : modifier le nœud `Définir Centres d'Intérêt`
- **Changer le nombre d'articles** : modifier `LIMIT 5` dans `BuildSearchQuery` et `.slice(0, 10)` dans `AggregateResults`
- **Changer la fenêtre temporelle du digest** : modifier `INTERVAL '25 hours'` dans `BuildSearchQuery`
- **Changer la durée de rétention** : modifier `INTERVAL '60 days'` dans le nœud `Init`
- **Changer le LLM** : modifier le modèle dans le nœud `OpenAI Chat Model`
- **Lancer le digest seul** : activer `Schedule Trigger1` et désactiver la connexion `InsertArticle → Définir Centres d'Intérêt`
- **Ajouter une catégorie FreshRSS** : dupliquer `FreshRSSGetItems-*` et ajouter une entrée dans le `Merge`

---

## Modèle d'embedding

Les deux phases utilisent `bge_multilingual_gemma2` (BAAI/BGE-Multilingual-Gemma2) :
- **9B paramètres**, basé sur Gemma 2
- **3584 dimensions**
- Nativement multilingue (FR, EN, et beaucoup d'autres)
- Bien meilleur que `all-MiniLM-L12-v2` (English-only, 384 dims) pour de la veille mixte FR/EN

> Le modèle doit être **identique** entre l'ingestion et la recherche. Changer de modèle nécessite de vider et re-peupler la table `rss_articles`.
