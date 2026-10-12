# Grafana OSS threat model for scanner reports

## Scope

Scan the open-source code in `grafana/grafana`, including its Go server, HTTP APIs, frontend, bundled plugins, and application modules. Treat Grafana Cloud services and private Enterprise code that is absent from this repository as out of scope. A finding must identify the affected code, required configuration or feature flags, and the attacker position.

## Trust boundaries

- Browser clients and API callers supply requests, authentication material, dashboard and panel content, queries, and provisioning inputs. Check authentication, authorization, and organization/resource scope at the server endpoint that processes each action.
- Grafana stores and uses data-source credentials and other secrets. Check whether lower-privilege users, plugins, or data-source responses can read them or cause Grafana to send them to an unintended destination.
- Data sources, plugins, and imported dashboards can supply untrusted content that reaches the browser or server. Check output encoding and sanitization in the actual rendering path, as well as server-side request and file boundaries.
- Cross-organization and cross-user access, role or permission changes, and server-side requests to internal destinations are high-value cases. Do not infer exploitability from a missing check in isolation; trace the reachable route and its other guards.

## Report requirements

Provide a reproducible request or UI sequence, the lowest privileges needed, relevant settings and feature flags, the observed outcome, and a proposed patch where possible. Distinguish source-level possibilities from a demonstrated result. Do not include live credentials or customer data in a report.

The image retains `/src`, the built Grafana binary under `bin/`, frontend assets under `public/`, Go modules, and Yarn dependencies. Run focused Go tests with `go test` and frontend tests with `yarn jest --runInBand --watch=false` from `/src`; start the server with a local test configuration when an end-to-end reproducer is needed. The scanner environment has no Internet access after the image build.
