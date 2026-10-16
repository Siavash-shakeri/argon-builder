# Argon Builder — API contract

Base: `https://api.github.com/repos/Siavash-shakeri/argon-builder`
Auth: `Authorization: Bearer <TOKEN>` on every call, `Accept: application/vnd.github+json`.
The token must have the **`repo`** scope (classic Personal Access Token).
Create one at GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic).

The APK artifact is kept for **7 days** — download it promptly.

---

## 1. Trigger a build

`POST /dispatches`

```bash
HTML_B64=$(base64 -w0 myapp.html)   # -w0: no line wrapping (macOS: base64 -b0 … | tr -d '\n')

curl -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_PAT" \
  https://api.github.com/repos/Siavash-shakeri/argon-builder/dispatches \
  -d "{
    \"event_type\": \"build-apk\",
    \"client_payload\": {
      \"appName\": \"ماشین حساب من\",
      \"htmlBase64\": \"$HTML_B64\",
      \"versionCode\": 1
    }
  }"
```

`client_payload` fields:

| field        | type   | required | notes                                                        |
|--------------|--------|----------|--------------------------------------------------------------|
| `appName`    | string | no       | Launcher label. Plain text, max ~60 chars. Default `"Argon App"`. |
| `htmlBase64` | string | **yes**  | Base64 of the complete HTML file (UTF-8, must contain `<html`, max 2 MB). |
| `versionCode`| number | no       | Android `versionCode` (integer). Default `1`. Bump per build to allow updating an installed build. |

A `204 No Content` response means the event was accepted (the workflow run appears a few seconds later).

## 2. Poll the run

Find the newest `repository_dispatch` run:

```bash
curl -s \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_PAT" \
  "https://api.github.com/repos/Siavash-shakeri/argon-builder/actions/runs?event=repository_dispatch&per_page=1" \
  | python3 -c "import json,sys; r=json.load(sys.stdin)['workflow_runs'][0]; print(r['id'], r['status'], r['conclusion'], r['created_at'])"
```

Then check one run directly:

```bash
RUN_ID=123456789
curl -s \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_PAT" \
  https://api.github.com/repos/Siavash-shakeri/argon-builder/actions/runs/$RUN_ID \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['status'], d['conclusion'])"
```

- `status`: `queued` → `in_progress` → `completed` (poll every ~15–30 s; a build takes ~2–5 min).
- When `status == completed`: `conclusion` should be `success`. Anything else (`failure`, `cancelled`) means the build broke — check the run's logs on github.com.

## 3. Download the APK

```bash
RUN_ID=123456789

# 1) find the artifact id
ART_ID=$(curl -s \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_PAT" \
  https://api.github.com/repos/Siavash-shakeri/argon-builder/actions/runs/$RUN_ID/artifacts \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['artifacts'][0]['id'])")

# 2) download the ZIP (auth header required; -L follows the redirect)
curl -L \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_PAT" \
  -o app-apk.zip \
  https://api.github.com/repos/Siavash-shakeri/argon-builder/actions/artifacts/$ART_ID/zip

# 3) unzip → app-debug.apk
unzip -o app-apk.zip -d ./out
ls -la ./out/app-debug.apk
```

Install `out/app-debug.apk` on the phone (allow "Install unknown apps" when asked).
Package is always `com.argonlab.webapp`; the launcher name is your `appName`.

## Notes / limits

- GitHub free Actions on public repos: effectively unlimited minutes — builds are free.
- One build at a time per trigger is fine; parallel triggers queue normally.
- `versionCode` must increase to update an already-installed build of the same page.
- The APK is **debug-signed** (auto-generated debug key) — perfect for sideloading, not for Play Store.
