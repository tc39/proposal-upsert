# `Map.prototype.emplace`

ECMAScript proposal and reference implementation for `Map.prototype.emplace`.

**Author:** Brad Farias (GoDaddy)

**Champion:** Erica Pramer (GoDaddy)

**Stage:** 2

## Motivation

Adding and updating values of a Map are tasks that developers often perform 
in conjunction. There are currently no `Map` prototype methods for either of
those two things, let alone a method that does both. The workarounds involve
multiple lookups and developer inconvenience while avoiding encouraging code
that is surprising or is potentially error prone.

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
  map.set(key, updated);
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

### Why not have a single function that has a boolean if performing an update and the potentially existing value?

  - By naming the handlers, you can increase readability and reduce overall boilerplate. Additionally, generally there are not common workflows that have code paths that cover both updating and insertion. See the following which only seeks to insert a value if none exists:

  ```mjs
  x = map.emplace(key, (updating, value) => updating ? value : []);
  ```

  The proposal allows a handler to avoid the boilerplate condition and focus only on the relevant workflows:

  ```mjs
  x = map.emplace(key, {
    insert: () => []
  });
  ```

### Why not have a single function that inserts the value if no such key is mapped?

  - By only having a single function that inserts a variety of workflows become
  less clear. In particular, by only having insert, a usage of a default value must
  be inserted with an anti-update operation already applied.

  ```mjs
  // have to set the default value to -1, not 0
  const n = counts.emplace(key, () => -1);
  // have to perform an additional set afterwards
  counts.set(key, n + 1);
  ```

  The proposal allows a handler to avoid the odd default value and avoid the
  extra `.set`. This does still require coding logic for both, but keeps the
  intent more readable and localized.

  ```mjs
  counts.emplace(key, {
    insert: () => 0,
    update: (v) => v + 1
  });
  ```
  
  - This can also lead to insertion of values that are incomplete/invalid regarding
  the intended type of the Map:
  
  
  ```mjs
  updateNameOf(key) {
    // contacts only has objects which must have a string value
    const contact = contacts.insert(key, () => ({name:null}));
    // inserted and can be assured of same reference on .get
    contact.name = getName();
    // if getName throws, contact is incomplete/invalid
    // .name remains null
  }
  ```
  
  This can be rewritten to be less error prone by moving `getName` above
  
  ```
  updateNameOf(key) {
    // inserted and can be assured of same reference on .get
    contact.name = getName();
    // contacts only has objects which must have a string value
    const contact = contacts.insert(key, () => ({name:null}));
    contact.name = name;
  }
  ```
  
  This code is less error prone (unless contact.name fails assignment for
  example), but it still has an invalid value being inserted into the map.
  
  This can again be rewritten to be less constraint breaking by avoiding
  `insert()` entirely.
  
  ```mjs
  updateNameOf(key) {
    const name = getName();
    const existing = contacts.get(key) ?? {name:null};
    contact.name = name;
    contacts.set(key);
  }
  ```
  
  This design does avoid ever inserting the invalid value into the map, but is
  a bit confusing to read. This proposal by combinding update and insert can be
  a little clearer:
  
  ```mjs
  updateNameOf(key, name) {
    const name = getName();
    contacts.emplace(key, {
      insert: () => ({name}),
      update: (contact) => contact.name = name
    });
  }
  ```
  
  by having both operations co-located it would feel odd to try and `getName`
  after `.emplace` and no invalid value is inserted into the map.

### Why not have a single function that updates the value if no if the key is mapped?

  - By only having a single function that updates a variety of workflows
  become less clear. In particular in order to guarantee that the result type
  is not a union with `undefined` it should error if the key is not mapped.

  ```mjs
  let n;
  try {
    n = counts.emplace(key, (existing) => existing + 1);
  } catch (e) {
    // this is a fragile detection and quite hard to determine it was
    // counts.emplace that caused an error, and not something internal to
    // the update callback
    if (e instanceof MissingEntryError) {
      counts.set(key, n = 0);
    }
  }
  if (n > RETRIES) {
    // ...
  }
  ```

  A alteration to return a boolean to see if an action is taken requires
  boilerplate and reduces the utility of the return value:

  ```mjs
  let updated = counts.emplace(key, (existing) => existing + 1);
  if (!updated) {
    counts.set(key, 0);
  }
  let n = counts.get(key);
  if (n > RETRIES) {
    // ...
  }
  ```

  The proposal allows a handler to avoid the odd default value and avoid the
  extra `.set`.

  ```mjs
  let n = counts.emplace(key, {
    insert: () => 0,
    update: (v) => v + 1
  });
  if (n > RETRIES) {
    // ...
  }
  ```

### Why not have an API that exposes an Entry to collection types?

  - An Entry API is not prevented by this proposal. Explicit thought about
  re-entrancy was taken into consideration and was designed not to conflict with
  such an API. Desires for such an API should be done in a separate proposal.
  - An Entry API has much stricter implications on how implementations must
  store the backing data for a collection due to creating persistent references.
  - An Entry API is extremely complex regarding shared mutability and should be
  considered to be an extreme increase in scope to the goals of this proposal.
  See complexity such as the following about needing to think of an design an
  entire lifecycle and sharing scheme for multiple entry references:

  ```mjs
  let entry1 = map.mutableEntry(key);
  let entry2 = map.mutableEntry(key);
  entry2.remove();
  entry1.insertIfMissing(0);
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
    - in the case that only an `insert` operation is provided it may do neither update nor insert.
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
