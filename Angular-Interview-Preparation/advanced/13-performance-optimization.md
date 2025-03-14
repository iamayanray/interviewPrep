# Advanced Angular Performance Optimization

## Question
What are the advanced performance optimization techniques in Angular? Explain change detection, lazy loading, and bundle optimization with examples.

## Answer

### Introduction to Advanced Angular Performance Optimization

Angular provides powerful performance optimization capabilities to ensure fast and efficient applications. Here's a comprehensive guide to advanced performance optimization techniques.

### 1. Advanced Change Detection

#### Custom Change Detection Strategy

```typescript
// custom-change-detector.ts
@Injectable()
export class CustomChangeDetector {
  private changeDetectionQueue: Component[] = [];
  private isProcessing = false;
  private readonly BATCH_SIZE = 10;
  private readonly FRAME_DURATION = 16; // ~60fps

  constructor(private zone: NgZone) {}

  markForCheck(component: Component): void {
    if (!this.changeDetectionQueue.includes(component)) {
      this.changeDetectionQueue.push(component);
      this.scheduleChangeDetection();
    }
  }

  private scheduleChangeDetection(): void {
    if (this.isProcessing) return;

    this.isProcessing = true;
    this.zone.runOutsideAngular(() => {
      requestAnimationFrame(() => this.processChangeDetection());
    });
  }

  private processChangeDetection(): void {
    const startTime = performance.now();
    let processedCount = 0;

    while (
      this.changeDetectionQueue.length > 0 &&
      processedCount < this.BATCH_SIZE &&
      performance.now() - startTime < this.FRAME_DURATION
    ) {
      const component = this.changeDetectionQueue.shift();
      if (component) {
        this.detectChanges(component);
        processedCount++;
      }
    }

    if (this.changeDetectionQueue.length > 0) {
      this.isProcessing = false;
      this.scheduleChangeDetection();
    } else {
      this.isProcessing = false;
    }
  }

  private detectChanges(component: Component): void {
    // Implement custom change detection logic
    if (component instanceof ChangeDetectorRef) {
      component.detectChanges();
    }
  }
}

// Using in component
@Component({
  selector: 'app-performance',
  template: `
    <div *ngFor="let item of items">
      {{ item.name }}
    </div>
  `,
  providers: [CustomChangeDetector]
})
export class PerformanceComponent implements OnInit {
  items: any[] = [];

  constructor(
    private customDetector: CustomChangeDetector,
    private cdr: ChangeDetectorRef
  ) {}

  ngOnInit(): void {
    // Mark component for custom change detection
    this.customDetector.markForCheck(this.cdr);
  }

  updateItems(newItems: any[]): void {
    this.items = newItems;
    this.customDetector.markForCheck(this.cdr);
  }
}
```

#### OnPush Change Detection with Observables

```typescript
// performance.component.ts
@Component({
  selector: 'app-performance',
  template: `
    <div *ngIf="data$ | async as data">
      <div *ngFor="let item of data.items">
        {{ item.name }}
      </div>
      <div class="stats">
        Total: {{ data.total }}
        Active: {{ data.active }}
      </div>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class PerformanceComponent implements OnInit {
  data$ = new BehaviorSubject<PerformanceData>({
    items: [],
    total: 0,
    active: 0
  });

  constructor(
    private dataService: DataService,
    private cdr: ChangeDetectorRef
  ) {}

  ngOnInit(): void {
    // Use async pipe for automatic change detection
    this.data$ = this.dataService.getData().pipe(
      shareReplay(1),
      catchError(error => {
        console.error('Error loading data:', error);
        return of({ items: [], total: 0, active: 0 });
      })
    );
  }

  refreshData(): void {
    this.dataService.refreshData().subscribe();
  }
}
```

### 2. Advanced Lazy Loading

#### Preloading Strategy

```typescript
// custom-preloading-strategy.ts
@Injectable({ providedIn: 'root' })
export class CustomPreloadingStrategy implements PreloadAllModules {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (this.shouldPreload(route)) {
      return load();
    }
    return of(null);
  }

  private shouldPreload(route: Route): boolean {
    const preload = route.data?.['preload'];
    if (typeof preload === 'boolean') {
      return preload;
    }

    // Check user interaction
    if (this.hasUserInteraction()) {
      return true;
    }

    // Check device capabilities
    if (this.isHighPerformanceDevice()) {
      return true;
    }

    return false;
  }

  private hasUserInteraction(): boolean {
    return document.visibilityState === 'visible' &&
           !navigator.connection?.saveData;
  }

  private isHighPerformanceDevice(): boolean {
    return navigator.hardwareConcurrency > 4 &&
           !navigator.connection?.saveData;
  }
}

// Using in routing
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    data: { preload: true }
  },
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.module').then(m => m.DashboardModule),
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
// dynamic-component-loader.ts
@Injectable({ providedIn: 'root' })
export class DynamicComponentLoader {
  private componentCache = new Map<string, Type<any>>();

  constructor(
    private componentFactoryResolver: ComponentFactoryResolver,
    private compiler: Compiler
  ) {}

  loadComponent(
    componentPath: string,
    container: ViewContainerRef
  ): Observable<ComponentRef<any>> {
    if (this.componentCache.has(componentPath)) {
      return this.createComponent(
        this.componentCache.get(componentPath)!,
        container
      );
    }

    return this.loadComponentModule(componentPath).pipe(
      switchMap(component => {
        this.componentCache.set(componentPath, component);
        return this.createComponent(component, container);
      })
    );
  }

  private loadComponentModule(
    componentPath: string
  ): Observable<Type<any>> {
    return from(import(componentPath)).pipe(
      map(module => module.default),
      switchMap(component => {
        if (component instanceof NgModule) {
          return this.compileModule(component);
        }
        return of(component);
      })
    );
  }

  private compileModule(module: Type<any>): Observable<Type<any>> {
    return from(this.compiler.compileModuleAsync(module)).pipe(
      map(moduleFactory => {
        const moduleRef = moduleFactory.create(this.injector);
        return moduleRef.instance;
      })
    );
  }

  private createComponent(
    component: Type<any>,
    container: ViewContainerRef
  ): Observable<ComponentRef<any>> {
    const factory = this.componentFactoryResolver.resolveComponentFactory(component);
    container.clear();
    return of(container.createComponent(factory));
  }
}

// Using in component
@Component({
  selector: 'app-dynamic',
  template: `
    <div #container></div>
    <button (click)="loadComponent()">Load Component</button>
  `
})
export class DynamicComponent {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;

  constructor(private loader: DynamicComponentLoader) {}

  loadComponent(): void {
    this.loader
      .loadComponent('./feature/feature.component', this.container)
      .subscribe(componentRef => {
        // Handle component instance
        console.log('Component loaded:', componentRef.instance);
      });
  }
}
```

### 3. Advanced Bundle Optimization

#### Custom Webpack Configuration

```typescript
// webpack.config.js
const path = require('path');
const TerserPlugin = require('terser-webpack-plugin');
const CompressionPlugin = require('compression-webpack-plugin');
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      maxInitialRequests: 25,
      minSize: 20000,
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name(module) {
            const packageName = module.context.match(
              /[\\/]node_modules[\\/](.*?)([\\/]|$)/
            )[1];
            return `vendor.${packageName.replace('@', '')}`;
          },
          priority: 20
        },
        common: {
          minChunks: 2,
          priority: 10,
          reuseExistingChunk: true
        }
      }
    },
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: true,
            drop_debugger: true
          },
          mangle: true,
          output: {
            comments: false
          }
        }
      })
    ]
  },
  plugins: [
    new CompressionPlugin({
      algorithm: 'gzip',
      test: /\.(js|css|html|svg)$/,
      threshold: 10240,
      minRatio: 0.8
    }),
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      reportFilename: 'bundle-report.html'
    })
  ]
};
```

#### Tree Shaking Configuration

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "module": "esnext",
    "moduleResolution": "node",
    "importHelpers": true,
    "target": "es2015",
    "lib": ["es2018", "dom"],
    "skipLibCheck": true,
    "sourceMap": true,
    "declaration": false,
    "downlevelIteration": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "baseUrl": "./",
    "paths": {
      "@app/*": ["src/app/*"],
      "@core/*": ["src/app/core/*"],
      "@shared/*": ["src/app/shared/*"]
    }
  },
  "angularCompilerOptions": {
    "enableI18nLegacyMessageIdFormat": false,
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictTemplates": true
  }
}
```

### 4. Advanced Memory Management

#### Subscription Management

```typescript
// subscription-manager.ts
@Injectable({ providedIn: 'root' })
export class SubscriptionManager {
  private subscriptions = new Map<string, Subscription>();

  add(key: string, subscription: Subscription): void {
    this.unsubscribe(key);
    this.subscriptions.set(key, subscription);
  }

  unsubscribe(key: string): void {
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
  selector: 'app-data',
  template: `
    <div *ngIf="data$ | async as data">
      {{ data | json }}
    </div>
  `
})
export class DataComponent implements OnInit, OnDestroy {
  data$: Observable<any>;

  constructor(
    private dataService: DataService,
    private subscriptionManager: SubscriptionManager
  ) {
    this.data$ = this.dataService.getData();
  }

  ngOnInit(): void {
    // Add subscription to manager
    this.subscriptionManager.add(
      'data',
      this.data$.subscribe()
    );
  }

  ngOnDestroy(): void {
    // Clean up subscription
    this.subscriptionManager.unsubscribe('data');
  }
}
```

#### Memory Leak Prevention

```typescript
// memory-manager.ts
@Injectable({ providedIn: 'root' })
export class MemoryManager {
  private componentInstances = new Map<string, ComponentRef<any>>();
  private readonly MAX_INSTANCES = 10;

  constructor(private componentFactoryResolver: ComponentFactoryResolver) {}

  createComponent(
    component: Type<any>,
    container: ViewContainerRef,
    key: string
  ): ComponentRef<any> {
    // Clean up old instances if limit reached
    if (this.componentInstances.size >= this.MAX_INSTANCES) {
      this.cleanupOldestInstance();
    }

    const factory = this.componentFactoryResolver.resolveComponentFactory(component);
    const componentRef = container.createComponent(factory);
    this.componentInstances.set(key, componentRef);

    return componentRef;
  }

  destroyComponent(key: string): void {
    const componentRef = this.componentInstances.get(key);
    if (componentRef) {
      componentRef.destroy();
      this.componentInstances.delete(key);
    }
  }

  private cleanupOldestInstance(): void {
    const oldestKey = this.componentInstances.keys().next().value;
    this.destroyComponent(oldestKey);
  }

  cleanupAll(): void {
    this.componentInstances.forEach((_, key) => this.destroyComponent(key));
  }
}

// Using in component
@Component({
  selector: 'app-dynamic',
  template: `
    <div #container></div>
    <button (click)="createComponent()">Create Component</button>
  `
})
export class DynamicComponent implements OnDestroy {
  @ViewChild('container', { read: ViewContainerRef })
  container!: ViewContainerRef;

  constructor(private memoryManager: MemoryManager) {}

  createComponent(): void {
    const key = `component-${Date.now()}`;
    this.memoryManager.createComponent(
      DynamicChildComponent,
      this.container,
      key
    );
  }

  ngOnDestroy(): void {
    this.memoryManager.cleanupAll();
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
  private readonly CACHE_DURATION = 5 * 60 * 1000; // 5 minutes

  constructor() {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    if (request.method !== 'GET') {
      return next.handle(request);
    }

    const cacheKey = this.getCacheKey(request);
    const cachedResponse = this.getFromCache(cacheKey);

    if (cachedResponse) {
      return of(cachedResponse);
    }

    return next.handle(request).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          this.addToCache(cacheKey, event);
        }
      })
    );
  }

  private getCacheKey(request: HttpRequest<any>): string {
    return `${request.method}-${request.url}`;
  }

  private getFromCache(key: string): HttpResponse<any> | null {
    const entry = this.cache.get(key);
    if (!entry) return null;

    if (Date.now() - entry.timestamp > this.CACHE_DURATION) {
      this.cache.delete(key);
      return null;
    }

    return entry.response;
  }

  private addToCache(key: string, response: HttpResponse<any>): void {
    this.cache.set(key, {
      response,
      timestamp: Date.now()
    });
  }

  clearCache(): void {
    this.cache.clear();
  }
}
```

#### Route Data Caching

```typescript
// route-cache.service.ts
@Injectable({ providedIn: 'root' })
export class RouteCacheService {
  private cache = new Map<string, CacheEntry>();
  private readonly CACHE_DURATION = 5 * 60 * 1000; // 5 minutes

  constructor(private router: Router) {
    // Clear cache on navigation
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd),
      takeUntil(this.destroy$)
    ).subscribe(() => {
      this.cleanupCache();
    });
  }

  getData<T>(key: string): Observable<T> {
    const entry = this.cache.get(key);
    if (entry && Date.now() - entry.timestamp < this.CACHE_DURATION) {
      return of(entry.data as T);
    }
    return throwError(() => new Error('Cache miss'));
  }

  setData<T>(key: string, data: T): void {
    this.cache.set(key, {
      data,
      timestamp: Date.now()
    });
  }

  private cleanupCache(): void {
    const now = Date.now();
    for (const [key, entry] of this.cache.entries()) {
      if (now - entry.timestamp > this.CACHE_DURATION) {
        this.cache.delete(key);
      }
    }
  }

  clearCache(): void {
    this.cache.clear();
  }
}

// Using in resolver
@Injectable({ providedIn: 'root' })
export class DataResolver implements Resolve<any> {
  constructor(
    private dataService: DataService,
    private cacheService: RouteCacheService
  ) {}

  resolve(route: ActivatedRouteSnapshot): Observable<any> {
    const cacheKey = `route-${route.routeConfig?.path}`;

    return this.cacheService.getData(cacheKey).pipe(
      catchError(() => {
        return this.dataService.getData().pipe(
          tap(data => this.cacheService.setData(cacheKey, data))
        );
      })
    );
  }
}
```

### Conclusion

Key points to remember:

1. **Change Detection**:
   - Use OnPush strategy
   - Implement custom detection
   - Optimize with observables
   - Handle async operations
   - Manage component updates

2. **Lazy Loading**:
   - Implement preloading
   - Load components dynamically
   - Cache loaded modules
   - Handle loading states
   - Optimize bundle size

3. **Bundle Optimization**:
   - Split chunks
   - Enable tree shaking
   - Compress assets
   - Analyze bundles
   - Optimize imports

4. **Memory Management**:
   - Track subscriptions
   - Clean up resources
   - Handle component lifecycle
   - Prevent memory leaks
   - Monitor memory usage

5. **Caching Strategies**:
   - Cache HTTP responses
   - Cache route data
   - Handle cache invalidation
   - Set cache duration
   - Clean up old cache

Remember to:
- Monitor performance
- Profile application
- Optimize assets
- Handle edge cases
- Test performance
- Document optimizations
- Regular maintenance
- Monitor metrics 