# This TV (JVCO "SMART_TV", CVTE/HiSilicon TP.SK529.PB802)

## Device
- HiSilicon Hi3751V352, Android 14 (SDK 34), armeabi-v7a, ~1 GB RAM, 1080p @ 240dpi
- SELinux permissive, userdebug (dev-keys)
- ADB: port 5555, IP may move (currently 192.168.0.102), persists across reboot
- ADB enable: factory menu (hold Menu, Left Up Left Up, then Back) -> ADB ON
- `adb root` required after every reboot before file operations
- Remounts (as root): /system: `mount -o remount,rw /` (system-as-root, mmcblk0p21);
  /vendor: `mount -o remount,rw /vendor`

## Launcher state
- `com.gathin.tvhome` v2.13 (versionCode 16) on the latest upstream launcher base (3.3-era), HOME role holder. TV boots straight into it.
- Source `..\launcher\`; JDK 21 (`C:\Program Files\Eclipse Adoptium\jdk-21.0.11.10-hotspot`), SDK `C:\Android\sdk`.
- Two lanes, must be byte-identical (md5 match):
  - Boot lane: `/vendor/priv-app/tvhome/tvhome.apk` (root:root, 644, `u:object_r:vendor_app_file:s0`).
    Boot daemon launches this directly, bypassing PM — disabling/hiding the package for user 0 does not stop boot.
  - PM lane: user 0 record points at `/data/app/~~<hash>/.../base.apk` — this is what runs.
- `com.cvte.tv.launcher.mars`: fallback HOME, firmware boot dependency (`persist.cbd.launcher`), has an
  updated `/data/app` copy (UPDATED_SYSTEM_APP). NEVER uninstall — boot hangs on logo.

## PM quirks (constraints)
- `pm uninstall --user 0` on the launcher: hides package only — wipes data, keeps the `/data/app` dir.
- `pm install-existing` re-binds to that `/data` record, never to vendor. The `/data` copy is undeletable; keep both lanes identical.
- REBOOT permission denied; launcher reboots via `Runtime.exec("setprop","sys.powerctl","reboot")`.
- DELETE_PACKAGES denied, refuses `pm grant`. REQUEST_DELETE_PACKAGES is a normal permission on this firmware -> declared in manifest, auto-granted; powers the tile Uninstall dialog.
- privapp whitelist files in /vendor/etc/permissions are ignored.
- Home-while-on-home resolves CATEGORY_TV_HOME; launcher claims it (priority 1) over `com.cvte.tv.setting/.TvSettingActivity` (priority 0). Home key never opens TvSettingActivity — use the TV settings menu entry. Applies to adb-injected KEYCODE_HOME too.
- If resolution becomes ambiguous (e.g. system data cleared): resolver pops once on Home; pick launcher + "Always" (persisted in `/data/system/users/0/package-restrictions.xml`).

## Launcher data
- Grid order: `/data/user_de/0/com.gathin.tvhome/files/order.txt` (uid:gid 10086:10086, 600):
  `tvhome-order 1` header, `dock <comma-joined dock_apps>` line, then one flat component key per row
  (dock first, then the rest in display order). `docked <n>` is omitted: dock_apps decides the dock.
- Missing/unparseable order.txt -> grid resets to config defaults (dock_apps, then alphabetical).
- Rescue: restore copy, chown 10086:10086, `restorecon -RF /data/user_de/0/com.gathin.tvhome`.
- App config: `launcher/app/src/main/res/values/config.xml` (dock_apps, hidden_apps, columns, ...
  and `app_library` on), overlaid by theme's config.json.

## Launcher UX (v2.13 on upstream 3.3-era base)
- Grid shows all installed launchables, alphabetical; order.txt pins the fitted ones first,
  dock row on top. New installs land in the grid alphabetically.
- Tile menu (long-press / hold OK, `tile_menu`): Move (D-pad reorder), Remove (library),
  App info, Uninstall (system delete dialog, `REQUEST_DELETE_PACKAGES`).
- App library on (`app_library`): pill after the last Home row — add back, restore hidden.
- Settings pill: Reboot and Power off rows (confirm dialogs; both `Runtime.exec setprop sys.powerctl`,
  the only reboot route — REBOOT permission is denied). Upstream's "Restart…" power-dialog row stays
  inert: it needs the HomeKeyService accessibility service, which is not enabled here.
- No other accessibility services. `enabled_accessibility_services` is HiRM-only — keep it.

## Device tweaks / package state (user 0)
- Animation scales all 0.5: `window_animation_scale`, `transition_animation_scale`, `animator_duration_scale`. Revert: `settings put global <name> 1`.
- Disabled (restore any: `pm enable <pkg>`):
  adservices.api (x2), ondevicepersonalization, federatedcompute, feedback, partnersetup,
  apps.restore, gms.location.history, dynsystem, virtualmachine.res,
  cvte.factory.at, cvte.factory.service, cvte.tv.steward,
  cvte.tv.tools (factory test harness), cvte.tv.provision (FTI wizard; factory reset re-enables),
  cvte.tv.bluetoothguide (remote-locator beacon + CVTE silent-auth wakeup),
  com.stark.bytelloremoteserver (APK stays at /system/app/LiteTvBytelloRemote; listed in
  /system/etc/central_control/performance_config.xml; disabling is safe),
  com.google.android.onetimeinitializer.

## App inventory (user 0)
- Restorable (hidden system package): `com.cvte.tv.pictorial` -> APK `/vendor/app/LiteTvPictorial` (~16 MB);
  `cmd package install-existing --user 0 com.cvte.tv.pictorial`.
- Gone, no pm record and no APK (recoverable only by flashing): twitch, facebook, tcl.browser, gov.nasa,
  gaia.ad.service, smarthome, stark.store.
- Gone, no pm record, APK dirs deleted 2026-10-26 (~41 MB freed): com.android.traceur,
  com.cvte.tv.systemupgrade, com.cvte.tv.weather, com.cvte.industries, com.google.android.katniss.
- Kept enabled (load-bearing): CVTE settings + factory menu (com.cvte.fac.menu hosts the ADB unlock),
  tvservice/com.cvte.tv.media (HDMI/TV input), HiSilicon services incl. HiRM (accessibility note),
  com.cvte.tv.ucservice (UCService2, CVTE auth/entitlement — load-bearing),
  Metax Lite Browser `com.metax.browser.cvte` (only browser; on the launcher grid), Google core incl.
  Play Store, mars launcher + mars.skin + RRO overlays.

## Revert launcher -> stock (nothing else touched)
```
adb connect <tv-ip>:5555
adb root
adb shell "cmd package set-home-activity --user 0 com.cvte.tv.launcher.mars/.ui.home.HomeActivity"
```

## Remove launcher completely
```
adb shell "pm uninstall --user 0 com.gathin.tvhome"     # hides it, wipes app data
adb shell "mount -o remount,rw /vendor && rm -rf /vendor/priv-app/tvhome && mount -o remount,ro /vendor"
adb shell "setprop sys.powerctl reboot"
adb shell "cmd package set-home-activity --user 0 com.cvte.tv.launcher.mars/.ui.home.HomeActivity"
```
(Final set-home line required: uninstalling the launcher clears the HOME preference.)

## Build / redeploy launcher
```
cd C:\Users\obaid\Downloads\cleanyourtv\launcher
set JAVA_HOME=C:\Program Files\Eclipse Adoptium\jdk-21.0.11.10-hotspot
set ANDROID_HOME=C:\Android\sdk
gradlew.bat assembleRelease

adb connect <tv-ip>:5555
adb root
adb push app\build\outputs\apk\release\app-release.apk /data/local/tmp/tvhome.apk
adb shell "pm install -r /data/local/tmp/tvhome.apk && rm /data/local/tmp/tvhome.apk"
adb shell "cmd package set-home-activity --user 0 com.gathin.tvhome/.MainActivity"
adb shell "pm grant com.gathin.tvhome android.permission.ACCESS_FINE_LOCATION 2>/dev/null"
adb shell "mount -o remount,rw /vendor && cp /data/app/~~*/com.gathin.tvhome-*/base.apk /vendor/priv-app/tvhome/tvhome.apk && chmod 644 /vendor/priv-app/tvhome/tvhome.apk && chown root:root /vendor/priv-app/tvhome/tvhome.apk && restorecon /vendor/priv-app/tvhome/tvhome.apk && mount -o remount,ro /vendor"
adb shell "setprop sys.powerctl reboot"
```
Both lanes must carry the same APK (md5 equal): boot daemon reads the vendor file, pm runs the /data copy.
