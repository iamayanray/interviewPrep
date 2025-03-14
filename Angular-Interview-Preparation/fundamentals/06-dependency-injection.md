# Angular Dependency Injection

## Question
Explain Angular's dependency injection system. How does it work, and what are the different ways to provide dependencies?

## Answer

### What is Dependency Injection?

Dependency Injection (DI) is a design pattern where dependencies (services or objects that a class needs) are provided to a class instead of the class creating them itself. Angular's DI system helps in making components reusable, maintainable, and testable.

### How Angular's DI System Works

Angular's DI system is built on top of the TypeScript decorators and metadata. Here's how it works:

1. **Decorator Metadata**: When you use the `@Injectable()` decorator, Angular creates metadata about the service.
2. **Provider Registration**: Services are registered with providers in modules or components.
3. **Injection**: Angular's DI system looks up the registered provider and injects the instance into the constructor.

### Basic Service Injection

```typescript
// user.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({
  providedIn: 'root' // Service is available application-wide
})
export class UserService {
  constructor(private http: HttpClient) {}

  getUsers() {
    return this.http.get('/api/users');
  }
}

// user.component.ts
import { Component } from '@angular/core';
import { UserService } from './user.service';

@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users">
      {{ user.name }}
    </div>
  `
})
export class UserListComponent {
  users: any[] = [];

  constructor(private userService: UserService) {
    this.userService.getUsers().subscribe(users => {
      this.users = users;
    });
  }
}
```

### Different Ways to Provide Dependencies

#### 1. Root Level (Application-wide)

```typescript
@Injectable({
  providedIn: 'root'
})
export class GlobalService {
  // This service will be a singleton across the entire application
}
```

#### 2. Module Level

```typescript
// shared.module.ts
import { NgModule } from '@angular/core';
import { SharedService } from './shared.service';

@NgModule({
  providers: [SharedService]
})
export class SharedModule { }
```

#### 3. Component Level

```typescript
@Component({
  selector: 'app-user',
  providers: [UserService] // Service is available only to this component and its children
})
export class UserComponent {
  constructor(private userService: UserService) {}
}
```

### Custom Providers

#### 1. Class Provider

```typescript
@NgModule({
  providers: [
    { provide: LoggerService, useClass: ConsoleLoggerService }
  ]
})
export class AppModule { }
```

#### 2. Value Provider

```typescript
const API_CONFIG = {
  baseUrl: 'https://api.example.com',
  version: 'v1'
};

@NgModule({
  providers: [
    { provide: 'API_CONFIG', useValue: API_CONFIG }
  ]
})
export class AppModule { }
```

#### 3. Factory Provider

```typescript
@NgModule({
  providers: [
    {
      provide: DataService,
      useFactory: (http: HttpClient) => {
        return new DataService(http);
      },
      deps: [HttpClient]
    }
  ]
})
export class AppModule { }
```

### Optional Dependencies

```typescript
@Component({
  selector: 'app-optional',
  template: 'Optional Service Demo'
})
export class OptionalComponent {
  constructor(@Optional() private optionalService: OptionalService) {
    if (this.optionalService) {
      this.optionalService.doSomething();
    }
  }
}
```

### Self Dependencies

```typescript
@Component({
  selector: 'app-self',
  providers: [SelfService],
  template: 'Self Service Demo'
})
export class SelfComponent {
  constructor(@Self() private selfService: SelfService) {
    // This will only look for SelfService in this component's providers
  }
}
```

### SkipSelf Dependencies

```typescript
@Component({
  selector: 'app-skip-self',
  providers: [SkipSelfService],
  template: 'SkipSelf Service Demo'
})
export class SkipSelfComponent {
  constructor(@SkipSelf() private skipSelfService: SkipSelfService) {
    // This will skip the current component's providers and look in parent components
  }
}
```

### Host Dependencies

```typescript
@Component({
  selector: 'app-host',
  template: 'Host Service Demo'
})
export class HostComponent {
  constructor(@Host() private hostService: HostService) {
    // This will only look for HostService in the current component or its host element
  }
}
```

### Multi Providers

```typescript
@NgModule({
  providers: [
    { provide: 'LOGGERS', useClass: ConsoleLogger, multi: true },
    { provide: 'LOGGERS', useClass: FileLogger, multi: true }
  ]
})
export class AppModule { }

// Usage
@Injectable()
export class LoggingService {
  constructor(@Inject('LOGGERS') private loggers: Logger[]) {}
  
  log(message: string) {
    this.loggers.forEach(logger => logger.log(message));
  }
}
```

### Dependency Injection in Standalone Components

```typescript
@Component({
  selector: 'app-standalone',
  standalone: true,
  imports: [CommonModule],
  providers: [StandaloneService],
  template: 'Standalone Component'
})
export class StandaloneComponent {
  constructor(private standaloneService: StandaloneService) {}
}
```

### Best Practices

1. **Use Root Level for Global Services**:
   ```typescript
   @Injectable({
     providedIn: 'root'
   })
   export class GlobalService {}
   ```

2. **Use Module Level for Feature-Specific Services**:
   ```typescript
   @NgModule({
     providers: [FeatureService]
   })
   export class FeatureModule {}
   ```

3. **Use Component Level for Component-Specific Services**:
   ```typescript
   @Component({
     providers: [ComponentService]
   })
   export class MyComponent {}
   ```

4. **Use Factory Providers for Complex Initialization**:
   ```typescript
   providers: [
     {
       provide: ComplexService,
       useFactory: (config: ConfigService) => {
         return new ComplexService(config);
       },
       deps: [ConfigService]
     }
   ]
   ```

5. **Use Value Providers for Configuration**:
   ```typescript
   providers: [
     { provide: 'API_URL', useValue: 'https://api.example.com' }
   ]
   ```

### Common Pitfalls

1. **Circular Dependencies**:
   ```typescript
   // Avoid this
   @Injectable()
   export class ServiceA {
     constructor(private serviceB: ServiceB) {}
   }

   @Injectable()
   export class ServiceB {
     constructor(private serviceA: ServiceA) {}
   }
   ```

2. **Multiple Instances**:
   ```typescript
   // Be careful with component-level providers
   @Component({
     providers: [SharedService] // Creates new instance for each component
   })
   ```

3. **Missing Providers**:
   ```typescript
   // Always provide required dependencies
   @NgModule({
     providers: [RequiredService]
   })
   ```

### Testing with Dependency Injection

```typescript
describe('UserComponent', () => {
  let component: UserComponent;
  let userService: jasmine.SpyObj<UserService>;

  beforeEach(() => {
    userService = jasmine.createSpyObj('UserService', ['getUsers']);
    
    TestBed.configureTestingModule({
      declarations: [UserComponent],
      providers: [
        { provide: UserService, useValue: userService }
      ]
    });
    
    component = TestBed.createComponent(UserComponent).componentInstance;
  });

  it('should load users', () => {
    const mockUsers = [{ id: 1, name: 'John' }];
    userService.getUsers.and.returnValue(of(mockUsers));
    
    component.ngOnInit();
    
    expect(component.users).toEqual(mockUsers);
  });
});
```

### Conclusion

Angular's dependency injection system is a powerful feature that helps in:
- Making components more maintainable and testable
- Managing service instances efficiently
- Providing flexibility in how dependencies are created and shared
- Supporting different scopes for services (root, module, component)
- Enabling advanced features like optional dependencies and multi-providers

Understanding how to effectively use dependency injection is crucial for building scalable and maintainable Angular applications. By following best practices and avoiding common pitfalls, you can leverage the DI system to create well-structured and efficient applications. 