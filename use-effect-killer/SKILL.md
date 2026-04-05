---
name: use-effect-killer
description: >-
  Scan React codebase for useEffect anti-patterns (derived state, event logic in effects,
  effect chains, prop-driven resets, missing cleanup, parent notifications, etc.) and propose
  idiomatic fixes. Based on React's "You Might Not Need an Effect" guide. Use when asked to
  audit, review, or clean up useEffect usage.
argument-hint: "[path/to/scan]"
allowed-tools: Read Grep Glob Agent
---

# useEffect Killer

Audit a React codebase for unnecessary or misused `useEffect` calls and propose idiomatic alternatives.

Reference: https://react.dev/learn/you-might-not-need-an-effect

## Workflow

1. **Find all useEffect usages.** Grep for `useEffect` across `$ARGUMENTS` (or the entire repo if no path given). Filter to `.tsx`, `.ts`, `.jsx`, `.js` files.

2. **Read each file** containing useEffect. For every useEffect call, classify it against the anti-pattern catalog below. Skip effects that are legitimate (event listeners with cleanup for component-scoped DOM, animation setup, true synchronization with external systems).

3. **For each finding**, record:
   - File path and line number
   - Which anti-pattern it matches
   - The problematic code snippet
   - A concrete suggested fix using the recommended alternative

4. **Report findings** using the output format at the bottom. Group by anti-pattern. Include a summary with counts.

5. If `$ARGUMENTS` is empty, scan all React component files in the project.

## Anti-Pattern Catalog

Use these patterns to classify each useEffect you encounter.

### 1. Derived State

State that is computed from other state or props, updated via useEffect.

**Detect:** `useEffect` body calls a setter, and the value being set can be expressed as a pure function of state/props already available during render.

**Bad:**
```tsx
const [firstName, setFirstName] = useState('');
const [lastName, setLastName] = useState('');
const [fullName, setFullName] = useState('');
useEffect(() => {
  setFullName(firstName + ' ' + lastName);
}, [firstName, lastName]);
```

**Fix:** Remove the extra state and effect. Compute inline.
```tsx
const fullName = firstName + ' ' + lastName;
```

### 2. Expensive Derived Computation

Same as #1 but the computation is expensive, so the developer reached for useEffect + state to "cache" it.

**Detect:** `useEffect` that transforms or filters data from props/state into another state variable. Often involves `.filter()`, `.map()`, `.sort()`, `.reduce()`.

**Bad:**
```tsx
const [visibleTodos, setVisibleTodos] = useState([]);
useEffect(() => {
  setVisibleTodos(getFilteredTodos(todos, filter));
}, [todos, filter]);
```

**Fix:** Use `useMemo`.
```tsx
const visibleTodos = useMemo(
  () => getFilteredTodos(todos, filter),
  [todos, filter]
);
```

### 3. Reset All State on Prop Change

Using useEffect to reset state variables when an identity prop (e.g., `userId`, `itemId`) changes.

**Detect:** `useEffect` whose body sets one or more state variables to initial values, with a prop in the dependency array.

**Bad:**
```tsx
function ProfilePage({ userId }) {
  const [comment, setComment] = useState('');
  useEffect(() => {
    setComment('');
  }, [userId]);
```

**Fix:** Extract into a child component and use `key` to reset.
```tsx
function ProfilePage({ userId }) {
  return <Profile userId={userId} key={userId} />;
}
function Profile({ userId }) {
  const [comment, setComment] = useState('');
```

### 4. Adjust Some State on Prop Change

Using useEffect to partially update state when props change, but not a full reset.

**Detect:** `useEffect` with a prop dependency that conditionally updates state — often nulling out a selection or resetting a sub-field.

**Bad:**
```tsx
function List({ items }) {
  const [selection, setSelection] = useState(null);
  useEffect(() => {
    setSelection(null);
  }, [items]);
```

**Fix (preferred):** Derive the value during render instead of storing it.
```tsx
function List({ items }) {
  const [selectedId, setSelectedId] = useState(null);
  const selection = items.find(item => item.id === selectedId) ?? null;
```

**Fix (alternative):** Store previous props and adjust during render. Note: the `items !== prevItems` guard is mandatory — without it, you get an infinite render loop. This pattern also triggers one extra re-render.
```tsx
const [prevItems, setPrevItems] = useState(items);
if (items !== prevItems) {
  setPrevItems(items);
  setSelection(null);
}
```

### 5. Event Logic in Effect

Side effects (toast, notification, navigation, analytics for a user action) placed in useEffect instead of in the event handler that triggered them.

**Detect:** `useEffect` that calls functions like `showNotification`, `toast`, `navigate`, `router.push`, `alert`, or logs analytics — triggered by a state variable that was set in an event handler.

**Bad:**
```tsx
useEffect(() => {
  if (product.isInCart) {
    showNotification(`Added ${product.name} to cart!`);
  }
}, [product]);
```

**Fix:** Move into the event handler.
```tsx
function handleBuyClick() {
  addToCart(product);
  showNotification(`Added ${product.name} to cart!`);
}
```

### 6. POST / Mutation in Effect

Using a state variable as a trigger to fire a network request from useEffect.

**Detect:** `useEffect` that calls `fetch`, `post`, `axios`, `mutate`, `useMutation`, tRPC mutations, or similar — where the dependency is a state variable set by an event handler (often a "submit payload" state).

**Bad:**
```tsx
const [jsonToSubmit, setJsonToSubmit] = useState(null);
useEffect(() => {
  if (jsonToSubmit !== null) {
    post('/api/register', jsonToSubmit);
  }
}, [jsonToSubmit]);
```

**Fix:** Call directly from the event handler.
```tsx
function handleSubmit(e) {
  e.preventDefault();
  post('/api/register', { firstName, lastName });
}
```

### 7. Effect Chains

Multiple useEffects where one sets state that triggers the next, forming a cascade.

**Detect:** Two or more `useEffect` calls in the same component where the dependency of one includes a state variable set inside another. Look for sequential state-setting patterns.

**Bad:**
```tsx
useEffect(() => { if (card?.gold) setGoldCardCount(c => c + 1); }, [card]);
useEffect(() => { if (goldCardCount > 3) { setRound(r => r + 1); setGoldCardCount(0); } }, [goldCardCount]);
useEffect(() => { if (round > 5) setIsGameOver(true); }, [round]);
```

**Fix:** Derive what you can (`const isGameOver = round > 5`), consolidate the rest into the event handler.
```tsx
const isGameOver = round > 5;

function handlePlaceCard(nextCard) {
  setCard(nextCard);
  if (nextCard.gold) {
    if (goldCardCount < 3) {
      setGoldCardCount(goldCardCount + 1);
    } else {
      setGoldCardCount(0);
      setRound(round + 1);
    }
  }
}
```

### 8. App Initialization in Effect

One-time setup code (auth checks, localStorage reads, config loading) inside a `useEffect([], ...)` that breaks under Strict Mode double-mount.

**Detect:** `useEffect` with `[]` dependency array that runs init-style logic: auth token checks, localStorage reads, global config. Especially problematic if the logic has side effects that shouldn't run twice.

**Bad:**
```tsx
useEffect(() => {
  loadDataFromLocalStorage();
  checkAuthToken();
}, []);
```

**Fix:** Guard with a module-level flag so it runs once, even under Strict Mode double-mount.
```tsx
let didInit = false;

function App() {
  useEffect(() => {
    if (!didInit) {
      didInit = true;
      loadDataFromLocalStorage();
      checkAuthToken();
    }
  }, []);
}
```

For code that truly has no side effects and doesn't need React lifecycle, module-level execution is also valid:
```tsx
if (typeof window !== 'undefined') {
  // Only for pure reads — not for auth tokens or anything with side effects
  const cachedTheme = localStorage.getItem('theme');
}
```

### 9. Notify Parent via Effect

Calling a parent callback (like `onChange`) inside useEffect after a state update, instead of alongside it.

**Detect:** `useEffect` whose body calls a prop callback (`onChange`, `onUpdate`, `onSelect`, etc.) passing the current state value.

**Bad:**
```tsx
function Toggle({ onChange }) {
  const [isOn, setIsOn] = useState(false);
  useEffect(() => {
    onChange(isOn);
  }, [isOn, onChange]);
```

**Fix:** Call the callback in the event handler.
```tsx
function handleClick() {
  const nextIsOn = !isOn;
  setIsOn(nextIsOn);
  onChange(nextIsOn);
}
```

Or make the component fully controlled (remove local state, let parent own it).

### 10. Pass Data to Parent via Effect

Child fetches or computes data, then pushes it up to the parent through an effect callback.

**Detect:** `useEffect` calling a prop callback like `onFetched`, `onData`, `onLoaded` with data obtained from a hook or fetch inside the child.

**Bad:**
```tsx
function Child({ onFetched }) {
  const data = useSomeAPI();
  useEffect(() => {
    if (data) onFetched(data);
  }, [onFetched, data]);
```

**Fix:** Lift the data fetching to the parent. Pass data down, not up.
```tsx
function Parent() {
  const data = useSomeAPI();
  return <Child data={data} />;
}
```

### 11. External Store Subscription

Manual `addEventListener`/`removeEventListener` or subscription logic inside useEffect to sync with browser APIs or external stores.

**Detect:** `useEffect` with `addEventListener`/`removeEventListener` on `window`, `document`, or `navigator` (not on a component ref) that syncs external state into React state. Also `.subscribe()` / `.unsubscribe()` patterns for global stores. Do NOT flag ref-scoped DOM listeners with proper cleanup — those are legitimate.

**Bad:**
```tsx
useEffect(() => {
  function update() { setIsOnline(navigator.onLine); }
  window.addEventListener('online', update);
  window.addEventListener('offline', update);
  return () => {
    window.removeEventListener('online', update);
    window.removeEventListener('offline', update);
  };
}, []);
```

**Fix:** Use `useSyncExternalStore`.
```tsx
const isOnline = useSyncExternalStore(
  (cb) => {
    window.addEventListener('online', cb);
    window.addEventListener('offline', cb);
    return () => {
      window.removeEventListener('online', cb);
      window.removeEventListener('offline', cb);
    };
  },
  () => navigator.onLine,
  () => true
);
```

### 12. Initialize State from Props via Effect

Using useEffect to set state from props on first render, instead of passing the prop to `useState` directly.

**Detect:** `useEffect` with `[]` dependency that calls a setter with a prop value, or `useEffect` with `[prop]` where the state has a different initial value (like `null`) and is immediately overwritten.

**Bad:**
```tsx
function Editor({ initialContent }) {
  const [content, setContent] = useState(null);
  useEffect(() => {
    setContent(initialContent);
  }, []);
}
```

**Fix:** Pass the prop directly as the initial state value.
```tsx
function Editor({ initialContent }) {
  const [content, setContent] = useState(initialContent);
}
```

This eliminates the flicker where the component first renders with `null`, then immediately re-renders with the prop value.

### 13. Fetch Without Cleanup

Data fetching in useEffect without handling stale responses (race conditions).

**Detect:** `useEffect` containing `fetch`, `axios`, or async calls that set state on completion — without a cleanup function that sets an `ignore` flag or calls `AbortController.abort()`.

**Bad:**
```tsx
useEffect(() => {
  fetchResults(query).then(json => {
    setResults(json);
  });
}, [query]);
```

**Fix:** Add an ignore flag or abort controller. Better yet, use a data-fetching library (TanStack Query, SWR, etc.).
```tsx
useEffect(() => {
  let ignore = false;
  fetchResults(query).then(json => {
    if (!ignore) setResults(json);
  });
  return () => { ignore = true; };
}, [query]);
```

## Legitimate useEffect Usages (Do NOT Flag)

Skip these — they are valid uses of useEffect:

- Subscribing to **ref-scoped** DOM events with proper cleanup (e.g., ResizeObserver on a specific element ref, IntersectionObserver on a ref). Note: window/document-level subscriptions that drive render state should use `useSyncExternalStore` instead (see #11).
- Running animations or timers scoped to component mount
- Synchronizing with truly external systems (WebSocket connections, third-party widget initialization)
- Analytics/logging that should fire on component display (not on a user action)
- Focus management on mount

## Output Format

Structure your report as follows:

    ## useEffect Audit Results

    ### Summary
    - **Files scanned:** N
    - **useEffect instances found:** N
    - **Anti-patterns detected:** N
    - **Legitimate usages (skipped):** N

    ### Findings

    #### 1. Derived State (X found)

    **`src/components/UserProfile.tsx:42`**
    (show the current code)
    **Problem:** fullName is derived from first and last — no effect needed.
    **Fix:** `const fullName = first + ' ' + last;` and remove the `fullName` state.

    ---
    (repeat for each finding, grouped by anti-pattern)

    ### Clean Files
    Files with useEffect that passed review — list briefly so the user knows they were checked.
