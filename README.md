# Video Game Deal Discovery Search Engine

A vertical information retrieval (IR) system for **PC games**. Users search for games by title or by attributes such as genre, developer/publisher, popularity, and deal availability. The system ranks relevant games and shows their current deals and prices across stores.

Course project for ITCS441 Text Mining, Faculty of ICT, Mahidol University.

> **Project status (as of Meeting 3):** design stage. The system design and diagrams are written; the data pipeline, Elasticsearch index, backend, and frontend are **not implemented yet**. Every feature below has a status label.

| Label | Meaning |
|---|---|
| ✅ Implemented | Exists in this repository now |
| 🚧 Planned | Part of the MVP, not built yet |
| 💡 Optional/Future | Only if time permits, or after the MVP |

---

## Contents

1. [Overview and problem](#1-overview-and-problem)
2. [Scope and MVP features](#2-scope-and-mvp-features)
3. [Search scenarios and query terms](#3-search-scenarios-and-query-terms)
4. [Technology stack](#4-technology-stack)
5. [System architecture](#5-system-architecture)
6. [Data sources](#6-data-sources)
7. [Data processing workflow](#7-data-processing-workflow)
8. [Entity matching and game identity](#8-entity-matching-and-game-identity)
9. [Data refresh strategy](#9-data-refresh-strategy)
10. [Elasticsearch document, mapping, and store comparison](#10-elasticsearch-document-mapping-and-store-comparison)
11. [Search and ranking](#11-search-and-ranking)
12. [Folder structure](#12-folder-structure)
13. [Prerequisites](#13-prerequisites)
14. [Environment variables and API-key safety](#14-environment-variables-and-api-key-safety)
15. [Setup and run](#15-setup-and-run)
16. [API endpoints (planned contract)](#16-api-endpoints-planned-contract)
17. [Testing](#17-testing)
18. [Known limitations and future work](#18-known-limitations-and-future-work)

---

## 1. Overview and problem

PC gamers on a budget have to check several stores to find a game they would enjoy at a good price. Price-comparison sites help when you already know the title, but they are weak at answering questions like *"cheap RPGs"* or *"Capcom games on sale"*.

This project combines two things:

- **Full-text retrieval and relevance ranking** (BM25 in Elasticsearch) over game information such as titles, descriptions, genres, and tags.
- **Structured filters and deal comparison** over current store prices.

It is meant to be a search engine, not a database lookup: results are ranked by how relevant they are to the query, and deal information is used to filter and re-order the relevant results.

**Target users:** budget-conscious PC gamers who want to find games matching their interests, discover current discounts, and compare prices across stores.

## 2. Scope and MVP features

**Scope: PC games only.** Console-only games are excluded.

| Feature | Status |
|---|---|
| System architecture and sequence diagrams ([docs/diagrams.md](docs/diagrams.md)) | ✅ Implemented |
| Python pipeline: fetch RAWG + CheapShark, clean, filter to PC, match, build game documents | 🚧 Planned |
| Elasticsearch index with explicit mapping and bulk indexing | 🚧 Planned |
| Full-text search with BM25 (title, description, genres, tags, developer/publisher) | 🚧 Planned |
| Filters: genre, store, on sale, maximum price, release year | 🚧 Planned |
| Sort modes: relevance, lowest price, biggest discount, top rated | 🚧 Planned |
| Store-by-store price comparison per game | 🚧 Planned |
| Express REST API | 🚧 Planned |
| HTML/CSS/JavaScript search UI with game cards | 🚧 Planned |
| Daily deal-refresh script | 🚧 Planned |
| "Best deal" sort mode (CheapShark deal rating or a Deal Quality Score) | 💡 Optional/Future |
| Thai intent words (e.g. ราคาถูก → cheap) | 💡 Optional/Future |
| Steam review text | 💡 Optional/Future |
| Early Access / demo badges (only if a source provides reliable data) | 💡 Optional/Future |

## 3. Search scenarios and query terms

The search scenarios agreed by the team in Meeting 2 (genre + cheap, genre + on sale, on sale + year, store + on sale, popular + year, developer/publisher + on sale/cheap, direct title) are listed with example queries in [Testing](#17-testing).

### Query-term definitions

| Term | Definition | Status |
|---|---|---|
| **Year** | The game's **release year**, not a deal date. Titles can contain years (`Football Manager 2024`), so a year is not removed from the text query blindly; a UI year filter is preferred. | Decided |
| **"Cheap"** | The **cheapest current deals among games relevant to the rest of the query**. `cheap RPG` returns RPG games with the cheapest deals ranked high. The whole index is never sorted by price. | Decided |
| **"On sale"** | Requires a current deal with discount > 0; sorted by biggest discount. | Proposed |
| **"Popular"** | Sorted by RAWG rating, with a minimum number of ratings so games with very few votes cannot top the list. | Proposed |
| **Query language** | The data is in English, so BM25 will not match Thai words. English queries plus UI filters are the baseline; mapping a few Thai intent words is optional. | Open |
| **Free games** | Whether $0 games are included in "cheap" results. | Open |

## 4. Technology stack

| Layer | Technology | Responsibility |
|---|---|---|
| Data collection and processing | Python, Pandas | Fetch, inspect, clean, normalize, match, and build game documents |
| Search engine | Elasticsearch | Inverted index, retrieval, BM25 relevance scores, filters |
| Backend | Node.js + Express.js | REST API, query/filter parsing, sort modes, JSON responses |
| Frontend | HTML + CSS + JavaScript | Search UI, filters, game cards, store price comparison |
| API testing | Postman | Manual endpoint testing (not part of the running system) |
| Local environment | Docker (recommended) | Run Elasticsearch consistently |
| Version control | Git + GitHub | Collaboration |

**Team areas:** data processing and the Elasticsearch index; Express backend and API; wireframes and frontend. The data/search side and the backend side agree on the game document schema and API contract together.

## 5. System architecture

The system has two flows that share only the Elasticsearch index:

- **Ingestion / indexing (offline, Python):** RAWG + CheapShark → fetch (raw responses cached to disk) → preprocessing and PC filtering → entity matching → canonical game documents → Elasticsearch (Python client). A separate deal-refresh script updates only the deals.
- **Runtime search:** User → frontend → Express → Elasticsearch (JavaScript client) → Express → frontend.

Python and Express never talk to each other directly. Python does not run during a search.

📄 **Diagrams:** [docs/diagrams.md](docs/diagrams.md) contains the System Architecture Diagram and the runtime Sequence Diagram (Mermaid; GitHub renders them).

## 6. Data sources

| Source | Role | Status |
|---|---|---|
| [RAWG API](https://rawg.io/apidocs) | Game metadata and searchable text: title, description, genres, tags, platforms, release date, rating, images, developers/publishers | 🚧 Planned |
| [CheapShark API](https://apidocs.cheapshark.com/) | Current deals across PC stores: sale price, normal price, savings, deal rating, store, deal link, Steam rating fields | 🚧 Planned |
| Steam reviews API | Review text | 💡 Future (excluded from MVP) |

**Why Steam reviews are excluded from the MVP:** CheapShark's deal data already includes Steam's rating percent, rating text, and rating count, which covers the recommendation signal. Review text would add noise to text search and needs an extra matching step and a terms-of-use check.

Notes:
- **RAWG:** description, developers, and publishers may need one detail request per game (to be confirmed).
- **CheapShark:** some numbers arrive as strings and are converted; `dealRating` (0–10) uses an undocumented formula. Store and currency limits are listed in [Known limitations](#18-known-limitations-and-future-work).

## 7. Data processing workflow

🚧 Planned. Steps:

```text
Fetch API data (cache raw responses to disk)
  → Select useful fields
  → Filter to PC-compatible games
  → Flatten nested structures (e.g. genre objects → ["Action", "RPG"])
  → Clean and normalize values; convert numbers and dates
  → Handle missing values and duplicates
  → Match games across sources
  → Build one document per game
  → Validate documents
  → Bulk index into Elasticsearch
```

**Missing is not zero.** Rules the pipeline follows:
- A record without a source ID or title is rejected.
- Missing description, rating, or release date: the game is kept and the field is left empty. A missing rating is never replaced with 0.
- A deal without a price is excluded from price filters and sorting.
- Early Access / demo status is never assumed to be `false` when the source does not say.
- Titles keep their numbers (`Portal 2`, `Baldur's Gate 3`); search normalization is left to Elasticsearch analyzers.

The pipeline will log statistics: records fetched, PC-compatible records, unique games, matched/unmatched, rejected, and indexed.

## 8. Entity matching and game identity

RAWG, CheapShark, and Steam IDs are separate ID systems; records are never joined just because IDs have the same value.

Each game gets an internal canonical **`game_id`**, which is also its Elasticsearch `_id`. Matching order:
1. A shared external ID when available (e.g. CheapShark's Steam App ID).
2. Otherwise, a normalized title plus other evidence such as release year and platform.
3. Fuzzy title matching only when needed, with manual checks for ambiguous cases (remakes, games sharing a title).

A game does not need to exist in every source. A RAWG game with no current deal is kept with `deals: []`.

## 9. Data refresh strategy

🚧 Planned. Game metadata changes slowly; deals change daily. So there are two scripts:

| Script | Sources | What it does | When |
|---|---|---|---|
| Full pipeline | RAWG + CheapShark | Fetch → process → match → bulk index (overwrites by `game_id`) | Initial load, then monthly or manually |
| Deal refresh | CheapShark only | Re-fetches deals by the stored CheapShark game ID; partially updates `deals`, the `best_*` fields, and `deals_updated_at`; removes ended deals | Daily, and always before demos |

- **`deals_updated_at`** (ISO 8601, UTC) is stored on every game and returned by the API, so the UI can show "Prices as of …".
- **Rate limits:** short delays between requests and retry with backoff on HTTP 429. The RAWG key has a monthly request quota, so raw responses are cached and later steps never call the API.
- **Missing games: delayed fallback.** There is no live API call at search time. The backend logs zero-result queries; the next pipeline run looks them up in RAWG/CheapShark and indexes any games it finds. The main defense is a broad preload (popular PC games + games with current deals).

## 10. Elasticsearch document, mapping, and store comparison

🚧 Planned. **One game = one document**, with each store offer in a `nested` `deals` array so that a store's price and discount stay together.

**Illustrative example only (not real data):**

```json
{
  "game_id": "project_game_001",
  "title": "Example RPG",
  "description": "An open-world role-playing game...",
  "genres": ["RPG", "Action"],
  "tags": ["Open World", "Fantasy"],
  "platforms": ["PC"],
  "developers": ["Example Studio"],
  "publishers": ["Example Publisher"],
  "rating": 4.4,
  "ratings_count": 1520,
  "release_date": "2022-01-01",
  "background_image": "https://example.invalid/game-image.jpg",
  "deals": [
    { "store": "Steam", "price": 19.99, "normal_price": 39.99, "discount": 50.0, "deal_rating": 8.7, "deal_url": "https://example.invalid/deal" },
    { "store": "Epic Games Store", "price": 21.99, "normal_price": 39.99, "discount": 45.0, "deal_rating": 7.9, "deal_url": "https://example.invalid/deal" }
  ],
  "best_price": 19.99,
  "best_discount": 50.0,
  "best_deal_rating": 8.7,
  "rawg_id": 12345,
  "cheapshark_game_id": "67890",
  "deals_updated_at": "2026-11-06T03:00:00Z"
}
```

Fields are included only if the real API responses provide them.

**Draft mapping** (to be finalized after checking real data):

| Field | Type | Purpose |
|---|---|---|
| `title` | `text` + `keyword` subfield | Full-text title search; exact lookup |
| `description` | `text` | Full-text search |
| `genres`, `tags`, `platforms` | `keyword` | Filters and matching |
| `developers`, `publishers` | `keyword` and/or `text` | Filtering and name search |
| `rating` | `float` | "Top rated" sort |
| `ratings_count` | `integer` | Minimum-votes guard for "popular" |
| `release_date` | `date` | Release-year filter |
| `background_image` | `keyword` (not searched) | Image URL for the UI |
| `deals` | `nested` | One object per store offer |
| `deals.store` | `keyword` | Store filter |
| `deals.price`, `deals.normal_price`, `deals.discount`, `deals.deal_rating` | `float` | Price filters, sorting, display |
| `deals.deal_url` | `keyword` (not searched) | Link to the deal |
| `best_price`, `best_discount`, `best_deal_rating` | `float` | Per-game summary computed at index time, for simple sorting |
| `rawg_id`, `cheapshark_game_id` | `keyword` | Source IDs, used by the refresh script |
| `deals_updated_at` | `date` | When deals were last refreshed |

**Store comparison and current price:**
- Each game shows **every current store offer**: store, current price, normal price, discount, and a link to the deal. A game with no current deals shows "No current deals".
- **Lowest current price** = the minimum price among the game's current deals (`best_price`). When a store or price filter is active, only the deals that pass the filter count.
- This is **not a historical low**. CheapShark's ordinary deals response is not assumed to contain historical prices, so the UI will not label the current cheapest price as an all-time low.

## 11. Search and ranking

🚧 Planned.

**Inverted index.** Elasticsearch (Apache Lucene) builds an inverted index for `text` fields: a map from each term to the documents containing it. This makes retrieval fast; it is not the ranking itself.

**BM25.** Elasticsearch's default relevance score. It rewards terms that appear often in a document, gives more weight to rare terms, and normalizes for document length. A BM25 score is relative to the query, not a percentage. `title` gets a higher boost than `description`.

**Filters.** Genre, store, on sale, maximum price, and release year are hard filters, not ranking hints. Store and price filters apply to the `nested` deals, so "Steam games under $20" means a single Steam deal under $20.

**Sort design: relevant set, then sort mode** (proposed, to be confirmed by the team). Meeting 2 agreed on the priority *relevance → price → rating*. A plain tie-breaker would almost never take effect, because BM25 scores rarely tie exactly. So the design is:

1. **Retrieve:** BM25 + hard filters.
2. **Keep the relevant set:** the top-K results, or results scoring at least a fraction of the top score (values not decided).
3. **Apply the sort mode** (UI dropdown; intent words in the query choose the default):

| Sort mode | Default for | Status |
|---|---|---|
| Relevance (BM25) | Most queries | 🚧 Planned |
| Lowest price | "cheap" | 🚧 Planned |
| Biggest discount | "on sale" | 🚧 Planned |
| Top rated (with minimum ratings count) | "popular" | 🚧 Planned |
| Best deal | Chosen in the UI | 💡 Optional, under evaluation |

**Best deal / Deal Quality Score (DQS).** An optional mode under evaluation. Candidates are CheapShark's `dealRating` or a combined Deal Quality Score (savings, price, deal rating). It is only added if an evaluation on judged test queries shows it ranks better than the simple modes. BM25 remains the core relevance ranking either way.

## 12. Folder structure

**Current (actual):**

```text
.
├── .env.example        # placeholder environment variables
├── .gitignore
├── README.md
└── docs/
    └── diagrams.md     # architecture + sequence diagrams (Mermaid)
```

**Planned (🚧, names may change when the code is added):**

```text
data/raw/          # cached raw API responses (gitignored)
data/processed/    # processed game documents
pipeline/          # Python fetch, processing, indexing, deal refresh
backend/           # Express API
frontend/          # HTML/CSS/JS
experiments/       # optional deal-ranking evaluation
```

## 13. Prerequisites

| Tool | Needed for | Version |
|---|---|---|
| Python 3 | Data pipeline | TBD |
| Node.js | Express backend | TBD |
| Elasticsearch | Search index | TBD |
| Docker (recommended) | Running Elasticsearch locally | TBD |
| RAWG API key | Fetching game metadata | Free key from [rawg.io/apidocs](https://rawg.io/apidocs) |

CheapShark's public API does not need a key.

## 14. Environment variables and API-key safety

1. Copy the template: `cp .env.example .env` (Windows PowerShell: `Copy-Item .env.example .env`).
2. Put your own values in `.env`.

| Variable | Purpose | Status |
|---|---|---|
| `RAWG_API_KEY` | RAWG API access | 🚧 Used by the planned pipeline |
| Elasticsearch connection settings | Pipeline and backend | TBD: added when implemented |

**Safety rules:**
- `.env` is in `.gitignore`. Never commit it, and never paste a real key into code, notebooks, issues, or this README.
- Do not print keys in logs or notebook output.
- If a key is committed by mistake, regenerate it on RAWG; deleting the commit is not enough.

## 15. Setup and run

```bash
git clone https://github.com/TeeRachen/ITCS441-Video-Game-Deals-Search-Engine.git
cd ITCS441-Video-Game-Deals-Search-Engine
```

| Component | Instructions |
|---|---|
| Elasticsearch | TBD: added when the index is implemented |
| Data pipeline (full load) | TBD: added when the pipeline is implemented |
| Deal refresh | TBD: added when the refresh script is implemented |
| Backend | TBD: added when the backend is implemented |
| Frontend | TBD: added when the frontend is implemented |

The intended order once built: start Elasticsearch → run the full pipeline → start the backend → open the frontend.

## 16. API endpoints (planned contract)

🚧 Planned. Field names and parameters will be finalized in the Data Schema / API Contract agreement.

| Method | Endpoint | Purpose | Status |
|---|---|---|---|
| GET | `/api/search` | Search with query text, filters, and sort mode | 🚧 Planned |
| GET | `/api/games/:id` | One game with all current deals | 🚧 Planned |
| GET | `/api/genres` | Genre list for the filter UI | 🚧 Planned |
| GET | `/api/stores` | Store list for the filter UI | 💡 Optional |

**Example request (illustrative):**

```http
GET /api/search?q=open+world+RPG&store=Steam&max_price=20&sort=lowest_price
```

**Example response (illustrative, not real data):**

```json
{
  "query": "open world RPG",
  "sort": "lowest_price",
  "total": 1,
  "deals_updated_at": "2026-11-06T03:00:00Z",
  "results": [
    {
      "game_id": "project_game_001",
      "title": "Example RPG",
      "background_image": "https://example.invalid/game-image.jpg",
      "genres": ["RPG", "Action"],
      "rating": 4.4,
      "developers": ["Example Studio"],
      "score": 7.31,
      "best_price": 19.99,
      "deals": [
        { "store": "Steam", "price": 19.99, "normal_price": 39.99, "discount": 50.0, "deal_url": "https://example.invalid/deal" }
      ]
    }
  ]
}
```

With no results, `results` is an empty list. A game with no current deals has `deals: []`.

## 17. Testing

🚧 Planned. Acceptance tests for the MVP. They cover the assignment's required demos (one-word, multiword, partial-match, ranking) and the search scenarios agreed in Meeting 2 (marked M2):

| # | Test | Example | Checks |
|---|---|---|---|
| 1 | One-word query | `RPG` | Relevant games returned with BM25 scores |
| 2 | Multiword query | `open world RPG` | Games matching more terms rank higher |
| 3 | Partial title | `Elden` | Finds `Elden Ring` |
| 4 | Genre + cheap (M2) | `cheap RPG` | RPG games; cheapest current deals first |
| 5 | Genre + on sale (M2) | `RPG on sale` | Only RPG games with a current discount |
| 6 | On sale + year (M2) | `games on sale 2026` | Only games released in 2026 with a current discount |
| 7 | Store + on sale (M2) | `Steam games on sale` | Only Steam deals with a discount |
| 8 | Popular + year (M2) | `popular games 2026` | Games released in 2026, highest rated first |
| 9 | Developer/publisher + on sale or cheap (M2) | `games by Capcom on sale` | Capcom games (by developer/publisher); discounted or cheapest first |
| 10 | Direct title (M2) | `Elden Ring` | The game is the top result |
| 11 | Store comparison | One game with several stores | All current offers shown, lowest price correct |
| 12 | Sort modes | Same query, each mode | Order matches relevance / price / discount / rating |

Tools: Elasticsearch queries run directly (to inspect `_score`), Postman for the API, and the browser for the UI.

**Optional evaluation (💡):** if the Best-deal mode is kept, sort methods are compared on 10–15 test queries with team relevance judgments, using nDCG@10 and Precision@10.

## 18. Known limitations and future work

**Limitations:**
- PC games only, and only stores covered by CheapShark.
- Prices are as of the last refresh, not live. The UI shows "Prices as of …".
- No historical lowest price.
- English text search; Thai queries need UI filters or optional intent-word mapping.
- Matching games across sources by title can be wrong for remakes or shared titles.
- Currency is assumed to be USD until confirmed.
- RAWG's monthly request quota limits how many games can be loaded and refreshed.

**Future work (💡):**
- "Best deal" sort mode (CheapShark deal rating or DQS), if the evaluation supports it.
- Steam review text as a low-weight field or display-only.
- Thai intent-word mapping.
- Historical-low prices, if a reliable source is added.
- Early Access / demo badges, if reliable data is found.
- Automatic scheduling of the deal refresh.
