# Remotable: Class Decomposition, Design Rationale, and Guided Re-Assembly

## Overview

The `Remotable` class is a modern distributed state primitive for JavaScript, supporting reactivity, synchronization, composition, and immutability. To maximize clarity, maintainability, and extensibility, it can be decomposed into focused, single-responsibility components. This document provides a guided, improved breakdown and practical assembly instructions, along with design rationale, best practices, and extra guidance for future maintainers.

---

## 1. State Management: `State`

### Purpose

Manages the internal value, revision, and UUID. Handles all value validation, serialization, and ensures state changes are tracked with revision updates.

### Responsibilities

- Store and provide access to the value.
- Validate and serialize value (must be JSON-serializable).
- Manage revision and revision UUID.
- Increment revision and update UUID on change.
- **Separation:** Do not handle notifications, subscriptions, or immutability—these are external concerns.

### Guidance

- Ensure value is always valid and serializable, both on construction and when updated.
- Consider using private fields (`#value`, etc.) if targeting modern JS.
- All state changes should be atomic and internally consistent.

### Example Implementation

```js
class State {
  constructor(initialValue, revision = 1, revisionUuid = generateTimestampedUUID()) {
    this.setValue(initialValue);
    this.revision = revision;
    this.revisionUuid = revisionUuid;
  }

  setValue(value) {
    if (!this.isSerializable(value)) {
      throw new TypeError("Value must be JSON serializable");
    }
    this.value = value;
  }

  getValue() {
    return this.value;
  }

  isSerializable(value) {
    try {
      JSON.stringify(value);
      return true;
    } catch (e) {
      return false;
    }
  }

  incrementRevision() {
    this.revision += 1;
  }

  updateRevisionUuid(newUuid) {
    this.revisionUuid = newUuid;
  }
}
```

---

## 2. Subscription Management: `Subscription`

### Purpose

Handles registration, deregistration, and invocation of listeners, and manages debounced delivery of notifications.

### Responsibilities

- Add/remove listeners.
- Debounce and fire change notifications.
- Isolate errors in listeners.
- **Separation:** Should not set or mutate state; only observes and notifies.

### Guidance

- Use a Set for listeners to avoid duplicates and improve removal performance.
- Provide immediate invocation on subscribe, per Remotable RFC.
- Debounce should be configurable and robust (clear timeouts, support 0ms).

### Example Implementation

```js
class Subscription {
  constructor() {
    this.listeners = new Set();
    this.debounceTime = 10;
    this._timeout = null;
  }

  subscribe(listener) {
    this.listeners.add(listener);
    return () => this.listeners.delete(listener);
  }

  notifySubscribers(newValue, oldValue, revision, revisionUuid) {
    if (this._timeout) clearTimeout(this._timeout);
    this._timeout = setTimeout(() => {
      this.listeners.forEach((listener) => {
        try {
          listener(newValue, oldValue, revision, revisionUuid);
        } catch (e) {
          // Log or ignore errors in listeners
        }
      });
    }, this.debounceTime);
  }

  setDebounceTime(debounceTime) {
    if (debounceTime < 0 || isNaN(debounceTime)) {
      throw new Error("Debounce time must be a non-negative number.");
    }
    this.debounceTime = debounceTime;
  }
}
```

---

## 3. Immutability: `FrozenState`

### Purpose

Enforces immutability. Once frozen, prevents any further modifications to the value.

### Responsibilities

- Implement `.freeze()` to lock modifications.
- Provide `.frozen` getter for status checks.

### Guidance

- Should be queried before any value change, and throw or block if frozen.
- All derived/aggregated Remotables must start frozen.

### Example Implementation

```js
class FrozenState {
  constructor() {
    this.isFrozen = false;
  }

  freeze() {
    this.isFrozen = true;
  }

  get frozen() {
    return this.isFrozen;
  }
}
```

---

## 4. Remote Synchronization: `RemoteSync`

### Purpose

Handles synchronization with remote peers, using revision and UUID to resolve update conflicts.

### Responsibilities

- Accept updates from external sources.
- Apply updates if remote revision/UUID is "ahead".
- Use revision and UUID comparison logic per RFC.

### Guidance

- Never mutate state directly except through the State interface.
- Should not notify subscribers itself; only update State.
- Always check both revision and UUID for proper conflict resolution.

### Example Implementation

```js
class RemoteSync {
  constructor(state) {
    this.state = state;
  }

  remoteUpdate(remoteRevision, remoteUuid, newValue) {
    if (
      remoteRevision > this.state.revision ||
      (remoteRevision === this.state.revision && remoteUuid > this.state.revisionUuid)
    ) {
      this.state.setValue(newValue);
      this.state.incrementRevision();
      this.state.updateRevisionUuid(remoteUuid);
    }
  }
}
```

---

## 5. Derived and Aggregated State: `DerivedState`

### Purpose

Provides `.map()` for derived, read-only states, and `.merge()` for aggregated Remotables. Derived/merged Remotables are always frozen and update reactively.

### Responsibilities

- Create mapped (transformed) Remotables.
- Create merged (aggregated) Remotables.
- Ensure all derived Remotables are frozen and read-only.
- Propagate parent changes reactively.

### Guidance

- Use subscriptions to observe parent(s) and update derived value.
- Always throw on `.value = ...` attempts for derived Remotables.
- Make sure merged Remotable only accepts Remotable instances; validate inputs.

### Example Implementation

```js
class DerivedState {
  constructor(parent, callback) {
    this.parent = parent;
    this.callback = callback;
    this.value = this.callback(this.parent.getValue());
    this.parent.subscribe(() => {
      this.value = this.callback(this.parent.getValue());
    });
  }

  map(callback) {
    return new DerivedState(this, callback);
  }

  merge(...remotables) {
    if (!remotables.length) throw new Error("At least one Remotable required for merge");
    const all = [this, ...remotables];
    const getValues = () => all.map(r => r.value);
    const merged = new DerivedState(this, getValues);
    all.forEach(r => r.subscribe(() => merged.value = getValues()));
    return merged;
  }

  get frozen() {
    return true;
  }
}
```

---

## 6. Assembling the Complete `Remotable` Class

### How to Compose

- `Remotable` composes (holds) all subcomponents.
- It exposes a friendly API: `value`, `subscribe`, `freeze`, `frozen`, `remote`, `map`, `merge`, etc.
- All mutation is blocked if frozen.
- Subscriptions and debouncing are handled through the `Subscription` class.
- Synchronization is handled through `RemoteSync` and only modifies state as per RFC rules.
- Derived/merged Remotables are always frozen and update reactively.

### Example Implementation

```js
class Remotable {
  constructor(initialValue, revision = 1, revisionUuid = generateTimestampedUUID()) {
    this.state = new State(initialValue, revision, revisionUuid);
    this.subscription = new Subscription();
    this.frozenState = new FrozenState();
    this.remoteSync = new RemoteSync(this.state);
    this.derivedState = null;
  }

  get value() {
    return this.state.getValue();
  }

  set value(newValue) {
    if (this.frozenState.frozen) {
      throw new Error("Cannot modify a frozen Remotable.");
    }
    const oldValue = this.state.getValue();
    this.state.setValue(newValue);
    this.state.incrementRevision();
    this.subscription.notifySubscribers(
      this.state.getValue(),
      oldValue,
      this.state.revision,
      this.state.revisionUuid
    );
  }

  subscribe(listener) {
    // Immediate call for current value as per RFC
    listener(this.value, null, this.state.revision, this.state.revisionUuid);
    return this.subscription.subscribe(listener);
  }

  set debounceTime(ms) {
    this.subscription.setDebounceTime(ms);
  }

  freeze() {
    this.frozenState.freeze();
  }

  get frozen() {
    return this.frozenState.frozen;
  }

  remote(remoteRevision, remoteRevisionUuid, newValue) {
    this.remoteSync.remoteUpdate(remoteRevision, remoteRevisionUuid, newValue);
    this.subscription.notifySubscribers(
      this.state.getValue(),
      this.state.getValue(),
      this.state.revision,
      this.state.revisionUuid
    );
  }

  map(callback) {
    this.derivedState = new DerivedState(this, callback);
    return this.derivedState;
  }

  merge(...remotables) {
    return this.derivedState.merge(...remotables);
  }
}
```

---

## 7. Best Practices and Additional Guidance

- **Encapsulation:** Prefer private fields (`#field`) and methods for internal state, especially if targeting ES2022+.
- **Validation:** Always validate inputs and provide descriptive errors (especially for serialization, freezing, and merges).
- **Testing:** Write isolated tests for each subcomponent. Use integration tests for the full Remotable.
- **Performance:** Minimize redundant serialization and notification. Use debouncing for high-frequency updates.
- **Extensibility:** New features (e.g., custom conflict resolution or advanced merges) should be implemented by subclassing or extending the relevant component.
- **Composition:** Derived and merged Remotables should update reactively and remain immutable.
- **Documentation:** Document each method, side-effect, and error clearly.

---

## 8. Summary Table

| Subclass       | Responsibility                       | Key Methods                | Guidance |
|----------------|--------------------------------------|----------------------------|----------|
| `State`        | Value, revision, UUID management     | `setValue`, `getValue`     | Internal only |
| `Subscription` | Listener management, debouncing      | `subscribe`, `notify`      | Never mutate state |
| `FrozenState`  | Immutability enforcement             | `freeze`, `frozen`         | Query before mutate |
| `RemoteSync`   | Distributed update/conflict handling | `remoteUpdate`             | RFC conflict rules |
| `DerivedState` | Mapping/merging of Remotables        | `map`, `merge`             | Always frozen |
| `Remotable`    | User-facing API, composition         | `value`, `subscribe`, etc. | Expose only unified API |

---

## 9. Example Usage

```js
// Create a new Remotable
const counter = new Remotable(0);

// Subscribe to changes
counter.subscribe((newVal, oldVal, revision, revisionUuid) => {
  console.log(`Value changed from ${oldVal} to ${newVal}, revision: ${revision}, uuid: ${revisionUuid}`);
});

// Change the value and notify listeners
counter.value = 1;

// Freeze the Remotable
counter.freeze();
try {
  counter.value = 2;  // Throws error
} catch (e) {
  console.error(e.message);
}

// Derived Remotable (mapping)
const doubleCounter = counter.map(value => value * 2);
console.log(doubleCounter.value);  // 2

// Change the original counter's value
counter.value = 2;
console.log(doubleCounter.value);  // 4

// Sync with remote update
counter.remote(3, 'remote-uuid-123', 10);
console.log(counter.value);  // 10
```

---

## 10. Final Notes

- **The modular breakdown allows each concern to evolve independently.**
- **Follow the RFC for edge case and error handling.**
- **Use `.inform()` or similar for debugging and graph visualization.**
- **Test, document, and design for distributed, reactive, and composable state.**

---
