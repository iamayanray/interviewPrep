# Advanced Angular Security Practices

## Question
What are the advanced security practices in Angular? Explain XSS protection, CSRF protection, and other security measures with examples.

## Answer

### Introduction to Advanced Angular Security Practices

Angular provides robust security features to protect applications from common web vulnerabilities. Here's a comprehensive guide to advanced security practices.

### 1. Advanced XSS Protection

#### Custom Sanitizer

```typescript
// custom-sanitizer.service.ts
@Injectable({ providedIn: 'root' })
export class CustomSanitizerService {
  constructor(private sanitizer: DomSanitizer) {}

  sanitizeHtml(html: string): SafeHtml {
    return this.sanitizer.bypassSecurityTrustHtml(
      this.sanitizeHtmlContent(html)
    );
  }

  sanitizeUrl(url: string): SafeUrl {
    return this.sanitizer.bypassSecurityTrustUrl(
      this.sanitizeUrlContent(url)
    );
  }

  sanitizeStyle(style: string): SafeStyle {
    return this.sanitizer.bypassSecurityTrustStyle(
      this.sanitizeStyleContent(style)
    );
  }

  sanitizeScript(script: string): SafeScript {
    return this.sanitizer.bypassSecurityTrustScript(
      this.sanitizeScriptContent(script)
    );
  }

  sanitizeResourceUrl(url: string): SafeResourceUrl {
    return this.sanitizer.bypassSecurityTrustResourceUrl(
      this.sanitizeResourceUrlContent(url)
    );
  }

  private sanitizeHtmlContent(html: string): string {
    // Remove potentially dangerous HTML tags
    return html.replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '')
               .replace(/<style\b[^<]*(?:(?!<\/style>)<[^<]*)*<\/style>/gi, '')
               .replace(/on\w+="[^"]*"/g, '')
               .replace(/on\w+='[^']*'/g, '');
  }

  private sanitizeUrlContent(url: string): string {
    // Validate URL format and protocol
    try {
      const parsedUrl = new URL(url);
      if (!['http:', 'https:'].includes(parsedUrl.protocol)) {
        throw new Error('Invalid URL protocol');
      }
      return url;
    } catch {
      return 'about:blank';
    }
  }

  private sanitizeStyleContent(style: string): string {
    // Remove potentially dangerous CSS properties
    return style.replace(/expression\s*\(/g, '')
                .replace(/url\s*\(/g, '')
                .replace(/javascript:/g, '');
  }

  private sanitizeScriptContent(script: string): string {
    // Remove potentially dangerous JavaScript code
    return script.replace(/eval\s*\(/g, '')
                 .replace(/Function\s*\(/g, '')
                 .replace(/setTimeout\s*\(/g, '')
                 .replace(/setInterval\s*\(/g, '');
  }

  private sanitizeResourceUrlContent(url: string): string {
    // Validate resource URL format and protocol
    try {
      const parsedUrl = new URL(url);
      if (!['http:', 'https:', 'data:', 'blob:'].includes(parsedUrl.protocol)) {
        throw new Error('Invalid resource URL protocol');
      }
      return url;
    } catch {
      return 'about:blank';
    }
  }
}

// Using in component
@Component({
  selector: 'app-content',
  template: `
    <div [innerHTML]="sanitizedHtml"></div>
    <a [href]="sanitizedUrl">Link</a>
    <div [style]="sanitizedStyle">Styled Content</div>
  `
})
export class ContentComponent {
  sanitizedHtml: SafeHtml;
  sanitizedUrl: SafeUrl;
  sanitizedStyle: SafeStyle;

  constructor(private sanitizer: CustomSanitizerService) {
    this.sanitizedHtml = this.sanitizer.sanitizeHtml(
      '<div onclick="alert(1)">Content</div>'
    );
    this.sanitizedUrl = this.sanitizer.sanitizeUrl(
      'javascript:alert(1)'
    );
    this.sanitizedStyle = this.sanitizer.sanitizeStyle(
      'background: expression(alert(1))'
    );
  }
}
```

#### Content Security Policy

```typescript
// security-headers.service.ts
@Injectable({ providedIn: 'root' })
export class SecurityHeadersService {
  private readonly CSP_HEADER = {
    'Content-Security-Policy': `
      default-src 'self';
      script-src 'self' 'unsafe-inline' 'unsafe-eval';
      style-src 'self' 'unsafe-inline';
      img-src 'self' data: https:;
      font-src 'self' data:;
      connect-src 'self' https://api.example.com;
      frame-ancestors 'none';
      form-action 'self';
      base-uri 'self';
      object-src 'none';
      media-src 'self';
      frame-src 'self';
      worker-src 'self' blob:;
      manifest-src 'self';
      upgrade-insecure-requests;
    `.replace(/\s+/g, ' ').trim()
  };

  private readonly SECURITY_HEADERS = {
    ...this.CSP_HEADER,
    'X-Content-Type-Options': 'nosniff',
    'X-Frame-Options': 'DENY',
    'X-XSS-Protection': '1; mode=block',
    'Referrer-Policy': 'strict-origin-when-cross-origin',
    'Permissions-Policy': 'geolocation=(), microphone=(), camera=()'
  };

  applySecurityHeaders(response: HttpResponse<any>): HttpResponse<any> {
    Object.entries(this.SECURITY_HEADERS).forEach(([key, value]) => {
      response.headers.set(key, value);
    });
    return response;
  }
}

// Using in interceptor
@Injectable()
export class SecurityHeadersInterceptor implements HttpInterceptor {
  constructor(private securityHeaders: SecurityHeadersService) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    return next.handle(request).pipe(
      map(event => {
        if (event instanceof HttpResponse) {
          return this.securityHeaders.applySecurityHeaders(event);
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
export class CsrfInterceptor implements HttpInterceptor {
  private readonly CSRF_HEADER = 'X-CSRF-Token';
  private readonly CSRF_COOKIE = 'XSRF-TOKEN';

  constructor(private cookieService: CookieService) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    if (this.shouldAddCsrfToken(request)) {
      const token = this.getCsrfToken();
      if (token) {
        request = request.clone({
          headers: request.headers.set(this.CSRF_HEADER, token)
        });
      }
    }
    return next.handle(request);
  }

  private shouldAddCsrfToken(request: HttpRequest<any>): boolean {
    return request.method !== 'GET' &&
           !request.headers.has(this.CSRF_HEADER) &&
           !this.isExcludedUrl(request.url);
  }

  private getCsrfToken(): string | null {
    return this.cookieService.get(this.CSRF_COOKIE);
  }

  private isExcludedUrl(url: string): boolean {
    const excludedUrls = [
      '/api/public',
      '/api/auth/login',
      '/api/auth/logout'
    ];
    return excludedUrls.some(excludedUrl => url.includes(excludedUrl));
  }
}
```

#### Double Submit Cookie Pattern

```typescript
// csrf-token.service.ts
@Injectable({ providedIn: 'root' })
export class CsrfTokenService {
  private readonly TOKEN_LENGTH = 32;
  private readonly TOKEN_COOKIE = 'XSRF-TOKEN';
  private readonly TOKEN_HEADER = 'X-CSRF-Token';

  constructor(
    private cookieService: CookieService,
    private http: HttpClient
  ) {}

  generateToken(): string {
    const token = this.generateRandomToken();
    this.setTokenCookie(token);
    return token;
  }

  validateToken(token: string): boolean {
    const cookieToken = this.getTokenCookie();
    return token === cookieToken;
  }

  private generateRandomToken(): string {
    const array = new Uint8Array(this.TOKEN_LENGTH);
    crypto.getRandomValues(array);
    return Array.from(array, byte => byte.toString(16).padStart(2, '0')).join('');
  }

  private setTokenCookie(token: string): void {
    this.cookieService.set(this.TOKEN_COOKIE, token, {
      secure: true,
      sameSite: 'Strict',
      httpOnly: true
    });
  }

  private getTokenCookie(): string | null {
    return this.cookieService.get(this.TOKEN_COOKIE);
  }

  refreshToken(): Observable<void> {
    return this.http.post<void>('/api/csrf/refresh', {}).pipe(
      tap(() => {
        const newToken = this.generateToken();
        this.setTokenCookie(newToken);
      })
    );
  }
}

// Using in component
@Component({
  selector: 'app-form',
  template: `
    <form (ngSubmit)="onSubmit()">
      <input type="text" [(ngModel)]="data" name="data">
      <button type="submit">Submit</button>
    </form>
  `
})
export class FormComponent {
  data: string = '';

  constructor(
    private csrfService: CsrfTokenService,
    private http: HttpClient
  ) {}

  onSubmit(): void {
    const token = this.csrfService.generateToken();
    this.http.post('/api/data', { data: this.data }, {
      headers: new HttpHeaders().set('X-CSRF-Token', token)
    }).subscribe();
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
    private router: Router
  ) {}

  login(credentials: LoginCredentials): Observable<AuthResponse> {
    return this.http.post<AuthResponse>('/api/auth/login', credentials).pipe(
      tap(response => this.handleAuthentication(response))
    );
  }

  refreshToken(): Observable<AuthResponse> {
    const refreshToken = localStorage.getItem(this.REFRESH_TOKEN_KEY);
    return this.http.post<AuthResponse>('/api/auth/refresh', { refreshToken }).pipe(
      tap(response => this.handleAuthentication(response))
    );
  }

  logout(): void {
    localStorage.removeItem(this.TOKEN_KEY);
    localStorage.removeItem(this.REFRESH_TOKEN_KEY);
    if (this.tokenExpirationTimer) {
      clearTimeout(this.tokenExpirationTimer);
    }
    this.router.navigate(['/login']);
  }

  private handleAuthentication(response: AuthResponse): void {
    localStorage.setItem(this.TOKEN_KEY, response.token);
    localStorage.setItem(this.REFRESH_TOKEN_KEY, response.refreshToken);
    this.setTokenExpirationTimer(response.expiresIn);
  }

  private setTokenExpirationTimer(expiresIn: number): void {
    if (this.tokenExpirationTimer) {
      clearTimeout(this.tokenExpirationTimer);
    }
    this.tokenExpirationTimer = setTimeout(() => {
      this.refreshToken().subscribe();
    }, expiresIn * 1000);
  }

  getToken(): string | null {
    return localStorage.getItem(this.TOKEN_KEY);
  }

  isAuthenticated(): boolean {
    return !!this.getToken();
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

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    const token = this.authService.getToken();
    if (token) {
      request = this.addToken(request, token);
    }
    return next.handle(request).pipe(
      catchError(error => {
        if (error.status === 401) {
          return this.handle401Error(request, next);
        }
        return throwError(() => error);
      })
    );
  }

  private addToken(request: HttpRequest<any>, token: string): HttpRequest<any> {
    return request.clone({
      headers: request.headers.set('Authorization', `Bearer ${token}`)
    });
  }

  private handle401Error(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    return this.authService.refreshToken().pipe(
      switchMap(() => {
        const token = this.authService.getToken();
        if (token) {
          return next.handle(this.addToken(request, token));
        }
        return throwError(() => new Error('Authentication failed'));
      }),
      catchError(() => {
        this.authService.logout();
        return throwError(() => new Error('Authentication failed'));
      })
    );
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

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean> | Promise<boolean> | boolean {
    const requiredRoles = route.data['roles'] as string[];
    if (!requiredRoles) {
      return true;
    }

    return this.authService.getUserRoles().pipe(
      map(roles => {
        const hasRole = requiredRoles.some(role => roles.includes(role));
        if (!hasRole) {
          this.router.navigate(['/unauthorized']);
        }
        return hasRole;
      }),
      catchError(() => {
        this.router.navigate(['/login']);
        return of(false);
      })
    );
  }
}

// Using in routing
const routes: Routes = [
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [AuthGuard],
    data: { roles: ['ADMIN'] }
  },
  {
    path: 'user',
    component: UserComponent,
    canActivate: [AuthGuard],
    data: { roles: ['USER', 'ADMIN'] }
  }
];
```

#### Permission-Based Access Control

```typescript
// permission.directive.ts
@Directive({
  selector: '[appPermission]',
  exportAs: 'appPermission'
})
export class PermissionDirective implements OnInit {
  @Input('appPermission') permission: string;
  @Input('appPermissionMode') mode: 'hide' | 'disable' = 'hide';

  constructor(
    private elementRef: ElementRef,
    private renderer: Renderer2,
    private authService: AuthService
  ) {}

  ngOnInit(): void {
    this.checkPermission();
  }

  private checkPermission(): void {
    this.authService.hasPermission(this.permission).subscribe(
      hasPermission => {
        if (!hasPermission) {
          if (this.mode === 'hide') {
            this.renderer.setStyle(
              this.elementRef.nativeElement,
              'display',
              'none'
            );
          } else {
            this.renderer.setAttribute(
              this.elementRef.nativeElement,
              'disabled',
              'true'
            );
          }
        }
      }
    );
  }
}

// Using in template
@Component({
  selector: 'app-content',
  template: `
    <button appPermission="create" (click)="create()">Create</button>
    <button appPermission="edit" [appPermissionMode]="'disable'" (click)="edit()">
      Edit
    </button>
    <button appPermission="delete" (click)="delete()">Delete</button>
  `
})
export class ContentComponent {}
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
    'Referrer-Policy': 'strict-origin-when-cross-origin',
    'Permissions-Policy': 'geolocation=(), microphone=(), camera=()',
    'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
    'Content-Security-Policy': `
      default-src 'self';
      script-src 'self' 'unsafe-inline' 'unsafe-eval';
      style-src 'self' 'unsafe-inline';
      img-src 'self' data: https:;
      font-src 'self' data:;
      connect-src 'self' https://api.example.com;
      frame-ancestors 'none';
      form-action 'self';
      base-uri 'self';
      object-src 'none';
      media-src 'self';
      frame-src 'self';
      worker-src 'self' blob:;
      manifest-src 'self';
      upgrade-insecure-requests;
    `.replace(/\s+/g, ' ').trim()
  };

  applySecurityHeaders(response: HttpResponse<any>): HttpResponse<any> {
    Object.entries(this.SECURITY_HEADERS).forEach(([key, value]) => {
      response.headers.set(key, value);
    });
    return response;
  }
}

// Using in interceptor
@Injectable()
export class SecurityHeadersInterceptor implements HttpInterceptor {
  constructor(private securityHeaders: SecurityHeadersService) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    return next.handle(request).pipe(
      map(event => {
        if (event instanceof HttpResponse) {
          return this.securityHeaders.applySecurityHeaders(event);
        }
        return event;
      })
    );
  }
}
```

### 6. Advanced Input Validation

#### Custom Validators

```typescript
// custom.validators.ts
export class CustomValidators {
  static passwordStrength(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      const value = control.value;
      if (!value) return null;

      const hasUpperCase = /[A-Z]+/.test(value);
      const hasLowerCase = /[a-z]+/.test(value);
      const hasNumeric = /[0-9]+/.test(value);
      const hasSpecialChar = /[!@#$%^&*()]+/.test(value);
      const isLongEnough = value.length >= 8;

      const errors: ValidationErrors = {};
      if (!hasUpperCase) errors['hasUpperCase'] = true;
      if (!hasLowerCase) errors['hasLowerCase'] = true;
      if (!hasNumeric) errors['hasNumeric'] = true;
      if (!hasSpecialChar) errors['hasSpecialChar'] = true;
      if (!isLongEnough) errors['isLongEnough'] = true;

      return Object.keys(errors).length ? errors : null;
    };
  }

  static preventXss(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      const value = control.value;
      if (!value) return null;

      const xssPattern = /<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi;
      return xssPattern.test(value) ? { xss: true } : null;
    };
  }
}

// Using in component
@Component({
  selector: 'app-form',
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <input formControlName="username">
      <input type="password" formControlName="password">
      <textarea formControlName="content"></textarea>
      <button type="submit">Submit</button>
    </form>
  `
})
export class FormComponent {
  form = this.fb.group({
    username: ['', [Validators.required, CustomValidators.preventXss()]],
    password: ['', [Validators.required, CustomValidators.passwordStrength()]],
    content: ['', [Validators.required, CustomValidators.preventXss()]]
  });

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
  private readonly ALLOWED_TYPES = [
    'image/jpeg',
    'image/png',
    'application/pdf'
  ];

  onFileSelected(event: Event): void {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (file) {
      if (this.validateFile(file)) {
        this.selectedFile = file;
      } else {
        alert('Invalid file type or size');
      }
    }
  }

  private validateFile(file: File): boolean {
    // Check file size
    if (file.size > this.MAX_FILE_SIZE) {
      return false;
    }

    // Check file type
    if (!this.ALLOWED_TYPES.includes(file.type)) {
      return false;
    }

    // Check file content
    return this.validateFileContent(file);
  }

  private validateFileContent(file: File): boolean {
    // Implement content validation logic
    // For example, check for malicious content
    return true;
  }

  uploadFile(): void {
    if (!this.selectedFile) return;

    const formData = new FormData();
    formData.append('file', this.selectedFile);

    this.http.post('/api/upload', formData, {
      headers: new HttpHeaders({
        'Accept': 'application/json'
      })
    }).subscribe(
      response => {
        console.log('Upload successful:', response);
        this.selectedFile = null;
      },
      error => {
        console.error('Upload failed:', error);
      }
    );
  }
}
```

### Conclusion

Key points to remember:

1. **XSS Protection**:
   - Sanitize user input
   - Use Content Security Policy
   - Escape output
   - Validate URLs
   - Handle HTML safely

2. **CSRF Protection**:
   - Use tokens
   - Implement double submit
   - Validate requests
   - Handle token refresh
   - Secure cookies

3. **Authentication**:
   - Use JWT tokens
   - Implement refresh tokens
   - Handle token expiration
   - Secure storage
   - Validate credentials

4. **Authorization**:
   - Implement RBAC
   - Use guards
   - Check permissions
   - Handle roles
   - Secure routes

5. **Security Headers**:
   - Set CSP
   - Enable HSTS
   - Prevent clickjacking
   - Control referrer
   - Set permissions

6. **Input Validation**:
   - Validate all input
   - Prevent XSS
   - Check file types
   - Validate formats
   - Handle edge cases

7. **File Upload**:
   - Validate files
   - Check content
   - Limit size
   - Secure storage
   - Handle errors

Remember to:
- Follow security best practices
- Regular security audits
- Keep dependencies updated
- Monitor security events
- Document security measures
- Test security features
- Handle edge cases
- Monitor vulnerabilities 