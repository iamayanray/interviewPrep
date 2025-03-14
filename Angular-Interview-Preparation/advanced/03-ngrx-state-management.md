# Advanced NgRx State Management

## Question
What are the advanced concepts in NgRx state management? Explain effects, selectors, and entity adapters with examples.

## Answer

### Introduction to Advanced NgRx

NgRx provides powerful state management capabilities for complex Angular applications. Here's a comprehensive guide to advanced NgRx concepts.

### 1. Advanced Effects

#### Effect with Error Handling

```typescript
// user.effects.ts
@Injectable()
export class UserEffects {
  loadUsers$ = createEffect(() => this.actions$.pipe(
    ofType(UserActions.loadUsers),
    mergeMap(() => this.userService.getUsers().pipe(
      map(users => UserActions.loadUsersSuccess({ users })),
      catchError(error => of(UserActions.loadUsersFailure({ error })))
    ))
  ));

  constructor(
    private actions$: Actions,
    private userService: UserService
  ) {}
}
```

#### Effect with Retry Logic

```typescript
// api.effects.ts
@Injectable()
export class ApiEffects {
  fetchData$ = createEffect(() => this.actions$.pipe(
    ofType(ApiActions.fetchData),
    mergeMap(() => this.apiService.getData().pipe(
      retry(3),
      map(data => ApiActions.fetchDataSuccess({ data })),
      catchError(error => of(ApiActions.fetchDataFailure({ error })))
    ))
  ));

  constructor(
    private actions$: Actions,
    private apiService: ApiService
  ) {}
}
```

#### Effect with Debounce

```typescript
// search.effects.ts
@Injectable()
export class SearchEffects {
  search$ = createEffect(() => this.actions$.pipe(
    ofType(SearchActions.search),
    debounceTime(300),
    distinctUntilChanged(),
    mergeMap(({ query }) => this.searchService.search(query).pipe(
      map(results => SearchActions.searchSuccess({ results })),
      catchError(error => of(SearchActions.searchFailure({ error })))
    ))
  ));

  constructor(
    private actions$: Actions,
    private searchService: SearchService
  ) {}
}
```

### 2. Advanced Selectors

#### Memoized Selectors

```typescript
// user.selectors.ts
export const selectUserState = createFeatureSelector<UserState>('user');

export const selectAllUsers = createSelector(
  selectUserState,
  (state: UserState) => state.users
);

export const selectUserById = (userId: number) => createSelector(
  selectAllUsers,
  (users) => users.find(user => user.id === userId)
);

export const selectActiveUsers = createSelector(
  selectAllUsers,
  (users) => users.filter(user => user.isActive)
);

export const selectUserStats = createSelector(
  selectAllUsers,
  (users) => ({
    total: users.length,
    active: users.filter(user => user.isActive).length,
    inactive: users.filter(user => !user.isActive).length
  })
);
```

#### Selector with Props

```typescript
// product.selectors.ts
export const selectProductById = createSelector(
  selectProductState,
  (state: ProductState, props: { id: number }) =>
    state.products.find(product => product.id === props.id)
);

// Using in component
@Component({
  selector: 'app-product',
  template: `
    <div *ngIf="product$ | async as product">
      <h2>{{ product.name }}</h2>
      <p>{{ product.price }}</p>
    </div>
  `
})
export class ProductComponent {
  product$ = this.store.select(selectProductById, { id: 1 });
}
```

### 3. Entity Adapter

```typescript
// user.state.ts
export interface User {
  id: number;
  name: string;
  email: string;
}

export interface UserState extends EntityState<User> {
  selectedUserId: number | null;
  loading: boolean;
  error: string | null;
}

export const adapter: EntityAdapter<User> = createEntityAdapter<User>({
  selectId: (user: User) => user.id,
  sortComparer: (a, b) => a.name.localeCompare(b.name)
});

export const initialState: UserState = adapter.getInitialState({
  selectedUserId: null,
  loading: false,
  error: null
});

// user.reducer.ts
export const userReducer = createReducer(
  initialState,
  on(UserActions.loadUsers, (state) => ({
    ...state,
    loading: true,
    error: null
  })),
  on(UserActions.loadUsersSuccess, (state, { users }) =>
    adapter.setAll(users, { ...state, loading: false })
  ),
  on(UserActions.loadUsersFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error
  }))
);

// Using in component
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngIf="users$ | async as users">
      <div *ngFor="let user of users">
        {{ user.name }} - {{ user.email }}
      </div>
    </div>
  `
})
export class UserListComponent {
  users$ = this.store.select(selectAllUsers);

  constructor(private store: Store) {
    this.store.dispatch(UserActions.loadUsers());
  }
}
```

### 4. Meta-Reducers

```typescript
// logger.meta-reducer.ts
export function logger(reducer: ActionReducer<any>): ActionReducer<any> {
  return function(state, action) {
    console.log('state', state);
    console.log('action', action);
    const nextState = reducer(state, action);
    console.log('next state', nextState);
    return nextState;
  };
}

// storage.meta-reducer.ts
export function storage(reducer: ActionReducer<any>): ActionReducer<any> {
  return function(state, action) {
    const nextState = reducer(state, action);
    localStorage.setItem('state', JSON.stringify(nextState));
    return nextState;
  };
}

// Using in app.module.ts
@NgModule({
  imports: [
    StoreModule.forRoot(reducers, {
      metaReducers: [logger, storage]
    })
  ]
})
export class AppModule { }
```

### 5. Action Creators with Payload

```typescript
// user.actions.ts
export const createUser = createAction(
  '[User] Create User',
  props<{ user: User }>()
);

export const createUserSuccess = createAction(
  '[User] Create User Success',
  props<{ user: User }>()
);

export const createUserFailure = createAction(
  '[User] Create User Failure',
  props<{ error: any }>()
);

// Using in component
@Component({
  selector: 'app-user-form',
  template: `
    <form [formGroup]="userForm" (ngSubmit)="onSubmit()">
      <input formControlName="name">
      <input formControlName="email">
      <button type="submit">Create User</button>
    </form>
  `
})
export class UserFormComponent {
  userForm = this.fb.group({
    name: ['', Validators.required],
    email: ['', [Validators.required, Validators.email]]
  });

  constructor(
    private fb: FormBuilder,
    private store: Store
  ) {}

  onSubmit() {
    if (this.userForm.valid) {
      this.store.dispatch(createUser({ user: this.userForm.value }));
    }
  }
}
```

### 6. Store DevTools

```typescript
// app.module.ts
@NgModule({
  imports: [
    StoreModule.forRoot(reducers),
    StoreDevtoolsModule.instrument({
      maxAge: 25,
      logOnly: environment.production,
      autoPause: true,
      trace: false,
      traceLimit: 75
    })
  ]
})
export class AppModule { }
```

### Conclusion

Key points to remember:

1. **Effects**:
   - Handle side effects
   - Implement error handling
   - Use retry logic
   - Apply debounce for search

2. **Selectors**:
   - Create memoized selectors
   - Use selector props
   - Combine multiple selectors
   - Optimize performance

3. **Entity Adapter**:
   - Manage collections
   - Handle CRUD operations
   - Optimize updates
   - Maintain order

4. **Meta-Reducers**:
   - Add logging
   - Implement persistence
   - Track state changes
   - Debug application

5. **Action Creators**:
   - Type-safe actions
   - Handle payloads
   - Create success/failure actions
   - Maintain consistency

Remember to:
- Use appropriate effects
- Implement proper error handling
- Optimize selectors
- Use entity adapters for collections
- Add meta-reducers when needed
- Follow NgRx best practices
- Test state changes
- Monitor performance 