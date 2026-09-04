# Breakout on Charybdis / cosmicstone — complete integration

This document shows, step by step, how the Breakout add-on is integrated into
the `cosmicstone` keyboard (keyboard: `cosmicstone/cosmicstone`, keymap
`default`). Nothing else in the keymap is touched: trackball, DPI, TL_MEDIA
tap/hold, the encoder volume actions, the HUD and the key combos keep working
exactly as before.

The keyboard has a 0.91" 128×32 SSD1306 mounted **vertically**
(`OLED_ROTATION_270`), so the game field is **32 px wide × 128 px tall** —
perfect for Breakout.

## Game flow on this keyboard

1. `macro1 + macro2` (the existing combo) → enters **GAME layer** (layer 2),
   exactly as before.
2. On the GAME layer, press **`ESC` + `DEL` together** → the splash screen
   appears (the chord is the only way in, so it cannot be triggered by
   accident). Press **`E`** to start.
3. Move the paddle: **`S`** (left), **`F`** (right), or the **encoder**
   (fine control, 3 px/notch).
4. Press **`ESC`** to exit the game (back to the GAME layer HUD), or
   **`E`** at any time to restart from level 1 (also from the
   game-over screen).
5. `macro1 + macro2` again → leaves the GAME layer, HUD back to normal.

Notes on this layout:

- The physical `ESC` key is `KC_ESC` on the GAME layer (top-left cell) and
  the physical `DEL` key is transparent there, so the trigger `ESC`+`DEL`
  uses two real keys, no remap needed. While the game is **not** running
  neither key is consumed: they behave exactly like normal keys on the
  GAME layer. While the game **is** running, the *presses* of `ESC` and
  `DEL` are reserved by the game (`ESC` exits, `DEL` is a no-op), while
  the *releases* always reach the host, so a key that was pressed before
  the game started is never left stuck on the host machine.
- The GAME layer keeps a shifted-letter remap on the home row; to keep the
  paddle keys predictable, the physical `S` and `F` keys are explicitly
  mapped to `KC_S` and `KC_F` there (two cells in row 2). The in-game
  RESTART key is the physical `E` key, pinned to `KC_E` (row 1) because
  the shifted-letter remap would otherwise make it send `KC_W`.

## 1. Copy the files

```
keymaps/default/breakout.c
keymaps/default/breakout.h
```

## 2. `keymaps/default/rules.mk`

```make
# Breakout easter-egg game (OLED)
# Sources are not auto-picked-up because the keymap folder has a
# keymap.json (new QMK structure): add them to the build explicitly.
SRC += $(KEYMAP_PATH)/breakout.c
# Trigger: ESC + DEL held together on the GAME layer (2).
# On layer 2 the physical ESC key is KC_ESC and the physical DEL key is
# KC_DEL (transparent cell, see keymap.c).
OPT_DEFS += -DBREAKOUT_ENABLE
OPT_DEFS += -DBREAKOUT_TRIGGER_LAYER=2
OPT_DEFS += -DBREAKOUT_TRIGGER_KEYCODE_1=KC_ESC
OPT_DEFS += -DBREAKOUT_TRIGGER_KEYCODE_2=KC_DEL
OPT_DEFS += -DBREAKOUT_RESTART_KEYCODE=KC_E
OPT_DEFS += -DBREAKOUT_KEY_LEFT=KC_S
OPT_DEFS += -DBREAKOUT_KEY_RIGHT=KC_F
# Flush all dirty display blocks in one pass (QMK default: 1 per cycle), so
# the game's full-frame resync is a single burst and the paddle tracks the
# input without trailing.
OPT_DEFS += -DOLED_UPDATE_PROCESS_LIMIT=16
```

If the field ever renders upside down on your board (paddle on top, HUD
below the bricks), add `OPT_DEFS += -DBREAKOUT_FLIP_Y=1` to flip the Y axis
at render time.

## 3. `keymaps/default/keymap.c`

### 3.1 Include

After the other includes:

```c
#include "breakout.h"
```

### 3.2 Layer 2: paddle keys on the home row

The GAME layer keeps a shifted-letter remap on the top rows, so the
paddle keys are pinned to their natural keycodes so that physical
`S`/`F` steer the paddle (`E` is pinned too: it is the in-game RESTART
key):

```c
    /*.  -, Q, W, E, R, T, ... */
    { _______, KC_T, KC_Q, KC_E, KC_E, KC_R, _______, ... },

    /*.  -, A, S, D, F, G, ... */
    { _______, KC_G, KC_S, KC_S, KC_F, KC_F, _______, ... },
```

(cells: physical `S` → `KC_S`, physical `F` → `KC_F`, physical `E` →
`KC_E`, the RESTART key)

### 3.3 `process_record_user()` — first lines of the function

```c
bool process_record_user(uint16_t keycode, keyrecord_t *record) {
    // Breakout easter egg: let it claim the key if it wants to.
#ifdef BREAKOUT_ENABLE
    if (breakout_on_key_record(keycode, record)) {
        return false;
    }
#endif
    switch (keycode) {
    ...
```

### 3.4 `encoder_update_user()` — first lines of the function

```c
bool encoder_update_user(uint8_t index, bool clockwise) {
    // Breakout easter egg: encoder moves the paddle while the game is on.
#ifdef BREAKOUT_ENABLE
    if (breakout_on_encoder(clockwise)) {
        return false;
    }
#endif
    uint8_t layer = get_active_layer();
    switch (layer) {
    ...
```

### 3.5 `housekeeping_task_user()` — add the call (end of function)

```c
void housekeeping_task_user(void) {
    if (timer_elapsed(last_keycode_change_time) >= 3000) {
        ...
    }

    // Breakout easter egg (no-op when the game is not running).
#ifdef BREAKOUT_ENABLE
    breakout_on_housekeeping();
#endif
}
```

### 3.6 `oled_task_user()` — right after the left-half check

```c
void oled_task_user(void) {
    if (is_keyboard_left()) {
        return false;
    }

    // Breakout easter egg: while the game is running it owns the screen.
#ifdef BREAKOUT_ENABLE
    if (breakout_on_oled()) {
        return false;
    }
#endif

    static uint16_t last_dpi = 0;
    ...
```

## 4. Build

```
make cosmicstone:default
```

## What is (and is not) changed in the keymap

| Area                       | Change |
|----------------------------|--------|
| Trackball / pointing       | none   |
| DPI key + HUD              | none   |
| TL_MEDIA tap/hold          | none   |
| Encoder (layer 0 and 2, outside the game) | none |
| Combos (`macro1+macro2`, …) | none  |
| Layer 2, cell (0,0)        | `_______` → `KC_ESC` (consumed by the game on that layer) |
| Four user hooks            | two lines of guarded calls each |

To turn the easter egg off completely, remove `-DBREAKOUT_ENABLE` from
`rules.mk` and restore cell (0,0) of layer 2 to `_______`.
