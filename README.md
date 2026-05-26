# Table of Contents

1. [Mental Model: What React Actually Is](#mental-model-what-react-actually-is)
2. [Virtual DOM and Reconciliation](#virtual-dom-and-reconciliation)
3. [The Fiber Architecture](#the-fiber-architecture)
4. [Rendering Phases](#rendering-phases)
5. [Hooks — Mechanics](#hooks--mechanics)
6. [State Management](#state-management)
7. [Performance Optimization](#performance-optimization)
8. [React 18+ Concurrent Features](#react-18-concurrent-features)
9. [Component Patterns](#component-patterns)
10. [Testing](#testing)
11. [TypeScript + React](#typescript--react)
12. [Common Interview Traps](#common-interview-traps)

---

# Mental Model: What React Actually Is

React is **a runtime that maintains two trees and a scheduler that reconciles them**.

- **The element tree** — what your JSX returns. Plain JS objects (`{ type, props, children }`). Cheap, disposable.
- **The fiber tree** — React's internal representation. Persistent, mutable, holds state, effects, and traversal pointers.
- **The host tree** — the actual DOM (or native views). React mutates this as a side effect of reconciliation.

When you call `setState`, you don't update the DOM. You schedule work. React's reconciler walks the fiber tree, diffs against new elements, computes effects, then a separate commit phase applies changes to the host tree.

This separation — render vs commit — is the single most important mental model in React. Almost every confusing behavior (stale closures, double-rendering, effect timing, batching) comes from misunderstanding which phase you're in.

---

# Virtual DOM and Reconciliation

## The naive question: "Isn't the virtual DOM slow?"

The virtual DOM (VDOM) isn't a performance optimization in the "faster than DOM" sense. The real DOM is fast; mutating it isn't expensive *per operation*. What's expensive is:

1. **Style recalculation and layout** (reflow) — triggered when you read computed dimensions or change geometry
2. **Cascading work** — one mutation triggering paints across multiple subtrees
3. **Forced synchronous layout** (layout thrashing) when reads and writes are interleaved

The VDOM exists so React can **batch mutations and order them deterministically**, not because diffing is faster than mutation.

## The reconciliation algorithm

A full diff of two trees is `O(n³)` (classic tree edit distance). React reduces this to `O(n)` with two heuristics:

1. **Different element types produce different trees.** `<div>` → `<span>` unmounts the entire subtree and remounts. React doesn't try to reuse anything.
2. **Stable keys identify siblings across renders.** Without keys, React diffs children by index — moving an item to the front re-renders everything. With keys, React matches by identity.

```jsx
// Bad — array shift on insert/delete remounts everything after the change
{items.map((item, i) => <Row key={i} data={item} />)}

// Good — stable identity, only the new node is created
{items.map(item => <Row key={item.id} data={item} />)}
```

## Why `key={index}` is a footgun

Indices are stable as positions, not as identities. If you prepend an item to a list of indices `[0,1,2]`, the new item gets key `0`, the old `0` becomes key `1`, etc. React thinks every row "changed," so:

- Internal component state at index `0` is now attached to the new (wrong) data
- DOM nodes are reused with stale text inputs, checkbox states, focus
- Animations and transitions misfire

Use indices only when the list is **append-only and order never changes** — and even then, prefer a real ID.

## Diffing siblings

React's child reconciler:
1. Iterates new and old children in parallel
2. While types match at the same position, it updates in place
3. On a mismatch, it consults the key map (built lazily) to find a match elsewhere
4. Matched nodes are *moved* (reordered) instead of recreated
5. Unmatched old nodes are deleted; unmatched new nodes are inserted

Moving in the DOM uses `insertBefore`, which is cheap. Recreating involves teardown, allocation, re-mounting effects, losing focus — expensive in aggregate.

---

# The Fiber Architecture

Fiber (shipped in React 16, January 2017) replaced the original stack-based reconciler. The original reconciler was a recursive depth-first walk — it couldn't be paused, prioritized, or aborted mid-flight. A 50ms render on the main thread meant 50ms of jank.

## What a fiber is

A **fiber** is a plain JS object representing a unit of work — typically one component instance. Roughly:

```js
{
  type,           // function, class, or DOM tag string
  key,
  stateNode,      // class instance or DOM node
  child,          // first child fiber
  sibling,        // next sibling fiber
  return,         // parent fiber
  pendingProps,
  memoizedProps,
  memoizedState,  // hook list head (for function components)
  flags,          // bitmask: needs update, deletion, placement, etc.
  alternate,      // current ↔ work-in-progress link
  lanes,          // priority bitmask
}
```

The tree is a **linked list of linked lists** — not a recursive structure. Traversal uses pointers (`child`, `sibling`, `return`), which means React can suspend a walk after any fiber and resume it later. The runtime checks `shouldYield()` (~5ms slice by default) between fibers.

## Double buffering: current vs work-in-progress

React maintains two fiber trees:
- **Current** — what's on screen
- **Work-in-progress (WIP)** — being built during render

Each fiber has an `alternate` pointer to its counterpart in the other tree. When render completes, React swaps the roots (`root.current = workInProgress`) — atomic, no torn UI.

If a render is thrown away (interrupted by higher-priority work), the WIP tree is discarded. The current tree is untouched. This is why **render phase must be pure** — it can run, restart, or abort at any time.

## Why this matters for you

- `useState` setters during render can cause an immediate re-render of the same component (legal, used by `getDerivedStateFromProps` replacement patterns)
- Effects from a thrown-away render never fire — they're attached to fibers that get discarded
- StrictMode's double-invocation in development simulates this discard-and-retry behavior to surface impurity

---

# Rendering Phases

Every commit has three phases. Knowing which phase you're in dictates what's safe.

## 1. Render phase (pure)

Function bodies run. Hooks run. JSX is built. Nothing is committed to the DOM yet.

**Constraints:**
- Must be pure — same inputs → same output
- Can run multiple times for one update (concurrent mode, error recovery, StrictMode)
- No DOM access, no subscriptions, no timers
- No `setState` from inside the render of *another* component (you can `setState` on yourself during render, with a guard, to derive state — rare)

## 2. Commit phase (synchronous, blocking)

React mutates the DOM, runs refs, and runs `useLayoutEffect` callbacks.

**Sub-phases:**
1. **Before mutation** — `getSnapshotBeforeUpdate` reads from the DOM before changes
2. **Mutation** — DOM is updated; refs are detached from old nodes
3. **Layout** — `useLayoutEffect` fires, refs are attached to new nodes

`useLayoutEffect` runs **synchronously after DOM mutation but before browser paint**. Use it for measurements or DOM reads that would cause a visible flash if deferred (e.g., positioning a tooltip relative to a measured node). It's blocking — don't do expensive work here.

## 3. Passive effects phase (asynchronous)

`useEffect` callbacks fire **after paint**. The browser has already shown the new frame. This is where data fetching, subscriptions, logging, and other non-visual work belongs.

Cleanup functions from the previous effect run **before** the new effect — for both `useEffect` and `useLayoutEffect`.

---

# Hooks — Mechanics

## How hooks are actually stored

A function component's fiber holds a **linked list of hook records**, in call order:

```
fiber.memoizedState → hook1 → hook2 → hook3 → ...
```

Each hook record stores its state, queue of pending updates, and (for effects) the effect's create/destroy functions and dependency array. The list is rebuilt every render by calling hooks in the same order — which is *exactly* why the Rules of Hooks exist. React doesn't track which hook is which by name; it tracks by call order.

If you conditionally skip a hook on render N+1, every subsequent hook is now reading the wrong record. State leaks across hooks. This is why ESLint's `react-hooks/rules-of-hooks` is a hard error, not a warning.

## `useState`

```js
const [count, setCount] = useState(0);
```

**Initial value** runs once per mount. If it's expensive, pass a lazy initializer:

```js
const [data, setData] = useState(() => expensiveCompute());
```

The function form runs once; the inline form runs every render and the result is discarded after the first.

**Setters are stable.** `setCount` is the same reference across renders, safe to omit from dependency arrays.

**Updates are queued, not applied immediately.** Inside an event handler:

```js
setCount(c => c + 1);
setCount(c => c + 1);
setCount(c => c + 1);
// count is now 3, not 1
```

But:

```js
setCount(count + 1);
setCount(count + 1);
setCount(count + 1);
// count is now 1 — all three saw the same stale `count`
```

Use the function form when the new state depends on the previous state.

**Bailout.** If the new state equals the current state (`Object.is` comparison), React skips the re-render of that component (but may still re-render if the parent re-rendered).

## `useEffect`

```js
useEffect(() => {
  // setup
  return () => {
    // cleanup
  };
}, [deps]);
```

**Lifecycle:**
- Mount: setup runs
- Update (deps changed): cleanup runs, then setup runs
- Unmount: cleanup runs

**Dependency comparison** is `Object.is` per element. New object/array/function references count as "changed" even if structurally equal — this is the source of most infinite-loop bugs.

```js
// Bug — `options` is a new object every render
useEffect(() => { fetch(url, options); }, [url, options]);

// Fix — memoize the object, or inline what you actually depend on
useEffect(() => { fetch(url, { method, headers }); }, [url, method, headers]);
```

**Empty deps `[]`** means "run once on mount, clean up on unmount." It does *not* mean "always use the initial closure" — your effect captures the closure from the render it was created in. If you don't include something you use, you'll read stale values.

**Missing deps lead to stale closures.** The ESLint rule `react-hooks/exhaustive-deps` is correct ~98% of the time. The remaining 2% needs `useRef`, `useReducer`, or a refactor — not suppression.

## `useLayoutEffect`

Identical API to `useEffect`, but fires **synchronously after DOM mutation, before paint**. Use only when you need to read layout (`getBoundingClientRect`, scroll position) and re-apply changes before the user sees the first frame.

**Cost:** It blocks paint. Heavy `useLayoutEffect` work causes jank. If you can defer to `useEffect`, do.

In SSR, `useLayoutEffect` triggers a warning because there's no DOM. Use the `useIsomorphicLayoutEffect` pattern:

```js
const useIsoLayoutEffect = typeof window !== 'undefined' ? useLayoutEffect : useEffect;
```

## `useMemo`

```js
const value = useMemo(() => expensiveCompute(a, b), [a, b]);
```

Memoizes a value across renders. Recomputes only when deps change.

**Important: `useMemo` is a hint, not a guarantee.** React may discard the cache to free memory (in practice, it currently doesn't, but the docs explicitly reserve the right to). Never use `useMemo` for correctness — only optimization.

**Correct uses:**
- Expensive pure computations (sorting, filtering large lists)
- Stable reference for downstream `React.memo` or `useEffect` deps

**Incorrect uses:**
- Memoizing every value "just in case" — the memoization itself has overhead (hook slot, equality check)
- Caching side effects (use `useEffect` or `useRef`)

## `useCallback`

```js
const handleClick = useCallback(() => { doThing(id); }, [id]);
```

Sugar for `useMemo(() => fn, deps)`. Same caveats. Only useful when the function reference matters — passed to a memoized child, or used as an effect dep.

## `useRef`

```js
const ref = useRef(initialValue);
```

Returns a stable `{ current }` object. Mutating `current` does **not** trigger a re-render.

**Two uses:**
1. **DOM refs** — `<div ref={ref} />` populates `ref.current` during the commit phase
2. **Instance variables** — values that persist across renders without triggering them (timer IDs, previous values, mutable caches)

The ref object identity is stable for the component's lifetime. Reading `.current` during render is mostly safe (but you don't see updates until next render).

## `useReducer`

```js
const [state, dispatch] = useReducer(reducer, initialState, initFn);
```

Same model as Redux but local. Use when:
- State transitions are complex enough that ad-hoc `setState` is error-prone
- Multiple pieces of state update together
- You want to test the reducer in isolation

`dispatch` is stable like setters. Reducers must be pure (they may run multiple times in concurrent mode).

## `useContext`

```js
const value = useContext(MyContext);
```

Subscribes the component to a context. **Any context value change re-renders all consumers**, even if they only use a subset of the value.

Common mitigation: split context, or memoize the value object.

```jsx
// Bad — new object every render, every consumer re-renders
<Ctx.Provider value={{ user, theme }}>

// Better — memoize
<Ctx.Provider value={useMemo(() => ({ user, theme }), [user, theme])}>

// Best for unrelated values — separate providers
<UserCtx.Provider value={user}>
  <ThemeCtx.Provider value={theme}>
```

## Custom hooks

A custom hook is any function that:
- Starts with `use`
- Calls other hooks

That's it. There's no registration, no special syntax. The naming convention is what lets the linter enforce rules.

**Compose, don't inherit.** A `useDebouncedSearch(query)` is a real abstraction. `useUser()` is fine. `useEverything()` is a code smell.

**Custom hooks share *logic*, not *state*.** Calling `useUser()` in two components gives two independent subscriptions, not one shared user. To share state, you need lifted state or a global store.

---

# State Management

## The spectrum

| Scope | Use |
|---|---|
| Single component | `useState` |
| Component + descendants | Prop drilling (a few levels), or `useState` lifted up |
| Wider subtree, infrequent updates | Context |
| Wider subtree, frequent updates | External store (Redux, Zustand, Jotai) |
| Cross-tab, persisted | Store + persistence layer |
| Server data | React Query / SWR — *not* a general-purpose store |

The mistake most senior engineers have made at least once: using Redux for server data. It's possible but you reinvent caching, deduplication, stale-while-revalidate, retry, and refetch-on-focus. React Query is purpose-built for it.

## Context — what it is and isn't

Context is a **dependency injection mechanism**, not a state manager. It provides a value to a subtree without prop drilling. It doesn't optimize re-renders; it doesn't have selectors; it doesn't dedupe.

When a Provider's `value` changes (by reference), every consumer re-renders. There's no way to subscribe to "just `value.user.name`" from context alone.

This is why combining `useContext` + `useReducer` works for small apps but breaks at scale — every dispatch re-renders every consumer.

## Redux — mental model

Redux is three things:
1. **A single store** holding the entire app state
2. **Pure reducers** that compute new state from `(state, action)`
3. **A subscription mechanism** so views re-render on state changes

The single-store model exists for **time-travel debugging, predictable serialization, and replay**. If you don't need those, Redux's overhead may not pay off.

### `react-redux` internals (briefly)

`useSelector(selectFn)`:
1. Subscribes the component to the store on mount
2. After each dispatch, runs `selectFn(state)` and compares result with previous (`===` by default)
3. If different, schedules a re-render
4. Cleanup unsubscribes on unmount

The `===` comparison is why you must memoize selector outputs that return new objects/arrays — otherwise every dispatch re-renders.

```js
// Bad — new array every time, even if items are the same
const items = useSelector(s => s.list.filter(x => x.active));

// Good — `reselect` memoizes by inputs
const selectActive = createSelector([s => s.list], list => list.filter(x => x.active));
const items = useSelector(selectActive);
```

### Redux Toolkit (RTK)

Modern Redux. `createSlice` generates action creators and reducers with Immer under the hood (so you can "mutate" inside reducers and Immer produces immutable updates). `createAsyncThunk` standardizes async flows. RTK Query is a built-in data fetching layer competitive with React Query.

If you start a new project today, the choice is RTK or Zustand/Jotai. Plain Redux is legacy.

## Zustand — the alternative

```js
const useStore = create((set) => ({
  count: 0,
  increment: () => set(s => ({ count: s.count + 1 })),
}));

function Counter() {
  const count = useStore(s => s.count);
  return <button onClick={useStore.getState().increment}>{count}</button>;
}
```

Hook-first API. No provider needed. Selectors work like Redux but the boilerplate is minimal. Same caveat: memoize complex selector outputs (Zustand provides `shallow` and `useShallow`).

## Jotai / Recoil — atomic state

Atomic libraries flip the model: instead of one big tree, you compose many small atoms. Components subscribe to specific atoms. This avoids the "select a slice" pattern entirely — you read the atom directly.

Useful when state is highly granular and many components need different overlapping subsets. Less useful for app-wide transactional updates.

## A note on "global state" overreach

A lot of state that ends up in Redux/Zustand doesn't belong there:
- **Form state** → keep in the form component or use `react-hook-form` / `formik`
- **Server data** → React Query / SWR
- **URL state** (filters, pagination, modal-open flags) → URL params, parsed via the router
- **Theme, locale** → Context (changes rarely, OK to re-render)

Global state is for things that are genuinely shared, mutable, and not derivable from elsewhere.

---

# Performance Optimization

## When to optimize

Default position: **don't**. React is fast enough that most apps need no optimization. Premature memoization adds complexity and slows reads of code. Optimize when:

1. You can **measure** a problem (DevTools Profiler, Lighthouse, real user metrics)
2. The user perceives jank, slow input response, or slow initial load
3. You've identified a specific component or interaction as the bottleneck

## What actually causes slow React apps

In rough order of frequency:

1. **Too many re-renders** of components with non-trivial work
2. **Large lists rendered without virtualization**
3. **Expensive computations inline in render** (filtering, sorting, transforming on every render)
4. **Layout thrashing** — reading layout inside loops, or interleaving reads and writes
5. **Large bundle / slow initial JS parse and execute**
6. **Synchronous network or storage calls blocking the main thread**

## Diagnosing re-renders

Use React DevTools Profiler:
- Record a problematic interaction
- Look at the flame graph — wide bars are slow components
- Use "Why did this render?" (must enable in settings) to see which props/state/hooks caused each render

A component re-renders when:
- Its parent re-renders (default behavior, regardless of props)
- Its state changes
- A context it consumes changes
- A hook it uses signals an update

`React.memo` prevents re-render-from-parent if props are shallowly equal.

## `React.memo` — when it pays off

```js
const Row = React.memo(function Row({ data, onSelect }) {
  return <li onClick={() => onSelect(data.id)}>{data.label}</li>;
});
```

`memo` shallow-compares props. If equal, render is skipped.

**It pays off when:**
- The component is rendered many times (list rows)
- Its render is non-trivial
- Its props are typically stable

**It doesn't help when:**
- The parent passes a new object/function/array every render — props are never equal
- The component is cheap to render anyway
- Children re-render anyway because of context or hooks

Common gotcha:

```jsx
// Even with React.memo on Row, every render creates a new onSelect — memo doesn't help
<Row onSelect={(id) => handleSelect(id)} />

// Wrap in useCallback so the reference is stable
const onSelect = useCallback(id => handleSelect(id), [handleSelect]);
<Row onSelect={onSelect} />
```

## Virtualization

For lists of more than a few hundred items, render only the visible window. Libraries: `react-window`, `react-virtuoso`, `@tanstack/react-virtual`.

The DOM cost is proportional to nodes rendered, not data size. Virtualization keeps the DOM small regardless of dataset.

Trade-offs:
- `Cmd+F` browser search only finds rendered items
- Variable row heights need measurement or estimation
- Accessibility — ensure screen readers still get a sensible structure

## Code splitting

```js
const Settings = lazy(() => import('./Settings'));

<Suspense fallback={<Spinner />}>
  <Settings />
</Suspense>
```

`React.lazy` + `Suspense` defers the JS for a component until it's rendered. Use for:
- Route-level splits (most impactful)
- Heavy modals or rarely-used features
- Anything with a large dependency tree (charts, editors, maps)

Webpack/Vite handle the chunk splitting. Verify with bundle analyzer (`source-map-explorer`, `webpack-bundle-analyzer`).

## The `useTransition` lever

```js
const [isPending, startTransition] = useTransition();

const onChange = (e) => {
  setInput(e.target.value); // urgent
  startTransition(() => {
    setFilter(e.target.value); // non-urgent, can be interrupted
  });
};
```

Tells React the wrapped update is low priority. Typing stays responsive even if the downstream render is slow. Pairs naturally with expensive filtered lists.

## Web Workers for CPU-bound work

If you have genuinely expensive synchronous work (parsing a 10MB CSV, image processing, complex calculations), no amount of React optimization helps — JS is single-threaded. Move the work to a Web Worker.

`Comlink` makes worker communication ergonomic.

---

# React 18+ Concurrent Features

## Concurrent rendering

The big shift in React 18: **rendering is interruptible**. A render started for a low-priority update can be paused if a higher-priority update (user input) comes in. The interrupted render is discarded (not just stalled), and React starts fresh with the new state.

This requires that **render be pure**. If your render has side effects (mutating module-level state, scheduling timers), discarded renders cause bugs.

**StrictMode** in development double-invokes render, effects (mount→unmount→mount), and reducers to surface these bugs.

## Automatic batching

Pre-18, only updates inside React event handlers were batched. Updates inside `setTimeout`, promises, native event handlers fired separate renders.

React 18 batches **all** updates by default, including across async boundaries:

```js
setTimeout(() => {
  setA(1);
  setB(2);
  // 18: one render
  // 17: two renders
}, 0);
```

Opt out with `flushSync` if you genuinely need to commit between two updates (rare — usually for measurement-based logic).

## `useTransition` and `useDeferredValue`

`useTransition` marks updates as transitions (lower priority).

`useDeferredValue` is the read-side equivalent — defers reading a value until urgent work is done:

```js
function SearchResults({ query }) {
  const deferredQuery = useDeferredValue(query);
  const results = useMemo(() => search(deferredQuery), [deferredQuery]);
  return <List items={results} />;
}
```

`deferredQuery` may lag behind `query` during heavy work, but typing stays smooth.

## Suspense for data fetching

`<Suspense>` boundaries let components "suspend" while waiting for data. The framework (React Query, Relay, Next.js) integrates with React's suspense protocol.

```jsx
<Suspense fallback={<Skeleton />}>
  <UserProfile id={id} />
</Suspense>
```

Inside `UserProfile`, calling `use(userPromise)` (React 19) or a suspense-aware hook unmounts to the fallback while loading. Nested suspense boundaries create staggered loading UIs.

## Streaming SSR

`renderToPipeableStream` (Node) and `renderToReadableStream` (Web) stream HTML as it's generated. Suspense boundaries flush as their data resolves — the user sees skeleton chrome first, then content fills in. Combined with selective hydration, this is the foundation of Next.js App Router, Remix, and similar frameworks.

## Server Components (React 19 / Next.js App Router)

Server Components run only on the server. They:
- Have zero JS in the client bundle
- Can directly call databases, file systems, APIs
- Can't use hooks, state, or event handlers
- Pass serialized data to Client Components

Marking is via `"use client"` (boundary) and `"use server"` (server-only). The mental model is similar to PHP/JSP server rendering, but with React's component model and interleaving.

This is a major architectural shift — most existing patterns (Redux, Context across the whole app) need rethinking in a server-first app.

---

# Component Patterns

## Composition over configuration

The most powerful React pattern is "pass children." It avoids prop explosions and keeps components decoupled.

```jsx
// Configuration — every variant needs a new prop
<Card title="..." icon="..." footer="..." actions={[...]} />

// Composition — the parent decides the contents
<Card>
  <Card.Header>...</Card.Header>
  <Card.Body>...</Card.Body>
  <Card.Footer>...</Card.Footer>
</Card>
```

## Compound components

A parent component exposes related children that share implicit state via context:

```jsx
function Tabs({ children, defaultValue }) {
  const [active, setActive] = useState(defaultValue);
  return (
    <TabsContext.Provider value={{ active, setActive }}>
      {children}
    </TabsContext.Provider>
  );
}

Tabs.List = function List({ children }) { return <div role="tablist">{children}</div>; };
Tabs.Tab = function Tab({ value, children }) {
  const { active, setActive } = useContext(TabsContext);
  return <button aria-selected={active === value} onClick={() => setActive(value)}>{children}</button>;
};
Tabs.Panel = function Panel({ value, children }) {
  const { active } = useContext(TabsContext);
  return active === value ? <div>{children}</div> : null;
};

// Usage
<Tabs defaultValue="a">
  <Tabs.List>
    <Tabs.Tab value="a">A</Tabs.Tab>
    <Tabs.Tab value="b">B</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel value="a">Panel A</Tabs.Panel>
  <Tabs.Panel value="b">Panel B</Tabs.Panel>
</Tabs>
```

Used by Radix, Reach UI, Headless UI. Excellent API ergonomics — the consumer controls layout while the parent owns state.

## Render props

Pass a function as `children` (or any prop) to give the consumer control over rendering:

```jsx
<Mouse>{({ x, y }) => <div>{x},{y}</div>}</Mouse>
```

Historically used for behavior reuse (mouse tracking, data fetching). Largely superseded by hooks — custom hooks are usually cleaner. Still useful when you need to compose render logic with structure.

## Higher-Order Components (HOC)

A function that takes a component and returns a new component with added behavior:

```jsx
function withAuth(Component) {
  return function Wrapped(props) {
    const user = useContext(AuthContext);
    if (!user) return <Login />;
    return <Component {...props} user={user} />;
  };
}

const ProfilePage = withAuth(Profile);
```

Falling out of fashion. Issues:
- Prop name collisions
- Wrapping order matters and isn't obvious
- TypeScript inference is harder
- Stack traces become noisy

Hooks replaced almost all HOC use cases. Still seen in older codebases and some libraries (`connect`, `withRouter`).

## Container / Presentational

Split components into:
- **Container** — knows about data, dispatches actions, no markup
- **Presentational** — pure, props in / JSX out, easy to test and storybook

Origin of this pattern (Dan Abramov, 2015) explicitly retracted by him after hooks. The seam between "smart" and "dumb" is more fluid with hooks. Still useful conceptually for testing — keep components that touch I/O thin, and keep pure rendering components dependency-free.

## Controlled vs uncontrolled

A **controlled** input has its value owned by React state:
```jsx
<input value={text} onChange={e => setText(e.target.value)} />
```

An **uncontrolled** input lets the DOM own the value; you read it via ref:
```jsx
<input ref={ref} defaultValue="..." />
// Read with ref.current.value
```

Use controlled when:
- You need to validate on every keystroke
- The value drives other UI
- You need to format or mask input

Use uncontrolled when:
- The value is read only on submit
- Performance matters (no re-render on every keystroke)
- Integrating with non-React code

`react-hook-form` is largely uncontrolled under the hood, which is why it's faster than Formik for big forms.

---

# Testing

## The pyramid for React

- **Unit tests** — pure functions, custom hooks (with `@testing-library/react-hooks` or `renderHook`), reducers
- **Component tests** — render a component, simulate user interaction, assert on visible output (React Testing Library)
- **Integration tests** — multiple components working together, mocked at network boundary (MSW)
- **E2E tests** — full app in a real browser (Playwright, Cypress)

Aim for a fat middle: lots of component and integration tests, fewer E2E.

## React Testing Library — philosophy

Test what the user sees and does, not implementation details. Query by accessible role, label, text — not by class name or test ID (unless nothing else works).

```js
// Bad — couples test to DOM structure
expect(container.querySelector('.submit-button')).toBeDisabled();

// Good — queries the way assistive tech does
expect(screen.getByRole('button', { name: /submit/i })).toBeDisabled();
```

This forces tests to mirror real usage and incidentally drives better accessibility.

## `@testing-library/user-event`

Prefer `userEvent` over `fireEvent`. `userEvent` simulates a full interaction sequence — `click` involves `mousedown`, `mouseup`, `focus`, `click`. `fireEvent` fires single synthetic events, which can miss bugs.

```js
import userEvent from '@testing-library/user-event';

const user = userEvent.setup();
await user.click(screen.getByRole('button'));
await user.type(screen.getByLabelText(/email/i), 'a@b.com');
```

## Mocking network — MSW

Mock Service Worker intercepts requests at the network layer (using a Service Worker in browser, a Node interceptor in Jest). Tests run against real-looking HTTP responses without hitting the network.

Pros: no `jest.mock('axios')` per test, same handlers reusable across unit/integration/E2E/Storybook.

## Testing hooks

```js
import { renderHook, act } from '@testing-library/react';

const { result } = renderHook(() => useCounter());
act(() => result.current.increment());
expect(result.current.count).toBe(1);
```

`act` wraps updates so React applies them synchronously in the test. Most user events do this automatically; manual state mutations need explicit `act`.

## Async testing

```js
// Wait for an element to appear
await screen.findByText(/loaded/i);

// Wait for a condition
await waitFor(() => expect(handler).toHaveBeenCalled());

// Wait for an element to disappear
await waitForElementToBeRemoved(screen.queryByText(/loading/i));
```

Don't use arbitrary `setTimeout`s in tests — flaky and slow.

## What not to test

- Implementation details (state shape, internal function names)
- Third-party libraries
- Framework behavior (does `useState` work? React tests that)
- Trivial pass-through components

---

# TypeScript + React

## Component typing

```tsx
// Function component
type Props = { name: string; onClick?: () => void };

function Greeting({ name, onClick }: Props) {
  return <button onClick={onClick}>Hi {name}</button>;
}

// With children
type Props = { title: string; children: React.ReactNode };

// Forwarding refs
const Input = forwardRef<HTMLInputElement, Props>(function Input(props, ref) {
  return <input ref={ref} {...props} />;
});
```

Avoid `React.FC` — it implicitly adds `children` (debatable), and historically had issues with `defaultProps` and generics. Just type props directly.

## `React.ReactNode` vs `JSX.Element` vs `React.ReactElement`

- `ReactNode` — anything React can render: elements, strings, numbers, `null`, fragments, arrays. Use for `children`.
- `ReactElement` — a JSX-produced object. More restrictive — excludes strings, numbers, null.
- `JSX.Element` — basically `ReactElement<any, any>`. Returned by JSX expressions.

Use `ReactNode` for the most flexibility; use `ReactElement` when you specifically need a JSX result (e.g., a function that must return a renderable element).

## Hooks with TypeScript

```tsx
// Type inference works for most cases
const [count, setCount] = useState(0); // number

// Explicit when initial value doesn't match the eventual type
const [user, setUser] = useState<User | null>(null);

// useRef variants
const inputRef = useRef<HTMLInputElement>(null); // for DOM refs
const timerRef = useRef<number | undefined>(undefined); // for mutable values

// useReducer — type the state and actions
type Action = { type: 'inc' } | { type: 'set'; value: number };
const reducer = (state: number, action: Action): number => { ... };
```

## Generic components

```tsx
type ListProps<T> = {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
};

function List<T>({ items, renderItem }: ListProps<T>) {
  return <ul>{items.map((item, i) => <li key={i}>{renderItem(item)}</li>)}</ul>;
}

// Usage — TS infers T
<List items={users} renderItem={u => u.name} />
```

For `forwardRef` with generics, you'll need to cast or use a helper type — the typings don't infer cleanly. This is a known pain point.

## Discriminated unions for prop variants

```tsx
type ButtonProps =
  | { variant: 'link'; href: string }
  | { variant: 'submit'; onSubmit: () => void };

function Button(props: ButtonProps) {
  if (props.variant === 'link') return <a href={props.href}>...</a>;
  return <button onClick={props.onSubmit}>...</button>;
}
```

TypeScript narrows the type based on the discriminator. Far better than optional props that "shouldn't be combined."

## Event handlers

```tsx
const onClick: React.MouseEventHandler<HTMLButtonElement> = (e) => {
  e.preventDefault();
};

const onChange: React.ChangeEventHandler<HTMLInputElement> = (e) => {
  setValue(e.target.value);
};
```

The specific element type matters — `e.target.value` exists on `HTMLInputElement` but not on `HTMLButtonElement`.

## `as` prop pattern (polymorphic components)

```tsx
type ButtonProps<E extends React.ElementType> = {
  as?: E;
} & Omit<React.ComponentPropsWithoutRef<E>, 'as'>;

function Button<E extends React.ElementType = 'button'>({ as, ...props }: ButtonProps<E>) {
  const Component = as ?? 'button';
  return <Component {...props} />;
}

<Button as="a" href="/x">Link</Button>
```

Notoriously hard to type correctly with refs. Libraries like `@radix-ui/react-polymorphic` exist to handle this.

---

# Common Interview Traps

## "Why is my `useEffect` running twice?"

In development with `<StrictMode>`, React intentionally mounts → unmounts → remounts every component to surface effects that aren't safe to re-run. Production runs once.

If your effect breaks in StrictMode, it has a real bug — fix the bug, don't disable the mode.

## "Why isn't my state updating immediately?"

Setters are asynchronous within an event handler. The new value isn't visible until the next render.

```js
setCount(count + 1);
console.log(count); // still old value
```

To compute against the latest, use the functional form. To run code after the update, use `useEffect`.

## "Why does this `console.log` show old state?"

Stale closures. The handler was defined in a render where `count` was 0. It captures `count = 0` forever unless re-created.

```js
useEffect(() => {
  const id = setInterval(() => {
    console.log(count); // always 0
  }, 1000);
  return () => clearInterval(id);
}, []); // missing `count` dep
```

Either add `count` to deps (recreates the interval), use a ref, or use the functional setter form.

## "Should I use `useMemo` here?"

Probably not. Default is no. The overhead of memoization (dependency comparison + storage) often exceeds the cost it's avoiding. Profile first.

Justified `useMemo` uses:
- Expensive computation (real cost, measured)
- Stable reference for `React.memo` children or effect deps
- Avoiding new object/array refs that break referential equality

## "How does React know what changed?"

It doesn't — it doesn't track mutations. Every `setState` triggers a re-render of that component and its descendants (subject to `memo`). React then diffs the new VDOM against the old. The diff is what produces the minimal set of DOM mutations. State mutation tracking is *not* how React works (that's Vue/MobX/Solid).

## "What's the difference between state and refs?"

Both persist across renders. State triggers re-renders; refs don't. State is read during render; refs should generally be read in effects or handlers, not during render (reading a ref during render is allowed but you don't see latest values reliably).

Rule of thumb: if the UI should react to it, use state. If it's plumbing (timer ID, DOM node, cached calculation), use a ref.

## "Why are my context consumers all re-rendering?"

Context doesn't bail out on partial updates. Any change to the provider's `value` (by reference) re-renders all consumers.

Fixes: split context, memoize the value object, use a selector library (`use-context-selector`), or move to an external store.

## "What's React's render flow for `setState`?"

1. Caller invokes setter
2. React enqueues the update on the fiber's update queue
3. React schedules a re-render (synchronous if inside event handler, deferred otherwise)
4. **Render phase:** React calls the component function again with new state, builds a new fiber tree
5. React diffs new tree against current tree
6. **Commit phase:** mutations applied to DOM, refs attached, layout effects fire
7. After paint: passive effects (`useEffect`) fire

## "What is reconciliation vs rendering vs committing?"

- **Rendering** — calling component functions, producing the new element tree
- **Reconciliation** — comparing new tree against current, computing the work list
- **Committing** — applying that work list to the DOM

Render and reconciliation are interruptible (in concurrent mode). Commit is synchronous and atomic.

---

## Further Reading

- [React docs](https://react.dev) — the rewrite is genuinely excellent, worth re-reading even as a senior engineer
- [Overreacted](https://overreacted.io) — Dan Abramov's blog, deep mental models
- [Just JavaScript](https://justjavascript.com) — Dan's mental model course for JS itself
- [React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture) — Andrew Clark's original notes
- [Build your own React](https://pomb.us/build-your-own-react/) — implements React in ~200 lines, clarifies internals
- [React source code](https://github.com/facebook/react/tree/main/packages/react-reconciler) — the reconciler is the most educational part

---
