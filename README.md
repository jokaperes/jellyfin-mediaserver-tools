# Jellyfin Mediaserver Automation Tools

A unified, lightweight, and robust pipeline for automating subtitles, library indexing, and mobile push notifications on personal Jellyfin servers.

This suite is perfect for media servers running on low-resource machines (like a Raspberry Pi 5) and consists of three core components:

1. **`fetchsub`**: An intelligent subtitle downloader that automatically searches, matches, and grabs the best Portuguese (`pt-BR`) subtitles from OpenSubtitles (via their REST API using parent TV Show IMDb lookups) and SubDL.
2. **`postdl`**: A post-download automation hook for qBittorrent that triggers `fetchsub`, rescans your Jellyfin library, and pushes elegant, emoji-rich status notifications directly to your phone via `ntfy.sh`.
3. **`grab`**: A command-line helper to easily queue torrent or magnet links into qBittorrent under the correct Jellyfin directory categories (`anime`, `movies`, `shows`).

---

## 🚀 Key Features

* **Smart TV Show Subtitle Matching:** Resolves the parent TV Show IMDb ID dynamically to get extremely reliable episode subtitle matches, bypassing OpenSubtitles' fragile text search constraints.
* **Hash-based Movie Matching:** Matches movies directly by file hashes to ensure flawless subtitle synchronization.
* **No-Compromise Security:** All credentials, API keys, URLs, and paths are completely externalized into a single local `config.env` file—keeping your code clean and perfectly safe for public hosting.
* **Automated Jellyfin Library Rescans:** Instantly alerts Jellyfin to refresh when new content is ready, so sidecar subtitles and new media appear immediately.
* **Rich Push Notifications:** Uses [ntfy.sh](https://ntfy.sh) to send neat, formatted status updates with custom titles, priority tagging, and subtitle confirmation notices in Portuguese (e.g., `📺 The Boondocks S03E13 já está disponível no Jellyfin — com legenda PT-BR`).

---

## 🛠️ Repository Structure

```text
jellyfin-mediaserver-tools/
├── fetchsub           # Python script for intelligent subtitle downloading
├── postdl             # Bash script triggered by qBittorrent on completion
├── grab               # Bash script to queue torrents with auto-categories
├── config.env.example # Template environment file for credentials
├── .gitignore         # Prevents committing environment configs
└── README.md          # This documentation
```

---

## 📦 Setup & Installation

### 1. Prerequisite Directory Structure
Ensure your config directories exist:
```bash
mkdir -p ~/.config/mediaserver
mkdir -p ~/.config/opensubtitles
mkdir -p ~/.config/subdl
```

### 2. Configure Your Environment Keys
Copy the example configuration to your local config folder:
```bash
cp config.env.example ~/.config/mediaserver/config.env
chmod 600 ~/.config/mediaserver/config.env
```
Open `~/.config/mediaserver/config.env` and populate your API credentials:
* **`JELLYFIN_KEY`**: Create an API key in your Jellyfin Dashboard > Advanced > API Keys.
* **`NTFY_URL`**: Set up a custom `ntfy.sh` topic (e.g., `https://ntfy.sh/your_custom_topic`).
* **`QBT_URL`**: The Web UI address of your qBittorrent instance.

For the OpenSubtitles/SubDL API keys used by `fetchsub`, add them directly:
```bash
echo "your_opensubtitles_api_key" > ~/.config/opensubtitles/api_key
echo "your_subdl_api_key" > ~/.config/subdl/api_key
```

### 3. Deploy the Scripts
Install the scripts in `/usr/local/bin` and make them executable:
```bash
sudo cp fetchsub postdl grab /usr/local/bin/
sudo chmod +x /usr/local/bin/fetchsub /usr/local/bin/postdl /usr/local/bin/grab
```

### 4. Wire Up qBittorrent
Open your qBittorrent Web UI or settings, and navigate to **Downloads** > **Run external program on torrent completion**. Paste the following hook command:

```bash
/usr/local/bin/postdl "%F" "%L" "%N" "%D"
```

---

## 📖 Usage Examples

### Manual Subtitle Downloading
To download subtitles for a single video file or a folder of files:
```bash
fetchsub "/srv/media/movies/Taxi Driver (1976)/Taxi Driver.mkv"
```
Or force a search by parent IMDb ID:
```bash
fetchsub --imdb tt0373732 "/srv/media/shows/The Boondocks/S01E01.mkv"
```

### Queuing Downloads from CLI
Queue magnet or torrent links instantly under a specific category folder:
```bash
grab anime "magnet:?xt=urn:btih:..."
grab movies "https://yts.mx/...torrent"
```

---

## 🔒 Security & Privacy Notice
This suite respects your server's security:
* Does **not** include hardcoded tokens or secrets.
* The `.gitignore` prevents `config.env` and local configuration backups from ever being tracked by git.
