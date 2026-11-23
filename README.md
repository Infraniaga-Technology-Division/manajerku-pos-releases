# manajerku-pos-releases

Public release channel for ManajerkuPOS. This repository contains **no source code**. It exists so installed apps can learn, without any credentials, that a newer version is available, and so direct-channel binaries have a public download location.

Installed apps (Android, iOS, Windows desktop) fetch:

```
https://raw.githubusercontent.com/Infraniaga-Technology-Division/manajerku-pos-releases/main/version.json
```

anonymously (no token). The GitHub raw CDN caches for roughly 5 minutes, so a change can take a few minutes to reach apps.

## version.json schema

One entry per platform + channel. A single global "latest version" is not allowed, because each store approves builds on its own schedule.

```json
{
  "android/play":   { "latest_version": "0.4.3", "min_version": "", "update_url": "https://play.google.com/store/apps/details?id=com.manajerkupos", "release_notes": "" },
  "android/direct": { "latest_version": "0.4.3", "min_version": "", "update_url": "<GitHub Release APK URL>", "release_notes": "" },
  "ios/store":      { "latest_version": "0.4.3", "min_version": "", "update_url": "<App Store URL>", "release_notes": "" },
  "windows/store":  { "latest_version": "0.4.3", "min_version": "", "update_url": "ms-windows-store://pdp/?productid=<id>", "release_notes": "" },
  "windows/direct": { "latest_version": "0.4.3", "min_version": "", "update_url": "<GitHub Release installer URL>", "release_notes": "" }
}
```

| Field | Rule |
|---|---|
| `latest_version` | Semver `MAJOR.MINOR.PATCH` (e.g. `0.4.4`, no `v` prefix, no suffix). Installed version lower than this → the app shows a dismissible "update available" notice. |
| `min_version` | Empty string = unused. A non-empty semver means installed versions lower than this get a **blocking** update screen they cannot dismiss. Use only for critical fixes. |
| `update_url` | Where the "Perbarui" button goes: store page, APK, or installer URL. Desktop only opens `https:` and `ms-windows-store:` URLs. An empty value means the app shows nothing for that channel. |
| `release_notes` | Plain text (Indonesian) shown in the notice. May be empty. |

Channel keys:

| Runtime | Key |
|---|---|
| Android installed from Play Store | `android/play` |
| Android installed from APK | `android/direct` |
| iOS | `ios/store` |
| Windows from Microsoft Store (MSIX) | `windows/store` |
| Windows from the direct installer (NSIS, incl. current unsigned installs) | `windows/direct` |

Apps treat any problem (missing key, invalid JSON, invalid semver, empty `update_url`, network error, timeout) as "no update" and show nothing. A broken file therefore never blocks cashiers, but it also silently stops notifications — validate the JSON before merging.

## Editing rules

- **Changes land via pull request only.** No direct pushes to `main`. Protect the branch and require a review.
- Validate the file is well-formed JSON and every version is strict `MAJOR.MINOR.PATCH` before merging.
- **Update values manually, only after the store has approved and published the build.** Raising `android/play.latest_version` before Play has rolled the build out sends cashiers to a store page with no update. There is no automatic pipeline that edits this file.
- Change only the channel(s) whose build actually shipped.
- Raise `min_version` sparingly; it blocks every older install on that channel until it updates.

## GitHub Releases (binaries)

Direct-channel binaries are published as assets on this repository's GitHub Releases (never in the private source repo):

- Windows direct installer (NSIS `ManajerkuPOS-Setup.exe`, currently unsigned) plus its `latest.yml`, one release per version (see below).
- Android APK for `android/direct`.

After publishing a release, set the matching `update_url` to the asset's download URL and bump `latest_version` via PR.

### Windows direct release (`windows-v{version}`)

Installed `windows/direct` apps self-update with electron-updater's **generic** provider from exactly the version `version.json` announces:

```
https://github.com/Infraniaga-Technology-Division/manajerku-pos-releases/releases/download/windows-v{latest_version}/latest.yml
```

They never read the GitHub "latest" release (this repo also hosts APK releases) and use no token. Procedure:

1. Build the installer in the source repo: `bun run desktop:build` (unsigned, current free setup). Once a code-signing certificate is bought, use `bun run desktop:build:direct` instead (needs `WIN_CSC_LINK` + `WIN_CSC_KEY_PASSWORD`; fails without a certificate). Unsigned installs accept a later signed update; after switching to signed builds, keep releasing signed builds under the same publisher name.
2. Create a release with tag **`windows-v{version}`** (e.g. `windows-v0.4.5`; strict `MAJOR.MINOR.PATCH`, no other suffix) and upload **both** `release/ManajerkuPOS-Setup.exe` and `release/latest.yml` from that same build. Do not rename the `.exe`: `latest.yml` references it by name and its sha512.
3. Only then open the PR on `version.json`: set `windows/direct.update_url` to the installer download URL (`.../releases/download/windows-v{version}/ManajerkuPOS-Setup.exe`, the fallback for apps that cannot self-update) and `windows/direct.latest_version` to `{version}`.

Apps download in the background once the notice shows and install only when the cashier taps "Restart & Update". If the tag is missing, `latest.yml` names another version, or the sha512 check fails (or, for signed builds, the signature check), the button falls back to "Perbarui" (opens `update_url`). Unsigned installers show a SmartScreen warning only on the first manual install ("More info" → "Run anyway"); in-app updates do not trigger it. Never delete or re-upload assets of a release that `version.json` already points to.
