# Advanced Angular Error Handling and Logging

## Question
What are the advanced error handling and logging techniques in Angular? Explain global error handling, custom error handlers, and logging services with examples.

## Answer

### Introduction to Advanced Angular Error Handling and Logging

Angular provides robust error handling and logging capabilities to help developers manage and debug application issues effectively. Here's a comprehensive guide to advanced error handling and logging techniques.

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
    this.notificationService.showError(
      this.getUserFriendlyMessage(error)
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

  private getUserFriendlyMessage(error: any): string {
    if (error instanceof HttpErrorResponse) {
      switch (error.status) {
        case 401:
          return 'Please log in to continue';
        case 403:
          return 'You do not have permission to perform this action';
        case 404:
          return 'The requested resource was not found';
        case 500:
          return 'An unexpected error occurred. Please try again later';
        default:
          return 'An error occurred. Please try again';
      }
    }
    return 'An unexpected error occurred. Please try again';
  }

  private handleUnauthorizedError(): void {
    // Handle unauthorized access
    this.logger.logError('Unauthorized access attempt');
  }

  private handleForbiddenError(): void {
    // Handle forbidden access
    this.logger.logError('Forbidden access attempt');
  }

  private handleNotFoundError(): void {
    // Handle not found error
    this.logger.logError('Resource not found');
  }

  private handleServerError(): void {
    // Handle server error
    this.logger.logError('Server error occurred');
  }

  private handleGenericHttpError(error: HttpErrorResponse): void {
    // Handle generic HTTP error
    this.logger.logError(`HTTP Error: ${error.status}`);
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
    private logger: ErrorLoggingService,
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
    this.logger.logError({
      url: request.url,
      method: request.method,
      status: error.status,
      message: error.message,
      error: error.error
    });

    // Handle specific status codes
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
        this.handleGenericError(error);
    }
  }

  private handleUnauthorizedError(): void {
    // Refresh token or redirect to login
    this.authService.refreshToken().pipe(
      catchError(() => {
        this.authService.logout();
        return throwError(() => new Error('Authentication failed'));
      })
    ).subscribe();
  }

  private handleForbiddenError(): void {
    this.notificationService.showError(
      'You do not have permission to perform this action'
    );
  }

  private handleNotFoundError(): void {
    this.notificationService.showError(
      'The requested resource was not found'
    );
  }

  private handleServerError(): void {
    this.notificationService.showError(
      'An unexpected error occurred. Please try again later'
    );
  }

  private handleGenericError(error: HttpErrorResponse): void {
    this.notificationService.showError(
      'An error occurred. Please try again'
    );
  }
}
```

### 3. Advanced Error Logging Service

```typescript
// error-logging.service.ts
@Injectable({ providedIn: 'root' })
export class ErrorLoggingService {
  private readonly MAX_QUEUE_SIZE = 100;
  private logQueue: ErrorLog[] = [];
  private isProcessing = false;
  private readonly RETRY_DELAY = 5000;
  private readonly MAX_RETRIES = 3;

  constructor(
    private http: HttpClient,
    private environment: Environment
  ) {}

  logError(error: any): void {
    const errorLog: ErrorLog = {
      timestamp: new Date(),
      message: error.message || 'Unknown error',
      stack: error.stack,
      type: error.name || 'Unknown',
      url: window.location.href,
      userAgent: navigator.userAgent,
      additionalInfo: this.getAdditionalInfo(error)
    };

    this.logQueue.push(errorLog);

    if (this.logQueue.length >= this.MAX_QUEUE_SIZE) {
      this.processLogQueue();
    } else if (!this.isProcessing) {
      this.scheduleProcessing();
    }
  }

  private getAdditionalInfo(error: any): any {
    return {
      ...error,
      environment: this.environment.name,
      version: this.environment.version,
      timestamp: new Date().toISOString()
    };
  }

  private scheduleProcessing(): void {
    setTimeout(() => this.processLogQueue(), 1000);
  }

  private processLogQueue(): void {
    if (this.isProcessing || this.logQueue.length === 0) {
      return;
    }

    this.isProcessing = true;
    const logs = this.logQueue.splice(0, this.MAX_QUEUE_SIZE);

    this.sendLogs(logs).pipe(
      finalize(() => {
        this.isProcessing = false;
        if (this.logQueue.length > 0) {
          this.scheduleProcessing();
        }
      })
    ).subscribe();
  }

  private sendLogs(logs: ErrorLog[]): Observable<void> {
    return this.http.post<void>('/api/logs', logs).pipe(
      retry({
        count: this.MAX_RETRIES,
        delay: this.RETRY_DELAY,
        resetOnSuccess: true
      }),
      catchError(error => {
        // Put failed logs back in queue
        this.logQueue.unshift(...logs);
        return throwError(() => error);
      })
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
  @Input() fallback: TemplateRef<any> | null = null;
  @Output() error = new EventEmitter<Error>();

  hasError = false;
  errorMessage = 'An unexpected error occurred';
  private errorHandler: (error: Error) => void;

  constructor(
    private logger: ErrorLoggingService,
    private changeDetector: ChangeDetectorRef
  ) {
    this.errorHandler = this.handleError.bind(this);
  }

  ngOnInit(): void {
    window.addEventListener('error', this.errorHandler);
  }

  ngOnDestroy(): void {
    window.removeEventListener('error', this.errorHandler);
  }

  private handleError(error: Error): void {
    this.hasError = true;
    this.errorMessage = error.message;
    this.error.emit(error);
    this.logger.logError(error);
    this.changeDetector.detectChanges();
  }

  retry(): void {
    this.hasError = false;
    this.errorMessage = 'An unexpected error occurred';
    this.changeDetector.detectChanges();
  }
}

// Using in component
@Component({
  selector: 'app-feature',
  template: `
    <app-error-boundary (error)="onError($event)">
      <app-complex-feature></app-complex-feature>
    </app-error-boundary>
  `
})
export class FeatureComponent {
  onError(error: Error): void {
    // Handle error at parent level
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
    } else {
      return new GenericErrorRecoveryStrategy(
        this.logger,
        this.notificationService
      );
    }
  }
}

// Error Recovery Strategies
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
        return this.handleGenericError(error, context);
    }
  }

  private handleUnauthorizedError(
    error: HttpErrorResponse,
    context: ErrorContext
  ): Observable<any> {
    // Implement retry logic with token refresh
    return throwError(() => error);
  }

  private handleForbiddenError(
    error: HttpErrorResponse,
    context: ErrorContext
  ): Observable<any> {
    this.notificationService.showError(
      'You do not have permission to perform this action'
    );
    return throwError(() => error);
  }

  private handleNotFoundError(
    error: HttpErrorResponse,
    context: ErrorContext
  ): Observable<any> {
    // Try to load from cache or show alternative content
    return throwError(() => error);
  }

  private handleServerError(
    error: HttpErrorResponse,
    context: ErrorContext
  ): Observable<any> {
    // Implement retry logic with exponential backoff
    return throwError(() => error);
  }

  private handleGenericError(
    error: HttpErrorResponse,
    context: ErrorContext
  ): Observable<any> {
    this.notificationService.showError(
      'An error occurred. Please try again'
    );
    return throwError(() => error);
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
  private readonly TIME_WINDOW = 5 * 60 * 1000; // 5 minutes
  private errorTimestamps: number[] = [];

  constructor(
    private logger: ErrorLoggingService,
    private notificationService: NotificationService
  ) {}

  trackError(error: any): void {
    this.errorCount++;
    this.errorTimestamps.push(Date.now());

    // Clean up old timestamps
    this.cleanupOldTimestamps();

    // Check if error rate exceeds threshold
    if (this.isErrorRateExceeded()) {
      this.notifyAdministrators();
    }

    // Log error
    this.logger.logError(error);
  }

  private cleanupOldTimestamps(): void {
    const cutoff = Date.now() - this.TIME_WINDOW;
    this.errorTimestamps = this.errorTimestamps.filter(
      timestamp => timestamp > cutoff
    );
  }

  private isErrorRateExceeded(): boolean {
    return this.errorTimestamps.length > this.ERROR_THRESHOLD;
  }

  private notifyAdministrators(): void {
    this.notificationService.showError(
      `High error rate detected: ${this.errorTimestamps.length} errors in the last 5 minutes`
    );
  }

  getErrorStats(): ErrorStats {
    return {
      totalErrors: this.errorCount,
      recentErrors: this.errorTimestamps.length,
      errorRate: this.calculateErrorRate()
    };
  }

  private calculateErrorRate(): number {
    return this.errorTimestamps.length / (this.TIME_WINDOW / 1000);
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
   - Implement retry logic
   - Show notifications
   - Log details

3. **Error Logging**:
   - Queue logs
   - Batch processing
   - Retry failed logs
   - Include context
   - Monitor queue size

4. **Error Boundaries**:
   - Catch component errors
   - Show fallback UI
   - Allow recovery
   - Log errors
   - Handle retry

5. **Error Recovery**:
   - Implement strategies
   - Handle specific cases
   - Retry operations
   - Use cached data
   - Show alternatives

6. **Error Monitoring**:
   - Track error rates
   - Set thresholds
   - Notify administrators
   - Calculate statistics
   - Monitor trends

Remember to:
- Log meaningful data
- Handle all error cases
- Implement recovery
- Monitor error rates
- Show user messages
- Clean up resources
- Test error handling
- Document strategies 