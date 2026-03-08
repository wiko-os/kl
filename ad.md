# Mailroom UI - Architecture & Workflows Guide

> **Frontend Architecture Document** - A comprehensive guide to understanding the application structure, data flow patterns, and development workflows.

---

## Table of Contents

1. [Application Layers](#1-application-layers)
2. [Core Architecture](#2-core-architecture)
3. [Feature Module Pattern](#3-feature-module-pattern)
4. [Data Flow Architecture](#4-data-flow-architecture)
5. [Component Architecture](#5-component-architecture)
6. [Routing Architecture](#6-routing-architecture)
7. [State Management](#7-state-management)
8. [Service Patterns](#8-service-patterns)
9. [Shared UI Components](#9-shared-ui-components)
10. [Feature Workflows](#10-feature-workflows)

---

## 1. Application Layers

The application is organized in **4 distinct layers** following a clean architecture approach:

```mermaid
graph TB
    subgraph "UI Layer - Presentation"
        direction TB
        Pages["📄 Pages<br/>Feature pages<br/>Route containers"]
        Containers["🎯 Containers<br/>Smart components<br/>State management"]
        Components["🧩 Components<br/>Dumb components<br/>Pure UI"]
    end
    
    subgraph "Service Layer - Business Logic"
        direction TB
        Services["⚙️ Services<br/>Data transformation<br/>State coordination"]
    end
    
    subgraph "API Layer - Remote Data"
        direction TB
        HttpService["🌐 HTTP Services<br/>API endpoints<br/>Response mapping"]
        Tokens["🎫 Injection Tokens<br/>Service abstraction"]
    end
    
    subgraph "Core Layer - Infrastructure"
        direction TB
        Interceptors["🔗 Interceptors<br/>Auth, Error, i18n<br/>HTTP pipeline"]
        Guards["🛡️ Guards<br/>Route protection<br/>Authorization"]
        Config["⚙️ Config<br/>App initialization<br/>Environment"]
    end
    
    Pages --> Containers
    Containers --> Services
    Services --> HttpService
    HttpService --> Interceptors
    Containers --> Guards
    HttpService --> Tokens
```

### Layer Responsibilities

| Layer | Responsibility | Examples |
|-------|---|---|
| **UI** | Render UI, emit events | Pages, Containers, Components |
| **Service** | Business logic, data transformation | MailboxService, DocumentService |
| **API** | HTTP communication, response mapping | MailboxApiHttpService |
| **Core** | Cross-cutting concerns | Auth, Error handling, Config |

---

## 2. Core Architecture

### 2.1 Application Bootstrap

```mermaid
graph LR
    A["main.ts"] -->|bootstrapApplication| B["AppComponent"]
    B --> C["app.config.ts"]
    C --> D["provideCore()"]
    C --> E["provideRouter()"]
    C --> F["provideI18n()"]
    D --> G["HTTP Client"]
    D --> H["Interceptors<br/>4x HTTP pipeline"]
    E --> I["Routes Config"]
    F --> J["TranslateService"]
```

### 2.2 Dependency Injection Container

```typescript
// app.config.ts - Central DI configuration
export const appConfig: ApplicationConfig = {
  providers: [
    provideCore(),           // HTTP, Interceptors, Auth
    provideI18n(),          // Translation service
    provideAppConfig(),     // Config initialization
    provideRouter(routes),  // Routing
    
    // API Implementations (Injection Tokens)
    { provide: MAILBOX_API, useClass: MailboxApiHttpService },
    { provide: DOCUMENT_API, useClass: DocumentApiHttpService },
    { provide: VIEWER_API, useClass: ViewerApiHttpService },
    { provide: BUNDLE_API, useClass: BundleApiHttpService },
  ]
};
```

### 2.3 HTTP Interceptor Pipeline

```mermaid
sequenceDiagram
    participant App as Application
    participant Auth as Auth Interceptor
    participant Tenant as Tenant Interceptor
    participant Lang as Accept-Language
    participant Error as Error Interceptor
    participant Backend as Backend API
    
    App->>Auth: 1. HttpRequest
    Auth->>Auth: Add Authorization header
    Auth->>Tenant: 2. Request
    Tenant->>Tenant: Add Tenant header
    Tenant->>Lang: 3. Request
    Lang->>Lang: Add Accept-Language
    Lang->>Backend: 4. Final Request
    Backend-->>Error: Response
    Error->>Error: Check status code
    Error-->>App: 5. Success or Error
```

---

## 3. Feature Module Pattern

### 3.1 Feature Folder Structure

Every feature follows this **consistent structure**:

```
features/
├── mailboxes/                    # Feature module name
│   ├── mailboxes.routes.ts      # Route definitions
│   ├── api/
│   │   ├── mailbox-api.ts       # Interface (abstraction)
│   │   ├── mailbox-api.token.ts # Injection token
│   │   └── mailbox-api.http.service.ts  # Implementation
│   ├── services/
│   │   └── mailbox.service.ts   # Business logic
│   ├── models/
│   │   └── mailbox.model.ts     # TypeScript interfaces
│   ├── mocks/
│   │   └── mailboxes.mock.ts    # Dev/test data
│   ├── pages/
│   │   └── mailbox-page/
│   │       ├── mailbox-page.component.ts
│   │       ├── mailbox-page.component.html
│   │       └── mailbox-page.component.scss
│   ├── containers/
│   │   └── mailbox/
│   │       ├── mailbox.container.ts    # Smart component
│   │       ├── mailbox.container.html
│   │       └── mailbox.container.scss
│   └── components/
│       ├── mailbox-list/
│       └── document-list/
```

### 3.2 Feature Module Layers (Detailed)

```mermaid
graph TD
    subgraph "Feature: Mailboxes"
        A["🌐 Routes<br/>mailboxes.routes.ts"]
        
        B["📄 Page<br/>mailbox-page.component"]
        
        C["🎯 Container<br/>mailbox.container"]
        
        D["🧩 Components<br/>mailbox-list<br/>document-list"]
        
        E["⚙️ Service<br/>mailbox.service"]
        
        F["🌐 API Layer"]
        subgraph "F"
            F1["Token<br/>MAILBOX_API"]
            F2["HTTP Service<br/>MailboxApiHttpService"]
        end
        
        G["📊 Models<br/>Interfaces"]
        
        H["🎭 Mocks<br/>Dev data"]
    end
    
    A -->|navigate| B
    B -->|renders| C
    C -->|delegates to| D
    C -->|state| E
    E -->|depends| F
    F -->|uses| G
    H -->|alternative to| F
```

---

## 4. Data Flow Architecture

### 4.1 Master Data Flow (Request → Response)

```mermaid
graph LR
    A["📱 User<br/>Interaction"]
    B["🧩 Component<br/>@Output event"]
    C["🎯 Container<br/>toSignal"]
    D["⚙️ Service<br/>Observable"]
    E["🌐 HTTP Service<br/>API call"]
    F["🔗 Interceptor<br/>Auth + headers"]
    G["📡 Backend<br/>API"]
    H["🎁 Response"]
    I["📊 Signals<br/>State"]
    J["🎨 Template<br/>Render"]
    
    A -->|click/change| B
    B -->|emit| C
    C -->|switchMap| D
    D -->|inject| E
    E -->|check headers| F
    F -->|HTTP GET/POST| G
    G -->|200 OK| H
    H -->|map + share| D
    D -->|toSignal| I
    I -->|binding| J
```

### 4.2 Real-world Example: Load Mailboxes

```typescript
// Step 1: User navigates to /mailboxes
// ↓
// Step 2: Resolver triggers mailboxResolver
// ↓
// Step 3: MailboxService initiates request
readonly mailboxes$ = this.refresh$.pipe(
  startWith(void 0),                    // Trigger immediately
  switchMap(() => this.api.getMailboxes()),  // Call API
  shareReplay(1)                        // Share result, cache
);

// Step 4: Container converts to Signal
readonly mailboxes = toSignal(
  this.mailboxService.mailboxes$.pipe(map(...)),
  { initialValue: [] }
);

// Step 5: Transform data (add avatars, initials)
map(sections => sections.map(section => ({
  ...section,
  mailboxes: section.mailboxes.map(m => ({
    ...m,
    initials: this.avatarUtils.getInitials(m.label),
    avatarClass: this.avatarUtils.getAvatarClass(m.label)
  }))
})))

// Step 6: Pass to component as input Signal
[sections]="mailboxes()"    // Call signal to get value

// Step 7: Component renders with UiListComponent
<app-ui-list [items]="sections()" (selectionChange)="onSelectionChange($event)">
```

---

## 5. Component Architecture

### 5.1 Component Type Hierarchy

```mermaid
graph TD
    A["Component Types"]
    
    B["Smart Components<br/>Containers"]
    C["Wrapper Components<br/>Feature lists"]
    D["Presentational Components<br/>Shared UI"]
    
    B -->|State| E["toSignal<br/>Observable to Signal"]
    B -->|Selection| F["signal<br/>Local state"]
    B -->|Events| G["output<br/>Emit events"]
    
    C -->|Input| H["input.required<br/>Props"]
    C -->|Events| I["selectItem.emit<br/>Notify parent"]
    C -->|Render| J["UiListComponent<br/>Generic list"]
    
    D -->|Generic| K["@Input signal<br/>items, density"]
    D -->|Flexible| L["@Output emitters<br/>itemClick, selection"]
    D -->|Template| M["@ContentChild<br/>itemTemplate"]
    
    A --> B
    A --> C
    A --> D
```

### 5.2 Input/Output Pattern (Modern)

```typescript
// ✅ NEW: Signals API (Angular 17+)
@Component({...})
export class MailboxListComponent {
  // Input as Signal
  mailboxes = input.required<Mailbox[]>();
  selectedMailboxId = input<number | null>(null);
  
  // Computed derived state
  selectedMailbox = computed(() => {
    const id = this.selectedMailboxId();
    if (!id) return null;
    return this.mailboxes()?.find(m => m.id === id) ?? null;
  });
  
  // Output for events
  selectMailbox = output<number>();
  
  onSelectionChange(items: Mailbox[]) {
    this.selectMailbox.emit(items[0].id);
  }
}

// Template usage
[mailboxes]="mailboxes()"           // ✅ Call signal
[selectedMailboxId]="selectedId()"  // ✅ Call signal
(selectMailbox)="onSelect($event)"  // ✅ Event listener
```

### 5.3 Generic Reusable Component: UiListComponent

```mermaid
graph TB
    A["UiListComponent&lt;T&gt;"]
    
    B["Inputs"]
    subgraph "B"
        B1["items: T[]"]
        B2["keyLabel?: keyof T"]
        B3["keyValue?: keyof T"]
        B4["selectable: boolean"]
        B5["density: xs|sm|md|lg"]
    end
    
    C["Outputs"]
    subgraph "C"
        C1["itemClick: T"]
        C2["selectionChange: T[]"]
        C3["scrolledBottom: void"]
    end
    
    D["Content Child"]
    subgraph "D"
        D1["itemTemplate: TemplateRef"]
    end
    
    E["Usage in Features"]
    subgraph "E"
        E1["MailboxListComponent"]
        E2["ConsoleListComponent"]
        E3["Custom list types"]
    end
    
    A --> B
    A --> C
    A --> D
    A --> E
```

---

## 6. Routing Architecture

### 6.1 Route Hierarchy

```
/
├── /select-tenant              ← Tenant selection
├── /                           ← Home (canActivate: tenantGuard)
│   ├── /mailboxes              ← Mailbox list
│   ├── /mailboxes/:id          ← Specific mailbox
│   ├── /outbox                 ← Outbox feature
│   └── /console                ← Console feature
└── /tenant-selector            ← Admin tenant selector
```

### 6.2 Route Resolution & Guards

```mermaid
sequenceDiagram
    participant User as User
    participant Router as Router
    participant Guard as tenantGuard
    participant Resolver as mailboxResolver
    participant Service as MailboxService
    participant Backend as API
    
    User->>Router: navigate(/mailboxes)
    Router->>Guard: canActivate()
    Guard->>Guard: Check tenant context
    Guard-->>Router: ✅ Pass
    Router->>Resolver: Execute resolver
    Resolver->>Service: getMailboxes()
    Service->>Backend: GET /api/mailboxes
    Backend-->>Service: [Mailbox[]]
    Service-->>Resolver: Observable
    Resolver-->>Router: Resolved data
    Router-->>User: 📄 Route activated
```

### 6.3 Lazy Load with MAILBOX_ROUTES

```typescript
// app.routes.ts - Main router configuration
export const routes: Routes = [
  {
    path: 'mailboxes',
    component: MailboxPageComponent,
    resolve: { mailboxes: mailboxResolver },  // Pre-load data
  },
  {
    path: 'mailboxes/:mailboxId',
    component: MailboxPageComponent,
    resolve: { mailboxes: mailboxResolver },
  },
];

// mailboxes.routes.ts - Feature routes
export const MAILBOX_ROUTES: Routes = [
  {
    path: '',
    pathMatch: 'full',
    redirectTo: 'mailboxes',
  },
  ...
];
```

---

## 7. State Management

### 7.1 State Management Strategy

```mermaid
graph TB
    A["State Types"]
    
    B["Global State"]
    subgraph "B"
        B1["ViewerStateService"]
        B2["AuthService + User"]
        B3["ErrorService"]
    end
    
    C["Feature State"]
    subgraph "C"
        C1["MailboxService<br/>Observable + Signals"]
        C2["DocumentService"]
        C3["BundleService"]
    end
    
    D["Local Component State"]
    subgraph "D"
        D1["signal(selected)"]
        D2["computed(...)"]
        D3["effect(...)"]
    end
    
    E["Data Flow"]
    subgraph "E"
        E1["Backend Observable"]
        E2["toSignal()"]
        E3["computed()"]
        E4["effect()"]
    end
    
    A --> B
    A --> C
    A --> D
    C --> E
```

### 7.2 Signal-based State (Modern Example)

```typescript
// MailboxService - Feature state management
@Injectable({ providedIn: 'root' })
export class MailboxService {
  private readonly api = inject(MAILBOX_API);
  
  // Signals for state
  private readonly searchQuerySignal = signal('');
  private readonly hasMoreSearchResultsSignal = signal(true);
  readonly loading = signal(true);
  
  // Subjects for actions
  private readonly refresh$ = new Subject<void>();
  private readonly searchInput$ = new Subject<MailboxSearch>();
  
  // Observable state (from API)
  readonly mailboxes$ = this.refresh$.pipe(
    startWith(void 0),
    switchMap(() => this.api.getMailboxes()),
    shareReplay(1)  // Cache result
  );
  
  // Derived signals
  readonly hasSearchResults = computed(() => 
    this.searchQuerySignal().length > 0
  );
}

// Container - Convert Observable to Signal
export class MailboxContainer {
  readonly mailboxes = toSignal(
    this.mailboxService.mailboxes$.pipe(map(...)),
    { initialValue: [] }
  );
  
  readonly selectedMailboxId = signal<number | null>(null);
  
  selectMailbox(id: number) {
    this.selectedMailboxId.set(id);
  }
}
```

### 7.3 When to Use Signal vs Observable

| Use Case | Pattern | Example |
|----------|---------|---------|
| **Local state** | `signal()` | `selectedId = signal(null)` |
| **Derived state** | `computed()` | `computed(selectedMailbox)` |
| **Effect** | `effect()` | `effect(() => router.navigate(...))` |
| **API response** | `Observable` | `this.api.getMailboxes()` |
| **Convert to signal** | `toSignal()` | `toSignal(observable$)` |
| **Subscription cleanup** | `takeUntilDestroyed()` | Automatic cleanup |

---

## 8. Service Patterns

### 8.1 API Service Pattern (Token + Implementation)

```mermaid
graph LR
    A["Abstraction Layer"]
    subgraph "A"
        A1["mailbox-api.ts<br/>interface MailboxApi"]
    end
    
    B["Injection Token"]
    subgraph "B"
        B1["MAILBOX_API<br/>InjectionToken"]
    end
    
    C["Implementation"]
    subgraph "C"
        C1["MailboxApiHttpService<br/>implements MailboxApi"]
    end
    
    D["DI Container"]
    subgraph "D"
        D1["app.config.ts<br/>provide MAILBOX_API"]
    end
    
    E["Consumer"]
    subgraph "E"
        E1["MailboxService<br/>inject(MAILBOX_API)"]
    end
    
    A --> B
    B --> C
    C --> D
    D --> E
```

```typescript
// 1️⃣ Abstraction - mailbox-api.ts
export interface MailboxApi {
  getMailboxes(): Observable<MailboxSection[]>;
  getMailboxById(id: number): Observable<Mailbox>;
  searchMailbox(query: string): Observable<MailboxSearch[]>;
}

// 2️⃣ Token - mailbox-api.token.ts
export const MAILBOX_API = new InjectionToken<MailboxApi>('MAILBOX_API');

// 3️⃣ Implementation - mailbox-api.http.service.ts
@Injectable()
export class MailboxApiHttpService implements MailboxApi {
  private readonly http = inject(HttpClient);
  private readonly config = inject(AppConfigService);
  
  getMailboxes(): Observable<MailboxSection[]> {
    return this.http
      .get<MailboxResponse>(`${this.config.appConfig.apiBaseUrl}/mailboxes`)
      .pipe(map(res => res.data ?? []));
  }
}

// 4️⃣ Registration - app.config.ts
{ provide: MAILBOX_API, useClass: MailboxApiHttpService }

// 5️⃣ Usage - mailbox.service.ts
export class MailboxService {
  private readonly api = inject(MAILBOX_API);  // Get from DI
  
  readonly mailboxes$ = this.api.getMailboxes();
}
```

### 8.2 Service Responsibilities

```
MailboxService (⚙️ Business Logic)
├── Coordinate API calls
├── Manage feature state (Signals + Observables)
├── Transform data
├── Handle search/pagination
└── Expose Observables for components

MailboxApiHttpService (🌐 HTTP)
├── Make HTTP requests
├── Map API responses
├── Apply HttpParams
└── Handle URL building

MailboxContainer (🎯 Smart Component)
├── Subscribe to service
├── Convert Observable → Signal
├── Handle user interactions
└── Navigate/update URL
```

---

## 9. Shared UI Components

### 9.1 Component Library Architecture

```mermaid
graph TB
    subgraph "Shared UI Library"
        A["Generic Components"]
        subgraph "A"
            A1["UiListComponent&lt;T&gt;"]
            A2["UiAccordion &amp; UiAccordionItem"]
            A3["UiButton"]
            A4["UiToast / UiModal"]
        end
        
        B["Material Wrappers"]
        subgraph "B"
            B1["ExpansionPanel"]
            B2["Tooltip"]
            B3["Dialog"]
        end
        
        C["Feature Wrappers"]
        subgraph "C"
            C1["MailboxListComponent"]
            C2["DocumentListComponent"]
            C3["ConsoleListComponent"]
        end
    end
    
    D["Usage"]
    subgraph "D"
        D1["mailbox.container.html"]
        D2["document.container.html"]
        D3["console.container.html"]
    end
    
    A --> C
    B --> C
    C --> D
```

### 9.2 Generic List Pattern

**UiListComponent** is the **core reusable component** for all list rendering:

```typescript
// Generic signature
@Component({...})
export class UiListComponent<T> {
  items = input.required<T[]>();
  keyLabel = input<keyof T | null>(null);      // Which field to display
  keyValue = input<keyof T | null>(null);      // Unique identifier
  selectable = input(false);                    // Enable selection
  multiSelect = input(false);                   // Multiple selection
  
  itemClick = output<T>();
  selectionChange = output<T[]>();
  scrolledBottom = output<void>();
  
  itemTemplate = contentChild(...);             // Custom render template
}
```

**Feature-specific wrappers** adapt UiListComponent:

```typescript
// MailboxListComponent - Wraps UiListComponent for mailboxes
@Component({
  selector: 'app-mailbox-list',
  template: `
    <app-ui-list
      [items]="mailboxes()"
      [selectedItem]="selectedMailbox()"
      (selectionChange)="onSelectionChange($event)">
      <ng-template #itemTemplate let-mailbox>
        <div>{{ mailbox.label }}</div>
        <span class="unread">{{ getUnreadLabel(mailbox.unreadCount) }}</span>
      </ng-template>
    </app-ui-list>
  `
})
export class MailboxListComponent {
  mailboxes = input.required<Mailbox[]>();
  selectMailbox = output<number>();
  
  onSelectionChange(items: Mailbox[]) {
    this.selectMailbox.emit(items[0].id);
  }
}
```

---

## 10. Feature Workflows

### 10.1 Complete Feature Workflow: Load & Display Mailboxes

```mermaid
graph TB
    A["👤 User navigates to /mailboxes"]
    B["🔗 Router checks tenantGuard"]
    C["✅ Guard passes, route resolves"]
    D["📄 MailboxPageComponent loads"]
    E["🎯 MailboxContainer created"]
    F["⚙️ MailboxService triggered"]
    G["🌐 API call: GET /api/mailboxes"]
    H["🔗 HTTP Interceptor adds headers"]
    I["📡 Backend returns MailboxSection[]"]
    J["📊 Service transforms data"]
    K["🎁 toSignal converts Observable→Signal"]
    L["📱 MailboxContainer has Signal<MailboxSection[]>"]
    M["🧩 MailboxContainer renders MailboxListComponent"]
    N["🎨 MailboxListComponent uses UiListComponent"]
    O["📋 UiListComponent renders list items"]
    P["👆 User clicks mailbox"]
    Q["🎯 MailboxListComponent emits selectMailbox($event)"]
    R["🎯 MailboxContainer calls selectMailbox(id)"]
    S["📍 Router.navigate redirects to /mailboxes/:id"]
    
    A --> B --> C --> D --> E --> F --> G --> H --> I --> J --> K --> L --> M --> N --> O
    O --> P --> Q --> R --> S
```

### 10.2 Accordion Component Workflow (Single Mode)

```mermaid
graph TB
    A["UiAccordionComponent<br/>mode='single'"]
    B["UiAccordionItemComponent"]
    C["Template Content"]
    
    D["State: selectedIndex = 0"]
    E["Track: lastOpenedIndex"]
    
    F["User opens Item 1"]
    G["Item 1 emits openedChange"]
    H["Container closes Item 0"]
    I["Update lastOpenedIndex = 1"]
    
    J["User opens Item 2"]
    K["Item 2 emits openedChange"]
    L["Container closes Item 1<br/>(using lastOpenedIndex)"]
    M["Update lastOpenedIndex = 2"]
    
    A --> B --> C
    A --> D
    D --> E
    F --> G --> H --> I
    I --> J --> K --> L --> M
```

### 10.3 Document Viewing Workflow

```mermaid
sequenceDiagram
    participant User as User
    participant Page as MailboxPage
    participant BundleContainer as BundleContainer
    participant DocumentContainer as DocumentContainer
    participant BundleService as BundleService
    participant API as Backend API
    
    User->>Page: Click mailbox
    Page->>BundleContainer: Load bundles for mailboxId
    BundleContainer->>BundleService: getBundle(mailboxId)
    BundleService->>API: GET /bundles/:mailboxId
    API-->>BundleService: Bundles[]
    BundleService-->>BundleContainer: Signal<Bundles[]>
    BundleContainer->>BundleContainer: Display list
    
    User->>BundleContainer: Select bundle
    BundleContainer->>DocumentContainer: onBundleSelected(bundleId)
    DocumentContainer->>BundleService: getDocuments(bundleId)
    BundleService->>API: GET /bundles/:id/documents
    API-->>DocumentContainer: Documents[]
    DocumentContainer->>DocumentContainer: Display documents
    
    User->>DocumentContainer: Click document
    DocumentContainer->>Page: onDocumentSelected(docId)
    Page->>Page: Open document viewer
```

---

## 11. Development Workflow Best Practices

### 11.1 Creating a New Feature

#### Step 1: Setup Feature Structure
```bash
features/myfeature/
├── myfeature.routes.ts
├── api/
│   ├── myfeature-api.ts           # Interface
│   ├── myfeature-api.token.ts     # Token
│   └── myfeature-api.http.service.ts  # Implementation
├── services/
│   └── myfeature.service.ts       # Business logic
├── models/
│   └── myfeature.model.ts
├── pages/
│   └── myfeature-page/
├── containers/
│   └── myfeature/
└── components/
    └── myfeature-list/
```

#### Step 2: Define API Contract

```typescript
// myfeature-api.ts (Abstraction)
export interface MyFeatureApi {
  getItems(): Observable<MyFeatureItem[]>;
  getItem(id: string): Observable<MyFeatureItem>;
}
```

#### Step 3: Implement API Service

```typescript
// myfeature-api.http.service.ts
@Injectable()
export class MyFeatureApiHttpService implements MyFeatureApi {
  private readonly http = inject(HttpClient);
  private readonly config = inject(AppConfigService);

  getItems(): Observable<MyFeatureItem[]> {
    return this.http
      .get<ApiResponse<MyFeatureItem[]>>(
        `${this.config.appConfig.apiBaseUrl}/items`
      )
      .pipe(map(res => res.data ?? []));
  }
}
```

#### Step 4: Create Feature Service

```typescript
// myfeature.service.ts
@Injectable({ providedIn: 'root' })
export class MyFeatureService {
  private readonly api = inject(MYFEATURE_API);
  
  readonly items$ = this.api.getItems().pipe(shareReplay(1));
}
```

#### Step 5: Create Container Component

```typescript
// containers/myfeature/myfeature.container.ts
@Component({
  selector: 'app-myfeature-container',
  template: `<app-myfeature-list [items]="items()"></app-myfeature-list>`,
  standalone: true,
  imports: [CommonModule, MyFeatureListComponent],
})
export class MyFeatureContainer {
  private readonly service = inject(MyFeatureService);
  
  readonly items = toSignal(this.service.items$, { initialValue: [] });
}
```

#### Step 6: Create Feature List Component

```typescript
// components/myfeature-list/myfeature-list.component.ts
@Component({
  selector: 'app-myfeature-list',
  template: `
    <app-ui-list [items]="items()" (selectionChange)="onSelect($event)">
      <ng-template #itemTemplate let-item>
        <!-- Custom render -->
      </ng-template>
    </app-ui-list>
  `,
  standalone: true,
})
export class MyFeatureListComponent {
  items = input.required<MyFeatureItem[]>();
  selectItem = output<MyFeatureItem>();
  
  onSelect(items: MyFeatureItem[]) {
    this.selectItem.emit(items[0]);
  }
}
```

#### Step 7: Register Routes & DI

```typescript
// app.routes.ts
{ path: 'myfeature', component: MyFeaturePageComponent }

// app.config.ts
{ provide: MYFEATURE_API, useClass: MyFeatureApiHttpService }
```

### 11.2 Common Gotchas

| Problem | Solution |
|---------|----------|
| **Signal not updating in template** | Call signal with `()`: `[items]="items()"` |
| **Can't style template in container** | Add `ViewEncapsulation.None` to component |
| **Memory leaks from subscriptions** | Use `takeUntilDestroyed()` or `toSignal()` |
| **Type errors with computed/input** | Always invoke with `()` in template |
| **Form not resetting** | Use `markAllAsTouched()` and proper Signal updates |
| **Accordion single mode closing multiple** | Track `lastOpenedIndex` and only close that one |

---

## 12. Workflow Decision Tree

```mermaid
graph TD
    A["Need to add functionality?"]
    
    B{"Is it a New Feature?"}
    B -->|Yes| C["Create feature folder structure<br/>See 11.1"]
    B -->|No| D{"What needs updating?"}
    
    D -->|Service logic| E["Update service.ts<br/>Add/modify Observable"]
    D -->|UI rendering| F["Update component.ts/html<br/>Add input/output"]
    D -->|API contract| G["Update api.ts interface<br/>Update implementation"]
    D -->|State| H["Add signal/computed<br/>Update template"]
    
    C --> I["1. API interface"]
    I --> J["2. HTTP service"]
    J --> K["3. Feature service"]
    K --> L["4. Container"]
    L --> M["5. List component"]
    M --> N["6. Register routes & DI"]
    
    E --> O["Register Observable<br/>Make shareable<br/>Document side effects"]
    
    F --> P["Use input signals<br/>Call with ()<br/>Emit output events"]
    
    G --> Q["Extend interface<br/>Update implementation<br/>Update mocks"]
    
    H --> R["Use signal/computed<br/>Add effect if needed<br/>Test reactivity"]
```

---

## 13. Performance Patterns

### 13.1 Observable Sharing

```typescript
// ❌ WRONG - Creates new subscription each time
readonly mailboxes$ = this.api.getMailboxes();

// ✅ RIGHT - Share result, prevent multiple requests
readonly mailboxes$ = this.api.getMailboxes().pipe(
  shareReplay(1)  // Replay to new subscribers
);
```

### 13.2 Single Mode Accordion Performance

```typescript
// ❌ WRONG - O(n) complexity, close all then open new
items.forEach((item, idx) => {
  if (idx !== selectedIndex) item.opened = false;
});

// ✅ RIGHT - O(1) complexity, only update necessary items
if (lastOpenedIndex !== null && lastOpenedIndex !== selectedIndex) {
  this.items.get(lastOpenedIndex)?.openedChange.emit(false);
}
lastOpenedIndex = selectedIndex;
```

### 13.3 Signal Subscriptions

```typescript
// ❌ WRONG - Manual subscription without cleanup
this.service$.subscribe(data => this.process(data));

// ✅ RIGHT - Signal with automatic cleanup
readonly data = toSignal(this.service$, { initialValue: [] });
effect(() => this.process(this.data()));  // Auto cleanup on destroy
```

---

## 14. Testing Patterns

### 14.1 Container Testing

```typescript
it('should load mailboxes on init', () => {
  // Arrange
  const mockMailboxes: Mailbox[] = [{ id: 1, label: 'Inbox' }];
  spyOn(service, 'mailboxes$').and.returnValue(of(mockMailboxes));
  
  // Act
  const component = TestBed.createComponent(MailboxContainer);
  
  // Assert
  expect(component.mailboxes()).toEqual(mockMailboxes);
});
```

### 14.2 Component Testing

```typescript
it('should emit selectMailbox on selection', () => {
  // Arrange
  const mailbox: Mailbox = { id: 1, label: 'Inbox' };
  spyOn(component.selectMailbox, 'emit');
  
  // Act
  component.onSelectionChange([mailbox]);
  
  // Assert
  expect(component.selectMailbox.emit).toHaveBeenCalledWith(1);
});
```

---

## 15. Deployment Architecture

```mermaid
graph LR
    A["Development<br/>ng serve"]
    B["Build<br/>ng build"]
    C["Staging<br/>Preview env"]
    D["Production<br/>Live env"]
    
    E["Feature Branch"]
    F["PR Review"]
    G["Main Branch"]
    H["Release Tag"]
    
    E --> F --> G --> B --> C
    C --> |Approve| D
    G --> H
    
    style A fill:#4CAF50
    style B fill:#2196F3
    style C fill:#FFC107
    style D fill:#F44336
```

---

## Quick Reference

### Key Files to Know
- `app.config.ts` - DI container & providers
- `app.routes.ts` - Route definitions
- `core/core.providers.ts` - HTTP setup
- `features/[feature]/[feature].routes.ts` - Feature routes
- `features/[feature]/services/` - Business logic
- `features/[feature]/api/` - API contracts

### Common Commands
```bash
# Serve locally
ng serve

# Build for production
ng build --prod

# Run linter
npx eslint src/**/*.{ts,html}

# Run tests
ng test
```

### Architecture Principles
1. **Single Responsibility** - Each service has one reason to change
2. **Dependency Injection** - Inject dependencies via constructor
3. **Reactive** - Use Observables/Signals for state
4. **Composable** - Build features from reusable components
5. **Testable** - Separate concerns, mockable APIs
6. **Performance** - Cache with shareReplay(), use OnPush detection

---

## Glossary

| Term | Meaning |
|------|---------|
| **Container** | Smart component managing state |
| **Page** | Route-level component |
| **Component** | Presentational, reusable UI |
| **Service** | Business logic layer |
| **API** | HTTP contract/interface |
| **Token** | DI injection identifier |
| **Observable** | Async data stream (RxJS) |
| **Signal** | New Angular state primitive |
| **toSignal** | Convert Observable → Signal |
| **effect()** | Auto-running side effect |
| **computed()** | Derived reactive value |
| **input** | Component input signal |
| **output** | Component event emitter |

---

**Document Version:** 1.0  
**Last Updated:** 2026-03-08  
**Architecture:** Angular 17+ with Signals, Modern Standalone Components  
**Status:** Active Project Pattern

