# RGB-Pi Theme Builder (Tauri shell)

Scaffold from **Step 1** of `../BUILD_PLAN.md`: Tauri 2 + Vite + TypeScript (vanilla).

Run all `npm` / `tauri` commands from **this directory** (`theme_builder/app/`), not from `theme_builder/`.

## Commands

```bash
npm install
npm run tauri dev    # desktop window + Vite dev server
npm run build        # front-end only (tsc + vite build)
npm run tauri build  # full app installer/bundle
```

Prerequisites: follow [Tauri v2 prerequisites](https://v2.tauri.app/start/prerequisites/) for your OS.

If `tauri build` fails during **AppImage** / `linuxdeploy` (common in minimal or headless environments), the compiled binary may still exist under `src-tauri/target/release/`. You can limit bundles, for example: `npm run tauri build -- --bundles deb`.
