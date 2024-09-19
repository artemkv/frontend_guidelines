v0.1

Disclamer: you can treat this document as my personal opinion. You don't have to agree with this, and feel free to completely disregard all I say. I am looking to be right, my aim is to share.

# Intro

Building complex front-end applications is a complex task. Fortunately, there has been a lot of advancements in this area and many patterns emerged that helped to reduce this complexity, maintainance cost, and, as a result, a number of bugs.

Today, many frontend developers start their journey in a particular framework/library, such as _React_, _Angular_ etc. Working with these frameworks and libraries, they learn library- or framework-specific patterns and practices that, while useful, do not always provide a solid fundamentals for understanding the bigger picture.

This document aims at providing comprehensive guide for reasoning about the frontend application architecture in a framework-independent way.

When building a frontend application, most of the (technical) complexity lies in **(1)** handling UI component dependencies (as in "this view should update when that button is clicked") and **(2)** reconciling state updates triggered by user actions and business logic.

UI component dependencies usually obey "if and only if" logic:
- Necessary condition: "component A should only update when component B changes";
- Sufficient condition: "if component B changes, component A must get updated".

If you don't respect these rules, you will either have UI that updates too often, resulting in flickering and poor performance; or too seldom, resulting in stale, out-of-sync views.

To make things worse, you have multiple sources of updates: application business logic and user actions. In a poorly designed frontend application or framework you may encounter the situation when data flows in both directions: from app business logic to the UI component and from the UI component to the app business logic, which easily gets out of control and breeds bugs related to inconsist state.


# Principles

Solving the challenges of complex frontend applications relies on following principles:
- Unidirectional flow of data
- Pure views
- Global, consistent, and immutable state
- Pure state update function
- Isolated and well controlled side effects

The good place to start is [Elm architecture](https://guide.elm-lang.org/architecture/). _Elm_ is a purely functional language that resembles _Haskell_ and the architecture it promotes emerges from its purely functional nature.

_Elm_ architecture is not just a fun reading. It has fundamental practical and historical importance, as it has inspired libraries like _Redux_ and, consequently, state management in _React_, _Angular_ and others. Essentially, it is THE architecture for the frontend app and the idea behind the modern frontend libraries.

You don't have to follow the _Elm_ architecture strictly, as it can sometimes feel too rigid, but it's important to know when you deviate from this architecture, and be able to articulate the advantages and drawbacks of the alternative solution.


## Unidirectional flow of data

Current State -> View -> Events -> Update -> New State

- The state is the model of your app;
- This state flows into the views;
- The views can trigger events;
- The events bubble to the update function;
- The update produces the new state;
- The new state flows into new views.

While this looks like a loop, it is actually a spiral: nothing is ever mutated, only the new stuff is created; and all the dependencies are one-way.

We will now look at all the parts in more details.


## Pure views

Also known as _presentational components_ or _pure components_.

The pure components are pure functions that receive the fragments of **state** as an input and return (virtual) **DOM elements** as an output.

Example with _React_:

```js
function Welcome(props) {
  return <div>
    <h1>Hello, {props.name}</h1>
    <button onclick="clicked_hello">
        Say hello
    </button>
  </div>;
}
```

Making views out of pure function brings many advantages:
- Easy to unit-test, if you desire to do so (allows input-output test, although, personally, I would probably not find these tests very useful);
- Easy to understand, contains minimal logic;
- Easy to develop in isolation, e.g. using tools like _Storybook_ (only requires providing inputs for rendering);
- Allows results to be cached as long as the inputs remain the same, which is extremely important to avoid unnecessary re-rendering.

The last statement is really important. In case of Elm, the purity is esured by the compiler, so you can always cache the component as long as the inputs stay the same. In case of _TypeScript_ and _React_, you, as a developer, have the responsibility to keep the components pure and to ensure the outputs are cached.

TODO: memo

### Container components

If, for any reason, you feel that you need to push the state management on a component itself (see below on state), do not mix everything in one place: create a purely representational component and a container component.


## Global, consistent, and immutable state

Also known as *model*.

The state can be hosted in the _Redux store_ or simply in the root app component (using state hook, in case of _React_). It doesn't really matter (and you could write your own Redux library in about half an hour).

What is important:
- State is the most important part of your application. It represents your application business domain and provides the [**Ubiquitous Language**](https://martinfowler.com/bliki/UbiquitousLanguage.html), i.e. the common vocabulary for the team;
- State must be always consistent, so you should design it in a way that renders it impossible to construct an illegal/inconsistent state (e.g. no null or undefined properties);
- State must be immutable. The only way to update the state of the application should be by creating a new state from the old state; this, in turn, would require re-executing views to generate new DOM, and update the parts of DOM that have changed, this is a task of a runtime;
- State flows down into views;

### Always consistent state

One of the most imprtant properties of the state is to be always consistent. I highly recommend [Domain-Driven Design](https://www.amazon.com/gp/product/0321125215) by Eric Evans.

Let's consider an example.

One of the common tasks in a typical frontend application is loading of the data that needs to be displayed.

Many developers would come up with something as follows:

```ts
enum DataLoadingState {
    Loading,
    Success,
    Error
}

interface ComponentState {
    loadingState: DataLoadingState,
    result?: Data;
    error?: string;
}
```

Naturally, when `loadingState` is `Success`, you should expect `result` to be filled, and when `loadingState` is `Error`, the `error` field should be filled.

This sounds logical and simple, but this is very problematic.

- This datatype just invites inconsitencies. There is nothing that prevents me to set both `result` and `error` or none of those. Consequentially, the task of validating the state and handling inconsistent state fall on the rest of the application;
- The dependencies between properties might be obvious in such a small example, but, as your state grows, it may become completely unclear which properties depend on each other, it is simply impossible to say what is the valid state looking at the data type;
- The complexity of the state grows, as you need to declare every property that may ever be non-null on the same data type.

Fortunately, there is a simple solution that relies on discriminated unions:

```ts
enum DataLoadingState {
    Loading,
    Success,
    Error
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

type ComponentState = Loading | Error | Sucess
```

Notice that we have completely eliminated all the issues mentioned above.

TODO: use generics, i.e. `Error<T>` and `Success<T>`

### Single global state

It is natural for developers to tame complexity by splitting complex things in multiple simple things. This is why the advise to have a single global application state may seem completely crazy. After all, didn't we invent OOP just to avoid global variables?

That is correct, the global variables are evil, but global variables are mutable: any part of the application can modify the value of a global variable at any moment.

This is not the case with the approach we are discussing here, where the state is immutable and creation of a new state is localized inside of a _state update function_.

Having the complete application state in a single place has a great advantage. In many frontend applications, the components are highly dependent. Right now, as I'm typing in the Visual Code editor window, the status bar, a completely separate component, is updating the line length while preview on the right is showing rendered markdown. If we made the text to be stored on a level of an editor component, we would have a hard time keeping the rest of UI in sync.

This is why React is suggesting [lifting state up](https://react.dev/learn/sharing-state-between-components). I find this confusing, as in my opinion, it suggests that the local state should be a default.

I would rather advocate for having all the state on top by default, and exceptionally pushing state down, when it's really needed (e.g. when parts of the application UI are truly independent and can be seen as separate application).

### Note on state complexity

Another argument against having all the state in a single state tree might be a complexity of that tree ("would get too big").

However, I would argue that, if you follow the approach explained in "Always consistent state", your state tree, at any point in time, will only include the properties that are valid and required by the currently displayed UI components.

So your state tree will never get more complex than a UI at any single point in time during the application execution, which is the right level of complexity.

### Anti-corruption layer

This is simple: basically, don't throw the data you don't control directly on your UI components, especially when retrieving the data from the schemaless databases.

This means: process the data before putting it into the global store, namely:
- Validate the data. All the fields that are mandatory have to be present and in the correct format;
- Sanitize the data. If some of the properties are optional, they might be not present in the JSON, sanitizing means you add the property and set it to null;
- Impute the missing values: use app defaults, when applicable;
- Convert into app internal format (e.g. parse datetime from string to a number);
- Versioning: convert the records from any version to a recent canonical format.


## Pure state update function (aka Reduce)

Views may interact with the user, and report the events up. The exact mechanism is irrelevant (_events_ in _React_, _actions_ in _Redux_, _messages_ in _Elm_ etc.). You can make events bubble through components or dispatch events from low-level components directly to _update_ function using _dispatch_ function.

What is important:
- The event should be self-contained, i.e. carry all the relevant information, such as id of an element selected, text entered etc;
- Events flow from the views up;
- Events should be processed in a single place, usually called _reducer_ or _update_, a pure function that accepts the event/action + current state and returns the new state;
- All the data dependencies should be handled here (update field `A` when field `B` changes), keeping all your application logic consolidated in a single place.

- TODO: only update what needs to be updated, to avoid re-rendering (i.e. don't update refs)
- TODO: consider using lens
- TODO: example


## Isolated and well controlled side effects

The app would not be completely usable if it didn't allow side effects (e.g. loading the data from the backend server). The usual approach is to keep all the side effects is a dedicated place and control them, so that they do spill into the rest of your application, producing hundreds of bugs.

Different frameworks handle it differently: _Angular NgRx_ has _effects_, _Elm_ has _commands_, _React_ offers _hooks_ etc.

The important thing is that side effects should interact with the rest of the application by integrating with the unidirectional data flow discussed above.

TODO: examples etc.


## Exceptions

- TODO: low-level micro-updates (text box, every character typed by the user triggers update)
- TODO: animations
- TODO: state selectors (e.g. Angular NgRx)
