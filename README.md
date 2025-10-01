# Proposal Upsert

ECMAScript proposal and reference implementation for `Map.prototype.getOrInsert`, `Map.prototype.getOrInsertComputed`,
`WeakMap.prototype.getOrInsert`, and `WeakMap.prototype.getOrInsertComputed`.

**Authors:** Daniel Minor (Mozilla) Lauritz Thoresen Angeltveit (Bergen) Jonas Haukenes (Bergen) Sune Lianes (Bergen) Vetle Larsen (Bergen) Mathias Hop Ness (Bergen)

**Champion:** Daniel Minor (Mozilla)

**Original Author:** Brad Farias (GoDaddy)

**Former Champion:** Erica Pramer (GoDaddy)

**Stage:** 3

## Motivation

A common problem when using a `Map` or `WeakMap` is how to handle doing an update
when you're not sure if the key already exists in the map. This can be handled
by first checking if the key is present, and then inserting or updating
depending upon the result, but this is both inconvenient for the developer,
and less than optimal, because it requires multiple lookups in the map
that could otherwise be handled in a single call.

## Solution: `getOrInsert`

We propose the addition of a method that will return the value associated
with `key` if it is already present in the `Map` or `WeakMap`, and otherwise insert
the `key` with the provided default value, or the result of calling a provided
callback function, and then return that value.

Earlier versions of this proposal had an `getOrInsert` method that provided two callbacks,
one for `insert` and the other for `update`, however the current champion thinks
that the get / insert if necessary is a sufficiently common usecase that it makes
sense to focus on it, rather than trying to create an API with maximum flexibility.
It also strongly follows precedent from other languages, in particular Python.

## Examples & Proposed API

### Handling default values

Using `getOrInsert` simplifies handling default values because it will not overwrite
an existing value.

```js
// Currently
let prefs = getUserPrefsMap();
if (!prefs.has("useDarkmode")) {
  prefs.set("useDarkmode", true); // default to true
}

// Using getOrInsert
let prefs = getUserPrefsMap();
prefs.getOrInsert("useDarkmode", true); // default to true
```

By using `getOrInsert`, default values can be applied at different times, with the
assurance that later defaults will not overwrite an existing value. For example,
in a situation where there are user preferences, operating system preferences,
and application defaults, we can use getOrInsert to apply the user preferences,
and then the operating system preferences, and then the application defaults,
without worrying about overwriting the user's preferences.

### Grouping data incrementally

A typical usecase is grouping data based upon key as new values become available.
This is simplified by being able to specify a default value rather than having to
check for whether the key is already present in the `Map` before trying to update.

```js
// Currently
let grouped = new Map();
for (let [key, ...values] of data) {
  if (grouped.has(key)) {
    grouped.get(key).push(...values);
  } else {
    grouped.set(key, values);
  }
}

// Using getOrInsert
let grouped = new Map();
for (let [key, ...values] of data) {
  grouped.getOrInsert(key, []).push(...values);
}
```

It's true that a common usecase for this pattern is already covered by
`Map.groupBy`. However, that method requires that all data be available
prior to building the groups; using `getOrInsert` would allow the Map to be
built and used incrementally. It also provides flexibility to work with
data other than objects, such as the array example above.

### Maintaining a counter

Another common use case is maintaining a counter associated with a
particular key. Using `getOrInsert` makes this more concise, and is the
kind of access and then mutate pattern that is easily optimizable
by engines.

```js
// Currently
let counts = new Map();
if (counts.has(key)) {
  counts.set(key, counts.get(key) + 1);
} else {
  counts.set(key, 1);
}

// Using getOrInsert
let counts = new Map();
counts.set(key, counts.getOrInsert(key, 0) + 1);
```

### Computing a default value

For some usecases, determining the default value is potentially a costly operation that
would be best avoided if it will not be used. In this case, we can use `getOrInsertComputed`.

```js
// Using getOrInsertComputed
let grouped = new Map();
for (let [key, ...values] of data) {
  grouped.getOrInsertComputed(key, () => []).push(...values);
}
```

## Implementations in other languages

Similar functionality exists in other languages.

**Java**

* [`computeIfPresent`](https://docs.oracle.com/javase/9/docs/api/java/util/Map.html#computeIfPresent-K-java.util.function.BiFunction-) remaps existing entry
* [`computeIfAbsent`](https://docs.oracle.com/javase/9/docs/api/java/util/Map.html#computeIfAbsent-K-java.util.function.Function-) insert if empty. computes
the insertion value with a mapping function

**Scala**

* [`getOrElseUpdate`](https://scala-lang.org/api/3.7.3/scala/collection/mutable/MapOps.html#getOrElseUpdate-fffff230) inserts if missing.

**C++**

* [`emplace`](https://en.cppreference.com/w/cpp/container/map/emplace) inserts if missing
* [`map[] assignment opts`](https://en.cppreference.com/w/cpp/container/map/operator_at) inserts if missing
at `key` but also returns a value if it exists at `key`
* [`insert_or_assign`](https://en.cppreference.com/w/cpp/container/map/insert_or_assign) inserts if missing. updates existing value by replacing with a
specific new one, not by applying a function to the existing value

**C#**

* [`GetOrAdd`](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2.getoradd?view=net-9.0) Adds a key/value pair if the key does not already exist and returns the new value, or the existing value if the key already exists.

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

## Specification

* [Ecmarkup source](https://github.com/tc39/proposal-upsert/blob/master/spec.emu)
* [HTML version](https://tc39.es/proposal-upsert/)

## Polyfill

The proposal is trivially polyfillable:

```js
Map.prototype.getOrInsert = function (key, defaultValue) {
  if (!this.has(key)) {
    this.set(key, defaultValue);
  }
  return this.get(key);
};

Map.prototype.getOrInsertComputed = function (key, callbackFunction) {
  if (!this.has(key)) {
    this.set(key, callbackFunction(key));
  }
  return this.get(key);
};
```
