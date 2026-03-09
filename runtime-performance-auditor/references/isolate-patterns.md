# Isolate Patterns Reference

## Isolate API Decision Table

| API | When to use | When NOT to use |
|---|---|---|
| `Isolate.run(fn)` | Single one-shot computation; Dart 2.19+ / Flutter 3.7+. Auto-manages lifecycle. | Long-lived workers; bi-directional streaming communication |
| `compute(fn, message)` | Legacy compatibility with Flutter < 3.7; top-level or static function only | New code on Flutter 3.7+ — prefer `Isolate.run()`. Message must be serializable. |
| `Isolate.spawn(entry, message)` | Long-lived background worker needing `SendPort`/`ReceivePort` bi-directional channel | One-shot tasks — `Isolate.run()` is simpler and auto-disposes |

## Serialization Constraints

Objects passed to `compute()` or `Isolate.run()` as the argument must be serializable across the isolate boundary. Dart uses copying semantics — the object is serialized into a message and reconstructed in the spawned isolate.

**Cannot be passed:**
- `BuildContext`
- `Stream` or `StreamController`
- Closures that capture mutable state from the parent isolate
- `ChangeNotifier` or any object with `Listenable` dependency
- `File` handles or open sockets

**Can be passed:**
- Primitive types: `int`, `double`, `bool`, `String`
- `List`, `Map`, `Set` of serializable types
- `Uint8List`, `ByteData` — preferred for large binary data (zero-copy in some cases)
- Plain Dart data classes (no Flutter dependencies)

## Correct Isolate.run() Usage

```dart
// One-shot JSON decoding
final List<Product> products = await Isolate.run(() {
  final decoded = jsonDecode(rawJson) as List;
  return decoded.map((e) => Product.fromJson(e as Map<String, dynamic>)).toList();
});
```

```dart
// One-shot sort of large list
final sorted = await Isolate.run(() {
  final copy = List<Item>.from(unsortedItems);
  copy.sort((a, b) => a.price.compareTo(b.price));
  return copy;
});
```

## Correct Isolate.spawn() Worker Pattern

For long-lived background tasks (offline sync, search indexing):

```dart
class BackgroundWorker {
  late final Isolate _isolate;
  late final SendPort _sendPort;
  final _receivePort = ReceivePort();

  Future<void> start() async {
    _isolate = await Isolate.spawn(_workerEntry, _receivePort.sendPort);
    _sendPort = await _receivePort.first as SendPort;
  }

  Future<T> process<T>(dynamic message) {
    final responsePort = ReceivePort();
    _sendPort.send([responsePort.sendPort, message]);
    return responsePort.first as Future<T>;
  }

  void dispose() {
    _isolate.kill();
    _receivePort.close();
  }

  static void _workerEntry(SendPort callerSendPort) {
    final workerReceivePort = ReceivePort();
    callerSendPort.send(workerReceivePort.sendPort);

    workerReceivePort.listen((message) {
      final [SendPort replyPort, dynamic data] = message as List;
      replyPort.send(expensiveProcess(data));
    });
  }
}
```

## Stream.map Is NOT Off-Isolate

This is a critical misunderstanding:

```dart
// VIOLATION — Stream.map runs on the main isolate
final results = await Stream.fromIterable(rawItems)
    .map((item) => heavyParse(item))  // executes on main isolate
    .toList();

// CORRECT — move the whole processing to a background isolate
final results = await Isolate.run(() {
  return rawItems.map(heavyParse).toList();
});
```

`asyncMap` does NOT offload to a background isolate either:

```dart
// VIOLATION — asyncMap only means the transform is async, not off-isolate
final results = await stream.asyncMap((item) async {
  return await heavyAsyncParse(item); // still runs on main isolate
}).toList();
```

## Flutter Version Compatibility

| Flutter version | Recommended API |
|---|---|
| < 3.7 | `compute(fn, message)` — requires top-level/static function |
| ≥ 3.7 | `Isolate.run(fn)` — accepts any closure, auto-disposes isolate |
| All versions | `Isolate.spawn()` for long-lived workers |

## Threshold Guidelines for Offloading

| Operation | Typical duration on mid-range device | Recommended action |
|---|---|---|
| `jsonDecode` < 10 KB | < 0.5 ms | Main isolate acceptable |
| `jsonDecode` 50–200 KB | 5–20 ms | Offload to `Isolate.run()` |
| `jsonDecode` > 200 KB | 20–100 ms | Always offload |
| `.sort()` < 1 000 items | < 1 ms | Main isolate acceptable |
| `.sort()` 10 000 items | 5–15 ms | Offload |
| `bcrypt` (cost 10) | 100–300 ms | Always offload |
| `sha256` single payload | < 1 ms | Main isolate acceptable |
| Image decode 4 MP, no resize | 80–150 ms | Always offload + set `targetWidth`/`targetHeight` |
