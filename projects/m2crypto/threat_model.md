# Threat model

## What M2Crypto does and where untrusted input enters

M2Crypto provides Python bindings to OpenSSL for cryptographic operations, X.509
and other ASN.1 formats, S/MIME, and TLS. Applications pass untrusted data to
these APIs, including certificates, public keys, encrypted or signed messages,
and data received from TLS peers. The library also processes local private keys
and other secrets supplied by its callers.

## Components that matter most

The SWIG-generated native extension and its C/OpenSSL interactions are central
to the security boundary. Pay particular attention to input validation, buffer
lengths, object ownership and lifetime, error handling, cryptographic operation
selection, certificate and hostname verification, and TLS defaults. Python
wrappers are in scope where they affect those properties or expose unsafe
native behavior.

## How to exercise it

The Dockerfile builds the extension and runs the unittest suite with `make
check`. Tests are under `tests/`; many use local test certificates and exercise
parsing and cryptographic APIs. Prefer a focused reproducer or regression test
for a finding.

## Severity guidance

Treat memory corruption reachable through public APIs, private-key disclosure,
cryptographic verification bypass, or a TLS identity/authentication bypass as
high severity or above depending on exploitability and attacker reach. Reserve
critical severity for broadly exploitable issues such as remote code execution
or compromise of secrets across applications without unusual preconditions.
Failures requiring a caller to misuse an explicitly unsafe or deprecated API
should include those assumptions when assessing severity.
