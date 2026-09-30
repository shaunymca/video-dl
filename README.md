# video-dl

One command to download videos from **YouTube**, **Dropout.tv** and **Twitch**.

It's a thin wrapper around [`yt-dlp`](https://github.com/yt-dlp/yt-dlp) that picks
sensible per-site defaults, so you don't have to remember format selectors,
output templates or auth flags.

```bash
video-dl 'https://www.youtube.com/watch?v=...'
video-dl -b chrome 'https://watch.dropout.tv/dimension-20-fantasy-high/season:1'
video-dl 'https://www.twitch.tv/videos/123456789'
```

## Install

```bash
brew install yt-dlp ffmpeg          # dependencies
./video-dl --setup                  # pick your download folder & quality
```

`--setup` prompts you interactively. If it can't find a terminal on stdin — a
pipe, a CI job, or Claude Code's `!` prefix — it writes defaults instead of
hanging, and you can supply the values as flags **before** `--setup`:

```bash
./video-dl -o ~/Movies/Downloads -q 1080 -b chrome --setup
```

To call it from anywhere, put it on your `PATH`:

```bash
ln -s "$PWD/video-dl" /opt/homebrew/bin/video-dl
```

Check everything's wired up with `video-dl --check`.

## Usage

| Option | What it does |
| --- | --- |
| `-o, --dest DIR` | Where to save (default `~/Videos`, with a per-site subfolder) |
| `-q, --quality N` | Cap height: `480`, `720`, `1080`, `1440`, `2160`, or `best` |
| `-a, --audio` | Audio only, as `.m4a` |
| `-b, --browser NAME` | Borrow login cookies from `chrome`/`safari`/`firefox`/`brave`/`edge` |
| `--any-codec` | Allow AV1/VP9/Opus (smaller files, worse Apple playback — see below) |
| `--keep-brand` | Keep Dropout's brand prefix (`dimension_20_fantasy_high/`) |
| `--show NAME` | Override the show folder, e.g. `fantasy_high` |
| `--season NAME` | Override the season folder, e.g. `season_1` |
| `--no-subs` | Don't embed subtitles |
| `--archive` | Skip anything already downloaded — handy for re-running a playlist |
| `-n, --dry-run` | Print the `yt-dlp` command instead of running it |

Anything after `--` goes straight to `yt-dlp`:

```bash
video-dl 'https://www.youtube.com/playlist?list=...' -- --playlist-items 1-5
```

## Where files land

Everything goes under your configured folder, in a per-site tree:

```
downloads/youtube/<Channel>/<Title> [<id>].mp4
downloads/twitch/<Streamer>/2024-08-20 - <Title> [<id>].mp4
downloads/dropout/<show>/<season>/S01E01 - <Title>.mp4
```

Dropout shows are slugged to lowercase with underscores. Dropout names shows
`Brand: Show` — Fantasy High is reported as `Dimension 20: Fantasy High` — and
since each campaign is its own show to a viewer, the brand prefix comes off:

```
downloads/dropout/fantasy_high/season_1/S01E01 - The Beginning Begins.mp4
```

Pass `--keep-brand` to file it under `dimension_20_fantasy_high/` instead.

Dropout describes its catalogue in two different ways depending on the show —
sometimes the show name is the series and seasons are numbered, sometimes the
series is the brand ("Dimension 20") and the season itself is named. `video-dl`
makes one cheap metadata lookup per URL to tell which, and lays the folders out
accordingly. Shows with no season metadata at all (one-off specials) skip the
season folder and get a bare `<Title>.mp4` instead of `S00E00 - <Title>.mp4`.

If you'd rather name the folders yourself, override either level:

```bash
video-dl --show fantasy_high --season season_1 'https://watch.dropout.tv/...'
```

## Logging in to Dropout

Dropout is subscriber-only, so every request needs your session cookies. Log in
normally in your browser, then point `video-dl` at it:

```bash
video-dl -b chrome 'https://watch.dropout.tv/dimension-20-fantasy-high/season:1'
```

Use the `watch.dropout.tv` links from the player — that's the host yt-dlp's
Dropout extractor understands. Bare `dropout.tv` and `www.dropout.tv` links are
rewritten to it automatically.

Set `BROWSER="chrome"` in `~/.config/video-dl/config` to make that the default.

Two macOS gotchas:

- **Chrome/Brave/Edge**: cookies are encrypted with a key in your login
  keychain, so the first run pops a keychain prompt. Allow it.
- **Safari**: reading its cookie jar needs Full Disk Access for your terminal
  (System Settings → Privacy & Security → Full Disk Access).

The same flag works for sub-only Twitch VODs and age-restricted YouTube videos.

## A note on codecs

YouTube's highest-quality streams are AV1 or VP9 video with Opus audio. Those
are more efficient, but QuickTime, Photos and iMovie won't play them — even
inside an `.mp4` container. So by default `video-dl` asks for **H.264 + AAC**,
which plays everywhere on a Mac, and only falls back to the modern codecs when
nothing else is available (common above 1080p).

If you'd rather have the smaller files and play everything in VLC or IINA, use
`--any-codec`, or set `ANY_CODEC=1` in your config.

## Config

`~/.config/video-dl/config` is plain shell, sourced at startup:

```bash
DEST="/Users/you/dev/video-dl/downloads"
QUALITY="720"
BROWSER="chrome"
# ANY_CODEC=1
# SUBS=0
```

## Keeping it working

Streaming sites change constantly and `yt-dlp` keeps up; the wrapper rarely
needs touching. When a download suddenly breaks, update first:

```bash
brew upgrade yt-dlp
```

## Scope

This downloads things you already have access to — public YouTube videos, VODs
on a Twitch channel you can watch, and Dropout content behind your own
subscription. It doesn't bypass DRM or paywalls.
