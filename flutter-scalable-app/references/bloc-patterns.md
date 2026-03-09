# BLoC / Cubit — State Management Patterns

## BLoC vs Cubit Decision

| Use **Cubit** | Use **BLoC** |
|---|---|
| Simple state, no event branching | Multiple events → complex branching |
| Counters, toggles, form state | External stream subscriptions |
| Stateless business logic | Undo/redo, event history matters |
| Clear linear state transitions | Debounced input (search, typeahead) |

---

## Full BLoC with Freezed — Canonical Pattern

### events

```dart
// features/product/presentation/bloc/product_event.dart
part of 'product_bloc.dart';

@freezed
class ProductEvent with _$ProductEvent {
  const factory ProductEvent.started()                  = ProductStarted;
  const factory ProductEvent.refreshed()                = ProductRefreshed;
  const factory ProductEvent.itemSelected(String id)    = ProductItemSelected;
}
```

### states

```dart
// features/product/presentation/bloc/product_state.dart
part of 'product_bloc.dart';

@freezed
class ProductState with _$ProductState {
  const factory ProductState.initial()                              = _Initial;
  const factory ProductState.loading()                             = _Loading;
  const factory ProductState.loaded(List<ProductEntity> products)  = _Loaded;
  const factory ProductState.error(String message)                 = _Error;
}
```

### bloc

```dart
// features/product/presentation/bloc/product_bloc.dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:injectable/injectable.dart';
import 'package:dartz/dartz.dart';

part 'product_event.dart';
part 'product_state.dart';
part 'product_bloc.freezed.dart';

@injectable
class ProductBloc extends Bloc<ProductEvent, ProductState> {
  final GetProductsUseCase _getProducts;

  ProductBloc(this._getProducts) : super(const ProductState.initial()) {
    on<ProductStarted>(_onStarted);
    on<ProductRefreshed>(_onRefreshed);
  }

  Future<void> _onStarted(
    ProductStarted event,
    Emitter<ProductState> emit,
  ) async {
    emit(const ProductState.loading());
    await _fetchProducts(emit);
  }

  Future<void> _onRefreshed(
    ProductRefreshed event,
    Emitter<ProductState> emit,
  ) async {
    // Keep current data visible while refreshing
    await _fetchProducts(emit);
  }

  Future<void> _fetchProducts(Emitter<ProductState> emit) async {
    final result = await _getProducts(NoParams());
    result.fold(
      (failure) => emit(ProductState.error(failure.message)),
      (products) => emit(ProductState.loaded(products)),
    );
  }
}
```

---

## Cubit — Simpler State Pattern

```dart
@injectable
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);

  void increment() => emit(state + 1);
  void decrement() => emit(state > 0 ? state - 1 : 0);
  void reset() => emit(0);
}
```

---

## Page + BlocProvider + View — Split Pattern

Always split the Page (BLoC provider + DI) from the View (UI):

```dart
// features/product/presentation/screen/product_screen.dart
@RoutePage()
class ProductScreen extends StatelessWidget {
  const ProductPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => getIt<ProductBloc>()..add(const ProductStarted()),
      child: const ProductView(),
    );
  }
}

// Private View — no DI knowledge, only BLoC access via context
class ProductView extends StatelessWidget {
  const ProductView();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Products')),
      body: BlocBuilder<ProductBloc, ProductState>(
        builder: (context, state) => state.when(
          initial: () => const SizedBox.shrink(),
          loading: () => const Center(child: CircularProgressIndicator()),
          loaded:  (products) => _ProductList(products: products),
          error:   (msg) => _ErrorView(message: msg),
        ),
      ),
    );
  }
}
```

---

## BlocListener — Side Effects Only

Use `BlocListener` for navigation, dialogs, snackbars — never for building UI:

```dart
BlocListener<AuthBloc, AuthState>(
  listenWhen: (previous, current) =>
      current is _Unauthenticated && previous is! _Unauthenticated,
  listener: (context, state) {
    state.whenOrNull(
      unauthenticated: () =>
          context.router.replaceAll([const LoginRoute()]),
      error: (message) =>
          ScaffoldMessenger.of(context).showSnackBar(
            SnackBar(content: Text(message)),
          ),
    );
  },
  child: const _HomeView(),
)
```

---

## BlocConsumer — Build + Listen Together

```dart
BlocConsumer<FormBloc, FormState>(
  listenWhen: (prev, curr) => curr.isSubmitSuccess,
  listener:  (ctx, state)  => ctx.router.replace(const SuccessRoute()),
  buildWhen: (prev, curr)  => prev.isLoading != curr.isLoading,
  builder:   (ctx, state)  => _FormBody(isLoading: state.isLoading),
)
```

---

## BlocSelector — Surgical Rebuilds

Prefer `BlocSelector` when only a sub-field of state matters:

```dart
// ❌ Rebuilds on every ProductState change
BlocBuilder<ProductBloc, ProductState>(
  builder: (context, state) => Badge(count: state.cartCount),
)

// ✅ Rebuilds ONLY when cartCount changes
BlocSelector<ProductBloc, ProductState, int>(
  selector: (state) => state.maybeWhen(
    loaded: (products) => products.where((p) => p.inCart).length,
    orElse: () => 0,
  ),
  builder: (context, count) => Badge(count: count),
)
```

---

## MultiBlocProvider — Feature Root

```dart
@RoutePage()
class CheckoutPage extends StatelessWidget {
  const CheckoutPage({super.key});

  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        BlocProvider(
          create: (_) => getIt<CartBloc>()..add(const CartLoaded()),
        ),
        BlocProvider(create: (_) => getIt<PaymentBloc>()),
        BlocProvider(
          create: (_) => getIt<AddressBloc>()..add(const AddressLoaded()),
        ),
      ],
      child: const _CheckoutView(),
    );
  }
}
```

---

## Debounce — Search / Typeahead

```dart
import 'package:rxdart/rxdart.dart';

EventTransformer<T> debounceSequential<T>(Duration duration) =>
    (events, mapper) =>
        events.debounceTime(duration).switchMap(mapper);

// In BLoC constructor:
on<SearchQueryChanged>(
  _onQueryChanged,
  transformer: debounceSequential(const Duration(milliseconds: 400)),
);

Future<void> _onQueryChanged(
  SearchQueryChanged event,
  Emitter<SearchState> emit,
) async {
  if (event.query.trim().isEmpty) {
    emit(const SearchState.initial());
    return;
  }
  emit(const SearchState.loading());
  final result = await _search(SearchParams(query: event.query));
  result.fold(
    (f) => emit(SearchState.error(f.message)),
    (r) => emit(SearchState.loaded(r)),
  );
}
```

---

## Stream Subscriptions — Firestore Real-Time

```dart
@injectable
class NotificationsBloc
    extends Bloc<NotificationsEvent, NotificationsState> {
  final WatchNotificationsUseCase _watch;
  StreamSubscription<Either<Failure, List<NotificationEntity>>>? _sub;

  NotificationsBloc(this._watch)
      : super(const NotificationsState.initial()) {
    on<NotificationsWatchStarted>(_onStarted);
    on<NotificationsReceived>(_onReceived);
    on<NotificationsWatchStopped>(_onStopped);
  }

  Future<void> _onStarted(
    NotificationsWatchStarted _,
    Emitter<NotificationsState> emit,
  ) async {
    emit(const NotificationsState.loading());
    _sub = _watch(NoParams()).listen(
      (result) => add(NotificationsReceived(result)),
    );
  }

  void _onReceived(
    NotificationsReceived event,
    Emitter<NotificationsState> emit,
  ) {
    event.result.fold(
      (f) => emit(NotificationsState.error(f.message)),
      (n) => emit(NotificationsState.loaded(n)),
    );
  }

  void _onStopped(
    NotificationsWatchStopped _,
    Emitter<NotificationsState> emit,
  ) {
    _sub?.cancel();
    _sub = null;
  }

  @override
  Future<void> close() {
    _sub?.cancel();
    return super.close();
  }
}
```

---

## Optimistic Updates

```dart
on<ProductLiked>((event, emit) async {
  // Snapshot current state for rollback
  final snapshot = state;

  // 1. Update UI immediately (optimistic)
  if (state is _Loaded) {
    final loaded = state as _Loaded;
    emit(ProductState.loaded(
      loaded.products.map((p) =>
        p.id == event.productId
          ? p.copyWith(isLiked: !p.isLiked)
          : p,
      ).toList(),
    ));
  }

  // 2. Persist to backend
  final result = await _toggleLike(ToggleLikeParams(id: event.productId));

  // 3. Rollback on failure
  result.fold(
    (failure) => emit(snapshot),
    (_) => null, // optimistic state was correct
  );
});
```

---

## Testing BLoC with bloc_test

```dart
// test/features/product/bloc/product_bloc_test.dart
void main() {
  late ProductBloc bloc;
  late MockGetProductsUseCase mockGetProducts;

  setUp(() {
    mockGetProducts = MockGetProductsUseCase();
    bloc = ProductBloc(mockGetProducts);
  });

  tearDown(() => bloc.close());

  group('ProductStarted', () {
    blocTest<ProductBloc, ProductState>(
      'emits [loading, loaded] on success',
      build: () {
        when(() => mockGetProducts(any()))
            .thenAnswer((_) async => Right(tProducts));
        return bloc;
      },
      act: (b) => b.add(const ProductStarted()),
      expect: () => [
        const ProductState.loading(),
        ProductState.loaded(tProducts),
      ],
      verify: (_) => verify(() => mockGetProducts(NoParams())).called(1),
    );

    blocTest<ProductBloc, ProductState>(
      'emits [loading, error] on failure',
      build: () {
        when(() => mockGetProducts(any()))
            .thenAnswer((_) async => Left(ServerFailure()));
        return bloc;
      },
      act: (b) => b.add(const ProductStarted()),
      expect: () => [
        const ProductState.loading(),
        isA<_Error>(),
      ],
    );
  });
}
```