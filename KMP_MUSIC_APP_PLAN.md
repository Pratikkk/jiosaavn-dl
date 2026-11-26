# Kotlin Multiplatform Music Streaming App Plan

**Source API:** `jiosaavn-dl` Python CLI (`jiosaavn_core.py` — `Jiosaavn` class)
**Target:** Kotlin Multiplatform (KMP) app with Compose Multiplatform UI
**Branch:** `feature/kmp-music-app-plan` (separate from main Python repo)
**Status:** Planning — not yet implemented

---

## 1. Overview

Build a native music streaming application for Android and iOS that consumes the
same public JioSaavn API endpoints used by the `jiosaavn-dl` Python CLI. The app
will:

* Browse/search songs, albums, playlists, and artists
* Stream AAC audio (M4A container) directly — no transcoding, no FFmpeg needed
* Display metadata (title, album, artists, year, genre, lyrics, cover art)
* Support offline downloads with resume
* Share business logic via Kotlin Multiplatform (KMP) shared module
* Use Compose Multiplatform for UI on both Android and iOS

The Python CLI (`jiosaavn_core.py`) serves as the **reference implementation**
for API behavior, URL parsing, response handling, CDN URL generation, and
caching strategy. The KMP app reimplements this logic in Kotlin.

---

## 2. API Reference (from `jiosaavn_core.py`)

### Base endpoint

```
https://www.jiosaavn.com/api.php
```

### Common query parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `__call`  | API method name | e.g. `webapi.get`, `song.generateAuthToken`, `search.getResults`, `lyrics.getLyrics`, `artist.getAlbums` |
| `_format` | `json` | Response format |
| `_marker` | `0` | Marker |
| `ctx`     | `web6dot0` | Client context |

### API methods

| Method | Params | Used for |
|--------|--------|----------|
| `webapi.get` | `token`, `type`, `n`, `p` | Song/album/playlist metadata |
| `song.generateAuthToken` | `url`, `bitrate`, `api_version` | CDN audio URL |
| `search.getResults` | `q`, `p`, `n` | Search |
| `lyrics.getLyrics` | `lyrics_id`, `api_version` | Lyrics (synced/plain) |
| `artist.getAlbums` | `artist_id`, `p`, `n` | Artist albums |

### Response handling

* JSON responses may be wrapped in a callback (`callback({...})`) — extract the
  first complete JSON object.
* `data.get("error")` indicates an API error.
* Non-dict or empty responses are errors.

### CDN URL transformation

The `auth_url` from `song.generateAuthToken` may contain `web.` host prefix.
Replace `web.` with `aac.` to get the AAC stream URL:
```
web.saavncdn.com/... -> aac.saavncdn.com/...
```

---

## 3. KMP Project Structure

```
jiosaavn-kmp/
├── .github/
│   └── workflows/
│       ├── ci.yml                    # Android + iOS CI
│       └── smoke-test.yml            # Weekly API compatibility smoke test
├── build.gradle.kts                  # Root build config
├── settings.gradle.kts               # Module includes
├── gradle.properties                 # KMP / Compose versions
├── gradle/
│   └── libs.versions.toml            # Version catalog
├── shared/                           # KMP shared module (common + platform)
│   ├── src/
│   │   ├── commonMain/
│   │   │   ├── kotlin/
│   │   │   │   ├── api/              # JioSaavn API client
│   │   │   │   │   ├── JiosaavnApi.kt        # HTTP client + endpoints
│   │   │   │   │   ├── models.kt             # Data models (Song, Album, Playlist, Artist)
│   │   │   │   │   ├── url.kt                  # URL parsing/validation
│   │   │   │   │   ├── cache.kt                # Memory + disk cache
│   │   │   │   │   └── cdn.kt                  # CDN URL generation
│   │   │   │   ├── domain/               # Use cases / repository
│   │   │   │   │   ├── SongRepository.kt
│   │   │   │   │   ├── DownloadUseCase.kt
│   │   │   │   │   └── SearchUseCase.kt
│   │   │   │   └── ui/                   # Shared UI (Compose Multiplatform)
│   │   │   │       ├── theme.kt
│   │   │   │       ├── components/
│   │   │   │       ├── screens/
│   │   │   │       └── viewmodel/
│   │   │   ├── androidMain/
│   │   │   │   └── kotlin/               # Android-specific impls
│   │   │   └── iosMain/
│   │   │       └── kotlin/               # iOS-specific impls
│   ├── build.gradle.kts
│   └── shared.def
├── iosApp/                           # iOS native wrapper (SwiftUI)
│   ├── iosApp/
│   │   └── ...
│   └── iosApp.xcodeproj/
├── androidApp/                       # Android native wrapper
│   └── src/
│       └── main/
│           └── ...
├── KMP_MUSIC_APP_PLAN.md             # This file
└── README.md
```

---

## 4. Module Breakdown

### 4.1 `shared` — KMP shared module

Contains all business logic, API client, data models, and shared UI.

**Common source set (`commonMain`):**

* **API client** — Ktor HTTP client with Kotlinx.serialization JSON. Handles
  retries, rate limiting, caching, and response parsing.
* **Data models** — Kotlin data classes mirroring the JSON fields from the
  Python reference:
  * `Song` — id, song (title), album, year, primary_artists, encrypted_media_url,
    image, has_lyrics, explicit_content, language, featured_artists, singers,
    starring, label, release_date, copyright_text, genre, music, disc_number,
    total_discs, compilation, isrc, more_info
  * `Album` — title, songs, list_count, image, year, primary_artists
  * `Playlist` — listname or title, songs, list_count
  * `Artist` — name, albums
* **URL parsing** — `parseUrl(url)` returns `(kind, token)`. Validates scheme,
  hostname (allowlist), token characters (rejects `/ \ ? # \r\n \x00`, `.`, `..`),
  fully decodes percent-encoding iteratively to prevent double-encoding bypass.
* **Cache** — Memory cache (ConcurrentHashMap) + optional disk cache (SQLite
  via kotlinx-sqlite or similar). Metadata, CDN URLs, cover art, lyrics.
* **Repository** — `SongRepository` interface with platform-specific
  implementations for file storage, audio playback, and downloads.

**Platform source sets:**

* `androidMain` — Android file system, ExoPlayer for audio, Android-specific
  storage paths.
* `iosMain` — iOS file system, AVPlayer/AVFoundation for audio,
  iOS-specific storage paths.

### 4.2 `androidApp` — Android wrapper

* Minimal AndroidActivity hosting Compose Multiplatform UI
* ExoPlayer integration for audio streaming/download
* Android file storage (internal/external)
* Notification media session (lock screen controls)
* Android manifest permissions (internet, storage)

### 4.3 `iosApp` — iOS wrapper

* SwiftUI hosting Compose Multiplatform UI via `UIViewController`
* AVFoundation/AVPlayer for audio
* iOS file storage (Documents directory)
- Background audio mode
* Now Playing info center

---

## 5. Key Features (MVP)

### Phase 1 — Core API & Playback

1. **API client** — Ktor client with retry (3 retries, 429/5xx), rate limiting,
   User-Agent header matching the Python reference.
2. **URL parsing** — `parseUrl()` with the same validation as Python:
   - Scheme: http/https only
   - Hostname: `jiosaavn.com`, `www.jiosaavn.com`, `m.jiosaavn.com`,
     `saavn.com`, `www.saavn.com`, `m.saavn.com`, plus short-link hosts
   - Token: full percent-decode loop (max 10 iterations), reject forbidden chars
3. **Metadata fetch** — `webapi.get` for song/album/playlist/artist
4. **CDN URL** — `song.generateAuthToken` with `web.` → `aac.` host replacement
5. **Search** — `search.getResults`, filter to song/album/playlist kinds
6. **Audio streaming** — Play M4A/AAC stream from CDN URL (ExoPlayer / AVPlayer)
7. **Basic UI** — Home (search), song detail, album detail, playlist detail,
   now playing

### Phase 2 — Downloads & Offline

1. **Download manager** — Concurrent downloads (configurable workers, default 4,
   max 16), resume with HTTP Range requests
2. **File management** — Sanitized filenames, album/playlist folders,
   M4A container, `.part` temp files for resume
3. **Metadata tagging** — Embed ID3-like tags in M4A (title, album, artist,
   year, genre, cover art, lyrics)
4. **Offline library** — Scan downloaded files, show in "Downloads" tab
5. **Skip existing** — Don't re-download tracks already in library (by song ID)

### Phase 3 — Enhanced Features

1. **Lyrics** — Synced LRC and plain lyrics display
2. **Cover art** — Download and cache 500x500 artwork
3. **Playlists** — User-created local playlists from downloaded tracks
4. **Explicit content filter** — Toggle to skip explicit tracks
5. **Year filters** — After/before year filtering
6. **Cache** — Optional disk cache with TTL (default 24h)
7. **Rate limiting** — Token bucket for API requests

### Phase 4 — Polish

1. **Themes** — Light/dark theme
2. **Translations** — Multi-language support
3. **Widgets** — Home screen widgets (Android), iOS widgets
4. **Crossfade** — Gapless playback
5. **Equalizer** — Basic EQ
6. **Sleep timer**

---

## 6. Data Models (Kotlin)

```kotlin
data class Song(
    val id: String?,
    val song: String?,           // title
    val album: String?,
    val year: String?,
    val primaryArtists: String?,
    val encryptedMediaUrl: String?,
    val image: String?,
    val hasLyrics: Boolean,
    val explicitContent: Boolean,
    val language: String?,
    val featuredArtists: List<String> = emptyList(),
    val singers: List<String> = emptyList(),
    val starring: List<String> = emptyList(),
    val label: String?,
    val releaseDate: String?,
    val copyrightText: String?,
    val genre: List<String> = emptyList(),
    val music: List<String> = emptyList(),
    val discNumber: Int? = null,
    val totalDiscs: Int? = null,
    val compilation: Boolean = false,
    val isrc: String? = null,
    val moreInfo: Map<String, Any?> = emptyMap()
)

data class Album(
    val title: String?,
    val songs: List<Song>,
    val listCount: Int?,
    val image: String?,
    val year: String?,
    val primaryArtists: String?
)

data class Playlist(
    val title: String?,
    val songs: List<Song>,
    val listCount: Int?
)

data class Artist(
    val id: String?,
    val name: String?,
    val albums: List<Album> = emptyList()
)

data class SearchResult(
    val kind: String,        // "song", "album", "playlist"
    val url: String,
    val title: String,
    val subtitle: String?
)

data class CdnUrlResult(
    val url: String,
    val quality: String
)
```

---

## 7. API Client Interface (Kotlin)

```kotlin
interface JiosaavnApi {
    suspend fun metadata(kind: String, token: String, page: Int?): ApiResponse
    suspend fun getCdnUrl(encUrl: String, bitrate: String): CdnUrlResult
    suspend fun search(query: String, limit: Int): List<SearchResult>
    suspend fun getLyrics(lyricsId: String): LyricsResult
    suspend fun getArtistAlbums(artistId: String, page: Int, limit: Int): ArtistAlbumsResult
}
```

Implementation uses Ktor with:
* `HttpClient` + `Serialization` + `Json`
* Retry interceptor (3 retries, 429/500/502/503/504)
- User-Agent: `Mozilla/5.0 ... Chrome/122.0.0.0 Safari/537.36`
- Accept: `application/json, text/plain, */*`
- Accept-Language: `en-US,en;q=0.9`
- Referer: `https://www.jiosaavn.com/`

---

## 8. Security Considerations (from audit)

1. **URL validation** — Host allowlist, scheme restriction, no credentials/ports
2. **Token decoding** — Iterative percent-decode to prevent double-encoding bypass
   (e.g. `%252F` → `%2F` → `/` rejected)
3. **Path traversal** — Sanitize filenames, reject `..`, `.`, and forbidden chars
4. **No arbitrary code execution** — No eval/exec equivalent in Kotlin
5. **Caching** — Disk cache with TTL to prevent stale auth URLs (max 1 hour TTL)
6. **Rate limiting** — Token bucket to avoid overwhelming the API

---

## 9. Build Configuration

### `gradle/libs.versions.toml`

```toml
[versions]
kotlin = "2.0.0"
compose = "1.7.0"
ktor = "2.3.0"
kotlinx-serialization = "1.6.3"
kotlinx-coroutines = "1.8.0"
exoplayer = "1.1.3"
sqlDelight = "1.7.0"  # optional, for disk cache

[libraries]
kotlin-stdlib = { module = "org.jetbrains.kotlin:kotlin-stdlib", version.ref = "kotlin" }
ktor-client-core = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-okhttp = { module = "io.ktor:ktor-client-okhttp", version.ref = "ktor" }
ktor-client-cio = { module = "io.ktor:ktor-client-cio", version.ref = "ktor" }
ktor-serialization = { module = "io.ktor:ktor-client-serialization", version.ref = "ktor" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinx-serialization" }
kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "kotlinx-coroutines" }
compose-bom = { module = "org.jetbrains.compose:compose-bom", version.ref = "compose" }
compose-ui = { module = "org.jetbrains.compose.ui:ui" }
compose-material3 = { module = "org.jetbrains.compose.material3:material3" }
```

---

## 10. CI/CD

### GitHub Actions workflows

1. **`ci.yml`** — Run on push/PR:
   - Android: Build + unit tests on `ubuntu-latest`
   - iOS: Build + unit tests on `macos-latest` (requires Xcode)
   - KMP common tests on both

2. **`smoke-test.yml`** — Weekly API compatibility (Monday 02:00 UTC) + manual
   dispatch:
   - Run Kotlin tests that hit the live JioSaavn API
   - Non-blocking (`continue-on-error`)
   - Test song, album, playlist metadata + CDN URL generation

---

## 11. Timeline (Milestones)

| Milestone | Duration | Deliverables |
|-----------|----------|--------------|
| M1: Project setup | 1 week | KMP project structure, build config, API client interface |
| M2: Core API | 2 weeks | URL parsing, metadata, search, CDN URL, basic models |
| M3: Audio playback | 2 weeks | ExoPlayer/AVPlayer integration, streaming M4A |
| M4: Basic UI | 2 weeks | Compose Multiplatform screens (home, search, detail, now playing) |
| M5: Downloads | 3 weeks | Download manager, resume, file management, offline library |
| M6: Metadata & lyrics | 2 weeks | Tagging, cover art, lyrics display |
| M7: Polish | 2 weeks | Themes, explicit filter, year filter, caching |
| M8: Release | 1 week | Beta APK / TestFlight, CI/CD, smoke tests |

**Total: ~15 weeks**

---

## 12. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| API changes break compatibility | Weekly smoke test detects breakage; adapter pattern isolates API client |
| Audio URL expiration | Short TTL cache for CDN URLs (max 1 hour); regenerate on demand |
| Region restrictions | Graceful error handling; user notification |
| Rate limiting | Token bucket; exponential backoff; respect 429/Retry-After |
| iOS build complexity | Use Xcode Cloud or MacStadium CI runners |
| Compose Multiplatform iOS maturity | Start with simple UI; iterate as framework matures |

---

## 13. Next Steps

1. Create `jiosaavn-kmp/` directory in the repo (or separate repo)
2. Set up Gradle build with KMP + Compose Multiplatform
3. Implement API client (Ktor) mirroring Python reference
4. Implement data models
5. Implement URL parsing with same validation
6. Write unit tests for URL parsing and models
7. Set up CI/CD workflows
8. Implement audio streaming (Phase 1)
9. Build basic UI (Phase 1)
10. Iterate through remaining phases

---

## 14. Reference Files

* `/Users/pratik/gitclones/jiosaavn-dl/jiosaavn_core.py` — Python reference implementation
* `/Users/pratik/gitclones/jiosaavn-dl/README.md` — CLI documentation (defines expected behavior)
* `KMP_MUSIC_APP_PLAN.md` — This plan (living document)
