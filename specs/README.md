# Spec Library Charter

## What this library is
Every project cloned from openvibe carries a durable `specs/<capability>.md` library. One file per capability. Each spec describes the system's CURRENT externally-visible behavior in a format agents can read as compressed context — replacing the need to re-grep the whole codebase every task.

## File naming
`specs/<capability>.md`, kebab-case, one file per capability. Examples: `specs/user-auth.md`, `specs/data-export.md`, `specs/payments.md`. New capabilities get new files as the project grows.

## Format skeleton

```
# <Capability> Specification

## Purpose
<1-2 sentences on what this capability does and why it exists>

## Requirements

### Requirement: <name>
<What the system SHALL do. Use RFC 2119 keywords: MUST/SHALL for absolute requirements, SHOULD for recommended, MAY for optional.>

#### Scenario: <name>
- **GIVEN** <precondition>
- **WHEN** <trigger>
- **THEN** <expected outcome>

#### Scenario: <error case>
- **WHEN** <trigger>
- **THEN** <observable error behavior>
```

## Hard rules
1. **Scenarios MUST use exactly `####` (4 hashtags).** Using 3 hashtags or bullet-form scenarios is a silent format error — reviewers will miss them.
2. **Every requirement MUST have at least one scenario.** A requirement without a scenario is untestable and decorative.

## What a spec IS (and is NOT)
A spec is a **behavior contract**, not an implementation plan.

**Include:** observable behavior users or downstream systems rely on; inputs, outputs, and error conditions that are part of the system's guarantees; scenarios that can be tested (each scenario is a potential test case). Write scenarios concrete enough to verify against — e.g. "WHEN user clicks Export THEN a CSV downloads with all rows," not vague "THEN it works." Vague scenarios turn the library into a token-waster; reading useless specs costs tokens for nothing.

**Avoid:** internal class/function names, library or framework choices, step-by-step implementation details. Those belong in PLAN.md. If the implementation can change without changing externally visible behavior, it does NOT belong in the spec.

## Progressive rigor
**Lite spec (default):** Short behavior-first requirements, clear scope, a few concrete scenarios. The right size for most capabilities. **Full spec:** Only for cross-cutting changes, API/contract changes, migrations, or security/privacy concerns where ambiguity would be expensive. Default Lite.

## In-place update convention
Specs always describe CURRENT behavior. When a change alters a capability's externally-visible requirements, the build agent edits the spec IN PLACE: **add** a new `### Requirement:` block for new behavior; **edit** an existing requirement's text/scenarios for changed behavior; **remove** a requirement block for removed behavior. Git history is the delta and audit trail. No separate delta files, no ADDED/MODIFIED/REMOVED delta sections in the spec, no archive step. The spec IS the current truth.

## Pipeline contract
- **@plan** reads `specs/*.md` as context before grepping the codebase. Specs are a head start, not an authority — the plan agent must still verify specifics against actual code. When a task alters a capability's behavior, the plan directs a spec update via `specs/<capability>.md · SPEC UPDATE: <add/modify/remove which requirements>` in FILES TO TOUCH.
- **@build** applies the spec update in place (format in this README). Plan-directed spec updates are load-bearing — they are part of building what the plan specifies, not optional polish.
- **@1-plan-review** flags plans that change behavior with no spec-update directive, or that silently contradict existing spec requirements. Spot-checks directive form/grounding: modify/remove targets must exist, add targets must not duplicate, create targets must not already exist.
- **@2-code-review** verifies spec directives were correctly applied (form and correct-application only — never traces whether code satisfies spec behavior, which is @1-plan-review's lane). Sanctioned spec files missing from the diff = OMISSION. Unsanctioned spec touches = OUT-OF-SCOPE. Misapplied edits = DEVIATION. Behavior changed with no directive = CROSS-CUTTING (route to @plan).

## Example
A 1-requirement, 2-scenario spec — the default Lite size:

```
# Data Export Specification

## Purpose
Allow users to export their data in standard, machine-readable formats for portability and compliance.

## Requirements

### Requirement: CSV Export
The system SHALL generate and download a CSV file containing all of the authenticated user's records.

#### Scenario: Successful export
- **WHEN** user clicks "Export" on the dashboard
- **THEN** a CSV file downloads containing all records belonging to the authenticated user
- **AND** each row includes id, created_at, and summary fields

#### Scenario: No data to export
- **WHEN** user with zero records clicks "Export"
- **THEN** a CSV file downloads containing only the header row
```
