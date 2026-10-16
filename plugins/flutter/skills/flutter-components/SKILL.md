---
name: flutter-components
description: Create and use a shared Flutter component library in lib/core/components — AppButton (variants/sizes/loading), AppTextField (+ validators), AppImage (network/asset/svg), AppScaffold, AppCard, AppStatusView (loading/error/empty/success), AppBottomSheet, AppDialog, AppSnackBar — with a barrel export and rules for when to reuse vs create. Use when the user wants to build common/reusable widgets, a design-system component, a new shared component, or asks how to use the existing components on a screen.
---

# Flutter Components — Shared UI Library

Part of the `flutter-*` skill set. Depends on `flutter-theme` (`context.colors`,
`context.textTheme`, `AppSpacing`, `AppRadius`, `AppGap`) and `ViewStatus` from
`flutter-init`. Replace `app` in `package:app/...` with the package name.

```bash
flutter pub add flutter_svg cached_network_image
```

## Before creating anything

1. **Check `lib/core/components/` first.** If a component exists, use it or extend it with
   a new optional parameter or variant. Don't make a near-duplicate (`AppButton2`,
   `CustomButtonNew`).
2. Used by **one feature only** → `features/<f>/presentation/widgets/`.
   Used by **2+ features** → promote to `core/components/`.
3. If the project already has components with different names, keep them and match their API style.

## Component rules

- Stateless where possible, `const` constructors, `super.key`.
- **Dumb:** no `sl<>()`, no cubits, no API calls, no navigation inside. Data in, callbacks out.
- **Themed:** colors from `context.colors`, text from `context.textTheme`, sizes from tokens.
  No hex, no magic numbers.
- **Strings come from the caller** (already localized). No hardcoded user-facing text
  except optional defaults passed as parameters.
- Variants via **enums** (`AppButtonVariant.outline`), not boolean soup (`isOutline`, `isRed`).
- Accessible: tap targets ≥ 48dp, `Semantics`/`tooltip` for icon-only controls, respect
  text scaling (no fixed heights on text containers), RTL-safe (`EdgeInsetsDirectional`,
  `AlignmentDirectional`).
- One component per file, `app_` prefix, exported from `components.dart`.

## Structure

```
lib/core/components/
├── components.dart          # barrel: export everything below
├── buttons/app_button.dart
├── inputs/app_text_field.dart
├── media/app_image.dart
├── layout/app_scaffold.dart
├── layout/app_card.dart
├── feedback/app_status_view.dart
├── overlays/app_bottom_sheet.dart
├── overlays/app_dialog.dart
└── overlays/app_snack_bar.dart
lib/core/utils/validators/validators.dart
```

Screens import one line: `import 'package:app/core/components/components.dart';`

---

## buttons/app_button.dart

```dart
import 'package:app/core/theme/app_spacing.dart';
import 'package:app/core/theme/theme_ext.dart';
import 'package:flutter/material.dart';

enum AppButtonVariant { primary, secondary, outline, text, danger }

enum AppButtonSize {
  small(36, 14, AppSpacing.md),
  medium(48, 16, AppSpacing.lg),
  large(56, 16, AppSpacing.xl);

  const AppButtonSize(this.height, this.fontSize, this.hPadding);
  final double height;
  final double fontSize;
  final double hPadding;
}

class AppButton extends StatelessWidget {
  const AppButton({
    super.key,
    required this.label,
    required this.onPressed,
    this.variant = AppButtonVariant.primary,
    this.size = AppButtonSize.medium,
    this.icon,
    this.isLoading = false,
    this.expanded = true,
  });

  final String label;
  final VoidCallback? onPressed; // null = disabled
  final AppButtonVariant variant;
  final AppButtonSize size;
  final IconData? icon;
  final bool isLoading;
  final bool expanded;

  @override
  Widget build(BuildContext context) {
    final c = context.colors;
    final (bg, fg, side) = switch (variant) {
      AppButtonVariant.primary => (c.primary, c.onPrimary, BorderSide.none),
      AppButtonVariant.secondary => (c.primarySoft, c.primary, BorderSide.none),
      AppButtonVariant.outline => (Colors.transparent, c.primary, BorderSide(color: c.primary)),
      AppButtonVariant.text => (Colors.transparent, c.primary, BorderSide.none),
      AppButtonVariant.danger => (c.error, c.onPrimary, BorderSide.none),
    };
    final transparent = bg == Colors.transparent;

    final button = TextButton(
      onPressed: isLoading ? null : onPressed,
      style: TextButton.styleFrom(
        backgroundColor: bg,
        foregroundColor: fg,
        disabledBackgroundColor: transparent ? bg : bg.withValues(alpha: 0.45),
        disabledForegroundColor: fg.withValues(alpha: 0.6),
        minimumSize: Size(size.height, size.height),
        padding: EdgeInsets.symmetric(horizontal: size.hPadding),
        shape: RoundedRectangleBorder(borderRadius: AppRadius.mdAll, side: side),
        textStyle: context.textTheme.labelLarge?.copyWith(fontSize: size.fontSize),
      ),
      child: isLoading
          ? SizedBox.square(
              dimension: 20,
              child: CircularProgressIndicator(strokeWidth: 2, color: fg),
            )
          : Row(
              mainAxisSize: MainAxisSize.min,
              children: [
                if (icon != null) ...[Icon(icon, size: 20), AppGap.h8],
                Flexible(child: Text(label, maxLines: 1, overflow: TextOverflow.ellipsis)),
              ],
            ),
    );

    return expanded ? SizedBox(width: double.infinity, child: button) : button;
  }
}
```

## inputs/app_text_field.dart

```dart
import 'package:app/core/theme/app_spacing.dart';
import 'package:app/core/theme/theme_ext.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

class AppTextField extends StatefulWidget {
  const AppTextField({
    super.key,
    this.controller,
    this.label,
    this.hint,
    this.validator,
    this.onChanged,
    this.onSubmitted,
    this.onTap,
    this.keyboardType,
    this.textInputAction = TextInputAction.next,
    this.inputFormatters,
    this.prefixIcon,
    this.suffix,
    this.isPassword = false,
    this.readOnly = false,
    this.enabled = true,
    this.maxLines = 1,
    this.maxLength,
    this.autofillHints,
  });

  final TextEditingController? controller;
  final String? label;
  final String? hint;
  final FormFieldValidator<String>? validator;
  final ValueChanged<String>? onChanged;
  final ValueChanged<String>? onSubmitted;
  final VoidCallback? onTap; // with readOnly: true → date/country pickers
  final TextInputType? keyboardType;
  final TextInputAction textInputAction;
  final List<TextInputFormatter>? inputFormatters;
  final IconData? prefixIcon;
  final Widget? suffix;
  final bool isPassword;
  final bool readOnly;
  final bool enabled;
  final int maxLines;
  final int? maxLength;
  final Iterable<String>? autofillHints;

  @override
  State<AppTextField> createState() => _AppTextFieldState();
}

class _AppTextFieldState extends State<AppTextField> {
  late bool _obscure = widget.isPassword;

  @override
  Widget build(BuildContext context) {
    final c = context.colors;
    return Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      mainAxisSize: MainAxisSize.min,
      children: [
        if (widget.label != null) ...[
          Text(widget.label!, style: context.textTheme.labelMedium),
          AppGap.v8,
        ],
        TextFormField(
          controller: widget.controller,
          validator: widget.validator,
          onChanged: widget.onChanged,
          onFieldSubmitted: widget.onSubmitted,
          onTap: widget.onTap,
          keyboardType: widget.keyboardType,
          textInputAction: widget.textInputAction,
          inputFormatters: widget.inputFormatters,
          obscureText: _obscure,
          readOnly: widget.readOnly,
          enabled: widget.enabled,
          maxLines: widget.isPassword ? 1 : widget.maxLines,
          maxLength: widget.maxLength,
          autofillHints: widget.autofillHints,
          autovalidateMode: AutovalidateMode.onUserInteraction,
          style: context.textTheme.bodyLarge,
          decoration: InputDecoration(
            hintText: widget.hint,
            counterText: '',
            prefixIcon: widget.prefixIcon == null
                ? null
                : Icon(widget.prefixIcon, color: c.textSecondary, size: 20),
            suffixIcon: widget.isPassword
                ? IconButton(
                    tooltip: _obscure ? 'Show password' : 'Hide password',
                    icon: Icon(_obscure ? Icons.visibility_off : Icons.visibility,
                        color: c.textSecondary, size: 20),
                    onPressed: () => setState(() => _obscure = !_obscure),
                  )
                : widget.suffix,
          ),
        ),
      ],
    );
  }
}
```

## utils/validators/validators.dart

```dart
typedef Validator = String? Function(String? value);

abstract final class Validators {
  static Validator required([String message = 'This field is required']) =>
      (v) => (v == null || v.trim().isEmpty) ? message : null;

  static Validator email([String message = 'Enter a valid email']) => (v) {
        if (v == null || v.isEmpty) return null; // combine with required() if mandatory
        return RegExp(r'^[\w.+-]+@[\w-]+\.[\w.-]+$').hasMatch(v.trim()) ? null : message;
      };

  static Validator minLength(int min, [String? message]) =>
      (v) => (v != null && v.isNotEmpty && v.length < min)
          ? (message ?? 'Must be at least $min characters')
          : null;

  static Validator match(String Function() other, [String message = 'Values do not match']) =>
      (v) => v == other() ? null : message;

  /// Runs validators in order, returns the first error.
  static Validator compose(List<Validator> validators) => (v) {
        for (final validate in validators) {
          final error = validate(v);
          if (error != null) return error;
        }
        return null;
      };
}
```

## media/app_image.dart

```dart
import 'package:app/core/theme/app_spacing.dart';
import 'package:app/core/theme/theme_ext.dart';
import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter/material.dart';
import 'package:flutter_svg/flutter_svg.dart';

/// One widget for network URLs, asset PNG/JPG and SVG (asset or network).
class AppImage extends StatelessWidget {
  const AppImage(
    this.source, {
    super.key,
    this.width,
    this.height,
    this.fit = BoxFit.cover,
    this.borderRadius,
    this.circle = false,
    this.color,
    this.semanticLabel,
  });

  final String? source;
  final double? width;
  final double? height;
  final BoxFit fit;
  final BorderRadius? borderRadius;
  final bool circle;
  final Color? color; // tint, mostly for SVG icons
  final String? semanticLabel;

  bool get _isNetwork => source?.startsWith('http') ?? false;
  bool get _isSvg => source?.toLowerCase().split('?').first.endsWith('.svg') ?? false;

  @override
  Widget build(BuildContext context) {
    final src = source;
    final Widget image;

    if (src == null || src.isEmpty) {
      image = _placeholder(context, icon: Icons.image_not_supported_outlined);
    } else if (_isSvg) {
      final filter = color == null ? null : ColorFilter.mode(color!, BlendMode.srcIn);
      image = _isNetwork
          ? SvgPicture.network(src, width: width, height: height, fit: fit, colorFilter: filter)
          : SvgPicture.asset(src, width: width, height: height, fit: fit, colorFilter: filter);
    } else if (_isNetwork) {
      image = CachedNetworkImage(
        imageUrl: src,
        width: width,
        height: height,
        fit: fit,
        color: color,
        placeholder: (context, _) => _placeholder(context),
        errorWidget: (context, _, __) => _placeholder(context, icon: Icons.broken_image_outlined),
      );
    } else {
      image = Image.asset(
        src,
        width: width,
        height: height,
        fit: fit,
        color: color,
        errorBuilder: (context, _, __) => _placeholder(context, icon: Icons.broken_image_outlined),
      );
    }

    final clipped = circle
        ? ClipOval(child: image)
        : borderRadius != null
            ? ClipRRect(borderRadius: borderRadius!, child: image)
            : image;

    return semanticLabel == null
        ? clipped
        : Semantics(label: semanticLabel, image: true, child: clipped);
  }

  Widget _placeholder(BuildContext context, {IconData? icon}) => Container(
        width: width,
        height: height,
        color: context.colors.surfaceAlt,
        alignment: Alignment.center,
        child: icon == null ? null : Icon(icon, color: context.colors.textHint),
      );
}
```

## layout/app_scaffold.dart

```dart
import 'package:app/core/theme/app_spacing.dart';
import 'package:app/core/theme/theme_ext.dart';
import 'package:flutter/material.dart';

class AppScaffold extends StatelessWidget {
  const AppScaffold({
    super.key,
    required this.body,
    this.title,
    this.actions,
    this.leading,
    this.bottom,
    this.floatingActionButton,
    this.padding = AppSpacing.screen,
    this.showAppBar = true,
    this.isLoading = false,
  });

  final Widget body;
  final String? title;
  final List<Widget>? actions;
  final Widget? leading;
  final Widget? bottom; // sticky bottom area, e.g. a submit AppButton
  final Widget? floatingActionButton;
  final EdgeInsetsGeometry padding;
  final bool showAppBar;
  final bool isLoading; // blocking overlay for submits

  @override
  Widget build(BuildContext context) {
    return Stack(
      children: [
        Scaffold(
          appBar: showAppBar
              ? AppBar(title: title == null ? null : Text(title!), actions: actions, leading: leading)
              : null,
          floatingActionButton: floatingActionButton,
          body: SafeArea(child: Padding(padding: padding, child: body)),
          bottomNavigationBar: bottom == null
              ? null
              : SafeArea(
                  minimum: const EdgeInsets.fromLTRB(
                      AppSpacing.lg, AppSpacing.sm, AppSpacing.lg, AppSpacing.lg),
                  child: bottom!,
                ),
        ),
        if (isLoading)
          Positioned.fill(
            child: ColoredBox(
              color: context.colors.shadow,
              child: const Center(child: CircularProgressIndicator()),
            ),
          ),
      ],
    );
  }
}
```

## layout/app_card.dart

```dart
import 'package:app/core/theme/app_spacing.dart';
import 'package:app/core/theme/theme_ext.dart';
import 'package:flutter/material.dart';

class AppCard extends StatelessWidget {
  const AppCard({super.key, required this.child, this.onTap, this.padding = AppSpacing.card, this.color});

  final Widget child;
  final VoidCallback? onTap;
  final EdgeInsetsGeometry padding;
  final Color? color;

  @override
  Widget build(BuildContext context) {
    return Material(
      color: color ?? context.colors.surface,
      shape: RoundedRectangleBorder(
        borderRadius: AppRadius.lgAll,
        side: BorderSide(color: context.colors.border),
      ),
      clipBehavior: Clip.antiAlias,
      child: InkWell(onTap: onTap, child: Padding(padding: padding, child: child)),
    );
  }
}
```

## feedback/app_status_view.dart

A single widget for the four async states. Pair it with `ViewStatus` in every cubit-driven screen.

```dart
import 'package:app/core/components/buttons/app_button.dart';
import 'package:app/core/enums/view_status.dart';
import 'package:app/core/theme/app_spacing.dart';
import 'package:app/core/theme/theme_ext.dart';
import 'package:flutter/material.dart';

class AppStatusView extends StatelessWidget {
  const AppStatusView({
    super.key,
    required this.status,
    required this.onSuccess,
    this.errorMessage,
    this.onRetry,
    this.isEmpty = false,
    this.emptyTitle = 'Nothing here yet',
    this.emptyMessage,
    this.retryLabel = 'Try again',
    this.loading,
  });

  final ViewStatus status;
  final WidgetBuilder onSuccess;
  final String? errorMessage;
  final VoidCallback? onRetry;
  final bool isEmpty;
  final String emptyTitle;
  final String? emptyMessage;
  final String retryLabel;
  final Widget? loading; // e.g. a skeleton list

  @override
  Widget build(BuildContext context) {
    return AnimatedSwitcher(
      duration: AppDurations.normal,
      child: switch (status) {
        ViewStatus.initial || ViewStatus.loading =>
          loading ?? const Center(child: CircularProgressIndicator()),
        ViewStatus.failure => AppMessageView(
            icon: Icons.error_outline,
            iconColor: context.colors.error,
            title: errorMessage ?? 'Something went wrong',
            action: onRetry == null
                ? null
                : AppButton(label: retryLabel, onPressed: onRetry, expanded: false,
                    variant: AppButtonVariant.outline),
          ),
        ViewStatus.success when isEmpty => AppMessageView(
            icon: Icons.inbox_outlined,
            title: emptyTitle,
            message: emptyMessage,
          ),
        ViewStatus.success => Builder(builder: onSuccess),
      },
    );
  }
}

/// Centered icon + title + message + optional action. Used for empty/error/info states.
class AppMessageView extends StatelessWidget {
  const AppMessageView({super.key, required this.title, this.message, this.icon, this.iconColor, this.action});

  final String title;
  final String? message;
  final IconData? icon;
  final Color? iconColor;
  final Widget? action;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Padding(
        padding: const EdgeInsets.all(AppSpacing.xl),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            if (icon != null) Icon(icon, size: 56, color: iconColor ?? context.colors.textHint),
            AppGap.v16,
            Text(title, style: context.textTheme.titleMedium, textAlign: TextAlign.center),
            if (message != null) ...[
              AppGap.v8,
              Text(message!, style: context.textTheme.bodySmall, textAlign: TextAlign.center),
            ],
            if (action != null) ...[AppGap.v24, action!],
          ],
        ),
      ),
    );
  }
}
```

## overlays/app_bottom_sheet.dart

```dart
import 'package:app/core/theme/app_spacing.dart';
import 'package:app/core/theme/theme_ext.dart';
import 'package:flutter/material.dart';

abstract final class AppBottomSheet {
  static Future<T?> show<T>(
    BuildContext context, {
    required Widget child,
    String? title,
    bool isDismissible = true,
    bool scrollable = true,
  }) {
    return showModalBottomSheet<T>(
      context: context,
      isScrollControlled: true,
      useSafeArea: true,
      isDismissible: isDismissible,
      enableDrag: isDismissible,
      builder: (context) => Padding(
        padding: EdgeInsets.only(bottom: MediaQuery.viewInsetsOf(context).bottom),
        child: Padding(
          padding: const EdgeInsets.fromLTRB(AppSpacing.lg, 0, AppSpacing.lg, AppSpacing.lg),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              if (title != null) ...[
                Text(title, style: context.textTheme.titleLarge, textAlign: TextAlign.center),
                AppGap.v16,
              ],
              Flexible(child: scrollable ? SingleChildScrollView(child: child) : child),
            ],
          ),
        ),
      ),
    );
  }
}
```

## overlays/app_dialog.dart

```dart
import 'package:app/core/components/buttons/app_button.dart';
import 'package:app/core/theme/app_spacing.dart';
import 'package:flutter/material.dart';

abstract final class AppDialog {
  /// Returns true only if the user confirmed.
  static Future<bool> confirm(
    BuildContext context, {
    required String title,
    String? message,
    String confirmLabel = 'Confirm',
    String cancelLabel = 'Cancel',
    bool isDestructive = false,
  }) async {
    final result = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: Text(title),
        content: message == null ? null : Text(message),
        actionsPadding: const EdgeInsets.all(AppSpacing.lg),
        actions: [
          Row(
            children: [
              Expanded(
                child: AppButton(
                  label: cancelLabel,
                  variant: AppButtonVariant.outline,
                  onPressed: () => Navigator.of(context).pop(false),
                ),
              ),
              AppGap.h12,
              Expanded(
                child: AppButton(
                  label: confirmLabel,
                  variant: isDestructive ? AppButtonVariant.danger : AppButtonVariant.primary,
                  onPressed: () => Navigator.of(context).pop(true),
                ),
              ),
            ],
          ),
        ],
      ),
    );
    return result ?? false;
  }
}
```

(Dialogs and sheets use `Navigator.of(context).pop` — that's correct even with go_router.)

## overlays/app_snack_bar.dart

```dart
import 'package:app/core/theme/theme_ext.dart';
import 'package:flutter/material.dart';

enum SnackType { success, error, info, warning }

abstract final class AppSnackBar {
  static void show(BuildContext context, String message, {SnackType type = SnackType.info}) {
    final c = context.colors;
    final (bg, icon) = switch (type) {
      SnackType.success => (c.success, Icons.check_circle),
      SnackType.error => (c.error, Icons.error),
      SnackType.warning => (c.warning, Icons.warning_amber),
      SnackType.info => (c.info, Icons.info),
    };
    ScaffoldMessenger.of(context)
      ..hideCurrentSnackBar()
      ..showSnackBar(
        SnackBar(
          backgroundColor: bg,
          content: Row(
            children: [
              Icon(icon, color: Colors.white),
              const SizedBox(width: 12),
              Expanded(child: Text(message, style: const TextStyle(color: Colors.white))),
            ],
          ),
        ),
      );
  }
}
```

## components.dart (barrel)

```dart
export 'buttons/app_button.dart';
export 'feedback/app_status_view.dart';
export 'inputs/app_text_field.dart';
export 'layout/app_card.dart';
export 'layout/app_scaffold.dart';
export 'media/app_image.dart';
export 'overlays/app_bottom_sheet.dart';
export 'overlays/app_dialog.dart';
export 'overlays/app_snack_bar.dart';
```

Every new component gets a line here.

---

## Common usage

### Form screen

```dart
class _LoginView extends StatefulWidget { … }

class _LoginViewState extends State<_LoginView> {
  final _formKey = GlobalKey<FormState>();
  final _email = TextEditingController();
  final _password = TextEditingController();

  @override
  void dispose() {
    _email.dispose();
    _password.dispose();
    super.dispose();
  }

  void _submit() {
    if (!_formKey.currentState!.validate()) return;
    context.read<LoginCubit>().login(_email.text.trim(), _password.text);
  }

  @override
  Widget build(BuildContext context) {
    return BlocConsumer<LoginCubit, LoginState>(
      listenWhen: (p, c) => p.status != c.status,
      listener: (context, state) {
        if (state.status.isFailure) AppSnackBar.show(context, state.errorMessage!, type: SnackType.error);
        if (state.status.isSuccess) context.goTo(AppRoute.home);
      },
      builder: (context, state) => AppScaffold(
        title: 'Sign in',
        bottom: AppButton(label: 'Sign in', isLoading: state.status.isLoading, onPressed: _submit),
        body: Form(
          key: _formKey,
          child: AutofillGroup(
            child: ListView(
              children: [
                AppTextField(
                  controller: _email,
                  label: 'Email',
                  hint: 'name@example.com',
                  prefixIcon: Icons.mail_outline,
                  keyboardType: TextInputType.emailAddress,
                  autofillHints: const [AutofillHints.email],
                  validator: Validators.compose([Validators.required(), Validators.email()]),
                ),
                AppGap.v16,
                AppTextField(
                  controller: _password,
                  label: 'Password',
                  isPassword: true,
                  textInputAction: TextInputAction.done,
                  onSubmitted: (_) => _submit(),
                  validator: Validators.compose([Validators.required(), Validators.minLength(8)]),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}
```

### List screen with async states

```dart
BlocBuilder<ItemsCubit, ItemsState>(
  builder: (context, state) => AppStatusView(
    status: state.status,
    errorMessage: state.errorMessage,
    onRetry: context.read<ItemsCubit>().load,
    isEmpty: state.items.isEmpty,
    emptyTitle: 'No items yet',
    onSuccess: (_) => RefreshIndicator(
      onRefresh: context.read<ItemsCubit>().load,
      child: ListView.separated(
        itemCount: state.items.length,
        separatorBuilder: (_, __) => AppGap.v12,
        itemBuilder: (context, i) {
          final item = state.items[i];
          return AppCard(
            onTap: () => context.pushTo(AppRoute.productDetails, params: {'id': item.id}),
            child: Row(
              children: [
                AppImage(item.imageUrl, width: 56, height: 56, borderRadius: AppRadius.mdAll),
                AppGap.h12,
                Expanded(child: Text(item.name, style: context.textTheme.titleMedium)),
              ],
            ),
          );
        },
      ),
    ),
  ),
)
```

### Confirm a destructive action

```dart
final ok = await AppDialog.confirm(
  context,
  title: 'Delete item?',
  message: 'This cannot be undone.',
  confirmLabel: 'Delete',
  isDestructive: true,
);
if (ok && context.mounted) context.read<ItemsCubit>().delete(id);
```

### Bottom sheet returning a value

```dart
final sort = await AppBottomSheet.show<SortOption>(
  context,
  title: 'Sort by',
  child: Column(
    children: SortOption.values
        .map((o) => ListTile(title: Text(o.label), onTap: () => Navigator.pop(context, o)))
        .toList(),
  ),
);
```

### Buttons

```dart
AppButton(label: 'Continue', onPressed: next)                                   // primary, full width
AppButton(label: 'Skip', onPressed: skip, variant: AppButtonVariant.text, expanded: false)
AppButton(label: 'Add', icon: Icons.add, size: AppButtonSize.small, expanded: false, onPressed: add)
AppButton(label: 'Pay', onPressed: canPay ? pay : null, isLoading: paying)       // disabled when null
```

## Creating a new shared component (recipe)

1. Confirm it's needed in 2+ features and nothing existing can be extended.
2. Pick a folder (`buttons/ inputs/ media/ layout/ feedback/ overlays/`), name `app_<thing>.dart`.
3. Follow the rules above: const, themed, dumb, enum variants, accessible.
4. Export from `components.dart`.
5. Add a short usage example to this project's docs (or a widget test in
   `test/core/components/`) covering the default + one variant.
6. Run `flutter analyze`.
