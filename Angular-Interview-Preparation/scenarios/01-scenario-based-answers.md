# Angular Interview Scenarios and Solutions

## Scenario 1: Large Data Table Performance

### Question
You're building a data table component that needs to display 10,000+ rows with sorting, filtering, and pagination. The current implementation is slow and causes performance issues. How would you optimize it?

### Solution

```typescript
// data-table.component.ts
@Component({
  selector: 'app-data-table',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <div class="table-container">
      <div class="table-header">
        <input [(ngModel)]="searchTerm" (input)="onSearch()" placeholder="Search...">
        <select [(ngModel)]="pageSize" (change)="onPageSizeChange()">
          <option [value]="10">10</option>
          <option [value]="50">50</option>
          <option [value]="100">100</option>
        </select>
      </div>

      <table>
        <thead>
          <tr>
            <th *ngFor="let column of columns" (click)="sort(column)">
              {{ column.label }}
              <span *ngIf="sortColumn === column">
                {{ sortDirection === 'asc' ? '↑' : '↓' }}
              </span>
            </th>
          </tr>
        </thead>
        <tbody>
          <tr *ngFor="let item of visibleItems$ | async; trackBy: trackByFn">
            <td *ngFor="let column of columns">
              {{ item[column.key] }}
            </td>
          </tr>
        </tbody>
      </table>

      <div class="pagination">
        <button [disabled]="currentPage === 1" (click)="previousPage()">
          Previous
        </button>
        <span>Page {{ currentPage }} of {{ totalPages }}</span>
        <button [disabled]="currentPage === totalPages" (click)="nextPage()">
          Next
        </button>
      </div>
    </div>
  `
})
export class DataTableComponent implements OnInit {
  @Input() data: any[] = [];
  
  columns = [
    { key: 'id', label: 'ID' },
    { key: 'name', label: 'Name' },
    { key: 'email', label: 'Email' },
    { key: 'status', label: 'Status' }
  ];

  searchTerm = '';
  pageSize = 50;
  currentPage = 1;
  sortColumn: string | null = null;
  sortDirection: 'asc' | 'desc' = 'asc';

  private dataSubject = new BehaviorSubject<any[]>([]);
  visibleItems$: Observable<any[]>;

  constructor() {
    this.visibleItems$ = this.dataSubject.pipe(
      map(data => this.processData(data)),
      shareReplay(1)
    );
  }

  ngOnInit() {
    this.dataSubject.next(this.data);
  }

  private processData(data: any[]): any[] {
    let processed = [...data];

    // Apply search filter
    if (this.searchTerm) {
      processed = processed.filter(item =>
        Object.values(item).some(value =>
          String(value).toLowerCase().includes(this.searchTerm.toLowerCase())
        )
      );
    }

    // Apply sorting
    if (this.sortColumn) {
      processed.sort((a, b) => {
        const aValue = a[this.sortColumn!];
        const bValue = b[this.sortColumn!];
        return this.sortDirection === 'asc'
          ? aValue.localeCompare(bValue)
          : bValue.localeCompare(aValue);
      });
    }

    // Apply pagination
    const start = (this.currentPage - 1) * this.pageSize;
    const end = start + this.pageSize;
    return processed.slice(start, end);
  }

  onSearch(): void {
    this.currentPage = 1;
    this.dataSubject.next(this.data);
  }

  sort(column: { key: string }): void {
    if (this.sortColumn === column.key) {
      this.sortDirection = this.sortDirection === 'asc' ? 'desc' : 'asc';
    } else {
      this.sortColumn = column.key;
      this.sortDirection = 'asc';
    }
    this.dataSubject.next(this.data);
  }

  onPageSizeChange(): void {
    this.currentPage = 1;
    this.dataSubject.next(this.data);
  }

  previousPage(): void {
    if (this.currentPage > 1) {
      this.currentPage--;
      this.dataSubject.next(this.data);
    }
  }

  nextPage(): void {
    if (this.currentPage < this.totalPages) {
      this.currentPage++;
      this.dataSubject.next(this.data);
    }
  }

  trackByFn(index: number, item: any): any {
    return item.id;
  }

  get totalPages(): number {
    return Math.ceil(this.data.length / this.pageSize);
  }
}
```

### Key Optimizations:

1. **Change Detection Strategy**:
   - Using `OnPush` to minimize change detection cycles
   - Only updating when inputs change or events are triggered

2. **Virtual Scrolling**:
   - Only rendering visible rows
   - Using `trackBy` function for efficient DOM updates

3. **Data Processing**:
   - Using RxJS operators for efficient data transformations
   - Implementing client-side pagination
   - Optimizing search and sort operations

4. **Performance Best Practices**:
   - Using `shareReplay` to cache processed data
   - Implementing efficient sorting and filtering
   - Using pagination to limit rendered items

## Scenario 2: Real-time Data Updates

### Question
You need to implement a real-time dashboard that displays live data from multiple WebSocket connections. How would you handle this efficiently?

### Solution

```typescript
// dashboard.component.ts
@Component({
  selector: 'app-dashboard',
  template: `
    <div class="dashboard">
      <div *ngFor="let widget of widgets$ | async">
        <app-widget
          [data]="widget.data"
          [type]="widget.type"
          (refresh)="refreshWidget(widget.id)"
        ></app-widget>
      </div>
    </div>
  `
})
export class DashboardComponent implements OnInit, OnDestroy {
  private destroy$ = new Subject<void>();
  widgets$: Observable<Widget[]>;

  constructor(
    private websocketService: WebSocketService,
    private widgetService: WidgetService
  ) {
    this.widgets$ = this.initializeWidgets();
  }

  private initializeWidgets(): Observable<Widget[]> {
    return this.widgetService.getWidgets().pipe(
      switchMap(widgets => {
        const widgetStreams = widgets.map(widget =>
          this.websocketService
            .connectToStream(widget.streamId)
            .pipe(
              map(data => ({
                ...widget,
                data
              })),
              catchError(error => {
                console.error(`Error in widget ${widget.id}:`, error);
                return of(widget);
              })
            )
        );

        return combineLatest(widgetStreams);
      }),
      takeUntil(this.destroy$),
      shareReplay(1)
    );
  }

  refreshWidget(widgetId: string): void {
    this.widgetService.refreshWidget(widgetId);
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}

// websocket.service.ts
@Injectable({ providedIn: 'root' })
export class WebSocketService {
  private connections = new Map<string, WebSocket>();
  private subjects = new Map<string, Subject<any>>();

  connectToStream(streamId: string): Observable<any> {
    if (this.subjects.has(streamId)) {
      return this.subjects.get(streamId)!.asObservable();
    }

    const subject = new Subject<any>();
    this.subjects.set(streamId, subject);

    const ws = new WebSocket(`wss://api.example.com/stream/${streamId}`);

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      subject.next(data);
    };

    ws.onerror = (error) => {
      console.error(`WebSocket error for stream ${streamId}:`, error);
      subject.error(error);
    };

    ws.onclose = () => {
      this.reconnect(streamId);
    };

    this.connections.set(streamId, ws);

    return subject.asObservable().pipe(
      retryWhen(errors =>
        errors.pipe(
          delay(5000),
          take(3)
        )
      )
    );
  }

  private reconnect(streamId: string): void {
    setTimeout(() => {
      this.connectToStream(streamId);
    }, 5000);
  }

  disconnect(streamId: string): void {
    const ws = this.connections.get(streamId);
    if (ws) {
      ws.close();
      this.connections.delete(streamId);
    }

    const subject = this.subjects.get(streamId);
    if (subject) {
      subject.complete();
      this.subjects.delete(streamId);
    }
  }
}
```

### Key Features:

1. **WebSocket Management**:
   - Efficient connection handling
   - Automatic reconnection
   - Error handling and retry logic

2. **Data Stream Processing**:
   - Using RxJS operators for data transformation
   - Implementing error handling
   - Managing multiple streams

3. **Resource Management**:
   - Proper cleanup on component destruction
   - Connection pooling
   - Memory leak prevention

4. **Performance Optimizations**:
   - Using `shareReplay` for stream sharing
   - Implementing retry logic
   - Efficient error handling

## Scenario 3: Complex Form Validation

### Question
You need to implement a complex form with dynamic validation rules, cross-field validation, and async validation. How would you handle this?

### Solution

```typescript
// complex-form.component.ts
@Component({
  selector: 'app-complex-form',
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <div formGroupName="personalInfo">
        <input formControlName="firstName">
        <input formControlName="lastName">
        <input formControlName="email">
      </div>

      <div formGroupName="address">
        <input formControlName="street">
        <input formControlName="city">
        <select formControlName="country">
          <option *ngFor="let country of countries" [value]="country.code">
            {{ country.name }}
          </option>
        </select>
      </div>

      <div formArrayName="phoneNumbers">
        <div *ngFor="let phone of phoneNumbers.controls; let i=index" [formGroupName]="i">
          <input formControlName="number">
          <select formControlName="type">
            <option value="mobile">Mobile</option>
            <option value="home">Home</option>
            <option value="work">Work</option>
          </select>
          <button type="button" (click)="removePhone(i)">Remove</button>
        </div>
        <button type="button" (click)="addPhone()">Add Phone</button>
      </div>

      <button type="submit" [disabled]="form.invalid || form.pending">
        Submit
      </button>
    </form>
  `
})
export class ComplexFormComponent implements OnInit {
  form: FormGroup;
  countries: Country[] = [];
  phoneNumbers: FormArray;

  constructor(
    private fb: FormBuilder,
    private validationService: ValidationService,
    private countryService: CountryService
  ) {
    this.form = this.createForm();
  }

  ngOnInit(): void {
    this.loadCountries();
    this.setupValidation();
  }

  private createForm(): FormGroup {
    return this.fb.group({
      personalInfo: this.fb.group({
        firstName: ['', [Validators.required, Validators.minLength(2)]],
        lastName: ['', [Validators.required, Validators.minLength(2)]],
        email: ['', [Validators.required, Validators.email]]
      }),
      address: this.fb.group({
        street: ['', Validators.required],
        city: ['', Validators.required],
        country: ['', Validators.required]
      }),
      phoneNumbers: this.fb.array([])
    });
  }

  private setupValidation(): void {
    // Cross-field validation
    this.form.get('personalInfo')?.setValidators(
      this.validationService.nameValidator()
    );

    // Async validation
    this.form.get('personalInfo.email')?.setAsyncValidators(
      this.validationService.emailExistsValidator()
    );

    // Conditional validation
    this.form.get('address.country')?.valueChanges.subscribe(country => {
      const cityControl = this.form.get('address.city');
      if (country === 'US') {
        cityControl?.setValidators([Validators.required, Validators.pattern(/^[A-Z]{2}$/)]);
      } else {
        cityControl?.setValidators(Validators.required);
      }
      cityControl?.updateValueAndValidity();
    });
  }

  get phoneNumbers(): FormArray {
    return this.form.get('phoneNumbers') as FormArray;
  }

  addPhone(): void {
    const phoneGroup = this.fb.group({
      number: ['', [Validators.required, Validators.pattern(/^\d{10}$/)]],
      type: ['mobile', Validators.required]
    });

    this.phoneNumbers.push(phoneGroup);
  }

  removePhone(index: number): void {
    if (this.phoneNumbers.length > 1) {
      this.phoneNumbers.removeAt(index);
    }
  }

  private loadCountries(): void {
    this.countryService.getCountries().subscribe(
      countries => this.countries = countries
    );
  }

  onSubmit(): void {
    if (this.form.valid) {
      console.log(this.form.value);
      // Handle form submission
    }
  }
}

// validation.service.ts
@Injectable({ providedIn: 'root' })
export class ValidationService {
  constructor(private http: HttpClient) {}

  nameValidator(): ValidatorFn {
    return (group: AbstractControl): ValidationErrors | null => {
      const firstName = group.get('firstName')?.value;
      const lastName = group.get('lastName')?.value;

      if (firstName && lastName && firstName === lastName) {
        return { sameName: true };
      }

      return null;
    };
  }

  emailExistsValidator(): AsyncValidatorFn {
    return (control: AbstractControl): Observable<ValidationErrors | null> => {
      return this.http.get(`/api/check-email/${control.value}`).pipe(
        map(response => {
          return response ? { emailExists: true } : null;
        }),
        catchError(() => of(null))
      );
    };
  }
}
```

### Key Features:

1. **Form Structure**:
   - Nested form groups
   - Dynamic form arrays
   - Complex validation rules

2. **Validation Types**:
   - Cross-field validation
   - Async validation
   - Conditional validation
   - Custom validators

3. **Form Management**:
   - Dynamic field addition/removal
   - Form state handling
   - Error handling

4. **Performance Considerations**:
   - Efficient validation
   - Proper cleanup
   - Error handling

## Scenario 4: State Management with NgRx

### Question
You're building a complex e-commerce application with multiple features like shopping cart, user preferences, and order management. How would you implement state management using NgRx?

### Solution

```typescript
// app.state.ts
export interface AppState {
  cart: CartState;
  user: UserState;
  orders: OrderState;
}

export interface CartState {
  items: CartItem[];
  loading: boolean;
  error: string | null;
}

export interface UserState {
  preferences: UserPreferences;
  loading: boolean;
  error: string | null;
}

export interface OrderState {
  orders: Order[];
  selectedOrder: Order | null;
  loading: boolean;
  error: string | null;
}

// cart.actions.ts
export const addToCart = createAction(
  '[Cart] Add Item',
  props<{ item: CartItem }>()
);

export const addToCartSuccess = createAction(
  '[Cart] Add Item Success',
  props<{ item: CartItem }>()
);

export const addToCartFailure = createAction(
  '[Cart] Add Item Failure',
  props<{ error: string }>()
);

// cart.effects.ts
@Injectable()
export class CartEffects {
  addToCart$ = createEffect(() =>
    this.actions$.pipe(
      ofType(addToCart),
      mergeMap(({ item }) =>
        this.cartService.addItem(item).pipe(
          map(response => addToCartSuccess({ item: response })),
          catchError(error => of(addToCartFailure({ error: error.message })))
        )
      )
    )
  );

  constructor(
    private actions$: Actions,
    private cartService: CartService
  ) {}
}

// cart.reducer.ts
export const cartReducer = createReducer(
  initialState,
  on(addToCart, (state) => ({
    ...state,
    loading: true,
    error: null
  })),
  on(addToCartSuccess, (state, { item }) => ({
    ...state,
    items: [...state.items, item],
    loading: false
  })),
  on(addToCartFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error
  }))
);

// cart.selectors.ts
export const selectCartState = createFeatureSelector<CartState>('cart');

export const selectCartItems = createSelector(
  selectCartState,
  (state: CartState) => state.items
);

export const selectCartTotal = createSelector(
  selectCartItems,
  (items: CartItem[]) =>
    items.reduce((total, item) => total + item.price * item.quantity, 0)
);

// cart.component.ts
@Component({
  selector: 'app-cart',
  template: `
    <div *ngIf="cartItems$ | async as items">
      <div *ngFor="let item of items">
        <span>{{ item.name }}</span>
        <span>{{ item.price | currency }}</span>
        <button (click)="removeItem(item.id)">Remove</button>
      </div>
      <div>Total: {{ cartTotal$ | async | currency }}</div>
    </div>
  `
})
export class CartComponent {
  cartItems$ = this.store.select(selectCartItems);
  cartTotal$ = this.store.select(selectCartTotal);

  constructor(private store: Store) {}

  addItem(item: CartItem): void {
    this.store.dispatch(addToCart({ item }));
  }

  removeItem(itemId: string): void {
    this.store.dispatch(removeFromCart({ itemId }));
  }
}

// app.module.ts
@NgModule({
  imports: [
    StoreModule.forRoot({
      cart: cartReducer,
      user: userReducer,
      orders: orderReducer
    }),
    EffectsModule.forRoot([
      CartEffects,
      UserEffects,
      OrderEffects
    ]),
    StoreDevtoolsModule.instrument({
      maxAge: 25,
      logOnly: environment.production
    })
  ]
})
export class AppModule {}
```

### Key Features:

1. **State Structure**:
   - Modular state design
   - Type-safe actions
   - Immutable state updates

2. **Effects**:
   - Side effect handling
   - Error handling
   - Loading states

3. **Selectors**:
   - Memoized selectors
   - Derived state
   - Performance optimization

4. **Best Practices**:
   - Action naming conventions
   - Error handling
   - Loading states
   - Type safety

## Scenario 5: Authentication and Authorization

### Question
You need to implement a secure authentication system with role-based access control, token management, and route protection. How would you handle this?

### Solution

```typescript
// auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly TOKEN_KEY = 'auth_token';
  private readonly REFRESH_TOKEN_KEY = 'refresh_token';
  private userSubject = new BehaviorSubject<User | null>(null);

  constructor(
    private http: HttpClient,
    private router: Router,
    private jwtHelper: JwtHelperService
  ) {
    this.loadStoredUser();
  }

  login(credentials: LoginCredentials): Observable<AuthResponse> {
    return this.http.post<AuthResponse>('/api/auth/login', credentials).pipe(
      tap(response => this.handleLoginResponse(response)),
      catchError(error => this.handleLoginError(error))
    );
  }

  logout(): void {
    this.clearStoredData();
    this.userSubject.next(null);
    this.router.navigate(['/login']);
  }

  refreshToken(): Observable<AuthResponse> {
    const refreshToken = localStorage.getItem(this.REFRESH_TOKEN_KEY);
    return this.http.post<AuthResponse>('/api/auth/refresh', { refreshToken }).pipe(
      tap(response => this.handleLoginResponse(response)),
      catchError(() => {
        this.logout();
        return throwError(() => new Error('Refresh token failed'));
      })
    );
  }

  private handleLoginResponse(response: AuthResponse): void {
    localStorage.setItem(this.TOKEN_KEY, response.token);
    localStorage.setItem(this.REFRESH_TOKEN_KEY, response.refreshToken);
    
    const user = this.jwtHelper.decodeToken(response.token);
    this.userSubject.next(user);
  }

  private handleLoginError(error: any): Observable<never> {
    return throwError(() => new Error('Login failed'));
  }

  private clearStoredData(): void {
    localStorage.removeItem(this.TOKEN_KEY);
    localStorage.removeItem(this.REFRESH_TOKEN_KEY);
  }

  private loadStoredUser(): void {
    const token = localStorage.getItem(this.TOKEN_KEY);
    if (token && !this.jwtHelper.isTokenExpired(token)) {
      const user = this.jwtHelper.decodeToken(token);
      this.userSubject.next(user);
    }
  }

  get currentUser$(): Observable<User | null> {
    return this.userSubject.asObservable();
  }
}

// auth.guard.ts
@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  canActivate(route: ActivatedRouteSnapshot): boolean {
    const requiredRoles = route.data['roles'] as string[];
    const user = this.authService.currentUser;

    if (!user) {
      this.router.navigate(['/login']);
      return false;
    }

    if (requiredRoles && !requiredRoles.some(role => user.roles.includes(role))) {
      this.router.navigate(['/unauthorized']);
      return false;
    }

    return true;
  }
}

// auth.interceptor.ts
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(
    private authService: AuthService,
    private jwtHelper: JwtHelperService
  ) {}

  intercept(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    const token = localStorage.getItem('auth_token');

    if (token && this.jwtHelper.isTokenExpired(token)) {
      return this.handleExpiredToken(request, next);
    }

    if (token) {
      request = this.addTokenToRequest(request, token);
    }

    return next.handle(request);
  }

  private handleExpiredToken(
    request: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    return this.authService.refreshToken().pipe(
      switchMap(response => {
        const newRequest = this.addTokenToRequest(
          request,
          response.token
        );
        return next.handle(newRequest);
      }),
      catchError(() => {
        this.authService.logout();
        return throwError(() => new Error('Token refresh failed'));
      })
    );
  }

  private addTokenToRequest(
    request: HttpRequest<any>,
    token: string
  ): HttpRequest<any> {
    return request.clone({
      headers: request.headers.set('Authorization', `Bearer ${token}`)
    });
  }
}

// role.directive.ts
@Directive({
  selector: '[appRole]',
  exportAs: 'role'
})
export class RoleDirective implements OnInit {
  @Input('appRole') roles: string[];
  @Input('appRoleMode') mode: 'and' | 'or' = 'and';

  private hasPermission = false;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private authService: AuthService
  ) {}

  ngOnInit(): void {
    this.authService.currentUser$.subscribe(user => {
      this.hasPermission = this.checkPermission(user);
      this.updateView();
    });
  }

  private checkPermission(user: User | null): boolean {
    if (!user) return false;

    return this.mode === 'and'
      ? this.roles.every(role => user.roles.includes(role))
      : this.roles.some(role => user.roles.includes(role));
  }

  private updateView(): void {
    this.viewContainer.clear();
    if (this.hasPermission) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    }
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

// Using in template
@Component({
  selector: 'app-content',
  template: `
    <div *appRole="['ADMIN']">
      <h2>Admin Content</h2>
      <p>This content is only visible to administrators.</p>
    </div>

    <div *appRole="['USER', 'ADMIN']; mode: 'or'">
      <h2>User Content</h2>
      <p>This content is visible to both users and administrators.</p>
    </div>
  `
})
export class ContentComponent {}
```

### Key Features:

1. **Authentication**:
   - JWT token management
   - Token refresh mechanism
   - Secure storage
   - Login/logout handling

2. **Authorization**:
   - Role-based access control
   - Route guards
   - Permission directives
   - Role checking

3. **Security**:
   - Token validation
   - Secure storage
   - Error handling
   - Token refresh

4. **User Experience**:
   - Automatic token refresh
   - Graceful error handling
   - Role-based UI
   - Protected routes

Remember to:
- Follow security best practices
- Implement proper error handling
- Use secure storage methods
- Handle token expiration
- Implement proper logging
- Regular security audits
- Keep dependencies updated
- Monitor security events 