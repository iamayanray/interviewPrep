# Angular Pipes

## Question
What are Angular pipes? How do you use them, and how would you create a custom pipe?

## Answer

### What are Angular Pipes?

Angular pipes are a way to transform data in your templates. They take data as input and transform it into the desired output format. Pipes are useful for formatting dates, numbers, strings, currency values, and more without modifying the original data.

### Built-in Pipes

Angular provides several built-in pipes for common data transformations:

#### 1. DatePipe

Formats dates according to locale rules.

```html
<!-- Basic usage -->
<p>Today is {{ today | date }}</p>

<!-- With format -->
<p>Today is {{ today | date:'fullDate' }}</p>
<p>The time is {{ today | date:'shortTime' }}</p>

<!-- Custom format -->
<p>Custom: {{ today | date:'dd/MM/yyyy HH:mm:ss' }}</p>

<!-- With locale -->
<p>German date: {{ today | date:'fullDate':'':'de' }}</p>
```

#### 2. UpperCasePipe and LowerCasePipe

Transform text to uppercase or lowercase.

```html
<p>{{ 'Hello World' | uppercase }}</p> <!-- Output: HELLO WORLD -->
<p>{{ 'Hello World' | lowercase }}</p> <!-- Output: hello world -->
```

#### 3. TitleCasePipe

Transforms text to title case.

```html
<p>{{ 'hello world' | titlecase }}</p> <!-- Output: Hello World -->
```

#### 4. CurrencyPipe

Formats a number as currency.

```html
<p>{{ 1234.56 | currency }}</p> <!-- Output: $1,234.56 -->
<p>{{ 1234.56 | currency:'EUR' }}</p> <!-- Output: €1,234.56 -->
<p>{{ 1234.56 | currency:'INR':'code' }}</p> <!-- Output: INR 1,234.56 -->
```

#### 5. DecimalPipe

Formats a number as decimal.

```html
<p>{{ 3.14159265359 | number }}</p> <!-- Output: 3.142 -->
<p>{{ 3.14159265359 | number:'1.2-5' }}</p> <!-- Output: 3.14159 -->
<p>{{ 1000000 | number }}</p> <!-- Output: 1,000,000 -->
```

#### 6. PercentPipe

Formats a number as percentage.

```html
<p>{{ 0.259 | percent }}</p> <!-- Output: 25.9% -->
<p>{{ 0.259 | percent:'2.2-4' }}</p> <!-- Output: 25.90% -->
```

#### 7. SlicePipe

Creates a new array or string containing a subset of the elements.

```html
<p>{{ [1, 2, 3, 4, 5] | slice:1:4 }}</p> <!-- Output: [2, 3, 4] -->
<p>{{ 'Hello World' | slice:0:5 }}</p> <!-- Output: Hello -->
```

#### 8. JsonPipe

Converts a value into a JSON string.

```html
<pre>{{ user | json }}</pre>
<!-- Output:
{
  "name": "John",
  "age": 30,
  "address": {
    "city": "New York",
    "country": "USA"
  }
}
-->
```

#### 9. AsyncPipe

Unwraps a value from an asynchronous primitive (Promise or Observable).

```html
<!-- With Observable -->
<div *ngIf="data$ | async as data">
  {{ data.name }}
</div>

<!-- With Promise -->
<p>{{ greeting | async }}</p>
```

#### 10. KeyValuePipe

Transforms an Object or Map into an array of key-value pairs.

```html
<div *ngFor="let item of object | keyvalue">
  {{item.key}}: {{item.value}}
</div>
```

### Chaining Pipes

Pipes can be chained to apply multiple transformations.

```html
<p>{{ today | date:'fullDate' | uppercase }}</p>
```

### Pipe Parameters

Many pipes accept parameters to customize their behavior.

```html
<!-- date pipe with format parameter -->
{{ today | date:'dd/MM/yyyy' }}

<!-- slice pipe with start and end parameters -->
{{ [1, 2, 3, 4, 5] | slice:1:3 }}
```

### Creating Custom Pipes

You can create custom pipes for specific transformation needs.

#### Basic Custom Pipe

```typescript
// truncate.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate'
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit: number = 25, completeWords: boolean = false, ellipsis: string = '...'): string {
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

#### Using the Custom Pipe

```html
<!-- Basic usage -->
<p>{{ longText | truncate }}</p>

<!-- With parameters -->
<p>{{ longText | truncate:10:true:'...' }}</p>
```

#### Registering the Custom Pipe

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppComponent } from './app.component';
import { TruncatePipe } from './truncate.pipe';

@NgModule({
  declarations: [
    AppComponent,
    TruncatePipe // Register the pipe
  ],
  imports: [BrowserModule],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

#### Standalone Custom Pipe (Angular 14+)

```typescript
// truncate.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate',
  standalone: true
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit: number = 25, completeWords: boolean = false, ellipsis: string = '...'): string {
    // Implementation as above
  }
}
```

### Pure vs Impure Pipes

By default, pipes are pure, meaning they only execute when their input values change. Impure pipes execute on every change detection cycle.

#### Pure Pipe (Default)

```typescript
@Pipe({
  name: 'myPurePipe'
})
export class MyPurePipe implements PipeTransform {
  transform(value: any): any {
    // This will only execute when the input value changes
    return transformedValue;
  }
}
```

#### Impure Pipe

```typescript
@Pipe({
  name: 'myImpurePipe',
  pure: false // Makes the pipe impure
})
export class MyImpurePipe implements PipeTransform {
  transform(value: any): any {
    // This will execute on every change detection cycle
    return transformedValue;
  }
}
```

### More Complex Custom Pipes

#### 1. Filter Pipe

```typescript
// filter.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'filter',
  pure: false // Impure pipe as it needs to update when array changes
})
export class FilterPipe implements PipeTransform {
  transform(items: any[], searchText: string, property?: string): any[] {
    if (!items) {
      return [];
    }
    if (!searchText) {
      return items;
    }
    
    searchText = searchText.toLowerCase();
    
    return items.filter(item => {
      if (property) {
        return item[property].toLowerCase().includes(searchText);
      }
      return JSON.stringify(item).toLowerCase().includes(searchText);
    });
  }
}
```

Usage:

```html
<input [(ngModel)]="searchText" placeholder="Search...">

<div *ngFor="let item of items | filter:searchText:'name'">
  {{ item.name }}
</div>
```

#### 2. Sort Pipe

```typescript
// sort.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'sort'
})
export class SortPipe implements PipeTransform {
  transform(array: any[], property: string, direction: 'asc' | 'desc' = 'asc'): any[] {
    if (!array || !property) {
      return array;
    }

    const sortedArray = [...array].sort((a, b) => {
      if (a[property] < b[property]) {
        return direction === 'asc' ? -1 : 1;
      }
      if (a[property] > b[property]) {
        return direction === 'asc' ? 1 : -1;
      }
      return 0;
    });

    return sortedArray;
  }
}
```

Usage:

```html
<div *ngFor="let user of users | sort:'name':'asc'">
  {{ user.name }} - {{ user.age }}
</div>
```

#### 3. Time Ago Pipe

```typescript
// time-ago.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'timeAgo'
})
export class TimeAgoPipe implements PipeTransform {
  transform(value: Date | string): string {
    if (!value) {
      return '';
    }

    const date = typeof value === 'string' ? new Date(value) : value;
    const now = new Date();
    const seconds = Math.floor((now.getTime() - date.getTime()) / 1000);

    if (seconds < 60) {
      return 'just now';
    }

    const intervals = {
      year: 31536000,
      month: 2592000,
      week: 604800,
      day: 86400,
      hour: 3600,
      minute: 60
    };

    let counter;
    for (const [key, value] of Object.entries(intervals)) {
      counter = Math.floor(seconds / value);
      if (counter > 0) {
        return `${counter} ${key}${counter === 1 ? '' : 's'} ago`;
      }
    }

    return 'just now';
  }
}
```

Usage:

```html
<p>Posted: {{ post.createdAt | timeAgo }}</p>
```

#### 4. Highlight Search Pipe

```typescript
// highlight.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Pipe({
  name: 'highlight'
})
export class HighlightPipe implements PipeTransform {
  constructor(private sanitizer: DomSanitizer) {}

  transform(text: string, search: string): SafeHtml {
    if (!search || !text) {
      return text;
    }

    const pattern = search.replace(/[\-\[\]\/\{\}\(\)\*\+\?\.\\\^\$\|]/g, '\\$&');
    const regex = new RegExp(pattern, 'gi');

    const result = text.replace(regex, match => `<span class="highlight">${match}</span>`);
    
    return this.sanitizer.bypassSecurityTrustHtml(result);
  }
}
```

Usage:

```html
<p [innerHTML]="content | highlight:searchTerm"></p>
```

With CSS:

```css
.highlight {
  background-color: yellow;
  font-weight: bold;
}
```

### Best Practices for Pipes

1. **Use Pure Pipes When Possible**:
   Pure pipes are more performant as they only execute when input values change.

2. **Avoid Complex Logic in Templates**:
   Use pipes to keep templates clean and move complex logic to the pipe implementation.

3. **Be Careful with Impure Pipes**:
   Impure pipes run on every change detection cycle, which can impact performance.

4. **Consider Memoization for Expensive Operations**:
   Cache results for expensive transformations to improve performance.

   ```typescript
   @Pipe({ name: 'expensiveTransform' })
   export class ExpensiveTransformPipe implements PipeTransform {
     private cachedValue: any = null;
     private cachedInput: any = null;

     transform(value: any): any {
       if (this.cachedInput === value) {
         return this.cachedValue;
       }

       this.cachedInput = value;
       this.cachedValue = this.performExpensiveOperation(value);
       return this.cachedValue;
     }

     private performExpensiveOperation(value: any): any {
       // Expensive operation here
       return transformedValue;
     }
   }
   ```

5. **Use AsyncPipe for Observables and Promises**:
   AsyncPipe automatically subscribes and unsubscribes to Observables, preventing memory leaks.

6. **Combine Pipes with Other Angular Features**:
   Pipes work well with directives, components, and services.

7. **Test Your Pipes**:
   Write unit tests for custom pipes to ensure they work correctly.

   ```typescript
   // truncate.pipe.spec.ts
   import { TruncatePipe } from './truncate.pipe';

   describe('TruncatePipe', () => {
     let pipe: TruncatePipe;

     beforeEach(() => {
       pipe = new TruncatePipe();
     });

     it('should create an instance', () => {
       expect(pipe).toBeTruthy();
     });

     it('should truncate text if it exceeds the limit', () => {
       expect(pipe.transform('This is a long text', 10)).toBe('This is a...');
     });

     it('should not truncate text if it does not exceed the limit', () => {
       expect(pipe.transform('Short', 10)).toBe('Short');
     });

     it('should respect complete words option', () => {
       expect(pipe.transform('This is a long text', 10, true)).toBe('This is...');
     });

     it('should use custom ellipsis', () => {
       expect(pipe.transform('This is a long text', 10, false, '***')).toBe('This is a***');
     });
   });
   ```

### Common Pitfalls

1. **Using Impure Pipes for Filtering Arrays**:
   While convenient, impure pipes for filtering large arrays can cause performance issues. Consider using component methods or RxJS operators instead.

2. **Not Handling Null or Undefined Values**:
   Always check for null or undefined values in your custom pipes.

3. **Forgetting to Register Pipes**:
   Pipes must be declared in a module or marked as standalone to be used in templates.

4. **Complex Transformations in Templates**:
   Avoid chaining too many pipes in templates, as it can make the code hard to read and maintain.

### Conclusion

Angular pipes are a powerful feature for transforming data in templates. They help keep your components clean by moving data transformation logic out of the component class. By understanding built-in pipes and knowing how to create custom pipes, you can efficiently format and transform data in your Angular applications.

Whether you're formatting dates, filtering arrays, or creating complex data transformations, pipes provide a clean, reusable way to handle these operations in your templates. 