# ascend - design

## Intent
Releasing a Flutter app by hand means bumping `pubspec.yaml` and `project.pbxproj`, building, and uploading to Google Play Console and App Store Connect, optionally through Shorebird. Build numbers must exceed what both stores already hold, so manual bumps go wrong. `/ascend` reads store state, computes the next version, edits version files, builds, and uploads, with user confirmation before anything leaves the machine.

## Goals
- Next version derived from latest Play + Apple builds, not from pubspec alone.
- One shared build number across both platforms.
- Edit `pubspec.yaml` and `project.pbxproj` (`MARKETING_VERSION`, `CURRENT_PROJECT_VERSION`) only.
- Shorebird `release` (bump + store upload) and `patch` (no bump, no store).
- Dry-run preview.

## Non-goals
- Committing or pushing (user or `/carve` does that).
- Store metadata, screenshots, review submission copy.
- Android signing / iOS provisioning setup.
- Non-Flutter apps.

## Anti-goals
- Lowering or reusing a build number.
- Uploading without explicit confirmation.
- Guessing version when store query fails.
- Leaking credentials.

## Decisions (all author calls; user opted out of brainstorming questions)
- Name `ascend`, chosen by user from options.
- Single markdown skill, no scripts. Shells out to `flutter`, `fastlane`, `shorebird`.
- fastlane for store query and upload; fallback offered, not chosen silently.
- Build N = max(Play, Apple, pubspec) + 1.
- Version already live: apply bump level. Version only in review/testing: keep, bump build.
- pbxproj: update Runner configs; `$(FLUTTER_BUILD_*)` values left alone; extension targets only if user says they share version; replacement count verified.
- Flutter version: `--flutter` flag, else `.fvmrc` (run via `fvm flutter`), else PATH. Shown and confirmed in the version table. Shorebird `release` gets `--flutter-version=<resolved>`; `patch` must match its release. Missing FVM SDK: ask before `fvm install`. `.fvmrc` never edited.
- Patch mode never builds via `flutter build` and never uploads to stores.
- Inputs read from fastlane/shorebird config and env first; ask only for gaps.

## Open for builder
- Exact fastlane lane/action wiring when user has no Fastfile.
- Flavor handling per Shorebird command.

## Verification
- Trace sample: Play 41, Apple 43, pubspec `1.2.0+40`, 1.2.0 live, `--bump patch` gives `1.2.1+44`.
- Scratch Flutter project: `--dry-run` writes nothing; real run changes only the two version files.
- `patch` mode: no `flutter build` call.
