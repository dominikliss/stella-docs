# Marketing — YouTube integration (plan)

**Status:** **phase 1 + Analytics** — Data API sync as above; **YouTube Analytics API** via Google **OAuth** (`YouTubeOAuthService`, `YouTubeAnalyticsApiService`, `YouTubeAnalyticsSyncService`), tables `dls_youtube_analytics_sync_runs` + `dls_youtube_analytics_chunks`, REST under `dls/v1/youtube/oauth/*` and `POST /dls/v1/youtube/analytics/sync`. Verwaltung: OAuth client, “Mit Google verbinden”, Analytics sync (many report queries × date windows).  
**Goal:** PHP service + custom DB tables to sync and store **all public metadata/statistics** for **every video** on your channel (historical catalog), with a path toward **Analytics API** metrics later.

---

## 1. What “all data from every video” means in practice

Google splits concerns across **two** APIs:

| API | What you get | Auth |
|-----|----------------|------|
| **YouTube Data API v3** | Video metadata (`snippet`, `contentDetails`, `status`, `statistics`, `topicDetails`, …), channel info, playlist items (enumerate uploads). | **API key** works for **public** data if you know `channelId`. **OAuth** required for `mine=true`, private/unlisted details, and some operations. |
| **YouTube Analytics API** | Watch time, impressions, CTR, revenue, geography, retention — **dashboard-style** metrics. | **OAuth 2.0** only; must be **channel owner** (or delegated). Not the same quota bucket as Data API. |

**This plan (phase 1)** focuses on **Data API v3**: full **catalog** + rich **per-video** documents stored in your DB.  
**Phase 2** (later): connect **Analytics API** + separate tables or JSON blobs for time-series metrics.

---

## 2. How to enumerate *every* video (reliable pattern)

1. **`channels.list`**  
   - `part=contentDetails`  
   - Identify channel by `id=<CHANNEL_ID>` (API key OK for public channel) **or** `mine=true` (OAuth).

2. Read **`contentDetails.relatedPlaylists.uploads`** → **uploads playlist ID**.

3. **`playlistItems.list`**  
   - `playlistId=<uploads_playlist_id>`  
   - `maxResults=50` (max allowed)  
   - Paginate with **`nextPageToken`** until no more pages.  
   - Collect **video IDs** (`contentDetails.videoId` / `snippet.resourceId.videoId`).

4. **`videos.list`**  
   - `id=<comma-separated up to 50 IDs>`  
   - `part=snippet,contentDetails,statistics,status,topicDetails,recordingDetails,…` (as needed)  
   - Repeat in batches until all IDs are covered.

**Edge cases**

- **New uploads** after sync: incremental sync can re-walk the uploads playlist from a stored **page token** or **“last seen video publishedAt”** (compare with DB).
- **Deleted / private videos:** IDs may disappear from uploads or return partial errors; DB should mark **deleted_at** or **last_seen** instead of hard-delete if you want history.
- **Quota:** each `list` call costs units (typically **1 unit per call** for many read methods; see current [quota calculator](https://developers.google.com/youtube/v3/determine_quota_cost)). For large channels, a **full** sync is a **batch job** (cron / manual trigger), not a single HTTP request.

---

## 3. Data you can store (Data API — suggested columns)

Minimum useful set for analytics and marketing later:

| Field | Source (part) | Notes |
|-------|----------------|-------|
| `video_id` | stable PK | `id` |
| `title`, `description` | snippet | |
| `published_at`, `channel_id` | snippet | |
| `thumbnails` | snippet | store JSON or default URL only |
| `tags` | snippet | JSON array |
| `category_id`, `default_language` | snippet | |
| `duration` | contentDetails | ISO 8601 duration string |
| `definition`, `caption` | contentDetails | |
| `view_count`, `like_count`, `comment_count` | statistics | **snapshot at sync time** |
| `privacy_status`, `upload_status` | status | |

Optional: `topicDetails`, `recordingDetails`, `liveStreamingDetails` (if you use live).

**Important:** `statistics` are **point-in-time**. For “growth over time” you either:

- store **time-series** rows (`video_id`, `captured_at`, `view_count`, …), or  
- **only** use **Analytics API** for official time-series (phase 2).

---

## 4. Custom DB tables (WordPress)

Use **`$wpdb`** + **`dbDelta()`** on theme activation (or a one-time migration hook). Prefix: e.g. `{$wpdb->prefix}dls_youtube_*`.

### 4.1 Proposed tables

**`dls_youtube_channels`** (even if single channel today)

| Column | Purpose |
|--------|---------|
| `id` | PK |
| `channel_id` | YouTube channel ID (unique) |
| `title`, `custom_url`, `thumbnail_url` | snapshot |
| `uploads_playlist_id` | cache to avoid extra `channels.list` |
| `sync_cursor` | optional: last playlist page token or last `published_at` processed |
| `created_at`, `updated_at` | |

**`dls_youtube_videos**

| Column | Purpose |
|--------|---------|
| `id` | PK (auto increment) |
| `video_id` | unique, **YouTube video id** |
| `channel_id` | FK logical to channels table |
| `snippet_*` / `*_json` | either normalized columns or **one JSON column `raw_payload`** + a few indexed columns |
| `view_count`, `like_count`, `comment_count` | last snapshot |
| `published_at` | indexed for sorting |
| `synced_at` | last full row update |
| `deleted_at` | nullable — soft delete when API no longer returns video |

**`dls_youtube_sync_runs`** (audit / debugging)

| Column | Purpose |
|--------|---------|
| `id` | PK |
| `started_at`, `finished_at` | |
| `mode` | `full` / `incremental` |
| `videos_fetched`, `api_units_estimate` | |
| `status` | `running` / `success` / `error` |
| `error_message` | text |

**Optional:** `dls_youtube_video_stats_daily` for time-series if you want charts without Analytics API first (one row per video per day).

### 4.2 JSON vs normalized

- **Normalized:** easier SQL reporting, migrations heavier.  
- **`raw_payload` JSON** (full `videos.list` item): fastest to implement, resilient to API field additions; add generated columns or indexes on `video_id`, `published_at` only.

**Recommendation for v1:** **JSON column** for the full API response + **indexed** `video_id`, `channel_id`, `published_at`, `synced_at`. Add normalized columns later if queries need them.

---

## 5. PHP architecture (fits ddashboard)

| Piece | Responsibility |
|-------|----------------|
| **`DLS\Services\YouTubeDataService`** or **`DlData\YouTubeService`** | HTTP client (`wp_remote_get`), build URLs, handle API key / OAuth header, parse JSON, throw on 403/429. |
| **`DLS\Services\YouTubeSyncService`** | Orchestration: enumerate playlist → batch `videos.list` → upsert DB rows → write `sync_runs`. |
| **`inc/routes/youtube-*.php`** | `dls/v1` endpoints: `POST .../youtube/sync`, `GET .../youtube/status` (permission: logged-in). |
| **Credentials** | Store **API key** in `OptionService` / options table; **OAuth** refresh token encrypted when Analytics phase comes. |
| **Cron** | Optional `wp_schedule_event` for nightly incremental sync. |

**Security**

- Never expose API key to browser; only server-side PHP.  
- `dls/v1` routes already require login.

---

## 6. Quota & operations

- Default project quota is **10,000 units/day** (can request increase).  
- Full sync of a large channel may require **chunking across days** or **one-off** run with monitoring.  
- Implement **exponential backoff** on `429` and log `sync_runs`.

---

## 7. Implementation phases (suggested)

| Phase | Deliverable |
|-------|-------------|
| **A** | Options: API key + channel ID; `YouTubeDataService`: `channels.list`, `playlistItems.list` page, `videos.list` batch. |
| **B** | `dbDelta` tables; `YouTubeSyncService` full sync; REST `POST /dls/v1/youtube/sync-full` (manual). |
| **C** | Incremental sync + `sync_cursor`; optional WP-Cron. |
| **D** | Marketing UI: React page to trigger sync + list videos (read from custom tables via new REST endpoint or direct endpoint returning JSON). |
| **E** | OAuth + **YouTube Analytics API** + new tables for metrics / time-series. |

---

## 8. Open decisions (need your input before coding)

1. **Auth for Data API:** Is **API key + public channel ID** enough for your channel, or do you need **OAuth** from day one (e.g. unlisted videos)?  
2. **Single channel** only, or multi-channel later? (Affects `dls_youtube_channels` design.)  
3. **Storage shape:** prefer **full JSON blob per video** + indexes, or **fully normalized** columns from the start?  
4. **Historical stats:** Do you want **daily snapshots** in DB (table `video_stats_daily`) or only Analytics API later?  
5. **Sync trigger:** Manual button only first, or **cron** immediately?

---

## 9. Next step

After you confirm the decisions in §8, implementation can start with **Phase A + B** (service + tables + one manual sync endpoint) without touching the React Marketing UI beyond a placeholder button if desired.
