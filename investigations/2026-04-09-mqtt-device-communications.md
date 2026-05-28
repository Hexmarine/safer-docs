# MQTT Device Communications

- Date: 2026-04-09
- Scope: understand how MQTT communications between the cloud backend and field controllers are organized, and identify visible authentication or verification controls
- Related systems: Sensor Hub or controller, field alarms, backend API, MQTT broker tier, provisioning and activation flows

## Objective

Describe the current MQTT topic structure, client organization, and trust model, with special attention to whether device-originated MQTT traffic is authenticated or otherwise verified beyond broker connectivity.

## Sources Used

- Repository files:
  - `code/sensor-alarm-backend/src/services/mqtt/subscriber.ts`
  - `code/sensor-alarm-backend/src/controllers/alarms.controller.ts`
  - `code/sensor-alarm-backend/src/entities/alarms.entity.ts`
  - `code/sensor-alarm-backend/src/config/app.ts`
  - `code/mqtt-prototype/firmware.md`
  - `code/mqtt-prototype/Constants.cs`
  - `code/mqtt-prototype/Program.cs`
  - `code/mqtt-prototype/Controller.cs`
  - `code/mqtt-prototype/ApiClient.cs`
- AWS commands or consoles:
  - none required for this pass beyond prior architecture inventory
- External references:
  - `docs/upstream/Technical Documents/SENSORGLOB-Firmware & Protocol Specification` (reviewed 2026-04-12)
  - `docs/upstream/Technical Documents/SENSORGLOB-Domains & Hosting` (reviewed 2026-04-12)
  - `docs/upstream/Technical Documents/SENSORGLOB-KORE Super SIM` (reviewed 2026-04-12)
  - See cross-reference: `docs/investigations/2026-04-12-upstream-tech-docs-review.md`

## Findings

- The MQTT design is hub-centric, not individual-alarm-centric.
- Each controller or hub appears to have a serial-number-scoped command topic and response topic.
- The backend authenticates to the broker with a username and password from environment variables.
- The prototype controller also authenticates to the broker with a username and password, and uses the controller serial number as the MQTT client ID.
- I did not find message-level authentication, payload signing, HMAC validation, or certificate-based device identity checks in the backend MQTT handler.
- `verifyEventTriggerSource` is not a security check. It is workflow control and deduplication using Redis.
- There is evidence of provisioning and activation outside MQTT:
  - controllers can be registered in the backend by uploading serial and SIM mappings
  - activation URLs and QR code generation exist in the prototype tooling
- I did not find checked-in broker ACL or broker authorization config proving topic-level enforcement. That may exist only on the broker hosts or in an untracked private config.

### Topic organization

- The firmware spec defines two serial-scoped topics:
  - command topic: `sg/sas/cmd/{sn}`
  - response topic: `sg/sas/resp/{sn}`
- The prototype code uses the same topic scheme directly.
- The backend subscribes to a shared subscription over the response topics:
  - `$share/sensorGroup/sg/sas/resp/+`
- The backend derives the controller serial number from the last topic segment and uses that as the key identity in downstream logic.

### Command model

- MQTT payloads are JSON command objects.
- The visible command set includes:
  - `VERIFY`
  - `ADD`
  - `REMOVE`
  - `TEST`
  - `BATTERY`
  - `ALARMS`
  - `POWER`
  - `DISCONNECT`
  - `RECONNECT`
  - `LOW_BATTERY`
  - `TAMPERED`
  - `HUSH`
  - `ALERT`
  - `RESET`
  - `REBOOT`
- The backend both consumes controller-originated events and publishes commands back to the controller topic.
- The backend can also initiate commands based on operator actions in the application, such as remove, change, verify, and reboot flows.

### Authentication and verification visible in code

- Backend-side broker auth:
  - `src/config/app.ts` loads `MQTT_HOST`, `MQTT_USERNAME`, `MQTT_PASSWORD`, and `MQTT_PORT` from environment variables
  - `src/services/mqtt/subscriber.ts` connects to the broker using those values
- Prototype controller-side broker auth:
  - `mqtt-prototype/Program.cs` uses MQTT 3.1.1, client ID equal to controller serial number, optional TLS, and username/password credentials
  - `mqtt-prototype/Constants.cs` shows topic generation and configuration fields for broker credentials and TLS toggle
- What I did not find in the cloud MQTT handler:
  - message signatures
  - HMAC or checksum validation
  - nonce or replay-protection logic
  - mTLS or client certificate checks in the application code
  - payload-origin verification beyond trusting the broker-delivered topic and payload format

### What `verifyEventTriggerSource` actually does

- `verifyEventTriggerSource` uses Redis keys like `<controllerSerial>_HUB_VERIFY`.
- Its purpose is to decide whether certain follow-up steps should happen for workflows such as installation, manual tests, and scheduled tests.
- It helps suppress or allow downstream actions like chained `VERIFY` or `ALARMS` handling.
- It does not authenticate the controller, the MQTT client, or the payload.

### Provisioning and identity outside MQTT

- The backend has an admin API to import controller serial numbers and SIM IDs from a CSV in S3.
- `checkControllerAndupdate` stores the controller serial and SIM mapping in the application database.
- The prototype tooling shows a companion onboarding path:
  - generate activation URLs with serials
  - generate CSVs of serial and SIM mappings
  - upload those mappings to `/api/v1/admins/alarms/upload-controller`
- This suggests device identity is at least partly managed operationally through:
  - serial number
  - SIM ID
  - activation workflow
  - application database records
- That said, this is not the same as proving that incoming MQTT messages are cryptographically bound to that identity.

### Notable inconsistencies or gaps

- **QoS mismatch — formally confirmed (2026-04-12):** the firmware spec explicitly states "All commands will use QoS 2 – Exactly Once unless otherwise specified." The backend subscribes at `QoS 0` and its generic send helper publishes without an explicit QoS override. This is no longer a suspected drift — it is a confirmed deviation from the authoritative spec.
- The prototype supports TLS as a toggle, but the backend MQTT client code does not explicitly configure TLS options; it may rely on the broker URL scheme or environment value instead.
- No broker-side ACL configuration was found in the checked-in repositories, so topic-level authorization remains unverified from repo evidence alone.
- **MQTT port 18756 (confirmed 2026-04-12 from Domains doc):** all MQTT hostnames (`mqtt.sensorglobal.com`, `mqttsandbox.sensorglobal.com`, etc.) run on port **18756**, not the standard 1883 or 8883. This non-standard port must be open in every environment's security group.
- **KORE SIM branding:** the firmware spec still refers to "Twilio Super SIM." The KORE Super SIM doc confirms the SIM product is now KORE (previously Twilio). The firmware spec is outdated on this point. KORE connects hubs via LTE Cat-M with carrier preference: Telstra first, then Optus, then Vodafone (OPLMN-forced initial Telstra preference).
- **FAULT and LOW_BATTERY are spec'd events with no backend handler (confirmed 2026-04-12):** see `2026-04-09-alarm-event-traces.md` for detail. `FAULT CODE 10` (cleaning required) and standalone `LOW_BATTERY` (hub 10%/5% thresholds) are both defined in the firmware spec but silently discarded by the backend.

## Risks Or Constraints

- The trust model currently visible in code appears to rely heavily on broker authentication plus topic naming conventions.
- If the broker does not enforce strong per-client topic ACLs, a client with valid broker credentials could potentially publish into another controller’s response topic by spoofing the serial in the topic path.
- If TLS is not consistently enforced in production, broker credentials and device traffic could be more exposed than intended.
- The mismatch between the written protocol spec and the backend’s observed QoS behavior may affect delivery guarantees or may simply mean the implementation drifted from the spec.
- **QoS mismatch is a confirmed risk (2026-04-12):** the firmware spec explicitly requires QoS 2 (exactly-once). The backend uses QoS 0. Under network instability, alarm events or commands could be lost silently without the backend or hub detecting the loss.
- Because broker-side configuration is not present here, some real security controls may exist but are not visible in this repository.

## Cost Optimization Opportunities

- None from this pass directly, other than the broader broker-environment duplication already noted in architecture and cost investigations.

## Recommended Next Steps

- Inspect the MQTT broker hosts directly to confirm:
  - whether TLS is enforced
  - whether per-device or per-environment credentials are used
  - whether topic ACLs bind a client to only its own `cmd` and `resp` topics
- Pull environment configuration or secret-management references for `MQTT_HOST` and related values to see whether the backend uses `mqtt://` or `mqtts://`.
- Trace one full controller onboarding flow to determine how activation, serial registration, and SIM registration are tied together operationally.
- Compare backend publish/subscribe QoS behavior with live broker settings and device expectations.

## Open Questions

- Does each physical controller have unique broker credentials, or do multiple controllers share the same broker username and password?
- Does the broker enforce ACLs that restrict a controller to only `sg/sas/cmd/<its-serial>` and `sg/sas/resp/<its-serial>`?
- Is TLS mandatory in production for all MQTT clients?
- Is there any device-side cryptographic identity not represented in these repos, such as secure elements, certs, or signed payloads?
