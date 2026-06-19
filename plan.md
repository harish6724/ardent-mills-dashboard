# Ardent Mills Supply Chain Dashboard — PoC Implementation Plan

## Overview

A full-stack proof-of-concept application for an architect-role interview at Ardent Mills. Demonstrates end-to-end architect-level thinking: Angular 22 frontend with modern patterns (Signals, standalone components, functional interceptors, `httpResource`), an ASP.NET Core REST API, Entity Framework Core, and SQL Server.

**Business domain**: Ardent Mills sources grain from farms, mills it into flour/grain products at multiple mill locations across North America, and distributes to food company customers. The dashboard visualizes this supply chain end-to-end.

---

## Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| Frontend framework | Angular 22 (latest) | Latest version; standalone components, Signals, `httpResource` |
| UI component library | Angular Material 22 (M3) | Polished enterprise look, first-class Angular integration |
| Charts | ng2-charts v7 (Chart.js) | Supports Angular 22; ngx-charts only supports ≤ Angular 21 |
| State management | Angular Signals (`signal`, `computed`, `effect`) | Native Angular 17+; lighter than NgRx for PoC scale |
| Styling | SCSS + Angular Material M3 theme | Custom brand colors (Wheat Gold, Grain Brown) |
| Backend framework | ASP.NET Core Web API (.NET 9) | Enterprise Microsoft stack; likely what Ardent Mills uses |
| ORM | Entity Framework Core 9 | Code-first, migrations, LINQ queries |
| Database | SQL Server (LocalDB for dev) | Zero-install for demo; same provider as production SQL Server |
| API docs | Swagger / OpenAPI (Swashbuckle) | Self-documenting, browsable API during demo |

---

## Solution Structure

```
ardent-mills-dashboard/
  ArdentMills.sln
  ArdentMills.Api/               ← ASP.NET Core Web API
  ardent-mills-ui/               ← Angular 22 workspace
  README.md
```

---

## Part 1: ASP.NET Core Backend (`ArdentMills.Api`)

### 1.1 Bootstrap

```bash
dotnet new webapi -n ArdentMills.Api --framework net9.0
cd ArdentMills.Api
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Swashbuckle.AspNetCore
```

### 1.2 Project Structure

```
ArdentMills.Api/
  Controllers/
    ShipmentsController.cs
    MillsController.cs
    QualityController.cs
    DashboardController.cs
  Services/
    IShipmentService.cs  +  ShipmentService.cs
    IMillService.cs      +  MillService.cs
    IQualityService.cs   +  QualityService.cs
    IDashboardService.cs +  DashboardService.cs
  Data/
    AppDbContext.cs
    Seed/
      ShipmentSeed.cs
      MillSeed.cs
      QualityTestSeed.cs
  Models/                        ← EF Core entities
    Shipment.cs
    Mill.cs
    MillAlert.cs
    QualityTest.cs
  DTOs/                          ← API response shapes (decoupled from entities)
    ShipmentDto.cs
    ShipmentDetailDto.cs
    MillSummaryDto.cs
    MillDetailDto.cs
    QualityTestDto.cs
    DashboardKpisDto.cs
    TrendPointDto.cs
    PieSliceDto.cs
  Migrations/                    ← EF Core auto-generated
  Program.cs
  appsettings.json
  appsettings.Development.json
```

### 1.3 EF Core Entity Models

```csharp
// Models/Shipment.cs
public class Shipment
{
    public string Id { get; set; }              // "SHP-2026-04821"
    public string LoadNumber { get; set; }      // "LD-44821"
    public string GrainType { get; set; }       // "Hard Red Winter Wheat"
    public string ProductSku { get; set; }      // "AM-APF-50"
    public string ProductName { get; set; }     // "All-Purpose Bleached Flour 50lb"
    public int QuantityLbs { get; set; }        // 22000–48000 (truck vs. rail)
    public string OriginMillId { get; set; }
    public string OriginMillName { get; set; }
    public string DestinationCompany { get; set; } // "Bimbo Bakeries USA"
    public string DestinationCity { get; set; }
    public string DestinationState { get; set; }
    public string Carrier { get; set; }         // "BNSF Railway" | "Schneider National"
    public string Mode { get; set; }            // "Rail" | "Truck" | "Intermodal"
    public string Status { get; set; }          // "In Transit" | "Delivered" | "Delayed" | "Scheduled"
    public DateTime ScheduledDeparture { get; set; }
    public DateTime ScheduledArrival { get; set; }
    public DateTime? ActualArrival { get; set; }
    public string PoNumber { get; set; }        // "PO-2026-558321"
}

// Models/Mill.cs
public class Mill
{
    public string Id { get; set; }              // "MILL-DEN"
    public string Name { get; set; }            // "Denver Mill"
    public string City { get; set; }
    public string State { get; set; }
    public string Status { get; set; }          // "Operational" | "Reduced Capacity" | "Maintenance" | "Offline"
    public int CapacityTonsPerDay { get; set; }
    public int CurrentProductionTons { get; set; }
    public double EfficiencyPct { get; set; }   // 70–98
    public DateTime LastInspection { get; set; }
    public ICollection<MillAlert> Alerts { get; set; } = [];
}

// Models/MillAlert.cs
public class MillAlert
{
    public int Id { get; set; }
    public string MillId { get; set; }
    public string Severity { get; set; }        // "info" | "warning" | "critical"
    public string Message { get; set; }
    public DateTime RaisedAt { get; set; }
    public Mill Mill { get; set; }
}

// Models/QualityTest.cs
public class QualityTest
{
    public string Id { get; set; }              // "QC-2026-001234"
    public string BatchId { get; set; }         // "BAT-DEN-04821"
    public string MillId { get; set; }
    public string MillName { get; set; }
    public string GrainType { get; set; }
    public DateTime TestDate { get; set; }
    public double ProteinPct { get; set; }      // spec varies by flour type
    public double MoisturePct { get; set; }     // >14.0 = rejection for storage
    public double AshPct { get; set; }          // lower = whiter flour; whole wheat higher
    public int FallingNumberSec { get; set; }   // <250 = sprout damage (reject)
    public double TestWeightLbBu { get; set; }  // 58–62 lb/bushel
    public double DockagePct { get; set; }      // 0.0–2.0% foreign material
    public string Grade { get; set; }           // "US No. 1" .. "Sample Grade"
    public bool Passed { get; set; }
    public string Inspector { get; set; }
}
```

### 1.4 DbContext

```csharp
// Data/AppDbContext.cs
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Shipment> Shipments => Set<Shipment>();
    public DbSet<Mill> Mills => Set<Mill>();
    public DbSet<MillAlert> MillAlerts => Set<MillAlert>();
    public DbSet<QualityTest> QualityTests => Set<QualityTest>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        // Indexes
        b.Entity<Shipment>().HasIndex(s => s.Status);
        b.Entity<Shipment>().HasIndex(s => s.OriginMillId);
        b.Entity<MillAlert>().HasOne(a => a.Mill)
            .WithMany(m => m.Alerts).HasForeignKey(a => a.MillId);

        // Seed data (call static seed helpers)
        b.Entity<Mill>().HasData(MillSeed.Data);
        b.Entity<MillAlert>().HasData(MillSeed.Alerts);
        b.Entity<Shipment>().HasData(ShipmentSeed.Data);
        b.Entity<QualityTest>().HasData(QualityTestSeed.Data);
    }
}
```

### 1.5 Connection String (`appsettings.Development.json`)

```json
{
  "ConnectionStrings": {
    "Default": "Server=(localdb)\\mssqllocaldb;Database=ArdentMillsDev;Trusted_Connection=True;"
  }
}
```

**Dev**: SQL Server LocalDB — ships with Visual Studio / SQL Server Express, zero install.  
**Prod**: swap for Azure SQL or on-prem SQL Server connection string, same EF Core provider.

### 1.6 Controllers (thin — delegate to services)

```csharp
// Controllers/ShipmentsController.cs
[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class ShipmentsController(IShipmentService svc) : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IEnumerable<ShipmentDto>>> GetAll(
        [FromQuery] string? status,
        [FromQuery] string? grainType,
        [FromQuery] string? origin,
        [FromQuery] string? destination)
        => Ok(await svc.GetAllAsync(status, grainType, origin, destination));

    [HttpGet("{id}")]
    public async Task<ActionResult<ShipmentDetailDto>> GetById(string id)
    {
        var dto = await svc.GetByIdAsync(id);
        return dto is null ? NotFound() : Ok(dto);
    }
}
```

`MillsController`, `QualityController`, and `DashboardController` follow the same pattern.  
`DashboardController` has three endpoints: `/api/dashboard/kpis`, `/api/dashboard/trend`, `/api/dashboard/grain-mix`.

### 1.7 Service Layer

Controllers are thin; all EF Core queries and business logic live in services.

```csharp
// Services/IShipmentService.cs
public interface IShipmentService
{
    Task<IEnumerable<ShipmentDto>> GetAllAsync(
        string? status, string? grainType, string? origin, string? destination);
    Task<ShipmentDetailDto?> GetByIdAsync(string id);
}

// Services/ShipmentService.cs
public class ShipmentService(AppDbContext db) : IShipmentService
{
    public async Task<IEnumerable<ShipmentDto>> GetAllAsync(...)
    {
        var q = db.Shipments.AsQueryable();
        if (!string.IsNullOrEmpty(status))      q = q.Where(s => s.Status == status);
        if (!string.IsNullOrEmpty(grainType))   q = q.Where(s => s.GrainType == grainType);
        if (!string.IsNullOrEmpty(origin))      q = q.Where(s => s.OriginMillName.Contains(origin));
        if (!string.IsNullOrEmpty(destination)) q = q.Where(s => s.DestinationCompany.Contains(destination));
        return await q.Select(s => new ShipmentDto { ... }).ToListAsync();
    }
    // ...
}
```

### 1.8 `Program.cs` Registrations

```csharp
builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddScoped<IShipmentService, ShipmentService>();
builder.Services.AddScoped<IMillService, MillService>();
builder.Services.AddScoped<IQualityService, QualityService>();
builder.Services.AddScoped<IDashboardService, DashboardService>();

builder.Services.AddCors(o => o.AddDefaultPolicy(p =>
    p.WithOrigins("http://localhost:4200")
     .AllowAnyMethod()
     .AllowAnyHeader()));

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// ...

app.UseCors();
app.UseSwagger();
app.UseSwaggerUI();
app.MapControllers();
```

### 1.9 Migrations and Seed

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
dotnet run
# → http://localhost:5000/swagger  verify all endpoints return seed data
```

---

## Part 2: Angular 22 Frontend (`ardent-mills-ui`)

### 2.1 Bootstrap

```bash
npm install -g @angular/cli@latest          # Angular 22

ng new ardent-mills-ui --routing --style=scss --strict --ssr=false

cd ardent-mills-ui
ng add @angular/material
# Prompts: Custom theme | Typography: Yes | Animations: Include and Enable

ng generate @angular/material:m3-theme     # generates m3-theme.scss

npm install ng2-charts chart.js            # Angular 22-compatible chart library
```

> **Chart library decision**: `ngx-charts` peer deps cap at Angular 21 and do not support Angular 22. `ng2-charts` v7 (Chart.js) supports Angular 22. SVG vs. canvas is a trade-off; for an architect interview, cite that ecosystem compatibility takes priority over rendering preference, and flag Apache ECharts as the long-term evaluation candidate.

### 2.2 Angular Project Structure

```
src/
  app/
    app.component.ts             ← root; only <router-outlet>
    app.config.ts                ← ApplicationConfig; all providers
    app.routes.ts                ← lazy routes

    core/
      models/                    ← TypeScript interfaces matching API DTOs
        shipment.model.ts
        mill.model.ts
        quality-test.model.ts
        kpi.model.ts
        common.model.ts          ← PageRequest, PaginatedResult<T>, SortConfig
      interceptors/
        auth.interceptor.ts      ← adds Bearer header
        loading.interceptor.ts   ← LoadingService.start/stop + queueMicrotask guard
        error.interceptor.ts     ← catches 4xx/5xx → NotificationService.error()
      guards/
        feature-flag.guard.ts    ← CanMatchFn factory
      services/
        loading.service.ts       ← signal<number> + computed isLoading
        notification.service.ts  ← MatSnackBar wrapper
        feature-flags.service.ts ← signal<Record<string,boolean>> + isEnabled()
      api/
        shipments.api.ts         ← GET /api/shipments, /api/shipments/:id
        mills.api.ts
        quality.api.ts
        dashboard.api.ts

    shared/
      components/
        kpi-card/                ← dumb; OnPush; signal inputs
        status-badge/
        page-header/
        empty-state/
        loading-bar/             ← thin MatProgressBar stripe at top of page
      pipes/
        grade-color.pipe.ts      ← grade string → CSS class
        short-tons.pipe.ts       ← 44000 → "22.0 T"

    layout/
      shell/
        shell.component.ts       ← MatSidenav responsive shell
        components/
          sidenav-nav/           ← dumb; receives navItems[] @Input
          topbar/                ← dumb; emits hamburger toggle

    features/
      dashboard/                 ← lazy-loaded
        dashboard.component.ts   ← smart container
        dashboard.store.ts       ← forkJoin kpis + trend + grain-mix
        components/
          shipments-trend-chart/ ← dumb; ng2-charts line chart
          grain-mix-chart/       ← dumb; ng2-charts doughnut chart
      shipments/                 ← lazy-loaded
        shipments.routes.ts
        shipments.store.ts
        shipments-list/          ← smart container
        shipment-detail/         ← smart; httpResource pattern
        components/
          shipments-filter-form/ ← dumb; reactive form; emits ShipmentFilter
          shipments-table/       ← dumb; mat-table + paginator
      mills/                     ← lazy-loaded
        mills.routes.ts
        mills.store.ts
        mills-grid/              ← smart container
        mill-detail/             ← smart; httpResource pattern
        components/
          mill-card/             ← dumb; mat-card + progress bar
      quality/                   ← lazy-loaded
        quality.routes.ts
        quality.store.ts
        quality-list/            ← smart container; mat-table + filters

  environments/
    environment.ts               ← { apiBaseUrl: 'http://localhost:5000' }
    environment.prod.ts          ← { apiBaseUrl: '/api' }

  styles/
    _tokens.scss                 ← CSS custom properties (brand + status colors)
    _theme.scss                  ← M3 theme overrides
  styles.scss
```

### 2.3 `app.config.ts`

```ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(
      APP_ROUTES,
      withComponentInputBinding(),          // :id route param → typed input() signal
      withViewTransitions(),
      withInMemoryScrolling({ scrollPositionRestoration: 'enabled' }),
    ),
    provideAnimationsAsync(),
    provideHttpClient(
      withInterceptors([
        authInterceptor,      // 1. adds Bearer header
        loadingInterceptor,   // 2. signals loading state (queueMicrotask guard)
        errorInterceptor,     // 3. catches HTTP errors → MatSnackBar
      ]),
    ),
    provideNativeDateAdapter(),
    { provide: MAT_FORM_FIELD_DEFAULT_OPTIONS,
      useValue: { appearance: 'outline', subscriptSizing: 'dynamic' } },
  ],
};
```

### 2.4 `app.routes.ts`

```ts
export const APP_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./layout/shell/shell.component').then(m => m.ShellComponent),
    children: [
      { path: '', pathMatch: 'full', redirectTo: 'dashboard' },
      {
        path: 'dashboard',
        loadComponent: () => import('./features/dashboard/dashboard.component')
          .then(m => m.DashboardComponent),
        title: 'Dashboard | Ardent Mills',
      },
      {
        path: 'shipments',
        loadChildren: () => import('./features/shipments/shipments.routes')
          .then(m => m.SHIPMENTS_ROUTES),
      },
      {
        path: 'mills',
        canMatch: [featureFlagGuard('mills')],
        loadChildren: () => import('./features/mills/mills.routes')
          .then(m => m.MILLS_ROUTES),
      },
      {
        path: 'quality',
        canMatch: [featureFlagGuard('quality')],
        loadChildren: () => import('./features/quality/quality.routes')
          .then(m => m.QUALITY_ROUTES),
      },
    ],
  },
  { path: '**', redirectTo: 'dashboard' },
];
```

### 2.5 Core Services

#### `LoadingService`
```ts
@Injectable({ providedIn: 'root' })
export class LoadingService {
  private readonly _pending = signal(0);
  readonly isLoading = computed(() => this._pending() > 0);

  start(): void { this._pending.update(n => n + 1); }
  stop(): void  { this._pending.update(n => Math.max(0, n - 1)); }
}
```

#### `FeatureFlagsService`
```ts
@Injectable({ providedIn: 'root' })
export class FeatureFlagsService {
  private readonly _flags = signal<Record<string, boolean>>({
    mills:   true,
    quality: true,
    reports: false,    // disabled — placeholder for a feature not yet built
  });

  isEnabled(flag: string): boolean { return this._flags()[flag] ?? false; }
}
```

### 2.6 Interceptors

#### `auth.interceptor.ts`
```ts
export const authInterceptor: HttpInterceptorFn = (req, next) =>
  next(req.clone({ setHeaders: { Authorization: 'Bearer demo-token-ardent-mills' } }));
```

#### `loading.interceptor.ts`
```ts
export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  const loading = inject(LoadingService);
  // queueMicrotask prevents NG0600 (signal write during change detection)
  // when HTTP completes synchronously in tests/mocks
  queueMicrotask(() => loading.start());
  return next(req).pipe(
    finalize(() => queueMicrotask(() => loading.stop()))
  );
};
```

#### `error.interceptor.ts`
```ts
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const notify = inject(NotificationService);
  return next(req).pipe(
    catchError((err: HttpErrorResponse) => {
      const msg = err.status === 0
        ? 'Network error — is the API running?'
        : `Error ${err.status}: ${err.statusText}`;
      notify.error(msg);
      return throwError(() => err);
    })
  );
};
```

#### `feature-flag.guard.ts`
```ts
export function featureFlagGuard(flag: string): CanMatchFn {
  return () => {
    const flags  = inject(FeatureFlagsService);
    const router = inject(Router);
    return flags.isEnabled(flag) || router.createUrlTree(['/dashboard']);
  };
}
```

### 2.7 Signal Store Pattern (per feature)

```ts
// features/shipments/shipments.store.ts
@Injectable({ providedIn: 'root' })
export class ShipmentsStore {
  private readonly api    = inject(ShipmentsApi);
  private readonly notify = inject(NotificationService);

  private readonly _all     = signal<Shipment[]>([]);
  private readonly _filter  = signal<ShipmentFilter>({
    origin: '', destination: '', status: 'All', grainType: 'All'
  });
  private readonly _page    = signal<PageRequest>({ pageIndex: 0, pageSize: 10 });
  private readonly _loading = signal(false);

  readonly loading  = this._loading.asReadonly();
  readonly filter   = this._filter.asReadonly();

  readonly filtered = computed<Shipment[]>(() => {
    const f = this._filter();
    return this._all().filter(s =>
      (!f.origin      || s.originMillName.toLowerCase().includes(f.origin.toLowerCase())) &&
      (!f.destination || s.destinationCompany.toLowerCase().includes(f.destination.toLowerCase())) &&
      (f.status   === 'All' || s.status   === f.status) &&
      (f.grainType === 'All' || s.grainType === f.grainType)
    );
  });

  readonly total = computed(() => this.filtered().length);

  readonly paged = computed<Shipment[]>(() => {
    const { pageIndex, pageSize } = this._page();
    return this.filtered().slice(pageIndex * pageSize, (pageIndex + 1) * pageSize);
  });

  load(): void {
    this._loading.set(true);
    this.api.getAll().subscribe({
      next: data => { this._all.set(data); this._loading.set(false); },
      error: ()   => { this.notify.error('Failed to load shipments'); this._loading.set(false); },
    });
  }

  setFilter(patch: Partial<ShipmentFilter>): void {
    this._filter.update(f => ({ ...f, ...patch }));
    this._page.update(p => ({ ...p, pageIndex: 0 }));   // reset to page 1 on filter change
  }

  setPage(e: PageEvent): void {
    this._page.set({ pageIndex: e.pageIndex, pageSize: e.pageSize });
  }
}
```

`DashboardStore` uses `forkJoin`:
```ts
load(): void {
  this._loading.set(true);
  forkJoin({
    kpis:     this.api.getKpis(),
    trend:    this.api.getTrend(),
    grainMix: this.api.getGrainMix(),
  }).subscribe({
    next: ({ kpis, trend, grainMix }) => {
      this._kpis.set(kpis);
      this._trend.set(trend);
      this._grainMix.set(grainMix);
      this._loading.set(false);
    },
    error: () => { this.notify.error('Dashboard failed to load'); this._loading.set(false); },
  });
}
```

`MillsStore` and `QualityStore` follow the same structure as `ShipmentsStore`.

### 2.8 `httpResource` for Detail Pages (Angular 19+)

```ts
// features/shipments/shipment-detail/shipment-detail.component.ts
@Component({
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  // ...
})
export class ShipmentDetailComponent {
  // withComponentInputBinding() maps :id route param directly to this signal input
  readonly id = input.required<string>();

  private http = inject(HttpClient);

  // Re-fetches automatically when id() changes; auto-cancels on navigation
  shipment = httpResource<Shipment>(() => `/api/shipments/${this.id()}`);

  // Template: shipment.value(), shipment.isLoading(), shipment.error()
}
```

`MillDetailComponent` uses the same pattern. This replaces the `effect(() => subscribe())` anti-pattern.

### 2.9 Reactive Filter Form (dumb component)

```ts
// components/shipments-filter-form/shipments-filter-form.component.ts
@Component({ standalone: true, changeDetection: ChangeDetectionStrategy.OnPush, ... })
export class ShipmentsFilterFormComponent implements OnInit {
  filterChange = output<Partial<ShipmentFilter>>();

  form = new FormGroup({
    origin:      new FormControl(''),
    destination: new FormControl(''),
    status:      new FormControl<ShipmentStatus | 'All'>('All'),
    grainType:   new FormControl<GrainType | 'All'>('All'),
  });

  private destroyRef = inject(DestroyRef);

  ngOnInit(): void {
    this.form.valueChanges.pipe(
      debounceTime(250),
      distinctUntilChanged((a, b) => JSON.stringify(a) === JSON.stringify(b)),
      takeUntilDestroyed(this.destroyRef),
    ).subscribe(v => this.filterChange.emit(v as Partial<ShipmentFilter>));
  }

  clear(): void { this.form.reset({ status: 'All', grainType: 'All' }); }
}
```

Smart parent wires it: `<app-shipments-filter-form (filterChange)="store.setFilter($event)" />`

### 2.10 Chart Components (ng2-charts)

```ts
// features/dashboard/components/shipments-trend-chart/shipments-trend-chart.component.ts
@Component({ standalone: true, imports: [BaseChartDirective], changeDetection: OnPush })
export class ShipmentsTrendChartComponent {
  data = input.required<{ month: string; value: number }[]>();

  chartData = computed<ChartData<'line'>>(() => ({
    labels: this.data().map(d => d.month),
    datasets: [{
      label: 'Shipments',
      data: this.data().map(d => d.value),
      borderColor: '#D4A017',
      backgroundColor: 'rgba(212,160,23,0.1)',
      fill: true,
      tension: 0.3,
    }],
  }));

  chartOptions: ChartOptions<'line'> = {
    responsive: true,
    plugins: { legend: { display: false } },
    scales: { y: { beginAtZero: false } },
  };
}
```

`GrainMixChartComponent` uses `'doughnut'` type similarly.

### 2.11 Smart / Dumb Component Split

All components use `changeDetection: ChangeDetectionStrategy.OnPush`.

| Smart container | Dumb presentational |
|---|---|
| `DashboardComponent` (injects DashboardStore) | `KpiCardComponent`, `ShipmentsTrendChartComponent`, `GrainMixChartComponent` |
| `ShipmentsListComponent` (injects ShipmentsStore) | `ShipmentsFilterFormComponent`, `ShipmentsTableComponent`, `StatusBadgeComponent` |
| `ShipmentDetailComponent` (httpResource) | — |
| `MillsGridComponent` (injects MillsStore) | `MillCardComponent` |
| `MillDetailComponent` (httpResource) | — |
| `QualityListComponent` (injects QualityStore) | — |

### 2.12 Angular Material Component Mapping

| Feature | Material components used |
|---|---|
| Shell | `MatSidenav`, `MatSidenavContainer`, `MatToolbar`, `MatNavList`, `MatListItem` |
| Responsive nav | `BreakpointObserver` from `@angular/cdk/layout`; `Breakpoints.Handset` → `mode="over"`, desktop → `mode="side"` |
| KPI cards | `MatCard`, `MatIcon` (trend arrows), `MatDivider` |
| Shipments list | `MatTable`, `MatSort`, `MatPaginator`, `MatFormField`, `MatInput`, `MatSelect` |
| Shipment detail | `MatCard`, `MatList`, `MatListItem`, `MatChip` (lot codes) |
| Mills grid | CSS Grid + `MatCard`, `MatProgressBar` (utilization), `MatChipListbox` (status filter), `MatBadge` (alert count) |
| Quality table | `MatTable`, `MatSort`, `MatSlideToggle` ("Failures only"), `MatTooltip` on grade cells |
| Global | `MatSnackBar` (NotificationService), `MatProgressBar` (loading bar stripe), `MatIconButton` |

### 2.13 Theme

```scss
// src/styles/_tokens.scss
:root {
  --am-brand-gold:           #D4A017;   /* Wheat Gold */
  --am-brand-grain:          #6B4423;   /* Grain Brown */
  --am-status-operational:   #2e7d32;
  --am-status-reduced:       #f57f17;
  --am-status-maintenance:   #e65100;
  --am-status-offline:       #c62828;
  --am-status-delivered:     #2e7d32;
  --am-status-in-transit:    #1565c0;
  --am-status-delayed:       #c62828;
  --am-status-scheduled:     #6a1b9a;
  --am-pass:                 #2e7d32;
  --am-fail:                 #c62828;
  --am-warn:                 #f57f17;
}
```

---

## Domain Seed Data

### Mill Locations (10)

| ID | Name | City, ST | Cap t/day | Status |
|---|---|---|---|---|
| MILL-DEN | Denver Mill | Denver, CO | 1,800 | Operational |
| MILL-FTW | Fort Worth Mill | Fort Worth, TX | 1,400 | Operational |
| MILL-HOP | Hopkinsville Mill | Hopkinsville, KY | 1,600 | Operational |
| MILL-NEW | Newton Mill | Newton, KS | 2,200 | Operational |
| MILL-HAS | Hastings Mill | Hastings, NE | 900 | **Maintenance** |
| MILL-ALB | Albany Mill | Albany, OR | 1,100 | Operational |
| MILL-DEC | Decatur Mill | Decatur, AL | 1,500 | Operational |
| MILL-MTP | Mount Pocono Mill | Mount Pocono, PA | 1,000 | **Reduced Capacity** |
| MILL-CHL | Charlotte Mill | Charlotte, NC | 1,300 | Operational |
| MILL-MOD | Modesto Mill | Modesto, CA | 1,100 | **Reduced Capacity** |

### Product SKUs

| SKU | Product | Grain |
|---|---|---|
| AM-APF-50 | All-Purpose Bleached Flour 50lb | Hard Red Winter Wheat |
| AM-BRD-50 | High-Gluten Bread Flour 50lb | Hard Red Spring Wheat |
| AM-WHL-25 | Stone-Ground Whole Wheat 25lb | Hard Red Winter Wheat |
| AM-DUR-50 | Durum Semolina 50lb | Durum Wheat |
| AM-PST-25 | Pastry Flour 25lb | Soft White Wheat |
| AM-CRN-50 | Yellow Corn Meal 50lb | Yellow Corn |
| AM-OAT-25 | Rolled Oats 25lb | Oats |
| AM-RYE-25 | Medium Rye Flour 25lb | Rye |

### Quality Specifications

| Flour type | Protein % | Moisture % | Ash % | Falling # (sec) |
|---|---|---|---|---|
| Bread Flour (HRS Wheat) | 12.0–14.5 | 11.5–14.0 | 0.44–0.55 | 350–420 |
| All-Purpose (HRW Wheat) | 10.5–11.8 | 11.0–13.5 | 0.42–0.50 | 280–380 |
| Whole Wheat | 11.5–13.5 | 11.0–13.0 | 1.20–1.80 | 280–380 |
| Pastry Flour (SWW) | 8.0–9.5 | 11.0–13.0 | 0.40–0.46 | 300–400 |
| Durum Semolina | 12.0–14.0 | 11.0–13.5 | 0.80–0.95 | 360–440 |

**Failure criteria** (any one = `Passed: false`, ~8% of records):
- Moisture > 14.0%
- Falling number < 250 sec (sprout damage)
- Protein below product floor

### Dashboard KPIs

| Metric | Value | Delta |
|---|---|---|
| Total Shipments | 1,247 | +8.3% vs. prior 30 days |
| Active Mills | 9 / 10 | — |
| Quality Score | 96.4% | +1.2 pts |
| On-Time Delivery | 92.7% | –0.8 pts |

### Monthly Shipment Trend (Jul 2025 – Jun 2026)

| Month | Shipments |
|---|---|
| Jul '25 | 980 |
| Aug '25 | 1,020 |
| Sep '25 | 1,110 |
| Oct '25 | 1,180 |
| Nov '25 | 1,240 |
| Dec '25 | 1,190 |
| Jan '26 | 1,050 |
| Feb '26 | 980 |
| Mar '26 | 1,090 |
| Apr '26 | 1,170 |
| May '26 | 1,220 |
| Jun '26 | 1,247 |

### Grain Mix (tons milled per day)

Wheat: 5,420 · Corn: 1,860 · Soy: 940 · Oats: 480

### Carriers

- **Rail** (BNSF Railway, Union Pacific, CN Rail): 44,000–48,000 lbs per covered hopper car
- **Truck** (Schneider National, JB Hunt, Werner Enterprises): 22,000–26,000 lbs per 48-ft dry van
- **Intermodal**: 22,000–28,000 lbs

---

## Implementation Sequence

### Phase 1 — ASP.NET Core Backend
1. `dotnet new webapi` + package installs
2. Author entity models, DTOs, AppDbContext
3. Author service interfaces + implementations (EF Core IQueryable filtering)
4. Author controllers (thin — delegate to services)
5. Add seed data via `HasData` in OnModelCreating
6. Configure CORS for `http://localhost:4200`
7. `dotnet ef migrations add InitialCreate && dotnet ef database update`
8. `dotnet run` → verify Swagger UI at `/swagger`

### Phase 2 — Angular Scaffold & Core
9. `ng new` + `ng add @angular/material` + install ng2-charts
10. Author TypeScript interfaces in `core/models/`
11. Author three functional interceptors (auth, loading, error)
12. Author LoadingService, NotificationService, FeatureFlagsService
13. Author API services in `core/api/` using `environment.apiBaseUrl`
14. Register everything in `app.config.ts`

### Phase 3 — Shell & Navigation
15. ShellComponent (MatSidenav + BreakpointObserver responsive)
16. SidenavNavComponent (dumb) + TopbarComponent (dumb)
17. LoadingBarComponent bound to `LoadingService.isLoading()`
18. Stub routes → verify nav end-to-end with real API calls

### Phase 4 — Shared Components
19. KpiCardComponent, StatusBadgeComponent, PageHeaderComponent, EmptyStateComponent

### Phase 5 — Dashboard Feature
20. DashboardStore (forkJoin kpis + trend + grainMix)
21. DashboardComponent (smart) + ShipmentsTrendChartComponent + GrainMixChartComponent (both dumb)

### Phase 6 — Shipments Feature
22. ShipmentsStore
23. ShipmentsFilterFormComponent (reactive form, debounce)
24. ShipmentsTableComponent (mat-table + paginator)
25. ShipmentsListComponent (smart, wires store to dumb children)
26. ShipmentDetailComponent (httpResource pattern)

### Phase 7 — Mills Feature
27. MillsStore
28. MillCardComponent (mat-card + progress bar)
29. MillsGridComponent (smart, chip filter + CSS grid)
30. MillDetailComponent (httpResource)

### Phase 8 — Quality Feature
31. QualityStore
32. GradeColorPipe, ShortTonsPipe
33. QualityListComponent (smart, mat-table + slide toggle + color-coded grade cells)

### Phase 9 — Polish
34. Empty states on all async views
35. Mobile responsive check (hamburger opens sidenav, table scrolls horizontally)
36. ARIA labels on all icon buttons and form fields
37. `ng build --configuration production` + verify bundle size

---

## Interview Talking Points

1. **Full Microsoft stack alignment** — ASP.NET Core + SQL Server + Angular mirrors what large food/manufacturing companies like Ardent Mills typically run. Choosing this stack signals you can step into their environment on day one.

2. **Service layer in .NET** — controllers are thin entry points; all business logic and EF Core queries live in injected services. This enforces testability (mock `IShipmentService` in controller tests) and single responsibility.

3. **DTOs decouple API from DB schema** — `ShipmentDto` ≠ `Shipment` entity. The API contract stays stable even when the EF model evolves.

4. **Three-interceptor pipeline** — mirrors production: auth → loading metrics → error handling. Each interceptor has one job; the pipeline is composable.

5. **Functional interceptors** — tree-shakeable; `inject()` inside functions; no `HTTP_INTERCEPTORS` multi-provider array boilerplate.

6. **Signal stores right-sized for this scale** — no NgRx overhead for a PoC. Graduation path to `@ngrx/signals` `signalStore()` is 1:1 shape mapping when the app grows.

7. **`withComponentInputBinding`** — route `:id` param becomes a typed `input()` signal, zero `ActivatedRoute.snapshot` boilerplate, works naturally with `OnPush`.

8. **`httpResource`** (Angular 19+) — reactive resource primitive for route-driven data. Handles request cancellation on navigation. Exposes `value()`, `isLoading()`, `error()` as signals. Replaces the `effect(() => subscribe())` anti-pattern.

9. **OnPush everywhere + Signals** — safe without `markForCheck()` because Angular's reactive graph notifies views when any `computed()` or `signal()` in the template changes.

10. **`queueMicrotask` in loading interceptor** — prevents `NG0600` (write to signal during change detection) when HTTP completes synchronously (common in mocks and unit tests).

11. **ng2-charts over ngx-charts** — `ngx-charts` 24 peer deps cap at Angular 21; `ng2-charts` v7 supports Angular 22. Choosing a library with compatible peer deps is an architect responsibility; cite Apache ECharts as the evaluation candidate for Angular 22+ at production scale.

12. **Feature flag guard** — `CanMatchFn` factory pattern; route is not evaluated unless the flag is enabled. Demonstrates you think about controlled rollout and modular release strategy.

---

## Verification Checklist

```bash
# Backend
dotnet run --project ArdentMills.Api
# ✓ http://localhost:5000/swagger — all endpoints return data
# ✓ GET /api/shipments?status=Delayed returns filtered results
# ✓ GET /api/shipments/SHP-2026-00001 returns 200
# ✓ GET /api/shipments/INVALID returns 404

# Frontend
ng serve --open
# ✓ All nav routes load without errors
# ✓ Dashboard KPI cards show real data from API
# ✓ Line chart and doughnut chart render
# ✓ Shipments filter debounces 250ms, updates table in real time
# ✓ Pagination works; page resets to 0 when filter changes
# ✓ Shipment detail page loads via httpResource
# ✓ Loading bar appears during API calls
# ✓ Error snackbar appears when API is stopped
# ✓ Hamburger menu works on mobile viewport
# ✓ Quality "Failures only" toggle filters to passed: false rows

ng build --configuration production   # ✓ no compile errors
ng lint                               # ✓ no lint errors
```
