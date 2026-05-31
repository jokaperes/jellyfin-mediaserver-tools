# AGENTS.md — replication guide for coding agents

This file is for an AI coding agent (Claude Code, Codex, etc.) tasked with
**reproducing this Jellyfin automation pipeline on a fresh machine**, or
adapting it. It assumes you can run shell commands on the target host. Read it
end to end before touching anything.

## What you are building

A completion-driven media pipeline: `grab` queues torrents → qBittorrent runs
`postdl` on completion → `postdl` calls `fetchsub` (subtitles), triggers a
Jellyfin rescan, and pushes an ntfy notification. The three scripts are in the
repo root. Deploy them to `/usr/local/bin/`.

## Hard rules (do not violate)

1. **Never commit secrets.** API keys live only in `~/.config/opensubtitles/api_key`,
   `~/.config/subdl/api_key`, and `~/.config/mediaserver/config.env`. All three
   are gitignored. Do not hardcode keys, real IPs, hostnames, ntfy topics, or
   absolute home paths into tracked files. Before any commit, grep the diff for
   leaks (keys, `192.168.*`, tailnet `100.*` IPs, `ntfy.sh/<topic>`).
2. **Subtitle sidecars must be named `<video>.<iso639-2>.srt`** — `.por.srt`
   for Portuguese, `.eng.srt` for English, etc. Jellyfin reads the ISO-639-2
   code; it ignores `.pt-br`/`.pt-BR`/`.pb`. `fetchsub` derives this from the
   `SUB_LANG` env var (default `pt-br` → `por`); `postdl` mirrors the same map.
3. **Do not seed / do not open inbound ports** if the host owner runs privacy-
   first (firewalled, VPN-only). Lock the qBittorrent + Jellyfin APIs to
   localhost or the private network.
4. **Verify, don't assume.** A correctly-named `SxxExx` sub can be the wrong
   episode (rips reorder episodes). When in doubt, extract the embedded English
   track and compare timing before declaring a sub "good".

## Prerequisites on the target host

- Python 3 (stdlib only — no pip packages needed), `curl`, `bash`.
- qBittorrent (`qbittorrent-nox` is fine) with the Web UI API reachable.
- Jellyfin, with an API key (Dashboard → API Keys).
- An ntfy topic (https://ntfy.sh — pick an unguessable topic name).
- Optional but recommended for subtitle QA: `ffmpeg`/`ffprobe` (e.g.
  `jellyfin-ffmpeg` at `/usr/lib/jellyfin-ffmpeg/`) to extract embedded subs.

## Step-by-step

1. **Clone & inspect.** Read `fetchsub`, `postdl`, `grab` so you understand the
   data flow and the config/env vars each one consumes.
2. **Config.** `cp config.env.example ~/.config/mediaserver/config.env` and fill
   `JELLYFIN_URL`, `JELLYFIN_KEY`, `NTFY_URL`, `QBT_URL`. Leave log vars unset
   to use the per-user defaults.
3. **Keys.** Write `~/.config/opensubtitles/api_key` (required) and optionally
   `~/.config/subdl/api_key`; `chmod 600` both.
4. **Deploy.** `sudo install -m 755 fetchsub postdl grab /usr/local/bin/`.
5. **Smoke-test `fetchsub` standalone** on one existing video before wiring the
   hook: `fetchsub "/path/to/one episode.mkv"` and read
   `~/.config/opensubtitles/fetchsub.log`. Confirm a `.por.srt` appears.
6. **Wire qBittorrent:** set the completion program to
   `/usr/local/bin/postdl "%F" "%L" "%N" "%D"`. Categories must be one of
   `anime`/`movies`/`shows` for subtitle fetching to trigger.
7. **End-to-end test:** `grab movies "<magnet>"`, wait for completion, confirm
   the notification fires and the sidecar lands. Tail
   `~/.config/mediaserver/mediaserver.log` for the lifecycle lines.

## Subtitle logic you must preserve when modifying `fetchsub`

- **Language is config-driven via `SUB_LANG`** (default `pt-br`). The
  `LANG_TABLE` dict maps each language key to its OpenSubtitles codes, SubDL
  codes, and ISO-639-2 sidecar suffix. To support a new language, add a row —
  do *not* sprinkle language strings through the code.
- **Strict by default:** only the exact variant is queried and downloaded
  (`OS_ACCEPT` is a hard allow-list). A `pt-br` run must never produce a
  `pt-PT` file. `SUB_LANG_FALLBACK=1` opts into the broader variant.
- **TV uses `parent_imdb_id` + season/episode**, not free-text `query` (which
  returns 0 for many series). The show id is resolved via
  `/features?query=<title>&type=tv` and cached per run. For a TV file, `--imdb`
  means the *show's parent* id; for a movie it means the movie's id.
- **Ranking order** (`os_best`): moviehash match → language tier (exact variant
  beats the broader fallback) → source-type match (WEB-DL/BluRay…) → trusted →
  download count. Source-type matters: a more-downloaded WebRip sub can drift
  on a WEB-DL video.
- **SubDL language codes are non-standard:** Brazilian Portuguese is `BR_PT`
  (not `PT-BR`/`PB`/`BR`); Portugal is `PT`.
- `strip_ads` removes provider ad blocks, strips a leading BOM, renumbers cues.
  Keep that on every downloaded sub.

## Validating a subtitle (the reliable method)

Extract the video's embedded English and score a candidate pt-BR by
language-invariant **anchor tokens** (character names + numbers) at aligned
timestamps in ~5s buckets, trying small offsets:

```bash
/usr/lib/jellyfin-ffmpeg/ffmpeg -i "file.mkv" -map 0:<engSubIdx> /tmp/en.srt
# correct episode ≈ 10–50% anchor overlap; wrong episode ≈ 0–3%.
# high cosine but low time-aligned overlap = right episode, just desynced
# (fix with a constant time shift rather than swapping the file).
```

## Common failure modes (already solved here)

- **"No subtitles found" for a series** → you used free-text `query`; switch to
  `parent_imdb_id`.
- **Sub is the wrong episode despite right filename** → rip reorders episodes;
  match by content, not by `SxxExx`.
- **Container can't reach a host service** (e.g. a dockerised companion app) →
  a UFW `INPUT/FORWARD DROP` policy blocks container→host traffic; allow the
  bridge subnet explicitly.
- **Sub appears but Jellyfin won't show Portuguese** → wrong language code in
  the filename; it must be `.por.srt`.

## When done

Grep the working tree and the full git history for secrets one more time, then
report what you changed. Do not force-push or rewrite shared history without
the owner's say-so.
