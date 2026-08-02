# AiAmbA Protocol Translator

## Goal
AiAmbA IoT is a 24/7 Linux edge service (systemd `aiamba-iot.service`) that
discovers every device on a factory LAN, classifies its make/model/protocol,
persists the result per-org under Postgres RLS, then opens a dedicated
WebSocket per device so an AI agent can read state freely and — only after
human approval — write/configure it. Audience: the edge operator and the
agent runtime consuming the per-device WS channel.

## Core requirements
- **PostgreSQL only**; DynamoDB forbidden (`README.md`, `db.py`).
- **Reads free, writes gated** via `core/approval_gate.py::ApprovalGate`
  (deny-by-default, single-use tokens, confirm-timeout auto-deny, kill-switch
  `bridge.allow_device_writes` defaults `false`).
- **No hardcoded secrets** — resolved only via `credentials_ref` at runtime;
  DB password from env named by `db.password_env` (`core/base_translator.py`).
- **Per-org tenancy via RLS** — org resolved at boot from a factory static IP;
  every query runs `SET LOCAL app.org_id/user_id/role` (`db.py`).
- **One OS process per device sub-service**, RSS ceiling (384/512 MiB
  soft/hard), restart with backoff behind a crash-loop guard
  (`watchdog/supervisor.py`).
- **Field reliability (verified):** systemd `Type=notify`/`Restart=always`/
  `WatchdogSec=30`/`MemoryMax=3G`; DB retries with backoff, runs degraded
  (not crashed) if Postgres is down at boot (`main.py::_open_db_with_retry`);
  liveness loop tears a device down only after several missed pings plus a
  silence window, never one blip (`config.py::LivenessConfig`); a bounded
  telemetry queue caps memory on a slow WS consumer
  (`bridge/websocket.py::BoundedTelemetryQueue`).
- **HARD discovery tools are transient** — install-use-uninstall-verify, one
  resident at a time, re-purged at boot (`discovery/hard_runner.py`).

## Tech stack
| Layer | Technology | Version (exact) | Source of truth |
|---|---|---|---|
| Language | Python | 3.11+ (stdlib `tomllib`) | `config.py`, `README.md` |
| DB driver / pool | psycopg[binary] / psycopg-pool | `3.2.3` / `3.2.4` | `requirements.txt` |
| Database | PostgreSQL | `14` (`aibeste-main-db`) | `config/aiamba-iot.toml` |
| Cloud SDK | boto3 (Secrets Manager) | `1.35.71` | `requirements.txt` |
| Discovery | scapy / getmac / zeroconf / onvif-zeep / pysnmp / python-nmap | `2.6.1`/`0.9.5`/`0.136.2`/`0.2.12`/`6.2.6`/`0.7.1` | `requirements.txt` |
| Supervision | psutil | `6.1.1` | `requirements.txt` |
| Transport | websockets | `13.1` | `requirements.txt` |
| Mapping API | fastapi / uvicorn / pydantic | `0.115.6`/`0.34.0`/`>=2.7.0` | `requirements.txt`, `mapping/requirements.txt` |
| Translators (merged) | asyncssh / lxml / pyasn1 | `2.23.1`/`5.3.0`/`0.6.1` | `protocols/requirements-translators.txt` |
| Process model | systemd `Type=notify` unit | n/a | `systemd/aiamba-iot.service` |
| Test runner (referenced, absent) | pytest | unpinned | `README.md` — see Known gaps |

## Architecture
```
discovery/ (LIGHT sweep + HARD on-demand) → segregate/ (identified vs
un-found) → commtest/ (ping→connect→repair→secure, registry/factory.py)
→ bridge/ (assign_agent: singleton group↔agent; websocket: dedicated WS
channel 1:1 to a leased edge port) → watchdog/ (one OS sub-process per
device; liveness re-ping/teardown)
```
`registry/factory.py::build_translator` resolves a device descriptor
(make/protocol/category) to one of 17 brand-family translators under
`protocols/<family>/`, or the built-in `core.generic_probe.GenericProbe` (a
real TCP/TLS probe, never a mock). Every translator subclasses
`core/base_translator.py::BaseTranslator` for a uniform
`ping/connect/repair/secure/health/disconnect` lifecycle plus `read/subscribe`
(free) and `write/configure` (gated) verbs. Upstream consumer: an AI agent on
the per-device WebSocket (`bridge/websocket.py::WebSocketBridge`, port leased
from `core/ports.py::PortBroker`, pool base `42000`/512). A loopback-only
FastAPI (`mapping/api.py`) serves an operator UI for un-found devices and
manual mapping. **Runtime target:** single-process edge gateway — deploy is
`rsync` + venv + systemd unit (`README.md` §6), not a container in the
reviewed source, with `CAP_NET_RAW`/`CAP_NET_ADMIN` for discovery.

## Translator inventory
**Measured count: 17 brand/protocol translators + 1 generic fallback = 18
resolvable families.** Method: `grep -rl "class Translator(BaseTranslator)"
aiamba-iot/protocols/` returns exactly 17 files, one per family folder —
matching what `protocols/TRANSLATOR-MATRIX.md` and `REFINEMENTS.md` claim
for themselves. Every `registry/factory.py` resolver target maps to one of
these 17 folders or to `generic`; no dangling target found. **This
contradicts the "179+ live protocol translators" claim — the codebase
implements 17, not 179+.** No "179" occurs anywhere in the source tree (only
an unrelated MAC-OUI prefix `00179a` in `discovery/oui_lookup.py`). Paths
below are relative to `aiamba-iot/`.

| Protocol | Direction | Implementation file | Status |
|---|---|---|---|
| siemens (S7comm/OPC-UA) | 2-way | `protocols/siemens/__init__.py` | Built |
| mitsubishi (SLMP/MC) | 2-way | `protocols/mitsubishi/__init__.py` | Built |
| delta (Modbus TCP) | 2-way | `protocols/delta/__init__.py` | Built |
| fanuc (FOCAS/MTConnect) | 2-way | `protocols/fanuc/translator.py` | Built |
| ipcamera (ONVIF/RTSP) | 2-way | `protocols/ipcamera/__init__.py` | Built |
| biometric (ZKTeco/iClock) | 2-way | `protocols/biometric/__init__.py` | Built |
| printer (IPP/PJL/SNMP) | 2-way | `protocols/printer/__init__.py` | Built |
| tplink (Web/JSON, SNMP) | 2-way | `protocols/tplink/translator.py` | Built |
| tether (Kasa/Tapo) | 2-way | `protocols/tether/translator.py` | Built |
| dlink (SNMP/HNAP/ONVIF) | 2-way | `protocols/dlink/translator.py` | Built |
| cisco (SSH/RESTCONF/SNMP) | 2-way | `protocols/cisco/translator.py` | Built |
| linux (SSH/SNMP) | 2-way | `protocols/linux/translator.py` | Built |
| windows (WinRM/SMB) | 2-way | `protocols/windows/translator.py` | Built |
| macos (SSH/Bonjour) | 2-way | `protocols/macos/translator.py` | Built |
| ibm (Redfish/IPMI) | 2-way | `protocols/ibm/translator.py` | Built |
| jetson (SSH+NVIDIA tooling) | 2-way | `protocols/jetson/translator.py` | Built |
| cplink (ONVIF/RTSP/CGI) | 2-way | `protocols/cplink/translator.py` | Built |
| generic (TCP/TLS probe) | monitor-only | `core/generic_probe.py` | Built-in fallback |

## Naming conventions
- **Translator export**: `class Translator(BaseTranslator)` per family (e.g.
  `protocols/siemens/__init__.py`); `FAMILY` attr matches its folder.
- **Env vars**: `AIAMBA_<UPPER_SNAKE>` (e.g. `AIAMBA_DB_PASSWORD`,
  `AIAMBA_ORG_STATIC_IP`); secret refs → `AIAMBA_SECRET_<UPPER_REF>`
  (`core/base_translator.py`).
- **Config keys**: lower_snake_case TOML mirroring `Config` dataclasses —
  `[db]`, `[net]`, `[watchdog]`, `[bridge]`.
- **Gate verbs**: lower-case single words — `"read"`, `"write"`,
  `"configure"`, `"subscribe"` (`core/approval_gate.py`).
- **WS channel key**: `f"{agent_slug}:{hier_group_id}:{edge_port}"`
  (`bridge/websocket.py::open_channel`).

## Data types & models
| Entity | Fields (name : type) | Store | Defined in |
|---|---|---|---|
| `PhaseResult` | ok:bool, state:str, latency_ms:int\|None, secure_tls:bool, error_code/detail, detail:dict | in-memory, row/phase | `core/translator_iface.py` |
| `PendingApproval` | cmd_id, action, device_key, payload:dict, summary, requested_by, expires_at:float | in-memory (gate) | `core/approval_gate.py` |
| Device descriptor | device_id, name, ip_address, make, model, category, device_class, primary_protocol, credentials_ref | dict, from `hierarchy.device` join | `main.py`, `commtest/runner.py` |
| `PortLease` | device_key:str, port:int, host:str | in-memory + `device_connections.port` | `core/ports.py` |
| `WsChannel` | channel_id/key, org/wan/group/agent/edge_server_id, url, protocol, edge_port, state, is_live, bytes_in/out | Postgres `aibeste_net_ws_channels` | `bridge/websocket.py` |
| `AgentAssignment` | org_id, device_id, group_id, hierarchy_agent_id, registry_agent_id, agent_slug, reused:bool, provider | Postgres `hierarchy.agent`/`.group` | `bridge/assign_agent.py` |
| `HealthReport` | name, pid, state(enum), rss_mb, uptime_s, connected, last_error, ts | Unix datagram → watchdog | `core/health.py` |
| device_master_candidate | observed_make/model/class/protocols/ports, mac_address, raw_attributes, status | Postgres `device_master_candidate` | `mapping/manual_map.py` |
| `MapRequest` | device_id, category_code, make, model, primary_protocol, protocols?, ports?, mac_address?, net_protocol?, actor? | HTTP body | `mapping/api.py` |

## API surface
| Operation | Transport/Path | Request shape | Response shape | Auth | Defined in |
|---|---|---|---|---|---|
| Device channel | WSS, dedicated port (`42000`+) | `{type: ping\|command\|write\|control,...}` | `{type: pong\|result\|error,...}` | none seen; loopback/VPN trust | `bridge/websocket.py` |
| Un-found list | `GET /unfound` | — | `{unfound:[...]}` | optional `X-AiAmbA-Token` | `mapping/api.py` |
| Unmappable list | `GET /unmappable` | — | `{unmappable:[...]}` | optional token | `mapping/api.py` |
| Retry unmappable | `POST /unmappable/{id}/retry` | query `actor` | mapping-state dict | optional token | `mapping/api.py` |
| Category catalog | `GET /categories` | — | `{categories:[...]}` | optional token | `mapping/api.py` |
| Master search | `GET /masters?q=&category=` | query | `{masters:[...]}` | optional token | `mapping/api.py` |
| Submit mapping | `POST /map` | `MapRequest` JSON body | `{device_id,master_id,candidate_id,status,next}` | optional token | `mapping/manual_map.py` |
| Trigger comm-test | `POST /commtest/{id}` | query `actor` | comm-test report dict | optional token | `mapping/api.py` |
| Comm-test history | `GET /commtest/{id}/states` | — | `{device_id,states:[...]}` | optional token | `mapping/api.py` |
| Health | `GET /health` | — | `{status,service,org_static_ip}` | none | `mapping/api.py` |
| CLI comm-test | `python -m commtest.cli <id>` | CLI args | JSON to stdout, exit 0/1/2/3 | shell/env DB access | `commtest/cli.py` |

## CORS & headers
Not applicable as reviewed. `mapping/api.py` builds FastAPI with no CORS
middleware and no `allow_origins` in source — the API binds `127.0.0.1` only
(env `EDGE_API_HOST`), reached over a VPN/SSH tunnel per its own docstring.

## Security boundary
- **Auth**: mapping API accepts an optional shared-secret header
  `X-AiAmbA-Token` vs env `AIAMBA_API_TOKEN` (`mapping/api.py::_require_token`)
  — unset, the endpoint is open to anything reaching loopback/VPN. Device WS
  channel shows no per-connection auth in reviewed code; isolation is
  network-level only.
- **Write safety**: the sole gate on a device-changing action is the
  in-process `ApprovalGate` — documented as a safety control, not a
  network/host security boundary.
- **Network trust**: the edge LAN is zero-restriction by design ("Zero
  security restrictions on the network/edge", `AIAMBA-IOT.md`); only
  network-adjacent hardening is install→use→uninstall for HARD tools.
- **Secret sources (names only)**: `AIAMBA_DB_PASSWORD`, `AIAMBA_DB_DSN`,
  `AIAMBA_SECRET_<REF>`, `AIAMBA_API_TOKEN`/`EDGE_API_TOKEN` — read from env
  at runtime, none hardcoded in reviewed source; systemd sources them from a
  root:aiamba `0640` env file populated from AWS Secrets Manager.
- **Public vs private**: nothing in reviewed code binds a public interface by
  default — mapping API and WS listener default to `127.0.0.1`.

## Known gaps & risks
- **README claims a test suite absent from this checkout.** `README.md`
  documents `pytest tests/ -q # 9 passed`, but no `tests/` dir exists here;
  `.pytest_cache/v/cache/nodeids` evidences 17 previously-cached IDs
  (`test_manual_map.py`, `test_supervisor.py`) that ran once but are gone.
- **Three parallel DB-pool implementations**, unreconciled per the project's
  own `REFINEMENTS.md` R1: `db.py` (canonical), `core/db.py` (tuple rows),
  `segregate/db.py` (dict rows + elevated role) — "unify onto ONE pool" is
  explicitly deferred.
- **WS command-handler note is stale**: `REFINEMENTS.md` says it was still
  unbound as of the Completion-B review, but current
  `main.py::Supervisor.boot` does pass `command_handler=self._handle_ws_command`
  — trust `main.py`, not the older note.
- **Two OEM deps need out-of-band native installs**: `python-snap7` needs
  `libsnap7.so`; `pyfocas` needs FANUC's `libfwlib32` — neither
  pip-installable, so siemens/fanuc are code-complete but not runnable
  without these on the edge host.
- **Auth is opt-in on both control surfaces**: with no `AIAMBA_API_TOKEN` set,
  anything reaching loopback/VPN has full read+map access (writes still
  gated by the ApprovalGate).
