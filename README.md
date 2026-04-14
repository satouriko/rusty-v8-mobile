# rusty-v8-mobile

Prebuilt `librusty_v8.a` static libraries for mobile targets that the [official rusty_v8 releases](https://github.com/denoland/rusty_v8/releases) don't ship.

## Targets

| Target | Runner | Use case |
|--------|--------|----------|
| `aarch64-linux-android` | Linux | Android device |
| `x86_64-linux-android` | Linux | Android emulator |
| `aarch64-apple-ios` | macOS | iOS device |
| `aarch64-apple-ios-sim` | macOS | iOS simulator (Apple Silicon) |

## Usage

Set `RUSTY_V8_MIRROR` so rusty_v8's build script fetches the prebuilt archive instead of compiling V8:

```bash
RUSTY_V8_MIRROR=https://github.com/satouriko/rusty-v8-mobile/releases/download \
  cargo build --target aarch64-linux-android
```

Or download the `.a.gz` for your target manually from [Releases](https://github.com/satouriko/rusty-v8-mobile/releases), decompress, then:

```bash
RUSTY_V8_ARCHIVE=/path/to/librusty_v8_release_aarch64-linux-android.a \
  cargo build --target aarch64-linux-android
```

## Building a new version

Trigger the **Build** workflow via `workflow_dispatch`, specifying the rusty_v8 version (e.g. `130.0.7`). The workflow compiles V8 from source for all targets and uploads the artifacts as a GitHub Release.
