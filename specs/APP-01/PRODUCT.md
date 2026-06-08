# APP-02: OCPP 1.6 compliance coverage and validation correctness

## Summary

Bring the v1_6 crate surface and validation behavior in line with the documented OCPP 1.6 scope. The library should make it clear whether it supports core OCPP 1.6 only or OCPP 1.6 plus Security Whitepaper edition 2, and it should not accept or reject payloads contrary to the OCPP 1.6 schemas and operation rules.

## Problem

The current v1_6 implementation serializes its existing core message structs successfully against the bundled schemas, but the public scope is ambiguous and several runtime validations do not match OCPP 1.6 behavior.

The library advertises OCPP 1.6 support while omitting Security Whitepaper edition 2 operations such as certificate management, signed firmware, log upload, and security events. At the same time, consumers can construct or deserialize messages that violate OCPP 1.6 constraints unless they separately run JSON Schema validation and additional domain checks.

This creates two user-facing risks: integrators may assume the crate covers the full modern OCPP 1.6 message set, and applications may send or accept protocol-invalid payloads despite using typed Rust structs.

## Non-goals

- Implementing OCPP 2.0.1 or OCPP 2.1 behavior. This spec is only about the `v1_6` module.
- Changing the OCPP-J WebSocket envelope or transport behavior.
- Adding a runtime CSMS or Charge Point state machine.
- Enforcing deployment-specific business rules such as the physical maximum number of connectors at a site.
- Treating validation as authorization, tenant isolation, or application security.
- Removing existing public structs without a compatibility plan.
- Changing valid enum serialization strings that already match the OCPP 1.6 schemas.

## Figma

Figma: none. This is a Rust library protocol-compliance issue with no UI surface.

## Behavior

1. The documented v1_6 scope is explicit. The crate either supports OCPP 1.6 core only or includes the OCPP 1.6 Security Whitepaper edition 2 operations; the README, crate docs, and feature documentation must not imply unsupported messages are present.

2. If Security Whitepaper edition 2 support is in scope, v1_6 exposes typed request and response structs for: `CertificateSigned`, `DeleteCertificate`, `ExtendedTriggerMessage`, `GetInstalledCertificateIds`, `GetLog`, `InstallCertificate`, `LogStatusNotification`, `SecurityEventNotification`, `SignCertificate`, `SignedFirmwareStatusNotification`, and `SignedUpdateFirmware`.

3. If Security Whitepaper edition 2 support is out of scope, the omission is documented as a known limitation and core OCPP 1.6 support remains unchanged.

4. v1_6 schema fixtures cover every v1_6 operation the crate claims to support. There is no mismatch between documented operation coverage, message modules, and schema-validation tests.

5. `DataTransfer.req.vendorId` enforces the OCPP 1.6 `CiString255Type` limit.

6. `DataTransfer.req.data` and `DataTransfer.conf.data` are treated as OCPP 1.6 `Text` with no schema-defined maximum length. The library must not reject valid data solely because it is longer than 255 characters.

7. `GetConfiguration.req.key` validates each key string as `CiString50Type`; it must not treat 50 as the maximum number of keys unless an explicit application-level rule is added separately.

8. `GetConfiguration.conf.unknownKey` validates each unknown key string as `CiString50Type`; it must not treat 50 as the maximum number of unknown keys unless an explicit application-level rule is added separately.

9. Message fields whose OCPP 1.6 operation table says `connectorId` is `integer (> 0)` reject zero when runtime validation is used. This includes `StartTransaction.req`, `RemoteStartTransaction.req`, `TriggerMessage.req`, and `UnlockConnector.req`.

10. `UnlockConnector.req.connectorId` does not impose a protocol-level maximum such as 20 unless that limit is clearly modeled as a separate deployment-specific constraint outside OCPP 1.6.

11. `RemoteStartTransaction.req.chargingProfile`, when present, only validates as protocol-correct when `chargingProfilePurpose` is `TxProfile`.

12. Incoming v1_6 JSON payloads with undeclared fields are rejected by the typed deserialization/validation path or the crate clearly documents that consumers must run JSON Schema validation before accepting inbound payloads.

13. Empty arrays are rejected for fields with cardinality `1..*`, including `MeterValues.req.meterValue` and nested `MeterValue.sampledValue`, when runtime validation is used.

14. The Heartbeat module path is discoverable using the OCPP operation spelling. Existing `heart_beat` users are not broken without a deprecation/compatibility path.

15. Existing schema-valid examples for the current core OCPP 1.6 messages continue to serialize and validate successfully.

16. The all-feature test suite continues to pass after the compliance changes.
