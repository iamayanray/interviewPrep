# Angular Performance Optimization

## Question
How do you optimize Angular applications for better performance? Explain different optimization techniques and best practices.

## Answer

### Introduction to Angular Performance Optimization

Angular provides various techniques to optimize application performance. Here's a comprehensive guide to the most important optimization strategies.

### 1. Change Detection Optimization

#### OnPush Change Detection Strategy

```typescript
// user.component.ts
@Component({
  selector: 'app-user',
  template: `
    <div>
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserComponent {
  @Input() user: User | null = null;
}
```

#### Using TrackBy Function

```typescript
// user-list.component.ts
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users; trackBy: trackByUserId">
      {{ user.name }}
    </div>
  `
})
export class UserListComponent {
  users: User[] = [];

  trackByUserId(index: number, user: User): number {
    return user.id;
  }
}
```

### 2. Lazy Loading

#### Module Lazy Loading

```typescript
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'users',
    loadChildren: () => import('./users/users.module').then(m => m.UsersModule)
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  }
];

// users.module.ts
@NgModule({
  imports: [
    CommonModule,
    RouterModule.forChild([
      { path: '', component: UserListComponent },
      { path: ':id', component: UserDetailComponent }
    ])
  ],
  declarations: [UserListComponent, UserDetailComponent]
})
export class UsersModule { }
```

#### Component Lazy Loading

```typescript
// app.component.ts
@Component({
  selector: 'app-root',
  template: `
    <button (click)="loadUserComponent()">Load User Component</button>
    <ng-container #userComponentContainer></ng-container>
  `
})
export class AppComponent {
  @ViewChild('userComponentContainer', { read: ViewContainerRef })
  container!: ViewContainerRef;

  async loadUserComponent() {
    const { UserComponent } = await import('./user/user.component');
    this.container.createComponent(UserComponent);
  }
}
```

### 3. Virtual Scrolling

```typescript
// user-list.component.ts
@Component({
  selector: 'app-user-list',
  template: `
    <cdk-virtual-scroll-viewport itemSize="50" class="list-container">
      <div *cdkVirtualFor="let user of users">
        {{ user.name }}
      </div>
    </cdk-virtual-scroll-viewport>
  `,
  styles: [`
    .list-container {
      height: 400px;
      width: 100%;
    }
  `]
})
export class UserListComponent {
  users: User[] = [];
}
```

### 4. Memory Management

#### Proper Subscription Management

```typescript
// user.component.ts
export class UserComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.userService.getUsers()
      .pipe(takeUntil(this.destroy$))
      .subscribe(users => {
        this.users = users;
      });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

#### Using Async Pipe

```typescript
// user-list.component.ts
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users$ | async">
      {{ user.name }}
    </div>
  `
})
export class UserListComponent {
  users$ = this.userService.getUsers();
}
```

### 5. Image Optimization

#### Lazy Loading Images

```typescript
// image.component.ts
@Component({
  selector: 'app-image',
  template: `
    <img [src]="imageUrl" 
         loading="lazy"
         [alt]="altText"
         (error)="handleImageError()">
  `
})
export class ImageComponent {
  @Input() imageUrl: string = '';
  @Input() altText: string = '';

  handleImageError() {
    this.imageUrl = 'assets/fallback-image.jpg';
  }
}
```

#### Using WebP Format

```typescript
// image.service.ts
@Injectable({
  providedIn: 'root'
})
export class ImageService {
  getOptimizedImageUrl(url: string): string {
    const supportsWebP = this.checkWebPSupport();
    return supportsWebP ? url.replace('.jpg', '.webp') : url;
  }

  private checkWebPSupport(): boolean {
    const canvas = document.createElement('canvas');
    if (canvas.toDataURL('image/webp').indexOf('data:image/webp') === 0) {
      return true;
    }
    return false;
  }
}
```

### 6. Bundle Size Optimization

#### Tree Shaking

```typescript
// Import only what you need
import { map, filter, take } from 'rxjs/operators';
// Instead of
import { operators } from 'rxjs';
```

#### Using Production Build

```bash
ng build --configuration production
```

#### Analyzing Bundle Size

```bash
ng build --stats-json
npx webpack-bundle-analyzer dist/stats.json
```

### 7. Caching Strategies

#### HTTP Caching

```typescript
// user.service.ts
@Injectable({
  providedIn: 'root'
})
export class UserService {
  private cache = new Map<number, User>();

  getUser(id: number): Observable<User> {
    if (this.cache.has(id)) {
      return of(this.cache.get(id)!);
    }

    return this.http.get<User>(`/api/users/${id}`).pipe(
      tap(user => this.cache.set(id, user))
    );
  }
}
```

#### Route Data Caching

```typescript
// user-resolver.service.ts
@Injectable({
  providedIn: 'root'
})
export class UserResolver implements Resolve<User> {
  private cache = new Map<number, User>();

  resolve(route: ActivatedRouteSnapshot): Observable<User> {
    const id = route.paramMap.get('id');
    if (this.cache.has(+id!)) {
      return of(this.cache.get(+id!)!);
    }

    return this.userService.getUser(+id!).pipe(
      tap(user => this.cache.set(+id!, user))
    );
  }
}
```

### 8. Performance Monitoring

#### Using Angular DevTools

```typescript
// Enable performance profiling
import { enableProdMode } from '@angular/core';

if (environment.production) {
  enableProdMode();
}
```

#### Custom Performance Monitoring

```typescript
// performance.service.ts
@Injectable({
  providedIn: 'root'
})
export class PerformanceService {
  private metrics = new Map<string, number[]>();

  startMeasure(label: string) {
    if (!this.metrics.has(label)) {
      this.metrics.set(label, []);
    }
    this.metrics.get(label)!.push(performance.now());
  }

  endMeasure(label: string) {
    const times = this.metrics.get(label);
    if (times && times.length > 0) {
      const startTime = times.pop()!;
      const duration = performance.now() - startTime;
      console.log(`${label}: ${duration}ms`);
    }
  }
}
```

### 9. Best Practices

#### 1. Component Design

```typescript
// Break down large components
@Component({
  selector: 'app-user-profile',
  template: `
    <app-user-header [user]="user"></app-user-header>
    <app-user-details [user]="user"></app-user-details>
    <app-user-activity [userId]="user.id"></app-user-activity>
  `
})
export class UserProfileComponent {
  @Input() user!: User;
}
```

#### 2. Template Optimization

```typescript
// Use pure pipes
@Pipe({
  name: 'userFullName',
  pure: true
})
export class UserFullNamePipe implements PipeTransform {
  transform(user: User): string {
    return `${user.firstName} ${user.lastName}`;
  }
}

// Use OnPush change detection
@Component({
  selector: 'app-user-card',
  template: `...`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
```

#### 3. Network Optimization

```typescript
// Implement request debouncing
searchUsers(term: string) {
  this.searchSubject.next(term);
}

private searchSubject = new Subject<string>();

constructor() {
  this.searchSubject.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term => this.userService.searchUsers(term))
  ).subscribe(users => {
    this.users = users;
  });
}
```

### Conclusion

Key points to remember:

1. **Change Detection**:
   - Use OnPush strategy
   - Implement trackBy functions
   - Avoid unnecessary change detection cycles

2. **Loading Optimization**:
   - Implement lazy loading
   - Use virtual scrolling for large lists
   - Optimize images and assets

3. **Memory Management**:
   - Properly manage subscriptions
   - Use async pipe
   - Implement proper cleanup

4. **Bundle Size**:
   - Enable production mode
   - Use tree shaking
   - Analyze bundle size

5. **Caching**:
   - Implement HTTP caching
   - Cache route data
   - Use service caching

6. **Monitoring**:
   - Use Angular DevTools
   - Implement custom performance monitoring
   - Track key metrics

Remember to:
- Profile your application
- Monitor performance metrics
- Implement optimizations based on bottlenecks
- Test performance in different scenarios
- Keep dependencies up to date
- Use appropriate tools for analysis 