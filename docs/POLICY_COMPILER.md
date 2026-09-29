# Payroll Policy Compiler

Admins should not have to hand-format low-level policy structs for the
payroll contract. `compilePayrollPolicy` converts a human-readable
`PayrollPolicyInput` — settlement windows in whole seconds, capacity limits,
reserve requirements, audit settings, and optional compensation
effective/expiration dates — into a validated, deterministic
`CompiledPayrollPolicy` payload ready to pass to a contract call.

## Core concepts

| Concept | What it is |
|---|---|
| `PayrollPolicyInput` | The human-readable policy an admin authors: `policyId`, `asset`, `settlementWindow`, `capacityLimits`, `reserveRequirements`, `auditSettings`, plus optional `effectiveDate` and `endDate`. |
| `CompiledPayrollPolicy` | The validated, deterministic output. Bigint fields are serialized as decimal strings and dates as epoch-ms (`effectiveDateMs` / `endDateMs`, present only when provided) so the result is safe to `JSON.stringify` for logging or snapshotting. |
| `compilePayrollPolicy` | Validates and compiles the input. Returns `{ ok: true, value }` or `{ ok: false, errors }` — every problem is collected, not just the first. |
| `compilePayrollPolicyOrThrow` | Throws a `PolicyCompileError` (with every other error attached at `error.context.allErrors`) on failure; returns the compiled policy directly on success. |

## Minimal example

```ts
import { compilePayrollPolicy } from "@zk-payroll/core";

const result = compilePayrollPolicy({
  policyId: "default",
  asset: "native",
  effectiveDate: "2026-10-01T00:00:00Z",
  endDate: "2027-10-01T00:00:00Z",
  settlementWindow: { minDelaySeconds: 60, maxOpenSeconds: 3600 },
  capacityLimits: {
    maxBatchSize: 500,
    maxTotalPayout: 1_000_000_0000000n,
    maxPerRecipientPayout: 50_000_0000000n,
  },
  reserveRequirements: { minReserveBalance: 100_000_0000000n },
  auditSettings: { auditRequired: true, retentionDays: 365 },
});

if (result.ok) {
  await contract.setPolicy(result.value);
} else {
  for (const error of result.errors) {
    console.error(`${error.field}: ${error.message}`);
  }
}
```

Two ready-to-use fixtures ship with the module for tests and quick starts:
`MINIMAL_POLICY_FIXTURE` (the smallest policy that passes every rule) and
`STRICT_POLICY_FIXTURE` (a realistic production-grade policy with audit
logging and reserves enabled).

## Validation rules

- `asset` must normalize via [`normalizeAssetIdentity`](./MULTI_ASSET.md) —
  `"native"`, a `"CODE:ISSUER"` pair, or a Soroban contract id.
- `settlementWindow.minDelaySeconds` must be `>= 0` and strictly less than
  `maxOpenSeconds`.
- `capacityLimits.maxBatchSize` must be a positive integer;
  `maxTotalPayout` and `maxPerRecipientPayout` must be positive bigints, and
  `maxPerRecipientPayout` cannot exceed `maxTotalPayout`.
- `reserveRequirements.minReserveBalance` must be a non-negative bigint and
  cannot exceed `capacityLimits.maxTotalPayout`.
- `auditSettings.retentionDays` must be a non-negative integer, and must be
  at least `1` when `auditRequired` is `true`.
- `effectiveDate` (optional) must be a parseable ISO 8601 date/datetime
  string or epoch-milliseconds value, and must not be in the past.
- `endDate` (optional) requires `effectiveDate` to be set, must be parseable,
  and must be strictly after `effectiveDate`.

Failures of these date rules carry the `INVALID_EFFECTIVE_DATE` error code,
the offending field (`effectiveDate` or `endDate`), and a machine-readable
`context` (e.g. `effectiveDateMs`, `endDateMs`, `nowMs`) so dashboards can
highlight exactly what to fix.

## Reference time & determinism

`compilePayrollPolicy` is a pure function of its input — identical input
always compiles to identical output. Because the effective-date "not in the
past" check needs a reference time, `compilePayrollPolicy` accepts an
optional `options.now` (epoch milliseconds):

- Omitted: the current wall-clock time is used (typical for live flows).
- Provided: the fixed value is used, keeping compilation fully deterministic
  for tests, replay, and backfill scenarios.

```ts
compilePayrollPolicy(input, { now: Date.parse("2026-09-15T12:00:00Z") });
```

## Determinism & snapshot coverage

The compiled payload remains safe to lock down with Jest snapshot tests
(`tests/policy-compiler.test.ts`). Update the snapshot deliberately with
`npx jest -u` whenever a schema change is intentional, and bump
`CompiledPayrollPolicy.schemaVersion` if the compiled shape changes
incompatibly. Effective-date tests live in
`tests/policy-effective-date.test.ts` and use a fixed `now` so they never
depend on the wall clock.
