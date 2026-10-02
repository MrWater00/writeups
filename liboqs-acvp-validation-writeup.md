# Formally validating liboqs's ML-KEM against NIST's FIPS 203 via ACVP

Aman Singh · October 2026 · github.com/MrWater00

## In plain words

There are two different questions you can ask about a cryptography library:

1. Does it leak secrets through timing or other side channels?
2. Does its math actually produce the correct output, exactly as the government standard defines it?

My earlier work on liboqs answered the first question, using timing analysis and a patched Valgrind to look for secret-dependent instruction timing (including reproducing the KyberSlash class of bug on purpose, to prove the method works).

This note answers the second question: I ran liboqs's ML-KEM implementation through NIST's own official correctness test suite and got a passing result, independently confirmed by NIST's server.

## Background: CAVP and ACVP

NIST's Cryptographic Algorithm Validation Program (CAVP) is how cryptographic implementations get formally certified for US government use. The actual testing mechanism behind CAVP is the Automated Cryptographic Validation Protocol (ACVP): a client authenticates to NIST's server, requests test vectors for a given algorithm, computes the expected outputs using the implementation under test, and submits the answers back for grading.

NIST runs a public Demo environment separate from the Production environment used for real certificates. Demo access is free and open to anyone who requests it by email; Production access is restricted to accredited testing laboratories. This work used the Demo environment, so the result is a correctness validation, not an official government certification — but it uses the same protocol, the same server-side grading, and the same official test vectors.

## Setup

- Client: Cisco's libacvp (v2.3.1), the reference open-source ACVP client, which ships a built-in test harness specifically for liboqs
- Library under test: liboqs 0.16.0, built fresh as a static library with both ML-KEM and ML-DSA enabled
- Authentication: a client TLS certificate and private key issued by NIST after a CSR exchange, plus a TOTP (time-based one-time password) seed for two-factor login
- Server: demo.acvts.nist.gov

## Getting the pieces to talk to each other

Two real integration problems came up building this, worth documenting because they're the kind of thing that isn't in anyone's quickstart guide.

**libacvp's own configure script has an ordering bug.** When liboqs is built as a static library linked against OpenSSL, libacvp's `./configure --with-liboqs-dir=...` test-links against liboqs before it adds the `--with-ssl-dir` OpenSSL flags, so the link test fails with undefined OpenSSL symbols even though the right flags were given. The fix is to export `LIBS="-lcrypto -lssl -lpthread"` before running configure, since the script preserves any pre-existing `LIBS` value into its own link test.

**libacvp's liboqs/ML-DSA harness expects a function naming convention liboqs no longer exports.** The harness calls `pqcrystals_ml_dsa_44_ref_signature_internal` and similar functions — old PQClean-style names. Current liboqs's ML-DSA backend (mldsa-native) does not export any symbol under either that name or its own internal naming scheme; the deterministic sign/verify functions are compiled as library-internal only. This is a genuine version mismatch between the two projects, not a configuration issue. I added a small stub file providing those six symbols (returning an error code, since they're never meant to be called) purely to satisfy the linker, which disables ML-DSA testing in this build while leaving ML-KEM fully functional.

## Result

A full ACVP test session against the Demo server, registering ML-KEM capabilities for all three parameter sets:

| Vector set | Algorithm | Mode | Result |
|---|---|---|---|
| 4071542 | ML-KEM-512 / 768 / 1024 | keyGen | Pass |
| 4071543 | ML-KEM-512 / 768 / 1024 | encapDecap (encapsulation + decapsulation) | Pass |

Test session ID 774197. The pass result was returned twice independently: once at the end of the live test run, and again on a separate `--get_results` query against the server afterward, confirming it wasn't just a local cached status.

In plain terms: liboqs's key generation, encapsulation, and decapsulation for ML-KEM-512, ML-KEM-768, and ML-KEM-1024 all produce output that exactly matches NIST's official FIPS 203 reference answers.

## How this fits with the timing work

This result is a different, complementary claim to the side-channel analysis. ACVP validation says the math is correct. It says nothing about whether computing that correct answer takes a secret-dependent amount of time — a implementation can be perfectly ACVP-compliant and still leak keys through timing, which is exactly what the KyberSlash bug class demonstrated. Treating "formally validated" and "side-channel resistant" as the same property is a common mistake; they're orthogonal, and a serious security review needs both.

## Limits

- Demo environment, not Production — this is a correctness validation exercise, not an official NIST certificate
- ML-DSA is not covered, due to the libacvp/liboqs symbol mismatch described above
- One liboqs version (0.16.0), one build configuration (static, OpenSSL backend)

## Reproducing it

1. Request ACVTS Demo credentials by emailing acvts-demo@nist.gov, generate a CSR, and exchange it through NIST's Secure File Collaboration portal to receive a signed certificate and TOTP seed.
2. Build libacvp against a full liboqs install, exporting `LIBS="-lcrypto -lssl -lpthread"` before `./configure --with-liboqs-dir=<path> --with-libcurl-dir=/usr` to work around the linking order issue.
3. Set `ACV_SERVER`, `ACV_CERT_FILE`, `ACV_KEY_FILE`, and `ACV_TOTP_SEED` (base64, read directly from the seed file) as environment variables.
4. Run `acvp_app --ml_kem --sample` for a first test session that returns both the vectors and the correct answers together.

## Next steps

Extend this to a second library (PQClean's ML-KEM, say) to see whether ACVP validation and my own side-channel method agree or disagree on which implementations are solid. Revisit ML-DSA once liboqs exposes the internal sign/verify entry points libacvp's harness expects, or once libacvp updates its harness to match current liboqs.
