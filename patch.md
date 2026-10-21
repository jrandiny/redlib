# Fork status (vs upstream `redlib-org/redlib`)

Base: upstream `main` at `a4d36e9` (2026-04-24, incl. wreq switch, proxy support, RSS images).

Three local changes are kept on top:

## 1. CMAF video support — `a3abe21`

Original commit `174cd14` by Mateusz Jabłoński, re-applied onto the new base.

- `src/main.rs`: split `/vid/:id/:size` into `/vid/:id/dash/:size` → `DASH_{size}` and `/vid/:id/cmaf/:size` → `CMAF_{size}`
- `src/utils.rs`: `REGEX_URL_VIDEOS` matches `(DASH|CMAF)_…`, `format_url` emits `/vid/{id}/{dash|cmaf}/{size}.mp4`, tests updated

Reddit serves both DASH and CMAF renditions; without this, CMAF video links don't proxify.

## 2. Private GHCR docker workflow — `4dabe3a`

- `.github/workflows/main-docker.yml`: image `ghcr.io/jrandiny/redlib`, login via `GITHUB_TOKEN`, amd64-only matrix, `workflow_dispatch` trigger
- Deleted unused upstream workflows: `build-artifacts.yaml`, `main-rust.yml`, `pull-request.yml`

## 3. Thread-page client fix (upstream PR #592) — `8658bc8`, `c960823`

Cherry-picked from https://github.com/redlib-org/redlib/pull/592 (by cute-johnny, fixes #591). Touches only `src/client.rs`:

- `encode_thread_route()`: percent-encodes the `/r/<sub>/comments/...` slash (`%2f`) so Reddit's edge serves JSON instead of the "blocked by network security" HTML page for app-only tokens
- `non_json_response_error()`: 403 now reports "Reddit blocked this request" instead of a misleading JSON parse error; 5xx keeps the outage message
