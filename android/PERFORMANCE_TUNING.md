# Android performance experiment — LTO and instruction counters

This branch enables a measured optimization experiment for the ARM64 Android
runner. It does not change emulation timing, CPU scheduling, audio timing, or
frame pacing.

## LTO / IPO

`NDS_ANDROID_ENABLE_LTO=ON` is enabled by the Android Gradle build. The runner
uses CMake `CheckIPOSupported` and enables Release interprocedural optimization
for only these targets:

- `nds_banks` — generated ARM9/ARM7 code banks
- `nds_support`
- `nds_armv4t`
- `nds_runner`

SDL2 and unrelated third-party targets are intentionally excluded. If the NDK
toolchain reports IPO/LTO unsupported, CMake warns and continues without LTO.
Android title-bank translation units are compiled with `-O3` (previously `-O2`);
non-Android builds keep their previous `-O2` setting.

## Instruction counter audit

The generated banks increment `g_insn_count[g_nds_active]` for each retired
guest instruction. This is not the old out-of-line per-instruction hook: the
count increment is inline, and the bank build defines `NDS_STATIC_CPU=0` for
ARM9 sources and `NDS_STATIC_CPU=1` for ARM7 sources. In generated banks,
`g_nds_active` is therefore a compile-time constant, so the CPU-array index is
already folded by the compiler.

The exact counters are consumed by savestates, event/breakpoint handling,
trace collection, diagnostics, and the on-screen throughput display. Batching
or sampling these increments would make those consumers stale or inexact, and
can break instruction breakpoints. For that reason this experiment preserves
exact counter semantics rather than silently disabling or sampling them. LTO
and `-O3` are allowed to optimize the surrounding generated code, but the
per-instruction counter store remains intentional.

## Validation

Build the Android Release APK with GitHub Actions, then compare the same ROM,
room, camera position, and test duration against the previous build. Record
FPS, `emu` ms, `pres` ms, and audio behavior. LTO is an experiment, not a
guaranteed FPS increase; retain the baseline APK for A/B comparison.


## Audio stretch default

The adaptive audio stretch path in `frontend.cpp` is now opt-in. With no
`NDS_AUDIO_STRETCH` environment variable, audio is consumed at the normal
1.0 playback rate; set `NDS_AUDIO_STRETCH=1` before process startup to enable
the adaptive interpolation path. This isolates whether the stretch behavior
was causing the reported slow/distorted sound. It does not fix audio queue
underruns; if crackling remains, measure queue underruns separately.
