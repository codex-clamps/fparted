# Repository Guidelines

## Project Structure & Module Organization

- `lib/main.dart` boots the Flutter application.
- `lib/core/` contains disk models, filesystem logic, cached state, and privileged command runners for tools such as `parted`, `blkid`, and `dd`.
- `lib/layouts/` contains screens, dialogs, and reusable UI components. Keep business logic out of widgets when it belongs in `core/`.
- `android/`, `linux/`, and `web/` contain platform runners and resources. Runtime privilege handling currently targets Android and Linux.
- Tests live in `test/`; mirror the relevant `lib/` feature when adding coverage.

## Package Manager

- Use Flutter's Pub workflow. Run `flutter pub get` after changing `pubspec.yaml`, and commit intentional `pubspec.lock` updates.

## Build, Test, and Development Commands

| Command | Purpose |
| --- | --- |
| `flutter run -d <device>` | Run on an Android device or Linux desktop target. |
| `flutter build apk` | Produce an Android APK. |
| `flutter build linux` | Produce the Linux desktop bundle. |
| `flutter analyze` | Apply `flutter_lints` static checks. |
| `dart format --output=none --set-exit-if-changed lib test` | Check Dart formatting without rewriting files. |
| `flutter test` | Run the full test suite. |

For focused checks, use `dart analyze path/to/file.dart` and `flutter test test/<feature>_test.dart`.

## Coding Style & Naming Conventions

- Accept `dart format` output: two-space indentation, trailing commas where the formatter adds them, and no manual alignment.
- Follow `analysis_options.yaml`; do not suppress lints without a narrow, documented reason.
- Use `lower_snake_case.dart` for files, `UpperCamelCase` for types/widgets, `lowerCamelCase` for members, and `_leadingUnderscore` for private declarations.

## Testing Guidelines

- Use `flutter_test`; name files `*_test.dart` and tests by observable behavior.
- The existing generated counter test is stale relative to the disk-management UI; replace it when touching app startup.
- No coverage threshold is configured. Use `flutter test --coverage` when evaluating substantial logic changes.
- Mock command execution and parse fixtures instead of invoking privileged tools or real block devices.

## Security & Configuration

- Partition operations can destroy data. Use emulators, loop devices, or disposable media for manual validation, and state the target used in the PR.
- Android startup expects `android/app/src/main/assets/prebuilts.zip`; verify bundled binaries and executable permissions before release builds.

## Commit & Pull Request Guidelines

- History favors short, focused, mostly lowercase subjects such as `add job label` and `backend data refactor`; Conventional Commit prefixes are not required.
- PRs should explain behavior and risk, list checks run, link relevant issues, and include screenshots for UI changes. Call out changes to command construction, privilege handling, or bundled tools.

