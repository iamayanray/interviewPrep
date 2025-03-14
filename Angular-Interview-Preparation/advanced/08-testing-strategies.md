# Advanced Angular Testing Strategies

## Question
What are the advanced testing strategies in Angular? Explain component testing, service testing, and end-to-end testing with examples.

## Answer

### Introduction to Advanced Angular Testing

Angular provides powerful testing capabilities for complex applications. Here's a comprehensive guide to advanced testing strategies.

### 1. Advanced Component Testing

#### Testing Component with NgRx Store

```typescript
// user-list.component.spec.ts
describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let store: MockStore;
  let mockUsers: User[];

  const initialState = {
    users: {
      entities: {},
      ids: [],
      loading: false,
      error: null
    }
  };

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [UserListComponent],
      imports: [
        StoreModule.forRoot({ users: usersReducer }),
        EffectsModule.forRoot([UsersEffects])
      ],
      providers: [
        provideMockStore({ initialState }),
        provideMockActions(() => actions$)
      ]
    }).compileComponents();

    store = TestBed.inject(MockStore);
    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;

    mockUsers = [
      { id: 1, name: 'John', email: 'john@example.com' },
      { id: 2, name: 'Jane', email: 'jane@example.com' }
    ];
  });

  it('should load users on init', () => {
    // Arrange
    const expectedUsers = mockUsers;
    store.setState({
      users: {
        entities: expectedUsers.reduce((acc, user) => ({
          ...acc,
          [user.id]: user
        }), {}),
        ids: expectedUsers.map(user => user.id),
        loading: false,
        error: null
      }
    });

    // Act
    fixture.detectChanges();

    // Assert
    expect(component.users$).toBeTruthy();
    component.users$.subscribe(users => {
      expect(users).toEqual(expectedUsers);
    });
  });

  it('should handle loading state', () => {
    // Arrange
    store.setState({
      users: {
        ...initialState.users,
        loading: true
      }
    });

    // Act
    fixture.detectChanges();

    // Assert
    expect(component.loading$).toBeTruthy();
    component.loading$.subscribe(loading => {
      expect(loading).toBeTrue();
    });
  });

  it('should handle error state', () => {
    // Arrange
    const error = 'Failed to load users';
    store.setState({
      users: {
        ...initialState.users,
        error
      }
    });

    // Act
    fixture.detectChanges();

    // Assert
    expect(component.error$).toBeTruthy();
    component.error$.subscribe(err => {
      expect(err).toBe(error);
    });
  });

  it('should dispatch load users action on init', () => {
    // Arrange
    const expectedAction = loadUsers();

    // Act
    fixture.detectChanges();

    // Assert
    expect(store.dispatch).toHaveBeenCalledWith(expectedAction);
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
    mockAnalyticsService = jasmine.createSpyObj('AnalyticsService', ['trackPageView']);
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

  it('should load user statistics on init', fakeAsync(() => {
    // Arrange
    const mockStats = {
      totalUsers: 100,
      activeUsers: 80,
      newUsers: 20
    };
    mockUserService.getUserStats.and.returnValue(of(mockStats));

    // Act
    component.ngOnInit();
    tick();

    // Assert
    expect(component.userStats).toEqual(mockStats);
    expect(mockAnalyticsService.trackPageView).toHaveBeenCalledWith('dashboard');
  }));

  it('should handle error when loading statistics', fakeAsync(() => {
    // Arrange
    const error = 'Failed to load statistics';
    mockUserService.getUserStats.and.returnValue(throwError(() => error));

    // Act
    component.ngOnInit();
    tick();

    // Assert
    expect(mockNotificationService.showError).toHaveBeenCalledWith(error);
  }));

  it('should update chart data when stats change', () => {
    // Arrange
    const mockStats = {
      totalUsers: 100,
      activeUsers: 80,
      newUsers: 20
    };

    // Act
    component.updateChartData(mockStats);

    // Assert
    expect(component.chartData).toEqual({
      labels: ['Total Users', 'Active Users', 'New Users'],
      datasets: [{
        data: [100, 80, 20],
        backgroundColor: ['#FF6384', '#36A2EB', '#FFCE56']
      }]
    });
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
      name: 'John',
      email: 'john@example.com'
    };
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should create user', () => {
    // Arrange
    const newUser = {
      name: 'Jane',
      email: 'jane@example.com'
    };

    // Act
    service.createUser(newUser).subscribe(user => {
      expect(user).toEqual(mockUser);
    });

    // Assert
    const req = httpMock.expectOne('api/users');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(newUser);
    req.flush(mockUser);
  });

  it('should handle error response', () => {
    // Arrange
    const newUser = {
      name: 'Jane',
      email: 'jane@example.com'
    };
    const errorResponse = {
      status: 400,
      statusText: 'Bad Request',
      error: {
        message: 'Email already exists'
      }
    };

    // Act
    service.createUser(newUser).subscribe({
      error: error => {
        expect(error.status).toBe(400);
        expect(error.error.message).toBe('Email already exists');
      }
    });

    // Assert
    const req = httpMock.expectOne('api/users');
    req.flush(errorResponse, errorResponse);
  });

  it('should retry failed requests', () => {
    // Arrange
    const newUser = {
      name: 'Jane',
      email: 'jane@example.com'
    };

    // Act
    service.createUser(newUser).subscribe(user => {
      expect(user).toEqual(mockUser);
    });

    // Assert
    const requests = httpMock.match('api/users');
    expect(requests.length).toBe(3); // Initial request + 2 retries

    // Simulate network errors
    requests[0].error(new ErrorEvent('Network error'));
    requests[1].error(new ErrorEvent('Network error'));
    requests[2].flush(mockUser);
  });
});
```

#### Testing Service with RxJS Operators

```typescript
// data.service.spec.ts
describe('DataService', () => {
  let service: DataService;
  let httpMock: HttpTestingController;
  let mockData: any[];

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [DataService]
    });

    service = TestBed.inject(DataService);
    httpMock = TestBed.inject(HttpTestingController);

    mockData = [
      { id: 1, value: 'A' },
      { id: 2, value: 'B' },
      { id: 3, value: 'C' }
    ];
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should cache successful responses', () => {
    // Arrange
    const url = 'api/data';

    // Act
    service.getData(url).subscribe();
    service.getData(url).subscribe(data => {
      expect(data).toEqual(mockData);
    });

    // Assert
    const req = httpMock.expectOne(url);
    req.flush(mockData);
  });

  it('should handle concurrent requests', fakeAsync(() => {
    // Arrange
    const url = 'api/data';
    const results: any[] = [];

    // Act
    service.getData(url).subscribe(data => results.push(data));
    service.getData(url).subscribe(data => results.push(data));
    tick();

    // Assert
    expect(results.length).toBe(2);
    expect(results[0]).toEqual(mockData);
    expect(results[1]).toEqual(mockData);

    const req = httpMock.expectOne(url);
    req.flush(mockData);
  }));

  it('should refresh cache after timeout', fakeAsync(() => {
    // Arrange
    const url = 'api/data';
    const updatedData = [...mockData, { id: 4, value: 'D' }];

    // Act
    service.getData(url).subscribe();
    tick(3600000); // 1 hour
    service.getData(url).subscribe(data => {
      expect(data).toEqual(updatedData);
    });

    // Assert
    const req = httpMock.expectOne(url);
    req.flush(updatedData);
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
    // Arrange
    const newUser = {
      name: 'John Doe',
      email: 'john@example.com',
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

  it('should edit existing user', () => {
    // Arrange
    const updatedName = 'John Updated';

    // Act
    cy.get('[data-testid="edit-user-button"]').first().click();
    cy.get('[data-testid="name-input"]').clear().type(updatedName);
    cy.get('[data-testid="submit-button"]').click();

    // Assert
    cy.get('[data-testid="success-message"]')
      .should('be.visible')
      .and('contain', 'User updated successfully');
    cy.get('[data-testid="user-list"]')
      .should('contain', updatedName);
  });

  it('should delete user', () => {
    // Act
    cy.get('[data-testid="delete-user-button"]').first().click();
    cy.get('[data-testid="confirm-delete-button"]').click();

    // Assert
    cy.get('[data-testid="success-message"]')
      .should('be.visible')
      .and('contain', 'User deleted successfully');
    cy.get('[data-testid="user-list"]')
      .should('have.length', 0);
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
    cy.intercept('GET', 'api/dashboard/stats', {
      statusCode: 200,
      body: {
        totalUsers: 100,
        activeUsers: 80,
        revenue: 50000
      }
    });

    // Act
    cy.visit('/dashboard');

    // Assert
    cy.get('[data-testid="total-users"]')
      .should('contain', '100');
    cy.get('[data-testid="active-users"]')
      .should('contain', '80');
    cy.get('[data-testid="revenue"]')
      .should('contain', '$50,000');
  });

  it('should handle API errors', () => {
    // Arrange
    cy.intercept('GET', 'api/dashboard/stats', {
      statusCode: 500,
      body: {
        message: 'Internal server error'
      }
    });

    // Act
    cy.visit('/dashboard');

    // Assert
    cy.get('[data-testid="error-message"]')
      .should('be.visible')
      .and('contain', 'Failed to load dashboard data');
  });

  it('should refresh data periodically', () => {
    // Arrange
    cy.intercept('GET', 'api/dashboard/stats').as('getStats');

    // Act
    cy.visit('/dashboard');
    cy.wait('@getStats');
    cy.wait(30000); // Wait for refresh interval
    cy.wait('@getStats');

    // Assert
    cy.get('@getStats.all').should('have.length', 2);
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
    // Act
    fixture.detectChanges();

    // Assert
    const element = fixture.nativeElement.querySelector('div');
    expect(element.style.backgroundColor).toBe('yellow');
  });

  it('should update highlight color', () => {
    // Act
    component.color = 'red';
    fixture.detectChanges();

    // Assert
    const element = fixture.nativeElement.querySelector('div');
    expect(element.style.backgroundColor).toBe('red');
  });

  it('should handle invalid color', () => {
    // Act
    component.color = 'invalid';
    fixture.detectChanges();

    // Assert
    const element = fixture.nativeElement.querySelector('div');
    expect(element.style.backgroundColor).toBe('transparent');
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
    // Act
    const result = pipe.transform(1000, 'USD');

    // Assert
    expect(result).toBe('$1,000.00');
  });

  it('should handle different currencies', () => {
    // Act
    const usd = pipe.transform(1000, 'USD');
    const eur = pipe.transform(1000, 'EUR');
    const gbp = pipe.transform(1000, 'GBP');

    // Assert
    expect(usd).toBe('$1,000.00');
    expect(eur).toBe('€1,000.00');
    expect(gbp).toBe('£1,000.00');
  });

  it('should handle null values', () => {
    // Act
    const result = pipe.transform(null, 'USD');

    // Assert
    expect(result).toBe('');
  });

  it('should handle invalid numbers', () => {
    // Act
    const result = pipe.transform('invalid', 'USD');

    // Assert
    expect(result).toBe('');
  });
});
```

### Conclusion

Key points to remember:

1. **Component Testing**:
   - Test with NgRx store
   - Mock complex dependencies
   - Test async operations
   - Verify UI updates
   - Handle error states

2. **Service Testing**:
   - Test HTTP requests
   - Mock responses
   - Handle errors
   - Test caching
   - Test retry logic

3. **E2E Testing**:
   - Test user flows
   - Test form validation
   - Test API integration
   - Handle errors
   - Test periodic updates

4. **Directive Testing**:
   - Test attribute changes
   - Test DOM updates
   - Handle invalid inputs
   - Test edge cases

5. **Pipe Testing**:
   - Test transformations
   - Handle edge cases
   - Test different formats
   - Handle invalid inputs

Remember to:
- Write meaningful tests
- Cover edge cases
- Test error scenarios
- Mock external dependencies
- Use appropriate testing tools
- Follow testing best practices
- Maintain test readability
- Keep tests maintainable 