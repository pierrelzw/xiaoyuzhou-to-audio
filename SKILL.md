---
name: xiaoyuzhou-to-audio
description: Download one or more Xiaoyuzhou (小宇宙) podcast episodes (up to 6 per call) as local .m4a files using aria2c multi-connection download. Use when the user provides one or more xiaoyuzhoufm.com episode URLs and asks to "下载小宇宙", "下载播客", "保存这期", or wants the audio file(s) of episodes.
---

# xiaoyuzhou-to-audio

Download up to **6 小宇宙 (Xiaoyuzhou FM) episodes** to local `.m4a` files in parallel, using `aria2c` with up to 6 connections per file. Self-contained.

> Transcription is handled separately. After download, point the user at the `video-to-srt` skill if they want a transcript.

## Limits

- **Max 6 URLs per invocation.** If the user provides more, take the first 6 and tell them to re-run for the rest. Beyond 6 the CDN starts throttling and progress reporting becomes unreadable.

## Prerequisites — install if missing

Check both tools first:

```bash
command -v aria2c ffprobe curl
```

If `aria2c` is missing, **install it before proceeding** — do not silently fall back to curl:

```bash
brew install aria2
```

If `ffprobe` is missing:

```bash
brew install ffmpeg
```

After install, re-run the `command -v` check to confirm. If `brew` itself is unavailable, stop and tell the user to install Homebrew (`https://brew.sh`).

## Workflow

Work in `~/Downloads/` by default.

### 1. Resolve audio URL and title for each episode

For each URL the user provided (loop, max 6):

```bash
HTML=$(curl -s -A "Mozilla/5.0" "<EPISODE_URL>")
AUDIO_URL=$(echo "$HTML" | grep -oE 'https://media\.xyzcdn\.net/[^"]*\.m4a' | head -1)
RAW_TITLE=$(echo "$HTML" | grep -oE '<title>[^<]+</title>' | head -1 | sed -E 's/<\/?title>//g')
```

If `AUDIO_URL` is empty for any episode, report that one as failed and continue with the rest.

Sanitize `RAW_TITLE` into `$PREFIX`: strip the trailing ` - <播客名> | 小宇宙 - 听播客，上小宇宙`, replace `/` and whitespace runs with `_`. Keep it short and shell-safe. Make sure the 6 prefixes are unique — if two collide, suffix with `_2`, `_3`, etc.

### 2. Download all episodes in parallel via aria2c

Build an aria2c input file (one URL + filename per pair), then run a single aria2c that downloads everything concurrently:

```bash
cd ~/Downloads
cat > .xyz_input.txt <<EOF
<AUDIO_URL_1>
  out=<PREFIX_1>.m4a
<AUDIO_URL_2>
  out=<PREFIX_2>.m4a
...
EOF

aria2c -i .xyz_input.txt \
  -j 6 -x 6 -s 6 -k 1M \
  --user-agent="Mozilla/5.0" \
  --console-log-level=warn --summary-interval=5 \
  --auto-file-renaming=false --allow-overwrite=false
```

Flags:
- `-j 6` up to 6 files in parallel
- `-x 6 -s 6` 6 connections per file, 6 segments (CDN-friendly cap)
- `-k 1M` 1MB segment size
- `--summary-interval=5` prints a progress summary every 5s (use this as the live progress, no separate Monitor needed for short jobs)
- `--allow-overwrite=false` skip files that already exist (idempotent re-runs)

For long downloads (>5 min expected), launch aria2c with `run_in_background: true` and use a Monitor that tails the log every 15–30s.

aria2c handles resume automatically — if a download is interrupted, just re-run the same command.

### 3. Verify each file

For each `$PREFIX.m4a` produced:

```bash
sz=$(stat -f%z ~/Downloads/$PREFIX.m4a)
dur=$(ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 ~/Downloads/$PREFIX.m4a)
echo "$PREFIX: $((sz/1024/1024)) MB, ${dur}s"
```

`aria2c` exit code 0 means all files downloaded completely; you only need ffprobe to surface the duration to the user. If exit code ≠ 0, list which files failed (aria2c prints a summary at the end).

### 4. Report

Per-episode line: `<title> — <MB> MB, <h:mm:ss>, <output path>`. Then list any failures.

Delete `.xyz_input.txt` when done.

## Notes & gotchas

- The CDN host is `media.xyzcdn.net`. URLs are public; no cookies/auth needed.
- Episode title sanitization: typical pattern is `<EP title> - <show name> | 小宇宙 - 听播客，上小宇宙` — strip everything from ` - ` onward (or just the ` | 小宇宙...` tail) to keep `$PREFIX` short.
- aria2c is dramatically faster than curl on xyzcdn (typical 30–50 MB/s vs 3–8 MB/s single-connection).
- If the user provides >6 URLs, take the first 6, finish them, then tell the user to re-invoke for the remaining ones.
