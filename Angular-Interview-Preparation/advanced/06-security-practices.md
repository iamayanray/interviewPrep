# Advanced Angular Security Practices

## Question
What are the advanced security practices in Angular? Explain XSS protection, CSRF protection, and other security measures with examples.

## Answer

### Introduction to Advanced Angular Security

Angular provides powerful security features to protect applications from common web vulnerabilities. Here's a comprehensive guide to advanced security practices.

### 1. Advanced XSS Protection

#### Custom Sanitizer

```typescript
// custom-sanitizer.ts
@Injectable({ providedIn: 'root' })
export class CustomSanitizer {
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

  constructor(private sanitizer: CustomSanitizer) {
    this.sanitizedHtml = this.sanitizer.sanitizeHtml('<div>Safe HTML</div>');
    this.sanitizedUrl = this.sanitizer.sanitizeUrl('javascript:alert(1)');
    this.sanitizedStyle = this.sanitizer.sanitizeStyle('background: url("javascript:alert(1)")');
  }
}
```

#### Content Security Policy

```typescript
// csp.service.ts
@Injectable({ providedIn: 'root' })
export class CSPService {
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
    `.replace(/\s+/g, ' ').trim()
  };

  applyCSPHeaders(res: Response): void {
    Object.entries(this.cspHeader).forEach(([key, value]) => {
      res.setHeader(key, value);
    });
  }
}

// Using in server
app.use((req, res, next) => {
  cspService.applyCSPHeaders(res);
  next();
});
```

### 2. Advanced CSRF Protection

#### Custom CSRF Interceptor

```typescript
// csrf.interceptor.ts
@Injectable()
export class CSRFInterceptor implements HttpInterceptor {
  constructor(private tokenService: TokenService) {}

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    if (this.shouldAddCSRFToken(request)) {
      const token = this.tokenService.getCSRFToken();
      request = request.clone({
        headers: request.headers.set('X-CSRF-Token', token)
      });
    }
    return next.handle(request);
  }

  private shouldAddCSRFToken(request: HttpRequest<any>): boolean {
    return request.method !== 'GET' && 
           !request.headers.has('X-CSRF-Token') &&
           !request.url.includes('public');
  }
}

// Token service
@Injectable({ providedIn: 'root' })
export class TokenService {
  private readonly TOKEN_KEY = 'csrf_token';

  getCSRFToken(): string | null {
    return localStorage.getItem(this.TOKEN_KEY);
  }

  setCSRFToken(token: string): void {
    localStorage.setItem(this.TOKEN_KEY, token);
  }

  removeCSRFToken(): void {
    localStorage.removeItem(this.TOKEN_KEY);
  }
}
```

#### Double Submit Cookie Pattern

```typescript
// double-submit.service.ts
@Injectable({ providedIn: 'root' })
export class DoubleSubmitService {
  private readonly COOKIE_NAME = 'csrf_token';
  private readonly HEADER_NAME = 'X-CSRF-Token';

  generateToken(): string {
    const token = crypto.randomBytes(32).toString('hex');
    this.setCookie(token);
    return token;
  }

  private setCookie(token: string): void {
    document.cookie = `${this.COOKIE_NAME}=${token}; SameSite=Strict; Secure`;
  }

  validateToken(headerToken: string): boolean {
    const cookieToken = this.getCookie(this.COOKIE_NAME);
    return cookieToken === headerToken;
  }

  private getCookie(name: string): string | null {
    const match = document.cookie.match(new RegExp('(^| )' + name + '=([^;]+)'));
    return match ? match[2] : null;
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

  constructor(
    private http: HttpClient,
    private router: Router,
    private tokenService: TokenService
  ) {}

  login(credentials: LoginCredentials): Observable<AuthResponse> {
    return this.http.post<AuthResponse>('/api/auth/login', credentials).pipe(
      tap(response => this.handleLoginResponse(response))
    );
  }

  refreshToken(): Observable<AuthResponse> {
    const refreshToken = this.tokenService.getRefreshToken();
    return this.http.post<AuthResponse>('/api/auth/refresh', { refreshToken }).pipe(
      tap(response => this.handleLoginResponse(response))
    );
  }

  private handleLoginResponse(response: AuthResponse): void {
    this.tokenService.setToken(response.token);
    this.tokenService.setRefreshToken(response.refreshToken);
  }

  logout(): void {
    this.tokenService.removeToken();
    this.tokenService.removeRefreshToken();
    this.router.navigate(['/login']);
  }
}

// Token interceptor
@Injectable()
export class TokenInterceptor implements HttpInterceptor {
  constructor(
    private authService: AuthService,
    private tokenService: TokenService
  ) {}

  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = this.tokenService.getToken();
    if (token) {
      request = request.clone({
        headers: request.headers.set('Authorization', `Bearer ${token}`)
      });
    }
    return next.handle(request).pipe(
      catchError(error => this.handleError(error))
    );
  }

  private handleError(error: HttpErrorResponse): Observable<HttpEvent<any>> {
    if (error.status === 401) {
      return this.authService.refreshToken().pipe(
        switchMap(() => {
          const token = this.tokenService.getToken();
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
  exportAs: 'permission'
})
export class PermissionDirective implements OnInit {
  @Input('appPermission') permission: string;
  @Input('appPermissionMode') mode: 'and' | 'or' = 'and';
  @Input('appPermissionFallback') fallback: TemplateRef<any>;

  private hasPermission = false;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private authService: AuthService
  ) {}

  ngOnInit() {
    this.checkPermission();
  }

  private checkPermission(): void {
    const userPermissions = this.authService.getUserPermissions();
    this.hasPermission = this.mode === 'and'
      ? this.permission.split(',').every(p => userPermissions.includes(p.trim()))
      : this.permission.split(',').some(p => userPermissions.includes(p.trim()));

    this.updateView();
  }

  private updateView(): void {
    this.viewContainer.clear();
    if (this.hasPermission) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    } else if (this.fallback) {
      this.viewContainer.createEmbeddedView(this.fallback);
    }
  }
}

// Using in template
@Component({
  selector: 'app-content',
  template: `
    <div *appPermission="'READ,WRITE'">
      <h2>Protected Content</h2>
      <p>This content is only visible to users with READ and WRITE permissions.</p>
    </div>

    <div *appPermission="'ADMIN'; mode: 'or'">
      <h2>Admin Content</h2>
      <p>This content is visible to users with ADMIN permission.</p>
    </div>

    <ng-template #fallback>
      <p>You don't have permission to view this content.</p>
    </ng-template>
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
  private readonly securityHeaders = {
    'X-Content-Type-Options': 'nosniff',
    'X-Frame-Options': 'DENY',
    'X-XSS-Protection': '1; mode=block',
    'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
    'Referrer-Policy': 'strict-origin-when-cross-origin',
    'Permissions-Policy': 'geolocation=(), microphone=(), camera=()',
    'Cross-Origin-Embedder-Policy': 'require-corp',
    'Cross-Origin-Opener-Policy': 'same-origin',
    'Cross-Origin-Resource-Policy': 'same-origin'
  };

  applySecurityHeaders(res: Response): void {
    Object.entries(this.securityHeaders).forEach(([key, value]) => {
      res.setHeader(key, value);
    });
  }
}

// Using in server
app.use((req, res, next) => {
  securityHeadersService.applySecurityHeaders(res);
  next();
});
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

  static noXSS(): ValidatorFn {
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
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <input formControlName="username">
      <input formControlName="password" type="password">
      <textarea formControlName="content"></textarea>
      <button type="submit">Submit</button>
    </form>
  `
})
export class FormComponent {
  userForm = this.fb.group({
    username: ['', [Validators.required, CustomValidators.noXSS()]],
    password: ['', [Validators.required, CustomValidators.passwordStrength()]],
    content: ['', [Validators.required, CustomValidators.noXSS()]]
  });
}
```

### Conclusion

Key points to remember:

1. **XSS Protection**:
   - Use DomSanitizer
   - Implement CSP
   - Sanitize user input
   - Escape output

2. **CSRF Protection**:
   - Use tokens
   - Implement double submit
   - Validate requests
   - Secure cookies

3. **Authentication**:
   - Use JWT
   - Implement refresh tokens
   - Secure storage
   - Handle token expiration

4. **Authorization**:
   - Implement RBAC
   - Use permission directives
   - Protect routes
   - Validate access

5. **Security Headers**:
   - Set appropriate headers
   - Configure CSP
   - Enable HSTS
   - Control referrer policy

Remember to:
- Follow security best practices
- Keep dependencies updated
- Implement proper validation
- Use secure communication
- Handle sensitive data
- Monitor security events
- Regular security audits
- Stay informed about vulnerabilities 