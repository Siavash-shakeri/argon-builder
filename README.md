# Argon Builder 🏭

Free cloud APK builder for the **Argon Chat** app's Code Lab.

Send an HTML file → get back a compiled, installable Android APK (a minimal WebView
wrapper app around your page). Built on **GitHub Actions**, which is free with
unlimited minutes on public repositories.

## How it works

1. A client (the Argon Chat app, or you via `curl`) sends a `repository_dispatch`
   event with the app name + the full HTML file (base64).
2. The [workflow](.github/workflows/build-apk.yml) injects the HTML into a tiny
   native WebView template (`template/`, package `com.argonlab.webapp`),
   compiles a debug APK and uploads it as a workflow artifact (7-day retention).
3. The client polls the run status, downloads the artifact ZIP and installs the APK.

**Client contract:** see [API.md](API.md) for the exact HTTP calls (trigger → poll → download) with copy-paste `curl` examples.

## Repo layout

```
argon-builder/
├── .github/workflows/build-apk.yml   # the builder (repository_dispatch → APK artifact)
├── template/                         # minimal Android WebView app (Java, no dependencies)
│   ├── app/src/main/java/.../MainActivity.java
│   ├── app/src/main/assets/www/index.html   # ← replaced per build
│   ├── gradle/wrapper/               # pinned Gradle 8.11.1 (properties only;
│   │                                 # the binary jar is NOT committed — CI downloads
│   │                                 # the pinned distribution itself; run `gradle wrapper`
│   │                                 # locally if you want ./gradlew to work on a clone)
├── API.md                            # client contract
└── README.md
```

## Notes

- Debug-signed APKs install fine via sideloading ("Install unknown apps").
- The app name shown on the launcher is set per build (`appName`); the package
  stays `com.argonlab.webapp`, so builds of *different* pages overwrite each other —
  bump `versionCode` per build if you need side-by-side installs (or ask for
  per-build applicationId suffixes in a future version).
- Nothing secret lives in this repo. HTML payloads are the user's own content.
