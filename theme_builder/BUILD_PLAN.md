# RGB-Pi Theme Builder — Tauri + Web UI

This document breaks work into **small steps**. After each step you should be able to run checks and confirm behavior **without depending on later steps**.

Bundled sample data for development lives in `theme_builder/themes/` (a copy of the repository’s top-level `themes/` folder).

---

## Step 0 — Toolchain sanity check

**Goal:** Confirm Rust, Node, and a C toolchain for Tauri are available.

**Do:**

- `rustc --version` (1.77+ recommended for recent Tauri 2)
- `node --version` and `npm --version` (LTS Node)
- On Linux: ensure WebKitGTK / GTK dev packages match [Tauri prerequisites](https://v2.tauri.app/start/prerequisites/) for your distro.

**Verify:** All commands print sensible versions; Tauri’s official prereq page has no red flags for your OS.

---

## Step 1 — Scaffold the Tauri project (empty shell)

**Goal:** Create `theme_builder/app/` (or your chosen subfolder) with **Tauri 2 + Vite + TypeScript** (vanilla or React—pick one and keep it for the whole project).

**Do:**

- From `theme_builder/`, run non-interactive scaffold (this repo uses **vanilla-ts**):

  `npx create-tauri-app@latest app -y -m npm -t vanilla-ts --tauri-version 2 --identifier com.rgbpi.themebuilder`

- Tauri **bundle identifiers** must not contain underscores (only `A–Z`, `a–z`, `0–9`, `-`, `.`). If `tauri build` fails validation, adjust `identifier` in `src-tauri/tauri.conf.json`.
- Set **product name**, **window title**, and `<title>` to “RGB-Pi Theme Builder”; set `package.json` `name` to something like `rgb-pi-theme-builder`.

**Verify:**

- From the **app folder** (not `theme_builder/` alone): `cd theme_builder/app && npm install && npm run tauri dev` opens a window with the default welcome content. There is **no** `package.json` in `theme_builder/` root—only under `theme_builder/app/`.
- `npm run build` and `npm run tauri build` complete (release build can be Step 1 optional if slow). Linux `.deb` / `.rpm` / AppImage bundling may need extra tools; the **Rust binary still counts as success** if bundling is skipped via `tauri build --bundles deb` or by fixing bundle deps later.

**Stop line:** No custom logic yet—only the template.

---

## Step 2 — Pin versions and repo hygiene

**Goal:** Reproducible installs and clear boundaries with the Python frontend.

**Do:**

- Commit `package-lock.json` / `pnpm-lock.yaml` (whichever you use).
- Ensure `theme_builder/app/README.md` documents `dev` / `build` (added during Step 1 in this repo; extend as needed).
- Optionally add root `.gitignore` entries if `themes/` duplicate is huge and you prefer symlink + ignore patterns later (your choice).

**Verify:** Fresh clone → install → `tauri dev` still works.

---

## Step 3 — Read-only access to bundled `themes/`

**Goal:** The desktop app can **list** theme folder names under `theme_builder/themes/` using the Tauri **filesystem** API (or path resolver), without writing yet.

**Do:**

- Configure Tauri **capabilities** / **scope** so the app may read `theme_builder/themes/**` in dev. Prefer resolving a path relative to the repo or an env var for dev vs a fixed user path for production later.
- Expose one command: e.g. `list_themes` → array of directory names.

**Verify:**

- In the UI, render the list returned from Rust (even as a plain `<ul>`).
- List matches `ls theme_builder/themes` (same names, sorted if you sort in code).

**Stop line:** Still no editing of `theme.ini` or images.

---

## Step 4 — Minimal “theme selected” state in the Web UI

**Goal:** Click a theme name → UI shows **which theme is selected** (global state / context / store).

**Do:**

- Front-end only: highlight selection, store `selectedTheme: string | null`.

**Verify:**

- Clicking themes updates the label; no Rust changes required beyond Step 3 if list is static-loaded once, or refetch on demand.

---

## Step 5 — Load and display `theme.ini` as text

**Goal:** For the selected theme, read `theme_builder/themes/<name>/theme.ini` and show contents in a read-only panel (textarea or monospace `<pre>`).

**Do:**

- Rust command: `read_theme_ini(themeName) -> string` with error handling if missing.
- Wire to UI when selection changes.

**Verify:**

- Pick “Mega Tech” → file content matches on-disk `theme.ini`.
- Pick a bogus name → UI shows a clear error (no panic).

---

## Step 6 — Parse `theme.ini` into a structured object (read path)

**Goal:** Parse INI in Rust **or** TS; pick one source of truth. Rust is nice for later validation and packaging.

**Do:**

- Parse sections/keys into a tree (e.g. `Record<string, Record<string, string>>`).
- Preserve order if you care (some INI writers reorder; for v1, order may not matter).

**Verify:**

- Log or display key counts per section; spot-check `pal`, `sys`, `bg` keys against raw file.
- Changing nothing in the file → parse → serialize round-trip optional test later (not required in this step).

---

## Step 7 — Write scope: save `theme.ini` (controlled)

**Goal:** Allow **saving** the same file you opened, with an explicit “Save” button (no auto-save yet).

**Do:**

- Extend Tauri FS scope for **write** to `theme_builder/themes/<selected>/**` (tighten to `theme.ini` only if your API allows).
- Command: `write_theme_ini(themeName, contents: string)` or write structured → emit INI from Rust.

**Verify:**

- Edit a harmless comment in UI → Save → `diff` on disk shows change.
- Reload app → change persists.

---

## Step 8 — Dedicated dev “workspace” path (optional but recommended)

**Goal:** Avoid writing directly into the committed `theme_builder/themes` copy during daily dev.

**Do:**

- Add “Open workspace folder” (Tauri dialog) pointing at a **copy** of a theme; or copy-on-first-edit into `theme_builder/workspace/<name>/`.
- Document workflow in `app/README.md`.

**Verify:**

- Edits land under `workspace/` and original bundled themes stay pristine.

---

## Step 9 — 320×240 preview canvas (static BMP first)

**Goal:** Show `images/background.bmp` for the selected theme in a canvas scaled to a comfortable zoom (e.g. 640×480 display area with nearest-neighbor).

**Do:**

- Rust: read file as bytes → base64 or write to temp URL; or use `convertFileSrc` if using Tauri’s asset pattern.
- Front-end: draw on `<canvas>` or CSS `image-rendering: pixelated` on `<img>`.

**Verify:**

- Switching theme switches background image; broken path shows placeholder.

---

## Step 10 — Animated GIF preview

**Goal:** If `theme.ini` says `ani_gif` (or file exists), preview `background.gif` / `background_tate.gif` with animation in the WebView.

**Do:**

- Simplest: `<img src="...">` with a tauri-safe URL for the GIF file (often works with `convertFileSrc` from Tauri 2).
- Fallback: document browser limits; if blocked, use a WASM/Rust decoder later.

**Verify:**

- A theme with GIF background animates in the preview panel; static BMP theme still works.

---

## Step 11 — INI field editor (structured, not raw text only)

**Goal:** Edit a **subset** of keys (e.g. `[sys]` positions/colors) via form controls; sync to structured model → save to INI.

**Do:**

- Start with 3–5 high-value keys (e.g. `title_x`, `list_y`, `list_select_color`).
- Validate: numbers, `center`, palette color names from `[pal]`.

**Verify:**

- Change a value → Save → reopen file → value updated.
- Boot rgb-pi-frontend (optional manual check) with that theme—only if you deploy theme to the Pi; not automated in this step.

---

## Step 12 — Font sheet + `.dat` viewer (read-only)

**Goal:** Display `fonts/title.bmp` and optionally overlay glyph rectangles from `title.dat` for debugging.

**Do:**

- Parse `.dat` line format (same rules as `rtk.load_font_map` in `rtk.py`).
- Draw rects on canvas or SVG overlay.

**Verify:**

- Character `0` cell matches `rtk` width/height; spot-check a few letters.

---

## Step 13 — Export / “Install path” story

**Goal:** Zip the theme folder or copy to a user-chosen directory for transfer to the Pi.

**Do:**

- “Export theme…” → save dialog → zip or recursive copy using Tauri or a small Rust crate.

**Verify:**

- Exported zip unpacks to a valid tree (`theme.ini`, `images/`, `fonts/`, `sounds/`).

---

## Step 14 — Polish, packaging, CI

**Goal:** Installers per OS, versioned releases.

**Do:**

- Icons, app id, updater (optional), GitHub Actions matrix `ubuntu/windows/macos`.

**Verify:**

- CI artifact runs on a clean machine and passes Steps 3–7 smoke tests.

---

## Reference (implementation source of truth)

- Theme load semantics: `rtk.py` (`load_theme_cfg`, `load_theme`, `load_background`, `load_font_map`, …).
- Theme list + refresh: `utils.py` (`load_themes`, `refresh_theme`).
- Example INI: `theme_builder/themes/Mega Tech/theme.ini`.

---

## Suggested order of manual testing

After each step, record **one command** and **one visual check** in your own changelog (e.g. “Step 5: select Mario → INI visible”). That keeps regressions obvious when you add GIF and FS writes later.
