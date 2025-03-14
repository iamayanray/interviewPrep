# Angular Decorators

## Question
What are Angular decorators? Explain @Component, @Directive, @Injectable, and @Pipe with examples.

## Answer

### What are Decorators?

Decorators are a design pattern and a TypeScript feature that allows you to annotate or modify classes and class members at design time. In Angular, decorators are used extensively to configure how Angular should process a class.

Decorators are functions that are prefixed with an `@` symbol and are placed immediately before the code you want to decorate. They provide metadata that tells Angular how to use the associated class.

### Key Angular Decorators

#### 1. @Component

The `@Component` decorator identifies a class as an Angular component and provides configuration metadata that determines how the component should be processed, instantiated, and used at runtime.

```typescript
import { Component } from '@angular/core';

@Component({
  // CSS selector that identifies this component in a template
  selector: 'app-user-profile',
  
  // URL to the component's HTML template
  templateUrl: './user-profile.component.html',
  // Alternatively, inline template:
  // template: `<h1>{{name}}</h1><p>{{bio}}</p>`,
  
  // URLs to the component's CSS stylesheets
  styleUrls: ['./user-profile.component.css'],
  // Alternatively, inline styles:
  // styles: [`h1 { color: blue; }`],
  
  // View encapsulation strategy
  encapsulation: ViewEncapsulation.Emulated,
  
  // Change detection strategy
  changeDetection: ChangeDetectionStrategy.OnPush,
  
  // Components that can be used in the template without being declared in the module
  standalone: true,
  imports: [CommonModule, UserAvatarComponent],
  
  // Animation definitions
  animations: [fadeInAnimation],
  
  // Providers available to this component and its children
  providers: [UserService]
})
export class UserProfileComponent {
  name = 'John Doe';
  bio = 'Software Developer';
  
  constructor() {
    console.log('Component created');
  }
}
```

**Key Properties of @Component:**
- `selector`: CSS selector that identifies this component in a template
- `template`/`templateUrl`: The HTML template for the component
- `styles`/`styleUrls`: CSS styles for the component
- `providers`: Services that this component requires
- `changeDetection`: Change detection strategy (Default or OnPush)
- `encapsulation`: View encapsulation strategy
- `animations`: Animation definitions
- `standalone`: Whether the component is standalone (Angular 14+)
- `imports`: Components, directives, and pipes that can be used in the template (for standalone components)

#### 2. @Directive

The `@Directive` decorator declares a class as an Angular directive, which allows you to attach custom behavior to elements in the DOM.

There are three types of directives in Angular:
1. Components (with templates) - using @Component
2. Structural directives (change DOM layout) - using @Directive
3. Attribute directives (change appearance or behavior) - using @Directive

```typescript
import { Directive, ElementRef, HostListener, Input } from '@angular/core';

@Directive({
  // CSS selector that identifies this directive in a template
  selector: '[appHighlight]',
  
  // Whether this directive can be used in standalone components without importing
  standalone: true,
  
  // Providers available to this directive
  providers: []
})
export class HighlightDirective {
  // Input property to receive data from the parent component
  @Input('appHighlight') highlightColor = 'yellow';
  
  constructor(private el: ElementRef) {}
  
  // Host listener to respond to events on the host element
  @HostListener('mouseenter') onMouseEnter() {
    this.highlight(this.highlightColor);
  }
  
  @HostListener('mouseleave') onMouseLeave() {
    this.highlight(null);
  }
  
  private highlight(color: string | null) {
    this.el.nativeElement.style.backgroundColor = color;
  }
}
```

**Usage in a template:**
```html
<p appHighlight="lightblue">Highlight me in light blue</p>
<p appHighlight>Highlight me in yellow (default)</p>
```

**Key Properties of @Directive:**
- `selector`: CSS selector that identifies this directive in a template
- `providers`: Services that this directive requires
- `standalone`: Whether the directive is standalone (Angular 14+)
- `host`: Host element bindings
- `exportAs`: Name that can be used in the template to assign this directive to a variable

#### 3. @Injectable

The `@Injectable` decorator marks a class as available to be provided and injected as a dependency. It's primarily used for services in Angular.

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { User } from './user.model';

@Injectable({
  // Service is registered at the root level and available application-wide
  providedIn: 'root',
  
  // Optional: specify a custom factory function
  // useFactory: () => new UserService(inject(HttpClient))
})
export class UserService {
  private apiUrl = 'https://api.example.com/users';
  
  constructor(private http: HttpClient) {}
  
  getUsers(): Observable<User[]> {
    return this.http.get<User[]>(this.apiUrl);
  }
  
  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`);
  }
  
  createUser(user: User): Observable<User> {
    return this.http.post<User>(this.apiUrl, user);
  }
}
```

**Key Properties of @Injectable:**
- `providedIn`: Specifies where the service should be provided:
  - `'root'`: Service is available application-wide as a singleton
  - `'platform'`: Service is available in all Angular applications on the page
  - `'any'`: Service has a separate instance in each lazy-loaded module
  - A specific NgModule: Service is available only within that module

**Using the service in a component:**
```typescript
import { Component } from '@angular/core';
import { UserService } from './user.service';
import { User } from './user.model';

@Component({
  selector: 'app-user-list',
  template: `
    <h2>User List</h2>
    <ul>
      <li *ngFor="let user of users">{{ user.name }}</li>
    </ul>
  `
})
export class UserListComponent {
  users: User[] = [];
  
  constructor(private userService: UserService) {
    this.userService.getUsers().subscribe(users => {
      this.users = users;
    });
  }
}
```

#### 4. @Pipe

The `@Pipe` decorator defines a class that transforms input values to output values for display in a template.

```typescript
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  // Name used in template expressions
  name: 'truncate',
  
  // Whether the pipe is pure or impure
  // Pure pipes are only re-evaluated when input value reference changes
  // Impure pipes are re-evaluated on each change detection cycle
  pure: true,
  
  // Whether this pipe can be used in standalone components without importing
  standalone: true
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit: number = 100, completeWords: boolean = false, ellipsis: string = '...'): string {
    if (!value) {
      return '';
    }
    
    if (value.length <= limit) {
      return value;
    }
    
    if (completeWords) {
      limit = value.substring(0, limit).lastIndexOf(' ');
    }
    
    return `${value.substring(0, limit)}${ellipsis}`;
  }
}
```

**Usage in a template:**
```html
<!-- Truncate to 50 characters -->
<p>{{ longText | truncate:50 }}</p>

<!-- Truncate to 100 characters, complete words, custom ellipsis -->
<p>{{ longText | truncate:100:true:'... Read more' }}</p>
```

**Key Properties of @Pipe:**
- `name`: The name used in template expressions
- `pure`: Whether the pipe is pure (default) or impure
- `standalone`: Whether the pipe is standalone (Angular 14+)

### Other Important Angular Decorators

#### @Input and @Output

These decorators are used for component communication:

```typescript
import { Component, Input, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-child',
  template: `
    <h2>{{ title }}</h2>
    <button (click)="incrementCounter()">Increment</button>
  `
})
export class ChildComponent {
  // Input property to receive data from parent
  @Input() title: string = 'Default Title';
  
  // Output property to send events to parent
  @Output() counterChanged = new EventEmitter<number>();
  
  private counter = 0;
  
  incrementCounter() {
    this.counter++;
    this.counterChanged.emit(this.counter);
  }
}
```

**Usage in parent component:**
```html
<app-child 
  [title]="parentTitle" 
  (counterChanged)="handleCounterChange($event)">
</app-child>
```

#### @HostBinding and @HostListener

These decorators are used to interact with the host element:

```typescript
import { Directive, HostBinding, HostListener } from '@angular/core';

@Directive({
  selector: '[appToggleClass]'
})
export class ToggleClassDirective {
  // Bind to a property of the host element
  @HostBinding('class.active') isActive = false;
  
  // Listen to events on the host element
  @HostListener('click') onClick() {
    this.isActive = !this.isActive;
  }
}
```

#### @ViewChild and @ContentChild

These decorators are used to access child components, directives, or DOM elements:

```typescript
import { Component, ViewChild, AfterViewInit, ElementRef } from '@angular/core';
import { ChildComponent } from './child.component';

@Component({
  selector: 'app-parent',
  template: `
    <div>
      <h1 #headerElement>Parent Component</h1>
      <app-child></app-child>
    </div>
  `
})
export class ParentComponent implements AfterViewInit {
  // Access a DOM element
  @ViewChild('headerElement') headerElement: ElementRef;
  
  // Access a child component
  @ViewChild(ChildComponent) childComponent: ChildComponent;
  
  ngAfterViewInit() {
    // Access DOM element
    console.log(this.headerElement.nativeElement.textContent);
    
    // Access child component properties and methods
    console.log(this.childComponent.title);
    this.childComponent.someMethod();
  }
}
```

### Creating Custom Decorators

You can also create your own custom decorators in Angular:

```typescript
// Custom method decorator
function Log() {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    // Save a reference to the original method
    const originalMethod = descriptor.value;
    
    // Rewrite the method
    descriptor.value = function(...args: any[]) {
      console.log(`Calling ${propertyKey} with:`, args);
      
      // Call the original method
      const result = originalMethod.apply(this, args);
      
      console.log(`Result:`, result);
      return result;
    };
    
    return descriptor;
  };
}

// Usage
class Calculator {
  @Log()
  add(a: number, b: number): number {
    return a + b;
  }
}

const calc = new Calculator();
calc.add(5, 3); // Logs method call and result
```

### Conclusion

Decorators are a fundamental part of Angular's architecture. They provide a clean, declarative way to add metadata to classes and their members, enabling Angular to understand how to process and use these classes. The most commonly used decorators are `@Component`, `@Directive`, `@Injectable`, and `@Pipe`, but Angular provides many more decorators to handle various aspects of application development.

Understanding how decorators work and how to use them effectively is essential for Angular development, as they form the backbone of Angular's component-based architecture. 