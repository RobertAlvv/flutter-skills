# Async Efficiency Reference

## The async/await Misconception

`async`/`await` in Dart does NOT move work to a background thread or isolate. It only allows the current isolate to yield control during I/O waiting (network, file, database). CPU work still blocks the main isolate.

```
MAIN ISOLATE TIMELINE

await http.get(...)  → yields, main isolate free ✅
heavy parse result   → runs on main isolate, blocks ❌
await dao.insert(…)  → yields, main isolate free ✅
```

## Sequential vs Parallel Awaits

### Decision Table

| Scenario | Recommended pattern |
|---|---|
| Operations depend on each other's results | Sequential `await` — no alternative |
| Operations are independent | `Future.wait([...])` |
| Need results as they complete (streaming) | `Future.wait` still; or `Stream.fromFutures` |
| One failure should cancel all | `Future.wait` with `eagerError: true` |

### Sequential (correct when dependent)

```dart
final user = await userRepo.getUser(id);
final orders = await orderRepo.getOrdersForUser(user.accountId); // depends on user
```

### Parallel (correct when independent)

```dart
final [user, orders, prefs] = await Future.wait([
  userRepo.getUser(id),
  orderRepo.getOrders(id),
  prefsRepo.getPrefs(id),
]);
```

## await in Loops — N+1 Async Pattern

```dart
// VIOLATION — N sequential round trips
for (final id in productIds) {
  final product = await productRepo.getById(id); // 1 round trip per item
  products.add(product);
}

// CORRECT — 1 batch request or N parallel requests
// Option A: batch API call
final products = await productRepo.getByIds(productIds);

// Option B: parallel (when no batch API exists, and count < ~50)
final products = await Future.wait(
  productIds.map((id) => productRepo.getById(id)),
);

// Option C: chunked parallel (when count may be large)
const chunkSize = 10;
final chunks = productIds.slices(chunkSize); // package:collection
final products = <Product>[];
for (final chunk in chunks) {
  products.addAll(await Future.wait(chunk.map(productRepo.getById)));
}
```

## Future.delayed(Duration.zero) — Common Misuse

`Future.delayed(Duration.zero, fn)` schedules `fn` as a macrotask on the *same* event loop. It does not create a background isolate.

```dart
// VIOLATION — misunderstood as CPU offloading
Future.delayed(Duration.zero, () {
  final sorted = largeList..sort(); // still main isolate
  emit(Loaded(sorted));
});

// WHAT IT ACTUALLY DOES
// Defers execution until the next event loop iteration.
// Useful for: deferring a setState() call until after build().
// NOT useful for: CPU offloading.

// CORRECT for CPU work
final sorted = await Isolate.run(() => [...largeList]..sort());
emit(Loaded(sorted));
```

## Synchronous Work Between Awaits

```dart
// VIOLATION — heavy work between two awaits blocks main isolate
Future<void> processData() async {
  final raw = await api.fetchData();        // yields ✅
  final result = parseComplexData(raw);     // CPU-heavy, main isolate ❌
  await database.save(result);              // yields ✅
}

// CORRECT — offload the CPU step
Future<void> processData() async {
  final raw = await api.fetchData();
  final result = await Isolate.run(() => parseComplexData(raw));
  await database.save(result);
}
```

## Future Deduplication (Cache In-Flight Requests)

When multiple callers invoke the same async method simultaneously, they each trigger a separate I/O operation unless the in-flight `Future` is cached:

```dart
// VIOLATION — 3 simultaneous callers each trigger a separate network request
class UserRepository {
  Future<User> getUser(String id) => api.fetchUser(id);
}

// CORRECT — all concurrent callers share the same in-flight Future
class UserRepository {
  final _inflight = <String, Future<User>>{};

  Future<User> getUser(String id) {
    return _inflight[id] ??= api.fetchUser(id).whenComplete(() {
      _inflight.remove(id);
    });
  }
}
```

## Severity Assignment

| Pattern | Severity |
|---|---|
| Heavy CPU work between awaits (matches Step 1 HIGH patterns) | HIGH |
| Sequential awaits for 3+ independent operations | MEDIUM |
| `await` inside `for` loop over > 10 items | MEDIUM |
| `Future.delayed(Duration.zero)` for CPU offloading | HIGH (misleading, main isolate) |
| Sequential awaits for 2 independent operations | LOW |
| No `Future` deduplication on a frequently-called method | MEDIUM |
