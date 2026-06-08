# APP-02: Tech Spec

## Context

See `PRODUCT.md` for the behavior contract. APP-02 captures the OCPP 1.6 review findings for the current `v1_6` module: coverage needs to be explicit, and runtime validation should match the OCPP 1.6 operation tables and JSON schemas.

Implementation anchors in the current code:

- `src/lib.rs` — crate-level docs say the library aims to implement `ocpp 1.6` and `ocpp 2.0.1`; `v1_6` is feature-gated behind `#[cfg(feature = "v1_6")]`.
- `Cargo.toml` — feature list defines `v1_6 = []`, `v2_0_1 = []`, and `wip_v2_1 = []`.
- `src/v1_6/messages/mod.rs` — exports 28 core OCPP 1.6 message modules. It does not export Security Whitepaper edition 2 modules.
- `src/tests/schema_validation/schemas/v1.6/json/` — contains 56 schemas for the 28 core request/response pairs.
- OCPP 1.6 reference comparison used 39 operations / 78 messages; the missing v1_6 operations are Security Whitepaper edition 2 operations.
- `src/v1_6/messages/data_transfer.rs` — `vendor_string` serializes as `vendorId`; `data` is currently validated with `max = 255` even though OCPP 1.6 `Text` has no schema-defined maximum.
- `src/v1_6/messages/get_configuration.rs` — `key: Option<Vec<String>>` and `unknown_key: Option<Vec<String>>` use `#[validate(length(min = 1, max = 50))]`, which validates vector length rather than per-item `CiString50Type` length.
- `src/v1_6/messages/start_transaction.rs` — `connector_id: u32` has no runtime range validation even though the operation table says `integer (> 0)`.
- `src/v1_6/messages/remote_start_transaction.rs` — optional `connector_id` has no runtime range validation; `charging_profile` comment says `ChargingProfilePurpose MUST be set to TxProfile`, but this is not enforced.
- `src/v1_6/messages/trigger_message.rs` — optional `connector_id` has no runtime range validation and the struct does not derive `Validate`.
- `src/v1_6/messages/unlock_connector.rs` — `connector_id` uses `#[validate(range(min = 0, max = 20))]`, which permits zero and adds a max not present in the OCPP 1.6 operation schema.
- `src/v1_6/messages/meter_values.rs`, `src/v1_6/types/meter_value.rs`, and `src/v1_6/types/sampled_value.rs` — required arrays do not currently enforce non-empty `1..*` cardinality through runtime validation.
- `src/v1_6/messages/heart_beat.rs` and `src/v1_6/messages/mod.rs` — module path is `heart_beat`, while the OCPP operation spelling is `Heartbeat`.
- The current v1_6 targeted validation command passed: `cargo test --features v1_6 --all-targets` with 57 tests.
- The current v1_6 clippy command passed: `cargo clippy --features v1_6 --all-targets -- -D warnings`.
- The current all-feature test command passed: `cargo test --all-features --all-targets` with 882 tests.

## Proposed changes

1. **Decide and document v1_6 coverage scope.** Update crate docs, README content, and feature notes so the public claim is either “OCPP 1.6 core profile” or “OCPP 1.6 including Security Whitepaper edition 2”. Do not leave the scope ambiguous.

2. **If full OCPP 1.6 support is chosen, add missing Security Whitepaper edition 2 modules.** Add typed request/response structs, module exports, schemas, and schema-validation tests for:

   - `CertificateSigned`
   - `DeleteCertificate`
   - `ExtendedTriggerMessage`
   - `GetInstalledCertificateIds`
   - `GetLog`
   - `InstallCertificate`
   - `LogStatusNotification`
   - `SecurityEventNotification`
   - `SignCertificate`
   - `SignedFirmwareStatusNotification`
   - `SignedUpdateFirmware`

   The canonical source for fields and enums should be the OCPP 1.6 Security Whitepaper edition 2 schemas and operation docs.

3. **If core-only support is chosen, add a known-limitation note.** Keep the existing 28 core message modules, but explicitly state that Security Whitepaper edition 2 messages are not implemented in `v1_6` yet.

4. **Fix `DataTransfer` validation.** In `src/v1_6/messages/data_transfer.rs`, add `CiString255Type` validation to the serialized `vendorId` field and remove the schema-inconsistent max-length validation from request `data`.

   ```rust path=null start=null
   #[validate(length(min = 1, max = 255))]
   #[serde(rename = "vendorId")]
   pub vendor_string: String,

   #[serde(skip_serializing_if = "Option::is_none")]
   pub data: Option<String>,
   ```

5. **Fix per-item validation for `GetConfiguration` key arrays.** Replace vector-length validation with per-string `CiString50Type` validation for `key` and `unknown_key`. Prefer a small reusable newtype if `validator` cannot express nested vector item constraints cleanly.

   ```rust path=null start=null
   #[derive(serde::Serialize, serde::Deserialize, Validate, Debug, Clone, PartialEq)]
   pub struct CiString50(#[validate(length(min = 1, max = 50))] pub String);
   ```

   If introducing public newtypes is too breaking, add custom validation functions that iterate through the `Vec<String>` and validate each item.

6. **Correct connector ID runtime validation.** Add or update `Validate` derivations and range checks for connector fields whose OCPP 1.6 operation table says `integer (> 0)`:

   - `StartTransactionRequest.connector_id`: `#[validate(range(min = 1))]`
   - `RemoteStartTransactionRequest.connector_id`: validate `Some(value) >= 1`
   - `TriggerMessageRequest.connector_id`: derive `Validate` and validate `Some(value) >= 1`
   - `UnlockConnectorRequest.connector_id`: change `min = 0, max = 20` to protocol-level `min = 1`, unless a separate deployment-specific max is added outside this struct.

7. **Enforce `RemoteStartTransaction` charging profile purpose.** Add custom validation so `RemoteStartTransactionRequest.charging_profile` is accepted only when `charging_profile_purpose == ChargingProfilePurposeType::TxProfile`. This validation should not reject absent `charging_profile`.

8. **Add inbound additional-property protection or document the validation contract.** Prefer adding `#[serde(deny_unknown_fields)]` to v1_6 request, response, and nested data structs so direct `serde_json` parsing follows the OCPP 1.6 `additionalProperties: false` schemas. If that is too breaking, document that consumers must run JSON Schema validation before accepting inbound payloads.

9. **Enforce `1..*` cardinality at runtime.** Add validation for required non-empty arrays, especially:

   - `MeterValuesRequest.meter_value`
   - `MeterValue.sampled_value`
   - `ChargingSchedule.charging_schedule_period`

   Review other v1_6 arrays against the operation docs and schemas before finalizing.

10. **Add a discoverable `heartbeat` module path.** Add `src/v1_6/messages/heartbeat.rs` or a public alias that exposes the same `HeartbeatRequest` and `HeartbeatResponse` types. Preserve `heart_beat` as a compatibility path initially and mark it deprecated only if the crate’s semver policy allows.

11. **Add regression tests for invalid examples, not only valid serialization.** The current schema tests prove valid examples serialize correctly. Add tests that assert invalid values are rejected by runtime validation or deserialization, including:

   - `DataTransferRequest.vendorId` length 256 rejects.
   - `DataTransferRequest.data` length greater than 255 is accepted by protocol validation.
   - `GetConfigurationRequest.key = vec![long_51_char_key]` rejects.
   - `GetConfigurationRequest.key` with more than 50 short keys is not rejected because of the CiString limit alone.
   - `StartTransactionRequest.connector_id = 0` rejects.
   - `UnlockConnectorRequest.connector_id = 0` rejects.
   - `RemoteStartTransactionRequest` with `chargingProfilePurpose != TxProfile` rejects when a charging profile is present.
   - Inbound JSON with an unknown field rejects if `deny_unknown_fields` is adopted.
   - `MeterValuesRequest.meter_value = []` rejects.

## End-to-end flow

```mermaid
sequenceDiagram
    participant User as Crate consumer
    participant Struct as v1_6 typed struct
    participant Validate as Runtime validation
    participant Schema as OCPP 1.6 schema/spec rules

    User->>Struct: Build or deserialize OCPP 1.6 payload
    Struct->>Validate: validate() or typed inbound parser
    Validate->>Schema: Enforce CiString, connectorId, cardinality, and domain rules
    Schema-->>Validate: Valid or protocol violation
    Validate-->>User: Accept valid OCPP payloads; reject invalid payloads with clear errors
```

## Risks and mitigations

1. **Adding Security Whitepaper edition 2 messages expands scope.** Mitigation: decide core-only vs full support first. If core-only is chosen, document the limitation instead of adding partially implemented security messages.

2. **`deny_unknown_fields` may be a breaking change for consumers relying on lenient deserialization.** Mitigation: treat it as a semver-relevant change, add release notes, or provide a strict parser path before changing default serde behavior.

3. **Public newtypes for CiString values may be breaking.** Mitigation: use custom validation functions on existing `String` / `Vec<String>` fields if preserving the public API is more important.

4. **Protocol constraints and deployment constraints can be confused.** Mitigation: only encode OCPP 1.6 constraints in protocol structs. Site-specific limits such as “max 20 connectors” should live in application configuration outside the crate.

5. **Schema tests alone can miss domain rules.** Mitigation: add runtime validation tests for rules that JSON Schema does not express, such as `RemoteStartTransaction.chargingProfilePurpose == TxProfile`.

6. **Compatibility path for `heart_beat` can duplicate modules.** Mitigation: re-export from one implementation module rather than maintaining two independent files.

## Testing and validation

Each `PRODUCT.md` behavior maps to concrete checks:

- Behaviors 1, 3: Documentation review for README, crate-level docs, and feature documentation. Confirm they state core-only support if security-extension messages remain missing.
- Behaviors 2, 4: Coverage test or script comparing supported v1_6 operation names against message modules and schema files. If full support is chosen, expected count is 39 operations / 78 request-response schemas.
- Behaviors 5, 6: Unit tests for `DataTransferRequest` runtime validation and schema validation.
- Behaviors 7, 8: Unit tests for per-item `GetConfiguration` key validation.
- Behaviors 9, 10: Unit tests for connectorId zero rejection and absence of protocol-level max connector cap.
- Behavior 11: Unit test for `RemoteStartTransactionRequest` with present `charging_profile` and non-`TxProfile` purpose.
- Behavior 12: Deserialization test with an undeclared field, or documentation test/review if strict serde parsing is intentionally not adopted.
- Behavior 13: Unit tests for empty `meterValue`, empty nested `sampledValue`, and empty `chargingSchedulePeriod`.
- Behavior 14: Compile test or public API test showing `v1_6::messages::heartbeat::HeartbeatRequest` is reachable while `heart_beat` compatibility remains intact.
- Behaviors 15, 16: Run the existing valid schema tests and all-feature tests.

Recommended command set:

```bash
cargo test --features v1_6 --all-targets
cargo clippy --features v1_6 --all-targets -- -D warnings
cargo test --all-features --all-targets
git diff --check
```

For review-only validation of this spec, the current baseline already passed:

```text
cargo test --features v1_6 --all-targets       # 57 passed
cargo clippy --features v1_6 --all-targets -- -D warnings
cargo test --all-features --all-targets        # 882 passed
```

## Follow-ups

- Consider adding a generated coverage report that compares operation docs, schemas, Rust modules, and tests for every protocol version.
- Consider adding reusable constrained string types for OCPP primitive aliases such as `CiString20Type`, `CiString50Type`, `CiString255Type`, and `IdToken`.
- Consider adding an explicit strict inbound parsing API that performs serde deserialization plus OCPP validation in one call.
- Consider adding OCTT-oriented negative tests for OCPP 1.6 edge cases once core validation is corrected.
