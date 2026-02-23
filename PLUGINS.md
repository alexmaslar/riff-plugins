# Riff Plugin Development Guide

Riff uses a WebAssembly (WASM) plugin system powered by [Extism](https://extism.org/) to extend the server with third-party streaming services, lyrics providers, scrobble integrations, and metadata enrichment. Plugins are standalone Rust crates compiled to `wasm32-unknown-unknown`, distributed as `.wasm` binaries alongside a `manifest.json`.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Manifest Reference](#manifest-reference)
- [Plugin Capabilities](#plugin-capabilities)
  - [Streaming](#streaming)
  - [Lyrics](#lyrics)
  - [Scrobble](#scrobble)
  - [Metadata](#metadata)
- [Exported Functions](#exported-functions)
- [Shared Types](#shared-types)
- [HTTP Requests](#http-requests)
- [Configuration](#configuration)
- [Plugin Variables (State)](#plugin-variables-state)
- [Host Functions](#host-functions)
- [Error Handling](#error-handling)
- [Building](#building)
- [Testing](#testing)
- [Publishing](#publishing)
- [Catalog Registration](#catalog-registration)
- [How the Server Loads Plugins](#how-the-server-loads-plugins)
- [Reference Implementations](#reference-implementations)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────┐
│  Riff Server (Axum)                                    │
│                                                        │
│  ┌──────────────┐   ┌──────────────┐                   │
│  │ /streaming/* │   │ /plugins/*   │   REST API        │
│  └──────┬───────┘   └──────┬───────┘                   │
│         │                  │                           │
│  ┌──────▼──────────────────▼───────┐                   │
│  │        PluginRegistry           │                   │
│  │  (streaming, lyrics, scrobble,  │                   │
│  │   metadata providers)           │                   │
│  └──────┬──────────────────────────┘                   │
│         │                                              │
│  ┌──────▼──────────────────────────┐                   │
│  │     WasmStreamingProvider       │                   │
│  │  (marshals JSON ↔ WASM calls)   │                   │
│  └──────┬──────────────────────────┘                   │
│         │                                              │
│  ┌──────▼──────────────────────────┐   ┌─────────────┐ │
│  │     WasmPluginInstance          │   │ Host Fns    │ │
│  │  (Extism runtime, permissions,  │◄──│ cache_get   │ │
│  │   config, 30s timeout)          │   │ cache_set   │ │
│  └──────┬──────────────────────────┘   └─────────────┘ │
│         │                                              │
└─────────┼──────────────────────────────────────────────┘
          │ WASM function calls (JSON in/out)
          │
┌─────────▼──────────────────────────────────────────────┐
│  Plugin WASM Binary (.wasm)                            │
│                                                        │
│  Exports: riff_search, riff_get_album,                 │
│           riff_get_artist_albums, riff_get_stream_url, │
│           riff_health_check                            │
│                                                        │
│  Uses: extism_pdk::http for outbound HTTP              │
│        extism_pdk::config for user settings            │
│        extism_pdk::var for persistent state            │
└────────────────────────────────────────────────────────┘
```

**Key points:**

- Plugins run inside an Extism WASM sandbox with a **30-second execution timeout**.
- Outbound HTTP is only allowed to hosts declared in `manifest.json`.
- All communication between host and plugin uses **JSON-serialized strings**.
- Plugin settings are passed as Extism config keys (string → string).
- Plugins can store state across calls using Extism variables (`var::get` / `var::set`).
- The host provides cache functions (`host_cache_get` / `host_cache_set`) with TTL support.

---

## Quick Start

1. **Create a new Rust project:**

```bash
cargo new --lib riff-plugin-myplugin
cd riff-plugin-myplugin
```

2. **Set up `Cargo.toml`:**

```toml
[package]
name = "riff-plugin-myplugin"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
extism-pdk = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"

[profile.release]
opt-level = "z"
lto = true
strip = true
```

3. **Create `manifest.json`:**

```json
{
    "id": "myplugin",
    "name": "My Plugin",
    "version": "0.1.0",
    "description": "What the plugin does",
    "author": "your-github-username",
    "capabilities": ["streaming"],
    "permissions": {
        "http": {
            "reason": "Why HTTP access is needed",
            "required_hosts": [
                "api.example.com",
                "*.example.com"
            ]
        }
    },
    "settings": []
}
```

4. **Implement the exported functions in `src/lib.rs`** (see [Exported Functions](#exported-functions)).

5. **Build:**

```bash
rustup target add wasm32-unknown-unknown  # first time only
cargo build --release --target wasm32-unknown-unknown
```

The output is at `target/wasm32-unknown-unknown/release/riff_plugin_myplugin.wasm`.

---

## Project Structure

Follow this layout for streaming plugins (the most common type):

```
riff-plugin-myplugin/
├── Cargo.toml          # cdylib crate with extism-pdk + serde
├── manifest.json       # Plugin metadata, permissions, settings
└── src/
    ├── lib.rs          # Exported PDK functions (#[plugin_fn])
    ├── types.rs        # Shared types (copy from reference — identical interface)
    ├── models.rs       # API-specific response models (serde structs)
    └── client.rs       # HTTP client using extism_pdk::http
```

**Convention:** Keep `types.rs` identical across all plugins. It defines the contract between your plugin and the server. Only `models.rs` and `client.rs` contain API-specific logic.

---

## Manifest Reference

The `manifest.json` file tells the server about your plugin's identity, capabilities, permissions, and configurable settings.

```json
{
    "id": "myplugin",
    "name": "My Plugin",
    "version": "0.1.0",
    "description": "Short description of what the plugin does",
    "author": "github-username",
    "capabilities": ["streaming"],
    "permissions": {
        "http": {
            "reason": "Human-readable explanation of why HTTP is needed",
            "required_hosts": [
                "api.example.com",
                "*.example.com"
            ]
        }
    },
    "settings": [
        {
            "key": "quality",
            "label": "Audio Quality",
            "field_type": "select",
            "required": false,
            "help_text": "Streaming quality preference"
        },
        {
            "key": "api_key",
            "label": "API Key",
            "field_type": "secret",
            "required": true,
            "help_text": "Your API key from example.com"
        },
        {
            "key": "region",
            "label": "Region",
            "field_type": "string",
            "required": false,
            "help_text": "Two-letter country code. Default: US"
        }
    ]
}
```

### Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | yes | Unique identifier (lowercase, alphanumeric, hyphens). Must match the catalog entry `name`. |
| `name` | string | yes | Human-readable display name. |
| `version` | string | yes | Semver version string. |
| `description` | string | yes | One-line description shown in the UI. |
| `author` | string | yes | GitHub username or organization. |
| `capabilities` | string[] | yes | One or more of: `"streaming"`, `"lyrics"`, `"scrobble"`, `"metadata"`. |
| `permissions` | object | yes | Permission declarations (currently only `http`). |
| `settings` | array | no | User-configurable settings shown in the iOS app. |

### HTTP Permissions

The `permissions.http.required_hosts` array declares which hostnames the plugin may contact. Extism uses **glob pattern matching** on the hostname:

- `"api.example.com"` — matches exactly `api.example.com`
- `"*.example.com"` — matches any subdomain of `example.com`

**Important:** If your API proxies or CDNs use different domains for content delivery (e.g., the API is at `api.example.com` but audio streams come from `cdn.example.com`), include both. The plugin cannot make HTTP requests to unlisted hosts — the Extism sandbox will block the request.

### Setting Field Types

| `field_type` | Description | UI Widget |
|---|---|---|
| `"string"` | Free-text input | Text field |
| `"secret"` | Sensitive value (API keys, tokens) | Masked text field with "Change" button |
| `"select"` | Dropdown from catalog-defined options | Picker |

**Note:** Setting field types in the manifest are simple strings. The **catalog entry** in `catalog.json` defines the actual select options with their values and labels. The manifest `field_type` just hints at the widget type for the iOS UI.

Settings are passed to the plugin as Extism config keys. Read them with:

```rust
let value = config::get("quality")?.unwrap_or_else(|| "default".to_string());
```

---

## Plugin Capabilities

Riff supports four capability types. Each requires a different set of exported functions.

### Streaming

The most common plugin type. Provides search, album/artist browsing, and audio stream URLs.

**Required exports:**

| Function | Input | Output | Description |
|---|---|---|---|
| `riff_search` | `SearchInput` | `StreamingSearchResults` | Search for albums, artists, and tracks |
| `riff_get_album` | `IdInput` | `StreamingAlbumDetail` | Get album with track listing |
| `riff_get_artist_albums` | `IdInput` | `Vec<StreamingAlbum>` | Get an artist's album discography |
| `riff_get_stream_url` | `StreamUrlInput` | `StreamUrl` | Resolve a playable audio URL for a track |
| `riff_health_check` | `String` (ignored) | `String` | Verify the service is reachable |

**Server-side flow:**

1. `GET /streaming/search?q=...&limit=...` → server calls `riff_search` on all streaming plugins concurrently, aggregates results by provider.
2. `GET /streaming/albums/{provider}/{id}` → server calls `riff_get_album` on the matching provider.
3. `GET /streaming/artists/{provider}/{id}/albums` → server calls `riff_get_artist_albums`.
4. `GET /streaming/tracks/{provider}/{id}/stream?quality=...` → server calls `riff_get_stream_url`, then **proxies** the audio from the CDN URL back to the client, forwarding `Range` headers for seeking support.

**Quality enum values sent by the server:** `hires`, `lossless`, `high`, `low` (defaults to `lossless` if omitted).

### Lyrics

Provides synced or plain-text lyrics for tracks.

**Required exports:**

| Function | Input | Output | Description |
|---|---|---|---|
| `riff_get_lyrics` | `LyricsInput` | `LyricsResult` | Find lyrics for a track |
| `riff_health_check` | `String` | `String` | Health check |

```rust
// LyricsInput
struct LyricsInput {
    title: String,
    artist: String,
    album: Option<String>,
    duration_secs: Option<u32>,
}

// LyricsResult
struct LyricsResult {
    plain_text: String,
    synced_lines: Option<Vec<SyncedLine>>,
    source: String,
    source_url: Option<String>,
}

struct SyncedLine {
    time_ms: u64,
    text: String,
}
```

### Scrobble

Reports listening activity to external services (e.g., Last.fm, ListenBrainz).

**Required exports:**

| Function | Input | Output | Description |
|---|---|---|---|
| `riff_now_playing` | `NowPlayingInput` | `String` | Report currently playing track |
| `riff_scrobble` | `ScrobbleInput` | `String` | Submit a completed listen |
| `riff_health_check` | `String` | `String` | Health check |

```rust
struct NowPlayingInput {
    user_token: String,
    title: String,
    artist: String,
    album: String,
    duration_secs: u32,
}

struct ScrobbleInput {
    user_token: String,
    title: String,
    artist: String,
    album: String,
    duration_secs: u32,
    played_at: i64,  // Unix timestamp
}
```

### Metadata

Enriches album and artist data with genres, styles, labels, images, and bios.

**Required exports:**

| Function | Input | Output | Description |
|---|---|---|---|
| `riff_enrich_album` | `EnrichAlbumInput` | `MetadataEnrichment` | Enrich an album's metadata |
| `riff_enrich_artist` | `EnrichArtistInput` | `MetadataEnrichment` | Enrich an artist's metadata |
| `riff_health_check` | `String` | `String` | Health check |

```rust
struct EnrichAlbumInput {
    title: String,
    artist: String,
    year: Option<i32>,
}

struct EnrichArtistInput {
    name: String,
}

struct MetadataEnrichment {
    genre: Option<Vec<String>>,
    style: Option<Vec<String>>,
    label: Option<String>,
    year: Option<i32>,
    image_url: Option<String>,
    bio: Option<String>,
    extra: serde_json::Value,
}
```

---

## Exported Functions

All exported functions use the Extism PDK `#[plugin_fn]` macro. Input and output are JSON-serialized.

### Function Signature Pattern

```rust
use extism_pdk::*;
use serde::Deserialize;

#[derive(Deserialize)]
struct SearchInput {
    query: String,
    limit: u32,
}

#[plugin_fn]
pub fn riff_search(Json(input): Json<SearchInput>) -> FnResult<Json<StreamingSearchResults>> {
    // ... your implementation
    Ok(Json(results))
}
```

### Input Types

These structs are deserialized from JSON sent by the host:

```rust
// For riff_search
#[derive(Deserialize)]
struct SearchInput {
    query: String,
    limit: u32,
}

// For riff_get_album, riff_get_artist_albums
#[derive(Deserialize)]
struct IdInput {
    id: String,
}

// For riff_get_stream_url
#[derive(Deserialize)]
struct StreamUrlInput {
    id: String,
    quality: StreamingQuality,
}
```

### Health Check

Every plugin **must** export `riff_health_check`. It receives an empty string and should return `"healthy"` on success or return an error. The server calls this on plugin load and via `GET /plugins/status`.

```rust
#[plugin_fn]
pub fn riff_health_check(_input: String) -> FnResult<String> {
    let client = create_client()?;
    client.search("test", 1)?;  // lightweight connectivity test
    Ok("healthy".to_string())
}
```

---

## Shared Types

The `types.rs` file defines the data contract between your plugin and the server. **Copy this file exactly** from the reference implementation — do not modify the field names or types.

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum StreamingQuality {
    Low,
    High,
    Lossless,
    HiRes,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StreamingArtist {
    pub provider_id: String,
    pub name: String,
    pub image_url: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StreamingAlbum {
    pub provider_id: String,
    pub title: String,
    pub artist: StreamingArtist,
    pub year: Option<i32>,
    pub cover_url: Option<String>,
    pub track_count: u32,
    pub available_qualities: Vec<StreamingQuality>,
    pub album_type: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StreamingTrack {
    pub provider_id: String,
    pub title: String,
    pub artist_name: String,
    pub album_title: String,
    pub track_number: u32,
    pub disc_number: u32,
    pub duration_secs: u32,
    pub available_qualities: Vec<StreamingQuality>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StreamingAlbumDetail {
    pub album: StreamingAlbum,
    pub tracks: Vec<StreamingTrack>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StreamingSearchResults {
    pub albums: Vec<StreamingAlbum>,
    pub artists: Vec<StreamingArtist>,
    pub tracks: Vec<StreamingTrack>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StreamUrl {
    pub url: String,
    pub mime_type: String,
    pub quality: StreamingQuality,
}
```

### Field Notes

| Field | Notes |
|---|---|
| `provider_id` | The upstream service's ID for this entity (string). The server uses this to route subsequent requests back to your plugin. |
| `artist_name` | On `StreamingTrack`, this is a flat string (not a nested artist object). Use the performer/artist name. |
| `album_title` | On `StreamingTrack`, the album title as a string. May be empty if the track is a single. |
| `disc_number` | Defaults to `1` if the source API doesn't provide disc information. |
| `available_qualities` | List of quality tiers this content supports. Shown in the UI. |
| `album_type` | Optional string like `"ALBUM"`, `"SINGLE"`, `"EP"`, `"COMPILATION"`. Set to `None` if unknown. |
| `cover_url` | Direct URL to album art or artist image. Prefer 640px or larger for album art, 480px for artist images. |
| `mime_type` | On `StreamUrl`, the MIME type of the audio stream (e.g., `"audio/flac"`, `"audio/mpeg"`, `"audio/mp4"`). |

---

## HTTP Requests

Plugins make HTTP requests using `extism_pdk::http`. This is **synchronous** (no async in WASM) and subject to the host's allowed-hosts policy.

### Basic GET Request

```rust
use extism_pdk::*;

fn get_json<T: serde::de::DeserializeOwned>(url: &str) -> Result<T, Error> {
    let req = HttpRequest::new(url).with_method("GET");
    let resp = http::request::<()>(&req, None)?;

    let status = resp.status_code();
    if status < 200 || status >= 300 {
        return Err(Error::msg(format!("HTTP {status} for {url}")));
    }

    serde_json::from_slice(&resp.body())
        .map_err(|e| Error::msg(format!("JSON parse error: {e}")))
}
```

### Adding Headers

```rust
let req = HttpRequest::new(url)
    .with_method("GET")
    .with_header("Authorization", "Bearer my-token")
    .with_header("Accept", "application/json");
let resp = http::request::<()>(&req, None)?;
```

### POST with Body

```rust
let body = serde_json::to_vec(&payload)?;
let req = HttpRequest::new(url)
    .with_method("POST")
    .with_header("Content-Type", "application/json");
let resp = http::request(&req, Some(body))?;
```

### URL Encoding

Since plugins can't use the `urlencoding` crate in WASM easily, include a minimal encoder:

```rust
mod urlencoding {
    pub fn encode(input: &str) -> String {
        let mut result = String::with_capacity(input.len() * 3);
        for byte in input.bytes() {
            match byte {
                b'A'..=b'Z' | b'a'..=b'z' | b'0'..=b'9'
                | b'-' | b'_' | b'.' | b'~' => {
                    result.push(byte as char);
                }
                _ => {
                    result.push('%');
                    result.push_str(&format!("{:02X}", byte));
                }
            }
        }
        result
    }
}
```

---

## Configuration

User settings defined in `manifest.json` are passed to the plugin as Extism config keys. Read them with:

```rust
use extism_pdk::*;

// Returns Ok(Some("value")) if set, Ok(None) if not set
let quality = config::get("quality")?.unwrap_or_else(|| "LOSSLESS".to_string());
let api_key = config::get("api_key")?
    .ok_or_else(|| Error::msg("api_key is required"))?;
```

**Important:** Config values are always strings. If you need numbers, parse them:

```rust
let timeout: u64 = config::get("timeout")?
    .and_then(|s| s.parse().ok())
    .unwrap_or(5);
```

---

## Plugin Variables (State)

Extism provides key-value variable storage that persists across function calls within the same plugin instance (but not across server restarts). Use this for caching instance lists, session tokens, or other state.

```rust
use extism_pdk::*;

// Store a value
var::set("my_key", "my_value")?;

// Retrieve a value
let value: Option<String> = var::get("my_key")?;

// Store complex data as JSON
let instances = vec!["https://api1.example.com", "https://api2.example.com"];
let json = serde_json::to_string(&instances)?;
var::set("instances", &json)?;

// Retrieve and parse
let json = var::get::<String>("instances")?.unwrap_or_default();
let instances: Vec<String> = serde_json::from_str(&json)?;
```

**Use cases:**
- Round-robin instance index (see Tidal plugin for example)
- Cached API instance lists
- Session tokens or auth state

---

## Host Functions

The server provides two host functions for shared TTL-based caching:

### `host_cache_get(key: String) -> Vec<u8>`

Returns cached bytes for the given key, or an empty `Vec` if expired or missing.

### `host_cache_set(key: String, value: Vec<u8>, ttl_secs: u64)`

Stores bytes with a time-to-live in seconds.

These are available as Extism host imports but are **not** exposed through the standard PDK. To use them, you would call them via raw Extism host function bindings. For most plugins, `var::get`/`var::set` is sufficient.

---

## Error Handling

Use `extism_pdk::Error` for all error cases. The host wraps errors and surfaces them in API responses and health checks.

```rust
use extism_pdk::*;

// Create an error
Err(Error::msg("something went wrong"))

// Convert from other error types
let value: u64 = input.id.parse()
    .map_err(|e| Error::msg(format!("invalid id: {e}")))?;

// Propagate with ?
let body = http::request::<()>(&req, None)?;
let data: MyType = serde_json::from_slice(&body.body())
    .map_err(|e| Error::msg(format!("parse error: {e}")))?;
```

**Guidelines:**
- Return descriptive error messages — they appear in server logs and health check responses.
- For `riff_search`, prefer returning empty results over errors for "no results found."
- For `riff_get_stream_url`, always return an error if no URL is available — the server needs to know.
- For `riff_health_check`, a returned error means the plugin is unhealthy. The error message is shown to the user.

---

## Building

### Prerequisites

```bash
rustup target add wasm32-unknown-unknown
```

### Debug Build

```bash
cargo build --target wasm32-unknown-unknown
```

### Release Build (for distribution)

```bash
cargo build --release --target wasm32-unknown-unknown
```

The release profile should be optimized for size:

```toml
[profile.release]
opt-level = "z"   # optimize for size
lto = true        # link-time optimization
strip = true      # strip debug symbols
```

Typical output sizes: 200-300 KB.

### Output Location

```
target/wasm32-unknown-unknown/release/<crate_name>.wasm
```

Note: Cargo uses underscores in the output filename (e.g., `riff_plugin_tidal.wasm`). Rename to `plugin.wasm` when publishing.

---

## Testing

### Manual Testing

1. Build the WASM plugin.
2. Copy `plugin.wasm` and `manifest.json` to the server's plugin directory:
   ```bash
   mkdir -p ~/Library/Application\ Support/riff/plugins/myplugin/
   cp target/wasm32-unknown-unknown/release/riff_plugin_myplugin.wasm \
      ~/Library/Application\ Support/riff/plugins/myplugin/plugin.wasm
   cp manifest.json ~/Library/Application\ Support/riff/plugins/myplugin/
   ```
3. Enable the plugin in `~/Library/Application Support/riff/config.yaml`:
   ```yaml
   plugins:
     myplugin:
       enabled: true
       settings:
         api_key: "your-key"
   ```
4. Restart the server and check logs for `hot-loaded wasm plugin: myplugin`.
5. Test endpoints:
   ```bash
   # Health check
   curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/plugins/status

   # Search
   curl -H "Authorization: Bearer $TOKEN" \
     "http://localhost:8080/streaming/search?q=radiohead&limit=5"

   # Album detail
   curl -H "Authorization: Bearer $TOKEN" \
     "http://localhost:8080/streaming/albums/myplugin/ALBUM_ID"

   # Stream URL (proxied audio)
   curl -H "Authorization: Bearer $TOKEN" \
     "http://localhost:8080/streaming/tracks/myplugin/TRACK_ID/stream?quality=lossless"
   ```

### Testing Without the Server

You can use the [Extism CLI](https://extism.org/docs/install/) to call plugin functions directly:

```bash
extism call plugin.wasm riff_health_check \
  --input "" \
  --allow-host "*.example.com" \
  --set-config '{"api_key": "test"}'

extism call plugin.wasm riff_search \
  --input '{"query":"test","limit":5}' \
  --allow-host "*.example.com" \
  --set-config '{"quality": "LOSSLESS"}'
```

---

## Publishing

### GitHub Release

1. Create a GitHub repository for your plugin (e.g., `username/riff-plugin-myplugin`).
2. Build the release WASM binary.
3. Create a GitHub release with both files:
   ```bash
   cp target/wasm32-unknown-unknown/release/riff_plugin_myplugin.wasm plugin.wasm
   gh release create v0.1.0 plugin.wasm manifest.json \
     --title "v0.1.0" \
     --notes "Initial release"
   ```

**Important:** Do not mark the release as a "pre-release" — the server uses `/releases/latest/download/` URLs which skip pre-releases.

### Download URLs

The expected URL pattern is:

```
https://github.com/USERNAME/riff-plugin-myplugin/releases/latest/download/plugin.wasm
https://github.com/USERNAME/riff-plugin-myplugin/releases/latest/download/manifest.json
```

Using `/releases/latest/download/` ensures users always get the most recent version.

---

## Catalog Registration

To make your plugin discoverable by all Riff servers:

1. Fork [alexmaslar/riff-plugins](https://github.com/alexmaslar/riff-plugins).
2. Add an entry to `catalog.json`:

```json
{
    "name": "myplugin",
    "display_name": "My Plugin",
    "description": "Short description.",
    "author": "your-github-username",
    "capabilities": ["streaming"],
    "settings": [
        {
            "key": "quality",
            "label": "Audio Quality",
            "field_type": {
                "type": "select",
                "options": [
                    { "value": "lossless", "label": "Lossless (FLAC)" },
                    { "value": "high", "label": "High (320kbps)" }
                ]
            },
            "required": false,
            "help_text": "Default: Lossless"
        },
        {
            "key": "api_key",
            "label": "API Key",
            "field_type": { "type": "secret" },
            "required": true,
            "help_text": "Get your key from example.com/developers"
        }
    ],
    "wasm_url": "https://github.com/you/riff-plugin-myplugin/releases/latest/download/plugin.wasm",
    "manifest_url": "https://github.com/you/riff-plugin-myplugin/releases/latest/download/manifest.json"
}
```

3. Open a pull request.

### Catalog vs. Manifest Settings

The **catalog** `settings` array defines the UI presentation (select options with labels, field type rendering). The **manifest** `settings` array declares the same keys but with simpler field types. Both must list the same setting keys.

**Catalog `field_type`** is an object with `type` and optional `options`:
```json
{ "type": "select", "options": [{ "value": "lossless", "label": "Lossless" }] }
{ "type": "secret" }
{ "type": "string" }
```

**Manifest `field_type`** is a simple string: `"select"`, `"secret"`, or `"string"`.

---

## How the Server Loads Plugins

Understanding the server-side lifecycle helps debug issues:

1. **Catalog fetch** — On startup (and every 6 hours), the server fetches `catalog.json` from the community registry.

2. **Config check** — For each catalog entry, the server checks `config.yaml` to see if the plugin is enabled.

3. **Download** — If enabled but not installed, the server downloads `plugin.wasm` and `manifest.json` from the catalog URLs to `<plugin_dir>/<plugin_id>/`.

4. **Load** — The server reads the WASM bytes and manifest, creates an Extism plugin instance with:
   - Allowed HTTP hosts from `manifest.json`
   - Config keys from user settings in `config.yaml`
   - WASI enabled
   - 30-second execution timeout
   - Host cache functions

5. **Health check** — Immediately after loading, the server calls `riff_health_check`. If it fails, the plugin is still registered but marked unhealthy. The health status is returned in the config update response.

6. **Registration** — The plugin instance is registered in the `PluginRegistry` under its declared capabilities. For streaming plugins, a `WasmStreamingProvider` wrapper is also registered.

7. **Hot reload** — When plugin settings change via the config API, the server unregisters and re-loads all WASM plugins (clean slate). This happens synchronously so the API response includes health check results.

### Plugin Directory

Plugins are stored at:
- **macOS:** `~/Library/Application Support/riff/plugins/<plugin_id>/`
- Each plugin directory contains `plugin.wasm` and `manifest.json`.

---

## Reference Implementations

### Tidal Plugin (Simple with Failover)

[`riff-plugin-tidal`](https://github.com/alexmaslar/riff-plugin-tidal) — Demonstrates:
- Round-robin failover across multiple proxy instances
- Instance discovery from a remote JSON endpoint
- Instance list stored in Extism variables (`var::set`/`var::get`)
- Base64 manifest decoding for stream URLs
- DASH XML fallback (hi-res → lossless quality downgrade)

### Qobuz Plugin (Minimal)

[`riff-plugin-qobuz`](https://github.com/alexmaslar/riff-plugin-qobuz) — Demonstrates:
- Single-proxy architecture (no failover)
- Custom serde deserializers for API quirks (`StringOrU64`, object-or-string names)
- Direct stream URL (no manifest decoding)
- Per-request header injection (`Token-Country`)

---

## Troubleshooting

### "HTTP request to X is not allowed"

Your `manifest.json` `required_hosts` is missing the hostname. Add it and rebuild. Remember:
- Include both the API domain and any CDN domains used for audio streams.
- Use `*.example.com` wildcards for subdomains.
- The check uses glob matching against the URL's hostname only.

### Plugin loads but health check fails

Common causes:
- The upstream API/proxy is down (check by curling the URL directly).
- Anti-bot protection (JavaScript challenge page instead of JSON).
- Missing or incorrect config values (check `config.yaml`).
- Network issues from the server host.

The error message from `riff_health_check` is shown in `GET /plugins/status` and in the config update response under `plugin_health`.

### WASM build fails

- Ensure the target is installed: `rustup target add wasm32-unknown-unknown`
- Only use dependencies that compile to WASM. Crates that depend on `tokio`, `reqwest`, `std::net`, or OS-level I/O won't work. Use `extism_pdk::http` for HTTP.
- The `serde`, `serde_json`, and `extism-pdk` crates all work in WASM.
- If you need base64 encoding, the `base64` crate works in WASM.

### Plugin not appearing in search results

- Verify the plugin is loaded: `GET /plugins/status`
- Verify it has the `streaming` capability in the manifest.
- Check server logs for errors during the `riff_search` call.
- Try searching with a known good query that returns results from the upstream API.

### Stream URL returns 403/404

- The CDN domain may not be in `required_hosts`. Check the URL domain and add it to the manifest.
- The upstream service may require authentication headers that aren't being sent.
- Stream URLs may expire — they should be resolved just before playback, not cached.

### "failed to download wasm plugin"

- Ensure the GitHub release is **not** a pre-release.
- Verify the download URLs return 200: `curl -sI -L <wasm_url>`
- GitHub release downloads redirect (302 → 302 → 200). The server's HTTP client follows redirects automatically.
