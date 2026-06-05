# EVAMED Home Page — Performance Analysis & Fixes

## Problems Identified

### 🔴 HIGH — 15+ API calls fired simultaneously on init
**Where:** `home-evamed.component.ts` constructor + `ngOnInit()`

Every catalog (countries, materials, sections, energy types, conversions, standards, etc.) was fetched from scratch on every page load. The constructor fired ~12 HTTP requests, then `ngOnInit` awaited 16+ more sequentially before rendering any card content.

---

### 🔴 HIGH — Heavy client-side computation per project (blocking UI thread)
**Where:** `ngOnInit()` — `this.calculos.OperacionesDeFase()` called for each project after all data loaded

Full lifecycle impact results (Producción / Construcción / Uso / FinDeVida phases, chart data, sub-stage breakdowns) were recomputed in JavaScript for every project on every page load. This was the primary reason the Results tab was slow.

---

### 🔴 HIGH — Missing DB index on `UserPlatform.email`
**Where:** `backend/evamed-api/projects_api/models.py`

`email` is used as the login lookup key (`searchUser`) but had no index, causing a full table scan on every login and user session check.

---

### 🟠 MEDIUM — No `select_related` / `prefetch_related` on ViewSets (N+1 queries)
**Where:** `backend/evamed-api/projects_api/views.py`

All ViewSets used bare `Model.objects.all()`. When serializers accessed related objects, Django issued a separate SQL query per row — classic N+1 on the backend.

---

### 🟠 MEDIUM — `search_fields` using LIKE on integer/FK columns
**Where:** `backend/evamed-api/projects_api/views.py` — multiple `SearchFilter` ViewSets

DRF `SearchFilter` without the `=` prefix generates `ICONTAINS` (SQL `LIKE`) queries. On integer FK columns this bypasses indexes entirely, causing full table scans even when indexes exist.

---

### 🟠 MEDIUM — No caching on static catalog observables
**Where:** `CatalogsService` and `AnalisisService`

Catalogs like countries, sections, energy types, standards, and conversions never change between page loads but were re-fetched on every component init, including on every navigation back to the home page.

---

### 🟠 MEDIUM — Duplicate `getECDP()` API call in constructor
**Where:** `home-evamed.component.ts` constructor

`endLifeService.getECDP()` was subscribed twice with no caching, firing two identical HTTP requests for the same data every time the page loaded.

---

### 🟠 MEDIUM — Results tab data loaded eagerly for all projects
**Where:** `home-evamed.component.ts` `ngOnInit()`

The full lifecycle impact results for every project were fetched from the backend on page load, even if the user never opened the Results tab. With N projects, this meant N API calls fired upfront whether or not the user needed them.

---

### 🟡 LOW — O(n²) client-side filtering on render
**Where:** `serchSections()`, `serchConstructiveSection()`, `serchEndLifeSection()`

Each iterates the full sections array × the full data array on every card render.

---

### 🟡 LOW — Excessive decimal precision in DB fields
**Where:** `models.py` — `MaterialSchemeProject.value`

Uses `DecimalField(max_digits=45, decimal_places=35)` — slows serialization and numeric comparisons unnecessarily.

---

## Fixes Implemented

### ✅ Fix #1 — DB index on `UserPlatform.email` + fix LIKE queries on FK fields

**Files changed:**
- `backend/evamed-api/projects_api/models.py` — added `db_index=True` to `UserPlatform.email`
- `backend/evamed-api/projects_api/migrations/0078_userplatform_email_index.py` — migration applied
- `backend/evamed-api/projects_api/views.py` — prefixed 12+ `search_fields` entries with `=` to use exact-match instead of `ICONTAINS` on integer/FK fields

**Impact:** Eliminates full table scans on user lookup and all filtered FK queries.

---

### ✅ Fix #2 — `select_related` / `prefetch_related` on all ViewSet querysets

**Files changed:**
- `backend/evamed-api/projects_api/views.py` — added `select_related(...)` or `prefetch_related(...)` to 17 ViewSet querysets

**Impact:** Collapses N+1 queries into single JOINs for all list/detail endpoints. For example, `MaterialSchemeProjectViewSet` previously issued one query per row to fetch `material`, `origin`, `section`, etc.; now fetched in one query.

---

### ✅ Fix #3 — Cache static catalog observables + remove duplicate `getECDP()` call

**Files changed:**
- `frontend/evamed/src/app/core/services/catalogs/catalogs.service.ts` — all 17 read-only catalog methods now use the lazy `shareReplay(1)` pattern; HTTP request fires once per app session
- `frontend/evamed/src/app/core/services/analisis/analisis.service.ts` — same treatment for 11 static catalog methods (`getDB`, `getPotentialTypes`, `getUsefulLife`, `getStandars`, `getMaterials`, `getConversion`, `getPotentialTransport`, `getTypeEnergy`, `getSourceInformation`, `getMaterialSchemeProyect`, `getSectionsList`); mutable per-project endpoints left uncached
- `frontend/evamed/src/app/home-evamed/components/home-evamed/home-evamed.component.ts` — collapsed two `getECDP()` constructor subscriptions into one

**Impact:** Repeated navigation to the home page no longer re-fetches any static reference data. ~28 HTTP calls reduced to at most 1 per catalog per session.

---

### ✅ Fix #4 — Move `OperacionesDeFase()` computation to the backend

**Files changed:**
- `backend/evamed-api/projects_api/views.py` — added `ProjectResultsView` class (full Python port of the JS `OperacionesDeFase` logic: Producción A1/A2/A3 + EPiC, Construcción A4/A5, Uso B4/B6, FinDeVida C1–C4)
- `backend/evamed-api/projects_api/urls.py` — added `projects/<int:project_id>/results/` URL
- `frontend/evamed/src/app/core/services/analisis/analisis.service.ts` — added `getProjectResults(projectId, databases?)` method
- `frontend/evamed/src/app/home-evamed/components/home-evamed/home-evamed.component.ts` — replaced JS computation with backend call; updated `ajusteUsoBaseDatos` to call the API with optional `?databases=` param

**Key optimizations inside `ProjectResultsView`:**
- `MaterialSchemeData` filtered by `material_id__in=material_ids` (no full table scan)
- `SourceInformationData`, `TypeEnergyData`, `Conversions` all filtered to only relevant IDs
- `PotentialTransport` and `Standard` loaded in full (small reference tables)

**Impact:** The Results tab no longer blocks the JS thread. Environmental impact computation moved entirely to the backend.

---

### ✅ Fix #6 — Lazy-load Results tab per project card

**Files changed:**
- `frontend/evamed/src/app/home-evamed/components/home-evamed/home-evamed.component.ts` — `ngOnInit` now builds a skeleton `auxDataProjectList` (with `resultsLoaded: false`) immediately after the project list loads, without fetching any results; added `onTabChange(event, i)` and `loadProjectResults(i)` methods
- `frontend/evamed/src/app/home-evamed/components/home-evamed/home-evamed.component.html` — outer `mat-tab-group` gets `(selectedTabChange)="onTabChange($event, i)"`; Results tab shows a centered Material Design spinner (`<mat-spinner diameter="48">`) until the project's results are loaded
- `frontend/evamed/src/app/home-evamed/components/home-evamed/home-evamed.component.scss` — added `.loading-results` flex centering style

**Impact:** Cards render immediately on page load. The backend results call for a project only fires when the user opens that project's Results tab. Users with many projects see a dramatically faster initial load.

---

## Fixes Pending

| # | Fix | Effort | Impact |
|---|-----|--------|--------|
| 5 | Paginate the projects list (10 per page) | Medium | High (for users with many projects) |
| 7 | Add `Cache-Control` headers to static catalog endpoints in Django | Low | Medium |
