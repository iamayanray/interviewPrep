# Component Communication in Angular

## Question
What are the different ways components can communicate with each other in Angular? Explain parent-child communication, service-based communication, and other methods.

## Answer

### Introduction to Component Communication

In Angular applications, components are the building blocks of the user interface. As applications grow in complexity, these components need to share data and communicate with each other. Angular provides several mechanisms for component communication, each suited for different scenarios.

### 1. Parent to Child Communication: @Input Decorator

The most straightforward way for a parent component to pass data to a child component is through property binding using the `@Input` decorator.

#### Parent Component

```typescript
// parent.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-parent',
  template: `
    <h2>Parent Component</h2>
    <app-child [data]="parentData" [user]="currentUser"></app-child>
    <button (click)="updateData()">Update Data</button>
  `
})
export class ParentComponent {
  parentData = 'Data from parent';
  currentUser = { name: 'John', role: 'Admin' };

  updateData() {
    this.parentData = 'Updated data from parent';
    this.currentUser = { ...this.currentUser, role: 'Super Admin' };
  }
}
```

#### Child Component

```typescript
// child.component.ts
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-child',
  template: `
    <h3>Child Component</h3>
    <p>Data received: {{ data }}</p>
    <p>User: {{ user.name }} ({{ user.role }})</p>
  `
})
export class ChildComponent implements OnChanges {
  @Input() data: string;
  @Input() user: { name: string, role: string };

  ngOnChanges(changes: SimpleChanges) {
    // This lifecycle hook is called when an input property changes
    console.log('Input property changed:', changes);
  }
}
```

### 2. Child to Parent Communication: @Output Decorator and EventEmitter

Children can communicate with their parent components by emitting events using the `@Output` decorator and `EventEmitter`.

#### Child Component

```typescript
// child.component.ts
import { Component, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-child',
  template: `
    <h3>Child Component</h3>
    <button (click)="sendMessage()">Send Message to Parent</button>
    <button (click)="sendData()">Send Data to Parent</button>
  `
})
export class ChildComponent {
  @Output() messageEvent = new EventEmitter<string>();
  @Output() dataEvent = new EventEmitter<any>();

  sendMessage() {
    this.messageEvent.emit('Hello from child!');
  }

  sendData() {
    const data = {
      id: 1,
      timestamp: new Date(),
      value: Math.random() * 100
    };
    this.dataEvent.emit(data);
  }
}
```

#### Parent Component

```typescript
// parent.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-parent',
  template: `
    <h2>Parent Component</h2>
    <p *ngIf="message">Message from child: {{ message }}</p>
    <p *ngIf="data">Data from child: ID {{ data.id }}, Value: {{ data.value }}</p>
    <app-child 
      (messageEvent)="receiveMessage($event)"
      (dataEvent)="receiveData($event)">
    </app-child>
  `
})
export class ParentComponent {
  message: string;
  data: any;

  receiveMessage(event: string) {
    this.message = event;
  }

  receiveData(event: any) {
    this.data = event;
    console.log('Received data:', this.data);
  }
}
```

### 3. Parent to View Child: @ViewChild Decorator

The `@ViewChild` decorator allows a parent component to access properties and methods of a child component directly.

#### Child Component

```typescript
// child.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-child',
  template: `<h3>Child Component</h3>`
})
export class ChildComponent {
  message = 'Hello from child component!';

  showMessage() {
    console.log(this.message);
    return this.message;
  }

  updateMessage(newMessage: string) {
    this.message = newMessage;
  }
}
```

#### Parent Component

```typescript
// parent.component.ts
import { Component, ViewChild, AfterViewInit } from '@angular/core';
import { ChildComponent } from './child.component';

@Component({
  selector: 'app-parent',
  template: `
    <h2>Parent Component</h2>
    <button (click)="accessChildMethod()">Access Child Method</button>
    <button (click)="updateChildProperty()">Update Child Property</button>
    <app-child></app-child>
  `
})
export class ParentComponent implements AfterViewInit {
  @ViewChild(ChildComponent) childComponent: ChildComponent;

  ngAfterViewInit() {
    // Note: You should access ViewChild properties in or after ngAfterViewInit
    console.log('Child message:', this.childComponent.message);
  }

  accessChildMethod() {
    const message = this.childComponent.showMessage();
    console.log('Called child method, got:', message);
  }

  updateChildProperty() {
    this.childComponent.updateMessage('Updated from parent');
    console.log('Updated child message:', this.childComponent.message);
  }
}
```

### 4. Parent to Content Child: @ContentChild and ng-content

The `@ContentChild` decorator and `<ng-content>` allow a parent component to project content into a child component.

#### Child Component

```typescript
// child.component.ts
import { Component, ContentChild, ElementRef, AfterContentInit } from '@angular/core';

@Component({
  selector: 'app-child',
  template: `
    <h3>Child Component</h3>
    <div class="content-container">
      <ng-content></ng-content>
    </div>
    <div class="specific-content">
      <ng-content select="[footer]"></ng-content>
    </div>
  `
})
export class ChildComponent implements AfterContentInit {
  @ContentChild('projectedParagraph') projectedParagraph: ElementRef;

  ngAfterContentInit() {
    if (this.projectedParagraph) {
      console.log('Projected content:', this.projectedParagraph.nativeElement.textContent);
    }
  }
}
```

#### Parent Component

```typescript
// parent.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-parent',
  template: `
    <h2>Parent Component</h2>
    <app-child>
      <p #projectedParagraph>This content is projected from parent to child</p>
      <div>Another projected content</div>
      <div footer>This goes to the footer section</div>
    </app-child>
  `
})
export class ParentComponent {}
```

### 5. Sibling Communication via Parent Component

Siblings can communicate through their parent component by combining `@Output` and `@Input` decorators.

#### Parent Component

```typescript
// parent.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-parent',
  template: `
    <h2>Parent Component</h2>
    <app-sibling-a (dataEvent)="receiveDataFromA($event)"></app-sibling-a>
    <app-sibling-b [dataFromA]="dataFromSiblingA"></app-sibling-b>
  `
})
export class ParentComponent {
  dataFromSiblingA: any;

  receiveDataFromA(data: any) {
    this.dataFromSiblingA = data;
  }
}
```

#### Sibling A Component

```typescript
// sibling-a.component.ts
import { Component, Output, EventEmitter } from '@angular/core';

@Component({
  selector: 'app-sibling-a',
  template: `
    <h3>Sibling A Component</h3>
    <button (click)="sendData()">Send Data to Sibling B</button>
  `
})
export class SiblingAComponent {
  @Output() dataEvent = new EventEmitter<any>();

  sendData() {
    const data = {
      message: 'Hello from Sibling A',
      timestamp: new Date()
    };
    this.dataEvent.emit(data);
  }
}
```

#### Sibling B Component

```typescript
// sibling-b.component.ts
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-sibling-b',
  template: `
    <h3>Sibling B Component</h3>
    <div *ngIf="dataFromA">
      <p>Message: {{ dataFromA.message }}</p>
      <p>Timestamp: {{ dataFromA.timestamp | date:'medium' }}</p>
    </div>
  `
})
export class SiblingBComponent implements OnChanges {
  @Input() dataFromA: any;

  ngOnChanges(changes: SimpleChanges) {
    if (changes['dataFromA'] && changes['dataFromA'].currentValue) {
      console.log('Received data from Sibling A:', this.dataFromA);
    }
  }
}
```

### 6. Service-based Communication

For components that are not directly related (not parent-child), a shared service with observables is the recommended approach.

#### Shared Service

```typescript
// data.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable } from 'rxjs';

@Injectable({
  providedIn: 'root'
})
export class DataService {
  private messageSource = new BehaviorSubject<string>('Initial message');
  private dataSource = new BehaviorSubject<any>(null);

  // Expose observables for components to subscribe to
  currentMessage$ = this.messageSource.asObservable();
  currentData$ = this.dataSource.asObservable();

  // Methods to update the shared data
  updateMessage(message: string): void {
    this.messageSource.next(message);
  }

  updateData(data: any): void {
    this.dataSource.next(data);
  }
}
```

#### Component A

```typescript
// component-a.component.ts
import { Component } from '@angular/core';
import { DataService } from './data.service';

@Component({
  selector: 'app-component-a',
  template: `
    <h3>Component A</h3>
    <input [(ngModel)]="message" placeholder="Enter message">
    <button (click)="sendMessage()">Send Message</button>
    <button (click)="sendData()">Send Data</button>
  `
})
export class ComponentAComponent {
  message = '';

  constructor(private dataService: DataService) {}

  sendMessage() {
    this.dataService.updateMessage(this.message);
  }

  sendData() {
    const data = {
      source: 'Component A',
      value: Math.random() * 100,
      timestamp: new Date()
    };
    this.dataService.updateData(data);
  }
}
```

#### Component B

```typescript
// component-b.component.ts
import { Component, OnInit, OnDestroy } from '@angular/core';
import { DataService } from './data.service';
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-component-b',
  template: `
    <h3>Component B</h3>
    <p *ngIf="message">Received message: {{ message }}</p>
    <div *ngIf="data">
      <p>Source: {{ data.source }}</p>
      <p>Value: {{ data.value }}</p>
      <p>Timestamp: {{ data.timestamp | date:'medium' }}</p>
    </div>
  `
})
export class ComponentBComponent implements OnInit, OnDestroy {
  message: string;
  data: any;
  private subscriptions = new Subscription();

  constructor(private dataService: DataService) {}

  ngOnInit() {
    // Subscribe to the message observable
    this.subscriptions.add(
      this.dataService.currentMessage$.subscribe(message => {
        this.message = message;
        console.log('Component B received message:', message);
      })
    );

    // Subscribe to the data observable
    this.subscriptions.add(
      this.dataService.currentData$.subscribe(data => {
        this.data = data;
        if (data) {
          console.log('Component B received data:', data);
        }
      })
    );
  }

  ngOnDestroy() {
    // Unsubscribe to prevent memory leaks
    this.subscriptions.unsubscribe();
  }
}
```

### 7. Using NgRx Store for State Management

For complex applications with many components that need to share state, NgRx provides a robust state management solution.

#### Store Setup

```typescript
// app.state.ts
export interface AppState {
  message: string;
  data: any;
}

// message.actions.ts
import { createAction, props } from '@ngrx/store';

export const updateMessage = createAction(
  '[Message] Update',
  props<{ message: string }>()
);

export const updateData = createAction(
  '[Data] Update',
  props<{ data: any }>()
);

// message.reducer.ts
import { createReducer, on } from '@ngrx/store';
import * as MessageActions from './message.actions';

export interface MessageState {
  message: string;
  data: any;
}

export const initialState: MessageState = {
  message: 'Initial message',
  data: null
};

export const messageReducer = createReducer(
  initialState,
  on(MessageActions.updateMessage, (state, { message }) => ({
    ...state,
    message
  })),
  on(MessageActions.updateData, (state, { data }) => ({
    ...state,
    data
  }))
);
```

#### Component A with NgRx

```typescript
// component-a.component.ts
import { Component } from '@angular/core';
import { Store } from '@ngrx/store';
import { AppState } from './app.state';
import * as MessageActions from './message.actions';

@Component({
  selector: 'app-component-a',
  template: `
    <h3>Component A (NgRx)</h3>
    <input [(ngModel)]="message" placeholder="Enter message">
    <button (click)="sendMessage()">Send Message</button>
    <button (click)="sendData()">Send Data</button>
  `
})
export class ComponentAComponent {
  message = '';

  constructor(private store: Store<AppState>) {}

  sendMessage() {
    this.store.dispatch(MessageActions.updateMessage({ message: this.message }));
  }

  sendData() {
    const data = {
      source: 'Component A',
      value: Math.random() * 100,
      timestamp: new Date()
    };
    this.store.dispatch(MessageActions.updateData({ data }));
  }
}
```

#### Component B with NgRx

```typescript
// component-b.component.ts
import { Component, OnInit } from '@angular/core';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';
import { AppState } from './app.state';

@Component({
  selector: 'app-component-b',
  template: `
    <h3>Component B (NgRx)</h3>
    <p *ngIf="(message$ | async) as message">Received message: {{ message }}</p>
    <div *ngIf="(data$ | async) as data">
      <p>Source: {{ data.source }}</p>
      <p>Value: {{ data.value }}</p>
      <p>Timestamp: {{ data.timestamp | date:'medium' }}</p>
    </div>
  `
})
export class ComponentBComponent implements OnInit {
  message$: Observable<string>;
  data$: Observable<any>;

  constructor(private store: Store<AppState>) {}

  ngOnInit() {
    // Select the message from the store
    this.message$ = this.store.select(state => state.message);
    
    // Select the data from the store
    this.data$ = this.store.select(state => state.data);
  }
}
```

### 8. Using Query Parameters and Router State

Components can communicate through URL query parameters and router state.

#### Component A

```typescript
// component-a.component.ts
import { Component } from '@angular/core';
import { Router } from '@angular/router';

@Component({
  selector: 'app-component-a',
  template: `
    <h3>Component A (Router)</h3>
    <input [(ngModel)]="message" placeholder="Enter message">
    <button (click)="navigateWithParams()">Navigate with Params</button>
  `
})
export class ComponentAComponent {
  message = '';

  constructor(private router: Router) {}

  navigateWithParams() {
    this.router.navigate(['/component-b'], {
      queryParams: { message: this.message },
      state: { timestamp: new Date().toISOString() }
    });
  }
}
```

#### Component B

```typescript
// component-b.component.ts
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';

@Component({
  selector: 'app-component-b',
  template: `
    <h3>Component B (Router)</h3>
    <p *ngIf="message">Received message: {{ message }}</p>
    <p *ngIf="timestamp">Timestamp: {{ timestamp | date:'medium' }}</p>
    <button (click)="goBack()">Go Back</button>
  `
})
export class ComponentBComponent implements OnInit {
  message: string;
  timestamp: string;

  constructor(
    private route: ActivatedRoute,
    private router: Router
  ) {
    // Access router state
    const navigation = this.router.getCurrentNavigation();
    if (navigation && navigation.extras.state) {
      this.timestamp = navigation.extras.state['timestamp'];
    }
  }

  ngOnInit() {
    // Access query parameters
    this.route.queryParams.subscribe(params => {
      this.message = params['message'];
    });
  }

  goBack() {
    this.router.navigate(['/component-a']);
  }
}
```

### 9. Using Local Storage or Session Storage

Components can communicate by storing and retrieving data from browser storage.

#### Storage Service

```typescript
// storage.service.ts
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class StorageService {
  setItem(key: string, value: any): void {
    localStorage.setItem(key, JSON.stringify(value));
  }

  getItem(key: string): any {
    const item = localStorage.getItem(key);
    return item ? JSON.parse(item) : null;
  }

  removeItem(key: string): void {
    localStorage.removeItem(key);
  }

  clear(): void {
    localStorage.clear();
  }
}
```

#### Component A

```typescript
// component-a.component.ts
import { Component } from '@angular/core';
import { StorageService } from './storage.service';

@Component({
  selector: 'app-component-a',
  template: `
    <h3>Component A (Storage)</h3>
    <input [(ngModel)]="message" placeholder="Enter message">
    <button (click)="saveMessage()">Save Message</button>
  `
})
export class ComponentAComponent {
  message = '';

  constructor(private storageService: StorageService) {}

  saveMessage() {
    this.storageService.setItem('message', {
      text: this.message,
      timestamp: new Date().toISOString()
    });
    console.log('Message saved to storage');
  }
}
```

#### Component B

```typescript
// component-b.component.ts
import { Component, OnInit } from '@angular/core';
import { StorageService } from './storage.service';

@Component({
  selector: 'app-component-b',
  template: `
    <h3>Component B (Storage)</h3>
    <button (click)="loadMessage()">Load Message</button>
    <div *ngIf="messageData">
      <p>Message: {{ messageData.text }}</p>
      <p>Timestamp: {{ messageData.timestamp | date:'medium' }}</p>
    </div>
  `
})
export class ComponentBComponent {
  messageData: any;

  constructor(private storageService: StorageService) {}

  loadMessage() {
    this.messageData = this.storageService.getItem('message');
    console.log('Message loaded from storage:', this.messageData);
  }
}
```

### Best Practices for Component Communication

1. **Choose the Right Method for the Relationship**:
   - Parent to Child: Use `@Input`
   - Child to Parent: Use `@Output` and `EventEmitter`
   - Unrelated Components: Use a service with observables or NgRx

2. **Keep Components Focused**:
   - Components should have a single responsibility
   - Avoid making components too dependent on each other

3. **Use Strong Typing**:
   - Define interfaces for data being passed between components
   - Avoid using `any` type when possible

4. **Handle Subscriptions Properly**:
   - Always unsubscribe from observables in `ngOnDestroy`
   - Consider using the async pipe when possible

5. **Consider Performance**:
   - Be mindful of change detection triggers
   - Use `OnPush` change detection strategy for performance-critical components

6. **Document Component Interfaces**:
   - Clearly document `@Input` and `@Output` properties
   - Use JSDoc comments to explain expected data formats

### Comparison of Communication Methods

| Method | Use Case | Pros | Cons |
|--------|----------|------|------|
| `@Input` | Parent to Child | Simple, direct | One-way communication |
| `@Output` | Child to Parent | Event-based, clear | Limited to parent-child |
| `@ViewChild` | Parent to Child | Direct access to methods | Tight coupling |
| `ng-content` | Content Projection | Flexible UI composition | Limited to UI elements |
| Service + RxJS | Any components | Decoupled, reactive | More complex setup |
| NgRx | Complex state management | Centralized state, time-travel debugging | Steep learning curve, boilerplate |
| Router | Cross-page communication | Persists across page refreshes | Limited data complexity |
| Local Storage | Cross-session persistence | Persists across browser sessions | Limited storage, serialization issues |

### Conclusion

Angular provides multiple ways for components to communicate, each with its own strengths and appropriate use cases. For simple parent-child relationships, `@Input` and `@Output` decorators are usually sufficient. For more complex scenarios involving unrelated components, services with observables or state management solutions like NgRx are more appropriate.

Choosing the right communication method depends on the specific requirements of your application, the relationship between components, and the complexity of the data being shared. By understanding these different approaches, you can build more maintainable and efficient Angular applications. 