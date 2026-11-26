# Tools

AI summaries of YouTube videos, as a command line tool and as a small web front end.

| Project | Description | Docs |
| ------- | ----------- | ---- |
| [yt-digest](yt-digest/README.md) | .NET CLI. Downloads subtitles with `yt-dlp` and summarizes them with an LLM (Claude, DeepSeek or Ollama). Digest mode processes all channels from `channels.txt`; single-video mode prints one summary as Markdown to stdout. | [yt-digest/README.md](yt-digest/README.md) |
| [yt-digest-web](yt-digest-web/README.md) | ASP.NET Core (.NET 10) web app. Enter a YouTube URL and a subtitle language and get the summary rendered in the browser. Password-protected; also offers `POST /api/summarize`. It runs `yt-digest` as an external process. | [yt-digest-web/README.md](yt-digest-web/README.md) |

```
Browser ──► yt-digest-web ──► yt-digest ──► yt-dlp (subtitles)
                                   └──────► LLM API (summary)
```

## Docker

`Dockerfile.yt-digest-web` in the repository root builds a single image containing the web app,
the `yt-digest` CLI and all its dependencies. It uses a multi-stage build:

1. `yt-digest` is published as a self-contained single-file `linux-x64` executable.
2. `yt-digest-web` is published framework-dependent.
3. The final `mcr.microsoft.com/dotnet/aspnet:10.0` image receives both, plus `yt-dlp`,
   `deno`, `python3` and `ffmpeg`. `yt-digest` is installed in `/usr/local/bin` (in `PATH`).

The image listens on port `8080` (`ASPNETCORE_URLS=http://+:8080`).

### Build

Run from the repository root (the build context must be the root, since both projects are copied):

```bash
docker build -f Dockerfile.yt-digest-web -t yt-digest-web .
```

The image is `linux-x64`. For ARM (e.g. Raspberry Pi) change the `--runtime` of the
`yt-digest` publish step and the `yt-dlp` download URL (see comments in the Dockerfile).

### Run

```bash
docker run -d --name yt-digest-web \
  -p 8080:8080 \
  -e Auth__Password="my-secret" \
  -e DEEPSEEK_API_KEY="sk-..." \
  -v yt-digest-data:/root/.local/share/yt-digest \
  yt-digest-web
```

Then open <http://localhost:8080> and log in with the password.

- `Auth__Password` – login password (the default `123` is for development only).
- `DEEPSEEK_API_KEY` (or `ANTHROPIC_API_KEY`) – LLM API key. Alternatively mount `claude.key` /
  `deepseek.key` into the data directory of `yt-digest`. Without any key `yt-digest` falls back to
  Ollama, which must then be reachable from the container.
- The volume is optional; it keeps the summaries and `processed_videos.json` between runs. The
  data path is the app data directory of the user running the container (root by default).

The app uses plain HTTP, so put it behind a TLS reverse proxy before exposing it beyond a trusted
network.

### Stop / remove

```bash
docker stop yt-digest-web && docker rm yt-digest-web
```
