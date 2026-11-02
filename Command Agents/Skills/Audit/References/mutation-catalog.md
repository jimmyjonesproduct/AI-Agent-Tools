# Bugs to plant, by layer (Versaterm Command)

Plant them in files the target change or flow runs through. A good audit has at least one bug the UI hides, one the API hides, and one that only shows in stored state, logs or the other surface.

## UI only (data is fine, the screen lies)
- A category tally or column count off by one, or counting the wrong category.
- A list that drops its last row, or sorts in the wrong order.
- A status that never refreshes after a realtime update, while the API is correct.
- A `VDSButton` whose label is right but whose `onPress` calls the wrong action.
- An error banner the UI never shows (swallowed 4xx/5xx state).
- A live-region announcement that is removed or announces the wrong value.
- Category or status conveyed by colour only (text label dropped — WCAG 1.4.1).
- Correct in dark mode, broken in light mode (a static `color.*` import inside `makeStyles`).

## API (the response is wrong)
- Wrong status on success (200 instead of 201) or on failure (200 with an error body).
- A validator removed in `mock-backend/` so bad input is accepted.
- A field missing or renamed in the response (wire vs client vocabulary drift).
- `If-Match` ignored, so a stale write returns 200 instead of 409/428.
- An append-only sub-resource that now returns 409.
- `requirePermission` loosened to a weaker key.

## Stored state (the response says yes, nothing is saved)
- The repo write commented out, while the response returns the new object.
- A write that hits the wrong id, or persists a stale copy (version not bumped).
- An audit-chain row not appended, or appended with the wrong `op`.
- A `null` tri-state coerced to `false`/`0` (`||` instead of `??`).

## Compliance (HIPAA / CJIS / tenancy)
- PHI or crew names written to `localStorage` / AsyncStorage.
- An identity field leaking into a log, telemetry event or realtime payload.
- A list read returning identity fields without the sequester.
- A query that bypasses `withTenant` or reads another tenant's row.

## Parity (web ↔ RN)
- A component unmounted from one shell (`prototype-app.jsx` or `DockLayoutScreen.js`) while its tests still pass.
- A change in a shared UMD (`react-native/*.js` loaded by both) so parity still matches but both are wrong — only an answer key catches it.
- A feature flag or `GATED_LEAF_MODULE` change that hides a leaf on one surface only.

## Shared code behind existing flows
- A change in `api-client.js`, a normalizer or `config-schema.js` that breaks an existing flow the new feature never uses.

## Startup (at most one)
- A missing env default or broken import so the backend or Metro fails to start. This only tests the health check.
