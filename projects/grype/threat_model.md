# Threat model

## What this project does and where untrusted input enters
Grype finds known vulnerabilities in container images, filesystems and SBOMs by matching packages against a vulnerability database. It ships as a CLI (`cmd/grype`) and as a Go library (`grype/`). Grype embeds Syft to catalog packages, so every untrusted input Syft handles (image layers, archives, package metadata, binaries) reaches Grype too. Bugs in that cataloging belong to Syft, unless Grype's own use of the results is the problem.

Untrusted input:
- The artifact being scanned (images, `dir:`, `file:`), and the packages and versions Syft reports from it.
- SBOMs passed in with `sbom:` or on stdin (syft-json, SPDX, CycloneDX), and packages given directly as purls (`pkg:...`, or `purl:<file>`) or CPEs (`cpe:...`).
- VEX documents (`grype/vex/`, OpenVEX and CSAF) given with `--vex`.
- The vulnerability DB. The listing (`latest.json`) and archive are fetched over HTTPS, and archives can also come from `grype db import FILE|URL`. An archive is untrusted until its checksum is verified, and its extraction (via go-getter) handles untrusted paths.

Trusted input: CLI flags, config files and environment variables (including registry credentials), output templates (`-o template`), and a DB that has passed checksum verification. These come from the operator or from Anchore's signed publishing pipeline.

## Nothing from the scanned content is executed
Like Syft, Grype never runs anything from the content it scans. Packages come from Syft's parsers and from SBOM, purl and VEX documents, all parsed directly. Any path where scanning or matching executes something from the scanned content or from an SBOM is a risk we want reported, even without a working exploit. That includes execution inside a dependency that Grype hands untrusted content to. Execution during Syft's cataloging belongs in Syft's threat model, but it reaches Grype too.

Running tools from the host as part of Grype's own operation does not count. Examples: talking to the docker, podman or containerd daemons to fetch an image, or the `zstd` binary the DB build tooling calls (which is out of scope anyway).

## Components that matter most / least
Most:
- DB distribution and install (`grype/db/v6/distribution/`, `grype/db/v6/installation/`, `grype/db/internal/tarutil/`): checksum verification, temp-dir extraction, and replacing the active DB. Accepting an unverified archive, or writing outside the DB directory, would be serious.
- Ingestion of SBOMs, purls and CPEs (`grype/pkg/`), and of VEX (`grype/vex/`).
- Version parsing and comparison (`grype/version/`). It runs on attacker-controlled version strings for every ecosystem, so watch for panics and pathological regex or backtracking.
- Secret redaction in output (`internal/redact/`, the presenters under `grype/presenter/`). Registry credentials must never reach reports. Past advisory: [GHSA-6gxw-85q2-q646](https://github.com/anchore/grype/security/advisories/GHSA-6gxw-85q2-q646) (registry credentials included in JSON written with `--file` or `-o json=<file>`).

Also in scope: matchers (`grype/matcher/`), and output that untrusted package data flows into (JSON, SARIF, CycloneDX, table, template), for example injection that would affect whoever consumes the report.

Out of scope:
- `test/`, including the quality gate under `test/quality/`, plus `.make/` and `.github/`.
- The DB build pipeline (`grype/db/v6/build/`, `grype/db/processors/`, `grype/db/provider/`). It only runs inside Anchore's publishing pipeline, against vunnel output.
- The accuracy of vulnerability data itself. Report false positives or negatives as issues, not security reports.

## How you rate severity
Malicious input that makes Grype miss or misreport vulnerabilities (tampered package metadata, packages hidden from Syft) is **not** a vulnerability. Detecting purposely malicious content is out of scope.

The trust boundary is the system running Grype:
- **critical**: code execution while scanning or while updating or importing the DB, including execution of anything from the scanned content (see above). Grype accepting a DB archive that fails checksum verification.
- **high**: reading or writing files outside the intended directory (path traversal while extracting the DB or an artifact). Leaking registry credentials or other secrets into reports or other output that may be shared.
- **medium**: resource exhaustion driven by input, such as huge allocations, temp space never released, or a corrupted DB left behind that breaks later runs. Output injection that affects consumers of a report. Secrets leaked only into local debug logs.
- **low**: a crash or hang on malformed input that ends with the process and leaves nothing behind.

Reports should include the input that triggers the issue (or a script that builds it) and the Grype version or commit.

## How to exercise it
- The CLI is built at `/usr/local/bin/grype`, with a current vulnerability DB already installed, so matching works offline. Image references like `grype alpine:latest` need a registry or container daemon, and neither exists here. Scan the image archives saved under `/images` instead (`grype docker-archive:/images/alpine.tar`, `grype docker-archive:/images/debian.tar`).
- Other inputs work directly: directories with `grype dir:/path`, SBOMs with `grype sbom:/path/to/sbom.json`, a single package with `grype pkg:npm/lodash@4.17.20`, or a file of purls with `grype purl:/path/to/purls.txt`.
- `grype db status` shows the installed DB. `grype db import /path/to/archive.tar.zst` exercises the import and extraction path.
- Unit tests: `go test ./grype/... ./internal/...`. The build tags and the two skips this sandbox needs are already set in `GOFLAGS`.
- The suites under `test/` (integration, CLI, install, quality) need docker or network and can't run here.
