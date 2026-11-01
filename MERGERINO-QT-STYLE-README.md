# Mergerino (Qt) — iPhone Message Style, Style Only

This is the **true Qt Mergerino source** (public repo
`github.com/EbrahimTahernejad/Mergerino`, version 1.3.4, commit `a47a1c0`
"Release Mergerino 1.3.4"), edited **for style only** on 2026-11-01.
It is **not** a recompiled Windows `.exe`, and it is **not** the Electron
wrapper from the earlier iPhone build. No Windows `.exe` was compiled here,
and no Qt build or live Qt run was tested on this Linux VM — see
"Plain limits" below.

## The style spec (the 12 answers govern)

From `~/workspace/mergerino-iphone/answers.md` (the 12 questions, answered
one at a time). In short:

1. Chats separate by default per typed channel; header shows that channel.
   Purple `+` adds Twitch, green `+` adds Kick, purple/green gradient `+`
   merges both — that is Mergerino's existing channel-adding behaviour,
   unchanged here.
2. Left = received, right = sent by the logged-in account
   (`official_robust_clan` on Twitch and on Kick).
3. Purple bubble outline = Twitch, green bubble outline = Kick, on both
   sides.
4. Username above each left bubble; logged-in account name above right
   bubbles.
5. Avatar next to each left name, from the official profile picture where
   Mergerino already provides one; initials fallback, never a guessed face.
6. No timestamps, just bubbles.
7. Emotes as images, as they already render per platform.
8. Bottom typing bar (existing split input, unchanged).
9. Light/dark toggle — here, the two iPhone themes below.
10. Full iPhone frame / Settings iPhone-model dropdown from the Electron
    version are **translated, not copied** (see below).

## How to turn it on after building

1. Build and run (below).
2. Settings → General → Messages → tick **"iPhone message style"**.
   (Optional: tick **"Show avatar initials in iPhone style"** — on by
   default.)
3. Settings → Appearance → Themes → pick **"iPhone (Light)"** for light
   mode or **"iPhone (Dark)"** for dark mode. Switching themes is the
   light/dark toggle for this style; the checkbox alone only switches
   the bubble layout on and off.
4. Log in on Twitch and/or Kick as usual. Your own messages then move to
   the right; everyone else's stay on the left.

## What was changed, file by file

### `src/singletons/Settings.hpp`
- Added `iphoneMessageStyle` (`/appearance/messages/iphoneMessageStyle`,
  default `false`) — the master switch for the bubble layout.
- Added `iphoneMessageShowAvatars`
  (`/appearance/messages/iphoneMessageShowAvatars`, default `true`) —
  gates the avatar-initial circle for received messages only.
- No other setting was changed, renamed, or removed.

### `src/messages/MessageFlag.hpp`
- Added one layout-only flag: `IPhoneOwnMessage = (1LL << 47)`.
  Bits 47–63 were unused. It is set **only on a local copy** of the flags
  in `MessageLayout::actuallyLayout` and is never written back to
  `Message::flags`, never persisted, and never used for moderation,
  highlights, or protocol decisions.

### `src/messages/layouts/MessageLayout.cpp`
- Added style-only helpers in the anonymous namespace:
  - `iphoneStyleEnabled()` — reads the new setting.
  - `iphoneIsOwnMessage()` — true when a message's `loginName` matches
    the current Twitch account (`twitch.getCurrent()->getUserName()`,
    non-anonymous) **or** the current Kick account
    (`kick.current()->username()`, non-anonymous), case-insensitively.
    So a user logged in as `official_robust_clan` on both platforms is
    "own" on both. Anonymous accounts never match.
  - `iphoneOutlineColor()` — reuses Mergerino's own platform accent:
    `message.platformAccentColor` if set, else `defaultPlatformAccent`,
    which is already purple `(145, 70, 255)` for Twitch/`AnyOrTwitch`
    and green `(83, 252, 24)` for Kick. No new colour mapping was
    invented for the outlines.
  - `iphoneBubbleFill()` — sent = iOS message blue `#0a84ff` in both
    themes; received = iOS system grey `#e9e9eb` (light) / `#2c2c2e`
    (dark), inferred from the chat text colour's lightness.
  - `iphoneAvatarInitial()` — first letter of the display name, upper
    case; `?` if the name is empty.
- `MessageLayout::paintBackground()` — new early branch, only when the
  style is on, the message has a sender (`loginName` non-empty), and it
  is not a system message. It paints:
  - a rounded rectangle (14px radius, 2px outline in the platform
    colour) as the row bubble;
  - for received messages with avatars on, a 24px circle at the row's
    left containing the sender's initial, with the bubble starting
    after it (so the avatar sits outside the bubble, iMessage-style).
  - Highlighted / highlighted-whisper messages keep their highlight
    colour as the bubble fill instead of the normal fill.
  - Everything else returns to the original full-width background code
    unchanged, including all event-highlight logic.
- `MessageLayout::actuallyLayout()` —
  - Sets `IPhoneOwnMessage` on the local flags copy for own messages.
  - Once per message, breaks the line after the first `Username`
    element, so the username (and any badges before it) sits on its own
    line above the message text, instead of inline before it. Normal
    Twitch/Kick messages carry exactly one Username element (`name:`),
    so this breaks once. Whisper rows carry two Username elements and
    break after the first — a known cosmetic edge case of this style.
  - Includes added: `controllers/accounts/AccountController.hpp`,
    `providers/kick/KickAccount.hpp`, `providers/twitch/TwitchAccount.hpp`
    (only for the own-message account check above).

### `src/messages/layouts/MessageLayoutContainer.cpp`
- `MessageLayoutContainer::breakLine()` — new iPhone branch, only when
  the style is on and the line is not `Centered`:
  - Own messages (`IPhoneOwnMessage`): right-align each line, using the
    same arithmetic as the existing `Centered` branch but without the
    `/2` — i.e. shift = `width − margins − lastElement.right()`.
  - Received messages with avatars on: indent each line by 32px × scale
    (24px avatar + 8px gap), matching the avatar circle painted in
    `paintBackground`.
  - This is per-line alignment, not per-bubble: a multi-line message's
    lines each align independently. See "Plain limits".

### `src/singletons/WindowManager.cpp`
- `updateWordTypeMask()` — timestamps are now added to the word mask
  only when `showTimestamps` is on **and** iPhone style is off. This is
  how Q8 ("no timestamps, just bubbles") is implemented: the Timestamp
  element is simply not laid out. The underlying `parseTime` /
  `serverReceivedTime` on each `Message` is untouched, so turning the
  style off restores timestamps exactly.
- Constructor listeners: `iphoneMessageStyle` was added to both
  `updateWordTypeMaskListener` (rebuild the mask on toggle) and
  `forceLayoutChannelViewsListener` (re-lay out chat on toggle);
  `iphoneMessageShowAvatars` was added to the force-layout listener.
  Toggling either setting therefore takes effect without a restart.

### `src/singletons/Theme.cpp`
- Registered two built-in themes after `Dark` (deliberately **not**
  before it, because `fallbackTheme` is `builtInThemes.at(1)`, i.e.
  Dark — inserting before Dark would silently change the fallback):
  - `iPhoneLight` → `:/themes/iPhoneLight.json`, shown as
    **"iPhone (Light)"**.
  - `iPhoneDark` → `:/themes/iPhoneDark.json`, shown as
    **"iPhone (Dark)"**.
- No existing theme was edited, renamed, or removed.

### `resources/themes/iPhoneLight.json` (new)
- Based on upstream `Light.json`, same schema and sections. Changed:
  accent `#0a84ff`; message backgrounds regular `#ffffff` /
  alternate `#f2f2f7`; window `#f2f2f7`; splits background `#f2f2f7`,
  header/input white with black text; message link `#0a84ff`.
- Note: the theme JSON does **not** carry the per-side bubble colours
  or the purple/green outlines — the theme system only has
  regular/alternate row backgrounds. Those are painted in
  `MessageLayout::paintBackground` (above), from the platform accent.

### `resources/themes/iPhoneDark.json` (new)
- Based on upstream `Dark.json`, same schema and sections. Changed:
  accent `#0a84ff`; message backgrounds regular `#1c1c1e` /
  alternate `#2c2c2e`; splits background `#000000`, header/input
  `#1c1c1e` with white text; message link `#0a84ff`.
- Theme files under `resources/themes/` are auto-globbed into the Qt
  resources by `cmake/resources/generate_resources.cmake`, so no
  `.qrc` or CMake edit was needed to bundle them.

### `src/widgets/settingspages/GeneralPage.cpp`
- Added two checkboxes under General → Messages, after "Alternate
  background color": **"iPhone message style"** and **"Show avatar
  initials in iPhone style"**, with tooltips and search keywords
  (`iphone`, `imessage`, `bubble`, `avatar`, `initials`). No other
  settings-page code was touched.

### This file (`MERGERINO-QT-STYLE-README.md`, new)
- Documentation only; not part of the upstream source.

## What was deliberately NOT changed

- **No protocol code:** no Twitch IRC, Kick Pusher/WebSocket, YouTube,
  or TikTok code was touched. No new network calls were added.
- **No channel logic:** channel adding, merged-channel handling
  (`MergedChannel`), splits, tabs, and the purple/green/gradient `+`
  behaviour from the Electron build's spec are Mergerino's existing
  behaviour and are unchanged here. The 12-answer "separate by default,
  merged on gradient +" is how Mergerino already works; this edit does
  not re-implement it.
- **No moderation, highlights, or mentions systems:** the only
  highlight interaction is that a highlighted message's bubble fill
  uses its existing highlight colour (see above).
- **No emote code:** Twitch/Kick/7TV/BTTV/FFZ emote rendering is
  untouched and still renders as images, as before. Emote *lookup
  order* was not changed.
- **No accounts or settings persistence changes** beyond the two new
  boolean settings above. No saved Mergerino settings are migrated or
  rewritten by this edit.
- **No YouTube/TikTok sources added or removed** — this style edit is
  platform-agnostic in code; the 12-answer spec is Twitch + Kick, and
  YouTube/TikTok messages, if present, simply get their existing
  platform accent as their outline.
- **No input bar, split header, or notebook changes:** the bottom
  typing bar is the existing split input, unchanged.

## Translation notes: Electron concepts that do not exist in Qt

- **Full iPhone phone frame / Settings iPhone-model dropdown
  (Dynamic Island, Notch, Home button):** a desktop Qt chat window has
  no phone frame to draw, and faking one around the window would be a
  window-chrome change, not a chat style. Translated to the two
  **iPhone themes** above (Light/Dark) selectable in
  Settings → Appearance → Themes. If a future change wants a model
  dropdown, it would belong in the Themes page as a theme variant
  selector, not in chat rendering.
- **Contact-style adding:** Mergerino's existing channel-adding UI
  (split header / channel switcher) is unchanged. The Electron build's
  contact-card adding is an Electron-app concept and was not ported
  into Qt, because it would be a new dialog, not a style.
- **"Right = sent, left = received" for the *logged-in* account:**
  implemented via the account check above. If you are not logged in on
  a platform, its messages cannot be "own" and all stay on the left —
  that is correct, not a bug.

## Avatar honesty (read this before expecting photos)

Qt chat rows in upstream Mergerino do **not** carry a per-message
profile photo for normal Twitch/Kick messages:

- Kick chat payloads handled in `KickMessageBuilder::makeChatMessage`
  carry `sender.username` / `sender.id` / `identity`, but no
  `profile_pic` for the sender, so there is nothing to render in-row
  without a new per-chatter lookup.
- Twitch profile pictures are resolved on demand for the user-info
  popup (`TwitchUsers::resolveID` → `profilePictureUrl`), not stored
  on each chat message.

Therefore this style edit paints **initials only** in the avatar
circle (first letter of the display name, platform-coloured circle).
It never fetches, invents, or guesses a face or photo. Real profile
pictures remain available exactly as before, in the user-info popup
(click a username) and in shared-chat badges. A future *functional*
change — not a style change — could store a sender picture URL on
`Message` (e.g. from Kick's user lookup or Twitch's user lookup) and
swap the initials circle for a `CircularImageElement`; that fetch,
caching, and fallback work is intentionally out of scope here.

## Plain limits — what could NOT be done in Qt style alone

1. **Bubbles are row bubbles, not content-tight phone bubbles.**
   Qt chat rows are full-width and wrap at full row width. The bubble
   is an inset rounded rectangle spanning the row (narrower on the
   left only by the avatar space), with right-aligned *text lines*
   for own messages. A true content-tight bubble (width = longest
   line) would require reworking `MessageLayoutContainer`'s width
   model and selection painting, which is a layout-engine change,
   not a style change, and was not attempted.
2. **Username-above is a line break, not a separate label widget.**
   The username line still contains any badges that precede the
   username, in their original order, and the trailing `:` from
   upstream (`name:`) is kept. Removing the colon or restyling the
   label would touch the message builders, not just the style.
3. **Avatar = initials, not photos** (see above).
4. **No phone frame, no iPhone-model dropdown** (see translation notes).
5. **Emote, badge, and text colours inside bubbles are upstream's.**
   Sent (blue-fill) messages keep their upstream text colour, so a
   sent message's text may not be pure white in every theme — the
   bubble fill is iOS blue, the text is whatever the theme / username
   colour rules already produce. Tuning sent-text contrast would be a
   further theme change.
6. **Not compiled or run here.** See below.

## How to build it on Windows (from upstream `docs/building-windows.md`)

Prerequisites (exact, from upstream):

- Windows 10 or Windows 11, x64
- Visual Studio 2022 with **Desktop development with C++**
- Qt 6 for `msvc2022_64` with the **Qt Image Formats** module —
  the release build currently uses **Qt 6.9.3**; matching that is the
  safest starting point (other close versions may work)
- Python 3
- Conan 2 — release build uses **Conan 2.11.0**
- CMake
- Ninja
- Git

Steps (run in an **x64 Native Tools Command Prompt for VS 2022**,
from the extracted source root — this zip, not a git clone):

```cmd
set PATH=C:\Qt\6.9.3\msvc2022_64\bin;%PATH%
python -m pip install "conan==2.11.0"
conan profile detect --force
mkdir build-conan
cd build-conan
conan install .. -s build_type=Release -c tools.cmake.cmaketoolchain:generator=Ninja --build=missing --output-folder=. -o with_openssl3=True
cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE=conan_toolchain.cmake -DUSE_PRECOMPILED_HEADERS=ON -DCHATTERINO_LTO=OFF -DCHATTERINO_SPELLCHECK=ON ..
cmake --build . --parallel
```

Run it:

```cmd
build-conan\bin\mergerino.exe
```

If your prompt is still inside `build-conan`, that is `bin\mergerino.exe`.

Standalone bundle (optional, from upstream):

```cmd
windeployqt build-conan\bin\mergerino.exe --release --no-compiler-runtime --no-translations --no-opengl-sw --dir build-conan\bin
```

The GitHub release workflow additionally copies dependency DLLs from
the build `bin` folder and deploys the Visual C++ runtime before
zipping; see `.github/workflows/` in the source for that packaging.

Notes specific to this zip:

- The `lib/` dependency sources are included in this zip (they live in
  the upstream repository), so no separate submodule step is needed in
  the normal case. If CMake complains about a missing
  `lib/.../CMakeLists.txt`, run
  `git submodule update --init --recursive` after extracting into a
  git checkout — that is upstream's documented fallback, not a step
  this style edit introduced.
- YouTube OAuth: official release builds inject a private YouTube
  OAuth client secret during CI; it is **not** in this source. Without
  it, Mergerino reads the secret from the `MRAB` environment variable
  at runtime (`set MRAB=<your-oauth-client-secret>` before running).
  Anonymous YouTube viewing may still work; authenticated YouTube
  features need your own Google OAuth credentials. This is unchanged
  from upstream and unrelated to the iPhone style.
- Troubleshooting (from upstream): `Could not find Qt6` → verify the
  kit matches `msvc2022_64`, or pass
  `-DCMAKE_PREFIX_PATH=C:\Qt\6.9.3\msvc2022_64` to CMake. Plugin
  configure errors → first try `-DCHATTERINO_PLUGINS=OFF` to isolate
  the base app build.

## Verification done here (and NOT done here)

Done on this Linux VM:

- `git diff --stat` on the modified tree: 7 tracked files changed
  (282 insertions, 1 deletion — the 1 deletion is the single
  `showTimestamps` condition line replaced in `WindowManager.cpp`),
  plus 2 new untracked theme JSONs and this README.
- Brace/paren/bracket balance checked on every edited C++ file —
  all balanced.
- Both new theme JSONs parsed with Python's `json` module — valid.
- Every new identifier used in the edits was checked against the
  upstream source before writing: `iphoneMessageStyle` /
  `iphoneMessageShowAvatars` (new, defined in `Settings.hpp`),
  `MessageFlag::IPhoneOwnMessage` (new, defined in
  `MessageFlag.hpp`), `twitch.getCurrent()->getUserName()` /
  `isAnon()`, `kick.current()->username()` / `isAnonymous()`,
  `platformAccentColor` / `defaultPlatformAccent`,
  `MessageElementFlag::Username`, `MessageFlag::System`,
  `MessageFlag::Highlighted` / `HighlightedWhisper`,
  `highlightColor`, `ctx.messageColors.regularText`,
  `getSettings()->iphoneMessageStyle.getValue()` (same accessor
  pattern as `showTimestamps.getValue()` elsewhere).
- Zip contents checked: README at the zip root, both theme JSONs,
  and all 7 changed source files present.

**NOT done here, and not claimed:**

- No Qt toolchain is installed on this VM, so the code was **not
  compiled** (no `cmake`/`ninja` Qt build, no compiler errors
  checked). The edits are syntactically plausible and reference-checked
  as above, but a Windows Qt build is the first real compile check —
  if it reports an error, it will be in one of the files listed above,
  and the error message will name the exact line.
- No Windows `.exe` was produced from this source here.
- No live Qt run was tested here — bubble geometry, right alignment,
  and the avatar circle's exact pixel alignment with the username
  line are the first things to eyeball on Windows, and may need a
  1–2px tweak in `paintBackground` / `breakLine` (the 32px avatar
  indent, 24px circle, 14px corner radius, and 2px outline are the
  constants to adjust).
- No upstream test suite was run here (the Qt test target also
  requires the Qt toolchain).

## Upstream credit

Mergerino is an unofficial fork of Chatterino
(`github.com/Chatterino/chatterino2`). This edit changes only the
style layer described above; all chat, account, and protocol credit
remains with upstream Mergerino/Chatterino. This modified source is
distributed under the same terms as the upstream repository (see
`LICENSE` / `LICENSES` in the source tree).
