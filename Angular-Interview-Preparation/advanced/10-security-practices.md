# Advanced Angular Security Practices

## Question
What are the advanced security practices in Angular? Explain XSS protection, CSRF protection, and other security measures with examples.

## Answer

### Introduction to Advanced Angular Security

Angular provides powerful security features to protect applications from common web vulnerabilities. Here's a comprehensive guide to advanced security practices.

### 1. Advanced XSS Protection

#### Custom Sanitizer

```typescript
// custom-sanitizer.service.ts
@Injectable({ providedIn: 'root' })
export class CustomSanitizer {
  constructor(private sanitizer: DomSanitizer) {}

  sanitizeHtml(html: string): SafeHtml {
    // Remove potentially dangerous tags and attributes
    const sanitizedHtml = html
      .replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '')
      .replace(/<iframe\b[^<]*(?:(?!<\/iframe>)<[^<]*)*<\/iframe>/gi, '')
      .replace(/on\w+="[^"]*"/g, '')
      .replace(/on\w+='[^']*'/g, '');

    return this.sanitizer.bypassSecurityTrustHtml(sanitizedHtml);
  }

  sanitizeUrl(url: string): SafeUrl {
    // Validate URL format and protocol
    if (!this.isValidUrl(url)) {
      return this.sanitizer.bypassSecurityTrustUrl('about:blank');
    }

    return this.sanitizer.bypassSecurityTrustUrl(url);
  }

  sanitizeStyle(style: string): SafeStyle {
    // Remove potentially dangerous CSS properties
    const sanitizedStyle = style
      .replace(/expression\s*\(/g, '')
      .replace(/url\s*\(/g, '')
      .replace(/javascript:/g, '');

    return this.sanitizer.bypassSecurityTrustStyle(sanitizedStyle);
  }

  private isValidUrl(url: string): boolean {
    try {
      const parsedUrl = new URL(url);
      return ['http:', 'https:'].includes(parsedUrl.protocol);
    } catch {
      return false;
    }
  }
}

// Using in component
@Component({
  selector: 'app-content',
  template: `
    <div [innerHTML]="sanitizedHtml"></div>
    <a [href]="sanitizedUrl">Link</a>
    <div [style]="sanitizedStyle">Styled content</div>
  `
})
export class ContentComponent {
  sanitizedHtml: SafeHtml;
  sanitizedUrl: SafeUrl;
  sanitizedStyle: SafeStyle;

  constructor(private sanitizer: CustomSanitizer) {
    this.sanitizedHtml = this.sanitizer.sanitizeHtml('<p>User content</p>');
    this.sanitizedUrl = this.sanitizer.sanitizeUrl('https://example.com');
    this.sanitizedStyle = this.sanitizer.sanitizeStyle('color: red;');
  }
}
```

#### Content Security Policy

```typescript
// csp.service.ts
@Injectable({ providedIn: 'root' })
export class CSPService {
  private readonly CSP_HEADER = {
    'Content-Security-Policy': `
      default-src 'self';
      script-src 'self' 'unsafe-inline' 'unsafe-eval';
      style-src 'self' 'unsafe-inline';
      img-src 'self' data: https:;
      font-src 'self';
      connect-src 'self' https://api.example.com;
      frame-ancestors 'none';
      form-action 'self';
      base-uri 'self';
      object-src 'none';
      media-src 'self';
      worker-src 'self' blob:;
      manifest-src 'self';
      upgrade-insecure-requests;
    `.replace(/\s+/g, ' ').trim()
  };

  applyCSPHeaders(response: HttpResponse<any>): HttpResponse<any> {
    Object.entries(this.CSP_HEADER).forEach(([key, value]) => {
      response.headers.set(key, value);
    });
    return response;
  }
}

// Using in HTTP interceptor
@Injectable()
export class SecurityInterceptor implements HttpInterceptor {
  constructor(private cspService: CSPService) {}

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(request).pipe(
      map(event => {
        if (event instanceof HttpResponse) {
          return this.cspService.applyCSPHeaders(event);
        }
        return event;
      })
    );
  }
}
```

### 2. Advanced CSRF Protection

#### Custom CSRF Interceptor

```typescript
// csrf.interceptor.ts
@Injectable()
export class CSRFInterceptor implements HttpInterceptor {
  private readonly CSRF_HEADER = 'X-CSRF-Token';
  private readonly CSRF_COOKIE = 'XSRF-TOKEN';

  constructor(
    private cookieService: CookieService,
    private http: HttpClient
  ) {}

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // Skip CSRF for GET requests and non-mutating operations
    if (this.shouldSkipCSRF(request)) {
      return next.handle(request);
    }

    // Get CSRF token from cookie
    const token = this.cookieService.get(this.CSRF_COOKIE);
    if (!token) {
      return this.refreshToken().pipe(
        switchMap(newToken => this.handleRequest(request, next, newToken))
      );
    }

    return this.handleRequest(request, next, token);
  }

  private shouldSkipCSRF(request: HttpRequest<any>): boolean {
    return request.method === 'GET' || 
           request.headers.has('Skip-CSRF') ||
           !this.isSameOrigin(request.url);
  }

  private isSameOrigin(url: string): boolean {
    const currentOrigin = window.location.origin;
    const requestOrigin = new URL(url).origin;
    return currentOrigin === requestOrigin;
  }

  private handleRequest(
    request: HttpRequest<any>,
    next: HttpHandler,
    token: string
  ): Observable<HttpEvent<any>> {
    const modifiedRequest = request.clone({
      headers: request.headers.set(this.CSRF_HEADER, token)
    });
    return next.handle(modifiedRequest);
  }

  private refreshToken(): Observable<string> {
    return this.http.get<{ token: string }>('/api/csrf-token').pipe(
      map(response => {
        const token = response.token;
        this.cookieService.set(this.CSRF_COOKIE, token, {
          secure: true,
          sameSite: 'strict'
        });
        return token;
      })
    );
  }
}
```

#### Double Submit Cookie Pattern

```typescript
// csrf.service.ts
@Injectable({ providedIn: 'root' })
export class CSRFService {
  private readonly TOKEN_COOKIE = 'XSRF-TOKEN';
  private readonly TOKEN_HEADER = 'X-CSRF-Token';
  private readonly TOKEN_LENGTH = 32;

  constructor(
    private cookieService: CookieService,
    private cryptoService: CryptoService
  ) {}

  generateToken(): string {
    const token = this.cryptoService.generateRandomString(this.TOKEN_LENGTH);
    this.setTokenCookie(token);
    return token;
  }

  validateToken(token: string): boolean {
    const cookieToken = this.cookieService.get(this.TOKEN_COOKIE);
    return token === cookieToken;
  }

  private setTokenCookie(token: string): void {
    this.cookieService.set(this.TOKEN_COOKIE, token, {
      secure: true,
      sameSite: 'strict',
      httpOnly: true
    });
  }

  getTokenHeader(): string {
    return this.TOKEN_HEADER;
  }
}

// Using in HTTP interceptor
@Injectable()
export class CSRFInterceptor implements HttpInterceptor {
  constructor(private csrfService: CSRFService) {}

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    if (this.shouldSkipCSRF(request)) {
      return next.handle(request);
    }

    const token = this.csrfService.generateToken();
    const modifiedRequest = request.clone({
      headers: request.headers.set(
        this.csrfService.getTokenHeader(),
        token
      )
    });

    return next.handle(modifiedRequest);
  }

  private shouldSkipCSRF(request: HttpRequest<any>): boolean {
    return request.method === 'GET' || 
           request.headers.has('Skip-CSRF');
  }
}
```

### 3. Advanced Authentication

#### JWT Authentication Service

```typescript
// auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly TOKEN_KEY = 'auth_token';
  private readonly REFRESH_TOKEN_KEY = 'refresh_token';
  private tokenExpirationTimer: any;

  constructor(
    private http: HttpClient,
    private router: Router,
    private storageService: StorageService
  ) {}

  login(credentials: LoginCredentials): Observable<AuthResponse> {
    return this.http.post<AuthResponse>('/api/auth/login', credentials).pipe(
      tap(response => this.handleLogin(response))
    );
  }

  refreshToken(): Observable<AuthResponse> {
    const refreshToken = this.storageService.get(this.REFRESH_TOKEN_KEY);
    return this.http.post<AuthResponse>('/api/auth/refresh', { refreshToken }).pipe(
      tap(response => this.handleLogin(response))
    );
  }

  logout(): void {
    this.storageService.remove(this.TOKEN_KEY);
    this.storageService.remove(this.REFRESH_TOKEN_KEY);
    if (this.tokenExpirationTimer) {
      clearTimeout(this.tokenExpirationTimer);
    }
    this.router.navigate(['/login']);
  }

  private handleLogin(response: AuthResponse): void {
    this.storageService.set(this.TOKEN_KEY, response.token);
    this.storageService.set(this.REFRESH_TOKEN_KEY, response.refreshToken);
    this.setTokenExpirationTimer(response.expiresIn);
  }

  private setTokenExpirationTimer(expiresIn: number): void {
    this.tokenExpirationTimer = setTimeout(() => {
      this.refreshToken().subscribe({
        error: () => this.logout()
      });
    }, expiresIn * 1000);
  }

  isAuthenticated(): boolean {
    return !!this.storageService.get(this.TOKEN_KEY);
  }

  getToken(): string | null {
    return this.storageService.get(this.TOKEN_KEY);
  }
}
```

#### Token Interceptor

```typescript
// token.interceptor.ts
@Injectable()
export class TokenInterceptor implements HttpInterceptor {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.authService.getToken();
    if (!token) {
      return next.handle(request);
    }

    const modifiedRequest = request.clone({
      headers: request.headers.set('Authorization', `Bearer ${token}`)
    });

    return next.handle(modifiedRequest).pipe(
      catchError(error => this.handleError(error))
    );
  }

  private handleError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    if (error.status === 401) {
      return this.authService.refreshToken().pipe(
        switchMap(() => {
          const token = this.authService.getToken();
          const request = error.request.clone({
            headers: error.request.headers.set('Authorization', `Bearer ${token}`)
          });
          return next.handle(request);
        }),
        catchError(() => {
          this.authService.logout();
          return throwError(() => error);
        })
      );
    }
    return throwError(() => error);
  }
}
```

### 4. Advanced Authorization

#### Role-Based Access Control

```typescript
// auth.guard.ts
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(route: ActivatedRouteSnapshot): boolean {
    if (!this.authService.isAuthenticated()) {
      this.router.navigate(['/login']);
      return false;
    }

    const requiredRoles = route.data['roles'] as string[];
    if (requiredRoles) {
      const userRoles = this.authService.getUserRoles();
      if (!this.hasRequiredRoles(userRoles, requiredRoles)) {
        this.router.navigate(['/unauthorized']);
        return false;
      }
    }

    return true;
  }

  private hasRequiredRoles(userRoles: string[], requiredRoles: string[]): boolean {
    return requiredRoles.some(role => userRoles.includes(role));
  }
}

// Using in routing
const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [AuthGuard],
    data: { roles: ['admin'] }
  },
  {
    path: 'user',
    component: UserComponent,
    canActivate: [AuthGuard],
    data: { roles: ['user', 'admin'] }
  }
];
```

#### Permission-Based Access Control

```typescript
// permission.directive.ts
@Directive({
  selector: '[appPermission]',
  exportAs: 'permission'
})
export class PermissionDirective implements OnInit {
  @Input('appPermission') permission: string;
  @Input('appPermissionMode') mode: 'hide' | 'disable' = 'hide';

  constructor(
    private elementRef: ElementRef,
    private renderer: Renderer2,
    private authService: AuthService
  ) {}

  ngOnInit() {
    this.checkPermission();
  }

  private checkPermission(): void {
    const hasPermission = this.authService.hasPermission(this.permission);
    
    if (this.mode === 'hide') {
      this.renderer.setStyle(
        this.elementRef.nativeElement,
        'display',
        hasPermission ? '' : 'none'
      );
    } else {
      this.renderer.setProperty(
        this.elementRef.nativeElement,
        'disabled',
        !hasPermission
      );
    }
  }
}

// Using in template
@Component({
  selector: 'app-feature',
  template: `
    <button appPermission="edit" appPermissionMode="disable">
      Edit
    </button>
    <div appPermission="admin">
      Admin content
    </div>
  `
})
export class FeatureComponent {}
```

### 5. Advanced Security Headers

#### Security Headers Service

```typescript
// security-headers.service.ts
@Injectable({ providedIn: 'root' })
export class SecurityHeadersService {
  private readonly SECURITY_HEADERS = {
    'X-Content-Type-Options': 'nosniff',
    'X-Frame-Options': 'DENY',
    'X-XSS-Protection': '1; mode=block',
    'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
    'Referrer-Policy': 'strict-origin-when-cross-origin',
    'Permissions-Policy': 'geolocation=(), microphone=(), camera=()',
    'Content-Security-Policy': this.getCSPHeader()
  };

  applySecurityHeaders(response: HttpResponse<any>): HttpResponse<any> {
    Object.entries(this.SECURITY_HEADERS).forEach(([key, value]) => {
      response.headers.set(key, value);
    });
    return response;
  }

  private getCSPHeader(): string {
    return `
      default-src 'self';
      script-src 'self' 'unsafe-inline' 'unsafe-eval';
      style-src 'self' 'unsafe-inline';
      img-src 'self' data: https:;
      font-src 'self';
      connect-src 'self' https://api.example.com;
      frame-ancestors 'none';
      form-action 'self';
      base-uri 'self';
      object-src 'none';
      media-src 'self';
      worker-src 'self' blob:;
      manifest-src 'self';
      upgrade-insecure-requests;
    `.replace(/\s+/g, ' ').trim();
  }
}

// Using in HTTP interceptor
@Injectable()
export class SecurityHeadersInterceptor implements HttpInterceptor {
  constructor(private securityHeadersService: SecurityHeadersService) {}

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(request).pipe(
      map(event => {
        if (event instanceof HttpResponse) {
          return this.securityHeadersService.applySecurityHeaders(event);
        }
        return event;
      })
    );
  }
}
```

### 6. Advanced Input Validation

#### Custom Form Validators

```typescript
// password.validator.ts
@Injectable({ providedIn: 'root' })
export class PasswordValidator {
  static strong(control: AbstractControl): ValidationErrors | null {
    const value = control.value;
    if (!value) return null;

    const hasUpperCase = /[A-Z]+/.test(value);
    const hasLowerCase = /[a-z]+/.test(value);
    const hasNumeric = /[0-9]+/.test(value);
    const hasSpecialChar = /[!@#$%^&*()]+/.test(value);
    const isLongEnough = value.length >= 8;

    const valid = hasUpperCase && 
                 hasLowerCase && 
                 hasNumeric && 
                 hasSpecialChar && 
                 isLongEnough;

    return valid ? null : {
      passwordStrength: {
        hasUpperCase,
        hasLowerCase,
        hasNumeric,
        hasSpecialChar,
        isLongEnough
      }
    };
  }

  static match(control: AbstractControl): ValidationErrors | null {
    const password = control.get('password')?.value;
    const confirmPassword = control.get('confirmPassword')?.value;

    return password === confirmPassword ? null : {
      passwordMatch: true
    };
  }
}

// Using in component
@Component({
  selector: 'app-registration',
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <input formControlName="password" type="password">
      <input formControlName="confirmPassword" type="password">
      <button type="submit">Register</button>
    </form>
  `
})
export class RegistrationComponent {
  form = this.fb.group({
    password: ['', [Validators.required, PasswordValidator.strong]],
    confirmPassword: ['', Validators.required]
  }, { validators: PasswordValidator.match });

  constructor(private fb: FormBuilder) {}
}
```

### 7. Advanced Secure File Upload

```typescript
// file-upload.component.ts
@Component({
  selector: 'app-file-upload',
  template: `
    <input type="file" (change)="onFileSelected($event)">
    <button (click)="uploadFile()" [disabled]="!selectedFile">
      Upload
    </button>
  `
})
export class FileUploadComponent {
  selectedFile: File | null = null;
  private readonly MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
  private readonly ALLOWED_TYPES = ['image/jpeg', 'image/png', 'application/pdf'];

  onFileSelected(event: Event): void {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (!file) return;

    if (!this.validateFile(file)) {
      return;
    }

    this.selectedFile = file;
  }

  private validateFile(file: File): boolean {
    if (file.size > this.MAX_FILE_SIZE) {
      this.showError('File size exceeds 5MB limit');
      return false;
    }

    if (!this.ALLOWED_TYPES.includes(file.type)) {
      this.showError('Invalid file type');
      return false;
    }

    return true;
  }

  uploadFile(): void {
    if (!this.selectedFile) return;

    const formData = new FormData();
    formData.append('file', this.selectedFile);

    this.http.post('/api/upload', formData).subscribe({
      next: () => this.showSuccess('File uploaded successfully'),
      error: error => this.showError('Upload failed')
    });
  }

  private showError(message: string): void {
    // Implement error notification
  }

  private showSuccess(message: string): void {
    // Implement success notification
  }
}
```

### Conclusion

Key points to remember:

1. **XSS Protection**:
   - Sanitize user input
   - Use security headers
   - Implement CSP
   - Escape values in templates
   - Validate HTML content

2. **CSRF Protection**:
   - Use tokens
   - Implement double submit pattern
   - Validate requests
   - Handle token refresh
   - Secure cookie settings

3. **Authentication**:
   - Use JWT tokens
   - Implement refresh tokens
   - Handle token expiration
   - Secure token storage
   - Manage sessions

4. **Authorization**:
   - Implement RBAC
   - Use permission-based access
   - Protect routes
   - Validate user roles
   - Handle unauthorized access

5. **Security Headers**:
   - Set appropriate headers
   - Implement CSP
   - Use HSTS
   - Configure CORS
   - Set referrer policy

Remember to:
- Follow security best practices
- Keep dependencies updated
- Monitor security threats
- Implement proper logging
- Handle errors securely
- Test security measures
- Document security policies
- Regular security audits 