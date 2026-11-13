# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Traccar web app: a React 19 + Material UI + MapLibre single-page app (plain JavaScript/JSX, no TypeScript) for the Traccar GPS tracking server. It talks to the back end only through the REST API (`api/...`) and the WebSocket at `api/socket`. This repo is usually checked out as the `traccar-web` submodule of the Java server repo (the parent directory).

## Commands

```shell
npm install
npm start            # Vite dev server on :3000, proxies /api and /api/socket to a Traccar server on localhost:8082
npm run build        # production bundle into build/ (CI runs this)
npm run lint         # eslint with --max-warnings 0 (CI runs this; also needs `npm ci --prefix demo`)
npm run lint:fix
npm run demo         # dev server against the fake read-only back end in demo/ (no Java server needed)
```

There is no test suite; lint and build are the checks. Formatting is Prettier via ESLint (single quotes, 100-char lines, 2-space indent), so a formatting problem is a lint failure.

To run against a real back end, start the server from the parent repo (`org.traccar.Main` with `debug.xml`, port 8082), then `npm start`. Note that `debug.xml` serves `traccar-web/simple` (a standalone single-file React 17 demo client), not this app's build, and reads translations from `src/resources/l10n`.

## Architecture

**Bootstrap.** `src/index.jsx` wraps everything in Redux `Provider` → `LocalizationProvider` → theme → `ServerProvider` (fetches `api/server` into `state.session.server` before rendering anything) → `BrowserRouter` → `Navigation`. `Navigation.jsx` holds the whole route table (all pages except `MainPage` are `lazy()`-loaded) and consumes `locale`/`token`/`uniqueId`/`openid` query params. Routes under `/` render `App.jsx`, which loads `api/session` (redirecting to `/login` or `/register` if missing), shows the terms dialog when required, and mounts the headless controllers.

**Live data.** `SocketController.jsx` loads `api/devices` and opens `api/socket`; incoming `devices`/`positions`/`events`/`logs` are dispatched into Redux. If the socket closes it re-fetches over REST and reconnects (close code 4000 means logout). `CachingController.js` preloads geofences, groups, drivers, maintenances and calendars into the store. Pages read live state from Redux and fetch everything else directly.

**Redux (`src/store`).** Redux Toolkit slices, each exporting `fooActions`/`fooReducer` and re-exported from `store/index.js`. Entities are kept as `items` maps keyed by id. `session` holds `server`, `user`, the latest `positions` per device, and logs. `throttleMiddleware` batches high-frequency updates.

**Fetching and errors.** Use relative URLs (`api/...`, not `/api/...`) because the app is served with `base: './'` and a router basename from `document.baseURI`. Use `common/util/fetchOrThrow` (throws on a non-2xx with the response body as the message) together with the helpers in `src/reactHelper.js`:
- `useAsyncTask(async ({ signal }) => {...}, deps)` for effects that load data; it aborts on unmount and pushes errors to the snackbar via `messagesActions`. It may return a cleanup function.
- `useCatch` / `useCatchCallback` for event handlers.
ESLint checks the deps of `useCatchCallback`/`useAsyncTask` (`@eslint-react/exhaustive-deps`).

**Settings and preferences.** Read config through `common/util/preferences.js`: `usePreference` for top-level fields and `useAttributePreference` for `attributes` keys. Both resolve user vs server values and respect `server.forceSettings`. UI feature toggles (`ui.disable*` attributes) come from `common/util/useFeatures` and are always off for administrators. Permission helpers are in `common/util/permissions.js`. The hooks in `common/attributes/` define the known custom attribute keys offered in edit forms.

**Pages by area.** `main/` is the live map with the device list. `settings/` holds CRUD pages built on `settings/components/EditItemView` (an edit page whose load and save go through `api/<endpoint>[/<id>]`) and the list/collection components next to it. `reports/` pages share `reports/components/ReportFilter` and `common/util/exportExcel`. `other/` contains replay, geofences, the event/position detail pages, the emulator, and video streaming (hls.js). `login/` holds the unauthenticated pages.

**Map (`src/map`).** `core/MapView.jsx` creates one module-level MapLibre `map` singleton that is reused across pages (its DOM element moves between pages, never recreated). Feature components (`MapMarkers`, `MapGeofence`, `MapRoutePath`, `main/*`, `control/*`, ...) are mostly render-null components that import `map` directly and add sources and layers in effects (see `core/useMapLayer.js`). Wait for `useMapReady()` when needed. Basemap styles are in `core/useMapStyles.js`, overlays in `overlay/useMapOverlays.js`, geofence drawing in `draw/` (mapbox-gl-draw), and icons are preloaded by `core/preloadImages.js`.

**Styling.** MUI with `tss-react` (`makeStyles()(theme => ...)`). The theme is in `common/theme/`, and RTL is supported via a stylis plugin in `AppThemeProvider.jsx`.

**Localization.** `src/resources/l10n/en.json` is the only source file. Add new keys there and call `const t = useTranslation(); t('key')` from `LocalizationProvider`. The other languages are pulled from Transifex by a CI workflow, so don't edit them by hand. Other locale JSON and dayjs locales are lazy-loaded.

**Server-templated HTML.** `index.html` and the PWA manifest contain `${title}`, `${description}`, `${colorPrimary}` and `<!--${customHead}-->` placeholders. The Java server fills them in (`org.traccar.web.OverrideTextFilter` in the parent repo), and the `demo/` Vite config replaces them too. Keep them intact.

**Native apps.** `common/components/NativeInterface.js` bridges to the Android/iOS wrapper apps (`window.appInterface` / `webkit.messageHandlers.appInterface`) for login tokens, push notifications and server changes.
