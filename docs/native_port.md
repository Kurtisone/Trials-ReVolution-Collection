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

The library is a newer build than Skate's: matching Skate's functions with addresses masked mostly fails. Partial matches
with few candidates put the render state setters at `826C3FE8` to `826C6100`, for instance `826C50D8` (30 of 32
instructions, Skate's `82CB8558`, pixel center) and `826C5140` (28 of 32, index restart). The game's interrupt callback is
`826C3270`. The rest has to be found from the registers each function writes.

## Order of work

1. Name the D3D entry points in that range: draws, shaders, textures, render targets, resolve, swap.
2. Read the engine's shader files, or capture every shader when it is created, for the whole set.
3. Choose the recompiler path above, and render the title screen from a frame capture.
4. Memory export: the deciding step, without which tracks have no textures.
