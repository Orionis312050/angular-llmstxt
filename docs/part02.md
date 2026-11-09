### Template variable declaration
Source: https://angular.dev/guide/templates/variables

# Variables in templates

Angular has two types of variable declarations in templates: local template variables and template reference variables.

HELPFUL: In this guide, “template” does not mean the entire HTML template file. It refers only to a specific template construct or expression within the file.

## Local template variables with `@let`

Angular's `@let` syntax allows you to define a local variable and re-use it across a template, similar to the [JavaScript `let` syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let).

### Using `@let`

Use `@let` to declare a variable whose value is based on the result of a template expression. Angular automatically keeps the variable's value up-to-date with the given expression, similar to [bindings](/guide/templates/binding).

```angular-html
@let name = user.name;
@let greeting = 'Hello, ' + name;
@let data = data$ | async;
@let pi = 3.14159;
@let coordinates = {x: 50, y: 100};
@let longExpression =
  'Lorem ipsum dolor sit amet, consectetur adipiscing elit ' +
  'sed do eiusmod tempor incididunt ut labore et dolore magna ' +
  'Ut enim ad minim veniam...';
```

Each `@let` block can declare exactly one variable. You cannot declare multiple variables in the same block with a comma.

### Referencing the value of `@let`

Once you've declared a variable with `@let`, you can reuse it in the same template:

```angular-html
@let user = user$ | async;

@if (user) {

    @for (snack of user.favoriteSnacks; track snack.id) {
    }

}
```

### Assignability

A key difference between `@let` and JavaScript's `let` is that `@let` cannot be reassigned after declaration. However, Angular automatically keeps the variable's value up-to-date with the given expression.

```angular-html
@let value = 1;

```

### Variable scope

`@let` declarations are scoped to the current view and its descendants. Angular creates a new view at component boundaries and wherever a template might contain dynamic content, such as control flow blocks, `@defer` blocks, or structural directives.

Since `@let` declarations are not hoisted, they **cannot** be accessed by parent views or siblings:

```angular-html
@let topLevel = value;

  @let insideDiv = value;

{{ topLevel }}
{{ insideDiv }}

@if (condition) {
  {{ topLevel + insideDiv }}

  @let nested = value;

  @if (condition) {
    {{ topLevel + insideDiv + nested }}
  }
}

{{ nested }}
```

### Full syntax

The `@let` syntax is formally defined as:

- The `@let` keyword.
- Followed by one or more whitespaces, not including new lines.
- Followed by a valid JavaScript name and zero or more whitespaces.
- Followed by the = symbol and zero or more whitespaces.
- Followed by an Angular expression which can be multi-line.
- Terminated by the `;` symbol.

## Template reference variables

Template reference variables give you a way to declare a variable that references a value from an element in your template.

A template reference variable can refer to the following:

- a DOM element within a template (including [custom elements](https://developer.mozilla.org/en-US/docs/Web/API/Web_components/Using_custom_elements))
- an Angular component or directive
- a [TemplateRef](/api/core/TemplateRef) from an [ng-template](/api/core/ng-template)

You can use template reference variables to read information from one part of the template in another part of the same template.

### Declaring a template reference variable

You can declare a variable on an element in a template by adding an attribute that starts with the hash character (`#`) followed by the variable name.

```angular-html
```

### Assigning values to template reference variables

Angular assigns a value to template variables based on the element on which the variable is declared.

If you declare the variable on a Angular component, the variable refers to the component instance.

```angular-html
```

If you declare the variable on an `<ng-template>` element, the variable refers to a TemplateRef instance which represents the template. For more information, see [How Angular uses the asterisk, \*, syntax](/guide/directives/structural-directives#structural-directive-shorthand) in [Structural directives](/guide/directives/structural-directives).

```angular-html
```

If you declare the variable on any other displayed element, the variable refers to the `HTMLElement` instance.

```angular-html
```

#### Assigning a reference to an Angular directive

Angular directives may have an `exportAs` property that defines a name by which the directive can be referenced in a template:

```angular-ts
@Directive({
  selector: '[dropZone]',
  exportAs: 'dropZone',
})
export class DropZone {
  /* ... */
}
```

When you declare a template variable on an element, you can assign that variable a directive instance by specifying this `exportAs` name:

```angular-html
```

You cannot refer to a directive that does not specify an `exportAs` name.

### Using template reference variables with queries

In addition to using template variables to read values from another part of the same template, you can also use this style of variable declaration to "mark" an element for [component and directive queries](/guide/components/queries).

When you want to query for a specific element in a template, you can declare a template variable on that element and then query for the element based on the variable name.

```angular-html
```

```angular-ts
@Component({
  /* ... */,
  template: `<input #description value="Original description">`,
})
export class AppComponent {
  // Query for the input element based on the template variable name.
  @ViewChild('description') input: ElementRef | undefined;
}
```

See [Referencing children with queries](/guide/components/queries) for more information on queries.

### Template variable scope

Just like variables in JavaScript or TypeScript code, template variables are scoped to the template that declares them.

Similarly, [control flow blocks](guide/templates/control-flow) like `@if` and `@for`, [structural directives](guide/directives/structural-directives), and `<ng-template>` declarations create a new nested template scope, much like JavaScript's control flow statements like `if` and `for` create new lexical scopes. You cannot access template variables within one of these nested templates from outside of its boundaries.

HELPFUL: Define a variable only once in the template so the runtime value remains predictable.

#### Accessing in a nested template

An inner template can access template variables that the outer template defines.

In the following example, changing the text in the `<input>` changes the value in the `<span>` because Angular immediately updates changes through the template variable, `ref1`.

```angular-html

@if (true) {
}
```

In this case, the `@if` block creates a new template scope, which includes the `ref1` variable from its parent scope.

However, accessing a template variable from a child scope in the parent template doesn't work:

```angular-html {avoid}
@if (true) {
}

```

Here, `ref2` is declared in the child scope created by `@if`, and is not accessible from the parent template.

---

### Deferred loading of components
Source: https://angular.dev/guide/templates/defer

# Deferred loading with `@defer`

Deferrable views, also known as `@defer` blocks, reduce the initial bundle size of your application by deferring the loading of code that is not strictly necessary for the initial rendering of a page. This often results in a faster initial load and improvement in Core Web Vitals (CWV), primarily Largest Contentful Paint (LCP) and Time to First Byte (TTFB).

To use this feature, you can declaratively wrap a section of your template in a @defer block:

```angular-html
@defer {
}
```

The code for any components, directives, and pipes inside the `@defer` block is split into a separate JavaScript file and loaded only when necessary, after the rest of the template has been rendered.

Deferrable views support a variety of triggers, prefetching options, and sub-blocks for placeholder, loading, and error state management.

## Which dependencies are deferred?

Components, directives, pipes, and any component CSS styles can be deferred when loading an application.

In order for the dependencies within a `@defer` block to be deferred, they need to meet two conditions:

1. **They must be standalone.** Non-standalone dependencies cannot be deferred and are still eagerly loaded, even if they are inside of `@defer` blocks.
1. **They cannot be referenced outside of `@defer` blocks within the same file.** If they are referenced outside the `@defer` block or referenced within ViewChild queries, the dependencies will be eagerly loaded.

The _transitive_ dependencies of the components, directives and pipes used in the `@defer` block do not strictly need to be standalone; transitive dependencies can still be declared in an `NgModule` and participate in deferred loading.

Angular's compiler produces a [dynamic import](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import) statement for each component, directive, and pipe used in the `@defer` block. The main content of the block renders after all the imports resolve. Angular does not guarantee any particular order for these imports.

## How to manage different stages of deferred loading

`@defer` blocks have several sub blocks to allow you to gracefully handle different stages in the deferred loading process.

### `@defer`

This is the primary block that defines the section of content that is lazily loaded. It is not rendered initially– deferred content loads and renders once the specified [trigger](#controlling-deferred-content-loading-with-triggers) occurs or the `when` condition is met.

By default, a `@defer` block is triggered when the browser state becomes [idle](/guide/templates/defer#idle).

```angular-html
@defer {
}
```

### Show placeholder content with `@placeholder`

By default, defer blocks do not render any content before they are triggered.

The `@placeholder` is an optional block that declares what content to show before the `@defer` block is triggered.

```angular-html
@defer {
} @placeholder {
}
```

While optional, certain triggers may require the presence of either a `@placeholder` or a [template reference variable](/guide/templates/variables#template-reference-variables) to function. See the [Triggers](#controlling-deferred-content-loading-with-triggers) section for more details.

Angular replaces placeholder content with the main content once loading is complete. You can use any content in the placeholder section including plain HTML, components, directives, and pipes. Keep in mind the _dependencies of the placeholder block are eagerly loaded_.

The `@placeholder` block accepts an optional parameter to specify the `minimum` amount of time that this placeholder should be shown after the placeholder content initially renders.

```angular-html
@defer {
} @placeholder (minimum 500ms) {
}
```

This `minimum` parameter is specified in time increments of milliseconds (ms) or seconds (s). You can use this parameter to prevent fast flickering of placeholder content in the case that the deferred dependencies are fetched quickly.

### Show loading content with `@loading`

The `@loading` block is an optional block that allows you to declare content that is shown while deferred dependencies are loading. It replaces the `@placeholder` block once loading is triggered.

```angular-html
@defer {
} @loading {
} @placeholder {
}
```

Its dependencies are eagerly loaded (similar to `@placeholder`).

The `@loading` block accepts two optional parameters to help prevent fast flickering of content that may occur when deferred dependencies are fetched quickly,:

- `minimum` - the minimum amount of time that this placeholder should be shown
- `after` - the amount of time to wait after loading begins before showing the loading template

```angular-html
@defer {
} @loading (after 100ms; minimum 1s) {
}
```

Both parameters are specified in time increments of milliseconds (ms) or seconds (s). In addition, the timers for both parameters begin immediately after the loading has been triggered.

### Show error state when deferred loading fails with `@error`

The `@error` block is an optional block that displays if deferred loading fails. Similar to `@placeholder` and `@loading`, the dependencies of the @error block are eagerly loaded.

```angular-html
@defer {
} @error {
}
```

## Controlling deferred content loading with triggers

You can specify **triggers** that control when Angular loads and displays deferred content.

When a `@defer` block is triggered, it replaces placeholder content with lazily loaded content.

Multiple event triggers can be defined by separating them with a semicolon, `;` and will be evaluated as OR conditions.

There are two types of triggers: `on` and `when`.

### `on`

`on` specifies a condition for when the `@defer` block is triggered.

The available triggers are as follows:

| Trigger                       | Description                                                            |
| ----------------------------- | ---------------------------------------------------------------------- |
| [`idle`](#idle)               | Triggers when the browser is idle. Supports an optional timeout.       |
| [`viewport`](#viewport)       | Triggers when specified content enters the viewport                    |
| [`interaction`](#interaction) | Triggers when the user interacts with specified element                |
| [`hover`](#hover)             | Triggers when the mouse hovers over specified area                     |
| [`immediate`](#immediate)     | Triggers immediately after non-deferred content has finished rendering |
| [`timer`](#timer)             | Triggers after a specific duration                                     |

#### `idle`

The `idle` trigger loads the deferred content once the browser has reached an idle state, based on requestIdleCallback. This is the default behavior with a defer block.

You can optionally specify a timeout in milliseconds that is passed to [`requestIdleCallback`](https://developer.mozilla.org/docs/Web/API/Window/requestIdleCallback). If the browser doesn't schedule the callback soon enough, the work will run no later than the specified timeout.

```angular-html
@defer {
} @placeholder {
}

@defer (on idle(500)) {
}
```

##### Customizing `idle` behavior

You can customize the `idle` trigger by providing your own `IdleService` implementation and registering it with `provideIdleServiceWith` in your application's providers.

```ts
@Service()
class CustomIdleService implements IdleService {
  requestOnIdle(callback: (deadline?: IdleDeadline) => void, options?: IdleRequestOptions): number {
    // Custom idle scheduling logic can be implemented here.
    return requestIdleCallback(callback, options);
  }

  cancelOnIdle(id: number) {
    // Implement custom idle cancellation here.
  }
}

bootstrapApplication(App, {
  providers: [provideIdleServiceWith(CustomIdleService)],
});
```

#### `viewport`

The `viewport` trigger loads the deferred content when the specified content enters the viewport using the [Intersection Observer API](https://developer.mozilla.org/docs/Web/API/Intersection_Observer_API). Observed content may be `@placeholder` content or an explicit element reference.

By default, the `@defer` watches for the placeholder entering the viewport. Placeholders used this way must have a single root element.

```angular-html
@defer (on viewport) {
} @placeholder {
}
```

Alternatively, you can specify a [template reference variable](/guide/templates/variables) in the same template as the `@defer` block as the element that is watched to enter the viewport. This variable is passed in as a parameter on the viewport trigger.

```angular-html
@defer (on viewport(greeting)) {
}
```

If you want to customize the options of the `IntersectionObserver`, the `viewport` trigger supports passing in an object literal. The literal supports all properties from the second parameter of `IntersectionObserver`, except for `root`. When using the object literal notation, you have to pass your trigger using the `trigger` property.

```angular-html

@defer (
  on viewport({trigger: greeting, rootMargin: '100px', scrollMargin: '50px', threshold: 0.5})
) {
}

@defer (on viewport({rootMargin: '100px', scrollMargin: '50px', threshold: 0.5})) {
} @placeholder {
}
```

#### `interaction`

The `interaction` trigger loads the deferred content when the user interacts with the specified element through `click` or `keydown` events.

By default, the placeholder acts as the interaction element. Placeholders used this way must have a single root element.

```angular-html
@defer (on interaction) {
} @placeholder {
}
```

Alternatively, you can specify a [template reference variable](/guide/templates/variables) in the same template as the `@defer` block as the element that is watched for interactions. This variable is passed in as a parameter on the interaction trigger.

```angular-html
@defer (on interaction(greeting)) {
}
```

#### `hover`

The `hover` trigger loads the deferred content when the mouse has hovered over the triggered area through the `mouseover` and `focusin` events.

By default, the placeholder acts as the interaction element. Placeholders used this way must have a single root element.

```angular-html
@defer (on hover) {
} @placeholder {
}
```

Alternatively, you can specify a [template reference variable](/guide/templates/variables) in the same template as the `@defer` block as the element that is hovered over. This variable is passed in as a parameter on the hover trigger.

```angular-html
@defer (on hover(greeting)) {
}
```

#### `immediate`

The `immediate` trigger loads the deferred content immediately. This means that the deferred block loads as soon as all other non-deferred content has finished rendering.

```angular-html
@defer (on immediate) {
} @placeholder {
}
```

#### `timer`

The `timer` trigger loads the deferred content after a specified duration.

```angular-html
@defer (on timer(500ms)) {
} @placeholder {
}
```

The duration parameter must be specified in milliseconds (`ms`) or seconds (`s`).

### `when`

The `when` trigger accepts a custom conditional expression and loads the deferred content when the condition becomes truthy.

```angular-html
@defer (when condition) {
} @placeholder {
}
```

This is a one-time operation– the `@defer` block does not revert back to the placeholder if the condition changes to a falsy value after becoming truthy.

## Prefetching data with `prefetch`

In addition to specifying a condition that determines when deferred content is shown, you can optionally specify a **prefetch trigger**. This trigger lets you load the JavaScript associated with the `@defer` block before the deferred content is shown.

Prefetching enables more advanced behaviors, such as letting you start to prefetch resources before a user has actually seen or interacted with a defer block, but might interact with it soon, making the resources available faster.

You can specify a prefetch trigger similarly to the block's main trigger, but prefixed with the `prefetch` keyword. The block's main trigger and prefetch trigger are separated with a semi-colon character (`;`).

In the example below, the prefetching starts when a browser becomes idle and the contents of the block is rendered only once the user interacts with the placeholder.

```angular-html
@defer (on interaction; prefetch on idle) {
} @placeholder {
}

@defer (on interaction; prefetch on idle(500)) {
}
```

## Testing `@defer` blocks

Angular provides TestBed APIs to simplify the process of testing `@defer` blocks and triggering different states during testing. By default, `@defer` blocks in tests play through like a defer block would behave in a real application. If you want to manually step through states, you can switch the defer block behavior to `Manual` in the TestBed configuration.

```angular-ts
it('should render a defer block in different states', async () => {
  // configures the defer block behavior to start in "paused" state for manual control.
  TestBed.configureTestingModule({deferBlockBehavior: DeferBlockBehavior.Manual});
  @Component({
    // ...
    template: `
      @defer {
      } @placeholder {
        Placeholder
      } @loading {
        Loading...
      }
    `,
  })
  class ExampleA {}
  // Create component fixture.
  const componentFixture = TestBed.createComponent(ExampleA);
  // Retrieve the list of all defer block fixtures and get the first block.
  const deferBlockFixture = (await componentFixture.getDeferBlocks())[0];
  // Renders placeholder state by default.
  expect(componentFixture.nativeElement.innerHTML).toContain('Placeholder');
  // Render loading state and verify rendered output.
  await deferBlockFixture.render(DeferBlockState.Loading);
  expect(componentFixture.nativeElement.innerHTML).toContain('Loading');
  // Render final state and verify the output.
  await deferBlockFixture.render(DeferBlockState.Complete);
  expect(componentFixture.nativeElement.innerHTML).toContain('large works!');
});
```

## Does `@defer` work with `NgModule`?

`@defer` blocks are compatible with both standalone and NgModule-based components, directives and pipes. However, **only standalone components, directives and pipes can be deferred**. NgModule-based dependencies are not deferred and are included in the eagerly loaded bundle.

## Compatibility between `@defer` blocks and Hot Module Reload (HMR)

When Hot Module Replacement (HMR) is active, all `@defer` block chunks are fetched eagerly, overriding any configured triggers. To restore the standard trigger behavior, you must disable HMR by serving your application with the `--no-hmr` flag.

## How does `@defer` work with server-side rendering (SSR) and static-site generation (SSG)?

By default, when rendering an application on the server (either using SSR or SSG), defer blocks always render their `@placeholder` (or nothing if a placeholder is not specified) and triggers are not invoked. On the client, the content of the `@placeholder` is hydrated and triggers are activated.

To render the main content of `@defer` blocks on the server (both SSR and SSG), you can enable [the Incremental Hydration feature](/guide/incremental-hydration) and configure `hydrate` triggers for the necessary blocks.

## Barrel files and lazy chunks

If you're using `@defer` but not seeing a separate lazy chunk in your build output, check how you're importing the deferred component. Importing through a barrel file (`index.ts`) is a common culprit — bundlers see the barrel as a single module and keep all its exports together, so your component ends up in the main bundle regardless of `@defer`.

```typescript {header: "index.ts"}
export {HeavyComponent} from './heavy.component';
export {OtherComponent} from './other.component';
```

```typescript {header: "parent.component.ts"}
import {Component} from '@angular/core';
import {HeavyComponent} from './index'; // pulls in OtherComponent too

@Component({
  imports: [HeavyComponent],
  template: `@defer {
  }`,
})
export class ParentComponent {}
```

The fix is straightforward — import directly from the component's own file:

```typescript
import {HeavyComponent} from './heavy.component';
```

That's enough for the bundler to split it into its own chunk and load it lazily when the trigger fires.

## Best practices for deferring views

### Avoid cascading loads with nested `@defer` blocks

When you have nested `@defer` blocks, they should have different triggers in order to avoid loading simultaneously, which causes cascading requests and may negatively impact page load performance.

### Avoid layout shifts

Avoid deferring components that are visible in the user’s viewport on initial load. Doing this may negatively affect Core Web Vitals by causing an increase in cumulative layout shift (CLS).

In the event this is necessary, avoid `immediate`, `timer`, `viewport`, and custom `when` triggers that cause the content to load during the initial page render.

### Keep accessibility in mind

When using `@defer` blocks, consider the impact on users with assistive technologies like screen readers.
Screen readers that focus on a deferred section will initially read the placeholder or loading content, but may not announce changes when the deferred content loads.

To ensure deferred content changes are announced to screen readers, you can wrap your `@defer` block in an element with a live region:

```angular-html
  @defer (on timer(2000)) {
  } @placeholder {
    Loading user profile...
  } @loading {
    Please wait...
  } @error {
    Failed to load profile
  }
```

This ensures that changes are announced to the user when transitions (placeholder &rarr; loading &rarr; content/error) occur.

---

### Expression syntax
Source: https://angular.dev/guide/templates/expression-syntax

# Expression Syntax

Angular expressions are based on JavaScript, but differ in some key ways. This guide walks through the similarities and differences between Angular expressions and standard JavaScript.

## Value literals

Angular supports a subset of [literal values](https://developer.mozilla.org/en-US/docs/Glossary/Literal) from JavaScript.

### Supported value literals

| Literal type           | Example values                  |
| ---------------------- | ------------------------------- |
| String                 | `'Hello'`, `"World"`            |
| Boolean                | `true`, `false`                 |
| Number                 | `123`, `3.14`                   |
| Object                 | `{name: 'Alice'}`               |
| Array                  | `['Onion', 'Cheese', 'Garlic']` |
| null                   | `null`                          |
| RegExp                 | `/\d+/`                         |
| Template string        | `` `Hello ${name}` ``           |
| Tagged template string | `` tag`Hello ${name}` ``        |

### Unsupported value literals

| Literal type | Example values |
| ------------ | -------------- |
| BigInt       | `1n`           |

## Globals

Angular expressions support the following [globals](https://developer.mozilla.org/en-US/docs/Glossary/Global_object):

- [undefined](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/undefined)
- [$any](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#any)

No other JavaScript globals are supported. Common JavaScript globals include `Number`, `Boolean`, `NaN`, `Infinity`, `parseInt`, and more.

## Local variables

Angular automatically makes special local variables available for use in expressions in specific contexts. These special variables always start with the dollar sign character (`$`).

For example, `@for` blocks make several local variables corresponding to information about the loop, such as `$index`.

## What operators are supported?

### Supported operators

Angular supports the following operators from standard JavaScript.

| Operator                      | Example(s)                                     |
| ----------------------------- | ---------------------------------------------- |
| Add / Concatenate             | `1 + 2`                                        |
| Subtract                      | `52 - 3`                                       |
| Multiply                      | `41 * 6`                                       |
| Divide                        | `20 / 4`                                       |
| Remainder (Modulo)            | `17 % 5`                                       |
| Exponentiation                | `10 ** 3`                                      |
| Parenthesis                   | `9 * (8 + 4)`                                  |
| Conditional (Ternary)         | `a > b ? true : false`                         |
| And (Logical)                 | `&&`                                           |
| Or (Logical)                  | `\|\|`                                         |
| Not (Logical)                 | `!`                                            |
| Nullish Coalescing            | `possiblyNullValue ?? 'default'`               |
| Comparison Operators          | `<`, `<=`, `>`, `>=`, `==`, `===`, `!==`, `!=` |
| Unary Negation                | `-x`                                           |
| Unary Plus                    | `+y`                                           |
| Property Accessor             | `person['name']`                               |
| typeof                        | `typeof 42`                                    |
| void                          | `void 1`                                       |
| in                            | `'model' in car`                               |
| instanceof                    | `car instanceof Automobile`                    |
| Assignment                    | `a = b`                                        |
| Increment                     | `a++`, `++a`                                   |
| Decrement                     | `a--`, `--a`                                   |
| Addition Assignment           | `a += b`                                       |
| Subtraction Assignment        | `a -= b`                                       |
| Multiplication Assignment     | `a *= b`                                       |
| Division Assignment           | `a /= b`                                       |
| Remainder Assignment          | `a %= b`                                       |
| Exponentiation Assignment     | `a **= b`                                      |
| Logical AND Assignment        | `a &&= b`                                      |
| Logical OR Assignment         | `a \|\|= b`                                    |
| Nullish Coalescing Assignment | `a ??= b`                                      |
| Spread in object literals     | `{...obj, foo: 'bar'}`                         |
| Spread in array literals      | `[...arr, 1, 2, 3]`                            |
| Rest in function calls        | `fn(...args)`                                  |
| Arrow function                | `items.filter((i) => i.active)`                |

NOTE: Assignment and increment/decrement operators are only valid in [event listener statements](#event-listener-statements). Using them in a binding or interpolation is a compile error, unless they are inside an arrow function.

Arrow functions must have an expression body. Pipes cannot be used inside an arrow function, but you can pass an arrow function as an argument to a pipe.

Angular expressions additionally also support the following non-standard operators:

| Operator                        | Example(s)                     |
| ------------------------------- | ------------------------------ |
| [Pipe](/guide/templates/pipes)  | `{{ total \| currency }}`      |
| Optional chaining\*             | `someObj.someProp?.nestedProp` |
| Non-null assertion (TypeScript) | `someObj!.someProp`            |

### Safe navigation migration

Prior to Angular 22, the optional chaining operator (`?.`) returned `null` when the left-hand side is `null` or `undefined`, whereas standard JavaScript's `?.` returns `undefined`.
Since Angular 22, the optional chaining operator behavior in Angular expressions is aligned with the standard JavaScript behavior.

During the migration to v22, the `ng update` schematics adds a `$safeNavigationMigration` function to existing expressions to preserve the previous behavior.

```html
{{ $safeNavigationMigration(foo?.bar) }}
```

`$safeNavigationMigration` is a **temporary migration aid only**. It instructs the compiler to compile the wrapped safe-navigation expression using the legacy null-returning semantics rather than the standard JavaScript `?.` semantics. It is not a real function and cannot be called from TypeScript.

NOTE: Prefer migrating expressions to no longer rely on `null` vs `undefined` distinctions so that `$safeNavigationMigration` can be removed. This function may be removed in a future version of Angular.

### Unsupported operators

| Operator              | Example(s)                        |
| --------------------- | --------------------------------- |
| All bitwise operators | `&`, `&=`, `~`, `\|=`, `^=`, etc. |
| Object destructuring  | `const { name } = person`         |
| Array destructuring   | `const [firstItem] = items`       |
| Comma operator        | `x = (x++, x)`                    |
| new                   | `new Car()`                       |

## Lexical context for expressions

Angular expressions are evaluated within the context of the component class as well as any relevant [template variables](/guide/templates/variables), locals, and globals.

When referring to component class members, `this` is always implied. However, if a template declares a [template variables](guide/templates/variables) with the same name as a member, the variable shadows that member. You can unambiguously reference such a class member by explicitly using `this.`. This can be useful when creating an `@let` declaration that shadows a class member, e.g. for signal narrowing purposes.

## Declarations

Generally speaking, declarations are not supported in Angular expressions. This includes, but is not limited to:

| Declarations                      | Example(s)                                  |
| --------------------------------- | ------------------------------------------- |
| Variables                         | `let label = 'abc'`, `const item = 'apple'` |
| Functions                         | `function myCustomFunction() { }`           |
| Arrow functions with block bodies | `() => { }`                                 |
| Classes                           | `class Rectangle { }`                       |

## Event listener statements

Event handlers are **statements** rather than expressions. While they support all of the same syntax as Angular expressions, there are two key differences:

1. Statements **do support** assignment operators (but not destructuring assignments)
1. Statements **do not support** pipes

---


## Directives

### Directives overview
Source: https://angular.dev/guide/directives

Directives add behavior to elements and components in your Angular applications.

A directive can change how an element looks, how it behaves, or how it fits into the DOM. Angular ships with several built-in directives, and you can write your own.

## When to use a directive

Directives are most effective when they encapsulate **reusable** behavior that you want to apply to an existing element or component.

Common examples include:

- Applying the same appearance or behavior across many elements, such as autofocus or a tooltip.
- Reading from or writing to the host element's DOM, attributes, or classes.
- Adding behavior to a component you don't own without changing its source.

If you need to render your own markup or manage a piece of UI with its own template, reach for a [component](guide/components), a specialized directive with its own template.

## A quick example

Suppose you want elements to highlight when the user hovers over them with a mouse, changing their background color to yellow. Rather than repeat the same event-handling logic on every element, you can package that behavior in a directive and apply it wherever you need it.

The following `appHighlight` directive sets the host element's background color when the mouse enters and clears it when the mouse leaves:

```ts
import {Directive, signal} from '@angular/core';

@Directive({
  selector: '[appHighlight]',
  host: {
    '(mouseenter)': 'isHovered.set(true)',
    '(mouseleave)': 'isHovered.set(false)',
    '[style.background-color]': 'isHovered() ? "yellow" : null',
  },
})
export class HighlightDirective {
  protected isHovered = signal(false);
}
```

The `host` metadata listens for mouse events to update the `isHovered` signal, and binds the host element's `background-color` style to the signal's value.

Apply the directive by adding its selector as an attribute on an element:

```angular-html
```

Every element that carries the `appHighlight` attribute gains the same hover behavior, with the logic defined in one place.

## Types of directives

Angular has three primary types of directives:

| Directive type                                                  | Details                                                                           |
| :-------------------------------------------------------------- | :-------------------------------------------------------------------------------- |
| [Components](guide/components)                                  | Define reusable UI with their own template.                                       |
| [Attribute directives](guide/directives/attribute-directives)   | Change the appearance or behavior of an element, component, or another directive. |
| [Structural directives](guide/directives/structural-directives) | Change the DOM layout by adding and removing DOM elements.                        |

## What's next

Learn more about each type of directive in the following guides.


---

### Attribute directives
Source: https://angular.dev/guide/directives/attribute-directives

# Attribute directives

Attribute directives change the appearance or behavior of DOM elements and Angular components.

## Use template bindings for one-off behavior

Angular's template syntax already covers changing a single element's classes, styles, properties, and events:

- [Class and style bindings](guide/templates/binding#css-class-and-style-property-bindings) add and remove CSS classes and inline styles.
- [Property and attribute bindings](guide/templates/binding) set DOM properties and HTML attributes.
- [Event listeners](guide/templates/event-listeners) respond to user interaction.

Attribute directives are useful when you want to package this kind of behavior into a reusable unit that you can apply to any element or component.

## Building an attribute directive

A custom attribute directive is a JavaScript class with the `@Directive()` decorator. The decorator's `selector` defines the attribute that applies the directive. The square brackets make this an attribute selector, so the directive matches elements that carry the attribute. By convention, use a prefix such as `app` to avoid naming collisions:

```ts
import {Directive} from '@angular/core';

@Directive({
  selector: '[appHighlight]',
})
export class HighlightDirective {}
```

HELPFUL: The CLI command [`ng generate directive`](tools/cli/schematics) scaffolds a directive along with its test file.

A directive can change its host declaratively through host bindings or imperatively through a reference to the host element. This example [injects](guide/di) [`ElementRef`](api/core/ElementRef) and accesses the element through its `nativeElement` property to set the background to yellow:


IMPORTANT: Directives _do not_ support namespaces.

```angular-html {avoid}
```

## Applying an attribute directive

To apply the directive, add its selector as an attribute on an element:


Angular creates an instance of `HighlightDirective` for that `<p>` element, injects a reference to the element, and sets its background to yellow.

## Handling user events

To respond to user interaction, bind host element events to handler methods through the `host` property of the `@Directive()` decorator. The following directive highlights the host element while the pointer is over it and clears the highlight when the pointer leaves:


The `host` property maps the `mouseenter` and `mouseleave` events to the `onMouseEnter()` and `onMouseLeave()` methods, which delegate to a `highlight()` helper that sets the background color on the host element. For more on host event bindings, see [binding to the host element](guide/components/host-elements#binding-to-the-host-element).

## Accepting input values

Like components, directives accept inputs through the [`input()`](guide/components/inputs) function. Give an input the same name as the selector so that a single binding both applies the directive and passes a value to it:


Read the input by calling it as a signal, and fall back to a default when no color is set:


In the template, bind the value to the selector. Because the input shares the selector's name, `[appHighlight]` both applies the directive and sets its value. Here the bound `color` is a property on the component:



A directive can declare more than one input. The following directive adds a `defaultColor` input, then falls back through `appHighlight`, `defaultColor`, and finally `red`:


Bind both inputs on the same element. Because `defaultColor` takes a static string rather than a dynamic expression, it doesn't need square brackets:


## Deactivating Angular processing with `NgNonBindable`

To prevent expression evaluation in the browser, add `ngNonBindable` to the host element.
`ngNonBindable` deactivates interpolation, directives, and binding in templates.

In the following example, the expression `{{ 1 + 1 }}` renders just as it does in your code editor, and does not display `2`.


Applying `ngNonBindable` to an element stops binding for that element's child elements.
However, `ngNonBindable` still lets directives work on the element where you apply `ngNonBindable`.
In the following example, the `appHighlight` directive is still active but Angular does not evaluate the expression `{{ 1 + 1 }}`.


If you apply `ngNonBindable` to a parent element, Angular disables interpolation and binding of any sort, such as property binding or event binding, for the element's children.

## What's next


---

### Structural directives
Source: https://angular.dev/guide/directives/structural-directives

# Structural directives

Structural directives are directives applied to an `<ng-template>` element that conditionally or repeatedly render the content of that `<ng-template>`.

For everyday conditional and repeated rendering, use Angular's built-in [control flow blocks](guide/templates/control-flow) (`@if`, `@for`, and `@switch`). Write a structural directive when you need reusable rendering behavior that control flow doesn't cover, such as gating content behind a permission check or providing a template with data from an external source.

## Example use case

This guide uses a directive called `SelectDirective` as its running example. The directive fetches data from a given data source and renders its template when that data is available. It is named after the SQL keyword `SELECT` and matched with an attribute selector, `[select]`.

`SelectDirective` has an input naming the data source to use, called `selectFrom`. The `select` prefix for this input is important for the [shorthand syntax](#structural-directive-shorthand). The directive instantiates its `<ng-template>` with a template context providing the selected data.

Using this directive directly on an `<ng-template>` looks like this:

```angular-html
```

The structural directive can wait for the data to become available and then render its `<ng-template>`.

HELPFUL: Angular's `<ng-template>` element defines a template that doesn't render anything by default. If you wrap elements in an `<ng-template>` without applying a structural directive, those elements are not rendered.

For more information, see the [ng-template API](api/core/ng-template) documentation.

## Structural directive shorthand

Angular supports a shorthand syntax for structural directives which avoids the need to explicitly author an `<ng-template>` element.

Structural directives can be applied directly on an element by prefixing the directive attribute selector with an asterisk (`*`), such as `*select`. Angular transforms the asterisk in front of a structural directive into an `<ng-template>` that hosts the directive and surrounds the element and its descendants.

You can use this with `SelectDirective` as follows:

```angular-html
```

This example shows the flexibility of structural directive shorthand syntax, which is sometimes called _microsyntax_.

When used in this way, only the structural directive and its bindings are applied to the `<ng-template>`. Any other attributes or bindings on the `<p>` tag are left alone. For example, these two forms are equivalent:

```angular-html

```

Shorthand syntax is expanded through a set of conventions. A more thorough [grammar](#structural-directive-syntax-reference) is defined below, but in the above example, this transformation can be explained as follows:

The first part of the `*select` expression is `let data`, which declares a template variable `data`. Since no assignment follows, the template variable is bound to the template context property `$implicit`.

The second piece of syntax is a key-expression pair, `from source`. `from` is a binding key and `source` is a regular template expression. Binding keys are mapped to properties by transforming them to PascalCase and prepending the structural directive selector. The `from` key is mapped to `selectFrom`, which is then bound to the expression `source`. This is why many structural directives will have inputs that are all prefixed with the structural directive's selector.

## One structural directive per element

You can only apply one structural directive per element when using the shorthand syntax. This is because there is only one `<ng-template>` element onto which that directive gets unwrapped. Multiple directives would require multiple nested `<ng-template>`, and it's unclear which directive should be first. `<ng-container>` can be used to create wrapper layers when multiple structural directives need to be applied around the same physical DOM element or component, which allows the user to define the nested structure.

## Creating a structural directive

A structural directive is a directive class that injects two dependencies:

- [`TemplateRef`](api/core/TemplateRef) gives the directive access to the content of the `<ng-template>` it is applied to.
- [`ViewContainerRef`](api/core/ViewContainerRef) represents the location in the DOM where the directive can render that template.

The directive controls rendering by creating, or not creating, embedded views from the template in the view container. The complete `SelectDirective` looks like this:

```ts
import {Directive, TemplateRef, ViewContainerRef, inject, input} from '@angular/core';

export interface DataSource<T> {
  load(): Promise<T>;
}

@Directive({
  selector: '[select]',
})
export class SelectDirective {
  private templateRef = inject(TemplateRef);
  private viewContainerRef = inject(ViewContainerRef);

  selectFrom = input.required<DataSource<unknown>>();

  async ngOnInit() {
    const data = await this.selectFrom().load();
    this.viewContainerRef.createEmbeddedView(this.templateRef, {
      // Create the embedded view with a context object that contains
      // the data via the key `$implicit`.
      $implicit: data,
    });
  }
}
```

The `selectFrom` input names the data source the directive reads from. It uses [`input.required()`](guide/components/inputs#required-inputs) because the directive can't do anything useful without a data source.

When Angular initializes the directive, it loads the data and then renders the template by calling `createEmbeddedView()`. The second argument is the template's _context object_: values the template can bind to with `let` declarations. Assigning the data to the `$implicit` key makes it the default value that `let-data` (or `let data` in shorthand) receives.

NOTE: This example renders its template once, when the directive initializes. It does not re-render when the bound data source changes.

Once the directive works, consider [adding template type-checking support](#typing-the-directives-context).

HELPFUL: The CLI command [`ng generate directive`](tools/cli/schematics) scaffolds a directive along with its test file.

## Structural directive syntax reference

When you write your own structural directives, use the following syntax:

```ts {hideCopy}
_: prefix = "( :let | :expression ) (';' | ',')? ( :let | :as | :keyExp )_";
```

The following patterns describe each portion of the structural directive grammar:

```ts
as = :export "as" :local ";"?
keyExp = :key ":"? :expression ("as" :local)? ";"?
let = "let" :local "=" :export ";"?
```

| Keyword      | Details                                            |
| :----------- | :------------------------------------------------- |
| `prefix`     | HTML attribute key                                 |
| `key`        | HTML attribute key                                 |
| `local`      | Local variable name used in the template           |
| `export`     | Value exported by the directive under a given name |
| `expression` | Standard Angular expression                        |

### How Angular translates shorthand

Angular translates structural directive shorthand into the normal binding syntax as follows:

| Shorthand                       | Translation                                                     |
| :------------------------------ | :-------------------------------------------------------------- |
| `prefix` and naked `expression` | `[prefix]="expression"`                                         |
| `keyExp`                        | `[prefixKey]="expression"` (The `prefix` is added to the `key`) |
| `let local`                     | `let-local="export"`                                            |

### Shorthand examples

The following table provides shorthand examples:

| Shorthand                                                             | How Angular interprets the syntax                                                                             |
| :-------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------ |
| `*myDir="let item of [1,2,3]"`                                        | `<ng-template myDir let-item [myDirOf]="[1, 2, 3]">`                                                          |
| `*myDir="let item of [1,2,3] as items; trackBy: myTrack; index as i"` | `<ng-template myDir let-item [myDirOf]="[1,2,3]" let-items="myDirOf" [myDirTrackBy]="myTrack" let-i="index">` |
| `*ngComponentOutlet="componentClass"`                                 | `<ng-template [ngComponentOutlet]="componentClass">`                                                          |
| `*ngComponentOutlet="componentClass; inputs: myInputs"`               | `<ng-template [ngComponentOutlet]="componentClass" [ngComponentOutletInputs]="myInputs">`                     |
| `*myDir="exp as value"`                                               | `<ng-template [myDir]="exp" let-value="myDir">`                                                               |

## Improving template type checking for custom directives

You can improve template type checking for custom directives by adding template guards to your directive definition.
These guards help the Angular template type checker find mistakes in the template at compile time, which can avoid runtime errors.
Two different types of guards are possible:

- `ngTemplateGuard_(input)` lets you control how an input expression should be narrowed based on the type of a specific input.
- `ngTemplateContextGuard` is used to determine the type of the context object for the template, based on the type of the directive itself.

This section provides examples of both kinds of guards.
For more information, see [Template type checking](tools/cli/template-typecheck 'Template type-checking guide').

### Type narrowing with template guards

A structural directive in a template controls whether that template is rendered at run time. Some structural directives want to perform type narrowing based on the type of input expression.

There are two narrowings which are possible with input guards:

- Narrowing the input expression based on a TypeScript type assertion function.
- Narrowing the input expression based on its truthiness.

To narrow the input expression by defining a type assertion function:

```ts
// This directive only renders its template if the actor is a user.
// You want to assert that within the template, the type of the `actor`
// expression is narrowed to `User`.
@Directive(...)
class ActorIsUser {
  actor = input<User | Robot>();

  static ngTemplateGuard_actor(dir: ActorIsUser, expr: User | Robot): expr is User {
    // The return statement is unnecessary in practice, but included to
    // prevent TypeScript errors.
    return true;
  }
}
```

Type-checking will behave within the template as if the `ngTemplateGuard_actor` has been asserted on the expression bound to the input.

Some directives only render their templates when an input is truthy. It's not possible to capture the full semantics of truthiness in a type assertion function, so instead a literal type of `'binding'` can be used to signal to the template type-checker that the binding expression itself should be used as the guard:

```ts
@Directive(...)
class CustomIf {
  condition = input.required<boolean>();

  static ngTemplateGuard_condition: 'binding';
}
```

The template type-checker will behave as if the expression bound to `condition` was asserted to be truthy within the template.

### Typing the directive's context

If your structural directive provides a context to the instantiated template, you can properly type it inside the template by providing a static `ngTemplateContextGuard` type assertion function. This function can use the type of the directive to derive the type of the context, which is useful when the type of the directive is generic.

For the `SelectDirective` described above, you can implement an `ngTemplateContextGuard` to correctly specify the data type, even if the data source is generic.

```ts
// Declare an interface for the template context:
export interface SelectTemplateContext<T> {
  $implicit: T;
}

@Directive(...)
export class SelectDirective<T> {
  // The directive's generic type `T` will be inferred from the `DataSource` type
  // passed to the input.
  selectFrom = input.required<DataSource<T>>();

  // Narrow the type of the context using the generic type of the directive.
  static ngTemplateContextGuard<T>(dir: SelectDirective<T>, ctx: any): ctx is SelectTemplateContext<T> {
    // As before the guard body is not used at runtime, and included only to avoid
    // TypeScript errors.
    return true;
  }
}
```

## What's next


---

### Directive composition
Source: https://angular.dev/guide/directives/directive-composition-api

# Directive composition API

Angular directives offer a great way to encapsulate reusable behaviors. Directives can apply attributes, CSS classes, and event listeners to an element.

The _directive composition API_ lets you apply directives to a component's host element from _within_ the component TypeScript class.

## Adding directives to a component

You apply directives to a component by adding a `hostDirectives` property to a component's decorator. Such directives are called _host directives_.

In this example, the `MenuBehavior` directive is applied to the host element of `AdminMenu`. This works similarly to applying the `MenuBehavior` to the `<admin-menu>` element in a template.

```ts
@Component({
  selector: 'admin-menu',
  templateUrl: './admin-menu.html',
  hostDirectives: [MenuBehavior],
})
export class AdminMenu {}
```

When the framework renders a component, Angular also creates an instance of each host directive. The directives' host bindings apply to the component's host element. By default, host directive inputs and outputs are not exposed as part of the component's public API. See [Including inputs and outputs](#including-inputs-and-outputs) below for more information.

Host directives come with the following constraints:

- **Angular applies host directives statically at compile time.** You cannot dynamically add
  directives at runtime.
- **Directives used in `hostDirectives` may not specify `standalone: false`.**
- **Angular ignores the `selector` of directives applied in the `hostDirectives` property.**

## Including inputs and outputs

When you apply `hostDirectives` to your component, the inputs and outputs from the host directives are not included in your component's API by default. You can explicitly include inputs and outputs in your component's API by expanding the entry in `hostDirectives`:

```ts
@Component({
  selector: 'admin-menu',
  templateUrl: './admin-menu.html',
  hostDirectives: [
    {
      directive: MenuBehavior,
      inputs: ['menuId'],
      outputs: ['menuClosed'],
    },
  ],
})
export class AdminMenu {}
```

By explicitly specifying the inputs and outputs, consumers of the component with `hostDirectives` can bind them in a template:

```angular-html
```

Furthermore, you can alias inputs and outputs from a host directive to customize the API of your component:

```ts
@Component({
  selector: 'admin-menu',
  templateUrl: './admin-menu.html',
  hostDirectives: [
    {
      directive: MenuBehavior,
      inputs: ['menuId: id'],
      outputs: ['menuClosed: closed'],
    },
  ],
})
export class AdminMenu {}
```

```angular-html
```

## Adding directives to another directive

You can also add `hostDirectives` to other directives, in addition to components. This enables the transitive aggregation of multiple behaviors.

The following example defines two directives, `Menu` and `Tooltip`, then composes their behavior in `MenuWithTooltip`. Finally, it applies `MenuWithTooltip` to `SpecializedMenuWithTooltip`.

When `SpecializedMenuWithTooltip` is used in a template, it creates instances of all of `Menu`, `Tooltip`, and `MenuWithTooltip`. Each of these directives' host bindings apply to the host element of `SpecializedMenuWithTooltip`.

```ts
@Directive({/* ... */})
export class Menu {}

@Directive({/* ... */})
export class Tooltip {}

// MenuWithTooltip can compose behaviors from multiple other directives
@Directive({
  hostDirectives: [Tooltip, Menu],
})
export class MenuWithTooltip {}

// CustomWidget can apply the already-composed behaviors from MenuWithTooltip
@Directive({
  hostDirectives: [MenuWithTooltip],
})
export class SpecializedMenuWithTooltip {}
```

## Host directive semantics

### Directive execution order

Host directives go through the same lifecycle as components and directives used directly in a template. However, host directives always execute their constructor, lifecycle hooks, and bindings _before_ the component or directive on which they are applied.

The following example shows minimal use of a host directive:

```ts
@Component({
  selector: 'admin-menu',
  templateUrl: './admin-menu.html',
  hostDirectives: [MenuBehavior],
})
export class AdminMenu {}
```

The order of execution here is:

1. `MenuBehavior` instantiated
2. `AdminMenu` instantiated
3. `MenuBehavior` receives inputs (`ngOnInit`)
4. `AdminMenu` receives inputs (`ngOnInit`)
5. `MenuBehavior` applies host bindings
6. `AdminMenu` applies host bindings

This order of operations means that components with `hostDirectives` can override any host bindings specified by a host directive.

This order of operations extends to nested chains of host directives, as shown in the following example.

```ts
@Directive({...})
export class Tooltip { }

@Directive({
  hostDirectives: [Tooltip],
})
export class CustomTooltip { }

@Directive({
  hostDirectives: [CustomTooltip],
})
export class EvenMoreCustomTooltip { }
```

In the example above, the order of execution is:

1. `Tooltip` instantiated
2. `CustomTooltip` instantiated
3. `EvenMoreCustomTooltip` instantiated
4. `Tooltip` receives inputs (`ngOnInit`)
5. `CustomTooltip` receives inputs (`ngOnInit`)
6. `EvenMoreCustomTooltip` receives inputs (`ngOnInit`)
7. `Tooltip` applies host bindings
8. `CustomTooltip` applies host bindings
9. `EvenMoreCustomTooltip` applies host bindings

### Dependency injection

A component or directive that specifies `hostDirectives` can inject the instances of those host directives and vice versa.

When applying host directives to a component, both the component and host directives can define providers.

If a component or directive with `hostDirectives` and those host directives both provide the same injection token, the providers defined by class with `hostDirectives` take precedence over providers defined by the host directives.

### Host directive de-duplication

When the same directive appears more than once in the resolved host directive tree, it is automatically de-duplicated rather than throwing an error. Two deterministic rules are used to decide which match survives.

#### Template match takes precedence

If a directive matches an element once through a **template selector** and also appears as a **host directive**, Angular keeps only the template match and discards all host directive matches.

The mental model is that a host directive match represents `Partial<YourDirective>`, a partial application where only the inputs and outputs explicitly listed in `hostDirectives` are exposed, while a template match represents the full directive with its complete public API.

@Directive({selector: '[hoverable]'})
export class Hoverable {}

```ts
@Component({
  selector: 'app-button',
  hostDirectives: [Hoverable],
})
export class Button {}
```

```angular-html
```

#### Multiple host directive matches are merged

If the same directive appears **more than once as a host directive**, for example, when two directives both declare a common dependency in their `hostDirectives`, Angular merges all instances into a single directive instance. The input and output mappings from all instances are combined.

This resolves the classic [diamond problem](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem) in host directive composition:

```ts
// A shared behavior that both triggers need
@Directive({
  host: {
    '[attr.data-trigger-id]': 'triggerId()',
  },
})
export class TriggerRef {
  readonly triggerId = input(`trigger-${crypto.randomUUID()}`);
}

// Two separate triggers, each declaring TriggerRef as a host directive
@Directive({
  selector: '[popoverTrigger]',
  hostDirectives: [TriggerRef],
})
export class PopoverTrigger {
  readonly triggerRef = inject(TriggerRef);
}

@Directive({
  selector: '[dropdownTrigger]',
  hostDirectives: [TriggerRef],
})
export class DropdownTrigger {
  readonly triggerRef = inject(TriggerRef);
}
```

```angular-html
```

HELPFUL: Because Angular produces only one instance of the shared directive, both `PopoverTrigger` and `DropdownTrigger` receive the same `TriggerRef` instance when they inject it.

#### Conflicting aliases

When Angular merges duplicate host directive matches it also merges their input and output mappings.

If two instances of the same host directive expose the **same input or output under different aliases**, Angular throws an error at compile time ([NG8024](errors/NG8024)).

```ts
@Directive({
  selector: '[popoverTrigger]',
  hostDirectives: [{directive: TriggerRef, inputs: ['triggerId: popoverTriggerId']}],
})
export class PopoverTrigger {}

@Directive({
  selector: '[dropdownTrigger]',
  hostDirectives: [
    {directive: TriggerRef, inputs: ['triggerId: dropdownTriggerId']}, // different alias!
  ],
})
export class DropdownTrigger {}
```

```angular-html
```

To resolve this, ensure that both paths expose the shared input or output under the same alias, or do not expose it at all.

## What's next


---

### Optimizing images
Source: https://angular.dev/guide/image-optimization

# Getting started with NgOptimizedImage

The `NgOptimizedImage` directive makes it easy to adopt performance best practices for loading images.

The directive ensures that the loading of the [Largest Contentful Paint (LCP)](https://web.dev/lcp) image is prioritized by:

- Automatically setting the `fetchpriority` attribute on the `<img>` tag
- Lazy loading other images by default
- Automatically generating a preconnect link tag in the document head
- Automatically generating a `srcset` attribute
- Generating a [preload hint](https://developer.mozilla.org/docs/Web/HTML/Link_types/preload) if app is using SSR

In addition to optimizing the loading of the LCP image, `NgOptimizedImage` enforces a number of image best practices, such as:

- Using [image CDN URLs to apply image optimizations](https://web.dev/image-cdns/#how-image-cdns-use-urls-to-indicate-optimization-options)
- Preventing layout shift by requiring `width` and `height`
- Warning if `width` or `height` have been set incorrectly
- Warning if the image will be visually distorted when rendered

If you're using a background image in CSS, [start here](#how-to-migrate-your-background-image).

NOTE: Although the `NgOptimizedImage` directive was made a stable feature in Angular version 15, it has been backported and is available as a stable feature in versions 13.4.0 and 14.3.0 as well.

## Getting Started

Import `NgOptimizedImage` directive from `@angular/common`:

```ts
import {NgOptimizedImage} from '@angular/common';
```

and include it into the `imports` array of a standalone component or an NgModule:

```ts
imports: [
  NgOptimizedImage,
  // ...
],
```

An image loader is not **required** in order to use NgOptimizedImage, but using one with an image CDN enables powerful performance features, including automatic `srcset`s for your images.

A brief guide for setting up a loader can be found in the [Configuring an Image Loader](#configuring-an-image-loader-for-ngoptimizedimage) section at the end of this page.
To activate the `NgOptimizedImage` directive, replace your image's `src` attribute with `ngSrc`.

```html
```

If you're using a [built-in third-party loader](#built-in-loaders), make sure to omit the base URL path from `src`, as that will be prepended automatically by the loader.
Always mark the [LCP image](https://web.dev/lcp/#what-elements-are-considered) on your page as `priority` to prioritize its loading.

```html
```

Marking an image as `priority` applies the following optimizations:

- Sets `fetchpriority=high` (read more about priority hints [here](https://web.dev/priority-hints))
- Sets `loading=eager` (read more about native lazy loading [here](https://web.dev/browser-level-image-lazy-loading))
- Automatically generates a [preload link element](https://developer.mozilla.org/docs/Web/HTML/Link_types/preload) if [rendering on the server](guide/ssr).

Angular displays a warning during development if the LCP element is an image that does not have the `priority` attribute. A page’s LCP element can vary based on a number of factors - such as the dimensions of a user's screen, so a page may have multiple images that should be marked `priority`. See [CSS for Web Vitals](https://web.dev/css-web-vitals/#images-and-largest-contentful-paint-lcp) for more details.
In order to prevent [image-related layout shifts](https://web.dev/css-web-vitals/#images-and-layout-shifts), NgOptimizedImage requires that you specify a height and width for your image, as follows:

```html
```

For **responsive images** (images which you've styled to grow and shrink relative to the viewport), the `width` and `height` attributes should be the intrinsic size of the image file. For responsive images it's also important to [set a value for `sizes`.](#responsive-images)

For **fixed size images**, the `width` and `height` attributes should reflect the desired rendered size of the image. The aspect ratio of these attributes should always match the intrinsic aspect ratio of the image.

NOTE: If you don't know the size of your images, consider using "fill mode" to inherit the size of the parent container, as described below.

## Using `fill` mode

In cases where you want to have an image fill a containing element, you can use the `fill` attribute. This is often useful when you want to achieve a "background image" behavior. It can also be helpful when you don't know the exact width and height of your image, but you do have a parent container with a known size that you'd like to fit your image into (see "object-fit" below).

When you add the `fill` attribute to your image, you do not need and should not include a `width` and `height`, as in this example:

```html
```

You can use the [object-fit](https://developer.mozilla.org/docs/Web/CSS/object-fit) CSS property to change how the image will fill its container. If you style your image with `object-fit: "contain"`, the image will maintain its aspect ratio and be "letterboxed" to fit the element. If you set `object-fit: "cover"`, the element will retain its aspect ratio, fully fill the element, and some content may be "cropped" off.

See visual examples of the above at the [MDN object-fit documentation.](https://developer.mozilla.org/docs/Web/CSS/object-fit)

You can also style your image with the [object-position property](https://developer.mozilla.org/docs/Web/CSS/object-position) to adjust its position within its containing element.

IMPORTANT: For the "fill" image to render properly, its parent element **must** be styled with `position: "relative"`, `position: "fixed"`, or `position: "absolute"`.

## How to migrate your background image

Here's a simple step-by-step process for migrating from `background-image` to `NgOptimizedImage`. For these steps, we'll refer to the element that has an image background as the "containing element":

1. Remove the `background-image` style from the containing element.
2. Ensure that the containing element has `position: "relative"`, `position: "fixed"`, or `position: "absolute"`.
3. Create a new image element as a child of the containing element, using `ngSrc` to enable the `NgOptimizedImage` directive.
4. Give that element the `fill` attribute. Do not include a `height` and `width`.
5. If you believe this image might be your [LCP element](https://web.dev/lcp/), add the `priority` attribute to the image element.

You can adjust how the background image fills the container as described in the [Using fill mode](#using-fill-mode) section.

## Using placeholders

### Automatic placeholders

NgOptimizedImage can display an automatic low-resolution placeholder for your image if you're using a CDN or image host that provides automatic image resizing. Take advantage of this feature by adding the `placeholder` attribute to your image:

```html
```

Adding this attribute automatically requests a second, smaller version of the image using your specified image loader. This small image will be applied as a `background-image` style with a CSS blur while your image loads. If no image loader is provided, no placeholder image can be generated and an error will be thrown.

The default size for generated placeholders is 30px wide. You can change this size by specifying a pixel value in the `IMAGE_CONFIG` provider, as seen below:

```ts
providers: [
  {
    provide: IMAGE_CONFIG,
    useValue: {
      placeholderResolution: 40
    }
  },
],
```

If you want sharp edges around your blurred placeholder, you can wrap your image in a containing `<div>` with the `overflow: hidden` style. As long as the `<div>` is the same size as the image (such as by using the `width: fit-content` style), the "fuzzy edges" of the placeholder will be hidden.

### Data URL placeholders

You can also specify a placeholder using a base64 [data URL](https://developer.mozilla.org/docs/Web/HTTP/Basics_of_HTTP/Data_URLs) without an image loader. The data url format is `data:image/[imagetype];[data]`, where `[imagetype]` is the image format, just as `png`, and `[data]` is a base64 encoding of the image. That encoding can be done using the command line or in JavaScript. For specific commands, see [the MDN documentation](https://developer.mozilla.org/docs/Web/HTTP/Basics_of_HTTP/Data_URLs#encoding_data_into_base64_format). An example of a data URL placeholder with truncated data is shown below:

```html
```

However, large data URLs increase the size of your Angular bundles and slow down page load. If you cannot use an image loader, the Angular team recommends keeping base64 placeholder images smaller than 4KB and using them exclusively on critical images. In addition to decreasing placeholder dimensions, consider changing image formats or parameters used when saving images. At very low resolutions, these parameters can have a large effect on file size.

### Non-blurred placeholders

By default, NgOptimizedImage applies a CSS blur effect to image placeholders. To render a placeholder without blur, provide a `placeholderConfig` argument with an object that includes the `blur` property, set to false. For example:

```html
```

## Adjusting image styling

Depending on the image's styling, adding `width` and `height` attributes may cause the image to render differently. `NgOptimizedImage` warns you if your image styling renders the image at a distorted aspect ratio.

You can typically fix this by adding `height: auto` or `width: auto` to your image styles. For more information, see the [web.dev article on the `<img>` tag](https://web.dev/patterns/web-vitals-patterns/images/img-tag).

If the `width` and `height` attribute on the image are preventing you from sizing the image the way you want with CSS, consider using `fill` mode instead, and styling the image's parent element.

## Performance Features

NgOptimizedImage includes a number of features designed to improve loading performance in your app. These features are described in this section.

### Add resource hints

A [`preconnect` resource hint](https://web.dev/preconnect-and-dns-prefetch) for your image origin ensures that the LCP image loads as quickly as possible.

Preconnect links are automatically generated for domains provided as an argument to a [loader](#optional-set-up-a-loader). If an image origin cannot be automatically identified, and no preconnect link is detected for the LCP image, `NgOptimizedImage` will warn during development. In that case, you should manually add a resource hint to `index.html`. Within the `<head>` of the document, add a `link` tag with `rel="preconnect"`, as shown below:

```html
```

To disable preconnect warnings, inject the `PRECONNECT_CHECK_BLOCKLIST` token:

```ts

providers: [
{provide: PRECONNECT_CHECK_BLOCKLIST, useValue: 'https://your-domain.com'}
],

```

See more information on automatic preconnect generation [here](#why-is-a-preconnect-element-not-being-generated-for-my-image-domain).

### Request images at the correct size with automatic `srcset`

Defining a [`srcset` attribute](https://developer.mozilla.org/docs/Web/API/HTMLImageElement/srcset) ensures that the browser requests an image at the right size for your user's viewport, so it doesn't waste time downloading an image that's too large. `NgOptimizedImage` generates an appropriate `srcset` for the image, based on the presence and value of the [`sizes` attribute](https://developer.mozilla.org/docs/Web/API/HTMLImageElement/sizes) on the image tag.

#### Fixed-size images

If your image should be "fixed" in size (i.e. the same size across devices, except for [pixel density](https://web.dev/codelab-density-descriptors/)), there is no need to set a `sizes` attribute. A `srcset` can be generated automatically from the image's width and height attributes with no further input required.

Example srcset generated:

```html
```

#### Responsive images

If your image should be responsive (i.e. grow and shrink according to viewport size), then you will need to define a [`sizes` attribute](https://developer.mozilla.org/docs/Web/API/HTMLImageElement/sizes) to generate the `srcset`.

If you haven't used `sizes` before, a good place to start is to set it based on viewport width. For example, if your CSS causes the image to fill 100% of viewport width, set `sizes` to `100vw` and the browser will select the image in the `srcset` that is closest to the viewport width (after accounting for pixel density). If your image is only likely to take up half the screen (ex: in a sidebar), set `sizes` to `50vw` to ensure the browser selects a smaller image. And so on.

If you find that the above does not cover your desired image behavior, see the documentation on [advanced sizes values](#advanced-sizes-values).

Note that for lazy-loaded images, `NgOptimizedImage` automatically prepends `"auto"` to the provided `sizes` value. This is an optimization that increases the accuracy of srcset selection on browsers which support `sizes="auto"`, and is ignored by browsers which do not. Images that are not lazy-loaded, such as `priority` images, keep the `sizes` value as provided.

By default, the responsive breakpoints are:

`[16, 32, 48, 64, 96, 128, 256, 384, 640, 750, 828, 1080, 1200, 1920, 2048, 3840]`

If you would like to customize these breakpoints, you can do so using the `IMAGE_CONFIG` provider:

```ts
providers: [
  {
    provide: IMAGE_CONFIG,
    useValue: {
      breakpoints: [16, 48, 96, 128, 384, 640, 750, 828, 1080, 1200, 1920]
    }
  },
],
```

If you would like to manually define a `srcset` attribute, you can provide your own using the `ngSrcset` attribute:

```html
```

If the `ngSrcset` attribute is present, `NgOptimizedImage` generates and sets the `srcset` based on the sizes included. Do not include image file names in `ngSrcset` - the directive infers this information from `ngSrc`. The directive supports both width descriptors (e.g. `100w`) and density descriptors (e.g. `1x`).

```html
```

### Disabling automatic srcset generation

To disable srcset generation for a single image, you can add the `disableOptimizedSrcset` attribute on the image:

```html
```

### Disabling image lazy loading

By default, `NgOptimizedImage` sets `loading=lazy` for all images that are not marked `priority`. You can disable this behavior for non-priority images by setting the `loading` attribute. This attribute accepts values: `eager`, `auto`, and `lazy`. [See the documentation for the standard image `loading` attribute for details](https://developer.mozilla.org/docs/Web/API/HTMLImageElement/loading#value).

```html
```

### Controlling image decoding

By default, `NgOptimizedImage` sets `decoding="auto"` for all images. This allows the browser to decide the optimal time to decode an image after it has been fetched. When an image is marked as `priority`, Angular automatically sets `decoding="sync"` to ensure the image is decoded and painted as early as possible helping improve **Largest Contentful Paint (LCP)** performance.

You can override the default `auto` for non-priority images by explicitly setting the `decoding` attribute. Priority images always use `decoding="sync"`.  
[See the documentation for the standard image `decoding` attribute for details](https://developer.mozilla.org/docs/Web/HTML/Element/img#decoding).

```html



```

**Allowed values**

- `auto` (default): lets the browser choose the optimal strategy.
- `async`: decodes the image asynchronously, avoiding main‑thread blocking where possible.
- `sync`: decodes the image immediately; can block rendering but ensures pixels are ready as soon as the image is available.

### Advanced 'sizes' values

You may want to have images displayed at varying widths on differently-sized screens. A common example of this pattern is a grid- or column-based layout that renders a single column on mobile devices, and two columns on larger devices. You can capture this behavior in the `sizes` attribute, using a "media query" syntax, such as the following:

```html
```

The `sizes` attribute in the above example says "I expect this image to be 100 percent of the screen width on devices under 768px wide. Otherwise, I expect it to be 50 percent of the screen width.

For additional information about the `sizes` attribute, see [web.dev](https://web.dev/learn/design/responsive-images/#sizes) or [mdn](https://developer.mozilla.org/docs/Web/API/HTMLImageElement/sizes).

## Configuring an image loader for `NgOptimizedImage`

A "loader" is a function that generates an [image transformation URL](https://web.dev/image-cdns/#how-image-cdns-use-urls-to-indicate-optimization-options) for a given image file. When appropriate, `NgOptimizedImage` sets the size, format, and image quality transformations for an image.

`NgOptimizedImage` provides both a generic loader that applies no transformations, as well as loaders for various third-party image services. It also supports writing your own custom loader.

| Loader type                            | Behavior                                                                                                                                                                                                                       |
| :------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Generic loader                         | The URL returned by the generic loader will always match the value of `src`. In other words, this loader applies no transformations. Sites that use Angular to serve images are the primary intended use case for this loader. |
| Loaders for third-party image services | The URL returned by the loaders for third-party image services will follow API conventions used by that particular image service.                                                                                              |
| Custom loaders                         | A custom loader's behavior is defined by its developer. You should use a custom loader if your image service isn't supported by the loaders that come preconfigured with `NgOptimizedImage`.                                   |

Based on the image services commonly used with Angular applications, `NgOptimizedImage` provides loaders preconfigured to work with the following image services:

| Image Service             | Angular API               | Documentation                                                               |
| :------------------------ | :------------------------ | :-------------------------------------------------------------------------- |
| Cloudflare Image Resizing | `provideCloudflareLoader` | [Documentation](https://developers.cloudflare.com/images/image-resizing/)   |
| Cloudinary                | `provideCloudinaryLoader` | [Documentation](https://cloudinary.com/documentation/resizing_and_cropping) |
| ImageKit                  | `provideImageKitLoader`   | [Documentation](https://docs.imagekit.io/)                                  |
| Imgix                     | `provideImgixLoader`      | [Documentation](https://docs.imgix.com/)                                    |
| Netlify                   | `provideNetlifyLoader`    | [Documentation](https://docs.netlify.com/image-cdn/overview/)               |

To use the **generic loader** no additional code changes are necessary. This is the default behavior.

### Built-in Loaders

To use an existing loader for a **third-party image service**, add the provider factory for your chosen service to the `providers` array. In the example below, the Imgix loader is used:

```ts
providers: [
  provideImgixLoader('https://my.base.url/'),
],
```

The base URL for your image assets should be passed to the provider factory as an argument. For most sites, this base URL should match one of the following patterns:

- <https://yoursite.yourcdn.com>
- <https://subdomain.yoursite.com>
- <https://subdomain.yourcdn.com/yoursite>

You can learn more about the base URL structure in the docs of a corresponding CDN provider.

### Custom Loaders

To use a **custom loader**, provide your loader function as a value for the `IMAGE_LOADER` DI token. In the example below, the custom loader function returns a URL starting with `https://example.com` that includes `src`, `width`, and `height` as URL parameters.

```ts
providers: [
  {
    provide: IMAGE_LOADER,
    useValue: (config: ImageLoaderConfig) => {
      return `https://example.com/images?src=${config.src}&width=${config.width}&height=${config.height}`;
    },
  },
],
```

A loader function for the `NgOptimizedImage` directive takes an object with the `ImageLoaderConfig` type (from `@angular/common`) as its argument and returns the absolute URL of the image asset. The `ImageLoaderConfig` object contains the `src` property, and optional `width`, `height`, and `loaderParams` properties.

NOTE: even though the `width` property may not always be present, a custom loader must use it to support requesting images at various widths in order for `ngSrcset` to work properly.

### The `loaderParams` Property

There is an additional attribute supported by the `NgOptimizedImage` directive, called `loaderParams`, which is specifically designed to support the use of custom loaders. The `loaderParams` attribute takes an object with any properties as a value, and does not do anything on its own. The data in `loaderParams` is added to the `ImageLoaderConfig` object passed to your custom loader, and can be used to control the behavior of the loader.

A common use for `loaderParams` is controlling advanced image CDN features.

### Using the `transform` property with built-in loaders

The built-in loaders for Cloudinary, Cloudflare, ImageKit, and Imgix support a special `transform` property within `loaderParams`. This property allows you to apply custom image transformations provided by your CDN.

The `transform` property accepts two formats:

#### String format

Provide transformations as a comma-separated string using your CDN's transformation syntax:

```html
<img
  ngSrc="my-image.jpg"
  width="400"
  height="300"
  [loaderParams]="{transform: 'e_grayscale,r_10'}"
/>
```

#### Object format

Provide transformations as an object with key-value pairs.

```html
<img
  ngSrc="my-image.jpg"
  width="400"
  height="300"
  [loaderParams]="{transform: {e: 'grayscale', r: 10}}"
/>
```

NOTE: The `transform` property is not supported by the Netlify loader, as Netlify's image CDN does not provide custom transformation parameters.

### Example custom loader

The following shows an example of a custom loader function. This example function concatenates `src`, `width`, and `height`, and uses `loaderParams` to control a custom CDN feature for rounded corners:

```ts
const myCustomLoader = (config: ImageLoaderConfig) => {
  let url = `https://example.com/images/${config.src}?`;
  let queryParams = [];
  if (config.width) {
    queryParams.push(`w=${config.width}`);
  }
  if (config.height) {
    queryParams.push(`h=${config.height}`);
  }
  if (config.loaderParams?.roundedCorners) {
    queryParams.push('mask=corners&corner-radius=5');
  }
  return url + queryParams.join('&');
};
```

Note that in the above example, we've invented the 'roundedCorners' property name to control a feature of our custom loader. We could then use this feature when creating an image, as follows:

```html
```

## Frequently Asked Questions

### Does NgOptimizedImage support the `background-image` css property?

The NgOptimizedImage does not directly support the `background-image` css property, but it is designed to easily accommodate the use case of having an image as the background of another element.

For a step-by-step process for migration from `background-image` to `NgOptimizedImage`, see the [How to migrate your background image](#how-to-migrate-your-background-image) section above.

### Why can't I use `src` with `NgOptimizedImage`?

The `ngSrc` attribute was chosen as the trigger for NgOptimizedImage due to technical considerations around how images are loaded by the browser. NgOptimizedImage makes programmatic changes to the `loading` attribute -- if the browser sees the `src` attribute before those changes are made, it will begin eagerly downloading the image file, and the loading changes will be ignored.

### Why is a preconnect element not being generated for my image domain?

Preconnect generation is performed based on static analysis of your application. That means that the image domain must be directly included in the loader parameter, as in the following example:

```ts
providers: [
  provideImgixLoader('https://my.base.url/'),
],
```

If you use a variable to pass the domain string to the loader, or you're not using a loader, the static analysis will not be able to identify the domain, and no preconnect link will be generated. In this case you should manually add a preconnect link to the document head, as [described above](#add-resource-hints).

### Can I use two different image domains in the same page?

The [image loaders](#configuring-an-image-loader-for-ngoptimizedimage) provider pattern is designed to be as simple as possible for the common use case of having only a single image CDN used within a component. However, it's still very possible to manage multiple image CDNs using a single provider.

To do this, we recommend writing a [custom image loader](#custom-loaders) which uses the [`loaderParams` property](#the-loaderparams-property) to pass a flag that specifies which image CDN should be used, and then invokes the appropriate loader based on that flag.

### Can you add a new built-in loader for my preferred CDN?

For maintenance reasons, we don't currently plan to support additional built-in loaders in the Angular repository. Instead, we encourage developers to publish any additional image loaders as third-party packages.

### Can I use this with the `<picture>` tag

No, but this is on our roadmap, so stay tuned.

If you're waiting on this feature, please upvote the GitHub issue [here](https://github.com/angular/angular/issues/56594).

### How do I find my LCP image with Chrome DevTools?

1. Using the performance tab of the Chrome DevTools, click on the "start profiling and reload page" button on the top left. It looks like a page refresh icon.

2. This will trigger a profiling snapshot of your Angular application.

3. Once the profiling result is available, select "LCP" in the timings section.

4. A summary entry should appear in the panel at the bottom. You can find the LCP element in the row for "related node". Clicking on it will reveal the element in the Elements panel.


NOTE: This only identifies the LCP element within the viewport of the page you are testing. It is also recommended to use mobile emulation to identify the LCP element for smaller screens.

---


## Signals

### Signals overview
Source: https://angular.dev/guide/signals

Angular Signals is a system that granularly tracks how and where your state is used throughout an application, allowing the framework to optimize rendering updates.

TIP: Check out Angular's [Essentials](essentials/signals) before diving into this comprehensive guide.

## What are signals?

A **signal** is a wrapper around a value that notifies interested consumers when that value changes. Signals can contain any value, from primitives to complex data structures.

You read a signal's value by calling its getter function, which allows Angular to track where the signal is used.

Signals may be either _writable_ or _read-only_.

### Writable signals

Writable signals provide an API for updating their values directly. You create writable signals by calling the `signal` function with the signal's initial value:

```ts
const count = signal(0);

// Signals are getter functions - calling them reads their value.
console.log('The count is: ' + count());
```

To change the value of a writable signal, either `.set()` it directly:

```ts
count.set(3);
```

or use the `.update()` operation to compute a new value from the previous one:

```ts
// Increment the count by 1.
count.update((value) => value + 1);
```

Writable signals have the type `WritableSignal`.

#### Converting writable signals to readonly

`WritableSignal` provides an `asReadonly()` method that returns a readonly version of the signal. This is useful when you want to expose a signal's value to consumers without allowing them to modify it directly:

```ts
@Service()
export class CounterState {
  // Private writable state
  private readonly _count = signal(0);

  readonly count = this._count.asReadonly(); // public readonly

  increment() {
    this._count.update((v) => v + 1);
  }
}

@Component({/* ... */})
export class AwesomeCounter {
  state = inject(CounterState);

  count = this.state.count; // can read but not modify

  increment() {
    this.state.increment();
  }
}
```

The readonly signal reflects any changes made to the original writable signal, but cannot be modified using `set()` or `update()` methods.

IMPORTANT: The readonly signals do **not** have any built-in mechanism that would prevent deep-mutation of their value.

### Computed signals

**Computed signals** are read-only signals that derive their value from other signals. You define computed signals using the `computed` function and specifying a derivation:

```typescript
const count: WritableSignal<number> = signal(0);
const doubleCount: Signal<number> = computed(() => count() * 2);
```

The `doubleCount` signal depends on the `count` signal. Whenever `count` updates, Angular knows that `doubleCount` needs to update as well.

#### Computed signals are both lazily evaluated and memoized

`doubleCount`'s derivation function does not run to calculate its value until the first time you read `doubleCount`. The calculated value is then cached, and if you read `doubleCount` again, it will return the cached value without recalculating.

If you then change `count`, Angular knows that `doubleCount`'s cached value is no longer valid, and the next time you read `doubleCount` its new value will be calculated.

As a result, you can safely perform computationally expensive derivations in computed signals, such as filtering arrays.

#### Computed signals are not writable signals

You cannot directly assign values to a computed signal. That is,

```ts
doubleCount.set(3);
```

produces a compilation error, because `doubleCount` is not a `WritableSignal`.

#### Computed signal dependencies are dynamic

Only the signals actually read during the derivation are tracked. For example, in this `computed` the `count` signal is only read if the `showCount` signal is true:

```ts
const showCount = signal(false);
const count = signal(0);
const conditionalCount = computed(() => {
  if (showCount()) {
    return `The count is ${count()}.`;
  } else {
    return 'Nothing to see here!';
  }
});
```

When you read `conditionalCount`, if `showCount` is `false` the "Nothing to see here!" message is returned _without_ reading the `count` signal. This means that if you later update `count` it will _not_ result in a recomputation of `conditionalCount`.

If you set `showCount` to `true` and then read `conditionalCount` again, the derivation will re-execute and take the branch where `showCount` is `true`, returning the message which shows the value of `count`. Changing `count` will then invalidate `conditionalCount`'s cached value.

Note that dependencies can be removed during a derivation as well as added. If you later set `showCount` back to `false`, then `count` will no longer be considered a dependency of `conditionalCount`.

## Reactive contexts

A **reactive context** is a runtime state where Angular monitors signal reads to establish a dependency. The code reading the signal is the _consumer_, and the signal being read is the _producer_.

Angular automatically enters a reactive context when:

- Executing an `effect`, `afterRenderEffect` callback.
- Evaluating a `computed` signal.
- Evaluating a `linkedSignal`.
- Evaluating a `resource`'s params function.
- Rendering a component template (including bindings in the [host property](guide/components/host-elements#binding-to-the-host-element)).

During these operations, Angular creates a _live_ connection. If a tracked signal changes, Angular will _eventually_ re-run the consumer.

### Asserts the reactive context

Angular provides the `assertNotInReactiveContext` helper function to assert that code is not executing within a reactive context. Pass a reference to the calling function so the error message points to the correct API entry point if the assertion fails. This produces a clearer, more actionable error message than a generic reactive context error.

```ts
import {assertNotInReactiveContext} from '@angular/core';

function subscribeToEvents() {
  assertNotInReactiveContext(subscribeToEvents);
  // Safe to proceed - subscription logic here
}
```

### Reading without tracking dependencies

Rarely, you may want to execute code which may read signals within a reactive function such as `computed` or `effect` _without_ creating a dependency.

For example, suppose that when `currentUser` changes, the value of a `counter` should be logged. You could create an `effect` which reads both signals:

```ts
effect(() => {
  console.log(`User set to ${currentUser()} and the counter is ${counter()}`);
});
```

This example will log a message when _either_ `currentUser` or `counter` changes. However, if the effect should only run when `currentUser` changes, then the read of `counter` is only incidental and changes to `counter` shouldn't log a new message.

You can prevent a signal read from being tracked by calling its getter with `untracked`:

```ts
effect(() => {
  console.log(`User set to ${currentUser()} and the counter is ${untracked(counter)}`);
});
```

`untracked` is also useful when an effect needs to invoke some external code which shouldn't be treated as a dependency:

```ts
effect(() => {
  const user = currentUser();
  untracked(() => {
    // If the `loggingService` reads signals, they won't be counted as
    // dependencies of this effect.
    this.loggingService.log(`User set to ${user}`);
  });
});
```

### Reactive context and async operations

The reactive context is only active for synchronous code. Any signal reads that occur after an asynchronous boundary will not be tracked as dependencies.

```ts {avoid}
effect(async () => {
  const data = await fetchUserData();
  // Reactive context is lost here - theme() won't be tracked
  console.log(`User: ${data.name}, Theme: ${theme()}`);
});
```

To ensure all signal reads are tracked, read signals before the `await`. This includes passing them as arguments to the awaited function, since arguments are evaluated synchronously:

```ts {prefer}
effect(async () => {
  const currentTheme = theme(); // Read before await
  const data = await fetchUserData();
  console.log(`User: ${data.name}, Theme: ${currentTheme}`);
});
```

```ts {prefer}
effect(async () => {
  // Also works: signal is read before await (as function argument)
  await renderContent(docContent());
});
```

## Advanced derivations

While `computed` handles simple readonly derivations, you might find yourself needing a writable state that is dependent on other signals.
For more information see the [Dependent state with linkedSignal](/guide/signals/linked-signal) guide.

All signal APIs are synchronous— `signal`, `computed`, `input`, etc. However, applications often need to deal with data that is available asynchronously. A `Resource` gives you a way to incorporate async data into your application's signal-based code and still allow you to access its data synchronously. For more information see the [Async reactivity with resources](/guide/signals/resource) guide.

## Executing side effects on non-reactive APIs

Synchronous or asynchronous derivations are recommended when we want to react to state changes. However, this doesn't cover all the possible use cases, and you'll sometimes find yourself in a situation where you need to react to signal changes on non-reactive APIs. Use `effect` or `afterRenderEffect` for those specific use cases. For more information see [Side effects for non-reactive APIs](/guide/signals/effect) guide.

## Reading signals in `OnPush` components

When you read a signal within an `OnPush` component's template, Angular tracks the signal as a dependency of that component. When the value of that signal changes, Angular automatically [marks](api/core/ChangeDetectorRef#markForCheck) the component to ensure it gets updated the next time change detection runs. Refer to the [Skipping component subtrees](best-practices/skipping-subtrees) guide for more information about `OnPush` components.

## Advanced topics

### Signal equality functions

When creating a signal, you can optionally provide an equality function, which will be used to check whether the new value is actually different than the previous one.

```ts
import {signal} from '@angular/core';
import isEqual from 'lodash/isEqual';

const data = signal(['test'], {equal: isEqual});

// Even though this is a different array instance, the deep equality
// function will consider the values to be equal, and the signal won't
// trigger any updates.
data.set(['test']);
```

Equality functions can be provided to both writable and computed signals.

HELPFUL: By default, signals use referential equality ([`Object.is()`](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Object/is) comparison).

### Type checking signals

You can use `isSignal` to check if a value is a `Signal`:

```ts
const count = signal(0);
const doubled = computed(() => count() * 2);

isSignal(count); // true
isSignal(doubled); // true
isSignal(42); // false
```

To specifically check if a signal is writable, use `isWritableSignal`:

```ts
const count = signal(0);
const doubled = computed(() => count() * 2);

isWritableSignal(count); // true
isWritableSignal(doubled); // false
```

## Using signals with RxJS

See [RxJS interop with Angular signals](ecosystem/rxjs-interop) for details on interoperability between signals and RxJS.

## Debugging signals

Angular DevTools can visualize the signal graph of a component or directive, showing how its signals, computeds, effects, and other reactive nodes depend on each other.
See [Debug the signal graph](tools/devtools/signals) to learn how to open, read, and inspect the graph.

---

### Dependent state with linkedSignal
Source: https://angular.dev/guide/signals/linked-signal

# Dependent state with `linkedSignal`

You can use the `signal` function to hold some state in your Angular code. Sometimes, this state depends on some _other_ state. For example, imagine a component that lets the user select a shipping method for an order:

```typescript
@Component({
  /* ... */
})
export class ShippingMethodPicker {
  shippingOptions: Signal<ShippingMethod[]> = getShippingOptions();

  // Select the first shipping option by default.
  selectedOption = signal(this.shippingOptions()[0]);

  changeShipping(newOptionIndex: number) {
    this.selectedOption.set(this.shippingOptions()[newOptionIndex]);
  }
}
```

In this example, the `selectedOption` defaults to the first option, but changes if the user selects another option. But `shippingOptions` is a signal— its value may change! If `shippingOptions` changes, `selectedOption` may contain a value that is no longer a valid option.

**The `linkedSignal` function lets you create a signal to hold some state that is intrinsically _linked_ to some other state.** Revisiting the example above, `linkedSignal` can replace `signal`:

```ts
@Component({
  /* ... */
})
export class ShippingMethodPicker {
  shippingOptions: Signal<ShippingMethod[]> = getShippingOptions();

  // Initialize selectedOption to the first shipping option.
  selectedOption = linkedSignal(() => this.shippingOptions()[0]);

  changeShipping(index: number) {
    this.selectedOption.set(this.shippingOptions()[index]);
  }
}
```

`linkedSignal` works similarly to `signal` with one key difference— instead of passing a default value, you pass a _computation function_, just like `computed`. When the value of the computation changes, the value of the `linkedSignal` changes to the computation result. This helps ensure that the `linkedSignal` always has a valid value.

The following example shows how the value of a `linkedSignal` can change based on its linked state:

```ts
const shippingOptions = signal(['Ground', 'Air', 'Sea']);
const selectedOption = linkedSignal(() => shippingOptions()[0]);
console.log(selectedOption()); // 'Ground'

selectedOption.set(shippingOptions()[2]);
console.log(selectedOption()); // 'Sea'

shippingOptions.set(['Email', 'Will Call', 'Postal service']);
console.log(selectedOption()); // 'Email'
```

## Accounting for previous state

In some cases, the computation for a `linkedSignal` needs to account for the previous value of the `linkedSignal`.

In the example above, `selectedOption` always updates back to the first option when `shippingOptions` changes. You may, however, want to preserve the user's selection if their selected option is still somewhere in the list. To accomplish this, you can create a `linkedSignal` with a separate _source_ and _computation_:

```ts
interface ShippingMethod {
  id: number;
  name: string;
}

@Component({
  /* ... */
})
export class ShippingMethodPicker {
  constructor() {
    this.changeShipping(2);
    this.changeShippingOptions();
    console.log(this.selectedOption()); // {"id":2,"name":"Postal Service"}
  }

  shippingOptions = signal<ShippingMethod[]>([
    {id: 0, name: 'Ground'},
    {id: 1, name: 'Air'},
    {id: 2, name: 'Sea'},
  ]);

  selectedOption = linkedSignal<ShippingMethod[], ShippingMethod>({
    // `selectedOption` is set to the `computation` result whenever this `source` changes.
    source: this.shippingOptions,
    computation: (newOptions, previous) => {
      // If the newOptions contain the previously selected option, preserve that selection.
      // Otherwise, default to the first option.
      return newOptions.find((opt) => opt.id === previous?.value.id) ?? newOptions[0];
    },
  });

  changeShipping(index: number) {
    this.selectedOption.set(this.shippingOptions()[index]);
  }

  changeShippingOptions() {
    this.shippingOptions.set([
      {id: 0, name: 'Email'},
      {id: 1, name: 'Sea'},
      {id: 2, name: 'Postal Service'},
    ]);
  }
}
```

When you create a `linkedSignal`, you can pass an object with separate `source` and `computation` properties instead of providing just a computation.

The `source` can be any signal, such as a `computed` or component `input`. The `linkedSignal` updates its value when the `source` changes or when any signal referenced in the `computation` changes, updating its value with the result of the provided `computation`.

The `computation` is a function that receives the new value of `source` and a `previous` object. The `previous` object has two properties— `previous.source` is the previous value of `source`, and `previous.value` is the previous value of the `linkedSignal`. You can use these previous values to decide the new result of the computation.

HELPFUL: When using the `previous` parameter, it is necessary to provide the generic type arguments of `linkedSignal` explicitly. The first generic type corresponds with the type of `source` and the second generic type determines the output type of `computation`.

## Custom equality comparison

`linkedSignal`, as any other signal, can be configured with a custom equality function. This function is used by downstream dependencies to determine if that value of the `linkedSignal` (result of a computation) changed:

```typescript
const activeUser = signal({id: 123, name: 'Morgan', isAdmin: true});

const activeUserEditCopy = linkedSignal(() => activeUser(), {
  // Consider the user as the same if it's the same `id`.
  equal: (a, b) => a.id === b.id,
});

// Or, if separating `source` and `computation`
const activeUserEditCopy = linkedSignal({
  source: activeUser,
  computation: (user) => user,
  equal: (a, b) => a.id === b.id,
});
```

## Customizing the set operation

Sometimes you may want the `set` and `update` operations of a `linkedSignal` to write back to the source of truth instead of updating the `linkedSignal`'s value directly. You can customize this behavior by passing a `set` function in the options.

The custom `set` function receives two arguments:

1. The new value being set.
2. A `rawSet` function, which you can invoke to update the `linkedSignal`'s internal state directly (matching the default behavior).

NOTE: Using `rawSet` allows you to update the `linkedSignal`'s value directly. This can be useful to prevent the computation from running, for example if it is an expensive derivation and you already know the result.

### Writing back to a source signal

Consider a component that displays and allows editing temperature in Fahrenheit, but uses a Celsius signal as its source of truth:

```typescript
const tempC = signal(0);
const tempF = linkedSignal(() => (tempC() * 9) / 5 + 32, {
  set: (valF) => tempC.set(((valF - 32) * 5) / 9),
});

console.log(tempF()); // 32

// Setting Fahrenheit updates Celsius, which reactively updates Fahrenheit
tempF.set(212);
console.log(tempC()); // 100
console.log(tempF()); // 212
```

### Updating a property inside a parent object

Another common scenario is updating a specific property inside a parent object. The parent is held in a signal, and you link to a nested property:

```typescript
interface Order {
  id: number;
  shippingMethod: string;
}

const order = signal<Order>({
  id: 42,
  shippingMethod: 'Ground',
});

const shippingMethod = linkedSignal(() => order().shippingMethod, {
  set: (newMethod) => {
    // Perform an immutable update to write the change back to the order
    order.update((currentOrder) => ({
      ...currentOrder,
      shippingMethod: newMethod,
    }));
  },
});

console.log(shippingMethod()); // 'Ground'

// Updating the shippingMethod updates the parent order object
shippingMethod.set('Air');
console.log(order()); // { id: 42, shippingMethod: 'Air' }
console.log(shippingMethod()); // 'Air'
```

---

