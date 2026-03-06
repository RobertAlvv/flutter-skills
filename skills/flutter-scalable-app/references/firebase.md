# Firebase Integration Patterns

## Package Versions (March 2026)

```yaml
dependencies:
  firebase_core: ^3.13.0
  firebase_auth: ^5.5.2
  cloud_firestore: ^5.6.5
  firebase_storage: ^12.4.4
  cloud_functions: ^5.2.5
  firebase_messaging: ^15.1.6    # Push notifications
```

Run `flutterfire configure` to generate `firebase_options.dart`.

---

## DI Module

```dart
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

## Firebase Auth — Full Flow

### Repository Interface
```dart
// features/auth/domain/repositories/auth_repository.dart
abstract class AuthRepository {
  Stream<Option<UserEntity>> get authStateChanges;
  Future<Either<AuthFailure, UserEntity>> signInWithEmail({
    required String email, required String password,
  });
  Future<Either<AuthFailure, UserEntity>> registerWithEmail({
    required String email, required String password,
  });
  Future<Either<AuthFailure, Unit>> signOut();
  UserEntity? get currentUser;
}
```

### Repository Implementation
```dart
@LazySingleton(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository {
  final FirebaseAuth _auth;
  AuthRepositoryImpl(this._auth);

  @override
  Stream<Option<UserEntity>> get authStateChanges =>
      _auth.authStateChanges().map(
        (user) => user == null
            ? const None()
            : Some(user.toEntity()),
      );

  @override
  Future<Either<AuthFailure, UserEntity>> signInWithEmail({
    required String email,
    required String password,
  }) async {
    try {
      final credential = await _auth.signInWithEmailAndPassword(
        email: email,
        password: password,
      );
      return Right(credential.user!.toEntity());
    } on FirebaseAuthException catch (e) {
      return Left(AuthFailure.fromCode(e.code));
    }
  }

  @override
  Future<Either<AuthFailure, Unit>> signOut() async {
    try {
      await _auth.signOut();
      return const Right(unit);
    } catch (_) {
      return Left(const AuthFailure('Sign out failed'));
    }
  }

  @override
  UserEntity? get currentUser => _auth.currentUser?.toEntity();
}

// Extension to map Firebase User → domain entity
extension on User {
  UserEntity toEntity() => UserEntity(
    id: uid,
    email: email ?? '',
    displayName: displayName,
    photoUrl: photoURL,
  );
}
```

---

## Firestore — CRUD with Either

```dart
@LazySingleton(as: ProductRepository)
class ProductRepositoryImpl implements ProductRepository {
  final FirebaseFirestore _firestore;
  ProductRepositoryImpl(this._firestore);

  CollectionReference get _collection =>
      _firestore.collection('products');

  @override
  Future<Either<Failure, List<ProductEntity>>> getProducts() async {
    try {
      final snapshot = await _collection
          .orderBy('createdAt', descending: true)
          .limit(50)
          .get();

      final products = snapshot.docs.map((doc) {
        return ProductModel.fromJson({
          ...doc.data() as Map<String, dynamic>,
          'id': doc.id,
        }).toEntity();
      }).toList();

      return Right(products);
    } on FirebaseException catch (e) {
      return Left(ServerFailure(e.message ?? 'Firestore error'));
    } catch (_) {
      return Left(const ParseFailure());
    }
  }

  @override
  Future<Either<Failure, Unit>> createProduct(
    ProductEntity product,
  ) async {
    try {
      await _collection.doc(product.id).set(
        ProductModel.fromEntity(product).toJson(),
      );
      return const Right(unit);
    } on FirebaseException catch (e) {
      return Left(ServerFailure(e.message ?? 'Create failed'));
    }
  }

  @override
  Future<Either<Failure, Unit>> deleteProduct(String id) async {
    try {
      await _collection.doc(id).delete();
      return const Right(unit);
    } on FirebaseException catch (e) {
      return Left(ServerFailure(e.message ?? 'Delete failed'));
    }
  }
}
```

---

## Firestore — Real-Time Streams

```dart
@override
Stream<Either<Failure, List<NotificationEntity>>> watchNotifications(
  String userId,
) {
  return _firestore
      .collection('users')
      .doc(userId)
      .collection('notifications')
      .orderBy('timestamp', descending: true)
      .limit(50)
      .snapshots()
      .map((snapshot) {
        try {
          final items = snapshot.docs.map((doc) {
            return NotificationModel.fromJson({
              ...doc.data(),
              'id': doc.id,
            }).toEntity();
          }).toList();
          return Right<Failure, List<NotificationEntity>>(items);
        } catch (_) {
          return Left<Failure, List<NotificationEntity>>(
            const ParseFailure(),
          );
        }
      });
}
```

---

## Firestore — Cursor Pagination

```dart
@injectable
class ProductPaginationCubit extends Cubit<ProductPaginationState> {
  final FirebaseFirestore _firestore;
  DocumentSnapshot? _lastDocument;
  static const _pageSize = 20;
  bool _hasMore = true;

  ProductPaginationCubit(this._firestore)
      : super(const ProductPaginationState.initial());

  Future<void> loadFirstPage() async {
    if (state is _Loading) return;
    _lastDocument = null;
    _hasMore = true;
    emit(const ProductPaginationState.loading());
    await _loadPage(replace: true);
  }

  Future<void> loadNextPage() async {
    if (!_hasMore || state is _LoadingMore) return;
    final current = state.items;
    emit(ProductPaginationState.loadingMore(current));
    await _loadPage(replace: false);
  }

  Future<void> _loadPage({required bool replace}) async {
    Query query = _firestore
        .collection('products')
        .orderBy('createdAt', descending: true)
        .limit(_pageSize);

    if (_lastDocument != null) {
      query = query.startAfterDocument(_lastDocument!);
    }

    final snapshot = await query.get();

    if (snapshot.docs.isNotEmpty) {
      _lastDocument = snapshot.docs.last;
    }

    _hasMore = snapshot.docs.length == _pageSize;

    final newItems = snapshot.docs.map((doc) {
      return ProductModel.fromJson({...doc.data() as Map<String, dynamic>, 'id': doc.id})
          .toEntity();
    }).toList();

    final allItems = replace ? newItems : [...state.items, ...newItems];

    emit(ProductPaginationState.loaded(
      items: allItems,
      hasMore: _hasMore,
    ));
  }
}
```

---

## Firebase Storage — Upload with Progress

```dart
@injectable
class UploadMediaUseCase {
  final FirebaseStorage _storage;
  UploadMediaUseCase(this._storage);

  Stream<UploadProgress> call(UploadParams params) async* {
    final ref = _storage
        .ref()
        .child('${params.folder}/${params.userId}/${DateTime.now().millisecondsSinceEpoch}.jpg');

    final uploadTask = ref.putData(
      params.bytes,
      SettableMetadata(contentType: 'image/jpeg'),
    );

    await for (final snapshot in uploadTask.snapshotEvents) {
      final progress = snapshot.bytesTransferred / snapshot.totalBytes;

      if (snapshot.state == TaskState.success) {
        final url = await snapshot.ref.getDownloadURL();
        yield UploadProgress.complete(url);
      } else if (snapshot.state == TaskState.error) {
        yield const UploadProgress.error('Upload failed');
        return;
      } else {
        yield UploadProgress.uploading(progress);
      }
    }
  }
}

// BLoC usage
on<MediaUploadStarted>((event, emit) async {
  await emit.forEach<UploadProgress>(
    _uploadMedia(UploadParams(bytes: event.bytes, userId: event.userId)),
    onData: (progress) => progress.when(
      uploading: (p) => MediaState.uploading(p),
      complete: (url) => MediaState.uploaded(url),
      error:    (msg) => MediaState.error(msg),
    ),
  );
});
```

---

## Cloud Functions — Call from Flutter

```dart
@injectable
class CallAnalyticsUseCase {
  final FirebaseFunctions _functions;
  CallAnalyticsUseCase(this._functions);

  Future<Either<Failure, Map<String, dynamic>>> call(
    AnalyticsParams params,
  ) async {
    try {
      final callable = _functions.httpsCallable('generateReport');
      final result = await callable.call(params.toJson());
      return Right(result.data as Map<String, dynamic>);
    } on FirebaseFunctionsException catch (e) {
      return Left(ServerFailure(e.message ?? 'Function call failed'));
    }
  }
}
```

---

## Firestore Security Rules Checklist

Before deploying, verify:
```
✅ Authenticated users can only read/write their own documents
✅ Sensitive fields are write-protected (role, balance, admin)
✅ Document size limits enforced
✅ No wildcard reads on large collections
✅ Functions validate data server-side before writing
```

---

## Freezed Model with @JsonSerializable

```dart
// features/product/data/model/product_model.dart
@freezed
class ProductModel with _$ProductModel {
  const factory ProductModel({
    required String id,
    required String name,
    required double price,
    required String categoryId,
    @JsonKey(name: 'image_url') required String imageUrl,
    @JsonKey(name: 'created_at') required DateTime createdAt,
    @Default(false) bool isAvailable,
  }) = _ProductModel;

  factory ProductModel.fromJson(Map<String, dynamic> json) =>
      _$ProductModelFromJson(json);

  // Convenience factory from domain entity
  factory ProductModel.fromEntity(ProductEntity e) => ProductModel(
    id: e.id,
    name: e.name,
    price: e.price,
    categoryId: e.categoryId,
    imageUrl: e.imageUrl,
    createdAt: e.createdAt,
    isAvailable: e.isAvailable,
  );
}

// Extension for clean mapping to domain
extension ProductModelX on ProductModel {
  ProductEntity toEntity() => ProductEntity(
    id: id,
    name: name,
    price: price,
    categoryId: categoryId,
    imageUrl: imageUrl,
    createdAt: createdAt,
    isAvailable: isAvailable,
  );
}
```