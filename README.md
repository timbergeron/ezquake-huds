# ezquake-huds

HUD configs for [ezQuake](https://github.com/ezQuake/ezquake-source).

No artwork is included here — see [Artwork](#artwork) for why, and for the two ways
to supply it.

## hud_qrack.cfg

A two-row status bar (sbar + ibar) based on the woods/qrack layout, reworked so that
every element is anchored to its parent group rather than to absolute screen pixels.
It holds its layout at any `vid_conwidth` and across windowed/fullscreen.

### Install

1. Copy `hud_qrack.cfg` to `<quakedir>/ezquake/cfg/hud_qrack.cfg`
2. Add this line to `<quakedir>/ezquake/autoexec.cfg` (create the file if absent):

   ```
   exec cfg/hud_qrack.cfg
   ```

That second step is not optional. `config.cfg` is read during `ConfigManager_Init()`
early in startup and carries a saved snapshot of *every* `hud_*` cvar. `autoexec.cfg`
is exec'd later, near the end of `Host_Init`, so it is the exec that makes this file
authoritative. Without it your saved config silently wins and the HUD reverts.

## Artwork

The config references two group backgrounds:

```
hud_group1_picture "sbar"
hud_group2_picture "ibar"
```

ezQuake resolves those through `HUD_GROUP_PIC_BASEPATH`, which is `"gfx/%s"` — so it
looks for **`gfx/sbar.*`** and **`gfx/ibar.*`**. If they are missing the bar renders as
a plain dark strip and the console prints:

```
Couldn't load picture sbar for 'hud_group1_picture'
```

A common mistake is putting them in `textures/wad/`. That path is real, but it is the
24-bit replacement for the *classic* status bar drawn by `Draw_CacheWadPic` — the
`scr_newhud 0` code path. Group backgrounds never look there.

Group pictures must be image files. `hud_groups.c` calls
`Draw_CachePicSafe(path, false, true)`, and that last argument is `only24bit`, so a raw
WAD lump or `.lmp` will not load for a group background.

### Option A — a 24-bit HUD pack

Drop any ezQuake HUD texture pack into `<quakedir>/ezquake/`, then make sure the two bar
images also exist at `gfx/sbar.tga` and `gfx/ibar.tga`. Most packs ship them only under
`textures/wad/`, so copy them across:

```sh
cd <quakedir>/ezquake
mkdir -p gfx
unzip -o -j yourpack.pk3 "textures/wad/sbar.tga" "textures/wad/ibar.tga" -d gfx/
```

### Option B — stock Quake bars

The layout also fits the original artwork exactly. Extract the `SBAR` and `IBAR` lumps
from `gfx.wad` inside your own `pak0.pak`, convert them with the Quake palette, and save
them as `gfx/sbar.tga` and `gfx/ibar.tga`. They are 320x24 each.

No position changes are needed — the qrack replacement art was painted over the classic
geometry. Measured in group coordinates:

| feature | stock | qrack pack |
|---|---|---|
| ibar divider | y 10.5 | y 10.5 |
| ibar slot seams (x) | 0.0, 24.0, 49.1, 74.1, 99.2, 124.2, 149.3, 199.4 | 0.8, 24.8, 49.8, 74.9, 100.2, 125.2, 149.3, 199.9 |
| sbar armor slot | x 0.0–25.0 | x 0.3–25.8 |
| sbar face slot | x 116.9–141.9 | x 117.4–141.2 |
| sbar ammo slot | x 233.8–258.8 | x 236.4–256.2 |

## Layout notes

The bar is 334 units wide and centred, so console widths below about 340 will clip it.
`vid_conwidth` / `vid_conheight` do not need pinning — but if you do pin them, set
**both**. With only `vid_conwidth` set, `vid_conheight` is derived from the live window
aspect ratio, so it still shifts between windowed and fullscreen.

Weapon icons sit in the ibar slot row via `group6`, which is anchored to `group2` at
(1, 10) at a 25-unit pitch. Frag cells use original Quake's geometry — 32-unit pitch, no
gap, four cells, right edge 2 units in — drawn above the ibar, which is both where stock
Quake puts them (`vid.height - SBAR_HEIGHT - 23`) and the only place they fit, since the
ammo counter occupies `group1` x 260–334.

### Known limitation

Holding Tab in single player hides the whole HUD and draws the engine's own solo
scoreboard in its place. Both behaviours are hardcoded: `hud.c` returns early for any
element without the compile-time `HUD_ON_SCORES` flag, and `Sbar_SoloScoreboard()` draws
the level name at y=4 with kills/skill/secrets at y=12 — 8 apart with an 8-tall font at
scale 1, so the two lines sit flush against each other. No cvar gates either, and nothing
in this config can move them.

## Credits

Layout derived from the woods/qrack ezQuake HUD. Original author unknown — if that is
you, open an issue and I will credit you properly.

The config in this repository is free to use and modify. Any artwork you pair it with
remains the property of its respective owner; Quake game data from `pak0.pak` is
copyright id Software and is not redistributable.
