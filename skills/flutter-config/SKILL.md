---
name: flutter-config
description: Set up Flutter app configuration — build environments (dev/staging/prod) via --dart-define-from-file, a typed AppConfig, optional Android/iOS flavors, a Dio ApiClient with auth/logging interceptors and exception mapping, local + secure storage wrappers, and app constants. Use when the user asks about environments, base URLs, flavors, config, API client/networking setup, interceptors, tokens storage, or constants in a Flutter project.
---

# Flutter Config — Environments, Network, Storage, Constants

Part of the `flutter-*` skill set (run after `flutter-init`). Replace `app` in
`package:app/...` with the package name from `pubspec.yaml`. Inspect existing code
first; if a config/network layer exists, extend it rather than replacing it.

## Principles

- **One source of truth per environment:** `env/<env>.json` → compiled in with
  `--dart-define-from-file`. No `if (isProd)` URL switches scattered in code.
- **No real secrets in the client.** Anything in the app binary can be extracted.
  Env files hold URLs, feature flags and public keys only. Server secrets stay on
  the server.
- **Commit `env/*.example.json`**; add real env files to `.gitignore` if they hold
  anything non-public.
- Config is read-only at runtime and accessed through `AppConfig`, never through
  `String.fromEnvironment` elsewhere.

## 1. Environment files

```
env/
├── dev.json
├── staging.json
├── prod.json
└── example.json      # committed template
```

`env/dev.json`

```json
{
  "ENV": "dev",
  "APP_NAME": "App Dev",
  "API_BASE_URL": "https://dev-api.example.com/v1/",
  "ENABLE_LOGS": true,
  "CONNECT_TIMEOUT_SECONDS": 30
}
```

Run / build:

```bash
flutter run --dart-define-from-file=env/dev.json
flutter build apk --release --dart-define-from-file=env/prod.json
flutter build ipa --release --dart-define-from-file=env/prod.json
```

`.vscode/launch.json` (create or merge):

```json
{
  "version": "0.2.0",
  "configurations": [
    { "name": "dev", "request": "launch", "type": "dart", "args": ["--dart-define-from-file=env/dev.json"] },
    { "name": "staging", "request": "launch", "type": "dart", "args": ["--dart-define-from-file=env/staging.json"] },
    { "name": "prod", "request": "launch", "type": "dart", "flutterMode": "release", "args": ["--dart-define-from-file=env/prod.json"] }
  ]
}
```

## 2. `lib/core/config/app_environment.dart`

```dart
enum AppEnvironment {
  dev,
  staging,
  prod;

  static AppEnvironment fromName(String value) =>
      values.firstWhere((e) => e.name == value, orElse: () => AppEnvironment.dev);

  bool get isProd => this == prod;
}
```

## 3. `lib/core/config/app_config.dart`

```dart
import 'package:app/core/config/app_environment.dart';

abstract final class AppConfig {
  static final AppEnvironment environment =
      AppEnvironment.fromName(const String.fromEnvironment('ENV', defaultValue: 'dev'));

  static const String appName = String.fromEnvironment('APP_NAME', defaultValue: 'App');
  static const String apiBaseUrl = String.fromEnvironment('API_BASE_URL');
  static const bool enableLogs = bool.fromEnvironment('ENABLE_LOGS');
  static const Duration connectTimeout =
      Duration(seconds: int.fromEnvironment('CONNECT_TIMEOUT_SECONDS', defaultValue: 30));
  static const Duration receiveTimeout = Duration(seconds: 30);

  static bool get isProd => environment.isProd;

  /// Call once in bootstrap. Fails fast when the app was launched without an env file.
  static void validate() {
    if (apiBaseUrl.isEmpty) {
      throw StateError(
        'API_BASE_URL is empty. Run with --dart-define-from-file=env/<env>.json',
      );
    }
  }
}
```

`String.fromEnvironment` only works in a `const` context — keep these as `static const`.
Call `AppConfig.validate()` in `bootstrap()` before `initDependencies()`.

To add a new value: add it to **every** `env/*.json` + `example.json`, then add one
typed field here.

## 4. Flavors (optional)

Only needed when environments must install side by side (different bundle id,
app name or icon). Ask the user before adding — it touches native projects.

Android `android/app/build.gradle.kts`:

```kotlin
android {
    flavorDimensions += "env"
    productFlavors {
        create("dev") { dimension = "env"; applicationIdSuffix = ".dev"; resValue("string", "app_name", "App Dev") }
        create("staging") { dimension = "env"; applicationIdSuffix = ".stg"; resValue("string", "app_name", "App Stg") }
        create("prod") { dimension = "env"; resValue("string", "app_name", "App") }
    }
}
```

Set `android:label="@string/app_name"` in `AndroidManifest.xml`. iOS requires Xcode
schemes + build configurations per flavor (Debug-dev, Release-dev, …) — walk the user
through it in Xcode rather than editing `project.pbxproj` by hand.
Run with `flutter run --flavor dev --dart-define-from-file=env/dev.json`.

## 5. Constants

`lib/core/constants/app_constants.dart`

```dart
abstract final class AppConstants {
  static const int pageSize = 20;
  static const Duration debounce = Duration(milliseconds: 400);
  static const Duration snackBarDuration = Duration(seconds: 3);
  static const String dateFormat = 'dd MMM yyyy';
}
```

`lib/core/constants/storage_keys.dart`

```dart
abstract final class StorageKeys {
  // SharedPreferences
  static const String themeMode = 'theme_mode';
  static const String locale = 'locale';
  static const String onboardingSeen = 'onboarding_seen';

  // Secure storage
  static const String accessToken = 'access_token';
  static const String refreshToken = 'refresh_token';
}
```

Rule: no string literal keys anywhere else.

## 6. Local storage — `lib/core/local/local_storage.dart`

```bash
flutter pub add shared_preferences flutter_secure_storage
```

```dart
import 'package:app/core/constants/storage_keys.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:shared_preferences/shared_preferences.dart';

class LocalStorage {
  LocalStorage(this._prefs, this._secure);
  final SharedPreferences _prefs;
  final FlutterSecureStorage _secure;

  // Plain values
  String? getString(String key) => _prefs.getString(key);
  Future<void> setString(String key, String value) => _prefs.setString(key, value);
  bool getBool(String key, {bool fallback = false}) => _prefs.getBool(key) ?? fallback;
  Future<void> setBool(String key, {required bool value}) => _prefs.setBool(key, value);
  Future<void> remove(String key) => _prefs.remove(key);

  // Tokens — always in secure storage
  Future<String?> get accessToken => _secure.read(key: StorageKeys.accessToken);
  Future<String?> get refreshToken => _secure.read(key: StorageKeys.refreshToken);

  Future<void> saveTokens({required String access, String? refresh}) async {
    await _secure.write(key: StorageKeys.accessToken, value: access);
    if (refresh != null) await _secure.write(key: StorageKeys.refreshToken, value: refresh);
  }

  Future<void> clearSession() async {
    await _secure.delete(key: StorageKeys.accessToken);
    await _secure.delete(key: StorageKeys.refreshToken);
  }
}
```

## 7. Network layer — `lib/core/network/`

```
network/
├── api_client.dart
├── api_endpoints.dart
├── safe_call.dart
└── interceptors/
    └── auth_interceptor.dart
```

### `api_endpoints.dart`

```dart
abstract final class ApiEndpoints {
  // Auth
  static const String login = 'auth/login';
  static const String refresh = 'auth/refresh';

  // Example feature
  static const String products = 'products';
  static String product(String id) => 'products/$id';
}
```

Paths are relative (no leading `/`) so they append to a base URL ending in `/`.

### `api_client.dart`

```dart
import 'package:app/core/config/app_config.dart';
import 'package:app/core/error/app_exception.dart';
import 'package:dio/dio.dart';

class ApiClient {
  ApiClient({required List<Interceptor> interceptors, Dio? dio})
      : _dio = dio ??
            Dio(
              BaseOptions(
                baseUrl: AppConfig.apiBaseUrl,
                connectTimeout: AppConfig.connectTimeout,
                receiveTimeout: AppConfig.receiveTimeout,
                headers: {'Accept': 'application/json'},
              ),
            ) {
    _dio.interceptors.addAll(interceptors);
    if (AppConfig.enableLogs) {
      _dio.interceptors.add(LogInterceptor(requestBody: true, responseBody: true));
    }
  }

  final Dio _dio;
  Dio get dio => _dio;

  Future<T> get<T>(String path, {Map<String, dynamic>? query, Options? options}) =>
      _send(() => _dio.get<T>(path, queryParameters: query, options: options));

  Future<T> post<T>(String path, {Object? data, Map<String, dynamic>? query, Options? options}) =>
      _send(() => _dio.post<T>(path, data: data, queryParameters: query, options: options));

  Future<T> put<T>(String path, {Object? data, Options? options}) =>
      _send(() => _dio.put<T>(path, data: data, options: options));

  Future<T> patch<T>(String path, {Object? data, Options? options}) =>
      _send(() => _dio.patch<T>(path, data: data, options: options));

  Future<T> delete<T>(String path, {Object? data, Options? options}) =>
      _send(() => _dio.delete<T>(path, data: data, options: options));

  Future<T> _send<T>(Future<Response<T>> Function() request) async {
    try {
      final response = await request();
      return response.data as T;
    } on DioException catch (e) {
      throw _map(e);
    }
  }

  AppException _map(DioException e) {
    switch (e.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.sendTimeout:
      case DioExceptionType.receiveTimeout:
      case DioExceptionType.connectionError:
        return const NetworkException();
      case DioExceptionType.badResponse:
        final code = e.response?.statusCode;
        if (code == 401) return const UnauthorizedException();
        return ServerException(_messageFrom(e.response?.data), statusCode: code);
      case DioExceptionType.cancel:
      case DioExceptionType.badCertificate:
      case DioExceptionType.unknown:
        return ServerException(e.message ?? 'Unexpected error');
    }
  }

  /// Adjust to the backend's error envelope.
  String _messageFrom(Object? data) {
    if (data is Map<String, dynamic>) {
      final msg = data['message'] ?? data['error'] ?? data['title'];
      if (msg is String && msg.isNotEmpty) return msg;
    }
    return 'Something went wrong';
  }
}
```

If the backend wraps every response (e.g. `{ "data": …, "success": true }`), unwrap it in
`_send` once instead of in every data source.

### `interceptors/auth_interceptor.dart`

`QueuedInterceptor` serializes requests so only one token refresh happens at a time.

```dart
import 'package:app/core/config/app_config.dart';
import 'package:app/core/constants/storage_keys.dart';
import 'package:app/core/local/local_storage.dart';
import 'package:app/core/network/api_endpoints.dart';
import 'package:dio/dio.dart';

class AuthInterceptor extends QueuedInterceptor {
  AuthInterceptor(this._storage, {this.onSessionExpired});

  final LocalStorage _storage;
  final void Function()? onSessionExpired;
  final Dio _refreshDio = Dio(BaseOptions(baseUrl: AppConfig.apiBaseUrl));

  static const _retried = 'auth_retried';

  @override
  Future<void> onRequest(RequestOptions options, RequestInterceptorHandler handler) async {
    final token = await _storage.accessToken;
    if (token != null && token.isNotEmpty) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }

  @override
  Future<void> onError(DioException err, ErrorInterceptorHandler handler) async {
    final is401 = err.response?.statusCode == 401;
    final alreadyRetried = err.requestOptions.extra[_retried] == true;
    if (!is401 || alreadyRetried) return handler.next(err);

    final refreshed = await _refresh();
    if (!refreshed) {
      await _storage.clearSession();
      onSessionExpired?.call();
      return handler.next(err);
    }

    final opts = err.requestOptions
      ..extra[_retried] = true
      ..headers['Authorization'] = 'Bearer ${await _storage.accessToken}';
    try {
      handler.resolve(await _refreshDio.fetch<dynamic>(opts));
    } on DioException catch (e) {
      handler.next(e);
    }
  }

  Future<bool> _refresh() async {
    final refresh = await _storage.refreshToken;
    if (refresh == null) return false;
    try {
      final res = await _refreshDio.post<Map<String, dynamic>>(
        ApiEndpoints.refresh,
        data: {StorageKeys.refreshToken: refresh}, // adjust to backend contract
      );
      final access = res.data?['access_token'] as String?;
      if (access == null) return false;
      await _storage.saveTokens(access: access, refresh: res.data?['refresh_token'] as String?);
      return true;
    } on DioException {
      return false;
    }
  }
}
```

Adapt the refresh payload/response field names to the actual API. If the app has no
auth, skip this interceptor.

### `safe_call.dart` — exception → failure, used by every repository

```dart
import 'dart:developer';

import 'package:app/core/error/app_exception.dart';
import 'package:app/core/error/failure.dart';
import 'package:app/core/utils/result.dart';

Future<Result<T>> safeCall<T>(Future<T> Function() call) async {
  try {
    return Ok(await call());
  } on AppException catch (e) {
    return Err(switch (e) {
      NetworkException() => NetworkFailure(e.message),
      UnauthorizedException() => UnauthorizedFailure(e.message),
      CacheException() => CacheFailure(e.message),
      ServerException() => ServerFailure(e.message, statusCode: e.statusCode),
    });
  } catch (e, s) {
    log('Unhandled error in safeCall', error: e, stackTrace: s);
    return const Err(UnknownFailure());
  }
}
```

## 8. Register in `injection_container.dart` → `_initCore()`

```dart
final prefs = await SharedPreferences.getInstance();
sl
  ..registerSingleton<SharedPreferences>(prefs)
  ..registerLazySingleton(() => const FlutterSecureStorage())
  ..registerLazySingleton(() => LocalStorage(sl(), sl()))
  ..registerLazySingleton(() => AuthInterceptor(sl()))
  ..registerLazySingleton(() => ApiClient(interceptors: [sl<AuthInterceptor>()]));
```

Multiple backends? Register named clients:
`sl.registerLazySingleton(() => ApiClient(...), instanceName: 'payments')` and give each
its own base URL key in the env files.

## Checklist

- [ ] `env/*.json` for each environment + committed `example.json`
- [ ] `AppConfig.validate()` called in bootstrap
- [ ] launch configs for each env
- [ ] no hardcoded URLs/keys outside env files; no `String.fromEnvironment` outside AppConfig
- [ ] tokens only in secure storage
- [ ] `flutter analyze` clean
