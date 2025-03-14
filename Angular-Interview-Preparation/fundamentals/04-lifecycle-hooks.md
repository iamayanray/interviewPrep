# Angular Lifecycle Hooks

## Question
Explain the Angular lifecycle hooks. Provide examples of when you would use each hook. (e.g., ngOnInit, ngOnChanges, ngOnDestroy)

## Answer

Angular components and directives go through a lifecycle from creation to destruction. Angular provides hooks that allow you to tap into key moments in this lifecycle to perform specific actions. These hooks are implemented as interfaces that your component or directive class can implement.

### Overview of Lifecycle Sequence

When Angular creates a component, it goes through the following lifecycle hooks in this order:

1. **ngOnChanges**: Called when an input property changes
2. **ngOnInit**: Called once after the first ngOnChanges
3. **ngDoCheck**: Called during every change detection run
4. **ngAfterContentInit**: Called after content (ng-content) has been projected into the component
5. **ngAfterContentChecked**: Called after every check of projected content
6. **ngAfterViewInit**: Called after the component's view (and child views) has been initialized
7. **ngAfterViewChecked**: Called after every check of the component's view (and child views)
8. **ngOnDestroy**: Called just before the component is destroyed

Let's explore each hook in detail with examples of when and how to use them.

### 1. ngOnChanges

**Interface**: `OnChanges`

**Purpose**: Responds when Angular sets or resets data-bound input properties. The method receives a `SimpleChanges` object containing current and previous property values.

**When to use**:
- When you need to react to changes in input properties
- When you need to perform actions based on the changed values
- When you need to validate input property values

**Example**:

```typescript
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-child',
  template: `
    <div>
      <h2>Child Component</h2>
      <p>Name: {{ name }}</p>
      <p>Version: {{ version }}</p>
      <p>Change Count: {{ changeCount }}</p>
    </div>
  `
})
export class ChildComponent implements OnChanges {
  @Input() name: string;
  @Input() version: number;
  
  changeCount = 0;
  
  ngOnChanges(changes: SimpleChanges) {
    // Increment change count
    this.changeCount++;
    
    // Log all changes
    for (const propName in changes) {
      const change = changes[propName];
      const current = JSON.stringify(change.currentValue);
      const previous = JSON.stringify(change.previousValue);
      
      console.log(`${propName}: currentValue = ${current}, previousValue = ${previous}`);
    }
    
    // React to specific property changes
    if (changes['name'] && !changes['name'].firstChange) {
      console.log(`Name changed from ${changes['name'].previousValue} to ${changes['name'].currentValue}`);
      // Perform some action based on name change
    }
    
    // Validate input
    if (changes['version'] && changes['version'].currentValue < 0) {
      console.warn('Version cannot be negative');
      this.version = 0; // Correct invalid input
    }
  }
}
```

**Parent component usage**:

```typescript
@Component({
  selector: 'app-parent',
  template: `
    <div>
      <h1>Parent Component</h1>
      <button (click)="updateName()">Update Name</button>
      <button (click)="incrementVersion()">Increment Version</button>
      <app-child [name]="name" [version]="version"></app-child>
    </div>
  `
})
export class ParentComponent {
  name = 'Initial Name';
  version = 1;
  
  updateName() {
    this.name = 'Updated Name ' + new Date().toLocaleTimeString();
  }
  
  incrementVersion() {
    this.version++;
  }
}
```

### 2. ngOnInit

**Interface**: `OnInit`

**Purpose**: Called once after the first ngOnChanges. Used for initialization logic.

**When to use**:
- For one-time initialization tasks
- To set up component properties
- To fetch initial data from services
- To set up subscriptions to observables

**Example**:

```typescript
import { Component, OnInit } from '@angular/core';
import { UserService } from './user.service';
import { User } from './user.model';

@Component({
  selector: 'app-user-profile',
  template: `
    <div *ngIf="user">
      <h2>{{ user.name }}</h2>
      <p>Email: {{ user.email }}</p>
      <p>Role: {{ user.role }}</p>
    </div>
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error">{{ error }}</div>
  `
})
export class UserProfileComponent implements OnInit {
  user: User | null = null;
  loading = true;
  error: string | null = null;
  
  constructor(private userService: UserService) {
    // Constructor should only be used for injecting dependencies
    // Avoid putting initialization logic here
    console.log('Constructor called');
  }
  
  ngOnInit() {
    console.log('ngOnInit called');
    
    // Initialize component properties
    this.loading = true;
    
    // Fetch data from a service
    this.userService.getCurrentUser().subscribe({
      next: (user) => {
        this.user = user;
        this.loading = false;
      },
      error: (err) => {
        this.error = 'Failed to load user profile';
        this.loading = false;
        console.error(err);
      }
    });
  }
}
```

### 3. ngDoCheck

**Interface**: `DoCheck`

**Purpose**: Called during every change detection run, immediately after ngOnChanges and ngOnInit.

**When to use**:
- When you need to implement custom change detection
- When Angular's change detection doesn't catch the changes you need to act on
- For detecting changes that Angular doesn't detect on its own (like changes within objects or arrays)

**Note**: Use this hook with caution as it can be called very frequently and can impact performance if not implemented carefully.

**Example**:

```typescript
import { Component, DoCheck, Input, KeyValueDiffer, KeyValueDiffers } from '@angular/core';

@Component({
  selector: 'app-user-card',
  template: `
    <div class="card">
      <h3>{{ user.name }}</h3>
      <p>Email: {{ user.email }}</p>
      <p>Last Updated: {{ lastUpdated | date:'medium' }}</p>
    </div>
  `
})
export class UserCardComponent implements DoCheck {
  @Input() user: any;
  lastUpdated: Date;
  
  private userDiffer: KeyValueDiffer<string, any>;
  
  constructor(private differs: KeyValueDiffers) {}
  
  ngDoCheck() {
    // Initialize the differ if it doesn't exist
    if (this.user && !this.userDiffer) {
      this.userDiffer = this.differs.find(this.user).create();
    }
    
    // Check if the user object has changed
    if (this.userDiffer) {
      const changes = this.userDiffer.diff(this.user);
      
      if (changes) {
        this.lastUpdated = new Date();
        
        // Log what changed
        changes.forEachChangedItem(item => {
          console.log(`Changed: ${item.key} from ${item.previousValue} to ${item.currentValue}`);
        });
        
        changes.forEachAddedItem(item => {
          console.log(`Added: ${item.key} with value ${item.currentValue}`);
        });
        
        changes.forEachRemovedItem(item => {
          console.log(`Removed: ${item.key} with value ${item.previousValue}`);
        });
      }
    }
  }
}
```

### 4. ngAfterContentInit

**Interface**: `AfterContentInit`

**Purpose**: Called once after the first ngDoCheck, when the component's content has been initialized. Content refers to the content projected into the component via `<ng-content>`.

**When to use**:
- When you need to interact with content projected into your component
- When you need to access elements projected via `<ng-content>` using `@ContentChild` or `@ContentChildren`

**Example**:

```typescript
import { 
  Component, 
  AfterContentInit, 
  ContentChild, 
  ContentChildren, 
  QueryList 
} from '@angular/core';
import { TabComponent } from './tab.component';

@Component({
  selector: 'app-tabs-container',
  template: `
    <div class="tabs-container">
      <div class="tabs-header">
        <button 
          *ngFor="let tab of tabs; let i = index" 
          [class.active]="selectedIndex === i"
          (click)="selectTab(i)">
          {{ tab.title }}
        </button>
      </div>
      
      <div class="tab-content">
        <ng-content></ng-content>
      </div>
    </div>
  `,
  styles: [`
    .tabs-container { border: 1px solid #ccc; }
    .tabs-header { display: flex; background: #f5f5f5; }
    button { padding: 10px; border: none; background: none; cursor: pointer; }
    button.active { border-bottom: 2px solid blue; font-weight: bold; }
    .tab-content { padding: 15px; }
  `]
})
export class TabsContainerComponent implements AfterContentInit {
  @ContentChildren(TabComponent) tabs: QueryList<TabComponent>;
  selectedIndex = 0;
  
  ngAfterContentInit() {
    console.log('Content initialized');
    
    // Make sure we have tabs
    if (this.tabs.length === 0) {
      console.warn('No tabs found in the tabs container');
      return;
    }
    
    // Set the first tab as active by default
    this.selectTab(0);
    
    // Subscribe to changes in the tabs collection
    this.tabs.changes.subscribe(tabs => {
      console.log('Tabs collection changed:', tabs);
      
      // If the active tab was removed, select the first tab
      if (this.selectedIndex >= tabs.length) {
        this.selectTab(0);
      }
    });
  }
  
  selectTab(index: number) {
    if (this.tabs && this.tabs.length > index) {
      // Deactivate all tabs
      this.tabs.forEach(tab => tab.active = false);
      
      // Activate the selected tab
      this.tabs.toArray()[index].active = true;
      this.selectedIndex = index;
    }
  }
}

// Child tab component
@Component({
  selector: 'app-tab',
  template: `
    <div [hidden]="!active">
      <ng-content></ng-content>
    </div>
  `
})
export class TabComponent {
  @Input() title: string;
  active = false;
}
```

**Usage**:

```html
<app-tabs-container>
  <app-tab title="Profile">
    <h3>User Profile</h3>
    <p>Profile content goes here...</p>
  </app-tab>
  
  <app-tab title="Settings">
    <h3>User Settings</h3>
    <p>Settings content goes here...</p>
  </app-tab>
  
  <app-tab title="Notifications">
    <h3>Notifications</h3>
    <p>Notifications content goes here...</p>
  </app-tab>
</app-tabs-container>
```

### 5. ngAfterContentChecked

**Interface**: `AfterContentChecked`

**Purpose**: Called after every check of the component's content by Angular's change detection mechanism.

**When to use**:
- When you need to respond to changes in content
- When you need to perform additional checks or actions after content has been checked

**Note**: This hook is called frequently, so be careful about performance implications.

**Example**:

```typescript
import { Component, AfterContentChecked, ContentChildren, QueryList } from '@angular/core';
import { FormFieldComponent } from './form-field.component';

@Component({
  selector: 'app-form-section',
  template: `
    <div class="form-section">
      <h3>{{ title }}</h3>
      <ng-content></ng-content>
      <div *ngIf="hasErrors" class="error-summary">
        Please fix the errors in this section.
      </div>
    </div>
  `,
  styles: [`
    .form-section { border: 1px solid #eee; padding: 15px; margin-bottom: 20px; }
    .error-summary { color: red; margin-top: 10px; }
  `]
})
export class FormSectionComponent implements AfterContentChecked {
  @Input() title: string;
  @ContentChildren(FormFieldComponent) fields: QueryList<FormFieldComponent>;
  
  hasErrors = false;
  
  ngAfterContentChecked() {
    // Check if any of the form fields have errors
    this.hasErrors = this.fields ? this.fields.some(field => field.hasError) : false;
    
    if (this.hasErrors) {
      console.log(`Errors detected in form section: ${this.title}`);
    }
  }
}

// Form field component
@Component({
  selector: 'app-form-field',
  template: `
    <div class="form-field" [class.has-error]="hasError">
      <label>{{ label }}</label>
      <ng-content></ng-content>
      <div *ngIf="hasError" class="error-message">{{ errorMessage }}</div>
    </div>
  `
})
export class FormFieldComponent {
  @Input() label: string;
  @Input() hasError = false;
  @Input() errorMessage = 'This field has an error';
}
```

### 6. ngAfterViewInit

**Interface**: `AfterViewInit`

**Purpose**: Called once after Angular initializes the component's views and child views.

**When to use**:
- When you need to interact with the DOM elements of the view
- When you need to initialize third-party UI libraries that need access to DOM elements
- When you need to access child components via `@ViewChild` or `@ViewChildren`

**Example**:

```typescript
import { 
  Component, 
  AfterViewInit, 
  ViewChild, 
  ElementRef, 
  OnDestroy 
} from '@angular/core';
import * as Chart from 'chart.js';

@Component({
  selector: 'app-dashboard-chart',
  template: `
    <div class="chart-container">
      <h3>{{ title }}</h3>
      <canvas #chartCanvas></canvas>
    </div>
  `,
  styles: [`
    .chart-container { width: 100%; max-width: 600px; margin: 0 auto; }
    canvas { width: 100% !important; height: 300px !important; }
  `]
})
export class DashboardChartComponent implements AfterViewInit, OnDestroy {
  @Input() title: string = 'Data Visualization';
  @Input() chartData: any;
  
  @ViewChild('chartCanvas') chartCanvas: ElementRef;
  
  private chart: Chart;
  
  ngAfterViewInit() {
    console.log('View initialized');
    
    // Initialize the chart after the view is ready
    this.initializeChart();
  }
  
  private initializeChart() {
    if (!this.chartCanvas || !this.chartData) {
      console.warn('Cannot initialize chart: Missing canvas or data');
      return;
    }
    
    const ctx = this.chartCanvas.nativeElement.getContext('2d');
    
    this.chart = new Chart(ctx, {
      type: 'bar',
      data: this.chartData,
      options: {
        responsive: true,
        maintainAspectRatio: false
      }
    });
  }
  
  ngOnDestroy() {
    // Clean up the chart when the component is destroyed
    if (this.chart) {
      this.chart.destroy();
    }
  }
}
```

### 7. ngAfterViewChecked

**Interface**: `AfterViewChecked`

**Purpose**: Called after every check of the component's view and child views by Angular's change detection mechanism.

**When to use**:
- When you need to respond to changes in the view
- When you need to update the DOM after Angular has checked the view
- When you need to synchronize with third-party UI libraries after view updates

**Note**: This hook is called frequently, so be careful about performance implications.

**Example**:

```typescript
import { 
  Component, 
  AfterViewChecked, 
  ViewChild, 
  ElementRef, 
  Renderer2 
} from '@angular/core';

@Component({
  selector: 'app-auto-resize-textarea',
  template: `
    <div class="textarea-container">
      <textarea 
        #textareaElement
        [value]="value"
        (input)="onInput($event)"
        [placeholder]="placeholder">
      </textarea>
    </div>
  `,
  styles: [`
    .textarea-container { width: 100%; }
    textarea { width: 100%; min-height: 50px; resize: none; }
  `]
})
export class AutoResizeTextareaComponent implements AfterViewChecked {
  @Input() value: string = '';
  @Input() placeholder: string = 'Enter text...';
  @Output() valueChange = new EventEmitter<string>();
  
  @ViewChild('textareaElement') textareaElement: ElementRef;
  
  private previousHeight: number;
  
  constructor(private renderer: Renderer2) {}
  
  onInput(event: Event) {
    const textarea = event.target as HTMLTextAreaElement;
    this.value = textarea.value;
    this.valueChange.emit(this.value);
  }
  
  ngAfterViewChecked() {
    this.adjustTextareaHeight();
  }
  
  private adjustTextareaHeight() {
    const textarea = this.textareaElement.nativeElement;
    
    // Reset height to auto to get the correct scrollHeight
    this.renderer.setStyle(textarea, 'height', 'auto');
    
    const newHeight = textarea.scrollHeight;
    
    // Only update if height has changed to avoid triggering another change detection cycle
    if (newHeight !== this.previousHeight) {
      this.renderer.setStyle(textarea, 'height', `${newHeight}px`);
      this.previousHeight = newHeight;
    }
  }
}
```

### 8. ngOnDestroy

**Interface**: `OnDestroy`

**Purpose**: Called just before Angular destroys the component or directive.

**When to use**:
- To clean up resources to prevent memory leaks
- To unsubscribe from observables
- To detach event handlers
- To clear timers
- To unregister from global or application services

**Example**:

```typescript
import { 
  Component, 
  OnInit, 
  OnDestroy 
} from '@angular/core';
import { Subject, interval, Subscription } from 'rxjs';
import { takeUntil } from 'rxjs/operators';
import { Router, NavigationStart } from '@angular/router';

@Component({
  selector: 'app-data-monitor',
  template: `
    <div class="monitor">
      <h3>Data Monitor</h3>
      <p>Updates: {{ updateCount }}</p>
      <p>Last Update: {{ lastUpdate | date:'medium' }}</p>
      <p>Status: {{ status }}</p>
      
      <button (click)="startMonitoring()" [disabled]="isMonitoring">Start</button>
      <button (click)="stopMonitoring()" [disabled]="!isMonitoring">Stop</button>
    </div>
  `
})
export class DataMonitorComponent implements OnInit, OnDestroy {
  updateCount = 0;
  lastUpdate: Date | null = null;
  status = 'Idle';
  isMonitoring = false;
  
  // For the first approach: using a subscription
  private dataSubscription: Subscription | null = null;
  
  // For the second approach: using takeUntil
  private destroy$ = new Subject<void>();
  
  // For the third approach: using a timer
  private updateTimer: any;
  
  // For the fourth approach: event listeners
  private boundWindowResizeHandler: any;
  
  constructor(private router: Router) {}
  
  ngOnInit() {
    // Listen for router navigation events
    this.router.events
      .pipe(takeUntil(this.destroy$))
      .subscribe(event => {
        if (event instanceof NavigationStart) {
          console.log('Navigation started, stopping monitoring');
          this.stopMonitoring();
        }
      });
    
    // Add window resize event listener
    this.boundWindowResizeHandler = this.handleWindowResize.bind(this);
    window.addEventListener('resize', this.boundWindowResizeHandler);
  }
  
  startMonitoring() {
    this.isMonitoring = true;
    this.status = 'Monitoring';
    
    // Approach 1: Store subscription for later cleanup
    this.dataSubscription = interval(2000).subscribe(() => {
      this.updateCount++;
      this.lastUpdate = new Date();
      console.log('Data updated');
    });
    
    // Approach 2: Using takeUntil for automatic cleanup
    interval(5000)
      .pipe(takeUntil(this.destroy$))
      .subscribe(() => {
        this.status = 'Active - ' + new Date().toLocaleTimeString();
      });
    
    // Approach 3: Using a timer
    this.updateTimer = setInterval(() => {
      console.log('Checking for updates...');
    }, 10000);
  }
  
  stopMonitoring() {
    this.isMonitoring = false;
    this.status = 'Stopped';
    
    // Clean up approach 1
    if (this.dataSubscription) {
      this.dataSubscription.unsubscribe();
      this.dataSubscription = null;
    }
    
    // Clean up approach 3
    if (this.updateTimer) {
      clearInterval(this.updateTimer);
      this.updateTimer = null;
    }
  }
  
  handleWindowResize() {
    console.log('Window resized');
  }
  
  ngOnDestroy() {
    console.log('Component is being destroyed');
    
    // Clean up approach 1
    if (this.dataSubscription) {
      this.dataSubscription.unsubscribe();
    }
    
    // Clean up approach 2
    this.destroy$.next();
    this.destroy$.complete();
    
    // Clean up approach 3
    if (this.updateTimer) {
      clearInterval(this.updateTimer);
    }
    
    // Clean up approach 4
    window.removeEventListener('resize', this.boundWindowResizeHandler);
  }
}
```

### Best Practices for Lifecycle Hooks

1. **Keep hooks focused**: Each hook should have a specific purpose and responsibility.

2. **Avoid heavy operations in frequently called hooks**: Be careful with performance in `ngDoCheck`, `ngAfterContentChecked`, and `ngAfterViewChecked`.

3. **Initialize in ngOnInit, not in the constructor**: The constructor should only be used for dependency injection. All initialization logic should go in `ngOnInit`.

4. **Clean up in ngOnDestroy**: Always clean up resources in `ngOnDestroy` to prevent memory leaks.

5. **Use the appropriate hook for the task**:
   - For initialization: `ngOnInit`
   - For reacting to input changes: `ngOnChanges`
   - For DOM manipulation: `ngAfterViewInit`
   - For cleanup: `ngOnDestroy`

6. **Avoid changing the view in AfterViewInit and AfterViewChecked**: This can cause the "Expression has changed after it was checked" error.

### Lifecycle Hooks in Standalone Components

Lifecycle hooks work the same way in standalone components as they do in traditional components:

```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-standalone-example',
  standalone: true,
  imports: [CommonModule],
  template: `<div>Standalone Component</div>`
})
export class StandaloneExampleComponent implements OnInit, OnDestroy {
  ngOnInit() {
    console.log('Standalone component initialized');
  }
  
  ngOnDestroy() {
    console.log('Standalone component destroyed');
  }
}
```

### Conclusion

Angular's lifecycle hooks provide a powerful way to tap into different stages of a component's lifecycle. By understanding when each hook is called and what it's best used for, you can create more efficient and maintainable components.

Remember that not all components need to implement all hooks. Only implement the hooks that your component actually needs to perform its required functionality. This keeps your code cleaner and more focused.

The most commonly used hooks are `ngOnInit` for initialization, `ngOnChanges` for reacting to input changes, and `ngOnDestroy` for cleanup. The other hooks are more specialized and should be used only when needed for specific use cases. 