# Advanced Angular Testing Strategies

## Question
What are the advanced testing strategies in Angular? Explain component testing, service testing, and end-to-end testing with examples.

## Answer

### Introduction to Advanced Angular Testing Strategies

Angular provides a robust testing framework that allows developers to write comprehensive tests for their applications. Here's a detailed guide to advanced testing strategies in Angular.

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
              entities: mockUsers,
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
    fixture.detectChanges();
  });

  it('should load users from store', () => {
    const userElements = fixture.debugElement.queryAll(
      By.css('.user-item')
    );
    expect(userElements.length).toBe(2);
    expect(userElements[0].nativeElement.textContent)
      .toContain('John Doe');
  });

  it('should show loading state', () => {
    store.setState({
      users: {
        entities: [],
        loading: true,
        error: null
      }
    });
    fixture.detectChanges();

    const loadingElement = fixture.debugElement.query(
      By.css('.loading-spinner')
    );
    expect(loadingElement).toBeTruthy();
  });

  it('should show error state', () => {
    const errorMessage = 'Failed to load users';
    store.setState({
      users: {
        entities: [],
        loading: false,
        error: errorMessage
      }
    });
    fixture.detectChanges();

    const errorElement = fixture.debugElement.query(
      By.css('.error-message')
    );
    expect(errorElement.nativeElement.textContent)
      .toContain(errorMessage);
  });

  it('should dispatch load users action on init', () => {
    const expectedAction = loadUsers();
    const actions$ = TestBed.inject(Actions);
    const actions: Action[] = [];

    actions$.subscribe(action => actions.push(action));
    component.ngOnInit();

    expect(actions).toContain(expectedAction);
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
    mockUserService = jasmine.createSpyObj('UserService', [
      'getUserStats',
      'getRecentActivity'
    ]);
    mockAnalyticsService = jasmine.createSpyObj('AnalyticsService', [
      'trackPageView',
      'trackEvent'
    ]);
    mockNotificationService = jasmine.createSpyObj('NotificationService', [
      'showSuccess',
      'showError'
    ]);

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

  it('should load user statistics', fakeAsync(() => {
    const mockStats = {
      totalUsers: 100,
      activeUsers: 75,
      newUsers: 10
    };
    mockUserService.getUserStats.and.returnValue(
      of(mockStats)
    );

    component.loadUserStats();
    tick();

    expect(component.userStats).toEqual(mockStats);
    expect(mockAnalyticsService.trackEvent)
      .toHaveBeenCalledWith('stats_loaded');
  }));

  it('should handle error loading statistics', fakeAsync(() => {
    const error = new Error('Failed to load stats');
    mockUserService.getUserStats.and.returnValue(
      throwError(() => error)
    );

    component.loadUserStats();
    tick();

    expect(mockNotificationService.showError)
      .toHaveBeenCalledWith('Failed to load statistics');
  }));

  it('should load recent activity', fakeAsync(() => {
    const mockActivity = [
      { id: 1, type: 'login', timestamp: new Date() },
      { id: 2, type: 'update', timestamp: new Date() }
    ];
    mockUserService.getRecentActivity.and.returnValue(
      of(mockActivity)
    );

    component.loadRecentActivity();
    tick();

    expect(component.recentActivity).toEqual(mockActivity);
  }));

  it('should track page view on init', () => {
    component.ngOnInit();
    expect(mockAnalyticsService.trackPageView)
      .toHaveBeenCalledWith('dashboard');
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
  let mockUser: User;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserService]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
    mockUser = {
      id: 1,
      name: 'John Doe',
      email: 'john@example.com'
    };
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should create user', () => {
    service.createUser(mockUser).subscribe(user => {
      expect(user).toEqual(mockUser);
    });

    const req = httpMock.expectOne('api/users');
    expect(req.request.method).toBe('POST');
    req.flush(mockUser);
  });

  it('should handle error response', () => {
    const errorResponse = {
      status: 400,
      statusText: 'Bad Request',
      error: { message: 'Invalid user data' }
    };

    service.createUser(mockUser).subscribe({
      error: error => {
        expect(error.status).toBe(400);
        expect(error.error.message).toBe('Invalid user data');
      }
    });

    const req = httpMock.expectOne('api/users');
    req.flush(errorResponse, errorResponse);
  });

  it('should retry failed request', () => {
    const errorResponse = {
      status: 500,
      statusText: 'Internal Server Error'
    };

    service.createUser(mockUser).subscribe(user => {
      expect(user).toEqual(mockUser);
    });

    // First request fails
    const req1 = httpMock.expectOne('api/users');
    req1.flush(errorResponse, errorResponse);

    // Second request succeeds
    const req2 = httpMock.expectOne('api/users');
    req2.flush(mockUser);
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
    const mockData = { id: 1, value: 'test' };

    // First request
    service.getData(1).subscribe(data => {
      expect(data).toEqual(mockData);
    });
    const req1 = httpMock.expectOne('api/data/1');
    req1.flush(mockData);

    // Second request should use cache
    service.getData(1).subscribe(data => {
      expect(data).toEqual(mockData);
    });
    httpMock.expectNone('api/data/1');
  });

  it('should refresh cache when requested', () => {
    const mockData = { id: 1, value: 'test' };
    const updatedData = { id: 1, value: 'updated' };

    // Initial request
    service.getData(1).subscribe(data => {
      expect(data).toEqual(mockData);
    });
    const req1 = httpMock.expectOne('api/data/1');
    req1.flush(mockData);

    // Refresh request
    service.getData(1, true).subscribe(data => {
      expect(data).toEqual(updatedData);
    });
    const req2 = httpMock.expectOne('api/data/1');
    req2.flush(updatedData);
  });

  it('should handle concurrent requests', fakeAsync(() => {
    const mockData = { id: 1, value: 'test' };
    let requestCount = 0;

    // Make multiple concurrent requests
    service.getData(1).subscribe();
    service.getData(1).subscribe();
    service.getData(1).subscribe();

    // Verify only one HTTP request is made
    const req = httpMock.expectOne('api/data/1');
    req.flush(mockData);
    httpMock.verify();

    expect(requestCount).toBe(1);
  }));
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

  it('should create new user', () => {
    cy.intercept('POST', '/api/users').as('createUser');

    cy.get('[data-test="create-user-button"]').click();
    cy.get('[data-test="user-form"]').within(() => {
      cy.get('[data-test="name-input"]').type('John Doe');
      cy.get('[data-test="email-input"]').type('john@example.com');
      cy.get('[data-test="submit-button"]').click();
    });

    cy.wait('@createUser').then(interception => {
      expect(interception.response?.statusCode).to.equal(201);
      expect(interception.response?.body).to.have.property('id');
    });

    cy.get('[data-test="success-message"]')
      .should('be.visible')
      .and('contain', 'User created successfully');
  });

  it('should edit existing user', () => {
    cy.intercept('PUT', '/api/users/*').as('updateUser');

    cy.get('[data-test="user-list"]')
      .find('[data-test="user-item"]')
      .first()
      .find('[data-test="edit-button"]')
      .click();

    cy.get('[data-test="user-form"]').within(() => {
      cy.get('[data-test="name-input"]')
        .clear()
        .type('Updated Name');
      cy.get('[data-test="submit-button"]').click();
    });

    cy.wait('@updateUser').then(interception => {
      expect(interception.response?.statusCode).to.equal(200);
      expect(interception.response?.body.name).to.equal('Updated Name');
    });

    cy.get('[data-test="success-message"]')
      .should('be.visible')
      .and('contain', 'User updated successfully');
  });

  it('should delete user', () => {
    cy.intercept('DELETE', '/api/users/*').as('deleteUser');

    cy.get('[data-test="user-list"]')
      .find('[data-test="user-item"]')
      .first()
      .find('[data-test="delete-button"]')
      .click();

    cy.get('[data-test="confirm-dialog"]')
      .find('[data-test="confirm-button"]')
      .click();

    cy.wait('@deleteUser').then(interception => {
      expect(interception.response?.statusCode).to.equal(204);
    });

    cy.get('[data-test="success-message"]')
      .should('be.visible')
      .and('contain', 'User deleted successfully');
  });

  it('should handle form validation', () => {
    cy.get('[data-test="create-user-button"]').click();
    cy.get('[data-test="user-form"]').within(() => {
      cy.get('[data-test="submit-button"]').click();
    });

    cy.get('[data-test="name-error"]')
      .should('be.visible')
      .and('contain', 'Name is required');
    cy.get('[data-test="email-error"]')
      .should('be.visible')
      .and('contain', 'Email is required');
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
    cy.intercept('GET', '/api/dashboard/stats').as('getStats');
    cy.intercept('GET', '/api/dashboard/activity').as('getActivity');

    cy.wait(['@getStats', '@getActivity']).then(interceptions => {
      const [stats, activity] = interceptions;
      expect(stats.response?.statusCode).to.equal(200);
      expect(activity.response?.statusCode).to.equal(200);
    });

    cy.get('[data-test="stats-container"]')
      .should('be.visible')
      .and('not.contain', 'Loading...');
    cy.get('[data-test="activity-list"]')
      .should('be.visible')
      .and('not.contain', 'Loading...');
  });

  it('should handle loading state', () => {
    cy.intercept('GET', '/api/dashboard/stats', {
      delay: 1000
    }).as('getStats');

    cy.get('[data-test="stats-container"]')
      .should('contain', 'Loading...');
    cy.get('[data-test="activity-list"]')
      .should('contain', 'Loading...');

    cy.wait('@getStats');
    cy.get('[data-test="stats-container"]')
      .should('not.contain', 'Loading...');
  });

  it('should handle error state', () => {
    cy.intercept('GET', '/api/dashboard/stats', {
      statusCode: 500,
      body: { message: 'Internal server error' }
    }).as('getStats');

    cy.wait('@getStats');
    cy.get('[data-test="error-message"]')
      .should('be.visible')
      .and('contain', 'Failed to load dashboard data');
  });

  it('should refresh data', () => {
    cy.intercept('GET', '/api/dashboard/stats').as('getStats');

    cy.get('[data-test="refresh-button"]').click();
    cy.wait('@getStats');

    cy.get('[data-test="last-updated"]')
      .should('be.visible')
      .and('not.contain', 'Never');
  });
});
```

### 4. Testing Custom Directives

```typescript
// highlight.directive.spec.ts
describe('HighlightDirective', () => {
  let fixture: ComponentFixture<TestComponent>;
  let component: TestComponent;

  @Component({
    template: `
      <div [appHighlight]="color">
        Test content
      </div>
    `
  })
  class TestComponent {
    color = 'yellow';
  }

  beforeEach(() => {
    TestBed.configureTestingModule({
      declarations: [HighlightDirective, TestComponent]
    });

    fixture = TestBed.createComponent(TestComponent);
    component = fixture.componentInstance;
  });

  it('should apply highlight color', () => {
    fixture.detectChanges();
    const element = fixture.debugElement.query(
      By.directive(HighlightDirective)
    );
    expect(element.nativeElement.style.backgroundColor)
      .toBe('yellow');
  });

  it('should update highlight color', () => {
    fixture.detectChanges();
    component.color = 'red';
    fixture.detectChanges();

    const element = fixture.debugElement.query(
      By.directive(HighlightDirective)
    );
    expect(element.nativeElement.style.backgroundColor)
      .toBe('red');
  });

  it('should handle invalid color', () => {
    component.color = 'invalid';
    fixture.detectChanges();

    const element = fixture.debugElement.query(
      By.directive(HighlightDirective)
    );
    expect(element.nativeElement.style.backgroundColor)
      .toBe('transparent');
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
    expect(pipe.transform(1000, 'USD')).toBe('$1,000.00');
    expect(pipe.transform(1000, 'EUR')).toBe('€1,000.00');
    expect(pipe.transform(1000, 'GBP')).toBe('£1,000.00');
  });

  it('should handle different currencies', () => {
    expect(pipe.transform(1000, 'JPY')).toBe('¥1,000');
    expect(pipe.transform(1000, 'CNY')).toBe('¥1,000.00');
    expect(pipe.transform(1000, 'INR')).toBe('₹1,000.00');
  });

  it('should handle null or invalid values', () => {
    expect(pipe.transform(null, 'USD')).toBe('');
    expect(pipe.transform(undefined, 'USD')).toBe('');
    expect(pipe.transform('invalid', 'USD')).toBe('');
  });

  it('should handle decimal places', () => {
    expect(pipe.transform(1000.5, 'USD')).toBe('$1,000.50');
    expect(pipe.transform(1000.567, 'USD')).toBe('$1,000.57');
  });
});
```

### Conclusion

Key points to remember:

1. **Component Testing**:
   - Test with NgRx store
   - Mock complex dependencies
   - Test loading states
   - Test error states
   - Test user interactions

2. **Service Testing**:
   - Test HTTP requests
   - Test error handling
   - Test retry logic
   - Test caching
   - Test concurrent requests

3. **E2E Testing**:
   - Test user flows
   - Test API integration
   - Test loading states
   - Test error states
   - Test form validation

4. **Directive Testing**:
   - Test style application
   - Test input changes
   - Test invalid inputs
   - Test DOM updates
   - Test edge cases

5. **Pipe Testing**:
   - Test formatting
   - Test different inputs
   - Test edge cases
   - Test null values
   - Test invalid values

Remember to:
- Write meaningful tests
- Cover edge cases
- Test error scenarios
- Mock external dependencies
- Use appropriate testing tools
- Follow testing best practices
- Maintain test coverage
- Regular test maintenance 