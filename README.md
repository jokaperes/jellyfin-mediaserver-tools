# Jellyfin Mediaserver Tools

A collection of lightweight scripts to automate subtitle downloads, library rescans, and mobile push notifications for Jellyfin media servers.

## Components

1. **fetchsub**: Downloads Portuguese (pt-BR) subtitles from OpenSubtitles and SubDL.
2. **postdl**: Triggered by qBittorrent on completion. Runs fetchsub, triggers a Jellyfin library rescan, and sends a push notification via ntfy.sh.
3. **grab**: Command-line utility to queue torrents into qBittorrent under correct categories (anime, movies, shows).

## Installation

### 1. Configuration File
Create the configuration directory and copy the template:
```bash
mkdir -p ~/.config/mediaserver
cp config.env.example ~/.config/mediaserver/config.env
```
Fill in the API keys and URLs inside `~/.config/mediaserver/config.env`.

### 2. OpenSubtitles API Keys
Save your API keys to their respective paths:
```bash
echo "your_opensubtitles_api_key" > ~/.config/opensubtitles/api_key
echo "your_subdl_api_key" > ~/.config/subdl/api_key
```

### 3. Deploy Scripts
Move the scripts to your system binaries directory and make them executable:
```bash
sudo cp fetchsub postdl grab /usr/local/bin/
sudo chmod +x /usr/local/bin/fetchsub /usr/local/bin/postdl /usr/local/bin/grab
```

### 4. qBittorrent Integration
Add the following command to qBittorrent's "Run external program on torrent completion" setting:
```bash
/usr/local/bin/postdl "%F" "%L" "%N" "%D"
```
