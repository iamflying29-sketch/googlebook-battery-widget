# Googlebook Battery Widget Fix

This project makes Google's built-in Batteries widget on a Lenovo Googlebook 15 show the battery level of
connected Bluetooth earbuds next to the laptop's own battery. The fix is a one-time permission change made over
adb from the Linux Terminal. The small command in this repository, `fix-battery-widget`, applies the change,
makes it stick, and prints the result, so you can re-apply it in seconds if the widget ever goes blank again.

## What it does

I wanted to see my earbuds' battery levels in the Batteries widget. On my Googlebook the widget only ever showed
the laptop. This fix makes it show connected Bluetooth devices, such as my
Galaxy Buds4 and Pixel Buds 2a, next to the laptop's own battery.

It is a one-time permission change, not an app. Nothing is installed on the Googlebook. The
`fix-battery-widget` command in the Linux Terminal repeats the whole change and shows whether it worked, which
is useful after a factory reset or if the widget ever goes blank again.

## The problem

The earbuds showed up in Bluetooth Settings, but the Batteries widget only ever showed the laptop. The earbuds
and the Bluetooth system were working normally. The Bluetooth service itself had a working battery connection to
the Galaxy Buds4, and the widget was even being told every time a Bluetooth device connected or disconnected. It
simply could not read the devices.

The widget belongs to a preinstalled Google app called Settings Services
(`com.google.android.settings.intelligence`). The widget provider is
`com.google.android.settings.intelligence.modules.batterywidget.impl.BatteryAppWidgetProvider`.

Since Android 12, an app that wants to list connected Bluetooth devices and read their battery levels needs the
runtime permission `android.permission.BLUETOOTH_CONNECT`, which Android shows as "Nearby devices". Settings
Services asks for that permission in its manifest, but on my Googlebook it had never been granted for my Android
user. The package manager showed it plainly:

```
android.permission.BLUETOOTH_CONNECT: granted=false, flags=[ USER_SENSITIVE_WHEN_GRANTED]
```

Without that permission the widget cannot see any Bluetooth devices, so it shows only the laptop. Bluetooth
Settings still showed the earbuds because it runs with its own system privileges. I do not know why GooglebookOS
shipped without this permission granted. I only know that it was missing on my device and that granting it fixed
the widget.

## How I solved it

1. **I granted the missing permission.** From the Linux Terminal, over adb, I granted `BLUETOOTH_CONNECT` to
   Settings Services for my Android user (user 10 on my Googlebook):

   ```
   adb shell pm grant --user 10 com.google.android.settings.intelligence android.permission.BLUETOOTH_CONNECT
   ```

2. **I made the widget refresh.** The widget refreshes when a Bluetooth device connects, and it only lists
   devices that are connected at that moment. Forcing a refresh from adb does not work: both the standard widget
   update broadcast and the widget's own update broadcast are rejected with a `SecurityException`. So I simply
   reconnected the earbuds. The Galaxy Buds4 and the Pixel Buds 2a both appeared in the widget right away.

3. **I made the fix stick.** I marked the permission as my own choice by setting the `USER_SET` and `USER_FIXED`
   permission flags, so the system treats it as a deliberate setting:

   ```
   adb shell pm set-permission-flags --user 10 com.google.android.settings.intelligence android.permission.BLUETOOTH_CONNECT user-set user-fixed
   ```

   I also turned off Android's automatic removal of permissions from unused apps for Settings Services:

   ```
   adb shell appops set --user 10 com.google.android.settings.intelligence AUTO_REVOKE_PERMISSIONS_IF_UNUSED ignore
   ```

4. **I wrote a repair command.** `fix-battery-widget` repeats all three steps and then prints the permission
   line from the package manager, so you can see at once that it worked. After all steps, the line reads:

   ```
   android.permission.BLUETOOTH_CONNECT: granted=true, flags=[ USER_SET|USER_FIXED|USER_SENSITIVE_WHEN_GRANTED]
   ```

## Requirements

- A Lenovo Googlebook 15. I tested on GooglebookOS build `CL3B.260622.271.R1` (Android 17). Settings Services
  was version `1.1.0.937862210.sr` when I applied the fix on October 8, 2026.
- The Linux Terminal with `adb` installed, and adb paired and connected to the Googlebook over Wireless
  debugging. The shared [Getting Started guide](https://github.com/iamflying29-sketch/googlebook-projects/blob/main/GETTING-STARTED.md)
  explains how to set this up.
- The `timeout` command, which is part of coreutils and is already installed in the Linux Terminal's Debian
  system.
- Bluetooth earbuds or other devices that report their battery level. I tested with Samsung Galaxy Buds4 and
  Google Pixel Buds 2a.
- No root and no extra apps.

## Installation

1. Set up adb in the Linux Terminal and connect it to the Googlebook over Wireless debugging, as described in the
   [Getting Started guide](https://github.com/iamflying29-sketch/googlebook-projects/blob/main/GETTING-STARTED.md).

2. Download this repository and install the command into `~/.local/bin`:

   ```
   git clone https://github.com/iamflying29-sketch/googlebook-battery-widget.git
   mkdir -p ~/.local/bin
   cp googlebook-battery-widget/bin/fix-battery-widget ~/.local/bin/
   chmod +x ~/.local/bin/fix-battery-widget
   ```

   The default Debian `~/.profile` adds `~/.local/bin` to your `PATH` when that folder exists, so open a new
   Linux Terminal window if the command is not found yet.

3. Run the command. If adb is already connected to the Googlebook, you do not need an address:

   ```
   fix-battery-widget
   ```

   Otherwise, give it the address shown under Settings > Developer options > Wireless debugging:

   ```
   fix-battery-widget <GOOGLEBOOK_IP>:<PORT>
   ```

4. Check the last line of the output. It should say `granted=true` and include `USER_SET|USER_FIXED`.

5. Reconnect your earbuds so the widget refreshes. For example, put them in their case, close the lid, and take
   them out again. A device that is paired but not connected will not appear until it connects.

If you prefer to do it by hand, the three `adb shell` commands in [How I solved it](#how-i-solved-it) are
exactly what the script runs. Replace `10` with your own Android user ID, which
`adb shell am get-current-user` prints.

## Using it

There is nothing to do day to day. The Batteries widget now lists connected Bluetooth devices next to the
laptop's battery. Keep these points in mind:

- The widget lists only devices that are connected at that moment.
- The widget refreshes when a Bluetooth device connects. I have only confirmed that behavior, so if a device is
  missing, reconnect it.
- Run `fix-battery-widget` again whenever the widget goes blank.

The command accepts these options:

| Option | What it does |
|---|---|
| `fix-battery-widget` | Uses the device that adb is already connected to. If no device or more than one device is connected, it stops with a message. |
| `fix-battery-widget IP:PORT` | Runs `adb connect IP:PORT` first and then uses that device. |
| `ANDROID_USER=<id> fix-battery-widget` | Fixes a specific Android user. By default the script asks the Googlebook for the current user with `am get-current-user`. |

Every adb call in the script has a 20 second time limit (10 seconds for finding the device), so it cannot hang.

## Updating, and what happens after app or GooglebookOS updates

To update the command, pull the repository again and copy the new file over the old one:

```
cd googlebook-battery-widget
git pull
cp bin/fix-battery-widget ~/.local/bin/
```

The `USER_SET` and `USER_FIXED` flags are how Android records a permission choice made by the owner of the
device, and Android normally keeps such choices through app updates, system updates and reboots. Turning off
automatic permission removal protects the grant when Settings Services is not opened for a long time.

Settings Services was updated from the Play Store on October 11, 2026 (to version `1.1.0.983130915.sr`). I have
not checked the grant again since that update, so I cannot promise that it survived. If the widget goes blank,
run `fix-battery-widget` again. To check the grant without changing anything, run:

```
adb shell dumpsys package com.google.android.settings.intelligence | grep -a "BLUETOOTH_CONNECT: granted"
```

These things will undo the fix for certain:

- A factory reset, or removing and re-adding your user account.
- Turning off the Nearby devices permission for Settings Services by hand.
- A future change by Google that makes the widget depend on something else. Nothing on your side can prevent
  that.

## Uninstalling

To undo the change, clear the flags, revoke the permission and restore the default app-op setting. I have not
needed to undo it myself, so these commands were not tested on my Googlebook:

```
U=$(adb shell am get-current-user | tr -d '\r')
P=com.google.android.settings.intelligence
adb shell pm clear-permission-flags --user "$U" $P android.permission.BLUETOOTH_CONNECT user-set user-fixed
adb shell pm revoke --user "$U" $P android.permission.BLUETOOTH_CONNECT
adb shell appops set --user "$U" $P AUTO_REVOKE_PERMISSIONS_IF_UNUSED default
```

Then remove the command:

```
rm ~/.local/bin/fix-battery-widget
```

The widget will go back to showing only the laptop.

## Troubleshooting

- **"No single connected adb device found."** adb is not connected, or more than one device is connected. Run
  the command with the address from the Wireless debugging screen, for example
  `fix-battery-widget <GOOGLEBOOK_IP>:<PORT>`. You can also set `ANDROID_SERIAL` to pick one of several
  connected devices.
- **The script exits right after `adb connect`.** The connection failed. The Wireless debugging port changes after
  a restart and sometimes after Wi-Fi reconnects. Open Settings > Developer options > Wireless debugging and use
  the address and port shown there. If the Linux Terminal was never paired, pair it first as described in the
  Getting Started guide.
- **"Could not determine the Android user ID."** The Googlebook did not answer `am get-current-user`. Check the
  connection, or set your Android user ID yourself, for example `ANDROID_USER=10 fix-battery-widget`.
- **The last line says `granted=false`.** Make sure you fixed the Android user ID you actually use. On my
  Googlebook it is user 10, not user 0. `adb shell am get-current-user` prints the right number.
- **The widget still shows only the laptop.** The widget lists only connected devices, and it refreshes when a
  device connects. Reconnect the earbuds. Forcing a refresh from adb is not possible, because the system rejects
  those broadcasts with a `SecurityException`. Removing the widget and adding it again is another thing you can
  try. I did not need it.

## Safety and privacy notes

- The fix changes one runtime permission for one preinstalled Google app, sets two flags on that permission, and
  turns off automatic permission removal for that same app. Nothing else is changed.
- Granting Nearby devices lets Settings Services see and connect to nearby Bluetooth devices. That is the
  permission the app already asks for, and it is what the widget needs to read battery levels.
- The script needs no root, installs nothing on the Googlebook, and sends nothing anywhere. It only talks to the
  Googlebook through adb and prints one line from the package manager.
- adb has broad control over the Googlebook. Pair Wireless debugging only with your own Linux Terminal.

## Limitations and known issues

- I do not know why GooglebookOS shipped without this permission granted. Google may fix it in a later update, or
  change the widget in a way this fix does not cover.
- The widget lists only connected devices, and it cannot be forced to refresh from adb.
- I have not confirmed that the grant survived the Settings Services update of October 11, 2026.
- I tested on one Googlebook, one GooglebookOS build and one Android user.
- A factory reset or removing your user account undoes the fix. Run the command again afterwards.

## Files in this repository

| File | Purpose |
|---|---|
| `bin/fix-battery-widget` | The repair command. It grants the permission, sets the flags, turns off automatic removal and prints the result. |
| `README.md` | This document. |
| `LICENSE` | The MIT License. |
| `.gitignore` | Keeps build outputs, keys and logs out of the repository. |

## Related projects

- [Googlebook Projects](https://github.com/iamflying29-sketch/googlebook-projects): the overview of all my
  Googlebook projects and the shared Getting Started guide.
- [Buds LE Link](https://github.com/iamflying29-sketch/googlebook-buds-le-link): keeps Galaxy Buds4 fully
  connected to Samsung's Galaxy Wearable app on the Googlebook, including its own Buds widget.

## Disclaimer and license

This is an independent, unofficial project. It is not affiliated with, endorsed by, or sponsored by Google,
Lenovo or any other company named here. All product names belong to their owners. Released under the MIT
License. Copyright (c) 2026 Thomas Liviakis.
