# Occupancy Engine — Design

- Date: 2026-06-28
- Status: Approved (design); implementation not started
- Roadmap: Phase 5 (occupancy) + Phase 7 (real-time), delivered together in one cycle

## Context

GymPulse already ships gym discovery: `GET /api/clubs` (search + city/province filters +
pagination) and `GET /api/clubs/{id}`, with a React list view bound to the live API. The
product's headline feature — estimated crowd levels — is not built yet: there is no occupancy
in club responses, no occupancy endpoint, no simulated engine, and no real-time delivery.

This cycle adds the simulated occupancy engine, surfaces occupancy over REST, pushes live
updates over SignalR, and shows scannable crowd-status badges in the UI.

The supporting docs already pin down the shape: crowd-level enum (`Empty`, `Moderate`, `Busy`,
`Packed`), the `occupancy` response block, an in-memory simulated source, a single-source
interface for later replacement, and `source: "simulated"` / `isEstimated: true` labelling.
See `docs/requirements.md`, `docs/architecture.md`, `docs/api-design.md`, `docs/database_schema.md`.

## Decisions

These were settled during brainstorming:

- **Scope:** full live cycle in one go — engine + REST + SignalR (not split across cycles).
- **Simulation model:** baseline hour-of-week busyness curve, shaped per gym by a stable
  popularity factor, plus small per-tick jitter (not a cumulative random walk).
- **Backend rename:** rename `GoodLifePulse.Api` → `GymPulse.Api` everywhere as a prep step,
  so new occupancy code is born in the right namespace. The README already references
  `backend/GymPulse.Api`, so this aligns code with existing docs.

## Goals

- Show a current estimated crowd level for every active gym, labelled as an estimate.
- Update visible occupancy in real time (target refresh well under 30s) without a page refresh,
  with a polling fallback when a socket can't be held.
- Keep the occupancy source swappable so community reports or a paid API can replace the
  simulator later without changing controllers, DTOs, or the frontend.
- Stand up the project's first automated test suite covering the core success/failure paths.

## Out of scope

- Persisting occupancy to the `OccupancySnapshots` table (schema doc says it stays empty this build).
- User-submitted crowd reports feeding occupancy (later phase).
- Auth, favorites, club-detail page, `Brand`/`OsmId` fields (separate cycles).
- Per-client / per-club SignalR subscription groups (broadcast to all for now).

---

## Prep step — rename `GoodLifePulse.Api` → `GymPulse.Api`

Done first so nothing is renamed twice. Verified safe against the repo: there is **no `.sln`**,
the `.csproj` sets no explicit `<RootNamespace>`/`<AssemblyName>`, and EF migrations are tracked
by `MigrationId` (not namespace/filename), so history is preserved.

Touches:
- `git mv backend/GoodLifePulse.Api` → `backend/GymPulse.Api` (folder).
- `git mv` `GoodLifePulse.Api.csproj` → `GymPulse.Api.csproj`, `GoodLifePulse.Api.http` → `GymPulse.Api.http`.
- Find/replace the `GoodLifePulse` token across all `.cs` files (`namespace`/`using`) and the `.http` file.
- `docker-compose.yml`: `container_name: goodlife-db` → `gympulse-db`, volume `goodlife-db-data` → `gympulse-db-data`
  (renaming the volume orphans old data, harmless — dev startup re-migrates and re-seeds).
- Delete `bin/` and `obj/`, then `dotnet build` so artifacts regenerate as `GymPulse.Api.*`.
- Config files (`appsettings*.json`, `launchSettings.json`) keep their conventional names; launch
  profiles (`http`/`https`/`IIS Express`) carry no brand and need no change.
- Design mockups `website_design/GoodLife_Pulse_*.png` left as-is (cosmetic; out of scope unless requested).

Verification: `dotnet build` succeeds; `dotnet run` boots, applies migrations, seeds clubs,
and serves `/health` and `/api/clubs`.

---

## Section 1 — Engine internals

New namespace `GymPulse.Api.Services.Occupancy`. Each unit has one job.

- **`CrowdLevel` (enum)** — `Empty, Moderate, Busy, Packed`. Serialized to its string name in JSON.
- **`OccupancyReading` (record)** — in-memory value object: `ClubId, Percent (0–100), CrowdLevel,
  IsEstimated (always true), Source ("simulated"), LastUpdatedAt (UTC)`. Distinct from the EF
  `OccupancySnapshot` table entity, which stays unused this build.
- **`CrowdLevelMapper` (pure static)** — `FromPercent(int) → CrowdLevel`:

  | Percent | Level |
  |---------|----------|
  | 0–25    | Empty    |
  | 26–60   | Moderate |
  | 61–85   | Busy     |
  | 86–100  | Packed   |

  (55 → Moderate, matching the API-design example.)
- **`IOccupancyCalculator` → `BaselineOccupancyCalculator`** — `GetBaselinePercent(clubId, utcNow) → int`,
  fully deterministic (no jitter, no clock dependency beyond the passed `utcNow`). Combines:
  - an hour-of-week busyness curve (0..1): mornings ~6–9 and evenings ~16–20 busy, midday moderate,
    late night near-empty, flatter weekend bump;
  - a stable per-gym popularity factor (~0.7–1.1) from a deterministic hash of `clubId`.
- **`IOccupancySource` → `SimulatedOccupancySource` (singleton)** — the swap point. Read interface:
  `Get(clubId)` and `GetMany(ids)`. Wraps a `ConcurrentDictionary<int, OccupancyReading>` plus an
  internal `Update(reading)` the simulator calls. On a miss, `Get`/`GetMany` compute a baseline via
  the calculator, cache it, and return it — **never null** for any id asked (callers only ask for
  ids that came from the DB as active clubs, so validity is guaranteed by the caller). This removes
  the cold-start race without an empty/placeholder state.

**Driver: `OccupancySimulator : BackgroundService` (hosted, singleton)**
- Loads active club IDs at startup (creates a DI scope to read the scoped `AppDbContext`).
- Runs an initial computation immediately, then a `PeriodicTimer` every **15s** (well under 30s).
- Per club per tick: `target = calculator.GetBaselinePercent(id, now)`;
  `percent = clamp(target + jitter(±5), 0, 100)`; `level = CrowdLevelMapper.FromPercent(percent)`;
  write the reading to the source. (Broadcast diff in Section 3.)
- Jitter lives here, not in the calculator: keeps the calculator deterministically testable, while
  per-tick (non-cumulative) jitter keeps values near baseline yet still crosses thresholds so there
  is something to broadcast between hour boundaries.

**DI (Program.cs):** `AddSingleton<IOccupancyCalculator, BaselineOccupancyCalculator>`;
`AddSingleton<SimulatedOccupancySource>` + `AddSingleton<IOccupancySource>(sp => sp.GetRequiredService<SimulatedOccupancySource>())`;
`AddHostedService<OccupancySimulator>`. Interval and jitter as plain constants for now (no `IOptions`).

---

## Section 2 — REST read surface

**New DTOs (`GymPulse.Api.Dtos`):**
- `OccupancyDto(string CrowdLevel, int Percent, bool IsEstimated, string Source, DateTime LastUpdatedAt)` — nested block.
- `ClubOccupancyDto(int ClubId, string CrowdLevel, int Percent, bool IsEstimated, string Source, DateTime LastUpdatedAt)` — standalone endpoint + SignalR payload item.

**`ClubDto` change:** append `OccupancyDto? Occupancy`. Nullable in the contract for forward-compat
(a future reports source may lack data), though the simulated source always populates it. The
frontend `Club` type gains a matching optional `occupancy`.

**Enrichment in `ClubService` (key change):** occupancy lives in memory, not SQL, so it cannot be
projected inside the EF query. Both read methods become two-step:
1. Run the DB query and materialize the club rows.
2. Look up readings from the injected `IOccupancySource` (`GetMany(ids)` for the list, `Get(id)`
   for detail) and map `(club, reading) → ClubDto` in memory.

`ToDto` changes from an EF-translatable expression to an in-memory mapper. `ClubService` gains an
`IOccupancySource` dependency.

**Endpoint:** `GET /api/clubs/{clubId}/occupancy` added to the existing `ClubsController` as
`[HttpGet("{clubId:int}/occupancy")]` (club-scoped sub-resource). Backed by a new
`IClubService.GetClubOccupancyAsync(clubId, ct) → ClubOccupancyDto?`: returns null when the club is
missing or inactive (controller → `NotFound()`, matching the detail endpoint's exclusion of inactive
clubs), otherwise the reading from `IOccupancySource`.

**Behavior:** `isEstimated` always `true`, `source` always `"simulated"` this build; `CrowdLevel`
serialized as its string name; inactive/nonexistent clubs excluded (404 on the endpoint).

---

## Section 3 — SignalR real-time

- **Hub:** `OccupancyHub : Hub` (namespace `GymPulse.Api.Hubs`), mapped at `/hubs/occupancy`. No
  client→server methods; clients only receive. Broadcasts to `Clients.All` (every client shows many
  gyms). Per-club groups are a later optimization.
- **Broadcast policy — one batched event per tick.** Each tick the simulator diffs new readings
  against the previous and collects gyms whose percent or level changed, then sends a single
  `occupancyUpdated` event carrying an **array** of changed readings (each item is a
  `ClubOccupancyDto`). One message per tick rather than N — far less chatter, and it naturally
  quiets down for a future static (reports) source.
- **Wiring (Program.cs):** `builder.Services.AddSignalR()`; `app.MapHub<OccupancyHub>("/hubs/occupancy")`;
  add `.AllowCredentials()` to the existing `FrontendDev` CORS policy (valid because origins are
  explicit, not `AllowAnyOrigin`) for the websocket transport.
- **Simulator ↔ hub seam:** `OccupancySimulator` holds `IHubContext<OccupancyHub>`. Tick: read
  previous readings → compute new → write to source → diff → if anything changed,
  `await hub.Clients.All.SendAsync("occupancyUpdated", changed, stoppingToken)`. The
  "compute next + diff" step is separated from the send so it is unit-testable (Section 5).

---

## Section 4 — Frontend (crowd status + live updates)

**Types (`src/types/club.ts`):** add `CrowdLevel` union and `Occupancy` type; `Club` gains
`occupancy: Occupancy | null`.

**`CrowdBadge` (new, `src/components/CrowdBadge.tsx`):** the scannable status pill.
- Color per level: Empty → green, Moderate → amber, Busy → orange, Packed → red; `null` → neutral
  gray "Unknown".
- Not color-only (accessibility): colored dot **plus** text label; `aria-label` e.g.
  "Crowd level: Moderate, estimated 55%".
- Estimate caveat surfaced (`title="Estimated — not official gym capacity"` / small "est.").
- Last-updated shown as relative time in a `<time dateTime={lastUpdatedAt}>` (satisfies
  "show when last updated").

**`ClubCard` (`src/components/ClubCard.tsx`):** render `<CrowdBadge occupancy={club.occupancy} />`
in the existing `justify-between` header, top-right of the name. No other card changes.

**`useClubsWithLiveOccupancy` hook (new, `src/hooks/`):** owns the data lifecycle so `App` stays
declarative. Loads clubs via `getClubs`, opens a SignalR connection and merges live updates by
`clubId`, drives the polling fallback, and exposes `{ state, connectionStatus }`.

**SignalR client (`src/api/occupancy.ts`):** add `@microsoft/signalr`.
`HubConnectionBuilder().withUrl(`${API_BASE_URL}/hubs/occupancy`).withAutomaticReconnect().build()`.
On `occupancyUpdated` (batched array) immutably merge each reading into the matching club's
`occupancy`. Start on mount, stop on unmount.

**Polling fallback:** `withAutomaticReconnect()` handles transient drops. For "can't hold a socket
at all," the list view re-fetches `getClubs` every ~20s (its response carries occupancy) rather than
firing N single-club requests; polling starts when the connection is `Disconnected` and stops on
reconnect. The single-club `GET /api/clubs/{id}/occupancy` endpoint is the fallback for the future
club-detail view.

**Live status + states:** a small header indicator — green "Live" when connected, "Reconnecting…" /
"Updates paused" otherwise. Existing loading/empty/error states in `App.tsx` stay; `App` consumes the hook.

---

## Section 5 — Testing

First test project in the repo; proportionate, aligned with Phase 5 exit criteria.

**Backend — `backend/GymPulse.Api.Tests/` (xUnit)**, `ProjectReference` to the API (no `.sln`;
CI runs `dotnet test` on the test csproj). Packages: xUnit, `Microsoft.NET.Test.Sdk`,
`Microsoft.AspNetCore.Mvc.Testing`, `Microsoft.EntityFrameworkCore.Sqlite`.

Unit tests:
- `CrowdLevelMapper.FromPercent` — boundaries (0, 25/26, 60/61, 85/86, 100).
- `BaselineOccupancyCalculator.GetBaselinePercent` — determinism, 0–100 range, stable per-gym
  popularity, sane shape (evening peak > 3am trough).
- `SimulatedOccupancySource` — `Get` on miss computes + caches + never null; `GetMany` covers all ids.
- `OccupancySimulator` diff logic — "compute next + determine changed" separated from the hub send;
  changed inputs → expected broadcast set, no change → empty set. Socket transport deferred to Phase 8.

Integration tests (`WebApplicationFactory<Program>`):
- `GET /api/clubs` → each item has an `occupancy` block (valid level, percent 0–100,
  `isEstimated: true`, `source: "simulated"`).
- `GET /api/clubs/{id}/occupancy` valid active club → 200 with `clubId` + fields; missing/inactive → 404.
- DB: override the DbContext registration to **SQLite in-memory** (so `EF.Functions.Like` translates),
  seed a couple of test clubs in the factory.
- No timing flakiness: compute-on-miss gives a stable jitter-free baseline even before the timer ticks.
- Add `public partial class Program { }` to `Program.cs` so the factory can reference the entry point.

**Frontend — light footprint (add Vitest + React Testing Library; full UI-flow tests stay Phase 8):**
- `CrowdBadge`: each level → correct label + accessible name; `null` → "Unknown".
- Occupancy-merge extracted as a pure reducer `mergeOccupancy(clubs, updates)` and unit-tested for
  merge-by-`clubId`.

---

## Documentation reconciliations (do during implementation)

Two places where the implementation intentionally improves on the current docs:
1. `docs/api-design.md`: the `occupancyUpdated` event carries a **batch array** of readings, not a
   single gym. Update the Real-Time Updates section accordingly.
2. `docs/api-design.md` / `docs/architecture.md`: note the list view's polling fallback re-fetches
   `GET /api/clubs`; the single-club `GET /api/clubs/{id}/occupancy` is the detail-view fallback.

## Acceptance criteria

- `GymPulse.Api` builds, boots, migrates, seeds, and serves `/health` + `/api/clubs`; no
  `GoodLifePulse` token remains in source.
- `GET /api/clubs` and `GET /api/clubs/{id}` include a populated `occupancy` block for active clubs.
- `GET /api/clubs/{id}/occupancy` returns the reading for an active club and 404 otherwise.
- A connected browser sees crowd badges change without refresh; killing the socket falls back to
  ~20s list polling and recovers on reconnect.
- Occupancy is clearly labelled an estimate and shows when it was last updated.
- Backend unit + integration tests and frontend unit tests pass via `dotnet test` / `npm test`.
