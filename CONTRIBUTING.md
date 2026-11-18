# Contributing to SimpleMapper

Thanks for helping. SimpleMapper stays small on purpose: a window and a menu bar menu. Please open an issue before larger changes.

## Prerequisites

- macOS 14+ to run the app.
- Xcode 26+ to compile it (the app icon is an Icon Composer `.icon`).
- XcodeGen:

  ```sh
  brew install xcodegen
  ```

## Setup

Generate the Xcode project:

```sh
xcodegen generate
```

`SimpleMapper.xcodeproj` is generated from `project.yml` and gitignored. Change settings, targets and packages in `project.yml`, never in the `.xcodeproj`.

## Run

```sh
scripts/dev.sh           # xcodegen generate, Debug build, launch
scripts/dev.sh --clean   # clean build first
```

Debug is signed with the same "Developer ID Application" identity as Release, so Debug (Xcode's Run) and the installed DMG build share one Input Monitoring grant. `scripts/dev.sh` and `scripts/screenshots.sh` therefore need that certificate; without it, build with your own identity:

```sh
xcodebuild -project SimpleMapper.xcodeproj -scheme SimpleMapper CODE_SIGN_STYLE=Automatic CODE_SIGN_IDENTITY="Apple Development" DEVELOPMENT_TEAM=<your team>
```

If Detect key says access is needed while SimpleMapper is already on in Input Monitoring, the grant belongs to a differently signed build: run `tccutil reset ListenEvent dinh.thi.SimpleMapper`, relaunch and allow it again.

Quit Karabiner-Elements while testing: it seizes keyboards, so `hidutil` mappings have no effect while it runs.

## Tests

```sh
xcodebuild test -project SimpleMapper.xcodeproj -scheme SimpleMapper
```

Unit tests cover the pure and injectable parts:

- `Settings`: JSON codec, validation, merge (device wins over All) and migration of the old single mapping.
- `Remapper`: the apply and clear command planner, with a fake runner instead of `hidutil`.
- `SettingsStore`: local file, iCloud copy and restore, with temp folders and a private `UserDefaults` suite.
- `KeyCatalog`.

`AppModel.start()` never runs under tests (the app skips all setup when it runs as a test host). Tests must never touch `hidutil`, the real Application Support folder, iCloud Drive or `UserDefaults.standard`.

## Screenshots and measurements

```sh
scripts/screenshots.sh   # README screenshots into assets/screenshots/
scripts/measure.sh       # RAM and CPU of SimpleMapper vs Karabiner-Elements
```

`screenshots.sh` builds Debug and runs the app in demo mode (`SIMPLEMAPPER_DEMO=1`, compiled only in Debug): fixed demo settings in a temp folder, iCloud off and no `hidutil` calls, so your settings and keyboard are untouched. A SimpleMapper that was running is quit during the capture (its mappings are cleared meanwhile) and relaunched in the background at the end. It needs Screen Recording permission for the terminal. `measure.sh [seconds]` samples the running processes with `top`, so start SimpleMapper (a Release build) and Karabiner-Elements first.

## Xcode cache

Never pass `-derivedDataPath` to local `xcodebuild` builds and tests. The default DerivedData folder is the one Xcode uses, so Xcode's Run reuses the CLI build, and the app path stays stable so the Input Monitoring grant survives rebuilds.

## Code layout

- `SimpleMapper/SimpleMapperApp.swift`: AppKit entry point (no SwiftUI `App`), the main window and About panel, applies the mappings at launch and clears them on quit.
- `SimpleMapper/MainView.swift`: the main window in SwiftUI (toolbar with the keyboard picker, Import, Export and +; the mapping list).
- `SimpleMapper/NewMappingView.swift`: the New mapping window (From, To, Detect key).
- `SimpleMapper/MenuBarView.swift`: the menu bar menu, an AppKit `StatusMenu` (no SwiftUI, for a small footprint).
- `SimpleMapper/AppModel.swift`: app state, editing rules, import/export, launch at login.
- `SimpleMapper/Settings.swift`: the settings model, JSON codec, validation and merge.
- `SimpleMapper/SettingsStore.swift`: the local settings file, the iCloud Drive copy and restore.
- `SimpleMapper/Remapper.swift`: plans and runs the `/usr/bin/hidutil` commands.
- `SimpleMapper/KeyCatalog.swift`: the list of keys, names and HID usages.
- `SimpleMapper/KeyboardMonitor.swift`: lists connected keyboards and reports reconnects.
- `SimpleMapper/KeyDetector.swift`: Detect key (one key press, Input Monitoring).
- `SimpleMapper/Updater.swift`: Sparkle updater.
- `SimpleMapper/Info.plist`: Sparkle keys, merged into the generated Info.plist.
- `SimpleMapper/logo.icon`: app icon (Icon Composer; light background, glyph layer `logo_nobg.png`).
- `assets/logo.png`: square logo for the README; the About panel uses the same artwork at 256 px.
- `SimpleMapper/Assets.xcassets`: `MenuBarIcon` (menu bar template image) and `AboutLogo` (logo shown in the About panel).
- `SimpleMapperTests/`: unit tests.
- `scripts/`: dev, screenshots, measure and release scripts.

## Adding a key

1. Add one `Key(` entry per line to `KeyCatalog.all` in `SimpleMapper/KeyCatalog.swift`, with the Karabiner-Elements name and the usage `(page << 32) | usage`. There are no default keys: the New mapping box starts empty.
2. Add the matching row to `docs/KEYS.md`. The row count must equal the number of `Key(` entries:

   ```sh
   test "$(grep -c '^| `' docs/KEYS.md)" = "$(grep -c 'Key(' SimpleMapper/KeyCatalog.swift)"
   ```

## Update feed

- **Feed:** `SUFeedURL` (since 1.1.5) is `https://simplemapper.coprimal.dev/appcast.xml`, served by GitHub Pages from `.github/workflows/pages.yml`, which publishes the root `appcast.xml` on every push to `main` that changes it. Settings > Pages > Source must stay "GitHub Actions", with the custom domain `simplemapper.coprimal.dev` and Enforce HTTPS. Cloudflare DNS (zone `coprimal.dev`) has a CNAME `simplemapper` → `coprimal.github.io`, DNS only.
- **Old address:** the repo moved from `dinhanhthi/SimpleMapper` to `Coprimal/SimpleMapper` in November 2026 (1.1.5). Builds <= 1.1.4 read `https://dinhanhthi.github.io/SimpleMapper/appcast.xml`, which GitHub does not redirect after a transfer. The user site `dinhanhthi/dinhanhthi.github.io` serves a copy at `SimpleMapper/appcast.xml`; `scripts/publish-appcast.sh` updates it on every release through `scripts/sync-feed-mirror.sh` (rerun that script by hand if it failed). Never edit the copy by hand.
- **Never:** delete the mirror repo or give its site a custom domain, create or fork a repo named `dinhanhthi/SimpleMapper` (it breaks GitHub's redirects to this repo), change the Pages source, or remove the custom domain or its DNS record.
- **Feed check:** `curl -sI https://simplemapper.coprimal.dev/appcast.xml` answers 200, `curl -s https://simplemapper.coprimal.dev/appcast.xml | cmp - appcast.xml` and `curl -sL https://dinhanhthi.github.io/SimpleMapper/appcast.xml | cmp - appcast.xml` print nothing.

## Commit style

One-line conventional commits: `<type>(<scope>): <summary>`, for example `fix(remapper): clear the previous keyboard mapping`. Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`. No body.

## Releases

Releases are maintainer-only.

## License

By contributing, you agree that your contributions are licensed under the [MIT License](LICENSE).
