# FPARTED Modernization and Completion Plan

Status: proposed

Planning baseline: 2026-07-18

Application repository: `codex-clamps/fparted`

Proposed toolchain repository ("Repo A"): `codex-clamps/fparted-toolchain`

## 1. Objective

Deliver a release-ready rooted Android partition editor by:

1. upgrading the application to the latest pinned Flutter stable SDK;
2. building a reproducible Android/Bionic filesystem toolchain in Repo A for the exact application prefix `/data/data/vn.shadichy.parted/files`;
3. gating startup on `/system/bin/su` and on a verified compatible toolchain;
4. fixing partition-graph sizing so no layout can overflow;
5. replacing placeholder and no-op paths with safe, tested partition and filesystem operations; and
6. establishing CI, artifact integrity, destructive-operation safeguards, and release criteria.

The work must be split into reviewable pull requests. “Complete” means every visible production action is implemented or deliberately disabled with a clear reason; no placeholder command, unreachable `UnimplementedError`, commented-out Apply path, or empty button handler may remain.

## 2. Current baseline and known gaps

The current codebase is an early prototype:

- `pubspec.yaml` targets Dart `^3.7.2`; the initial upgrade target recorded on 2026-07-18 is Flutter **3.44.6** with Dart **3.12.2**. Re-check the Flutter stable archive when implementation starts, then pin the exact latest stable version.
- Android uses generated Gradle defaults and debug signing for release builds.
- `MainActivity` extracts a bundled `prebuilts.zip` once using a marker file. It has no release manifest, version compatibility, archive-path validation, per-file integrity verification, recovery journal, or update strategy.
- startup collapses initialization failures into a generic “Missing toolchain” screen;
- root discovery currently searches `PATH`, falls back to `/system/bin/su`, and mixes binary discovery with grant acquisition;
- the command queue is untyped and widgets can create arbitrary shell jobs;
- disk create/edit/delete/format methods still return placeholder strings such as `create`, `edit`, `delete`, and `format`;
- partition type and resize commands are incomplete;
- Apply and Cancel actions are no-ops or commented out;
- the partition graph can allocate widths whose total exceeds its boundary and can pass negative inner dimensions to widgets;
- the generated counter widget test is stale;
- the repository has no application CI workflow and no release-quality integration tests.

## 3. Decisions and fixed contracts

### 3.1 Supported platform

The release target is rooted Android. Linux may remain useful as a development/test harness, but it must not weaken Android-specific safety checks.

Initial Android support is limited to the primary Android user on internal storage, because the toolchain is linked/configured for the literal prefix:

```text
/data/data/vn.shadichy.parted/files
```

Secondary users, work profiles, and adoptable-storage installs are out of scope until Repo A can build a relocatable toolchain. The app must detect an incompatible canonical `filesDir` and fail before executing any tool.

### 3.2 Root contract

The first startup gate must test the exact path:

```text
/system/bin/su
```

It must not substitute another `su` from `PATH` for this gate.

If the path is absent:

- do not scan disks;
- do not access Repo A;
- show a non-dismissible informational dialog with exactly one action, **Exit**;
- disable back/outside dismissal; and
- when Exit is activated, call a native `finishAndRemoveTask()` path, with `SystemNavigator.pop()` only as a fallback.

After path availability succeeds, perform a separate root-grant probe using `/system/bin/su -c true`. Grant denial and execution failure are distinct typed errors, not “missing toolchain.”

### 3.3 Toolchain contract

Repo A will publish immutable, per-ABI release bundles. The application defaults to `codex-clamps/fparted-toolchain`, but the repository and release channel must be build-time configurable for staging.

Android ABI mapping:

| Android ABI | Termux architecture |
| --- | --- |
| `arm64-v8a` | `aarch64` |
| `armeabi-v7a` | `arm` |
| `x86` | `i686` |
| `x86_64` | `x86_64` |

All target binaries and recursive target dependencies must be built from source for the custom package name/prefix. Do not mix binaries from the official `com.termux` repository into the target rootfs.

The bundle must install executable programs at `files/bin`, libraries at `files/lib`, and other prefix data directly under `files`; the application must change its current `/files/usr/bin` search path accordingly.

### 3.4 Package-name normalization

Keep user-facing aliases in a checked-in machine-readable file and fail on unknown names. The initial normalization is:

| Requested name | Canonical build target | Treatment |
| --- | --- | --- |
| `btrfs-props` | `btrfs-progs` | spelling alias |
| `dosfstools` | `dosfstools` | upstream recipe |
| `e2fsprogs` | `e2fsprogs` | upstream recipe |
| `erofs-utils` | `erofs-utils` | currently a root-package recipe |
| `f2fs-props` | `f2fs-tools` | spelling alias; deduplicate |
| `f2fs-tools` | `f2fs-tools` | overlay if absent upstream |
| `hfsprops` | `hfsprogs` | spelling alias |
| `jfsutils` | `jfsutils` | overlay if absent upstream |
| `ntfsprogs-plus` | `ntfsprogs-plus` | overlay if absent upstream |
| `squashfs-tools` | `squashfs-tools` | overlay unless an explicit compatibility decision accepts another implementation |
| `xfsprogs` | `xfsprogs` | overlay if absent upstream |
| `libisofs` | `libisofs` | upstream library recipe |

Do not silently replace `squashfs-tools` with `squashfs-tools-ng`: verify command and format compatibility first and document any accepted substitution.

The app itself also requires base/runtime tools not fully represented in the request. Repo A must include at least the resolved providers for `parted`, `blkid`, `dd`, `test`, and every filesystem executable exposed by the UI. `exfatprogs` should remain included while exFAT actions are visible. A generated `required-binaries.json` is the source of truth shared with the app.

## 4. Target architecture

### 4.1 Application layers

Refactor toward these boundaries:

- `bootstrap/`: root probe, ABI selection, release lookup, download state, verification, installation, and startup state machine;
- `toolchain/`: manifest models, installed-version store, executable registry, environment construction, and platform installer channel;
- `storage/`: disk discovery, parsers, immutable disk/partition/filesystem models;
- `operations/`: typed operation requests, validation, dependency ordering, command planning, execution, progress, and results;
- `ui/`: presentation only; widgets dispatch intents and render state, never construct raw shell commands;
- `platform/android/`: exact root/path probes, safe extraction, chmod/symlink operations, and process-exit bridge.

Use dependency injection for root probes, release clients, command execution, filesystem access, and clocks so all startup and operation paths can be tested without root or block devices.

### 4.2 Startup state machine

The only allowed startup order is:

```text
coldStart
  -> checkingRootBinary
  -> rootBinaryMissing -> one-action Exit dialog
  -> requestingRootGrant
  -> rootDenied / rootExecutionFailed
  -> checkingToolchain
  -> selectingCompatibleRelease
  -> downloading
  -> verifying
  -> installing
  -> validatingInstalledToolchain
  -> scanningDisks
  -> ready
```

Every state must have an explicit UI and typed failure. Disk scanning cannot begin before root and toolchain validation succeed.

### 4.3 Tool execution

Replace free-form string jobs with a typed model such as:

```dart
sealed class StorageOperation {}
final class CreatePartition extends StorageOperation { ... }
final class FormatFilesystem extends StorageOperation { ... }

final class PlannedCommand {
  final ExecutableId executable;
  final List<String> arguments;
  final String displayLabel;
  final RiskLevel risk;
  ...
}
```

A single command planner converts validated operations to commands. A single privileged executor converts argument vectors into the `su -c` transport.

Because `su -c` accepts a shell string, implement and unit-test POSIX single-quote escaping for every argument, reject NUL characters, and do not interpolate labels, filesystem names, device paths, or downloaded metadata into unquoted shell text. Set a deterministic environment (`PATH`, `HOME`, `TMPDIR`, `LC_ALL=C`) for every invocation.

## 5. Repo A: `fparted-toolchain`

### 5.1 Repository layout

Proposed layout:

```text
.
├── .github/workflows/
│   ├── ci.yml
│   ├── build.yml
│   └── release.yml
├── config/
│   ├── packages.yaml
│   ├── aliases.yaml
│   ├── required-binaries.yaml
│   └── termux-lock.json
├── overlays/
│   ├── packages/
│   └── root-packages/
├── patches/
│   └── termux-prefix.patch
├── scripts/
│   ├── resolve-packages.py
│   ├── build-rootfs.sh
│   ├── assemble-rootfs.py
│   ├── validate-rootfs.py
│   └── make-release-manifest.py
├── schemas/
│   ├── release-manifest.schema.json
│   └── rootfs-manifest.schema.json
├── tests/
└── README.md
```

### 5.2 Pinned upstream

`config/termux-lock.json` records:

- the exact `termux/termux-packages` commit;
- Android NDK/SDK values derived from that commit;
- overlay commit/version metadata;
- package roots and normalized aliases; and
- expected license/source metadata.

Never build a release from floating `master`. A scheduled dependency-update workflow may propose a lock update in a PR, but only reviewed lock changes can ship.

### 5.3 Custom Termux properties

Build with:

```text
TERMUX_APP__PACKAGE_NAME=vn.shadichy.parted
TERMUX_APP__DATA_DIR=/data/data/vn.shadichy.parted
TERMUX__ROOTFS=/data/data/vn.shadichy.parted/files
TERMUX__PREFIX=/data/data/vn.shadichy.parted/files
TERMUX__PREFIX_SUBDIR=
__TERMUX_BUILD_PROPS__VALIDATE_TERMUX_PREFIX_USR_MERGE_FORMAT=false
```

Apply the smallest maintainable patch/config override to the pinned checkout. CI must grep the final target artifacts and fail on unexpected `/data/data/com.termux`, `/files/usr`, build-host paths, or temporary staging paths.

### 5.4 Dependency build and rootfs assembly

For each architecture:

1. check out the pinned Termux commit;
2. apply the custom prefix patch;
3. overlay missing/modified package recipes;
4. validate aliases and ensure every root resolves;
5. use Termux's build-order/dependency machinery to build each root and all recursive target dependencies from source;
6. reject any downloaded target package compiled for the standard Termux prefix;
7. extract only runtime package payloads into a clean rootfs staging directory;
8. prune headers, static libraries, package-manager databases, caches, tests, and documentation only through an explicit allow/deny policy;
9. normalize timestamps, owners, ordering, and permissions for reproducibility;
10. generate a complete file manifest and package/license SBOM;
11. package and validate the result.

Overlay recipes must include upstream source URL, immutable source hash, license, maintainer, dependency list, patches with rationale, and smoke-test command. A package may be marked blocked only with a linked issue and an app-side capability flag that prevents exposing its actions.

### 5.5 Bundle format

Publish one deterministic bundle per Android ABI:

```text
fparted-rootfs-<toolchain-version>-<android-abi>.zip
```

A normal Java ZIP does not preserve all Unix metadata, so Repo A must also generate `rootfs-manifest.json` with, for every entry:

- relative path;
- type (`directory`, `file`, or `symlink`);
- mode;
- SHA-256 for regular files;
- symlink target;
- package owner; and
- mutable/immutable classification.

Materialize hard links as regular files unless the installer explicitly supports and validates hard links. Do not follow symlinks while assembling or extracting.

Each GitHub release contains:

```text
release-manifest.json
release-manifest.sig
fparted-rootfs-...-arm64-v8a.zip
fparted-rootfs-...-armeabi-v7a.zip
fparted-rootfs-...-x86.zip
fparted-rootfs-...-x86_64.zip
SHA256SUMS
packages.spdx.json
source-offer.json
```

`release-manifest.json` contains schema version, toolchain version, Termux commit, prefix, app compatibility, minimum Android API, per-ABI asset URL/name/size/hash, required binaries, package versions, and SBOM hash. Sign it with an offline or protected release key; embed only the verification public key in the app.

### 5.6 GitHub Actions

`ci.yml` on pull requests:

- validate YAML/JSON schemas and aliases;
- run ShellCheck and Python tests/lint/type checks;
- verify every source URL is pinned by hash;
- resolve the full dependency graph for all architectures;
- build at least every changed overlay package for every affected architecture;
- run reproducibility and forbidden-prefix checks.

`build.yml` as a reusable workflow:

- matrix: `aarch64`, `arm`, `i686`, `x86_64`;
- official Termux Docker build environment or the pinned Android NDK setup used by upstream;
- caches keyed by Termux commit, overlay hash, architecture, NDK, and package graph;
- upload rootfs, manifests, logs, package inventory, and SBOM as workflow artifacts;
- fail the matrix if any required binary or shared library is absent.

`release.yml` on signed version tags:

- call the full matrix build;
- compare two clean builds for reproducibility or document the remaining nondeterministic files;
- validate ELF architecture, Bionic linkage, interpreter, needed libraries, and absence of forbidden paths;
- run command smoke checks in an Android-compatible environment where possible;
- sign the release manifest;
- publish immutable GitHub Release assets and provenance attestations.

Use a self-hosted rooted Android device job for end-to-end smoke tests when available. Hosted Linux validation alone is not sufficient evidence that root filesystem tools work on a real Android kernel.

## 6. Application implementation phases

### Phase 0 — Freeze the contract and build an inventory

Tasks:

- [ ] Add constants for package ID, exact rootfs prefix, toolchain repo/channel, manifest schema, and required binaries.
- [ ] Generate a feature inventory from every visible menu, button, dialog, and runner.
- [ ] Map each feature to executable, supported filesystem variants, preconditions, command plan, and tests.
- [ ] Mark every placeholder/no-op with an issue and remove dead paths from production UI until implemented.
- [ ] Decide minimum Android API and supported ABI set based on Flutter and Repo A output.
- [ ] Record test disk-image fixtures and destructive-test rules.

Exit criteria:

- every visible action has an owner and acceptance test;
- package aliases and missing overlays are explicit;
- no ambiguity remains about the prefix, ABI mapping, root gate, or manifest format.

### Phase 1 — Upgrade and pin Flutter

Tasks:

- [ ] Re-check the official Flutter stable archive.
- [ ] Pin the exact latest stable SDK (initial target: Flutter 3.44.6 / Dart 3.12.2) in `.fvmrc` or the repository's selected version-manager file and in CI.
- [ ] Run `flutter create . --platforms=android` in a temporary comparison checkout and intentionally port generated Android/Gradle/Kotlin changes while preserving `vn.shadichy.parted`, `MainActivity`, and required resources.
- [ ] Update the Dart SDK constraint and dependencies with `flutter pub outdated` and `flutter pub upgrade --major-versions`.
- [ ] Apply `dart fix --apply`, format, and resolve analyzer failures without broad lint suppressions.
- [ ] Replace debug release signing with a documented CI signing path; debug builds may continue to use the debug key.
- [ ] Add application CI for format, analyze, unit/widget tests, Android lint, and debug/release compile smoke tests.
- [ ] Replace the generated counter test.

Exit criteria:

```text
dart format --output=none --set-exit-if-changed .
flutter analyze
flutter test
flutter build apk --debug
```

all pass on the pinned SDK, and the Android project contains no stale generated migration TODOs.

### Phase 2 — Bootstrap Repo A

Tasks:

- [ ] Create `codex-clamps/fparted-toolchain`.
- [ ] Pin `termux/termux-packages`.
- [ ] Implement package alias validation and dependency graph resolution.
- [ ] Port or overlay missing recipes.
- [ ] Build one architecture end to end, then enable the full matrix.
- [ ] Assemble deterministic rootfs bundles and manifests.
- [ ] Add ELF, path, dependency, license, SBOM, and smoke validations.
- [ ] Publish a pre-release consumed by app integration tests.
- [ ] Publish a stable signed release only after real-device validation.

Exit criteria:

- all four ABI assets exist;
- every required binary starts or reports its version/help on a compatible rooted Android test device;
- no target file contains the standard Termux prefix;
- all required shared libraries resolve inside the bundle or Android system;
- the release manifest signature and all asset hashes verify.

### Phase 3 — Root and toolchain startup gates

Tasks:

- [ ] Introduce `BootstrapController` and typed bootstrap states.
- [ ] Implement exact `/system/bin/su` availability probe.
- [ ] Implement the one-action, non-dismissible Exit dialog.
- [ ] Separate root binary availability from grant/execution errors.
- [ ] Add `android.permission.INTERNET`.
- [ ] Select the asset from `Build.SUPPORTED_ABIS`; reject unsupported ABIs with a clear screen.
- [ ] Read an installed manifest rather than a marker file.
- [ ] Query only stable compatible Repo A releases.
- [ ] Download to app cache with status checks, size limits, timeouts, cancellation, and `.partial` cleanup.
- [ ] Verify signature, manifest schema, asset SHA-256, size, prefix, package ID, ABI, and app compatibility before extraction.
- [ ] Extract to a staging directory with zip-slip, symlink, duplicate-path, path-length, and size protections.
- [ ] Recreate modes and symlinks from `rootfs-manifest.json`.
- [ ] Validate every staged file hash and every required executable before activation.
- [ ] Activate through an install journal and per-top-level atomic renames; preserve mutable data and roll back incomplete activation on next start.
- [ ] Write the installed manifest last and fsync it.
- [ ] Remove `prebuilts.zip`, `.prebuilts_extracted`, and extraction from `Activity.onCreate`.
- [ ] Add progress, retry, diagnostics, and offline behavior.

Installer-owned paths must be explicit. Never recursively delete `filesDir`; preserve app/user state such as `home`, logs, preferences, and unrelated files.

Exit criteria:

- missing `su` produces only the required Exit dialog;
- an existing compatible toolchain starts without network access;
- a clean install downloads and activates the correct ABI;
- corrupt, truncated, wrong-ABI, wrong-prefix, unsigned, oversized, and zip-slip bundles are rejected without changing the active toolchain;
- process death during each activation step recovers to either the old or new valid installation.

### Phase 4 — Secure execution and operation planning

Tasks:

- [ ] Replace raw `Job` construction with typed operations and planned commands.
- [ ] Centralize executable lookup in a manifest-backed registry.
- [ ] Implement robust `su -c` argument quoting and reject unsafe input.
- [ ] Capture exit code, stdout, stderr, duration, operation ID, and redacted command diagnostics.
- [ ] Make queue execution asynchronous and stream progress.
- [ ] Stop on the first failed command.
- [ ] Support “cancel remaining” but do not claim an in-flight filesystem mutation can always be safely interrupted.
- [ ] Rescan affected devices after success or failure.
- [ ] Add preflight checks for root grant, device existence, read-only state, mount/busy state, geometry, alignment, filesystem support, and free space.
- [ ] Generate a human-readable plan for the destructive confirmation dialog.
- [ ] Back up partition-table metadata before destructive table changes where tooling supports it.
- [ ] Never advertise automatic rollback of destructive storage operations.

Exit criteria:

- no widget imports a concrete binary wrapper or builds shell text;
- all user-controlled strings pass quoting/adversarial tests;
- command failure is visible and prevents later queued operations;
- logs are useful without leaking unrelated private data.

### Phase 5 — Complete the functional surface

Implement in dependency-safe order.

#### Disk and partition discovery

- [ ] Robustly parse all supported `parted`/`blkid` output with versioned fixtures.
- [ ] Refresh and reconcile UI state after device changes.
- [ ] Display model, size, sector sizes, table, partition identifiers, flags, filesystem, labels, mountpoints, used/free space, and unsupported states.
- [ ] Handle removable devices, devices without a table, malformed output, and device disappearance.

#### Partition table and partition operations

- [ ] Create supported partition tables.
- [ ] Create partitions with validated start/end/alignment/type.
- [ ] Delete partitions.
- [ ] Edit name, flags, and type ID where supported.
- [ ] Move/resize partitions only with correct filesystem ordering and capability checks.
- [ ] Prevent overlap, out-of-disk geometry, invalid logical/extended layouts, and operations on mounted/busy partitions.
- [ ] Refresh kernel partition information and rescan after changes.

#### Filesystem operations

For each filesystem, expose only operations supported by the installed binaries and proven command plans:

- [ ] Create/format.
- [ ] Label/rename.
- [ ] Check and repair.
- [ ] Grow.
- [ ] Shrink only where the filesystem/tool safely supports it.
- [ ] Read metadata and space usage.

Required ordering examples:

- shrink: unmount -> check -> shrink filesystem -> shrink/move partition;
- grow: grow/move partition -> grow filesystem;
- reformat: validate/unmount -> create filesystem -> set label -> rescan.

Never infer generic flags across tools; maintain a per-filesystem capability table with fixtures and real-device evidence.

#### Queue and UX

- [ ] Preview pending geometry and filesystem changes without mutating the real disk.
- [ ] Make Undo, Clear, Cancel, Apply, Help, and all dialog actions functional.
- [ ] Show exact operations and data-loss warnings before Apply.
- [ ] Show live progress and command diagnostics.
- [ ] Preserve a completed/failed operation report.
- [ ] Disable unsupported operations with an explanation.
- [ ] Add accessibility semantics, keyboard/focus behavior, and complete English strings; either complete Vietnamese strings or remove the unsupported locale declaration.

#### Dump/restore and auxiliary features

- [ ] Inventory the current dump/folder-picker paths.
- [ ] Implement them safely with Storage Access Framework streams, or remove them from release UI.
- [ ] Define whether `libisofs` enables an application feature or is only a requested runtime dependency.
- [ ] Do not declare feature completeness for an unused library.

Exit criteria:

- every production control has an observable tested result;
- no `UnimplementedError`, placeholder command, empty handler, or commented production execution remains;
- unsupported operations cannot be queued;
- all destructive actions require a validated plan and explicit confirmation.

### Phase 6 — Fix the partition graph

Replace the mutable-width `FlexRow` algorithm with a pure integer-pixel allocator.

Algorithm requirements:

1. Compute the exact drawable width after external padding and inter-segment gaps.
2. Clamp invalid logical sizes and normalize each segment by disk bytes using high-precision/integer math.
3. Allocate ideal widths by largest-remainder apportionment so the final sum equals the drawable boundary exactly.
4. Apply a visual minimum only when feasible:
   - reserve minimum pixels for tiny segments;
   - distribute the remainder proportionally among other segments;
   - if `segmentCount * minimum > drawableWidth`, reduce the minimum to a feasible value or aggregate tiny segments for display;
   - never increase total width beyond the boundary.
5. Assign final rounding residuals deterministically to the largest-remainder segments.
6. Clamp filesystem-used/free ratios to `[0, 1]`.
7. Never subtract fixed padding/border values from a child width without clamping to zero.
8. Keep logical geometry separate from visual width and hit-testing metadata.
9. Hide text based on measured available space, not arbitrary nested negative sizes.

Prefer a dedicated `CustomPainter`, `CustomMultiChildLayout`, or a tested layout delegate over nested fixed-width containers.

Tests must cover:

- zero, one, and many partitions;
- a one-byte partition on a multi-terabyte disk;
- hundreds/thousands of tiny areas;
- all-free and no-free layouts;
- inconsistent filesystem-space metadata;
- widths from zero through large tablets;
- portrait/landscape and text scale up to at least 2.0;
- exact width-sum invariant;
- no negative constraints and no Flutter overflow exceptions.

Exit criteria:

- the allocator is a pure unit-tested function;
- graph widgets pass golden/widget tests at narrow and wide constraints;
- no yellow/black overflow stripe appears for any fixture.

### Phase 7 — Verification and release

Application CI gates:

- Dart formatting and analysis;
- unit tests for parsing, geometry, quoting, planner, bootstrap, and manifest validation;
- widget/golden tests for startup states, graph, dialogs, queue, progress, and error views;
- Android lint and debug APK build;
- release APK/AAB compile with injected non-production signing in CI;
- dependency and secret scanning.

Integration gates:

- sparse disk images and disposable loop devices on privileged Linux CI;
- never host block devices;
- each supported partition-table/filesystem operation tested where kernel/tool support permits;
- injected/fake Android bootstrap tests for all error states;
- real rooted Android matrix for at least `arm64-v8a`, plus another ABI when hardware/emulator support exists;
- install from empty state, offline restart, corrupt update, interrupted update, downgrade rejection, and recovery tests.

Release gates:

- no debug signing;
- app version and toolchain compatibility declared;
- Repo A stable assets immutable and signed;
- licenses/source offer included;
- clean install and destructive-operation manual checklist completed on disposable media;
- screenshots and release notes updated;
- all critical/high defects closed.

## 7. Suggested pull-request sequence

1. **Planning and invariants** — this plan, `AGENTS.md`, issues, constants.
2. **Flutter upgrade and CI baseline** — no functional behavior change.
3. **Pure geometry allocator and graph tests** — isolated UI fix.
4. **Typed bootstrap/root gate** — fake toolchain client first.
5. **Repo A skeleton and one-ABI proof**.
6. **Repo A full matrix, manifests, signatures, and pre-release**.
7. **Downloader/installer and recovery journal**.
8. **Typed operation model, quoting, executor, and queue UI**.
9. **Discovery/parsing hardening**.
10. **Partition-table/create/delete operations**.
11. **Filesystem format/label/check/repair operations**.
12. **Resize/move ordering and capability matrix**.
13. **Dump/restore or deliberate removal**.
14. **Real-device hardening, signing, and release candidate**.

Avoid combining the Flutter upgrade, bootstrap installer, and destructive operation implementation in one PR.

## 8. Definition of done

The project is complete for the declared v1 scope only when:

- the repository pins and passes the latest selected Flutter stable SDK;
- Repo A reproducibly publishes signed, verified bundles for all supported ABIs with the exact prefix;
- startup obeys the root and toolchain state machine;
- the missing `/system/bin/su` path always leads to the one-action Exit dialog;
- toolchain installation is verified, recoverable, and never trusts archive paths or hashes blindly;
- partition graph width allocation is bounded and tested;
- every visible app action is implemented or intentionally disabled;
- no placeholder/no-op/unreachable production path remains;
- destructive operations are preflighted, ordered, confirmed, logged, and stopped on failure;
- automated tests and disposable-device validation cover the supported feature matrix;
- release builds are properly signed and include license/source obligations; and
- `AGENTS.md`, user documentation, support limits, and recovery guidance match the shipped behavior.

## 9. Primary risks

| Risk | Mitigation |
| --- | --- |
| Requested packages are missing or unmaintained in current Termux upstream | pinned overlays, source hashes, per-package owners, capability flags, explicit blockers |
| Custom prefix accidentally mixes `com.termux` artifacts | build every target dependency locally; forbidden-string/ELF validation |
| Android kernel lacks a filesystem driver or ioctl | capability probe; disable action with explanation; real-device matrix |
| Root managers expose `su` outside `/system/bin` | intentional product requirement; document exact gate |
| Archive compromise or corruption | signed manifest, per-asset and per-file hashes, schema/prefix/ABI checks |
| Interrupted installation leaves mixed binaries | staging, install journal, controlled path swaps, startup recovery |
| Shell injection through labels/paths | typed arguments, centralized POSIX quoting, adversarial tests |
| Destructive operation cannot be rolled back | explicit preflight/confirmation, metadata backup, stop-on-failure, no false rollback claim |
| Graph has more tiny segments than pixels | feasible minimum/aggregation and exact apportionment |
| “Complete functionality” expands without bound | frozen v1 feature inventory and capability matrix; remove unsupported UI |

## 10. Reference sources

- Flutter SDK archive: <https://docs.flutter.dev/install/archive>
- Flutter release notes: <https://docs.flutter.dev/release/release-notes>
- Termux package build system: <https://github.com/termux/termux-packages>
- Termux build-package documentation: <https://github.com/termux/termux-packages/wiki/Build-package-management>
- Termux execution environment: <https://github.com/termux/termux-packages/wiki/Termux-execution-environment>
- Termux filesystem layout: <https://github.com/termux/termux-packages/wiki/Termux-file-system-layout>
