# Installing Super Nonogram on Your iPhone (Free Apple ID)

This is a practical guide for building Super Nonogram from this repo
and installing it on **your own iPhone** using a **free Apple ID** — no
$99/year Apple Developer Program, no TestFlight, no App Store review.

The trade-off: builds signed this way **expire after 7 days** and have
to be reinstalled. For local personal use that's usually fine.

If you want a 1-year cert, TestFlight, push notifications, or
distribution to other people, you need the paid Apple Developer
Program — that's a different doc.

## What you'll end up with

- The current `main` (or your local branch) of Super Nonogram installed
  on your iPhone.
- Signed by your personal Apple ID, valid for 7 days.
- Reinstallable any time by plugging the phone in and re-running
  `flutter run`, which resets the 7-day clock.

## Prerequisites

- **A Mac** running macOS recent enough for current Xcode (Apple's
  signing tooling is Mac-only).
- **Xcode** installed from the Mac App Store, opened at least once so
  it can install its command-line tools and accept the license.
- **Flutter ≥ 3.41** — verify with `flutter doctor` and resolve any
  red items, especially the iOS section.
- **CocoaPods** — installed via Ruby's `gem install cocoapods` or
  Homebrew's `brew install cocoapods`.
- **A free Apple ID** — any iCloud Apple ID works. No paid program
  needed.
- **An iPhone + cable** — Lightning or USB-C depending on your phone.

## One-time iPhone setup: enable Developer Mode

iOS 16+ requires you to opt into Developer Mode before running
unsigned-by-the-App-Store apps.

1. Plug the phone into your Mac and unlock it (you can skip the plug
   for this step, but doing it now means the toggle appears
   immediately).
2. On the phone: **Settings → Privacy & Security → Developer Mode**.
   If the menu isn't there yet, try a Flutter/Xcode run first — the
   first attempted install makes iOS surface the toggle.
3. Turn **Developer Mode** on. iOS will prompt to restart the phone.
4. After reboot, confirm the warning dialog.

## One-time Xcode setup: add your Apple ID

1. Open Xcode.
2. **Xcode → Settings → Accounts** (or **Preferences** on older
   macOS).
3. Click `+` → **Apple ID**, sign in with the Apple ID you'll use to
   sign the app.
4. After signing in, you should see a team named **"Personal Team"**
   under that account. That's your free signing team.

## Project-side changes you need to make

The repo currently ships with the published-app team
(`com.adilhanney.supernonogram`, team `3DB8QX4Z23`). Free Apple IDs
**cannot reuse another team's bundle identifier**, so you need to
change the bundle ID and the team before you can sign locally.

1. From the repo root, install pods so the workspace is complete:

   ```sh
   flutter pub get
   cd ios && pod install && cd ..
   ```

2. Open the **workspace**, not the project, in Xcode:

   ```sh
   open ios/Runner.xcworkspace
   ```

3. In Xcode's left sidebar, select the blue `Runner` project at the
   top, then the **`Runner` target** in the editor pane.

4. Go to the **Signing & Capabilities** tab.

5. Make sure **Automatically manage signing** is checked.

6. **Change the Bundle Identifier** from
   `com.adilhanney.supernonogram` to something unique to you, e.g.
   `com.<yourname>.supernonogram`. Bundle IDs are namespaced globally
   by Apple — pick something nobody else will have used.

7. **Change the Team** dropdown to your **Personal Team**.

8. Xcode will provision a development signing certificate and
   profile in the background. The "Status" area should land on a
   green checkmark within a few seconds.

> Free-team limits worth knowing: you can register at most ~10 unique
> bundle IDs per 7-day window, and only 3 free-signed apps can be
> active on a single device at a time. You almost certainly won't hit
> these for personal use, but they're real.

## Build & install

1. Plug the iPhone in, unlock it, and "Trust This Computer" if
   prompted.
2. Confirm Flutter sees the device:

   ```sh
   flutter devices
   ```

   You should see an entry like `Your iPhone (mobile) • 00008XXX-XXXX…`.
3. Install and run in one shot:

   ```sh
   flutter run -d <device-id-from-the-list>
   ```

   Or, equivalently, hit the **Run** button (▶ / ⌘R) in Xcode with
   your iPhone selected as the target.

The first build takes a while (CocoaPods, code signing, archive). After
that, the app launches automatically on the phone.

## First-launch trust

The very first time you open a personal-team-signed app, iOS refuses
to run it until you explicitly trust the developer profile:

1. On the phone: **Settings → General → VPN & Device Management**.
2. Under "Developer App", tap the entry for your Apple ID.
3. Tap **Trust "…"**, confirm.
4. Reopen the app from the home screen.

This trust step is one-time per Apple ID per device.

## Known limitations of the free path

- **7-day expiry.** The app stops launching after 7 days. To extend,
  plug in and re-run `flutter run` — it reinstalls in place and resets
  the clock. No data loss.
- **Game Center / `games_services` won't sign in.** The free team
  doesn't get the Game Center entitlement, so the achievements
  feature is effectively disabled. The app still runs; you'll just
  see sign-in failures in the logs.
- **No push notifications, no associated domains, no iCloud
  containers, no TestFlight.** Those need the paid program.
- **3 active free-signed apps per device.** If you've maxed out, iOS
  will refuse the install until you delete one.

## Troubleshooting

**`No profiles for 'com.adilhanney.supernonogram' were found`**
You skipped the Bundle ID change. Go back to Signing & Capabilities
and set a unique ID you own.

**`Could not launch ... process launch failed: Security`**
You haven't done the first-launch trust step. Follow the "First-launch
trust" section above.

**`ineligible device` or `signing certificate has been revoked`**
Xcode → Settings → Accounts → select your Apple ID → **Manage
Certificates** → delete revoked entries → let Xcode re-issue.

**Pod install fails or complains about outdated specs**
```sh
cd ios
pod repo update
pod install
cd ..
```

**`flutter run` hangs at "Waiting for device" but the phone is
plugged in**
Unlock the phone, tap "Trust This Computer", then re-run. On macOS,
double-check the cable is data-capable (some charging-only cables
won't surface the device).

**`Unable to install` followed by `Maximum number of apps for free
developer account has been reached`**
Delete one of the other free-signed apps from the iPhone and retry.

## Refreshing before the 7 days are up

Just plug the iPhone in and run:

```sh
flutter run -d <device-id>
```

This rebuilds, re-signs, and reinstalls in place. Your saved progress
(everything stored via `stows` / `SharedPreferences` /
`path_provider`) survives.
