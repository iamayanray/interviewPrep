# Angular Testing

## Question
How do you test Angular applications? Explain different types of testing and best practices.

## Answer

### Introduction to Angular Testing

Angular provides a comprehensive testing framework with tools for unit testing, integration testing, and end-to-end testing.

### 1. Unit Testing with Jasmine and Karma

#### Basic Component Testing

```typescript
// user.component.ts
@Component({
  selector: 'app-user',
  template: `
    <div *ngIf="user">
      <h2>{{ user.name }}</h2>
      <p>{{ user.email }}</p>
    </div>
  `
})
export class UserComponent {
  @Input() user: User | null = null;
}

// user.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { UserComponent } from './user.component';

describe('UserComponent', () => {
  let component: UserComponent;
  let fixture: ComponentFixture<UserComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [ UserComponent ]
    })
    .compileComponents();
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(UserComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should display user information when user is provided', () => {
    const user = { name: 'John Doe', email: 'john@example.com' };
    component.user = user;
    fixture.detectChanges();

    const compiled = fixture.nativeElement;
    expect(compiled.querySelector('h2').textContent).toContain('John Doe');
    expect(compiled.querySelector('p').textContent).toContain('john@example.com');
  });
});
```

#### Service Testing

```typescript
// user.service.ts
@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(private http: HttpClient) {}

  getUser(id: number): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`);
  }
}

// user.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { UserService } from './user.service';

describe('UserService', () => {
  let service: UserService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [ HttpClientTestingModule ],
      providers: [ UserService ]
    });

    service = TestBed.inject(UserService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should be created', () => {
    expect(service).toBeTruthy();
  });

  it('should fetch user data', () => {
    const mockUser = { id: 1, name: 'John Doe', email: 'john@example.com' };

    service.getUser(1).subscribe(user => {
      expect(user).toEqual(mockUser);
    });

    const req = httpMock.expectOne('/api/users/1');
    expect(req.request.method).toBe('GET');
    req.flush(mockUser);
  });
});
```

#### Directive Testing

```typescript
// highlight.directive.ts
@Directive({
  selector: '[appHighlight]',
  host: {
    '(mouseenter)': 'onMouseEnter()',
    '(mouseleave)': 'onMouseLeave()'
  }
})
export class HighlightDirective {
  @Input() highlightColor = 'yellow';

  constructor(private el: ElementRef) {}

  onMouseEnter() {
    this.highlight(this.highlightColor);
  }

  onMouseLeave() {
    this.highlight(null);
  }

  private highlight(color: string | null) {
    this.el.nativeElement.style.backgroundColor = color;
  }
}

// highlight.directive.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { HighlightDirective } from './highlight.directive';

describe('HighlightDirective', () => {
  let component: HighlightDirective;
  let fixture: ComponentFixture<HighlightDirective>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [ HighlightDirective ]
    })
    .compileComponents();
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(HighlightDirective);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should highlight element on mouseenter', () => {
    const element = fixture.nativeElement;
    element.dispatchEvent(new MouseEvent('mouseenter'));
    expect(element.style.backgroundColor).toBe('yellow');
  });

  it('should remove highlight on mouseleave', () => {
    const element = fixture.nativeElement;
    element.dispatchEvent(new MouseEvent('mouseenter'));
    element.dispatchEvent(new MouseEvent('mouseleave'));
    expect(element.style.backgroundColor).toBe('');
  });
});
```

### 2. Integration Testing

#### Component with Service Integration

```typescript
// user-list.component.ts
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngIf="users$ | async as users">
      <div *ngFor="let user of users">
        {{ user.name }}
      </div>
    </div>
  `
})
export class UserListComponent {
  users$ = this.userService.getUsers();

  constructor(private userService: UserService) {}
}

// user-list.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { UserListComponent } from './user-list.component';
import { UserService } from './user.service';
import { of } from 'rxjs';

describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let userService: jasmine.SpyObj<UserService>;

  beforeEach(async () => {
    const spy = jasmine.createSpyObj('UserService', ['getUsers']);
    spy.getUsers.and.returnValue(of([
      { id: 1, name: 'John Doe' },
      { id: 2, name: 'Jane Smith' }
    ]));

    await TestBed.configureTestingModule({
      declarations: [ UserListComponent ],
      providers: [ { provide: UserService, useValue: spy } ]
    })
    .compileComponents();
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
    userService = TestBed.inject(UserService) as jasmine.SpyObj<UserService>;
    fixture.detectChanges();
  });

  it('should display users from service', () => {
    const compiled = fixture.nativeElement;
    expect(compiled.querySelectorAll('div').length).toBe(3); // Including container div
    expect(userService.getUsers).toHaveBeenCalled();
  });
});
```

### 3. End-to-End Testing with Cypress

```typescript
// cypress/integration/user.spec.ts
describe('User Management', () => {
  beforeEach(() => {
    cy.visit('/users');
  });

  it('should display user list', () => {
    cy.get('.user-list').should('exist');
    cy.get('.user-item').should('have.length.greaterThan', 0);
  });

  it('should navigate to user details', () => {
    cy.get('.user-item').first().click();
    cy.url().should('include', '/users/');
  });

  it('should create new user', () => {
    cy.get('.create-user-btn').click();
    cy.get('input[name="name"]').type('New User');
    cy.get('input[name="email"]').type('new@example.com');
    cy.get('button[type="submit"]').click();
    
    cy.get('.user-list').should('contain', 'New User');
  });
});
```

### 4. Testing Best Practices

#### 1. Test Organization

```typescript
describe('Component/Service Name', () => {
  // Setup
  beforeEach(() => {
    // Common setup code
  });

  // Group related tests
  describe('when condition A', () => {
    it('should behave in way X', () => {
      // Test code
    });
  });

  describe('when condition B', () => {
    it('should behave in way Y', () => {
      // Test code
    });
  });
});
```

#### 2. Mocking Dependencies

```typescript
// Using TestBed
TestBed.configureTestingModule({
  declarations: [ ComponentToTest ],
  providers: [
    { provide: DependencyService, useValue: mockDependencyService }
  ]
});

// Using spies
const spy = spyOn(service, 'methodName').and.returnValue(of(mockData));
```

#### 3. Testing Async Operations

```typescript
it('should handle async operation', fakeAsync(() => {
  let result: any;
  service.asyncOperation().subscribe(r => result = r);
  
  tick(1000); // Simulate time passing
  expect(result).toBeDefined();
}));
```

#### 4. Testing Error Cases

```typescript
it('should handle error', () => {
  service.errorOperation().subscribe({
    error: (error) => {
      expect(error.message).toBe('Expected error message');
    }
  });
});
```

### 5. Testing Tools and Utilities

#### 1. TestBed Configuration

```typescript
TestBed.configureTestingModule({
  imports: [
    HttpClientTestingModule,
    RouterTestingModule,
    FormsModule,
    ReactiveFormsModule
  ],
  declarations: [
    ComponentToTest,
    MockComponents,
    MockDirectives
  ],
  providers: [
    { provide: Service, useClass: MockService },
    { provide: APP_BASE_HREF, useValue: '/' }
  ],
  schemas: [ NO_ERRORS_SCHEMA ]
});
```

#### 2. Custom Test Utilities

```typescript
// test-utils.ts
export function createComponent<T>(component: Type<T>): ComponentFixture<T> {
  return TestBed.createComponent(component);
}

export function getTestBed(): TestBed {
  return TestBed;
}

export function mockHttpResponse<T>(url: string, data: T) {
  const httpMock = TestBed.inject(HttpTestingController);
  const req = httpMock.expectOne(url);
  req.flush(data);
}
```

### 6. Testing Common Scenarios

#### 1. Form Testing

```typescript
describe('LoginFormComponent', () => {
  let component: LoginFormComponent;
  let fixture: ComponentFixture<LoginFormComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ ReactiveFormsModule ],
      declarations: [ LoginFormComponent ]
    }).compileComponents();
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(LoginFormComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should validate required fields', () => {
    const form = component.loginForm;
    expect(form.valid).toBeFalsy();

    form.controls['email'].setValue('test@example.com');
    form.controls['password'].setValue('password123');
    expect(form.valid).toBeTruthy();
  });

  it('should handle form submission', () => {
    const spy = spyOn(component, 'onSubmit');
    const form = component.loginForm;
    
    form.controls['email'].setValue('test@example.com');
    form.controls['password'].setValue('password123');
    component.onSubmit();
    
    expect(spy).toHaveBeenCalled();
  });
});
```

#### 2. Router Testing

```typescript
describe('AppRoutingModule', () => {
  let router: Router;
  let location: Location;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [ RouterTestingModule.withRoutes(routes) ]
    });
    router = TestBed.inject(Router);
    location = TestBed.inject(Location);
  });

  it('should navigate to user details', fakeAsync(() => {
    router.navigate(['/users/1']);
    tick();
    expect(location.path()).toBe('/users/1');
  }));
});
```

### Conclusion

Key points to remember:

1. **Test Types**:
   - Unit tests for components, services, and directives
   - Integration tests for component interactions
   - E2E tests for complete user flows

2. **Testing Tools**:
   - Jasmine for test framework
   - Karma for test runner
   - Cypress for E2E testing
   - TestBed for Angular testing utilities

3. **Best Practices**:
   - Organize tests logically
   - Mock external dependencies
   - Test both success and error cases
   - Use async testing utilities appropriately
   - Keep tests focused and isolated

4. **Common Scenarios**:
   - Form validation
   - HTTP requests
   - Router navigation
   - Component interactions
   - Error handling

Remember to:
- Write meaningful test descriptions
- Follow the Arrange-Act-Assert pattern
- Keep tests maintainable
- Use appropriate testing utilities
- Mock external dependencies
- Test edge cases and error scenarios 