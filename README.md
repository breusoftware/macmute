# MacMute

A lightweight menu bar utility that mutes your Mac's current default input device at the CoreAudio hardware level. The mute applies to apps using that default device, not just one app's own mute button. An app that explicitly selects a different input device is outside MacMute's control.

## Features

- Menu bar icon (🎙 / 🔇) reflecting mute state
- Explicit unavailable-state icon when CoreAudio cannot verify the microphone state
- Global hotkey (default: ⌥⌘M, or bind the standalone `fn` key), configurable in Preferences:
  - **Tap (shorter than the selected delay)**: toggles mute/unmute permanently on release
  - **Hold (at least the selected delay)**: temporarily flips mute/unmute, then restores the prior state on release
  - Each tap acts independently, including two quick taps
- Main menu shows microphone state, hotkey status, tap/hold help, and a saved delay slider from 0.2 to 1.0 seconds in 0.2-second increments (default 0.4)
- Launch at Login (in Preferences)
- Tracks the default input device — if you switch microphones while muted, the new device is muted too
- Preserves the microphone's existing state when MacMute launches

## How it works

All user mute changes pass through `MicrophoneActionController`: keyboard edges and the hold timer drive one gesture state machine; the menu displays state and gesture help. One write boundary submits the desired state to `MicMuteController.setMuted`. Hardware writes, device transfers, and pending restoration retries remain owned by `MicMuteController` and its single `applyMute` implementation. No hotkey modes or double-click actions exist.

MacMute never records or transmits audio. It only flips the input device's writable `Mute` property (or, on devices without one, zeroes the writable master input volume and restores the verified per-device baseline) through CoreAudio. Every requested state change is read back before the menu bar reports success.

## Xcode

Open `MacMute.xcodeproj` and select the shared **MacMute** scheme. It includes the native macOS app and `MacMuteAppTests` targets, uses the existing sources and Info.plist, and generates the app icon from `Resources/RaptorIcon.png`.

Run with **Product → Run**, test with **Product → Test**, or archive the Release configuration with **Product → Archive**. Signing uses Breu Software LLC's team (`6Q66ZYGK4J`) with automatic signing and hardened runtime. For direct distribution, choose **Developer ID** in Organizer. The project is configured for direct distribution; App Store distribution would require a separate sandbox/capabilities review.

The shell release pipeline below remains available for a tested, signed, notarized DMG.

## Build (no Xcode required)

Requires Xcode Command Line Tools (`xcode-select --install`) and macOS 13+.

```sh
./Scripts/build_app.sh
```

This builds a universal Apple Silicon and Intel release binary, assembles `MacMute.app`, ad-hoc signs it for local development, and verifies the resulting bundle. It does not modify the microphone merely by building or launching the app.

Run the regression suite with:

```sh
swift test
```

## Install

```sh
./Scripts/build_dmg.sh
```

Builds the app and packages it into `MacMute-<version>.dmg` — a disk image containing `MacMute.app` and an `Applications` shortcut, so you open it and drag the app in like any other Mac app. Installing to `/Applications` this way (rather than running the app from wherever it was built) matters for Launch at Login: macOS's `SMAppService` registers login items by the app's installed location, so it works most reliably once the app lives somewhere stable like `/Applications`.

## Run

```sh
open MacMute.app
```

The app has no Dock icon or main window — look for the mic icon in the menu bar.

- **Click** the menu bar icon: opens the dropdown (microphone state, hotkey status, tap/hold help, Preferences…, About, Quit)
- **Preferences…**: change the global hotkey or enable Launch at Login
- **About**: version and credits

## BreuSoftware LLC Release Signing

Local builds intentionally use ad-hoc signing. Public releases must use BreuSoftware LLC's Apple-issued **Developer ID Application** certificate plus Apple's notarization service. The release scripts refuse to create a release DMG with an ad-hoc, self-signed, or differently named identity.

First save App Store Connect notarization credentials in Keychain using `xcrun notarytool store-credentials`. Then supply the exact Keychain identity and saved profile:

```sh
export MACMUTE_RELEASE=1
export MACMUTE_SIGNING_IDENTITY='Developer ID Application: Breu Software LLC (6Q66ZYGK4J)'
export MACMUTE_NOTARY_PROFILE='breusoftware-notary'
./Scripts/build_dmg.sh
```

The repository pins BreuSoftware LLC's Apple Team ID in `Resources/BreuSoftwareTeamID.txt`. Replace the example profile name above if your saved Keychain profile uses a different name. Release mode requires pristine tracked and untracked release inputs, runs the strict test suite with aggregate and critical-file coverage floors, builds and verifies a universal Apple Silicon/Intel executable, embeds the source revision, applies the hardened runtime and secure timestamp, verifies the signed app against the pinned Team ID and Apple trust assessment, mounts and inspects the DMG before and after notarization, and only then publishes the app and DMG together. `MACMUTE_TEAM_ID` is optional; if supplied by CI it must match the pinned value. Never create or globally trust a self-signed certificate for a public release.

## Notes

- On first launch, macOS may prompt for microphone-related permission when CoreAudio enumerates input devices. MacMute never opens an audio input stream.
- To change the hotkey: open Preferences, click the shortcut button, then press your desired key combination (must include a modifier, or press `fn` alone).
- Binding `fn` requires granting MacMute Accessibility permission (System Settings → Privacy & Security → Accessibility) — macOS will prompt for this the first time.
- If your menu bar already has other mic/audio-related icons (system input indicator, call-app mute buttons, etc.), MacMute's plain mic icon can be easy to miss at a glance — check for it near other third-party menu bar icons, not just the system clock cluster.
