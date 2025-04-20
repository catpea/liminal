# RFC: Remotable – A Distributed, Reactive, Synchronizable, and Composable State Class
**Version:** 2.0
**Status:** Draft
**Authors:** catpea, Copilot, and the Liminal OSS community
**License:** GPL-3.0
**Last Updated:** 2025-04-20

---

## 1. Introduction

The `Remotable` class proposes a foundational primitive for next-generation distributed, reactive, synchronizable, and composable state in JavaScript. Building upon the best concepts from signals, observables, and distributed systems, Remotable enables stateful values to be synchronized across devices and applications, reactively update subscribers, and derive new state through mapping and merging. It is designed for modern networked applications, collaborative software, and real-time UIs.

Remotable unifies local and remote state handling, revision/UUID-based conflict resolution, and functional composition with a minimal, ergonomic API. It is extensible for future paradigms such as computed signals, CRDTs, and graph-based state management. This document is written for future AI and human developers, with friendly MDN-style explanations, practical code examples, and robust test snippets.

---

## 2. Goals

- **Simplicity:** Provide a single, learnable, composable API for both local and distributed state.
- **Reactivity:** Enable synchronous, immediate subscriptions and efficient change notification via debouncing.
- **Synchronization:** Integrate revision/UUID tracking for deterministic distributed conflict resolution.
- **Composition:** Support `.map()` for derived signals and `.merge()` for aggregation, enabling expressive data flows and dependency graphs.
- **Safety:** Enforce JSON-serializability for values, with early and descriptive error reporting.
- **Extensibility:** Reserve hooks for future features (dispose, advanced merging, custom conflict resolution, etc.).
- **Performance:** Optimize for fast synchronous updates and minimal notification overhead.
- **Interoperability:** Design for web, Node.js, and cross-language compatibility (e.g., via JSON wire protocols).
- **Observability:** Include debugging and introspection facilities (e.g., `.inform()` to visualize state graphs).

---

## 3. Class Construction

```js
const state = new Remotable(initialValue, revision = 1, revisionUuid = generateTimestampedUUID());
```

### Parameters

- **`initialValue`**: Any JSON-serializable value (primitive, array, object...). Defaults to `undefined`. Must not be a function, symbol, or circular structure.
- **`revision`**: Integer, default `1`. Must be `>= 1` and strictly increasing for local/remote updates.
- **`revisionUuid`**: String, default: generated via `generateTimestampedUUID()` (e.g., `168999999-abc123`); must be non-empty and unique for each revision.

### Example

```js
const counter = new Remotable(0);
const user = new Remotable({ name: "Catpea" }, 3, "abc-123");
```

### Validation

- Throws `Error` if `revision` is less than 1, not a number, or not an integer.
- Throws `Error` if `revisionUuid` is not a non-empty string.

#### Test Snippet

```js
assert.throws(() => new Remotable(0, 0), /Revision must be 1 or greater/);
assert.throws(() => new Remotable(0, NaN), /Revision must be an integer/);
assert.throws(() => new Remotable(0, 1, ""), /Revision UUID must be a non-empty string/);
```

---

## 4. Value Management

### `.value` Getter

Returns the current unserialized value. No side effects, always returns the latest state.

```js
const state = new Remotable(123);
console.log(state.value); // 123
```

### `.value` Setter

- Accepts only JSON-serializable values.
- If the value changes (by deep serialization, not just reference), increments revision and updates UUID.
- Notifies all listeners (debounced).
- Throws if `.frozen` is `true` (derived Remotables are always frozen).
- Throws `TypeError` if value is not JSON-serializable.

#### Example

```js
const state = new Remotable(5);
state.value = 10; // OK
state.value = 10; // No-op (no change)
```

#### Test Snippet

```js
const s = new Remotable([1, 2]);
s.value = [1, 2]; // No-op, same by JSON.stringify
assert.strictEqual(s.revision, 1);
assert.throws(() => { s.value = function(){}; }, /JSON-serializable/);
s.freeze();
assert.throws(() => { s.value = 42; }, /frozen/);
```

---

## 5. Subscriptions

### `.subscribe(listener)`

Registers a function to be called on value changes.
Listener signature: `(newValue, oldValue, revision, revisionUuid)`

- Fires immediately with current value (oldValue is `null`).
- Returns an unsubscribe function.
- Listener exceptions are caught and ignored; do not affect state or other listeners.

#### Example

```js
const state = new Remotable("hello");
const unsub = state.subscribe((newVal, oldVal, rev, uuid) => {
  console.log(`Now: ${newVal}, Was: ${oldVal}, rev: ${rev}, uuid: ${uuid}`);
});
state.value = "world";
// Output: Now: world, Was: hello, rev: 2, uuid: ...
unsub();
```

#### Test Snippet

```js
let called = false;
const s = new Remotable(1);
const unsub = s.subscribe(() => { called = true; });
unsub();
s.value = 2;
assert.strictEqual(called, true); // Only initial call
```

---

## 6. Debouncing

### `.debounce` Getter/Setter

- Controls notification delay in ms (default: 10).
- Must be a non-negative integer.
- All change notifications are batched and fired after the debounce interval.

#### Example

```js
const state = new Remotable(1);
state.debounce = 50; // Wait 50ms before notifying
```

#### Test Snippet

```js
const s = new Remotable(1);
s.debounce = 20;
let n = 0;
s.subscribe(() => { n++; });
s.value = 2;
s.value = 3;
setTimeout(() => assert.strictEqual(n, 2), 50);
```

---

## 7. Remote Synchronization

### `.remote(remoteRevision, remoteRevisionUuid, newValue)`

Accepts updates from a remote source (e.g., another device, server, peer).

- If `remoteRevision > state.revision`, update.
- If same revision but `remoteRevisionUuid > state.revisionUuid`, update (alphanumeric comparison).
- Otherwise, ignore (stale or duplicate).
- Only updates if value changes (by serialization).
- Triggers debounced notification to listeners.

#### Example

```js
const state = new Remotable("a", 1, "x");
state.remote(2, "y", "b"); // Updates to "b", revision 2, uuid "y"
state.remote(2, "z", "c"); // Updates to "c", revision 2, uuid "z" (z > y)
state.remote(2, "z", "c"); // No-op (duplicate)
```

#### Test Snippet

```js
const s = new Remotable(1, 1, "a");
s.remote(2, "b", 2);
assert.strictEqual(s.value, 2);
assert.strictEqual(s.revision, 2);
s.remote(2, "b", 2); // No change
assert.strictEqual(s.value, 2);
```

---

## 8. Immutability

### `.freeze()`

Freezes the instance. After freezing, further `.value` sets throw an error. All derived Remotables from `.map()` or `.merge()` are always frozen.

### `.frozen` Getter

Returns `true` if frozen.

#### Example

```js
const state = new Remotable(123);
state.freeze();
assert.throws(() => { state.value = 456; }, /frozen/);
```

---

## 9. Mapping (Derived Remotables)

### `.map(callback)`

Creates a new **frozen** Remotable whose value is derived from this one's value via `callback`.

- The derived Remotable is **read-only** (frozen).
- Updates automatically and reactively when the parent Remotable changes.
- Parent tracks children for propagation and debugging.
- Errors in the mapping callback are caught and do not affect parent or sibling signals.
- The mapping callback receives the current value and returns the derived value.

#### Example

```js
const tempC = new Remotable(25);
const tempF = tempC.map(c => (c * 9/5) + 32);
console.log(tempF.value); // 77
tempC.value = 30;
console.log(tempF.value); // 86
assert.throws(() => { tempF.value = 0; }, /frozen/);
```

#### Test Snippet

```js
const base = new Remotable(10);
const derived = base.map(x => x + 1);
assert.strictEqual(derived.value, 11);
base.value = 20;
assert.strictEqual(derived.value, 21);
assert.throws(() => { derived.value = 100; }, /frozen/);
```

---

## 10. Merging (Aggregated Remotables)

### `.merge(...remotables)`

Creates a new **frozen** Remotable combining the values of this and other Remotables.

- Value is an array: `[this.value, ...others.values]`
- Updates whenever any source Remotable changes.
- Subscribers receive unpacked arguments: `callback(val1, val2, ...)`
- All arguments must be Remotable instances.
- Throws if no arguments are provided.

#### Example

```js
const a = new Remotable(1);
const b = new Remotable(2);
const merged = a.merge(b);
console.log(merged.value); // [1, 2]
merged.subscribe((valA, valB) => {
  console.log(valA, valB);
});
a.value = 10; // Notifies with [10, 2]
b.value = 20; // Notifies with [10, 20]
```

#### Test Snippet

```js
const x = new Remotable("x");
const y = new Remotable("y");
const z = new Remotable("z");
const group = x.merge(y, z);

assert.deepStrictEqual(group.value, ["x", "y", "z"]);
let captured = null;
group.subscribe((a, b, c) => captured = [a, b, c]);
x.value = "X";
assert.deepStrictEqual(captured, ["X", "y", "z"]);
assert.throws(() => x.merge(), /At least one Remotable required/);
```

---

## 11. Debugging and Graph Introspection

### `.inform()`

Prints a tree summary of the Remotable and its children (from `.map()` or `.merge()`).

- Each node shows value, revision, UUID, and frozen state.
- Children are indented and preceded by arrows (→).
- Useful for debugging, visualization, and understanding signal graphs.

#### Example Output

```
Remotable(value: 1, revision: 1, uuid: 168999999-abc, frozen: false)
  → Merged Remotable(value: [1, 2], revision: 1, uuid: ..., frozen: true)
    → Mapped Remotable(value: "A: 1, B: 2", revision: 1, uuid: ..., frozen: true)
```

---

## 12. Error Handling

- All errors throw descriptive messages: invalid revisions, non-serializable values, type mismatches, frozen state, etc.
- All listener exceptions are caught and ignored (user's responsibility).
- Map and merge callbacks are isolated: one error does not affect other signals or propagation.
- Setting non-serializable values throws `TypeError`.

---

## 13. Extensibility and Advanced Features

- **Graph Relationships:** Remotable tracks its children (from `.map()`/`.merge()`) for efficient propagation, disposal, and debugging.
- **Disposal:** `.dispose()` (future) will clean up listeners and parent/child links, making Remotables eligible for GC (garbage collection).
- **Custom Conflict Resolution:** Pluggable conflict strategies (future).
- **Advanced Merging:** Custom merging functions, computed signals, and CRDTs (future).
- **Persistence and Serialization:** Can be easily serialized as JSON for network sync or disk persistence.
- **Cross-platform:** Designed to work identically in browsers, Node.js, and any JS-compatible environment.

---

## 14. Full Example and Test Suite

```js
// Example: Collaborative Document State

const title = new Remotable("Untitled");
const wordCount = title.map(t => t ? t.split(/\s+/).length : 0);
const meta = new Remotable({ synced: true, revision: 1 });
const docState = title.merge(meta);

docState.subscribe((curTitle, curMeta) => {
  console.log(`Title: ${curTitle} | Synced: ${curMeta.synced} | Rev: ${curMeta.revision}`);
});

// Simulate remote update to meta
meta.remote(2, "peer-xyz", { synced: false, revision: 2 });

// Attempt to set value on derived
try { wordCount.value = 0; } catch(e) { console.log(e.message); }
```

#### Behaviour Driven Test Snippet

```js
describe('Remotable', function() {
  it('should synchronize remote changes and resolve conflicts', function() {
    const a = new Remotable("start", 1, "a");
    a.remote(2, "b", "b_val");
    assert.strictEqual(a.value, "b_val");
    a.remote(2, "z", "z_val");
    assert.strictEqual(a.value, "z_val");
    a.remote(1, "zzz", "should_ignore");
    assert.strictEqual(a.value, "z_val");
  });
  it('should support mapping and merging', function() {
    const base = new Remotable(10);
    const derived = base.map(x => x + 1);
    assert.strictEqual(derived.value, 11);
    base.value = 20;
    assert.strictEqual(derived.value, 21);
    const a = new Remotable(1), b = new Remotable(2);
    const merged = a.merge(b);
    assert.deepStrictEqual(merged.value, [1, 2]);
    a.value = 3;
    assert.deepStrictEqual(merged.value, [3, 2]);
  });
});
```

---

## 15. Glossary

- **Revision:** Monotonically increasing integer on each accepted update.
- **Revision UUID:** A unique string for each revision, used for conflict resolution.
- **Debounce:** Wait time (in ms) before notifying subscribers, to batch rapid changes.
- **Frozen:** State cannot be changed (read-only).
- **Mapped Remotable:** Derived, read-only Remotable whose value is computed from another.
- **Merged Remotable:** Read-only Remotable representing the values of multiple Remotables as an array.
- **Disposal:** Releasing all subscriptions and graph links to allow GC (not implemented yet).
