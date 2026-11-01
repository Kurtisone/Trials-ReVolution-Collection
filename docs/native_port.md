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

The library is a newer build than Skate's and split in two. Part of it sits in `826C0000` to `826E0000`. The part that runs
for each draw was compiled in with the engine's code, around `82220000` to `82230000`, probably by link time optimization.
The device is `40852600`, passed in r3; some functions take `40852A00`, 0x400 further.

Traced on the first track at 20 FPS, by function. "Sure" means the arguments fit Skate's function.

| Function | Calls per frame | What | Confidence |
| --- | --- | --- | --- |
| `82228DB0` | 1,400 | DrawIndexedVertices: device, 4 (triangle list), base vertex, start index, index count | sure |
| `822294B0` | 1,400 | SetIndices, device in r3; best match of Skate's `82CB95A8` | likely |
| `8222FDF8` | not traced in numbers yet | DrawVertices, best match of Skate's `82CB3500` | guess |
| `82221AB0` | | SetVertexShader, best match of Skate's `82CBE2E8` | guess |
| `822230D8` | | SetTexture, best match of Skate's `82CCB4D0` | guess |
| `82229228`, `8222DA80`, `822287C8` | 1,400, 940, 940 | Engine's draw submission, callers of the above | seen |
| `826CB7B8` | 1 | Swap, device `40852A00`, front buffer `1F830000` in a callee | sure |
| `826C3270` | 1 | Interrupt callback | sure |
| `826C2780` | 2 million+ | Spin wait of the Render thread (caller `8225B138`) | seen |

So draws go through D3D as in Skate, and hooking them is possible. An earlier version of this file said the opposite: it
had traced only the `826C` part.

The frame's commands: 4 segments of about 16,000 words at `1FAF0000` to `1FB30000`, written on the Render thread by
`82228DB0` and `8222DA80`, plus 42 small buffers replayed from `1FC93000` and `1FC94980`.

The segments run 2 or 3 times in a frame (`1FB0` to `1FB2` twice, `1FB3` three times, `1FAF` once), the way D3D replays a
scene for each tile with predicated tiling, as in Skate. The small buffers come in 14 groups of 24 `DRAW_INDX_2`, 336 draws.
So 1,400 draws, about twice, plus 336, gives the 2,500 to 3,400 the GPU counts: no draws are missing. The tile functions
are not named yet. This reading comes from one frame's buffer list. Hardware watchpoints on the small buffers, set from
the menu through track loading, caught no writes, so they are written earlier or through another view.

That replay is also a cost on the emulated path. In Skate, forcing the scene out of tiling cut draws from 2,060 to 800 a
frame (`src/scene_tiling.cpp` there). `gpu_force_msaa_1x` does not do this.

## Order of work

1. Name the rest of the state functions around `82220000` to `82230000`, and the tile functions.
2. Shaders: the raw microcode, for the whole set, through the SDK's translator.
3. Render the title screen from a frame capture.
4. Memory export: the deciding step, without which tracks have no textures.

To find what writes a buffer: attach gdb once in game, `watch -l` on the guest address + `0x1A0000000`.
