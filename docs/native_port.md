# Native renderer for Trials Fusion

The aim, as in Skate1Native: no Xenos emulation, the game's D3D calls go to a native renderer. Not started in code. This is
what was measured on `default.xex` 0.0.0.3 before writing any.

## Shaders

Skate's way does not apply. Its shaders are D3DX containers (magic `0x102A110x`), found in its archives and read by
XenosRecomp, which needs their constant table and vertex fetch data.

* No container in `data.pak`, `data_tex.pak` or the decoded executable, and none in 3 GB of the running game's memory.
* The executable never loads that magic as a constant. Skate does in many places. The engine does not use containers.
* The microcode is in physical memory on page boundaries, each right after its literal constants: `c251` of
  `3EBCA95DEB27B0CD` (1.0667, 16, 1, 0.9375) sits just before its code. Files are named `shaders\sd\...`.

So the input to a shader recompiler has to be raw microcode, captured or read from the game's own shader files, with what
a container would have said (constants, interpolators, vertex layout) worked out from the microcode. XenosRecomp would need
that as a new input path, or the SDK's own microcode to SPIR-V translator could feed the native renderer instead.

Of 92 shaders seen from the menus to the first track, 4 use memory export. Neither XenosRecomp nor the native renderer
handle it, and the virtual textures depend on it.

## D3D entry points

Comparing whole functions with Skate's aligns 274 of them, nearly all in `826C0000` to `826E0000`, so that is the library.
A trace of the 985 functions from `826BC000` to `826F0000` on the first track, 20 FPS, gave:

| Function | Calls per frame | What |
| --- | --- | --- |
| `826CB7B8` | 1 | Swap (r3 = device `40852A00`, front buffer `1F830000` in a callee) |
| `826C3270` | 1 | Interrupt callback |
| `826C9D38`, `826C9BE0` | 4.5 | Hold a `DRAW_INDX_2` packet |
| `826C2780` | 2 million+ | Spin wait of the Render thread (caller `8225B138`) |
| `826DF930`, `826DF928` | 1,800 | Small copies, caller `8221B2A0`: not D3D calls |

## Draws do not go through D3D

This is the main difference from Skate. The track sends 2,500 to 3,400 draws a frame to the GPU, and no D3D function runs
anywhere near that often. The `DRAW_INDX_2` headers found in code are all in the library, on those few calls, while
engine functions `82225058` and `822282E0` build `IM_LOAD_IMM` packets themselves. So the engine writes most of its command
stream itself. Where its draw packets are built is not found yet: their headers don't appear as immediates, and
`821FE4A0` and the callers around `8221B2A0` are the first places to read. Replaying prebuilt command buffers would also
fit these numbers, and has not been ruled out.

Hooking D3D as Skate does would therefore miss the scene. A native renderer has to sit on the engine's own draw layer,
which has to be reverse engineered, or read the command stream, which is what the emulator's command processor does.

## Order of work

1. Find what emits the draws: read the callers around `8221B2A0`, `82225058`, `822282E0`, `821FE4A0`.
2. Name the engine's draw layer and the state it sets, as `docs/d3d_functions.md` does for Skate.
3. Shaders: the raw microcode, for the whole set, through the SDK's translator.
4. Render the title screen from a frame capture.
5. Memory export: the deciding step, without which tracks have no textures.
