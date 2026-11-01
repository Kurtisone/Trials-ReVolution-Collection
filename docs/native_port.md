# Native renderer for Trials Fusion

The aim, as in Skate1Native: no Xenos emulation. Skate takes the game's D3D calls; Fusion has to use its command stream (see below). Not started in code. This is
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
The device is passed in r3.

Traced on the first track at 20 FPS, by function. "Sure" means the arguments fit Skate's function.

| Function | Calls per frame | What | Confidence |
| --- | --- | --- | --- |
| `82228DB0` | 1,394 | DrawIndexedVertices: device, 4 (triangle list), base vertex, start index, index count; holds the draw packets | sure |
| `8222FDF8` | 51 | DrawVertices: device, primitive type, start, count; holds the draw packets | sure |
| `822294B0` | 1,396 | SetIndices | likely |
| `82226160` | 58 | Resolve, best match of Skate's `82CC7FF0`, holds a draw packet | likely |
| `8220DD40` | 4 | ClearF, best match of Skate's `82CC7510` | likely |
| `82221AB0` | 150 | SetVertexShader, best match of Skate's `82CBE2E8` | guess |
| `826C91A0` | not traced | SetPixelShader, best match of Skate's `82CBDFE8` | guess |
| `826DA710` | 1 | SetPredication, best match of Skate's `82CC4E78` | likely |
| `82229228`, `8222DA80`, `822287C8` | 1,382, 940, 937 | Engine's draw submission, callers of the above | seen |
| `826CB7B8` | 1 | Swap, device `40852A00`, front buffer `1F830000` in a callee | sure |
| `826C3270` | 1 | Interrupt callback | sure |
| `826C2780` | 2 million+ | Spin wait of the Render thread (caller `8225B138`) | seen |

So draws go through D3D as in Skate, and hooking them is possible. An earlier version of this file said the opposite: it
had traced only the `826C` part. `BeginTiling` has no match.

Draw packet headers have the predicate bit set here (`C0032201`, `C0012201`, `C0003601`), which an early scan for them
missed. Other functions holding them: `822272B8`, `82208F18`, `826C2E00`, `826DA130`, `826D6698`.

The frame's commands: one run of about 12 segments, each executed once: 4 long ones of about 16,000 words and shorter ones,
in a ring from `1FA90000` to `1FC90000`, written on the Render thread by `82228DB0` and `8222DA80`, with 42 small buffers from
`1FC93000` and `1FC94980` in between. There is no replay and no sign of predicated tiling, so Skate's single tile shortcut
does not apply. An earlier version of this file claimed a replay: it had grouped buffers by address prefix.

The small buffers come in 14 groups of 24 `DRAW_INDX_2`, 336 draws. Known draws add up to about 1,840 a frame against the
2,500 to 3,400 the GPU counted, on other frames and on the Odin. That gap is not explained yet. Hardware watchpoints on the
small buffers, set from the menu through track loading, caught no writes.

## What Skate's renderer needs, and what Fusion's device holds

Skate's native layer takes about ten hooks (draws, clear, resolve, tiling, swap, inline vertices). Everything else it reads
at each draw from the D3D device.

The device was `40852A00` in the runs that dumped it; `40852600` appears in one trace, so it can move between runs. Take it
from r3 at the hook. `82228DB0` flushes state exactly as Skate's draw does: dirty masks at `0` to `40`, registers from the
same offsets, constants from `1920` and `6016`. Dumped with gdb, stopped on a real draw on the first track:

| Data | Skate offset | Fusion | State at a draw |
| --- | --- | --- | --- |
| Register groups | `10368`, `10444`, `10528`, `10548`, `10596` | same | Filled, plausible values |
| Fetch constants | `0x480` | same | Filled |
| Index buffer | `12428` | `12812` (+384) | Filled |
| Float constants | `0x780`, `0x1780` | same place in the code | **All zero** |
| Shaders | `12684`, `12688` | `13068`, `13072` (+384) | **0** |

So registers, textures and index buffers can be read as in Skate. Constants and shaders are not in the device at a draw: the
engine must set them another way. `82225058`, called from the draw and holding an `IM_LOAD_IMM` packet, is the first thing to
read for shaders. `8222C768` (1,689 calls a frame) and `8222C6D8` (1,637) are candidates for constants: the XDK has calls
that give the caller a pointer straight into the command buffer, which would leave nothing in the device.

## Where the time goes on the Odin 2 Portal

Measured on the first track, Qualcomm driver, 23 FPS, with `vulkan_gpu_timestamp_buckets = true`.

The GPU is the limit. The Adreno is 97 to 99% busy at its top clock of 680 MHz, at 43 degrees, and the Render thread spends
85% of its time in `8225B0A0`, D3D's block on fence, waiting for it.

| Bucket | ms a frame | What |
| --- | --- | --- |
| PassEntry | 17 | Entering 86 render passes: tile loads, and the deferred work of the pass before |
| Main | 7 | The game's own draws |
| TexUpload | 6.3 | Nearly all copies into images |
| Resolves and dumps | 5.5 | |
| RtTransfer | 4.9 | Moving EDRAM contents between render targets |
| Other | 6 | |

Each frame sends 167 render passes, 106 render target transfers, 260 dispatches, 325 barriers and 2,300 draws.

PassEntry is mostly pixel shading, deferred by the tile based GPU to the end of the pass before: with every draw scissored to
the top half of the screen it falls from 17 to 7.7 ms and the frame from 44 to 34 ms, with tile loads and stores left as
they were. So about 24 ms of the frame is the game's own drawing, and EDRAM emulation is at most about 10 to 15 ms of the
rest. An earlier version of this file said the emulation was nearly all of the cost; that was wrong.

### The frame

1. 4x pass: depth at tile 0, `k_16_16` at tile 832, more than 1,000 draws, then 1x G-buffer pass, about 330 draws, on `@494` and `@988`.
2. Five rounds of one or two 4x depth only draws that write stencil, then a 1x full screen light that reads it.
3. Post processing, each step on EDRAM still holding another format.

The surfaces are 1040 x 608 at 1x. Those labelled 4x are 520 x 304 over the same samples: the depth moves between the two
views eight times a frame, 45% of all transfers by area.

`SetVertexShader` is called only with 0 and `SetPixelShader` never on a track. The engine sends shaders and constants itself,
constants with `SET_SHADER_CONSTANTS` (`0x56`) through `8222C6D8`. **So the device cannot describe a draw and hooks on D3D
calls will not work for Fusion, unlike Skate.** That also explains the draw count gap above. The native renderer has to take
its state from the command stream, where the SDK already has every register, constant and shader address.

### Tried, none kept

| Change | Result |
| --- | --- |
| Skip all transfers | 38 FPS on a broken, emptier scene, so it overstates the gain |
| The 4x as 1x test below with the screen scissor overflowing, or a draw skipped | 27 to 30 FPS: false, those draws were not rendered |
| Skip transfers into depth | 33 FPS, black scene |
| Skip colour to colour transfers | 24 FPS, wrong smoke: they are cheap |
| Keep clean copies of depth views | No gain: the stencil rounds really write |
| Draw depth only 4x draws into the 1x surface, viewport doubled | 2 ms, lighter walls |
| Reload big textures once a frame | 41 ms; the rider looked wrong, not cleared up |
| `vulkan_deferred_resolve_clears` | Nothing |
| `vsync = false` | Nothing |
| `readback_resolve = none` | 27 FPS, no textures |
| All 4x surfaces drawn as the 1x surface of twice the size, on registers doubled per draw | 23 FPS, right picture: transfers lose 2 ms, the 4x passes now shade four times the pixels |
| Shaders through `spirv-opt -O`, a third smaller | No change: the driver's compiler already does this |

The virtual texture table (2048 x 2048, 8 MB) is reloaded whole 4 to 5 times a frame, about 73 MB, most of TexUpload.

## Verdict so far

60 FPS on the Odin 2 Portal is out of reach by removing emulation alone. With the 44 ms split as above, a renderer with no
EDRAM emulation at all would still spend about 24 ms on the same draws with the same shaders, about 40 FPS at the very best
and with texture uploads and resolves still to remove. The one lever measured to work is fewer pixels: half the area takes
10 ms off. The game renders at 1024 x 608, and where the engine sets that has not been found: no constant pair in the code.

Next, in order of expected gain for the work:

1. Find the engine's render resolution and lower it, aiming at a steady 30 FPS.
2. Load the virtual texture table only where it changed, not whole 4 to 5 times a frame (4 ms).
3. The native renderer, only if the two above fall short.

Native work left, if taken up:

0. Move the native renderer's entry to the command stream: take draws from the SDK's `IssueDraw`, keep the render targets simple.
2. Shaders: the raw microcode, for the whole set, through the SDK's translator.
3. Render the title screen from a frame capture.
4. Memory export: the deciding step, without which tracks have no textures.

To find what writes a buffer: attach gdb once in game, `watch -l` on the guest address + `0x1A0000000`.
