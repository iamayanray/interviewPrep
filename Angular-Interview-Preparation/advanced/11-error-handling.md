# Advanced Angular Error Handling and Logging

## Question
What are the advanced error handling and logging techniques in Angular? Explain global error handling, custom error handlers, and logging services with examples.

## Answer

### Introduction to Advanced Angular Error Handling

Angular provides powerful error handling and logging capabilities to help developers manage and debug application issues effectively. Here's a comprehensive guide to advanced error handling and logging techniques.

### 1. Advanced Global Error Handling

#### Custom Error Handler

```typescript
// custom-error-handler.ts
@Injectable()
export class CustomErrorHandler implements ErrorHandler {
  constructor(
    private errorLoggingService: ErrorLoggingService,
    private notificationService: NotificationService
  ) {}

  handleError(error: Error): void {
    // Log the error
    this.errorLoggingService.logError(error);

    // Handle specific error types
    if (error instanceof HttpErrorResponse) {
      this.handleHttpError(error);
    } else if (error instanceof TypeError) {
      this.handleTypeError(error);
    } else {
      this.handleGenericError(error);
    }

    // Show user-friendly message
    this.notificationService.showError(
      'An unexpected error occurred. Please try again later.'
    );
  }

  private handleHttpError(error: HttpErrorResponse): void {
    switch (error.status) {
      case 401:
        this.handleUnauthorizedError();
        break;
      case 403:
        this.handleForbiddenError();
        break;
      case 404:
        this.handleNotFoundError();
        break;
      case 500:
        this.handleServerError();
        break;
      default:
        this.handleGenericHttpError(error);
    }
  }

  private handleUnauthorizedError(): void {
    this.notificationService.showError('Please log in to continue.');
    // Redirect to login
  }

  private handleForbiddenError(): void {
    this.notificationService.showError('You do not have permission to access this resource.');
    // Redirect to home
  }

  private handleNotFoundError(): void {
    this.notificationService.showError('The requested resource was not found.');
    // Redirect to 404 page
  }

  private handleServerError(): void {
    this.notificationService.showError('Server error. Please try again later.');
  }

  private handleTypeError(error: TypeError): void {
    this.errorLoggingService.logError({
      message: 'Type Error',
      error: error,
      stack: error.stack
    });
  }

  private handleGenericError(error: Error): void {
    this.errorLoggingService.logError({
      message: 'Generic Error',
      error: error,
      stack: error.stack
    });
  }

  private handleGenericHttpError(error: HttpErrorResponse): void {
    this.errorLoggingService.logError({
      message: `HTTP Error ${error.status}`,
      error: error,
      url: error.url
    });
  }
}

// Register in app.module.ts
@NgModule({
  providers: [
    { provide: ErrorHandler, useClass: CustomErrorHandler }
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
    private errorLoggingService: ErrorLoggingService,
    private notificationService: NotificationService,
    private authService: AuthService
  ) {}

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(request).pipe(
      catchError(error => this.handleError(error))
    );
  }

  private handleError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    // Log error details
    this.errorLoggingService.logError({
      message: `HTTP Error ${error.status}`,
      error: error,
      url: error.url,
      method: error.error?.method,
      timestamp: new Date().toISOString()
    });

    // Handle specific HTTP errors
    switch (error.status) {
      case 401:
        return this.handleUnauthorizedError(error);
      case 403:
        return this.handleForbiddenError(error);
      case 404:
        return this.handleNotFoundError(error);
      case 500:
        return this.handleServerError(error);
      default:
        return this.handleGenericHttpError(error);
    }
  }

  private handleUnauthorizedError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    // Try to refresh token
    return this.authService.refreshToken().pipe(
      switchMap(() => {
        // Retry the original request
        const newRequest = this.cloneRequest(error.request);
        return next.handle(newRequest);
      }),
      catchError(() => {
        this.notificationService.showError('Session expired. Please log in again.');
        this.authService.logout();
        return throwError(() => error);
      })
    );
  }

  private handleForbiddenError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    this.notificationService.showError('Access denied. You do not have permission.');
    return throwError(() => error);
  }

  private handleNotFoundError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    this.notificationService.showError('Resource not found.');
    return throwError(() => error);
  }

  private handleServerError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    this.notificationService.showError('Server error. Please try again later.');
    return throwError(() => error);
  }

  private handleGenericHttpError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    this.notificationService.showError('An error occurred. Please try again.');
    return throwError(() => error);
  }

  private cloneRequest(request: HttpRequest<any>): HttpRequest<any> {
    return request.clone({
      headers: request.headers.set('Authorization', `Bearer ${this.authService.getToken()}`)
    });
  }
}
```

### 3. Advanced Error Logging Service

```typescript
// error-logging.service.ts
@Injectable({ providedIn: 'root' })
export class ErrorLoggingService {
  private readonly MAX_RETRIES = 3;
  private readonly RETRY_DELAY = 1000; // 1 second
  private errorQueue: ErrorLog[] = [];
  private isProcessing = false;

  constructor(
    private http: HttpClient,
    private environment: Environment
  ) {}

  logError(error: ErrorLog): void {
    // Add error to queue
    this.errorQueue.push({
      ...error,
      timestamp: new Date().toISOString(),
      environment: this.environment.name,
      version: this.environment.version
    });

    // Process queue if not already processing
    if (!this.isProcessing) {
      this.processErrorQueue();
    }
  }

  private processErrorQueue(): void {
    if (this.errorQueue.length === 0) {
      this.isProcessing = false;
      return;
    }

    this.isProcessing = true;
    const error = this.errorQueue.shift();

    if (error) {
      this.sendErrorToServer(error).pipe(
        retryWhen(errors =>
          errors.pipe(
            delay(this.RETRY_DELAY),
            take(this.MAX_RETRIES)
          )
        ),
        catchError(err => {
          console.error('Failed to log error:', err);
          // Put error back in queue
          this.errorQueue.unshift(error);
          return of(null);
        })
      ).subscribe(() => {
        this.processErrorQueue();
      });
    }
  }

  private sendErrorToServer(error: ErrorLog): Observable<void> {
    return this.http.post<void>('/api/error-logs', error);
  }

  // Additional logging methods
  logWarning(message: string, data?: any): void {
    this.logError({
      level: 'warning',
      message,
      data,
      timestamp: new Date().toISOString()
    });
  }

  logInfo(message: string, data?: any): void {
    this.logError({
      level: 'info',
      message,
      data,
      timestamp: new Date().toISOString()
    });
  }

  logDebug(message: string, data?: any): void {
    if (this.environment.debug) {
      this.logError({
        level: 'debug',
        message,
        data,
        timestamp: new Date().toISOString()
      });
    }
  }
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
        <button (click)="retry()">Try Again</button>
      </div>
    </ng-template>
  `
})
export class ErrorBoundaryComponent implements OnInit, OnDestroy {
  @Input() fallbackComponent?: Type<any>;
  @Output() error = new EventEmitter<Error>();

  hasError = false;
  errorMessage = 'An unexpected error occurred';
  private errorHandler: (error: Error) => void;

  constructor(
    private errorLoggingService: ErrorLoggingService,
    private componentFactoryResolver: ComponentFactoryResolver,
    private viewContainerRef: ViewContainerRef
  ) {
    this.errorHandler = this.handleError.bind(this);
  }

  ngOnInit(): void {
    // Subscribe to error events
    window.addEventListener('error', this.errorHandler);
  }

  ngOnDestroy(): void {
    // Clean up error handler
    window.removeEventListener('error', this.errorHandler);
  }

  private handleError(error: Error): void {
    this.hasError = true;
    this.errorMessage = error.message;
    this.error.emit(error);

    // Log the error
    this.errorLoggingService.logError({
      message: 'Component Error',
      error: error,
      stack: error.stack
    });

    // Show fallback component if provided
    if (this.fallbackComponent) {
      this.showFallbackComponent();
    }
  }

  private showFallbackComponent(): void {
    const factory = this.componentFactoryResolver.resolveComponentFactory(
      this.fallbackComponent
    );
    this.viewContainerRef.clear();
    this.viewContainerRef.createComponent(factory);
  }

  retry(): void {
    this.hasError = false;
    this.errorMessage = 'An unexpected error occurred';
  }
}

// Using in template
@Component({
  selector: 'app-feature',
  template: `
    <app-error-boundary [fallbackComponent]="fallbackComponent">
      <app-complex-component></app-complex-component>
    </app-error-boundary>
  `
})
export class FeatureComponent {
  fallbackComponent = FallbackComponent;
}
```

### 5. Advanced Error Recovery Strategies

```typescript
// error-recovery.service.ts
@Injectable({ providedIn: 'root' })
export class ErrorRecoveryService {
  constructor(
    private errorLoggingService: ErrorLoggingService,
    private cacheService: CacheService
  ) {}

  recoverFromError(error: Error, context: ErrorContext): Observable<any> {
    return this.determineRecoveryStrategy(error, context).pipe(
      switchMap(strategy => this.executeRecoveryStrategy(strategy, context)),
      catchError(recoveryError => {
        this.errorLoggingService.logError({
          message: 'Recovery failed',
          error: recoveryError,
          originalError: error
        });
        return throwError(() => recoveryError);
      })
    );
  }

  private determineRecoveryStrategy(
    error: Error,
    context: ErrorContext
  ): Observable<RecoveryStrategy> {
    if (error instanceof HttpErrorResponse) {
      return this.determineHttpRecoveryStrategy(error, context);
    }
    return of(RecoveryStrategy.RETRY);
  }

  private determineHttpRecoveryStrategy(
    error: HttpErrorResponse,
    context: ErrorContext
  ): Observable<RecoveryStrategy> {
    switch (error.status) {
      case 401:
        return of(RecoveryStrategy.REAUTHENTICATE);
      case 503:
        return of(RecoveryStrategy.RETRY_WITH_BACKOFF);
      case 500:
        return this.checkCacheAvailability(context);
      default:
        return of(RecoveryStrategy.FAIL);
    }
  }

  private checkCacheAvailability(
    context: ErrorContext
  ): Observable<RecoveryStrategy> {
    return this.cacheService.hasData(context.cacheKey).pipe(
      map(hasData => hasData ? RecoveryStrategy.USE_CACHE : RecoveryStrategy.FAIL)
    );
  }

  private executeRecoveryStrategy(
    strategy: RecoveryStrategy,
    context: ErrorContext
  ): Observable<any> {
    switch (strategy) {
      case RecoveryStrategy.RETRY:
        return this.retryOperation(context);
      case RecoveryStrategy.RETRY_WITH_BACKOFF:
        return this.retryWithBackoff(context);
      case RecoveryStrategy.USE_CACHE:
        return this.useCache(context);
      case RecoveryStrategy.REAUTHENTICATE:
        return this.reauthenticateAndRetry(context);
      case RecoveryStrategy.FAIL:
        return throwError(() => new Error('Recovery failed'));
    }
  }

  private retryOperation(context: ErrorContext): Observable<any> {
    return timer(1000).pipe(
      switchMap(() => context.operation())
    );
  }

  private retryWithBackoff(context: ErrorContext): Observable<any> {
    return interval(1000).pipe(
      take(3),
      switchMap(attempt => 
        context.operation().pipe(
          catchError(() => 
            attempt === 2 ? throwError(() => new Error('Max retries reached')) : EMPTY
          )
        )
      )
    );
  }

  private useCache(context: ErrorContext): Observable<any> {
    return this.cacheService.getData(context.cacheKey);
  }

  private reauthenticateAndRetry(context: ErrorContext): Observable<any> {
    return this.authService.refreshToken().pipe(
      switchMap(() => context.operation())
    );
  }
}

// Using in component
@Component({
  selector: 'app-data-fetch',
  template: `
    <div *ngIf="data$ | async as data">
      {{ data | json }}
    </div>
  `
})
export class DataFetchComponent {
  data$ = this.fetchData();

  constructor(private errorRecoveryService: ErrorRecoveryService) {}

  private fetchData(): Observable<any> {
    return this.http.get('/api/data').pipe(
      catchError(error => 
        this.errorRecoveryService.recoverFromError(error, {
          operation: () => this.http.get('/api/data'),
          cacheKey: 'data'
        })
      )
    );
  }
}
```

### 6. Advanced Error Monitoring

```typescript
// error-monitoring.service.ts
@Injectable({ providedIn: 'root' })
export class ErrorMonitoringService {
  private readonly ERROR_THRESHOLD = 5;
  private readonly TIME_WINDOW = 60000; // 1 minute
  private errorCount = 0;
  private errorWindow: Error[] = [];

  constructor(
    private errorLoggingService: ErrorLoggingService,
    private notificationService: NotificationService
  ) {
    // Clean up old errors periodically
    interval(this.TIME_WINDOW).subscribe(() => {
      this.cleanupErrorWindow();
    });
  }

  trackError(error: Error): void {
    this.errorWindow.push(error);
    this.errorCount++;

    // Check if error threshold is exceeded
    if (this.errorCount >= this.ERROR_THRESHOLD) {
      this.handleErrorThresholdExceeded();
    }

    // Log error
    this.errorLoggingService.logError({
      message: 'Application Error',
      error: error,
      stack: error.stack,
      timestamp: new Date().toISOString()
    });
  }

  private handleErrorThresholdExceeded(): void {
    // Notify administrators
    this.notificationService.showError(
      'High error rate detected. Please check the application.'
    );

    // Log detailed error report
    this.errorLoggingService.logError({
      message: 'Error Threshold Exceeded',
      errorCount: this.errorCount,
      errors: this.errorWindow,
      timestamp: new Date().toISOString()
    });

    // Reset counters
    this.errorCount = 0;
    this.errorWindow = [];
  }

  private cleanupErrorWindow(): void {
    const now = new Date().getTime();
    this.errorWindow = this.errorWindow.filter(error => {
      const errorTime = new Date(error.timestamp).getTime();
      return now - errorTime < this.TIME_WINDOW;
    });
    this.errorCount = this.errorWindow.length;
  }

  getErrorStats(): ErrorStats {
    return {
      totalErrors: this.errorCount,
      recentErrors: this.errorWindow.length,
      timeWindow: this.TIME_WINDOW
    };
  }
}

// Using in component
@Component({
  selector: 'app-error-monitor',
  template: `
    <div *ngIf="errorStats$ | async as stats">
      <p>Total Errors: {{ stats.totalErrors }}</p>
      <p>Recent Errors: {{ stats.recentErrors }}</p>
    </div>
  `
})
export class ErrorMonitorComponent {
  errorStats$ = interval(5000).pipe(
    map(() => this.errorMonitoringService.getErrorStats())
  );

  constructor(private errorMonitoringService: ErrorMonitoringService) {}
}
```

### Conclusion

Key points to remember:

1. **Global Error Handling**:
   - Implement custom error handler
   - Handle specific error types
   - Show user-friendly messages
   - Log errors appropriately
   - Provide recovery options

2. **HTTP Error Handling**:
   - Use HTTP interceptors
   - Handle specific status codes
   - Implement retry logic
   - Manage authentication errors
   - Cache error responses

3. **Error Logging**:
   - Implement logging service
   - Queue error logs
   - Handle retries
   - Include context
   - Support different log levels

4. **Error Boundaries**:
   - Catch component errors
   - Show fallback UI
   - Provide retry options
   - Log error details
   - Handle cleanup

5. **Error Recovery**:
   - Implement recovery strategies
   - Use caching
   - Handle retries
   - Manage authentication
   - Provide fallbacks

Remember to:
- Log errors appropriately
- Handle errors gracefully
- Provide user feedback
- Implement recovery strategies
- Monitor error rates
- Clean up resources
- Test error scenarios
- Document error handling 