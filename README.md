# NND Dynamic Reveal

A pair of companion mods that turn [NPCs Names Distributor (NND)](https://www.nexusmods.com/skyrimspecialedition/mods/73081)'s name-obscuring feature into a roleplay mechanic for [SkyrimNet](https://github.com/MinLL/SkyrimNet-GamePlugin): your character doesn't know an NPC's name until they say it themselves in conversation, or until you decide they should and reveal it yourself with a hotkey. Works for generic NPCs (guards, etc.) just as well as named vanilla characters.

- **`NND-DynamicReveal`** — SKSE plugin. Listens for SkyrimNet dialogue: when an NPC says their own name, it's revealed automatically. Press **H** (configurable) to reveal the name of whoever's under your crosshair manually instead.
- **`NPCsNamesDistributor`** — A fork of NND.

**Quirk:** the auto-reveal only matches words four letters or longer, so a short name (three letters or fewer) will never get picked up automatically even if the NPC says it. The hotkey covers that gap — and more generally, it's there for whenever you'd rather decide for yourself that your character now "knows" someone, regardless of what's actually been said.

## Requirements

- [SKSE64](https://skse.silverlock.org)
- [NPCs Names Distributor](https://www.nexusmods.com/skyrimspecialedition/mods/73081) — the base framework this builds on (this zip's `NPCsNamesDistributor.dll` replaces the official one)
- [SKSEMenuFramework](https://www.nexusmods.com/skyrimspecialedition/mods/120352) — for NND's own settings menu
- [SkyrimNet](https://github.com/MinLL/SkyrimNet-GamePlugin) — this mod's entire premise is reacting to SkyrimNet dialogue; without it there's little reason to use it over plain NND

## Installation

1. Download `NNDDynamicReveal.zip` from the [releases page](https://github.com/jayross93/NPCsNameDistributor-DynamicReveal/releases) and drag it into MO2 (or extract `Data/SKSE/Plugins/` into your Skyrim `Data` folder manually).
2. Make sure `NNDDynamicReveal` has **higher priority** than the official NPCs Names Distributor mod in your load order, so its bundled `NPCsNamesDistributor.dll` wins the file conflict.
3. In NND's MCM (open with the SKSEMenuFramework key), under **Obscurity**, turn off "Reveal in Greetings", "Reveal when Looting", and "Reveal when Pickpocketing". This mod's hotkey and SkyrimNet auto-reveal are meant to be the only ways names get revealed automatically — leaving NND's own reveal toggles on will reveal names through those paths too.

The plugin creates `Data/SKSE/Plugins/NNDDynamicReveal.ini` on first run. Change `sRevealKey` to remap the hotkey.

## Building

Requires Visual Studio 2022 ("Desktop development with C++" workload), CMake, and [vcpkg](https://github.com/microsoft/vcpkg) with `VCPKG_ROOT` set. cmake must be run from a VS Developer Command Prompt.

`NPCsNamesDistributor/extern/` (CommonLibSSE, CLibUtil, SKSEHooking) is vendored directly in this repo rather than pulled via git submodules, since the working copies here carry local fixes not present on their public branches.

**Plugin:**
```powershell
cd NND-DynamicReveal
cmake --preset release
cmake --build build/release
# Output: NNDDynamicReveal.zip, written to the repo root
```

**NND fork** (only needed to change NND behaviour):
```powershell
cd NPCsNamesDistributor
cmake --preset vs2022-windows-vcpkg-ae
cmake --build buildae --config Release
# Copy buildae/Release/NPCsNamesDistributor.dll → NND-DynamicReveal/vendor/
# Then rebuild the plugin.
```

Always use the `-ae` preset, not `-se` — the game runtime is the Anniversary Edition (1.6.x), and an `-se` build silently fails to export the version metadata modern SKSE requires, so SKSE refuses to load it at all (no crash, no error — it just never loads).

## Repository layout

```
NND-DynamicReveal/      SKSE plugin source
  plugin.cpp            all plugin logic (~350 lines)
  KeyCodes.h            hotkey name → DirectInput scan code
  NND_API.h             NND modder API (verbatim copy)
  vendor/               prebuilt NPCsNamesDistributor.dll

NPCsNamesDistributor/   NND fork source
  src/Hooks.cpp         key fix: line 461, dialogue reveal respects MCM
  src/ModAPI.cpp        RevealName() — kDialogue bypasses MCM, kGreeting respects it
```

## Technical details — how it works

### NND fork

`NPCsNamesDistributor/src/Hooks.cpp`, the `Character_SetDialogueWithPlayer` vtable hook (~line 461): the original condition revealed a name whenever vanilla dialogue started, regardless of NND's "Reveal on greeting" MCM setting, as long as it was player-initiated:

```cpp
if (a_this && inDialogue && (Options::Obscurity::greetings || !forceGreet)) {
```

This fork drops the `|| !forceGreet`, so vanilla dialogue always respects the toggle:

```cpp
if (a_this && inDialogue && Options::Obscurity::greetings) {
```

### SkyrimNet auto-reveal

`NND-DynamicReveal/plugin.cpp` integrates with SkyrimNet's native plugin API by dynamically resolving `PublicRegisterDecorator` and `PublicRegisterEventCallback` from `SkyrimNet.dll` via `GetModuleHandleA`/`GetProcAddress` — no compile-time SkyrimNet SDK dependency.

- **`nnd_name` decorator** — exposes an actor's real (un-obscured) NND name to SkyrimNet's prompt templates, so the AI can know and choose to say its own name in dialogue.
- **`dialogue` event callback** — on every SkyrimNet dialogue event, the originating actor's FormID and the spoken text are pulled out of the event JSON via lightweight string parsing. The handler queues a pending reveal, consumed on the next input event on the main thread (actor calls aren't safe off the main thread).
- **Name matching** (`DialogueMentionsName`) — splits the actor's name into words, keeps only those 4+ characters long, and checks case-insensitively whether any appear in the spoken text. NND's `GetRawName()` returns empty for actors that keep their original vanilla name — NND only stores a name of its own for actors it generated one for (see `NNDData::UpdateDisplayName` in the fork) — so the matcher falls back to `actor->GetName()` in that case.
- On a match, `RevealName(actor, NND_API::RevealReason::kDialogue)` is called — `kDialogue` bypasses all of NND's MCM gating by design, since this path is meant to fire unconditionally whenever SkyrimNet auto-reveal is enabled (`[SkyrimNet] bAutoReveal` in the plugin's own ini).
