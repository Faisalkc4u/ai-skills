---
name: flutter-init
description: Initialize a Flutter app with a feature-first clean-architecture folder structure (core/ + features/), dependency injection, bootstrap, error/result types, and a repeatable "add a feature" scaffold. Use when the user wants to set up, initialize, bootstrap or restructure a Flutter project, create the folder structure, or add a new feature module. Orchestrates flutter-config, flutter-theme, flutter-routing and flutter-components.
---

# Flutter Init — Folder Structure & Project Bootstrap

Sets up a scalable, feature-first Flutter project. This skill is the entry point;
the other `flutter-*` skills fill in the details:

| Order | Skill                | Produces                                              |
|-------|----------------------|-------------------------------------------------------|
| 1     | `flutter-init`       | Folders, DI, bootstrap, Result/Failure, feature scaffold |
| 2     | `flutter-config`     | Environments, AppConfig, API client, storage, constants |
| 3     | `flutter-theme`      | Colors, fonts, text styles, spacing, light/dark theme |
| 4     | `flutter-routing`    | Route enum, go_router, guards, enums conventions      |
| 5     | `flutter-components` | Shared UI components + usage rules                    |

## Step 0 — Inspect before writing

1. Read `pubspec.yaml`: get the package `name:` (used in every `package:<name>/` import),
   the Dart SDK constraint, and existing dependencies.
2. List `lib/` to see what already exists.
3. **Adapt to existing conventions.** If the project already uses Riverpod/Provider,
   auto_route, a different folder style, etc., keep it and only fill the gaps.
   Tell the user what you found and what you plan to change.
4. **Never overwrite or delete an existing file without asking.** For a new project,
   the default `lib/main.dart` counter app can be replaced.

In every snippet below, replace `app` in `package:app/...` with the real package name.

## Step 1 — Default stack (confirm only if the user hasn't said)

| Concern          | Default                     | Package(s)                                 |
|------------------|-----------------------------|--------------------------------------------|
| State management | Cubit                       | `flutter_bloc`, `equatable`                |
| DI               | Service locator             | `get_it`                                   |
| Networking       | Dio                         | `dio`                                      |
| Routing          | go_router                   | `go_router`                                |
| Storage          | Prefs + secure storage      | `shared_preferences`, `flutter_secure_storage` |
| Images           | SVG + cached network        | `flutter_svg`, `cached_network_image`      |
| Lints            | Official                    | `flutter_lints` (dev)                      |

Install with the CLI (never hand-edit versions):

```bash
flutter pub add flutter_bloc equatable get_it dio go_router shared_preferences flutter_secure_storage flutter_svg cached_network_image
```

Minimum toolchain assumed: Flutter 3.27+ / Dart 3.6+ (uses `Color.withValues`,
sealed classes, enhanced enums, `CardThemeData`). If older, adapt APIs.

## Step 2 — Folder structure

```
lib/
├── main.dart                     # entry: calls bootstrap()
├── bootstrap.dart                # binding, error handlers, DI, runApp
├── app.dart                      # MaterialApp.router (theme, router, locale)
├── injection_container.dart      # GetIt `sl` + initDependencies()
│
├── core/                         # shared, feature-agnostic code ONLY
│   ├── config/                   # app_config.dart, app_environment.dart   (flutter-config)
│   ├── constants/                # app_constants.dart, storage_keys.dart   (flutter-config)
│   ├── enums/                    # cross-feature enums, view_status.dart   (flutter-routing)
│   ├── error/                    # failure.dart, app_exception.dart
│   ├── network/                  # api_client.dart, api_endpoints.dart, interceptors/
│   ├── local/                    # local_storage.dart
│   ├── router/                   # app_route.dart, app_router.dart, navigation_ext.dart
│   ├── theme/                    # app_colors, app_text_styles, app_fonts, app_spacing,
│   │                             # app_assets, app_theme, theme_cubit    (flutter-theme)
│   ├── components/               # shared widgets + components.dart barrel (flutter-components)
│   │   ├── buttons/  inputs/  feedback/  media/  overlays/  layout/
│   └── utils/
│       ├── extensions/           # context_ext, string_ext, date_ext …
│       ├── validators/
│       └── result.dart
│
└── features/
    └── <feature_name>/           # snake_case, one folder per business feature
        ├── data/
        │   ├── datasources/      # <feature>_remote_data_source.dart (+ local)
        │   ├── models/           # DTOs with fromJson/toJson, extend/convert to entities
        │   └── repositories/     # <feature>_repository_impl.dart
        ├── domain/
        │   ├── entities/         # pure Dart, no Flutter/JSON
        │   ├── enums/            # feature-only enums
        │   ├── repositories/     # abstract <feature>_repository.dart
        │   └── usecases/         # optional — only when logic is non-trivial or reused
        ├── presentation/
        │   ├── cubit/            # <name>_cubit.dart + <name>_state.dart
        │   ├── pages/            # routed screens: <name>_page.dart
        │   └── widgets/          # widgets used only by this feature
        └── <feature>_injection.dart

assets/
├── images/   icons/   fonts/   animations/   translations/
env/          # dev.json, staging.json, prod.json   (flutter-config)
test/         # mirrors lib/
```

### Dependency rules (enforce these)

- `presentation → domain ← data`. Domain imports nothing from data/presentation.
- `features/a` must **not** import `features/b` internals. Share via `core/`, or
  expose a small public API (entity/repository) and import only that.
- `core/` never imports from `features/`.
- Widgets never call `sl<>()` for repositories — they talk to a Cubit.
- Only `pages/` are routed. `widgets/` are private building blocks.

### Naming

- Files `snake_case.dart`, classes `PascalCase`, one public class per file.
- Suffixes: `…Page`, `…Cubit`, `…State`, `…Model`, `…Entity` (or plain name),
  `…Repository`, `…RepositoryImpl`, `…RemoteDataSource`, `…UseCase`.

## Step 3 — Core files

Create the directories, add `.gitkeep` to empty ones, and register asset folders in
`pubspec.yaml` (`flutter: assets:` with `assets/images/`, `assets/icons/`, …).

### `lib/main.dart`

```dart
import 'package:app/app.dart';
import 'package:app/bootstrap.dart';

Future<void> main() => bootstrap(() => const App());
```

### `lib/bootstrap.dart`

```dart
import 'dart:async';
import 'dart:developer';
import 'dart:ui';

import 'package:app/injection_container.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

Future<void> bootstrap(Widget Function() builder) async {
  WidgetsFlutterBinding.ensureInitialized();

  FlutterError.onError = (details) {
    FlutterError.presentError(details);
    // TODO: forward to crash reporting (Sentry/Crashlytics).
  };
  PlatformDispatcher.instance.onError = (error, stack) {
    log('Uncaught error', error: error, stackTrace: stack);
    // TODO: forward to crash reporting.
    return true;
  };

  await SystemChrome.setPreferredOrientations([DeviceOrientation.portraitUp]);
  await initDependencies();

  runApp(builder());
}
```

### `lib/app.dart`

```dart
import 'package:app/core/router/app_router.dart';
import 'package:app/core/theme/app_theme.dart';
import 'package:app/core/theme/theme_cubit.dart';
import 'package:app/injection_container.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider.value(
      value: sl<ThemeCubit>(),
      child: BlocBuilder<ThemeCubit, ThemeMode>(
        builder: (context, themeMode) => MaterialApp.router(
          title: 'App',
          debugShowCheckedModeBanner: false,
          theme: AppTheme.light,
          darkTheme: AppTheme.dark,
          themeMode: themeMode,
          routerConfig: sl<AppRouter>().router,
          // Add localizationsDelegates / supportedLocales / locale here if the app is localized.
        ),
      ),
    );
  }
}
```

### `lib/injection_container.dart`

```dart
import 'package:get_it/get_it.dart';

final GetIt sl = GetIt.instance;

Future<void> initDependencies() async {
  await _initCore();
  _initFeatures();
}

Future<void> _initCore() async {
  // Registered by the other skills:
  //  flutter-config  → SharedPreferences, FlutterSecureStorage, LocalStorage, ApiClient
  //  flutter-theme   → ThemeCubit
  //  flutter-routing → AppRouter
}

void _initFeatures() {
  // One line per feature, e.g. initProductsFeature();
}
```

Registration rules: `registerLazySingleton` for clients/data sources/repositories/
use cases and app-wide cubits (theme, session); `registerFactory` for screen cubits
(a fresh instance per screen, closed by `BlocProvider`).

### `lib/core/error/app_exception.dart` (thrown in data layer)

```dart
sealed class AppException implements Exception {
  const AppException(this.message, {this.statusCode});
  final String message;
  final int? statusCode;
}

final class ServerException extends AppException {
  const ServerException(super.message, {super.statusCode});
}

final class NetworkException extends AppException {
  const NetworkException([super.message = 'No internet connection']);
}

final class UnauthorizedException extends AppException {
  const UnauthorizedException([super.message = 'Session expired']) : super(statusCode: 401);
}

final class CacheException extends AppException {
  const CacheException([super.message = 'Local storage error']);
}
```

### `lib/core/error/failure.dart` (returned to presentation)

```dart
import 'package:equatable/equatable.dart';

sealed class Failure extends Equatable {
  const Failure(this.message);
  final String message;

  @override
  List<Object?> get props => [message];
}

final class ServerFailure extends Failure {
  const ServerFailure(super.message, {this.statusCode});
  final int? statusCode;
}

final class NetworkFailure extends Failure {
  const NetworkFailure([super.message = 'No internet connection']);
}

final class UnauthorizedFailure extends Failure {
  const UnauthorizedFailure([super.message = 'Session expired']);
}

final class CacheFailure extends Failure {
  const CacheFailure([super.message = 'Local storage error']);
}

final class UnknownFailure extends Failure {
  const UnknownFailure([super.message = 'Something went wrong']);
}
```

### `lib/core/utils/result.dart`

Dependency-free alternative to `dartz`/`fpdart` `Either`. If the project already uses
`Either<Failure, T>`, keep that instead.

```dart
import 'package:app/core/error/failure.dart';

sealed class Result<T> {
  const Result();

  R fold<R>(R Function(Failure failure) onFailure, R Function(T data) onSuccess) =>
      switch (this) {
        Ok<T>(:final data) => onSuccess(data),
        Err<T>(:final failure) => onFailure(failure),
      };
}

final class Ok<T> extends Result<T> {
  const Ok(this.data);
  final T data;
}

final class Err<T> extends Result<T> {
  const Err(this.failure);
  final Failure failure;
}
```

### `lib/core/enums/view_status.dart`

```dart
enum ViewStatus {
  initial,
  loading,
  success,
  failure;

  bool get isLoading => this == loading;
  bool get isSuccess => this == success;
  bool get isFailure => this == failure;
}
```

## Step 4 — Hand off

Run the other skills in order (config → theme → routing → components), then:

```bash
dart format lib test
flutter analyze
flutter test
```

Fix every analyzer error before reporting done. Summarize the created files to the user.

---

## Adding a new feature (reusable recipe)

When the user says "add feature X", generate these files, using `products` / `Product`
as the example name. Skip `usecases/` for simple CRUD — the cubit can call the repository.

**`domain/entities/product.dart`**

```dart
import 'package:equatable/equatable.dart';

class Product extends Equatable {
  const Product({required this.id, required this.name, required this.price});
  final String id;
  final String name;
  final double price;

  @override
  List<Object?> get props => [id, name, price];
}
```

**`data/models/product_model.dart`**

```dart
import 'package:app/features/products/domain/entities/product.dart';

class ProductModel extends Product {
  const ProductModel({required super.id, required super.name, required super.price});

  factory ProductModel.fromJson(Map<String, dynamic> json) => ProductModel(
        id: json['id'].toString(),
        name: json['name'] as String? ?? '',
        price: (json['price'] as num?)?.toDouble() ?? 0,
      );

  Map<String, dynamic> toJson() => {'id': id, 'name': name, 'price': price};
}
```

Parse defensively (nullable casts + defaults) — never let one bad field crash a list.

**`domain/repositories/products_repository.dart`**

```dart
import 'package:app/core/utils/result.dart';
import 'package:app/features/products/domain/entities/product.dart';

abstract interface class ProductsRepository {
  Future<Result<List<Product>>> getProducts();
  Future<Result<Product>> getProduct(String id);
}
```

**`data/datasources/products_remote_data_source.dart`**

```dart
import 'package:app/core/network/api_client.dart';
import 'package:app/core/network/api_endpoints.dart';
import 'package:app/features/products/data/models/product_model.dart';

abstract interface class ProductsRemoteDataSource {
  Future<List<ProductModel>> getProducts();
  Future<ProductModel> getProduct(String id);
}

class ProductsRemoteDataSourceImpl implements ProductsRemoteDataSource {
  const ProductsRemoteDataSourceImpl(this._client);
  final ApiClient _client;

  @override
  Future<List<ProductModel>> getProducts() async {
    final data = await _client.get<List<dynamic>>(ApiEndpoints.products);
    return data.map((e) => ProductModel.fromJson(e as Map<String, dynamic>)).toList();
  }

  @override
  Future<ProductModel> getProduct(String id) async {
    final data = await _client.get<Map<String, dynamic>>(ApiEndpoints.product(id));
    return ProductModel.fromJson(data);
  }
}
```

**`data/repositories/products_repository_impl.dart`**

```dart
import 'package:app/core/network/safe_call.dart';
import 'package:app/core/utils/result.dart';
import 'package:app/features/products/data/datasources/products_remote_data_source.dart';
import 'package:app/features/products/domain/entities/product.dart';
import 'package:app/features/products/domain/repositories/products_repository.dart';

class ProductsRepositoryImpl implements ProductsRepository {
  const ProductsRepositoryImpl(this._remote);
  final ProductsRemoteDataSource _remote;

  @override
  Future<Result<List<Product>>> getProducts() => safeCall(_remote.getProducts);

  @override
  Future<Result<Product>> getProduct(String id) => safeCall(() => _remote.getProduct(id));
}
```

(`safeCall` maps exceptions → failures; defined in `flutter-config`.)

**`presentation/cubit/products_state.dart`**

```dart
part of 'products_cubit.dart';

class ProductsState extends Equatable {
  const ProductsState({
    this.status = ViewStatus.initial,
    this.items = const [],
    this.errorMessage,
  });

  final ViewStatus status;
  final List<Product> items;
  final String? errorMessage;

  ProductsState copyWith({ViewStatus? status, List<Product>? items, String? errorMessage}) =>
      ProductsState(
        status: status ?? this.status,
        items: items ?? this.items,
        errorMessage: errorMessage,
      );

  @override
  List<Object?> get props => [status, items, errorMessage];
}
```

**`presentation/cubit/products_cubit.dart`**

```dart
import 'package:app/core/enums/view_status.dart';
import 'package:app/features/products/domain/entities/product.dart';
import 'package:app/features/products/domain/repositories/products_repository.dart';
import 'package:equatable/equatable.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

part 'products_state.dart';

class ProductsCubit extends Cubit<ProductsState> {
  ProductsCubit(this._repository) : super(const ProductsState());
  final ProductsRepository _repository;

  Future<void> load() async {
    emit(state.copyWith(status: ViewStatus.loading));
    final result = await _repository.getProducts();
    if (isClosed) return;
    result.fold(
      (failure) => emit(state.copyWith(status: ViewStatus.failure, errorMessage: failure.message)),
      (items) => emit(state.copyWith(status: ViewStatus.success, items: items)),
    );
  }
}
```

**`presentation/pages/products_page.dart`** — provide the cubit at the page level and
render with `AppStatusView` from `flutter-components`:

```dart
class ProductsPage extends StatelessWidget {
  const ProductsPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocProvider(
      create: (_) => sl<ProductsCubit>()..load(),
      child: const _ProductsView(),
    );
  }
}

class _ProductsView extends StatelessWidget {
  const _ProductsView();

  @override
  Widget build(BuildContext context) {
    return AppScaffold(
      title: 'Products',
      body: BlocBuilder<ProductsCubit, ProductsState>(
        builder: (context, state) => AppStatusView(
          status: state.status,
          errorMessage: state.errorMessage,
          onRetry: context.read<ProductsCubit>().load,
          isEmpty: state.items.isEmpty,
          onSuccess: (_) => ListView.separated(
            itemCount: state.items.length,
            separatorBuilder: (_, __) => AppGap.v8,
            itemBuilder: (_, i) => ProductTile(product: state.items[i]),
          ),
        ),
      ),
    );
  }
}
```

**`products_injection.dart`**

```dart
void initProductsFeature() {
  sl
    ..registerLazySingleton<ProductsRemoteDataSource>(() => ProductsRemoteDataSourceImpl(sl()))
    ..registerLazySingleton<ProductsRepository>(() => ProductsRepositoryImpl(sl()))
    ..registerFactory(() => ProductsCubit(sl()));
}
```

Then: call `initProductsFeature()` in `_initFeatures()`, add endpoints to `ApiEndpoints`,
add the route to `AppRoute` + `AppRouter` (see `flutter-routing`), and add a cubit test
under `test/features/products/`.
