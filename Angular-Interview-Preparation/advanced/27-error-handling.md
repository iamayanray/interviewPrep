# Advanced Angular Error Handling and Logging

## Question
What are the advanced error handling and logging techniques in Angular? Explain global error handling, custom error handlers, and logging services with examples.

## Answer

### Introduction to Advanced Angular Error Handling and Logging

Angular provides robust error handling and logging capabilities to help developers manage and debug application issues effectively. Here's a comprehensive guide to advanced error handling and logging techniques in Angular.

### 1. Advanced Global Error Handling

#### Custom Error Handler

```typescript
// custom-error-handler.ts
@Injectable()
export class CustomErrorHandler implements ErrorHandler {
  constructor(
    private logger: ErrorLoggingService,
    private notificationService: NotificationService
  ) {}

  handleError(error: any): void {
    // Log the error
    this.logger.logError(error);

    // Handle specific error types
    if (error instanceof HttpErrorResponse) {
      this.handleHttpError(error);
    } else if (error instanceof TypeError) {
      this.handleTypeError(error);
    } else {
      this.handleGenericError(error);
    }
  }

  private handleHttpError(error: HttpErrorResponse): void {
    switch (error.status) {
      case 401:
        this.notificationService.showError('Unauthorized access. Please login.');
        // Handle unauthorized access
        break;
      case 403:
        this.notificationService.showError('Access forbidden.');
        break;
      case 404:
        this.notificationService.showError('Resource not found.');
        break;
      case 500:
        this.notificationService.showError('Server error. Please try again later.');
        break;
      default:
        this.notificationService.showError('An error occurred. Please try again.');
    }
  }

  private handleTypeError(error: TypeError): void {
    this.notificationService.showError('Type error occurred. Please check your input.');
    console.error('Type Error:', error);
  }

  private handleGenericError(error: any): void {
    this.notificationService.showError('An unexpected error occurred.');
    console.error('Generic Error:', error);
  }
}

// Using in app.module.ts
@NgModule({
  providers: [
    {
      provide: ErrorHandler,
      useClass: CustomErrorHandler
    }
  ]
})
export class AppModule {}
```

### 2. Advanced HTTP Error Handling

#### HTTP Error Interceptor

```typescript
// http-error.interceptor.ts
@Injectable()
export class HttpErrorInterceptor implements HttpInterceptor {
  constructor(
    private notificationService: NotificationService,
    private authService: AuthService
  ) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    return next.handle(request).pipe(
      catchError(error => {
        if (error instanceof HttpErrorResponse) {
          this.handleHttpError(error, request);
        }
        return throwError(() => error);
      })
    );
  }

  private handleHttpError(
    error: HttpErrorResponse,
    request: HttpRequest<any>
  ): void {
    // Log error details
    console.error('HTTP Error:', {
      url: request.url,
      method: request.method,
      status: error.status,
      message: error.message,
      error: error.error
    });

    switch (error.status) {
      case 401:
        this.handleUnauthorized();
        break;
      case 403:
        this.handleForbidden();
        break;
      case 404:
        this.handleNotFound();
        break;
      case 500:
        this.handleServerError();
        break;
      default:
        this.handleGenericError();
    }
  }

  private handleUnauthorized(): void {
    this.notificationService.showError('Session expired. Please login again.');
    this.authService.logout();
  }

  private handleForbidden(): void {
    this.notificationService.showError('Access forbidden.');
  }

  private handleNotFound(): void {
    this.notificationService.showError('Resource not found.');
  }

  private handleServerError(): void {
    this.notificationService.showError('Server error. Please try again later.');
  }

  private handleGenericError(): void {
    this.notificationService.showError('An error occurred. Please try again.');
  }
}
```

### 3. Advanced Error Logging Service

```typescript
// error-logging.service.ts
@Injectable({ providedIn: 'root' })
export class ErrorLoggingService {
  private readonly MAX_RETRIES = 3;
  private readonly RETRY_DELAY = 5000; // 5 seconds
  private errorQueue: ErrorLog[] = [];
  private isProcessing = false;

  constructor(private http: HttpClient) {}

  logError(error: any): void {
    const errorLog: ErrorLog = {
      timestamp: new Date(),
      error: this.formatError(error),
      url: window.location.href,
      userAgent: navigator.userAgent
    };

    this.errorQueue.push(errorLog);
    this.processErrorQueue();
  }

  private formatError(error: any): string {
    if (error instanceof Error) {
      return error.stack || error.message;
    }
    return JSON.stringify(error);
  }

  private processErrorQueue(): void {
    if (this.isProcessing || this.errorQueue.length === 0) {
      return;
    }

    this.isProcessing = true;
    const batch = this.errorQueue.splice(0, 10); // Process 10 errors at a time

    this.sendErrorBatch(batch).subscribe({
      next: () => {
        this.isProcessing = false;
        if (this.errorQueue.length > 0) {
          this.processErrorQueue();
        }
      },
      error: () => {
        this.isProcessing = false;
        this.errorQueue.unshift(...batch); // Put failed batch back in queue
        this.retryFailedBatch(batch);
      }
    });
  }

  private sendErrorBatch(batch: ErrorLog[]): Observable<any> {
    return this.http.post('/api/error-logs', batch).pipe(
      retry(this.MAX_RETRIES),
      delay(this.RETRY_DELAY)
    );
  }

  private retryFailedBatch(batch: ErrorLog[]): void {
    setTimeout(() => {
      this.processErrorQueue();
    }, this.RETRY_DELAY);
  }
}

interface ErrorLog {
  timestamp: Date;
  error: string;
  url: string;
  userAgent: string;
}
```

### 4. Advanced Error Boundary Component

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
  @Input() errorMessage = 'An error occurred. Please try again.';
  hasError = false;
  error: any;

  constructor() {}

  ngOnInit(): void {
    // Subscribe to error events
    this.handleErrors();
  }

  private handleErrors(): void {
    // Handle component errors
    this.handleComponentErrors();
    // Handle service errors
    this.handleServiceErrors();
  }

  private handleComponentErrors(): void {
    // Implementation for handling component errors
  }

  private handleServiceErrors(): void {
    // Implementation for handling service errors
  }

  retry(): void {
    this.hasError = false;
    this.error = null;
    // Retry logic
  }
}

// Using in component
@Component({
  selector: 'app-feature',
  template: `
    <app-error-boundary>
      <app-child-component></app-child-component>
    </app-error-boundary>
  `
})
export class FeatureComponent {}
```

### 5. Advanced Error Recovery Strategies

```typescript
// error-recovery.service.ts
@Injectable({ providedIn: 'root' })
export class ErrorRecoveryService {
  constructor(
    private http: HttpClient,
    private notificationService: NotificationService
  ) {}

  recoverFromError(error: any): Observable<any> {
    if (error instanceof HttpErrorResponse) {
      return this.recoverFromHttpError(error);
    } else if (error instanceof NetworkError) {
      return this.recoverFromNetworkError(error);
    }
    return throwError(() => error);
  }

  private recoverFromHttpError(error: HttpErrorResponse): Observable<any> {
    switch (error.status) {
      case 401:
        return this.handleUnauthorizedError();
      case 503:
        return this.handleServiceUnavailableError();
      default:
        return this.handleGenericHttpError(error);
    }
  }

  private handleUnauthorizedError(): Observable<any> {
    return this.refreshToken().pipe(
      catchError(() => {
        this.notificationService.showError('Session expired. Please login.');
        return throwError(() => new Error('Unauthorized'));
      })
    );
  }

  private handleServiceUnavailableError(): Observable<any> {
    return timer(5000).pipe(
      switchMap(() => this.retryLastRequest())
    );
  }

  private handleGenericHttpError(error: HttpErrorResponse): Observable<any> {
    return this.notificationService.showError('An error occurred. Retrying...').pipe(
      delay(2000),
      switchMap(() => this.retryLastRequest())
    );
  }

  private recoverFromNetworkError(error: NetworkError): Observable<any> {
    return this.checkConnectivity().pipe(
      switchMap(isConnected => {
        if (isConnected) {
          return this.retryLastRequest();
        }
        return throwError(() => error);
      })
    );
  }

  private refreshToken(): Observable<any> {
    // Implementation for token refresh
    return of(null);
  }

  private retryLastRequest(): Observable<any> {
    // Implementation for retrying the last failed request
    return of(null);
  }

  private checkConnectivity(): Observable<boolean> {
    return of(navigator.onLine);
  }
}
```

### 6. Advanced Error Monitoring

```typescript
// error-monitoring.service.ts
@Injectable({ providedIn: 'root' })
export class ErrorMonitoringService {
  private errorCount = 0;
  private readonly ERROR_THRESHOLD = 10;
  private readonly TIME_WINDOW = 60000; // 1 minute
  private errorTimestamps: number[] = [];

  constructor(
    private notificationService: NotificationService,
    private analyticsService: AnalyticsService
  ) {}

  trackError(error: any): void {
    this.errorCount++;
    this.errorTimestamps.push(Date.now());

    // Clean up old timestamps
    this.cleanupOldTimestamps();

    // Check error rate
    this.checkErrorRate();

    // Send to analytics
    this.analyticsService.trackError(error);
  }

  private cleanupOldTimestamps(): void {
    const cutoff = Date.now() - this.TIME_WINDOW;
    this.errorTimestamps = this.errorTimestamps.filter(
      timestamp => timestamp > cutoff
    );
  }

  private checkErrorRate(): void {
    const errorRate = this.errorTimestamps.length / (this.TIME_WINDOW / 1000);
    
    if (errorRate > this.ERROR_THRESHOLD) {
      this.notificationService.showError(
        'High error rate detected. Please check the application.'
      );
    }
  }

  getErrorStats(): ErrorStats {
    return {
      totalErrors: this.errorCount,
      recentErrors: this.errorTimestamps.length,
      errorRate: this.errorTimestamps.length / (this.TIME_WINDOW / 1000)
    };
  }
}

interface ErrorStats {
  totalErrors: number;
  recentErrors: number;
  errorRate: number;
}
```

### Conclusion

Key points to remember:

1. **Global Error Handling**:
   - Implement custom error handler
   - Handle specific error types
   - Provide user-friendly messages
   - Log errors appropriately
   - Handle recovery strategies

2. **HTTP Error Handling**:
   - Use interceptors
   - Handle status codes
   - Implement retry logic
   - Manage authentication
   - Log error details

3. **Error Logging**:
   - Queue error logs
   - Batch processing
   - Retry mechanism
   - Error formatting
   - Error persistence

4. **Error Boundaries**:
   - Catch component errors
   - Provide fallback UI
   - Handle retry logic
   - Isolate errors
   - Maintain state

5. **Error Recovery**:
   - Implement strategies
   - Handle network errors
   - Manage authentication
   - Retry failed requests
   - Check connectivity

6. **Error Monitoring**:
   - Track error rates
   - Set thresholds
   - Monitor trends
   - Send analytics
   - Alert on issues

Remember to:
- Handle errors gracefully
- Provide user feedback
- Log errors properly
- Monitor error rates
- Implement recovery
- Test error scenarios
- Document error handling
- Regular maintenance 