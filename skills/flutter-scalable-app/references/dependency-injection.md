# Dependency Injection — get_it 8.x + injectable 2.x

## Setup

```yaml
dependencies:
  get_it: ^8.0.3
  injectable: ^2.5.0
dev_dependencies:
  injectable_generator: ^2.5.0
  build_runner: ^2.4.14
```

---

## Core Annotations Cheatsheet

| Annotation | Lifetime | Use When |
|---|---|---|
| `@injectable` | Transient (new instance per request) | BLoC, UseCases |
| `@lazySingleton` | Singleton (created on first access) | Repositories, Services, Router |
| `@singleton` | Singleton (created at app start) | Critical services initialized eagerly |
| `@factoryMethod` | On a static factory | Complex construction logic |
| `@module` | On an abstract class | 3rd-party instances (Firebase, Dio) |
| `@Named('name')` | Any | Multiple registrations of same type |
| `@preResolve` | Async | Futures that must complete before app runs |

---

## injection.dart

```dart
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';
import 'injection.config.dart';

final getIt = GetIt.instance;

@InjectableInit(
  initializerName: 'init',
  preferRelativeImports: true,
  asExtension: true,
)
void configureDependencies() => getIt.init();
```

---

## Firebase Module — 3rd-party Instances

```dart
// core/di/firebase_module.dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:firebase_storage/firebase_storage.dart';
import 'package:cloud_functions/cloud_functions.dart';
import 'package:injectable/injectable.dart';

@module
abstract class FirebaseModule {
  @lazySingleton
  FirebaseFirestore get firestore => FirebaseFirestore.instance;

  @lazySingleton
  FirebaseAuth get auth => FirebaseAuth.instance;

  @lazySingleton
  FirebaseStorage get storage => FirebaseStorage.instance;

  @lazySingleton
  FirebaseFunctions get functions => FirebaseFunctions.instance;
}
```

---

## Network Module — Dio

```dart
// core/di/network_module.dart
@module
abstract class NetworkModule {
  @lazySingleton
  Dio get dio {
    final dio = Dio(
      BaseOptions(
        baseUrl: Env.apiBaseUrl,
        connectTimeout: const Duration(seconds: 15),
        receiveTimeout: const Duration(seconds: 15),
        headers: {'Content-Type': 'application/json'},
      ),
    );
    dio.interceptors.addAll([
      LogInterceptor(
        requestBody: true,
        responseBody: true,
        logPrint: (o) => debugPrint(o.toString()),
      ),
      // Add AuthInterceptor for Bearer token injection
    ]);
    return dio;
  }
}
```

---

## Repository Registration

Always register implementation with its abstract interface:

```dart
// features/product/data/repositories/product_repository_impl.dart
@LazySingleton(as: ProductRepository)  // binds impl to interface
class ProductRepositoryImpl implements ProductRepository {
  final ProductRemoteDataSource _remote;
  final ProductLocalDataSource? _local;

  ProductRepositoryImpl(this._remote, {this._local});

  @override
  Future<Either<Failure, List<ProductEntity>>> getProducts() async {
    try {
      final models = await _remote.fetchProducts();
      return Right(models.map((m) => m.toEntity()).toList());
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    } on NetworkException {
      return Left(const NetworkFailure());
    }
  }
}
```

---

## UseCase Registration

```dart
// features/product/domain/usecases/get_products_usecase.dart
@injectable
class GetProductsUseCase {
  final ProductRepository _repository;
  GetProductsUseCase(this._repository);

  Future<Either<Failure, List<ProductEntity>>> call(NoParams _) =>
      _repository.getProducts();
}

// Shared params
class NoParams extends Equatable {
  const NoParams();
  @override
  List<Object> get props => [];
}
```

---

## BLoC Registration

```dart
@injectable  // transient — new instance per BlocProvider.create
class ProductBloc extends Bloc<ProductEvent, ProductState> {
  final GetProductsUseCase _getProducts;
  ProductBloc(this._getProducts) : super(const ProductState.initial()) {
    on<ProductStarted>(_onStarted);
  }
  // ...
}

// Usage in page:
BlocProvider(
  create: (_) => getIt<ProductBloc>()..add(const ProductStarted()),
  child: const _ProductView(),
)
```

---

## Async Dependencies with @preResolve

For dependencies that require async initialization before the app runs:

```dart
@module
abstract class SharedPrefsModule {
  @preResolve
  @lazySingleton
  Future<SharedPreferences> get sharedPreferences =>
      SharedPreferences.getInstance();
}

// main.dart — must await configureDependencies when @preResolve is used
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await dotenv.load(fileName: '.env');
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  await configureDependencies();  // ← await because of @preResolve
  runApp(const App());
}
```

---

## Scoped Dependencies (per Feature)

For dependencies that should only live during a specific feature's lifecycle:

```dart
// Register a scope
getIt.pushNewScope(scopeName: 'checkout');
getIt.registerFactory<CheckoutBloc>(() => CheckoutBloc(getIt()));

// Dispose scope when feature is done
context.router.maybePop().then((_) {
  getIt.dropScope('checkout');
});
```

---

## Named Registrations (Multiple of Same Type)

```dart
@module
abstract class ApiModule {
  @Named('primary')
  @lazySingleton
  Dio get primaryDio => Dio(BaseOptions(baseUrl: Env.apiBaseUrl));

  @Named('auth')
  @lazySingleton
  Dio get authDio => Dio(BaseOptions(baseUrl: Env.authUrl));
}

// Inject by name
@injectable
class ProductRemoteDataSource {
  final Dio _dio;
  ProductRemoteDataSource(@Named('primary') this._dio);
}
```

---

## AppRouter Registration

```dart
// core/router/app_router.dart
@lazySingleton                   // ← one instance for the entire app
@AutoRouterConfig(replaceInRouteName: 'Page,Route')
class AppRouter extends RootStackRouter {
  // ...
}

// main.dart
class App extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      routerConfig: getIt<AppRouter>().config(),
    );
  }
}
```

---

## Debugging the DI Container

```dart
// Print all registered instances (debug only)
void debugPrintRegistrations() {
  // get_it 8.x: iterate over all registrations
  if (kDebugMode) {
    print(getIt.toString());
  }
}
```

If `get_it` throws "Object/factory not registered", checklist:
1. Did you run `build_runner`?
2. Is the class annotated with `@injectable`, `@lazySingleton`, or `@singleton`?
3. Did you call `configureDependencies()` in `main.dart` before `runApp`?
4. For `@LazySingleton(as: Interface)`, is the interface imported?