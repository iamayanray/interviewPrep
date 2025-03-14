# Advanced Angular Performance Optimization

## Question
What are the advanced performance optimization techniques in Angular? Explain change detection, lazy loading, and bundle optimization with examples.

## Answer

### Introduction to Advanced Angular Performance Optimization

Angular provides various techniques to optimize application performance. Here's a comprehensive guide to advanced performance optimization strategies.

### 1. Advanced Change Detection

#### Custom Change Detection Strategy

```typescript
// custom-change-detection-strategy.ts
@Injectable()
export class CustomChangeDetectionStrategy extends ChangeDetectionStrategy {
  constructor() {
    super();
  }

  detectChanges(view: ViewData): void {
    // Implement custom change detection logic
    const context = view.context;
    const oldValues = view.oldValues;
    const newValues = view.newValues;

    // Compare old and new values
    const hasChanges = this.compareValues(oldValues, newValues);

    if (hasChanges) {
      // Update view
      this.updateView(view);
    }
  }

  private compareValues(oldValues: any, newValues: any): boolean {
    // Implement deep comparison logic
    return JSON.stringify(oldValues) !== JSON.stringify(newValues);
  }

  private updateView(view: ViewData): void {
    // Update view with new values
    view.context = view.newValues;
    view.markForCheck();
  }
}

// Using in component
@Component({
  selector: 'app-performance',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div *ngFor="let item of items">
      {{ item.name }}
    </div>
  `
})
export class PerformanceComponent {
  @Input() items: Item[] = [];

  constructor(
    private cdr: ChangeDetectorRef,
    private changeDetection: CustomChangeDetectionStrategy
  ) {}

  updateItems(newItems: Item[]): void {
    this.items = newItems;
    this.cdr.markForCheck();
  }
}
```

#### OnPush Change Detection with Observables

```typescript
// observable-change-detection.component.ts
@Component({
  selector: 'app-observable',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div *ngIf="data$ | async as data">
      {{ data.name }}
    </div>
  `
})
export class ObservableChangeDetectionComponent {
  data$: Observable<Data>;

  constructor(private dataService: DataService) {
    this.data$ = this.dataService.getData().pipe(
      shareReplay(1),
      catchError(error => {
        console.error('Error loading data:', error);
        return of(null);
      })
    );
  }

  refreshData(): void {
    this.data$ = this.dataService.getData().pipe(
      shareReplay(1)
    );
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

    // Implement custom preloading logic
    return load().pipe(
      catchError(error => {
        console.error('Preloading failed:', error);
        return of(null);
      })
    );
  }
}

// Using in routing module
const routes: Routes = [
  {
    path: 'feature',
    loadChildren: () => import('./feature/feature.module')
      .then(m => m.FeatureModule),
    data: { preload: true }
  },
  {
    path: 'lazy',
    loadChildren: () => import('./lazy/lazy.module')
      .then(m => m.LazyModule),
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

  async ngOnInit(): Promise<void> {
    const module = await import('./lazy-feature/lazy-feature.module');
    const component = module.LazyFeatureComponent;
    const factory = this.componentFactoryResolver.resolveComponentFactory(component);
    
    this.container.createComponent(factory);
    this.isLoaded = true;
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

```typescript
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
            "commonChunk": true,
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
export class SubscriptionManager {
  private subscriptions = new Map<string, Subscription>();

  add(key: string, subscription: Subscription): void {
    this.subscriptions.set(key, subscription);
  }

  remove(key: string): void {
    const subscription = this.subscriptions.get(key);
    if (subscription) {
      subscription.unsubscribe();
      this.subscriptions.delete(key);
    }
  }

  clear(): void {
    this.subscriptions.forEach(subscription => {
      subscription.unsubscribe();
    });
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
    private dataService: DataService,
    private subscriptionManager: SubscriptionManager
  ) {}

  ngOnInit(): void {
    this.items$ = this.dataService.getItems().pipe(
      shareReplay(1)
    );

    this.subscriptionManager.add(
      'items',
      this.items$.subscribe()
    );
  }

  ngOnDestroy(): void {
    this.subscriptionManager.clear();
  }
}
```

#### Memory Leak Prevention

```typescript
// memory-leak-prevention.component.ts
@Component({
  selector: 'app-memory-leak',
  template: `
    <div *ngIf="isActive">
      <ng-container #container></ng-container>
    </div>
  `
})
export class MemoryLeakPreventionComponent implements OnInit, OnDestroy {
  @ViewChild('container', { read: ViewContainerRef })
  container: ViewContainerRef;

  private componentRef: ComponentRef<any>;
  private isActive = true;

  constructor(
    private componentFactoryResolver: ComponentFactoryResolver,
    private changeDetectorRef: ChangeDetectorRef
  ) {}

  ngOnInit(): void {
    this.loadComponent();
  }

  private async loadComponent(): Promise<void> {
    if (!this.isActive) return;

    const module = await import('./dynamic/dynamic.module');
    const component = module.DynamicComponent;
    const factory = this.componentFactoryResolver
      .resolveComponentFactory(component);

    this.componentRef = this.container.createComponent(factory);
    this.changeDetectorRef.detectChanges();
  }

  ngOnDestroy(): void {
    this.isActive = false;
    if (this.componentRef) {
      this.componentRef.destroy();
    }
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

  constructor(
    private cacheService: CacheService,
    private environment: Environment
  ) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    if (request.method !== 'GET') {
      return next.handle(request);
    }

    const cacheKey = this.getCacheKey(request);
    const cachedResponse = this.cache.get(cacheKey);

    if (cachedResponse && !this.isExpired(cachedResponse)) {
      return of(cachedResponse.response);
    }

    return next.handle(request).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          this.cache.set(cacheKey, {
            response: event,
            timestamp: Date.now()
          });
        }
      })
    );
  }

  private getCacheKey(request: HttpRequest<any>): string {
    return `${request.method}-${request.url}`;
  }

  private isExpired(entry: CacheEntry): boolean {
    const maxAge = this.environment.cacheMaxAge;
    return Date.now() - entry.timestamp > maxAge;
  }
}
```

#### Route Data Caching

```typescript
// route-data-cache.service.ts
@Injectable({ providedIn: 'root' })
export class RouteDataCacheService {
  private cache = new Map<string, any>();

  set(key: string, data: any): void {
    this.cache.set(key, {
      data,
      timestamp: Date.now()
    });
  }

  get(key: string): any {
    const entry = this.cache.get(key);
    if (!entry) return null;

    if (this.isExpired(entry)) {
      this.cache.delete(key);
      return null;
    }

    return entry.data;
  }

  private isExpired(entry: CacheEntry): boolean {
    const maxAge = 5 * 60 * 1000; // 5 minutes
    return Date.now() - entry.timestamp > maxAge;
  }

  clear(): void {
    this.cache.clear();
  }
}

// Using in resolver
@Injectable()
export class DataResolver implements Resolve<any> {
  constructor(private cacheService: RouteDataCacheService) {}

  resolve(route: ActivatedRouteSnapshot): Observable<any> {
    const cacheKey = this.getCacheKey(route);
    const cachedData = this.cacheService.get(cacheKey);

    if (cachedData) {
      return of(cachedData);
    }

    return this.loadData(route).pipe(
      tap(data => this.cacheService.set(cacheKey, data))
    );
  }

  private getCacheKey(route: ActivatedRouteSnapshot): string {
    return `${route.routeConfig.path}-${route.params.id}`;
  }

  private loadData(route: ActivatedRouteSnapshot): Observable<any> {
    // Implement data loading logic
    return of(null);
  }
}
```

### Conclusion

Key points to remember:

1. **Change Detection**:
   - Use OnPush strategy
   - Implement custom strategy
   - Optimize with observables
   - Minimize change detection
   - Track performance

2. **Lazy Loading**:
   - Implement preloading
   - Use component loading
   - Optimize loading time
   - Handle loading errors
   - Monitor performance

3. **Bundle Optimization**:
   - Configure Webpack
   - Enable tree shaking
   - Split chunks
   - Compress assets
   - Monitor bundle size

4. **Memory Management**:
   - Manage subscriptions
   - Prevent memory leaks
   - Clean up resources
   - Monitor memory usage
   - Regular maintenance

5. **Caching Strategies**:
   - Implement HTTP caching
   - Cache route data
   - Set cache policies
   - Handle cache invalidation
   - Monitor cache hit rates

Remember to:
- Monitor performance
- Optimize assets
- Regular maintenance
- Test performance
- Document optimizations
- Profile application
- Use performance tools
- Regular audits 