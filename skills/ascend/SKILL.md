---
name: ascend
description: Deliver a Flutter build to Google Play Console and App Store Connect. Reads latest store versions, bumps pubspec.yaml and project.pbxproj (MARKETING_VERSION, CURRENT_PROJECT_VERSION), builds, uploads, with Shorebird release/patch support. Use when user types /ascend or asks to ship/release/upload a build to the stores.
trigger: /ascend
---

# /ascend

Ship a Flutter build to Google Play and App Store (TestFlight). Version numbers come from the stores, never guessed.

Usage: `/ascend [ios|android|both] [release|patch] [--bump patch|minor|major|build] [--flutter <version>] [--dry-run]`

- `release` (default): bump version, build, upload to store(s). With Shorebird present, uses `shorebird release`.
- `patch`: Shorebird over-the-air patch. No bump, no store upload.
- `--dry-run`: stop after Step 4 preview. Write nothing, build nothing.
- Missing platform or mode: ask once.

## Step 1 - Preflight

Stop and report on first failure.

- `pubspec.yaml` exists at repo root, `version:` is `X.Y.Z+N`.
- Flutter version (see below) resolved. On PATH: `flutter` (or `fvm`). Also `fastlane` (store query + upload), `shorebird` (only if used or `shorebird.yaml` exists).
- Working tree clean, except `pubspec.yaml` and `ios/Runner.xcodeproj/project.pbxproj`.
- On default branch (main/master): ask whether release-from-main is intended. Never assume.
- Gather inputs (below). Read from `fastlane/Appfile`, `fastlane/Fastfile`, env, `shorebird.yaml` first. Ask only for what is still missing.

### Flutter version

Resolve in this order:

1. `--flutter <version>` given: use it.
2. `.fvmrc` (or legacy `.fvm/fvm_config.json`) at repo root: use its version, run Flutter as `fvm flutter ...`.
3. Neither: plain `flutter`, whatever is on PATH.

Rules:
- Show the resolved version in the Step 3 table. User confirms or overrides.
- `fvm` mode: installed SDK for that version missing: stop, ask before `fvm install`.
- `--flutter` differs from `.fvmrc`: warn, ask which wins. Do not edit `.fvmrc`.
- Shorebird `release`: always pass `--flutter-version=<resolved>`. Without it Shorebird uses its own default, not `.fvmrc`.
- Shorebird `patch`: must match the version of the release being patched. Check with `shorebird releases list`. Mismatch: stop, ask.
- Shorebird manages its own Flutter checkout. The `--flutter-version` flag is the only link to `.fvmrc`. Do not export FVM paths into Shorebird.

### Inputs

| Needed | For |
|--------|-----|
| Android package name, iOS bundle ID | Store queries |
| Play service-account JSON path, track (internal/alpha/beta/production) | Play query + upload |
| ASC API key: key id, issuer id, `.p8` path, team id | Apple query + upload |
| iOS destination: TestFlight only, or submit for review | Upload |
| Flavor, `--dart-define` / `--dart-define-from-file`, ExportOptions.plist | Build |
| Shorebird: logged in (`shorebird login` or `SHOREBIRD_TOKEN`), flavors | `release` / `patch` |
| Default bump level | Step 3 |
| Xcode extension targets share app version? | Step 4 |

No fastlane: offer to set it up, or fall back to `flutter build` + `xcrun altool` / Gradle Play Publisher. Ask, do not choose.

Credentials: never print, never write into the repo.

## Step 2 - Read store state

Read-only. Only for selected platforms.

| Store | fastlane action |
|-------|-----------------|
| Play | `google_play_track_version_codes`, `google_play_track_release_names` for the target track (also check production) |
| Apple | `latest_testflight_build_number`, `app_store_build_number` (plus live and in-review version strings) |

Query fails: stop, ask. Never fall back to pubspec alone.

## Step 3 - Compute next version

Skip in `patch` mode: use pubspec version as is.

- **Build N** = max(Play versionCode, Apple build, pubspec build) + 1. One shared number for both platforms, even if only one is selected: keeps stores aligned. Never lower.
- **Marketing X.Y.Z**:
  - Base = highest of Play name, Apple version, pubspec.
  - Base already live in a store (released): apply `--bump` (default per user). `build` bump keeps X.Y.Z.
  - Base only in review / TestFlight / Play testing track: keep X.Y.Z, bump build only. Ask if user wanted a new version instead.

Show table: pubspec now / Play / Apple / proposed, plus resolved Flutter version and its source (`--flutter`, `.fvmrc`, PATH). User confirms or edits.

Example: Play 41, Apple 43, pubspec `1.2.0+40`, 1.2.0 live, `--bump patch` gives `1.2.1+44`.

## Step 4 - Write versions

Preview diff first. `--dry-run` ends here.

1. `pubspec.yaml`: `version: X.Y.Z+N`.
2. `ios/Runner.xcodeproj/project.pbxproj`: set every `MARKETING_VERSION = X.Y.Z;` and `CURRENT_PROJECT_VERSION = N;` in Runner build configs (Debug, Release, Profile).
   - Value is `$(FLUTTER_BUILD_NAME)` / `$(FLUTTER_BUILD_NUMBER)`: leave, report. Flutter injects from pubspec.
   - Extension targets (widget, notification service): list them, apply same values if user said they share version.
   - Count matches before and replacements after. Mismatch: revert, abort.
3. Android: nothing, Flutter feeds `versionCode`/`versionName` from pubspec. Hardcoded in `build.gradle(.kts)`: show and ask.
4. Check: `git diff --stat` shows only these files.

## Step 5 - Build and upload

Show the exact commands, then ask go. Upload is outward-facing: confirm every time.

### release

| Shorebird | Android | iOS |
|-----------|---------|-----|
| No | `flutter build appbundle` | `flutter build ipa` |
| Yes | `shorebird release android --flutter-version=<v>` | `shorebird release ios --flutter-version=<v>` |

`flutter` means `fvm flutter` in fvm mode. Add flavor, dart-defines, export options. Then upload with fastlane: `upload_to_play_store` (aab, track), `upload_to_testflight` (ipa; `deliver` if submitting for review).

Run platforms sequentially. One fails: stop, report exact state (what uploaded, what not). Do not roll back the bump.

### patch (Shorebird only)

- No bump, no store upload.
- `release-version` = pubspec `X.Y.Z+N`. Confirm against `shorebird releases list` first. Not found: stop, ask.
- `shorebird patch android --release-version=<v>` / `shorebird patch ios --release-version=<v>`. Flutter version must match the release (see Flutter version).
- Never invoke `flutter build` here.

## Step 6 - Report

Per platform: version shipped, artifact path, store/track, Shorebird release or patch number. Remind: version bump uncommitted, `/carve` or commit when ready.

## Rules

- Never commit or push. User says when.
- Never lower or reuse a build number.
- Store query or auth fails: stop, ask. No guessing.
- Confirm before any upload or Shorebird publish.
- Never print or store credentials.
