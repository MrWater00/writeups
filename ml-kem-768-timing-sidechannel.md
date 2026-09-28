# Timing Side-Channel Analysis of ML-KEM-768 Decapsulation in liboqs

**Target:** `OQS_KEM_decaps()`, ML-KEM-768, liboqs 0.16.0 (commit `5a1a854b`, 2026-07-09 release)
**Author:** Aman Singh
**Methods:** Bracketed statistical timing analysis + Valgrind-based static leak detection

---

## 1. Summary

This investigation tested whether ML-KEM-768 decapsulation in liboqs leaks secret-key information through execution timing. Two independent methods were used — a bracketed statistical timing harness against the compiled library, and a Valgrind-based static analysis of secret-dependent control flow — against two separate code paths (the AVX2-optimized backend and the plain-C reference backend).

**Result:** No timing or control-flow dependency on secret data was detected in either code path, by either method, within the sensitivity each method could support. Full detail, including two harness bugs found and corrected along the way, is below.

---

## 2. Scope and test environment

| | |
|---|---|
| Library | liboqs 0.16.0, commit `5a1a854b` (2026-07-09 release) |
| Algorithm | ML-KEM-768 (`OQS_KEM_alg_ml_kem_768`), decapsulation only |
| Compiler | gcc 15.2.0 (Debian) |
| OS | Kali Linux, kernel 6.19, glibc 2.43-4 |
| CPU | Intel Core i3-8130U @ 2.20 GHz, 2 cores, 1 thread/core |
| CPU features | AVX2, BMI2, POPCNT present; AVX-512 absent |
| Virtualization | VirtualBox (`systemd-detect-virt` → `oracle`) |
| Timing clock | TSC (`constant_tsc`, `nonstop_tsc`, `rdtscp` present; clocksource = `tsc`) |

Two builds of the library were tested against the same source commit:

1. **AVX2 backend** — the library's default runtime-dispatched build (`OQS_DIST_BUILD=ON`), which on this CPU selects the hand-optimized x86_64 assembly implementation of ML-KEM (confirmed via the CPU-feature dispatch check in `kem_ml_kem_768.c`).
2. **Plain-C backend** — a second build with `OQS_DIST_BUILD=OFF`, `OQS_OPT_TARGET=generic`, `OQS_USE_AVX2_INSTRUCTIONS=OFF`, forcing the portable C reference implementation. Confirmed via `oqsconfig.h` that the x86_64 variant was not compiled in.

Both builds used `CMAKE_BUILD_TYPE=Release`.

**Note on source drift:** initial code-reading (to understand the constant-time design — value barriers, masking, branchless comparison, the FO-transform structure) was done against a separate, newer clone of the repository (commit `f85f9d4`, a later point on `main`) than the commit actually compiled and tested (`5a1a854b`, the 0.16.0 release tag). The two commits differ in file layout for the ML-KEM implementation. The security concepts read carry over, but exact line numbers may not match between what was read and what was measured. This is recorded as a limitation (§6).

---

## 3. Method 1: Statistical timing analysis

### 3.1 Design

A C harness linked directly against the compiled library called `OQS_KEM_decaps()` up to 1,000,000 times per run, timing each call with `CLOCK_MONOTONIC_RAW` (nanosecond resolution, immune to NTP adjustment). Each call was randomly assigned to one of two classes (e.g. "valid ciphertext" vs "invalid ciphertext"), and the two classes' timing distributions were compared with Welch's t-test — the same test used by the established `dudect` methodology. Following `dudect` convention, `|t| > 4.5` was treated as the alarm threshold for a detected difference.

To reduce the effect of OS/hypervisor interrupt noise, each run reports the t-statistic at four crop levels (using 100%, 99%, 90%, and 75% of the fastest samples), since a small number of extreme outliers can otherwise dominate the raw mean. The largest `|t|` across all four crop levels was treated as the result for that run. All timing runs were pinned to a single CPU core (`taskset -c 1`) to avoid cross-core scheduling noise.

### 3.2 Positive control (critical to the method)

Every real test was bracketed by a **positive control**: an identical harness run with a small, known artificial delay (an empty busy-loop of a fixed iteration count) injected into one class. This served two purposes:

- It proved the harness could actually detect a difference of a given size *in that specific session*.
- Because VM timing sensitivity varied significantly between sessions (observed floor ranged from roughly 1 ns to 50+ ns depending on host load and thermal state), no "no difference detected" result was treated as meaningful unless a control of comparable or smaller size was caught immediately before and after it, in the same session.

A **null control** (both classes drawn from statistically identical input, only the random labels differing) was run alongside every batch to confirm the harness itself introduced no artificial signal.

### 3.3 Harness bugs found and corrected

Two flaws were found in the harness itself during calibration — both caught by the null control failing when it shouldn't have, and both are documented here because finding and correcting them is part of the result, not incidental noise.

**Bug 1 — label-dependent branch.** The first version selected each sample's input with an `if` statement inside the timed region. This introduced a small, consistent branch-prediction asymmetry between the two classes, causing the null control to falsely alarm on cropped data (while passing on uncropped data — the tell that something subtle was wrong). Fixed by precomputing all inputs and labels *before* the timing loop begins, so the loop body is identical for both classes.

**Bug 2 — asymmetric cache footprint.** A later test comparing one fixed, reused ciphertext against ciphertexts drawn from a large pool of random ones showed a small (~7 ns) but suspicious timing gap. The fixed ciphertext stayed "hot" in CPU cache across all calls, while the pooled ones were pulled from cold memory each time — a memory-access artifact unrelated to the algorithm. Fixed by drawing *both* classes from equally sized (1024-entry), equally "cold" pools, so cache footprint is symmetric between classes.

### 3.4 Timing results

All results below are from bracketed batches: reported only where a positive control run in the same session, close in time, was detected.

| Backend | Test | Bracket floor | Result |
|---|---|---|---|
| AVX2 (optimized) | valid vs. random-invalid ciphertext | ~8 ns | No difference detected (max cropped gap 0.3 ns, max \|t\| 1.72) |
| AVX2 (optimized) | fixed vs. random-invalid ciphertext | ~1–8 ns (session-dependent) | No difference detected above the session floor; **one unreplicated ~0.3–0.8 ns hint** in a single seed did not reproduce in follow-up seeds (see §6) |
| Plain-C | valid vs. random-invalid ciphertext | ~8 ns | No difference detected (max cropped gap 0.4 ns, max \|t\| 1.72) |
| Plain-C | fixed vs. random-invalid ciphertext | ~8 ns | No difference detected (max cropped gap 2.6 ns, max \|t\| 1.16) |

Null controls across all sessions remained quiet (max \|t\| well under the 4.5 alarm threshold in every batch used for a final result).

**Interpretation:** across roughly 1–8 million total decapsulation calls per configuration, no consistent timing difference was detected between an accepted ciphertext and a rejected one, nor between a fixed ciphertext and randomly varying ones, on either the assembly-optimized or the portable-C code path. The claim is bounded by measurement floor: this rules out a leak larger than approximately 8 ns (roughly 15–20 CPU cycles) with reasonable confidence; it does not rule out a smaller one.

---

## 4. Method 2: Static leak detection with Valgrind

### 4.1 Design

Timing measurement has a noise floor. As a complementary, noise-free method, Valgrind's Memcheck tool was repurposed as a static constant-time checker: the secret key buffer was marked "undefined" via `VALGRIND_MAKE_MEM_UNDEFINED` before each call, causing Valgrind to flag *any* instruction whose outcome (a branch taken, or a memory address computed) depends on that data — regardless of whether the resulting timing difference would be measurable. A companion "defined" call restored the marking afterward to avoid unrelated noise in cleanup code.

liboqs ships a built-in constant-time test option (`OQS_ENABLE_TEST_CONSTANT_TIME`), but it refuses to activate on non-Debug builds, and rebuilding as Debug would have meant testing different compiled binaries than the ones already timing-tested. A standalone harness was written instead, run against the exact Release binaries already used in §3.

A positive control (a function deliberately branching on the secret byte-by-byte) was included in every run to confirm Valgrind's detection was actually active.

### 4.2 Harness bug found and corrected

The first run produced roughly 9,000–10,000 warnings, almost all tracing through the matrix-generation routine (`gen_matrix` / `poly_rej_uniform`). Investigation of ML-KEM-768's secret-key byte layout (from `params.h`) explained why:

| Offset | Length | Contents | Secrecy |
|---|---|---|---|
| 0 | 1152 B | secret vector `s` | secret |
| 1152 | 1184 B | embedded copy of the public key | **public** |
| 2336 | 32 B | hash of the public key | public |
| 2368 | 32 B | implicit-rejection value `z` | secret |

(1152 + 1184 + 32 + 32 = 2400 bytes, matching the library's reported secret-key size.)

ML-KEM's secret key format embeds a full copy of the public key (required by the FO-transform's re-encryption check). Marking the *entire* 2400-byte buffer undefined also poisoned the embedded public-key region, and matrix generation legitimately reads the public seed contained there — producing a large volume of correct-but-irrelevant warnings, not evidence of a real leak.

**Fix:** the check was narrowed to mark only the two genuinely secret regions (`s` at offset 0, 1152 bytes; `z` at offset 2368, 32 bytes) as undefined, leaving the embedded public-key copy and its hash untouched.

### 4.3 Static analysis results

After the fix, on both the AVX2 and plain-C builds:

- **Zero warnings** originated from inside `OQS_KEM_decaps()`, for either a valid or an invalid ciphertext.
- The positive control correctly produced **exactly 1152 warnings** — one per byte of the secret vector it was deliberately made to branch on — confirming the detection mechanism itself was working correctly throughout.

**Interpretation:** no branch or secret-dependent memory address was found anywhere in the compiled decapsulation routine, on either code path, for the specific inputs tested (one valid ciphertext, one random invalid ciphertext, single key pair). This is a deterministic result with no sample-size or noise caveat, but it inherits the coverage limits described below.

---

## 5. Combined result

| | Timing analysis (dynamic) | Valgrind analysis (static) |
|---|---|---|
| AVX2, accept vs. reject | no diff. > ~8 ns | no secret-dependent branch/address found |
| Plain-C, accept vs. reject | no diff. > ~8 ns | no secret-dependent branch/address found |
| Value-dependence (fixed vs. random ct) | no diff. > session floor | not covered by this method |

Two methods with different, non-overlapping blind spots agree: no timing side channel was found in ML-KEM-768 decapsulation under the conditions tested.

---

## 6. Limitations

- **Virtualized environment.** All timing was performed inside a VirtualBox VM, not bare metal. The measurement floor varied noticeably between sessions (observed range roughly 1–50 ns), which is the reason every claim above is bounded to "~8 ns" rather than a single fixed number. A sub-8 ns leak cannot be excluded by the timing method alone.
- **One unreplicated hint.** A single seed of the "fixed vs. random-invalid" AVX2 test showed a ~0.3–0.8 ns gap that alarmed under an unusually quiet session. It did not reproduce across three follow-up seeds, and those follow-up controls themselves failed to alarm (the session had become noisier), so the replication attempt was inconclusive rather than negative. This remains an open item.
- **Source-commit drift.** Conceptual code review was performed against a newer `main`-branch commit than the one actually compiled and tested; see §2.
- **Valgrind's blind spot.** This method detects secret-dependent *branches* and *memory addresses* only. It cannot detect a genuine timing leak caused by a variable-latency-but-branchless instruction (e.g. hardware integer division whose latency depends on its inputs) — the mechanism behind the real-world "KyberSlash" class of bug. The dynamic timing method in §3 is the relevant check for that class, within its own precision limits.
- **Narrow input coverage.** Both methods tested one key pair and a small, fixed set of ciphertext categories (one valid, and either random or pooled-random invalid ones). Neither exhaustively covers the input space.
- **Single toolchain.** Only gcc 15.2.0 on one CPU model was tested. Results may not generalize to other compilers or microarchitectures.
- **Algorithm coverage.** Only ML-KEM-768 decapsulation was tested; ML-KEM-512, ML-KEM-1024, and encapsulation were not.

---

## 7. Suggested follow-up work

1. Repeat the timing analysis on bare metal to lower the noise floor and tighten the ~8 ns bound.
2. Attempt to replicate or rule out the unreplicated sub-nanosecond hint under controlled, quiet conditions.
3. Rebuild and re-test against the exact newer `main` commit that was originally read, to close the source-drift gap.
4. Extend both methods to ML-KEM-512 and ML-KEM-1024.
5. Repeat on a second compiler (e.g. Clang) and a second CPU microarchitecture.
