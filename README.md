# Telegram Admin (Android)

A small Android app that runs a personal Telegram bot in the background and lets you manage
your own phone from a chat: check it's alive, pull a quick system status, browse/open/upload/
download files, and restart the app (with an optional device-owner path to reboot the whole
phone). Built to be stable, easy to reconfigure without rebuilding, and easy to extend with new
commands.

## What this can and can't do (read this first)

Android sandboxes apps much more tightly than a desktop OS, so a few things work differently
than "admin tool" might imply:

- **No true shutdown, and reboot needs one-time setup.** A normal app cannot power off or
  reboot the device - that requires root or Device Owner privileges. `/restart` always works
  and restarts the *app process*. `/reboot` reboots the *device* but only after you provision
  this app as Device Owner once via `adb` (see below) - without that it replies explaining why
  it can't. There's no `/shutdown`; nothing short of root can power off an Android phone
  programmatically, so it isn't implemented rather than faking it.
- **"Opening" a file launches a viewer app**, the same as tapping the file would. If the screen
  is off/locked, it'll unlock into that app when you next look at the phone (or fail silently if
  nothing on the device can open that file type - the bot will tell you).
- **File access needs one manual permission grant** ("All files access"), because Android
  doesn't allow requesting that one through a normal permission dialog. The app has a button
  for it; commands that touch files will tell you if it's missing.
- **The service runs as a foreground service** (a persistent low-priority notification while
  it's active). That's the standard, expected way to keep any background connection alive on
  modern Android - without it, the OS will suspend the network connection within minutes.

## Configuring the bot token and owner ID

Two sources are supported, matching what you asked for (env vars / an appsettings-style file),
adapted to how Android actually builds and runs:

1. **Build-time (like an env var / .env file):** copy `local.properties.example` to
   `local.properties` and fill in `botToken` / `ownerId`, **or** just set the environment
   variables `TELEGRAM_BOT_TOKEN` and `TELEGRAM_OWNER_ID` before building (handy for CI or a
   reproducible build script). `local.properties` is git-ignored so secrets never get committed.
   These become baked-in defaults via `BuildConfig`.
2. **Runtime (the "appsettings file" equivalent):** the app also has a Settings screen (just
   two fields + Save) that writes to a private JSON file at
   `/data/data/com.example.telegramadmin/files/config.json`. This is what lets you change the
   token or owner ID without rebuilding the app at all. If this file has a value, it wins over
   the build-time default.

Your **owner chat ID** is the numeric Telegram user ID of the only account allowed to issue
commands - every other chat is silently ignored. The easiest way to find yours: message
[@userinfobot](https://t.me/userinfobot) on Telegram and it will reply with your ID. Create your
bot itself via [@BotFather](https://t.me/BotFather) (`/newbot`) to get a token.

## Permissions you'll be asked to grant

| Permission | Why |
|---|---|
| Notifications | Required to show the "bot is running" foreground notification (Android 13+). |
| All files access (MANAGE_EXTERNAL_STORAGE) | Needed for `/ls`, `/open`, `/get`, and incoming file uploads to reach files outside the app's own sandbox. Granted from the app's "Grant All files access" button, which opens the relevant system settings screen. |
| Location (foreground + background) | `/location` needs a fresh GPS fix even when you're not looking at the app (it's a background service). Tap "Grant location access" twice: once for the basic permission, then a second tap sends you to system Settings to pick "Allow all the time" — Android won't offer that option until the app has explicitly requested it once. |
| Battery optimization exemption (recommended, manual) | Not requested automatically, but for the most reliable long-running connection, disable battery optimization for this app in Android's Settings → Apps → Telegram Admin → Battery. |

## Architecture

```
config/         BotConfig + ConfigManager (env/local.properties → BuildConfig, runtime JSON file)
telegram/       TelegramClient (OkHttp long-polling wrapper) + model/IncomingMessage
service/        BotForegroundService (the reconnect loop + dispatch) and BootReceiver
commands/       Command interface, CommandRegistry, CommandContext, IncomingFileHandler
commands/impl/  One file per command (help, ping, status, ls, open, get, restart, stop, reboot)
admin/          AdminReceiver (Device Admin boilerplate, only matters if you enable /reboot)
ui/             MainActivity (settings screen + start/stop/permission buttons)
```

**Reconnect behavior:** `BotForegroundService` runs a single loop calling Telegram's
`getUpdates` long-poll endpoint (30s timeout, per Telegram's current docs). Any failure
(network drop, Telegram-side error, timeout) is caught, the notification switches to
"Reconnecting…", and the loop retries with exponential backoff (2s → 4s → 8s… capped at 60s),
resetting back to 2s after the next successful call. Nothing about a dropped connection ever
kills the service - only an explicit `/stop` (or you closing it from the app) does.

**Graceful stop:** stopping the service (from the app, from `/stop` in chat, or Android tearing
it down) cancels the single coroutine job backing the poll loop, which immediately unblocks any
in-flight wait and exits the loop - no dangling requests, no half-sent state.

## Adding a new command

1. Create a class in `commands/impl/` implementing `Command` (`name`, `description`,
   `suspend fun execute(ctx: CommandContext): String`).
2. Add an instance to the list in `CommandRegistry.buildDefault()`.

That's the entire extension surface - `/help` picks it up automatically, and nothing else in
the service or dispatch logic needs to change. `CommandContext` already carries the Telegram
client, config, the incoming message, parsed args, service start time, and a callback to stop
the service, so most commands need nothing beyond that bundle.

## Optional: enabling `/reboot` (Device Owner)

This is entirely optional and only needed if you want the phone to actually reboot itself from
chat, not just the app. It requires **no root**, but does require a one-time `adb` command run
from a computer, on a device with no other accounts/managed profiles set up (Device Owner can
normally only be set on a "fresh" device or right after a factory reset):

```
adb shell dpm set-device-owner com.example.telegramadmin/.admin.AdminReceiver
```

If that succeeds, `/reboot` will work going forward. If it doesn't apply to your phone (existing
Google account already configured, etc.), `/reboot` will just tell you it's unavailable and
`/restart` remains available for the app itself.

## Building and running

Open the `TelegramAdminApp/` folder in Android Studio (Koala or newer) and let it sync - it will
offer to generate the Gradle wrapper jar automatically if it's missing. Minimum SDK 26,
target/compile SDK 34.

```
# from the command line, once the Gradle wrapper is present:
./gradlew assembleDebug      # builds app/build/outputs/apk/debug/app-debug.apk
./gradlew testDebugUnitTest  # runs the test project below
```

Change `applicationId`/`namespace` in `app/build.gradle.kts` if you want your own package name
before installing alongside other apps.

## Test project

`app/src/test/` contains JVM/Robolectric unit tests covering the logic that doesn't need a real
device:

- `TelegramUpdateParsingTest` - parsing raw Telegram `getUpdates` JSON into `IncomingMessage`,
  including edge cases (non-message updates, missing `update_id`, documents with captions).
- `CommandRegistryTest` - every built-in command is registered and findable, lookup is
  case-insensitive, and no two commands accidentally share a name.
- `ConfigManagerTest` - the runtime config file round-trips correctly, an unconfigured app
  reports itself invalid, and the token is never exposed in full by `redactedToken()`.

Run them with `./gradlew testDebugUnitTest` or the test runner gutter icons in Android Studio.

## Sending/receiving files in chat

- **Download from phone → chat:** `/get ~/Download/report.pdf` (paths starting with `~/` are
  relative to shared storage root; absolute paths also work). Capped at Telegram's 50MB bot
  upload limit.
- **Upload from chat → phone:** just send a file to the bot. With no caption it's saved to
  `Download/TelegramAdmin/`; add a caption with a path (e.g. `~/Documents`) to choose the
  destination folder. Capped at Telegram's 20MB bot download limit.

## v2 media capture (user-approved)

Version 2 adds `/front`, `/back`, and `/mic` commands. These are intentionally **not silent remote surveillance controls**. A Telegram request only creates a local notification. The phone owner must tap that notification, open the capture screen, and approve Android's camera/microphone permission before anything is captured. Camera photos and microphone recordings are then sent to the authorized Telegram chat.

- `/front` — user-approved front-camera photo.
- `/back` — user-approved rear-camera photo.
- `/mic` — user-approved microphone recording, automatically limited to 30 seconds.
- No camera/microphone capture occurs while the app is merely polling Telegram.
- Android's normal privacy indicators/permission controls remain in effect.

This design is deliberate: Android camera/microphone access should be visible to the device user rather than silently activated from a remote chat command.

## Fast ZIP transfer (v2.0)

`/zip <authorized-folder>` now uses a performance-oriented transfer pipeline:

- ZIP entries use the **STORED** method, so file contents are not DEFLATE-compressed.
- The next ZIP part is prepared in parallel with the current Telegram upload.
- Only one part is prefetched, keeping temporary storage and memory bounded.
- Parts remain capped at 45 MiB to stay below the Telegram upload limit configured by the app.
- Files larger than the per-part limit are skipped with the existing warning behavior.

Because a true STORED ZIP entry needs its CRC32 before it is written, each source file is read once for CRC calculation and once for the archive copy. This removes compression CPU work at the cost of additional storage reads.
