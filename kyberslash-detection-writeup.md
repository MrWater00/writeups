# Catching a KyberSlash-class bug in liboqs ML-KEM-768 with a patched Valgrind

Aman Singh · September 2026 · github.com/MrWater00

## In plain words

Some CPU instructions take a different amount of time depending on the numbers they are given. Division is the classic one. If a program divides by, or divides something derived from, a secret, then how long the division takes can leak the secret. KyberSlash (2023-24) was exactly this: a plain division in Kyber/ML-KEM code let researchers recover secret keys from timing.

This note shows that a patched Valgrind can catch this bug class in a real library, and that it stays quiet on fixed code. I reproduced KyberSlash1 on purpose in a private copy of liboqs, compared it against the unmodified code under identical compiler flags, and the tool flagged the bug in the same function where it was originally found.

## Background

KyberSlash1 was first noticed by Tamvada, Bhargavan and Kiefer, and independently by Bernstein. KyberSlash2 was found by Ravi and Kannwischer. The full paper is Bernstein et al., "KyberSlash: Exploiting secret-dependent division timings in Kyber implementations", IACR TCHES 2025, issue 2, pp. 209-234 (doi 10.46586/tches.v2025.i2.209-234).

The authors also published a patch that adds a check to Valgrind's Memcheck tool. Stock Memcheck already flags secret data used in branches and memory addresses. The patch adds a third check: a secret value used as an operand of a variable-latency instruction, mainly integer and floating-point division, plus square root and floating-point remainder. It is switched on with `--variable-latency-errors=yes`.

## Setup

- OS and hardware: Kali Linux in VirtualBox, Intel i3-8130U
- Compiler: gcc 15.2.0
- Library: liboqs 0.16.0 release (commit 5a1a854b), ML-KEM-768 from the vendored mlkem-native code
- Valgrind: git commit 112f1080b (2025-08-05) with the KyberSlash option patch and division patch dated 20250805, built into its own prefix
- Builds tested: plain C at `-O3` (Release), AVX2 with runtime dispatch, and plain C at `-Os` (MinSizeRel)

## Method

The harness is a small C program linked against liboqs. It generates a key pair, then marks only the genuinely secret parts of the 2400-byte secret key as undefined, which Valgrind treats as "secret": the secret vector `s` (bytes 0-1151) and the value `z` (bytes 2368-2399). It then runs decapsulation on a valid and on a random invalid ciphertext.

Two deliberate positive controls run in the same program: a loop that branches on secret bytes, and a loop that divides by a value derived from secret bytes. A quiet result only counts if these controls fire.

Marking only `s` and `z` matters. An earlier version marked the whole secret key, which also tainted the public-key copy stored inside it, and produced a large number of warnings from matrix generation plus one unexplained warning inside decapsulation. With the narrower marking, all of those disappeared and only the deliberate branch control remained, so they were a harness artifact and not leaks.

## The compiler finding

The KyberSlash1 rounding step is `((u*2 + Q/2) / Q) % 2` with Q = 3329. The divisor is a constant, so compilers often replace the division with a multiply. I compiled that one line with gcc 15.2 at five optimization levels and counted real `div` instructions in the object code:

| Flag | div instructions |
|---|---|
| -O0 | 0 |
| -O1 | 0 |
| -O2 | 0 |
| -Os | 1 |
| -O3 | 0 |

The same source line is safe or vulnerable depending on one compiler flag. Correct maths, constant-time behaviour and compiler output are three separate properties, so the binary that ships is the thing to test. This also explains why the original researchers saw the bug in `-Os` builds.

## Results

| Build | Source | Real div instructions | Variable-latency warnings inside liboqs |
|---|---|---|---|
| Plain C, -O3 | unmodified | 0 | 0 |
| AVX2, dispatch build | unmodified | 0 | 0 |
| Plain C, -Os | unmodified | 0 | 0 |
| Plain C, -Os | KyberSlash1 division put back | 1 | 2 |

In every run the deliberate division control was flagged, so the tool was demonstrably active.

For the reintroduced bug, the two warnings are:

```
Variable-latency instruction operand of size 8 is secret/uninitialised
   at PQCP_MLKEM_NATIVE_MLKEM768_C_poly_tomsg
   by PQCP_MLKEM_NATIVE_MLKEM768_C_indcpa_dec
   by PQCP_MLKEM_NATIVE_MLKEM768_C_dec
   by main
```

They appear once for the valid-ciphertext decapsulation and once for the invalid one, because they come from different call sites. `poly_tomsg` inside decryption is where KyberSlash1 was originally found. The tool located it without being told where to look.

The change was one edit in `mlk_scalar_compress_d1`: the multiply-and-shift code (magic constant 1290168) was replaced by

```c
return (uint8_t)((((uint32_t)u * 2 + MLKEM_Q / 2) / MLKEM_Q) & 1);
```

Everything else, including compiler and flags, was held identical between the fixed and bug builds.

## Limits

- The tool flags a variable-latency instruction operating on secret data. It does not prove the timing difference is exploitable. That depends on the CPU, since some processors divide in near-constant time.
- One compiler version, one parameter set (ML-KEM-768), one machine.
- The bug reintroduction was done only on the plain-C `-Os` build. The AVX2 build was checked in its normal state only.
- Only `s` and `z` are treated as secret. Other secret-derived values are covered only through taint propagation.
- I did not run a timing measurement on the bug build, so this note shows detection, not measured leakage.

## Reproducing it

1. Download the two patch files from kyberslash.cr.yp.to/papers.html (version 20250805).
2. Clone Valgrind, check out the last commit before 2025-08-06, apply the option patch and then the division patch with `git apply`, and build with `--prefix` set to a private folder.
3. Sanity test: a program that divides by a value from `malloc`'d memory should be silent normally and flagged with `--variable-latency-errors=yes`.
4. Build liboqs at the 0.16.0 commit with `-DCMAKE_BUILD_TYPE=MinSizeRel -DOQS_DIST_BUILD=OFF -DOQS_OPT_TARGET=generic -DOQS_MINIMAL_BUILD="KEM_ml_kem_768" -DOQS_BUILD_ONLY_LIB=ON -DBUILD_SHARED_LIBS=OFF`, once unmodified and once with the division put back.
5. Link the harness against each build and run it under the patched Valgrind with `-q --variable-latency-errors=yes`.

## Next steps

Run the same A/B test on ML-KEM-512 and 1024 and on ML-DSA signing, try a second library, and add a timing measurement of the reintroduced bug on hardware where the division is known to be variable-latency.
