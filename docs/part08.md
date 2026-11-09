### CDK Installation

The [Component Dev Kit (CDK)](https://material.angular.dev/cdk/categories) is a set of behavior primitives for building components. To use the component harnesses, first install `@angular/cdk` from npm. You can do this from your terminal using the Angular CLI:

```shell
ng add @angular/cdk
```

## Extending `ComponentHarness`

The abstract `ComponentHarness` class is the base class for all component harnesses. To create a custom component harness, extend `ComponentHarness` and implement the static property `hostSelector`.

The `hostSelector` property identifies elements in the DOM that match this harness subclass. In most cases, the `hostSelector` should be the same as the selector of the corresponding `Component` or `Directive`. For example, consider a simple popup component:

```ts
@Component({
  selector: 'my-popup',
  template: `
    @if (isOpen()) {
    }
  `,
})
class MyPopup {
  triggerText = input('');

  isOpen = signal(false);

  toggle() {
    this.isOpen.update((value) => !value);
  }
}
```

In this case, a minimal harness for the component would look like the following:

```ts
class MyPopupHarness extends ComponentHarness {
  static hostSelector = 'my-popup';
}
```

While `ComponentHarness` subclasses require only the `hostSelector` property, most harnesses should also implement a static `with` method to generate `HarnessPredicate` instances. The [filtering harnesses section](guide/testing/using-component-harnesses#filtering-harnesses) covers this in more detail.

## Finding elements in the component's DOM

Each instance of a `ComponentHarness` subclass represents a particular instance of the corresponding component. You can access the component's host element via the `host() `method from the `ComponentHarness` base class.

`ComponentHarness` also offers several methods for locating elements within the component's DOM. These methods are `locatorFor()`, `locatorForOptional()`, and `locatorForAll()`. These methods create functions that find elements, they do not directly find elements. This approach safeguards against caching references to out-of-date elements. For example, when an `@if` block hides and then shows an element, the result is a new DOM element; using functions ensures that tests always reference the current state of the DOM.

See the [ComponentHarness API reference page](/api/cdk/testing/ComponentHarness) for the full list details of the different `locatorFor` methods.

For example, the `MyPopupHarness` example discussed above could provide methods to get the trigger and content elements as follows:

```ts
class MyPopupHarness extends ComponentHarness {
  static hostSelector = 'my-popup';

  // Gets the trigger element
  getTriggerElement = this.locatorFor('button');

  // Gets the content element.
  getContentElement = this.locatorForOptional('.my-popup-content');
}
```

## Working with `TestElement` instances

`TestElement` is an abstraction designed to work across different test environments (Unit tests, WebDriver, etc). When using harnesses, you should perform all DOM interaction via this interface. Other means of accessing DOM elements, such as `document.querySelector()`, do not work in all test environments.

`TestElement` has a number of methods to interact with the underlying DOM, such as `blur()`, `click()`, `getAttribute()`, and more. See the [TestElement API reference page](/api/cdk/testing/TestElement) for the full list of methods.

Do not expose `TestElement` instances to harness users unless it's an element the component consumer defines directly, such as the component's host element. Exposing `TestElement` instances for internal elements leads users to depend on a component's internal DOM structure.

Instead, provide more narrow-focused methods for specific actions the end-user may take or particular state they may observe. For example, `MyPopupHarness` from previous sections could provide methods like `toggle` and `isOpen`:

```ts
class MyPopupHarness extends ComponentHarness {
  static hostSelector = 'my-popup';

  protected getTriggerElement = this.locatorFor('button');
  protected getContentElement = this.locatorForOptional('.my-popup-content');

  /** Toggles the open state of the popup. */
  async toggle() {
    const trigger = await this.getTriggerElement();
    return trigger.click();
  }

  /** Checks if the popup us open. */
  async isOpen() {
    const content = await this.getContentElement();
    return !!content;
  }
}
```

## Loading harnesses for subcomponents

Larger components often compose sub-components. You can reflect this structure in a component's harness as well. Each of the `locatorFor` methods on `ComponentHarness` has an alternate signature that can be used for locating sub-harnesses rather than elements.

See the [ComponentHarness API reference page](/api/cdk/testing/ComponentHarness) for the full list of the different locatorFor methods.

For example, consider a menu build using the popup from above:

```ts
@Directive({
  selector: 'my-menu-item',
})
class MyMenuItem {}

@Component({
  selector: 'my-menu',
  template: `
  `,
})
class MyMenu {
  triggerText = input('');

  items = contentChildren(MyMenuItem);
}
```

The harness for `MyMenu` can then take advantage of other harnesses for `MyPopup` and `MyMenuItem`:

```ts
class MyMenuHarness extends ComponentHarness {
  static hostSelector = 'my-menu';

  protected getPopupHarness = this.locatorFor(MyPopupHarness);

  /** Gets the currently shown menu items (empty list if menu is closed). */
  getItems = this.locatorForAll(MyMenuItemHarness);

  /** Toggles open state of the menu. */
  async toggle() {
    const popupHarness = await this.getPopupHarness();
    return popupHarness.toggle();
  }
}

class MyMenuItemHarness extends ComponentHarness {
  static hostSelector = 'my-menu-item';
}
```

## Filtering harness instances with `HarnessPredicate`

When a page contains multiple instances of a particular component, you may want to filter based on some property of the component to get a particular component instance. For example, you may want a button with some specific text, or a menu with a specific ID. The `HarnessPredicate` class can capture criteria like this for a `ComponentHarness` subclass. While the test author is able to construct `HarnessPredicate` instances manually, it's easier when the `ComponentHarness` subclass provides a helper method to construct predicates for common filters.

You should create a static `with()` method on each `ComponentHarness` subclass that returns a `HarnessPredicate` for that class. This allows test authors to write easily understandable code, e.g. `loader.getHarness(MyMenuHarness.with({selector: '#menu1'}))`. In addition to the standard selector and ancestor options, the `with` method should add any other options that make sense for the particular subclass.

Harnesses that need to add additional options should extend the `BaseHarnessFilters` interface and additional optional properties as needed. `HarnessPredicate` provides several convenience methods for adding options: `stringMatches()`, `addOption()`, and `add()`. See the [HarnessPredicate API page](/api/cdk/testing/HarnessPredicate) for the full description.

For example, when working with a menu it is useful to filter based on trigger text and to filter menu items based on their text:

```ts
interface MyMenuHarnessFilters extends BaseHarnessFilters {
  /** Filters based on the trigger text for the menu. */
  triggerText?: string | RegExp;
}

interface MyMenuItemHarnessFilters extends BaseHarnessFilters {
  /** Filters based on the text of the menu item. */
  text?: string | RegExp;
}

class MyMenuHarness extends ComponentHarness {
  static hostSelector = 'my-menu';

  /** Creates a `HarnessPredicate` used to locate a particular `MyMenuHarness`. */
  static with(options: MyMenuHarnessFilters): HarnessPredicate<MyMenuHarness> {
    return new HarnessPredicate(MyMenuHarness, options).addOption(
      'trigger text',
      options.triggerText,
      (harness, text) => HarnessPredicate.stringMatches(harness.getTriggerText(), text),
    );
  }

  protected getPopupHarness = this.locatorFor(MyPopupHarness);

  /** Gets the text of the menu trigger. */
  async getTriggerText(): Promise<string> {
    const popupHarness = await this.getPopupHarness();
    return popupHarness.getTriggerText();
  }
}

class MyMenuItemHarness extends ComponentHarness {
  static hostSelector = 'my-menu-item';

  /** Creates a `HarnessPredicate` used to locate a particular `MyMenuItemHarness`. */
  static with(options: MyMenuItemHarnessFilters): HarnessPredicate<MyMenuItemHarness> {
    return new HarnessPredicate(MyMenuItemHarness, options).addOption(
      'text',
      options.text,
      (harness, text) => HarnessPredicate.stringMatches(harness.getText(), text),
    );
  }

  /** Gets the text of the menu item. */
  async getText(): Promise<string> {
    const host = await this.host();
    return host.text();
  }
}
```

You can pass a `HarnessPredicate` instead of a `ComponentHarness` class to any of the APIs on `HarnessLoader`, `LocatorFactory`, or `ComponentHarness`. This allows test authors to easily target a particular component instance when creating a harness instance. It also allows the harness author to leverage the same `HarnessPredicate` to enable more powerful APIs on their harness class. For example, consider the `getItems` method on the `MyMenuHarness` shown above. Adding a filtering API allows users of the harness to search for particular menu items:

```ts
class MyMenuHarness extends ComponentHarness {
  static hostSelector = 'my-menu';

  /** Gets a list of items in the menu, optionally filtered based on the given criteria. */
  async getItems(filters: MyMenuItemHarnessFilters = {}): Promise<MyMenuItemHarness[]> {
    const getFilteredItems = this.locatorForAll(MyMenuItemHarness.with(filters));
    return getFilteredItems();
  }
  ...
}
```

## Creating `HarnessLoader` for elements that use content projection

Some components project additional content into the component's template. See the [content projection guide](guide/components/content-projection) for more information.

Add a `HarnessLoader` instance scoped to the element containing the `<ng-content>` when you create a harness for a component that uses content projection. This allows the user of the harness to load additional harnesses for whatever components were passed in as content. `ComponentHarness` has several methods that can be used to create HarnessLoader instances for cases like this: `harnessLoaderFor()`, `harnessLoaderForOptional()`, `harnessLoaderForAll()`. See the [HarnessLoader interface API reference page](/api/cdk/testing/HarnessLoader) for more details.

For example, the `MyPopupHarness` example from above can extend `ContentContainerComponentHarness` to add support to load harnesses within the `<ng-content>` of the component.

```ts
class MyPopupHarness extends ContentContainerComponentHarness<string> {
  static hostSelector = 'my-popup';
}
```

## Accessing elements outside of the component's host element

There are times when a component harness might need to access elements outside of its corresponding component's host element. For example, code that displays a floating element or pop-up often attaches DOM elements directly to the document body, such as the `Overlay` service in Angular CDK.

In this case, `ComponentHarness` provides a method that can be used to get a `LocatorFactory` for the root element of the document. The `LocatorFactory` supports most of the same APIs as the `ComponentHarness` base class, and can then be used to query relative to the document's root element.

Consider if the `MyPopup` component above used the CDK overlay for the popup content, rather than an element in its own template. In this case, `MyPopupHarness` would have to access the content element via `documentRootLocatorFactory()` method that gets a locator factory rooted at the document root.

```ts
class MyPopupHarness extends ComponentHarness {
  static hostSelector = 'my-popup';

  /** Gets a `HarnessLoader` whose root element is the popup's content element. */
  async getHarnessLoaderForContent(): Promise<HarnessLoader> {
    const rootLocator = this.documentRootLocatorFactory();
    return rootLocator.harnessLoaderFor('my-popup-content');
  }
}
```

## Waiting for asynchronous tasks

The methods on `TestElement` automatically trigger Angular's change detection and wait for tasks inside the `NgZone`. In most cases no special effort is required for harness authors to wait on asynchronous tasks. However, there are some edge cases where this may not be sufficient.

Under some circumstances, Angular animations may require a second cycle of change detection and subsequent `NgZone` stabilization before animation events are fully flushed. In cases where this is needed, the `ComponentHarness` offers a `forceStabilize()` method that can be called to do the second round.

You can use `NgZone.runOutsideAngular()` to schedule tasks outside of NgZone. Call the `waitForTasksOutsideAngular()` method on the corresponding harness if you need to explicitly wait for tasks outside `NgZone` since this does not happen automatically.

---


## Animations

### Animations your content
Source: https://angular.dev/guide/animations/css

# Animating your Application with CSS

CSS offers a robust set of tools for you to create beautiful and engaging animations within your application.

## How to write animations in native CSS

If you've never written any native CSS animations, there are a number of excellent guides to get you started. Here are a few of them:
[MDN's CSS Animations guide](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_animations/Using_CSS_animations)
[W3Schools CSS3 Animations guide](https://www.w3schools.com/css/css3_animations.asp)
[The Complete CSS Animations Tutorial](https://www.lambdatest.com/blog/css-animations-tutorial/)
[CSS Animation for Beginners](https://thoughtbot.com/blog/css-animation-for-beginners)

and a couple of videos:
[Learn CSS Animation in 9 Minutes](https://www.youtube.com/watch?v=z2LQYsZhsFw)
[Net Ninja CSS Animation Tutorial Playlist](https://www.youtube.com/watch?v=jgw82b5Y2MU&list=PL4cUxeGkcC9iGYgmEd2dm3zAKzyCGDtM5)

Check out some of these guides and tutorials, then come back to this guide.

## Creating Reusable Animations

You can create reusable animations that can be shared across your application using `@keyframes`. Define keyframe animations in a shared CSS file, and you'll be able to re-use those keyframe animations wherever you want within your application.


Adding the class `animated-class` to an element would trigger the animation on that element.

## Animating a Transition

### Animating State and Styles

You may want to animate between two different states, for example when an element is opened or closed. You can accomplish this by using CSS classes, either with a keyframe animation or transition styling.


Triggering the `open` or `closed` state is done by toggling classes on the element in your component. You can find examples of how to do this in our [template guide](guide/templates/binding#css-class-and-style-property-bindings).

You can see similar examples in the template guide for [animating styles directly](guide/templates/binding#css-style-properties).

### Transitions, Timing, and Easing

Animating often requires adjusting timing, delays, and easing behaviors. This can be done using several css properties or shorthand properties.

Specify `animation-duration`, `animation-delay`, and `animation-timing-function` for a keyframe animation in CSS, or alternatively use the `animation` shorthand property.


Similarly, you can use `transition-duration`, `transition-delay`, and `transition-timing-function` and the `transition` shorthand for animations that are not using `@keyframes`.


### Triggering an Animation

Animations can be triggered by toggling CSS styles or classes. Once a class is present on an element, the animation will occur. Removing the class will revert the element back to whatever CSS is defined for that element. Here's an example:


## Transition and Triggers

### Animating Auto Height

You can use CSS Grid to animate to auto height.


If you don't have to worry about supporting all browsers, you can also check out `calc-size()`, which is the true solution to animating auto height. See [MDN's docs](https://developer.mozilla.org/en-US/docs/Web/CSS/calc-size) and [this tutorial](https://frontendmasters.com/blog/one-of-the-boss-battles-of-css-is-almost-won-transitioning-to-auto/) for more information.

### Animate entering and leaving a view

You can create animations for when an item enters a view or leaves a view. Let's start by looking at how to animate an element entering a view. We'll do this with `animate.enter`, which will apply animation classes when an element enters the view.


Animating an element when it leaves the view is similar to animating when entering a view. Use `animate.leave` to specify which CSS classes to apply when the element leaves the view.


NOTE: Child `animate.leave` animations fire only within the same component template. Nested component `animate.leave` animations will not fire when a parent element is removed.

For more information on `animate.enter` and `animate.leave`, see the [Enter and Leave animations guide](guide/animations).

### Animating increment and decrement

Animating on increment and decrement is a common pattern in applications. Here's an example of how you can accomplish that behavior.


### Disabling an animation or all animations

If you'd like to disable the animations that you've specified, you have multiple options.

1. Create a custom class that forces animation and transition to `none`.

```css
.no-animation {
  animation: none !important;
  transition: none !important;
}
```

Applying this class to an element prevents any animation from firing on that element. You could alternatively scope this to your entire DOM or section of your DOM to enforce this behavior. However, this prevents animation events from firing. If you are awaiting animation events for element removal, this solution won't work. A workaround is to set durations to 1 millisecond instead.

2. Use the [`prefers-reduced-motion`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion) media query to ensure no animations play for users that prefer less animation.

3. Prevent adding animation classes programmatically

### Animation Callbacks

If you have actions you would like to execute at certain points during animations, there are a number of available events you can listen to. Here's a few of them.

[`OnAnimationStart`](https://developer.mozilla.org/en-US/docs/Web/API/Element/animationstart_event)  
[`OnAnimationEnd`](https://developer.mozilla.org/en-US/docs/Web/API/Element/animationend_event)  
[`OnAnimationIteration`](https://developer.mozilla.org/en-US/docs/Web/API/Element/animationiteration_event)  
[`OnAnimationCancel`](https://developer.mozilla.org/en-US/docs/Web/API/Element/animationcancel_event)

[`OnTransitionStart`](https://developer.mozilla.org/en-US/docs/Web/API/Element/transitionstart_event)  
[`OnTransitionRun`](https://developer.mozilla.org/en-US/docs/Web/API/Element/transitionrun_event)  
[`OnTransitionEnd`](https://developer.mozilla.org/en-US/docs/Web/API/Element/transitionend_event)  
[`OnTransitionCancel`](https://developer.mozilla.org/en-US/docs/Web/API/Element/transitioncancel_event)

The Web Animations API has a lot of additional functionality. [Take a look at the documentation](https://developer.mozilla.org/en-US/docs/Web/API/Web_Animations_API) to see all the available animation APIs.

NOTE: Be aware of bubbling issues with these callbacks. If you are animating children and parents, the events bubble up from children to parents. Consider stopping propagation or looking at more details within the event to determine if you're responding to the desired event target rather than an event bubbling up from a child node. You can examine the `animationname` property or the properties being transitioned to verify you have the right nodes.

## Complex Sequences

Animations are often more complicated than just a simple fade in or fade out. You may have lots of complicated sequences of animations you may want to run. Let's take a look at some of those possible scenarios.

### Staggering animations in a list

One common effect is to stagger the animations of each item in a list to create a cascade effect. This can be accomplished by utilizing `animation-delay` or `transition-delay`. Here is an example of what that CSS might look like.


### Parallel Animations

You can apply multiple animations to an element at once using the `animation` shorthand property. Each can have their own durations and delays. This allows you to compose animations together and create complicated effects.

```css
.target-element {
  animation:
    rotate 3s,
    fade-in 2s;
}
```

In this example, the `rotate` and `fade-in` animations fire at the same time, but have different durations.

### Animating the items of a reordering list

Items in a `@for` loop will be removed and re-added, which will fire off animations using `@starting-styles` for entry animations. Alternatively, you can use `animate.enter` for this same behavior. Use `animate.leave` to animate elements as they are removed, as seen in the example below.


## Programmatic control of animations

You can retrieve animations off an element directly using [`Element.getAnimations()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/getAnimations). This returns an array of every [`Animation`](https://developer.mozilla.org/en-US/docs/Web/API/Animation) on that element. You can use the `Animation` API to do much more than you could with what the `AnimationPlayer` from the animations package offered. From here you can `cancel()`, `play()`, `pause()`, `reverse()` and much more. This native API should provide everything you need to control your animations.

## More on Angular animations

You might also be interested in the following:


---

### Route transition animation
Source: https://angular.dev/guide/routing/route-transition-animations

# Route transition animations

Route transition animations enhance user experience by providing smooth visual transitions when navigating between different views in your Angular application. [Angular Router](/guide/routing) includes built-in support for the browser's View Transitions API, enabling seamless animations between route changes in supported browsers.

## How View Transitions work

View transitions use the browser's native [`document.startViewTransition` API](https://developer.mozilla.org/en-US/docs/Web/API/Document/startViewTransition) to create smooth animations between different states of your application. The API works by:

1. **Capturing the current state** - The browser takes a screenshot of the current page
2. **Executing the DOM update** - Your callback function runs to update the DOM
3. **Capturing the new state** - The browser captures the updated page state
4. **Playing the transition** - The browser animates between the old and new states

Here's the basic structure of the `startViewTransition` API:

```ts
document.startViewTransition(async () => {
  await updateTheDOMSomehow();
});
```

For more details about the browser API, see the [Chrome Explainer](https://developer.chrome.com/docs/web-platform/view-transitions).

## How the Router uses view transitions

Angular Router integrates view transitions into the navigation lifecycle to create seamless route changes. During navigation, the Router:

1. **Completes navigation preparation** - Route matching, [lazy loading](guide/routing/loading-strategies#lazily-loaded-components-and-routes), [guards](/guide/routing/route-guards), [resources](/guide/routing/data-fetching-with-resources), and [resolvers](/guide/routing/data-resolvers) execute
2. **Initiates the view transition** - Router calls `startViewTransition` when routes are ready for activation
3. **Updates the DOM** - Router activates new routes and deactivates old ones within the transition callback
4. **Finalizes the transition** - The transition Promise resolves when Angular completes rendering

The Router's view transition integration acts as a [progressive enhancement](https://developer.mozilla.org/en-US/docs/Glossary/Progressive_Enhancement). When browsers don't support the View Transitions API, the Router performs normal DOM updates without animation, ensuring your application works across all browsers.

## Enabling View Transitions in the Router

Enable view transitions by adding the `withViewTransitions` feature to your [router configuration](/guide/routing/define-routes#adding-the-router-to-your-application). Angular supports both standalone and NgModule bootstrap approaches:

### Standalone bootstrap

```ts
import {bootstrapApplication} from '@angular/platform-browser';
import {provideRouter, withViewTransitions} from '@angular/router';
import {routes} from './app.routes';

bootstrapApplication(MyApp, {
  providers: [provideRouter(routes, withViewTransitions())],
});
```

### NgModule bootstrap

```ts
import {NgModule} from '@angular/core';
import {RouterModule} from '@angular/router';

@NgModule({
  imports: [RouterModule.forRoot(routes, {enableViewTransitions: true})],
})
export class AppRouting {}
```

[Try the "count" example on StackBlitz](https://stackblitz.com/edit/stackblitz-starters-2dnvtm?file=src%2Fmain.ts)

This example demonstrates how router navigation can replace direct `startViewTransition` calls for counter updates.

## Customizing transitions with CSS

You can customize view transitions using CSS to create unique animation effects. The browser creates separate transition elements that you can target with CSS selectors.

To create custom transitions:

1. **Add view-transition-name** - Assign unique names to elements you want to animate
2. **Define global animations** - Create CSS animations in your global styles
3. **Target transition pseudo-elements** - Use `::view-transition-old()` and `::view-transition-new()` selectors

Here's an example that adds a rotation effect to a counter element:

```css
/* Define keyframe animations */
@keyframes rotate-out {
  to {
    transform: rotate(90deg);
  }
}

@keyframes rotate-in {
  from {
    transform: rotate(-90deg);
  }
}

/* Target view transition pseudo-elements */
::view-transition-old(count),
::view-transition-new(count) {
  animation-duration: 200ms;
  animation-name: -ua-view-transition-fade-in, rotate-in;
}

::view-transition-old(count) {
  animation-name: -ua-view-transition-fade-out, rotate-out;
}
```

IMPORTANT: Define view transition animations in your global styles file, not in component styles. Angular's [view encapsulation](/guide/components/styling#style-scoping) scopes component styles, which prevents them from targeting the transition pseudo-elements correctly.

[Try the updated “count” example on StackBlitz](https://stackblitz.com/edit/stackblitz-starters-fwn4i7?file=src%2Fmain.ts)

## Advanced transition control with onViewTransitionCreated

The `withViewTransitions` feature accepts an options object with an `onViewTransitionCreated` callback for advanced control over view transitions. This callback:

- Runs in an [injection context](/guide/di/dependency-injection-context#run-within-an-injection-context)
- Receives a [`ViewTransitionInfo`](/api/router/ViewTransitionInfo) object containing:
  - The `ViewTransition` instance from `startViewTransition`
  - The [`ActivatedRouteSnapshot`](/api/router/ActivatedRouteSnapshot) for the route being navigated from
  - The [`ActivatedRouteSnapshot`](/api/router/ActivatedRouteSnapshot) for the route being navigated to

Use this callback to customize transition behavior based on navigation context. For example, you can skip transitions for specific navigation types:

```ts
import {inject} from '@angular/core';
import {Router, withViewTransitions, isActive, IsActiveMatchOptions} from '@angular/router';

withViewTransitions({
  onViewTransitionCreated: ({transition}) => {
    const router = inject(Router);
    const targetUrl = router.currentNavigation()!.finalUrl!;

    // Skip transition if only fragment or query params change
    const config: Partial<IsActiveMatchOptions> = {
      paths: 'exact',
      matrixParams: 'exact',
      fragment: 'ignored',
      queryParams: 'ignored',
    };

    const isTargetRouteCurrent = isActive(targetUrl, router, config);

    if (isTargetRouteCurrent()) {
      transition.skipTransition();
    }
  },
});
```

This example skips the view transition when navigation only changes the [URL fragment or query parameters](/guide/routing/read-route-state#query-parameters) (such as anchor links within the same page). The `skipTransition()` method prevents the animation while still allowing the navigation to complete.

## Respecting the user's motion preferences

The Router runs view transitions regardless of the user's [`prefers-reduced-motion`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion) setting. A preference for reduced motion doesn't necessarily mean the user wants no motion at all, so the framework leaves the choice to the application: you can [provide more subtle animations](https://developer.chrome.com/docs/web-platform/view-transitions/same-document#react_to_the_reduced_motion_preference) for these users, or skip the transition entirely with the `onViewTransitionCreated` callback:

```ts
withViewTransitions({
  onViewTransitionCreated: ({transition}) => {
    if (matchMedia('(prefers-reduced-motion: reduce)').matches) {
      transition.skipTransition();
    }
  },
});
```

## Examples from the Chrome explainer adapted to Angular

The following examples demonstrate various view transition techniques adapted from the Chrome team's documentation for use with Angular Router:

### Transitioning elements don't need to be the same DOM element

Elements can transition smoothly between different DOM elements as long as they share the same `view-transition-name`.

- [Chrome Explainer](https://developer.chrome.com/docs/web-platform/view-transitions/same-document#transitioning_elements_dont_need_to_be_the_same_dom_element)
- [Angular Example on StackBlitz](https://stackblitz.com/edit/stackblitz-starters-dh8npr?file=src%2Fmain.ts)

### Custom entry and exit animations

Create unique animations for elements entering and leaving the viewport during route transitions.

- [Chrome Explainer](https://developer.chrome.com/docs/web-platform/view-transitions/same-document#custom_entry_and_exit_transitions)
- [Angular Example on StackBlitz](https://stackblitz.com/edit/stackblitz-starters-8kly3o)

### Async DOM updates and waiting for content

Angular Router prioritizes immediate transitions over waiting for additional content to load.

- [Chrome Explainer](https://developer.chrome.com/docs/web-platform/view-transitions/same-document#async_dom_updates_and_waiting_for_content)

NOTE: Angular Router does not provide a way to delay view transitions. This design choice prevents pages from becoming non-interactive while waiting for additional content. As the Chrome documentation notes: "During this time, the page is frozen, so delays here should be kept to a minimum…in some cases it's better to avoid the delay altogether, and use the content you already have."

### Handle multiple view transition styles with view transition types

Use view transition types to apply different animation styles based on navigation context.

- [Chrome Explainer](https://developer.chrome.com/docs/web-platform/view-transitions/same-document#view-transition-types)
- [Angular Example on StackBlitz](https://stackblitz.com/edit/stackblitz-starters-vxzcam)

### Handle multiple view transition styles with a class name on the view transition root (deprecated)

This approach uses CSS classes on the transition root element to control animation styles.

- [Chrome Explainer](https://developer.chrome.com/docs/web-platform/view-transitions/same-document#changing-on-navigation-type)
- [Angular Example on StackBlitz](https://stackblitz.com/edit/stackblitz-starters-nmnzzg?file=src%2Fmain.ts)

### Transitioning without freezing other animations

Maintain other page animations during view transitions to create more dynamic user experiences.

- [Chrome Explainer](https://developer.chrome.com/docs/web-platform/view-transitions/same-document#transitioning-without-freezing)
- [Angular Example on StackBlitz](https://stackblitz.com/edit/stackblitz-starters-76kgww)

### Animating with JavaScript

Control view transitions programmatically using JavaScript APIs for complex animation scenarios.

- [Chrome Explainer](https://developer.chrome.com/docs/web-platform/view-transitions/same-document#animating-with-javascript)
- [Angular Example on StackBlitz](https://stackblitz.com/edit/stackblitz-starters-cklnkm)

---

### Migrating to native CSS animations
Source: https://next.angular.dev/guide/animations/migration

# Migrating away from Angular's Animations package

The `@angular/animations` package is deprecated as of v20.2, which also introduced the new `animate.enter` and `animate.leave` feature to add animations to your application. Using these new features, you can replace all animations based on `@angular/animations` with plain CSS or JS animation libraries. Removing `@angular/animations` from your application can significantly reduce the size of your JavaScript bundle. Native CSS animations generally offer superior performance, as they can benefit from hardware acceleration. This guide walks through the process of refactoring your code from `@angular/animations` to native CSS animations.

## How to write animations in native CSS

If you've never written any native CSS animations, there are a number of excellent guides to get you started. Here are a few of them:  
[MDN's CSS Animations guide](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_animations/Using_CSS_animations)  
[W3Schools CSS3 Animations guide](https://www.w3schools.com/css/css3_animations.asp)  
[The Complete CSS Animations Tutorial](https://www.lambdatest.com/blog/css-animations-tutorial/)  
[CSS Animation for Beginners](https://thoughtbot.com/blog/css-animation-for-beginners)

and a couple of videos:  
[Learn CSS Animation in 9 Minutes](https://www.youtube.com/watch?v=z2LQYsZhsFw)  
[Net Ninja CSS Animation Tutorial Playlist](https://www.youtube.com/watch?v=jgw82b5Y2MU&list=PL4cUxeGkcC9iGYgmEd2dm3zAKzyCGDtM5)

Check out some of these guides and tutorials, then come back to this guide.

## Creating Reusable Animations

Just like with the animations package, you can create reusable animations that can be shared across your application. The animations package version of this had you using the `animation()` function in a shared typescript file. The native CSS version of this is similar, but lives in a shared CSS file.

#### With Animations Package


#### With Native CSS


Adding the class `animated-class` to an element would trigger the animation on that element.

## Animating a Transition

### Animating State and Styles

The animations package allowed you to define various states using the [`state()`](api/animations/state) function within a component. Examples might be an `open` or `closed` state containing the styles for each respective state within the definition. For example:

#### With Animations Package {#animating-state-and-styles-with-animations-package}


This same behavior can be accomplished natively by using CSS classes, either with a keyframe animation or transition styling.

#### With Native CSS {#animating-state-and-styles-with-native-css}


Triggering the `open` or `closed` state is done by toggling classes on the element in your component. You can find examples of how to do this in our [template guide](guide/templates/binding#css-class-and-style-property-bindings).

You can see similar examples in the template guide for [animating styles directly](guide/templates/binding#css-style-properties).

### Transitions, Timing, and Easing

The animations package `animate()` function allows for providing timing, like duration, delays and easing. This can be done natively with CSS using several css properties or shorthand properties.

Specify `animation-duration`, `animation-delay`, and `animation-timing-function` for a keyframe animation in CSS, or alternatively use the `animation` shorthand property.


Similarly, you can use `transition-duration`, `transition-delay`, and `transition-timing-function` and the `transition` shorthand for animations that are not using `@keyframes`.


### Triggering an Animation

The animations package required specifying triggers using the `trigger()` function and nesting all of your states within it. With native CSS, this is unnecessary. Animations can be triggered by toggling CSS styles or classes. Once a class is present on an element, the animation will occur. Removing the class will revert the element back to whatever CSS is defined for that element. This results in significantly less code to do the same animation. Here's an example:

#### With Animations Package {#triggering-an-animation-with-animations-package}


#### With Native CSS {#triggering-an-animation-with-native-css}


## Transition and Triggers

### Predefined State and wildcard matching

The animations package offers the ability to match your defined states to a transition via strings. For example, animating from open to closed would be `open => closed`. You can use wildcards to match any state to a target state, like `* => closed` and the `void` keyword can be used for entering and exiting states. For example: `* => void` for when an element leaves a view or `void => *` for when the element enters a view.

These state matching patterns are not needed at all when animating with CSS directly. You can manage what transitions and `@keyframes` animations apply based on whatever classes you set and / or styles you set on the elements. You can also add `@starting-style` to control how the element looks upon immediately entering the DOM.

### Automatic Property Calculation with Wildcards

The animations package offers the ability to animate things that have been historically difficult to animate, like animating a set height to `height: auto`. You can now do this with pure CSS as well.

#### With Animations Package {#automatic-property-calculation-with-animations-package}


You can use CSS Grid to animate to auto height.

#### With Native CSS {#automatic-property-calculation-with-native-css}


If you don't have to worry about supporting all browsers, you can also check out `calc-size()`, which is the true solution to animating auto height. See [MDN's docs](https://developer.mozilla.org/en-US/docs/Web/CSS/calc-size) and (this tutorial)[https://frontendmasters.com/blog/one-of-the-boss-battles-of-css-is-almost-won-transitioning-to-auto/] for more information.

### Animate entering and leaving a view

The animations package offered the previously mentioned pattern matching for entering and leaving but also included the shorthand aliases of `:enter` and `:leave`.

#### With Animations Package {#enter-and-leave-with-animations-package}


#### With Native CSS {#enter-with-native-css}


#### With Native CSS {#leave-with-native-css}


For more information on `animate.enter` and `animate.leave`, see the [Enter and Leave animations guide](guide/animations).

### Animating increment and decrement

Along with the aforementioned `:enter` and `:leave`, there's also `:increment` and `:decrement`. You can animate these also by adding and removing classes. Unlike the animation package built-in aliases, there is no automatic application of classes when the values go up or down. You can apply the appropriate classes programmatically. Here's an example:

#### With Animations Package {#increment-and-decrement-with-animations-package}


#### With Native CSS {#increment-and-decrement-with-native-css}


### Parent / Child Animations

Unlike the animations package, when multiple animations are specified within a given component, no animation has priority over another and nothing blocks any animation from firing. Any sequencing of animations would have to be handled by your definition of your CSS animation, using animation / transition delay, and / or using `animationend` or `transitionend` to handle adding the next css to be animated.

Child animations fire only within the same component template. In the `@angular/animations` package, parent animations could query and trigger animations in nested child components using `query()` and `animateChild()`. With `animate.leave` and native CSS animations, animations defined inside nested child component templates will not fire when a parent component removes an element or view. Only nested animations within the same Angular component template will execute. See the [Enter and Leave Animations guide](guide/animations#element-removal-order) for more information on this.

### Disabling an animation or all animations

With native CSS animations, if you'd like to disable the animations that you've specified, you have multiple options.

1. Create a custom class that forces animation and transition to `none`.

```css
.no-animation {
  animation: none !important;
  transition: none !important;
}
```

Applying this class to an element prevents any animation from firing on that element. You could alternatively scope this to your entire DOM or section of your DOM to enforce this behavior. However, this prevents animation events from firing. If you are awaiting animation events for element removal, this solution won't work. A workaround is to set durations to 1 millisecond instead.

2. Use the [`prefers-reduced-motion`](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion) media query to ensure no animations play for users that prefer less animation.

3. Prevent adding animation classes programmatically

### Animation Callbacks

The animations package exposed callbacks for you to use in the case that you want to do something when the animation has finished. Native CSS animations also have these callbacks.

[`OnAnimationStart`](https://developer.mozilla.org/en-US/docs/Web/API/Element/animationstart_event)  
[`OnAnimationEnd`](https://developer.mozilla.org/en-US/docs/Web/API/Element/animationend_event)  
[`OnAnimationIteration`](https://developer.mozilla.org/en-US/docs/Web/API/Element/animationiteration_event)  
[`OnAnimationCancel`](https://developer.mozilla.org/en-US/docs/Web/API/Element/animationcancel_event)

[`OnTransitionStart`](https://developer.mozilla.org/en-US/docs/Web/API/Element/transitionstart_event)  
[`OnTransitionRun`](https://developer.mozilla.org/en-US/docs/Web/API/Element/transitionrun_event)  
[`OnTransitionEnd`](https://developer.mozilla.org/en-US/docs/Web/API/Element/transitionend_event)  
[`OnTransitionCancel`](https://developer.mozilla.org/en-US/docs/Web/API/Element/transitioncancel_event)

The Web Animations API has a lot of additional functionality. [Take a look at the documentation](https://developer.mozilla.org/en-US/docs/Web/API/Web_Animations_API) to see all the available animation APIs.

NOTE: Be aware of bubbling issues with these callbacks. If you are animating children and parents, the events bubble up from children to parents. Consider stopping propagation or looking at more details within the event to determine if you're responding to the desired event target rather than an event bubbling up from a child node. You can examine the `animationname` property or the properties being transitioned to verify you have the right nodes.

## Complex Sequences

The animations package has built-in functionality for creating complex sequences. These sequences are all entirely possible without the animations package.

### Targeting specific elements

In the animations package, you could target specific elements by using the `query()` function to find specific elements by a CSS class name, similar to [`document.querySelector()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/querySelector). This is unnecessary in a native CSS animation world. Instead, you can use your CSS selectors to target sub-classes and apply any desired `transform` or `animation`.

To toggle classes for child nodes within a template, you can use class and style bindings to add the animations at the right points.

### Stagger()

The `stagger()` function allowed you to delay the animation of each item in a list of items by a specified time to create a cascade effect. You can replicate this behavior in native CSS by utilizing `animation-delay` or `transition-delay`. Here is an example of what that CSS might look like.

#### With Animations Package {#stagger-with-animations-package}


#### With Native CSS {#stagger-with-native-css}


### Parallel Animations

The animations package has a `group()` function to play multiple animations at the same time. In CSS, you have full control over animation timing. If you have multiple animations defined, you can apply all of them at once.

```css
.target-element {
  animation:
    rotate 3s,
    fade-in 2s;
}
```

In this example, the `rotate` and `fade-in` animations fire at the same time.

### Animating the items of a reordering list

Items reordering in a list works out of the box using the previously described techniques. No additional special work is required. Items in a `@for` loop will be removed and re-added properly, which will fire off animations using `@starting-styles` for entry animations. Alternatively, you can use `animate.enter` for this same behavior. Use `animate.leave` to animate elements as they are removed, as seen in the example above.

#### With Animations Package {#reordering-list-with-animations-package}


#### With Native CSS {#reordering-list-with-native-css}


## Migrating usages of AnimationPlayer

The `AnimationPlayer` class allows access to an animation to do more advanced things like pause, play, restart, and finish an animation through code. All of these things can be handled natively as well.

You can retrieve animations off an element directly using [`Element.getAnimations()`](https://developer.mozilla.org/en-US/docs/Web/API/Element/getAnimations). This returns an array of every [`Animation`](https://developer.mozilla.org/en-US/docs/Web/API/Animation) on that element. You can use the `Animation` API to do much more than you could with what the `AnimationPlayer` from the animations package offered. From here you can `cancel()`, `play()`, `pause()`, `reverse()` and much more. This native API should provide everything you need to control your animations.

## Route Transitions

You can use view transitions to animate between routes. See the [Route Transition Animations Guide](guide/routing/route-transition-animations) to get started.

---


## APIs

### API reference
Source: https://angular.dev/api

### API reference
Source: https://angular.dev/api

The API reference is an index of every public Angular API symbol, grouped by package
(@angular/core, @angular/common, @angular/router, @angular/forms, ...).
Individual symbols are documented at https://angular.dev/api/<symbol>
(for example https://angular.dev/api/core/component).
The index is generated and contains several thousand entries, so it is not inlined here;
all guide pages linked from the table of contents below are fully inlined.


---

### CLI command reference
Source: https://angular.dev/cli

### CLI command reference
Source: https://angular.dev/cli

#### `ng `
Command: ``



#### `ng add`
Command: `ng add <collection>`

Adds support for an external library to your project.

Adds the npm package for a published library to your workspace, and configures
the project in the current working directory to use that library, as specified by the library's schematic.
For example, adding `@angular/pwa` configures your project for PWA support:

```bash
ng add @angular/pwa
```


#### `ng analytics`
Command: `ng analytics`

Configures the gathering of Angular CLI usage metrics.

You can help the Angular Team to prioritize features and improvements by permitting the Angular team to send command-line command usage statistics to Google.
The Angular Team does not collect usage statistics unless you explicitly opt in. When installing the Angular CLI you are prompted to allow global collection of usage statistics.
If you say no or skip the prompt, no data is collected.

### What is collected?

Usage analytics include the commands and selected flags for each execution.
Usage analytics may include the following information:

- Your operating system \(macOS, Linux distribution, Windows\) and its version.
- Package manager name and version \(local version only\).
- Node.js version \(local version only\).
- Angular CLI version \(local version only\).
- Command name that was run.
- Workspace information, the number of application and library projects.
- For schematics commands \(add, generate and new\), the schematic collection and name and a list of selected flags.
- For build commands \(build, serve\), the builder name, the number and size of bundles \(initial and lazy\), compilation units, the time it took to build and rebuild, and basic Angular-specific API usage.

Only Angular owned and developed schematics and builders are reported.
Third-party schematics and builders do not send data to the Angular Team.


#### `ng build`
Command: `ng build [project]`

Compiles an Angular application or library into an output directory named dist/ at the given output path.

The command can be used to build a project of type "application" or "library".
Depending on the project type, the CLI invokes either the application builder (`@angular/build:application`) or the library builder (`@angular/build:library`), each supporting its own set of options.

The application builder uses the [esbuild](https://esbuild.github.io/) build tool, with default configuration options specified in the workspace configuration file (`angular.json`) or with a named alternative configuration.
A "development" configuration is created by default when you use the CLI to create the project, and you can use that configuration by specifying the `--configuration development`.

The configuration options generally correspond to the command options.
You can override individual configuration defaults by specifying the corresponding options on the command line.
The command can accept option names given in dash-case.
Note that in the configuration file, you must specify names in camelCase.

Some additional options can only be set through the configuration file,
either by direct editing or with the `ng config` command.
These include `assets`, `styles`, and `scripts` objects that provide runtime-global resources to include in the project.
Resources in CSS, such as images and fonts, are automatically written to the `media` directory of the browser output
(configurable with the `outputPath.media` option) and fingerprinted when `outputHashing` is set to `all` or `media`.

For further details, see [Workspace Configuration](reference/configs/workspace-config).


#### `ng cache`
Command: `ng cache`

Configure persistent disk cache and retrieve cache statistics.

Angular CLI saves a number of cachable operations on disk by default.

When you re-run the same build, the build system restores the state of the previous build and re-uses previously performed operations, which decreases the time taken to build and test your applications and libraries.

To amend the default cache settings, add the `cli.cache` object to your [Workspace Configuration](reference/configs/workspace-config).
The object goes under `cli.cache` at the top level of the file, outside the `projects` sections.

```jsonc
{
  "$schema": "./node_modules/@angular/cli/lib/config/schema.json",
  "version": 1,
  "cli": {
    "cache": {
      // ...
    },
  },
  "projects": {},
}
```

For more information, see [cache options](reference/configs/workspace-config#cache-options).

### Cache environments

By default, disk cache is only enabled for local environments. The value of environment can be one of the following:

- `all` - allows disk cache on all machines.
- `local` - allows disk cache only on development machines.
- `ci` - allows disk cache only on continuous integration (CI) systems.

To change the environment setting to `all`, run the following command:

```bash
ng config cli.cache.environment all
```

For more information, see `environment` in [cache options](reference/configs/workspace-config#cache-options).


The Angular CLI checks for the presence and value of the `CI` environment variable to determine in which environment it is running.


### Cache path

By default, `.angular/cache` is used as a base directory to store cache results.

To change this path to `.cache/ng`, run the following command:

```bash
ng config cli.cache.path ".cache/ng"
```


#### `ng completion`
Command: `ng completion`

Set up Angular CLI autocompletion for your terminal.

Setting up autocompletion configures your terminal, so pressing the `<TAB>` key while in the middle
of typing will display various commands and options available to you. This makes it very easy to
discover and use CLI commands without lots of memorization.

![A demo of Angular CLI autocompletion in a terminal. The user types several partial `ng` commands,
using autocompletion to finish several arguments and list contextual options.
](assets/images/guide/cli/completion.gif)

## Automated setup

The CLI should prompt and ask to set up autocompletion for you the first time you use it (v14+).
Simply answer "Yes" and the CLI will take care of the rest.

```
$ ng serve
? Would you like to enable autocompletion? This will set up your terminal so pressing TAB while typing Angular CLI commands will show possible options and autocomplete arguments. (Enabling autocompletion will modify configuration files in your home directory.) Yes
Appended `source <(ng completion script)` to `/home/my-username/.bashrc`. Restart your terminal or run:

source <(ng completion script)

to autocomplete `ng` commands.

# Serve output...
```

If you already refused the prompt, it won't ask again. But you can run `ng completion` to
do the same thing automatically.

This modifies your terminal environment to load Angular CLI autocompletion, but can't update your
current terminal session. Either restart it or run `source <(ng completion script)` directly to
enable autocompletion in your current session.

Test it out by typing `ng ser<TAB>` and it should autocomplete to `ng serve`. Ambiguous arguments
will show all possible options and their documentation, such as `ng generate <TAB>`.

## Manual setup

Some users may have highly customized terminal setups, possibly with configuration files checked
into source control with an opinionated structure. `ng completion` only ever appends Angular's setup
to an existing configuration file for your current shell, or creates one if none exists. If you want
more control over exactly where this configuration lives, you can manually set it up by having your
shell run at startup:

```bash
if [[ $- == *i* ]]; then
  source <(ng completion script)
fi
```

This is equivalent to what `ng completion` will automatically set up, and gives power users more
flexibility in their environments when desired.

## Platform support

Angular CLI supports autocompletion for the Bash and Zsh shells on MacOS and Linux operating
systems. On Windows, Git Bash and [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/)
using Bash or Zsh are supported.

## Global install

Autocompletion works by configuring your terminal to invoke the Angular CLI on startup to load the
setup script. This means the terminal must be able to find and execute the Angular CLI, typically
through a global install that places the binary on the user's `$PATH`. If you get
`command not found: ng`, make sure the CLI is installed globally which you can do with the `-g`
flag:

```bash
npm install -g @angular/cli
```


#### `ng config`
Command: `ng config [json-path] [value]`

Retrieves or sets Angular configuration values in the angular.json file for the workspace.

A workspace has a single CLI configuration file, `angular.json`, at the top level.
The `projects` object contains a configuration object for each project in the workspace.

You can edit the configuration directly in a code editor,
or indirectly on the command line using this command.

The configurable property names match command option names,
except that in the configuration file, all names must use camelCase,
while on the command line options can be given dash-case.

For further details, see [Workspace Configuration](reference/configs/workspace-config).

For configuration of CLI usage analytics, see [ng analytics](cli/analytics).


#### `ng deploy`
Command: `ng deploy [project]`

Invokes the deploy builder for a specified project, the only project with a deploy target, or the project in the current working directory.

The command takes an optional project name, as specified in the `projects` section of the `angular.json` workspace configuration file.
When a project name is not supplied, executes the `deploy` builder for the only project that has a `deploy` target,
or for the project in the current working directory.

To use the `ng deploy` command, use `ng add` to add a package that implements deployment capabilities to your favorite platform.
Adding the package automatically updates your workspace configuration, adding a deployment
[CLI builder](tools/cli/cli-builder).
For example:

```json
"projects": {
  "my-project": {
    ...
    "architect": {
      ...
      "deploy": {
        "builder": "@angular/fire:deploy",
        "options": {}
      }
    }
  }
}
```


#### `ng e2e`
Command: `ng e2e [project]`

Builds and serves an Angular application, then runs end-to-end tests.

#### `ng extract-i18n`
Command: `ng extract-i18n [project]`

Extracts i18n messages from source code.

#### `ng generate`
Command: `ng generate <schematic>`

Generates and/or modifies files based on a schematic.

#### `ng lint`
Command: `ng lint [project]`

Runs linting tools on Angular application code in a given project folder.

The command takes an optional project name, as specified in the `projects` section of the `angular.json` workspace configuration file.
When a project name is not supplied, executes the `lint` builder for all projects.

To use the `ng lint` command, use `ng add` to add a package that implements linting capabilities. Adding the package automatically updates your workspace configuration, adding a lint [CLI builder](tools/cli/cli-builder).
For example:

```json
"projects": {
  "my-project": {
    ...
    "architect": {
      ...
      "lint": {
        "builder": "@angular-eslint/builder:lint",
        "options": {}
      }
    }
  }
}
```


#### `ng new`
Command: `ng new [name]`

Creates a new Angular workspace.

Creates and initializes a new Angular workspace with an initial application.

Provides interactive prompts for optional configuration, such as adding routing support.
All prompts can safely be allowed to default.

- The new workspace folder is given the specified project name, and contains configuration files at the top level.

- By default, the files for a new initial application (with the same name as the workspace) are placed in the `src/` subfolder.
- The new application's configuration appears in the `projects` section of the `angular.json` workspace configuration file, under its project name.

- Subsequent applications that you generate in the workspace reside in the `projects/` subfolder.

If you plan to have multiple applications in the workspace, you can create an empty workspace by using the `--no-create-application` option.
You can then use `ng generate application` to create an initial application.
This allows a workspace name different from the initial app name, and ensures that all applications reside in the `/projects` subfolder, matching the structure of the configuration file.


#### `ng run`
Command: `ng run <target>`

Runs an Architect target with an optional custom builder configuration defined in your project.

Architect is the tool that the CLI uses to perform complex tasks such as compilation, according to provided configurations.
The CLI commands run Architect targets such as `build`, `serve`, `test`, and `lint`.
Each named target has a default configuration, specified by an `options` object,
and an optional set of named alternate configurations in the `configurations` object.

For example, the `serve` target for a newly generated app has a predefined
alternate configuration named `production`.

You can define new targets and their configuration options in the `architect` section
of the `angular.json` file, which you can run from the command line using the `ng run` command.


#### `ng serve`
Command: `ng serve [project]`

Builds and serves your application, rebuilding on file changes.

#### `ng test`
Command: `ng test [project]`

Runs unit tests in a project.

Takes the name of the project, as specified in the `projects` section of the `angular.json` workspace configuration file.
When a project name is not supplied, it will execute for all projects.


#### `ng update`
Command: `ng update [packages..]`

Updates your workspace and its dependencies. See https://update.angular.dev/.

Perform a basic update to the current stable release of the core framework and CLI by running the following command.

```
ng update @angular/cli @angular/core
```

To update to the next beta or pre-release version, use the `--next` option.

To update from one major version to another, use the format

```
ng update @angular/cli@^<major_version> @angular/core@^<major_version>
```

We recommend that you always update to the latest patch version, as it contains fixes we released since the initial major release.
For example, use the following command to take the latest 21.x.x version and use that to update.

```
ng update @angular/cli@^21 @angular/core@^21
```

For detailed information and guidance on updating your application, see the interactive [Angular Update Guide](/update-guide).


#### `ng version`
Command: `ng version`

Outputs Angular CLI version.


---


## Others

### Zoneless
Source: https://angular.dev/guide/zoneless

# Angular without ZoneJS (Zoneless)

## Why use Zoneless?

The main advantages to removing ZoneJS as a dependency are:

- **Improved performance**: ZoneJS uses DOM events and async tasks as indicators of when application state _might_ have updated and subsequently triggers application synchronization to run change detection on the application's views. ZoneJS does not have any insight into whether application state actually changed and so this synchronization is triggered more frequently than necessary.
- **Improved Core Web Vitals**: ZoneJS brings a fair amount of overhead, both in payload size and startup time.
- **Improved debugging experience**: ZoneJS makes debugging code more difficult. Stack traces are harder to understand with ZoneJS. It's also difficult to understand when code breaks as a result of being outside the Angular Zone.
- **Better ecosystem compatibility**: ZoneJS works by patching browser APIs but does not automatically have patches for every new browser API. Some APIs, such as `async`/`await`, cannot be patched effectively and must be downleveled to work with ZoneJS. Sometimes libraries in the ecosystem are also incompatible with the way ZoneJS patches the native APIs. Removing ZoneJS as a dependency ensures better long-term compatibility by removing a source of complexity, monkey patching, and ongoing maintenance.

## Enabling Zoneless in an application

Zoneless is the default in Angular v21+ so you do not need to do anything to enable it. You should verify that `provideZoneChangeDetection` is not used anywhere to override the default configuration.

If you are using Angular v20, enable zoneless change detection by adding `provideZonelessChangeDetection()` at bootstrap:

```ts {header: 'standalone bootstrap'}
bootstrapApplication(MyApp, {providers: [provideZonelessChangeDetection()]});
```

```ts {header: 'NgModule bootstrap'}
platformBrowser().bootstrapModule(AppModule);

@NgModule({
  providers: [provideZonelessChangeDetection()],
})
export class AppModule {}
```

## Removing ZoneJS

Zoneless applications should remove ZoneJS entirely from the build to reduce bundle size. ZoneJS is typically
loaded via the `polyfills` option in `angular.json`, both in the `build` and `test` targets. Remove `zone.js`
and `zone.js/testing` from both to remove it from the build. Projects which use an explicit `polyfills.ts` file
should remove `import 'zone.js';` and `import 'zone.js/testing';` from the file.

After removing ZoneJS from the build, there is no longer a need for a `zone.js` dependency either and the
package can be removed entirely:

```shell
npm uninstall zone.js
```

## Requirements for Zoneless compatibility

Angular relies on notifications from core APIs in order to determine when to run change detection and on which views.
These notifications include:

- `ChangeDetectorRef.markForCheck` (called automatically by `AsyncPipe`)
- `ComponentRef.setInput`
- Updating a signal that's read in a template
- Bound host or template listeners callbacks
- Attaching a view that was marked dirty by one of the above

### `OnPush`-compatible components

One way to ensure that a component is using the correct notification mechanisms from above is to
use [ChangeDetectionStrategy.OnPush](/best-practices/skipping-subtrees#using-onpush).

The `OnPush` change detection strategy is not required, but it is a recommended step towards zoneless compatibility for application components. It is not always possible for library components to use `ChangeDetectionStrategy.OnPush`.
When a library component is a host for user-components which might use `ChangeDetectionStrategy.Eager`/`Default`, it cannot use `OnPush` because that would prevent the child component from being refreshed if it is not `OnPush` compatible and relies on ZoneJS to trigger change detection. Components can use the `Default` strategy as long as they notify Angular when change detection needs to run (calling `markForCheck`, using signals, `AsyncPipe`, etc.).
Being a host for a user component means using an API such as `ViewContainerRef.createComponent` and not just hosting a portion of a template from a user component (i.e. content projection or using a template ref input).

### Remove `NgZone.onMicrotaskEmpty`, `NgZone.onUnstable`, `NgZone.isStable`, or `NgZone.onStable`

Applications and libraries need to remove uses of `NgZone.onMicrotaskEmpty`, `NgZone.onUnstable`, and `NgZone.onStable`.
These observables will never emit when an Application enables zoneless change detection.
Similarly, `NgZone.isStable` will always be `true` and should not be used as a condition for code execution.

The `NgZone.onMicrotaskEmpty` and `NgZone.onStable` observables are most often used to wait for Angular to
complete change detection before performing a task. Instead, these can be replaced by `afterNextRender`
if they need to wait for a single change detection or `afterEveryRender` if there is some condition that might span
several change detection rounds. In other cases, these observables were used because they happened to be
familiar and have similar timing to what was needed. More straightforward or direct DOM APIs can be used instead,
such as `MutationObserver` when code needs to wait for certain DOM state (rather than waiting for it indirectly
through Angular's render hooks).

`NgZone.run` and `NgZone.runOutsideAngular` do not need to be removed in order for code to be compatible with
Zoneless applications. In fact, removing these calls can lead to performance regressions for libraries that
are used in applications that still rely on ZoneJS.

### `PendingTasks` for Server Side Rendering (SSR)

If you are using SSR with Angular, you may know that it relies on ZoneJS to help determine when the application
is "stable" and can be serialized. If there are asynchronous tasks that should prevent serialization, an application
not using ZoneJS must make Angular aware of these with the [PendingTasks](/api/core/PendingTasks) service. Serialization
will wait for the first moment that all pending tasks have been removed.

The two most straightforward uses of pending tasks are the `run` method:

```typescript
const taskService = inject(PendingTasks);
taskService.run(async () => {
  const someResult = await doSomeWorkThatNeedsToBeRendered();
  this.someState.set(someResult);
});
```

For more complicated use-cases, you can manually add and remove a pending task:

```typescript
const taskService = inject(PendingTasks);
const taskCleanup = taskService.add();
try {
  await doSomeWorkThatNeedsToBeRendered();
} catch {
  // handle error
} finally {
  taskCleanup();
}
```

In addition, the [pendingUntilEvent](/api/core/rxjs-interop/pendingUntilEvent) helper in `rxjs-interop` ensures
the application remains unstable until the observable emits, completes, errors, or is unsubscribed.

```typescript
readonly myObservableState = someObservable.pipe(pendingUntilEvent());
```

The framework uses this service internally as well to prevent serialization until asynchronous tasks are complete. These include, but are not limited to, an ongoing Router navigation and an incomplete `HttpClient` request.

### Reactive forms in zoneless applications

Reactive forms model updates (`setValue`, `patchValue`, `FormArray.push`, and similar APIs) update form state and emit form observables, but they do not automatically schedule component change detection.

If a template depends on reactive forms state, connect forms observables to a change-detection notification (for example `ChangeDetectorRef.markForCheck()`), or reflect the data through signals consumed by the template.

## Testing and Debugging

### Using Zoneless in `TestBed`

`TestBed` uses zoneless change detection by default, even when `zone.js` is loaded via the `polyfills`.

To use Zone-based change detection in a test, add `provideZoneChangeDetection()` to the `TestBed` providers. Adding `provideZonelessChangeDetection()` is optional:

```typescript
TestBed.configureTestingModule({
  // Optional: include the provider to force the testing environment
  // uses the same zoneless behavior as a zoneless application.
  providers: [provideZonelessChangeDetection()],
});

const fixture = TestBed.createComponent(MyComponent);
await fixture.whenStable();
```

To ensure tests have the most similar behavior to production code,
avoid using `fixture.detectChanges()` when possible. This forces
change detection to run when Angular might otherwise have not
scheduled change detection. Tests should ensure these notifications
are happening and allow Angular to handle when to synchronize
state rather than manually forcing it to happen in the test.

For existing test suites, using `fixture.detectChanges()` is a common pattern
and it is likely not worth the effort of converting these to
`await fixture.whenStable()`. `TestBed` will still enforce that the
fixture's component is `OnPush` compatible and throws `ExpressionChangedAfterItHasBeenCheckedError`
if it finds that template values were updated without a
change notification (i.e. `fixture.componentInstance.someValue = 'newValue';`).
If the component is used in production, this issue should be addressed by updating
the component to use signals for state or call `ChangeDetectorRef.markForCheck()`.
If the component is only used as a test wrapper and never used in an application,
it is acceptable to use `ChangeDetectionStrategy.Eager` for it and call `fixture.changeDetectorRef.markForCheck()`.

IMPORTANT: `fixture.changeDetectorRef` belongs to the fixture's host view, so its `markForCheck()`
doesn't refresh the template of an `OnPush` component. Your application components shouldn't need it
in tests. If a test needs it, the component usually isn't fully reactive or
[zoneless compatible](#requirements-for-zoneless-compatibility).

### Debug-mode check to ensure updates are detected

IMPORTANT: `provideCheckNoChangesConfig` is in [developer preview](reference/releases#developer-preview).

Angular also provides an additional tool to help verify that an application is making
updates to state in a zoneless-compatible way. `provideCheckNoChangesConfig({exhaustive: true, interval: <milliseconds>})`
can be used to periodically check to ensure that no bindings have been updated
without a notification. Angular throws `ExpressionChangedAfterItHasBeenCheckedError`
if there is an updated binding that would not have refreshed by the zoneless change
detection.

---

### Error encyclopedia
Source: https://angular.dev/errors

# Error Encyclopedia

## Runtime errors

| Code      | Name                                                                                 |
| :-------- | :----------------------------------------------------------------------------------- |
| `NG0100`  | [Expression Changed After Checked](errors/NG0100)                                    |
| `NG0200`  | [Circular Dependency in DI](errors/NG0200)                                           |
| `NG0201`  | [No Provider Found](errors/NG0201)                                                   |
| `NG0203`  | [`inject()` must be called from an injection context](errors/NG0203)                 |
| `NG0204`  | [Invalid Injection Token](errors/NG0204)                                             |
| `NG0205`  | [Injector has already been destroyed](errors/NG0205)                                 |
| `NG0207`  | [EnvironmentProviders in wrong context](errors/NG0207)                               |
| `NG0209`  | [Invalid multi provider](errors/NG0209)                                              |
| `NG0300`  | [Selector Collision](errors/NG0300)                                                  |
| `NG0301`  | [Export Not Found](errors/NG0301)                                                    |
| `NG0302`  | [Pipe Not Found](errors/NG0302)                                                      |
| `NG0318`  | [Invalid style property binding value](errors/NG0318)                                |
| `NG0401`  | [Missing platform](errors/NG0401)                                                    |
| `NG0403`  | [Bootstrapped NgModule doesn't specify which component to initialize](errors/NG0403) |
| `NG0500`  | [Hydration Node Mismatch](errors/NG0500)                                             |
| `NG0501`  | [Hydration Missing Siblings](errors/NG0501)                                          |
| `NG0502`  | [Hydration Missing Node](errors/NG0502)                                              |
| `NG0503`  | [Hydration Unsupported Projection of DOM Nodes](errors/NG0503)                       |
| `NG0504`  | [Skip hydration flag is applied to an invalid node](errors/NG0504)                   |
| `NG0505`  | [No hydration info in server response](errors/NG0505)                                |
| `NG0506`  | [NgZone remains unstable](errors/NG0506)                                             |
| `NG0507`  | [HTML content was altered after SSR](errors/NG0507)                                  |
| `NG0600`  | [Signal write in a disallowed context](errors/NG0600)                                |
| `NG0602`  | [Disallowed function call inside reactive context](errors/NG0602)                    |
| `NG0750`  | [@defer dependencies failed to load](errors/NG0750)                                  |
| `NG0751`  | [@defer behavior when HMR is enabled](errors/NG0751)                                 |
| `NG0904`  | [Unsafe value used in a resource URL context](errors/NG0904)                         |
| `NG0910`  | [Unsafe bindings on an element](errors/NG0910)                                       |
| `NG0912`  | [Component ID generation collision](errors/NG0912)                                   |
| `NG0913`  | [Runtime Performance Warnings](errors/NG0913)                                        |
| `NG0919`  | [Circular Dependency Detected](errors/NG0919)                                        |
| `NG0950`  | [Required input is accessed before a value is set](errors/NG0950)                    |
| `NG0951`  | [Child query result is required but no value is available](errors/NG0951)            |
| `NG0955`  | [Track expression resulted in duplicated keys for a given collection](errors/NG0955) |
| `NG0956`  | [Tracking expression caused re-creation of the DOM structure](errors/NG0956)         |
| `NG0991`  | [Resource completed before producing a value](errors/NG0991)                         |
| `NG01002` | [Missing Control Value](errors/NG01002)                                              |
| `NG01055` | [Missing control for `formControlName`](errors/NG01055)                              |
| `NG01101` | [Wrong Async Validator Return Type](errors/NG01101)                                  |
| `NG01203` | [Missing value accessor](errors/NG01203)                                             |
| `NG01902` | [Orphan field in signal forms](errors/NG01902)                                       |
| `NG02200` | [Missing Iterable Differ](errors/NG02200)                                            |
| `NG02800` | [JSONP support in HttpClient configuration](errors/NG02800)                          |
| `NG02802` | [Headers not transferred by HttpTransferCache](errors/NG02802)                       |
| `NG02825` | [Fetch response body exceeds the configured limit](errors/NG02825)                   |
| `NG05000` | [Hydration with unsupported Zone.js instance.](errors/NG05000)                       |
| `NG05104` | [Root element was not found.](errors/NG05104)                                        |
| `NG05106` | [Insertion reference node not found](errors/NG05106)                                 |
| `NG05201` | [Unsafe resource URL passed to DomSanitizer](errors/NG05201)                         |
| `NG05703` | [Suspicious URL origin change during SSR](errors/NG05703)                            |

## Compiler errors

| Code     | Name                                                        |
| :------- | :---------------------------------------------------------- |
| `NG1001` | [Argument Not Literal](errors/NG1001)                       |
| `NG2003` | [Missing Token](errors/NG2003)                              |
| `NG2009` | [Invalid Shadow DOM selector](errors/NG2009)                |
| `NG3003` | [Import Cycle Detected](errors/NG3003)                      |
| `NG6100` | [NgModule.id Set to module.id anti-pattern](errors/NG6100)  |
| `NG8001` | [Invalid Element](errors/NG8001)                            |
| `NG8002` | [Invalid Attribute](errors/NG8002)                          |
| `NG8003` | [Missing Reference Target](errors/NG8003)                   |
| `NG8011` | [Control Flow Preventing Content Projection](errors/NG8011) |
| `NG8023` | [Multiple Components Match Same Element](errors/NG8023)     |
| `NG8024` | [Conflicting Host Directive Binding](errors/NG8024)         |

---

### Extended diagnostics
Source: https://angular.dev/extended-diagnostics

# Extended Diagnostics

There are many coding patterns that are technically valid to the compiler or runtime, but which may have complex nuances or caveats.
These patterns may not have the intended effect expected by a developer, which often leads to bugs.
The Angular compiler includes "extended diagnostics" which identify many of these patterns, in order to warn developers about the potential issues and enforce common best practices within a codebase.

## Diagnostics

Currently, Angular supports the following extended diagnostics:

| Code     | Name                                                                  |
| :------- | :-------------------------------------------------------------------- |
| `NG8101` | [`invalidBananaInBox`](extended-diagnostics/NG8101)                   |
| `NG8102` | [`nullishCoalescingNotNullable`](extended-diagnostics/NG8102)         |
| `NG8103` | [`missingControlFlowDirective`](extended-diagnostics/NG8103)          |
| `NG8104` | [`textAttributeNotBinding`](extended-diagnostics/NG8104)              |
| `NG8105` | [`missingNgForOfLet`](extended-diagnostics/NG8105)                    |
| `NG8106` | [`suffixNotSupported`](extended-diagnostics/NG8106)                   |
| `NG8107` | [`optionalChainNotNullable`](extended-diagnostics/NG8107)             |
| `NG8108` | [`skipHydrationNotStatic`](extended-diagnostics/NG8108)               |
| `NG8109` | [`interpolatedSignalNotInvoked`](extended-diagnostics/NG8109)         |
| `NG8111` | [`uninvokedFunctionInEventBinding`](extended-diagnostics/NG8111)      |
| `NG8112` | [`unusedLetDeclaration`](extended-diagnostics/NG8112)                 |
| `NG8113` | [`unusedStandaloneImports`](extended-diagnostics/NG8113)              |
| `NG8114` | [`unparenthesizedNullishCoalescing`](extended-diagnostics/NG8114)     |
| `NG8115` | [`uninvokedTrackFunction`](extended-diagnostics/NG8115)               |
| `NG8116` | [`missingStructuralDirective`](extended-diagnostics/NG8116)           |
| `NG8117` | [`uninvokedFunctionInTextInterpolation`](extended-diagnostics/NG8117) |
| `NG8119` | [`numericCssPropertyRequiresUnit`](extended-diagnostics/NG8119)       |
| `NG8021` | [`deferTriggerMisconfiguration`](extended-diagnostics/NG8021)         |

## Configuration

Extended diagnostics are warnings by default and do not block compilation.
Each diagnostic can be configured as either:

| Error category | Effect                                                                                                                                                                   |
| :------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `warning`      | Default - The compiler emits the diagnostic as a warning but does not block compilation. The compiler will still exist with status code 0, even if warnings are emitted. |
| `error`        | The compiler emits the diagnostic as an error and fails the compilation. The compiler will exit with a non-zero status code if one or more errors are emitted.           |
| `suppress`     | The compiler does _not_ emit the diagnostic at all.                                                                                                                      |

Check severity can be configured as an [Angular compiler option](reference/configs/angular-compiler-options):

```json
{
  "angularCompilerOptions": {
    "extendedDiagnostics": {
      "checks": {
        "invalidBananaInBox": "suppress"
      },
      "defaultCategory": "error"
    }
  }
}
```

The `checks` field maps the name of individual diagnostics to their associated category.
See [Diagnostics](#diagnostics) for a complete list of extended diagnostics and the name to use for configuring them.

The `defaultCategory` field is used for any diagnostics that are not explicitly listed under `checks`.
If not set, such diagnostics will be treated as `warning`.

Extended diagnostics will emit when [`strictTemplates`](tools/cli/template-typecheck#strict-mode) is enabled.
This is required to allow the compiler to better understand Angular template types and provide accurate and meaningful diagnostics.

## Semantic Versioning

The Angular team intends to add or enable new extended diagnostics in **minor** versions of Angular (see [semver](https://docs.npmjs.com/about-semantic-versioning)).
This means that upgrading Angular may show new warnings in your existing codebase.
This enables the team to deliver features more quickly and to make extended diagnostics more accessible to developers.

However, setting `"defaultCategory": "error"` will promote such warnings to hard errors.
This can cause a minor version upgrade to introduce compilation errors, which may be seen as a semver non-compliant breaking change.
Any new diagnostics can be suppressed or demoted to warnings via the above [configuration](#configuration), so the impact of a new diagnostic should be minimal to
projects that treat extended diagnostics as errors by default.
Defaulting to error is a very powerful tool; just be aware of this semver caveat when deciding if `error` is the right default for your project.

## New Diagnostics

The Angular team is always open to suggestions about new diagnostics that could be added.
Extended diagnostics should generally:

- Detect a common, non-obvious developer mistake with Angular templates
- Clearly articulate why this pattern can lead to bugs or unintended behavior
- Suggest one or more clear solutions
- Have a low, preferably zero, false-positive rate
- Apply to the vast majority of Angular applications (not specific to an unofficial library)
- Improve program correctness or performance (not style, that responsibility falls to a linter)

If you have an idea for an extended diagnostic which fits these criteria, consider filing a [feature request](https://github.com/angular/angular/issues/new?template=2-feature-request.yaml).

---

### Update guide
Source: https://angular.dev/update-guide

# Keeping your Angular projects up-to-date

Just like Web and the entire web ecosystem, Angular is continuously improving.
Angular balances continuous improvement with a strong focus on stability and making updates straightforward.
Keeping your Angular application up-to-date enables you to take advantage of leading-edge new features, as well as optimizations and bug fixes.

This document contains information and resources to help you keep your Angular applications and libraries up-to-date.

For information about our versioning policy and practices — including support and deprecation practices, as well as the release schedule — see [Angular versioning and releases](reference/releases 'Angular versioning and releases').

HELPFUL: If you are currently using AngularJS, see [Upgrading from AngularJS](https://angular.io/guide/upgrade 'Upgrading from AngularJS').
_AngularJS_ is the name for all v1.x versions of Angular.

## Getting notified of new releases

To be notified when new releases are available, follow [@angular](https://x.com/angular '@angular on X') on X (formerly Twitter) or subscribe to the [Angular blog](https://blog.angular.dev 'Angular blog').

## Learning about new features

What's new? What's changed? We share the most important things you need to know on the Angular blog in [release announcements](https://blog.angular.dev/ 'Angular blog - release announcements').

To review a complete list of changes, organized by version, see the [Angular change log](https://github.com/angular/angular/blob/main/CHANGELOG.md 'Angular change log').

## Checking your version of Angular

To check your application's version of Angular use the `ng version` command from within your project directory.

## Finding the current version of Angular

The most recent stable released version of Angular appears [on npm](https://www.npmjs.com/package/@angular/core 'Angular on npm') under "Version." For example, `16.2.4`.

You can also find the most current version of Angular by using the CLI command [`ng update`](cli/update).
By default, [`ng update`](cli/update) (without additional arguments) lists the updates that are available to you.

## Updating your environment and apps

To make updating uncomplicated, we provide complete instructions in the interactive [Angular Update Guide](update-guide).

The Angular Update Guide provides customized update instructions, based on the current and target versions that you specify.
It includes basic and advanced update paths, to match the complexity of your applications.
It also includes troubleshooting information and any recommended manual changes to help you get the most out of the new release.

For simple updates, the CLI command [`ng update`](cli/update) is all you need.
Without additional arguments, [`ng update`](cli/update) lists the updates that are available to you and provides recommended steps to update your application to the most current version.

[Angular Versioning and Releases](reference/releases#angular-versioning 'Angular Release Practices, Versioning') describes the level of change that you can expect based on a release's version number.
It also describes supported update paths.

## Resource summary

- Release announcements:
  [Angular blog - release announcements](https://blog.angular.dev/ 'Angular blog announcements about recent releases')

- Release details:
  [Angular change log](https://github.com/angular/angular/blob/main/CHANGELOG.md 'Angular change log')

- Update instructions:
  [Angular Update Guide](update-guide)

- Update command reference:
  [Angular CLI `ng update` command reference](cli/update)

- Versioning, release, support, and deprecation practices:
  [Angular versioning and releases](reference/releases 'Angular versioning and releases')

---

### Contribute to Angular
Source: https://github.com/angular/angular/blob/main/CONTRIBUTING.md

# Contributing to Angular

We would love for you to contribute to Angular and help make it even better than it is today!
As a contributor, here are the guidelines we would like you to follow:

- [Code of Conduct](#coc)
- [Question or Problem?](#question)
- [Issues and Bugs](#issue)
- [Feature Requests](#feature)
- [Submission Guidelines](#submit)
- [Coding Rules](#rules)
- [Commit Message Guidelines](#commit)
- [Signing the CLA](#cla)

## <a name="coc"></a> Code of Conduct

Help us keep Angular open and inclusive.
Please read and follow our [Code of Conduct][coc].

## <a name="question"></a> Got a Question or Problem?

Do not open issues for general support questions as we want to keep GitHub issues for bug reports and feature requests.
Instead, we recommend using [Stack Overflow](https://stackoverflow.com/questions/tagged/angular) to ask support-related questions. When creating a new question on Stack Overflow, make sure to add the `angular` tag.

Stack Overflow is a much better place to ask questions since:

- there are thousands of people willing to help on Stack Overflow
- questions and answers stay available for public viewing so your question/answer might help someone else
- Stack Overflow's voting system assures that the best answers are prominently visible.

To save your and our time, we will systematically close all issues that are requests for general support and redirect people to Stack Overflow.

If you would like to chat about the question in real-time, you can reach out via [the Angular community Discord server][discord].

## <a name="issue"></a> Found a Bug?

If you find a bug in the source code, you can help us by [submitting an issue](#submit-issue) to our [GitHub Repository][github].
Even better, you can [submit a Pull Request](#submit-pr) with a fix.

## <a name="feature"></a> Missing a Feature?

You can _request_ a new feature by [submitting an issue](#submit-issue) to our GitHub Repository.
If you would like to _implement_ a new feature, please consider the size of the change in order to determine the right steps to proceed:

- For a **Major Feature**, first open an issue and outline your proposal so that it can be discussed.
  This process allows us to better coordinate our efforts, prevent duplication of work, and help you to craft the change so that it is successfully accepted into the project.

  **Note**: Adding a new topic to the documentation, or significantly re-writing a topic, counts as a major feature.

- **Small Features** can be crafted and directly [submitted as a Pull Request](#submit-pr).

## <a name="submit"></a> Submission Guidelines

### <a name="submit-issue"></a> Submitting an Issue

Before you submit an issue, please search the issue tracker. An issue for your problem might already exist and the discussion might inform you of workarounds readily available.

We want to fix all the issues as soon as possible, but before fixing a bug, we need to reproduce and confirm it.
In order to reproduce bugs, we require that you provide a minimal reproduction.
Having a minimal reproducible scenario gives us a wealth of important information without going back and forth to you with additional questions.

A minimal reproduction allows us to quickly confirm a bug (or point out a coding problem) as well as confirm that we are fixing the right problem.

We require a minimal reproduction to save maintainers' time and ultimately be able to fix more bugs.
Often, developers find coding problems themselves while preparing a minimal reproduction.
We understand that sometimes it might be hard to extract essential bits of code from a larger codebase, but we really need to isolate the problem before we can fix it.

Unfortunately, we are not able to investigate / fix bugs without a minimal reproduction, so if we don't hear back from you, we are going to close an issue that doesn't have enough info to be reproduced.

You can file new issues by selecting from our [new issue templates](https://github.com/angular/angular/issues/new/choose) and filling out the issue template.

### <a name="pr-quality"></a> Contribution Quality

We strongly value open source contribution and pull requests from community contributors. Please note that every pull request is reviewed and merged by an actual person on the team, which does take time and effort. That is time and effort that does take away from other valuable work. With that in mind we have a minimum set of expectations that are required of any community contribution pull request that is opened.

1. Search [GitHub](https://github.com/angular/angular/pulls) for an open or closed PR that relates to your submission.
   - You don't want to duplicate existing efforts.
2. Be sure that an issue or pull request clearly describes the problem you're fixing, or documents the design for the feature you'd like to add. Issues require a _minimal_ reproduction.

3. Discussing the design in an issue upfront helps to ensure that we're ready to accept your work. Pull requests are not the right place to do design work.
   - When in doubt, open an issue first before doing any sort of speculative implementation work

4. Ideally the PR should be tied to an issue, but this is not required

5. The change should improve code quality (i.e. addressing a TODO) or should impact / improve a feature

6. Micro optimizations will only be accepted if they are validated by an actual benchmark

7. Do not open pull requests that are addressing feature requests that are not labeled as "help wanted" as they usually need additional design work before we could accept pull requests

8. The change should be well tested

If your pull request does not meet these minimum expectations, we may close your PR. Also, if your PR introduces a breaking change, it's possible the level of churn this breaking change causes may block our ability to move forward with it. We may close your PR in that situation, as well. Otherwise, we're excited to see your contributions and enthusiasm for Angular!

### <a name="submit-pr"></a> Submitting a Pull Request (PR)

Before you submit your Pull Request (PR) consider the following guidelines:

1. Please sign our [Contributor License Agreement (CLA)](#cla) before sending PRs.
   We cannot accept code without a signed CLA.
   Make sure you author all contributed Git commits with email address associated with your CLA signature.

2. [Fork](https://docs.github.com/en/github/getting-started-with-github/fork-a-repo) the [angular/angular](https://github.com/angular/angular/fork) repo.

3. In your forked repository, make your changes in a new git branch:

   ```shell
   git checkout -b my-fix-branch main
   ```

4. Create your patch, **including appropriate test cases**.

5. Follow our [Coding Rules](#rules).

6. Run the full Angular test suite, as described in the [developer documentation][dev-doc], and ensure that all tests pass.

7. Commit your changes using a descriptive commit message that follows our [commit message conventions][commit-message-guidelines].
   Adherence to these conventions is necessary because release notes are automatically generated from these messages.

   ```shell
   git commit --all
   ```

   Note: the optional commit `--all` command line option will automatically "add" and "rm" edited files.

8. Push your branch to GitHub:

   ```shell
   git push origin my-fix-branch
   ```

9. In GitHub, send a pull request to `angular:main`.

### Reviewing a Pull Request

The Angular team reserves the right not to accept pull requests from community members who haven't been good citizens of the community. Such behavior includes not following the [Angular code of conduct](https://github.com/angular/code-of-conduct) and applies within or outside of Angular managed channels.

#### Addressing review feedback

If we ask for changes via code reviews then:

1. Make the required updates to the code.

2. Re-run the Angular test suites to ensure tests are still passing.

3. Create a fixup commit and push to your GitHub repository (this will update your Pull Request):

   ```shell
   git commit --all --fixup HEAD
   git push
   ```

   For more info on working with fixup commits see [here](./contributing-docs/using-fixup-commits.md).

That's it! Thank you for your contribution!

##### Updating the commit message

A reviewer might often suggest changes to a commit message (for example, to add more context for a change or adhere to our [commit message guidelines][commit-message-guidelines]).
In order to update the commit message of the last commit on your branch:

1. Check out your branch:

   ```shell
   git checkout my-fix-branch
   ```

2. Amend the last commit and modify the commit message:

   ```shell
   git commit --amend
   ```

3. Push to your GitHub repository:

   ```shell
   git push --force-with-lease
   ```

> NOTE:<br />
> If you need to update the commit message of an earlier commit, you can use `git rebase` in interactive mode.
> See the [git docs](https://git-scm.com/docs/git-rebase#_interactive_mode) for more details.

#### After your pull request is merged

After your pull request is merged, you can safely delete your branch and pull the changes from the main (upstream) repository:

- Delete the remote branch on GitHub either through the GitHub web UI or your local shell as follows:

  ```shell
  git push origin --delete my-fix-branch
  ```

- Check out the main branch:

  ```shell
  git checkout main -f
  ```

- Delete the local branch:

  ```shell
  git branch -D my-fix-branch
  ```

- Update your local `main` with the latest upstream version:

  ```shell
  git pull --ff upstream main
  ```

## <a name="rules"></a> Coding Rules

To ensure consistency throughout the source code, keep these rules in mind as you are working:

- All features or bug fixes **must be tested** by one or more specs (unit-tests).
- All public API methods **must be documented**.
- We follow [Google's TypeScript Style Guide][ts-style-guide], but wrap all code at **100 characters**.

  An automated formatter is available, see [building-and-testing-angular.md](./contributing-docs/building-and-testing-angular.md#formatting-your-source-code).

## <a name="commit"></a> Commit Message Guidelines

We have very precise rules over how our Git commit messages must be formatted:

```
```

See [Commit Message Guidelines][commit-message-guidelines] for details.

## <a name="cla"></a> Signing the CLA

Please sign our Contributor License Agreement (CLA) before sending pull requests. For any code
changes to be accepted, the CLA must be signed. It's a quick process, we promise!

- For individuals, we have a [simple click-through form][individual-cla].
- For corporations, we'll need you to
  [print, sign and one of scan+email, fax or mail the form][corporate-cla].

If you have more than one GitHub account, or multiple email addresses associated with a single GitHub account, you must sign the CLA using the primary email address of the GitHub account used to author Git commits and send pull requests.

The following documents can help you sort out issues with GitHub accounts and multiple email addresses:

- https://help.github.com/articles/setting-your-commit-email-address-in-git/
- https://stackoverflow.com/questions/37245303/what-does-usera-committed-with-userb-13-days-ago-on-github-mean
- https://help.github.com/articles/about-commit-email-addresses/
- https://help.github.com/articles/blocking-command-line-pushes-that-expose-your-personal-email-address/

[coc]: https://github.com/angular/code-of-conduct/blob/main/CODE_OF_CONDUCT.md
[corporate-cla]: https://cla.developers.google.com/about/google-corporate
[dev-doc]: ./contributing-docs/building-and-testing-angular.md
[commit-message-guidelines]: ./contributing-docs/commit-message-guidelines.md
[github]: https://github.com/angular/angular
[discord]: https://discord.gg/angular
[individual-cla]: https://cla.developers.google.com/about/google-individual
[ts-style-guide]: https://google.github.io/styleguide/tsguide.html

---

### Angular's Roadmap
Source: https://angular.dev/roadmap

Learn how the Angular team is building momentum on the web.

As an open source project, Angular’s daily commits, PRs and momentum is all trackable on GitHub. To increase transparency into how this daily work connects to the framework’s future, our roadmap brings together the team’s current and future planned vision.

The following projects are not associated with a particular Angular version. We will release them on completion, and they will be part of a specific version based on our release schedule, following semantic versioning. For example, we release features in the next minor after completion or the next major if they include breaking changes.

Currently, Angular has three goals for the framework:

1. Improve the [AI experience for developers](/ai)
1. Improve the [Angular developer experience](#improving-the-angular-developer-experience)
1. Improve the framework’s performance

Continue reading to learn how we plan to deliver these objectives with specific project work.

## Explore modern Angular

Start developing with the latest Angular features from our roadmap. This list represents the current status of new features from our roadmap:

### Available to experiment with

- [Web MCP](/ai/webmcp)

### Production ready

- [Signal Forms](/guide/forms/signals/overview)
- [Resource API](/guide/signals/resource)
- [httpResource](/api/common/http/httpResource)
- [Zoneless change detection](/guide/zoneless)
- [Linked Signal API](/guide/signals/linked-signal)
- [Incremental hydration](/guide/incremental-hydration)
- [Effect API](/api/core/effect)
- [Event replay with SSR](/api/platform-browser/withEventReplay)
- [Route-level render mode](/guide/ssr)

## Improving the AI experience for Angular Developers

### Bringing the best of AI to Angular

  AI continues to shape the development landscape. It has changed how we develop apps and the types of user experiences that are possible. We plan to best support the developer community in AI-assisted coding and integrating AI in their applications.
  The team will continue to develop meaningful integrations with tools like Google AI Studio, Gemini CLI and other agentic tooling such as Agentic IDEs like Antigravity. We plan to launch solutions that stay in line with the rapidly evolving industry. Some examples include agent skills, new MCP features and AI SDKs.
  [Based on our research](https://blog.angular.dev/beyond-the-horizon-how-angular-is-embracing-ai-for-next-gen-apps-7a7ed706e1a3), code generation for Angular is already high quality with modern LLMs. We will continue our investments in improving code generation for Angular. This means we will regularly evaluate code generation quality using current models and work to improve it through system instructions, documentation, and tactical framework changes.
  There is a new frontier for Angular developers to explore with new concepts such as Dynamic UI generation. We started with building Angular support for A2UI and are actively searching out more opportunities to support modern application experiences.

## Improving the Angular developer experience

### Developer velocity

    Microsoft has spent the last year porting the TypeScript compiler to Go, with the promise of a 5 - 10x speedup for typical TypeScript compilations. Angular has perhaps one of the deepest integrations with the TypeScript compiler, which will require bigger architectural changes to support new tsgo-based workflows for both the compiler and language service.

We're in the process of prototyping and exploring what this support would look like, and will deliver an Angular compiler that is compatible with tsgo and brings the performance benefits of Microsoft's native port to the Angular ecosystem.

    Developers are mixing AI generated code with manually generated code, and want to make use of popular libraries and integrate new experiences quickly. Angular wants to integrate well into that ecosystem - developers should be able to use the tools they love and mix and match frameworks according to their requirements.

As part of this project, we'll explore the requirement space of cross framework interop and our build tooling to improve our compatibility.


  In Angular v21, we launched Angular Aria in developer preview, providing eight patterns for accessible, headless components. In Angular v22, we promoted Angular Aria to stable, and we plan to introduce new patterns where needed. We want to provide developers with a solid foundation for developing their own components using Angular Aria - we provide the interactions and you bring the style that matches your design systems. Developers will have the choice of developing custom components with Angular Aria, use interaction patterns from the CDK, or use ready-made styled Material Components.

For accessibility, we are continuously evaluating the components and patterns against accessibility standards such as WCAG and are working to fix any issues that arise from this process.

### Improve tooling

  Following the stable release of Vitest in Angular v21, it is now our primary test runner. We are now focused on promoting our experimental Karma to Vitest migration tool to stable as well as investigating new features to further refine and improve the developer testing workflow.

## Completed projects


  Signal Forms are now stable. This new approach allows developers to manage form state using signals, providing an ergonomic forms creation experience. We promoted Signal Forms to stable and enhanced interoperability with reactive forms - enabling teams to progressively migrate large forms at their own pace.

  We introduced new signal APIs, `resource()` and `httpResource()`, for flexible asynchronous data handling. We promoted these APIs to developer stable.

  With Zoneless being stable and default, we set the default change detection strategy to `OnPush`, to follow current best practices. We also renamed `ChangeDetectionStrategy.Default` to `ChangeDetectionStrategy.Eager`.

[See the RFC discussion for details](https://github.com/angular/angular/discussions/66779).

  We've added better tooling for debugging Signals using Angular DevTools. This change includes a new UI for inspecting and debugging signals.

  We're working towards faster edit/refresh cycle by enabling hot module replacement. In Angular v19 we shipped initial support for CSS and template HMR and in v20 we graduated template HMR to stable. We'll continue collecting feedback to make sure we're addressing developers' needs before we mark this project as complete.

In v18 we shipped experimental zoneless support in Angular. It enables developers to use the framework without including zone.js in their bundle, which improves performance, debugging experience, and interoperability. As part of the initial release we also introduced zoneless support to the Angular CDK and Angular Material.

In v19 we introduced zoneless support in server-side rendering, addressed some edge cases, and created a schematic to scaffold zoneless projects. We transitioned <a href="https://fonts.google.com/" target="_blank">Google Fonts</a> to zoneless which improved performance, developer experience, and allowed us to identify gaps that we need to address before moving this feature to developer preview.

As of Angular v20.2, Zoneless Angular is now stable and includes improvements in error handling and server-side rendering.

We're working towards enabling a more ergonomic route configuration on the server. We want to make it trivial to declare which routes should be server-side rendered, prerendered or client-side rendered.

In Angular v19 we shipped developer preview of route-level render mode which allows you to granularly configure which routes you want Angular to prerender, server-side render or client-side render. In Angular v20 we graduated it to stable.
In v17 we graduated hydration from developer preview and we've been consistently observing 40-50% improvements in LCP. Since then we started prototyping incremental hydration and shared a demo on stage at ng-conf.

In v19 we shipped the incremental hydration in developer preview mode, powered by `@defer` blocks. In Angular v20 we graduated it to stable!
This project rethinks the Angular reactivity model by introducing Signals as a reactivity primitive. The initial planning resulted in hundreds of discussions, conversations with developers, feedback sessions, user experience studies, and a series of RFCs, which received over 1,000 comments.

In Angular v20 we graduated all the fundamental reactivity primitives to stable including signal, effect, linkedSignal, signal-based queries, and inputs.
As part of this project, we implemented mixed orientation support for the Angular CDK drag and drop. This is one of the repository's most highly requested features.
In v18 we introduced an event replay functionality when using server-side rendering or prerendering. For this feature we depend on the event dispatch primitive (previously known as jsaction) that is running on Google.com.

In Angular v19 we graduated event replay to stable and enabled it by default for all new projects.
To make it easier for developers to use modern Angular APIs, we enabled integration between the Angular language service and schematics which allows you to refactor your app with a single click.
As part of this initiative, the language service automatically imports components and pipes in standalone and NgModule-based apps. Additionally, we've added a template diagnostic to highlight unused imports in standalone components, which should help make application bundles smaller.
We've released the support for local template variables in Angular, see [`@let`](/api/core/@let) for additional information.
To provide better customization of our Angular Material components and enable Material 3 capabilities, we'll be collaborating with Google's Material Design team on defining token-based theming APIs.

In v17.2 we shared experimental support for Angular Material 3 and in v18 we graduated it to stable.
In v17 we shipped deferrable views in developer preview, which provide an ergonomic API for deferred code loading. In v18 we enabled deferrable views for library developers and graduated the API to stable.
We enabled debugging and profiling of Angular apps embedded within an iframe on the page.
In v17 we shipped a vite and esbuild-based application builder and enabled it for new projects by default. It improves build time for projects using hybrid rendering with up to 87%. As part of v18 we shipped schematics and a guide that migrate existing projects using hybrid rendering to the new build pipeline.
Angular.dev is the new site, domain and home for Angular development. The new site contains updated documentation, tutorials and guidance that will help developers build with Angular’s latest features.
In v17 we shipped a developer preview version of a new control flow. It brings significant performance improvements and better ergonomics for template authoring. We also provided a migration of existing `*ngIf`, `*ngFor`, and `*ngSwitch` which you can run to move your project to the new implementation. As of v18 the built-in control flow is now stable.
Over the past two quarters, we developed a new [video](https://www.youtube.com/watch?v=xAT0lHYhHMY&list=PL1w1q3fL4pmj9k1FrJ3Pe91EPub2_h4jF) and [textual](/tutorials/learn-angular) tutorial based on standalone components.
In Angular v16, we released a developer preview of an esbuild-based builder with support for `ng build` and `ng serve`. The `ng serve` development server uses Vite and a multi-file compilation by esbuild and the Angular compiler. In v17 we graduated the build tooling from developer preview and enabled it by default for new projects.
To improve the debugging utilities of Angular and Angular DevTools, we'll work on APIs that provide access to the dependency injection runtime. As part of the project, we'll expose debugging methods that allow us to explore the injector hierarchy and the dependencies across their associated providers. As of v17, we shipped a feature that enables us to plug into the dependency injection life-cycle. We also launched a visualization of the injector tree and inspection of the providers declared inside each individual node,
We released a developer preview of the `ng new --standalone` schematics collection, allowing you to create apps free of NgModules. In v17 we switched the new application authoring format to standalone APIs and changed the documentation to reflect the recommendation. Additionally, we shipped schematics which support updating existing applications to standalone components, directives, and pipes. Even though NgModules will stick around for foreseeable future, we recommend you to explore the benefits of the new APIs to improve developer experience and benefit from the new features we build for them.
In v16, we released a developer preview of non-destructive full hydration, see the [hydration guide](guide/hydration) and the [blog post](https://blog.angular.dev/whats-next-for-server-side-rendering-in-angular-2a6f27662b67) for additional information. We're already seeing significant improvements to Core Web Vitals, including [LCP](https://web.dev/lcp) and [CLS](https://web.dev/cls). In lab tests, we consistently observed 45% better LCP of a real-world app.

In v17 we launched hydration outside developer preview and did a series of improvements in the server-side rendering story, including: route discovery at runtime for SSG, up to 87% faster build times for hybrid rendered applications, prompt that enables hybrid rendering for new projects.
In v16, we released a developer preview of non-destructive full hydration, which allows Angular to reuse existing DOM nodes on a server-side rendered page, instead of re-creating an app from scratch. See additional information in the hydration guide.
We released the Angular image directive as stable in v15. We introduced a new fill mode feature that enables images to fit within their parent container rather than having explicit dimensions. Over the past two months, the Chrome Aurora team backported the directive to v12 and newer.
Ensure all existing documentation fits into a consistent set of content types. Update excessive use of tutorial-style documentation into independent topics. We want to ensure the content outside the main tutorials is self-sufficient without being tightly coupled to a series of guides. In Q2 2022, we refactored the template content and dependency injection. In Q1 2023, we improved the HTTP guides, and with this, we're putting the documentation refactoring project on hold.
The Aurora and the Angular teams are working on the implementation of an image directive that aims to improve Core Web Vitals. We shipped a stable version of the image directive in v15.
The Web ecosystem evolves constantly and we want to reflect the latest modern standards in Angular. In this project we aim to provide guidelines on using modern CSS features in Angular to ensure developers follow best practices for layout, styling, etc. We shared official guidelines for layout and as part of the initiative stopped publishing flex layout.
A long-standing feature request is to add the ability to add directives to host elements. The feature lets developers augment their own components with additional behaviors without using inheritance. In v15 we shipped our directive composition API, which enables enhancing host elements with directives.
The Angular and the Chrome DevTools are working together to enable more readable stack traces for error messages. In v15 we released improved relevant and linked stack traces. As a lower priority initiative, we'll be exploring how to make the stack traces friendlier by providing more accurate call frame names for templates.
MDC Web is a library created by the Google Material Design team that provides reusable primitives for building Material Design components. The Angular team is incorporating these primitives into Angular Material. Using MDC Web aligns Angular Material more closely with the Material Design specification, expands accessibility, improves component quality, and improves the velocity of our team.
In the process of making Angular simpler, we are working on introducing APIs that allow developers to initialize apps, instantiate components, and use the router without NgModules. Angular v14 introduces developer preview of the APIs for standalone components, directives, and pipes. In the next few quarters we'll collect feedback from developers and finalize the project making the APIs stable. As the next step we will work on improving use cases such as TestBed, Angular elements, etc.
To improve the encapsulation of Angular components we enabled binding to protected members of the component instance. This way you'll no longer have to expose a field or a method as public to use it inside your templates.
Develop and publish an in-depth guide on change detection. Develop content for performance profiling of Angular apps. Cover how change detection interacts with Zone.js and explain when it gets triggered, how to profile its duration, as well as common practices for performance optimization.
In Q4 2021 we designed a solution for introducing strict typings for forms and in Q1 2022 we concluded the corresponding request for comments. Currently, we are implementing a rollout strategy with an automated migration step that will enable the improvements for existing projects. We are first testing the solution with more than 2,500 projects at Google to ensure a smooth migration path for the external community.
After the transition of all our internal tooling to Ivy is completed, we will remove the legacy View Engine for reduced Angular conceptual overhead, smaller package size, lower maintenance cost, and lower codebase complexity.
To simplify the Angular mental model and learning journey, we will be working on making NgModules optional. This work lets developers develop standalone components and implement an alternative API for declaring the compilation scope of the component. We kicked this project off with high-level design discussions that we captured in an RFC.
We will work on finding a way to implement stricter type checking for reactive forms with minimal backward incompatible implications. This way, we let developers catch more issues during development time, enable better text editor and IDE support, and improve the type checking for reactive forms.
To improve the integration of Angular DevTools with the framework, we are working on moving the codebase to the angular/angular monorepository. This includes transitioning Angular DevTools to Bazel and integrating it into the existing processes and CI pipeline.
Extend the diagnostics of the Angular compiler outside type checking. Introduce other correctness and conformance checks to further guarantee correctness and best practices.
To ensure we provide a future-proof e2e testing strategy, we want to evaluate the state of Protractor, community innovations, e2e best practices, and explore novel opportunities. As first steps of the effort, we shared an RFC and worked with partners to ensure smooth integration between the Angular CLI and state-of-the-art tooling for e2e testing. As the next step, we need to finalize the recommendations and compile a list of resources for the transition.
Earlier in 2020, we shared an RFC for Ivy library distribution. After invaluable feedback from the community, we developed a design of the project. We are now investing in the development of Ivy library distribution, including an update of the library package format to use Ivy compilation, unblock the deprecation of the View Engine library format, and ngcc.
To improve test time and create better isolation across tests, we want to change TestBed to automatically clean up and tear down the test environment after each test run.
Internet Explorer 11 (IE11) has been preventing Angular from taking advantage of some of the modern features of the Web platform. As part of this project we are going to deprecate and remove IE11 support to open the path for modern features that evergreen browsers provide. We ran an RFC to collect feedback from the community and decide on next steps to move forward.
Supporting modern browsers lets us take advantage of the more compact, expressive, and performant new syntax of JavaScript. As part of this project we will investigate what the blockers are to moving forward with this effort, and take the steps to enable it.
We are working on development tooling for Angular that provides utilities for debugging and performance profiling. This project aims to help developers understand the component structure and the change detection in an Angular app.
We want to consolidate release management tooling between the multiple GitHub repositories for Angular (angular/angular, angular/angular-cli, and angular/components). This effort lets us reuse infrastructure, unify and simplify processes, and improve the reliability of our release process.
We want to unify commit message requirements and conformance across Angular repositories (angular/angular, angular/components, and angular/angular-cli) to bring consistency to our development process and reuse infrastructure tooling.
The goal of this project is to improve the experience and remove legacy dependency by transitioning the language service to Ivy. Today the language service still uses the View Engine compiler and type checking, even for Ivy apps. We want to use the Ivy template parser and improved type checking for the Angular Language service to match app behavior. This migration is also a step towards unblocking the removal of View Engine, which will simplify Angular, reduce the npm package size, and improve the maintainability of the framework.
In collaboration with the Google security team, we are adding support for the new Trusted Types API. This web platform API helps developers build more secure web apps.
As part of the v11 release, we introduced an opt-in preview of webpack 5 in the Angular CLI. To ensure stability, we will continue iterating on the implementation to enable build speed and bundle size improvements.
Loading external stylesheets is a blocking operation, which means that the browser cannot start rendering your app until it loads all the referenced CSS. Having render-blocking resources in the header of a page can significantly impact its load performance, for example, its first contentful paint. To make apps faster, we have been collaborating with the Google Chrome team on inlining critical CSS and loading the rest of the styles asynchronously.
Error messages often bring limited actionable information to help developers resolve them. We have been working on making error messages more discoverable by adding associated codes, developing guides, and other materials to ensure a smoother debugging experience.
We will redefine the user learning journeys and refresh the introductory documentation. We will clearly state the benefits of Angular, how to explore its capabilities and provide guidance so developers can become proficient with the framework in as little time as possible.
Angular CDK introduced the concept of component test harnesses to Angular in version 9. Test harnesses let component authors create supported APIs for testing component interactions. We are continuing to improve this harness infrastructure and clarifying the best practices around using harnesses. We are also working to drive more harness adoption inside of Google.
Content projection is a core Angular concept that does not have the presence it deserves in the documentation. As part of this project we want to identify the core use cases and concepts for content projection and document them.
With the deprecation of TSLint we will be moving to ESLint. As part of the process, we will work on ensuring backward compatibility with our current recommended TSLint configuration, implement a migration strategy for existing Angular apps and introduce new tooling to the Angular CLI toolchain.
We are actively investing up to 50% of our engineering capacity on triaging issues and PRs until we have a clear understanding of broader community needs. After that, we will commit up to 20% of our engineering capacity to keep up with new submissions promptly.

---

