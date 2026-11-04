# bn-client-egui

Prototype of a mouse-first, screen-reader-accessible desktop client for Cataclysm: Bright Nights, built on egui/eframe
(AccessKit). The UI talks to a `GameModel` trait with two implementations: `FakeModel` (in-memory, used by most tests)
and `EngineModel` (unified engine protocol 0.2 over the engine's stdio, newline-delimited JSON-RPC).

Crates: `crates/core` (protocol, transport, replica, models, colours, keys), `crates/egui`, `crates/tui`.

License: AGPL-3.0-only (see LICENSE). Bundled fonts: [Noto Sans CJK KR and Noto Sans Mono CJK KR](https://github.com/notofonts/noto-cjk)
2.004, subset by `tools/subset_fonts.sh`, SIL Open Font License 1.1 (`crates/egui/assets/fonts/LICENSE`).

## Run

    cargo run --release -p bn-client-egui      # finds the engine itself (see below)
    cargo run -p bn-client-egui --features fake -- --fake  # in-memory fake model, no engine
    cargo run -p bn-client-tui                 # same engine lookup; --features fake -- --fake for the fake model
    cargo install --path crates/egui           # then just `bn-client-egui` (same for crates/tui: `bn-client-tui`)

The terminal client uses the same model and declares the terminal size to the engine as the map viewport.

## Engine and profile

The client takes the first engine it finds, in this order (`--help` lists the same):

1. `--engine PATH`
2. `BN_ENGINE`
3. `cataclysm-bn-tiles` or `cataclysm-bn` (+ `.exe`) next to the client executable or in its parent directory
4. The path saved in `$XDG_CONFIG_HOME/bn-client/config.json` (`~/.config/bn-client/config.json`)
5. `PATH`

If it finds no engine, the egui client opens the setup window with the list of searched places, and the TUI exits with
that list. Only the setup window saves the path, so later runs need no arguments. `--engine` applies to one run. Game
data comes from `BN_ROOT`, else the nearest ancestor of the chosen engine containing `data/json`.

The client starts the engine as `<engine> --client mcp` on the engine's default user and config directories. It uses
your saves, options and keybindings, and the Load tab and Save and Quit work as in Tiles. These options change that:

- `--userdir PATH` and `--configdir PATH` override the directories.
- `--fresh-profile` (or `BN_FRESH_PROFILE=1`) uses a private throwaway profile. Live tests always use one.
- `--seed TEXT` (or `BN_SEED`) passes the engine's `--seed`. The same seed starts the same world and game.

## Keys

The client binds no game keys. With every action, the engine sends the keys of the active input context (the game, a
menu, a popup, look mode), including the player's rebinds in the engine profile's `keybindings.json`. The client
forwards each press. Movement, menu navigation, paging, confirm, cancel, tab switching and look all use the engine's
bindings, so a rebound `CONFIRM` confirms in menus.

The egui client keeps these keys:

- Tab and Shift+Tab move focus in a menu. On the map and on composed or tabbed screens, Tab goes to the engine.
- Space and Enter act on a focused menu control.
- Esc closes a menu that can be cancelled.
- Alt+Up and Alt+Down move focus among the action rows.
- Alt+Left and Alt+Right turn the 3D view, unless the engine binds them.
- Shift+F10 opens the right-click menu on the cursor square.

The TUI keeps these keys:

- Ctrl-C quits.
- Tab opens the actions list, unless the engine binds Tab.
- PageUp and PageDown page the log.

The engine's look key starts and leaves the TUI's look mode, and the engine's direction keys move its cursor.

## Mouse

Left-click travels to the square. Right-click opens the engine's actions for the square. Hover shows a tooltip. Items
drag between Ground and Inventory, and each item row also has a button for the same move.

## Map

The map draws with the game's tilesets (`<game root>/gfx/<name>/tile_config.json`) like the native Tiles client, or as
ASCII glyphs. Ids without a sprite draw as their glyph. The engine publishes the sprite facts only it knows (tile
selection per square, light, season, facing, `looks_like`). See `docs/en/dev/reference/engine_client_protocol.md` in
the engine.

The `Client settings` window switches between Tiles and ASCII and picks the tileset. The choice is kept in
`$XDG_CONFIG_HOME/bn-client-egui/settings.json` (`~/.config/...` without it). With no choice saved, the client follows
the engine's `USE_TILES` and `TILES` options, then `UNDEAD_PEOPLE_BASE`.

## Sound

The egui client plays the sound effects, ambience and music the engine asks for, from the sound pack that the engine's
`Choose soundpack` option picks. The `Sound Enabled` option turns sound off, and the engine's volume options set the
volumes. `BN_SOUND_LOG=PATH` writes every file started and track stopped, with milliseconds since the audio device
opened.

## Tests

    cargo test    # egui_kittest UI tests, protocol fixtures, reconstruction, scripted engine
    cargo test -p bn-client-egui --test it ui_tiles::   # one test binary per crate: `crates/*/tests/it/<file>.rs`, filter by `<file>::`

Live integration tests need an engine build and are ignored by default. The TUI one drives its key handling.

    BN_ENGINE=/path/to/cataclysm-bn-tiles cargo test -p bn-client-core --test it live_engine:: -- --ignored
    BN_ENGINE=/path/to/cataclysm-bn-tiles cargo test -p bn-client-tui --test it live_tui:: -- --ignored
    BN_ENGINE=... cargo test --release -p bn-client-core --test it live_latency:: -- --ignored --nocapture   # timings

Live tests run on `BN_SEED`, else `2`. A test that needs another start names its own seed (`live_travel` uses `4`).

## Protocol contract

`crates/core/schema/0.2.schema.json` and `crates/core/tests/fixtures/0.2.examples/` are verbatim copies of the engine's
`docs/schema/engine-client/` files:

    deno run --allow-read --allow-write tools/sync_contract.ts <engine checkout>
    deno run --allow-read tools/sync_contract.ts <engine checkout> --check   # read-only

The hello names the schema's SHA-256 prefix. A mismatch fails the connection with the engine's error.

`crates/core/tests/fixtures/design.json` is a hand-written session, not captured from an engine.
`crates/core/tests/fixtures/menu-interaction.json` is the 0.2 menu examples with the `interaction` that the bespoke
menus still read.

## TODO

- Verify with NVDA on Windows and Orca on Linux that controls, the map status line and the live message log are read
  correctly.
- Map cells are painted, not individual accessibility nodes. Only the status line describes the cursor cell.
- Live announcement of identical consecutive messages is not distinguishable.
- Add a model readiness/coordinate-validity contract for loading and disconnects.
