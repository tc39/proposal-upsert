# `Map.prototype.emplace`

ECMAScript proposal and reference implementation for `Map.prototype.emplace`.

**Author:** Daniel Minor (Mozilla)

**Champion:** Daniel Minor (Mozilla)

**Original Author:** Brad Farias (GoDaddy)

**Former Champion:** Erica Pramer (GoDaddy)

**Stage:** 2

## Motivation

A common problem when using a `Map` is how to handle doing an update when
you're not sure if the key already exists in the `Map`. This can be handled
by first checking if the key is present, and then inserting or updating
depending upon the result, but this is both inconvienent for the developer,
and less than optimal, because it requires multiple lookups in the `Map`
that could otherwise be handled in a single call.

## Solution: `emplace`

We propose the addition of a method that will return the value associated
with `key` if it is already present in the dictionary, and otherwise insert
the `key` with the provided default value, and then return that value.

We also propose adding a new collection type, `DefaultMap` that will function
like `Map`, except that it will take a callback that will be called in the case
of a missing value, with the return value of that function inserted in the map
and returned to the user.

Although we expect that `DefaultMap` will be used in most cases, `emplace` is
useful in the case where the default value depends upon more information than
the `key` itself.

Earlier versions of this proposal had the `emplace` method provide two callbacks,
one for `insert` and the other for `update`, however the current champion thinks
that the get / insert if necessary is a sufficiently common usecase that it makes
sense to focus on it, rather than trying to create an API with maximum flexibility.
It also strongly follows precedent from other languages, in particular Python.

## Examples & Proposed API

TODO: Design question: Should the callback function take the `key` as an argument?

### Handling default values

Using `emplace` simplifies handling default values because it will not overwrite
an existing value.

```js
// Currently
let prefs = new getUserPrefs();
if (!prefs.has("useDarkmode")) {
  prefs.set("useDarkmode", true); // default to true
}

// Using emplace
let prefs = new getUserPrefs();
prefs.emplace("useDarkmode", true); // default to true
```

By using `emplace`, default values can be applied at different times, with the
assurance that later defaults will not overwrite an existing value. This is a
usecase for which `DefaultMap` is not as suitable, because it only allows a
single default value.

For example, in a situation where there are user preferences, operating system
preferencs, and application defaults, we can use emplace to apply the user
preferences, and then the operating system preferences, and then the application
defaults, without worrying about overwriting the user's preferences.

### Grouping data incrementally

A typical usecase is grouping data based upon key as new values become available.
This is simplified by being able to specify a default value rather than having to
check for whether the key is already present in the `Map` before trying to update.

```js
// Currently
let grouped = new Map();
for (let [key, ...values] of data) {
  if (grouped.has(key)) {
    grouped.get(key).push(values);
  } else {
    grouped.set(key, values);
  }
}

// Using emplace
let grouped = new Map();
for (let [key, ...values] of data) {
  grouped.emplace(key, []).push(values);
}

// Using DefaultMap
let grouped = new DefaultMap(() => []);
for (let [ key, ...values ] of data) {
  grouped.get(key).push(values);
}
```

It's true that a common usecase for this pattern is already covered by
`Map.groupBy`. However, that method requires that all data be available
prior to building the groups; using `emplace` or `DefaultMap` would
allow the Map to be built and used incrementally. It also provides
flexibility to work with data other than objects, such as the array
example above.

### Maintaining a counter

Another common use case is maintaining a counter associated with a
particular key. Using `emplace` or `DefaultDict` makes this more
concise, and is the kind of access and then mutate pattern that is
easily optimizable by engines. [TODO: Verify this claim!]

```js
// Currently
let counts = new Map();
if (counts.has(key)) {
  counts.set(key, counts.get(key) + 1);
} else {
  counts.set(key, 1);
}

// Using emplace
let counts = new Map();
counts.set(key, m.emplace(key, 0) + 1);

// Using DefaultMap
let counts = new DefaultMap(() => 0);
counts.set(counts.get(key) + 1);
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

* [`setdefault`](https://docs.python.org/3/library/stdtypes.html#dict.setdefault)
Performs a `get` and an `insert`
* [`defaultdict`](https://docs.python.org/3/library/collections.html#defaultdict-objects)
A subclass of `dict` that takes a callback function that is used to construct missing values on `get`.

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

  ```js
  if (!x.has(a)) x.set(a, []);
  if (!y.has(b)) y.set(b, []);
  x.get(a).push(1);
  y.get(b).push(2);
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

  ```js
  let entry1 = map.mutableEntry(key);
  let entry2 = map.mutableEntry(key);
  entry2.remove();
  entry1.insertIfMissing(0);
  ```

### Why are we calling this `emplace` and `DefaultMap`?

  - `upsert` was seen as too unique a term and the ordering was problematic as there was a desire to focus on insertion.
    - ~~It is a combination of "update" & "insert" that is already used in other programming situations and many SQL variants use that exact term.~~
  - `emplace` matches a naming precedent from C++, although the semantics are closer to `operator[]` than `emplace`.
  - `DefaultMap` matches a naming precedent from Python.

### What happens during re-entrancy?

  - TODO: We should double check this with the proposed API design.
  - Other methods like Array methods while iterating using a higher order function do not re-iterate if mutated in a re-entrant manner. This method will modify the underlying storage cell that contains the existing value and any mutation of the map will act on new storage cells if that cell is removed from the map. This method will not perform a second lookup if the storage cell in the collection for the key is replaced with a new one.
  - See [issue #9] for more.

## Specification

TODO: Update the specifications to reflect the new direction for the proposal.

* [Ecmarkup source](https://github.com/tc39/proposal-upsert/blob/master/spec.emu)
* [HTML version](https://tc39.es/proposal-upsert/)

## Polyfill

The proposal is trivially polyfillable:

```js
Map.prototype.emplace = function (key, defaultValue) {
  if (this.has(key)) {
    return this.get(key);
  }
  this.set(key, defaultValue);
  return this.get(key);
};

class DefaultMap extends Map {
  constructor(callback) {
    super();
    this.callback = callback;
  }

  get(key) {
    if (this.has(key)) {
      return super.get(key);
    }
    // Todo: verify that callback is callable
    var defaultValue = this.callback();
    this.set(key, defaultValue);
    return defaultValue;
  }
}
```
