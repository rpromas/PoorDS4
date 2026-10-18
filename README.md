# PoorDS4

> [!NOTE]
> **This is a fork of [ItsBlurf/PoorDS4](https://github.com/ItsBlurf/PoorDS4)** that adds
> support for apps built with the PS5 payload SDK, such as
> [ProsperoEden](https://github.com/blackbearreloaded/ProsperoEden), and fixes mixed
> DualSense + DS4 multiplayer. It carries three changes that are open as pull requests
> upstream:
>
> - [#37](https://github.com/ItsBlurf/PoorDS4/pull/37) (by freekvlier): hook `-fno-plt`
>   GOT imports, so PoorDS4 works in SDK-built apps such as ProsperoEden.
> - [#41](https://github.com/ItsBlurf/PoorDS4/pull/41): native DualSense players keep
>   their own input when a DS4 is in the game (the DS4 no longer mirrors to them).
> - [#42](https://github.com/ItsBlurf/PoorDS4/pull/42): a second, third and fourth DS4
>   each reach their own player.
>
> Tested on firmware 13.60 with ProsperoEden and 2 DualSense + 2 DS4 (four players).
> Builds are on the [Releases](https://github.com/rpromas/PoorDS4/releases) page.
> Report problems with these builds [here](https://github.com/rpromas/PoorDS4/issues),
> not upstream. This fork is temporary: once upstream merges the pull requests, use
> [ItsBlurf/PoorDS4](https://github.com/ItsBlurf/PoorDS4) again.

> PoorDS4 would not have been possible without the excellent work behind
> [Ghostcontrol](https://github.com/StonedModder/Ghostcontrol-PS5-USB-Controller-Patcher)
> by StonedModder. Its controller research and PS5 payload foundation made this
> project possible, this project will eventually be merged to it later on if stoned Modder agrees even though its different now.
> I'm only using this for testing and gathering logs from testers until its fully stable and make this implementation much cleaner.
> PoorDS4 also relies on the
> [PS5 Payload SDK](https://github.com/ps5-payload-dev/sdk) and the remote-syscall
> interface documented by [kstuff-lite](https://github.com/EchoStretch/kstuff-lite).

PoorDS4 lets a wireless DualShock 4 paired with a jailbroken PS5
control native PS5 games. It does not require USB, a DualSense, a second user,
or a profile-selection prompt. Native DualSense controllers remain on Sony's
original input path and can be used independently for local multiplayer.

This is experimental homebrew that modifies a running game's pad imports.
Compatibility checks fail closed, but untested firmware and games can still
crash. Save your gamesave before testing (it broke the save file of a game during very early builds testings, never happened again but there is stil the risk so this is your disclaimer to save your work).

## Requirements

- A jailbroken PS5 with a compatible HEN/kstuff environment and ELF loader.
- A wireless DualShock 4 paired with the PS5 and connected to a logged-in user.
- Firmware 11.60 for the currently live-tested configuration. See
  [firmware support](docs/FIRMWARE_SUPPORT.md) before testing another version.

## Quick start

1. **Bluetooth Pairing**: Put your DualShock 4 into pairing mode by holding `SHARE` + `PS Button` together until the lightbar starts double-blinking rapidly. On the PS5, go to `Settings > Accessories > Bluetooth Accessories` and select **DUALSHOCK 4**.
2. **Controller Profile**: Connect the DS4 to whichever user profile should play (Player 1 or Player 2). If Player 1 is already using a native DualSense, connect the DS4 under Player 2's profile.
3. **Deploy Payload**: Send `PoorDS4rc51.elf` to the console's ELF loader (port 9021). The game may already be running or launched afterward.
4. **Play**: Wait for the `wireless DS4 active` notification, then play normally.

Only run one automatic instance. PoorDS4 follows later game launches without reinjection. It performs safe cleanup before rest mode; reinject after waking. Use `PoorDS4-stop.elf` before replacing a running build.

## In-Game Hotkeys & Reset Combo

- **Universal Reset Shortcut (`L1 + R1 + L2 + R2` held for ~400ms)**:
  Holding `L1 + R1 + L2 + R2` simultaneously on **any connected controller** (DS4 or DualSense) triggers an instant in-game re-synchronization. PoorDS4 will quiesce active hooks, re-evaluate all game pad slots, re-bind the controller to the correct profile, and re-attach seamlessly without needing to reinject the payload.
  Use this shortcut if:
  - You switched user profiles in the middle of a game session.
  - Player 2 joined late after the title screen.
  - You want to force a clean re-detection of the controller table.

## Controller Compatibility & Multiplayer

- **Supported Controllers**: Sony DualShock 4 v1 (`054c:05c4`), DS4 v2 (`054c:09cc`), and official Sony wireless USB adapters (`054c:0ba0`).
- **Multi-Controller & Player 2**: Fully supported. DualSense controllers remain strictly on their original Sony hardware input path and are never clobbered or stolen. PoorDS4 detects open/waiting player slots and binds cleanly to Player 2 alongside a Player 1 DualSense.
- **Native DualSense Touchpad Emulation**: Touchpad geometry is spoofed at native `1920x1080` resolution matching DualSense hardware, ensuring full compatibility with Unreal Engine 4/5 titles, Stellar Blade, and EA Sports FC 26.

## How it works

1. Enumerate the logged-in and Invite users' pad slots.
2. Identify a live DS4 through public device metadata and Sony's DS4 API.
3. Read its 120-byte `ScePadData` state at 120 Hz in `SceRemotePlay`.
4. Wait for one fully initialized native game process.
5. Validate the game's pad ABI, client table, player slot, import owners, and
   target mappings before changing anything.
6. Redirect only validated pad imports to a game-owned anonymous mapping and
   publish translated frames through a double buffer.

The game is never ptraced, stopped, or used to run a borrowed thread. No eboot
or `libScePad` code page is patched, and no game heap, socket, or thread is
created. See the [architecture document](docs/ARCHITECTURE.md) for the exact
safety and cleanup invariants.

## Firmware compatibility

| Firmware | Status |
| --- | --- |
| 6.02 (`0x06020004`) | Exact manifest and dynamic table stride (`0x548`) verified from memory dumps |
| 8.60 (`0x08600004`) | RC39 installed cleanly; RC42/RC43 includes queued/state source reader, source-context setup, and native-frame preservation |
| 10.01 (`0x10010000`) | Exact manifest and stride (`0x5c8`) verified from memory dumps |
| 11.60 (`0x11600005`) | Live-tested across multiple games (Greak, Pragmata, JoJo, FC 26), multiple connection orders, reconnects, and rest cleanup |
| 12.40 (`0x12400009`) | Exact manifest verified from supplied reports and memory dumps |
| Other | Dynamic table stride discovery (`0x548`/`0x5c8`) and structural verification allow execution if ABI matches; fails closed safely |

Compatibility is based on proven ABI structure, not a broad `11.xx` or `12.xx`
version assumption. Unknown layouts fail closed and produce a report instead
of installing hooks. See [docs/FIRMWARE_SUPPORT.md](docs/FIRMWARE_SUPPORT.md).

## Build

Install the official [ps5-payload-sdk](https://github.com/ps5-payload-dev/sdk),
set `PS5_PAYLOAD_SDK`, then build from the repository root:

```sh
make -C payload clean
make -C payload release
```

On the Windows PS5 development workspace used for release builds, load the
environment and select its target wrapper explicitly:

```powershell
. .\ps5dev-env.ps1
make -C payload CC=ps5-clang.cmd clean
make -C payload CC=ps5-clang.cmd release audit
```

RC51 release assets use ps5-payload-sdk v0.42:

| Output | Purpose |
| --- | --- |
| `PoorDS4rc51.elf` | Automatic wireless DS4 bridge |
| `PoorDS4-status.elf` | Read-only bridge status snapshot |
| `PoorDS4-stop.elf` | Cooperative stop request |

## Logs and compatibility reports

Diagnostics are stored under `/data/poords4/`:

- `game-pad-bridge.log` and `game-pad-bridge.log.1`
- `game-pad-bridge-supervisor.txt`
- `game-pad-bridge-last.txt`
- `game-pad-bridge-status.txt` after running the status ELF
- `reports/source-fw-XXXXXXXX-pid-N.txt`
- `reports/fw-XXXXXXXX-pid-N.txt`

For a firmware or game incompatibility, copy the complete `/data/poords4/`
directory and include the steps that reproduced the failure. Source and game
reports identify `poords4_rc` and `report_schema`. Review reports before posting
them publicly because they contain runtime process addresses and controller
diagnostics.

## Multi-Controller Support (Experimental)

PoorDS4 now supports streaming multiple wireless DS4 controllers concurrently (up to 4 controllers as Player 1..Player 4), along with simulated controller channels and native DualSense passthrough. Please note that multi-controller support is currently **experimental** as I haven't done heavy testing with multiple physical DS4s yet because I don't have a reliable second DualShock 4 controller. If you have multiple DS4 controllers, please test them out and let me know how they work!

## Testers Needed

I need testers especially for firmwares **13.40**, **13.20**, **13.00**, **12.60**, and **9.60**. If you encounter any bugs, crashes, or want to contribute test logs, please zip the entire `/data/poords4/` folder on your PS5 and send it over in GitHub issues or reach out to me directly on Discord (`blurf.`).

## Special Thanks & Shoutout

A huge shoutout to [reyzinhoplayoliver-design](https://github.com/reyzinhoplayoliver-design) / `ElCauaRey` on Discord for helping with a lot of testing and finding bugs!

## Donations & Support

If you find PoorDS4 helpful and would like to support faster future development, hardware acquisition (like additional controllers and testing gear), and maintenance, please consider donating:

- **PayPal**: [paypal.me/nblurf](https://paypal.me/nblurf)
- **Bitcoin (BTC Network)**: `1CW8JWSnWc5w7yKhYk7ixAFG487nU5Jrzc`
- **USDT (Tron / TRC20)**: `TRcrCeonXzgyxjR4LGzo4JdfEHYGGqD3gX`

## Attribution and license

PoorDS4 is a focused wireless-DS4 derivative of Ghostcontrol. The original USB
implementations and binaries are intentionally not distributed in this tree.
See [NOTICE.md](NOTICE.md) for detailed credits and [LICENSE](LICENSE) for the
GPL-3.0-or-later terms.
