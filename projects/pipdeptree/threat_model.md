# Threat model



## Scope and untrusted input

pipdeptree discovers installed Python distributions and renders their dependency graph. It can also resolve requirements
against a package index or read a PEP 751 lock file, then emit text, JSON, CycloneDX, Mermaid, Graphviz or
Graphviz-rendered binary output. Most parsing and graph construction lives in the PyO3 Rust extension under `rust/src/`;
`src/pipdeptree/_runner.py` connects it to Python streams and the desktop opener.

Treat these inputs as hostile:

- `METADATA`, `direct_url.json`, `RECORD`, `entry_points.txt`, egg-link files and directory names beneath any scanned
  `site-packages` path;
- package names, versions, requirements, extras, markers, license fields, URLs and editable/VCS metadata;
- a PEP 751 TOML lock file and requirements files passed to the subcommands;
- package-index responses and artifacts inspected during resolution;
- interpreter paths and the JSON-like data returned by an interpreter queried with `--python`;
- Graphviz executables and output bytes when a rendered output format is requested;
- terminal width, encoding and output streams supplied through the Python entry point.

An explicitly selected interpreter and package index are trusted choices. Package metadata is data even when the
installed package's code is never imported, so malformed metadata must not execute code or escape the requested output.



## Components to examine

Prioritize metadata parsing in `rust/src/metadata/`, lock and requirement parsing, graph cycle handling, index
resolution and the process abstraction. Check every parser for bounded allocation and recursion on long fields, many
packages, deep graphs, cycles and duplicate metadata. Directory walks and package-size calculation must not follow
symlinks out of an environment or loop indefinitely.

Review output escaping separately for JSON/CycloneDX, Mermaid and Graphviz. Package-controlled labels, URLs and license
text must remain data. Graphviz invocation must use a fixed executable and argument array, write only the requested
output, and handle binary output without interpreting it as terminal text. `_runner.py` writes undecodable binary output
to a temporary file and asks the desktop to open it; check suffix selection, permissions and content-type confusion.

Interpreter probing executes the selected Python to obtain paths and marker values. Auto-detection must not select a
working-directory executable or turn returned strings into command arguments. The Rust/Python boundary should convert
errors without panics, use-after-free or uncaught unwinds.



## Exercising the image

The source is at `/src`, built with Rust and installed in editable mode in `/opt/venv`. A wheel is available under
`/opt/wheels`. Run `python -m pytest tests`. Create hostile fixtures by placing handcrafted `.dist-info` directories in
a temporary path and invoking `pipdeptree --path <path>`; use local requirements and lock files for resolver tests.

The scan has no network. Package-index behavior should use a local fixture or mocked process response. Graphviz may be
installed during a reproduction if its parser is relevant, but the report must distinguish pipdeptree argument handling
from a Graphviz vulnerability.



## Severity guidance

- High: merely inspecting package metadata or a lock file executes a command, writes outside an explicitly requested
  output path, or triggers memory corruption across the Rust/Python boundary.
- Medium: crafted metadata makes Graphviz or the desktop opener process an unintended local file or option, directory
  traversal exposes files outside the selected environment, or a small input causes sustained resource exhaustion.
- Low: crashes, malformed output, false dependency reports, or resource use proportional to a genuinely large
  environment.

Reports should include the command, complete metadata or lock fixture, output mode, interpreter and platform. Prefer a
test that does not require installing attacker code.



## Out of scope

- Code execution by an interpreter explicitly named with `--python` or a Graphviz executable already replaced on `PATH`
  by the user.
- Malicious package installation; the relevant boundary is inspection after installation without importing it.
- Incorrect or malicious dependency data returned faithfully from the selected package index.
- Vulnerabilities wholly inside Graphviz, a package index client or the operating system's desktop opener.
- Cosmetic graph layout differences and dependency conflicts pipdeptree correctly reports.
