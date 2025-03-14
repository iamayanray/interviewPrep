# Angular Error Handling

## Question
How do you handle errors in Angular applications? Explain different error handling strategies and best practices.

## Answer

### Introduction to Angular Error Handling

Angular provides various mechanisms for handling errors at different levels of the application. Here's a comprehensive guide to implementing effective error handling.

### 1. Global Error Handling

#### Custom Error Handler

```typescript
// global-error-handler.ts
import { ErrorHandler, Injectable } from '@angular/core';
import { HttpErrorResponse } from '@angular/common/http';
import { NotificationService } from './notification.service';

@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(private notificationService: NotificationService) {}

  handleError(error: Error | HttpErrorResponse) {
    if (error instanceof HttpErrorResponse) {
      // Handle HTTP errors
      this.handleHttpError(error);
    } else {
      // Handle client-side errors
      this.handleClientError(error);
    }
  }

  private handleHttpError(error: HttpErrorResponse) {
    switch (error.status) {
      case 400:
        this.notificationService.showError('Bad Request');
        break;
      case 401:
        this.notificationService.showError('Unauthorized');
        break;
      case 403:
        this.notificationService.showError('Forbidden');
        break;
      case 404:
        this.notificationService.showError('Not Found');
        break;
      case 500:
        this.notificationService.showError('Internal Server Error');
        break;
      default:
        this.notificationService.showError('An error occurred');
    }
  }

  private handleClientError(error: Error) {
    console.error('Client Error:', error);
    this.notificationService.showError('An unexpected error occurred');
  }
}

// app.module.ts
@NgModule({
  providers: [
    { provide: ErrorHandler, useClass: GlobalErrorHandler }
  ]
})
export class AppModule { }
```

### 2. HTTP Error Handling

#### HTTP Interceptor for Error Handling

```typescript
// error.interceptor.ts
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler, HttpEvent, HttpErrorResponse } from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError, retry } from 'rxjs/operators';
import { Router } from '@angular/router';
import { AuthService } from './auth.service';

@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  constructor(
    private router: Router,
    private authService: AuthService
  ) {}

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(request).pipe(
      retry(1),
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          // Handle unauthorized access
          this.authService.logout();
          this.router.navigate(['/login']);
          return throwError(() => new Error('Unauthorized access'));
        }

        if (error.status === 403) {
          return throwError(() => new Error('Access forbidden'));
        }

        if (error.status === 404) {
          return throwError(() => new Error('Resource not found'));
        }

        if (error.status === 500) {
          return throwError(() => new Error('Server error'));
        }

        return throwError(() => new Error('An error occurred'));
      })
    );
  }
}

// app.module.ts
@NgModule({
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true }
  ]
})
export class AppModule { }
```

### 3. Component-Level Error Handling

#### Error Boundary Component

```typescript
// error-boundary.component.ts
@Component({
  selector: 'app-error-boundary',
  template: `
    <ng-container *ngIf="!hasError; else errorTemplate">
      <ng-content></ng-content>
    </ng-container>
    <ng-template #errorTemplate>
      <div class="error-container">
        <h2>Something went wrong</h2>
        <p>{{ errorMessage }}</p>
        <button (click)="retry()">Retry</button>
      </div>
    </ng-template>
  `
})
export class ErrorBoundaryComponent implements OnInit {
  @Input() errorMessage = 'An error occurred';
  hasError = false;

  retry() {
    this.hasError = false;
  }

  handleError(error: any) {
    this.hasError = true;
    console.error('Error caught by boundary:', error);
  }
}

// Using the error boundary
@Component({
  selector: 'app-user-profile',
  template: `
    <app-error-boundary>
      <app-user-details [user]="user"></app-user-details>
    </app-error-boundary>
  `
})
export class UserProfileComponent {
  @Input() user!: User;
}
```

### 4. Form Error Handling

#### Custom Form Validator

```typescript
// custom-validator.ts
export function passwordValidator(): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const password = control.value;
    if (!password) return null;

    const hasUpperCase = /[A-Z]+/.test(password);
    const hasLowerCase = /[a-z]+/.test(password);
    const hasNumeric = /[0-9]+/.test(password);
    const hasSpecialChar = /[!@#$%^&*()]+/.test(password);

    const valid = hasUpperCase && hasLowerCase && hasNumeric && hasSpecialChar;

    return !valid ? {
      passwordStrength: {
        hasUpperCase,
        hasLowerCase,
        hasNumeric,
        hasSpecialChar
      }
    } : null;
  };
}

// Using the validator
@Component({
  selector: 'app-registration',
  template: `
    <form [formGroup]="registrationForm" (ngSubmit)="onSubmit()">
      <input formControlName="password" type="password">
      <div *ngIf="registrationForm.get('password')?.errors?.['passwordStrength'] as errors">
        <p *ngIf="!errors.hasUpperCase">Password must contain uppercase letters</p>
        <p *ngIf="!errors.hasLowerCase">Password must contain lowercase letters</p>
        <p *ngIf="!errors.hasNumeric">Password must contain numbers</p>
        <p *ngIf="!errors.hasSpecialChar">Password must contain special characters</p>
      </div>
      <button type="submit">Register</button>
    </form>
  `
})
export class RegistrationComponent {
  registrationForm = this.fb.group({
    password: ['', [Validators.required, passwordValidator()]]
  });

  constructor(private fb: FormBuilder) {}
}
```

### 5. RxJS Error Handling

#### Error Handling in Observables

```typescript
// user.service.ts
@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users').pipe(
      catchError(error => {
        if (error instanceof HttpErrorResponse) {
          // Handle specific HTTP errors
          switch (error.status) {
            case 404:
              return throwError(() => new Error('Users not found'));
            case 500:
              return throwError(() => new Error('Server error'));
            default:
              return throwError(() => new Error('Failed to fetch users'));
          }
        }
        return throwError(() => error);
      }),
      retry(3), // Retry failed requests
      timeout(5000) // Timeout after 5 seconds
    );
  }
}

// Using in component
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngIf="error">{{ error }}</div>
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="users">
      <div *ngFor="let user of users">{{ user.name }}</div>
    </div>
  `
})
export class UserListComponent implements OnInit {
  users: User[] = [];
  error: string | null = null;
  loading = false;

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.loadUsers();
  }

  loadUsers() {
    this.loading = true;
    this.error = null;

    this.userService.getUsers().subscribe({
      next: (users) => {
        this.users = users;
        this.loading = false;
      },
      error: (error) => {
        this.error = error.message;
        this.loading = false;
      }
    });
  }
}
```

### 6. Error Logging Service

```typescript
// error-logging.service.ts
@Injectable({
  providedIn: 'root'
})
export class ErrorLoggingService {
  constructor(private http: HttpClient) {}

  logError(error: Error | HttpErrorResponse) {
    const errorLog = {
      timestamp: new Date().toISOString(),
      message: error.message,
      stack: error instanceof Error ? error.stack : null,
      url: error instanceof HttpErrorResponse ? error.url : null,
      status: error instanceof HttpErrorResponse ? error.status : null
    };

    // Send to error logging service
    this.http.post('/api/error-logs', errorLog).subscribe({
      error: (loggingError) => {
        console.error('Failed to log error:', loggingError);
      }
    });
  }
}
```

### 7. Best Practices

#### 1. Error Types

```typescript
// custom-error.ts
export class AppError extends Error {
  constructor(
    message: string,
    public code: string,
    public severity: 'low' | 'medium' | 'high'
  ) {
    super(message);
    this.name = 'AppError';
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 'VALIDATION_ERROR', 'low');
  }
}

export class AuthenticationError extends AppError {
  constructor(message: string) {
    super(message, 'AUTH_ERROR', 'high');
  }
}
```

#### 2. Error Recovery

```typescript
// error-recovery.service.ts
@Injectable({
  providedIn: 'root'
})
export class ErrorRecoveryService {
  private retryAttempts = new Map<string, number>();
  private maxRetries = 3;

  shouldRetry(error: Error, operation: string): boolean {
    const attempts = this.retryAttempts.get(operation) || 0;
    if (attempts >= this.maxRetries) {
      return false;
    }

    this.retryAttempts.set(operation, attempts + 1);
    return true;
  }

  resetRetryCount(operation: string) {
    this.retryAttempts.delete(operation);
  }
}
```

### Conclusion

Key points to remember:

1. **Error Handling Levels**:
   - Global error handling
   - HTTP error handling
   - Component-level error handling
   - Form validation errors
   - RxJS error handling

2. **Error Handling Tools**:
   - ErrorHandler
   - HTTP Interceptors
   - Error Boundaries
   - Custom Validators
   - Error Logging Service

3. **Best Practices**:
   - Use appropriate error types
   - Implement proper error recovery
   - Log errors appropriately
   - Provide user-friendly error messages
   - Handle errors at the right level

4. **Error Prevention**:
   - Input validation
   - Type checking
   - Null checking
   - Error boundaries
   - Proper error handling in async operations

Remember to:
- Handle errors gracefully
- Provide meaningful error messages
- Log errors for debugging
- Implement proper error recovery
- Use appropriate error handling strategies
- Test error scenarios
- Monitor error rates
- Keep error handling consistent 