# xiaoyuzhou-to-audio

Download Xiaoyuzhou (小宇宙) podcast episodes to local `.m4a` files using `aria2c` multi-connection download — up to 6 episodes in parallel, 16 connections per file.

## Install

```bash
claude plugin marketplace add pierrelzw/zhiwei_skills && claude plugin install xiaoyuzhou-to-audio@pierrelzw --scope user
```

## Prerequisites

```bash
brew install aria2 ffmpeg
```

## Usage

Just paste one or more 小宇宙 episode URLs and ask Claude:

- `下载这期播客 https://www.xiaoyuzhoufm.com/episode/...`
- `保存这几期：<url1> <url2> <url3>`
- `download this xiaoyuzhou episode <url>`

The skill triggers on: "下载小宇宙", "下载播客", "保存这期", or any `xiaoyuzhoufm.com` URL with intent to save audio.

## Limits

- **Max 6 URLs per invocation.** Beyond that the CDN starts throttling. The skill takes the first 6 and tells you to re-run for the rest.

## Output

- Files land in `~/Downloads/<sanitized-title>.m4a`
- aria2c handles resume — re-run the same command to continue an interrupted download
- For transcripts, follow up with the [video-to-srt](https://github.com/pierrelzw/video-to-srt) skill

## License

MIT
