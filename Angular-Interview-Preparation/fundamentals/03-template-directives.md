# Template Directives: ng-template, ng-container, and ng-content

## Question
What is the difference between ng-template, ng-container, and ng-content? When would you use each?

## Answer

Angular provides several special elements for template manipulation and content projection. Understanding the differences between `ng-template`, `ng-container`, and `ng-content` is crucial for building flexible and reusable components.

### 1. ng-template

`<ng-template>` is a template element that Angular uses for defining template fragments. It's not rendered directly in the DOM but serves as a blueprint for rendering content.

#### Key Characteristics:

- **Not Rendered Directly**: Content inside `<ng-template>` is not displayed by default.
- **Template Reference**: Can be referenced using a template reference variable or `TemplateRef` in the component class.
- **Conditional Rendering**: Often used with structural directives like `*ngIf`, `*ngFor`, etc.
- **Programmatic Rendering**: Can be rendered programmatically using `ViewContainerRef`.

#### Example 1: Basic Usage with Template Reference

```typescript
// template-example.component.ts
import { Component, TemplateRef, ViewChild, ViewContainerRef } from '@angular/core';

@Component({
  selector: 'app-template-example',
  template: `
    <!-- Define a template with a reference variable -->
    <ng-template #myTemplate>
      <div class="alert alert-info">
        This content is from the template!
      </div>
    </ng-template>
    
    <!-- Button to render the template -->
    <button (click)="showTemplate()">Show Template</button>
    
    <!-- Container where the template will be rendered -->
    <div #container></div>
  `
})
export class TemplateExampleComponent {
  // Get references to the template and container
  @ViewChild('myTemplate') myTemplate: TemplateRef<any>;
  @ViewChild('container', { read: ViewContainerRef }) container: ViewContainerRef;
  
  showTemplate() {
    // Clear the container first
    this.container.clear();
    
    // Create an embedded view from the template and insert it
    this.container.createEmbeddedView(this.myTemplate);
  }
}
```

#### Example 2: Usage with Structural Directives

```html
<!-- Behind the scenes, Angular transforms structural directives to use ng-template -->

<!-- This: -->
<div *ngIf="isVisible">Content to show conditionally</div>

<!-- Is transformed by Angular to: -->
<ng-template [ngIf]="isVisible">
  <div>Content to show conditionally</div>
</ng-template>

<!-- Similarly, ngFor: -->
<div *ngFor="let item of items">{{ item.name }}</div>

<!-- Is transformed to: -->
<ng-template ngFor let-item [ngForOf]="items">
  <div>{{ item.name }}</div>
</ng-template>
```

#### Example 3: Custom Structural Directive

```typescript
// unless.directive.ts
import { Directive, Input, TemplateRef, ViewContainerRef } from '@angular/core';

@Directive({
  selector: '[appUnless]'
})
export class UnlessDirective {
  private hasView = false;

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}

  @Input() set appUnless(condition: boolean) {
    if (!condition && !this.hasView) {
      // If condition is false and view hasn't been created yet,
      // create the view from the template
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (condition && this.hasView) {
      // If condition is true and view exists, remove the view
      this.viewContainer.clear();
      this.hasView = false;
    }
  }
}

// Usage in template:
// <div *appUnless="condition">This content shows when condition is false</div>
```

#### When to Use ng-template:

1. When you need to define a template that will be rendered conditionally or multiple times.
2. When implementing custom structural directives.
3. When you need to pass template fragments to child components.
4. When you need to render content programmatically.

### 2. ng-container

`<ng-container>` is a grouping element that doesn't render any HTML element in the DOM. It's useful for applying structural directives without adding extra elements to the DOM.

#### Key Characteristics:

- **Invisible Wrapper**: Doesn't add any extra elements to the DOM.
- **Multiple Directives**: Allows applying multiple structural directives to a group of elements.
- **Cleaner DOM**: Helps keep the DOM clean by avoiding unnecessary wrapper elements.

#### Example 1: Basic Usage

```html
<!-- Without ng-container, we'd need an extra div -->
<div *ngIf="isVisible">
  <p>First paragraph</p>
  <p>Second paragraph</p>
</div>

<!-- With ng-container, no extra DOM element is created -->
<ng-container *ngIf="isVisible">
  <p>First paragraph</p>
  <p>Second paragraph</p>
</ng-container>
```

#### Example 2: Multiple Structural Directives

```html
<!-- This won't work because you can't have multiple structural directives on one element -->
<div *ngIf="isVisible" *ngFor="let item of items">{{ item.name }}</div>

<!-- Using ng-container to apply multiple structural directives -->
<ng-container *ngIf="isVisible">
  <div *ngFor="let item of items">{{ item.name }}</div>
</ng-container>

<!-- Or nesting ng-container elements -->
<ng-container *ngIf="isVisible">
  <ng-container *ngFor="let item of items">
    <div>{{ item.name }}</div>
  </ng-container>
</ng-container>
```

#### Example 3: Conditional Content in Lists

```html
<ul>
  <ng-container *ngFor="let item of items">
    <!-- First item gets special treatment -->
    <li *ngIf="item.isFirst" class="first-item">{{ item.name }} (First Item)</li>
    
    <!-- Regular items -->
    <li *ngIf="!item.isFirst">{{ item.name }}</li>
    
    <!-- Conditional separator -->
    <ng-container *ngIf="!item.isLast">
      <li class="separator">---</li>
    </ng-container>
  </ng-container>
</ul>
```

#### When to Use ng-container:

1. When you need to apply structural directives without adding extra elements to the DOM.
2. When you need to apply multiple structural directives to a group of elements.
3. When you want to conditionally include or exclude multiple elements as a group.
4. When you need to keep the DOM structure clean and semantic.

### 3. ng-content

`<ng-content>` is used for content projection, allowing components to receive and display content from their parent components. It's a powerful feature for creating reusable and flexible components.

#### Key Characteristics:

- **Content Projection**: Projects content from the parent component into the child component.
- **Transclusion**: Similar to transclusion in other frameworks.
- **Multiple Slots**: Can have multiple slots using the `select` attribute.
- **Default Slot**: Content without a selector goes into the default slot.

#### Example 1: Basic Content Projection

```typescript
// card.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <div class="card-header">
        <ng-content select="[card-header]"></ng-content>
      </div>
      <div class="card-body">
        <ng-content select="[card-body]"></ng-content>
      </div>
      <div class="card-footer">
        <ng-content select="[card-footer]"></ng-content>
      </div>
    </div>
  `,
  styles: [`
    .card {
      border: 1px solid #ccc;
      border-radius: 4px;
      margin-bottom: 20px;
    }
    .card-header {
      background-color: #f5f5f5;
      padding: 10px;
      border-bottom: 1px solid #ccc;
    }
    .card-body {
      padding: 15px;
    }
    .card-footer {
      background-color: #f5f5f5;
      padding: 10px;
      border-top: 1px solid #ccc;
    }
  `]
})
export class CardComponent {}
```

**Usage in parent component:**

```html
<app-card>
  <div card-header>
    <h3>Card Title</h3>
  </div>
  
  <div card-body>
    <p>This is the main content of the card.</p>
    <p>You can put any HTML here.</p>
  </div>
  
  <div card-footer>
    <button class="btn btn-primary">Action</button>
  </div>
</app-card>
```

#### Example 2: Default Content Projection

```typescript
// simple-panel.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-simple-panel',
  template: `
    <div class="panel">
      <ng-content></ng-content>
    </div>
  `,
  styles: [`
    .panel {
      border: 1px solid #ddd;
      padding: 15px;
      border-radius: 4px;
    }
  `]
})
export class SimplePanelComponent {}
```

**Usage:**

```html
<app-simple-panel>
  <h2>Panel Title</h2>
  <p>This content will be projected into the panel.</p>
</app-simple-panel>
```

#### Example 3: Multiple Content Projection with Different Selectors

```typescript
// tab.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-tab',
  template: `
    <div class="tab-container">
      <div class="tab-header">
        <!-- Project elements with the tab-title attribute -->
        <ng-content select="[tab-title]"></ng-content>
      </div>
      
      <div class="tab-content">
        <!-- Project elements with the tab-content attribute -->
        <ng-content select="[tab-content]"></ng-content>
      </div>
      
      <div class="tab-footer">
        <!-- Project elements with the tab-footer attribute -->
        <ng-content select="[tab-footer]"></ng-content>
      </div>
    </div>
  `
})
export class TabComponent {}
```

**Usage:**

```html
<app-tab>
  <div tab-title>
    <h3>Important Information</h3>
  </div>
  
  <div tab-content>
    <p>This is the main content of the tab.</p>
    <ul>
      <li>Item 1</li>
      <li>Item 2</li>
    </ul>
  </div>
  
  <div tab-footer>
    <button>Close</button>
  </div>
</app-tab>
```

#### Example 4: Conditional Content Projection

```typescript
// conditional-content.component.ts
import { Component, ContentChild, TemplateRef } from '@angular/core';

@Component({
  selector: 'app-conditional-content',
  template: `
    <div class="container">
      <div *ngIf="hasHeader">
        <ng-content select="[header]"></ng-content>
      </div>
      
      <div class="body">
        <ng-content></ng-content>
      </div>
      
      <div *ngIf="hasFooter">
        <ng-content select="[footer]"></ng-content>
      </div>
    </div>
  `
})
export class ConditionalContentComponent {
  @ContentChild('header') headerContent: TemplateRef<any>;
  @ContentChild('footer') footerContent: TemplateRef<any>;
  
  get hasHeader(): boolean {
    return !!this.headerContent;
  }
  
  get hasFooter(): boolean {
    return !!this.footerContent;
  }
}
```

#### When to Use ng-content:

1. When creating reusable components that need to be customized with different content.
2. When building component libraries or UI frameworks.
3. When implementing layout components like cards, panels, modals, etc.
4. When you want to allow parent components to inject content into specific areas of a child component.

### Comparison and Use Cases

| Feature | ng-template | ng-container | ng-content |
|---------|------------|--------------|------------|
| **Renders in DOM** | No | No | Yes (projects content) |
| **Primary Use** | Template definition | Grouping without extra DOM | Content projection |
| **Works with** | Structural directives, programmatic rendering | Structural directives | Parent-to-child content insertion |
| **Can be referenced** | Yes (TemplateRef) | No | Yes (ContentChild) |
| **Can have multiple instances** | Yes (with ngTemplateOutlet) | N/A | Yes (with select) |

### Real-World Example: Combining All Three

Here's an example that combines `ng-template`, `ng-container`, and `ng-content` to create a flexible data table component:

```typescript
// data-table.component.ts
import { Component, ContentChild, Input, TemplateRef } from '@angular/core';

@Component({
  selector: 'app-data-table',
  template: `
    <div class="data-table">
      <!-- Header projection -->
      <div class="table-header">
        <ng-content select="[table-header]"></ng-content>
      </div>
      
      <!-- Table content -->
      <div class="table-body">
        <ng-container *ngIf="items && items.length > 0; else noData">
          <ng-container *ngFor="let item of items">
            <!-- Use custom row template if provided, otherwise use default -->
            <ng-container *ngTemplateOutlet="rowTemplate || defaultRowTemplate; context: { $implicit: item }"></ng-container>
          </ng-container>
        </ng-container>
        
        <!-- No data template -->
        <ng-template #noData>
          <div class="no-data">
            <ng-content select="[no-data]"></ng-content>
            <p *ngIf="!hasNoDataContent">No data available</p>
          </div>
        </ng-template>
        
        <!-- Default row template -->
        <ng-template #defaultRowTemplate let-item>
          <div class="table-row">
            <div *ngFor="let key of objectKeys(item)" class="table-cell">
              {{ item[key] }}
            </div>
          </div>
        </ng-template>
      </div>
      
      <!-- Footer projection -->
      <div class="table-footer" *ngIf="hasFooterContent">
        <ng-content select="[table-footer]"></ng-content>
      </div>
    </div>
  `,
  styles: [`
    .data-table {
      width: 100%;
      border: 1px solid #ddd;
    }
    .table-header {
      background-color: #f5f5f5;
      font-weight: bold;
      padding: 10px;
      border-bottom: 2px solid #ddd;
    }
    .table-body {
      padding: 0;
    }
    .table-row {
      display: flex;
      border-bottom: 1px solid #eee;
    }
    .table-cell {
      flex: 1;
      padding: 10px;
    }
    .no-data {
      padding: 20px;
      text-align: center;
      color: #999;
    }
    .table-footer {
      background-color: #f5f5f5;
      padding: 10px;
      border-top: 1px solid #ddd;
    }
  `]
})
export class DataTableComponent {
  @Input() items: any[] = [];
  
  @ContentChild('rowTemplate') rowTemplate: TemplateRef<any>;
  @ContentChild('[no-data]') noDataContent: any;
  @ContentChild('[table-footer]') footerContent: any;
  
  get hasNoDataContent(): boolean {
    return !!this.noDataContent;
  }
  
  get hasFooterContent(): boolean {
    return !!this.footerContent;
  }
  
  objectKeys(obj: any): string[] {
    return Object.keys(obj);
  }
}
```

**Usage:**

```html
<app-data-table [items]="users">
  <!-- Custom header -->
  <div table-header>
    <div class="header-row">
      <div class="header-cell">ID</div>
      <div class="header-cell">Name</div>
      <div class="header-cell">Email</div>
      <div class="header-cell">Actions</div>
    </div>
  </div>
  
  <!-- Custom row template -->
  <ng-template #rowTemplate let-user>
    <div class="custom-row">
      <div class="cell">{{ user.id }}</div>
      <div class="cell">{{ user.name }}</div>
      <div class="cell">{{ user.email }}</div>
      <div class="cell">
        <button (click)="editUser(user)">Edit</button>
        <button (click)="deleteUser(user)">Delete</button>
      </div>
    </div>
  </ng-template>
  
  <!-- Custom no-data content -->
  <div no-data>
    <img src="assets/empty-state.svg" alt="No users found">
    <p>No users found. Click the button below to add a new user.</p>
    <button (click)="addUser()">Add User</button>
  </div>
  
  <!-- Custom footer -->
  <div table-footer>
    <div class="pagination">
      <button [disabled]="currentPage === 1" (click)="prevPage()">Previous</button>
      <span>Page {{ currentPage }} of {{ totalPages }}</span>
      <button [disabled]="currentPage === totalPages" (click)="nextPage()">Next</button>
    </div>
  </div>
</app-data-table>
```

### Conclusion

Understanding the differences and use cases for `ng-template`, `ng-container`, and `ng-content` is essential for building flexible and reusable Angular components:

- **ng-template**: Use for defining template fragments that can be rendered conditionally or multiple times.
- **ng-container**: Use as an invisible wrapper to apply structural directives without adding extra elements to the DOM.
- **ng-content**: Use for content projection to allow parent components to inject content into child components.

By combining these powerful template features, you can create highly reusable and customizable components that maintain clean DOM structure and follow best practices for component design. 