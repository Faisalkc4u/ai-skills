---
name: flutter-routing
description: Set up typed navigation in Flutter with go_router — an AppRoute enum (name/path/public flag), a central AppRouter with auth redirect guard, bottom-nav StatefulShellRoute, context navigation helpers, navigation without context, plus conventions for Dart enhanced enums (API value mapping, unknown fallback, labels/colors via extensions). Use when the user wants to add routes, screens navigation, deep links, route guards, bottom navigation, or to create/refactor enums in a Flutter app.
---

# Flutter Routing & Enums

Part of the `flutter-*` skill set. Replace `app` in `package:app/...` with the package name.
If the project already uses another router (auto_route, Navigator 1.0 helpers), follow
its pattern. Only migrate if the user asks.

```bash
flutter pub add go_router
```

```
lib/core/router/
├── app_route.dart          # enum of every route
├── app_router.dart         # GoRouter config + guards
├── navigation_ext.dart     # context.goTo / pushTo helpers
└── stream_listenable.dart  # refresh router on auth changes
```

## 1. `app_route.dart` — single source of truth

```dart
enum AppRoute {
  splash('/splash', isPublic: true),
  onboarding('/onboarding', isPublic: true),
  login('/login', isPublic: true),

  // Bottom-nav tabs
  home('/home'),
  explore('/explore'),
  profile('/profile'),

  // Detail pages (path params for anything deep-linkable)
  productDetails('/products/:id'),
  settings('/settings'),
  notFound('/404', isPublic: true);

  const AppRoute(this.path, {this.isPublic = false});

  final String path;
  final bool isPublic;

  static AppRoute? fromPath(String? path) {
    for (final r in values) {
      if (r.path == path) return r;
    }
    return null;
  }
}
```

Rules:
- Every screen has exactly one `AppRoute` entry. Its `name` is the go_router route name.
- Use **absolute paths** everywhere (also inside shell branches) so `fullPath` matches `path`.
- IDs go in **path params** (`/products/:id`). Filters go in **query params**.
  `extra` is only for optional, non-essential objects (for example a preloaded entity to
  avoid a refetch). It's lost on deep link, web refresh and process restore, so the page
  must still work from the id alone.

## 2. `stream_listenable.dart`

```dart
import 'dart:async';

import 'package:flutter/foundation.dart';

/// Makes GoRouter re-run `redirect` whenever a stream (e.g. auth cubit) emits.
class StreamListenable extends ChangeNotifier {
  StreamListenable(Stream<dynamic> stream) {
    _sub = stream.listen((_) => notifyListeners());
  }

  late final StreamSubscription<dynamic> _sub;

  @override
  void dispose() {
    _sub.cancel();
    super.dispose();
  }
}
```

## 3. `app_router.dart`

`SessionCubit` is whatever holds auth state (`bool isLoggedIn`). If the app has no auth,
drop `redirect`/`refreshListenable`.

```dart
import 'package:app/core/config/app_config.dart';
import 'package:app/core/router/app_route.dart';
import 'package:app/core/router/stream_listenable.dart';
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';

final GlobalKey<NavigatorState> rootNavigatorKey = GlobalKey<NavigatorState>(debugLabel: 'root');

class AppRouter {
  AppRouter(this._session);
  final SessionCubit _session;

  late final GoRouter router = GoRouter(
    navigatorKey: rootNavigatorKey,
    initialLocation: AppRoute.splash.path,
    debugLogDiagnostics: AppConfig.enableLogs,
    refreshListenable: StreamListenable(_session.stream),
    redirect: _guard,
    errorBuilder: (_, __) => const NotFoundPage(),
    routes: [
      _route(AppRoute.splash, (_, __) => const SplashPage()),
      _route(AppRoute.onboarding, (_, __) => const OnboardingPage()),
      _route(AppRoute.login, (_, __) => const LoginPage()),

      StatefulShellRoute.indexedStack(
        builder: (_, __, shell) => MainShell(shell: shell),
        branches: [
          StatefulShellBranch(routes: [_route(AppRoute.home, (_, __) => const HomePage())]),
          StatefulShellBranch(routes: [_route(AppRoute.explore, (_, __) => const ExplorePage())]),
          StatefulShellBranch(routes: [_route(AppRoute.profile, (_, __) => const ProfilePage())]),
        ],
      ),

      // Full-screen pages above the bottom bar → parentNavigatorKey: root
      _route(
        AppRoute.productDetails,
        (_, s) => ProductDetailsPage(id: s.pathParameters['id']!),
      ),
      _route(AppRoute.settings, (_, __) => const SettingsPage()),
    ],
  );

  GoRoute _route(
    AppRoute route,
    Widget Function(BuildContext, GoRouterState) builder, {
    List<RouteBase> routes = const [],
  }) =>
      GoRoute(
        path: route.path,
        name: route.name,
        parentNavigatorKey: route.isShellTab ? null : rootNavigatorKey,
        builder: builder,
        routes: routes,
      );

  String? _guard(BuildContext context, GoRouterState state) {
    final loggedIn = _session.state.isLoggedIn;
    final route = AppRoute.fromPath(state.fullPath);
    final isPublic = route?.isPublic ?? false;

    if (route == AppRoute.splash) return null; // splash decides where to go
    if (!loggedIn && !isPublic) {
      final from = Uri.encodeComponent(state.uri.toString());
      return '${AppRoute.login.path}?from=$from';
    }
    if (loggedIn && route == AppRoute.login) {
      return state.uri.queryParameters['from'] ?? AppRoute.home.path;
    }
    return null;
  }
}

extension on AppRoute {
  bool get isShellTab =>
      this == AppRoute.home || this == AppRoute.explore || this == AppRoute.profile;
}
```

Register: `sl.registerLazySingleton(() => AppRouter(sl()));` (after `SessionCubit`).

Custom transitions: use `pageBuilder: (_, s) => CustomTransitionPage(key: s.pageKey, child: …, transitionsBuilder: …)`
instead of `builder` for specific routes.

### Bottom navigation shell

```dart
class MainShell extends StatelessWidget {
  const MainShell({super.key, required this.shell});
  final StatefulNavigationShell shell;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: shell,
      bottomNavigationBar: NavigationBar(
        selectedIndex: shell.currentIndex,
        onDestinationSelected: (i) => shell.goBranch(i, initialLocation: i == shell.currentIndex),
        destinations: const [
          NavigationDestination(icon: Icon(Icons.home_outlined), selectedIcon: Icon(Icons.home), label: 'Home'),
          NavigationDestination(icon: Icon(Icons.explore_outlined), selectedIcon: Icon(Icons.explore), label: 'Explore'),
          NavigationDestination(icon: Icon(Icons.person_outline), selectedIcon: Icon(Icons.person), label: 'Profile'),
        ],
      ),
    );
  }
}
```

The order of branches must match the order of destinations. Tapping the active tab again pops it to root.

## 4. `navigation_ext.dart`

```dart
import 'package:app/core/router/app_route.dart';
import 'package:flutter/widgets.dart';
import 'package:go_router/go_router.dart';

extension AppNavigationX on BuildContext {
  /// Replace the current stack location (tabs, after login, top-level jumps).
  void goTo(
    AppRoute route, {
    Map<String, String> params = const {},
    Map<String, dynamic> query = const {},
    Object? extra,
  }) =>
      goNamed(route.name, pathParameters: params, queryParameters: query, extra: extra);

  /// Push on top (details, forms). Awaitable for a result.
  Future<T?> pushTo<T extends Object?>(
    AppRoute route, {
    Map<String, String> params = const {},
    Map<String, dynamic> query = const {},
    Object? extra,
  }) =>
      pushNamed<T>(route.name, pathParameters: params, queryParameters: query, extra: extra);

  void popOr(AppRoute fallback) => canPop() ? pop() : goTo(fallback);
}
```

Usage:

```dart
context.pushTo(AppRoute.productDetails, params: {'id': product.id});
context.goTo(AppRoute.home);
final picked = await context.pushTo<Address>(AppRoute.addressPicker);
context.pop(result); // return a value
```

**Without context** (interceptors, notification taps, services):

```dart
sl<AppRouter>().router.goNamed(AppRoute.login.name);
```

Rules: never `Navigator.push(MaterialPageRoute(...))` for app screens. Never
hardcode path strings (`context.go('/home')`). Always go through `AppRoute`.

## 5. Adding a route (recipe)

1. Add the enum entry to `AppRoute` (path, `isPublic` if needed).
2. Add a `_route(...)` in `AppRouter` in the right place (shell branch vs root).
3. Navigate with `context.pushTo/goTo`.
4. If it's deep-linkable, confirm the page loads from path params alone.

## 6. Deep links (when requested)

- Android: add an `<intent-filter android:autoVerify="true">` with `https` scheme + host to
  the main activity, and host `/.well-known/assetlinks.json`.
- iOS: Associated Domains `applinks:<host>` + `apple-app-site-association` on the server.
- go_router handles the incoming path automatically. The guard's `from` param restores
  the target after login.

---

## Enums conventions

Location: cross-feature enums → `lib/core/enums/`. Feature-only → `features/<f>/domain/enums/`.
One enum per file, file name = snake_case of the enum.

### Pattern: enum backed by API value, with a safe fallback

```dart
enum PaymentStatus {
  pending('PENDING'),
  paid('PAID'),
  failed('FAILED'),
  refunded('REFUNDED'),
  unknown('');

  const PaymentStatus(this.value);

  /// Wire value sent/received from the API. Don't use `.name` for serialization —
  /// renaming a case would silently break the contract.
  final String value;

  static PaymentStatus fromValue(String? raw) => values.firstWhere(
        (e) => e.value == raw?.toUpperCase(),
        orElse: () => PaymentStatus.unknown,
      );

  bool get isFinal => this == paid || this == failed || this == refunded;
}
```

Int-coded API? Same shape with `final int code;` and `fromCode(int?)`.

In models: `status: PaymentStatus.fromValue(json['status'] as String?)`, and `toJson` →
`'status': status.value`.

### UI mapping: keep Flutter out of the enum, put it in an extension

The enum stays pure Dart (usable in domain). UI concerns live in a presentation-side extension:

```dart
import 'package:app/core/theme/theme_ext.dart';
import 'package:flutter/material.dart';

extension PaymentStatusUi on PaymentStatus {
  String label(BuildContext context) => switch (this) {
        PaymentStatus.pending => 'Pending', // use localized strings if the app is localized
        PaymentStatus.paid => 'Paid',
        PaymentStatus.failed => 'Failed',
        PaymentStatus.refunded => 'Refunded',
        PaymentStatus.unknown => '—',
      };

  Color color(BuildContext context) => switch (this) {
        PaymentStatus.pending => context.colors.warning,
        PaymentStatus.paid => context.colors.success,
        PaymentStatus.failed => context.colors.error,
        PaymentStatus.refunded => context.colors.info,
        PaymentStatus.unknown => context.colors.textHint,
      };

  IconData get icon => switch (this) {
        PaymentStatus.pending => Icons.schedule,
        PaymentStatus.paid => Icons.check_circle,
        PaymentStatus.failed => Icons.error,
        PaymentStatus.refunded => Icons.undo,
        PaymentStatus.unknown => Icons.help_outline,
      };
}
```

### Enum rules

- Use **exhaustive `switch` expressions with no `default`/`_`**, so adding a case makes the
  compiler flag every place that needs updating.
- Always include an `unknown` (or sensible default) case for server-driven enums so a
  new backend value doesn't crash parsing.
- Never compare raw strings (`if (status == 'PAID')`). Parse to the enum at the model boundary.
- Enhanced-enum fields for static metadata (value, code, path, icon asset).
  Extensions for context-dependent things (localized label, theme color).
- Use enums for component variants too (`AppButtonVariant`, `SnackType`). See `flutter-components`.
- `ViewStatus` (initial/loading/success/failure) in `core/enums/` is the standard async UI
  state for cubits (see `flutter-init`).
