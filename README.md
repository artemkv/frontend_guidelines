v0.3.3

Disclaimer: you can treat this document as my personal opinion. You don't have to agree with this, and feel free to completely disregard all I say. I am looking to be right, my aim is to share.

# Intro

Building complex front-end applications is a complex task. Fortunately, there has been a lot of advancements in this area and many patterns emerged that helped to reduce this complexity, maintenance cost, and, as a result, a number of bugs.

Today, many frontend developers start their journey in a particular framework/library, such as _React_, _Angular_ etc. Working with these frameworks and libraries, they learn library- or framework-specific patterns and practices (e.g. React hooks) that, while useful, do not always provide a solid fundamentals for understanding the bigger picture and underlying principles.

This document aims at providing a background for reasoning about the frontend application architecture in a framework-independent way.

When building a frontend application, most of the (technical) complexity lies in **(1)** handling UI component dependencies (as in "this view should update when that button is clicked") and **(2)** reconciling state updates triggered by user actions and business logic.

UI component dependencies usually obey "if and only if" logic:

- Necessary condition: "component `A` should only update when component `B` changes";
- Sufficient condition: "if component `B` changes, component `A` must get updated".

If you don't respect these rules, you will either have UI that updates too often, resulting in flickering and poor UX; or too seldom, resulting in stale, out-of-sync views.

To make things worse, you have multiple sources of updates: application business logic and user actions. In a poorly designed frontend application or framework you may encounter the situation when data flows in both directions: from app business logic to the UI component and from the UI component to the app business logic, which easily gets out of control and breeds bugs related to inconsistent state. The notorious example is the Facebook messenger app that would show "X unread messages" while there was, in fact, none.

In order to solve this challenges, the declarative approach has emerged and gained a lot of popularity in recent years ("here is my desired state, please update the DOM to match it").

I don't think this is the only way to build frontend applications and you could definitely build a solid application using imperative approach. However, since the modern libraries like React are fundamentally build upon declarative paradigm, I feel it's absolutely vital to understand its core principles. The worst idea, in my opinion, would be to try and combine imperative and declarative approach (e.g. using React components to derive the UI from props while relying on the mental model "component `A` should update the component `B`").

# Principles

Solving the challenges of complex frontend applications (within the declarative paradigm) relies on following principles:

- Unidirectional flow of data
- Pure views
- Global, consistent, and immutable state (aka _Model_)
- Pure state update function (aka _Reducer_)
- Isolated and well controlled side effects

The good place to start is [Elm architecture](https://guide.elm-lang.org/architecture/). _Elm_ is a purely functional language that resembles _Haskell_ and the architecture it promotes emerges from its purely functional nature.

_Elm_ architecture is not just a fun reading. It has fundamental practical and historical importance, as it has inspired libraries like _Flux_ ([React and Flux: Building Applications with a Unidirectional Data Flow](https://youtu.be/i__969noyAM?si=FaQOBg7dHC7wzIbl), [Hacker Way: Rethinking Web App Development at Facebook](https://youtu.be/nYkdrAPrdcw?si=Z525QDkB8XQRweQn&t=610)), _Redux_ (see [Redux Essentials](https://redux.js.org/tutorials/essentials/part-1-overview-concepts)) and, consequently, state management in _React_ (see [Managing State](https://react.dev/learn/managing-state)), _Angular_ (see [@ngrx/store](https://ngrx.io/guide/store)) and others. In my view, it is "The architecture" for the frontend app and the idea behind the modern frontend libraries. A lot of differences boil down to how different frameworks name things and where they plug effects (and how they pervert the pure Elm architecture).

You don't have to follow the _Elm_ architecture strictly, as it can sometimes feel too rigid, but it's important to know when you deviate from this architecture, and be able to articulate the advantages and drawbacks of an alternative solution. Similarly, it is important to know how the specific techniques (e.g. hooks, state selectors) fit (or don't) into this architecture.

## Unidirectional flow of data

The resources mentioned in the previous section do a very good job explaining this concept, so here I will just briefly mention the main flow:

```
Current State -> View -> Events -> Update -> New State
```

- The **state** is the model of your app;
- This state flows into the **views**;
- The views can trigger **events** (aka _actions_, or _messages_);
- The events bubble to the **update function**;
- The update produces the **new state**;
- The new state flows into **new views** (that need to be re-rendered).

While this looks like a loop, it is actually a spiral (which you can unwind along the timeline): nothing is ever mutated, only the new stuff is created; and all the arrows are one-way (no loops).

We will now look at all the parts in more details.

## Pure views

Also known as _presentational components_ or _pure components_.

The pure components are pure functions that receive the fragments of **state** as an input and return (virtual) **DOM elements** as an output.

Example with _React_:

```js
function Welcome(props) {
  return (
    <div>
      <h1>Hello, {props.name}</h1>
      <button onclick={props.clicked_hello}>Say hello</button>
    </div>
  );
}
```

React guidelines, in fact, promote [Keeping Components Pure](https://react.dev/learn/keeping-components-pure), and the [StrictMode](https://react.dev/reference/react/StrictMode) is designed to detect impure functions.

From the [React documentation, Fixing bugs found by double rendering in development](https://react.dev/reference/react/StrictMode#fixing-bugs-found-by-double-rendering-in-development):

> React assumes that every component you write is a pure function. This means that React components you write must always return the same JSX given the same inputs (props, state, and context).

Making views out of pure functions brings many advantages:

- Easy to unit-test, if you desire to do so (allows input-output test, although, personally, I would probably not find these tests very useful);
- Easy to understand, contains minimal logic (ideally, no logic at all);
- Easy to develop in isolation, e.g. using tools like _Storybook_ (only requires providing inputs for rendering);
- Allows results to be cached as long as the inputs remain the same.

The last statement is really important in order to avoid unnecessary re-rendering. In case of _Elm_, the purity is ensured by the compiler, so you can always cache the component as long as the inputs stay the same. In case of _TypeScript_ and _React_, you, as a developer, have the responsibility to keep the components pure and to ensure the outputs are cached (although React is doing some optimizations internally, to avoid unnecessary re-renderings).

In React, use [`memo`](https://react.dev/reference/react/memo) to cache components. The common wisdom is not to wrap every component in `memo`, just the ones that are heavy.

```js
import { memo } from "react";

const WelcomeMemoed = memo(function Welcome(props) {
  return (
    <div>
      <h1>Hello, {props.name}</h1>
      <button onclick={props.clicked_hello}>Say hello</button>
    </div>
  );
});
```

However, the React only keep around the most recent value of the input and result, so if you are following the guideline and create pure components, it is totally safe to wrap all of them in `memo` by default.

### Container components

If, for any reason, you feel that you need to push the state management on a component itself (see below on state), do not mix everything in one place: create a purely representational component (no state management, only markup) and a container component (manages state but has no markup).

## Global, consistent, and immutable state

Also known as _model_.

The state can be hosted in the _Redux store_ or simply in the root app component (using [reducer hook](https://react.dev/reference/react/useReducer) or even simply a [state hook](https://react.dev/reference/react/useState), in case of _React_). It doesn't really matter (and you could write your own Redux library in about half an hour).

What is important:

- State is the most important part of your application. The model of the state represents your application business domain and provides the [**Ubiquitous Language**](https://martinfowler.com/bliki/UbiquitousLanguage.html), i.e. the common vocabulary for the team;
- State must be always consistent, so you should design it in a way that renders it impossible to construct an illegal/inconsistent state (e.g. no null or undefined properties);
- State must be immutable. The only way to update the state of the application should be by creating a new state from the old state; this, in turn, would require re-executing views to generate new DOM, and update the parts of DOM that have changed, this is a task of a runtime;
- State flows down into views;

### Always consistent state

One of the most important properties of the state is to be always consistent. I highly recommend [Domain-Driven Design](https://www.amazon.com/gp/product/0321125215) by Eric Evans.

The goal is to [Make Impossible States Impossible](https://www.youtube.com/watch?v=IcgmSRJHu_8). There is a fantastic series of articles [The "Designing with types" series](https://fsharpforfunandprofit.com/series/designing-with-types/) that explores the topic in the context of F# language. Finally, there is also a nice talk from Elm conference [The life of a file](https://www.youtube.com/watch?v=XpDsk374LDE).

Let's consider an example.

One of the common tasks in a typical frontend application is loading of the data that needs to be displayed.

Many developers would come up with something as follows:

```ts
enum DataLoadingState {
  Loading,
  Success,
  Error,
}

interface ComponentState {
  loadingState: DataLoadingState;
  result?: Data;
  error?: string;
}
```

Naturally, when `loadingState` is `Success`, you should expect `result` to be filled, and when `loadingState` is `Error`, the `error` field should be filled.

This sounds logical and simple, but this is very problematic.

- This model just invites inconsistencies. There is nothing that prevents me to set both `result` and `error` or none of those. Consequentially, the task of validating the state and handling inconsistent state fall on the rest of the application;
- The dependencies between properties might be obvious in such a small example, but, as your state grows, it may become completely unclear which properties depend on each other, it is simply impossible to say what is the valid state looking at the data type;
- The complexity of the state grows, as you need to declare every property that may ever be non-null on the same data type.

Fortunately, there is a simple solution that relies on discriminated unions:

```ts
enum DataLoadingState {
  Loading,
  Success,
  Error,
}

interface Loading {
  loadingState: DataLoadingState.Loading;
}

interface Error {
  loadingState: DataLoadingState.Error;
  error: string;
}

interface Sucess {
  loadingState: DataLoadingState.Success;
  result: Data;
}

type ComponentState = Loading | Error | Sucess;
```

Notice that we have completely eliminated all the issues mentioned above.

TODO: use generics, i.e. `Error<T>` and `Success<T>`

### Single global state

It is natural for developers to tame complexity by splitting complex things in multiple simple things. This is why the advise to have a single global application state may seem completely crazy. After all, didn't we invent OOP just to avoid global variables?

That is correct, the global variables are evil, but global variables are mutable: any part of the application can modify the value of a global variable at any moment.

This is not the case with the approach we are discussing here, where the state is immutable and creation of a new state is localized inside of a _state update function_ (see below).

Having the complete application state in a single place has a great advantage. In many frontend applications, the components are highly dependent. Right now, as I'm typing in the Visual Code editor window, the status bar, a completely separate component, is updating the line length while preview on the right is showing rendered markdown. If we made the text to be stored on a level of an editor component, we would have a hard time keeping the rest of UI in sync.

This is why React is suggesting [lifting state up](https://react.dev/learn/sharing-state-between-components). I find this confusing, as in my opinion, it suggests that the local state should be a default. In fact, some React docs (see [memo](https://react.dev/reference/react/memo)) directly says that: _prefer local state and don’t lift state up any further than necessary_.

Here is where I disagree with React. I would rather advocate for having **all the state on top by default**, and exceptionally pushing the state down, when it's really needed (e.g. when parts of the application UI are truly independent and can be seen as separate application; the rule of thumb: if you can imagine it as a separate frame, then it can be a new state root; see also: _ephemeral state_).

Having a big part of an application state spread across low-level components, with parts of it bubbled up to various levels of the component depth, possibly up to the root, means you completely give up on having a single consistent view on your application state. In fact, you completely lose track of what constitutes your state. And just like a state, there is a good chance in that case that all the business logic of state update is also spread across all components.

#### Ephemeral state

Some of your component state can be seen as ephemeral, i.e. truly belonging only a single component. Examples of such state are: text, as you are typing it inside of a form input field, until "submit" button is clicked. Another example: grid view sorting. Keeping this state at the top sometimes does feel unnecessary.

If you truly believe you have encountered one of those situations, it's up to you to make the decision to exceptionally push the state down.

### Note on state complexity

Another argument against having all the state in a single state tree might be a complexity of that tree ("would get too big").

However, I would argue that, if you follow the approach explained in _"Always consistent state"_, your state tree, at any point in time, will only include the properties that are valid and required by the currently displayed UI components.

In other words, your state tree will never get more complex than a UI at any single point in time during the application execution, which is the right level of complexity.

### Note on prop drilling

Prop drilling problem is not unique to UI components, it often manifests itself in the OOD (it is one of the factors, albeit not the most important, behind the idea of DI libraries). I personally see many advantages in passing every piece of data explicitly, as this makes dependencies very obvious. But I certainly understand how this can become tedious.

Before you resort to using some techniques like [React Context](https://react.dev/learn/passing-data-deeply-with-context), I do encourage you to see if there is no other way to handle the situation. Does it make sense to have so many nested levels of components? Can you group properties frequently passed together on the single object, to make it easier to do?

I think it's OK to use React Context to pass the things like current locale, current user etc. However, it also think it's OK to group those properties on a Context interface and allow it to drill.

### Note on reusability

I've seen people advocating against using props as this makes components "less reusable" and more fragile in case of UI refactorings. They would argue in favor of `<TodoItem />` comparing to `<TodoItem item={item} />`.

The idea seem to be that you could move such component anywhere inside the app, and the app would not break, while the component accepting props would require re-wiring those props at the new place, which requires more work. This is all true.

Naturally, the component still needs to get the data somehow, and if you don't use props, this means the component itself needs to be smart enough to go and fetch the required piece of the state from the state store. In practice, it means that seemingly very simple `<TodoItem />` component is actually a monster that has its tentacles reaching out far beyond itself, into the places you are completely unaware of.

And while it's true that you can cut and paste such a component anywhere _inside the app_, I struggle to call it re-use since everything breaks as long as you try to re-use such component in a different context. And the most obvious example would be the context of unit tests. We all have seen the components that require pages of mock setup to be able to test the most primitive logic.

A component that take all of its input from props can be truly re-used anywhere, inside or outside of an app.

### Anti-corruption layer

This is simple: basically, don't throw the data you don't control directly on your UI components, especially when retrieving the data from the schema-less databases (e.g. _DynamoDB_).

This means: process the data before putting it into the global store, namely:

- Validate the data. All the fields that are mandatory have to be present and in the correct format;
- Sanitize the data. For example, if some of the properties are optional, they might be or not present in the JSON, you could make sure the property is always there, but maybe set to null;
- Impute the missing values: use app defaults, when applicable;
- Convert into app internal format (e.g. parse datetime from string to a number);
- Versioning: convert the records from any version to a recent canonical format.

## Pure state update function (aka Reducer)

Views may interact with the user, and report the events up. The exact mechanism is irrelevant (callback props in _React_, _actions_ in _Redux_, _messages_ in _Elm_ etc.). You can make events bubble through components or dispatch events from low-level components directly to _update_ function using _dispatch_ function.

What is important:

- The event should be self-contained, i.e. carry all the relevant information, such as id of an element selected, text entered etc.;
- Events flow from the views up;
- Events should be processed in a single place, usually called _reducer_ or _update_, a pure function that accepts the event/action + current state and returns the new state;
- All the data dependencies should be handled here (update field `A` when field `B` changes), keeping all your application logic consolidated in a single place.

Hints:

- Only update what needs to be updated, to avoid re-rendering (i.e. don't update refs, you can use libraries that provide "lens", like `ramda`, see `R.lensPath`, `R.view` and `R.set`) TODO: example

### Business logic

It might be quite an obvious thing, but don't put the actual business logic literally inside the reducer. Have a library of functions that do calculations and call those functions. Keep the complexity of a reducer to the minimum (should basically be a large flat `switch`).

## Isolated and well controlled side effects

The app would not be completely usable if it didn't allow side effects (e.g. loading the data from the backend server). The usual approach is to keep all the side effects is a dedicated place and control them, so that they don't spill into the rest of your application, producing hundreds of bugs.

Different frameworks handle it differently: _Angular NgRx_ has _effects_, _Elm_ has _commands_, _React_ offers _hooks_ etc.

The important thing is that side effects should interact with the rest of the application by integrating with the unidirectional data flow discussed above.

TODO: examples etc.

My advice is to keep all the effects on the top of the application and avoid handling effects (i.e. using _React hooks_) in the components (keep components pure, as discussed above).

## Exceptions

###

- TODO: animations
- TODO: state selectors (e.g. Angular NgRx)
