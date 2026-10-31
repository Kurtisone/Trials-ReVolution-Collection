# Linux and Android

Where the port stands, and what is known not to be proven yet. How to build: [BUILDING.md](../BUILDING.md).

## How it is put together

* **SDK**: `tools/setup_sdk.sh` makes `third_party/rexglue-sdk` from ReXGlue v0.10.0, SDL 3.4.14 and
  `sdk-patches/linux-android`. Those patches are for Linux and Android only: Windows builds keep the SDK BUILDING.md
  describes. They hold three things:
  * the ARM64, Android and Vulkan work of [Skate1Native](https://github.com/Kurtisone/Skate1Native), merged with
    `sdk-patches/0001` to `0021`;
  * the renderer fixes that existed only on D3D12, ported to the SPIR-V translator (below);
  * the Android app folder's name as a build setting.
* **Renderer**: the SDK's Vulkan backend, `render_target_path_vulkan` as in each game's `port_defaults.cpp`.
* **Linux**: the launcher and the three games in one folder, built in a container so that it runs on the Steam Deck.
* **Android**: one APK per game. The launcher stays a desktop program.

## D3D12 fixes on Vulkan

| Patch | Fix | On Vulkan |
| --- | --- | --- |
| 0001, 0009 | Integer number format fetches, rounded only without an exponent bias | `spirv_translator_fetch.cpp` |
| 0002 | Unnormalized coordinates at the fetched mip level | `spirv_translator_fetch.cpp` |
| 0002, 0005, 0011 | Integer destinations and the full -32 to 32 range in 16 and 32 bpp resolves | XeSL sources compiled to SPIR-V, interface checked equal to the stock shaders |
| 0003 | Memory export at an even resolution scale | `spirv_translator.cpp` |
| 0004 | getCompTexLOD reports the console's LOD | `spirv_translator_fetch.cpp` |
| 0011 | k_16_16 render targets keep values below -1 | host render targets and fragment shader interlock |
| 0013 | `ps_position_guest_pixels` | `spirv_translator.cpp` |

Two additions, both from [snufflyyy/trials-hd-recompiled](https://github.com/snufflyyy/trials-hd-recompiled) (MIT):

* the SPIR-V translator read the texture exponent bias from fetch constant word 4; it is in word 3;
* Trials HD sets the integer flag on its 8 bit font textures, which must stay normalized, so
  `texture_fetch_integer_8bpc` is off for that game (the D3D12 path scales them).

That project's Vulkan fast paths for Trials HD (`vulkan_trials_fast_path`) are not in: they are a large performance
change that conflicts with the Android work. Look there first if Trials HD is too slow.

## What has been checked

* The SDK builds for Linux x86-64 and Android arm64.
* Skate runs on it on Linux (Vulkan, AMD), from the title screen to riding in game, with a new shader cache: same picture
  as before. Vulkan's validation layer reports nothing from start to title screen.
* All the games' own code compiles for both platforms, against stand-in headers for the generated code.
* The launcher runs on Linux. An APK is packaged, signed and read back (without the game's libraries).

## What has not

**No Trials game has run on this port**: the packages were not available when it was written. So not proven:

* code generation and linking with this SDK, on any of the three games;
* every Vulkan fix in the table above: Skate exercises the translator, not these formats;
* `trials_extract` and the package search in `build-linux.sh`, which have only met files that are not packages.

## Expected trouble

* **Trials Evolution on Android**: its title screen and far shadows need render targets emulated exactly. Linux uses
  fragment shader interlock for that; Adreno has none, so host render targets are used, and the "Press START" screen may
  stay black (as it did on D3D12 without ROV).
* **Android performance**: Evolution and Fusion stream virtual textures through memory export and CPU readbacks, costly
  on phone GPUs. They may not reach a playable speed at 720p.
* **Android controls**: a controller is required.
* **Above 720p on Evolution and Fusion**: as on Windows, 720p is the default and the launcher warns about more.
* **Linux packaging**: a folder, not yet the AppImage the launcher's updater expects, so no self-update.
