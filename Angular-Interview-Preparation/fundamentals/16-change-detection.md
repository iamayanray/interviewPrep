# Angular Change Detection and Performance Optimization

## Question
How does Angular's change detection work? How can you optimize performance in Angular applications?

## Answer

### Introduction to Change Detection

Change detection is the process by which Angular keeps the view in sync with the component's data. Understanding how it works is crucial for building performant Angular applications.

### How Change Detection Works

1. **Default Change Detection Strategy**:
   - Angular checks all components for changes after every event
   - Runs from top to bottom in the component tree
   - Checks all bindings in templates

```typescript
// Example of default change detection
@Component({
  selector: 'app-user',
  template: `
    <div>{{ user.name }}</div>
    <div>{{ user.email }}</div>
  `
})
export class UserComponent {
  user = { name: 'John', email: 'john@example.com' };
}
```

2. **OnPush Change Detection Strategy**:
   - Only checks components when:
     - Input properties change
     - Event handlers are triggered
     - Observable bound to template emits new value
     - Async pipe receives new value

```typescript
@Component({
  selector: 'app-user',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div>{{ user.name }}</div>
    <div>{{ user.email }}</div>
  `
})
export class UserComponent {
  @Input() user: User;
}
```

### Performance Optimization Techniques

1. **Using OnPush Change Detection**:

```typescript
// user-list.component.ts
@Component({
  selector: 'app-user-list',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div *ngFor="let user of users">
      <app-user [user]="user"></app-user>
    </div>
  `
})
export class UserListComponent {
  @Input() users: User[] = [];
}
```

2. **Using TrackBy Function**:

```typescript
// Without trackBy
@Component({
  template: `
    <div *ngFor="let user of users">
      {{ user.name }}
    </div>
  `
})
export class UserListComponent {
  users: User[] = [];
}

// With trackBy
@Component({
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

3. **Using Pure Pipes**:

```typescript
// Pure pipe
@Pipe({
  name: 'filterUsers',
  pure: true
})
export class FilterUsersPipe implements PipeTransform {
  transform(users: User[], searchTerm: string): User[] {
    return users.filter(user => 
      user.name.toLowerCase().includes(searchTerm.toLowerCase())
    );
  }
}

// Usage in template
@Component({
  template: `
    <div *ngFor="let user of users | filterUsers:searchTerm">
      {{ user.name }}
    </div>
  `
})
export class UserListComponent {
  users: User[] = [];
  searchTerm = '';
}
```

4. **Using Async Pipe**:

```typescript
@Component({
  template: `
    <div *ngIf="users$ | async as users">
      <div *ngFor="let user of users">
        {{ user.name }}
      </div>
    </div>
  `
})
export class UserListComponent {
  users$ = this.userService.getUsers();
}
```

5. **Lazy Loading**:

```typescript
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'users',
    loadChildren: () => import('./users/users.module').then(m => m.UsersModule)
  }
];
```

6. **Using Virtual Scrolling**:

```typescript
// app.module.ts
import { ScrollingModule } from '@angular/cdk/scrolling';

@NgModule({
  imports: [ScrollingModule]
})
export class AppModule { }

// user-list.component.ts
@Component({
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

7. **Using Web Workers**:

```typescript
// app.worker.ts
/// <reference lib="webworker" />

addEventListener('message', ({ data }) => {
  const response = heavyComputation(data);
  postMessage(response);
});

// app.component.ts
@Component({
  template: `
    <button (click)="startComputation()">Start</button>
    <div>{{ result }}</div>
  `
})
export class AppComponent {
  result: string;

  startComputation() {
    if (typeof Worker !== 'undefined') {
      const worker = new Worker('./app.worker.ts');
      worker.onmessage = ({ data }) => {
        this.result = data;
      };
      worker.postMessage('heavy data');
    }
  }
}
```

8. **Using Memoization**:

```typescript
// memoize.decorator.ts
export function Memoize() {
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const original = descriptor.value;
    const cache = new Map();

    descriptor.value = function (...args: any[]) {
      const key = JSON.stringify(args);
      if (cache.has(key)) {
        return cache.get(key);
      }
      const result = original.apply(this, args);
      cache.set(key, result);
      return result;
    };

    return descriptor;
  };
}

// Usage
export class UserService {
  @Memoize()
  getUserById(id: number): User {
    // Expensive operation
    return this.http.get<User>(`/api/users/${id}`);
  }
}
```

9. **Optimizing Images**:

```typescript
// image.component.ts
@Component({
  selector: 'app-image',
  template: `
    <img [src]="imageUrl" 
         loading="lazy"
         [alt]="alt"
         (error)="handleError()">
  `
})
export class ImageComponent {
  @Input() imageUrl: string;
  @Input() alt: string;
  fallbackUrl = 'assets/placeholder.png';

  handleError() {
    this.imageUrl = this.fallbackUrl;
  }
}
```

10. **Using ChangeDetectorRef**:

```typescript
@Component({
  template: `
    <div>{{ data }}</div>
  `
})
export class DataComponent implements OnInit {
  data: any;

  constructor(private cdr: ChangeDetectorRef) {}

  ngOnInit() {
    this.loadData();
  }

  loadData() {
    this.http.get('/api/data').subscribe(data => {
      this.data = data;
      this.cdr.detectChanges(); // Manually trigger change detection
    });
  }
}
```

### Best Practices for Performance

1. **Component Design**:
   - Keep components small and focused
   - Use OnPush change detection where possible
   - Avoid complex computations in templates

2. **Data Management**:
   - Use immutable data patterns
   - Implement proper trackBy functions
   - Use pure pipes for data transformations

3. **Template Optimization**:
   - Avoid complex expressions in templates
   - Use async pipe for observables
   - Implement proper error handling

4. **Network Optimization**:
   - Implement proper caching strategies
   - Use lazy loading for modules
   - Optimize API calls

5. **Memory Management**:
   - Properly unsubscribe from observables
   - Clean up event listeners
   - Use proper lifecycle hooks

### Common Performance Issues and Solutions

1. **Memory Leaks**:
```typescript
export class DataComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();

  ngOnInit() {
    this.dataService.getData()
      .pipe(takeUntil(this.destroy$))
      .subscribe(data => {
        this.data = data;
      });
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

2. **Expression Changed After Check**:
```typescript
@Component({
  template: `
    <div>{{ value }}</div>
  `
})
export class MyComponent implements AfterViewInit {
  value: string;

  constructor(private cdr: ChangeDetectorRef) {}

  ngAfterViewInit() {
    Promise.resolve().then(() => {
      this.value = 'Updated';
      this.cdr.detectChanges();
    });
  }
}
```

3. **Large Lists**:
```typescript
@Component({
  template: `
    <cdk-virtual-scroll-viewport [itemSize]="50" class="list-container">
      <div *cdkVirtualFor="let item of items; let i = index">
        {{ item.name }}
      </div>
    </cdk-virtual-scroll-viewport>
  `
})
export class ListComponent {
  items: any[] = [];
}
```

### Conclusion

Understanding Angular's change detection and implementing proper performance optimization techniques is crucial for building scalable applications. Key points to remember:

1. Use OnPush change detection where possible
2. Implement proper trackBy functions for lists
3. Use pure pipes for data transformations
4. Implement lazy loading for modules
5. Use virtual scrolling for large lists
6. Properly handle subscriptions and cleanup
7. Optimize images and assets
8. Use web workers for heavy computations
9. Implement proper caching strategies
10. Monitor and profile your application

By following these practices, you can build performant Angular applications that provide a great user experience. 