# NvToolKit

A single Windows exe for poking at the NVIDIA driver: profile settings, the
hidden registry DWORDs, and what the card is actually doing. No installer and
nothing to unpack. Download `nv_toolkit.exe` and run it.

You need Windows 10 or 11 (x64) and a recent NVIDIA driver. Anything that
writes to the registry needs the app running as administrator.

## How it's different from Profile Inspector

NVIDIA Profile Inspector is a great tool and NvToolKit started from the same
idea, so the D3D Settings tab will feel familiar. It also reads and writes the
same `.nip` files. The difference is in how far each one goes.

- **Profile Inspector only edits driver profiles.** Those are per-game
  settings stored in the driver's profile database. NvToolKit does that too,
  and it also covers things that live somewhere else:
  - the kernel driver's registry keys (DWORDs)
  - hardware information and per-process VRAM
  - an overclocking tab that's on its way
- **Many more settings have real names.** The reference database has about
  3,100 driver settings, each with a name, a description, its possible values
  and the driver version it appeared in, sorted into ten groups. Many of them
  are ones Profile Inspector only shows as an unknown hex id, if it shows
  them at all.
- **Bitmask settings get a proper editor.** You get 32 checkboxes instead of
  having to type a hex value.
- **You can compare two `.nip` files** side by side, which Profile Inspector
  can't do.
- **It doesn't need .NET.** It's a native exe, so it runs on a clean Windows
  install as is.

If all you want is to flip one setting for one game, either tool will do.
NvToolKit is for when you want to see everything the driver exposes.

## What's in it

### D3D Settings

This is the profile editor. On the left you pick a profile: Global, or the
profile of a specific game or program (there's a search box). The middle
shows that profile's settings, and the panel on the right explains the
selected setting: its group, its internal name, and what each value does.

Changes are staged first. You see what is pending, you can revert a single
setting, and nothing reaches the driver until you hit Apply. Restore Defaults
resets every profile in the driver, not just the one on screen, to how NVIDIA
ships them. Export a `.nip` first if you want to keep what you had.

Settings are split into groups, and you can show or hide any of them:

- **0 - Misc.** Everything that doesn't fit anywhere else. Many of these are
  workarounds NVIDIA added for specific games or hardware bugs.
- **1 - APIs.** This is the biggest group. It holds behaviour specific to
  DirectX, OpenGL, Vulkan, CUDA and OpenCL: shader cache, threading, how
  each API submits work to the GPU.
- **2 - Performance / Power.** Frame rate limiters, low-latency mode, power
  management, and the latency indicator overlay. This is the group most
  people tune for competitive games.
- **3 - Display / Stereo.** G-SYNC and VRR, how frames get flipped to the
  screen, overdrive on laptop panels, and the old 3D Vision stereo options.
- **4 - SLI / Multi-GPU.** These cover SLI rendering modes and multi-GPU
  behaviour. They're mostly of historical interest today.
- **5 - RTX / AI.** DLSS overrides and presets, frame generation, and the ray
  tracing core options.
- **6 - Capture / Stream.** ShadowPlay/NVFBC capture, Ansel, and the GeForce
  Experience integration flags.
- **7 - Driver internals.** These are the switches behind the Control Panel's
  own features (antialiasing, anisotropic filtering, ambient occlusion,
  battery boost), along with debug dumps.
- **8 - GPU Architecture.** These options only apply to one GPU generation
  (Maxwell, Ampere, Ada, Hopper), usually cache behaviour or a hardware bug
  workaround.
- **9 - Anti-Aliasing.** The AA modes, the transparency AA and gamma
  correction options, and how AA is forced or replayed.

Settings with no group in the database end up under Uncategorized, and
the ones NVIDIA flags as hidden end up under Hidden / Internal.

At the top there are `.nip` import and export. You can import one profile
into pending (so you can look at it before applying) or apply every profile
in the file at once. You can export only what you changed, every setting of
the current profile, every profile you've customized, or the whole driver
database. You can also drag and drop a `.nip` onto the window.

### Compare

Load two `.nip` files side by side and filter by settings that match, settings
that differ, or settings missing from one side. It's handy when someone sends
you their "competitive" profile and you want to know what it actually changes.

### DWORDs

These are settings for the kernel driver itself (`nvlddmkm.sys`), not for any
one game. They control things like power and clock gating, PCIe link power
saving, how the GPU schedules and preempts work, and a lot of display
behaviour. None of them show up in the NVIDIA Control Panel or in a profile,
and they apply to the whole GPU, all the time. That's why tweak guides love
them, and why they deserve care.

The real problem with those guides is that a list of keys is copied from
driver to driver for years. NVIDIA renames and removes keys between releases,
and a key the driver no longer reads does nothing at all. So before showing you
anything, NvToolKit scans the `nvlddmkm.sys` installed on your machine and
marks which keys your driver really reads. The rest of the list (about 900
documented keys) is there for reference, each with what it does and which
registry path it belongs to.

A wrong value here can do more than cost a few FPS. It can leave the driver
unable to start. NvToolKit saves the original value of everything it changes
before writing, and you can undo it from the app, one key at a time or all
at once. It also keeps an up-to-date `NvToolKit-restore.reg` in
`%LOCALAPPDATA%\NvToolKit`. If you ever boot to a black screen, start Windows
in Safe Mode, double-click that file and reboot.

Changes take effect after a reboot (or a driver restart).

### Info

Here you'll find the GPU, memory type and bus width with the bandwidth
worked out from them, PCI ids, VBIOS version and hash, driver version, the
current performance state, and a per-process VRAM table.

## OC (coming back)

The overclocking tab exists and works, but it's switched off in this build
while we finish testing it on more cards. So far it has been checked on a
GTX 1650, RTX 3080 Ti, RTX 4090 and RTX 5090, and they behave differently
enough that we'd rather not ship it half-checked.

When it's back it will have:

- **Core and memory offsets.** These are the usual sliders and the part most
  people will ever touch.
- **Per-domain clocks.** You get separate offsets for GPC, XBAR, memory, SYS
  and NVD, shown only for the domains your card actually lets you change. On
  Blackwell the XBAR offset moves the clock directly. On Ampere it doesn't,
  so there we go through the GPC→XBAR ratio instead.
- **The V/F curve.** You can view and shift the voltage/frequency curve.
- **Voltage rails.** NVVDD (graphics) and MSVDD (memory side, and XBAR on most
  cards) are shown with their live voltage and min/max limits.
- **Power and temperature limits.** These include the normal power target and
  the per-rail power channels (CORE and MEM) on cards that expose them.
- **Fans.** You can set a custom temperature curve or a fixed manual speed.
- **A diagnostic report export,** so you can send us something useful when a
  card misbehaves.

None of it will be a one-click "max everything" button. Every change is
applied, measured under load, and can be reset. Clock and voltage writes stay
in the driver until you reset them or reboot.

## Command line

The same exe has a headless mode for the stuff that doesn't belong in a
window, like read-only probes, dumps and measured tests. Run
`nv_toolkit.exe help` to see the list. For example:

```
nv_toolkit.exe clocks      measure every clock domain once
nv_toolkit.exe limits      what is holding the clocks down right now
nv_toolkit.exe report      everything safe, in one file, to send to someone
```

Some commands write to the GPU. They say so, and they restore what they
changed when they finish.

## If it crashes

The app writes `crash.log` to `%LOCALAPPDATA%\NvToolKit` and tells you where
it is before closing. Send us that file along with your GPU and driver
version.

## Credits

Made by [@Ixeoz_](https://x.com/Ixeoz_) and [@P467121](https://x.com/P467121).
Thanks to everyone who ran test builds on their cards and sent logs back.

Use at your own risk. Nothing here is supported or endorsed by NVIDIA.
