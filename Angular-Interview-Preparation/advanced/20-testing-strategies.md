# Advanced Angular Testing Strategies

## Question
What are the advanced testing strategies in Angular? Explain component testing, service testing, and end-to-end testing with examples.

## Answer

### Introduction to Advanced Angular Testing Strategies

Angular provides powerful testing capabilities to ensure application reliability and maintainability. Here's a comprehensive guide to advanced testing strategies.

### 1. Advanced Component Testing

#### Testing Component with NgRx Store

```typescript
// user-list.component.spec.ts
describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let store: MockStore;
  let mockUsers: User[];

  beforeEach(async () => {
    mockUsers = [
      { id: 1, name: 'John Doe', email: 'john@example.com' },
      { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
    ];

    await TestBed.configureTestingModule({
      declarations: [UserListComponent],
      imports: [
        StoreModule.forRoot({}),
        EffectsModule.forRoot([])
      ],
      providers: [
        provideMockStore({
          initialState: {
            users: {
              list: mockUsers,
              loading: false,
              error: null
            }
          }
        })
      ]
    }).compileComponents();

    store = TestBed.inject(MockStore);
    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
  });

  it('should load users on init', () => {
    // Arrange
    const dispatchSpy = spyOn(store, 'dispatch');

    // Act
    component.ngOnInit();

    // Assert
    expect(dispatchSpy).toHaveBeenCalledWith(loadUsers());
  });

  it('should display users when loaded', () => {
    // Arrange
    store.setState({
      users: {
        list: mockUsers,
        loading: false,
        error: null
      }
    });

    // Act
    fixture.detectChanges();

    // Assert
    const userElements = fixture.debugElement.queryAll(By.css('.user-item'));
    expect(userElements.length).toBe(mockUsers.length);
    expect(userElements[0].nativeElement.textContent).toContain('John Doe');
  });

  it('should show loading state', () => {
    // Arrange
    store.setState({
      users: {
        list: [],
        loading: true,
        error: null
      }
    });

    // Act
    fixture.detectChanges();

    // Assert
    const loadingElement = fixture.debugElement.query(By.css('.loading'));
    expect(loadingElement).toBeTruthy();
  });

  it('should show error state', () => {
    // Arrange
    const errorMessage = 'Failed to load users';
    store.setState({
      users: {
        list: [],
        loading: false,
        error: errorMessage
      }
    });

    // Act
    fixture.detectChanges();

    // Assert
    const errorElement = fixture.debugElement.query(By.css('.error'));
    expect(errorElement.nativeElement.textContent).toContain(errorMessage);
  });
});
```

#### Testing Component with Complex Dependencies

```typescript
// dashboard.component.spec.ts
describe('DashboardComponent', () => {
  let component: DashboardComponent;
  let fixture: ComponentFixture<DashboardComponent>;
  let mockUserService: jasmine.SpyObj<UserService>;
  let mockAnalyticsService: jasmine.SpyObj<AnalyticsService>;
  let mockNotificationService: jasmine.SpyObj<NotificationService>;

  beforeEach(async () => {
    mockUserService = jasmine.createSpyObj('UserService', ['getUserStats']);
    mockAnalyticsService = jasmine.createSpyObj('AnalyticsService', ['trackEvent']);
    mockNotificationService = jasmine.createSpyObj('NotificationService', ['showError']);

    await TestBed.configureTestingModule({
      declarations: [DashboardComponent],
      providers: [
        { provide: UserService, useValue: mockUserService },
        { provide: AnalyticsService, useValue: mockAnalyticsService },
        { provide: NotificationService, useValue: mockNotificationService }
      ]
    }).compileComponents();

    fixture = TestBed.createComponent(DashboardComponent);
    component = fixture.componentInstance;
  });

  it('should load user statistics on init', () => {
    // Arrange
    const mockStats = {
      totalUsers: 100,
      activeUsers: 75,
      newUsers: 10
    };
    mockUserService.getUserStats.and.returnValue(of(mockStats));

    // Act
    component.ngOnInit();

    // Assert
    expect(mockUserService.getUserStats).toHaveBeenCalled();
    expect(component.stats).toEqual(mockStats);
  });

  it('should handle error when loading statistics', () => {
    // Arrange
    const error = new Error('Failed to load statistics');
    mockUserService.getUserStats.and.returnValue(throwError(() => error));

    // Act
    component.ngOnInit();

    // Assert
    expect(mockNotificationService.showError).toHaveBeenCalledWith(
      'Failed to load dashboard statistics'
    );
  });

  it('should track analytics events', () => {
    // Arrange
    const mockStats = {
      totalUsers: 100,
      activeUsers: 75,
      newUsers: 10
    };
    mockUserService.getUserStats.and.returnValue(of(mockStats));

    // Act
    component.ngOnInit();

    // Assert
    expect(mockAnalyticsService.trackEvent).toHaveBeenCalledWith(
      'dashboard_viewed',
      { totalUsers: 100 }
    );
  });
});
```

### 2. Advanced Service Testing

#### Testing Service with HTTP Client

```typescript
// user.service.spec.ts
describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;
  let mockUsers: User[];

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
    mockUsers = [
      { id: 1, name: 'John Doe', email: 'john@example.com' },
      { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
    ];
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should create a user', () => {
    // Arrange
    const newUser = { name: 'New User', email: 'new@example.com' };
    const expectedResponse = { ...newUser, id: 3 };

    // Act
    service.createUser(newUser).subscribe(response => {
      expect(response).toEqual(expectedResponse);
    });

    // Assert
    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(newUser);
    req.flush(expectedResponse);
  });

  it('should handle error response', () => {
    // Arrange
    const newUser = { name: 'New User', email: 'new@example.com' };
    const errorResponse = {
      status: 400,
      statusText: 'Bad Request',
      error: { message: 'Invalid email format' }
    };

    // Act
    service.createUser(newUser).subscribe({
      error: error => {
        expect(error.status).toBe(400);
        expect(error.error.message).toBe('Invalid email format');
      }
    });

    // Assert
    const req = httpMock.expectOne('/api/users');
    req.flush(errorResponse, errorResponse);
  });

  it('should retry failed requests', () => {
    // Arrange
    const newUser = { name: 'New User', email: 'new@example.com' };
    const expectedResponse = { ...newUser, id: 3 };

    // Act
    service.createUser(newUser).subscribe(response => {
      expect(response).toEqual(expectedResponse);
    });

    // Assert
    const reqs = httpMock.match('/api/users');
    expect(reqs.length).toBe(3); // Initial request + 2 retries
    reqs[0].error(new ErrorEvent('Network error'));
    reqs[1].error(new ErrorEvent('Network error'));
    reqs[2].flush(expectedResponse);
  });
});
```

#### Testing Service with RxJS Operators

```typescript
// data.service.spec.ts
describe('DataService', () => {
  let service: DataService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [DataService]
    });

    service = TestBed.inject(DataService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should cache data and return cached data for subsequent requests', () => {
    // Arrange
    const mockData = { id: 1, name: 'Test Data' };

    // Act
    service.getData(1).subscribe();
    service.getData(1).subscribe(response => {
      expect(response).toEqual(mockData);
    });

    // Assert
    const req = httpMock.expectOne('/api/data/1');
    req.flush(mockData);
  });

  it('should refresh cache when requested', () => {
    // Arrange
    const mockData = { id: 1, name: 'Test Data' };
    const updatedData = { id: 1, name: 'Updated Data' };

    // Act
    service.getData(1).subscribe();
    service.refreshData(1).subscribe(response => {
      expect(response).toEqual(updatedData);
    });

    // Assert
    const reqs = httpMock.match('/api/data/1');
    expect(reqs.length).toBe(2);
    reqs[0].flush(mockData);
    reqs[1].flush(updatedData);
  });

  it('should handle concurrent requests', () => {
    // Arrange
    const mockData = { id: 1, name: 'Test Data' };

    // Act
    const requests = [
      service.getData(1),
      service.getData(1),
      service.getData(1)
    ];

    // Assert
    const req = httpMock.expectOne('/api/data/1');
    req.flush(mockData);

    forkJoin(requests).subscribe(responses => {
      expect(responses.every(r => r === mockData)).toBeTrue();
    });
  });
});
```

### 3. Advanced E2E Testing with Cypress

#### Testing Complex User Flows

```typescript
// user-management.cy.ts
describe('User Management', () => {
  beforeEach(() => {
    cy.login('admin', 'password');
    cy.visit('/users');
  });

  it('should create a new user', () => {
    // Arrange
    const newUser = {
      name: 'New User',
      email: 'new@example.com',
      role: 'user'
    };

    // Act
    cy.get('[data-testid="create-user-button"]').click();
    cy.get('[data-testid="name-input"]').type(newUser.name);
    cy.get('[data-testid="email-input"]').type(newUser.email);
    cy.get('[data-testid="role-select"]').select(newUser.role);
    cy.get('[data-testid="submit-button"]').click();

    // Assert
    cy.get('[data-testid="success-message"]')
      .should('be.visible')
      .and('contain', 'User created successfully');
    cy.get('[data-testid="user-list"]')
      .should('contain', newUser.name)
      .and('contain', newUser.email);
  });

  it('should handle form validation', () => {
    // Act
    cy.get('[data-testid="create-user-button"]').click();
    cy.get('[data-testid="submit-button"]').click();

    // Assert
    cy.get('[data-testid="name-error"]')
      .should('be.visible')
      .and('contain', 'Name is required');
    cy.get('[data-testid="email-error"]')
      .should('be.visible')
      .and('contain', 'Email is required');
  });

  it('should handle API errors', () => {
    // Arrange
    cy.intercept('POST', '/api/users', {
      statusCode: 400,
      body: { message: 'Email already exists' }
    });

    // Act
    cy.get('[data-testid="create-user-button"]').click();
    cy.get('[data-testid="name-input"]').type('Test User');
    cy.get('[data-testid="email-input"]').type('existing@example.com');
    cy.get('[data-testid="submit-button"]').click();

    // Assert
    cy.get('[data-testid="error-message"]')
      .should('be.visible')
      .and('contain', 'Email already exists');
  });
});
```

#### Testing API Integration

```typescript
// dashboard.cy.ts
describe('Dashboard', () => {
  beforeEach(() => {
    cy.login('admin', 'password');
    cy.visit('/dashboard');
  });

  it('should load dashboard data', () => {
    // Arrange
    const mockData = {
      totalUsers: 100,
      activeUsers: 75,
      revenue: 50000
    };

    cy.intercept('GET', '/api/dashboard', {
      statusCode: 200,
      body: mockData
    });

    // Act
    cy.get('[data-testid="refresh-button"]').click();

    // Assert
    cy.get('[data-testid="total-users"]')
      .should('contain', mockData.totalUsers);
    cy.get('[data-testid="active-users"]')
      .should('contain', mockData.activeUsers);
    cy.get('[data-testid="revenue"]')
      .should('contain', mockData.revenue);
  });

  it('should handle loading state', () => {
    // Arrange
    cy.intercept('GET', '/api/dashboard', (req) => {
      req.reply({
        delay: 1000,
        statusCode: 200,
        body: {
          totalUsers: 100,
          activeUsers: 75,
          revenue: 50000
        }
      });
    });

    // Act
    cy.get('[data-testid="refresh-button"]').click();

    // Assert
    cy.get('[data-testid="loading-spinner"]').should('be.visible');
    cy.get('[data-testid="loading-spinner"]').should('not.exist');
  });

  it('should handle error state', () => {
    // Arrange
    cy.intercept('GET', '/api/dashboard', {
      statusCode: 500,
      body: { message: 'Internal server error' }
    });

    // Act
    cy.get('[data-testid="refresh-button"]').click();

    // Assert
    cy.get('[data-testid="error-message"]')
      .should('be.visible')
      .and('contain', 'Failed to load dashboard data');
  });
});
```

### 4. Testing Custom Directives

```typescript
// highlight.directive.spec.ts
describe('HighlightDirective', () => {
  let fixture: ComponentFixture<TestComponent>;
  let component: TestComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [HighlightDirective, TestComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(TestComponent);
    component = fixture.componentInstance;
  });

  it('should apply highlight color', () => {
    // Arrange
    component.color = 'red';

    // Act
    fixture.detectChanges();

    // Assert
    const element = fixture.debugElement.query(By.directive(HighlightDirective));
    expect(element.nativeElement.style.backgroundColor).toBe('red');
  });

  it('should update highlight color', () => {
    // Arrange
    component.color = 'red';
    fixture.detectChanges();

    // Act
    component.color = 'blue';
    fixture.detectChanges();

    // Assert
    const element = fixture.debugElement.query(By.directive(HighlightDirective));
    expect(element.nativeElement.style.backgroundColor).toBe('blue');
  });

  it('should handle invalid color', () => {
    // Arrange
    component.color = 'invalid-color';

    // Act
    fixture.detectChanges();

    // Assert
    const element = fixture.debugElement.query(By.directive(HighlightDirective));
    expect(element.nativeElement.style.backgroundColor).toBe('');
  });
});
```

### 5. Testing Pipes

```typescript
// currency.pipe.spec.ts
describe('CurrencyPipe', () => {
  let pipe: CurrencyPipe;

  beforeEach(() => {
    pipe = new CurrencyPipe();
  });

  it('should format currency', () => {
    // Arrange
    const value = 1234.56;
    const currency = 'USD';

    // Act
    const result = pipe.transform(value, currency);

    // Assert
    expect(result).toBe('$1,234.56');
  });

  it('should handle different currencies', () => {
    // Arrange
    const value = 1234.56;

    // Act & Assert
    expect(pipe.transform(value, 'EUR')).toBe('€1,234.56');
    expect(pipe.transform(value, 'GBP')).toBe('£1,234.56');
    expect(pipe.transform(value, 'JPY')).toBe('¥1,235');
  });

  it('should handle null or invalid values', () => {
    // Act & Assert
    expect(pipe.transform(null, 'USD')).toBe('');
    expect(pipe.transform(undefined, 'USD')).toBe('');
    expect(pipe.transform('invalid', 'USD')).toBe('');
  });
});
```

### Conclusion

Key points to remember:

1. **Component Testing**:
   - Test with NgRx store
   - Mock dependencies
   - Test complex scenarios
   - Handle async operations
   - Test error states

2. **Service Testing**:
   - Test HTTP requests
   - Mock responses
   - Handle errors
   - Test caching
   - Test retry logic

3. **E2E Testing**:
   - Test user flows
   - Test API integration
   - Handle loading states
   - Test error scenarios
   - Test form validation

4. **Directive Testing**:
   - Test attribute changes
   - Test DOM updates
   - Handle edge cases
   - Test user interactions
   - Test style changes

5. **Pipe Testing**:
   - Test transformations
   - Handle edge cases
   - Test formatting
   - Test null values
   - Test invalid inputs

Remember to:
- Write meaningful tests
- Cover edge cases
- Test error scenarios
- Mock external dependencies
- Use test utilities
- Follow testing best practices
- Document test cases
- Maintain test coverage 