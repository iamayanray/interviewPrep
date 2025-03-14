# Advanced Angular Error Handling and Logging

## Question
What are the advanced error handling and logging techniques in Angular? Explain global error handling, custom error handlers, and logging services with examples.

## Answer

### Introduction to Advanced Angular Error Handling and Logging

Angular provides powerful error handling and logging capabilities to help developers manage and debug application issues effectively. Here's a comprehensive guide to advanced error handling and logging techniques.

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

    // Show user-friendly message
    this.showErrorMessage(error);
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

  private handleTypeError(error: TypeError): void {
    this.logger.logError({
      type: 'TypeError',
      message: error.message,
      stack: error.stack
    });
  }

  private handleGenericError(error: any): void {
    this.logger.logError({
      type: 'GenericError',
      message: error.message,
      stack: error.stack
    });
  }

  private showErrorMessage(error: any): void {
    const message = this.getUserFriendlyMessage(error);
    this.notificationService.showError(message);
  }

  private getUserFriendlyMessage(error: any): string {
    if (error instanceof HttpErrorResponse) {
      return this.getHttpErrorMessage(error);
    }
    return 'An unexpected error occurred. Please try again later.';
  }

  private getHttpErrorMessage(error: HttpErrorResponse): string {
    switch (error.status) {
      case 401:
        return 'Your session has expired. Please log in again.';
      case 403:
        return 'You do not have permission to perform this action.';
      case 404:
        return 'The requested resource was not found.';
      case 500:
        return 'A server error occurred. Please try again later.';
      default:
        return 'An error occurred while processing your request.';
    }
  }

  private handleUnauthorizedError(): void {
    // Handle unauthorized access
    this.notificationService.showError('Please log in to continue.');
    // Redirect to login page
  }

  private handleForbiddenError(): void {
    // Handle forbidden access
    this.notificationService.showError('Access denied.');
    // Redirect to home page
  }

  private handleNotFoundError(): void {
    // Handle not found error
    this.notificationService.showError('Resource not found.');
    // Redirect to 404 page
  }

  private handleServerError(): void {
    // Handle server error
    this.notificationService.showError('Server error occurred.');
    // Log to monitoring service
  }

  private handleGenericHttpError(error: HttpErrorResponse): void {
    // Handle other HTTP errors
    this.logger.logError({
      type: 'HttpError',
      status: error.status,
      message: error.message
    });
  }
}
```

### 2. Advanced HTTP Error Handling

#### HTTP Error Interceptor

```typescript
// http-error.interceptor.ts
@Injectable()
export class HttpErrorInterceptor implements HttpInterceptor {
  constructor(
    private errorHandler: CustomErrorHandler,
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
    switch (error.status) {
      case 401:
        this.handleUnauthorizedError(request);
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

  private handleUnauthorizedError(request: HttpRequest<any>): void {
    // Check if it's a refresh token request
    if (request.url.includes('/auth/refresh')) {
      this.authService.logout();
      return;
    }

    // Try to refresh token
    this.authService.refreshToken().pipe(
      catchError(() => {
        this.authService.logout();
        return throwError(() => new Error('Authentication failed'));
      })
    ).subscribe();
  }

  private handleForbiddenError(): void {
    this.errorHandler.handleError(
      new HttpErrorResponse({
        status: 403,
        statusText: 'Forbidden',
        error: 'Access denied'
      })
    );
  }

  private handleNotFoundError(): void {
    this.errorHandler.handleError(
      new HttpErrorResponse({
        status: 404,
        statusText: 'Not Found',
        error: 'Resource not found'
      })
    );
  }

  private handleServerError(): void {
    this.errorHandler.handleError(
      new HttpErrorResponse({
        status: 500,
        statusText: 'Internal Server Error',
        error: 'Server error occurred'
      })
    );
  }

  private handleGenericHttpError(error: HttpErrorResponse): void {
    this.errorHandler.handleError(error);
  }
}
```

### 3. Advanced Error Logging Service

```typescript
// error-logging.service.ts
@Injectable({ providedIn: 'root' })
export class ErrorLoggingService {
  private readonly LOG_QUEUE: ErrorLog[] = [];
  private readonly BATCH_SIZE = 10;
  private readonly MAX_RETRIES = 3;
  private readonly RETRY_DELAY = 5000; // 5 seconds
  private isProcessing = false;

  constructor(
    private http: HttpClient,
    private environment: Environment
  ) {}

  logError(error: any): void {
    const errorLog = this.createErrorLog(error);
    this.LOG_QUEUE.push(errorLog);
    this.processLogQueue();
  }

  private createErrorLog(error: any): ErrorLog {
    return {
      timestamp: new Date().toISOString(),
      type: this.getErrorType(error),
      message: error.message || 'Unknown error',
      stack: error.stack,
      url: window.location.href,
      userAgent: navigator.userAgent,
      environment: this.environment.name,
      version: this.environment.version
    };
  }

  private getErrorType(error: any): string {
    if (error instanceof HttpErrorResponse) {
      return 'HttpError';
    } else if (error instanceof TypeError) {
      return 'TypeError';
    } else if (error instanceof Error) {
      return 'Error';
    }
    return 'Unknown';
  }

  private processLogQueue(): void {
    if (this.isProcessing || this.LOG_QUEUE.length === 0) {
      return;
    }

    this.isProcessing = true;
    const batch = this.LOG_QUEUE.splice(0, this.BATCH_SIZE);

    this.sendLogs(batch).pipe(
      retryWhen(errors =>
        errors.pipe(
          delay(this.RETRY_DELAY),
          take(this.MAX_RETRIES)
        )
      ),
      catchError(error => {
        console.error('Failed to send logs:', error);
        // Put failed logs back in queue
        this.LOG_QUEUE.unshift(...batch);
        return of(null);
      }),
      finalize(() => {
        this.isProcessing = false;
        if (this.LOG_QUEUE.length > 0) {
          this.processLogQueue();
        }
      })
    ).subscribe();
  }

  private sendLogs(logs: ErrorLog[]): Observable<void> {
    return this.http.post<void>(
      `${this.environment.apiUrl}/logs/error`,
      logs
    );
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
  private errorSubscription: Subscription;

  constructor(
    private errorHandler: CustomErrorHandler,
    private logger: ErrorLoggingService
  ) {}

  ngOnInit(): void {
    this.errorSubscription = this.errorHandler.error$.subscribe(
      error => this.handleError(error)
    );
  }

  ngOnDestroy(): void {
    if (this.errorSubscription) {
      this.errorSubscription.unsubscribe();
    }
  }

  private handleError(error: Error): void {
    this.hasError = true;
    this.errorMessage = this.getErrorMessage(error);
    this.error.emit(error);
    this.logger.logError(error);
  }

  private getErrorMessage(error: Error): string {
    if (error instanceof HttpErrorResponse) {
      return this.getHttpErrorMessage(error);
    }
    return error.message || 'An unexpected error occurred';
  }

  private getHttpErrorMessage(error: HttpErrorResponse): string {
    switch (error.status) {
      case 401:
        return 'Your session has expired. Please log in again.';
      case 403:
        return 'You do not have permission to perform this action.';
      case 404:
        return 'The requested resource was not found.';
      case 500:
        return 'A server error occurred. Please try again later.';
      default:
        return 'An error occurred while processing your request.';
    }
  }

  retry(): void {
    this.hasError = false;
    this.errorMessage = 'An unexpected error occurred';
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
  onError(error: Error): void {
    // Handle error at component level
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
    private logger: ErrorLoggingService,
    private notificationService: NotificationService
  ) {}

  recover(error: any, context: ErrorContext): Observable<any> {
    const strategy = this.getRecoveryStrategy(error, context);
    return strategy.recover(error, context);
  }

  private getRecoveryStrategy(
    error: any,
    context: ErrorContext
  ): ErrorRecoveryStrategy {
    if (error instanceof HttpErrorResponse) {
      return new HttpErrorRecoveryStrategy(
        this.logger,
        this.notificationService
      );
    } else if (error instanceof NetworkError) {
      return new NetworkErrorRecoveryStrategy(
        this.logger,
        this.notificationService
      );
    }
    return new GenericErrorRecoveryStrategy(
      this.logger,
      this.notificationService
    );
  }
}

// Error recovery strategies
interface ErrorRecoveryStrategy {
  recover(error: any, context: ErrorContext): Observable<any>;
}

class HttpErrorRecoveryStrategy implements ErrorRecoveryStrategy {
  constructor(
    private logger: ErrorLoggingService,
    private notificationService: NotificationService
  ) {}

  recover(error: HttpErrorResponse, context: ErrorContext): Observable<any> {
    switch (error.status) {
      case 401:
        return this.handleUnauthorizedError(error, context);
      case 403:
        return this.handleForbiddenError(error, context);
      case 404:
        return this.handleNotFoundError(error, context);
      case 500:
        return this.handleServerError(error, context);
      default:
        return this.handleGenericHttpError(error, context);
    }
  }

  private handleUnauthorizedError(
    error: HttpErrorResponse,
    context: ErrorContext
  ): Observable<any> {
    // Implement retry with token refresh
    return throwError(() => error);
  }

  private handleForbiddenError(
    error: HttpErrorResponse,
    context: ErrorContext
  ): Observable<any> {
    this.notificationService.showError('Access denied');
    return throwError(() => error);
  }

  private handleNotFoundError(
    error: HttpErrorResponse,
    context: ErrorContext
  ): Observable<any> {
    this.notificationService.showError('Resource not found');
    return throwError(() => error);
  }

  private handleServerError(
    error: HttpErrorResponse,
    context: ErrorContext
  ): Observable<any> {
    // Implement retry with exponential backoff
    return throwError(() => error);
  }

  private handleGenericHttpError(
    error: HttpErrorResponse,
    context: ErrorContext
  ): Observable<any> {
    this.logger.logError(error);
    return throwError(() => error);
  }
}

class NetworkErrorRecoveryStrategy implements ErrorRecoveryStrategy {
  constructor(
    private logger: ErrorLoggingService,
    private notificationService: NotificationService
  ) {}

  recover(error: NetworkError, context: ErrorContext): Observable<any> {
    // Implement network error recovery
    return throwError(() => error);
  }
}

class GenericErrorRecoveryStrategy implements ErrorRecoveryStrategy {
  constructor(
    private logger: ErrorLoggingService,
    private notificationService: NotificationService
  ) {}

  recover(error: any, context: ErrorContext): Observable<any> {
    this.logger.logError(error);
    this.notificationService.showError('An unexpected error occurred');
    return throwError(() => error);
  }
}

// Using in component
@Component({
  selector: 'app-data',
  template: `
    <div *ngIf="data$ | async as data">
      {{ data | json }}
    </div>
  `
})
export class DataComponent {
  data$: Observable<any>;

  constructor(
    private dataService: DataService,
    private errorRecovery: ErrorRecoveryService
  ) {
    this.data$ = this.loadData();
  }

  private loadData(): Observable<any> {
    return this.dataService.getData().pipe(
      catchError(error => {
        return this.errorRecovery.recover(error, {
          component: 'DataComponent',
          action: 'loadData'
        });
      })
    );
  }
}
```

### 6. Advanced Error Monitoring

```typescript
// error-monitoring.service.ts
@Injectable({ providedIn: 'root' })
export class ErrorMonitoringService {
  private readonly ERROR_THRESHOLD = 10;
  private readonly TIME_WINDOW = 5 * 60 * 1000; // 5 minutes
  private errorCount = 0;
  private errorWindow: Error[] = [];

  constructor(
    private logger: ErrorLoggingService,
    private notificationService: NotificationService
  ) {}

  trackError(error: Error): void {
    this.addError(error);
    this.checkErrorThreshold();
  }

  private addError(error: Error): void {
    const now = Date.now();
    this.errorWindow = this.errorWindow.filter(
      e => now - e.timestamp < this.TIME_WINDOW
    );
    this.errorWindow.push({
      ...error,
      timestamp: now
    });
    this.errorCount = this.errorWindow.length;
  }

  private checkErrorThreshold(): void {
    if (this.errorCount >= this.ERROR_THRESHOLD) {
      this.notifyAdministrators();
    }
  }

  private notifyAdministrators(): void {
    const errorSummary = this.generateErrorSummary();
    this.notificationService.notifyAdministrators({
      type: 'ERROR_THRESHOLD_EXCEEDED',
      message: 'Error threshold exceeded',
      summary: errorSummary
    });
  }

  private generateErrorSummary(): ErrorSummary {
    return {
      totalErrors: this.errorCount,
      timeWindow: this.TIME_WINDOW,
      errors: this.errorWindow.map(error => ({
        type: error.type,
        message: error.message,
        timestamp: error.timestamp
      }))
    };
  }

  getErrorStats(): ErrorStats {
    return {
      totalErrors: this.errorCount,
      errorsByType: this.getErrorsByType(),
      recentErrors: this.getRecentErrors()
    };
  }

  private getErrorsByType(): Map<string, number> {
    const errorsByType = new Map<string, number>();
    this.errorWindow.forEach(error => {
      const count = errorsByType.get(error.type) || 0;
      errorsByType.set(error.type, count + 1);
    });
    return errorsByType;
  }

  private getRecentErrors(): Error[] {
    return [...this.errorWindow].sort(
      (a, b) => b.timestamp - a.timestamp
    );
  }
}

// Using in component
@Component({
  selector: 'app-monitoring',
  template: `
    <div *ngIf="errorStats$ | async as stats">
      <h3>Error Statistics</h3>
      <p>Total Errors: {{ stats.totalErrors }}</p>
      <div *ngFor="let [type, count] of stats.errorsByType | keyvalue">
        {{ type }}: {{ count }}
      </div>
      <div *ngFor="let error of stats.recentErrors">
        {{ error.message }}
      </div>
    </div>
  `
})
export class MonitoringComponent {
  errorStats$: Observable<ErrorStats>;

  constructor(private monitoring: ErrorMonitoringService) {
    this.errorStats$ = interval(60000).pipe(
      startWith(0),
      map(() => this.monitoring.getErrorStats())
    );
  }
}
```

### Conclusion

Key points to remember:

1. **Global Error Handling**:
   - Implement custom handler
   - Handle specific errors
   - Show user messages
   - Log errors
   - Manage recovery

2. **HTTP Error Handling**:
   - Use interceptors
   - Handle status codes
   - Retry requests
   - Refresh tokens
   - Show notifications

3. **Error Logging**:
   - Queue logs
   - Batch processing
   - Retry mechanism
   - Error details
   - Environment info

4. **Error Boundaries**:
   - Catch errors
   - Show fallback UI
   - Handle retry
   - Emit events
   - Manage state

5. **Error Recovery**:
   - Implement strategies
   - Handle specific cases
   - Retry operations
   - Show feedback
   - Log attempts

6. **Error Monitoring**:
   - Track errors
   - Set thresholds
   - Generate reports
   - Notify admins
   - Show statistics

Remember to:
- Handle all error cases
- Provide user feedback
- Log errors properly
- Monitor error rates
- Implement recovery
- Test error scenarios
- Document handling
- Regular maintenance 