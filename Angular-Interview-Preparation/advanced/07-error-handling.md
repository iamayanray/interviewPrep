# Advanced Angular Error Handling and Logging

## Question
What are the advanced error handling and logging techniques in Angular? Explain global error handling, custom error handlers, and logging services with examples.

## Answer

### Introduction to Advanced Angular Error Handling

Angular provides powerful error handling and logging capabilities for complex applications. Here's a comprehensive guide to advanced error handling and logging techniques.

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

  handleError(error: any): void {
    // Log error
    this.errorLoggingService.logError(error);

    // Show user-friendly message
    const userMessage = this.getUserFriendlyMessage(error);
    this.notificationService.showError(userMessage);

    // Handle specific error types
    if (error instanceof HttpErrorResponse) {
      this.handleHttpError(error);
    } else if (error instanceof TypeError) {
      this.handleTypeError(error);
    } else {
      this.handleGenericError(error);
    }
  }

  private getUserFriendlyMessage(error: any): string {
    if (error instanceof HttpErrorResponse) {
      switch (error.status) {
        case 404:
          return 'The requested resource was not found.';
        case 403:
          return 'You do not have permission to access this resource.';
        case 500:
          return 'An internal server error occurred. Please try again later.';
        default:
          return 'An unexpected error occurred. Please try again.';
      }
    }
    return 'An unexpected error occurred. Please try again.';
  }

  private handleHttpError(error: HttpErrorResponse): void {
    // Handle specific HTTP errors
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
    // Redirect to login
    this.router.navigate(['/login']);
  }

  private handleForbiddenError(): void {
    // Show access denied message
    this.notificationService.showError('Access denied');
  }

  private handleNotFoundError(): void {
    // Navigate to 404 page
    this.router.navigate(['/404']);
  }

  private handleServerError(): void {
    // Show server error message
    this.notificationService.showError('Server error occurred');
  }

  private handleGenericHttpError(error: HttpErrorResponse): void {
    // Log detailed error information
    this.errorLoggingService.logError({
      status: error.status,
      statusText: error.statusText,
      error: error.error,
      url: error.url,
      headers: error.headers
    });
  }

  private handleTypeError(error: TypeError): void {
    // Handle type errors
    this.errorLoggingService.logError({
      type: 'TypeError',
      message: error.message,
      stack: error.stack
    });
  }

  private handleGenericError(error: any): void {
    // Handle other types of errors
    this.errorLoggingService.logError({
      type: error.constructor.name,
      message: error.message,
      stack: error.stack
    });
  }
}

// Using in app.module.ts
@NgModule({
  providers: [
    { provide: ErrorHandler, useClass: CustomErrorHandler }
  ]
})
export class AppModule { }
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
      catchError(error => this.handleError(error, request))
    );
  }

  private handleError(error: HttpErrorResponse, request: HttpRequest<any>): Observable<HttpEvent<any>> {
    // Log error details
    this.errorLoggingService.logError({
      url: request.url,
      method: request.method,
      status: error.status,
      statusText: error.statusText,
      error: error.error,
      headers: error.headers
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
        return this.handleGenericError(error);
    }
  }

  private handleUnauthorizedError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    // Try to refresh token
    return this.authService.refreshToken().pipe(
      switchMap(() => {
        // Retry the original request
        const newRequest = this.addAuthHeader(error.request);
        return next.handle(newRequest);
      }),
      catchError(() => {
        // If refresh fails, logout user
        this.authService.logout();
        return throwError(() => error);
      })
    );
  }

  private handleForbiddenError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    this.notificationService.showError('Access denied');
    return throwError(() => error);
  }

  private handleNotFoundError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    this.notificationService.showError('Resource not found');
    return throwError(() => error);
  }

  private handleServerError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    this.notificationService.showError('Server error occurred');
    return throwError(() => error);
  }

  private handleGenericError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    this.notificationService.showError('An unexpected error occurred');
    return throwError(() => error);
  }

  private addAuthHeader(request: HttpRequest<any>): HttpRequest<any> {
    const token = this.authService.getToken();
    return request.clone({
      headers: request.headers.set('Authorization', `Bearer ${token}`)
    });
  }
}
```

### 3. Advanced Error Logging Service

```typescript
// error-logging.service.ts
@Injectable({ providedIn: 'root' })
export class ErrorLoggingService {
  private readonly ERROR_ENDPOINT = '/api/logs/errors';
  private readonly MAX_RETRIES = 3;
  private readonly RETRY_DELAY = 1000;

  constructor(
    private http: HttpClient,
    private environment: Environment
  ) {}

  logError(error: any): void {
    const errorLog = this.createErrorLog(error);
    
    // Log to console in development
    if (this.environment.development) {
      console.error('Error:', errorLog);
    }

    // Send to server with retry logic
    this.sendErrorToServer(errorLog).pipe(
      retryWhen(errors => errors.pipe(
        delay(this.RETRY_DELAY),
        take(this.MAX_RETRIES)
      )),
      catchError(err => {
        console.error('Failed to log error:', err);
        return of(null);
      })
    ).subscribe();
  }

  private createErrorLog(error: any): ErrorLog {
    return {
      timestamp: new Date().toISOString(),
      message: error.message || 'Unknown error',
      stack: error.stack,
      type: error.constructor.name,
      url: window.location.href,
      userAgent: navigator.userAgent,
      ...this.extractAdditionalInfo(error)
    };
  }

  private extractAdditionalInfo(error: any): any {
    if (error instanceof HttpErrorResponse) {
      return {
        status: error.status,
        statusText: error.statusText,
        url: error.url,
        headers: this.serializeHeaders(error.headers)
      };
    }
    return {};
  }

  private serializeHeaders(headers: HttpHeaders): any {
    const result: any = {};
    headers.keys().forEach(key => {
      result[key] = headers.get(key);
    });
    return result;
  }

  private sendErrorToServer(errorLog: ErrorLog): Observable<any> {
    return this.http.post(this.ERROR_ENDPOINT, errorLog);
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
        <button (click)="retry()">Retry</button>
      </div>
    </ng-template>
  `
})
export class ErrorBoundaryComponent implements OnInit, OnDestroy {
  @Input() errorMessage = 'An unexpected error occurred';
  @Output() error = new EventEmitter<any>();

  hasError = false;
  private errorSubscription: Subscription;

  constructor(
    private errorService: ErrorService,
    private changeDetectorRef: ChangeDetectorRef
  ) {}

  ngOnInit() {
    this.errorSubscription = this.errorService.errors$.subscribe(
      error => this.handleError(error)
    );
  }

  ngOnDestroy() {
    this.errorSubscription?.unsubscribe();
  }

  private handleError(error: any): void {
    this.hasError = true;
    this.error.emit(error);
    this.changeDetectorRef.detectChanges();
  }

  retry(): void {
    this.hasError = false;
    this.changeDetectorRef.detectChanges();
  }
}

// Using in component
@Component({
  selector: 'app-feature',
  template: `
    <app-error-boundary (error)="onError($event)">
      <app-feature-content></app-feature-content>
    </app-error-boundary>
  `
})
export class FeatureComponent {
  onError(error: any): void {
    console.error('Feature error:', error);
  }
}
```

### 5. Advanced Error Recovery Strategies

```typescript
// error-recovery.service.ts
@Injectable({ providedIn: 'root' })
export class ErrorRecoveryService {
  constructor(
    private errorLoggingService: ErrorLoggingService,
    private stateService: StateService
  ) {}

  recoverFromError(error: any): Observable<any> {
    // Log the error
    this.errorLoggingService.logError(error);

    // Try different recovery strategies
    return this.tryRecoveryStrategies(error).pipe(
      catchError(err => {
        // If all strategies fail, throw error
        return throwError(() => err);
      })
    );
  }

  private tryRecoveryStrategies(error: any): Observable<any> {
    const strategies = [
      this.tryCacheRecovery.bind(this),
      this.tryFallbackRecovery.bind(this),
      this.tryRetryRecovery.bind(this)
    ];

    return from(strategies).pipe(
      mergeMap(strategy => strategy(error)),
      catchError(() => of(null)),
      filter(result => result !== null),
      take(1)
    );
  }

  private tryCacheRecovery(error: any): Observable<any> {
    if (error instanceof HttpErrorResponse && error.status === 404) {
      return this.stateService.getFromCache(error.url).pipe(
        map(data => {
          if (data) {
            this.errorLoggingService.logError({
              type: 'CacheRecovery',
              message: 'Recovered from cache',
              url: error.url
            });
            return data;
          }
          throw error;
        })
      );
    }
    return throwError(() => error);
  }

  private tryFallbackRecovery(error: any): Observable<any> {
    if (error instanceof HttpErrorResponse && error.status === 500) {
      return this.stateService.getFallbackData(error.url).pipe(
        map(data => {
          if (data) {
            this.errorLoggingService.logError({
              type: 'FallbackRecovery',
              message: 'Recovered from fallback',
              url: error.url
            });
            return data;
          }
          throw error;
        })
      );
    }
    return throwError(() => error);
  }

  private tryRetryRecovery(error: any): Observable<any> {
    if (error instanceof HttpErrorResponse && error.status >= 500) {
      return timer(1000).pipe(
        switchMap(() => this.stateService.retryRequest(error.url)),
        map(data => {
          this.errorLoggingService.logError({
            type: 'RetryRecovery',
            message: 'Recovered after retry',
            url: error.url
          });
          return data;
        })
      );
    }
    return throwError(() => error);
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
  private errorWindow: number[] = [];

  constructor(
    private errorLoggingService: ErrorLoggingService,
    private notificationService: NotificationService
  ) {}

  trackError(error: any): void {
    this.updateErrorWindow();
    this.errorCount++;

    if (this.shouldAlert()) {
      this.sendAlert();
    }

    this.errorLoggingService.logError(error);
  }

  private updateErrorWindow(): void {
    const now = Date.now();
    this.errorWindow = this.errorWindow.filter(
      timestamp => now - timestamp < this.TIME_WINDOW
    );
    this.errorWindow.push(now);
  }

  private shouldAlert(): boolean {
    return this.errorWindow.length >= this.ERROR_THRESHOLD;
  }

  private sendAlert(): void {
    const alert = {
      type: 'ErrorThresholdExceeded',
      count: this.errorWindow.length,
      timeWindow: this.TIME_WINDOW,
      timestamp: new Date().toISOString()
    };

    this.errorLoggingService.logError(alert);
    this.notificationService.showError('High error rate detected');
  }
}
```

### Conclusion

Key points to remember:

1. **Global Error Handling**:
   - Implement custom error handler
   - Handle different error types
   - Show user-friendly messages
   - Log errors appropriately

2. **HTTP Error Handling**:
   - Use error interceptors
   - Handle specific status codes
   - Implement retry logic
   - Show appropriate messages

3. **Error Logging**:
   - Log to server
   - Include context
   - Implement retry logic
   - Handle logging failures

4. **Error Boundaries**:
   - Catch component errors
   - Provide fallback UI
   - Allow recovery
   - Emit error events

5. **Error Recovery**:
   - Implement recovery strategies
   - Use caching
   - Provide fallbacks
   - Handle retries

Remember to:
- Log errors appropriately
- Show user-friendly messages
- Implement recovery strategies
- Monitor error rates
- Handle edge cases
- Test error scenarios
- Maintain error logs
- Follow best practices 