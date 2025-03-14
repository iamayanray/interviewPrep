# Angular State Management

## Question
How do you manage state in Angular applications? Explain different state management solutions and when to use them.

## Answer

### Introduction to State Management

State management is crucial for handling complex application data in Angular. Different solutions exist, each with its own strengths and use cases.

### 1. NgRx (Redux Pattern)

NgRx is the most popular state management solution for Angular, following the Redux pattern.

#### Basic Setup

```typescript
// app.module.ts
import { StoreModule } from '@ngrx/store';
import { EffectsModule } from '@ngrx/effects';
import { StoreDevtoolsModule } from '@ngrx/store-devtools';

@NgModule({
  imports: [
    StoreModule.forRoot({}),
    EffectsModule.forRoot([]),
    StoreDevtoolsModule.instrument({
      maxAge: 25,
      logOnly: environment.production
    })
  ]
})
export class AppModule { }

// app.state.ts
export interface AppState {
  auth: AuthState;
  users: UserState;
}

// auth.state.ts
export interface AuthState {
  user: User | null;
  loading: boolean;
  error: string | null;
}

// auth.actions.ts
import { createAction, props } from '@ngrx/store';

export const login = createAction(
  '[Auth] Login',
  props<{ username: string; password: string }>()
);

export const loginSuccess = createAction(
  '[Auth] Login Success',
  props<{ user: User }>()
);

export const loginFailure = createAction(
  '[Auth] Login Failure',
  props<{ error: string }>()
);

// auth.reducer.ts
import { createReducer, on } from '@ngrx/store';
import * as AuthActions from './auth.actions';

export const initialState: AuthState = {
  user: null,
  loading: false,
  error: null
};

export const authReducer = createReducer(
  initialState,
  on(AuthActions.login, state => ({
    ...state,
    loading: true,
    error: null
  })),
  on(AuthActions.loginSuccess, (state, { user }) => ({
    ...state,
    user,
    loading: false
  })),
  on(AuthActions.loginFailure, (state, { error }) => ({
    ...state,
    error,
    loading: false
  }))
);

// auth.effects.ts
import { Injectable } from '@angular/core';
import { Actions, createEffect, ofType } from '@ngrx/effects';
import { of } from 'rxjs';
import { map, mergeMap, catchError } from 'rxjs/operators';
import { AuthService } from './auth.service';
import * as AuthActions from './auth.actions';

@Injectable()
export class AuthEffects {
  login$ = createEffect(() =>
    this.actions$.pipe(
      ofType(AuthActions.login),
      mergeMap(({ username, password }) =>
        this.authService.login(username, password).pipe(
          map(user => AuthActions.loginSuccess({ user })),
          catchError(error => of(AuthActions.loginFailure({ error: error.message })))
        )
      )
    )
  );

  constructor(
    private actions$: Actions,
    private authService: AuthService
  ) {}
}

// auth.selectors.ts
import { createSelector } from '@ngrx/store';
import { AppState } from './app.state';
import { AuthState } from './auth.state';

export const selectAuthState = (state: AppState) => state.auth;

export const selectUser = createSelector(
  selectAuthState,
  (state: AuthState) => state.user
);

export const selectAuthLoading = createSelector(
  selectAuthState,
  (state: AuthState) => state.loading
);

export const selectAuthError = createSelector(
  selectAuthState,
  (state: AuthState) => state.error
);

// Using in Component
@Component({
  selector: 'app-login',
  template: `
    <form (ngSubmit)="onSubmit()">
      <input [(ngModel)]="username" name="username">
      <input [(ngModel)]="password" name="password" type="password">
      <button type="submit">Login</button>
    </form>
    <div *ngIf="loading$ | async">Loading...</div>
    <div *ngIf="error$ | async as error">{{ error }}</div>
  `
})
export class LoginComponent {
  username = '';
  password = '';
  loading$ = this.store.select(selectAuthLoading);
  error$ = this.store.select(selectAuthError);

  constructor(private store: Store) {}

  onSubmit() {
    this.store.dispatch(AuthActions.login({
      username: this.username,
      password: this.password
    }));
  }
}
```

### 2. NGXS (Simpler Alternative)

NGXS provides a simpler state management solution with less boilerplate.

```typescript
// auth.state.ts
import { Injectable } from '@angular/core';
import { State, Action, StateContext, Selector } from '@ngxs/store';
import { AuthService } from './auth.service';
import { patch, append, removeItem, insertItem, updateItem } from '@ngxs/store/operators';

export interface AuthStateModel {
  user: User | null;
  loading: boolean;
  error: string | null;
}

@State<AuthStateModel>({
  name: 'auth',
  defaults: {
    user: null,
    loading: false,
    error: null
  }
})
@Injectable()
export class AuthState {
  @Selector()
  static user(state: AuthStateModel) {
    return state.user;
  }

  @Selector()
  static loading(state: AuthStateModel) {
    return state.loading;
  }

  @Selector()
  static error(state: AuthStateModel) {
    return state.error;
  }

  @Action(Login)
  login(ctx: StateContext<AuthStateModel>, action: Login) {
    ctx.patchState({ loading: true, error: null });
    
    return this.authService.login(action.username, action.password).pipe(
      tap(user => ctx.patchState({ user, loading: false })),
      catchError(error => {
        ctx.patchState({ error: error.message, loading: false });
        return throwError(() => error);
      })
    );
  }

  constructor(private authService: AuthService) {}
}

// auth.actions.ts
export class Login {
  static readonly type = '[Auth] Login';
  constructor(public username: string, public password: string) {}
}

// Using in Component
@Component({
  selector: 'app-login',
  template: `
    <form (ngSubmit)="onSubmit()">
      <input [(ngModel)]="username" name="username">
      <input [(ngModel)]="password" name="password" type="password">
      <button type="submit">Login</button>
    </form>
    <div *ngIf="loading$ | async">Loading...</div>
    <div *ngIf="error$ | async as error">{{ error }}</div>
  `
})
export class LoginComponent {
  username = '';
  password = '';
  loading$ = this.store.select(AuthState.loading);
  error$ = this.store.select(AuthState.error);

  constructor(private store: Store) {}

  onSubmit() {
    this.store.dispatch(new Login(this.username, this.password));
  }
}
```

### 3. Akita (Simple and Powerful)

Akita provides a simple yet powerful state management solution.

```typescript
// auth.store.ts
import { Injectable } from '@angular/core';
import { EntityStore, EntityState } from '@datorama/akita';

export interface User {
  id: number;
  name: string;
  email: string;
}

export interface AuthState extends EntityState<User> {
  loading: boolean;
  error: string | null;
}

@Injectable({ providedIn: 'root' })
export class AuthStore extends EntityStore<AuthState> {
  constructor() {
    super({
      loading: false,
      error: null
    });
  }

  setLoading(loading: boolean) {
    this.update({ loading });
  }

  setError(error: string | null) {
    this.update({ error });
  }
}

// auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  constructor(private authStore: AuthStore) {}

  login(username: string, password: string) {
    this.authStore.setLoading(true);
    this.authStore.setError(null);

    return this.http.post<User>('/api/auth/login', { username, password })
      .pipe(
        tap(user => {
          this.authStore.add(user);
          this.authStore.setLoading(false);
        }),
        catchError(error => {
          this.authStore.setError(error.message);
          this.authStore.setLoading(false);
          return throwError(() => error);
        })
      );
  }
}

// Using in Component
@Component({
  selector: 'app-login',
  template: `
    <form (ngSubmit)="onSubmit()">
      <input [(ngModel)]="username" name="username">
      <input [(ngModel)]="password" name="password" type="password">
      <button type="submit">Login</button>
    </form>
    <div *ngIf="loading$ | async">Loading...</div>
    <div *ngIf="error$ | async as error">{{ error }}</div>
  `
})
export class LoginComponent {
  username = '';
  password = '';
  loading$ = this.authStore.selectLoading();
  error$ = this.authStore.selectError();

  constructor(private authService: AuthService) {}

  onSubmit() {
    this.authService.login(this.username, this.password);
  }
}
```

### 4. Simple Service with BehaviorSubject

For smaller applications, a service with BehaviorSubject can be sufficient.

```typescript
// auth.service.ts
@Injectable({
  providedIn: 'root'
})
export class AuthService {
  private userSubject = new BehaviorSubject<User | null>(null);
  private loadingSubject = new BehaviorSubject<boolean>(false);
  private errorSubject = new BehaviorSubject<string | null>(null);

  user$ = this.userSubject.asObservable();
  loading$ = this.loadingSubject.asObservable();
  error$ = this.errorSubject.asObservable();

  constructor(private http: HttpClient) {}

  login(username: string, password: string) {
    this.loadingSubject.next(true);
    this.errorSubject.next(null);

    return this.http.post<User>('/api/auth/login', { username, password })
      .pipe(
        tap(user => {
          this.userSubject.next(user);
          this.loadingSubject.next(false);
        }),
        catchError(error => {
          this.errorSubject.next(error.message);
          this.loadingSubject.next(false);
          return throwError(() => error);
        })
      );
  }

  logout() {
    this.userSubject.next(null);
  }
}

// Using in Component
@Component({
  selector: 'app-login',
  template: `
    <form (ngSubmit)="onSubmit()">
      <input [(ngModel)]="username" name="username">
      <input [(ngModel)]="password" name="password" type="password">
      <button type="submit">Login</button>
    </form>
    <div *ngIf="loading$ | async">Loading...</div>
    <div *ngIf="error$ | async as error">{{ error }}</div>
  `
})
export class LoginComponent implements OnInit, OnDestroy {
  username = '';
  password = '';
  loading$ = this.authService.loading$;
  error$ = this.authService.error$;
  private destroy$ = new Subject<void>();

  constructor(private authService: AuthService) {}

  ngOnInit() {
    // Subscribe to user$ if needed
    this.authService.user$
      .pipe(takeUntil(this.destroy$))
      .subscribe(user => {
        if (user) {
          // Handle successful login
        }
      });
  }

  onSubmit() {
    this.authService.login(this.username, this.password);
  }

  ngOnDestroy() {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

### When to Use Each Solution

1. **NgRx**:
   - Large applications with complex state
   - Need for time-travel debugging
   - Multiple developers working on the same codebase
   - Complex state interactions

2. **NGXS**:
   - Medium to large applications
   - Want simpler syntax than NgRx
   - Need dependency injection
   - Prefer TypeScript decorators

3. **Akita**:
   - Applications with entity collections
   - Need for simple CRUD operations
   - Want built-in dev tools
   - Prefer simpler API

4. **Service with BehaviorSubject**:
   - Small to medium applications
   - Simple state management needs
   - Want minimal dependencies
   - Quick development

### Best Practices

1. **State Normalization**:
```typescript
// Normalized state structure
interface AppState {
  entities: {
    users: { [id: number]: User };
    posts: { [id: number]: Post };
  };
  ids: {
    users: number[];
    posts: number[];
  };
}
```

2. **Selectors**:
```typescript
// Memoized selectors
export const selectUserById = createSelector(
  selectUsers,
  (users: User[], props: { id: number }) => users.find(user => user.id === props.id)
);
```

3. **Effects**:
```typescript
// Error handling in effects
@Effect()
login$ = this.actions$.pipe(
  ofType(AuthActions.login),
  mergeMap(action =>
    this.authService.login(action.payload).pipe(
      map(user => AuthActions.loginSuccess({ user })),
      catchError(error => of(AuthActions.loginFailure({ error: error.message })))
    )
  )
);
```

4. **State Updates**:
```typescript
// Immutable state updates
this.store.dispatch(new UpdateUser({
  id: 1,
  changes: { name: 'New Name' }
}));
```

### Conclusion

Choosing the right state management solution depends on your application's needs:

1. For complex applications: Use NgRx
2. For medium applications: Use NGXS or Akita
3. For simple applications: Use a service with BehaviorSubject

Key considerations:
- Application size and complexity
- Team size and expertise
- Development timeline
- Maintenance requirements
- Performance needs

Remember to:
- Keep state normalized
- Use proper typing
- Implement proper error handling
- Follow best practices for the chosen solution
- Consider performance implications
- Maintain proper documentation
</rewritten_file> 