# Architecture: Feed Reader v2

## Overview

Evolve from a set of CLI scripts orchestrated by a Claude Code agent into a
self-contained web application that drives its own workflow and uses LLMs as a
pluggable service.

```
┌──────────────────────────────────────────────────┐
│                   Web GUI                        │
│          (FastAPI + Jinja2 + htmx)               │
├──────────────────────────────────────────────────┤
│               Digest Pipeline                    │
│   fetch → dedupe → scrape → summarize → curate    │
├────────────┬──────────────┬──────────────────────┤
│  SQLite DB │  LLM Provider│   Scraper            │
│  (feeds,   │  (Anthropic, │   (Playwright /      │
│   posts,   │   OpenAI,    │    PyMuPDF)           │
│   digests) │   Ollama)    │                       │
└────────────┴──────────────┴──────────────────────┘
```

---

## 1. Ubiquitous Language

These terms have precise meanings throughout the codebase. Use them
consistently in code, comments, and conversation.

**Feed** — A curated RSS/Atom subscription to a single publication or author.
The user has deliberately chosen to follow it. A Feed has a one-to-many
relationship with Posts: every Post it produces is included in the next Digest.

**Aggregator** — A discovery source (Hacker News, Lobsters, Reddit) that
surfaces links to content published elsewhere. An Aggregator has a many-to-many
relationship with Posts: a Post may appear on multiple Aggregators. The LLM
selects which Aggregator Posts to include in a Digest.

**Post** — A single piece of content identified by its canonical URL. The
deduplication unit across all sources. If a blog post appears both in a curated
Feed and on HN, it is one Post with two provenance records. Carries a title,
publication date, and an optional feed-provided excerpt.

**Excerpt** — The feed-provided snippet attached to a Post. Not an Article.
This is the author's blurb, not the full content. Column name: `excerpt`
(not `summary`, to avoid collision with LLM-generated summaries).

**Article** — The full retrieved text of a Post, stored as Markdown. A cached
value object: if re-retrieved, a new Article replaces the old one. Only Posts
from curated Feeds are retrieved into Articles; Aggregator-only Posts are
judged by title and excerpt alone.

**Canonical URL** — The normalised form of a Post's link, used as the
deduplication key. Strips UTM parameters, fragments, trailing slashes, and
`www.` prefix. Column name: `canonical_url` (not `link`).

**Digest** — The output of one pipeline run: an LLM-generated summary of Posts
from a given time window, with reading recommendations and notable links. A
Digest is immutable once complete. It is the primary artifact the user
consumes.

**Time Window** — The lookback period for a Digest, expressed in hours
(default: 36). A domain concept, not just a CLI flag.

**Snapshot** — Transient Playwright accessibility-tree output (YAML). An
infrastructure artifact, not a domain concept. Created during Article retrieval
and immediately discarded after conversion to Markdown.

**LLM Provider** — An external service that summarises Articles and generates
Digests. Infrastructure, not domain logic. The curation policy lives in the
prompt templates (application layer); the LLM executes them.

---

## 2. Domain Model

### Single Bounded Context, Three Subdomains

This is a personal tool built by one developer. A bounded context is a
deployment/team boundary — there is exactly one. Within it, three functional
subdomains cluster naturally:

**Source Management** — The registry of Feed and Aggregator sources. Answers:
*where do we look?*

**Content Acquisition** — Fetching, deduplicating, and retrieving content.
Answers: *what did those sources publish?*

**Digest Production** — Selecting, summarising, and presenting Posts as a
Digest. Answers: *what is worth reading?*

These are subdomains, not separate contexts. They share the same Post model,
the same database, and the same process.

### Entities, Value Objects, Services

#### Entities (identity persists across mutations)

| Entity | Identity | Notes |
|---|---|---|
| Feed | URL (unique) | Mutable: can be enabled/disabled, title refreshed |
| Aggregator | URL (unique) | Has `source_type` discriminator (hackernews, lobsters, reddit) |
| Post | Canonical URL (unique) | Central entity. Gains an Article over time, appears in multiple Digests |
| Digest | Surrogate ID + run_at | Has lifecycle: running → done / failed. Immutable once done |

#### Value Objects (defined by attributes, no independent identity)

| Value Object | Notes |
|---|---|
| Canonical URL | Normalised URL string. The dedup key. Two equal strings = same Post |
| Excerpt | Feed-provided snippet. Immutable once set from source |
| Article | Cached full-text markdown. Replaceable on re-retrieval, not versioned |
| Time Window | The `hours_back` integer parameterising a Digest run |
| Digest Status | Enum: `running`, `done`, `failed`. One-way transitions |
| DigestPost | Junction: (digest_id, post_id) + LLM summary. No identity outside its Digest |

**Why Article is a Value Object, not an Entity:** An Article has no lifecycle
independent of its Post. It is fetched, cached, and replaced wholesale. A
re-retrieval produces a new value, not an update to an existing one. There is
no scenario where two Articles for the same Post coexist.

#### Service Objects

| Service | Type | Responsibility |
|---|---|---|
| Digest Pipeline | Application Service | Orchestrates: fetch → dedupe → scrape → summarise → curate → store |
| Deduplication Service | Domain Service | Canonical URL normalisation + "curated wins" merge rule |
| LLM Provider | Infrastructure Service | `summarize_article()` and `generate_digest()` behind a Protocol |
| Article Retrieval | Infrastructure Service | Playwright/PDF → Markdown |
| Snapshot Converter | Infrastructure Service | YAML accessibility tree → Markdown. Pure function |
| Feed Parser | Infrastructure Service | RSS/Atom wire format → raw Post data |

---

## 3. SQLite Schema

Feeds and aggregators are separate concepts with different relationships to
posts:

- A **feed** is a curated RSS/Atom source (a blog). Each post belongs to
  exactly one feed (one-to-many).
- An **aggregator** is a discovery source (HN, Lobsters, Reddit). Posts appear
  on multiple aggregators and an aggregator links to many posts (many-to-many).

```sql
-- Curated RSS/Atom feeds (one-to-many with posts)
CREATE TABLE feeds (
    id          INTEGER PRIMARY KEY,
    url         TEXT NOT NULL UNIQUE,
    title       TEXT,
    enabled     INTEGER NOT NULL DEFAULT 1,
    added_at    DATETIME NOT NULL DEFAULT (datetime('now'))
);

-- Aggregator sources (many-to-many with posts)
CREATE TABLE aggregators (
    id          INTEGER PRIMARY KEY,
    url         TEXT NOT NULL UNIQUE,
    name        TEXT,                            -- "HN frontpage", "r/programming", etc.
    source_type TEXT NOT NULL,                   -- hackernews, lobsters, reddit
    enabled     INTEGER NOT NULL DEFAULT 1,
    added_at    DATETIME NOT NULL DEFAULT (datetime('now'))
);

-- Posts (deduplicated by canonical URL)
CREATE TABLE posts (
    id            INTEGER PRIMARY KEY,
    feed_id       INTEGER REFERENCES feeds(id) ON DELETE SET NULL,
    canonical_url TEXT NOT NULL UNIQUE,           -- normalised dedup key
    title         TEXT,
    published_at  DATETIME,
    fetched_at    DATETIME NOT NULL DEFAULT (datetime('now')),
    excerpt       TEXT                            -- feed-provided; NOT an LLM summary
);

-- Junction: which aggregators surfaced which posts
CREATE TABLE aggregator_posts (
    aggregator_id INTEGER NOT NULL REFERENCES aggregators(id) ON DELETE CASCADE,
    post_id       INTEGER NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    PRIMARY KEY (aggregator_id, post_id)
);

-- Full article content (cached value object)
CREATE TABLE articles (
    id            INTEGER PRIMARY KEY,
    post_id       INTEGER NOT NULL UNIQUE REFERENCES posts(id) ON DELETE CASCADE,
    retrieved_at  DATETIME NOT NULL DEFAULT (datetime('now')),
    content_md    TEXT
);

-- Digest runs
CREATE TABLE digests (
    id         INTEGER PRIMARY KEY,
    run_at     DATETIME NOT NULL DEFAULT (datetime('now')),
    hours_back INTEGER NOT NULL DEFAULT 36,
    status     TEXT NOT NULL DEFAULT 'running',  -- running, done, failed
    summary_md TEXT,                              -- LLM-generated digest
    model      TEXT
);

-- Which posts were included in a digest
CREATE TABLE digest_posts (
    digest_id    INTEGER NOT NULL REFERENCES digests(id) ON DELETE CASCADE,
    post_id      INTEGER NOT NULL REFERENCES posts(id),
    post_summary TEXT,                            -- per-article LLM summary
    PRIMARY KEY (digest_id, post_id)
);

CREATE TABLE settings (
    key   TEXT PRIMARY KEY,
    value TEXT
);

CREATE INDEX idx_posts_feed ON posts(feed_id, published_at DESC);
CREATE INDEX idx_posts_published ON posts(published_at DESC);
CREATE INDEX idx_digests_run_at ON digests(run_at DESC);
```

Key decisions:
- `posts.canonical_url` — named for the domain concept, not feedparser's field.
- `posts.excerpt` — avoids the `summary` name collision with LLM summaries.
- `posts.feed_id` — nullable FK. Aggregator-only posts have no feed.
  Simple and honest about the one edge case.
- `articles.retrieved_at` — domain language, not `scraped_at`.
- Interesting links and recommendations stay embedded in `summary_md`. YAGNI
  — they have no independent lifecycle for a personal tool. Extractable later
  if needed.
- Tables are minimal. We add columns when we need them.

---

## 4. LLM Provider Abstraction (replaces agent-as-orchestrator)

### Inversion of Control

**Before:** Claude Code skill calls a fetch script, reads output, writes digest.
The LLM is the orchestrator.

**After:** Python app drives the pipeline and calls an LLM API for
summarization. The LLM is a service, not the controller.

The curation policy lives in the prompt templates (application layer). The LLM
executes them. Swapping providers does not change domain behaviour.

### Interface

```python
# llm.py
from typing import Protocol
from dataclasses import dataclass

@dataclass
class DigestResult:
    summary: str                   # full digest markdown
    included_posts: list[str]      # canonical URLs the LLM chose to feature
    interesting_links: list[str]   # links surfaced from within articles

class LLMProvider(Protocol):
    def summarize_article(self, content: str, title: str, url: str) -> str: ...
    def generate_digest(self, posts: list[dict]) -> DigestResult: ...
```

### Providers

| Provider | Class | Config |
|---|---|---|
| Anthropic | `AnthropicProvider` | `LLM_API_KEY` |
| OpenAI | `OpenAICompatibleProvider` | `LLM_API_KEY` |
| Ollama | `OpenAICompatibleProvider` | `LLM_BASE_URL=http://localhost:11434/v1` |
| Any OpenAI-compat | `OpenAICompatibleProvider` | `LLM_BASE_URL`, `LLM_API_KEY` |

`OpenAICompatibleProvider` uses the `openai` SDK with a custom `base_url`,
covering OpenAI, Ollama, Together, Groq, etc. in one class.

### Configuration

```bash
LLM_PROVIDER=anthropic          # anthropic | openai | ollama | openai-compatible
LLM_MODEL=claude-sonnet-4-20250514
LLM_API_KEY=sk-ant-...
LLM_BASE_URL=                   # only for ollama / openai-compatible
```

Also configurable via the `settings` table / web UI.

---

## 5. Digest Pipeline (new orchestration)

```python
# digest.py
def run_digest(db, hours: int = 36) -> DigestResult:
    llm = get_provider()

    # 1. Fetch curated feeds → posts (one-to-many)
    feed_posts = fetch_feeds(db, hours)

    # 2. Fetch aggregators → posts (many-to-many, all top posts for the period)
    aggregator_posts = fetch_aggregators(db, hours)

    # 3. Deduplicate by canonical URL (curated version wins)
    all_posts = deduplicate(feed_posts, aggregator_posts)

    # 4. Retrieve articles for curated feed posts not already cached
    curated = [p for p in all_posts if p.feed_id]
    with browser() as b:
        for post in curated:
            if not already_cached(db, post):
                content = b(post.canonical_url)
                store_article(db, post, content)

    # 5. Summarize retrieved articles via LLM
    for post in curated:
        if not post.summary:
            article = load_article(db, post)
            post.summary = llm.summarize_article(article, post.title, post.canonical_url)

    # 6. Generate digest via LLM — receives ALL posts (curated + aggregator)
    #    and picks the interesting aggregator subset based on user preferences
    result = llm.generate_digest(all_posts)

    # 7. Store digest
    store_digest(db, result, posts=result.included_posts)
    return result
```

The app controls every step. Curated feed posts are all included and scraped.
Aggregator posts are all *fetched* but the LLM decides which ones are worth
including in the digest based on the user's known preferences — no hard caps.

---

## 6. Web GUI

### Stack: FastAPI + Jinja2 + htmx

- **FastAPI** — async, background tasks are first-class, clean route API.
- **Jinja2** — server-rendered templates, no JS build tooling.
- **htmx** — interactive partial updates (polling digest status, toggles)
  without a SPA.
- **No ORM** — raw `sqlite3` with a thin `db.py` helper. This is a personal
  tool, not a SaaS.

### Layout

```
web/
  app.py              # FastAPI app, routes
  db.py               # sqlite3 helpers, init_db()
  templates/
    base.html
    feeds.html        # manage curated feeds
    aggregators.html  # manage aggregator sources
    digest.html       # view a single digest
    digests.html      # digest history
    settings.html     # hours_back, LLM model, scraping toggle
  static/
    style.css
```

### Routes

```
GET  /                        → redirect to /digests
GET  /digests                 → list past digests
GET  /digests/{id}            → view digest (rendered markdown)
POST /digests/run             → trigger background digest, return 202
GET  /digests/{id}/status     → htmx polls this for progress

GET  /feeds                   → list curated feeds with enable toggle
POST /feeds                   → add feed
POST /feeds/{id}/toggle       → enable/disable
POST /feeds/{id}/delete       → remove feed

GET  /aggregators             → list aggregator sources with enable toggle
POST /aggregators             → add aggregator
POST /aggregators/{id}/toggle → enable/disable
POST /aggregators/{id}/delete → remove aggregator

GET  /settings                → app config form
POST /settings                → save config
```

### Digest trigger flow

1. User clicks "Run Digest" → `POST /digests/run`
2. Server creates `digests` row (status=running), starts `BackgroundTask`
3. UI shows spinner, htmx polls `/digests/{id}/status` every 2s
4. Background task runs the pipeline, updates row to status=done
5. Poll returns "Done — view digest" link

---

## 7. Aggregator ETL (HN, Lobsters, Reddit)

### Feed Sources

All aggregators use standard RSS, no custom APIs needed:

**Hacker News** (via hnrss.org, server-side quality filtering):
```
https://hnrss.org/frontpage?points=100&count=20
https://hnrss.org/ask?points=50&count=10
https://hnrss.org/show?points=50&count=10
```

**Lobste.rs** (tag-filtered, naturally high S/N):
```
https://lobste.rs/t/programming.rss
https://lobste.rs/t/practices.rss
https://lobste.rs/t/devops.rss
https://lobste.rs/t/security.rss
```

**Reddit** (top-of-day sort, limited):
```
https://www.reddit.com/r/ExperiencedDevs/top/.rss?t=day&limit=10
https://www.reddit.com/r/programming/top/.rss?t=day&limit=10
```

### Anti-Drowning Strategy

The key insight: **fetch everything, let the LLM curate.**

1. **Fetch curated feeds first**, build a `seen_urls` set. Curated posts always
   win over aggregator duplicates.

2. **Fetch all aggregator posts** for the time window. No hard caps — pull in
   everything that passes the source-level quality filters (HN points>=100,
   Reddit top/day, Lobsters tag filter).

3. **URL-level deduplication:** Normalize URLs (strip UTM params, trailing
   slashes, www prefix). A blog post that also appears on HN shows once as the
   blog version with full retrieved content.

4. **LLM picks the subset:** The digest generation prompt receives all curated
   posts (always included) plus the full list of aggregator posts. The LLM
   selects which aggregator posts are interesting based on user preferences,
   avoiding overlap with curated content.

5. **Only retrieve articles for curated feeds:** Aggregator posts link to
   external content. The LLM judges them by title + excerpt alone. If a post is
   selected, its URL is included for the user to read.

### Suggested Defaults

| Source | Quality filter | Articles retrieved? |
|---|---|---|
| Curated blogs | none (all included) | yes |
| HN frontpage | points>=100 | no |
| HN ask/show | points>=50 | no |
| Lobsters | tag filter | no |
| Reddit | top/day sort | no |

### URL Canonicalization

```python
def canonical_url(url: str) -> str:
    parsed = urllib.parse.urlparse(url)
    clean = parsed._replace(query="", fragment="").geturl()
    return clean.rstrip("/").removeprefix("https://www.").removeprefix("http://www.")
```

---

## Dependencies (inline PEP 723 style preserved)

New packages needed:
- `fastapi` + `uvicorn` — web server
- `jinja2` — templates
- `anthropic` — Claude API (optional, per provider choice)
- `openai` — OpenAI-compatible API (optional)
- `python-multipart` — form handling in FastAPI
- `mistune` or `markdown-it-py` — render digest markdown to HTML
