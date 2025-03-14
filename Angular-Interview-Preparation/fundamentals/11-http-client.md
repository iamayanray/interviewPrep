# Angular HTTP Client

## Question
How do you make HTTP requests in Angular? Explain the HttpClient module and how to handle responses, errors, and implement interceptors.

## Answer

### Introduction to Angular HttpClient

Angular's HttpClient is a built-in service that provides a simplified client HTTP API for Angular applications. It offers testability features, typed request and response objects, request and response interception, Observable APIs, and streamlined error handling.

### Setting Up HttpClient

To use HttpClient, you need to import the HttpClientModule in your application module:

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HttpClientModule } from '@angular/common/http';
import { AppComponent } from './app.component';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    HttpClientModule // Import HttpClientModule
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

For standalone components (Angular 14+):

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient } from '@angular/common/http';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient()
  ]
});
```

### Basic HTTP Requests

#### GET Request

```typescript
// data.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { User } from './user.model';

@Injectable({
  providedIn: 'root'
})
export class DataService {
  private apiUrl = 'https://api.example.com/users';

  constructor(private http: HttpClient) { }

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }

  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`);
  }
}
```

#### POST Request

```typescript
// data.service.ts
createUser(user: User): Observable<User> {
  return this.http.post<User>(this.apiUrl, user);
}
```

#### PUT Request

```typescript
// data.service.ts
updateUser(id: number, user: User): Observable<User> {
  return this.http.put<User>(`${this.apiUrl}/${id}`, user);
}
```

#### PATCH Request

```typescript
// data.service.ts
partialUpdateUser(id: number, partialUser: Partial<User>): Observable<User> {
  return this.http.patch<User>(`${this.apiUrl}/${id}`, partialUser);
}
```

#### DELETE Request

```typescript
// data.service.ts
deleteUser(id: number): Observable<void> {
  return this.http.delete<void>(`${this.apiUrl}/${id}`);
}
```

### Using HTTP Requests in Components

```typescript
// user-list.component.ts
import { Component, OnInit } from '@angular/core';
import { DataService } from './data.service';
import { User } from './user.model';

@Component({
  selector: 'app-user-list',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error">{{ error }}</div>
    <ul *ngIf="users">
      <li *ngFor="let user of users">{{ user.name }}</li>
    </ul>
  `
})
export class UserListComponent implements OnInit {
  users: User[] = [];
  loading = false;
  error: string | null = null;

  constructor(private dataService: DataService) { }

  ngOnInit(): void {
    this.loadUsers();
  }

  loadUsers(): void {
    this.loading = true;
    this.dataService.getUsers().subscribe({
      next: (users) => {
        this.users = users;
        this.loading = false;
      },
      error: (err) => {
        this.error = 'Failed to load users';
        this.loading = false;
        console.error('Error loading users', err);
      },
      complete: () => {
        console.log('Users request completed');
      }
    });
  }
}
```

### Request Configuration Options

HttpClient provides various options to configure requests:

```typescript
// data.service.ts
getUsers(page: number = 1, limit: number = 10): Observable<User[]> {
  const options = {
    params: {
      page: page.toString(),
      limit: limit.toString()
    },
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${this.authService.getToken()}`
    },
    observe: 'response' as const, // Get full response instead of just the body
    responseType: 'json' as const
  };

  return this.http.get<User[]>(this.apiUrl, options);
}
```

### Handling Responses

#### Type Safety with Interfaces

```typescript
// user.model.ts
export interface User {
  id: number;
  name: string;
  email: string;
  role: 'admin' | 'user';
}

// data.service.ts
getUserById(id: number): Observable<User> {
  return this.http.get<User>(`${this.apiUrl}/${id}`);
}
```

#### Full Response

```typescript
// data.service.ts
import { HttpResponse } from '@angular/common/http';

getUsers(): Observable<HttpResponse<User[]>> {
  return this.http.get<User[]>(this.apiUrl, { observe: 'response' });
}

// component.ts
this.dataService.getUsers().subscribe(response => {
  const users = response.body; // The response body
  const headers = response.headers; // Access to headers
  const status = response.status; // HTTP status code
});
```

#### Response Transformation

```typescript
// data.service.ts
import { map } from 'rxjs/operators';

getUserNames(): Observable<string[]> {
  return this.http.get<User[]>(this.apiUrl).pipe(
    map(users => users.map(user => user.name))
  );
}
```

### Error Handling

#### Basic Error Handling

```typescript
// component.ts
this.dataService.getUsers().subscribe({
  next: (users) => {
    this.users = users;
  },
  error: (error) => {
    this.errorMessage = 'Failed to load users';
    console.error('Error loading users', error);
  }
});
```

#### Global Error Handling

```typescript
// global-error-handler.service.ts
import { Injectable, ErrorHandler } from '@angular/core';
import { HttpErrorResponse } from '@angular/common/http';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  handleError(error: any): void {
    if (error instanceof HttpErrorResponse) {
      console.error('HTTP Error:', error);
      // Handle HTTP errors globally
    } else {
      console.error('Application Error:', error);
      // Handle other errors
    }
  }
}

// app.module.ts
import { ErrorHandler } from '@angular/core';
import { GlobalErrorHandler } from './global-error-handler.service';

@NgModule({
  providers: [
    { provide: ErrorHandler, useClass: GlobalErrorHandler }
  ]
})
export class AppModule { }
```

#### Error Handling with RxJS Operators

```typescript
// data.service.ts
import { Observable, throwError } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';

getUsers(): Observable<User[]> {
  return this.http.get<User[]>(this.apiUrl).pipe(
    retry(2), // Retry the request up to 2 times if it fails
    catchError(this.handleError)
  );
}

private handleError(error: HttpErrorResponse): Observable<never> {
  let errorMessage = 'An unknown error occurred';
  
  if (error.error instanceof ErrorEvent) {
    // Client-side error
    errorMessage = `Error: ${error.error.message}`;
  } else {
    // Server-side error
    errorMessage = `Error Code: ${error.status}\nMessage: ${error.message}`;
  }
  
  console.error(errorMessage);
  return throwError(() => new Error(errorMessage));
}
```

### HTTP Interceptors

Interceptors allow you to intercept and modify HTTP requests and responses globally.

#### Creating an Interceptor

```typescript
// auth.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpInterceptor
} from '@angular/common/http';
import { Observable } from 'rxjs';
import { AuthService } from './auth.service';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}

  intercept(request: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    // Get the auth token from the service
    const authToken = this.authService.getToken();

    // Clone the request and add the authorization header
    if (authToken) {
      const authReq = request.clone({
        headers: request.headers.set('Authorization', `Bearer ${authToken}`)
      });
      return next.handle(authReq);
    }

    // If no token, proceed with the original request
    return next.handle(request);
  }
}
```

#### Logging Interceptor

```typescript
// logging.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpInterceptor,
  HttpResponse
} from '@angular/common/http';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements HttpInterceptor {
  intercept(request: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    const startTime = Date.now();
    
    return next.handle(request).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          const endTime = Date.now();
          const duration = endTime - startTime;
          console.log(`Request to ${request.url} took ${duration}ms`);
        }
      })
    );
  }
}
```

#### Error Handling Interceptor

```typescript
// error.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpInterceptor,
  HttpErrorResponse
} from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  intercept(request: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    return next.handle(request).pipe(
      catchError((error: HttpErrorResponse) => {
        let errorMsg = '';
        
        if (error.error instanceof ErrorEvent) {
          // Client-side error
          errorMsg = `Error: ${error.error.message}`;
        } else {
          // Server-side error
          errorMsg = `Error Code: ${error.status},  Message: ${error.message}`;
          
          // Handle specific status codes
          if (error.status === 401) {
            // Redirect to login page or refresh token
          } else if (error.status === 404) {
            // Handle not found
          }
        }
        
        return throwError(() => new Error(errorMsg));
      })
    );
  }
}
```

#### Caching Interceptor

```typescript
// cache.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpInterceptor,
  HttpResponse
} from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { tap, shareReplay } from 'rxjs/operators';

@Injectable()
export class CacheInterceptor implements HttpInterceptor {
  private cache = new Map<string, HttpResponse<any>>();

  intercept(request: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    // Only cache GET requests
    if (request.method !== 'GET') {
      return next.handle(request);
    }

    // Check if we have a cached response for this URL
    const cachedResponse = this.cache.get(request.url);
    if (cachedResponse) {
      return of(cachedResponse.clone());
    }

    // If not cached, make the request and cache the response
    return next.handle(request).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          this.cache.set(request.url, event.clone());
        }
      }),
      shareReplay(1)
    );
  }
}
```

#### Registering Interceptors

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { HTTP_INTERCEPTORS, HttpClientModule } from '@angular/common/http';
import { AppComponent } from './app.component';
import { AuthInterceptor } from './auth.interceptor';
import { LoggingInterceptor } from './logging.interceptor';
import { ErrorInterceptor } from './error.interceptor';
import { CacheInterceptor } from './cache.interceptor';

@NgModule({
  declarations: [AppComponent],
  imports: [
    BrowserModule,
    HttpClientModule
  ],
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: LoggingInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: CacheInterceptor, multi: true }
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

For standalone components (Angular 14+):

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { AppComponent } from './app/app.component';
import { authInterceptor } from './app/auth.interceptor';
import { loggingInterceptor } from './app/logging.interceptor';
import { errorInterceptor } from './app/error.interceptor';
import { cacheInterceptor } from './app/cache.interceptor';

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(
      withInterceptors([
        authInterceptor,
        loggingInterceptor,
        errorInterceptor,
        cacheInterceptor
      ])
    )
  ]
});
```

### Advanced HTTP Features

#### Request Progress Events

```typescript
// file-upload.service.ts
import { HttpClient, HttpEvent, HttpEventType, HttpProgressEvent } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class FileUploadService {
  constructor(private http: HttpClient) {}

  uploadFile(file: File): Observable<number> {
    const formData = new FormData();
    formData.append('file', file);

    return this.http.post<any>('https://api.example.com/upload', formData, {
      reportProgress: true,
      observe: 'events'
    }).pipe(
      map(event => this.getUploadProgress(event))
    );
  }

  private getUploadProgress(event: HttpEvent<any>): number {
    switch (event.type) {
      case HttpEventType.UploadProgress:
        return Math.round((100 * event.loaded) / (event.total || 1));
      case HttpEventType.Response:
        return 100;
      default:
        return 0;
    }
  }
}
```

#### Cancelling Requests

```typescript
// search.service.ts
import { HttpClient } from '@angular/common/http';
import { Observable, Subject } from 'rxjs';
import { takeUntil } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class SearchService {
  private cancelSearch$ = new Subject<void>();

  constructor(private http: HttpClient) {}

  search(query: string): Observable<any[]> {
    // Cancel any ongoing search request
    this.cancelSearch$.next();

    return this.http.get<any[]>(`https://api.example.com/search?q=${query}`).pipe(
      takeUntil(this.cancelSearch$)
    );
  }

  cancelSearch(): void {
    this.cancelSearch$.next();
  }

  ngOnDestroy(): void {
    this.cancelSearch$.complete();
  }
}
```

#### Parallel Requests

```typescript
// dashboard.service.ts
import { HttpClient } from '@angular/common/http';
import { Observable, forkJoin } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class DashboardService {
  constructor(private http: HttpClient) {}

  getDashboardData(): Observable<[User[], Product[], Order[]]> {
    const users$ = this.http.get<User[]>('/api/users');
    const products$ = this.http.get<Product[]>('/api/products');
    const orders$ = this.http.get<Order[]>('/api/orders');

    return forkJoin([users$, products$, orders$]);
  }
}
```

#### Sequential Requests

```typescript
// user.service.ts
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { switchMap } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(private http: HttpClient) {}

  getUserWithPosts(userId: number): Observable<any> {
    return this.http.get<User>(`/api/users/${userId}`).pipe(
      switchMap(user => {
        return this.http.get<Post[]>(`/api/users/${userId}/posts`).pipe(
          map(posts => ({ user, posts }))
        );
      })
    );
  }
}
```

### Testing HTTP Requests

Angular provides the `HttpClientTestingModule` for testing HTTP requests:

```typescript
// data.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { DataService } from './data.service';
import { User } from './user.model';

describe('DataService', () => {
  let service: DataService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [DataService]
    });

    service = TestBed.inject(DataService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify(); // Verify that no unmatched requests are outstanding
  });

  it('should retrieve users', () => {
    const mockUsers: User[] = [
      { id: 1, name: 'John', email: 'john@example.com', role: 'user' },
      { id: 2, name: 'Jane', email: 'jane@example.com', role: 'admin' }
    ];

    service.getUsers().subscribe(users => {
      expect(users).toEqual(mockUsers);
    });

    const req = httpMock.expectOne('https://api.example.com/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers); // Respond with mock data
  });

  it('should handle errors', () => {
    service.getUsers().subscribe({
      next: () => fail('should have failed with 404 error'),
      error: (error) => {
        expect(error.status).toBe(404);
      }
    });

    const req = httpMock.expectOne('https://api.example.com/users');
    req.flush('Not Found', { status: 404, statusText: 'Not Found' });
  });
});
```

### Best Practices

1. **Use Type Safety**:
   ```typescript
   // Define interfaces for your API responses
   interface User {
     id: number;
     name: string;
     email: string;
   }

   // Use typed HTTP methods
   getUsers(): Observable<User[]> {
     return this.http.get<User[]>(this.apiUrl);
   }
   ```

2. **Centralize API Calls in Services**:
   ```typescript
   // Create dedicated services for API calls
   @Injectable({
     providedIn: 'root'
   })
   export class UserService {
     // All user-related API calls go here
   }
   ```

3. **Handle Errors Properly**:
   ```typescript
   // Use catchError to handle HTTP errors
   getUsers(): Observable<User[]> {
     return this.http.get<User[]>(this.apiUrl).pipe(
       catchError(this.handleError)
     );
   }
   ```

4. **Use Interceptors for Cross-Cutting Concerns**:
   ```typescript
   // Use interceptors for authentication, logging, etc.
   { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
   ```

5. **Unsubscribe from Observables**:
   ```typescript
   // Use takeUntil, async pipe, or store subscriptions to unsubscribe
   private destroy$ = new Subject<void>();

   ngOnInit() {
     this.dataService.getUsers().pipe(
       takeUntil(this.destroy$)
     ).subscribe(users => {
       this.users = users;
     });
   }

   ngOnDestroy() {
     this.destroy$.next();
     this.destroy$.complete();
   }
   ```

6. **Use Environment Variables for API URLs**:
   ```typescript
   // environment.ts
   export const environment = {
     production: false,
     apiUrl: 'https://api.dev.example.com'
   };

   // data.service.ts
   import { environment } from '../environments/environment';

   @Injectable({
     providedIn: 'root'
   })
   export class DataService {
     private apiUrl = `${environment.apiUrl}/users`;
   }
   ```

7. **Implement Retry Logic for Transient Failures**:
   ```typescript
   // Use retry or retryWhen for transient failures
   getUsers(): Observable<User[]> {
     return this.http.get<User[]>(this.apiUrl).pipe(
       retry(3), // Retry up to 3 times
       catchError(this.handleError)
     );
   }
   ```

### Conclusion

Angular's HttpClient provides a powerful and flexible way to make HTTP requests in your applications. By leveraging its features like typed responses, interceptors, and RxJS integration, you can build robust and maintainable applications that communicate effectively with backend services.

Understanding how to properly handle responses, manage errors, and implement interceptors is crucial for creating reliable Angular applications that provide a good user experience even when network issues occur. 