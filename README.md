# Rx-sanity

The human-friendly way to use RxJs.


```javascript
npm install --save rx-sanity
```

## 1. Syntax Basics

This syntax will be the base for powerful, intuitive, human-friendly code:

```javascript
import { Rx } from 'rx-sanity'

// React to an observable and call a callback
Rx(user$, user => console.log(user))

// React to *any* of the observables and call a callback
Rx([user$, tenant$], (user, tenant) => console.log(user, tenant))

// And of course: ifHasSubscribers
Rx(user$, { ifHasSubscribers: user$ }, user => console.log(user))
```

Also, it can **auto-unsubscribe**:

```javascript
@rxSanity // adds "this.Rx" which auto-unsubscribes onDestroy
class AnythingWithOnDestroy {
  constructor() {
    this.Rx(this.user$, user => console.log(user)) // auto-unsubscribes

    // Calls onProfileSave() by default when no callback is provided.
    // It also auto-unsubscribes.
    this.Rx(this.profileSave$)
  }

  onProfileSave() {}
}
```

***Question:*** "Doesn't that run even if it's not in the view?  It's not using async pipe. That's not good."

***Answer:*** Yes it does run even if not in the view, and... that depends.   (1) The `ifHasSubscribers` option is there for that, and (2) remember, this *is* a component afterall: if the component expects code to run when the component is on the page, it can do just that - cleanly.  *(For services and helpers it's a completely different story.  More on that later.)*


## 2. Clarity - in Components

Let's look at a simple component in a TODO app.  Ignoring the template, which is mostly the same between examples:

<br />

<table>
<tr>
<td style="width: 50%" width="50%" valign="top">

### Rx-Sanity RxJs

</td>
<td width="50%">

### Traditional RxJs

</td>
</tr>
<tr>
<td style="width: 50%" width="50%" valign="top">

```javascript
@rxSanity // defines setters/getters for MiniStores
class TodoListComponent {
  currentUser$ = this.userStore.currentUser$
  todos$ = new MiniStore(undefined)
  allDone$ = new MiniStore(undefined)
  error$ = new MiniStore(undefined)
  userName: string

  constructor(private userStore: UserStore) {
    this.Rx(this.currentUser$)
    this.Rx(this.todos$)
  }

  onCurrentUser(user) {
    this.userName = user?.name
    if (user) {
      this.todos = undefined
      switchFetch('todos', 'GET', todosPath(user.id))
        .then(todos => this.todos = todos)
        .catch(error => this.error = error)
    } else {
      this.todos = []
    }
  }

  onTodos(todos) {
    this.allDone = todos ? !todos.some(todo => !todo.done) : undefined
  }
}
```

</td>
<td width="50%">

```javascript

class TodoListComponent {
  currentUser$ = this.userStore.currentUser$
  todos$ = new BehaviorSubject(undefined)
  allDone$ = new BehaviorSubject(undefined)
  error$ = new BehaviorSubject(undefined)
  userName$: Observable<string>

  constructor(private userStore: UserStore) {
    this.userName$ = this.currentUser$.pipe(pluck('name'))

    this.todos$ = this.currentUser$.pipe(
      switchMap(currentUser =>
        currentUser
          ? fromFetch('GET', todosPath(currentUser.id))
              .pipe(startWith(undefined))
          : of([])
      ),
      catchError(error => this.error$.next(error)),
      share(),
    )

    this.allDone$ = this.todos$.pipe(
      map(todos =>
        todos ? !todos.some(todo => !todo.done) : undefined
      ),
    )
  }
}

```

</td>
</tr>
<tr>
<td style="width: 50%" width="50%" valign="top">


#### Pros
  * Human friendly.
  * Devtools and debugger friendly.
  * Testing is a breeze.
  * No "pipes-without-subscribers" timing issues: when the observables don't have subscribers at expected times: e.g. ngIf hides the element with the "async pipe".
  * No "leaked-hidden-subscriptions" timing issues: e.g. via `shareReplay`.


#### Cons
  * Could have "shared-state" timing issues: when multiple functions could set `currentUser` while another function is expecting `currentUser` to stay the same before and after an asynchronous call.
  * `switchFetch` is odd.
  * Imperative style.

</td>
<td width="50%">

#### Pros
  * Relationships between observables are perfectly solved for the dependent observable.
  * As long as side-effects don't happen (e.g. no calls to RxJS `tap`), there can be no timing issues from other areas of the code modifying shared state.
  * Declarative style.

#### Cons
  * Relationships between observables MUST be perfectly solved for the dependent observable - which is complicated when considering the presence or lack of subscribers and how that affects things like `withLatestFrom`.
  * Less human friendly.
  * Devtools and debugger unfriendly.
  * Everything is an `Observable<something>` or function argument.  If it's not in a pipe, it's probably wrong.
  * The pipes are "like callback hell" (not my quote).
  * The words `pipe` `switchMap` `of` `next` `share` and `map` are odd.


</td>
</tr>
</table>

<br />

If you're thinking **"Okay, but they're practically the same and the one on the right is declarative"**, you're right. So far the code looks mostly similar.

#### Let's add a feature.

Our PM insists that our client's customers' users want to be able to add a TODO task, not just stare at the list all day.

We'll assume there's a new component in the template for adding a TODO task which is wired up to call a new method `onAddTodo` with the new, unsaved task.  So we'll just refresh the list of TODOs after saving to make sure we have the exact and latest TODOs from the server.

<br />

<table>
<tr>
<td style="width: 50%" width="50%" valign="top">

### Rx-Sanity RxJs

</td>
<td width="50%">

### Traditional RxJs

</td>
</tr>
<tr>
<td style="width: 50%" width="50%" valign="top">

```javascript
@rxSanity
class TodoListComponent {
  currentUser$ = this.userStore.currentUser$
  todos$ = new MiniStore(undefined)
  allDone$ = new MiniStore(undefined)
  error$ = new MiniStore(undefined)
  userName: string
  saving$ = new MiniStore(false) // ADDED

  constructor(private userStore: UserStore) {
    this.Rx(this.currentUser$)
    this.Rx(this.todos$)
  }

  onCurrentUser(user) {
    this.userName = user?.name
    if (user) {
      this.todos = undefined
      this.fetchTodos() // CHANGED (refactored)
    } else {
      this.todos = []
    }
  }

  onTodos(todos) {
    this.allDone = todos ? !todos.some(todo => !todo.done) : undefined
  }

  // ADDED
  onAddTodo(todo) {
    this.saving = true
    fetch('POST', todosPath(), todo)
      .then(() => this.fetchTodos())
      .catch(error => this.error = error)
      .finally(() => this.saving = false)
  }

  // (refactored)
  fetchTodos() {
    switchFetch('todos', 'GET', todosPath(this.currentUser.id))
      .then(todos => this.todos = todos)
      .catch(error => this.error = error)
  }
}
```

</td>
<td width="50%">


```javascript

class TodoListComponent {
  currentUser$ = this.userStore.currentUser$
  todos$ = new BehaviorSubject(undefined)
  allDone$ = new BehaviorSubject(undefined)
  error$ = new BehaviorSubject(undefined)
  saving$ = new BehaviorSubject(false) // ADDED
  addTodo$ = new Subject() // ADDED
  addedTodo$ = new Subject() // ADDED

  constructor(private userStore: UserStore) {
    this.userName$ = this.currentUser$.pipe(pluck('name'))

    this.todos$ = combineLatest([ // CHANGED
      this.currentUser$,
      this.addedTodo$.pipe(startWith(undefined)), // ADDED
    ]).pipe(
      switchMap(([currentUser, _]) => // CHANGED
        currentUser
          ? fromFetch('GET', todosPath(currentUser.id))
              .pipe(startWith(undefined)) // WRONG
          : of([])
      ),
      catchError(error => this.error$.next(error)),
      share(),
    )

    this.allDone$ = this.todos$.pipe(
      map(todos =>
        todos ? !todos.some(todo => !todo.done) : undefined
      )
    )

    this.addedTodo$ = this.addTodo$.pipe( // ADDED
      tap(() => this.saving$.next(true)),
      switchMap(todo =>
        fromFetch('POST', todosPath(), todo).pipe(
          finalize(() => this.saving$.next(false))
        )
      ),
      catchError(error => this.error$.next(error)),
      share(),
    )
  }

  // ADDED
  onAddTodo(todo) {
    this.addTodo$.next(todo)
    // If the view is no longer subscribed to anything that
    // would cause the pipe to execute the POST request, we
    // still want to save the TODO task.  Subscribe here to
    // make that happen.
    this.addedTodo$.pipe(take(1)).subscribe()
  }
}

```

</td>
</tr>
<tr>
<td style="width: 50%" width="50%" valign="top">


#### Pros
  * Still easy to read
  * Still easy to test
  * Can use async/await if you want
  * Easy to organize / refactor
  * Code is cleaner than before
  * Adding functionality was achieved by adding, not changing or removing.

#### Cons
  * ... none?

</td>
<td width="50%">

#### Pros
  * (Mostly) perfectly represents the relationships between observables.
  * It's more precise: it doesn't do an unnecessary refresh of the TODOs after save when the view no longer had a subscription to `todos$` (by using the intermediate observable `addedTodo$`)

#### Cons
  * Harder to read
  * Harder to test
  * Could not make the distinction between using `startWith(undefined)` and not using it when fetching TODOs for when `currentUser` emits but not for when `addedTodo` emits (without adding even more complication).  So these implementations are no longer functionally equivalent.
  * The composition and pipes for `this.todos$` had to be changed.

</td>
</tr>
</table>

<br />

#### One More Feature?

Now we're being told the users want to **remove** TODOs, too.  One more round of estimating, tasking out, and design-test-implement later:

<br />


<table>
<tr>
<td style="width: 50%" width="50%" valign="top">

### Rx-Sanity RxJs

</td>
<td width="50%">

### Traditional RxJs

</td>
</tr>
<tr>
<td style="width: 50%" width="50%" valign="top">

```javascript
@rxSanity
class TodoListComponent { // CHANGED
  currentUser$ = this.userStore.currentUser$
  todos$ = new MiniStore(undefined)
  allDone$ = new MiniStore(undefined)
  error$ = new MiniStore(undefined)
  userName: string
  saving$ = new MiniStore(false)

  constructor(private userStore: UserStore) { // CHANGED
    this.Rx(this.currentUser$)
    this.Rx(this.todos$)
  }

  onCurrentUser(user) {
    this.userName = user?.name
    if (user) {
      this.todos = undefined
      this.fetchTodos()
    } else {
      this.todos = []
    }
  }

  onTodos(todos) {
    this.allDone = todos ? !todos.some(todo => !todo.done) : undefined
  }

  onAddTodo(todo) {
    this.saving = true
    fetch('POST', todosPath(), todo)
      .then(() => this.fetchTodos())
      .catch(error => this.error = error)
      .finally(() => this.saving = false)
  }

  // ADDED
  onRemoveTodo(todo) {
    this.saving = true
    fetch('DELETE', todoPath(todo.id))
      .then(() => this.fetchTodos())
      .catch(error => this.error = error)
      .finally(() => this.saving = false)
  }

  fetchTodos() {
    switchFetch('todos', 'GET', todosPath(this.currentUser.id))
      .then(todos => this.todos = todos)
      .catch(error => this.error = error)
  }
}
```

</td>
<td width="50%">

```javascript

class TodoListComponent {
  currentUser$ = this.userStore.currentUser$
  todos$ = new BehaviorSubject(undefined)
  allDone$ = new BehaviorSubject(undefined)
  error$ = new BehaviorSubject(undefined)
  userName$: Observable<string>
  saving$ = new BehaviorSubject(false)
  addTodo$ = new Subject()
  addedTodo$ = new Subject()
  removeTodo$ = new Subject() // ADDED
  removedTodo$ = new Subject() // ADDED

  constructor(private userStore: UserStore) {
    this.userName$ = this.currentUser$.pipe(pluck('name'))

    this.todos$ = combineLatest([
      this.currentUser$,
      this.addedTodo$.pipe(startWith(undefined)),
      this.removedTodo$.pipe(startWith(undefined)), // ADDED
    ]).pipe(
      switchMap(([currentUser, _, __]) => // CHANGED
        currentUser
          ? fromFetch('GET', todosPath(currentUser.id))
              .pipe(startWith(undefined)) // WRONG
          : of([])
      ),
      catchError(error => this.error$.next(error)),
      share(),
    )

    this.allDone$ = this.todos$.pipe(
      map(todos =>
        todos ? !todos.some(todo => !todo.done) : undefined
      )
    )

    this.addedTodo$ = this.addTodo$.pipe(
      tap(() => this.saving$.next(true)),
      switchMap(todo =>
        fromFetch('POST', todosPath(), todo).pipe(
          finalize(() => this.saving$.next(false))
        )
      ),
      catchError(error => this.error$.next(error)),
      share(),
    )

    // ADDED
    this.removedTodo$ = this.removeTodo$.pipe(
      tap(() => this.saving$.next(true)),
      switchMap(todo =>
        fromFetch('DELETE', todoPath(todo.id)).pipe(
          finalize(() => this.saving$.next(false))
        )
      ),
      catchError(error => this.error$.next(error)),
      share(),
    )
  }

  onAddTodo(todo) {
    this.addTodo$.next(todo)
    // If the view is no longer subscribed to anything that
    // would cause the pipe to execute the POST request, we
    // still want to save the TODO task.  Subscribe here to
    // make that happen.
    this.addedTodo$.pipe(take(1)).subscribe()
  }

  // ADDED
  onRemoveTodo(todo) {
    this.removeTodo$.next(todo)
    // If the view is no longer subscribed to anything that
    // would cause the pipe to execute the DELETE request, we
    // still want to delete the TODO task.  Subscribe here to
    // make that happen.
    this.removedTodo$.pipe(take(1)).subscribe()
  }
}

```

</td>
</tr>
<tr>
<td style="width: 50%" width="50%" valign="top">


#### Pros
  * Still easy to read.
  * Still easy to test.
  * Adding functionality was achieved again by adding, not changing or removing.  Removing this feature will be just as clean as adding it was.

#### Cons
  * ... none?

</td>
<td width="50%">

#### Pros
  * (Mostly) perfectly represents the relationships between observables.

#### Cons
  * A bit harder to test, but similar to the last revision
  * Its "bulk" is getting much more apparent. (Not counting the 8 lines of comments added with the new features.)
  * The `.pipe(take(1)).subscribe()` calls do **not** communicate the developer's intention.
  * Still could not make the distinction between using `startWith(undefined)` and not using it when fetching TODOs for when `currentUser` emits but not for when `addedTodo` or `removedTodo` emits (without adding even more complication).  So these implementations are still not functionally equivalent.
  * The composition and pipes for `this.todos$` had to be changed, again.
  * The churn on those lines of code is already adding up.  Merge conflicts are more likely.

</td>
</tr>
</table>

<br />

Lastly, let's see how hard it is when a change in requirements happens, everything before now were all purely additive.

**New requiement:** After removing a TODO task, just delete it from our local array of TODOs instead of refreshing the whole list from the server?

<br />

<table>
<tr>
<td style="width: 50%" width="50%" valign="top">

### Rx-Sanity RxJs

</td>
<td width="50%">

### Traditional RxJs

</td>
</tr>
<tr>
<td style="width: 50%" width="50%" valign="top">

```javascript
  ...

  onRemoveTodo(todo) {
    this.saving = true
    fetch('DELETE', todoPath(todo.id))
      .then(() =>
        this.todos = this.todos.remove(todo) // CHANGED
      )
      .catch(error => this.error = error)
      .finally(() => this.saving = false)
  }
  ...
```

</td>
<td width="50%">

```javascripts
  // No.
```

</td>
</tr>
</table>

<br />

It's very apparent how easily the "Traditional" RxJS approach taken on the right is not conducive to productivity OR correctness despite its attempt at declarative programming.  If I were to compare two messes, I'd rather deal with messy code written in the style on the left than the style on the right, and I'd trust a junior developer more to successfully fix the code on the left compared to code on the right.

<br />

## 3. Best Practices

  1. Use `BehaviorSubject`, avoid other `Subjects`.

      1. If you need to skip the starting value, use an `if` statement - you probably need to skip that value any time it appears.  Alternatively use `skip(1)` - rarely.

  1. Prefer `this.Rx` default combining RxJS operator.  E.g. generally avoid `{ using: 'zip' }` or any other RxJS combining methods.

  1. Continue to NOT do imperative DOM manipulations.  Just because a declarative style isn't best in the component file for assigning to the view-model observables that doesn't mean it's *always* best.

  1. Use `OnPush`.  And if you mutate nested data, keep using `OnPush` and use `this.changed` (provided by `@pushHelpers`)
      ```javascript
      @pushHelpers
      @Component({
        changeDetection: ChangeDetectionStrategy.OnPush
      })
      export class AppComponent {
        constructor(protected cdr: ChangeDetectorRef) {
          super();
        }

        functionThatMutatesNestedStuff() {
          this.obj.nested[0].property = "new value";
          this.changed("obj");
        }
      }
      ```

<br />

## 4. Advanced Usage
### Combining

Also, it's woth noting that by default `Rx` uses `combineLatest`, but that can be overridden in exceptional circumstances.

```javascript
Rx([user$, tenant$], { using: 'zip' }, (user, tenant) =>
  console.log(user, tenant)
)
```


## 5. Advanced: Implementation Details

```javascript
export function pushHelpers(klass) {
  let eigenClass = Object.create(PushHelpers.prototype)
  eigenClass.__proto__ = klass.prototype.__proto__
  klass.prototype.__proto__ = eigenClass
}

export function shallowClone(obj) {
  const result = Object.create(obj.__proto__);
  return Object.assign(result, obj);
}

export class PushHelpers {
  protected cdr;
  changed(prop) {
    this[prop] = shallowClone(this[prop]);
    this.cdr.detectChanges(); // hi
  }
}
```


