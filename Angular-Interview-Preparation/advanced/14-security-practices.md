# Advanced Angular Security Practices

## Question
What are the advanced security practices in Angular? Explain XSS protection, CSRF protection, and other security measures with examples.

## Answer

### Introduction to Advanced Angular Security

Angular provides robust security features to protect applications from common web vulnerabilities. Here's a comprehensive guide to advanced security practices.

### 1. Advanced XSS Protection

#### Custom Sanitizer

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
    this.sanitizedHtml = this.sanitizer.sanitizeHtml('<div>Safe HTML</div>');
    this.sanitizedUrl = this.sanitizer.sanitizeUrl('javascript:alert("xss")');
    this.sanitizedStyle = this.sanitizer.sanitizeStyle('background: url("javascript:alert(1)")');
  }
}
```

#### Content Security Policy

```typescript
// content-security-policy.service.ts
@Injectable({ providedIn: 'root' })
export class ContentSecurityPolicyService {
  private readonly CSP_HEADER = 'Content-Security-Policy';
  private readonly CSP_POLICY = `
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

  constructor(private http: HttpClient) {}

  applyCSP(): void {
    // Apply CSP header to all responses
    this.http.interceptors.push(
      new HttpInterceptor({
        intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
          const modifiedReq = req.clone({
            headers: req.headers.set(
              'Content-Security-Policy',
              this.CSP_POLICY
            )
          });
          return next.handle(modifiedReq);
        }
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
    if (this.isCsrfRequired(request)) {
      const token = this.getCsrfToken();
      if (token) {
        request = request.clone({
          headers: request.headers.set(this.CSRF_HEADER, token)
        });
      }
    }
    return next.handle(request);
  }

  private isCsrfRequired(request: HttpRequest<any>): boolean {
    return request.method !== 'GET' &&
           !request.headers.has(this.CSRF_HEADER) &&
           !this.isSameOrigin(request);
  }

  private getCsrfToken(): string | null {
    return this.cookieService.get(this.CSRF_COOKIE);
  }

  private isSameOrigin(request: HttpRequest<any>): boolean {
    const origin = request.headers.get('Origin');
    return origin === window.location.origin;
  }
}
```

#### Double Submit Cookie Pattern

```typescript
// csrf.service.ts
@Injectable({ providedIn: 'root' })
export class CsrfService {
  private readonly TOKEN_COOKIE = 'XSRF-TOKEN';
  private readonly TOKEN_HEADER = 'X-CSRF-Token';
  private readonly TOKEN_LENGTH = 32;

  constructor(
    private http: HttpClient,
    private cookieService: CookieService
  ) {}

  generateToken(): string {
    const token = this.generateRandomToken();
    this.setTokenCookie(token);
    return token;
  }

  validateToken(token: string): boolean {
    const cookieToken = this.cookieService.get(this.TOKEN_COOKIE);
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
      path: '/'
    });
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
    private http: HttpClient,
    private csrfService: CsrfService
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
    this.setAutoLogout(response.expiresIn);
  }

  private setAutoLogout(expiresIn: number): void {
    this.tokenExpirationTimer = setTimeout(() => {
      this.logout();
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
      catchError(error => this.handleError(error))
    );
  }

  private addToken(request: HttpRequest<any>, token: string): HttpRequest<any> {
    return request.clone({
      headers: request.headers.set('Authorization', `Bearer ${token}`)
    });
  }

  private handleError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    if (error.status === 401) {
      return this.authService.refreshToken().pipe(
        switchMap(response => {
          const newRequest = this.addToken(
            error.request,
            response.token
          );
          return next.handle(newRequest);
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

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): boolean | Observable<boolean> {
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
    path: 'dashboard',
    component: DashboardComponent,
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
  exportAs: 'appPermission'
})
export class PermissionDirective implements OnInit {
  @Input('appPermission') permission: string;
  @Input('appPermissionMode') mode: 'show' | 'hide' = 'show';

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private authService: AuthService
  ) {}

  ngOnInit(): void {
    this.authService.hasPermission(this.permission).subscribe(
      hasPermission => {
        if (this.mode === 'show' && hasPermission) {
          this.viewContainer.createEmbeddedView(this.templateRef);
        } else if (this.mode === 'hide' && !hasPermission) {
          this.viewContainer.createEmbeddedView(this.templateRef);
        }
      }
    );
  }
}

// Using in template
@Component({
  selector: 'app-content',
  template: `
    <div *appPermission="'edit'">
      <button (click)="edit()">Edit</button>
    </div>
    <div *appPermission="'delete'; mode: 'hide'">
      <button (click)="delete()">Delete</button>
    </div>
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
    'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
    'Referrer-Policy': 'strict-origin-when-cross-origin',
    'Permissions-Policy': 'geolocation=(), microphone=(), camera=()',
    'Content-Security-Policy': `
      default-src 'self';
      script-src 'self' 'unsafe-inline' 'unsafe-eval';
      style-src 'self' 'unsafe-inline';
      img-src 'self' data: https:;
      font-src 'self';
      connect-src 'self' https://api.example.com;
    `.replace(/\s+/g, ' ').trim()
  };

  constructor(private http: HttpClient) {}

  applySecurityHeaders(): void {
    this.http.interceptors.push(
      new HttpInterceptor({
        intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
          const modifiedReq = req.clone({
            headers: Object.entries(this.SECURITY_HEADERS).reduce(
              (headers, [key, value]) => headers.set(key, value),
              req.headers
            )
          });
          return next.handle(modifiedReq);
        }
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

  static preventXSS(): ValidatorFn {
    return (control: AbstractControl): ValidationErrors | null => {
      const value = control.value;
      if (!value) return null;

      const xssPattern = /<[^>]*>/g;
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
    username: ['', [Validators.required, CustomValidators.preventXSS()]],
    password: ['', [Validators.required, CustomValidators.passwordStrength()]],
    content: ['', [Validators.required, CustomValidators.preventXSS()]]
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
    <div class="upload-container">
      <input
        type="file"
        (change)="onFileSelected($event)"
        [accept]="allowedTypes"
        [max]="maxSize"
      >
      <div *ngIf="error" class="error">{{ error }}</div>
      <div *ngIf="uploading" class="progress">
        {{ progress }}%
      </div>
    </div>
  `
})
export class FileUploadComponent {
  @Input() allowedTypes: string = '.jpg,.jpeg,.png,.pdf';
  @Input() maxSize: number = 5 * 1024 * 1024; // 5MB

  error: string = '';
  uploading: boolean = false;
  progress: number = 0;

  constructor(
    private http: HttpClient,
    private sanitizer: DomSanitizer
  ) {}

  onFileSelected(event: Event): void {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (!file) return;

    if (!this.validateFile(file)) {
      return;
    }

    this.uploadFile(file);
  }

  private validateFile(file: File): boolean {
    // Check file type
    const extension = '.' + file.name.split('.').pop()?.toLowerCase();
    if (!this.allowedTypes.includes(extension)) {
      this.error = 'Invalid file type';
      return false;
    }

    // Check file size
    if (file.size > this.maxSize) {
      this.error = 'File too large';
      return false;
    }

    // Check file content
    const reader = new FileReader();
    reader.onload = (e) => {
      const content = e.target?.result as string;
      if (this.containsMaliciousContent(content)) {
        this.error = 'File contains malicious content';
      }
    };
    reader.readAsText(file);

    return true;
  }

  private containsMaliciousContent(content: string): boolean {
    const maliciousPatterns = [
      /<script>/i,
      /javascript:/i,
      /data:/i,
      /vbscript:/i
    ];
    return maliciousPatterns.some(pattern => pattern.test(content));
  }

  private uploadFile(file: File): void {
    const formData = new FormData();
    formData.append('file', file);

    this.uploading = true;
    this.error = '';

    this.http.post('/api/upload', formData, {
      reportProgress: true,
      observe: 'events'
    }).pipe(
      map(event => this.getUploadProgress(event)),
      catchError(error => {
        this.error = 'Upload failed';
        return throwError(() => error);
      }),
      finalize(() => {
        this.uploading = false;
      })
    ).subscribe();
  }

  private getUploadProgress(event: HttpEvent<any>): number {
    if (event.type === HttpEventType.UploadProgress) {
      const progress = Math.round(100 * event.loaded / event.total!);
      this.progress = progress;
      return progress;
    }
    return 0;
  }
}
```

### Conclusion

Key points to remember:

1. **XSS Protection**:
   - Use DomSanitizer
   - Implement CSP
   - Validate inputs
   - Escape output
   - Use security headers

2. **CSRF Protection**:
   - Use tokens
   - Validate requests
   - Set secure cookies
   - Implement double submit
   - Handle token refresh

3. **Authentication**:
   - Use JWT
   - Implement refresh tokens
   - Secure storage
   - Handle expiration
   - Validate tokens

4. **Authorization**:
   - Role-based access
   - Permission-based access
   - Route guards
   - Component guards
   - Directive-based control

5. **Security Headers**:
   - Set appropriate headers
   - Configure CSP
   - Enable HSTS
   - Control referrer
   - Set permissions

6. **Input Validation**:
   - Validate all inputs
   - Sanitize data
   - Prevent XSS
   - Check file uploads
   - Handle edge cases

Remember to:
- Keep dependencies updated
- Follow security best practices
- Regular security audits
- Monitor for vulnerabilities
- Implement logging
- Handle errors securely
- Use HTTPS
- Regular backups
- Security testing 