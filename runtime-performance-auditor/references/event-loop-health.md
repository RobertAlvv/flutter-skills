# Event Loop Health Reference

## Dart Event Loop Model

Dart's concurrency model within a single isolate uses a single-threaded event loop with two queues:

```
┌─────────────────────────────────────────────────────┐
│  MICROTASK QUEUE                                    │
│  Runs to completion before the next macrotask       │
│  Sources: Future.microtask(), scheduleMicrotask()   │
│           .then() callbacks, async/await continuations │
├─────────────────────────────────────────────────────┤
│  EVENT QUEUE (Macrotask)                            │
│  Processes one event per loop iteration             │
│  Sources: I/O completions, Timer callbacks,         │
│           Stream events, user input, frame callbacks │
└─────────────────────────────────────────────────────┘
```

Flutter's rendering engine submits frame callbacks into the event queue. If the microtask queue is kept non-empty, or single macrotasks take too long, frame callbacks are delayed, causing dropped frames even when no widget is being rebuilt.

## scheduleMicrotask — Correct vs Incorrect

```dart
// CORRECT — single lightweight deferred task
scheduleMicrotask(() => analytics.flush());

// VIOLATION — recursive scheduleMicrotask stalls the event loop indefinitely
void processNext(List<Item> items, int index) {
  if (index >= items.length) return;
  process(items[index]);
  scheduleMicrotask(() => processNext(items, index + 1)); // loop never yields to macrotask queue
}

// CORRECT — use a macrotask to process iteratively, yielding every iteration
Future<void> processAll(List<Item> items) async {
  for (final item in items) {
    process(item);
    await Future.delayed(Duration.zero); // yields to macrotask queue each iteration
  }
}

// BETTER — offload the whole batch to a background isolate
await Isolate.run(() => items.forEach(process));
```

## Stream.periodic — Frame-Rate Misuse

`Stream.periodic` creates a `Timer`-based stream on the current isolate. Using it to drive frame-rate updates competes directly with Flutter's frame callback:

```dart
// VIOLATION — manual 60fps frame loop on main isolate
Stream.periodic(const Duration(milliseconds: 16), (_) => tick())
    .listen((value) => setState(() => _value = value));

// CORRECT — use Ticker, which piggybacks on Flutter's frame scheduler
class MyWidget extends StatefulWidget { ... }

class _MyWidgetState extends State<MyWidget> with SingleTickerProviderStateMixin {
  late final Ticker _ticker;

  @override
  void initState() {
    super.initState();
    _ticker = createTicker((_) => setState(() => _value = compute()));
    _ticker.start();
  }

  @override
  void dispose() {
    _ticker.dispose();
    super.dispose();
  }
}
```

## Timer.periodic — Severity Guidelines

| Interval | Callback work | Verdict |
|---|---|---|
| ≥ 30s | Any lightweight work | Acceptable |
| 1–10s | Lightweight (counter increment, flag toggle) | Acceptable |
| 1–10s | Database read, network check | MEDIUM — consider longer interval or debounce |
| 100ms–1s | Lightweight | Acceptable |
| 100ms–1s | Non-trivial computation | MEDIUM |
| < 100ms | Any work | HIGH — competes with 60fps frame budget |
| ~16ms | Any work | HIGH — equivalent to a manual 60fps loop on main isolate |

## Future.wait Concurrency Limits

```dart
// Acceptable — small bounded set of parallel requests
await Future.wait([
  repo.getUser(id),
  repo.getOrders(id),
  repo.getPrefs(id),
]);

// MEDIUM risk — 50+ concurrent futures create a scheduling burst
await Future.wait(
  productIds.map(repo.getProduct), // 50 simultaneous network requests
);

// CORRECT for large sets — process in bounded chunks
Future<List<T>> chunkedWait<T>(
  List<Future<T> Function()> tasks, {
  int chunkSize = 10,
}) async {
  final results = <T>[];
  for (final chunk in tasks.slices(chunkSize)) {
    results.addAll(await Future.wait(chunk.map((f) => f())));
  }
  return results;
}
```

## Stream.fromIterable on Large Collections

```dart
// MEDIUM — iterates entire list synchronously in one microtask batch
await for (final item in Stream.fromIterable(largeList)) {
  await process(item);
}

// CORRECT for CPU-heavy per-item processing — offload entire batch
final results = await Isolate.run(() => largeList.map(process).toList());

// CORRECT for I/O per item — chunked parallel, yields to event loop per chunk
for (final chunk in largeList.slices(20)) {
  await Future.wait(chunk.map(processWithIO));
}
```

## Severity Assignment

| Pattern | Severity |
|---|---|
| `Stream.periodic` at frame rate driving UI updates | HIGH |
| `scheduleMicrotask` used recursively as a processing loop | HIGH |
| `Timer.periodic` at < 100ms doing non-trivial work | HIGH |
| `Future.wait` with > 100 simultaneous futures per user action | MEDIUM |
| `Stream.fromIterable(largeList)` with heavy per-item processing in `asyncMap` | MEDIUM |
| `Timer.periodic` at 100ms–1s with database/network callback | MEDIUM |
| Multiple `scheduleMicrotask` calls in a single event handler | LOW |
