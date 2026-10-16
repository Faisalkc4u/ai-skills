---
name: flutter-theme
description: Build a Flutter design-token theme system — brand color palette, semantic light/dark colors via ThemeExtension, custom fonts (including a second script/RTL font), a text style scale, spacing/radius/gap tokens, asset path constants, Material 3 ThemeData with component themes, and a persisted ThemeCubit. Use when the user wants to set up or change themes, colors, dark mode, fonts, typography, text styles, spacing or asset constants in a Flutter app.
---

# Flutter Theme — Colors, Fonts, Text Styles, Tokens

Part of the `flutter-*` skill set. Replace `app` in `package:app/...` with the package
name. Inspect `lib/core/theme/` first and migrate existing values rather than inventing new ones.

## Inputs to gather

Ask the user once, only for what's missing. Otherwise use the defaults.

| Input | Default |
|---|---|
| Primary / secondary brand colors (hex) | `#2563EB` / `#0EA5E9` |
| Font family (+ weights) | System font (no bundled font) |
| Second font for another script (e.g. Arabic) | none |
| Dark mode? | yes |
| Figma/design tokens? | if given, map them into the palette below |

## Architecture

```
lib/core/theme/
├── app_palette.dart       # raw brand colors — ONLY file with hex values
├── app_colors.dart        # semantic colors (ThemeExtension), light + dark
├── app_fonts.dart         # font family names
├── app_text_styles.dart   # TextTheme scale
├── app_spacing.dart       # spacing, radius, gaps, durations
├── app_assets.dart        # image/icon/animation paths
├── app_theme.dart         # ThemeData light/dark + component themes
├── theme_ext.dart         # context.colors / context.textTheme helpers
└── theme_cubit.dart       # ThemeMode, persisted
```

Rules:
- **Widgets never use `Color(0x…)` or `Colors.xxx`** (except `Colors.transparent`). They use
  `context.colors.<semantic>`.
- Colors are named by **role** (`textSecondary`, `surface`, `error`), not by look (`grey2`).
- Text uses `context.textTheme.<role>` then `.copyWith(...)` for one-off tweaks.
- Spacing uses `AppSpacing` / `AppGap`; radius uses `AppRadius`. No magic numbers.
- Theme is reactive through `Theme.of(context)`. Don't use static getters that read global
  state — they won't rebuild on theme change.

## 1. `app_palette.dart`

```dart
import 'package:flutter/material.dart';

/// Raw palette. Don't use directly in widgets — go through AppColors.
abstract final class AppPalette {
  static const Color primary = Color(0xFF2563EB);
  static const Color primaryDark = Color(0xFF60A5FA);
  static const Color secondary = Color(0xFF0EA5E9);

  static const Color white = Color(0xFFFFFFFF);
  static const Color black = Color(0xFF0B0F19);

  static const Color neutral50 = Color(0xFFF8FAFC);
  static const Color neutral100 = Color(0xFFF1F5F9);
  static const Color neutral200 = Color(0xFFE2E8F0);
  static const Color neutral400 = Color(0xFF94A3B8);
  static const Color neutral500 = Color(0xFF64748B);
  static const Color neutral700 = Color(0xFF334155);
  static const Color neutral800 = Color(0xFF1E293B);
  static const Color neutral900 = Color(0xFF0F172A);

  static const Color success = Color(0xFF16A34A);
  static const Color warning = Color(0xFFF59E0B);
  static const Color error = Color(0xFFDC2626);
  static const Color info = Color(0xFF0284C7);
}
```

## 2. `app_colors.dart` — semantic tokens as a ThemeExtension

```dart
import 'package:app/core/theme/app_palette.dart';
import 'package:flutter/material.dart';

@immutable
class AppColors extends ThemeExtension<AppColors> {
  const AppColors({
    required this.primary,
    required this.onPrimary,
    required this.primarySoft,
    required this.secondary,
    required this.background,
    required this.surface,
    required this.surfaceAlt,
    required this.textPrimary,
    required this.textSecondary,
    required this.textHint,
    required this.border,
    required this.divider,
    required this.success,
    required this.warning,
    required this.error,
    required this.info,
    required this.shadow,
  });

  final Color primary;
  final Color onPrimary;
  final Color primarySoft; // tinted backgrounds, chips, secondary buttons
  final Color secondary;
  final Color background; // scaffold
  final Color surface; // cards, sheets
  final Color surfaceAlt; // inputs, subtle containers
  final Color textPrimary;
  final Color textSecondary;
  final Color textHint;
  final Color border;
  final Color divider;
  final Color success;
  final Color warning;
  final Color error;
  final Color info;
  final Color shadow;

  static const light = AppColors(
    primary: AppPalette.primary,
    onPrimary: AppPalette.white,
    primarySoft: Color(0xFFE0EAFF),
    secondary: AppPalette.secondary,
    background: AppPalette.neutral50,
    surface: AppPalette.white,
    surfaceAlt: AppPalette.neutral100,
    textPrimary: AppPalette.neutral900,
    textSecondary: AppPalette.neutral500,
    textHint: AppPalette.neutral400,
    border: AppPalette.neutral200,
    divider: AppPalette.neutral200,
    success: AppPalette.success,
    warning: AppPalette.warning,
    error: AppPalette.error,
    info: AppPalette.info,
    shadow: Color(0x14000000),
  );

  static const dark = AppColors(
    primary: AppPalette.primaryDark,
    onPrimary: AppPalette.black,
    primarySoft: Color(0xFF1E2A44),
    secondary: AppPalette.secondary,
    background: AppPalette.black,
    surface: AppPalette.neutral900,
    surfaceAlt: AppPalette.neutral800,
    textPrimary: AppPalette.neutral50,
    textSecondary: AppPalette.neutral400,
    textHint: AppPalette.neutral500,
    border: AppPalette.neutral700,
    divider: AppPalette.neutral800,
    success: Color(0xFF22C55E),
    warning: Color(0xFFFBBF24),
    error: Color(0xFFF87171),
    info: Color(0xFF38BDF8),
    shadow: Color(0x40000000),
  );

  @override
  AppColors copyWith({
    Color? primary,
    Color? onPrimary,
    Color? primarySoft,
    Color? secondary,
    Color? background,
    Color? surface,
    Color? surfaceAlt,
    Color? textPrimary,
    Color? textSecondary,
    Color? textHint,
    Color? border,
    Color? divider,
    Color? success,
    Color? warning,
    Color? error,
    Color? info,
    Color? shadow,
  }) =>
      AppColors(
        primary: primary ?? this.primary,
        onPrimary: onPrimary ?? this.onPrimary,
        primarySoft: primarySoft ?? this.primarySoft,
        secondary: secondary ?? this.secondary,
        background: background ?? this.background,
        surface: surface ?? this.surface,
        surfaceAlt: surfaceAlt ?? this.surfaceAlt,
        textPrimary: textPrimary ?? this.textPrimary,
        textSecondary: textSecondary ?? this.textSecondary,
        textHint: textHint ?? this.textHint,
        border: border ?? this.border,
        divider: divider ?? this.divider,
        success: success ?? this.success,
        warning: warning ?? this.warning,
        error: error ?? this.error,
        info: info ?? this.info,
        shadow: shadow ?? this.shadow,
      );

  @override
  AppColors lerp(ThemeExtension<AppColors>? other, double t) {
    if (other is! AppColors) return this;
    Color l(Color a, Color b) => Color.lerp(a, b, t)!;
    return AppColors(
      primary: l(primary, other.primary),
      onPrimary: l(onPrimary, other.onPrimary),
      primarySoft: l(primarySoft, other.primarySoft),
      secondary: l(secondary, other.secondary),
      background: l(background, other.background),
      surface: l(surface, other.surface),
      surfaceAlt: l(surfaceAlt, other.surfaceAlt),
      textPrimary: l(textPrimary, other.textPrimary),
      textSecondary: l(textSecondary, other.textSecondary),
      textHint: l(textHint, other.textHint),
      border: l(border, other.border),
      divider: l(divider, other.divider),
      success: l(success, other.success),
      warning: l(warning, other.warning),
      error: l(error, other.error),
      info: l(info, other.info),
      shadow: l(shadow, other.shadow),
    );
  }
}
```

**Adding a token:** add the field, constructor param, `light` + `dark` values, `copyWith`,
and `lerp` entry. All six, or it won't compile or animate correctly.

**Multiple brand themes** (e.g. user-selectable accent): create extra `AppColors`
instances (`AppColors.light.copyWith(primary: …)`) and select one in `AppTheme`.

## 3. Fonts

1. Put `.ttf`/`.otf` files in `assets/fonts/<Family>/`.
2. Register in `pubspec.yaml` (weights must match the actual files):

```yaml
flutter:
  fonts:
    - family: Inter
      fonts:
        - asset: assets/fonts/Inter/Inter-Regular.ttf
          weight: 400
        - asset: assets/fonts/Inter/Inter-Medium.ttf
          weight: 500
        - asset: assets/fonts/Inter/Inter-SemiBold.ttf
          weight: 600
        - asset: assets/fonts/Inter/Inter-Bold.ttf
          weight: 700
```

3. `app_fonts.dart`:

```dart
import 'package:flutter/widgets.dart';

abstract final class AppFonts {
  static const String primary = 'Inter';
  static const String? secondaryScript = null; // e.g. 'NotoSansArabic' for an RTL locale

  /// Font family for a locale — lets one app use different fonts per script.
  static String familyFor(Locale? locale) {
    if (secondaryScript != null && locale?.languageCode == 'ar') return secondaryScript!;
    return primary;
  }
}
```

If the user has no font files, you can use the `google_fonts` package instead. Bundling
the files is preferred for production (works offline, no runtime fetch).

## 4. `app_text_styles.dart` — one type scale mapped to Material roles

```dart
import 'package:app/core/theme/app_colors.dart';
import 'package:flutter/material.dart';

abstract final class AppTextStyles {
  static TextTheme textTheme(AppColors c, String fontFamily) {
    TextStyle s(double size, FontWeight weight, double height, Color color) => TextStyle(
          fontFamily: fontFamily,
          fontSize: size,
          fontWeight: weight,
          height: height,
          color: color,
        );

    return TextTheme(
      displayLarge: s(32, FontWeight.w700, 1.25, c.textPrimary),
      displayMedium: s(28, FontWeight.w700, 1.28, c.textPrimary),
      displaySmall: s(24, FontWeight.w700, 1.33, c.textPrimary),
      headlineLarge: s(22, FontWeight.w600, 1.36, c.textPrimary),
      headlineMedium: s(20, FontWeight.w600, 1.4, c.textPrimary),
      headlineSmall: s(18, FontWeight.w600, 1.44, c.textPrimary),
      titleLarge: s(18, FontWeight.w600, 1.44, c.textPrimary), // app bar, section titles
      titleMedium: s(16, FontWeight.w600, 1.5, c.textPrimary), // card titles
      titleSmall: s(14, FontWeight.w600, 1.43, c.textPrimary),
      bodyLarge: s(16, FontWeight.w400, 1.5, c.textPrimary),
      bodyMedium: s(14, FontWeight.w400, 1.43, c.textPrimary), // default text
      bodySmall: s(12, FontWeight.w400, 1.33, c.textSecondary),
      labelLarge: s(16, FontWeight.w600, 1.25, c.textPrimary), // buttons
      labelMedium: s(14, FontWeight.w500, 1.29, c.textSecondary), // field labels
      labelSmall: s(11, FontWeight.w500, 1.27, c.textSecondary), // captions, badges
    );
  }
}
```

Usage cheat-sheet (put this in the project README/CLAUDE.md too):

| Need | Style |
|---|---|
| Screen title / app bar | `titleLarge` |
| Section header | `headlineSmall` |
| Card title | `titleMedium` |
| Body text | `bodyMedium` (`bodyLarge` for long reading) |
| Secondary / meta text | `bodySmall` |
| Button | `labelLarge` |
| Form label | `labelMedium` |
| Caption / badge | `labelSmall` |

## 5. `app_spacing.dart`

```dart
import 'package:flutter/widgets.dart';

abstract final class AppSpacing {
  static const double xxs = 2;
  static const double xs = 4;
  static const double sm = 8;
  static const double md = 12;
  static const double lg = 16;
  static const double xl = 24;
  static const double xxl = 32;
  static const double xxxl = 48;

  static const EdgeInsets screen = EdgeInsets.symmetric(horizontal: lg, vertical: lg);
  static const EdgeInsets card = EdgeInsets.all(lg);
}

abstract final class AppRadius {
  static const double sm = 6;
  static const double md = 10;
  static const double lg = 16;
  static const double xl = 24;
  static const double pill = 999;

  static final BorderRadius smAll = BorderRadius.circular(sm);
  static final BorderRadius mdAll = BorderRadius.circular(md);
  static final BorderRadius lgAll = BorderRadius.circular(lg);
}

abstract final class AppGap {
  static const SizedBox v4 = SizedBox(height: 4);
  static const SizedBox v8 = SizedBox(height: 8);
  static const SizedBox v12 = SizedBox(height: 12);
  static const SizedBox v16 = SizedBox(height: 16);
  static const SizedBox v24 = SizedBox(height: 24);
  static const SizedBox v32 = SizedBox(height: 32);
  static const SizedBox h4 = SizedBox(width: 4);
  static const SizedBox h8 = SizedBox(width: 8);
  static const SizedBox h12 = SizedBox(width: 12);
  static const SizedBox h16 = SizedBox(width: 16);
  static const SizedBox h24 = SizedBox(width: 24);
}

abstract final class AppDurations {
  static const Duration fast = Duration(milliseconds: 150);
  static const Duration normal = Duration(milliseconds: 250);
  static const Duration slow = Duration(milliseconds: 400);
}
```

## 6. `app_assets.dart`

```dart
abstract final class AppImages {
  static const String _p = 'assets/images';
  static const String logo = '$_p/logo.png';
  static const String emptyState = '$_p/empty_state.png';
}

abstract final class AppIcons {
  static const String _p = 'assets/icons';
  static const String home = '$_p/home.svg';
  static const String search = '$_p/search.svg';
}

abstract final class AppAnimations {
  static const String _p = 'assets/animations';
  static const String loading = '$_p/loading.json';
}
```

When the user adds asset files, add a constant here. Never write asset path strings in
widgets. Make sure each folder is listed under `flutter: assets:` in pubspec.

## 7. `app_theme.dart`

```dart
import 'package:app/core/theme/app_colors.dart';
import 'package:app/core/theme/app_fonts.dart';
import 'package:app/core/theme/app_spacing.dart';
import 'package:app/core/theme/app_text_styles.dart';
import 'package:flutter/material.dart';

abstract final class AppTheme {
  static ThemeData get light => _build(AppColors.light, Brightness.light);
  static ThemeData get dark => _build(AppColors.dark, Brightness.dark);

  static ThemeData _build(AppColors c, Brightness brightness, {String? fontFamily}) {
    final family = fontFamily ?? AppFonts.primary;
    final text = AppTextStyles.textTheme(c, family);
    final scheme = ColorScheme.fromSeed(
      seedColor: c.primary,
      brightness: brightness,
    ).copyWith(
      primary: c.primary,
      onPrimary: c.onPrimary,
      secondary: c.secondary,
      surface: c.surface,
      onSurface: c.textPrimary,
      error: c.error,
      outline: c.border,
    );

    final inputBorder = OutlineInputBorder(
      borderRadius: AppRadius.mdAll,
      borderSide: BorderSide(color: c.border),
    );

    return ThemeData(
      useMaterial3: true,
      brightness: brightness,
      colorScheme: scheme,
      fontFamily: family,
      textTheme: text,
      scaffoldBackgroundColor: c.background,
      extensions: [c],
      dividerTheme: DividerThemeData(color: c.divider, thickness: 1, space: 1),
      appBarTheme: AppBarTheme(
        backgroundColor: c.background,
        foregroundColor: c.textPrimary,
        surfaceTintColor: Colors.transparent,
        elevation: 0,
        centerTitle: true,
        titleTextStyle: text.titleLarge,
      ),
      cardTheme: CardThemeData(
        color: c.surface,
        elevation: 0,
        margin: EdgeInsets.zero,
        shape: RoundedRectangleBorder(
          borderRadius: AppRadius.lgAll,
          side: BorderSide(color: c.border),
        ),
      ),
      inputDecorationTheme: InputDecorationTheme(
        filled: true,
        fillColor: c.surfaceAlt,
        hintStyle: text.bodyMedium?.copyWith(color: c.textHint),
        contentPadding: const EdgeInsets.symmetric(
          horizontal: AppSpacing.lg,
          vertical: AppSpacing.md,
        ),
        border: inputBorder,
        enabledBorder: inputBorder,
        focusedBorder: inputBorder.copyWith(borderSide: BorderSide(color: c.primary, width: 1.5)),
        errorBorder: inputBorder.copyWith(borderSide: BorderSide(color: c.error)),
        focusedErrorBorder: inputBorder.copyWith(borderSide: BorderSide(color: c.error, width: 1.5)),
      ),
      filledButtonTheme: FilledButtonThemeData(
        style: FilledButton.styleFrom(
          minimumSize: const Size.fromHeight(48),
          shape: RoundedRectangleBorder(borderRadius: AppRadius.mdAll),
          textStyle: text.labelLarge,
        ),
      ),
      bottomSheetTheme: BottomSheetThemeData(
        backgroundColor: c.surface,
        surfaceTintColor: Colors.transparent,
        showDragHandle: true,
        shape: const RoundedRectangleBorder(
          borderRadius: BorderRadius.vertical(top: Radius.circular(AppRadius.xl)),
        ),
      ),
      dialogTheme: DialogThemeData(
        backgroundColor: c.surface,
        surfaceTintColor: Colors.transparent,
        shape: RoundedRectangleBorder(borderRadius: AppRadius.lgAll),
      ),
      snackBarTheme: SnackBarThemeData(
        behavior: SnackBarBehavior.floating,
        shape: RoundedRectangleBorder(borderRadius: AppRadius.mdAll),
      ),
    );
  }

  /// Use when the font depends on the locale.
  static ThemeData forLocale(Locale locale, {required bool dark}) => _build(
        dark ? AppColors.dark : AppColors.light,
        dark ? Brightness.dark : Brightness.light,
        fontFamily: AppFonts.familyFor(locale),
      );
}
```

Flutter < 3.27: use `CardTheme`/`DialogTheme` instead of `CardThemeData`/`DialogThemeData`.

## 8. `theme_ext.dart`

```dart
import 'package:app/core/theme/app_colors.dart';
import 'package:flutter/material.dart';

extension ThemeContextX on BuildContext {
  ThemeData get theme => Theme.of(this);
  AppColors get colors => Theme.of(this).extension<AppColors>()!;
  TextTheme get textTheme => Theme.of(this).textTheme;
  bool get isDark => Theme.of(this).brightness == Brightness.dark;
}

extension TextStyleX on TextStyle {
  TextStyle get w400 => copyWith(fontWeight: FontWeight.w400);
  TextStyle get w500 => copyWith(fontWeight: FontWeight.w500);
  TextStyle get w600 => copyWith(fontWeight: FontWeight.w600);
  TextStyle get w700 => copyWith(fontWeight: FontWeight.w700);
  TextStyle withColor(Color color) => copyWith(color: color);
}
```

## 9. `theme_cubit.dart` — persisted ThemeMode

```dart
import 'package:app/core/constants/storage_keys.dart';
import 'package:app/core/local/local_storage.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';

class ThemeCubit extends Cubit<ThemeMode> {
  ThemeCubit(this._storage) : super(_read(_storage));
  final LocalStorage _storage;

  static ThemeMode _read(LocalStorage s) => ThemeMode.values.firstWhere(
        (m) => m.name == s.getString(StorageKeys.themeMode),
        orElse: () => ThemeMode.system,
      );

  Future<void> setMode(ThemeMode mode) async {
    emit(mode);
    await _storage.setString(StorageKeys.themeMode, mode.name);
  }

  Future<void> toggle(Brightness current) =>
      setMode(current == Brightness.dark ? ThemeMode.light : ThemeMode.dark);
}
```

Register: `sl.registerLazySingleton(() => ThemeCubit(sl()));` — `App` wires it in (see `flutter-init`).

## Usage examples

```dart
Container(
  padding: AppSpacing.card,
  decoration: BoxDecoration(
    color: context.colors.surface,
    borderRadius: AppRadius.lgAll,
    border: Border.all(color: context.colors.border),
  ),
  child: Column(
    crossAxisAlignment: CrossAxisAlignment.start,
    children: [
      Text('Title', style: context.textTheme.titleMedium),
      AppGap.v4,
      Text('Subtitle', style: context.textTheme.bodySmall),
      Text('Paid', style: context.textTheme.labelSmall!.withColor(context.colors.success)),
    ],
  ),
);
```

## Checklist

- [ ] hex values only in `app_palette.dart`
- [ ] every semantic color has light + dark values
- [ ] fonts registered in pubspec with correct weights; `flutter pub get` run
- [ ] `grep -rn "Color(0x" lib --include=*.dart | grep -v core/theme` returns nothing new
- [ ] tested in light and dark mode, and at a larger OS text size
- [ ] `flutter analyze` clean
