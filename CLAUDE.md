# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Android library that packages the [Opus audio codec](https://opus-codec.org/) (vendored upstream Opus 1.5.2) behind a small Java API. Originally from `martoreto/opuscodec` (via the `spacen` fork); this fork (`com.github.chrisfraser`) is published via JitPack. The companion `example/` directory is a separate Gradle project demonstrating record→encode→decode→play.

## Build / publish

The root project is the library (`com.android.library`). There is no `gradlew` at the root — invoke Gradle through the example or from a parent project that includes this directory.

- Build the AAR: `./gradlew :opuscodec:assembleRelease` (from any consuming project; the root `settings.gradle` only declares `rootProject.name`, so the library is built by consumers)
- Build + run the example: `cd example && ./gradlew assembleDebug` — note the example uses AGP 3.0.1 and `jcenter()`, so it will not build with current toolchains without an upgrade.
- Native build is wired via `externalNativeBuild { ndkBuild { path "src/main/jni/Android.mk" } }`. The Android Gradle Plugin invokes `ndk-build` automatically — do not run `ndk-build` directly.
- Publishing: JitPack picks up tags and runs with JDK 17 (`jitpack.yml`). The `singleVariant("release")` block exposes the release AAR (with sources jar) as `com.github.chrisfraser:opuscodec:<tag>`.
- **Build host requirement**: Google does not ship a Linux/aarch64 NDK toolchain (only `linux-x86_64`, `darwin-x86_64`, `darwin-arm64`, `windows-x86_64`). Building inside a Linux/aarch64 container — e.g. Docker on Apple Silicon — fails at native compile with `ERROR: Unknown host CPU architecture: aarch64`. JitPack (x86_64 Linux) is fine. Java-only tasks (`compileReleaseJavaWithJavac`) work everywhere.

## Architecture

### Java ↔ native boundary

- Public API lives in [src/main/java/com/score/rahasak/utils/](src/main/java/com/score/rahasak/utils/): [OpusEncoder.java](src/main/java/com/score/rahasak/utils/OpusEncoder.java), [OpusDecoder.java](src/main/java/com/score/rahasak/utils/OpusDecoder.java), [OpusError.java](src/main/java/com/score/rahasak/utils/OpusError.java), [Annotations.java](src/main/java/com/score/rahasak/utils/Annotations.java).
- Each Java class holds a `long address` field that stores a pointer to a heap-allocated native `OpusEncoder`/`OpusDecoder` struct. The JNI code in [src/main/jni/com_score_rahasak_utils_OpusEncoder.c](src/main/jni/com_score_rahasak_utils_OpusEncoder.c) / [OpusDecoder.c](src/main/jni/com_score_rahasak_utils_OpusDecoder.c) reads/writes that field via JNI reflection. If you rename the field, the JNI code's `GetFieldID(..., "address", "J")` calls must change in lockstep.
- All native methods return a negative `int` on Opus error; [OpusError.throwIfError](src/main/java/com/score/rahasak/utils/OpusError.java) converts those into thrown `OpusError` runtime exceptions. This is the project's defining behavior change vs. the upstream `martoreto/opuscodec` (which returned raw error codes).
- The shared library is loaded as `System.loadLibrary("senz")` and the `LOCAL_MODULE` in [Android.mk](src/main/jni/Android.mk) is `senz`. The `senz`/`rahasak` naming comes from the original upstream project — do not rename either side in isolation; the loader will fail.

### Native build (Android.mk)

The makefile glues three things together:

1. The vendored Opus tree at `src/main/jni/opus/` — its source list is pulled in via `silk_sources.mk`, `celt_sources.mk`, `opus_sources.mk`.
2. Architecture-specific feature flags. The per-ABI blocks toggle different `-DOPUS_*` defines and which source variants get appended:
   - `armeabi-v7a`: full ARM inline ASM + NEON intrinsics + EDSP/MEDIA.
   - `arm64-v8a`: **NEON intrinsics only** — AArch64 inline assembly syntax differs from ARM32, so the inline-ASM defines are deliberately omitted. Do not copy the v7a flags here.
   - `x86` / `x86_64`: SSE/SSE2/SSE4.1/AVX with `OPUS_HAVE_RTCD` and `CPU_INFO_BY_C`.
3. The two JNI shim `.c` files.

`LOCAL_LDFLAGS := -Wl,-z,max-page-size=16384` is the **Android 15+ 16KB page-size compatibility flag**. Keep it — removing it will break loading on devices with 16KB pages.

### minSdk 19 + NDK r26 reconciliation

`minSdk 19` (KitKat) is an explicit project constraint, but NDK r26 dropped API 19 sysroots. Two settings keep both intents alive simultaneously:

- [gradle.properties](gradle.properties): `android.ndk.suppressMinSdkVersionError=21` silences the AGP-side configuration error.
- [build.gradle](build.gradle) inside `defaultConfig.externalNativeBuild.ndkBuild`: `arguments "APP_PLATFORM=android-21"` actually tells `ndk-build` to compile against the API 21 sysroot (without this, ndk-build still gets `APP_PLATFORM=android-19` and bails because the sysroot doesn't exist).

Net effect: the published manifest declares `minSdk 19`, but native code is built against API 21 headers. This is only safe because Opus and the JNI shims only touch ANSI C + pthreads, all of which existed in API 19's bionic. Adding any new bionic dependency would break this assumption — re-check before introducing one.

`abiFilters` in [build.gradle](build.gradle) and the per-ABI blocks in `Android.mk` must stay in sync. Currently both cover `armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64`.

### Example app

[example/](example/) is intentionally a standalone Gradle build (its own `settings.gradle` points back at `..` for `:opuscodec`). It is not part of the library's build and is not published. Treat it as documentation: [example/app/src/main/java/com/github/martoreto/opuscodecexample/MainActivity.java](example/app/src/main/java/com/github/martoreto/opuscodecexample/MainActivity.java) shows the encode→decode round-trip against `AudioRecord`/`AudioTrack`.

## Conventions worth knowing

- Java only (no Kotlin); uses `androidx.annotation` for `@IntDef` value constraints on sampling rate (8/12/16/24/48 kHz) and channels (1 or 2).
- `compileSdk` 33, `minSdk` 19, AGP 7.4.2, JDK 17 (per `jitpack.yml`).
- ProGuard rule `[proguard-rules.pro](proguard-rules.pro)` keeps private members of `com.score.rahasak.utils.*` — required because JNI reads the private `address` field reflectively.
- The `groupId` is `com.github.chrisfraser` (fork), not `com.github.martoreto` or `com.github.spacen`. Don't "fix" this back to an upstream id.
