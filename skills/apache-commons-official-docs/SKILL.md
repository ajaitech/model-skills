---
name: apache-commons-official-docs
description: Use when implementing, upgrading, debugging, or reviewing Java code with Apache Commons Net protocols (SMTP, FTP, FTPS, TFTP, NNTP, NTP, Telnet, Finger, Whois, rlogin, rexec, rshell), Commons Geometry, Commons BSF bean scripting or JSR-223, or Commons Collections APIs.
---

# Apache Commons official documentation

Use the repository's Maven/Gradle dependency and JDK as the compatibility boundary. Retrieve current official documentation instead of storing vendor content.

## Retrieval contract

1. Inspect build files, dependency resolution, JDK, and existing abstractions.
2. Read only the matching reference below.
3. Open the canonical user guide and exact API member first. Open a listed example only for its protocol flow.
4. Treat examples as reference inputs, not production templates. Verify signatures, security, timeouts, encodings, reply codes, cleanup, retries, cancellation, observability, and tests before applying code.
5. Use migration/release material only as secondary compatibility evidence. A changelog is not a skill or implementation authority.
6. Reuse the existing gateway/service; do not duplicate clients or embed credentials.

## Routes

| Task | Read |
|---|---|
| Internet protocols / Commons Net | [references/commons-net.md](references/commons-net.md) |
| Geometry | [references/commons-geometry.md](references/commons-geometry.md) |
| Bean scripting / BSF | [references/commons-bsf.md](references/commons-bsf.md) |
| Collections | [references/commons-collections.md](references/commons-collections.md) |
