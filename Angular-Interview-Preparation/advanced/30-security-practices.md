# Advanced Angular Security Practices

## Question
What are the advanced security practices in Angular? Explain XSS protection, CSRF protection, and other security measures with examples.

## Answer

### Introduction to Advanced Angular Security Practices

Angular provides robust security features to protect applications against common vulnerabilities. Here's a comprehensive guide to advanced security practices in Angular.

### 1. Advanced XSS Protection

#### Custom Sanitizer Service

```typescript
// custom-sanitizer.service.ts
@Injectable({ providedIn: 'root' })
export class CustomSanitizerService {
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

  sanitizeScript(script: string): SafeScript {
    return this.sanitizer.bypassSecurityTrustScript(script);
  }

  sanitizeResourceUrl(url: string): SafeResourceUrl {
    return this.sanitizer.bypassSecurityTrustResourceUrl(url);
  }

  sanitizeInput(input: string): string {
    return input
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#039;');
  }
}

// Using in component
@Component({
  selector: 'app-sanitizer',
  template: `
    <div [innerHTML]="sanitizedHtml"></div>
    <a [href]="sanitizedUrl">Link</a>
    <div [style]="sanitizedStyle">Styled content</div>
  `
})
export class SanitizerComponent {
  sanitizedHtml: SafeHtml;
  sanitizedUrl: SafeUrl;
  sanitizedStyle: SafeStyle;

  constructor(private sanitizer: CustomSanitizerService) {
    const userInput = '<script>alert("xss")</script>';
    this.sanitizedHtml = this.sanitizer.sanitizeHtml(userInput);
    this.sanitizedUrl = this.sanitizer.sanitizeUrl('javascript:alert("xss")');
    this.sanitizedStyle = this.sanitizer.sanitizeStyle('background: url("javascript:alert(1)")');
  }
}
```

#### Content Security Policy

```typescript
// security-headers.service.ts
@Injectable({ providedIn: 'root' })
export class SecurityHeadersService {
  private readonly cspHeader = {
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
      frame-src 'self';
      worker-src 'self' blob:;
      manifest-src 'self';
      upgrade-insecure-requests;
    `.replace(/\s+/g, ' ').trim()
  };

  applySecurityHeaders(response: HttpResponse<any>): HttpResponse<any> {
    Object.entries(this.cspHeader).forEach(([key, value]) => {
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
  constructor(
    private csrfService: CsrfService,
    private router: Router
  ) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    if (this.shouldAddCsrfToken(request)) {
      const token = this.csrfService.getToken();
      request = request.clone({
        headers: request.headers.set('X-CSRF-Token', token)
      });
    }
    return next.handle(request);
  }

  private shouldAddCsrfToken(request: HttpRequest<any>): boolean {
    return request.method !== 'GET' &&
           !request.url.includes('/auth') &&
           !request.url.includes('/public');
  }
}

// CSRF Service
@Injectable({ providedIn: 'root' })
export class CsrfService {
  private readonly TOKEN_KEY = 'csrf_token';

  constructor(private http: HttpClient) {}

  getToken(): string | null {
    return localStorage.getItem(this.TOKEN_KEY);
  }

  setToken(token: string): void {
    localStorage.setItem(this.TOKEN_KEY, token);
  }

  refreshToken(): Observable<string> {
    return this.http.get<{ token: string }>('/api/csrf-token').pipe(
      map(response => {
        this.setToken(response.token);
        return response.token;
      })
    );
  }
}
```

#### Double Submit Cookie Pattern

```typescript
// double-submit-csrf.service.ts
@Injectable({ providedIn: 'root' })
export class DoubleSubmitCsrfService {
  private readonly COOKIE_NAME = 'csrf_token';
  private readonly HEADER_NAME = 'X-CSRF-Token';

  constructor(
    private http: HttpClient,
    private cookieService: CookieService
  ) {}

  generateToken(): string {
    const token = this.generateRandomToken();
    this.setCookie(token);
    return token;
  }

  validateToken(token: string): boolean {
    const cookieToken = this.getCookie();
    return token === cookieToken;
  }

  private generateRandomToken(): string {
    return crypto.randomUUID();
  }

  private setCookie(token: string): void {
    this.cookieService.set(this.COOKIE_NAME, token, {
      secure: true,
      sameSite: 'Strict',
      httpOnly: true
    });
  }

  private getCookie(): string | null {
    return this.cookieService.get(this.COOKIE_NAME);
  }
}

// Using in component
@Component({
  selector: 'app-form',
  template: `
    <form (ngSubmit)="onSubmit()">
      <input type="hidden" [value]="csrfToken">
      <!-- form fields -->
    </form>
  `
})
export class FormComponent {
  csrfToken: string;

  constructor(private csrfService: DoubleSubmitCsrfService) {
    this.csrfToken = this.csrfService.generateToken();
  }

  onSubmit(): void {
    // Form submission logic
  }
}
```

### 3. Advanced Authentication

#### JWT Authentication Service

```typescript
// jwt-auth.service.ts
@Injectable({ providedIn: 'root' })
export class JwtAuthService {
  private readonly TOKEN_KEY = 'jwt_token';
  private readonly REFRESH_TOKEN_KEY = 'refresh_token';
  private tokenExpirationTimer: any;

  constructor(
    private http: HttpClient,
    private router: Router
  ) {}

  login(credentials: LoginCredentials): Observable<AuthResponse> {
    return this.http.post<AuthResponse>('/api/auth/login', credentials).pipe(
      tap(response => {
        this.setTokens(response);
        this.startTokenExpirationTimer();
      })
    );
  }

  refreshToken(): Observable<AuthResponse> {
    const refreshToken = this.getRefreshToken();
    return this.http.post<AuthResponse>('/api/auth/refresh', { refreshToken }).pipe(
      tap(response => {
        this.setTokens(response);
        this.startTokenExpirationTimer();
      })
    );
  }

  logout(): void {
    this.clearTokens();
    this.stopTokenExpirationTimer();
    this.router.navigate(['/login']);
  }

  private setTokens(response: AuthResponse): void {
    localStorage.setItem(this.TOKEN_KEY, response.token);
    localStorage.setItem(this.REFRESH_TOKEN_KEY, response.refreshToken);
  }

  private clearTokens(): void {
    localStorage.removeItem(this.TOKEN_KEY);
    localStorage.removeItem(this.REFRESH_TOKEN_KEY);
  }

  private startTokenExpirationTimer(): void {
    const token = this.getToken();
    if (token) {
      const expirationTime = this.getTokenExpirationTime(token);
      const timeUntilExpiration = expirationTime - Date.now();
      
      this.tokenExpirationTimer = setTimeout(() => {
        this.refreshToken().subscribe();
      }, timeUntilExpiration);
    }
  }

  private stopTokenExpirationTimer(): void {
    if (this.tokenExpirationTimer) {
      clearTimeout(this.tokenExpirationTimer);
    }
  }

  private getToken(): string | null {
    return localStorage.getItem(this.TOKEN_KEY);
  }

  private getRefreshToken(): string | null {
    return localStorage.getItem(this.REFRESH_TOKEN_KEY);
  }

  private getTokenExpirationTime(token: string): number {
    const payload = JSON.parse(atob(token.split('.')[1]));
    return payload.exp * 1000;
  }
}
```

#### Token Interceptor

```typescript
// token.interceptor.ts
@Injectable()
export class TokenInterceptor implements HttpInterceptor {
  constructor(
    private authService: JwtAuthService,
    private router: Router
  ) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    const token = this.authService.getToken();
    
    if (token) {
      request = request.clone({
        headers: request.headers.set('Authorization', `Bearer ${token}`)
      });
    }

    return next.handle(request).pipe(
      catchError(error => {
        if (error instanceof HttpErrorResponse && error.status === 401) {
          return this.handle401Error(request, next);
        }
        return throwError(() => error);
      })
    );
  }

  private handle401Error(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    return this.authService.refreshToken().pipe(
      switchMap(() => {
        const token = this.authService.getToken();
        request = request.clone({
          headers: request.headers.set('Authorization', `Bearer ${token}`)
        });
        return next.handle(request);
      }),
      catchError(error => {
        this.authService.logout();
        return throwError(() => error);
      })
    );
  }
}
```

### 4. Advanced Authorization

#### Role-Based Access Control

```typescript
// auth.guard.ts
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean> | Promise<boolean> | boolean {
    const requiredRoles = route.data.roles as string[];
    
    if (!requiredRoles) {
      return true;
    }

    return this.authService.getUserRoles().pipe(
      map(roles => {
        const hasRequiredRole = requiredRoles.some(role =>
          roles.includes(role)
        );

        if (!hasRequiredRole) {
          this.router.navigate(['/unauthorized']);
        }

        return hasRequiredRole;
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
  @Input() appPermission: string | string[];

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private authService: AuthService
  ) {}

  ngOnInit(): void {
    this.checkPermission();
  }

  private checkPermission(): void {
    this.authService.getUserPermissions().pipe(
      take(1)
    ).subscribe(permissions => {
      const hasPermission = Array.isArray(this.appPermission)
        ? this.appPermission.some(permission =>
            permissions.includes(permission)
          )
        : permissions.includes(this.appPermission);

      if (hasPermission) {
        this.viewContainer.createEmbeddedView(this.templateRef);
      }
    });
  }
}

// Using in template
@Component({
  selector: 'app-feature',
  template: `
    <div *appPermission="'create'">
      <button (click)="create()">Create</button>
    </div>
    <div *appPermission="['edit', 'delete']">
      <button (click)="edit()">Edit</button>
      <button (click)="delete()">Delete</button>
    </div>
  `
})
export class FeatureComponent {}
```

### 5. Advanced Security Headers

```typescript
// security-headers.service.ts
@Injectable({ providedIn: 'root' })
export class SecurityHeadersService {
  private readonly securityHeaders = {
    'X-Content-Type-Options': 'nosniff',
    'X-Frame-Options': 'DENY',
    'X-XSS-Protection': '1; mode=block',
    'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
    'Referrer-Policy': 'strict-origin-when-cross-origin',
    'Permissions-Policy': 'geolocation=(), microphone=(), camera=()',
    'Content-Security-Policy': this.getCspHeader()
  };

  applySecurityHeaders(response: HttpResponse<any>): HttpResponse<any> {
    Object.entries(this.securityHeaders).forEach(([key, value]) => {
      response.headers.set(key, value);
    });
    return response;
  }

  private getCspHeader(): string {
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
      frame-src 'self';
      worker-src 'self' blob:;
      manifest-src 'self';
      upgrade-insecure-requests;
    `.replace(/\s+/g, ' ').trim();
  }
}
```

### 6. Advanced Input Validation

```typescript
// custom-validators.ts
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
      
      if (!hasUpperCase) errors.uppercase = true;
      if (!hasLowerCase) errors.lowercase = true;
      if (!hasNumeric) errors.numeric = true;
      if (!hasSpecialChar) errors.specialChar = true;
      if (!isLongEnough) errors.length = true;

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

// Using in form
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
  form: FormGroup;

  constructor(private fb: FormBuilder) {
    this.form = this.fb.group({
      username: ['', [Validators.required]],
      password: ['', [
        Validators.required,
        CustomValidators.passwordStrength()
      ]],
      content: ['', [
        Validators.required,
        CustomValidators.preventXss()
      ]]
    });
  }

  onSubmit(): void {
    if (this.form.valid) {
      // Form submission logic
    }
  }
}
```

### 7. Advanced Secure File Upload

```typescript
// file-upload.component.ts
@Component({
  selector: 'app-file-upload',
  template: `
    <input
      type="file"
      (change)="onFileSelected($event)"
      accept=".jpg,.jpeg,.png,.pdf"
      #fileInput
    >
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
    const input = event.target as HTMLInputElement;
    const file = input.files?.[0];

    if (file) {
      if (this.validateFile(file)) {
        this.selectedFile = file;
      } else {
        input.value = '';
        this.selectedFile = null;
      }
    }
  }

  private validateFile(file: File): boolean {
    if (file.size > this.MAX_FILE_SIZE) {
      alert('File size exceeds 5MB limit');
      return false;
    }

    if (!this.ALLOWED_TYPES.includes(file.type)) {
      alert('Invalid file type');
      return false;
    }

    return true;
  }

  uploadFile(): void {
    if (!this.selectedFile) return;

    const formData = new FormData();
    formData.append('file', this.selectedFile);

    this.http.post('/api/upload', formData, {
      headers: new HttpHeaders({
        'X-File-Size': this.selectedFile.size.toString(),
        'X-File-Type': this.selectedFile.type
      })
    }).subscribe({
      next: response => {
        console.log('Upload successful:', response);
      },
      error: error => {
        console.error('Upload failed:', error);
      }
    });
  }
}
```

### Conclusion

Key points to remember:

1. **XSS Protection**:
   - Use sanitizers
   - Implement CSP
   - Validate input
   - Escape output
   - Use security headers

2. **CSRF Protection**:
   - Use tokens
   - Implement double submit
   - Validate requests
   - Secure cookies
   - Handle token refresh

3. **Authentication**:
   - Use JWT
   - Implement refresh tokens
   - Secure storage
   - Handle expiration
   - Validate tokens

4. **Authorization**:
   - Implement RBAC
   - Use guards
   - Check permissions
   - Secure routes
   - Handle access control

5. **Security Headers**:
   - Set appropriate headers
   - Configure CSP
   - Enable HSTS
   - Prevent clickjacking
   - Control permissions

6. **Input Validation**:
   - Validate all inputs
   - Prevent XSS
   - Check file uploads
   - Sanitize data
   - Handle errors

7. **File Upload**:
   - Validate files
   - Check types
   - Limit size
   - Scan content
   - Secure storage

Remember to:
- Follow security best practices
- Regular security audits
- Monitor vulnerabilities
- Update dependencies
- Test security measures
- Document security
- Train developers
- Regular maintenance 