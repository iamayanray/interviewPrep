# Angular Routing

## Question
Explain Angular's routing system. How do you set up routes, handle route parameters, and implement route guards?

## Answer

### Introduction to Angular Routing

Angular's routing system allows you to build single-page applications (SPAs) with multiple views and navigate between them without page reloads. The Router enables navigation from one view to another as users perform application tasks.

### Setting Up the Router

#### 1. Basic Router Setup

```typescript
// app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { HomeComponent } from './home/home.component';
import { AboutComponent } from './about/about.component';
import { ContactComponent } from './contact/contact.component';
import { NotFoundComponent } from './not-found/not-found.component';

const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent },
  { path: 'contact', component: ContactComponent },
  { path: '**', component: NotFoundComponent } // Wildcard route for 404
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

#### 2. Adding Router Outlet

```html
<!-- app.component.html -->
<nav>
  <a routerLink="/" routerLinkActive="active" [routerLinkActiveOptions]="{exact: true}">Home</a>
  <a routerLink="/about" routerLinkActive="active">About</a>
  <a routerLink="/contact" routerLinkActive="active">Contact</a>
</nav>

<router-outlet></router-outlet>
```

#### 3. Importing the Routing Module

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';
import { HomeComponent } from './home/home.component';
import { AboutComponent } from './about/about.component';
import { ContactComponent } from './contact/contact.component';
import { NotFoundComponent } from './not-found/not-found.component';

@NgModule({
  declarations: [
    AppComponent,
    HomeComponent,
    AboutComponent,
    ContactComponent,
    NotFoundComponent
  ],
  imports: [
    BrowserModule,
    AppRoutingModule
  ],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

### Route Parameters

#### 1. Defining Routes with Parameters

```typescript
const routes: Routes = [
  { path: 'product/:id', component: ProductDetailComponent }
];
```

#### 2. Accessing Route Parameters

```typescript
// product-detail.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

@Component({
  selector: 'app-product-detail',
  template: `<h2>Product Details for ID: {{productId}}</h2>`
})
export class ProductDetailComponent implements OnInit {
  productId: string;

  constructor(private route: ActivatedRoute) { }

  ngOnInit() {
    // Method 1: Snapshot (doesn't update if params change while component is active)
    this.productId = this.route.snapshot.paramMap.get('id');

    // Method 2: Observable (updates if params change while component is active)
    this.route.paramMap.subscribe(params => {
      this.productId = params.get('id');
    });
  }
}
```

#### 3. Optional Parameters

```typescript
// In the component
this.router.navigate(['/products'], { 
  queryParams: { category: 'electronics', sort: 'price' } 
});

// Accessing query parameters
this.route.queryParamMap.subscribe(params => {
  this.category = params.get('category');
  this.sortBy = params.get('sort');
});
```

### Route Navigation

#### 1. Imperative Navigation

```typescript
// In a component
import { Router } from '@angular/router';

@Component({...})
export class SomeComponent {
  constructor(private router: Router) {}

  navigateToProducts() {
    this.router.navigate(['/products']);
  }

  navigateToProductDetail(id: string) {
    this.router.navigate(['/product', id]);
  }

  navigateWithQueryParams() {
    this.router.navigate(['/products'], { 
      queryParams: { category: 'electronics' } 
    });
  }

  navigateWithPreserveQueryParams() {
    this.router.navigate(['/products'], { 
      queryParamsHandling: 'preserve' // or 'merge'
    });
  }
}
```

#### 2. Declarative Navigation

```html
<!-- Simple navigation -->
<a routerLink="/products">Products</a>

<!-- Dynamic route parameters -->
<a [routerLink]="['/product', product.id]">{{ product.name }}</a>

<!-- With query parameters -->
<a [routerLink]="['/products']" [queryParams]="{category: 'electronics'}">Electronics</a>

<!-- Active route styling -->
<a routerLink="/products" routerLinkActive="active-link">Products</a>

<!-- Exact path matching -->
<a routerLink="/" routerLinkActive="active" [routerLinkActiveOptions]="{exact: true}">Home</a>
```

### Route Guards

Route guards are interfaces that tell the router whether or not it should allow navigation to a requested route.

#### 1. CanActivate Guard

Checks if a user can access a route.

```typescript
// auth.guard.ts
import { Injectable } from '@angular/core';
import { CanActivate, ActivatedRouteSnapshot, RouterStateSnapshot, Router } from '@angular/router';
import { AuthService } from './auth.service';

@Injectable({
  providedIn: 'root'
})
export class AuthGuard implements CanActivate {
  constructor(private authService: AuthService, private router: Router) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): boolean {
    if (this.authService.isAuthenticated()) {
      return true;
    } else {
      this.router.navigate(['/login'], { 
        queryParams: { returnUrl: state.url } 
      });
      return false;
    }
  }
}

// Using the guard in routes
const routes: Routes = [
  { 
    path: 'admin', 
    component: AdminComponent, 
    canActivate: [AuthGuard] 
  }
];
```

#### 2. CanActivateChild Guard

Checks if a user can access any child route of a route.

```typescript
// admin.guard.ts
import { Injectable } from '@angular/core';
import { CanActivateChild, ActivatedRouteSnapshot, RouterStateSnapshot } from '@angular/router';
import { AuthService } from './auth.service';

@Injectable({
  providedIn: 'root'
})
export class AdminGuard implements CanActivateChild {
  constructor(private authService: AuthService) {}

  canActivateChild(
    childRoute: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): boolean {
    return this.authService.isAdmin();
  }
}

// Using the guard in routes
const routes: Routes = [
  { 
    path: 'admin', 
    component: AdminComponent,
    canActivateChild: [AdminGuard],
    children: [
      { path: 'users', component: AdminUsersComponent },
      { path: 'settings', component: AdminSettingsComponent }
    ]
  }
];
```

#### 3. CanDeactivate Guard

Checks if a user can leave a route (useful for unsaved changes).

```typescript
// can-deactivate.guard.ts
import { Injectable } from '@angular/core';
import { CanDeactivate } from '@angular/router';
import { Observable } from 'rxjs';

export interface CanComponentDeactivate {
  canDeactivate: () => Observable<boolean> | Promise<boolean> | boolean;
}

@Injectable({
  providedIn: 'root'
})
export class CanDeactivateGuard implements CanDeactivate<CanComponentDeactivate> {
  canDeactivate(
    component: CanComponentDeactivate
  ): Observable<boolean> | Promise<boolean> | boolean {
    return component.canDeactivate ? component.canDeactivate() : true;
  }
}

// In the component
@Component({...})
export class EditFormComponent implements CanComponentDeactivate {
  isDirty = false;

  canDeactivate(): boolean {
    if (this.isDirty) {
      return window.confirm('You have unsaved changes. Do you really want to leave?');
    }
    return true;
  }
}

// Using the guard in routes
const routes: Routes = [
  { 
    path: 'edit/:id', 
    component: EditFormComponent, 
    canDeactivate: [CanDeactivateGuard] 
  }
];
```

#### 4. Resolve Guard

Pre-fetches data before a route is activated.

```typescript
// product-resolver.service.ts
import { Injectable } from '@angular/core';
import { Resolve, ActivatedRouteSnapshot, RouterStateSnapshot } from '@angular/router';
import { Observable, of } from 'rxjs';
import { catchError } from 'rxjs/operators';
import { Product } from './product.model';
import { ProductService } from './product.service';

@Injectable({
  providedIn: 'root'
})
export class ProductResolver implements Resolve<Product> {
  constructor(private productService: ProductService) {}

  resolve(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<Product> {
    const id = route.paramMap.get('id');
    return this.productService.getProduct(id).pipe(
      catchError(error => {
        console.error('Error resolving product', error);
        return of(null);
      })
    );
  }
}

// Using the resolver in routes
const routes: Routes = [
  { 
    path: 'product/:id', 
    component: ProductDetailComponent, 
    resolve: { product: ProductResolver } 
  }
];

// Accessing resolved data in component
@Component({...})
export class ProductDetailComponent implements OnInit {
  product: Product;

  constructor(private route: ActivatedRoute) {}

  ngOnInit() {
    this.route.data.subscribe(data => {
      this.product = data.product;
    });
  }
}
```

#### 5. CanLoad Guard

Prevents lazy loaded modules from being loaded if a user doesn't have permission.

```typescript
// auth.guard.ts
import { Injectable } from '@angular/core';
import { CanLoad, Route, UrlSegment, Router } from '@angular/router';
import { AuthService } from './auth.service';

@Injectable({
  providedIn: 'root'
})
export class AuthGuard implements CanLoad {
  constructor(private authService: AuthService, private router: Router) {}

  canLoad(route: Route, segments: UrlSegment[]): boolean {
    if (this.authService.isAuthenticated()) {
      return true;
    } else {
      this.router.navigate(['/login']);
      return false;
    }
  }
}

// Using the guard in routes
const routes: Routes = [
  { 
    path: 'admin', 
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule), 
    canLoad: [AuthGuard] 
  }
];
```

### Lazy Loading

Lazy loading helps load modules only when needed, improving initial load time.

```typescript
// app-routing.module.ts
const routes: Routes = [
  { path: '', component: HomeComponent },
  { 
    path: 'products', 
    loadChildren: () => import('./products/products.module').then(m => m.ProductsModule) 
  },
  { 
    path: 'admin', 
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule),
    canLoad: [AuthGuard]
  }
];

// products-routing.module.ts
const routes: Routes = [
  { path: '', component: ProductListComponent },
  { path: ':id', component: ProductDetailComponent }
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule]
})
export class ProductsRoutingModule { }
```

### Named Outlets

Named outlets allow multiple router outlets in the same template.

```typescript
// app-routing.module.ts
const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'sidebar', component: SidebarComponent, outlet: 'sidebar' },
  { path: 'chat', component: ChatComponent, outlet: 'chat' }
];

// app.component.html
<router-outlet></router-outlet>
<router-outlet name="sidebar"></router-outlet>
<router-outlet name="chat"></router-outlet>

// Navigating to named outlets
this.router.navigate([{
  outlets: {
    primary: ['home'],
    sidebar: ['sidebar'],
    chat: ['chat']
  }
}]);
```

### Route Events

Angular Router emits events during the navigation lifecycle.

```typescript
// app.component.ts
import { Router, NavigationStart, NavigationEnd, NavigationCancel, NavigationError } from '@angular/router';
import { filter } from 'rxjs/operators';

@Component({...})
export class AppComponent implements OnInit {
  loading = false;

  constructor(private router: Router) {}

  ngOnInit() {
    this.router.events.subscribe(event => {
      if (event instanceof NavigationStart) {
        this.loading = true;
      } else if (
        event instanceof NavigationEnd ||
        event instanceof NavigationCancel ||
        event instanceof NavigationError
      ) {
        this.loading = false;
      }
    });

    // Or using RxJS operators
    this.router.events.pipe(
      filter(event => event instanceof NavigationEnd)
    ).subscribe((event: NavigationEnd) => {
      // Analytics tracking
      console.log('Navigation completed:', event.url);
    });
  }
}
```

### Location Strategies

Angular supports two location strategies:

1. **PathLocationStrategy** (default): Uses the HTML5 History API (URLs like `/products/1`)
2. **HashLocationStrategy**: Uses hash-based URLs (like `/#/products/1`)

```typescript
// app-routing.module.ts
@NgModule({
  imports: [RouterModule.forRoot(routes, { useHash: true })], // Enable hash strategy
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

### Route Configuration Options

```typescript
const routes: Routes = [
  { 
    path: 'products', 
    component: ProductsComponent,
    data: { title: 'Products', permissions: ['view-products'] }, // Custom data
    children: [...], // Child routes
    canActivate: [AuthGuard], // Guards
    resolve: { products: ProductsResolver }, // Resolvers
    runGuardsAndResolvers: 'always' // When to run guards and resolvers
  }
];
```

### Routing in Standalone Components

With Angular 14+, you can use routing with standalone components:

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { HomeComponent } from './home/home.component';
import { AboutComponent } from './about/about.component';

export const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'about', component: AboutComponent }
];

// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter } from '@angular/router';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes)
  ]
});
```

### Best Practices

1. **Organize Routes by Feature**:
   ```typescript
   // Feature module routing
   const routes: Routes = [
     { path: '', component: FeatureHomeComponent },
     { path: 'details', component: FeatureDetailsComponent }
   ];
   ```

2. **Use Route Guards Appropriately**:
   ```typescript
   // Use canActivate for authentication
   // Use canDeactivate for unsaved changes
   // Use resolve for pre-fetching data
   ```

3. **Implement Lazy Loading**:
   ```typescript
   { 
     path: 'feature', 
     loadChildren: () => import('./feature/feature.module').then(m => m.FeatureModule) 
   }
   ```

4. **Handle Route Parameters Safely**:
   ```typescript
   // Use paramMap.get with null checks
   const id = this.route.snapshot.paramMap.get('id');
   if (id) {
     // Process the id
   }
   ```

5. **Implement Error Handling**:
   ```typescript
   // Add a wildcard route at the end
   { path: '**', component: NotFoundComponent }
   ```

### Common Pitfalls

1. **Forgetting to Export RouterModule**:
   ```typescript
   // Always include this in your routing modules
   @NgModule({
     imports: [RouterModule.forChild(routes)],
     exports: [RouterModule] // Don't forget this
   })
   ```

2. **Route Order Matters**:
   ```typescript
   // Specific routes should come before wildcard routes
   const routes: Routes = [
     { path: 'specific', component: SpecificComponent },
     // ... other specific routes
     { path: '**', component: NotFoundComponent } // Always last
   ];
   ```

3. **Not Handling Unsubscription**:
   ```typescript
   // Always unsubscribe from router events
   private subscription: Subscription;

   ngOnInit() {
     this.subscription = this.router.events.subscribe(/* ... */);
   }

   ngOnDestroy() {
     this.subscription.unsubscribe();
   }
   ```

### Conclusion

Angular's routing system is a powerful feature that enables the creation of complex single-page applications with multiple views. By understanding how to set up routes, handle parameters, implement guards, and use lazy loading, you can create efficient and secure navigation in your Angular applications. The router provides a complete solution for navigation with features like route guards, resolvers, and events that help you control the user experience throughout the application lifecycle. 