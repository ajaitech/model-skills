# IoT Protocol Translation

## Applies when
- Deps on `pymodbus`, `opcua`/`asyncua`, `paho-mqtt`, `bacpypes`/`BAC0`, or equivalents.
- A `gateway/`/`edge/`/`translation/` layer maps field-device points to cloud telemetry.
- Config references registers, node IDs, topics, or BACnet object instances.

## Authoritative sources
| Need | URL |
|---|---|
| Modbus specifications | https://www.modbus.org/specs.php |
| Modbus Organization | https://www.modbus.org/ |
| OPC UA overview | https://opcfoundation.org/about/opc-technologies/opc-ua/ |
| OPC UA online reference | https://reference.opcfoundation.org/ |
| MQTT v5.0 (OASIS Standard) | https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html |
| MQTT.org | https://mqtt.org/ |
| BACnet standard overview | https://bacnet.org/about-bacnet-standard/ |
| BACnet Organization | https://bacnet.org/ |
| ASHRAE Standard 135 (BACnet) | https://www.ashrae.org/technical-resources/standards-and-guidelines |

## Non-obvious rules
- **Modbus addressing off-by-one.** Datasheet `40001` is 1-based (Modicon); wire address is 0-based — subtract `40001` (or `1`) before reading.
- **Modbus has no native data typing.** 32-bit values span two registers; word order (`AB CD` vs swapped `CD AB`) is vendor-specific — confirm against the device's own map.
- **Modbus TCP has no in-band keepalive** — a dropped session is silent until timeout; run a health check separate from per-request retry.
- **Modbus RTU is half-duplex, single-master** — two pollers on one bus corrupt frames; inter-frame silence timing is often violated by USB-RS485 adapters.
- **OPC-UA security failures are generic** — a rejected client cert under Sign/SignAndEncrypt surfaces as `BadSecurityChecksFailed`; check the trust list first.
- **OPC-UA production pattern is subscriptions, not polling** — too-small a sampling interval floods the server; use deadband filters.
- **OPC-UA subscriptions outlive a lost session briefly**, then auto-delete — reconnect logic must explicitly re-create them.
- **MQTT QoS is hop-by-hop, not end-to-end** — "delivered" guarantees only the next hop unless the whole bridge path is uniformly QoS ≥1.
- **MQTT online/offline relies on LWT + retained messages** — a device with no Last Will looks permanently online after a keepalive-missed disconnect.
- **MQTT wildcards leak across tenants** if topics aren't namespaced by tenant/site before `+`/`#` subscribes.
- **BACnet COV subscriptions expire** and need renewal before their lifetime runs out, or updates silently stop.
- **BACnet object identifiers are (type, instance) pairs** — `AnalogInput:1` and `BinaryInput:1` are distinct; a common bug keys maps on instance alone.
- **BACnet discovery doesn't cross subnets without a BBMD** — "not found" across VLANs is usually a missing Broadcast Management Device entry.
- **Unit/scale loss is the top cross-protocol bug** — carry engineering unit and scale factor alongside every raw value through the pipeline.
- **Field-network timeouts differ from IT timeouts** — serial/cellular links run higher latency; distinguish timeout vs protocol error vs device fault, don't collapse into one "offline" state.

## Production checklist
- [ ] Register/point map versioned and cross-checked against the datasheet
- [ ] Word/byte order confirmed per device model, not assumed
- [ ] Connection health check independent of per-request timeout/retry
- [ ] OPC-UA trust list managed; subscriptions recreated on reconnect
- [ ] MQTT LWT configured per device; topics namespaced by tenant/site
- [ ] BACnet COV renewal timer running; BBMD configured cross-subnet
- [ ] Every value carries unit + scale through the pipeline
- [ ] Timeout/retry tuned to the actual link, not LAN defaults
- [ ] Errors surfaced with layer distinguished (transport/protocol/device)

## Never
- Never assume a 1-based datasheet register number is wire-ready.
- Never assume multi-register word order without checking the device's map.
- Never subscribe OPC-UA MonitoredItems without a sized deadband.
- Never leave a BACnet COV subscription without a renewal timer.
- Never let a raw value cross a protocol boundary without its unit and scale.
- Never collapse timeout, protocol error, and device fault into one message.
