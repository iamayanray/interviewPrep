# Advanced Angular Testing

## Question
What are the advanced testing strategies in Angular? Explain component testing, service testing, and e2e testing with examples.

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

  const initialState = {
    users: {
      users: [
        { id: 1, name: 'John', email: 'john@example.com' },
        { id: 2, name: 'Jane', email: 'jane@example.com' }
      ],
      loading: false,
      error: null
    }
  };

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [UserListComponent],
      imports: [
        StoreModule.forRoot({ users: userReducer }),
        MockStoreModule
      ]
    }).compileComponents();

    store = TestBed.inject(MockStore);
    store.setState(initialState);
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should load users from store', () => {
    const userElements = fixture.debugElement.queryAll(By.css('.user-item'));
    expect(userElements.length).toBe(2);
    expect(userElements[0].nativeElement.textContent).toContain('John');
  });

  it('should dispatch loadUsers action on init', () => {
    const storeSpy = spyOn(store, 'dispatch');
    component.ngOnInit();
    expect(storeSpy).toHaveBeenCalledWith(loadUsers());
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

  beforeEach(async () => {
    mockUserService = jasmine.createSpyObj('UserService', ['getUserStats']);
    mockAnalyticsService = jasmine.createSpyObj('AnalyticsService', ['trackEvent']);

    await TestBed.configureTestingModule({
      declarations: [DashboardComponent],
      providers: [
        { provide: UserService, useValue: mockUserService },
        { provide: AnalyticsService, useValue: mockAnalyticsService }
      ]
    }).compileComponents();

    mockUserService.getUserStats.and.returnValue(of({
      totalUsers: 100,
      activeUsers: 80,
      revenue: 50000
    }));
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(DashboardComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should display user statistics', () => {
    const statsElement = fixture.debugElement.query(By.css('.stats'));
    expect(statsElement.nativeElement.textContent).toContain('100');
    expect(statsElement.nativeElement.textContent).toContain('80');
  });

  it('should track analytics event on refresh', () => {
    component.refreshStats();
    expect(mockAnalyticsService.trackEvent).toHaveBeenCalledWith('dashboard_refresh');
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
  let apiUrl: string;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        UserService,
        { provide: 'API_URL', useValue: 'http://api.example.com' }
      ]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
    apiUrl = TestBed.inject('API_URL');
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should create user', () => {
    const user = { name: 'John', email: 'john@example.com' };
    service.createUser(user).subscribe(response => {
      expect(response).toEqual({ ...user, id: 1 });
    });

    const req = httpMock.expectOne(`${apiUrl}/users`);
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(user);
    req.flush({ ...user, id: 1 });
  });

  it('should handle error response', () => {
    const errorResponse = {
      status: 400,
      statusText: 'Bad Request',
      error: { message: 'Invalid data' }
    };

    service.createUser({}).subscribe({
      error: error => {
        expect(error.status).toBe(400);
        expect(error.error.message).toBe('Invalid data');
      }
    });

    const req = httpMock.expectOne(`${apiUrl}/users`);
    req.flush('', errorResponse);
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

  it('should retry failed requests', (done) => {
    const errorResponse = { status: 500 };
    let retryCount = 0;

    service.getDataWithRetry().subscribe({
      next: data => {
        expect(data).toEqual({ success: true });
        expect(retryCount).toBe(2);
        done();
      }
    });

    const req = httpMock.expectOne('api/data');
    req.flush('', errorResponse);
    retryCount++;

    const retryReq = httpMock.expectOne('api/data');
    retryReq.flush({ success: true });
    retryCount++;
  });

  it('should cache successful responses', () => {
    const data = { id: 1, value: 'test' };
    let requestCount = 0;

    service.getCachedData(1).subscribe();
    service.getCachedData(1).subscribe();

    httpMock.expectOne('api/data/1').flush(data);
    expect(httpMock.match('api/data/1').length).toBe(1);
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

  it('should create new user', () => {
    cy.intercept('POST', '/api/users').as('createUser');

    cy.get('[data-test="add-user-button"]').click();
    cy.get('[data-test="user-form"]').within(() => {
      cy.get('[data-test="name-input"]').type('John Doe');
      cy.get('[data-test="email-input"]').type('john@example.com');
      cy.get('[data-test="submit-button"]').click();
    });

    cy.wait('@createUser').then(interception => {
      expect(interception.response?.statusCode).to.equal(201);
      expect(interception.response?.body).to.have.property('id');
    });

    cy.get('[data-test="user-list"]')
      .should('contain', 'John Doe')
      .and('contain', 'john@example.com');
  });

  it('should handle form validation', () => {
    cy.get('[data-test="add-user-button"]').click();
    cy.get('[data-test="submit-button"]').click();

    cy.get('[data-test="name-error"]')
      .should('be.visible')
      .and('contain', 'Name is required');
  });
});
```

#### Testing API Integration

```typescript
// api-integration.cy.ts
describe('API Integration', () => {
  beforeEach(() => {
    cy.intercept('GET', '/api/data').as('getData');
    cy.visit('/dashboard');
  });

  it('should load and display data', () => {
    cy.wait('@getData').then(interception => {
      const data = interception.response?.body;
      cy.get('[data-test="data-container"]').within(() => {
        cy.get('[data-test="total-count"]')
          .should('contain', data.total);
        cy.get('[data-test="active-count"]')
          .should('contain', data.active);
      });
    });
  });

  it('should handle API errors', () => {
    cy.intercept('GET', '/api/data', {
      statusCode: 500,
      body: { message: 'Internal Server Error' }
    }).as('getDataError');

    cy.visit('/dashboard');
    cy.wait('@getDataError');

    cy.get('[data-test="error-message"]')
      .should('be.visible')
      .and('contain', 'Failed to load data');
  });
});
```

### 4. Testing Custom Directives

```typescript
// highlight.directive.spec.ts
describe('HighlightDirective', () => {
  let fixture: ComponentFixture<TestComponent>;
  let element: HTMLElement;

  @Component({
    template: `
      <div appHighlight [highlightColor]="color">
        Test Content
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
    element = fixture.debugElement.query(By.directive(HighlightDirective))
      .nativeElement;
  });

  it('should apply highlight color', () => {
    fixture.detectChanges();
    expect(element.style.backgroundColor).toBe('yellow');
  });

  it('should update highlight color on input change', () => {
    fixture.componentInstance.color = 'red';
    fixture.detectChanges();
    expect(element.style.backgroundColor).toBe('red');
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

  it('should format currency with default options', () => {
    expect(pipe.transform(1000)).toBe('$1,000.00');
  });

  it('should format currency with custom options', () => {
    expect(pipe.transform(1000, 'EUR', 'symbol', '1.0-0')).toBe('€1,000');
  });

  it('should handle null values', () => {
    expect(pipe.transform(null)).toBe('');
  });

  it('should handle undefined values', () => {
    expect(pipe.transform(undefined)).toBe('');
  });
});
```

### Conclusion

Key points to remember:

1. **Component Testing**:
   - Test with NgRx store
   - Mock complex dependencies
   - Test component lifecycle
   - Verify template bindings

2. **Service Testing**:
   - Test HTTP requests
   - Handle error scenarios
   - Test RxJS operators
   - Verify caching

3. **E2E Testing**:
   - Test user flows
   - Mock API responses
   - Handle async operations
   - Test error scenarios

4. **Directive Testing**:
   - Test input properties
   - Verify DOM manipulation
   - Test event handling
   - Check style changes

5. **Pipe Testing**:
   - Test transformations
   - Handle edge cases
   - Test with different inputs
   - Verify formatting

Remember to:
- Use appropriate testing tools
- Mock external dependencies
- Test error scenarios
- Follow testing best practices
- Maintain test isolation
- Use meaningful assertions
- Test edge cases
- Keep tests maintainable 