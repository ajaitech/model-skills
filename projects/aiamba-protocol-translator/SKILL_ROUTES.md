Codebase: AiAmbA Protocol Translator (`aiamba-protocol-translator`) — Python edge IoT protocol-translation service (`aiamba-iot/`).

| Condition | Skill URL |
|---|---|
| Touching device discovery, protocol translators (`protocols/<family>/`), comm-test lifecycle, or the ApprovalGate write-gate | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/iot-protocol-translation.md |
| Touching Postgres access, RLS (`db.py`, `core/db.py`, `segregate/db.py`), or schema under `hierarchy.*` / `device_masters.*` / `aibeste_net.*` | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/data-postgres-dynamo.md |
| Touching AWS Secrets Manager credential resolution (`boto3`, `AIAMBA_SECRET_*`) or the systemd/EC2 edge deploy | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/aws-platform.md |
| Ad-hoc machine or system task, not product code | https://raw.githubusercontent.com/ajaitech/model-skills/main/machine/MACHINE_INDEX.md |

Fetch only rows whose condition is true right now.
