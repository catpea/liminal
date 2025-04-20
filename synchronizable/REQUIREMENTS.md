# Synchronizable Class – Finalized Functional Specification

## 1. Introduction

This document specifies the `Synchronizable` class, a JavaScript utility for managing reactive state with synchronization capabilities. The class supports:

- **Reactive State**: Stores a value with subscription-based change notifications.
- **Synchronization**: Tracks revisions and UUIDs for conflict resolution in distributed systems.
- **Debounced Notifications**: Delays listener notifications to optimize performance.
- **Serialization**: Maintains a JSON-serialized value for stable comparisons.
- **Extensibility**: Supports future mapping to derived read-only signals.

The system ensures robust error handling, predictable conflict resolution, and efficient state management for both local and remote updates.

---

## 2. Class Construction

```javascript
const sync = new Synchronizable(value = undefined, revision = 1, revisionUuid = generateTimestampedUUID());
```

The `Synchronizable` class is instantiated with an optional value, revision number, and revision UUID.

### Parameters

- **`value`**: The initial state (default: `undefined`). Can be any JSON-serializable JavaScript value (e.g., primitives, objects, arrays).
- **`revision`**: An integer representing the initial version (default: `1`). Must be ≥ `1`.
- **`revisionUuid`**: A unique string identifier for the revision (default: generated via `generateTimestampedUUID()`).

### Constraints

- **`revision`**:
  - Must be an integer (checked via `Number.isInteger`).
  - Must be ≥ `1` (zero-indexing not supported).
  - Throws an `Error`: `"Revision must be an integer"` if not an integer or `NaN`.
  - Throws an `Error`: `"Revision must be 1 or greater (zero-indexing not supported)"` if `< 1`.
- **`revisionUuid`**:
  - If provided, must be a non-empty string.
  - If not provided, generated using `generateTimestampedUUID()`.

### Internal State

- **`#unserializedValue`**: Stores the native JavaScript value.
- **`#serializedValue`**: Stores the JSON-stringified value (`JSON.stringify(value)`).
- **`#revision`**: Stores the current revision number.
- **`#revisionUuid`**: Stores the current revision UUID.
- **`#listeners`**: A `Set` of subscriber callbacks.
- **`#children`**: A `Set` reserved for future derived signals (e.g., via `.map()`).
- **`#debounce`**: Notification delay in milliseconds (default: `10`).
- **`#debounceTimeout`**: Timer reference for debounced notifications.

**Example:**

```javascript
const sync = new Synchronizable("hello"); // value: "hello", revision: 1, revisionUuid: <generated>
const sync2 = new Synchronizable({ count: 0 }, 2, "abc123"); // Custom revision and UUID
```

**Test Snippet:**

```javascript
const sync = new Synchronizable(42);
assert.strictEqual(sync.value, 42);
assert.strictEqual(sync.revision, 1);
assert.strictEqual(typeof sync.revisionUuid, "string");
assert.throws(() => new Synchronizable(null, 0), /Revision must be 1 or greater/);
assert.throws(() => new Synchronizable(null, "1"), /Revision must be an integer/);
```

---

## 3. Value Management

### `.value` Getter

- Returns the current `#unserializedValue`.
- No side effects.

**Example:**

```javascript
const sync = new Synchronizable(10);
assert.strictEqual(sync.value, 10);
```

### `.value` Setter

Sets a new value, updates revision metadata, and notifies subscribers.

#### Conditions

- The new value is JSON-serializable.
- The serialized new value (`JSON.stringify(newValue)`) differs from the current `#serializedValue`.

#### Behavior

- If the serialized value is unchanged, the setter is a no-op (no updates or notifications).
- If changed:
  - Increments `#revision` by 1.
  - Generates a new `#revisionUuid` via `generateTimestampedUUID()`.
  - Updates `#unserializedValue` to the new value.
  - Updates `#serializedValue` to `JSON.stringify(newValue)`.
  - Schedules a debounced notification to subscribers with the new and old unserialized values, current revision, and revision UUID.
- Note: Serialization assumes stable key ordering for objects (not guaranteed by `JSON.stringify`; external sorting recommended).

**Example:**

```javascript
const sync = new Synchronizable(5);
sync.value = 6; // Updates revision, notifies subscribers
sync.value = 6; // No-op (same serialized value)
sync.value = { x: 1 }; // Updates revision, notifies subscribers
```

**Test Snippet:**

```javascript
const sync = new Synchronizable(1);
let notified = null;
sync.subscribe((newVal, oldVal, rev, uuid) => (notified = { newVal, rev, uuid }));
sync.value = 2;
await new Promise((resolve) => setTimeout(resolve, 15)); // Wait for debounce
assert.strictEqual(notified.newVal, 2);
assert.strictEqual(notified.rev, 2);
assert.strictEqual(sync.value, 2);
sync.value = 2;
await new Promise((resolve) => setTimeout(resolve, 15));
assert.strictEqual(notified.newVal, 2); // Unchanged (no notification)
```

---

## 4. Subscriptions

### `.subscribe(listener)`

Registers a callback to be invoked on value changes.

- **Callback Signature**: `listener(newValue, oldValue, revision, revisionUuid)`.
  - `newValue`: The new `#unserializedValue`.
  - `oldValue`: The previous `#unserializedValue` (or `null` for initial invocation).
  - `revision`: The current `#revision`.
  - `revisionUuid`: The current `#revisionUuid`.
- **Immediate Invocation**:
  - Invoked immediately with the current `#unserializedValue` and `null` as `oldValue`, unless `#unserializedValue` is `null` or `undefined`.
- **Debounced Notifications**:
  - Subsequent notifications are delayed by `#debounce` milliseconds.
  - Only triggered when the serialized value changes.
- **Return Value**: An `unsubscribe` function that removes the listener.
- **Thread Safety**: Callbacks are executed synchronously within the debounced timer.

**Example:**

```javascript
const sync = new Synchronizable("Alice");
const log = [];
sync.subscribe((newVal, oldVal) => log.push([newVal, oldVal]));
assert.deepStrictEqual(log, [["Alice", null]]); // Immediate
sync.value = "Bob";
await new Promise((resolve) => setTimeout(resolve, 15));
assert.deepStrictEqual(log, [["Alice", null], ["Bob", "Alice"]]);
```

**Test Snippet:**

```javascript
const sync = new Synchronizable(1);
let result = null;
sync.subscribe((newVal) => (result = newVal));
assert.strictEqual(result, 1); // Immediate
sync.value = 2;
await new Promise((resolve) => setTimeout(resolve, 15));
assert.strictEqual(result, 2);
const sync2 = new Synchronizable(null);
let called = false;
sync2.subscribe(() => (called = true));
assert.strictEqual(called, false); // No initial call for null
```

### `.unsubscribe(listener)`

Removes a listener from the `#listeners` Set.

- **Behavior**: The listener no longer receives notifications.
- **Idempotent**: Removing a non-existent listener is a no-op.
- **No side effects** on other listeners.

**Example:**

```javascript
const sync = new Synchronizable(1);
const listener = () => {};
const unsubscribe = sync.subscribe(listener);
unsubscribe();
sync.value = 2;
await new Promise((resolve) => setTimeout(resolve, 15));
// Listener not called
```

**Test Snippet:**

```javascript
const sync = new Synchronizable(1);
let called = false;
const listener = () => (called = true);
const unsubscribe = sync.subscribe(listener);
unsubscribe();
sync.value = 2;
await new Promise((resolve) => setTimeout(resolve, 15));
assert.strictEqual(called, true); // Only initial call
```

---

## 5. Remote Updates

### `.remote(remoteRevision, remoteRevisionId, newUnserializedRemoteValue)`

Applies a remote update with conflict resolution based on revision numbers and UUIDs.

#### Parameters

- **`remoteRevision`**: The revision number from the remote source.
  - Must be an integer ≥ `1`.
  - Throws an `Error`: `"Remote revision must be an integer"` if not an integer or `NaN`.
  - Throws an `Error`: `"Remote revision must be 1 or greater"` if `< 1`.
- **`remoteRevisionId`**: The UUID of the remote revision.
  - Must be a non-empty string.
  - Throws an `Error`: `"Remote revision ID must be a non-empty string"`.
- **`newUnserializedRemoteValue`**: The new value from the remote source (JSON-serializable).

#### Conflict Resolution

- **Cases**:
  - **Update**: `remoteRevision > #revision`.
    - Updates `#revision`, `#revisionUuid`, `#unserializedValue`, and `#serializedValue` if the serialized value changes.
    - Notifies subscribers (debounced) with new and old values.
  - **Conflict**: `remoteRevision === #revision` and `remoteRevisionId !== #revisionUuid`.
    - Resolves by comparing UUIDs alphanumerically.
    - If `remoteRevisionId > #revisionUuid`:
      - Updates `#revision`, `#revisionUuid`, `#unserializedValue`, and `#serializedValue` if the serialized value changes.
      - Notifies subscribers (debounced).
    - Otherwise, no changes.
  - **Duplicate**: `remoteRevision === #revision` and `remoteRevisionId === #revisionUuid`.
    - No changes (assumes "at least once" delivery).
- **Serialization**: Compares `JSON.stringify(newUnserializedRemoteValue)` with `#serializedValue` to detect changes.
- **Notifications**: Only triggered if the serialized value changes.

**Example:**

```javascript
const sync = new Synchronizable(1, 1, "uuid1");
sync.remote(2, "uuid2", 2); // Update: sets value to 2, revision to 2
sync.remote(1, "uuid3", 3); // Conflict: uuid3 > uuid1, updates to 3
sync.remote(1, "uuid1", 1); // Duplicate: no change
```

**Test Snippet:**

```javascript
const sync = new Synchronizable(1, 1, "aaa");
let notified = null;
sync.subscribe((newVal) => (notified = newVal));
sync.remote(2, "bbb", 2);
await new Promise((resolve) => setTimeout(resolve, 15));
assert.strictEqual(notified, 2);
assert.strictEqual(sync.revision, 2);
sync.remote(2, "ccc", 2); // Conflict, ccc > bbb
await new Promise((resolve) => setTimeout(resolve, 15));
assert.strictEqual(notified, 2);
sync.remote(2, "bbb", 2); // Duplicate
await new Promise((resolve) => setTimeout(resolve, 15));
assert.strictEqual(notified, 2);
assert.throws(() => sync.remote(0, "id", 1), /Remote revision must be 1 or greater/);
```

---

## 6. Debounce Control

### `.debounce` Getter

- Returns the current `#debounce` timeout in milliseconds.

**Example:**

```javascript
const sync = new Synchronizable();
assert.strictEqual(sync.debounce, 10);
```

### `.debounce` Setter

- Sets the `#debounce` timeout for notifications.
- **Constraints**:
  - Must be a non-negative integer.
  - Throws an `Error`: `"Debounce timeout must be a non-negative integer"` if invalid.
- **Behavior**: Affects future notifications (existing timers use the old value).

**Example:**

```javascript
const sync = new Synchronizable();
sync.debounce = 50;
assert.strictEqual(sync.debounce, 50);
```

**Test Snippet:**

```javascript
const sync = new Synchronizable();
sync.debounce = 20;
assert.strictEqual(sync.debounce, 20);
assert.throws(() => { sync.debounce = -1 }, /Debounce timeout must be a non-negative integer/);
assert.throws(() => { sync.debounce = 1.5 }, /Debounce timeout must be a non-negative integer/);
```

---

## 7. Revision Metadata

### `.revision` Getter

- Returns the current `#revision` (read-only).
- No side effects.

**Example:**

```javascript
const sync = new Synchronizable(null, 5);
assert.strictEqual(sync.revision, 5);
```

### `.revisionUuid` Getter

- Returns the current `#revisionUuid` (read-only).
- No side effects.

**Example:**

```javascript
const sync = new Synchronizable(null, 1, "uuid123");
assert.strictEqual(sync.revisionUuid, "uuid123");
```

**Test Snippet:**

```javascript
const sync = new Synchronizable(1, 2, "test");
assert.strictEqual(sync.revision, 2);
assert.strictEqual(sync.revisionUuid, "test");
```

---

## 8. Future Extensibility (Mapping)

### Reserved `#children`

- A `Set` for tracking derived signals (e.g., via a future `.map()` method).
- Currently unused but reserved for creating read-only signals that transform the value.
- Potential `.map()` behavior:
  - Creates a new `Synchronizable` or read-only signal.
  - Subscribes to the parent and updates the derived signal’s value.
  - Adds the derived signal to `#children`.

**Example (Conceptual):**

```javascript
const sync = new Synchronizable(10);
const mapped = sync.map((x) => x * 2); // Future derived signal
assert.strictEqual(mapped.value, 20);
assert.strictEqual(sync.#children.has(mapped), true);
```

---

## 9. Error Handling

- **Constructor**:
  - Invalid `revision`: Throws descriptive errors (see Section 2).
- **Setter**:
  - Non-serializable values: Throws a `TypeError` from `JSON.stringify`.
- **Remote Updates**:
  - Invalid `remoteRevision` or `remoteRevisionId`: Throws descriptive errors (see Section 5).
- **Debounce**:
  - Invalid `ms`: Throws `"Debounce timeout must be a non-negative integer"`.
- **Subscriber Errors**:
  - Callbacks are not wrapped in `try-catch` (user responsibility).
  - Errors in listeners do not affect other listeners or state.

**Example:**

```javascript
const sync = new Synchronizable();
assert.throws(() => { sync.debounce = -1 }, /Debounce timeout must be a non-negative integer/);
assert.throws(() => sync.remote(1, "", 1), /Remote revision ID must be a non-empty string/);
```

**Test Snippet:**

```javascript
const sync = new Synchronizable();
assert.throws(() => new Synchronizable(null, NaN), /Revision must be an integer/);
assert.throws(() => { sync.value = () => {} }, /TypeError: Converting circular structure to JSON/);
```

---

## 10. Deliverables

### Implementation

- `Synchronizable` class with:
  - `.value` (getter/setter): Manages the state.
  - `.subscribe(listener)`: Registers change listeners.
  - `.unsubscribe(listener)`: Removes listeners.
  - `.remote(remoteRevision, remoteRevisionId, newValue)`: Applies remote updates.
  - `.debounce` (getter/setter): Controls notification delay.
  - `.revision` (getter): Exposes current revision.
  - `.revisionUuid` (getter): Exposes current revision UUID.
- Private members:
  - `#unserializedValue`: Native value.
  - `#serializedValue`: JSON-stringified value.
  - `#revision`: Version number.
  - `#revisionUuid`: Version UUID.
  - `#listeners`: Set of subscriber callbacks.
  - `#children`: Set for future derived signals.
  - `#debounce`: Notification delay in milliseconds.
  - `#debounceTimeout`: Timer reference.
- Utility function:
  - `generateTimestampedUUID()`: Generates unique revision IDs.

### Documentation (`README.md`)

- **Introduction**: Overview of reactive state and synchronization.
- **API Reference**: Detailed descriptions of all methods and properties.
- **Usage Examples**: Scenarios for local updates, remote synchronization, and subscriptions.
- **Behavior Notes**: Explanation of debouncing, conflict resolution, and serialization.

### Test Coverage (`test.js`)

- **Core Functionality**:
  - Value updates and serialization-based change detection.
  - Revision and UUID updates.
  - Debounced notifications.
- **Subscriptions**:
  - Immediate invocation (excluding `null`/`undefined`).
  - Listener addition and removal.
- **Remote Updates**:
  - Update, conflict, and duplicate cases.
  - Conflict resolution via UUID comparison.
- **Debounce**:
  - Timeout configuration and validation.
  - Notification timing.
- **Error Handling**:
  - Invalid constructor parameters.
  - Non-serializable values.
  - Invalid remote update parameters.
- **Edge Cases**:
  - Same serialized value (no-op).
  - Zero or negative revisions.
  - Empty or invalid UUIDs.

---

## 11. Additional Notes

- **Serialization Stability**: `JSON.stringify` does not guarantee stable key ordering for objects. Users should implement key sorting for reliable comparisons.
- **Synchronization**: External server synchronization is the user’s responsibility, facilitated by subscribing to changes.
- **Performance**: Debouncing optimizes notification frequency; the default 10ms delay suits most use cases.
- **Extensibility**: The `#children` Set and commented `.map()` suggest future support for derived signals.
- **Thread Safety**: Synchronous listener execution and debounced timers avoid race conditions in single-threaded JavaScript.
