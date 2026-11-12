Setup: rel `ps5-v0.1.0-beta.2` (`275b987-dirty`), PS5_Mesa `0f86253`, all switches force set with `setenv` in `hos_main.c` after the defaults loop with RX_VK_FORCE_MESH_FALLBACK being the most important one.

**Baseline**
For reference, this is how every run ended before, on the first 3D frame after the movies:
```
[Hookshot] radv/ps5: the GPU has not reached submission 20133 (at 20131) in 10 s
### suspendPoint() is not called for 45 seconds (suspendPoint is waiting for previous)
# exception: 0xa0d0c00c (GPU_FAULT_SUSPENDPOINT_TIMEOUT_IN_RUN_ASYNC)
```
`RX_VK_SKIP_CODE`, `RX_VK_SKIP_PROG` and `RX_VK_MESH_NOCULL` did not fix it. `RX_VK_NO_MESH=1` got past it, but without the geometry.

**Mesh fallback no hang/console crash with a bit more geometry compared to NO_MESH**
```c
setenv("RX_VK_FORCE_MESH_FALLBACK", "1", 1);
setenv("HOS_DYNRES", "0", 1);
setenv("HOS_PS5_CPU", "1", 1);
```

```
[Hookshot] [hos host] vk: no mesh shaders: task and mesh programs run as compute shaders (issue 640 fallback)
```

With this the forest plays, geometry included, for about five minutes with no GPU fault:

```
[Hookshot] [hos Presentation Thread] vk: 6480 presents, 30.0 fps; a frame: 1051 draws (0 skipped), 97 dispatches, 142 passes, ...
[Hookshot] [ps5] cpu: 448% of one core (user 360%, system 88%) over 5.0 s; guest pages 4975 MiB; heap 955 MiB (peak 1298); stacks 22 MiB in 89; direct memory 12016 MiB, largest free piece 3218 MiB
[Hookshot] [ps5] direct memory: 8492 MiB taken, 3523 free in 247 pieces; the driver's buffers 2326 MiB; ...
```

```
[Hookshot] [hos Presentation Thread] vi: res: fixed 1 (scale 1.000), +0xe20 30000458802, +0xe2c 0, +0xe50 1080 1440 810, 3D scale 0.750
```

**Area load runs out of memory**
In the same run the game's pages went from about 5.1 to 7.2 GB within roughly 15 seconds at an area load, and the console was left with a few hundred MiB in small pieces:

```
[Hookshot] [ps5] direct memory: 11597 MiB taken, 418 free in 436 pieces; the driver's buffers 3016 MiB; ...
[Hookshot] [hos host] vk: program 15e2ee930ba40899 not translated: std::bad_alloc
[Hookshot] [hos Presentation Thread] vk: a draw without a program (frame 12881); its code: 0:266a4d90df555008 1:4ef357996e294b9e
[Hookshot] [hos Presentation Thread] vi: 14701 frames presented (0.1 fps over the last 40.2 s, interval 1)
```

So after that point shaders fail to translate, their draws are dropped (missing geometry) and the game stalls. No GPU fault in this state.

**Texture budget experiments**
Loading a save made in that area goes straight to the heavy state. I tried `RX_VK_VRAM_MB` both ways. It is the texture budget (default here 1151 MB), not the total.
With `RX_VK_VRAM_MB=2048` (higher than default, so worse):
```
[Hookshot] [hos Presentation Thread] vk: VRAM: 648 MB used of 2560 MB allowed (0 MB of it free in the blocks); textures 109 MB (budget 2048 MB), 128 textures
[Hookshot] radv/ps5: 64 MiB in 3 pieces (no single free range)
[Hookshot] [hos Presentation Thread] vk: memory short (193 MB free, 1024 kept for the game): ASTC textures made at 32 texels at most for now (400 so far, 0 made again at their own size)
[ps5] operator new: no memory: waiting for some
[Hookshot] [ps5] direct memory: 11890 MiB taken, 125 free in 312 pieces; the driver's buffers 3462 MiB; ...
# reason:  abort is called(system)
# thread name: Presentation Th
```

With `RX_VK_VRAM_MB=768` it looked and ran a bit better for a while, then the same thing:
```
[Hookshot] [hos Presentation Thread] vk: image blocks: 33 blocks 1056 MB, 920 images 604 MB; 220 free ranges, the largest 31 MB
[Hookshot] [hos Presentation Thread] vk: VRAM: 2383 MB used of 2560 MB allowed (0 MB of it free in the blocks); textures 1209 MB (budget 0 MB), 1160 textures
[Hookshot] [hos Presentation Thread] vk:   288 MB: 0x54680a000 12288x12288x1 x1 levels 1 NVN 0x32 rendered, last used 1 presents ago
[Hookshot] [ps5] cpu: 187% of one core (user 181%, system 6%) over 5.0 s; guest pages 7360 MiB; heap 1165 MiB (peak 1231); stacks 22 MiB in 89; direct memory 12016 MiB, largest free piece 4 MiB
[Hookshot] [ps5] direct memory: 11945 MiB taken, 70 free in 299 pieces; the driver's buffers 3245 MiB; ...
[Hookshot] libc++abi: terminating due to uncaught exception of type std::bad_alloc: std::bad_alloc
```

The lower budget did not give memory back: under pressure the budget reads 0 and 1.2 GB of textures are still in use. What it did show is about 450 MB stranded inside the image blocks (1056 MB of blocks for 604 MB of images). `RX_VK_TEX_MAX_DIM=256` was set in all of these runs.

**For the mesh fallback**
`RX_VK_FORCE_MESH_FALLBACK=1` is the only thing that fixed the hang while keeping the geometry, and `RX_VK_NO_MESH=1` avoids it too, so I believe the native task/mesh path is what stalls the GPU on base hardware. 

**My guess is that the base GPU does not support mesh shaders the way the driver assumes from the Pro, but I cannot prove the hardware reason.
If the driver could tell the two consoles apart, defaulting base consoles to the fallback would make the release boot there.**

**For the memory budget**
My console reports 12,016 MiB of direct memory, against the 13.2 GB the release notes are tuned for. In the area that fails, the game's pages alone are about 7.2-7.4 GB, the driver's buffers 3.0-3.5 GB and the heap about 1.0-1.2 GB, which is the whole pool. Texture settings did not close that gap for me. Anything that returns memory at area loads (the zero-only game pages, textures of finalized pools, the space lost inside image blocks) would matter more on base hardware than on the Pro. 
