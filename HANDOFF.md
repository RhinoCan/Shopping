# Project Handoff: Shopping List PWA

## Tech Stack

- Vue 3 with `<script setup>` and TypeScript
- Vuetify 3 for UI components
- Vite 8 as the build tool
- `vite-plugin-pwa` for offline/PWA support (installed with `--legacy-peer-deps` due to Vite 8 peer dependency lag)
- Local storage for data persistence

## Project Location

- `C:\Vue\shopping`
- Repo not yet on GitHub

## Current State

- Project fully built and working as an installed PWA on an Android phone
- `App.vue` contains the complete application — no child components
- `HelloWorld.vue` has been deleted
- PWA icons in place, production build tested and deployed to phone

## Data Model

Two keys in local storage:

`masterList` — ordered array of objects:
```json
[
  { "id": 1716000000000, "name": "Milk 2%" },
  { "id": 1716000000001, "name": "Garbage bags large" }
]
```
- Order in array = display order in UI
- IDs generated with `Date.now()`
- No categories — just names
- Names can include brand/size details e.g. "Milk 2% Lactantia 4L"

`currentTrip` — array of IDs:
```json
[1716000000000, 1716000000001]
```

## UI Summary

**Header:**
- App title "Shopping" in Zilla Slab serif, with an item/selection count beside it (e.g. "64 ITEMS / 3 SELECTED")
- A `v-btn-toggle` pair: "All" (left) and "Selected" (right) — controls Show All vs Selected Only mode; default is "All"
- "Clear Trip" button — removes all IDs from `currentTrip`
- "Export" button — downloads `shopping-list.json` containing both `masterList` and `currentTrip`
- "Import" button (label wrapping a hidden file input) — reads a previously exported JSON and replaces local data
- Header controls use `flex-wrap: wrap` so buttons reflow to a second row on narrow screens

**Show All mode — each row contains:**
- Checkbox (checked if ID is in `currentTrip`)
- Item name — double-click or long-tap (500ms) to edit inline
- Up / Down arrow buttons — reorder; disabled at list boundaries
- Add button (＋) — inserts a new item immediately below this row via a small dialog
- Delete button (✕) — removes item from `masterList` and `currentTrip`

**Selected Only mode — each row contains:**
- Checkbox only (to allow unchecking at the store)
- Item name (read-only, no reorder/add/delete controls)

**Checked item appearance:**
- Bold, slightly larger font, dark navy (`#1a3a5c`) — high contrast against the cream background
- No strikethrough

**Colour palette / fonts:**
- Background: `#f5f0e8` (warm cream)
- Primary text: `#1a1a1a`
- Active toggle / confirm button: `#2d2d2d` background, `#f5f0e8` text
- Inactive toggle: `#f5f0e8` background, `#888` text
- Destructive (Clear Trip): `#b04030`
- Export/Import: `#3a5a3a`
- Header font: Zilla Slab 600 (Google Fonts)
- Body font: DM Sans 400/500 (Google Fonts)

## vite.config.mts — VitePWA configuration

```typescript
VitePWA({
  registerType: 'autoUpdate',
  includeAssets: ['favicon.ico'],
  manifest: {
    name: 'Shopping List',
    short_name: 'Shopping',
    description: 'My personal shopping list',
    theme_color: '#ffffff',
    icons: [
      {
        src: 'shopping_192x192.png',
        sizes: '192x192',
        type: 'image/png',
      },
      {
        src: 'shopping_512x512.png',
        sizes: '512x512',
        type: 'image/png',
      },
    ],
  },
}),
```

## PWA Icons

- `shopping_192x192.png` and `shopping_512x512.png` are in the `public/` folder
- `apple-touch-icon.png` removed from `includeAssets` — not needed for Android

## tsconfig.app.json Notes

- Contains `"ignoreDeprecations": "6.0"` to suppress a `baseUrl` deprecation warning
- This causes a TypeScript type-checker conflict: `"6.0"` is not yet a valid value for `ignoreDeprecations`, so `vue-tsc` exits with an error
- **Workaround:** do not use `npm run build`; use `npm run build-only` instead, which skips the type-check step and runs Vite directly — this completes cleanly

## Building and Deploying Updates to the Phone

1. `npm run build-only` — produces the `dist/` folder
2. `npm run preview -- --host` — serves the production build on the local network
3. Open the Network URL on the phone (both devices must be on the same Wi-Fi)
4. The service worker detects the update and downloads it in the background
5. Close the app fully on the phone (swipe away from recents) and reopen it — the update will be active

## Export / Import (cross-device data transfer)

The app has Export and Import buttons. The exported `shopping-list.json` contains both `masterList` and `currentTrip`. To move data between devices:
1. Export on the source device — downloads `shopping-list.json`
2. Transfer the file (email, Google Drive, etc.)
3. On Android, when the Import file picker appears, use the hamburger/sidebar to navigate to Downloads, or save the file to Google Drive first and pick it from there
4. Import replaces all local data on the target device

## Known Issues / Notes

- `vite-plugin-pwa` has 4 high severity vulnerabilities in its dependency chain (`serialize-javascript`, `@rollup/plugin-terser`, `workbox-build`) — all in dev/build dependencies only, none in browser-facing code; safe to ignore for a personal project
- `npm audit fix --force` would downgrade `vite-plugin-pwa` to 0.19.8 — do not run it
- The peer dependency mismatch between `vite-plugin-pwa` and Vite 8 is a version declaration lag, not an actual incompatibility
- The build output includes a large number of Roboto font variants (Cyrillic, Greek, Vietnamese, etc.) — this is Vuetify bundling its full font dependency; it is not a problem in practice since everything is cached after first load

## Possible Future Improvements

- If the two-row button layout in the header becomes bothersome, consider shortening button labels: "All"/"Trip" for the toggle, "Clear", "Exp", "Imp" for the others — likely enough to fit on one row
- Alternatively, remove the "Clear Trip" button entirely; manually unchecking items at the end of a trip is a workable alternative
- Publish to GitHub as a backup
- Deploy to a public host (Netlify, GitHub Pages) to make the app accessible outside the home Wi-Fi network
