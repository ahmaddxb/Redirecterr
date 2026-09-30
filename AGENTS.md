# Redirecterr - Agent Guidelines & Architecture

## Overview
**Redirecterr** is a lightweight TypeScript service running on [Bun](https://bun.sh) that intercepts webhook events from [Overseerr](https://overseerr.dev) / [Jellyseerr](https://github.com/Fallenbagel/jellyseerr), inspects the requested media's metadata, evaluates configured filter rules, and routes the request to target Radarr or Sonarr instances with custom server IDs, root folders, quality profiles, and tags before auto-approving.

---

## Architecture & Codebase Map

```
Redirecterr/
├── .github/workflows/
│   ├── pr-tests.yaml             # Runs Bun test suite on push and PRs to main
│   ├── publish-image.yaml        # Builds multi-arch images (amd64/arm64) to ghcr.io on push to main & releases
│   └── publish-beta-image.yaml   # Publishes beta tag images on pre-release
├── src/
│   ├── main.ts                   # Bun HTTP server on port 8481; routes POST /webhook
│   ├── api/
│   │   └── overseerr.ts          # Overseerr REST client (fetch metadata, PUT /request/:id, POST approve)
│   ├── config/
│   │   └── index.ts              # YAML config loader and Ajv schema validation (config.yaml)
│   ├── services/
│   │   ├── filter.ts             # Filter matching engine (findMatchingFilter, matchValue, keywords, ratings)
│   │   ├── instance.ts           # Instance dispatcher (sendToInstances, merges tags, applies config, approves)
│   │   └── webhook.ts            # Webhook workflow orchestrator (test events, music auto-approve, routing)
│   ├── types/
│   │   ├── config.ts             # Config interfaces (Config, InstanceConfig, Filter, FilterCondition)
│   │   ├── webhook.ts            # Webhook and Overseerr media metadata interfaces
│   │   └── index.ts              # Type barrel exports
│   └── utils/
│       ├── helpers.ts            # Normalizers (normalizeToArray, normalizeTags, mergeTags, getPostData)
│       └── logger.ts             # Winston logger with console and daily-rotating file transports
├── Dockerfile                    # Multi-arch Alpine Bun container
├── fields.md                     # Reference of all incoming Overseerr/TMDB fields usable in filter conditions
├── filters.test.ts               # Bun test suite covering filtering rules, matching, and tag merging
└── README.md                     # End-user documentation and sample configs
```

---

## Core Workflows

1. **Webhook Ingestion (`src/main.ts`)**:
   - Overseerr sends `POST /webhook` notifications for `Request Pending Approval` (or `MEDIA_AUTO_APPROVED`).
   - Validates that the payload contains `media` and `request` objects.

2. **Metadata Hydration (`src/services/webhook.ts`)**:
   - Calls Overseerr API `GET /api/v1/{media_type}/{tmdbId}` to fetch rich media details (genres, keywords, production companies, content ratings, original language, etc.).

3. **Rule Evaluation (`src/services/filter.ts`)**:
   - Evaluates configured `filters` in top-to-bottom order until the first match is found.
   - Supports prioritized conditions: `keywords`, `contentRatings`, and `max_seasons`.
   - Supports arbitrary metadata fields from `fields.md` with condition operators:
     - `require`: Exact match against all specified values.
     - `include`: Substring match against any specified value.
     - `exclude`: Excludes if any value matches.

4. **Instance Application & Tagging (`src/services/instance.ts`)**:
   - Resolves target instance configuration (`server_id`, `root_folder`, optional `quality_profile_id`).
   - Merges default instance tags with matching filter tags (deduplicated array of numeric tag IDs).
   - Updates Overseerr via `PUT /api/v1/request/{requestId}` with `{ serverId, rootFolder, profileId, tags, seasons, mediaType }`.
   - Approves request via `POST /api/v1/request/{requestId}/approve` unless `approve: false`.

5. **Fallback**:
   - If no filters match and `approve_on_no_match: true` is configured, approves the request with default Overseerr settings.

---

## Configuration Reference (`config.yaml`)

```yaml
overseerr_url: "http://overseerr:5055"
overseerr_api_token: "your-api-token"
approve_on_no_match: true

instances:
  sonarr_anime:
    server_id: 2
    root_folder: "/mnt/media/Anime"
    quality_profile_id: 1 # Optional
    approve: true         # Optional (default: true)
    tags: [1, 5]          # Optional: Numeric tag IDs in Sonarr

filters:
  - media_type: tv
    is_4k: false          # Optional: true | false | omit for both
    conditions:
      keywords:
        include: ["anime", "animation"]
      contentRatings:
        exclude: [12, 16]
      max_seasons: 2
    apply: sonarr_anime
    tags: [10]            # Optional: Merged with instance tags -> sends [1, 5, 10]
```

---

## Development & Testing Commands

All commands use Bun:

- **Run in development mode**:
  ```bash
  bun --watch src/main.ts
  ```
- **Run tests**:
  ```bash
  bun test
  ```
- **Type checking**:
  ```bash
  bun x tsc --noEmit
  ```

---

## CI/CD & Image Publishing

- **Container Registry**: GitHub Container Registry (`ghcr.io/ahmaddxb/redirecterr`)
- **Published Tags**:
  - `ghcr.io/ahmaddxb/redirecterr:latest`
  - `ghcr.io/ahmaddxb/redirecterr:sha-<short_sha>` (e.g. `sha-cf70bc4`)
- **Build Pipeline**: Docker Buildx multi-arch (`linux/amd64`, `linux/arm64`) with GitHub Actions cache backend (`type=gha`).
