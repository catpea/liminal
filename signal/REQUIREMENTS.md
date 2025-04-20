# JavaScript Signal System – Finalized Functional Specification

## 1. Introduction

This document specifies a reactive **Signal** system in JavaScript for managing single pieces of state with synchronous reactivity. Signals provide:

- **Type safety**: Runtime enforcement of consistent data types.
- **Immutability**: Support for frozen signals via a private `.#frozen` flag.
- **Reactivity**: Synchronous subscriptions with immediate callback invocation.
- **Composition**: Derived signals through `.map()` and `.merge()`.
- **Debugging**: Introspection via `.inform()` for signal relationships.

The system ensures predictable behavior, robust error handling, and clear APIs for reactive state management.

---

## 2. Signal Construction

```javascript
const signal = new Signal(initialValue);
```

A `Signal` is instantiated with an `initialValue` that defines its locked-in data type.

### Constraints

- `initialValue` must not be `null` or `undefined`.
- The constructor throws an `Error` with the message `"Invalid initial value: null or undefined"` if this constraint is violated.
- The signal’s type is determined by `typeof initialValue` (e.g., `"number"`, `"string"`, `"object"`).
- For objects and arrays, the type is enforced by checking `instanceof` or `Array.isArray`.

### Runtime Behavior

- The signal stores `initialValue` as its initial `.value`.
- The signal is mutable by default (`.#frozen = false`).
- The signal initializes an empty `Set` for tracking child signals (`.#children`).

**Example:**

```javascript
const user = new Signal({ name: "Alice" }); // Type: object
const counter = new Signal(0);              // Type: number
const bad = new Signal(null);              // Throws Error: Invalid initial value: null or undefined
```

**Test Snippet:**

```javascript
assert.throws(() => new Signal(null), /Invalid initial value: null or undefined/);
assert.throws(() => new Signal(undefined), /Invalid initial value: null or undefined/);
const s = new Signal(42);
assert.strictEqual(s.value, 42);
assert.strictEqual(s.#frozen, false);
```

---

## 3. Reading and Writing `.value`

### `.value` Getter

- Returns the signal’s current internal value.
- No side effects.

**Example:**

```javascript
const s = new Signal(10);
assert.strictEqual(s.value, 10);
```

### `.value` Setter

Sets the signal’s value if **all** of the following conditions are met:

1. The new value is neither `null` nor `undefined`.
2. The new value is strictly different (`!==`) from the current value.
3. The signal is not frozen (`.#frozen === false`).
4. The new value matches the signal’s locked-in type:
   - For primitive types, `typeof newValue === typeof initialValue`.
   - For objects, `newValue instanceof Object` and matches the constructor of the initial value.
   - For arrays, `Array.isArray(newValue)`.

### Setter Behavior

- If conditions are met:
  - Updates the internal value.
  - Notifies all subscribers with the new value.
  - Propagates updates to child signals (via `.map()` or `.merge()`).
- If any condition fails:
  - Throws an `Error` with a descriptive message:
    - `"Invalid value: null or undefined"` for `null`/`undefined`.
    - `"Cannot set value on frozen signal"` for frozen signals.
    - `"Type mismatch: expected <type>, got <type>"` for type violations.
  - No state changes occur.
- If the new value equals the current value (`===`), the setter is a no-op (no notifications).

**Example:**

```javascript
const s = new Signal(5);
s.value = 6;           // OK, notifies subscribers
s.value = 6;           // No-op, no notification
s.value = null;        // Throws: Invalid value: null or undefined
s.value = "6";         // Throws: Type mismatch: expected number, got string
s.#frozen = true;
s.value = 10;          // Throws: Cannot set value on frozen signal
```

**Test Snippet:**

```javascript
const s = new Signal(1);
s.value = 2;
assert.strictEqual(s.value, 2);
s.#frozen = true;
assert.throws(() => { s.value = 3 }, /Cannot set value on frozen signal/);
assert.throws(() => { s.value = "3" }, /Type mismatch: expected number, got string/);
```

---

## 4. Subscriptions

### `.subscribe(callback)`

Registers a callback to be invoked when the signal’s value changes.

- **Immediate Invocation**: The callback is invoked immediately with the current `.value` unless the value is `null` or `undefined` (not applicable due to constructor constraints).
- **Change Detection**: Subsequent invocations occur only when the new value differs (`!==`) from the previous value.
- **Return Value**: Returns an `unsubscribe` function that removes the callback from the subscriber list.
- **Callback Signature**: `callback(value)`, where `value` is the signal’s current value.
- **Thread Safety**: Callbacks are executed synchronously in the order they were subscribed.

**Example:**

```javascript
const name = new Signal("Alice");
const log = [];
const unsubscribe = name.subscribe((val) => log.push(`New name: ${val}`));
assert.deepStrictEqual(log, ["New name: Alice"]); // Immediate
name.value = "Bob";                              // Logs: New name: Bob
name.value = "Bob";                              // No-op
unsubscribe();
name.value = "Charlie";                          // No log
assert.deepStrictEqual(log, ["New name: Alice", "New name: Bob"]);
```

**Test Snippet:**

```javascript
let result = null;
const s = new Signal(1);
const unsubscribe = s.subscribe((v) => (result = v));
assert.strictEqual(result, 1); // Immediate
s.value = 2;
assert.strictEqual(result, 2);
unsubscribe();
s.value = 3;
assert.strictEqual(result, 2); // Unchanged
```

---

## 5. Signal Freezing

### `.#frozen`

A private boolean field indicating whether the signal is mutable.

- **Default**: `false` for new signals created via `new Signal()`.
- **Immutable Signals**: Signals created via `.map()` or `.merge()` have `.#frozen = true`.
- **Behavior**: Attempts to set `.value` on a frozen signal throw an `Error`: `"Cannot set value on frozen signal"`.

**Example:**

```javascript
const s = new Signal(true);
assert.strictEqual(s.#frozen, false);
s.#frozen = true;
assert.throws(() => { s.value = false }, /Cannot set value on frozen signal/);
```

**Test Snippet:**

```javascript
const s = new Signal(1);
s.#frozen = true;
assert.throws(() => { s.value = 2 }, /Cannot set value on frozen signal/);
```

---

## 6. Mapping Signals

### `.map(callback)`

Creates a new **frozen** signal whose value is derived from the parent signal’s value via `callback`.

- **Callback Signature**: `callback(parentValue) → newValue`.
- **Initial Value**: Computed as `callback(parent.value)`.
- **Reactivity**: Updates automatically when the parent signal’s value changes.
- **Immutability**: The derived signal is frozen (`.#frozen = true`).
- **Graph Tracking**: The parent signal adds the derived signal to its `.#children` Set.
- **Error Handling**: If `callback` throws, the error is caught and passed to `.onError` (see Section 10).

**Example:**

```javascript
const temperatureC = new Signal(25);
const temperatureF = temperatureC.map((c) => (c * 9) / 5 + 32);
assert.strictEqual(temperatureF.value, 77);
assert.strictEqual(temperatureF.#frozen, true);
temperatureC.value = 30;
assert.strictEqual(temperatureF.value, 86);
assert.throws(() => { temperatureF.value = 0 }, /Cannot set value on frozen signal/);
```

**Test Snippet:**

```javascript
const base = new Signal(10);
const derived = base.map((x) => x + 1);
assert.strictEqual(derived.value, 11);
assert.strictEqual(derived.#frozen, true);
base.value = 20;
assert.strictEqual(derived.value, 21);
assert.throws(() => { derived.value = 100 }, /Cannot set value on frozen signal/);
assert.strictEqual(base.#children.has(derived), true);
```

---

## 7. Merging Signals

### `.merge(...signals)`

Creates a new **frozen** signal that combines the values of the calling signal and additional signals.

- **Input**: One or more signals (including the caller).
- **Output Value**: An array `[this.value, signal1.value, signal2.value, ...]`.
- **Reactivity**: Updates when any source signal’s value changes (using `!==`).
- **Callback Signature**: Subscribers receive multiple arguments: `callback(value1, value2, ...)`.
- **Immutability**: The merged signal is frozen (`.#frozen = true`).
- **Graph Tracking**: Each source signal adds the merged signal to its `.#children` Set.
- **Constraints**:
  - At least one additional signal must be provided.
  - All arguments must be instances of `Signal`.
  - Throws an `Error`: `"At least one signal required for merge"` if no arguments are provided.
  - Throws an `Error`: `"All arguments must be signals"` if any argument is not a `Signal`.

**Example:**

```javascript
const a = new Signal(1);
const b = new Signal(2);
const merged = a.merge(b);
assert.deepStrictEqual(merged.value, [1, 2]);
assert.strictEqual(merged.#frozen, true);
const log = [];
merged.subscribe((valA, valB) => log.push([valA, valB]));
a.value = 10;
assert.deepStrictEqual(log, [[1, 2], [10, 2]]);
b.value = 20;
assert.deepStrictEqual(log, [[1, 2], [10, 2], [10, 20]]);
```

**Test Snippet:**

```javascript
const x = new Signal("x");
const y = new Signal("y");
const z = new Signal("z");
const group = x.merge(y, z);
assert.deepStrictEqual(group.value, ["x", "y", "z"]);
let captured = null;
group.subscribe((a, b, c) => (captured = [a, b, c]));
x.value = "X";
assert.deepStrictEqual(captured, ["X", "y", "z"]);
assert.throws(() => x.merge(), /At least one signal required for merge/);
assert.throws(() => x.merge(y, "z"), /All arguments must be signals/);
```

---

## 8. Signal Relationship Graph

Each signal maintains a private `.#children` Set to track derived signals created via `.map()` or `.merge()`.

- **Purpose**: Enables recursive updates and debugging via `.inform()`.
- **Behavior**:
  - When a signal’s value changes, it notifies its children to recompute their values.
  - Children are added to `.#children` during `.map()` or `.merge()`.
- **Constraints**: Only signals created via `.map()` or `.merge()` are added to `.#children`.

**Example:**

```javascript
const a = new Signal(1);
const b = a.map((x) => x * 2);
const c = a.merge(b);
assert.strictEqual(a.#children.has(b), true);
assert.strictEqual(a.#children.has(c), true);
assert.strictEqual(b.#children.has(c), true);
```

---

## 9. Debugging with `.inform()`

### `.inform()`

Recursively prints the signal’s structure and its children as a tree, including:

- Current `.value`.
- Frozen status (`.#frozen`).
- Child signals (from `.map()` or `.merge()`).

### Output Format

- Indented tree structure using spaces (e.g., 2 spaces per level).
- Format per signal: `Signal(value: <value>, frozen: <boolean>)`.
- Children are listed with `→` prefix.
- Uses `JSON.stringify` for values, with `undefined` for unserializable values.

**Example:**

```javascript
const a = new Signal(1);
const b = new Signal(2);
const merged = a.merge(b);
const summary = merged.map(([a, b]) => `A: ${a}, B: ${b}`);
a.inform();
```

**Output:**

```
Signal(value: 1, frozen: false)
  → Merged Signal(value: [1, 2], frozen: true)
    → Mapped Signal(value: "A: 1, B: 2", frozen: true)
```

**Test Snippet:**

```javascript
const x = new Signal("x");
const y = new Signal("y");
const z = x.merge(y).map(([a, b]) => `${a}-${b}`);
assert.doesNotThrow(() => x.inform());
```

---

## 10. Error Handling

### Subscriber Error Isolation

- All subscriber callbacks are executed in a `try-catch` block.
- If a callback throws:
  - The error is caught and does not affect other subscribers.
  - The error is logged to `console.error` with the message: `"Subscriber error: <error.message>"`.
  - If `.onError` is defined, the error and the current value are passed to it.

### `.onError(callback)`

- Registers a callback to handle errors from subscribers or derived signals.
- Signature: `callback(error, value)`, where:
  - `error` is the thrown `Error`.
  - `value` is the signal’s current value.
- Only one `.onError` callback can be registered; subsequent calls overwrite the previous.

**Example:**

```javascript
const s = new Signal(1);
s.onError((err, value) => {
  console.error(`Error for value ${value}: ${err.message}`);
});
s.subscribe((v) => {
  if (v === 2) throw new Error("Bad value");
});
s.value = 2; // Logs: Error for value 2: Bad value
```

**Test Snippet:**

```javascript
let caught = null;
const s = new Signal(0);
s.onError((err, value) => (caught = `${err.message} at ${value}`));
s.subscribe((v) => {
  if (v === 1) throw new Error("fail");
});
s.value = 1;
assert.strictEqual(caught, "fail at 1");
```

---

## 11. Disposal

### `.dispose()`

Cleans up the signal by:

- Removing all subscribers.
- Disconnecting from parent signals (if derived via `.map()` or `.merge()`).
- Removing itself from parent signals’ `.#children` Sets.
- Marking the signal as disposed (preventing further use).

### Behavior

- After `.dispose()`:
  - `.subscribe()` throws: `"Cannot subscribe to disposed signal"`.
  - `.value` setter throws: `"Cannot set value on disposed signal"`.
  - `.map()` and `.merge()` throw: `"Cannot derive from disposed signal"`.
- `.dispose()` is idempotent (calling multiple times has no additional effect).

**Example:**

```javascript
const s = new Signal(1);
s.dispose();
assert.throws(() => s.subscribe(() => {}), /Cannot subscribe to disposed signal/);
assert.throws(() => { s.value = 2 }, /Cannot set value on disposed signal/);
```

**Test Snippet:**

```javascript
const s = new Signal(1);
s.dispose();
assert.throws(() => s.subscribe(() => {}), /Cannot subscribe to disposed signal/);
s.dispose(); // Idempotent, no error
assert.throws(() => s.map((x) => x), /Cannot derive from disposed signal/);
```

---

## 12. Deliverables

### Implementation

- `Signal` class with:
  - `.value` (getter/setter)
  - `.subscribe(callback)`: Returns `unsubscribe` function.
  - `.onError(callback)`: Registers error handler.
  - `.dispose()`: Cleans up the signal.
  - `.map(callback)`: Creates derived signal.
  - `.merge(...signals)`: Combines signals.
  - `.inform()`: Prints signal tree.
- Private members:
  - `.#frozen`: Boolean indicating immutability.
  - `.#children`: Set of derived signals.
  - `.#subscribers`: Set of subscriber callbacks.
  - `.#disposed`: Boolean indicating disposal state.

### Documentation (`README.md`)

- **Introduction**: Overview of signals and reactivity.
- **API Reference**: Detailed descriptions of all methods and properties.
- **Usage Examples**: Practical examples for `.subscribe`, `.map`, `.merge`, `.inform`, and `.dispose`.
- **Behavior Notes**: Explanation of type safety, immutability, subscription immediacy, and error handling.

### Test Coverage (`test.js`)

- **Core Functionality**:
  - Value updates and type enforcement.
  - Subscription immediacy and change detection.
  - Frozen signal constraints.
- **Derived Signals**:
  - `.map()` value propagation and immutability.
  - `.merge()` array construction and multi-argument subscriptions.
- **Error Handling**:
  - Subscriber error isolation.
  - `.onError` callback invocation.
- **Disposal**:
  - Subscriber cleanup.
  - Parent-child disconnection.
- **Debugging**:
  - `.inform()` output format and tree structure.
- **Edge Cases**:
  - Empty merge attempts.
  - Invalid types or values.
  - Disposed signal operations.

---

## 13. Additional Notes

- **Type Enforcement**: Uses `typeof` for primitives and `instanceof`/`Array.isArray` for objects/arrays to ensure consistency.
- **Performance**: Subscription and child updates are synchronous and optimized for minimal overhead.
- **Extensibility**: The system supports future extensions (e.g., computed signals, batch updates) without breaking existing APIs.
- **Thread Safety**: Synchronous execution avoids race conditions in single-threaded JavaScript environments.
