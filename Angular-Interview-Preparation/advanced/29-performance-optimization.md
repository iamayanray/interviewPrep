# Advanced Angular Performance Optimization

## Question
What are the advanced performance optimization techniques in Angular? Explain change detection, lazy loading, and bundle optimization with examples.

## Answer

### Introduction to Advanced Angular Performance Optimization

Angular provides various techniques to optimize application performance. Here's a comprehensive guide to advanced performance optimization techniques in Angular.

### 1. Advanced Change Detection

#### Custom Change Detection Strategy

```typescript
// custom-change-detection-strategy.ts
@Injectable()
export class CustomChangeDetectionStrategy extends ChangeDetectionStrategy {
  detectChanges(view: ViewData): void {
    const oldValues = view.oldValues;
    const newValues = view.newValues;

    // Compare old and new values
    if (this.hasChanged(oldValues, newValues)) {
      // Update the view
      this.updateView(view);
    }
  }

  private hasChanged(oldValues: any[], newValues: any[]): boolean {
    return oldValues.some((oldValue, index) => {
      return !this.isEqual(oldValue, newValues[index]);
    });
  }

  private isEqual(oldValue: any, newValue: any): boolean {
    if (oldValue === newValue) return true;
    if (typeof oldValue !== typeof newValue) return false;
    if (typeof oldValue !== 'object') return false;
    if (oldValue === null || newValue === null) return false;

    return JSON.stringify(oldValue) === JSON.stringify(newValue);
  }

  private updateView(view: ViewData): void {
    // Update view logic
  }
}

// Using in component
@Component({
  selector: 'app-performance',
  changeDetection: ChangeDetectionStrategy.OnPush,
  providers: [
    {
      provide: ChangeDetectionStrategy,
      useClass: CustomChangeDetectionStrategy
    }
  ],
  template: `
    <div *ngFor="let item of items">
      {{ item.name }}
    </div>
  `
})
export class PerformanceComponent {
  items: Item[] = [];

  constructor() {
    // Initialize items
  }
}
```

#### OnPush Change Detection with Observables

```typescript
// performance.component.ts
@Component({
  selector: 'app-performance',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div *ngFor="let item of items$ | async">
      {{ item.name }}
    </div>
  `
})
export class PerformanceComponent {
  items$: Observable<Item[]>;

  constructor(private itemService: ItemService) {
    this.items$ = this.itemService.getItems().pipe(
      shareReplay(1),
      catchError(error => {
        console.error('Error loading items:', error);
        return of([]);
      })
    );
  }
}

// Item service
@Injectable({ providedIn: 'root' })
export class ItemService {
  private itemsSubject = new BehaviorSubject<Item[]>([]);

  getItems(): Observable<Item[]> {
    return this.itemsSubject.asObservable();
  }

  updateItems(items: Item[]): void {
    this.itemsSubject.next(items);
  }
}
```

### 2. Advanced Lazy Loading

#### Preloading Strategy

```typescript
// custom-preloading-strategy.ts
@Injectable()
export class CustomPreloadingStrategy implements PreloadAllModules {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (route.data?.preload === false) {
      return of(null);
    }

    return load().pipe(
      catchError(error => {
        console.error('Error preloading module:', error);
        return of(null);
      })
    );
  }
}

// Using in routing
const routes: Routes = [
  {
    path: 'feature',
    loadChildren: () => import('./feature/feature.module').then(m => m.FeatureModule),
    data: { preload: true }
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    data: { preload: false }
  }
];

@NgModule({
  imports: [
    RouterModule.forRoot(routes, {
      preloadingStrategy: CustomPreloadingStrategy
    })
  ]
})
export class AppRoutingModule {}
```

#### Component-Level Lazy Loading

```typescript
// lazy-component.ts
@Component({
  selector: 'app-lazy',
  template: `
    <ng-container *ngIf="isLoaded">
      <ng-container #container></ng-container>
    </ng-container>
  `
})
export class LazyComponent implements OnInit {
  @ViewChild('container', { read: ViewContainerRef })
  container: ViewContainerRef;

  isLoaded = false;

  constructor(private componentFactoryResolver: ComponentFactoryResolver) {}

  ngOnInit(): void {
    this.loadComponent();
  }

  private async loadComponent(): Promise<void> {
    try {
      const { DynamicComponent } = await import('./dynamic.component');
      const factory = this.componentFactoryResolver.resolveComponentFactory(DynamicComponent);
      this.container.createComponent(factory);
      this.isLoaded = true;
    } catch (error) {
      console.error('Error loading component:', error);
    }
  }
}
```

### 3. Advanced Bundle Optimization

#### Custom Webpack Configuration

```typescript
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      minSize: 20000,
      maxSize: 244000,
      minChunks: 1,
      maxAsyncRequests: 30,
      maxInitialRequests: 30,
      cacheGroups: {
        defaultVendors: {
          test: /[\\/]node_modules[\\/]/,
          priority: -10,
          reuseExistingChunk: true
        },
        default: {
          minChunks: 2,
          priority: -20,
          reuseExistingChunk: true
        }
      }
    }
  },
  plugins: [
    new CompressionPlugin({
      filename: '[path][base].gz',
      algorithm: 'gzip',
      test: /\.(js|css|html|svg)$/,
      threshold: 10240,
      minRatio: 0.8
    })
  ]
};
```

#### Tree Shaking Configuration

```json
// angular.json
{
  "projects": {
    "app": {
      "architect": {
        "build": {
          "options": {
            "optimization": {
              "scripts": true,
              "styles": true,
              "fonts": true,
              "images": true
            },
            "sourceMap": false,
            "namedChunks": false,
            "aot": true,
            "vendorChunk": false,
            "commonChunk": false,
            "buildOptimizer": true
          }
        }
      }
    }
  }
}
```

### 4. Advanced Memory Management

#### Subscription Management

```typescript
// subscription-manager.service.ts
@Injectable({ providedIn: 'root' })
export class SubscriptionManagerService {
  private subscriptions = new Map<string, Subscription>();

  addSubscription(key: string, subscription: Subscription): void {
    this.subscriptions.set(key, subscription);
  }

  removeSubscription(key: string): void {
    const subscription = this.subscriptions.get(key);
    if (subscription) {
      subscription.unsubscribe();
      this.subscriptions.delete(key);
    }
  }

  unsubscribeAll(): void {
    this.subscriptions.forEach(subscription => subscription.unsubscribe());
    this.subscriptions.clear();
  }
}

// Using in component
@Component({
  selector: 'app-memory',
  template: `
    <div *ngFor="let item of items$ | async">
      {{ item.name }}
    </div>
  `
})
export class MemoryComponent implements OnInit, OnDestroy {
  items$: Observable<Item[]>;

  constructor(
    private itemService: ItemService,
    private subscriptionManager: SubscriptionManagerService
  ) {}

  ngOnInit(): void {
    this.items$ = this.itemService.getItems().pipe(
      takeUntil(this.destroy$)
    );

    this.subscriptionManager.addSubscription(
      'items',
      this.items$.subscribe()
    );
  }

  ngOnDestroy(): void {
    this.subscriptionManager.unsubscribeAll();
  }
}
```

#### Memory Leak Prevention

```typescript
// memory-leak-prevention.component.ts
@Component({
  selector: 'app-memory-leak',
  template: `
    <div *ngIf="isVisible">
      <app-heavy-component></app-heavy-component>
    </div>
  `
})
export class MemoryLeakPreventionComponent implements OnInit, OnDestroy {
  isVisible = true;
  private destroy$ = new Subject<void>();

  constructor() {}

  ngOnInit(): void {
    // Use takeUntil for all subscriptions
    this.someObservable$.pipe(
      takeUntil(this.destroy$)
    ).subscribe();

    // Use async pipe in template
    this.items$ = this.itemService.getItems().pipe(
      takeUntil(this.destroy$)
    );
  }

  ngOnDestroy(): void {
    // Clean up all subscriptions
    this.destroy$.next();
    this.destroy$.complete();
  }

  toggleVisibility(): void {
    this.isVisible = !this.isVisible;
  }
}
```

### 5. Advanced Caching Strategies

#### HTTP Caching

```typescript
// http-cache.interceptor.ts
@Injectable()
export class HttpCacheInterceptor implements HttpInterceptor {
  private cache = new Map<string, CacheEntry>();

  constructor() {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    if (request.method !== 'GET') {
      return next.handle(request);
    }

    const cachedResponse = this.getFromCache(request);
    if (cachedResponse) {
      return of(cachedResponse);
    }

    return next.handle(request).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          this.addToCache(request, event);
        }
      })
    );
  }

  private getFromCache(request: HttpRequest<any>): HttpResponse<any> | null {
    const key = this.getCacheKey(request);
    const cached = this.cache.get(key);
    
    if (cached && !this.isExpired(cached)) {
      return cached.response;
    }
    
    return null;
  }

  private addToCache(
    request: HttpRequest<any>,
    response: HttpResponse<any>
  ): void {
    const key = this.getCacheKey(request);
    const entry: CacheEntry = {
      response,
      timestamp: Date.now()
    };
    
    this.cache.set(key, entry);
  }

  private getCacheKey(request: HttpRequest<any>): string {
    return request.url;
  }

  private isExpired(entry: CacheEntry): boolean {
    const maxAge = 5 * 60 * 1000; // 5 minutes
    return Date.now() - entry.timestamp > maxAge;
  }
}

interface CacheEntry {
  response: HttpResponse<any>;
  timestamp: number;
}
```

#### Route Data Caching

```typescript
// route-data-cache.service.ts
@Injectable({ providedIn: 'root' })
export class RouteDataCacheService {
  private cache = new Map<string, CachedData>();

  setData(key: string, data: any): void {
    this.cache.set(key, {
      data,
      timestamp: Date.now()
    });
  }

  getData(key: string): any | null {
    const cached = this.cache.get(key);
    if (cached && !this.isExpired(cached)) {
      return cached.data;
    }
    return null;
  }

  private isExpired(cached: CachedData): boolean {
    const maxAge = 10 * 60 * 1000; // 10 minutes
    return Date.now() - cached.timestamp > maxAge;
  }
}

interface CachedData {
  data: any;
  timestamp: number;
}

// Using in resolver
@Injectable()
export class DataResolver implements Resolve<any> {
  constructor(private cacheService: RouteDataCacheService) {}

  resolve(route: ActivatedRouteSnapshot): any {
    const key = this.getCacheKey(route);
    const cachedData = this.cacheService.getData(key);
    
    if (cachedData) {
      return cachedData;
    }

    return this.fetchData(route).pipe(
      tap(data => this.cacheService.setData(key, data))
    );
  }

  private getCacheKey(route: ActivatedRouteSnapshot): string {
    return `${route.routeConfig.path}-${route.params.id}`;
  }

  private fetchData(route: ActivatedRouteSnapshot): Observable<any> {
    // Implementation for fetching data
    return of(null);
  }
}
```

### Conclusion

Key points to remember:

1. **Change Detection**:
   - Use OnPush strategy
   - Implement custom strategy
   - Use observables
   - Optimize updates
   - Monitor performance

2. **Lazy Loading**:
   - Implement preloading
   - Use component-level loading
   - Configure routes
   - Handle errors
   - Monitor loading

3. **Bundle Optimization**:
   - Configure Webpack
   - Enable tree shaking
   - Split chunks
   - Compress assets
   - Monitor size

4. **Memory Management**:
   - Manage subscriptions
   - Prevent leaks
   - Clean up resources
   - Use async pipe
   - Monitor memory

5. **Caching Strategies**:
   - Cache HTTP responses
   - Cache route data
   - Set expiration
   - Handle invalidation
   - Monitor cache

Remember to:
- Monitor performance
- Optimize assets
- Regular maintenance
- Test performance
- Profile application
- Update dependencies
- Document optimizations
- Regular audits 