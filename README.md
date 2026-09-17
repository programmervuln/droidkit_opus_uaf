# Droidkit‑Opus Heap Use‑After‑Free PoC

## Overview
This repository contains the reproducible proof‑of‑concept for a **heap use‑after‑free vulnerability (CWE‑416)** discovered in the JNI layer of droidkit‑opus open‑source audio library.

The vulnerability arises from **unsynchronized concurrent access to the global `_opusFile` decoder pointer**. One thread may free the decoder context during cleanup, while another thread continues to operate on the stale, already‑freed pointer without any locking or state check, triggering a heap use‑after‑free race condition.

## Root Cause
In `audio.c`, the global decoder handle `_opusFile` is shared across threads with no synchronization primitives.
- One thread calls `cleanupPlayer()` to release and free the `OpusFile` instance.
- A concurrent worker thread invokes playback functions that dereference `_opusFile`.
- No mutex, no state validation: access to already‑freed heap memory occurs.

## PoC Notes
- Original vulnerable `audio.c` is unmodified.
- All multi‑threaded race‑trigger logic is implemented in the standalone harness `test_uaf.c`.
- This PoC requires AddressSanitizer to reliably catch the use‑after‑free crash.

## Compile
```bash
clang -I. -I/usr/lib/jvm/java-11-openjdk-amd64/include -I/usr/lib/jvm/java-11-openjdk-amd64/include/linux \
-fsanitize=address -g -O1 -pthread test_uaf.c -o uaf_test -lopus -lopusfile -logg

rerun reproduction
export ASAN_OPTIONS="detect_leaks=1:abort_on_error=1"
./uaf_test --decoder-uaf

Evidence
ASAN heap‑use‑after‑free crash log
Multi‑threaded race PoC source
Unmodified vulnerable source file
Disclosure
This vulnerability has been submitted for CVE assignment via MITRE CNA‑LR.
Research
Discovered by our custom UAF race‑vulnerability mining framework.
