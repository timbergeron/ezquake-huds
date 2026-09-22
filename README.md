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

Don't combine `qwnu.pk3` with a pack that ships `textures/wad/` replacements, such as
`qrack.pk3`. `Draw_CacheWadPic` tries `textures/wad/<name>` before `gfx/<name>`, so
those images would replace the hub's.

### Where each part comes from

| Part | On the hub | Here |
|---|---|---|
| Bottom HUD | [`config_qtv_v5.cfg`](https://github.com/quakeworldnu/hub.quakeworld.nu/blob/main/public/assets/fte/config_qtv_v5.cfg), run by FTE's ezhud plugin (`plug_sbar 3`) | The same `hud_*` block, with its FTE variables resolved (margin 12, nmargin -12, weapon scale 1.5) |
| Team overlay | HTML — [`PlayerInfo.tsx`](https://github.com/quakeworldnu/hub.quakeworld.nu/blob/main/src/pages/games/player/controls/PlayerInfo.tsx), Roboto Bold 14px | `hud_teaminfo` with a TrueType font, plus `hud_teamfrags` for the team-coloured header boxes |
| Top score box | HTML — [`Participants.tsx`](https://github.com/quakeworldnu/hub.quakeworld.nu/blob/main/src/pages/games/player/controls/Participants.tsx) | Not reproduced — see below |
| Artwork | The hub's own FTE asset set, listed in [`assets.ts`](https://github.com/quakeworldnu/hub.quakeworld.nu/blob/main/src/pages/games/fte/assets.ts) and served from `https://a.quake.world/fte/id1/` | The same 143 files: `gfx/*.png`, the `povo5f_xtm` charset, the `xtm_dot` crosshair, tracker icons |

### ezQuake-specific adjustments

| Setting | Why |
|---|---|
| `cvar_reset_re ^hud_` | Clean slate; otherwise elements this file does not mention keep stale values from `config.cfg` |
| `font_gradient_*` flat white | ezQuake bakes a gradient into TrueType glyphs (digits yellow to orange by default), which would tint every coloured value |
| `font_gradient_alternate_*` `179 127 58` | The hub's bracket colour. Brackets are the high-bit glyphs `$xdb` / `$xdd`, which use this gradient |
| `tp_name_rl` / `lg` / `rlg` / `sng` / `ssg` | Best-weapon text in the hub's colours. These also change how teamplay macros print weapon names; the hub config does the same |
| `hud_teaminfo_loc_width 3` | ezQuake clips at the cell edge; three cells hold about five Roboto characters, matching the hub's `substring(0, 5)` |
| `hud_teamfrags` on the team headers | The only element that draws real team colours. One element, so its two boxes are a fixed distance apart: rows are `8 × 1.5 = 12` units, and in 4on4 the second header is `(1 + 4 + 1) × 12 = 72` below the first, so `cell_height 12` + `space_y 60`, `pos_y -60` |
| `hud_teamfrags_style 3` | No own-team marker (`Frags_DrawColors`: "Draw nothing") |
| `hud_sortrules_teamsort 0`, `includeself 0` | Alphabetical team order whoever you track, as on the hub (`getTeams()` sorts with `localeCompare`) |
| `font_outline_width 0` | The outline is a dark dilation baked into the glyph; after the glyph texture is shrunk it becomes a ragged fringe |

### Known differences from the hub

These are engine limits, not settings:

- **Wider team overlay.** teaminfo lays text out in fixed character cells even with a
  proportional font, so the panel is roughly 1.6x the hub's width at the same text size.
- **Header boxes are exact in 4on4 only.** In 2on2 the second team's box lands two rows
  below its header. `hud_teamfrags` is one element with a fixed gap between its boxes,
  while the second header's position depends on team size.
- **Team name sits at the left** of the header row, not beside the box; teaminfo draws
  its own header text and it cannot be moved or hidden.
- **No top-centre score box.** It would need a second `hud_teamfrags`, and
  `hud_score_team` / `hud_score_enemy` can't draw real team colours (their box is a fixed
  frame colour, ordered by point of view rather than alphabetically).
- **Player names are right-aligned**; the hub left-aligns them.
- **No highlighted row** for the player being tracked.
- **Text is softer than in a browser.** ezQuake renders at 1× on Retina displays (it
  deliberately doesn't request high-DPI, `vid_sdl2.c`), and TrueType glyphs come from a
  2048×2048 texture with no mipmaps, shrunk about 4× at this size. Neither is configurable.

---

## Credits and provenance

- qrack layout derived from the woods/qrack ezQuake HUD. Original author unknown — if
  that is you, open an issue and I will credit you properly.
- qwnu layout from the QuakeWorld Hub
  ([quakeworldnu/hub.quakeworld.nu](https://github.com/quakeworldnu/hub.quakeworld.nu)),
  with base `hud_teamfrags` settings from vikpe's
  [qw-streambot-ezquake](https://github.com/vikpe/qw-streambot-ezquake).
- qwnu artwork is the QuakeWorld Hub's own FTE asset set, as listed in the hub's
  `src/pages/games/fte/assets.ts` and served from `https://a.quake.world/fte/`.
- Roboto by Google, Apache License 2.0; the licence ships inside `qwnu.pk3`.
- `qrack_lmp.pk3`'s two bar images are the `SBAR` and `IBAR` lumps from id Software's
  `pak0.pak`, converted to TGA. Quake game data is copyright id Software.

The configs are free to use and modify. Artwork remains the property of its respective
owners.
