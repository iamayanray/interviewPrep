# Angular Security Best Practices

## Question
What are the security best practices in Angular? How do you protect your Angular application from common security vulnerabilities?

## Answer

### Introduction to Angular Security

Angular provides built-in security features and best practices to help protect your application from common web vulnerabilities. Understanding and implementing these security measures is crucial for building secure applications.

### 1. Cross-Site Scripting (XSS) Protection

Angular automatically escapes values in templates to prevent XSS attacks.

```typescript
// Safe (Angular automatically escapes)
@Component({
  template: `
    <div>{{ userInput }}</div>
  `
})
export class SafeComponent {
  userInput = '<script>alert("xss")</script>';
}

// Unsafe (avoid using innerHTML)
@Component({
  template: `
    <div [innerHTML]="userInput"></div>  // Dangerous!
  `
})
export class UnsafeComponent {
  userInput = '<script>alert("xss")</script>';
}

// Safe way to use innerHTML with DomSanitizer
@Component({
  template: `
    <div [innerHTML]="safeHtml"></div>
  `
})
export class SafeHtmlComponent {
  userInput = '<script>alert("xss")</script>';
  safeHtml: SafeHtml;

  constructor(private sanitizer: DomSanitizer) {
    this.safeHtml = this.sanitizer.bypassSecurityTrustHtml(this.userInput);
  }
}
```

### 2. Cross-Site Request Forgery (CSRF) Protection

Angular's HttpClient includes CSRF protection:

```typescript
// app.module.ts
import { HttpClientModule, HttpClientXsrfModule } from '@angular/common/http';

@NgModule({
  imports: [
    HttpClientModule,
    HttpClientXsrfModule.withOptions({
      cookieName: 'XSRF-TOKEN',
      headerName: 'X-XSRF-TOKEN'
    })
  ]
})
export class AppModule { }

// data.service.ts
@Injectable({
  providedIn: 'root'
})
export class DataService {
  constructor(private http: HttpClient) {}

  postData(data: any) {
    return this.http.post('/api/data', data);
  }
}
```

### 3. Content Security Policy (CSP)

Implement CSP headers in your server configuration:

```typescript
// server.ts (Node.js/Express example)
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.example.com"]
    }
  }
}));
```

### 4. Authentication and Authorization

```typescript
// auth.guard.ts
@Injectable({
  providedIn: 'root'
})
export class AuthGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(): boolean {
    if (!this.authService.isAuthenticated()) {
      this.router.navigate(['/login']);
      return false;
    }
    return true;
  }
}

// auth.service.ts
@Injectable({
  providedIn: 'root'
})
export class AuthService {
  private token: string | null = null;

  login(credentials: { username: string, password: string }) {
    return this.http.post<{ token: string }>('/api/auth/login', credentials)
      .pipe(
        tap(response => {
          this.token = response.token;
          localStorage.setItem('token', response.token);
        })
      );
  }

  logout() {
    this.token = null;
    localStorage.removeItem('token');
  }

  isAuthenticated(): boolean {
    return !!this.token;
  }

  getToken(): string | null {
    return this.token;
  }
}

// http.interceptor.ts
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private authService: AuthService) {}

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getToken();
    
    if (token) {
      request = request.clone({
        setHeaders: {
          Authorization: `Bearer ${token}`
        }
      });
    }
    
    return next.handle(request);
  }
}
```

### 5. Route Protection

```typescript
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [AuthGuard],
    canActivateChild: [AdminGuard],
    children: [
      { path: 'users', component: UserManagementComponent },
      { path: 'settings', component: SettingsComponent }
    ]
  }
];
```

### 6. Secure HTTP Headers

```typescript
// server.ts (Node.js/Express example)
app.use(helmet({
  xssFilter: true,
  noSniff: true,
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  frameguard: { action: 'deny' },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  }
}));
```

### 7. Input Validation

```typescript
// user-form.component.ts
@Component({
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <input formControlName="email" type="email">
      <input formControlName="password" type="password">
      <button type="submit">Submit</button>
    </form>
  `
})
export class UserFormComponent {
  userForm = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [
      Validators.required,
      Validators.minLength(8),
      Validators.pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)[a-zA-Z\d]{8,}$/)
    ]]
  });

  constructor(private fb: FormBuilder) {}

  onSubmit() {
    if (this.userForm.valid) {
      // Process form data
    }
  }
}
```

### 8. Secure File Upload

```typescript
// file-upload.component.ts
@Component({
  template: `
    <input type="file" (change)="onFileSelected($event)">
    <button (click)="uploadFile()">Upload</button>
  `
})
export class FileUploadComponent {
  private selectedFile: File | null = null;
  private readonly MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
  private readonly ALLOWED_TYPES = ['image/jpeg', 'image/png', 'application/pdf'];

  onFileSelected(event: Event) {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (file) {
      if (file.size > this.MAX_FILE_SIZE) {
        alert('File size exceeds 5MB limit');
        return;
      }
      if (!this.ALLOWED_TYPES.includes(file.type)) {
        alert('Invalid file type');
        return;
      }
      this.selectedFile = file;
    }
  }

  uploadFile() {
    if (!this.selectedFile) return;

    const formData = new FormData();
    formData.append('file', this.selectedFile);

    this.http.post('/api/upload', formData)
      .pipe(
        catchError(error => {
          console.error('Upload failed:', error);
          return throwError(() => error);
        })
      )
      .subscribe(response => {
        console.log('Upload successful:', response);
      });
  }
}
```

### 9. Error Handling and Logging

```typescript
// error-handler.ts
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(
    private errorService: ErrorService,
    private logger: LoggerService
  ) {}

  handleError(error: Error) {
    this.logger.error('An error occurred:', error);
    
    // Don't expose sensitive information to users
    const userMessage = 'An unexpected error occurred. Please try again later.';
    
    this.errorService.showError(userMessage);
  }
}

// error.service.ts
@Injectable({
  providedIn: 'root'
})
export class ErrorService {
  private errorSubject = new Subject<string>();
  error$ = this.errorSubject.asObservable();

  showError(message: string) {
    this.errorSubject.next(message);
  }
}
```

### 10. Secure Storage

```typescript
// storage.service.ts
@Injectable({
  providedIn: 'root'
})
export class StorageService {
  constructor() {}

  setSecureItem(key: string, value: string) {
    // Use sessionStorage for sensitive data that shouldn't persist
    sessionStorage.setItem(key, this.encrypt(value));
  }

  getSecureItem(key: string): string | null {
    const value = sessionStorage.getItem(key);
    return value ? this.decrypt(value) : null;
  }

  private encrypt(value: string): string {
    // Implement encryption (e.g., using CryptoJS)
    return btoa(value); // Simple example, use proper encryption in production
  }

  private decrypt(value: string): string {
    // Implement decryption
    return atob(value); // Simple example, use proper decryption in production
  }
}
```

### 11. Environment Configuration

```typescript
// environment.ts
export const environment = {
  production: false,
  apiUrl: 'https://api.example.com',
  allowedOrigins: ['http://localhost:4200'],
  security: {
    tokenExpiration: 3600,
    refreshTokenExpiration: 86400,
    maxLoginAttempts: 5
  }
};

// environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.production.com',
  allowedOrigins: ['https://app.production.com'],
  security: {
    tokenExpiration: 1800,
    refreshTokenExpiration: 43200,
    maxLoginAttempts: 3
  }
};
```

### Best Practices Summary

1. **Input Validation**:
   - Validate all user inputs
   - Use Angular's built-in form validation
   - Implement custom validators when needed

2. **Output Encoding**:
   - Use Angular's built-in template escaping
   - Avoid using innerHTML
   - Use DomSanitizer when necessary

3. **Authentication**:
   - Implement proper authentication
   - Use secure token storage
   - Implement token refresh mechanism

4. **Authorization**:
   - Use route guards
   - Implement role-based access control
   - Validate permissions on both client and server

5. **HTTP Security**:
   - Use HTTPS
   - Implement CSRF protection
   - Set secure HTTP headers

6. **Data Protection**:
   - Encrypt sensitive data
   - Use secure storage methods
   - Implement proper session management

7. **Error Handling**:
   - Don't expose sensitive information
   - Implement proper error logging
   - Show user-friendly error messages

8. **File Upload**:
   - Validate file types and sizes
   - Scan for malware
   - Use secure upload endpoints

9. **Configuration**:
   - Use environment-specific settings
   - Keep sensitive data in environment files
   - Use secure configuration management

10. **Monitoring and Logging**:
    - Implement security event logging
    - Monitor for suspicious activities
    - Set up alerts for security incidents

### Conclusion

Security is a critical aspect of Angular application development. By implementing these best practices and security measures, you can protect your application from common vulnerabilities and ensure the safety of your users' data. Remember to:

1. Keep Angular and dependencies updated
2. Follow security best practices
3. Implement proper authentication and authorization
4. Validate all inputs and sanitize outputs
5. Use secure communication channels
6. Implement proper error handling
7. Monitor and log security events
8. Regular security audits and testing

A secure application is not just about implementing these measures but also about maintaining them and staying updated with the latest security practices and vulnerabilities. 