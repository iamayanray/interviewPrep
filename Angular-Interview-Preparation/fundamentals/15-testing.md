# Angular Testing

## Question
How do you test Angular applications? Explain unit testing, integration testing, and e2e testing approaches.

## Answer

### Introduction to Angular Testing

Angular provides a robust testing framework that makes it easy to write unit tests, integration tests, and end-to-end (e2e) tests. The framework includes:

- **Jasmine**: A behavior-driven development (BDD) testing framework
- **Karma**: A test runner that executes tests in various browsers
- **Angular Testing Utilities**: A set of utilities for testing Angular applications
- **Protractor**: An end-to-end testing framework (though Cypress is now more popular)

### Unit Testing

Unit testing in Angular focuses on testing individual components, services, and other classes in isolation.

#### Testing a Component

```typescript
// counter.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <div>
      <button (click)="increment()">+</button>
      <span>{{ count }}</span>
      <button (click)="decrement()">-</button>
    </div>
  `
})
export class CounterComponent {
  count = 0;

  increment() {
    this.count++;
  }

  decrement() {
    this.count--;
  }
}

// counter.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { CounterComponent } from './counter.component';

describe('CounterComponent', () => {
  let component: CounterComponent;
  let fixture: ComponentFixture<CounterComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [ CounterComponent ]
    })
    .compileComponents();
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(CounterComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should increment counter', () => {
    component.increment();
    expect(component.count).toBe(1);
  });

  it('should decrement counter', () => {
    component.count = 2;
    component.decrement();
    expect(component.count).toBe(1);
  });

  it('should update template when counter changes', () => {
    const compiled = fixture.nativeElement;
    const countDisplay = compiled.querySelector('span');
    
    component.increment();
    fixture.detectChanges();
    
    expect(countDisplay.textContent).toBe('1');
  });
});
```

#### Testing a Service

```typescript
// data.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class DataService {
  constructor(private http: HttpClient) {}

  getData(): Observable<any> {
    return this.http.get('/api/data');
  }

  postData(data: any): Observable<any> {
    return this.http.post('/api/data', data);
  }
}

// data.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { DataService } from './data.service';

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

  it('should be created', () => {
    expect(service).toBeTruthy();
  });

  it('should get data', () => {
    const mockData = { id: 1, name: 'Test' };
    
    service.getData().subscribe(data => {
      expect(data).toEqual(mockData);
    });

    const req = httpMock.expectOne('/api/data');
    expect(req.request.method).toBe('GET');
    req.flush(mockData);
  });

  it('should post data', () => {
    const mockData = { name: 'Test' };
    
    service.postData(mockData).subscribe(response => {
      expect(response).toEqual({ success: true });
    });

    const req = httpMock.expectOne('/api/data');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual(mockData);
    req.flush({ success: true });
  });
});
```

### Integration Testing

Integration testing focuses on testing how components work together.

```typescript
// user-list.component.ts
import { Component, OnInit } from '@angular/core';
import { UserService } from './user.service';

@Component({
  selector: 'app-user-list',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error">{{ error }}</div>
    <ul *ngIf="users">
      <li *ngFor="let user of users">{{ user.name }}</li>
    </ul>
  `
})
export class UserListComponent implements OnInit {
  users: any[] = [];
  loading = false;
  error: string | null = null;

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.loadUsers();
  }

  loadUsers() {
    this.loading = true;
    this.userService.getUsers().subscribe({
      next: (users) => {
        this.users = users;
        this.loading = false;
      },
      error: (err) => {
        this.error = 'Failed to load users';
        this.loading = false;
      }
    });
  }
}

// user-list.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { UserListComponent } from './user-list.component';
import { UserService } from './user.service';
import { of, throwError } from 'rxjs';

describe('UserListComponent Integration', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let userService: jasmine.SpyObj<UserService>;

  beforeEach(async () => {
    const userServiceSpy = jasmine.createSpyObj('UserService', ['getUsers']);
    
    await TestBed.configureTestingModule({
      declarations: [ UserListComponent ],
      providers: [
        { provide: UserService, useValue: userServiceSpy }
      ]
    })
    .compileComponents();

    userService = TestBed.inject(UserService) as jasmine.SpyObj<UserService>;
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
  });

  it('should load users successfully', () => {
    const mockUsers = [
      { id: 1, name: 'John' },
      { id: 2, name: 'Jane' }
    ];
    
    userService.getUsers.and.returnValue(of(mockUsers));
    fixture.detectChanges();

    expect(component.loading).toBeFalse();
    expect(component.error).toBeNull();
    expect(component.users).toEqual(mockUsers);
  });

  it('should handle error when loading users', () => {
    userService.getUsers.and.returnValue(throwError(() => new Error('API Error')));
    fixture.detectChanges();

    expect(component.loading).toBeFalse();
    expect(component.error).toBe('Failed to load users');
    expect(component.users).toEqual([]);
  });
});
```

### End-to-End Testing

E2E testing with Cypress (recommended over Protractor):

```typescript
// cypress/integration/app.spec.ts
describe('Angular App', () => {
  beforeEach(() => {
    cy.visit('/');
  });

  it('should display the home page', () => {
    cy.get('h1').should('contain', 'Welcome');
  });

  it('should navigate to about page', () => {
    cy.get('a[routerLink="/about"]').click();
    cy.url().should('include', '/about');
  });

  it('should handle form submission', () => {
    cy.get('input[name="username"]').type('testuser');
    cy.get('input[name="password"]').type('password123');
    cy.get('button[type="submit"]').click();
    
    cy.get('.success-message').should('be.visible');
  });
});
```

### Testing Best Practices

1. **Isolation**:
   - Test components and services in isolation
   - Use mocks and spies for dependencies
   - Avoid testing implementation details

2. **Coverage**:
   - Aim for high test coverage
   - Focus on critical paths and edge cases
   - Use `ng test --code-coverage` to generate coverage reports

3. **Readability**:
   - Use descriptive test names
   - Follow the Arrange-Act-Assert pattern
   - Keep tests focused and simple

4. **Maintenance**:
   - Keep tests up to date with code changes
   - Refactor tests when refactoring code
   - Use shared test utilities and helpers

### Common Testing Patterns

1. **Testing Async Operations**:
```typescript
it('should handle async operations', fakeAsync(() => {
  let result: string;
  
  service.getData().subscribe(data => {
    result = data;
  });
  
  tick(1000); // Simulate time passing
  expect(result).toBe('expected data');
}));
```

2. **Testing Router Navigation**:
```typescript
it('should navigate to detail page', () => {
  const router = TestBed.inject(Router);
  const spy = spyOn(router, 'navigate');
  
  component.navigateToDetail(1);
  
  expect(spy).toHaveBeenCalledWith(['/detail', 1]);
});
```

3. **Testing Form Validation**:
```typescript
it('should validate required fields', () => {
  const form = component.userForm;
  const nameControl = form.get('name');
  
  nameControl.setValue('');
  expect(nameControl.valid).toBeFalse();
  
  nameControl.setValue('John');
  expect(nameControl.valid).toBeTrue();
});
```

4. **Testing HTTP Interceptors**:
```typescript
it('should add auth header', () => {
  const httpRequest = new HttpRequest('GET', '/api/data');
  const next: HttpHandler = {
    handle: (req: HttpRequest<any>) => {
      expect(req.headers.has('Authorization')).toBeTrue();
      return of(new HttpResponse());
    }
  };
  
  interceptor.intercept(httpRequest, next).subscribe();
});
```

### Conclusion

Testing is a crucial part of Angular development. By following these practices and patterns, you can ensure your application is reliable and maintainable. Remember to:

1. Write tests for all critical functionality
2. Keep tests focused and maintainable
3. Use appropriate testing tools and utilities
4. Follow testing best practices
5. Maintain good test coverage

A well-tested Angular application is more reliable, easier to maintain, and provides better confidence when making changes. 