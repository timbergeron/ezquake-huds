# ezquake-huds

HUDs for [ezQuake](https://github.com/ezQuake/ezquake-source), as loose configs and as
drop-in packages.

| File | What it is |
|---|---|
| `pk3/qrack.pk3` | qrack HUD with 24-bit artwork (bars, numbers, faces, weapon and item icons) |
| `pk3/qrack_lmp.pk3` | Same layout with stock Quake graphics: only the two bar backgrounds are shipped |
| `pk3/qwnu.pk3` | The [QuakeWorld Hub](https://hub.quakeworld.nu) HUD, including its team overlay |
| `hud_qrack.cfg` | The qrack layout on its own (identical inside both qrack pk3s) |
| `hud_qwnu.cfg` | The hub layout on its own (identical to the one in `qwnu.pk3`) |

Install **one** qrack package, not both. They share `cfg/hud_qrack.cfg`,
`gfx/sbar.tga` and `gfx/ibar.tga`, and whichever archive the engine loads last wins.

## Why the autoexec line matters

Every package needs an `exec` line in `<quakedir>/ezquake/autoexec.cfg`. `config.cfg` is
read during `ConfigManager_Init()` early in startup and carries a saved snapshot of
*every* `hud_*` cvar. `autoexec.cfg` is exec'd later, near the end of `Host_Init`, so it is
what makes the HUD file authoritative. Without it your saved config silently wins.

Archives also take precedence over loose files in the same gamedir: `FS_AddPak` prepends
each pk3 ahead of the directory in the search path. With a pk3 installed, a loose copy of
the same `cfg/` file is ignored, so edit one or the other.

---

## qrack

A two-row status bar (sbar + ibar) based on the woods/qrack layout, reworked so every
element is anchored to its parent group rather than to absolute screen pixels. It holds its
layout at any `vid_conwidth` and across windowed/fullscreen.

### Install

1. Drop `qrack.pk3` **or** `qrack_lmp.pk3` into `<quakedir>/ezquake/`
2. Add to `<quakedir>/ezquake/autoexec.cfg`:

   ```
   exec cfg/hud_qrack.cfg
   ```

### Bar artwork

The config references two group backgrounds:

```
hud_group1_picture "sbar"
hud_group2_picture "ibar"
```

ezQuake resolves those through `HUD_GROUP_PIC_BASEPATH`, which is `"gfx/%s"` — so it looks
for **`gfx/sbar.*`** and **`gfx/ibar.*`**. If they are missing the bar renders as a plain
dark strip and the console prints `Couldn't load picture sbar for 'hud_group1_picture'`.

A common mistake is putting them only in `textures/wad/`. That path is the 24-bit
replacement for the *classic* status bar (`Draw_CacheWadPic`, the `scr_newhud 0` path);
group backgrounds never look there. `qrack.pk3` ships the bars in both places on purpose.

Group pictures must be image files. `hud_groups.c` calls
`Draw_CachePicSafe(path, false, true)`, and that last argument is `only24bit`, so a raw WAD
lump or `.lmp` will not load. That is why `qrack_lmp.pk3` contains the stock `SBAR` and
`IBAR` lumps converted to TGA with the Quake palette — pixel-identical to the originals,
with every other status bar graphic falling through to `gfx.wad` as normal.

The layout fits both art sets without changes; the qrack art was painted over the
classic geometry. Measured in group coordinates:

| feature | stock | qrack pack |
|---|---|---|
| ibar divider | y 10.5 | y 10.5 |
| ibar slot seams (x) | 0.0, 24.0, 49.1, 74.1, 99.2, 124.2, 149.3, 199.4 | 0.8, 24.8, 49.8, 74.9, 100.2, 125.2, 149.3, 199.9 |
| sbar armor slot | x 0.0–25.0 | x 0.3–25.8 |
| sbar face slot | x 116.9–141.9 | x 117.4–141.2 |
| sbar ammo slot | x 233.8–258.8 | x 236.4–256.2 |

### Layout notes

The bar is 334 units wide and centred, so console widths below about 340 will clip it.
`vid_conwidth` / `vid_conheight` do not need pinning — but if you pin them, set **both**.
With only `vid_conwidth` set, `vid_conheight` is derived from the live window aspect ratio.

Weapon icons sit in the ibar slot row via `group6`, anchored to `group2` at (1, 10) at a
25-unit pitch. Frag cells use original Quake's geometry — 32-unit pitch, no gap, four
cells, right edge 2 units in — drawn above the ibar, which is both where stock Quake puts
them (`vid.height - SBAR_HEIGHT - 23`) and the only place they fit, since the ammo counter
occupies `group1` x 260–334.

### Known limitation

Holding Tab in single player hides the whole HUD and draws the engine's own solo scoreboard
in its place. Both behaviours are hardcoded: `hud.c` returns early for any element without
the compile-time `HUD_ON_SCORES` flag, and `Sbar_SoloScoreboard()` draws the level name at
y=4 with kills/skill/secrets at y=12, flush against each other. No cvar gates either.

---

## qwnu — the QuakeWorld Hub HUD

### Install

1. Drop `qwnu.pk3` into `<quakedir>/ezquake/`
2. Copy `fonts/Roboto-Bold.ttf` out of the pk3 into a real folder, e.g.
   `<quakedir>/ezquake/fonts/`
3. Add to `<quakedir>/ezquake/autoexec.cfg`, in this order:

   ```
   sys_fontsdir "/full/path/to/quakedir/ezquake/fonts"
   exec cfg/hud_qwnu.cfg
   ```

Step 2 exists because ezQuake opens TrueType fonts with FreeType straight from
`<sys_fontsdir>/<name>.ttf` on disk (`fonts.c`, `FT_New_Face`). It cannot read a font from
inside a pk3, and `$gamedir` expands to a directory name rather than a path, so
`sys_fontsdir` has to be absolute and set per machine. If it is wrong the HUD still works;
the team overlay falls back to the normal Quake charset.

Location names in the team overlay need `.loc` files in `<quakedir>/qw/locs/`. The hub
fetches them from `https://assets.quake.world/maps/<map>.loc`.

### Where each part comes from

| Part | On the hub | Here |
|---|---|---|
| Bottom HUD | [`config_qtv_v5.cfg`](https://github.com/quakeworldnu/hub.quakeworld.nu/blob/main/public/assets/fte/config_qtv_v5.cfg), run by FTE's ezhud plugin (`plug_sbar 3`) | The same `hud_*` block, with its FTE variables resolved (margin 12, nmargin -12, weapon scale 1.5) |
| Team overlay | HTML — [`PlayerInfo.tsx`](https://github.com/quakeworldnu/hub.quakeworld.nu/blob/main/src/pages/games/player/controls/PlayerInfo.tsx), Roboto Bold 14px | `hud_teaminfo` with a TrueType font |
| Top score box | HTML — [`Participants.tsx`](https://github.com/quakeworldnu/hub.quakeworld.nu/blob/main/src/pages/games/player/controls/Participants.tsx) | `hud_teamfrags`, horizontal, names outside |
| Artwork | [`qw-ctf/qtube-assets`](https://github.com/qw-ctf/qtube-assets), branch `assets` | The same 143 HUD files |

### ezQuake-specific adjustments

| Setting | Why |
|---|---|
| `cvar_reset_re ^hud_` | Clean slate; otherwise elements this file does not mention keep stale values from `config.cfg` |
| `hud_gunN_scale 3` | FTE sizes WAD replacements by the image (48x32); ezQuake keeps the lump size (24x16). The hub's 1.5 in FTE is 3.0 here |
| `font_gradient_*` flat white | ezQuake bakes a gradient into TrueType glyphs (digits yellow to orange by default), which would tint every coloured value |
| `font_gradient_alternate_*` `179 127 58` | The hub's bracket colour. Brackets are the high-bit glyphs `$xdb` / `$xdd`, which use this gradient |
| `tp_name_rl` / `lg` / `rlg` / `sng` / `ssg` | Best-weapon text in the hub's colours. These also change how teamplay macros print weapon names; the hub config does the same |
| `hud_teaminfo_loc_width 3` | ezQuake clips at the cell edge; three cells hold about five Roboto characters, matching the hub's `substring(0, 5)` |
| `hud_teamfrags_fliptext 2` | Names on the outside: `blue [46] [57] red` |
| `hud_sortrules_includeself 2` | Own team in the right-hand slot, as on the hub |

### Known differences from the hub

These are limits of `hud_teaminfo`, not settings:

- **Wider team overlay.** teaminfo lays text out in fixed character cells even with a
  proportional font, so the panel is roughly 1.6x the hub's width at the same text size.
- **Team headers** show the name at the left and a small red dot plus the frags at the
  right, instead of `name [46]` in a coloured box. teaminfo draws headers as plain text,
  and a separate score box cannot follow the second header because its position depends
  on team size.
- **Player names are right-aligned**; the hub left-aligns them.
- **No highlighted row** for the player being tracked.
- **Face slot:** the hub shows a red `+`. That image is not in the hub's asset pack or any
  qw-ctf repository, so the pack's own face is shown instead.

---

## Credits and provenance

- qrack layout derived from the woods/qrack ezQuake HUD. Original author unknown — if
  that is you, open an issue and I will credit you properly.
- qwnu layout from the QuakeWorld Hub
  ([quakeworldnu/hub.quakeworld.nu](https://github.com/quakeworldnu/hub.quakeworld.nu)),
  with base `hud_teamfrags` settings from vikpe's
  [qw-streambot-ezquake](https://github.com/vikpe/qw-streambot-ezquake).
- qwnu artwork from [qw-ctf/qtube-assets](https://github.com/qw-ctf/qtube-assets), which
  assembles community packs from [gfx.quakeworld.nu](https://gfx.quakeworld.nu) — deurk's
  HUD, the "faithful" icon sets and dithe's HUD among them.
- Roboto by Google, Apache License 2.0; the licence ships inside `qwnu.pk3`.
- `qrack_lmp.pk3`'s two bar images are the `SBAR` and `IBAR` lumps from id Software's
  `pak0.pak`, converted to TGA. Quake game data is copyright id Software.

The configs are free to use and modify. Artwork remains the property of its respective
owners.
