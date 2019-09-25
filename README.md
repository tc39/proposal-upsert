# `Map.prototype.upsert`

ECMAScript proposal and reference implementation for `Map.prototype.upsert`.

**Author:** Brad Farias (GoDaddy)

**Champion:** Erica Pramer (GoDaddy)

**Stage:** 1

## Motivation

Adding and updating values of a Map are tasks that developers often perform 
in conjuction. There are currently no `Map` prototype methods for either of those two
things, let alone a method that does both. The workarounds involve multiple 
lookups and developer inconvenience.

## Solution: `upsert`

We propose the addition of a method that will add a value to a map if the map does not already have something at `key`, and will also update an existing 
value at `key`. 
It’s worthwhile having this API for the average case to cut down on lookups.
It is also worthwhile for developer convenience.

## Examples & Proposed API

The following examples would all be optimized and made simpler by `upsert`.
The proposed API allows a developer to do one lookup and update in place
```js
upsert(key, old => updated, () => insertionValue)
```

### Normalization of values during insertion
Currently you would need to do 3 lookups.
```js
if (!map.has(key)) {
  map.set(key, value);
}
map.get(key).doThing();
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

### Just insert if missing
You might omit an update if you're handling data that doesn't change, but
can still be appended.
```js
// two lookups
if (!map1.has(key)) {
  map1.set(key, value);
}
```

### Just update if present
You might want to omit an insert if you want to perform a function on
all existing values in a Map (ex. normalization).

```js
// two lookups
old = map.get(key);
updated = old.doThing();
map.set(key, updated);
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


## FAQ
- Is the goal to simplify the API or to optimize it?
  - both
- Why not use existing jsengines that optimize by coalescing the lookup and mutation? 
  - this does not cover all patterns
- Why use functions instead of values for the parameters?
  - you may want to apply a normalization function when inserting
  - when updating, we will be able to perform a function on the existing value
  instead of just replacing the value
- Why are we calling this upsert?
  - it is a combination of `update` & `insert`
  - `updateOrInsert` seems too wordy

## Specification

* [Ecmarkup source](https://github.com/thumbsupep/proposal-upsert/blob/master/spec.emu)
* [HTML version](https://thumbsupep.github.io/proposal-upsert/)
