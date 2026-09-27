# Agent guide: Sony Device Center

This file applies to the whole repository. It preserves the build, review, and
debugging knowledge from the September 2026 work so a new session can continue
on macOS. Follow the current user's task and check the checkout before acting.
The issue and PR status below is a dated snapshot; verify it with `gh` before
merging, closing issues, or claiming a fix is released.

## Start here

- Repository: `marconvcm/sony-device-center`; default branch: `main`.
- Inspect `git status --short`, branch, and recent commits. Preserve existing work.
- Use the GitHub CLI for issues, PR diffs, reviews, and CI results. Read the actual
  diff and issue discussion; a PR title or green build is insufficient evidence.
- Carry authorized work through implementation and appropriate checks. Respect
  the user's requested branch and delivery path; avoid repeated permission
  questions for work already authorized in the current task.
- Give short progress updates. Report code changes, validation, and remaining
  limitations separately. Record physical-headset results with model, firmware,
  OS version, commit, and whether the GUI used IPC or a direct session.

## Repository map and sources of truth

| Path | Purpose |
| --- | --- |
| `CMakeLists.txt` | Current root build, version, platform packaging |
| `apps/device-center/` | Modern Qt 6/QML GUI; controller caches and worker backend |
| `apps/sonyd/`, `apps/sonyctl/` | Daemon and diagnostic/control CLI |
| `libs/sony-transport/` | Transport adapters, discovery, Sony candidate filter |
| `libs/sony-protocol/` | Frames, sessions, V1/V2 commands, device profiles |
| `libs/sony-core/` | Device service, state, reconnect policy, JSON and Unix IPC |
| `Client/` | Legacy clients and native Bluetooth connectors still used by adapters |
| `tests/` | Catch2 protocol/transport/core tests, Qt tests, packaging tests |
| `packaging/macos/` | Qt deployment, DMG, signing, notarization, verification |
| `.github/workflows/cmake.yml` | Current Linux/Windows/macOS test matrix and sanitizers |
| `.github/workflows/release.yml` | Release packaging and optional macOS signing |

Read [IPC and lifecycle](docs/ipc-and-lifecycle.md), [packaging](packaging/README.md),
and the code relevant to the task. `docs/architecture-current.md` records the
**legacy migration baseline**, despite its name. Its statements that there is no
root build, Qt app, or separate SDK are historical. The legacy Xcode project and
`xcodebuild.yml` are not the normal build path for the modern Qt application.

Some documentation/help text still describes only five EQ bands or NSIS packages.
Current code supports the XM6 ten-band path, and release CI builds Windows MSI
with WiX 3 plus ZIP. Check implementation and workflows when these disagree.

## Build and test on a Mac

Run from the repository root. Install Xcode Command Line Tools if missing
(`xcode-select --install`), and have Homebrew available.

```sh
git submodule update --init --recursive
brew install cmake ninja qt@6 pipx
pipx install dmgbuild
export PATH="${PIPX_BIN_DIR:-$HOME/.local/bin}:$(brew --prefix qt@6)/bin:$PATH"

cmake -S . -B build-macos -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_TESTING=ON -DSONY_REQUIRE_QT=ON \
  -DCMAKE_PREFIX_PATH="$(brew --prefix qt@6)"
cmake --build build-macos --parallel 2
ctest --test-dir build-macos --output-on-failure --no-tests=error
python3 -m unittest discover -s tests -p 'test_macos_packaging.py' -v

open build-macos/apps/device-center/sony-device-center.app
```

- `SONY_REQUIRE_QT=ON` prevents a misleading successful build that skipped the GUI.
- Catch2 3 is used if installed; otherwise CMake downloads pinned Catch2 3.8.1.
  Initial configuration may need network access. Initialize the ImGui submodule
  even when concentrating on the modern application.
- Use a separate build directory when changing compiler, architecture, Qt, or
  sanitizer configuration. Start with the Mac's native architecture.
- Homebrew Qt supplies the host architecture. Universal `arm64;x86_64` builds
  require universal Qt and dependencies. Release CI uses the universal Qt archive;
  adding the universal CMake flag to a Homebrew build is insufficient.
- Current release CI selects Qt `6.10.*`; Homebrew versions can differ. Record
  the actual version. The configured macOS deployment target is 13.0; a newer
  dependency may raise the effective runtime requirement.
- Linux offscreen environment variables are for container tests. Use the native
  Cocoa platform when checking normal macOS windows and Bluetooth permissions.

### Build the distributable DMG

The development `.app` still depends on the build machine's Qt. The `dmg` target
deploys Qt and includes `sonyd` and `sonyctl` under `Contents/MacOS`.

```sh
cmake --build build-macos --target dmg
packaging/macos/verify-dmg.sh build-macos/*.dmg
```

If Qt deployment is not on PATH, set
`MACDEPLOYQT="$(brew --prefix qt@6)/bin/macdeployqt"` for the build command.
The image contains `Sony Device Center.app` and an Applications shortcut.
`verify-dmg.sh` mounts the image, checks dependencies and signatures, runs CLI
help, and starts the GUI for ten seconds. It does not exercise a headset.
CPack's macOS TGZ wraps the undeployed app and is not the user DMG.

### Hardware testing and diagnosing a stall

Record `git rev-parse HEAD`, `sw_vers`, `uname -m`, Qt version, model, firmware,
and the phase that failed: Gatekeeper, launch, discovery, connect, command,
reconnect, or quit. Distinguish a responsive GUI with a blocked backend from a
main-thread freeze. Useful commands, run individually:

```sh
build-macos/apps/sonyctl/sonyctl --help
build-macos/apps/sonyctl/sonyctl -v --direct devices
build-macos/apps/sonyctl/sonyctl -v --direct info
build-macos/apps/sonyd/sonyd -v --device 'AA:BB:CC:DD:EE:FF'
```

Replace the example address with the actual headset. Stop the daemon and close
any GUI holding a direct connection before direct CLI tests. `--direct` refuses
to compete with a reachable daemon; it does not coordinate with other Bluetooth
apps. Run the daemon command separately, then launch the GUI or ordinary CLI to
test IPC. Use the CLI bundled inside the installed app when testing that artifact.

Test discovery of renamed Sony devices, unrelated peripherals, and paired
headsets that macOS does not report as connected. Then test connection, readback,
ANC/Ambient transitions, model-appropriate EQ, power-cycle/reconnect, and quit
during an unavailable-device attempt. Compare settings with Sony Sound Connect;
an ACK alone does not establish the audible result. Phone multipoint connections
can affect reproduction and should be recorded.

For a hang, capture the affected process stack with Activity Monitor's Sample
Process or macOS `sample`, alongside verbose logs. Existing macOS concerns are
in `Client/macos/MacOSBluetoothConnector.mm`: early promise resolution after
async open, synchronous open/write, ignored native write status, and delayed
run-loop servicing. Issues #46/#47 remain relevant; a packaging success does
not establish that these paths work on hardware.

## macOS signing and Gatekeeper (#57)

PR #67 merged on 2026-09-27 as `ce35d0b55a0cbf0c1b16c0214815a1b036bd5651`.
It prepares signing and notarization; merging it does not certify existing assets.
At the last check, v0.1.5 was ad-hoc signed/unnotarized, and no repository signing
secrets or variables were configured. Recheck this before describing a release.

- `build-dmg.sh` signs nested Mach-O code and bundles before the outer app, then
  signs the DMG. Developer ID signing uses hardened runtime and timestamps.
- Only the GUI receives `app.entitlements` with `com.apple.security.cs.allow-jit`.
  Preserve this separation from CLI helpers; avoid broadening entitlements
  without evidence. Do not replace inside-out signing with blanket `--deep` signing.
- Notarization must return `Accepted`; the ticket is then stapled and validated.
- `SONY_REQUIRE_NOTARIZATION=true` requires signing and notary configuration.
  Verification then checks the ticket and Gatekeeper acceptance of DMG and app.
- Default CI builds remain ad-hoc. `codesign --verify` checks integrity; by itself
  it does not establish a trusted publisher or Gatekeeper acceptance.

For local signing, import a **Developer ID Application** identity into Keychain
and create a `notarytool` credential profile before running:

```sh
export SONY_CODESIGN_IDENTITY="Developer ID Application: Your Name (TEAMID)"
export SONY_NOTARY_PROFILE="notary"
export SONY_REQUIRE_NOTARIZATION=true
cmake --build build-macos --target dmg
packaging/macos/verify-dmg.sh build-macos/*.dmg
```

Optional `SONY_SIGNING_KEYCHAIN` selects an isolated keychain for both tools.
For GitHub releases, [packaging/README.md](packaging/README.md#github-actions-setup)
documents six secrets: `SONY_MACOS_CERTIFICATE_BASE64`,
`SONY_MACOS_CERTIFICATE_PASSWORD`, `SONY_CODESIGN_IDENTITY`,
`SONY_NOTARY_KEY_BASE64`, `SONY_NOTARY_KEY_ID`, `SONY_NOTARY_ISSUER_ID`.
The repository variable `SONY_MACOS_SIGNING_ENABLED=true` enables the path.
`prepare-signing.sh` imports credentials; the workflow cleans them up even after
failure. Keep private keys, certificates, and passwords out of commits and logs.

Use a manual Release workflow run to validate packaging before publishing a tag.
Manual runs upload artifacts without publishing a GitHub release. Finally test
a browser-downloaded DMG on a clean Mac, preserving quarantine. A locally built
or de-quarantined app cannot demonstrate that the reported first-launch block
is resolved. Bluetooth consent is a separate OS permission.

For a trusted existing unnotarized download, README/release notes document
**System Settings → Privacy & Security → Open Anyway** after one failed launch.
The old right-click override does not work on macOS 15+. The reported fallback is:

```sh
xattr -dr com.apple.quarantine "/Applications/Sony Device Center.app"
```

This bypasses quarantine for that copy; it does not sign, notarize, or verify it.
The recursive `-r` matters because bundled files can also be quarantined. Keep
#57 open until an actual notarized release and clean-Mac launch are verified.

## Tests, containers, and lessons from failures

- Protocol/transport/core tests are hardware-independent. Core CTest timeout is
  **30 seconds**: lifecycle tests intentionally wait through protocol deadlines
  and exceeded the former ten-second limit on macOS. Do not lower it blindly.
- Use `tests/protocol/ReplyingFakeTransport.h` for replies that must follow a send.
  Pre-queuing ACKs can let the reader consume them before a request registers;
  this caused the XM6 test race fixed during PR #66 integration.
- Check literal packet bytes, generation boundaries, and observable behavior.
  Distinguish flaky fake transport timing from production transport failures.
- Run relevant tests and the affected native build. Docs-only edits need link,
  command, and whitespace checks, not a new full C++ matrix.
- Packaging changes: run the 11 Python tests above, `bash -n` on changed scripts,
  and `actionlint` on changed workflows when available. The Python tests mock
  Apple tools; real signing still needs credentials and a Mac.

The Ubuntu 24.04 container is reusable on Docker or Podman. It installs Qt/QML,
compiler tools, and checksum-verified Catch2 at image build time. Test runs need
no network or Bluetooth devices:

```sh
docker build -t sony-device-center-tests -f tests/container/Dockerfile tests/container
docker run --rm --network=none \
  -v "$PWD:/src:ro" -v sony-device-center-build:/build \
  sony-device-center-tests
```

Use a separate volume per checkout/configuration. For ASan/UBSan, add
`-e ENABLE_SANITIZERS=ON -e BUILD_TYPE=Debug` and use a separate build volume.
Arguments after the image name go to CTest, e.g.
`-R 'Speak-to-Chat|10-band|SonyDeviceDiscovery'`. For Fedora rootless Podman add
`--security-opt label=disable`, as documented in [tests/container/README.md](tests/container/README.md).
Docker on a Mac runs these Linux tests in a VM; native Bluetooth and Gatekeeper
still require the macOS build.

Last verified [PR #67 CI run](https://github.com/marconvcm/sony-device-center/actions/runs/36324524038):
Linux 137/137, Windows 134/134, macOS 136/136, sanitizer subset 135/135; 11
packaging tests passed on Linux and macOS, and native ad-hoc DMG verification
passed. Seven discovery/selection cases also passed in the Linux container.
These counts are a dated baseline, not a requirement that future counts match.
Real Developer ID signing/notarization and headset behavior were not validated
by that run.

## Protocol and ownership rules to preserve

- Opcode `0x22` is a battery inquiry on V2 but **POWER OFF on V1**. Do not probe
  unknown devices with V2 commands. Preserve `tests/core/ProtocolSafetyTests.cpp`.
- The modern service chooses a profile on every connection; unknown names fall
  back to V1 with empty capabilities. Native SDP detection does not automatically
  resolve missing model support in `SonyDevice::connect()`.
- Preserve XM6's ten EQ bands, raw range 0–12, and lack of separate Clear Bass.
  Legacy five-band EQ uses -10–10 plus Clear Bass. Check UI, JSON, CLI, simulator,
  notifications, and protocol code together when touching equalizer behavior.
- XM4 V1 Speak-to-Chat uses subtype `0x05`; enabling writes configuration for the
  Standard timeout before the enable command. PR #59 has independent packet tests.
- Preserve Sony OUI **or** name filtering and paired/connected hints. Renamed
  headsets and paired audio/BLE devices must not disappear through stricter filters.
- The GUI worker owns backend operations. Keep I/O off the GUI thread and update
  the daemon and GUI together for structured IPC version compatibility.
- A direct GUI can own RFCOMM before a later daemon starts. Investigate ownership
  before treating EBUSY or competing connections as a packet problem.
- Unix socket parents must be user-owned `0700`; sockets are `0600`. The default
  macOS fallback is `/private/tmp/sony-device-center-<uid>/sony-device-center.sock`.
  Do not weaken endpoint checks to make a test pass. Windows IPC is unsupported.

## PR and issue audit snapshot — 2026-09-27

PRs **#52, #44, #59** were merged through integration **#66**. It also added the
test container, fixed the XM6 ACK race, and raised the core timeout. PR **#67**
added the macOS packaging work above. All 16 open issues received code-linked
audit comments. None met its full closure criteria at that time.

| Issues | Remaining work |
| --- | --- |
| #57 | Configure Apple credentials, publish notarized DMG, verify downloaded launch. |
| #45 | Sony filtering and native metadata fixed by #52; QML still labels every non-active device “Available”. |
| #46, #47 | Bound native connection/write work, readiness callbacks, cancellation and shutdown; hardware traces needed. |
| #34 | Controller still overwrites remembered Ambient level with zero from ANC/Off, then sends zero. |
| #55 | XM4 Speak-to-Chat fixed by #59; Ambient/DSEE and reported EQ snap-back are not fully resolved. |
| #64 | ULT WEAR mode-switching overlaps #34; artwork falls back to XM5, DSEE effect unverified. |
| #56 | WF-XM4 needs verified NCASM `0x15` layout and wind byte; aggregate battery success returns before L/R/case inquiries. |
| #58, #65, #12 | XB910N, CH520, MDR-1000X lack explicit profiles; current fallback is not verified model support. |
| #11 | Battery refresh/error reporting improved; intermittent XM6 readings still need hardware confirmation. |
| #14, #35 | No persisted per-device artwork color selection or neutral unknown fallback. |
| #21 | Main.qml remains monolithic; per-page QML/module refactor not merged. |
| #7 | Windows signatures, action SHA pinning, and build provenance attestations remain outstanding. |

Open PR review findings to recheck against their latest heads:

- **#48/#50/#51:** overlapping/conflicting macOS lifecycle fixes. Consolidate and
  preserve #52's renamed-device discovery. #50 includes the useful Ambient fix,
  but its startup metadata handling had a Windows regression concern.
- **#53:** Windows paired-device discovery; #52 dependency is merged. Rebase and
  validate native connection metadata and hardware behavior.
- **#63:** large UI redesign with conflicts and bundled #60/#61/#62 work. Its
  five-band assumptions regressed the XM6 path; do not merge without reconciling it.
- **#61:** GUI simulator proposal used a fixed five-element array comparison
  incompatible with the new vector and lacked XM6 simulation support. Existing
  `sonyd --simulated` works independently of this unmerged GUI option.
- **#60:** Windows executable icon; straightforward, still needs its CI checks.
- **#62:** core timeout change is already incorporated through #66; PR is redundant.
- **#43:** Arch PKGBUILD needs clean Arch validation and updated release assumptions.

Close an issue only when its complete scope is resolved. Link the merged commit,
specific implementation, and relevant tests in the issue. Keep partially fixed
or hardware-unverified reports open and describe the remaining requirement.
An open PR, a mock test, and a shipping artifact establish different things.

Release tags must match the root `project(VERSION ...)`. Packaging currently
uses `BUILD_TESTING=OFF` and does not itself gate on the separate CI workflow;
verify CI for the exact commit before publishing. A merged fix on main is not
automatically included in the latest downloadable release.
