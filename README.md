# Breakout_QMK

A small, self-contained **Arkanoid/Breakout** game for any QMK
keyboard with an **OLED display** (128×32 SSD1306).

It is written as a *drop-in add-on*: the whole game lives in two files
(`breakout.c`, `breakout.h`) that you copy into a keymap folder and wire
through four standard QMK hooks. No board-specific code, no dependency on any
particular keyboard. When the game is not running, every hook is a no-op and
the host keymap behaves exactly as before.

## Gameplay

- Classic Arkanoid/Breakout gameplay.
- 4×5 bricks, 3 lives, level progression, score screen.
- The ball sits on the paddle for a moment, then launches automatically.
- Paddle position (where you hit the ball) changes the bounce angle.
- Scoring: 10 points per brick. A **combo** (consecutive bricks hit
  without the ball touching the paddle again) pays a bonus on every extra
  brick: `BREAKOUT_COMBO_BONUS_PCT` percent of the total points earned by
  the previous bricks of the same combo (100% by default, so the combo
  doubles its total with each brick). A level cleared **without
  losing a life** gives a +100 bonus and one extra life (while you have
  fewer than 3 lives).
- The ball starts gentle: for the first `BREAKOUT_BALL_SLOW_LEVELS` levels it
  advances at half speed, then speeds up (+1 px/tick every
  `BREAKOUT_BALL_SPEED_EVERY` levels, capped).
- The HUD: level (`L1`, starting at 1) on the left and lives (`x3`) on the right 
  of the top row. The score is only shown on the game over screen. Between levels there is only a short pause - no transition screen.
- On a 128×32 panel mounted vertically the field is 32 px wide × 128 px tall;
  mounted horizontally it uses the full 128×32 area.

## Controls (defaults)

| Action              | Default key |
|---------------------|-------------|
| Start game          | `ESC` + `DEL` pressed **together** while on the trigger layer (opens the splash screen) |
| Begin from splash   | `E` |
| Exit game           | `ESC`, any time while running |
| Restart game        | `E`, any time (also from the game-over screen) |
| Move paddle left    | `S` |
| Move paddle right   | `F` |
| Move paddle (fine)  | Encoder (3 px per notch, in either direction) |

All of these are configurable, see [Configuration](#configuration).

## Requirements

- QMK (classic) keymap.
- `OLED_ENABLE` with a 128×32 SSD1306 driver.
- `ENCODER_ENABLE` only if you want encoder paddle control.

## Installation

### 1. Copy the files

Put `breakout.c` and `breakout.h` into your keymap folder (next to
`keymap.c`). QMK compiles every `.c` file in the keymap directory, so no
`rules.mk` changes are needed for the sources themselves.

### 2. Enable and configure (keymap `rules.mk`)

```make
# Breakout game
OPT_DEFS += -DBREAKOUT_ENABLE
OPT_DEFS += -DBREAKOUT_TRIGGER_LAYER=2
OPT_DEFS += -DBREAKOUT_TRIGGER_KEYCODE_1=KC_ESC
OPT_DEFS += -DBREAKOUT_TRIGGER_KEYCODE_2=KC_DEL
OPT_DEFS += -DBREAKOUT_RESTART_KEYCODE=KC_E
OPT_DEFS += -DBREAKOUT_KEY_LEFT=KC_S
OPT_DEFS += -DBREAKOUT_KEY_RIGHT=KC_F
# Recommended: let the OLED driver flush every dirty display block in one
# pass (QMK default is 1 per cycle). The game rewrites the whole frame
# every 500 ms; with the default limit that takes ~16 main-loop cycles and
# the paddle visibly trails the input. This is a standard QMK define - no
# driver change.
OPT_DEFS += -DOLED_UPDATE_PROCESS_LIMIT=16
```

### 3. Wire the hooks (keymap `.c`)

```c
#include "breakout.h"

// Top of process_record_user():
bool process_record_user(uint16_t keycode, keyrecord_t *record) {
    #ifdef BREAKOUT_ENABLE
    if (breakout_on_key_record(keycode, record)) {
        return false; // the game consumed this key
    }
    #endif
    // ... existing code ...
}

// encoder_update_user() - if you have an encoder:
bool encoder_update_user(uint8_t index, bool clockwise) {
    #ifdef BREAKOUT_ENABLE
    if (breakout_on_encoder(clockwise)) {
        return false; // the game moved the paddle, skip the normal action
    }
    #endif
    // ... existing code ...
}

// housekeeping_task_user():
void housekeeping_task_user(void) {
    #ifdef BREAKOUT_ENABLE
    breakout_on_housekeeping();
    #endif
    // ... existing code ...
}

// oled_task_user() - before your own drawing:
void oled_task_user(void) {
    #ifdef BREAKOUT_ENABLE
    if (breakout_on_oled()) {
        return false; // the game owns the screen, but let the driver flush
    }
    #endif
    // ... existing code ...
}
```

That is all. While the game is running, `breakout_on_oled()` returns true and
the host OLED code is skipped entirely; while the game is off, nothing changes.

## Trigger mechanism

The game starts when the **two trigger keycodes** are held at the same time
while the **trigger layer** is active. A few consequences to know:

- The trigger is matched by *keycode*, so the two keycodes must be reachable
  from the keys you want on the trigger layer. If a physical key currently
  produces nothing (`_______`) on that layer, map it to a trigger keycode
  there.
- While the trigger layer is active, the two trigger keycodes are consumed by
  the game and never reach the host. On a layer where those keys had a
  normal job (e.g. typing), they become the easter-egg trigger instead.
- The **first** trigger key is also the in-game **exit** key; the in-game **restart** key is `BREAKOUT_RESTART_KEYCODE` (default `KC_E`), active at any time, including from the game-over screen.

See `examples/cosmicstone.md` for a complete, concrete integration on the
Charybdis/cosmicstone keyboard.

## Configuration

All options are optional; the shown value is the default.

### Trigger / controls

| Macro                        | Default | Meaning                                    |
|------------------------------|---------|-------------------------------------------|
| `BREAKOUT_TRIGGER_LAYER`     | `2`     | Layer on which the two trigger keys start the game |
| `BREAKOUT_TRIGGER_KEYCODE_1` | `KC_ESC`| First trigger key. Also the in-game EXIT key |
| `BREAKOUT_TRIGGER_KEYCODE_2` | `KC_DEL`| Second trigger key (the start chord) |
| `BREAKOUT_RESTART_KEYCODE`   | `KC_E`  | In-game RESTART key: new game from level 1, any time, including the game-over screen |
| `BREAKOUT_KEY_LEFT`          | `KC_S`  | Paddle left                               |
| `BREAKOUT_KEY_RIGHT`         | `KC_F`  | Paddle right                              |

### Field / geometry

| Macro                  | Default                    | Meaning                          |
|------------------------|----------------------------|----------------------------------|
| `BREAKOUT_FIELD_WIDTH` | (display width or height, see below) | Force field width (px)   |
| `BREAKOUT_FIELD_HEIGHT`| (display height or width)  | Force field height (px)          |

By default the field is derived from the display geometry and the current
OLED rotation: a 128×32 panel mounted vertically (`OLED_ROTATION_90/270`)
gives a 32×128 field; mounted horizontally, a 128×32 field. The rotation is
read from the driver's `oled_rotation` global. Define
`BREAKOUT_FIELD_WIDTH`/`BREAKOUT_FIELD_HEIGHT` to force the size (and to use
this add-on on QMK versions where that global is not accessible).

Orientation: depending on how the panel is mounted, the rotation may map
the logical top of the field (y = 0, where the HUD lives) to the *physical
bottom* of the display. If the game looks upside down (interface below the
bricks, paddle on top), add `-DBREAKOUT_FLIP_Y=1` to `rules.mk`.

### Gameplay

| Macro                  | Default | Meaning                               |
|------------------------|---------|---------------------------------------|
| `BREAKOUT_STEP_MS`     | `16`    | Physics tick (≈60 fps at 16 ms)       |
| `BREAKOUT_PADDLE_WIDTH`| `7`     | Paddle width (px)                     |
| `BREAKOUT_PADDLE_SPEED`| `2`     | Key-based paddle speed (px/tick)      |
| `BREAKOUT_PADDLE_BOTTOM`| `4`    | Paddle distance from field bottom (px)|
| `BREAKOUT_ENCODER_STEP`| `3`     | Encoder paddle step (px/notch)        |
| `BREAKOUT_BALL_SIZE`   | `2`     | Ball size (px)                        |
| `BREAKOUT_BALL_SLOW_LEVELS` | `2` | Levels 1..N that run at half ball speed (1 px every 2 ticks) |
| `BREAKOUT_BALL_SPEED`  | `1`     | Ball vertical speed at level 1 (px/tick) |
| `BREAKOUT_BALL_SPEED_EVERY` | `8` | Levels per +1 px/tick speed-up        |
| `BREAKOUT_BALL_SPEED_MAX`   | `3`   | Ball vertical speed cap (px/tick)     |
| `BREAKOUT_BRICK_ROWS`  | `5`     | Brick rows                            |
| `BREAKOUT_BRICK_COLS`  | `4`     | Brick columns (rows×cols ≤ 32)        |
| `BREAKOUT_BRICK_HEIGHT`| `3`     | Brick height (px)                     |
| `BREAKOUT_BRICK_TOP`   | `24`    | Y of the first brick row (px, below HUD)   |
| `BREAKOUT_START_LIVES` | `3`     | Initial lives                         |
| `BREAKOUT_READY_MS`    | `700`   | Ball-on-paddle delay before launch    |
| `BREAKOUT_PAUSE_MS`    | `900`   | Level transition duration (ms)        |
| `OLED_UPDATE_PROCESS_LIMIT` (QMK) | `16`  | Not a game define, but recommended in `rules.mk`: makes the OLED driver flush all dirty blocks in one pass so the periodic full-frame resync lands as a single burst. Default QMK value is 1 (one block per cycle), which makes the paddle trail the input. |
| `BREAKOUT_HUD_BLINK_MS` | `500` | Duration of the blink shown when the lives or the level counter is updated (two on/off cycles). `0` disables the blink. |
| `BREAKOUT_SPLASH_BLINK_MS` | `750` | Splash screen `START` blink duration per phase (slow blink). `0` disables the blink. |
| `BREAKOUT_RESYNC_MS`   | `500`   | Full-frame re-flush period (ms), `0` = off. Periodically rewrites the whole frame so stale panel content (e.g. a ghost of the paddle) can never persist. Uses only the public OLED API. |
| `BREAKOUT_BRICK_SCORE` | `10`    | Points per brick                      |
| `BREAKOUT_LEVEL_SCORE` | `100`   | Bonus for a level cleared without losing a life (also grants one extra life while lives < 3) |
| `BREAKOUT_COMBO_BONUS_PCT` | `100` | Combo bonus: each extra brick hit in a combo (no paddle touch in between) adds this percentage of the total points of the previous bricks of the combo. `0` disables the combo bonus |
| `BREAKOUT_FLIP_Y`      | `0`     | Mirror the Y axis at render time. Set to `1` when the panel/rotation maps y=0 to the physical bottom (HUD would appear below the bricks) |

## Notes

- **Split keyboards**: the game runs on the half that processes key events
  (the "master" half in QMK split setups). Put the trigger keys on that half.
- **OLED timeout**: while the game is running the display is kept on
  (`oled_on()` every housekeeping tick), so long games do not dim out.
- **Framebuffer**: the game redraws only the pixels that change (ball,
  paddle, hit bricks), so the OLED I2C traffic stays tiny.
- **Fixed HUD**: a single row at the top, always visible, with the level
  (`L1`, starting at the first level) on the left and the lives (`x3`) on
  the right; the score is only shown on the game-over screen. There are no
  intermediate screens: the ball launch delay and the between-levels pause
  are pure timers. When the lives or the level counter is updated, the
  affected counter blinks twice (`BREAKOUT_HUD_BLINK_MS`).
- **Splash**: shown once per entry through the trigger combo (never on
  in-game restarts): a big paddle with the ball floating on top, and a
  slowly blinking `START`; the restart key begins the game.
- **Scoring example** (default 100% combo bonus): a 4-brick combo without
  touching the paddle pays 10 + 20 + 40 + 80 = 150, not 40. The combo
  resets on every paddle touch (or when a life is lost).
- **Game over**: a dedicated screen (no HUD rows), top to bottom: the final
  **score**, then **GAME** and **OVER** in inverted text (black on white, the
  standard "bold" look in single-stroke fonts) on a single white rectangle —
  one white line on top, one between the words, one on the bottom — and the
  `PRESS E` hint below. Score and banners are drawn **pixel-exact** (the
  game carries the few 6x8 glyphs it needs) because the driver's grid cursor
  can never center a 24-px word in a 32-px field.
- **All on-screen text is in English.**
- **Text grid**: the QMK OLED text cursor is quantized — `oled_set_cursor`
  counts the column in font characters (`OLED_FONT_WIDTH` px) and the line
  in pages (`OLED_FONT_HEIGHT` px rows), while `oled_write_pixel` works in
  plain pixels. In-game text (HUD) goes through a small helper that
  converts from field pixel space, so the game keeps working on boards
  with other font sizes; the game-over score/banners/hints are drawn by a
  built-in pixel-exact renderer (the game carries the few 6x8 glyphs it
  needs), and the banners/bands/bricks (plain pixels) are always
  pixel-exact.
- **Encoder context**: on FreeRTOS-based QMK builds (e.g. RP2040) the
  encoder hook is called from the dedicated encoder task, not from the
  main loop. The game only writes a clamped 8-bit paddle position from
  there, which is safe without locks.
- **Ball / HUD separation**: the ball's top wall is the bottom edge of the
  HUD (`BREAKOUT_HUD_HEIGHT` = 1 font line, the level/lives row), so the
  ball can never cross the HUD text. The bricks stay where they are:
  the extra row only widens the gap between HUD and bricks.
- **Paddle redraw**: the paddle is cleared and redrawn unconditionally on
  every tick, so a stale image of the previous position can never linger.
- To disable the add-on later, just remove `-DBREAKOUT_ENABLE`; all hooks
  disappear with it (the keymap-side calls are `#ifdef` guarded).

## License

GPL-3.0-or-later.
