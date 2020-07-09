# `Map.prototype.emplace`

ECMAScript proposal and reference implementation for `Map.prototype.emplace`.

**Author:** Brad Farias (GoDaddy)

**Champion:** Erica Pramer (GoDaddy)

**Stage:** 2

## Motivation

Adding and updating values of a Map are tasks that developers often perform 
in conjunction. There are currently no `Map` prototype methods for either of
those two things, let alone a method that does both. The workarounds involve
multiple lookups and developer inconvenience.

## Solution: `emplace`

We propose the addition of a method that will add a value to a map if the map
does not already have something at `key`, and will also update an existing 
value at `key`. 
It’s worthwhile having this API for the average case to cut down on lookups.
It is also worthwhile for developer convenience and expression of intent.

## Examples & Proposed API

The following examples would all be optimized and made simpler by `emplace`.
The proposed API allows a developer to do one lookup and update in place:

```js
// given counts is a Map of object => id
counts.emplace(key, {
  insert(key, map) {
    return 0;
  },
  update(existing, key, map) {
    return existing + 1;
  }
});
```

### Normalization of values during insertion

Currently you would need to do 3 lookups:

```js
if (!map.has(key)) {
  map.set(key, value);
}
map.get(key).doThing();
```

With this proposal:

```js
map.emplace(key, {
  insert: () => value
}).doThing();
```

### Either update or insert for a specific key
You might get new data and want to calculate some aggregate if the key exists,
but just insert if it's the first value at that key.

```js
// two lookups
old = map.get(key);
if (!old) {
  map.set(key, value);
} else {
  map.set(key, old => updated);
}
```

With this proposal:

```js
map.emplace(key, {
  update: () => updated,
  insert: () => value
});
```

### Just insert if missing

You might omit an update if you're handling data that doesn't change, but
can still be appended.

```js
// two lookups
if (!map1.has(key)) {
  map1.set(key, value);
}
```

With this proposal:

```js
map.emplace(key, {
  insert: () => value
});
```

### Just update if present

You might want to omit an insert if you want to perform a function on
all existing values in a Map (ex. normalization).

```js
// three lookups
if (map.has(key)) {
  old = map.get(key);
  updated = old.doThing();
  map.set(key, updated);
}
```

With this proposal:

```js
if (map.has(key)) {
  map.emplace(key, {
    update: (old) => old.doThing()
  });
}
```

## Implementations in other languages

Similar functionality exists in other languages.

**Java**

* [`computeIfPresent`](https://docs.oracle.com/javase/9/docs/api/java/util/Map.html#computeIfPresent-K-java.util.function.BiFunction-) remaps existing entry
* [`computeIfAbsent`](https://docs.oracle.com/javase/9/docs/api/java/util/Map.html#computeIfAbsent-K-java.util.function.Function-) insert if empty. computes
the insertion value with a mapping function

**C++**

* [`emplace`](https://en.cppreference.com/w/cpp/container/map/emplace) inserts if missing
* [`map[] assignment opts`](https://en.cppreference.com/w/cpp/container/map/operator_at) inserts if missing
at `key` but also returns a value if it exists at `key`
* [`insert_or_assign`](https://en.cppreference.com/w/cpp/container/map/insert_or_assign) inserts if missing. updates existing value by replacing with a 
specific new one, not by applying a function to the existing value

**Rust**

* [`and_modify`](https://doc.rust-lang.org/std/collections/hash_map/enum.Entry.html#method.and_modify) Provides in-place mutable access to an occupied entry
* [`or_insert_with`](https://doc.rust-lang.org/std/collections/hash_map/enum.Entry.html#method.or_insert_with) inserts if empty. insertion value comes from
a mapping function

**Python**

* [`setDefault`](https://docs.python.org/3/library/stdtypes.html#dict.setdefault)
Performs a `get` and an `insert`

**Elixir**

* [`Map.update/4`](https://hexdocs.pm/elixir/Map.html#update/4) Updates the item with given function if key exists, otherwise inserts given initial value 

## FAQ

### Is the goal to simplify the API or to optimize it?

  - This proposal seeks to simplify expressing intent for programmers, and
  should ease optimization without complex analysis. For engines without
  complex analysis like IOT VMs this should see wins by avoiding multiple
  entry lookups, at potential call stack cost.

### Why not use existing JS engines that optimize by coalescing the lookup and mutation?

  - This does not cover all patterns (of which there are many), things such as
  ordering can cause the optimization to fail.

  ```mjs
  if (!x.has(a)) x.set(a, []);
  if (!y.has(b)) y.set(b, []);
  x.get(a).push(1);
  y.get(b).push(2);
  ```

### Why error if the key doesn't exist and no `insert` handler is provided.

  - This keeps the return type constrained to the union of insert and update without adding `undefined`. This alleviates a variety of static checker errors from code such as the following.

  ```mjs
  // map is a Map of object values
  let x;
  if (map.has(key)) {
    x = map.get(key); // can return undefined
  } else {
    x = {};
    map.set(key, x);
  }
  // x's type is `undefined | object` 
  ```

  The proposal could guarantee that the type does not include `undefined`:

  ```mjs
  // map is a Map of object values
  let x = map.emplace(key, {
    insert: () => { return {}; }
  });
  // x's type is `object`
  ```

### Why use functions instead of values for the parameters?

  - You may want to apply a factory function when inserting to avoid costs of
  potentially heavy allocation, or the key may be determined at insertion time.

  ```mjs
  // an example of when eager allocation of the value
  // is undesirable
  const sharedRequests = new Map();
  function request(url) {
    return sharedRequests.emplace(url, {
      insert: () => {
        return fetch(url).then(() => {
          sharedRequests.delete(url);
        });
      }
    });
  }
  ```

  - When updating, we will be able to perform a function on the existing value
  instead of just replacing the value. The action may also cause mutation or
  side-effects, which would want to be avoided if not updating.

  ```mjs
  const eventCounts = new Map();
  obj.onevent(
    (eventName) => {
      // this API allows working with value type and primitive values
      eventCounts.emplace(eventName, {
        update: (n) => n + 1,
        insert: () => 1
      });
    }
  );
  ```

  This is important as primitives like [BigInt], [Records, and Tuples](Records and Tuples), etc. are added to the language. This API should continue to be able to handle and work with such values as they are added.

### Why are we calling this `emplace`?

  - `updateOrInsert` and `insertOrUpdate` seem too wordy.
    - in the case that only an `insert` operation is provided it will do neither update nor insert.
  - `upsert` was seen as too unique a term and the ordering was problematic as there was a desire to focus on insertion.
    - ~~It is a combination of "update" & "insert" that is already used in other programming situations and many SQL variants use that exact term.~~
  - `emplace` matches a naming precedent from C++.

### What happens during re-entrancy?

  - Other methods like Array methods while iterating using a higher order function do not re-iterate if mutated in a re-entrant manner. This method will modify the underlying storage cell that contains the existing value and any mutation of the map will act on new storage cells if that cell is removed from the map. This method will not perform a second lookup if the storage cell in the collection for the key is replaced with a new one. 
  - See [issue #9] for more.

## Specification

* [Ecmarkup source](https://github.com/tc39/proposal-upsert/blob/master/spec.emu)
* [HTML version](https://tc39.es/proposal-upsert/)

## Polyfill

A polyfill is available in the [core-js](https://github.com/zloirock/core-js) library. You can find it in the [ECMAScript proposals section](https://github.com/zloirock/core-js#mapupsert).

[BigInt]: https://tc39.es/ecma262/#sec-terms-and-definitions-bigint-value
[Records and Tuples]: https://github.com/tc39/proposal-record-tuple
[issue #9]: https://github.com/tc39/proposal-upsert/issues/9#issuecomment-552490289
