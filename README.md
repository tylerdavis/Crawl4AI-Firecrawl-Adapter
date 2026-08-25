# Crawl4AI-Firecrawl-Adapter

A lightweight Firecrawl-compatible scrape adapter for Crawl4AI.

Built for LibreChat, but it can also work with other apps that only need Firecrawl-style scrape endpoints.

## What It Does

Some apps expect a Firecrawl-style API for scraping web pages.
Crawl4AI uses a different API.
This adapter sits in the middle and translates the requests.

Supported endpoints:

- `POST /v1/scrape`
- `POST /scrape`
- `POST /v0/scrape`
- `GET /health`

## Requirements

- Docker Compose
- Crawl4AI
- An app that can talk to a Firecrawl-compatible scrape API

## Install

### 1. Clone the repo

Clone this repo next to the folder that contains your existing `docker-compose.yml`:

```bash
git clone https://github.com/ZDOSt/Crawl4AI-Firecrawl-Adapter.git
```

To update later:

```bash
cd Crawl4AI-Firecrawl-Adapter
git pull
```

This repo contains the actual adapter in the `firecrawl-crawl4ai-adapter` folder.

### 2. Add the adapter to Docker Compose

Example:

```yaml
services:
  crawl4ai:
    image: unclecode/crawl4ai:latest
    container_name: crawl4ai
    restart: unless-stopped
    environment:
      CRAWL4AI_API_TOKEN: ${CRAWL4AI_API_TOKEN:-}
    volumes:
      - ./config.yml:/app/config.yml:ro
    shm_size: 2gb

  firecrawl-crawl4ai-adapter:
    build:
      context: ./Crawl4AI-Firecrawl-Adapter/firecrawl-crawl4ai-adapter
    container_name: firecrawl-crawl4ai-adapter
    environment:
      PORT: 3002
      CRAWL4AI_BASE_URL: http://crawl4ai:11235
      CRAWL4AI_API_TOKEN: ${CRAWL4AI_API_TOKEN:-}
      FIRECRAWL_API_KEY: change-me
    depends_on:
      - crawl4ai
    restart: unless-stopped
```

### Authenticating to Crawl4AI

Recent `unclecode/crawl4ai` images bind to loopback only and refuse to expose
the API on `0.0.0.0` unless a credential is present. Setting `CRAWL4AI_API_TOKEN`
on the `crawl4ai` service makes it bind on all interfaces and enables its auth
gate, which then requires every request to carry `Authorization: Bearer <token>`.

Set the same `CRAWL4AI_API_TOKEN` on the adapter service and it forwards that
bearer token on all upstream Crawl4AI calls. Leave it unset to keep the previous
unauthenticated, loopback-only behavior.

### Browser pool warming (scrape latency)

Crawl4AI pre-loads a single "permanent" browser at startup, keyed to its default
browser config. Requests that match that config reuse the warm browser; requests
carrying a custom browser config spawn a fresh browser (a ~2s cold start) that is
reaped after `crawler.pool.idle_ttl_sec` of inactivity.

For a standard scrape (no `mobile`, custom `headers`, or `ignoreHttpsErrors`) the
adapter now sends **no** `browser_config`, so the request rides Crawl4AI's warm
default browser instead of minting a cold one. A custom browser config is only
attached when one of those options is actually requested.

The optional `config.yml` in this repo is mounted over Crawl4AI's baked config to
keep warmed browsers alive longer (`idle_ttl_sec: 1800`) and to make Crawl4AI
refuse new browsers earlier under memory pressure (`memory_threshold_percent:
85`). It is a full copy of the image's hardened default config with only those two
values changed — if you upgrade Crawl4AI, re-sync it so you don't pin an outdated
config. Drop the `volumes:` mount to fall back to the image defaults.

If your stack uses custom Docker networks, make sure your app, `crawl4ai`, and `firecrawl-crawl4ai-adapter` are on the same network.

If you copy only the inner `firecrawl-crawl4ai-adapter` folder into your stack instead of cloning the whole repo, change the build path to:

```yaml
build:
  context: ./firecrawl-crawl4ai-adapter
```

### 3. Point your app at the adapter

Set your Firecrawl URL to the adapter.

For LibreChat:

```env
FIRECRAWL_API_URL=http://firecrawl-crawl4ai-adapter:3002
FIRECRAWL_API_KEY=change-me
FIRECRAWL_VERSION=v1
```

In `librechat.yaml`:

```yaml
webSearch:
  scraperProvider: "firecrawl"
  firecrawlApiKey: "${FIRECRAWL_API_KEY}"
  firecrawlApiUrl: "${FIRECRAWL_API_URL}"
```

### 4. Build and start

```bash
docker compose build firecrawl-crawl4ai-adapter
docker compose up -d firecrawl-crawl4ai-adapter
```

If your app also needs a restart to pick up the new settings, restart it too.

For LibreChat:

```bash
docker compose up -d librechat
```

## What You Can Change

- `FIRECRAWL_API_KEY`: can be any shared value, as long as your app and the adapter use the same one.
- `CRAWL4AI_BASE_URL`: change this if your Crawl4AI service uses a different service name or port.
- `CRAWL4AI_API_TOKEN`: set this (matching the token on the `crawl4ai` service) when Crawl4AI has authentication enabled; the adapter sends it as a bearer token upstream.
- `PORT`: change this if you want the adapter to listen on a different internal port.
- `context`: change this if the adapter folder is in a different location.
- Docker networks: all relevant services must be able to reach each other by service name.

## Notes

- This is a lightweight Firecrawl-compatible scrape adapter, not a full Firecrawl replacement.
- It is mainly meant for LibreChat web search scraping.
- It can also work with other apps that only need Firecrawl-style scrape endpoints.
- For LibreChat, this replaces `crawl4ai-proxy` for the Firecrawl scraper use case.
- Crawl4AI still does the actual scraping.
