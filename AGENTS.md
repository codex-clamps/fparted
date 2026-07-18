# AGENTS.md

## Scope and source of truth

These instructions apply to the entire `fparted` repository.

Read `PLAN.md` before making architectural, root/bootstrap, toolchain, partition-operation, or graph-layout changes. Keep implementation pull requests aligned with its staged delivery plan. When code and documentation disagree, update both in the same change.

This project controls privileged storage tools and can irreversibly destroy data. Correctness, explicit capability checks, and test isolation take priority over speed or UI convenience.

## Fixed product invariants

Do not change these without an explicit design decision in `PLAN.md` and the pull request:

- Android application ID: `vn.shadichy.parted`.
- Required toolchain prefix: `/data/data/vn.shadichy.parted/files`.
- Required initial root-binary gate: exact path `/system/bin/su`.
- Missing-root behavior: a non-dismissible informational dialog with exactly one action, **Exit**; activating it exits the task.
- Disk discovery starts only after root grant and compatible toolchain validation succeed.
- Downloaded toolchains come from the configured Repo A release channel and are accepted only after signature, schema, ABI, prefix, size, and hash validation.
- Widgets do not construct shell commands or directly launch privileged processes.
- Tests never operate on a developer's or runner's real block devices.

## Repository structure

Current and target ownership:

- `lib/main.dart`: composition root only; keep startup orchestration out of widget constructors.
- `lib/core/`: legacy models/runners being migrated. Do not add new placeholder behavior here.
- `lib/bootstrap/`: root and toolchain startup state machine.
- `lib/toolchain/`: release/installed manifests, executable registry, downloader interfaces, installer state.
- `lib/storage/`: disk/partition/filesystem models and parsers.
- `lib/operations/`: typed requests, validation, planning, queueing, execution, and results.
- `lib/layouts/` or a future `lib/ui/`: presentation. UI dispatches intents and renders state.
- `android/`: Android bridge, exact root/path probes, safe installer primitives, and task exit.
- `test/`: unit and widget tests mirroring `lib/`.
- `integration_test/`: app-level tests using fakes or explicitly provisioned disposable targets.
- `test/fixtures/`: versioned, sanitized command output and disk-layout fixtures.
- `tool/`: development scripts that do not belong in runtime code.

Do not create a second implementation of bootstrap, command execution, path constants, or executable discovery in a feature folder. These are single-owner services.

## Toolchain versions and setup

The planning baseline targets Flutter **3.44.6** with Dart **3.12.2**, the latest stable recorded on 2026-07-18. At the start of the SDK-upgrade work, re-check the official Flutter stable archive, pin the exact latest stable version, and commit the version-manager file. After that, the checked-in pin is authoritative.

Use Flutter's Pub workflow. Commit intentional `pubspec.lock` changes.

Expected local checks:

```bash
flutter --version
flutter doctor -v
flutter pub get
dart format --output=none --set-exit-if-changed .
flutter analyze
flutter test
flutter build apk --debug
```

When Android build files change, also run the applicable Gradle checks from `android/`:

```bash
./gradlew lint
./gradlew test
```

A focused test is acceptable during development, but the full relevant suite must pass before a PR is declared ready.

## Coding rules

### General Dart and Flutter

- Accept `dart format` output; do not hand-align code.
- Follow `analysis_options.yaml`. Suppress a lint only at the narrowest location with a reason.
- Prefer immutable value objects, sealed states/results, exhaustive switches, and dependency injection.
- Use typed exceptions/failures with stable error codes. Do not route unrelated failures to “Missing toolchain.”
- Do not use `print` in production paths. Use the repository logging abstraction with sensitive-value redaction.
- Check `mounted` after awaits before using a widget context.
- Keep blocking I/O, archive extraction, hashing, and command execution off the Android/UI thread.
- Do not add an empty handler, placeholder command, `TODO` in a production action, or unreachable `throw UnimplementedError()` after a return.
- A production action that is not implemented must be absent or disabled with an explanatory capability reason.

### Constants and paths

Define package ID, prefix, executable directories, manifest schema, Repo A identity/channel, and required binaries once. Import those constants; do not duplicate string literals.

For the selected non-`usr`-merge toolchain:

```text
rootfs/prefix = /data/data/vn.shadichy.parted/files
PATH begins    /data/data/vn.shadichy.parted/files/bin
libraries     /data/data/vn.shadichy.parted/files/lib
```

Do not reintroduce `/data/data/com.termux` or `/data/data/vn.shadichy.parted/files/usr` into target command paths.

### Startup and root behavior

Implement startup as explicit states. The required order is:

1. check exact `/system/bin/su`;
2. if absent, show the one-action Exit dialog and do nothing else;
3. request/probe root grant separately;
4. validate or install a compatible toolchain;
5. validate required executables;
6. scan disks;
7. enter the main UI.

The missing-root dialog:

- has one visible action named `Exit`;
- cannot be dismissed by back, escape, or tapping outside;
- does not offer Retry, Settings, Continue, or a download link;
- exits through the Android bridge (`finishAndRemoveTask`) with a Flutter fallback.

Use injectable probes in tests. Do not make widget tests depend on the host's `su`.

### Toolchain download and installation

Never trust a GitHub filename, redirect, archive entry, symlink target, or downloaded manifest.

Before activation, verify:

- release-manifest signature;
- supported schema;
- app compatibility;
- package ID and exact prefix;
- selected Android ABI;
- declared and actual byte size;
- asset SHA-256;
- every extracted regular-file SHA-256;
- every entry type and mode;
- required executable set; and
- absence of path traversal, duplicate normalized paths, forbidden absolute paths, and links escaping the root.

Extract to staging. Never extract directly over the active toolchain. Use a recovery journal and controlled atomic renames for installer-owned paths. Write the installed manifest last. On process death, the next startup must recover to a complete old or new installation.

Never recursively delete `filesDir`. Preserve mutable/application-owned data. Do not perform network or extraction work in `Activity.onCreate`.

### Command construction and privilege boundary

No feature widget may import a concrete binary wrapper, construct a `Job`, call `Process.run`, or concatenate a shell command.

The flow is:

```text
user intent
-> typed StorageOperation
-> preflight validation
-> ordered PlannedCommand list
-> explicit confirmation
-> privileged executor
-> structured results
-> device rescan
```

`su -c` is a shell boundary. The executor must:

- accept an executable identifier and argument vector, not a prebuilt shell string;
- resolve the executable from the verified toolchain registry;
- POSIX-single-quote every argument centrally;
- reject NUL and other unsupported control characters;
- set deterministic `PATH`, `HOME`, `TMPDIR`, and `LC_ALL=C`;
- capture exit code, stdout, stderr, duration, and operation ID;
- redact sensitive/unrelated paths from diagnostics; and
- stop the queue after the first failed command.

Never interpolate partition labels, filesystem labels, device paths, folder names, or downloaded values into unquoted command text. Add adversarial tests whenever quoting or argument handling changes.

### Destructive operation safety

Before queuing an operation, validate at least:

- selected device still exists and resolves to the expected identity;
- target is not the fallback/test device;
- read-only state;
- mounted/busy state;
- current disk generation has not changed since planning;
- geometry, alignment, boundaries, and non-overlap;
- partition-table constraints;
- filesystem capability and installed executable availability;
- shrink/grow ordering;
- expected free space; and
- root grant is still usable.

Show a human-readable, ordered plan before Apply. The confirmation must clearly identify the device and destructive effects.

Execute sequentially. Stop on the first failure and rescan. “Cancel” may cancel pending commands; do not promise that an already running filesystem mutation can be interrupted safely. Do not claim automatic rollback for destructive operations. Back up partition-table metadata when supported and explain recovery limits.

Use these ordering rules unless a filesystem-specific capability overrides them:

- shrink: unmount/check -> shrink filesystem -> shrink/move partition;
- grow: grow/move partition -> grow filesystem;
- format: validate/unmount -> create filesystem -> label -> rescan.

### Partition graph and layout math

Partition sizes can exceed JavaScript/Dart exact-double ranges and visual segments can be smaller than one pixel. Keep logical geometry in integer/`BigInt` form.

The graph allocator must be a pure tested function that:

- receives drawable integer pixels and logical byte weights;
- never returns a negative width;
- returns widths whose sum equals the drawable boundary exactly;
- applies minimum visual widths only when feasible;
- handles more segments than available pixels deterministically;
- clamps malformed used/free ratios;
- separates visual size from logical geometry; and
- does not mutate the remaining width while iterating children without a final normalization.

Do not fix overflow with clipping alone. Do not subtract padding/borders from a tiny child without clamping. Add narrow-screen and pathological-layout widget tests with every graph change.

## Testing rules

### Default tests

Use fakes for:

- root probes;
- release API and downloads;
- signature/hash verification;
- installer filesystem;
- privileged executor;
- disk discovery; and
- clocks/retry policies.

Fixture-based parser tests must preserve the source tool/version and cover malformed/truncated output.

Required test groups include:

- data-size and geometry arithmetic;
- device-name parsing;
- package/manifest schema validation;
- archive path and symlink validation;
- command quoting/injection attempts;
- operation preflight and ordering;
- stop-on-failure queue behavior;
- bootstrap state transitions;
- missing-root one-action dialog;
- graph width-sum/no-overflow invariants;
- enabled/disabled capability rendering; and
- startup recovery after interrupted installation.

### Destructive integration tests

Never run partition commands against `/dev/block/*`, `/dev/sd*`, `/dev/nvme*`, the root filesystem, or any pre-existing host device from automated tests.

Linux integration tests may use a newly created sparse file and an explicitly attached disposable loop device only when all of the following are true:

- the job is isolated/privileged for this purpose;
- `FPARTED_ALLOW_LOOP_DEVICE_TESTS=1` is set;
- the test records the loop device it created;
- it verifies the backing file is inside the test temporary directory before every mutation; and
- cleanup detaches that exact device in a trap/finalizer.

Android destructive tests run only on disposable emulator images or explicitly designated removable media/rooted test devices. Print the target and require an out-of-band test configuration; never infer a writable target.

## Repo A interface

The application and Repo A communicate only through versioned manifests. Do not scrape release filenames as the primary protocol.

The release manifest must provide:

- schema/toolchain version;
- Termux commit;
- exact package ID and prefix;
- min Android API and app compatibility;
- Android ABI to asset mapping;
- URL/name, size, and SHA-256 per asset;
- required binaries;
- package versions/licenses; and
- SBOM/provenance references.

The rootfs manifest must describe each entry's relative path, type, mode, hash or link target, package owner, and mutability.

Changes to either schema require:

- a schema version bump;
- backward/forward compatibility decision;
- tests in both repositories; and
- coordinated release notes.

## UI and localization

- All asynchronous states need progress and actionable typed errors.
- Destructive confirmations must use concrete device/operation text, not generic “Continue?” alone.
- Disabled operations must explain the missing capability/tool/precondition.
- Provide semantics and focus order for graph segments, tables, dialogs, and action buttons.
- Do not declare a locale supported while shipping incomplete generated/stale strings. Complete it or remove it from `supportedLocales`.
- Include screenshots or recordings for meaningful UI changes.

## Definition of done for a change

A change is not complete because it compiles. Before marking a PR ready:

- production paths contain no new placeholder/no-op;
- relevant unit/widget/integration tests exist and pass;
- formatting, analysis, and Android build checks pass;
- dangerous behavior and target devices are documented;
- startup/toolchain changes include failure and recovery tests;
- operation changes include command-plan and failure-path tests;
- graph changes include boundary/pathological layouts;
- public behavior and support limits are documented;
- `PLAN.md` checklists/issues are updated when milestone status changes; and
- the PR describes what was validated on a real rooted Android device, or explicitly states that real-device validation remains outstanding.

## Commit and pull-request guidance

Keep changes focused. Do not combine a Flutter migration, rootfs installer, and destructive operation implementation in one PR.

History uses short imperative subjects; Conventional Commit prefixes are optional. A PR should include:

- problem and scope;
- design/contract changes;
- user and data-loss impact;
- exact checks run;
- fixtures/devices used;
- screenshots for UI changes;
- Repo A/app compatibility implications; and
- remaining blockers or manual validation.

Never describe the app or a filesystem feature as complete while a visible control is a no-op, a required package is unvalidated, or a destructive path lacks real-device evidence.
