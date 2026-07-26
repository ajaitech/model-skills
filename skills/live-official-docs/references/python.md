# Python official sources

Match `.python-version`, `pyproject.toml`, lockfiles, runtime constraints, and `python --version`. Treat development releases separately from stable production versions.

| Need | Official URL | Crawl context |
|---|---|---|
| Supported release status | https://www.python.org/downloads/ | Identify the appropriate stable or security-supported line; do not upgrade implicitly. |
| Versioned documentation | https://docs.python.org/3/ | Switch to the project's exact minor version before reading syntax or library behavior. |
| Language grammar and semantics | https://docs.python.org/3/reference/ | Verify exact syntax and data-model semantics. |
| Packaging | https://packaging.python.org/en/latest/ | Check current `pyproject.toml`, build, metadata, dependency, and publishing guidance. |
| Accepted language standards | https://peps.python.org/ | Use the final PEP plus implemented-version evidence; a proposal alone is not runtime support. |
| Compatibility only: porting and changed behavior | https://docs.python.org/3/whatsnew/ | After the versioned reference, read applicable installed-to-target removals and deprecations. |

Official domains: `python.org`, `docs.python.org`, `packaging.python.org`, `peps.python.org`, and `pypi.org`.
