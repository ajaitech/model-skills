# AiAmbA Protocol Translator

## Goal
AiAmbA IoT: 24/7 Linux edge service (systemd `aiamba-iot.service`) — discovers every device on a factory LAN, classifies make/model/protocol, persists per-org under Postgres RLS, opens a dedicated WebSocket per device so an AI agent reads freely and, only after human approval, writes/configures.

## Core requirements
- **PostgreSQL only**; DynamoDB forbidden (`README.md`, `db.py`).
- **Reads free, writes gated** — deny-by-default `ApprovalGate`, single-use time-boxed tokens, timeout auto-deny (`core/approval_gate.py`); kill-switch `bridge.allow_device_writes` defaults `false` (`config.py`).
- **No hardcoded secrets** — device creds via `credentials_ref` → env `AIAMBA_SECRET_<UPPER_REF>`; DB password from env named by `db.password_env` (`core/base_translator.py`).
- **Per-org RLS** — org resolved at boot from the factory static IP (`db.py::_resolve_org` vs `hierarchy.factory`); every txn runs `SET LOCAL app.org_id/user_id/role`.
- **One OS process per device sub-service**; RSS 384/512 MiB soft/hard, restart+backoff behind a crash-loop guard (`watchdog/supervisor.py`).
- **Field reliability**: systemd `Type=notify`/`Restart=always`/`WatchdogSec=30`/`MemoryMax=3G`; boots degraded (not crashed) without Postgres (`main.py::_open_db_with_retry`); liveness teardown only after several missed pings + a silence window, never one blip (`config.py::LivenessConfig`); bounded telemetry queue caps memory on slow WS consumers (`bridge/websocket.py`).
- **HARD discovery tools transient** — install → use → ALWAYS uninstall in `finally` → verify removal; re-purged at boot (`discovery/hard_runner.py`).

## Tech stack
Sources: `requirements.txt`; translators `protocols/requirements-translators.txt`; DB name `config/aiamba-iot.toml`; Python floor `config.py`.
| Layer | Technology | Version |
|---|---|---|
| Language | Python | 3.11+ (stdlib `tomllib`) |
| Database | PostgreSQL (db `aibeste-main-db`); psycopg[binary] / psycopg-pool | `14`; `3.2.3` / `3.2.4` |
| Cloud SDK | boto3 (Secrets Manager, ap-south-1) | `1.35.71` |
| Discovery | scapy / getmac / zeroconf / onvif-zeep / pysnmp / python-nmap | `2.6.1`/`0.9.5`/`0.136.2`/`0.2.12`/`6.2.6`/`0.7.1` |
| Supervision / WS transport | psutil / websockets | `6.1.1` / `13.1` |
| Mapping API (+`mapping/requirements.txt`) | fastapi / uvicorn / pydantic | `0.115.6`/`0.34.0`/`>=2.7.0` |
| Translators (merged pins) | asyncssh / lxml / pyasn1 / python-snap7 / pyfocas | `2.23.1`/`5.3.0`/`0.6.1`/`1.3`/`0.2.1` |

## Architecture
Pipeline: `discovery/` (LIGHT sweep + HARD on-demand) → `segregate/` (identified vs un-found) → `commtest/` (ping→connect→repair→secure) → `bridge/` (assign_agent: singleton group↔agent; websocket: 1:1 WS on a leased edge port) → `watchdog/` (liveness re-ping/teardown).
`registry/factory.py::build_translator` resolves a descriptor to its family translator, else the generic probe (real TCP/TLS, never a mock). All subclass `core/base_translator.py::BaseTranslator`: `ping/connect/repair/secure/health/disconnect` + `read/subscribe` (free) and `write/configure` (gated). WS bridge: `bridge/websocket.py::WebSocketBridge`, port from `core/ports.py::PortBroker` (base `42000`, count `512`). OEM deps are SOFT-imported (`soft_import`) so modules import cleanly without wheels.

**Deploy** (README §6): `rsync -a aiamba-iot/ /opt/aiamba-iot/` + venv + systemd — no container. Unit grants `CAP_NET_RAW`/`CAP_NET_ADMIN`, is hardened (`ProtectSystem=strict`); mutable state under `/var/lib/aiamba-iot` (install dir read-only at runtime). **Import trap**: dir name has a hyphen — imports are top-level (`from core import db`) with `/opt/aiamba-iot` on `PYTHONPATH`. Config: env `AIAMBA_CONFIG` → `config/aiamba-iot.toml`. Verification is live-only: comm-test CLI + mapping API (no test suite — see gaps).

## Translator inventory
**Measured: 17 translators + 1 generic fallback = 18 families.** Method: grep `class Translator(BaseTranslator)` → 17 files, one per folder; `protocols/TRANSLATOR-MATRIX.md` agrees; all `registry/factory.py` targets resolve. **The "179+ live protocol translators" claim is false** ("179" absent from source; only OUI `00179a`). All 17 are 2-way (writes gated); generic is monitor-only.

| Implementation file | Families (protocol) |
|---|---|
| `protocols/<family>/__init__.py` | siemens (S7comm/OPC-UA), mitsubishi (SLMP/MC), delta (Modbus TCP), ipcamera (ONVIF/RTSP), biometric (ZKTeco/iClock), printer (IPP/PJL/SNMP) |
| `protocols/<family>/translator.py` | fanuc (FOCAS/MTConnect), tplink (web/JSON, SNMP), tether (Kasa/Tapo), dlink (SNMP/HNAP/ONVIF), cisco (SSH/RESTCONF/SNMP), linux (SSH/SNMP), windows (WinRM/SMB), macos (SSH/Bonjour), ibm (Redfish/IPMI), jetson (SSH+NVIDIA tooling), cplink (ONVIF/RTSP/CGI) |
| `core/generic_probe.py` | generic (TCP/TLS probe, fallback) |

## Naming conventions
`class Translator(BaseTranslator)`, `FAMILY` = folder; env `AIAMBA_<UPPER_SNAKE>`; TOML lower_snake mirrors `Config` (`[db]`,`[net]`,`[watchdog]`,`[bridge]`); gate verbs `read`/`write`/`configure`/`subscribe`; WS channel key `{agent_slug}:{hier_group_id}:{edge_port}`.

## Data & DB touchpoints
- `hierarchy.device` (join) → device descriptor read by `main.py`/`commtest/runner.py`: device_id, name, ip_address, make/model/category/device_class, primary_protocol, credentials_ref.
- `aibeste_net_ws_channels` ← `WsChannel` rows from `bridge/websocket.py`: channel key, org/wan/group/agent/edge_server ids, url, edge_port, state, is_live, bytes_in/out.
- `hierarchy.agent`/`.group` ← `AgentAssignment` from `bridge/assign_agent.py`: singleton agent per group, agent_slug, reused:bool.
- `device_connections.port` ← `PortLease` (`core/ports.py`); `device_master_candidate` ← observed make/model/class/protocols/ports + raw_attributes (`mapping/manual_map.py`).
- Single-file dataclasses (`PhaseResult`, `PendingApproval`, `PortLease`, `HealthReport`) live in `core/`.

## API surface
- **Device WS channel** (no auth seen; loopback/VPN trust): WSS on its leased edge port — `{type: ping|command|write|control,…}` → `{type: pong|result|error,…}` (`bridge/websocket.py`).
- **Mapping API** (`mapping/api.py`; binds `127.0.0.1`, env `EDGE_API_HOST`; optional `X-AiAmbA-Token` vs env `AIAMBA_API_TOKEN`/`EDGE_API_TOKEN`): `GET /unfound` · `/unmappable` · `/categories` · `/masters?q=&category=` · `/commtest/{id}/states` · `/health` (no auth); `POST /map` (`MapRequest` body → `{device_id,master_id,candidate_id,status,next}`; logic in `mapping/manual_map.py`) · `/unmappable/{id}/retry?actor=` · `/commtest/{id}?actor=` (→ comm-test report).
- **CLI**: `python -m commtest.cli <id>` → JSON stdout, exit 0/1/2/3 (`commtest/cli.py`).

## Security boundary
- Auth is opt-in on BOTH surfaces: mapping-API token unset ⇒ full read+map access for anything reaching loopback/VPN; device WS has no per-connection auth in reviewed code — network-level isolation only. No CORS middleware; nothing binds a public interface by default.
- `ApprovalGate` is the sole gate on device writes — a safety control, not a network/host boundary; the edge LAN is zero-restriction by design (`AIAMBA-IOT.md`).
- Secrets (names only): `AIAMBA_DB_PASSWORD`, `AIAMBA_DB_DSN`, `AIAMBA_API_TOKEN` — env-only; systemd sources a root:aiamba `0640` env file from AWS Secrets Manager.

## Known gaps & risks
- **Identifier/credential exposure in-source**: `AIAMBA-IOT.md`, `AiAmbA-IoT_Codex_CLI_Directive_v2.md`, `config/aiamba-iot.toml` + READMEs embed the real factory WAN static IP, edge EC2 elastic IP + instance id, DB host FQDN, an AWS account id, a Linux username, and one plaintext operator login (user:password, `AIAMBA-IOT.md`). Never copy these into public files — use env refs (`AIAMBA_ORG_STATIC_IP`, `AIAMBA_DB_DSN`). Blast radius: WAN IP + login = direct remote target.
- **README documents `pytest tests/ -q # 9 passed` but no `tests/` exists**; `.pytest_cache` retains 18 stale IDs from `test_manual_map.py`/`test_supervisor.py`.
- **Supply-chain pre-flight missing**: requirements files mandate `tools/email_scrub.py` over the resolved wheel tree before first import, but `tools/` is absent — the documented install flow cannot run.
- **Three parallel DB pools** (`REFINEMENTS.md` R1): `db.py` (canonical), `core/db.py` (tuple rows), `segregate/db.py` (dict rows + elevated role); unify deferred to P3.4.
- **Stale REFINEMENTS note**: claims the WS command handler is unbound, but `main.py::Supervisor.boot` passes `command_handler=self._handle_ws_command` — trust `main.py`.
- **OEM natives out-of-band**: python-snap7 needs `libsnap7.so`; pyfocas needs FANUC `libfwlib32` — not pip-installable; siemens/fanuc are code-complete but not runnable without them.
