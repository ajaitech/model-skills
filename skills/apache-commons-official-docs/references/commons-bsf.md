# Apache Commons BSF official sources

Use only when the resolved project dependency already requires Commons BSF or the user explicitly selects it. The official site is old and distinguishes incompatible 2.x and 3.x APIs; never infer one from the other.

| Need | Official URL | Crawl context |
|---|---|---|
| API family and status | https://commons.apache.org/proper/commons-bsf/ | Identify BSF 2.x versus JSR-223-based 3.x before reading examples. |
| BSF manual | https://commons.apache.org/proper/commons-bsf/manual.html | For BSF 2.x, verify `BSFManager`, `BSFEngine`, object registration, `eval`, and `exec` behavior. |
| Released artifacts | https://commons.apache.org/bsf/download_bsf.cgi | Confirm Apache-published release lines; dependency resolution remains authoritative. |
| Dependencies | https://commons.apache.org/proper/commons-bsf/dependencies.html | Check transitive language-engine and library compatibility. |
| Source | https://github.com/apache/commons-bsf | Inspect exact implementation only after matching the dependency tag/commit. |

Treat all evaluated scripts as untrusted input unless provenance is guaranteed. Apply engine allow-lists, capability isolation, bounded execution, controlled bindings, redacted logs, and tests. Never expose credentials, arbitrary host objects, filesystem, process, or network access by default.
