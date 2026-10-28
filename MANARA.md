# Manara patch layer

This repository is Manara Home: the [Home Assistant Android app](https://github.com/home-assistant/android) (Apache 2.0) with a thin Manara patch layer on top. The decision is recorded in [ManaraEngineering/manara-platform ADR 0003](https://github.com/ManaraEngineering/manara-platform/blob/develop/docs/adr/0003-client-app-fork.md).

Upstream's `README.md`, `AGENTS.md`, `CLAUDE.md` and `GEMINI.md` describe upstream. **This file overrides them for anything Manara-specific.**

## Branches and tags

| Ref | Contents | Rules |
|---|---|---|
| `upstream` | Pristine mirror of `home-assistant/android` `main` | Never commit to it. Only fast-forward from upstream. |
| upstream release tags (e.g. `2026.10.1`) | Upstream releases we build on | Pushed as-is; never moved. |
| `develop` (default) | Manara patch layer on the current base release tag | All Manara work lands here by PR. |
| `main` | Stable Manara releases | Updated from `develop` by PR. |

Current base: **`2026.10.1`**.

## Rebasing onto a new upstream release

1. Fetch the new release tag from upstream and push it to `origin`.
2. Rebase `develop` onto the new tag. Patches should apply cleanly; a conflict usually means a patch has grown outside the allowed surface below.
3. Disable any **new** upstream workflows in this repo (Settings → Actions, or `gh workflow disable`). Upstream workflows must never run here.
4. Build both flavours, update "Current base" above, and add a `CHANGES.md` entry.

`.github/workflows/rebase-check.yml` will automate detecting new tags (it is a stub for now).

## Allowed patch surface

Everything Manara changes must fit in one of these. Anything else needs a documented hook first, agreed in review.

1. **Gradle flavour block**: `mobile` (`in.co.manara.home`) and `tv` (`in.co.manara.home.tv`) flavours, built on upstream's `minimal` flavour. TODO(verify): upstream defines flavours in `build-logic/convention/.../AndroidFullMinimalFlavorConventionPlugin.kt` and the base application ID in `AndroidApplicationConventionPlugin.kt`. Decide whether to add a second flavour dimension in `app/build.gradle.kts` or a small Manara convention plugin, keeping upstream files untouched.
2. **Resource overrides**: branding, strings, colours and icons in flavour source sets (`app/src/mobile/res`, `app/src/tv/res`). Never edit upstream `res/` files.
3. **One config class**: a single Manara config class holding Manara constants (domain, deep-link scheme `manara://`, enrolment endpoints).
4. **TV manifest and activity**: in the `tv` source set. Note that upstream's `main` manifest already declares `android.software.leanback` (not required) and a `LEANBACK_LAUNCHER` entry, which may reduce what's needed.
5. **Small documented hooks**: minimal call sites into upstream code, each listed below with its reason.

### Hooks

| File | Change | Reason |
|---|---|---|
| — | none yet | — |

## Licence obligations (Apache 2.0)

- Keep upstream's `LICENSE.md` unchanged. Upstream has no `NOTICE` file. If it adds one, keep it, and add Manara attribution only in a separate section.
- Record every change to upstream files in [CHANGES.md](CHANGES.md).
- Modified upstream files keep their existing headers.

## Secrets

Never commit signing keys, keystores, `google-services.json`, API keys or tokens. Release signing happens only on the self-hosted runner on the Manara hub ([ADR 0010](https://github.com/ManaraEngineering/manara-platform/blob/develop/docs/adr/0010-free-tier-mvp.md)). Gitleaks runs as a pre-commit hook.

## Build

```sh
# One-time: upstream's mock Firebase config (gitignored; the minimal flavour still needs it to build)
for m in app wear automotive; do cp .github/mock-google-services.json $m/google-services.json; done

./gradlew :app:assembleMinimalDebug   # upstream minimal flavour (baseline)
```

Needs JDK 21 (Android Studio's bundled JBR works) and the Android SDK. A first build takes about 15 minutes; output goes to `app/build/outputs/apk/minimal/debug/`.

If the Gradle wrapper times out downloading the distribution on a slow link, download the zip from `distributionUrl` in `gradle/wrapper/gradle-wrapper.properties` into the wrapper's `dists/` folder and check it against `distributionSha256Sum`. Don't change upstream's wrapper settings.
