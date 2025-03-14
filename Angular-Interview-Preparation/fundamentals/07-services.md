# Angular Services

## Question
What are Angular services? How do you create and use them? What are their main purposes?

## Answer

### What are Angular Services?

Angular services are classes that are used to share data and functionality across components. They are singleton objects that get instantiated only once during the lifetime of an application. Services are a great way to organize and share code across your application.

### Main Purposes of Services

1. **Data Sharing**: Share data between components
2. **Business Logic**: Implement business logic that can be reused
3. **External Communication**: Handle HTTP requests and API calls
4. **State Management**: Manage application state
5. **Logging**: Implement logging functionality
6. **Authentication**: Handle user authentication and authorization

### Creating a Service

#### Basic Service

```typescript
// data.service.ts
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root' // Makes the service available application-wide
})
export class DataService {
  private data: any[] = [];

  constructor() {
    console.log('DataService created');
  }

  getData() {
    return this.data;
  }

  setData(newData: any[]) {
    this.data = newData;
  }

  addItem(item: any) {
    this.data.push(item);
  }

  removeItem(index: number) {
    this.data.splice(index, 1);
  }
}
```

#### HTTP Service

```typescript
// user.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { User } from './user.model';

@Injectable({
  providedIn: 'root'
})
export class UserService {
  private apiUrl = 'https://api.example.com/users';

  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }

  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`);
  }

  createUser(user: User): Observable<User> {
    return this.http.post<User>(this.apiUrl, user);
  }

  updateUser(id: number, user: User): Observable<User> {
    return this.http.put<User>(`${this.apiUrl}/${id}`, user);
  }

  deleteUser(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```

#### Authentication Service

```typescript
// auth.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

interface User {
  id: number;
  username: string;
  email: string;
}

interface AuthResponse {
  token: string;
  user: User;
}

@Injectable({
  providedIn: 'root'
})
export class AuthService {
  private currentUserSubject = new BehaviorSubject<User | null>(null);
  public currentUser$ = this.currentUserSubject.asObservable();

  constructor(private http: HttpClient) {
    // Check if user is logged in on service initialization
    const storedUser = localStorage.getItem('currentUser');
    if (storedUser) {
      this.currentUserSubject.next(JSON.parse(storedUser));
    }
  }

  login(username: string, password: string): Observable<AuthResponse> {
    return this.http.post<AuthResponse>('/api/auth/login', { username, password })
      .pipe(
        tap(response => {
          localStorage.setItem('token', response.token);
          localStorage.setItem('currentUser', JSON.stringify(response.user));
          this.currentUserSubject.next(response.user);
        })
      );
  }

  logout(): void {
    localStorage.removeItem('token');
    localStorage.removeItem('currentUser');
    this.currentUserSubject.next(null);
  }

  isAuthenticated(): boolean {
    return !!this.currentUserSubject.value;
  }

  getToken(): string | null {
    return localStorage.getItem('token');
  }
}
```

#### Logging Service

```typescript
// logging.service.ts
import { Injectable } from '@angular/core';

export enum LogLevel {
  Debug = 0,
  Info = 1,
  Warning = 2,
  Error = 3
}

@Injectable({
  providedIn: 'root'
})
export class LoggingService {
  private logLevel: LogLevel = LogLevel.Info;

  setLogLevel(level: LogLevel): void {
    this.logLevel = level;
  }

  debug(message: string, ...args: any[]): void {
    if (this.logLevel <= LogLevel.Debug) {
      console.debug(`[DEBUG] ${message}`, ...args);
    }
  }

  info(message: string, ...args: any[]): void {
    if (this.logLevel <= LogLevel.Info) {
      console.info(`[INFO] ${message}`, ...args);
    }
  }

  warn(message: string, ...args: any[]): void {
    if (this.logLevel <= LogLevel.Warning) {
      console.warn(`[WARN] ${message}`, ...args);
    }
  }

  error(message: string, ...args: any[]): void {
    if (this.logLevel <= LogLevel.Error) {
      console.error(`[ERROR] ${message}`, ...args);
    }
  }
}
```

### Using Services in Components

#### Basic Usage

```typescript
// user-list.component.ts
import { Component, OnInit } from '@angular/core';
import { UserService } from './user.service';
import { User } from './user.model';

@Component({
  selector: 'app-user-list',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error">{{ error }}</div>
    <div *ngIf="users">
      <div *ngFor="let user of users">
        {{ user.name }} - {{ user.email }}
      </div>
    </div>
  `
})
export class UserListComponent implements OnInit {
  users: User[] = [];
  loading = false;
  error: string | null = null;

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.loadUsers();
  }

  private loadUsers() {
    this.loading = true;
    this.userService.getUsers().subscribe({
      next: (users) => {
        this.users = users;
        this.loading = false;
      },
      error: (err) => {
        this.error = 'Failed to load users';
        this.loading = false;
      }
    });
  }
}
```

#### Using Multiple Services

```typescript
// dashboard.component.ts
import { Component, OnInit } from '@angular/core';
import { UserService } from './user.service';
import { LoggingService } from './logging.service';
import { AuthService } from './auth.service';

@Component({
  selector: 'app-dashboard',
  template: `
    <div *ngIf="isAuthenticated">
      <h2>Welcome, {{ currentUser?.username }}!</h2>
      <button (click)="logout()">Logout</button>
    </div>
    <div *ngIf="!isAuthenticated">
      <h2>Please log in</h2>
      <form (ngSubmit)="login()">
        <input [(ngModel)]="username" name="username" placeholder="Username">
        <input [(ngModel)]="password" name="password" type="password" placeholder="Password">
        <button type="submit">Login</button>
      </form>
    </div>
  `
})
export class DashboardComponent implements OnInit {
  username = '';
  password = '';
  currentUser: any = null;
  isAuthenticated = false;

  constructor(
    private userService: UserService,
    private authService: AuthService,
    private loggingService: LoggingService
  ) {}

  ngOnInit() {
    this.authService.currentUser$.subscribe(user => {
      this.currentUser = user;
      this.isAuthenticated = !!user;
    });
  }

  login() {
    this.authService.login(this.username, this.password).subscribe({
      next: () => {
        this.loggingService.info('User logged in successfully');
      },
      error: (error) => {
        this.loggingService.error('Login failed', error);
      }
    });
  }

  logout() {
    this.authService.logout();
    this.loggingService.info('User logged out');
  }
}
```

### Service with State Management

```typescript
// state.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

interface AppState {
  theme: 'light' | 'dark';
  language: string;
  notifications: boolean;
}

const initialState: AppState = {
  theme: 'light',
  language: 'en',
  notifications: true
};

@Injectable({
  providedIn: 'root'
})
export class StateService {
  private state = new BehaviorSubject<AppState>(initialState);
  state$ = this.state.asObservable();

  getState(): AppState {
    return this.state.value;
  }

  updateState(partialState: Partial<AppState>): void {
    this.state.next({
      ...this.state.value,
      ...partialState
    });
  }

  resetState(): void {
    this.state.next(initialState);
  }

  // Specific state updates
  setTheme(theme: 'light' | 'dark'): void {
    this.updateState({ theme });
  }

  setLanguage(language: string): void {
    this.updateState({ language });
  }

  toggleNotifications(): void {
    this.updateState({
      notifications: !this.state.value.notifications
    });
  }
}
```

### Service with Caching

```typescript
// caching.service.ts
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';

interface CacheItem<T> {
  data: T;
  timestamp: number;
}

@Injectable({
  providedIn: 'root'
})
export class CachingService {
  private cache = new Map<string, CacheItem<any>>();
  private defaultTTL = 5 * 60 * 1000; // 5 minutes

  set<T>(key: string, data: T, ttl: number = this.defaultTTL): void {
    this.cache.set(key, {
      data,
      timestamp: Date.now() + ttl
    });
  }

  get<T>(key: string): Observable<T> | null {
    const item = this.cache.get(key);
    
    if (!item) {
      return null;
    }

    if (Date.now() > item.timestamp) {
      this.cache.delete(key);
      return null;
    }

    return of(item.data);
  }

  clear(): void {
    this.cache.clear();
  }

  remove(key: string): void {
    this.cache.delete(key);
  }
}
```

### Best Practices

1. **Single Responsibility**:
   ```typescript
   // Good
   @Injectable()
   export class UserService {
     // Handle only user-related operations
   }

   // Bad
   @Injectable()
   export class DataService {
     // Don't mix user, product, and order operations
   }
   ```

2. **Dependency Injection**:
   ```typescript
   // Good
   constructor(private http: HttpClient) {}

   // Bad
   constructor() {
     this.http = new HttpClient();
   }
   ```

3. **Error Handling**:
   ```typescript
   // Good
   getData(): Observable<Data> {
     return this.http.get<Data>('/api/data').pipe(
       catchError(error => {
         this.loggingService.error('Failed to fetch data', error);
         return throwError(() => error);
       })
     );
   }
   ```

4. **Type Safety**:
   ```typescript
   // Good
   interface User {
     id: number;
     name: string;
     email: string;
   }

   // Bad
   getUsers(): Observable<any[]> {
     return this.http.get('/api/users');
   }
   ```

5. **Immutability**:
   ```typescript
   // Good
   updateUser(user: User): Observable<User> {
     return this.http.put<User>(`/api/users/${user.id}`, { ...user });
   }

   // Bad
   updateUser(user: User): Observable<User> {
     return this.http.put<User>(`/api/users/${user.id}`, user);
   }
   ```

### Testing Services

```typescript
// user.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { UserService } from './user.service';
import { User } from './user.model';

describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;
  const apiUrl = 'https://api.example.com/users';

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should be created', () => {
    expect(service).toBeTruthy();
  });

  it('should fetch users', () => {
    const mockUsers: User[] = [
      { id: 1, name: 'John', email: 'john@example.com' },
      { id: 2, name: 'Jane', email: 'jane@example.com' }
    ];

    service.getUsers().subscribe(users => {
      expect(users).toEqual(mockUsers);
    });

    const req = httpMock.expectOne(apiUrl);
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);
  });
});
```

### Conclusion

Angular services are a fundamental part of Angular applications, providing:
- A way to share code and functionality across components
- Centralized data management
- Business logic encapsulation
- External communication handling
- State management capabilities

By following best practices and using services effectively, you can create more maintainable, testable, and scalable Angular applications. Services help keep your components focused on presentation logic while handling complex business logic and data management separately. 