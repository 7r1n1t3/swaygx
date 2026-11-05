# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

SwayFX is a fork of sway (tracks upstream sway `original_version` in `meson.build`, currently 1.12.0) that renders through **scenefx** (`github.com/wlrfx/scenefx`) instead of plain wlr_scene, adding blur, rounded corners, shadows, dimming of unfocused windows, animations, and per-layer-shell effects. The project scope is eye-candy; window-management features outside i3's scope are generally out of scope (see README "Contributing").

## Build

Meson + ninja, C11, `werror=true` (any warning fails the build).

```sh
meson setup build/          # or: meson build/
ninja -C build/
./build/sway/sway           # run (nested inside an existing Wayland/X session works)
```

- Dependencies: `wlroots-0.20` (>=0.20.0,<0.21.0) and `scenefx-0.5`. Either can be supplied as a subproject in `subprojects/` (gitignored); `wrap_mode=nodownload` means meson will not fetch them for you.
- Nix: `nix develop` gives a shell with all deps and copies scenefx into `./subprojects/`; `nix build` produces `./result/bin/sway`.
- Docker / Debian: `docker compose up --build` runs `build.sh`, which builds wlroots → scenefx → swayfx into `opt/`. See `INSTALL-deb.md`.
- Build options are in `meson_options.txt` (e.g. `-Dswaybar=false`, `-Dtray=disabled`, `-Dman-pages=disabled`).
- `build-debug/` may exist locally and contains `compile_commands.json`.

There is no test suite. Verify changes by building and running the compositor (nested is fine).

## Formatting

`.clang-format` approximates the style in `CONTRIBUTING.md` but cannot express everything (notably: a continuation line that opens a block gets **two** extra tabs, not one). Treat its output as a starting point. Key rules from `CONTRIBUTING.md`:

- Tabs for indentation, braces on the same line (functions too), always brace `if`/`while`/`for`.
- Aim for 80 cols (tab = 4), up to 100 if it reads better.
- C11 + POSIX, no GNU extensions.
- Global names prefixed `sway_<submodule>_`; static functions have no `sway_` prefix.
- Constructor pairs: `init`/`finish` (caller-allocated) or `create`/`destroy` (malloc/free). Destroy functions must accept NULL.
- Commit subject ≤ 50 chars, imperative ("Fix crash when ..."). Recent history also uses `fix:` / `style:` prefixes.

## Architecture

Top-level programs: `sway/` (compositor), `swaymsg/`, `swaybar/`, `swaynag/`; `common/` and `client/` are shared helpers; `protocols/` holds Wayland protocol XML compiled by wayland-scanner.

### Tree and scene graph

- The layout tree is `root → output → workspace → container → (view)` (`sway/tree/`). Each node owns `wlr_scene` nodes (scenefx types) that are reparented as the tree changes.
- `sway/tree/output.c` builds the per-output scene layers in z-order: `shell_background`, `shell_bottom`, **`blur_layer`** (a `wlr_scene_optimized_blur`: a cached blur of what is behind windows), `tiling`, `fullscreen`, `shell_top`, `shell_overlay`, `session_lock`. When anything under the blur layer changes (background/bottom layer surfaces, blur config), call `wlr_scene_optimized_blur_mark_dirty(output->layers.blur_layer)`.
- Each `sway_container` has fx scene nodes alongside the sway ones: `blur`, `shadow`, `dim_rect`, plus per-container `alpha`, `corner_radius`, `blur_enabled`, `shadow_enabled`, `dim` (`include/sway/tree/container.h`). Allocation helpers like `alloc_scene_shadow`/`alloc_scene_blur` in `sway/tree/node.c` use a shared `bool *failed` fallthrough pattern.
- `scene_descriptor.c` tags scene nodes with what they represent (view, container, layer shell, popup...) so input hit-testing can map a scene node back to a sway object.

### Transactions and animations

- Layout changes go through `sway/desktop/transaction.c`: `pending` state is copied to `current` once clients ack their configures, then `arrange_*` positions the scene nodes. This file is also where swayfx's fx work lives: it sizes and positions blur/shadow/dim/corner nodes and drives open/close/move/resize/workspace-fade animations.
- `sway/animation_manager.c` is a single global timer that ticks every `struct animation` (ease-out-cubic, `progress` 0→1) and calls `update`/`complete` callbacks. With `animation_duration_ms 0`, animations complete immediately. Containers keep `from_*`/`to_*` geometry and alpha for interpolation.
- Closing containers keep animating after their view unmaps (`close_anim_complete_callback` in `transaction.c`). Close-path code must check whether the view was remapped mid-animation and leave it alone if so.

### Commands and config

- `sway/commands.c` has sorted handler tables (`handlers[]`, plus config-only and runtime-only tables) looked up with `bsearch`. **New entries must stay alphabetically sorted** or lookup silently breaks. Each command lives in `sway/commands/<name>.c` and must be added to `sway/meson.build` and declared in `include/sway/commands.h`.
- Global fx settings live on `struct sway_config` (`include/sway/config.h`: `corner_radius`, `blur_*`, `shadow_*`, `default_dim_inactive`, `animation_duration_ms`, ...). Per-window overrides come from `for_window` criteria (`sway/criteria.c`). Per-layer-shell effects (`layer_effects <namespace> ...`) are in `sway/layer_criteria.c` and applied from `sway/desktop/layer_shell.c`.
- Document new user-facing options in `sway/sway.5.scd` (man pages are scdoc) and in the README "New Configuration Options" list.

## AI usage Disclosure

Disclosure is always required when “AI” was involved in the creative process of coming up with the code, except in the following cases:

```text
    Trivial or small changes that wouldn’t be copyrightable regardless of “AI” involvement. For example something like a “min” function fit this category.
    Mechanical changes where the expected result is obvious and not up to interpretation, ie. when it would be the same regardless of what tooling was used. For example, autocomplete, refactoring the name of a variable and similar activities fit this category. In these cases it is still recommended (but not required) to disclose the tool used, eg. “AI”, “sed”, “cocinelle”, etc. in order to ease the review process.
```

We suggest the following scheme to disclose the level “AI” tooling was involved:

```text
    Assisted-by: TOOL (OPTIONAL: MODEL) for when “AI” was involved in making decisions or also generated parts of the code.
    Generated-by: TOOL (OPTIONAL: MODEL) for when almost all the code was generated through “AI”.
```

Do not use the Co-authored-by tag as this one is reserved for human co-authors.
