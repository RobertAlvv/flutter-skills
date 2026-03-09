# Memoization and Caching Reference

## Memoization vs Caching

| Concept | Definition | Scope |
|---|---|---|
| **Memoization** | Cache the output of a pure function keyed by its inputs | In-memory, within a session |
| **Caching** | Persist or buffer data to avoid re-fetching from an external source | Can span sessions (disk) or be in-memory |

Both reduce unnecessary computation. The distinction matters for eviction strategy: memoized values are valid as long as their inputs haven't changed; cached values expire after a TTL or explicit invalidation.

## Bloc State — Derived Value Memoization

The most common memoization target in Flutter/BLoC apps is a derived value computed from state:

```dart
// VIOLATION — recomputed on every state emission
class CartBloc extends Bloc<CartEvent, CartState> {
  double _total(List<CartItem> items) =>
      items.fold(0.0, (s, e) => s + e.price * e.quantity);

  void _onItemAdded(ItemAdded event, Emitter<CartState> emit) {
    final updated = [...state.items, event.item];
    emit(CartState(items: updated, total: _total(updated))); // recomputed
  }
}

// CORRECT — store derived value in state; recompute only when items change
@freezed
class CartState with _$CartState {
  const CartState._();
  const factory CartState({
    required List<CartItem> items,
    required double total,
  }) = _CartState;

  factory CartState.fromItems(List<CartItem> items) => CartState(
    items: items,
    total: items.fold(0.0, (s, e) => s + e.price * e.quantity),
  );
}
```

## Repository Future Caching

Cache the resolved `Future` object to deduplicate concurrent calls and avoid repeated fetches within a session:

```dart
class ProductRepository {
  Future<List<Product>>? _cache;
  DateTime? _cachedAt;
  static const _ttl = Duration(minutes: 5);

  Future<List<Product>> getProducts() {
    final isStale = _cachedAt == null ||
        DateTime.now().difference(_cachedAt!) > _ttl;
    if (isStale) {
      _cache = _fetchFromApi().then((products) {
        _cachedAt = DateTime.now();
        return products;
      });
    }
    return _cache!;
  }

  void invalidate() {
    _cache = null;
    _cachedAt = null;
  }

  Future<List<Product>> _fetchFromApi() async {
    final response = await dio.get('/products');
    return await Isolate.run(() => parseProducts(response.data));
  }
}
```

## useMemoized (flutter_hooks)

For expensive derivations in stateless UI that uses `flutter_hooks`:

```dart
// Recomputes only when `items` reference changes
final sortedItems = useMemoized(
  () => [...items]..sort((a, b) => a.name.compareTo(b.name)),
  [items],
);
```

## Keyed In-Memory Cache with LRU Eviction

For caches over unbounded key spaces (e.g., per-user data), add eviction to prevent unbounded memory growth:

```dart
class LruCache<K, V> {
  final int maxSize;
  final _cache = LinkedHashMap<K, V>();

  LruCache(this.maxSize);

  V? get(K key) {
    final value = _cache.remove(key);
    if (value != null) _cache[key] = value; // move to end (most recent)
    return value;
  }

  void put(K key, V value) {
    _cache.remove(key);
    if (_cache.length >= maxSize) _cache.remove(_cache.keys.first);
    _cache[key] = value;
  }
}
```

## HTTP Response Caching

Use `dio_cache_interceptor` for endpoint-level HTTP caching:

```dart
final cacheStore = MemCacheStore(); // or HiveCacheStore for persistence

final dio = Dio()
  ..interceptors.add(DioCacheInterceptor(
    options: CacheOptions(
      store: cacheStore,
      policy: CachePolicy.refreshForceCache,
      maxStale: const Duration(minutes: 10),
    ),
  ));
```

Cache-appropriate endpoints:
- Reference data (product catalog, categories, configuration) — TTL 5–60 minutes
- User profile — TTL 1–5 minutes or invalidate on mutation
- Real-time data (prices, inventory, messages) — do not cache or TTL ≤ 30s

## Audit Severity Table

| Pattern | Severity |
|---|---|
| Derived state recomputed on every Bloc event with no state caching | MEDIUM |
| `groupBy` / `sortedBy` on unmodified data on every navigation | MEDIUM |
| Repository method with no cache making the same API call 10+ times per session | MEDIUM |
| No caching on reference data endpoints (catalog, config) in a high-traffic app | MEDIUM |
| Concurrent callers of same repository method each trigger separate network request | MEDIUM |
| `useMemoized` absent in hooks widget computing O(n log n) derivation per rebuild | LOW |
| In-memory cache with no eviction strategy over unbounded key space | MEDIUM (memory risk) |
