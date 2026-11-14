# NovelTrax EPUB fetcher

Standalone Playwright + FlareSolverr sidecar that resolves the link-hider chain
behind a NovelTrax release's **Source** link and downloads the final EPUB.

## Chain

1. `POST /fetch` with the source URL (shortener such as `vplink.cc`, then
   `zshort.link`/`charexempire`/Turnstile gate).
2. If the page is a Cloudflare/Turnstile challenge, FlareSolverr
   (`request.get`) returns the cleared URL, cookies and User-Agent; Playwright
   rebuilds its context with those values.
3. Playwright follows countdown pages, clicks visible `Get Link`/`Download`/
   `Continue` controls, and captures the browser download (also handles a
   direct `*.epub` anchor).
4. The file is returned with its original filename.

A successfully solved URL is remembered, so a residual Turnstile iframe or
"verify you are human" banner does not re-trigger the gate before the page's
download controls are clicked. A manual checkbox click is only trusted after
the challenge is confirmed gone; otherwise FlareSolverr still runs.

## Run

```bash
docker build -t noveltrax-epub-fetcher .
docker run --rm -p 127.0.0.1:8010:8010 \
  -e EPUB_FETCHER_FLARESOLVERR_URL=http://flaresolverr:8191 \
  noveltrax-epub-fetcher
```

`docker-compose.yml` service:

```yaml
epub-fetcher:
  build: .
  environment:
    EPUB_FETCHER_FLARESOLVERR_URL: http://flaresolverr:8191
    # Optional: Tab count when a source's standalone Turnstile needs focus.
    EPUB_FETCHER_FLARESOLVERR_TABS: "1"
    # Post-solve wait so the challenged page can swap in its download controls.
    EPUB_FETCHER_FLARESOLVERR_WAIT: "2"
  expose: ["8010"]
```

Keep it on an internal network in production; `8010` is unauthenticated.
The `docker run` example above publishes only to loopback for local testing.

Standalone CLI (same image):

```bash
docker run --rm noveltrax-epub-fetcher \
  python epub_fetcher.py "https://vplink.cc/..." --out /tmp/books
```

Add `--show` for a visible browser only when running outside the container.

## NovelTrax settings

Set **Settings → Downloader → EPUB Fetcher URL** to `http://epub-fetcher:8010`
(or `http://localhost:8010` for a local run). The per-release **Fetch EPUB**
button downloads into `downloads_dir` and records the release as
`COMPLETED` (engine `http`) with the file path in `download_files_json`.

## Environment

| Variable | Default | Meaning |
| --- | --- | --- |
| `EPUB_FETCHER_FLARESOLVERR_URL` | empty | FlareSolverr base URL; required for Cloudflare/Turnstile gates |
| `EPUB_FETCHER_FLARESOLVERR_WAIT` | `2` | Seconds FlareSolverr waits after solving |
| `EPUB_FETCHER_FLARESOLVERR_TABS` | unset | Tab count for a standalone Turnstile widget |
| `EPUB_FETCHER_TIMEOUT` | `150` | Per-fetch browser deadline in seconds |
| `EPUB_FETCHER_OUTPUT_DIR` | system temp | Temporary download directory; files are deleted after the response |

## Tests

```bash
python tests/test_epub_fetcher.py
```

The helper suite is hermetic (no browser/network). The browser flow is expected
to be smoke-tested against a local challenge/countdown fixture in CI or before
deploying.

## Security

The service fetches whatever URL it is given and returns the bytes; bind it to
the internal Docker network or loopback only, never expose port 8010 publicly.

## Limits

- Playwright drives a real Chromium; selector/label heuristics can break when a
  shortener changes its UI. `/health` reports whether the browser is available,
  and failures return the last flow steps in `X-Fetch-Steps`.
- No CAPTCHA billing service is used: interactive captchas (not Turnstile) still
  block the flow by design.
- The file is fetched over the browser/HTTP client, so the fetcher must be able
  to reach the CDN host; it does not bypass DRM or paywalls.
