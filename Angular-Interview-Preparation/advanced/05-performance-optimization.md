# Advanced Angular Performance Optimization

## Question
What are the advanced performance optimization techniques in Angular? Explain change detection, lazy loading, and bundle optimization with examples.

## Answer

### Introduction to Advanced Angular Performance Optimization

Angular provides powerful performance optimization capabilities for complex applications. Here's a comprehensive guide to advanced performance optimization techniques.

### 1. Advanced Change Detection

#### Custom Change Detection Strategy

```typescript
// custom-change-detection.ts
@Injectable()
export class CustomChangeDetectorRef extends ChangeDetectorRef {
  private changeDetectionQueue: Set<any> = new Set();
  private isProcessing = false;

  markForCheck(component: any): void {
    this.changeDetectionQueue.add(component);
    if (!this.isProcessing) {
      this.processChangeDetectionQueue();
    }
  }

  private processChangeDetectionQueue(): void {
    this.isProcessing = true;
    requestAnimationFrame(() => {
      this.changeDetectionQueue.forEach(component => {
        component.detectChanges();
      });
      this.changeDetectionQueue.clear();
      this.isProcessing = false;
    });
  }
}

// Using in component
@Component({
  selector: 'app-performance',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div *ngFor="let item of items; trackBy: trackByFn">
      {{ item.name }}
    </div>
  `
})
export class PerformanceComponent {
  items: Item[] = [];

  constructor(private cdr: CustomChangeDetectorRef) {}

  trackByFn(index: number, item: Item): number {
    return item.id;
  }

  updateItems(newItems: Item[]): void {
    this.items = newItems;
    this.cdr.markForCheck(this);
  }
}
```

#### OnPush Change Detection with Observables

```typescript
// data.component.ts
@Component({
  selector: 'app-data',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div *ngIf="data$ | async as data">
      <h2>{{ data.title }}</h2>
      <p>{{ data.description }}</p>
    </div>
  `
})
export class DataComponent {
  data$ = this.dataService.getData().pipe(
    shareReplay(1),
    catchError(error => {
      console.error('Error loading data:', error);
      return of(null);
    })
  );

  constructor(private dataService: DataService) {}
}
```

### 2. Advanced Lazy Loading

#### Preloading Strategy

```typescript
// custom-preloading-strategy.ts
@Injectable({ providedIn: 'root' })
export class CustomPreloadingStrategy implements PreloadAllModules {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (route.data?.['preload'] === false) {
      return of(null);
    }

    const preloadDelay = route.data?.['preloadDelay'] || 0;
    return timer(preloadDelay).pipe(
      mergeMap(() => load())
    );
  }
}

// Using in routing
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    data: { preload: true, preloadDelay: 5000 }
  },
  {
    path: 'dashboard',
    loadChildren: () => import('./dashboard/dashboard.module').then(m => m.DashboardModule),
    data: { preload: false }
  }
];

// Using in app.module.ts
@NgModule({
  imports: [
    RouterModule.forRoot(routes, {
      preloadingStrategy: CustomPreloadingStrategy
    })
  ]
})
export class AppModule { }
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

  async loadComponent(component: Type<any>, data?: any): Promise<void> {
    const factory = this.componentFactoryResolver.resolveComponentFactory(component);
    const componentRef = this.viewContainerRef.createComponent(factory);
    
    if (data) {
      Object.assign(componentRef.instance, data);
    }
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
  container!: ViewContainerRef;

  constructor(private loader: DynamicComponentLoader) {}

  async ngOnInit() {
    await this.loader.loadComponent(HeavyComponent, {
      data: { id: 1 }
    });
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
      minSize: 20000,
      maxSize: 244000,
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
          priority: 20
        },
        common: {
          name: 'common',
          minChunks: 2,
          chunks: 'all',
          priority: 10,
          reuseExistingChunk: true,
          enforce: true
        }
      }
    }
  }
};
```

#### Tree Shaking Configuration

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "importHelpers": true,
    "target": "es2015",
    "module": "es2020",
    "moduleResolution": "node",
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noPropertyAccessFromIndexSignature": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  }
}
```

### 4. Advanced Memory Management

#### Subscription Management

```typescript
// subscription-manager.ts
@Injectable({ providedIn: 'root' })
export class SubscriptionManager {
  private subscriptions = new Set<Subscription>();

  add(subscription: Subscription): void {
    this.subscriptions.add(subscription);
  }

  remove(subscription: Subscription): void {
    this.subscriptions.delete(subscription);
  }

  unsubscribeAll(): void {
    this.subscriptions.forEach(sub => sub.unsubscribe());
    this.subscriptions.clear();
  }
}

// Using in component
@Component({
  selector: 'app-data-stream',
  template: `
    <div *ngFor="let item of items">
      {{ item.name }}
    </div>
  `
})
export class DataStreamComponent implements OnInit, OnDestroy {
  items: Item[] = [];

  constructor(
    private dataService: DataService,
    private subscriptionManager: SubscriptionManager
  ) {}

  ngOnInit() {
    const subscription = this.dataService.getDataStream().subscribe(
      items => this.items = items
    );
    this.subscriptionManager.add(subscription);
  }

  ngOnDestroy() {
    this.subscriptionManager.unsubscribeAll();
  }
}
```

#### Memory Leak Prevention

```typescript
// memory-leak-prevention.ts
@Injectable({ providedIn: 'root' })
export class MemoryLeakPrevention {
  private componentRefs = new Set<ComponentRef<any>>();

  trackComponent(componentRef: ComponentRef<any>): void {
    this.componentRefs.add(componentRef);
  }

  destroyComponent(componentRef: ComponentRef<any>): void {
    this.componentRefs.delete(componentRef);
    componentRef.destroy();
  }

  destroyAllComponents(): void {
    this.componentRefs.forEach(ref => ref.destroy());
    this.componentRefs.clear();
  }
}

// Using in service
@Injectable({ providedIn: 'root' })
export class DynamicComponentService {
  constructor(
    private componentFactoryResolver: ComponentFactoryResolver,
    private memoryLeakPrevention: MemoryLeakPrevention
  ) {}

  createComponent(component: Type<any>, container: ViewContainerRef): ComponentRef<any> {
    const factory = this.componentFactoryResolver.resolveComponentFactory(component);
    const componentRef = container.createComponent(factory);
    this.memoryLeakPrevention.trackComponent(componentRef);
    return componentRef;
  }

  destroyComponent(componentRef: ComponentRef<any>): void {
    this.memoryLeakPrevention.destroyComponent(componentRef);
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
      tap(event => {
        if (event instanceof HttpResponse) {
          this.addToCache(request.url, event);
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

  private addToCache(url: string, response: HttpResponse<any>): void {
    this.cache.set(url, {
      data: response.body,
      timestamp: Date.now()
    });
  }
}
```

#### Route Data Caching

```typescript
// route-data-cache.service.ts
@Injectable({ providedIn: 'root' })
export class RouteDataCacheService {
  private cache = new Map<string, any>();

  set(route: string, data: any): void {
    this.cache.set(route, {
      data,
      timestamp: Date.now()
    });
  }

  get(route: string): any | null {
    const cached = this.cache.get(route);
    if (cached && Date.now() - cached.timestamp < 5 * 60 * 1000) {
      return cached.data;
    }
    return null;
  }

  clear(): void {
    this.cache.clear();
  }
}

// Using in resolver
@Injectable({ providedIn: 'root' })
export class CachedDataResolver implements Resolve<any> {
  constructor(
    private dataService: DataService,
    private cacheService: RouteDataCacheService
  ) {}

  resolve(route: ActivatedRouteSnapshot): Observable<any> {
    const cachedData = this.cacheService.get(route.routeConfig?.path || '');
    if (cachedData) {
      return of(cachedData);
    }

    return this.dataService.getData().pipe(
      tap(data => this.cacheService.set(route.routeConfig?.path || '', data))
    );
  }
}
```

### Conclusion

Key points to remember:

1. **Change Detection**:
   - Use OnPush strategy
   - Implement custom change detection
   - Optimize template bindings
   - Use trackBy functions

2. **Lazy Loading**:
   - Implement custom preloading
   - Use component-level lazy loading
   - Optimize bundle splitting
   - Configure preloading delays

3. **Bundle Optimization**:
   - Configure webpack
   - Enable tree shaking
   - Split chunks
   - Optimize imports

4. **Memory Management**:
   - Track subscriptions
   - Prevent memory leaks
   - Clean up resources
   - Monitor memory usage

5. **Caching Strategies**:
   - Implement HTTP caching
   - Cache route data
   - Use service workers
   - Configure cache duration

Remember to:
- Monitor performance metrics
- Use performance profiling tools
- Implement proper error handling
- Follow Angular best practices
- Test performance optimizations
- Monitor memory usage
- Optimize bundle size
- Use appropriate caching strategies 