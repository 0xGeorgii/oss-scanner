# Threat model

## What this project does and where untrusted input enters
Syft generates SBOMs from container images, filesystems, archives and single files. It ships as a CLI (`cmd/syft`) and as a Go library (`syft/`) that other tools embed, Grype among them. Syft is often run in CI pipelines and services against third-party content, so treat everything it is pointed at as untrusted.

Untrusted input:
- The artifact being scanned: image manifests, configs and layer tarballs (via stereoscope), directories, single files and archives (tar, zip, nested java archives, etc.).
- Everything the catalogers parse inside it (`syft/pkg/cataloger/`): lock files and manifests, OS package databases (RPM sqlite/bdb, dpkg status, apk installed), package archives (rpm, deb, apk, wheels, jars), and binaries inspected for embedded metadata (ELF/PE/Mach-O, Go build info, .NET, GraalVM, UPX, kernel modules).
- SBOMs read back in by `syft convert` and by the `sbom` cataloger (`syft/format/` decoders: syft-json, SPDX, CycloneDX).
- Remote responses: registries when pulling images, the OCI model registry client (`syft/source/ocimodelsource/`), and the online enrichment sources reached with `--enrich` (Go module proxy, Maven Central, npm and PyPI license lookups).

Trusted input: CLI flags, config files and environment variables, and user-supplied output templates (`-o template`). These come from the operator.

## Nothing from the scanned content is executed
Syft never runs anything from the content it scans. Every catalog is built by parsing files directly: Syft reads a Go binary's build info rather than running `go version -m`, and opens an RPM database itself rather than calling `rpm`. Any path where cataloging executes something that came from the scanned content (a binary, a script, a plugin, an interpreter given a file from the artifact) is a risk we want reported, even if there is no working exploit yet. This includes execution that happens inside a dependency, such as stereoscope or a parsing library, when Syft hands it scanned content. Dependencies have done this in the past, so look closely at what each cataloger's dependencies do with their input.

This applies to cataloging only. Running tools from the host as part of Syft's own operation does not count: `syft attest` shelling out to the operator's `cosign` binary to attest the finished SBOM, or talking to the docker, podman or containerd daemons to fetch an image. These run software the operator installed, not content from the artifact.

## Components that matter most / least
Most:
- File resolution and source handling (`syft/source/`, `syft/internal/fileresolver/`): symlink and path resolution across image layers and directories, where an escape outside the scanned artifact is possible.
- Archive extraction and temp storage (`internal/file/`, the java archive handling, stereoscope's layer unpacking). Past advisories: [GHSA-rjcw-vg7j-m9rc](https://github.com/anchore/syft/security/advisories/GHSA-rjcw-vg7j-m9rc) (temp files left behind after exhaustion).
- Binary and package parsers that size buffers or loops from header fields. Past advisory: [GHSA-mw2c-m758-9v5q](https://github.com/anchore/syft/security/advisories/GHSA-mw2c-m758-9v5q) (small inputs triggering large allocations in the ELF, UPX, .NET, GraalVM, arch, deb and kernel module parsers). Look for other parsers with the same pattern.
- Secret redaction (`internal/redact/`, `cmd/syft/internal/options/secret.go`): registry credentials and attestation passwords must never reach logs or SBOM output. Past advisory: [GHSA-jp7v-3587-2956](https://github.com/anchore/syft/security/advisories/GHSA-jp7v-3587-2956).

Also in scope: SBOM encoders and decoders (`syft/format/`), the cache under `internal/cache/`, and the enrichment HTTP clients.

Out of scope: `test/`, `examples/`, `.make/`, `task.d/`, `.github/`, and release tooling.

## How you rate severity
Syft is not designed to detect purposely malicious content. Malicious input that makes Syft miss, skip or misreport packages is **not** a vulnerability. Examples: an altered lock file or package database, an archive Syft skips that a runtime would not, a self-modifying system.

The trust boundary is the system running Syft. Anything that harms that system, or makes Syft dangerous to itself, the system or the operator, is in scope:
- **critical**: code execution while scanning an artifact, including execution of anything from the scanned content during cataloging (see above).
- **high**: reading or writing files outside the scanned artifact or directory (path traversal, symlink escapes during extraction or resolution). Leaking credentials or secrets from the environment or config into SBOMs, attestations or other output that may be published.
- **medium**: resource exhaustion driven by input, such as a small file causing a huge allocation, a crash that skips cleanup, or temp space filled and never released. Secrets leaked only into local debug logs.
- **low**: a crash or hang on malformed input that ends with the process and leaves nothing behind.

Reports should include the input that triggers the issue (or a script that builds it) and the Syft version or commit.

## How to exercise it
- The CLI is built at `/usr/local/bin/syft`. Image references like `syft alpine:latest` need a registry or container daemon, and neither exists here. Scan the image archives saved under `/images` instead (`syft docker-archive:/images/alpine.tar`, `syft docker-archive:/images/debian.tar`), or build your own archive of a crafted image.
- Directories and single files work directly: `syft dir:/path`, `syft file:/path`.
- Unit tests: `go test ./syft/... ./internal/...`. The build tags and the two skips this sandbox needs are already set in `GOFLAGS`.
- Parser fixtures live in each cataloger's `testdata/`, and fuzz targets are in `syft/pkg/cataloger/parsers_fuzz_test.go`.
- The suites under `test/` (integration, CLI, install) need docker and can't run here.
