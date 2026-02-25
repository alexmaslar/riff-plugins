# Riff Community Plugin Registry

Community-maintained catalog of plugins for the [Riff](https://github.com/alexmaslar/riff) music server.

Riff servers fetch this catalog on startup and every 6 hours, making community plugins available alongside built-in ones.

## Submitting a Plugin

1. Build your plugin as a WASM binary with a `manifest.json` (see [Plugin Development Guide](PLUGINS.md))
2. Publish releases to your own GitHub repo
3. Fork this repo and add an entry to `catalog.json`
4. Open a PR

### catalog.json entry format

```json
{
  "name": "my-plugin",
  "display_name": "My Plugin",
  "description": "Short description of what the plugin does.",
  "author": "your-github-username",
  "capabilities": ["streaming"],
  "settings": [],
  "wasm_url": "https://github.com/you/riff-plugin-mine/releases/latest/download/plugin.wasm",
  "manifest_url": "https://github.com/you/riff-plugin-mine/releases/latest/download/manifest.json"
}
```

**Fields:**

| Field | Description |
|---|---|
| `name` | Unique plugin identifier (lowercase, alphanumeric + hyphens) |
| `display_name` | Human-readable name shown in the UI |
| `description` | One-line description |
| `author` | GitHub username or org |
| `capabilities` | Array of: `streaming`, `editorial`, `lyrics`, `scrobble`, `metadata` |
| `settings` | Array of setting field objects (passed through to the UI as-is) |
| `wasm_url` | Direct download URL for `plugin.wasm` |
| `manifest_url` | Direct download URL for `manifest.json` |

## Available Plugins

| Plugin | Author | Capabilities |
|---|---|---|
| [Qobuz](https://github.com/alexmaslar/riff-plugin-qobuz) | riff | Streaming |
| [Tidal](https://github.com/alexmaslar/riff-plugin-tidal) | riff | Streaming |
| [Pitchfork](https://github.com/alexmaslar/riff-plugin-pitchfork) | riff | Editorial |
