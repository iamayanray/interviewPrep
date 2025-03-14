# Advanced Angular Routing

## Question
What are the advanced routing concepts in Angular? Explain route resolvers, guards, and custom route strategies.

## Answer

### Introduction to Advanced Angular Routing

Angular provides powerful routing capabilities for complex navigation scenarios. Here's a comprehensive guide to advanced routing concepts.

### 1. Route Resolvers

#### Data Resolver

```typescript
// user.resolver.ts
@Injectable({ providedIn: 'root' })
export class UserResolver implements Resolve<User> {
  constructor(private userService: UserService) {}

  resolve(route: ActivatedRouteSnapshot): Observable<User> {
    const id = route.paramMap.get('id');
    return this.userService.getUser(+id!);
  }
}

// Using in routing
const routes: Routes = [
  {
    path: 'user/:id',
    component: UserComponent,
    resolve: {
      user: UserResolver
    }
  }
];

// Using in component
@Component({
  selector: 'app-user',
  template: `
    <div *ngIf="user$ | async as user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserComponent implements OnInit {
  user$ = this.route.data.pipe(
    map(data => data['user'])
  );

  constructor(private route: ActivatedRoute) {}
}
```

#### Multiple Data Resolvers

```typescript
// user-data.resolver.ts
@Injectable({ providedIn: 'root' })
export class UserDataResolver implements Resolve<UserData> {
  constructor(private userService: UserService) {}

  resolve(route: ActivatedRouteSnapshot): Observable<UserData> {
    const id = route.paramMap.get('id');
    return this.userService.getUserData(+id!);
  }
}

// user-posts.resolver.ts
@Injectable({ providedIn: 'root' })
export class UserPostsResolver implements Resolve<Post[]> {
  constructor(private postService: PostService) {}

  resolve(route: ActivatedRouteSnapshot): Observable<Post[]> {
    const id = route.paramMap.get('id');
    return this.postService.getUserPosts(+id!);
  }
}

// Using in routing
const routes: Routes = [
  {
    path: 'user/:id',
    component: UserProfileComponent,
    resolve: {
      userData: UserDataResolver,
      userPosts: UserPostsResolver
    }
  }
];
```

### 2. Advanced Route Guards

#### CanActivate Guard with Role Check

```typescript
// role.guard.ts
@Injectable({ providedIn: 'root' })
export class RoleGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): boolean {
    const requiredRoles = route.data['roles'] as string[];
    const userRoles = this.authService.getUserRoles();

    if (requiredRoles.some(role => userRoles.includes(role))) {
      return true;
    }

    this.router.navigate(['/unauthorized']);
    return false;
  }
}

// Using in routing
const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [RoleGuard],
    data: { roles: ['ADMIN'] }
  }
];
```

#### CanDeactivate Guard

```typescript
// unsaved-changes.guard.ts
@Injectable({ providedIn: 'root' })
export class UnsavedChangesGuard implements CanDeactivate<CanComponentDeactivate> {
  canDeactivate(
    component: CanComponentDeactivate,
    currentRoute: ActivatedRouteSnapshot,
    currentState: RouterStateSnapshot,
    nextState?: RouterStateSnapshot
  ): boolean | Observable<boolean> {
    if (component.hasUnsavedChanges()) {
      return confirm('You have unsaved changes. Do you want to leave?');
    }
    return true;
  }
}

// Interface for components that can be deactivated
export interface CanComponentDeactivate {
  hasUnsavedChanges(): boolean;
}

// Using in component
@Component({
  selector: 'app-user-form',
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <input formControlName="name">
      <button type="submit">Save</button>
    </form>
  `
})
export class UserFormComponent implements CanComponentDeactivate {
  userForm = this.fb.group({
    name: ['', Validators.required]
  });

  hasUnsavedChanges(): boolean {
    return this.userForm.dirty;
  }
}

// Using in routing
const routes: Routes = [
  {
    path: 'user/edit',
    component: UserFormComponent,
    canDeactivate: [UnsavedChangesGuard]
  }
];
```

### 3. Custom Route Reuse Strategy

```typescript
// custom-route-reuse-strategy.ts
@Injectable()
export class CustomRouteReuseStrategy implements RouteReuseStrategy {
  private storedRoutes = new Map<string, DetachedRouteHandle>();

  shouldDetach(route: ActivatedRouteSnapshot): boolean {
    return route.data['shouldReuse'] === true;
  }

  store(route: ActivatedRouteSnapshot, handle: DetachedRouteHandle): void {
    this.storedRoutes.set(this.getPath(route), handle);
  }

  shouldAttach(route: ActivatedRouteSnapshot): boolean {
    return this.storedRoutes.has(this.getPath(route));
  }

  retrieve(route: ActivatedRouteSnapshot): DetachedRouteHandle | null {
    return this.storedRoutes.get(this.getPath(route)) || null;
  }

  shouldReuseRoute(future: ActivatedRouteSnapshot, curr: ActivatedRouteSnapshot): boolean {
    return future.routeConfig === curr.routeConfig;
  }

  private getPath(route: ActivatedRouteSnapshot): string {
    return route.pathFromRoot
      .filter(v => v.routeConfig)
      .map(v => v.routeConfig!.path)
      .join('/');
  }
}

// Using in app.module.ts
@NgModule({
  providers: [
    { provide: RouteReuseStrategy, useClass: CustomRouteReuseStrategy }
  ]
})
export class AppModule { }
```

### 4. Lazy Loading with Preloading Strategy

```typescript
// custom-preloading-strategy.ts
@Injectable({ providedIn: 'root' })
export class CustomPreloadingStrategy implements PreloadAllModules {
  preload(route: Route, load: () => Observable<any>): Observable<any> {
    if (route.data?.['preload'] === false) {
      return of(null);
    }
    return load();
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

### 5. Route Data and Query Parameters

```typescript
// Using route data
@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="userData$ | async as userData">
      <h2>{{ userData.title }}</h2>
      <p>{{ userData.description }}</p>
    </div>
  `
})
export class UserProfileComponent {
  userData$ = this.route.data.pipe(
    map(data => data['userData'])
  );

  constructor(private route: ActivatedRoute) {}
}

// Using query parameters
@Component({
  selector: 'app-search',
  template: `
    <input [(ngModel)]="searchTerm" (keyup)="onSearch()">
    <div *ngFor="let result of searchResults$ | async">
      {{ result.name }}
    </div>
  `
})
export class SearchComponent implements OnInit {
  searchTerm = '';
  searchResults$: Observable<SearchResult[]>;

  constructor(
    private route: ActivatedRoute,
    private searchService: SearchService
  ) {
    this.searchResults$ = this.route.queryParams.pipe(
      map(params => params['q'] || ''),
      switchMap(term => this.searchService.search(term))
    );
  }

  onSearch() {
    this.router.navigate([], {
      relativeTo: this.route,
      queryParams: { q: this.searchTerm },
      queryParamsHandling: 'merge'
    });
  }
}
```

### 6. Custom Route Matcher

```typescript
// custom-route-matcher.ts
export function customRouteMatcher(segments: UrlSegment[]): UrlMatchResult | null {
  if (segments.length === 0) {
    return null;
  }

  const path = segments[0].path;
  if (path === 'custom') {
    return {
      consumed: segments,
      posParams: {
        id: segments[1]
      }
    };
  }

  return null;
}

// Using in routing
const routes: Routes = [
  {
    matcher: customRouteMatcher,
    component: CustomComponent
  }
];
```

### Conclusion

Key points to remember:

1. **Route Resolvers**:
   - Pre-fetch data before component activation
   - Handle multiple data dependencies
   - Improve user experience

2. **Route Guards**:
   - Protect routes based on conditions
   - Handle unsaved changes
   - Implement role-based access

3. **Route Reuse Strategy**:
   - Customize component reuse
   - Optimize performance
   - Handle component state

4. **Lazy Loading**:
   - Implement custom preloading
   - Optimize bundle size
   - Improve initial load time

5. **Route Data and Parameters**:
   - Handle route data
   - Manage query parameters
   - Implement search functionality

Remember to:
- Use appropriate guards
- Implement proper data resolvers
- Handle route parameters correctly
- Optimize lazy loading
- Consider performance implications
- Follow Angular best practices
- Test navigation scenarios
- Handle edge cases 