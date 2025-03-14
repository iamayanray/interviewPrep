# Angular Interview Preparation Guide
## A Comprehensive Guide to Angular Interview Questions and Answers

---

# Table of Contents

1. [Core Concepts](#core-concepts) .................................................. 1
   - [What is Angular?](#what-is-angular)
   - [Angular Architecture](#angular-architecture)
   - [Components](#components)
   - [Modules](#modules)
   - [Services](#services)
   - [Dependency Injection](#dependency-injection)

2. [Data Binding and Templates](#data-binding-and-templates) ........................ 15
   - [Interpolation](#interpolation)
   - [Property Binding](#property-binding)
   - [Event Binding](#event-binding)
   - [Two-way Binding](#two-way-binding)
   - [Template Reference Variables](#template-reference-variables)

3. [Directives](#directives) ..................................................... 25
   - [Structural Directives](#structural-directives)
   - [Attribute Directives](#attribute-directives)
   - [Custom Directives](#custom-directives)

4. [Pipes](#pipes) ............................................................. 35
   - [Built-in Pipes](#built-in-pipes)
   - [Custom Pipes](#custom-pipes)
   - [Pure vs Impure Pipes](#pure-vs-impure-pipes)

5. [Routing](#routing) ......................................................... 45
   - [Basic Routing](#basic-routing)
   - [Route Guards](#route-guards)
   - [Child Routes](#child-routes)
   - [Lazy Loading](#lazy-loading)

6. [Forms](#forms) ............................................................. 55
   - [Template-driven Forms](#template-driven-forms)
   - [Reactive Forms](#reactive-forms)
   - [Form Validation](#form-validation)
   - [Custom Validators](#custom-validators)

7. [HTTP Client](#http-client) ................................................. 65
   - [Making HTTP Requests](#making-http-requests)
   - [Error Handling](#error-handling)
   - [Interceptors](#interceptors)
   - [HTTP Headers](#http-headers)

8. [RxJS and Observables](#rxjs-and-observables) .................................. 75
   - [Observables](#observables)
   - [Operators](#operators)
   - [Subjects](#subjects)
   - [Error Handling](#error-handling)

9. [State Management](#state-management) ......................................... 85
   - [NgRx](#ngrx)
   - [Services](#services)
   - [BehaviorSubject](#behaviorsubject)
   - [State Management Patterns](#state-management-patterns)

10. [Testing](#testing) ......................................................... 95
    - [Unit Testing](#unit-testing)
    - [Integration Testing](#integration-testing)
    - [E2E Testing](#e2e-testing)
    - [Testing Best Practices](#testing-best-practices)

11. [Performance Optimization](#performance-optimization) .......................... 105
    - [Change Detection](#change-detection)
    - [Lazy Loading](#lazy-loading)
    - [Memory Management](#memory-management)
    - [Performance Best Practices](#performance-best-practices)

12. [Security](#security) ........................................................ 115
    - [XSS Protection](#xss-protection)
    - [CSRF Protection](#csrf-protection)
    - [Authentication](#authentication)
    - [Authorization](#authorization)

13. [Advanced Topics](#advanced-topics) ......................................... 125
    - [Dynamic Components](#dynamic-components)
    - [Custom Decorators](#custom-decorators)
    - [Angular Elements](#angular-elements)
    - [Micro Frontends](#micro-frontends)

14. [Interview Scenarios](#interview-scenarios) .................................. 135
    - [Large Data Table Performance](#large-data-table-performance)
    - [Real-time Data Updates](#real-time-data-updates)
    - [Complex Form Validation](#complex-form-validation)
    - [State Management with NgRx](#state-management-with-ngrx)
    - [Authentication and Authorization](#authentication-and-authorization)

15. [Best Practices](#best-practices) ........................................... 155
    - [Code Organization](#code-organization)
    - [Naming Conventions](#naming-conventions)
    - [Error Handling](#error-handling)
    - [Documentation](#documentation)

---

# Chapter 1: Core Concepts

## What is Angular?

Angular is a TypeScript-based open-source web application framework led by the Angular Team at Google. It's a complete rewrite of AngularJS and provides a robust framework for building scalable web applications.

### Key Features:
- Component-based architecture
- TypeScript support
- Dependency injection
- Two-way data binding
- Routing
- Forms handling
- HTTP client
- Testing utilities

### Example:
```typescript
// app.component.ts
@Component({
  selector: 'app-root',
  template: `
    <h1>{{ title }}</h1>
    <nav>
      <a routerLink="/home">Home</a>
      <a routerLink="/about">About</a>
    </nav>
    <router-outlet></router-outlet>
  `
})
export class AppComponent {
  title = 'My Angular App';
}
```

## Angular Architecture

Angular follows a component-based architecture with the following key concepts:

1. **Components**: Basic building blocks of the UI
2. **Modules**: Organize code into functional sets
3. **Services**: Share data and functionality
4. **Routing**: Handle navigation
5. **Dependency Injection**: Manage dependencies

### Example:
```typescript
// app.module.ts
@NgModule({
  declarations: [
    AppComponent,
    HomeComponent,
    AboutComponent
  ],
  imports: [
    BrowserModule,
    RouterModule.forRoot([
      { path: 'home', component: HomeComponent },
      { path: 'about', component: AboutComponent }
    ])
  ],
  providers: [DataService],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

[Continue with more chapters...]

---

# Chapter 14: Interview Scenarios

## Large Data Table Performance

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

[Continue with more scenarios...]

---

# Chapter 15: Best Practices

## Code Organization

### Project Structure
```
src/
├── app/
│   ├── core/           # Singleton services, guards, interceptors
│   ├── shared/         # Shared components, directives, pipes
│   ├── features/       # Feature modules
│   └── app.module.ts
├── assets/
└── environments/
```

### Naming Conventions

1. **Components**:
   - Use kebab-case for file names
   - Use PascalCase for class names
   - Suffix with 'Component'

2. **Services**:
   - Use kebab-case for file names
   - Use PascalCase for class names
   - Suffix with 'Service'

3. **Directives**:
   - Use kebab-case for file names
   - Use PascalCase for class names
   - Suffix with 'Directive'

### Example:
```typescript
// user-profile.component.ts
@Component({
  selector: 'app-user-profile',
  templateUrl: './user-profile.component.html'
})
export class UserProfileComponent {}

// auth.service.ts
@Injectable({
  providedIn: 'root'
})
export class AuthService {}

// highlight.directive.ts
@Directive({
  selector: '[appHighlight]'
})
export class HighlightDirective {}
```

[Continue with more best practices...]

---

# Appendix

## Common Interview Questions

1. What is the difference between Promise and Observable?
2. How does Angular's change detection work?
3. What is the purpose of NgModule?
4. How do you handle errors in Angular?
5. What is the difference between constructor and ngOnInit?

## Resources

- [Angular Official Documentation](https://angular.io/docs)
- [Angular GitHub Repository](https://github.com/angular/angular)
- [Angular Blog](https://blog.angular.io/)
- [Angular Community](https://angular.io/community)

---

# Index

A
- Angular Architecture .................................................. 1
- Angular Elements ..................................................... 125
- Authentication ....................................................... 115
- Authorization ........................................................ 115

B
- Best Practices ....................................................... 155
- Built-in Pipes ....................................................... 35

C
- Change Detection ..................................................... 105
- Components ......................................................... 1
- Custom Decorators ................................................... 125
- Custom Directives ................................................... 25
- Custom Pipes ........................................................ 35
- Custom Validators ................................................... 55

D
- Data Binding ........................................................ 15
- Dependency Injection ............................................... 1
- Dynamic Components ................................................. 125

E
- Error Handling ....................................................... 65
- Event Binding ....................................................... 15

F
- Forms ............................................................... 55
- Form Validation ..................................................... 55

H
- HTTP Client ......................................................... 65
- HTTP Headers ....................................................... 65

I
- Interceptors ........................................................ 65
- Interpolation ....................................................... 15

L
- Lazy Loading ....................................................... 45
- Large Data Table Performance ........................................ 135

M
- Memory Management .................................................. 105
- Micro Frontends .................................................... 125
- Modules ........................................................... 1

N
- Naming Conventions ................................................. 155
- NgRx ............................................................... 85

O
- Observables ........................................................ 75
- Operators ......................................................... 75

P
- Performance Optimization ............................................. 105
- Pipes ............................................................... 35
- Property Binding ................................................... 15
- Pure vs Impure Pipes ............................................... 35

R
- Reactive Forms ..................................................... 55
- Real-time Data Updates ............................................. 135
- Route Guards ....................................................... 45
- Routing ........................................................... 45
- RxJS ............................................................... 75

S
- Security ........................................................... 115
- Services ........................................................... 1
- State Management ................................................... 85
- State Management Patterns ........................................... 85
- Structural Directives ............................................... 25
- Subjects ........................................................... 75

T
- Template-driven Forms ............................................... 55
- Template Reference Variables ........................................ 15
- Testing ........................................................... 95
- Testing Best Practices ............................................. 95
- Two-way Binding ................................................... 15

U
- Unit Testing ....................................................... 95

X
- XSS Protection ..................................................... 115

---

# About the Author

This Angular Interview Preparation Guide was created to help developers prepare for Angular interviews. The guide covers core concepts, advanced topics, and real-world scenarios commonly encountered in Angular development.

---

# Copyright

© 2024 Angular Interview Preparation Guide. All rights reserved.

---

# Version History

- Version 1.0 (2024)
  - Initial release
  - Comprehensive coverage of Angular topics
  - Real-world scenarios and examples 