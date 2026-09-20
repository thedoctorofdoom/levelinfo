# LevelInfo

A GZDoom/UZDoom add-on that draws a live on-screen readout of level
statistics — kills, items, secrets, map time, par time, total time — plus
timers for active powerups, status hazards (morph, poison, radiation), and
remaining oxygen while underwater. It's a lightweight, self-contained HUD
overlay: no wads to merge, no PlayerPawn to replace, just a handful of lumps
loaded alongside your base mod.

This copy is a stripped-down fork of the original **LevelInfo** by *Tekish*,
trimmed to the stats/powerup/hazard display only (no map name/author
pop-up, no language translations) and tuned for compatibility with
**Project Brutality**.

## What it looks like in practice

A small block of text is drawn in a corner of the screen (position is
configurable) with two columns: a label on the left ("Kills", "Items",
"Time", "Berserk", "Rads", ...) and its value on the right ("34/50",
"12:04", ...). It updates continuously while playing and disappears when
there's nothing to show (e.g. on TITLEMAP, or a map with no kills/items/
secrets and every display option turned off).

## How it works

The add-on is split across ACS (game logic, runs on a timer) and ZScript
(rendering, runs every frame), glued together with `scriptcall`:

| File | Role |
|---|---|
| `mapinfo.txt` | `GAMEINFO` block that registers the `levelinforenderer` event handler with the engine. |
| `cvarinfo.txt` | Declares all `li_*` user cvars (the persisted settings). |
| `menudef.txt` | Adds "LevelInfo Options" to the Options menu, wired to those cvars. |
| `zscript.txt` | ZScript entry point; just sets the required engine version and includes `zscript/levelinfo.zsc`. |
| `zscript/levelinfo.zsc` | `levelinfobridge` — a tiny ACS-callable bridge exposing data ACS can't read directly (morph/poison/hazard tics, `level.totaltime`, console player). `levelinforenderer` — an `eventhandler` that receives the composed text from ACS and draws it every frame in `renderoverlay()`, handling font choice, position, scale, opacity, and alignment. |
| `source/levelinfo.acs` | The main logic. Reads cvars and level/actor state every few tics, formats the label/value strings, and calls into the renderer. Also defines the ACS scripts that hook level enter/respawn/death. |
| `source/levelassign.acs` | A lookup table (`levelassign()`) that maps a detected player class (e.g. `BrutalDoomer`, `Doom4Player`, `ArgPlayer`, ...) to the specific powerup inventory classes, on-screen labels, and color codes that mod uses, so the powerup timers work correctly across several supported gameplay mods. |
| `acs/levelinfo.o` | The **compiled** ACS library (bytecode) built from the two `source/*.acs` files above. This is what actually gets loaded — see "Editing the ACS source" below if you change the `.acs` files. |
| `loadacs.txt` | Lists `levelinfo`, telling the engine to auto-load `acs/levelinfo.o` as a library (equivalent to an `OPEN` script) as soon as the add-on loads. |
| `shafont.fon2` / `shadfont.fon2` | Two bundled bitmap fonts ("LevelInfo" and "Doom" in the font option) used to draw the HUD text. |

### Runtime flow

1. **Level enter** — the ACS `levelinfoenter` script (an `ENTER` script) runs
   once, calls `levelassign(getactorclass(0))` to populate the powerup table
   for the player's current class, then starts the looping `levelinfo`
   script. It bails out immediately on `TITLEMAP` or if the local player
   isn't the console player.
2. **Every 3 tics** — the `levelinfo` script re-reads the relevant `li_*`
   cvars, pulls stats via ACS built-ins (`getlevelinfo`, `timer()`,
   `getactorpoweruptics`, `getairsupply`) and pulls morph/poison/radiation/
   total-time data from ZScript via `scriptcall("levelinfobridge", ...)`
   (data ACS has no native accessor for). It formats everything into two
   parallel, newline-joined strings — one of labels, one of values — and
   hands them off with `scriptcall("levelinforenderer", "pushrows", ...)`.
3. **Every frame** — `levelinforenderer.renderoverlay()` (ZScript) takes the
   most recently pushed labels/values, splits them back into rows, measures
   the chosen font, computes an anchor point from the position/alignment
   cvars, and draws each row with `screen.drawtext`. If nothing has been
   pushed in the last ~4 seconds (140 tics) — e.g. the level changed under
   it — it stops drawing rather than show stale data.
4. **Respawn** — `levelinforespawn` (a `RESPAWN` script) simply restarts the
   `levelinfo` loop.
5. **Death** — `levelinfodeath` (a `DEATH` script) clears the renderer
   immediately and terminates the loop, so the HUD doesn't linger over the
   death screen.

## Requirements

- GZDoom or UZDoom **4.10.0 or newer** (declared by the `version` directive
  in `zscript.txt`).
- The `acc` ACS compiler is only needed if you intend to modify
  `source/*.acs` — the compiled `acs/levelinfo.o` is already committed and
  is all the engine needs at runtime.

## How to use it

Load this folder (or a zipped/`.pk3` copy of it) with `-file`, alongside
your base mod. Because `source/levelassign.acs` detects powerups by player
class, load it **after** whichever gameplay mod defines that class (e.g.
after Project Brutality) so the class is already present when `ENTER` fires:

```bash
gzdoom -iwad DOOM2.WAD \
  -file "ProjectBrutality.pk3" \
  "ProjectBrutalityAddon18 - Info03 - LevelInfo"
```

Once in-game, open **Options → LevelInfo Options** to configure it. The
available settings (all backed by cvars in `cvarinfo.txt`):

| Option | Cvar | Description |
|---|---|---|
| Show LevelInfo | `li_visible` | Master on/off switch for the whole HUD block. |
| Abbreviated Characters | `li_abbreviate` (0–8) | Truncates each label to N characters; `0` shows full labels. |
| Highlight Values | `li_highlightvalue` | Colors a value differently once it's "complete" (100% kills/items/secrets, or par time beaten). |
| Opacity Percentage | `li_opacity` (10–100) | Overall transparency of the drawn text. |
| Show Active Powerups | `li_showpowerups` | Show timers for currently-held powerups (per the mod-specific table in `levelassign.acs`). |
| Show Berserk Status | `li_showberserk` | Include the "berserk-style" powerup (index 0 in the table, e.g. Berserk pack) as a timer row. |
| Show Hazards | `li_showhazards` | Show morph, poison, and radiation-suit-damage timers. |
| Show Map Par Time | `li_showpar` | Show the map's par time alongside elapsed time. |
| Show Map Statistics | `li_showlevelinfo` | Show kill/item/secret counts. |
| Show Map Time | `li_showtime` | Show elapsed time on the current map. |
| Show Map Total Time | `li_showtotaltime` | Show cumulative time across all maps (`level.totaltime`). |
| Show on Automap | `li_showonmap` | Whether the HUD stays visible while the automap is open. |
| Show Oxygen Time | `li_showoxygen` | Show a countdown while submerged, if the player can drown. |
| Show Percentages | `li_showpercentage` | `No` / `With Statistics` (adds a `%` alongside counts) / `Replace Statistics` (shows only the `%`). |
| Font | `li_font` | `LevelInfo` (bundled `shafont.fon2`), `Doom` (bundled `shadfont.fon2`), `Mod Small Font`, `Mod Big Font`, or `Custom`. |
| Custom Font Lump | `li_fontcustom` | Lump name to use when Font is set to `Custom`. |
| Render Scale | `li_scale` (0–8, `Auto`) | `0`/`Auto` follows the engine's clean scale factor; higher values shrink the virtual canvas (making text appear larger). |
| Value Alignment | `li_alignvalue` | `Standard` (left-aligned values) or `Numeric` (right-aligned, good for columns of numbers). |
| Position Preset | `li_position` | `Custom`, or one of six screen-corner/edge presets (`Top Left`, `Center Left`, `Bottom Left`, `Top Right`, `Center Right`, `Bottom Right`). |
| Horizontal/Vertical Alignment | `li_alignx` / `li_aligny` | Only used when Position is `Custom`: anchor to the left/right or top/bottom edge. |
| Horizontal/Vertical Position | `li_x` (0–90) / `li_y` (0–50) | Only used when Position is `Custom`: offset from the anchored edge, in character cells. |

### Editing powerup support for another mod

If a gameplay mod isn't recognized, or you want to change which powerups
are tracked, edit `source/levelassign.acs`. Add (or modify) a branch keyed
on the player's class name:

```c
else if (class == "YourPlayerClassName")
{
    powerups[0][0] = true;   powerupsvalues[0][0] = "SomePowerupClass";
    powerupsvalues[0][1] = "Label";   powerupsvalues[0][2] = "g"; // ZDoom color code
}
```

Index `0` is treated specially as the "berserk-style" slot (gated by the
`li_showberserk` option); every other index is a normal timed powerup.
`powerupsvalues[n][0]` must be the exact inventory class name granted by
the powerup, `[n][1]` is the label shown on screen, and `[n][2]` is a
[ZDoom text color code](https://zdoom.org/wiki/Print#Colors).

**Because `acs/levelinfo.o` is the file actually loaded at runtime**, any
change to `source/levelinfo.acs` or `source/levelassign.acs` needs to be
recompiled with `acc` (or via SLADE's built-in ACS compiler) back into
`acs/levelinfo.o` before it will take effect in-game:

```bash
acc -i /path/to/zdoom-acc-includes source/levelinfo.acs acs/levelinfo.o
```

## Credits

Originally created by **Tekish**. This fork strips the feature set down to
level stats, powerup timers, and hazard/oxygen indicators, and adapts the
renderer for use alongside Project Brutality.
