# Advanced Angular Performance Optimization

## Question
What are the advanced performance optimization techniques in Angular? Explain change detection, lazy loading, and bundle optimization with examples.

## Answer

### Introduction to Advanced Angular Performance Optimization

Angular provides powerful performance optimization capabilities for complex applications. Here's a comprehensive guide to advanced performance optimization techniques.

### 1. Advanced Change Detection

#### Custom Change Detection Strategy

```typescript
// custom-change-detector.ts
@Injectable()
export class CustomChangeDetector {
  private changeDetectionQueue: Set<any> = new Set();
  private isProcessing = false;
  private readonly BATCH_SIZE = 10;
  private readonly PROCESSING_INTERVAL = 16; // ~60fps

  constructor(
    private changeDetectorRef: ChangeDetectorRef,
    private zone: NgZone
  ) {}

  markForCheck(component: any): void {
    this.changeDetectionQueue.add(component);
    this.scheduleProcessing();
  }

  private scheduleProcessing(): void {
    if (!this.isProcessing) {
      this.isProcessing = true;
      this.zone.runOutsideAngular(() => {
        requestAnimationFrame(() => this.processQueue());
      });
    }
  }

  private processQueue(): void {
    const batch = Array.from(this.changeDetectionQueue).slice(0, this.BATCH_SIZE);
    
    if (batch.length > 0) {
      batch.forEach(component => {
        this.changeDetectorRef.detectChanges();
        this.changeDetectionQueue.delete(component);
      });

      if (this.changeDetectionQueue.size > 0) {
        setTimeout(() => this.processQueue(), this.PROCESSING_INTERVAL);
      } else {
        this.isProcessing = false;
      }
    } else {
      this.isProcessing = false;
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

  constructor(private customChangeDetector: CustomChangeDetector) {}

  ngOnInit() {
    // Simulate data updates
    setInterval(() => {
      this.items = this.items.map(item => ({
        ...item,
        name: `${item.name} updated`
      }));
      this.customChangeDetector.markForCheck(this);
    }, 1000);
  }
}
```

#### OnPush Change Detection with Observables

```typescript
// performance.component.ts
@Component({
  selector: 'app-performance',
  template: `
    <div *ngFor="let item of items$ | async">
      {{ item.name }}
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class PerformanceComponent implements OnInit {
  items$: Observable<any[]>;

  constructor(private dataService: DataService) {
    this.items$ = this.dataService.getItems().pipe(
      shareReplay(1),
      catchError(error => {
        console.error('Error loading items:', error);
        return of([]);
      })
    );
  }

  ngOnInit() {
    // No need to manually trigger change detection
    // The async pipe handles it automatically
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
    if (route.data?.preload === false) {
      return of(null);
    }

    if (route.data?.preload === true) {
      return load();
    }

    // Preload based on user interaction
    return this.shouldPreload(route) ? load() : of(null);
  }

  private shouldPreload(route: Route): boolean {
    // Check if route is frequently accessed
    const isFrequentRoute = this.isFrequentRoute(route);
    
    // Check if user has good network connection
    const hasGoodConnection = this.hasGoodConnection();
    
    // Check if device has enough resources
    const hasEnoughResources = this.hasEnoughResources();

    return isFrequentRoute && hasGoodConnection && hasEnoughResources;
  }

  private isFrequentRoute(route: Route): boolean {
    const frequentRoutes = ['/dashboard', '/profile', '/settings'];
    return frequentRoutes.includes(route.path || '');
  }

  private hasGoodConnection(): boolean {
    return navigator.connection?.effectiveType === '4g' || 
           navigator.connection?.effectiveType === '5g';
  }

  private hasEnoughResources(): boolean {
    const memory = (navigator as any).deviceMemory;
    return memory && memory >= 4; // 4GB or more
  }
}

// Using in app-routing.module.ts
@NgModule({
  imports: [
    RouterModule.forRoot(routes, {
      preloadingStrategy: CustomPreloadingStrategy
    })
  ]
})
export class AppRoutingModule { }
```

#### Component-Level Lazy Loading

```typescript
// dynamic-component-loader.ts
@Injectable({ providedIn: 'root' })
export class DynamicComponentLoader {
  constructor(
    private componentFactoryResolver: ComponentFactoryResolver,
    private viewContainerRef: ViewContainerRef
  ) {}

  async loadComponent(component: Type<any>, data?: any): Promise<ComponentRef<any>> {
    const factory = this.componentFactoryResolver.resolveComponentFactory(component);
    const componentRef = this.viewContainerRef.createComponent(factory);

    if (data) {
      Object.assign(componentRef.instance, data);
    }

    return componentRef;
  }

  clear(): void {
    this.viewContainerRef.clear();
  }
}

// Using in component
@Component({
  selector: 'app-dynamic-content',
  template: `
    <div #container></div>
  `
})
export class DynamicContentComponent implements OnInit {
  @ViewChild('container', { read: ViewContainerRef })
  container: ViewContainerRef;

  constructor(private dynamicLoader: DynamicComponentLoader) {}

  async ngOnInit() {
    // Load components based on user interaction or application state
    if (this.shouldLoadHeavyComponent()) {
      const HeavyComponent = await import('./heavy.component').then(m => m.HeavyComponent);
      await this.dynamicLoader.loadComponent(HeavyComponent);
    }
  }

  private shouldLoadHeavyComponent(): boolean {
    // Check various conditions
    return this.hasGoodConnection() && 
           this.hasEnoughMemory() && 
           this.isUserInterested();
  }

  private hasGoodConnection(): boolean {
    return navigator.connection?.effectiveType === '4g' || 
           navigator.connection?.effectiveType === '5g';
  }

  private hasEnoughMemory(): boolean {
    const memory = (navigator as any).deviceMemory;
    return memory && memory >= 4;
  }

  private isUserInterested(): boolean {
    // Implement your logic to determine if user is interested
    return true;
  }
}
```

### 3. Advanced Bundle Optimization

#### Custom Webpack Configuration

```javascript
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      maxInitialRequests: Infinity,
      minSize: 0,
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name(module) {
            const packageName = module.context.match(/[\\/]node_modules[\\/](.*?)([\\/]|$)/)[1];
            return `vendor.${packageName.replace('@', '')}`;
          },
          priority: 20
        },
        common: {
          name: 'common',
          minChunks: 2,
          priority: 10,
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
    }),
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      reportFilename: 'bundle-report.html'
    })
  ]
};
```

#### Tree Shaking Configuration

```json
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
    "emitDecoratorMetadata": true,
    "types": ["node"],
    "typeRoots": ["node_modules/@types"]
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
    this.subscriptions.forEach(subscription => subscription.unsubscribe());
    this.subscriptions.clear();
  }
}

// Using in component
@Component({
  selector: 'app-subscription-handler',
  template: `
    <div>{{ data$ | async }}</div>
  `
})
export class SubscriptionHandlerComponent implements OnInit, OnDestroy {
  data$: Observable<any>;

  constructor(
    private dataService: DataService,
    private subscriptionManager: SubscriptionManager
  ) {
    this.data$ = this.dataService.getData().pipe(
      shareReplay(1),
      takeUntil(this.destroy$)
    );
  }

  private destroy$ = new Subject<void>();

  ngOnInit() {
    // Add subscription to manager
    this.subscriptionManager.add(
      'data',
      this.data$.subscribe()
    );
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
    this.subscriptionManager.clear();
  }
}
```

#### Memory Leak Prevention

```typescript
// memory-manager.ts
@Injectable({ providedIn: 'root' })
export class MemoryManager {
  private components = new Set<ComponentRef<any>>();
  private readonly MAX_COMPONENTS = 100;

  trackComponent(component: ComponentRef<any>): void {
    this.components.add(component);
    this.checkMemoryUsage();
  }

  destroyComponent(component: ComponentRef<any>): void {
    component.destroy();
    this.components.delete(component);
  }

  private checkMemoryUsage(): void {
    if (this.components.size > this.MAX_COMPONENTS) {
      this.cleanupOldComponents();
    }
  }

  private cleanupOldComponents(): void {
    const components = Array.from(this.components);
    const oldComponents = components.slice(0, components.length - this.MAX_COMPONENTS);
    
    oldComponents.forEach(component => {
      this.destroyComponent(component);
    });
  }
}

// Using in component
@Component({
  selector: 'app-dynamic-content',
  template: `
    <div #container></div>
  `
})
export class DynamicContentComponent implements OnInit {
  @ViewChild('container', { read: ViewContainerRef })
  container: ViewContainerRef;

  constructor(
    private dynamicLoader: DynamicComponentLoader,
    private memoryManager: MemoryManager
  ) {}

  async loadComponent(component: Type<any>): Promise<void> {
    const componentRef = await this.dynamicLoader.loadComponent(component);
    this.memoryManager.trackComponent(componentRef);
  }

  destroyComponent(componentRef: ComponentRef<any>): void {
    this.memoryManager.destroyComponent(componentRef);
  }
}
```

### 5. Advanced Caching Strategies

#### HTTP Caching

```typescript
// http-cache.interceptor.ts
@Injectable()
export class HttpCacheInterceptor implements HttpInterceptor {
  private cache = new Map<string, { data: any; timestamp: number }>();
  private readonly CACHE_DURATION = 5 * 60 * 1000; // 5 minutes

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    if (request.method !== 'GET') {
      return next.handle(request);
    }

    const cachedResponse = this.getFromCache(request.url);
    if (cachedResponse) {
      return of(cachedResponse);
    }

    return next.handle(request).pipe(
      tap(response => {
        if (response instanceof HttpResponse) {
          this.addToCache(request.url, response.body);
        }
      })
    );
  }

  private getFromCache(url: string): HttpResponse<any> | null {
    const cached = this.cache.get(url);
    if (cached && Date.now() - cached.timestamp < this.CACHE_DURATION) {
      return new HttpResponse({
        body: cached.data,
        status: 200
      });
    }
    return null;
  }

  private addToCache(url: string, data: any): void {
    this.cache.set(url, {
      data,
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
  private cache = new Map<string, { data: any; timestamp: number }>();
  private readonly CACHE_DURATION = 5 * 60 * 1000; // 5 minutes

  getData(route: ActivatedRoute): Observable<any> {
    const key = this.getCacheKey(route);
    const cached = this.cache.get(key);

    if (cached && Date.now() - cached.timestamp < this.CACHE_DURATION) {
      return of(cached.data);
    }

    return this.fetchData(route).pipe(
      tap(data => this.addToCache(key, data))
    );
  }

  private getCacheKey(route: ActivatedRoute): string {
    return route.snapshot.url.map(segment => segment.path).join('/');
  }

  private addToCache(key: string, data: any): void {
    this.cache.set(key, {
      data,
      timestamp: Date.now()
    });
  }

  private fetchData(route: ActivatedRoute): Observable<any> {
    // Implement your data fetching logic
    return of(null);
  }

  clearCache(): void {
    this.cache.clear();
  }
}
```

### Conclusion

Key points to remember:

1. **Change Detection**:
   - Use OnPush strategy
   - Implement custom strategies
   - Optimize with observables
   - Handle async operations
   - Manage change detection cycles

2. **Lazy Loading**:
   - Implement preloading strategies
   - Load components dynamically
   - Handle loading states
   - Manage dependencies
   - Optimize bundle size

3. **Bundle Optimization**:
   - Configure webpack
   - Implement tree shaking
   - Split chunks
   - Compress assets
   - Analyze bundles

4. **Memory Management**:
   - Track subscriptions
   - Clean up resources
   - Handle component lifecycle
   - Monitor memory usage
   - Prevent memory leaks

5. **Caching Strategies**:
   - Cache HTTP responses
   - Cache route data
   - Manage cache duration
   - Handle cache invalidation
   - Optimize cache size

Remember to:
- Monitor performance
- Profile applications
- Use performance tools
- Test on different devices
- Handle edge cases
- Follow best practices
- Maintain code quality
- Document optimizations 