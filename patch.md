# Fork status (vs upstream `redlib-org/redlib`)

Base: upstream `main` at `ba116c1` (2026-10-21, incl. v0.40.0 release, Fastly dual-stack, anonymous comments transport fix).

Two local changes are kept on top, plus one partial upstream-PR cherry-pick:

## 1. CMAF video support — `a3abe21`

Original commit `174cd14` by Mateusz Jabłoński, re-applied onto the new base.
Upstream PR: https://github.com/redlib-org/redlib/pull/506

- `src/main.rs`: split `/vid/:id/:size` into `/vid/:id/dash/:size` → `DASH_{size}` and `/vid/:id/cmaf/:size` → `CMAF_{size}`
- `src/utils.rs`: `REGEX_URL_VIDEOS` matches `(DASH|CMAF)_…`, `format_url` emits `/vid/{id}/{dash|cmaf}/{size}.mp4`, tests updated

Reddit serves both DASH and CMAF renditions; without this, CMAF video links don't proxify.

## 2. Private GHCR docker workflow — `4dabe3a`

- `.github/workflows/main-docker.yml`: image `ghcr.io/jrandiny/redlib`, login via `GITHUB_TOKEN`, amd64-only matrix, `workflow_dispatch` trigger
- Deleted unused upstream workflows: `build-artifacts.yaml`, `main-rust.yml`, `pull-request.yml`

## 3. Thread-page 403 error message (part of upstream PR #592) — `c960823`

Cherry-picked from https://github.com/redlib-org/redlib/pull/592 (by cute-johnny, fixes #591). Touches only `src/client.rs`:

- `non_json_response_error()`: 403 now reports "Reddit blocked this request" instead of a misleading JSON parse error; 5xx keeps the outage message
- The other half of PR #592 (`encode_thread_route()`, `8658bc8`) is **dropped**: upstream merged the equivalent fix as `0a3e9cd` + `57688c9` (`comments_transport_path()` / `comments_logical_path()`), which covers `/comments/...` short routes, keeps logical paths for caching, and decodes redirects. Merged with upstream's version on 2026-10-24.
