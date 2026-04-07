# YouTube-Agent.bundle: Plex TV Series & Movie metadata agent

A metadata agent for Plex that fetches titles, summaries, thumbnails, and other
metadata for downloaded YouTube videos by looking up the YouTube video ID embedded
in the filename.

> **This is a fork of [ZeroQI/YouTube-Agent.bundle](https://github.com/ZeroQI/YouTube-Agent.bundle)**
> with fixes for newer versions of Plex Media Server. See [Changes from upstream](#changes-from-upstream).

---

## Requirements

- [Absolute Series Scanner](https://github.com/ZeroQI/Absolute-Series-Scanner) — required to scan TV Series libraries
- A [YouTube Data API key](#youtube-api-key) — strongly recommended (the built-in key is shared and will hit quota limits)
- A [Plex token](#plex-token) — required for episode titles to work correctly on newer Plex versions

---

## Installation

1. Download this repo as a zip and unpack it, or clone it
2. Rename the folder to `YouTube-Agent.bundle` (no `-master` suffix)
3. Place it in your Plex `Plug-ins` folder:

| Platform | Path |
|---|---|
| Windows Vista/7/8 | `%LOCALAPPDATA%\Plex Media Server\Plug-ins\` |
| Windows XP | `%USERPROFILE%\Local Settings\Application Data\Plex Media Server\Plug-ins\` |
| macOS | `$HOME/Library/Application Support/Plex Media Server/Plug-ins/` |
| Linux (Debian/Ubuntu/Fedora) | `/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Plug-ins/` |
| Synology/Asustor | `/volume1/Plex/Library/Application Support/Plex Media Server/Plug-ins/` |
| QNAP | `/share/MD0_DATA/.qpkg/PlexMediaServer/Library/Plex Media Server/Plug-ins/` |

4. Restart Plex Media Server
5. Set up your [YouTube API key](#youtube-api-key)
6. Set up your [Plex token](#plex-token)

### Enable for a library

1. Open the library in Plex → **Manage Library** → **Edit** → **Advanced** tab
2. Set Agent to **YouTubeSeries** (for TV libraries) or **YoutubeMovie** (for movie libraries)

---

## Naming your files

The agent finds metadata by extracting the YouTube video ID from the filename or folder name. The ID must appear in square brackets.

**Supported formats:**
- `[xxxxxxxxxxx]` — bare video ID
- `[youtube-xxxxxxxxxxx]`

**Example filename:**
```
Person Of Interest Soundtrack [OR5EnqdnwK0].mp4
```

### TV Series library layout

Series folders must include a YouTube **channel ID** (`UC...`) or **playlist ID** (`PL...`) in the folder name, or in a `youtube.id` file inside the folder.

**Finding your channel ID:** YouTube channel URLs like `youtube.com/@channelname` are human-readable handles, not IDs. To find the actual ID:
1. Open the channel page in a browser
2. View page source (Ctrl+U) and search for `"channelId"` — you'll find something like `"channelId":"UCdQWs2nw6w77Rw0t-37a4OA"`

**Folder structure:**
```
Channel Name [UCdQWs2nw6w77Rw0t-37a4OA]/
  Season 2024/
    Video Title [videoId11char].mkv
    Video Title [videoId11char].mkv
  Season 2025/
    Video Title [videoId11char].mkv
```

Optional grouping subfolders (e.g. by series/playlist) are supported — they will appear as collections in Plex:
```
CaRtOoNz [UCdQWs2nw6w77Rw0t-37a4OA]/
  Ben and Ed/
    Ben and Ed - Episode Title [fRFr7L_qgEo].mkv
  Golf With Your Friends/
    Golf With Friends - Episode Title [81er8CP24h8].mkv
```

### Movie library layout

```
Video Title [videoId11char].mp4
```
or with youtube prefix:
```
Video Title [youtube-videoId11char].mp4
```

---

## YouTube API key

The agent ships with a built-in API key, but it is shared across all users of the default configuration — if quota runs out, metadata stops working for everyone. You should register your own key.

1. Go to [Google Developer Console](https://console.developers.google.com/)
2. Create or select a project
3. Follow the [registering an application](https://developers.google.com/youtube/registering_an_application) instructions to create an API key
4. [Enable the YouTube Data API](https://console.cloud.google.com/apis/library/youtube.googleapis.com) for the project
5. Create a file named `youtube-key.txt` in the plugin directory and paste your API key into it

The plugin directory is inside your `Plug-ins` folder: `YouTube-Agent.bundle/`

---

## Plex token

Newer versions of Plex no longer write agent-provided episode titles back to the
database automatically. This agent works around that by calling the Plex API
directly — but it needs your Plex token to do so. Without it, episode titles will
show as the raw filename instead of the YouTube video title.

The token also enables per-library log files, which is useful for troubleshooting.

### How to set it up

1. Find your Plex token:
   - Log into Plex Web in your browser
   - Open any media item → **...** menu → **Get Info** → **View XML**
   - The URL that opens will contain `X-Plex-Token=xxxxxxxxxxxx` — copy that value
2. Create a file named `X-Plex-Token.id` in your Plex root folder (the same folder that contains `Plug-ins/`) and paste just the token value into it — no newline, no quotes

**Example locations for `X-Plex-Token.id`:**
- Linux: `/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/X-Plex-Token.id`
- macOS: `$HOME/Library/Application Support/Plex Media Server/X-Plex-Token.id`

This is not a standard Plex configuration file — it is specific to this agent (and the HAMA agent).

---

## Downloading videos

Use [yt-dlp](https://github.com/yt-dlp/yt-dlp) (recommended, actively maintained fork of youtube-dl):

```
yt-dlp -o "%(uploader)s [%(channel_id)s]/%(upload_date>%Y)s/[%(upload_date)s] - %(title)s [%(id)s].%(ext)s" https://www.youtube.com/@channelname
```

This produces a layout compatible with this agent, with season folders by year.

Useful flags:
- `--write-info-json` — saves metadata locally; the agent will use this instead of the API, reducing API quota usage
- `--restrict-filenames` — use on Windows to avoid unsupported characters in filenames

---

## Changes from upstream

This fork fixes two bugs that affect **Plex Media Server 1.40+**:

### Episode titles show as filename instead of YouTube title

Newer Plex versions no longer apply legacy agent metadata back to the database after an agent run. The agent correctly fetches titles from the YouTube API and writes them to its metadata bundle, but Plex never reads them back into the database.

**Fix:** After fetching each episode title from YouTube, the agent now calls the Plex API directly (`PUT /library/metadata/{id}`) to update the title in the database. This requires the [Plex token](#plex-token) to be configured.

### Series title shows as "Season XXXX" instead of channel name

When using channel ID mode, the series title was being set from the wrong path component — the season folder (e.g. `Season 2025`) instead of the series root folder (e.g. `dangthatsalongname`). This was caused by the same Plex API access issue: without a valid token, the agent could not determine the library root path and fell back to incorrect values.

**Fix:** The agent now uses the correctly resolved series root folder name (derived from the Plex library API) for the series title.

---

## Troubleshooting

**Episode titles are filenames** — Set up the [Plex token](#plex-token).

**Series title is wrong (shows season folder name)** — Set up the [Plex token](#plex-token).

**No metadata at all** — Check your [YouTube API key](#youtube-api-key). The default key may have hit its quota limit.

**Files not showing up after scan** — This agent handles metadata only. If files aren't appearing, the issue is with Absolute Series Scanner, not this agent. Check scanner logs at:
`Plex Media Server/Logs/ASS Scanner Logs/`

**Metadata not updating after Refresh** — Check agent logs at:
`Plex Media Server/Logs/PMS Plugin Logs/com.plexapp.agents.youtube.log`

---

## History

Forked from [@ZeroQI](https://github.com/ZeroQI)'s [YouTube-Agent.bundle](https://github.com/ZeroQI/YouTube-Agent.bundle), which was itself forked from [@paulds8](https://github.com/paulds8) and [@sander1](https://github.com/sander1)'s earlier movie-only agents.
