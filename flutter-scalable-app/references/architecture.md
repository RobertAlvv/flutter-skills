# Architecture — Clean Architecture Flutter

## Los tres principios que guían esta arquitectura

1. **Las dependencias solo apuntan hacia adentro** — `presentation` depende de `domain`, `data` depende de `domain`, pero `domain` no depende de nadie.
2. **`domain` es puro Dart** — sin Flutter, sin Firebase, sin Dio. Solo lógica de negocio.
3. **`core` es infraestructura compartida** — lo que no pertenece a ningún feature pero todos necesitan.

---

## Estructura Completa del Proyecto

```
lib/
│
├── app/                                    # Punto de entrada y configuración global
│   ├── app.dart                            # Widget raíz (MaterialApp.router)
│   ├── app_module.dart                     # @module para dependencias app-level
│   └── flavor/
│       ├── app_flavor.dart                 # Enum: dev | staging | prod
│       └── flavor_config.dart              # Configuración por flavor
│
├── core/                                   # Infraestructura transversal (sin lógica de negocio)
│   ├── di/
│   │   ├── injection.dart                  # GetIt + @InjectableInit
│   │   └── injection.config.dart           # ← generado por injectable
│   ├── env/
│   │   └── env.dart                        # Env.apiBaseUrl, Env.firebaseKey
│   ├── error/
│   │   ├── failures.dart                   # Jerarquía de Failure (Equatable)
│   │   └── exceptions.dart                 # Excepciones crudas pre-mapeo
│   ├── network/
│   │   ├── dio_client.dart
│   │   └── interceptors/
│   │       ├── auth_interceptor.dart
│   │       └── logging_interceptor.dart
│   ├── router/
│   │   ├── app_router.dart                 # @AutoRouterConfig
│   │   ├── app_router.gr.dart              # ← generado por auto_route
│   │   └── guards/
│   │       └── auth_guard.dart
│   ├── theme/
│   │   ├── app_theme.dart
│   │   ├── app_colors.dart
│   │   └── app_typography.dart
│   ├── extensions/
│   │   ├── context_extensions.dart
│   │   └── string_extensions.dart
│   └── utils/
│       ├── no_params.dart
│       └── validators.dart
│
├── features/
│   └── [feature_name]/
│       │
│       ├── data/                           # ← Depende de: domain
│       │   ├── datasource/
│       │   │   └── [feature]_datasource.dart     # Acceso a Firebase / REST
│       │   ├── repository_impl/
│       │   │   └── [feature]_repository_impl.dart # Implementa el contrato de domain
│       │   └── model/
│       │       └── [feature]_model.dart          # Extiende la entity + toJson/fromJson
│       │
│       ├── domain/                         # ← Sin dependencias externas — Dart puro
│       │   ├── entity/
│       │   │   └── [feature]_entity.dart         # Clase base Equatable
│       │   ├── repository/
│       │   │   └── [feature]_repository.dart     # abstract interface
│       │   └── use_case/
│       │       ├── get_[feature]_use_case.dart
│       │       ├── create_[feature]_use_case.dart
│       │       └── watch_[feature]_use_case.dart # Para streams en tiempo real
│       │
│       └── presentation/                   # ← Depende de: domain
│           ├── screen/
│           │   └── [feature]_screen.dart         # @RoutePage() — orquesta BLoC + View
│           ├── bloc/
│           │   ├── [feature]_bloc.dart            # @injectable
│           │   ├── [feature]_event.dart           # @freezed
│           │   └── [feature]_state.dart           # @freezed
│           ├── cubit/                             # (si se requiere estado simple)
│           │   ├── [feature]_cubit.dart           # @injectable
│           │   └── [feature]_cubit_state.dart     # @freezed
│           ├── widgets/
│           │   └── [widget_name].dart             # Widgets reutilizables del feature
│           ├── view/                              # (si se requiere separar la UI)
│           │   └── [feature]_view.dart            # Widget privado de solo UI
│           └── utils/                             # (si se requiere)
│               └── [feature]_formatter.dart       # Helpers de presentación
│
└── main.dart
```

---

## Regla de Dependencias

```
┌─────────────────────────────────────────────────┐
│                  PRESENTATION                    │
│   screen · bloc · cubit · widgets · view · utils │
│               depende de ↓                      │
├─────────────────────────────────────────────────┤
│                    DOMAIN                        │
│      entity · repository · use_case             │
│     Dart puro — cero dependencias externas      │
│               ↑ implementado por                │
├─────────────────────────────────────────────────┤
│                     DATA                         │
│    datasource · repository_impl · model         │
└─────────────────────────────────────────────────┘
                       │
        disponible para todos los layers
┌─────────────────────────────────────────────────┐
│                    CORE + APP                    │
│   DI · Router · Theme · Network · Env · Error   │
└─────────────────────────────────────────────────┘
```

| Capa | Puede importar | No puede importar |
|---|---|---|
| `presentation` | `domain`, `core` | `data` directamente |
| `domain` | `core/error`, `core/utils` | `presentation`, `data`, Flutter SDK, Firebase |
| `data` | `domain`, `core` | `presentation` |
| `core` | paquetes externos | ningún `feature` |

---

## app/ — Punto de Entrada

### main.dart
```dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'app/app.dart';
import 'core/di/injection.dart';
import 'firebase_options.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await dotenv.load(fileName: '.env');
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  await configureDependencies();
  runApp(const App());
}
```

### app/app.dart
```dart
import 'package:flutter/material.dart';
import '../core/di/injection.dart';
import '../core/router/app_router.dart';
import '../core/theme/app_theme.dart';

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      title: 'My App',
      debugShowCheckedModeBanner: false,
      theme: AppTheme.light,
      darkTheme: AppTheme.dark,
      themeMode: ThemeMode.system,
      routerConfig: getIt<AppRouter>().config(),
    );
  }
}
```

### app/app_module.dart
```dart
import 'package:injectable/injectable.dart';
import '../core/router/app_router.dart';

@module
abstract class AppModule {
  @lazySingleton
  AppRouter get router => AppRouter();
}
```

---

## domain/ — Capa Pura

### entity/
La entidad es la clase base del dominio. Dart puro con Equatable. Sin imports de Firebase, Dio, ni Flutter.

```dart
// features/product/domain/entity/product_entity.dart
import 'package:equatable/equatable.dart';

class ProductEntity extends Equatable {
  final String id;
  final String name;
  final double price;
  final String categoryId;
  final String imageUrl;
  final DateTime createdAt;
  final bool isAvailable;

  const ProductEntity({
    required this.id,
    required this.name,
    required this.price,
    required this.categoryId,
    required this.imageUrl,
    required this.createdAt,
    this.isAvailable = true,
  });

  @override
  List<Object?> get props =>
      [id, name, price, categoryId, imageUrl, createdAt, isAvailable];
}
```

### repository/
Define el contrato. Solo interfaces — sin implementación, sin Firebase.

```dart
// features/product/domain/repository/product_repository.dart
import 'package:dartz/dartz.dart';
import '../../../../core/error/failures.dart';
import '../entity/product_entity.dart';

abstract interface class ProductRepository {
  Future<Either<Failure, List<ProductEntity>>> getProducts();
  Future<Either<Failure, ProductEntity>>       getProductById(String id);
  Future<Either<Failure, Unit>>                createProduct(ProductEntity product);
  Future<Either<Failure, Unit>>                deleteProduct(String id);
  Stream<Either<Failure, List<ProductEntity>>> watchProducts();
}
```

### use_case/
Un archivo por caso de uso. Una sola responsabilidad. Sin conocimiento de Flutter ni Firebase.

```dart
// features/product/domain/use_case/get_products_use_case.dart
import 'package:dartz/dartz.dart';
import 'package:injectable/injectable.dart';
import '../../../../core/error/failures.dart';
import '../../../../core/utils/no_params.dart';
import '../entity/product_entity.dart';
import '../repository/product_repository.dart';

@injectable
class GetProductsUseCase {
  final ProductRepository _repository;
  GetProductsUseCase(this._repository);

  Future<Either<Failure, List<ProductEntity>>> call(NoParams _) =>
      _repository.getProducts();
}
```

---

## data/ — Implementación de Contratos

### model/
El model **extiende la entity** y añade los métodos `toJson` y `fromJson`. No es una clase paralela — es la misma entity con capacidad de serialización.

```dart
// features/product/data/model/product_model.dart
import 'package:json_annotation/json_annotation.dart';
import '../../domain/entity/product_entity.dart';

part 'product_model.g.dart';

@JsonSerializable()
class ProductModel extends ProductEntity {
  const ProductModel({
    required super.id,
    required super.name,
    required super.price,
    @JsonKey(name: 'category_id') required super.categoryId,
    @JsonKey(name: 'image_url')   required super.imageUrl,
    @JsonKey(name: 'created_at')  required super.createdAt,
    super.isAvailable = true,
  });

  /// Construcción desde JSON (Firestore / REST)
  factory ProductModel.fromJson(Map<String, dynamic> json) =>
      _$ProductModelFromJson(json);

  /// Serialización a JSON para persistir o enviar
  Map<String, dynamic> toJson() => _$ProductModelToJson(this);

  /// Conversión desde una Entity de domain (para persistir)
  factory ProductModel.fromEntity(ProductEntity entity) => ProductModel(
    id:          entity.id,
    name:        entity.name,
    price:       entity.price,
    categoryId:  entity.categoryId,
    imageUrl:    entity.imageUrl,
    createdAt:   entity.createdAt,
    isAvailable: entity.isAvailable,
  );
}
```

> **Por qué `extends` y no `@freezed`:**
> El model hereda directamente de la entity para que `data` no introduzca un tipo nuevo al dominio.
> `@freezed` genera una clase sellada que no se puede extender — por eso aquí usamos `@JsonSerializable` + herencia.

### datasource/
Acceso directo a la fuente de datos (Firebase, REST, caché). Solo trabaja con `ProductModel`, nunca con `ProductEntity`.

```dart
// features/product/data/datasource/product_datasource.dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:injectable/injectable.dart';
import '../model/product_model.dart';

abstract interface class ProductDatasource {
  Future<List<ProductModel>> fetchProducts();
  Future<ProductModel>       fetchProductById(String id);
  Future<void>               saveProduct(ProductModel model);
  Future<void>               removeProduct(String id);
  Stream<List<ProductModel>> watchProducts();
}

@LazySingleton(as: ProductDatasource)
class ProductDatasourceImpl implements ProductDatasource {
  final FirebaseFirestore _firestore;
  ProductDatasourceImpl(this._firestore);

  CollectionReference<Map<String, dynamic>> get _col =>
      _firestore.collection('products');

  @override
  Future<List<ProductModel>> fetchProducts() async {
    final snap = await _col
        .orderBy('created_at', descending: true)
        .limit(50)
        .get();
    return snap.docs
        .map((d) => ProductModel.fromJson({...d.data(), 'id': d.id}))
        .toList();
  }

  @override
  Future<ProductModel> fetchProductById(String id) async {
    final doc = await _col.doc(id).get();
    if (!doc.exists) throw Exception('Product $id not found');
    return ProductModel.fromJson({...doc.data()!, 'id': doc.id});
  }

  @override
  Future<void> saveProduct(ProductModel model) =>
      _col.doc(model.id).set(model.toJson());

  @override
  Future<void> removeProduct(String id) => _col.doc(id).delete();

  @override
  Stream<List<ProductModel>> watchProducts() {
    return _col
        .orderBy('created_at', descending: true)
        .snapshots()
        .map((s) => s.docs
            .map((d) => ProductModel.fromJson({...d.data(), 'id': d.id}))
            .toList());
  }
}
```

### repository_impl/
Implementa el contrato de `domain/repository`. Mapea excepciones de infraestructura → `Failure`. Convierte `ProductModel` → `ProductEntity` antes de retornar (o simplemente retorna el model, ya que es una entity).

```dart
// features/product/data/repository_impl/product_repository_impl.dart
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:dartz/dartz.dart';
import 'package:injectable/injectable.dart';

import '../../../../core/error/failures.dart';
import '../../domain/entity/product_entity.dart';
import '../../domain/repository/product_repository.dart';
import '../datasource/product_datasource.dart';
import '../model/product_model.dart';

@LazySingleton(as: ProductRepository)
class ProductRepositoryImpl implements ProductRepository {
  final ProductDatasource _datasource;
  ProductRepositoryImpl(this._datasource);

  @override
  Future<Either<Failure, List<ProductEntity>>> getProducts() async {
    try {
      // ProductModel extiende ProductEntity — no necesita conversión explícita
      final models = await _datasource.fetchProducts();
      return Right(models);
    } on FirebaseException catch (e) {
      return Left(ServerFailure(e.message ?? 'Firestore error'));
    } catch (_) {
      return Left(const ParseFailure());
    }
  }

  @override
  Future<Either<Failure, ProductEntity>> getProductById(String id) async {
    try {
      return Right(await _datasource.fetchProductById(id));
    } on FirebaseException catch (e) {
      return Left(ServerFailure(e.message ?? 'Not found'));
    }
  }

  @override
  Future<Either<Failure, Unit>> createProduct(ProductEntity product) async {
    try {
      await _datasource.saveProduct(ProductModel.fromEntity(product));
      return const Right(unit);
    } on FirebaseException catch (e) {
      return Left(ServerFailure(e.message ?? 'Create failed'));
    }
  }

  @override
  Future<Either<Failure, Unit>> deleteProduct(String id) async {
    try {
      await _datasource.removeProduct(id);
      return const Right(unit);
    } on FirebaseException catch (e) {
      return Left(ServerFailure(e.message ?? 'Delete failed'));
    }
  }

  @override
  Stream<Either<Failure, List<ProductEntity>>> watchProducts() {
    return _datasource.watchProducts().map(
      (models) => Right<Failure, List<ProductEntity>>(models),
    );
  }
}
```

---

## presentation/ — BLoC + Screen + Widgets

### bloc/
Eventos y estados como archivos separados con `@freezed`. El BLoC es `@injectable` (transient — nueva instancia por `BlocProvider`).

```dart
// features/product/presentation/bloc/product_event.dart
part of 'product_bloc.dart';

@freezed
class ProductEvent with _$ProductEvent {
  const factory ProductEvent.started()            = ProductStarted;
  const factory ProductEvent.refreshed()          = ProductRefreshed;
  const factory ProductEvent.deleted(String id)   = ProductDeleted;
}

// features/product/presentation/bloc/product_state.dart
part of 'product_bloc.dart';

@freezed
class ProductState with _$ProductState {
  const factory ProductState.initial()                          = _Initial;
  const factory ProductState.loading()                         = _Loading;
  const factory ProductState.loaded(List<ProductEntity> items) = _Loaded;
  const factory ProductState.error(String message)             = _Error;
}

// features/product/presentation/bloc/product_bloc.dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:injectable/injectable.dart';

import '../../../../core/utils/no_params.dart';
import '../../domain/entity/product_entity.dart';
import '../../domain/use_case/get_products_use_case.dart';
import '../../domain/use_case/delete_product_use_case.dart';

part 'product_event.dart';
part 'product_state.dart';
part 'product_bloc.freezed.dart';

@injectable
class ProductBloc extends Bloc<ProductEvent, ProductState> {
  final GetProductsUseCase    _getProducts;
  final DeleteProductUseCase  _deleteProduct;

  ProductBloc(this._getProducts, this._deleteProduct)
      : super(const ProductState.initial()) {
    on<ProductStarted>(_onStarted);
    on<ProductRefreshed>(_onRefreshed);
    on<ProductDeleted>(_onDeleted);
  }

  Future<void> _onStarted(ProductStarted _, Emitter<ProductState> emit) async {
    emit(const ProductState.loading());
    final result = await _getProducts(const NoParams());
    result.fold(
      (f) => emit(ProductState.error(f.message)),
      (items) => emit(ProductState.loaded(items)),
    );
  }

  Future<void> _onRefreshed(ProductRefreshed _, Emitter<ProductState> emit) =>
      _onStarted(const ProductStarted(), emit);

  Future<void> _onDeleted(ProductDeleted event, Emitter<ProductState> emit) async {
    final result = await _deleteProduct(DeleteProductParams(id: event.id));
    result.fold(
      (f) => emit(ProductState.error(f.message)),
      (_) => add(const ProductRefreshed()),
    );
  }
}
```

### cubit/ (si se requiere)
Usar Cubit cuando la lógica es lineal y no necesita múltiples eventos. Por ejemplo: estado de un formulario, un toggle, un contador de pasos.

```dart
// features/checkout/presentation/cubit/checkout_step_cubit_state.dart
part of 'checkout_step_cubit.dart';

@freezed
class CheckoutStepState with _$CheckoutStepState {
  const factory CheckoutStepState({
    @Default(0) int currentStep,
    @Default(3) int totalSteps,
  }) = _CheckoutStepState;
}

// features/checkout/presentation/cubit/checkout_step_cubit.dart
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import 'package:injectable/injectable.dart';

part 'checkout_step_cubit_state.dart';
part 'checkout_step_cubit.freezed.dart';

@injectable
class CheckoutStepCubit extends Cubit<CheckoutStepState> {
  CheckoutStepCubit() : super(const CheckoutStepState());

  void nextStep() {
    if (state.currentStep < state.totalSteps - 1) {
      emit(state.copyWith(currentStep: state.currentStep + 1));
    }
  }

  void previousStep() {
    if (state.currentStep > 0) {
      emit(state.copyWith(currentStep: state.currentStep - 1));
    }
  }

  void goToStep(int step) => emit(state.copyWith(currentStep: step));
}
```

### screen/
La screen tiene **una sola responsabilidad**: proveer el BLoC/Cubit vía `BlocProvider` y montar la View. No contiene lógica de UI — eso va en `view/` o directamente en los widgets.

```dart
// features/product/presentation/screen/product_screen.dart
import 'package:auto_route/auto_route.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

import '../../../../core/di/injection.dart';
import '../bloc/product_bloc.dart';
import '../view/product_view.dart';    // si view/ existe
// o directamente: import '../widgets/product_list.dart';

@RoutePage()
class ProductScreen extends StatelessWidget {
  const ProductScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => getIt<ProductBloc>()..add(const ProductStarted()),
      child: const ProductView(),
    );
  }
}
```

### view/ (si se requiere)
La view contiene el `Scaffold` y la estructura visual principal de la screen. Se separa de la screen cuando la UI es lo suficientemente compleja para justificarlo.

```dart
// features/product/presentation/view/product_view.dart
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

import '../bloc/product_bloc.dart';
import '../widgets/product_card.dart';
import '../widgets/product_error_view.dart';
import '../widgets/product_shimmer_list.dart';

class ProductView extends StatelessWidget {
  const ProductView({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Productos')),
      body: BlocBuilder<ProductBloc, ProductState>(
        builder: (context, state) => state.when(
          initial: () => const SizedBox.shrink(),
          loading: () => const ProductShimmerList(),
          loaded:  (items) => _ProductList(items: items),
          error:   (msg)   => ProductErrorView(
            message: msg,
            onRetry: () => context.read<ProductBloc>().add(const ProductRefreshed()),
          ),
        ),
      ),
    );
  }
}

class _ProductList extends StatelessWidget {
  final List<ProductEntity> items;
  const _ProductList({required this.items});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: items.length,
      itemBuilder: (_, i) => ProductCard(product: items[i]),
    );
  }
}
```

### widgets/
Widgets reutilizables **solo dentro del feature**. Si un widget se comparte entre features, sube a `core/`.

```dart
// features/product/presentation/widgets/product_card.dart
class ProductCard extends StatelessWidget {
  final ProductEntity product;
  const ProductCard({super.key, required this.product});

  @override
  Widget build(BuildContext context) { /* ... */ }
}
```

### utils/ (si se requiere)
Helpers de presentación exclusivos del feature: formateadores, mapeo a strings de UI, cálculos de display.

```dart
// features/product/presentation/utils/product_formatter.dart
class ProductFormatter {
  ProductFormatter._();

  static String formatPrice(double price) =>
      '\$${price.toStringAsFixed(2)}';

  static String formatAvailability(bool isAvailable) =>
      isAvailable ? 'En stock' : 'Agotado';

  static String formatCreatedAt(DateTime date) =>
      '${date.day}/${date.month}/${date.year}';
}
```

---

## Resumen: Qué va en cada carpeta

### data/
| Carpeta | Contenido | Extiende / Implementa |
|---|---|---|
| `datasource/` | Acceso a Firebase / REST. Solo trabaja con `Model`. | — |
| `repository_impl/` | Mapea excepciones → `Failure`. Retorna `Entity`. | `domain/repository/` |
| `model/` | Extiende la entity. Añade `toJson` / `fromJson`. | `domain/entity/` |

### domain/
| Carpeta | Contenido | Regla |
|---|---|---|
| `entity/` | Clase base con Equatable. Dart puro. | Sin imports de Flutter ni libs externas |
| `repository/` | `abstract interface class`. Sin implementación. | Solo tipos del dominio |
| `use_case/` | Un archivo = un caso de uso. Llama al repository. | `@injectable` |

### presentation/
| Carpeta | Obligatorio | Contenido |
|---|---|---|
| `screen/` | ✅ siempre | `@RoutePage()` + `BlocProvider`. Sin lógica UI. |
| `bloc/` | ✅ siempre | event + state + bloc (`@injectable`, `@freezed`) |
| `widgets/` | ✅ siempre | Componentes UI del feature |
| `cubit/` | ⚙️ si se requiere | Para estado simple o sub-estado de un widget |
| `view/` | ⚙️ si se requiere | Scaffold + BlocBuilder cuando la UI es compleja |
| `utils/` | ⚙️ si se requiere | Formateadores y helpers de presentación |

---

## Flujo de Datos de Punta a Punta

```
UI: usuario abre ProductScreen
        │
        ▼
[presentation/screen] ProductScreen
  BlocProvider crea ProductBloc via getIt
  Dispara ProductStarted
        │
        ▼
[presentation/bloc] ProductBloc
  Llama GetProductsUseCase(NoParams())
        │
        ▼
[domain/use_case] GetProductsUseCase
  Llama ProductRepository.getProducts()
  (solo conoce la interfaz abstracta)
        │
        ▼
[data/repository_impl] ProductRepositoryImpl
  Llama ProductDatasource.fetchProducts()
  Captura FirebaseException → ServerFailure
  Retorna Either<Failure, List<ProductModel>>
  (ProductModel ya ES ProductEntity — extiende de ella)
        │
        ▼
[domain] Either sube de vuelta al BLoC
        │
        ▼
[presentation/bloc] ProductBloc emite ProductState.loaded(items)
[presentation/view] BlocBuilder reconstruye ProductList
        │
        ▼
UI muestra la lista
```

---

## core/ — Infraestructura Compartida

### core/di/injection.dart
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
Future<void> configureDependencies() => getIt.init();
```

### core/env/env.dart
```dart
import 'package:flutter_dotenv/flutter_dotenv.dart';

class Env {
  Env._();
  static String get apiBaseUrl  => _get('API_BASE_URL');
  static String get firebaseKey => _get('FIREBASE_API_KEY');
  static String get sentryDsn   => _get('SENTRY_DSN');

  static String _get(String key) {
    final value = dotenv.env[key];
    assert(value != null && value.isNotEmpty, '[$key] falta en .env');
    return value!;
  }
}
```

### core/error/failures.dart
```dart
import 'package:equatable/equatable.dart';

abstract class Failure extends Equatable {
  final String message;
  const Failure(this.message);
  @override
  List<Object> get props => [message];
}

class ServerFailure   extends Failure { const ServerFailure([super.message  = 'Error del servidor']); }
class NetworkFailure  extends Failure { const NetworkFailure([super.message = 'Sin conexión']); }
class CacheFailure    extends Failure { const CacheFailure([super.message   = 'Error de caché']); }
class ParseFailure    extends Failure { const ParseFailure([super.message   = 'Error al procesar datos']); }
class TimeoutFailure  extends Failure { const TimeoutFailure([super.message = 'Tiempo de espera agotado']); }

class AuthFailure extends Failure {
  const AuthFailure(super.message);
  factory AuthFailure.fromCode(String code) => switch (code) {
    'user-not-found'       => const AuthFailure('Usuario no encontrado'),
    'wrong-password'       => const AuthFailure('Contraseña incorrecta'),
    'email-already-in-use' => const AuthFailure('El correo ya está en uso'),
    'user-disabled'        => const AuthFailure('Cuenta deshabilitada'),
    'too-many-requests'    => const AuthFailure('Demasiados intentos'),
    _                      => const AuthFailure('Error de autenticación'),
  };
}
```

### core/utils/no_params.dart
```dart
import 'package:equatable/equatable.dart';

class NoParams extends Equatable {
  const NoParams();
  @override
  List<Object> get props => [];
}
```

---

## Code Generation

```bash
# Build completo
dart run build_runner build --delete-conflicting-outputs

# Watch durante desarrollo
dart run build_runner watch --delete-conflicting-outputs
```

Ejecutar siempre que:
- Se añade un `@RoutePage()` en `screen/`
- Se añade `@injectable`, `@lazySingleton` o `@singleton`
- Se modifica una clase `@freezed` en `bloc/` o `cubit/`
- Se modifica un `@JsonSerializable` en `model/`

| Archivo generado | Paquete |
|---|---|
| `app_router.gr.dart` | auto_route |
| `injection.config.dart` | injectable |
| `*.freezed.dart` | freezed |
| `*.g.dart` | json_serializable |

---

## Convenciones de Nombres

| Tipo | Convención | Ejemplo |
|---|---|---|
| Archivos | `snake_case` | `product_repository_impl.dart` |
| Clases | `PascalCase` | `ProductRepositoryImpl` |
| Entity | sustantivo + Entity | `ProductEntity` |
| Model | sustantivo + Model | `ProductModel` |
| Repository (interfaz) | sustantivo + Repository | `ProductRepository` |
| Repository (impl) | sustantivo + RepositoryImpl | `ProductRepositoryImpl` |
| Datasource (interfaz) | sustantivo + Datasource | `ProductDatasource` |
| Datasource (impl) | + Impl | `ProductDatasourceImpl` |
| UseCase | verbo + sustantivo + UseCase | `GetProductsUseCase` |
| BLoC events | verbo + sujeto | `ProductStarted`, `ProductRefreshed` |
| BLoC states | adjetivo / sustantivo | `_Initial`, `_Loaded`, `_Error` |
| Screen | sustantivo + Screen | `ProductScreen` |
| View | sustantivo + View | `ProductView` |
| Cubit | sustantivo + Cubit | `CheckoutStepCubit` |

---

## pubspec.yaml

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_bloc: ^9.1.0
  equatable: ^2.0.7
  auto_route: ^10.2.2
  get_it: ^8.0.3
  injectable: ^2.5.0
  freezed_annotation: ^2.4.4
  json_annotation: ^4.9.0
  flutter_dotenv: ^5.2.1
  firebase_core: ^3.13.0
  firebase_auth: ^5.5.2
  cloud_firestore: ^5.6.5
  firebase_storage: ^12.4.4
  cloud_functions: ^5.2.5
  dartz: ^0.10.1
  cached_network_image: ^3.4.1
  dio: ^5.8.0+1

dev_dependencies:
  flutter_test:
    sdk: flutter
  auto_route_generator: ^10.2.2
  injectable_generator: ^2.5.0
  freezed: ^2.5.8
  json_serializable: ^6.9.0
  build_runner: ^2.4.14
  bloc_test: ^10.0.0
  mocktail: ^1.0.4
```