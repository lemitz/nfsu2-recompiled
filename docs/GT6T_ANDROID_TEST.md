# Realme GT 6T Android test (2026-10-24)

This fork contains Android launcher/runtime changes only. It does **not** include
`SPEED2.EXE`, generated game code, game data, tester reports or an APK. Build
from your own game copy using the upstream README and `ports/dxvk_android/README.md`.
Do not publish an APK made from locally lifted game code as an asset of this fork.

## Observations

On a realme GT 6T (RMX3853, Snapdragon 7+ Gen 3, Android 16), the tester reported
that the upstream v0.14.10 arm64 APK ran normally in menus and while stationary,
but fell to roughly 2 FPS on driving. Its diagnostic ZIP contained only the
last ~36 seconds of logcat after closing the game, not guest D3D9 frame samples;
there is no measured upstream slow-frame breakdown.

A separately installed debug build of this fork was compiled locally from the
tester's own `SPEED2.EXE`. Its hash differed from the upstream documented binary;
the PE layout and checked widescreen hook addresses matched, and the lifter
found 27,742 functions with no lift failures. This is not a general compatibility
guarantee for other executable builds.

With system Vulkan, 50% internal resolution, two back buffers and a 120 FPS cap,
the tester reported that driving was playable. The private `frame-telemetry.txt`
recorded ~74 seconds of intervals with at least 15,000 D3D9 calls/frame:
49.5–114.4 presentation FPS (median ~79.9), median bridge 3.75 ms/frame and
median Present 0.22 ms/frame. The cap, local lifted executable, package identity,
game category and native log volume differed from the installed upstream APK.
**The cause of the original slowdown has not been isolated.** In particular,
these numbers do not prove that changing `android:appCategory` or reducing logging
alone fixes the issue.

## Changes

- Debug builds use a separate `.gt6t` application ID and label so they cannot
  replace the upstream-signed installation or its private settings.
- The manifest identifies the app as a game; this may affect OEM game handling,
  but no clock or policy measurements were taken.
- Surface-release INFO logs are sampled rather than emitted for every call;
  COM reference behavior is unchanged.
- Successful Present intervals report after 120 frames **or five seconds**.
  The last measurements are also written to a bounded app-private file and
  attached to launcher diagnostic ZIPs, so closing the game does not erase them
  when Android's logcat ring fills up.
- Audio-health log tags use `NFSU2`, allowing the launcher's log filter to
  capture them.
- The Android launcher, touch controls, dialogs, diagnostics and native startup
  messages use English. The in-game language is selected separately from the
  language files supplied with the user's game; no game text was translated.

Performance should be compared under the same scene, render resolution, FPS cap,
driver, game data and device conditions before attributing a gain to any one
change. The saved FPS measures successful D3D9 Presents, not unique rendered
frames or a full-race average.
