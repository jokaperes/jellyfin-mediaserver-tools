# jellyhand

*Hands-off Jellyfin automation.*


Three small, dependency-light scripts that automate a self-hosted Jellyfin
pipeline: **grab a torrent → fetch the best subtitle in your language →
rescan Jellyfin → get a phone notification.** Built for a headless Linux box
(originally a Raspberry Pi 5) but portable to any machine with Python 3,
`curl`, and a qBittorrent + Jellyfin install.

Subtitles default to **English** and support 16+ languages out of the box —
pick yours with one config line (see
[Choosing the subtitle language](#choosing-the-subtitle-language)).

> 🇧🇷 Versão em português: [`README.pt.md`](README.pt.md)

## Components

| Script     | Language | Role |
|------------|----------|------|
| `grab`     | bash     | Queue a magnet/torrent into qBittorrent under the right category (`anime`/`movies`/`shows`). |
| `postdl`   | bash     | qBittorrent "run on completion" hook. Calls `fetchsub`, triggers a Jellyfin rescan, sends an ntfy push. |
| `fetchsub` | python3  | Download the best subtitle (your chosen language) for a file/folder, saved as a `<video>.<iso>.srt` sidecar. Standalone-usable. |

Each is independent — you can use `fetchsub` on its own without the torrent
parts. They share one optional config file and one optional project log.

## How it fits together

```
   you ──grab anime <magnet>──▶ qBittorrent ──(on complete)──▶ postdl "%F" "%L" "%N" "%D"
                                                                  │
                                          ┌───────────────────────┼───────────────────────┐
                                          ▼                       ▼                       ▼
                                      fetchsub              Jellyfin rescan           ntfy push
                                 (writes .<iso>.srt)      (Library/Refresh)        (📺 to your phone)
```

## Install

### 1. Config file
```bash
mkdir -p ~/.config/mediaserver
cp config.env.example ~/.config/mediaserver/config.env
$EDITOR ~/.config/mediaserver/config.env   # fill in URLs + Jellyfin API key + ntfy topic
```

### 2. Subtitle provider keys
`fetchsub` reads keys from fixed paths (so they never live in the repo or env):
```bash
mkdir -p ~/.config/opensubtitles ~/.config/subdl
echo "YOUR_OPENSUBTITLES_API_KEY" > ~/.config/opensubtitles/api_key   # required
echo "YOUR_SUBDL_API_KEY"         > ~/.config/subdl/api_key           # optional (tier-2 fallback)
chmod 600 ~/.config/opensubtitles/api_key ~/.config/subdl/api_key
```
- OpenSubtitles key: https://www.opensubtitles.com/en/consumers (free tier ≈ 100 downloads/day).
- SubDL key: https://subdl.com/panel/api — **optional**. If the file is absent, `fetchsub` silently skips SubDL.

### 3. Deploy the scripts
```bash
sudo install -m 755 fetchsub postdl grab /usr/local/bin/
```

### 4. Wire qBittorrent
In qBittorrent: **Options → Downloads → "Run external program on torrent completion"**:
```
/usr/local/bin/postdl "%F" "%L" "%N" "%D"
```
`postdl` only runs `fetchsub` for the categories `movies`, `shows`, `anime`
(the ones `grab` assigns), so set categories accordingly.

## Usage

```bash
grab movies "magnet:?xt=urn:btih:..."          # one or many links
grab shows  "https://example.org/file.torrent"
grab anime  "magnet:?xt=..." "magnet:?xt=..."

fetchsub "/srv/media/movies/Some Movie (1999)/Some Movie (1999).mkv"
fetchsub "/srv/media/shows/A Show/Season 01"   # whole folder, recursive
fetchsub --imdb 133093 "/path/The Matrix.mkv"  # force a title id
```

## Choosing the subtitle language

The default is **English (`en`)**, but any language is a first-class citizen.
Set `SUB_LANG` to whatever you want — one-off:

```bash
SUB_LANG=es fetchsub "/path/Movie.mkv"      # Spanish, just this run
```
…or permanently in `~/.config/mediaserver/config.env` (`postdl` exports it to
`fetchsub`, so the whole pipeline follows):
```bash
SUB_LANG="fr"          # French; sidecars become .fre.srt
SUB_LANG_FALLBACK="0"  # 1 = also accept the broader variant when exact is missing
```

Built-in languages (each maps to the right OpenSubtitles code, SubDL code, and
Jellyfin `.iso` sidecar): `en` `es` `es-mx` `fr` `de` `it` `nl` `pl` `ru` `ja`
`ko` `zh-cn` `ar` `tr` `pt-br` `pt`. **Any other OpenSubtitles language code
also works** (best-effort sidecar). To add or fine-tune one, edit the
`LANG_TABLE` dict at the top of `fetchsub` — it's a single, well-commented table.

**Strict by default:** only the exact variant is downloaded — `es` won't grab
`es-MX`, and `pt-br` (Brazilian) will never grab `pt` (European Portuguese) or
vice-versa. Set `SUB_LANG_FALLBACK=1` if you'd rather take the broader language
than nothing.

## How `fetchsub` chooses a subtitle

It saves the sidecar as **`<video>.<iso>.srt`** (e.g. `.eng.srt` for English,
`.spa.srt` for Spanish, `.por.srt` for Portuguese) so Jellyfin auto-selects the
language — *not* `.en.srt` / `.pt-br.srt`, which Jellyfin won't recognise.

Provider order, stopping at the first confident match:

1. **OpenSubtitles** (`api.opensubtitles.com`)
   - **moviehash** (exact file hash) — best possible sync.
   - **TV episodes → `parent_imdb_id` + season/episode.** This is the key trick:
     OpenSubtitles' free-text `query` returns *nothing* for many series, so
     `fetchsub` resolves the show's IMDb id once via `/features?query=…&type=tv`
     (cached per run) and searches by that. For a TV file, `--imdb` is treated
     as the show's *parent* id.
   - **Movies → `imdb_id`** (when `--imdb` is given), else a title+year-filtered
     free-text query, relaxed once by dropping the year.
   - Ranking (`os_best`): moviehash match → language tier (exact variant beats
     the broader fallback, e.g. **pt-BR > pt-PT**) → **source-type match**
     (WEB-DL/BluRay/etc — same source ≈ same fps/cut ≈ correct timing) →
     trusted uploader → download count. Results outside the configured
     language set are discarded before ranking.
2. **SubDL** (`api.subdl.com`) — only if `~/.config/subdl/api_key` exists.
   ⚠️ **SubDL language codes are non-standard:** Brazilian Portuguese is
   `BR_PT` (not `PT-BR`/`PB`/`BR`, which error out); Portugal is `PT`.

Downloaded subs are run through `strip_ads` (removes provider ad blocks,
strips a leading BOM, and renumbers cues).

## Notes & gotchas

- **Episode-order mismatches happen.** Some rips number episodes differently
  from the subtitle DB, so a correctly-named `SxxExx` sidecar can still be the
  wrong episode. If a sub looks off, verify by content (extract the embedded
  English with `ffmpeg -i file.mkv -map 0:<idx> out.srt` and compare timing).
- **Logs:** verbose per-video log at `~/.config/opensubtitles/fetchsub.log`;
  concise shared lifecycle log at `~/.config/mediaserver/mediaserver.log`
  (both overridable via `LOG_FILE` / `PROJECT_LOG`). Check these first when
  something silently didn't appear.
- **Security:** no secrets live in this repo. Keys are read from
  `~/.config/...`, and `config.env` / `api_key` / `*.log` are gitignored. Keep
  it that way — don't paste keys, IPs, or your ntfy topic into tracked files.
- **qBittorrent auth:** `grab`/`postdl` hit the Web UI API with no credentials,
  which assumes either `bypass_local_auth` for localhost or an otherwise open
  local API. Lock the API down to localhost/your VPN.

See [`AGENTS.md`](AGENTS.md) for a step-by-step replication guide aimed at AI
coding agents.

## License

MIT — see [`LICENSE`](LICENSE).
