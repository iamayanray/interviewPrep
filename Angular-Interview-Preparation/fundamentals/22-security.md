# Angular Security Best Practices

## Question
How do you implement security in Angular applications? Explain different security measures and best practices.

## Answer

### Introduction to Angular Security

Angular provides built-in security features and best practices to protect applications from common web vulnerabilities. Here's a comprehensive guide to implementing security in Angular applications.

### 1. Cross-Site Scripting (XSS) Protection

#### Built-in XSS Protection

```typescript
// Angular automatically escapes values in templates
@Component({
  selector: 'app-user',
  template: `
    <!-- Safe - Angular automatically escapes -->
    <div>{{ userInput }}</div>
    
    <!-- Safe - Angular automatically escapes -->
    <div [innerHTML]="sanitizedHtml"></div>
  `
})
export class UserComponent {
  userInput = '<script>alert("xss")</script>';
  sanitizedHtml = this.sanitizer.bypassSecurityTrustHtml('<div>Safe HTML</div>');

  constructor(private sanitizer: DomSanitizer) {}
}
```

#### Using DomSanitizer

```typescript
// sanitizer.service.ts
@Injectable({
  providedIn: 'root'
})
export class SanitizerService {
  constructor(private sanitizer: DomSanitizer) {}

  sanitizeHtml(html: string): SafeHtml {
    return this.sanitizer.bypassSecurityTrustHtml(html);
  }

  sanitizeUrl(url: string): SafeUrl {
    return this.sanitizer.bypassSecurityTrustUrl(url);
  }

  sanitizeStyle(style: string): SafeStyle {
    return this.sanitizer.bypassSecurityTrustStyle(style);
  }
}
```

### 2. Cross-Site Request Forgery (CSRF) Protection

#### HTTP Interceptor for CSRF

```typescript
// csrf.interceptor.ts
@Injectable()
export class CsrfInterceptor implements HttpInterceptor {
  constructor(private tokenService: TokenService) {}

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.tokenService.getCsrfToken();
    
    if (token) {
      request = request.clone({
        headers: request.headers.set('X-CSRF-Token', token)
      });
    }

    return next.handle(request);
  }
}

// app.module.ts
@NgModule({
  providers: [
    { provide: HTTP_INTERCEPTORS, useClass: CsrfInterceptor, multi: true }
  ]
})
export class AppModule { }
```

### 3. Content Security Policy (CSP)

#### Implementing CSP Headers

```typescript
// server.ts (Node.js/Express example)
import helmet from 'helmet';

app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'", "'unsafe-inline'"],
    styleSrc: ["'self'", "'unsafe-inline'"],
    imgSrc: ["'self'", "data:", "https:"],
    connectSrc: ["'self'", "https://api.example.com"],
    fontSrc: ["'self'", "https://fonts.gstatic.com"],
    objectSrc: ["'none'"],
    mediaSrc: ["'self'"],
    frameSrc: ["'none'"]
  }
}));
```

### 4. Authentication and Authorization

#### Auth Guard

```typescript
// auth.guard.ts
@Injectable({
  providedIn: 'root'
})
export class AuthGuard implements CanActivate {
  constructor(
    private router: Router,
    private authService: AuthService
  ) {}

  canActivate(route: ActivatedRouteSnapshot): boolean {
    if (this.authService.isAuthenticated()) {
      return true;
    }

    this.router.navigate(['/login']);
    return false;
  }
}

// Using in routing
const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [AuthGuard]
  }
];
```

#### Role-based Authorization

```typescript
// role.guard.ts
@Injectable({
  providedIn: 'root'
})
export class RoleGuard implements CanActivate {
  constructor(
    private router: Router,
    private authService: AuthService
  ) {}

  canActivate(route: ActivatedRouteSnapshot): boolean {
    const requiredRoles = route.data['roles'] as string[];
    const userRoles = this.authService.getUserRoles();

    if (requiredRoles.some(role => userRoles.includes(role))) {
      return true;
    }

    this.router.navigate(['/unauthorized']);
    return false;
  }
}

// Using in routing
const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [AuthGuard, RoleGuard],
    data: { roles: ['ADMIN'] }
  }
];
```

### 5. Secure HTTP Headers

#### Using Helmet (Node.js/Express)

```typescript
// server.ts
import helmet from 'helmet';

app.use(helmet());
app.use(helmet.hsts({
  maxAge: 31536000,
  includeSubDomains: true,
  preload: true
}));
app.use(helmet.noSniff());
app.use(helmet.xssFilter());
app.use(helmet.frameguard({ action: 'deny' }));
```

### 6. Input Validation

#### Form Validation

```typescript
// registration.component.ts
@Component({
  selector: 'app-registration',
  template: `
    <form [formGroup]="registrationForm" (ngSubmit)="onSubmit()">
      <input formControlName="email" type="email">
      <input formControlName="password" type="password">
      <button type="submit">Register</button>
    </form>
  `
})
export class RegistrationComponent {
  registrationForm = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [
      Validators.required,
      Validators.minLength(8),
      Validators.pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$/)
    ]]
  });

  constructor(private fb: FormBuilder) {}
}
```

### 7. Secure File Upload

```typescript
// file-upload.component.ts
@Component({
  selector: 'app-file-upload',
  template: `
    <input type="file" (change)="onFileSelected($event)">
    <button (click)="uploadFile()">Upload</button>
  `
})
export class FileUploadComponent {
  private allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
  private maxSize = 5 * 1024 * 1024; // 5MB
  selectedFile: File | null = null;

  onFileSelected(event: Event) {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (file) {
      if (!this.allowedTypes.includes(file.type)) {
        alert('Invalid file type');
        return;
      }
      if (file.size > this.maxSize) {
        alert('File too large');
        return;
      }
      this.selectedFile = file;
    }
  }

  uploadFile() {
    if (!this.selectedFile) return;

    const formData = new FormData();
    formData.append('file', this.selectedFile);

    this.http.post('/api/upload', formData).subscribe({
      next: (response) => console.log('Upload successful', response),
      error: (error) => console.error('Upload failed', error)
    });
  }
}
```

### 8. Error Handling and Logging

```typescript
// error-handler.service.ts
@Injectable({
  providedIn: 'root'
})
export class ErrorHandlerService implements ErrorHandler {
  constructor(private http: HttpClient) {}

  handleError(error: Error) {
    // Log error securely
    this.logError(error);
    
    // Show user-friendly message
    this.showErrorMessage(error);
  }

  private logError(error: Error) {
    const errorLog = {
      timestamp: new Date().toISOString(),
      message: error.message,
      stack: error.stack,
      url: window.location.href
    };

    // Send to secure logging service
    this.http.post('/api/error-logs', errorLog).subscribe();
  }

  private showErrorMessage(error: Error) {
    // Show generic error message to user
    alert('An error occurred. Please try again later.');
  }
}
```

### 9. Secure Storage

```typescript
// storage.service.ts
@Injectable({
  providedIn: 'root'
})
export class StorageService {
  constructor() {}

  setSecureItem(key: string, value: string) {
    // Encrypt sensitive data before storing
    const encryptedValue = this.encrypt(value);
    localStorage.setItem(key, encryptedValue);
  }

  getSecureItem(key: string): string | null {
    const encryptedValue = localStorage.getItem(key);
    return encryptedValue ? this.decrypt(encryptedValue) : null;
  }

  private encrypt(value: string): string {
    // Implement encryption
    return btoa(value); // Basic example, use proper encryption in production
  }

  private decrypt(value: string): string {
    // Implement decryption
    return atob(value); // Basic example, use proper decryption in production
  }
}
```

### 10. Environment Configuration

```typescript
// environment.ts
export const environment = {
  production: false,
  apiUrl: 'https://api.example.com',
  allowedOrigins: ['https://app.example.com'],
  securityConfig: {
    tokenExpiration: 3600,
    refreshTokenExpiration: 86400,
    maxLoginAttempts: 5,
    lockoutDuration: 900
  }
};

// environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.production.example.com',
  allowedOrigins: ['https://app.production.example.com'],
  securityConfig: {
    tokenExpiration: 1800,
    refreshTokenExpiration: 43200,
    maxLoginAttempts: 3,
    lockoutDuration: 1800
  }
};
```

### Conclusion

Key points to remember:

1. **XSS Protection**:
   - Use Angular's built-in escaping
   - Use DomSanitizer for trusted content
   - Avoid innerHTML when possible

2. **CSRF Protection**:
   - Implement CSRF tokens
   - Use HTTP interceptors
   - Validate tokens server-side

3. **Authentication**:
   - Implement proper authentication
   - Use secure session management
   - Implement role-based access control

4. **Input Validation**:
   - Validate all user inputs
   - Use form validation
   - Implement server-side validation

5. **Secure Storage**:
   - Encrypt sensitive data
   - Use secure storage methods
   - Implement proper key management

6. **HTTP Security**:
   - Use HTTPS
   - Implement secure headers
   - Use proper CORS configuration

Remember to:
- Keep dependencies updated
- Follow security best practices
- Implement proper error handling
- Use secure communication
- Regular security audits
- Monitor for security incidents
- Implement proper logging
- Use environment-specific configurations 