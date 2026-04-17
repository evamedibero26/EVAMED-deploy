# EVAMED Performance Optimization Plan

> **Stack:** Angular 19 frontend · Django 2.2 backend · PostgreSQL 12 · Docker Compose
> **Date:** 2026-04-08
> **Status:** Phase 1 complete

---

## Top 5 Bottlenecks (Ranked)

| Rank | Bottleneck | Location | Severity |
|------|-----------|----------|----------|
| 1 | Gunicorn running 1 synchronous worker — all requests queue serially | `docker-compose.yml` | Critical |
| 2 | N+1 ORM queries on every list endpoint — up to 11N+1 queries per request on `MaterialSchemeProject` | `projects_api/views.py` | Critical |
| 3 | Angular bundle built without production config — unminified, source-mapped, 18 Material modules eagerly loaded | `angular.json`, `frontend/Dockerfile` | Critical |
| 4 | `DEBUG=True` + zero caching layer + no persistent DB connections | `settings.py` | Critical |
| 5 | No pagination + full-table scans filtered in JavaScript | `views.py`, `materials-stage.component.ts` | High |

---

## Phase 1: Quick Wins (Complete)

Low effort, high impact. Config changes only — no code rewrites.

| # | Task | File | Status |
|---|------|------|--------|
| 1 | Gunicorn: `--workers 4 --worker-class gthread --threads 2` via `gunicorn.conf.py` | `backend/evamed-api/gunicorn.conf.py` | Done |
| 2 | Add `CONN_MAX_AGE: 60` to `DATABASES` | `settings.py` | Done |
| 3 | `DEBUG = os.getenv('DJANGO_DEBUG', 'False') == 'True'` | `settings.py` | Done |
| 4 | Fix Dockerfile: `npm run build -- --configuration production` | `frontend/evamed/Dockerfile` | Done |
| 5 | Replace `PreloadAllModules` with `NoPreloading` | `app-routing.module.ts` | Done |
| 6 | Add `restart: unless-stopped` to all services | `docker-compose.yml` | Done |
| 7 | Add `healthcheck` on `db` + `depends_on: condition: service_healthy` on `api` | `docker-compose.yml` | Done |
| 8 | Move `CorsMiddleware` to position 1 in `MIDDLEWARE` | `settings.py` | Done |
| 9 | Add `collectstatic --noinput` to startup command | `docker-compose.yml` | Done |

**Expected combined result:** 8× throughput increase, ~60–75% bundle size reduction, stable memory usage.

---

## Phase 2: Medium-Term (Pending)

Estimated effort: 1–2 weeks. Requires code changes to backend and frontend.

| # | Task | Files | Expected Impact |
|---|------|-------|-----------------|
| 7 | Add `select_related` to all ViewSets with FK fields | `projects_api/views.py` | −90% DB queries on list views |
| 8 | Add Redis container + configure `CACHES` + cache static catalog endpoints | `docker-compose.yml`, `settings.py`, `views.py` | −50ms/request for reference data |
| 9 | Add global `PageNumberPagination` (50 items) + update Angular services to handle `{results:[]}` | `settings.py`, Angular services | Response size bounded |
| 10 | Replace per-item loop with `bulk_update` in `MaterialStageUpdateView` | `projects_api/views.py` | −95% queries on save operations |
| 11 | Add composite index on `Material.database_from` + `MaterialSchemeProject.project_id+section_id` | `projects_api/models.py` | Faster filtered queries |
| 12 | Move `database_from` filter to server-side via `django-filter` | `views.py`, Angular services | Eliminates full-table download |

### Key Details for Phase 2

**Fix #7 — select_related on MaterialSchemeProject:**
```python
# projects_api/views.py
class MaterialSchemeProjectViewSet(viewsets.ModelViewSet):
    queryset = models.MaterialSchemeProject.objects.select_related(
        'material_id', 'project_id', 'origin_id', 'section_id',
        'city_id_origin', 'state_id_origin', 'city_id_end',
        'transport_id_origin', 'transport_id_end',
    )
```

**Fix #8 — Redis cache:**
```python
# settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://redis:6379/0',
        'TIMEOUT': 300,
    }
}
```
```yaml
# docker-compose.yml
redis:
  image: redis:7-alpine
  restart: unless-stopped
  deploy:
    resources:
      limits:
        memory: 128M
```
Apply `@cache_page(60 * 60)` to: `CountryViewSet`, `StateViewSet`, `TransportViewSet`, `SectionViewSet`, `OriginViewSet`.

**Fix #9 — Global pagination:**
```python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 50,
    'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend'],
}
```

**Fix #10 — bulk_update in MaterialStageUpdateView:**
```python
# Before: 40 round-trips for 20 items
for item in item_updates:
    selection = models.MaterialStageSystemSelection.objects.filter(...).first()
    selection.save(update_fields=['is_selected'])

# After: 2 queries total
ids = [item['selection_id'] for item in item_updates]
selections = {s.id: s for s in models.MaterialStageSystemSelection.objects.filter(
    id__in=ids, project_id_id=project_id
)}
for item in item_updates:
    if s := selections.get(item['selection_id']):
        s.is_selected = item['is_selected']
models.MaterialStageSystemSelection.objects.bulk_update(selections.values(), ['is_selected'])
```

**Fix #11 — Indexes:**
```python
# projects_api/models.py
class Material(models.Model):
    database_from = models.CharField(max_length=255, db_index=True)
    class Meta:
        indexes = [
            models.Index(fields=['database_from', 'name_material']),
        ]

class MaterialSchemeProject(models.Model):
    class Meta:
        indexes = [
            models.Index(fields=['project_id', 'section_id']),
        ]
```

**Fix #12 — Server-side filter:**
```python
# projects_api/views.py
import django_filters

class MaterialFilter(django_filters.FilterSet):
    database_from = django_filters.CharFilter(lookup_expr='exact')
    class Meta:
        model = models.Material
        fields = ['database_from']

class MaterialViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = models.Material.objects.all()
    filterset_class = MaterialFilter
```

---

## Phase 3: Long-Term Architecture (Pending)

Estimated effort: 1–2 months. Significant refactoring required.

| # | Task | Impact |
|---|------|--------|
| 13 | Add nginx reverse proxy in front of Gunicorn and the Express static server | −40% response time for static assets, gzip, HTTP/2 |
| 14 | Implement `OnPush` change detection on the 10 most complex components | −30–60% Angular CPU usage |
| 15 | Upgrade Django 2.2 → 4.2 LTS | Security patches, async views, improved ORM |
| 16 | Replace `MaterialModule` barrel import with per-component Material imports | −200–400 KB initial bundle |
| 17 | Add `trackBy` to all `*ngFor` loops on large data tables | Eliminates full list re-renders on unrelated state changes |
| 18 | Upgrade PostgreSQL 12 → 16 (12 is EOL Nov 2024) | Security patches, query planner improvements |

### Key Details for Phase 3

**Fix #14 — OnPush change detection (top priority components):**
```typescript
// Apply to: materials-stage, analisis, results, home-evamed components
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
```

**Fix #16 — Replace MaterialModule barrel:**
Instead of importing `MaterialModule` in `SharedModule`, import only the specific Angular Material modules needed in each feature module (e.g., `MatTableModule` only in modules that use `<mat-table>`).

**Fix #17 — trackBy in ngFor:**
```typescript
// In component:
trackById(index: number, item: any): number { return item.id; }
```
```html
<!-- In template: -->
<tr *ngFor="let item of items; trackBy: trackById">
```

**Fix #13 — nginx config (example):**
```nginx
server {
    listen 80;
    gzip on;
    gzip_types text/plain application/javascript text/css application/json;

    location /api-projects/ {
        proxy_pass http://api:8000;
        proxy_set_header Host $host;
    }

    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## Known Secondary Issues (Not Yet Addressed)

These are real bugs/risks found during analysis that don't block performance work but should be tracked:

| Issue | Location | Notes |
|-------|----------|-------|
| `autosave` calls `JSON.stringify` on full Excel parse result every 5s | `materials-stage.component.ts` | Blocks main thread; replace `setInterval` with `debounceTime(10_000)` |
| 4 HTTP calls fired in constructor instead of `ngOnInit` | `materials-stage.component.ts` | Move to `ngOnInit`, combine with `forkJoin` |
| 210 `console.log` calls remain in production build | Various components | Add `terser` drop config or ESLint `no-console` rule |
| Same endpoint hit by 3 different services with no shared cache | `MaterialsService`, `AnalisisService`, `ProjectsService` | Consolidate or add `shareReplay(1)` |
| `TreatmentOfGeneratedWasteSerializer.create` maps `project_id` to wrong field (`stage`) | `projects_api/serializers.py` | Data corruption bug |
| `ExternalDistanceSerializer.create` sets `region` from `country_id_origin` | `projects_api/serializers.py` | Data corruption bug |
| `UserPlatform.password` stored in plaintext | `projects_api/models.py` | Security issue |
| No authentication/permissions on any `projects_api` ViewSet | `projects_api/views.py` | Any unauthenticated user can read/write all project data |
| `environment.prod.ts` API URL points to `localhost:8000` | `src/environments/environment.prod.ts` | Will break in any real deployment where API is on a different host |
| Firebase credentials hardcoded in environment files | `src/environments/` | Rotate keys, move to env vars |
| Django 2.2 is EOL (April 2022) | `requirements.txt` | No security patches |
| PostgreSQL 12 is EOL (November 2024) | `docker-compose.yml` | No security patches |
