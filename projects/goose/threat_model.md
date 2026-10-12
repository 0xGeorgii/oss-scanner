# Goose threat model

## Project and trust boundaries

Goose is an open-source AI agent framework with a Rust CLI, an Electron desktop
application, and reusable SDK crates. It connects model providers and MCP
extensions to tools that can read and write files, execute commands, and interact
with other systems on the user's behalf.

Treat model responses, tool responses, remote web content, repository files,
imported recipes, session data, and incoming ACP/MCP messages as potentially
untrusted. Provider credentials, local configuration, session history, and access
to the user's machine are sensitive assets. Trace how untrusted data crosses
authentication, permission, process, filesystem, and Electron IPC boundaries.

Goose deliberately executes tools and commands authorized by the user. Its
SECURITY.md documents the inherent risk of prompt injection and recommends
isolated environments and human confirmation for significant actions. Merely
showing that a model can follow instructions embedded in content is not a new
implementation vulnerability. Concrete bypasses of implemented permission checks,
authentication, sandbox boundaries, or isolation remain important findings.

## Areas to prioritize

- Rust agent orchestration, tool inspection and permission enforcement, recipes,
  sessions, scheduling, subprocess management, and configuration in crates/goose.
- CLI and ACP transports in crates/goose-cli; provider authentication and
  credential handling in crates/goose-providers.
- Built-in MCP tools in crates/goose-mcp, especially command execution, file
  operations, path handling, and processing of attacker-controlled documents.
- Electron main/preload/renderer boundaries, IPC validation, navigation and URL
  handling, extension content, and update/install flows in ui/desktop.
- SDK and protocol crates under crates/, including parsing and validation of
  messages received from less trusted peers.

Documentation, examples, and test fixtures may help explain intended behavior.
Prioritize reachable production vulnerabilities over hypothetical misuse of
sample code. Third-party dependency findings should demonstrate a reachable Goose
impact and distinguish an upstream defect from Goose's integration.

## Build and reproduction

The Dockerfile retains the checkout at /src, the Rust toolchain selected by
rust-toolchain.toml, Cargo's dependency cache, compiled workspace binaries and
tests in target/debug, Node.js 24, pnpm, UI dependencies, and a packaged Linux
desktop application under ui/desktop/out. Rust tests are compiled during the
online setup so offline analysis can use their executables directly.

Useful commands from /src in the offline scanner environment:

```sh
cargo test --locked --offline --workspace --test git_command_security
cargo test --locked --offline --workspace --test schedule_tool_security
cargo test --locked --offline --workspace --test tool_inspection_permission_precedence
cargo test --locked --offline --workspace --test acp_transport_auth_test
pnpm --dir ui/desktop run test:run
```

GOOSE_DISABLE_KEYRING=1 is set to avoid system keyring prompts. Tests that require
live model providers, Internet services, downloaded models, or externally
installed MCP servers cannot run unchanged offline. Use local mock providers,
local protocol peers, fixtures, and narrowly targeted tests for reproducers.
For desktop reproductions that need a display, xvfb-run is available.

## Reports and severity

Explain the attacker's starting privileges, the input they control, the boundary
crossed, and the resulting impact. Include a deterministic reproducer using local
resources, the affected code path, and a focused proposed patch with a regression
test when possible. Group reports with the same root cause.

Prioritize unauthorized code execution, credential or private-data disclosure,
authentication or permission bypasses, and arbitrary file access across an
intended boundary. Assess severity using exploitability, required interaction,
deployment assumptions, and the privileges gained. Distinguish remote or
lower-trust input from actions already authorized by the local user; distinguish
process crashes from loss of confidentiality or integrity.
