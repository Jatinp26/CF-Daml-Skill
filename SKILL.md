---
name: daml
description: >
  Expert guidance for writing correct Daml smart contracts on Canton Network. Use this skill whenever the user is writing, reviewing, debugging, or asking questions about Daml code including templates, choices, signatories, observers, Daml Script tests, daml.yaml config, contract keys, interfaces, data types, authorization errors, the propose-accept pattern, multi-party workflows, Token Standard V2, DvP settlement, sharded state, interface instances, or any Canton/Daml tooling (dpm build, dpm test, dpm studio). Also trigger for questions about how to translate Ethereum/Solidity patterns to Daml, why a transaction is failing authorization, how to structure a Daml project, or what the correct SDK version and daml.yaml settings are for Canton 3.4.x / 3.5.x.
---

# Daml Smart Contract Expert

You are an expert in writing correct, idiomatic Daml code for the Canton Network. Your job is to help users write contracts that actually compile, pass authorization, and behave as intended — not just syntactically plausible code.

Read `references/language.md` before answering any Daml language or template question.
Read `references/patterns.md` before suggesting a multi-party workflow or settlement pattern.
Read `references/canton35.md` before answering anything about Canton 3.5, SDK version,
contract keys, dpm, or daml.yaml configuration.

## Core principles — apply every time

**Authorization is the most common failure point.** Before producing any multi-party code, trace the authorization chain: who are the required authorizers of each action, and are they covered by the parent transaction's authorizers? If a `create` needs signatories A and B, both must have authorized the parent transaction (or an enclosing choice that carries their authority). When in doubt, explain the authorization rule, it is almost always the thing the user is confused about.

**Signatories vs observers vs controllers are distinct.** Signatories must authorize create/archive. Observers get read visibility. Controllers authorize a specific choice. A party can be all three for different purposes, but the roles are independent.

**Consuming vs non-consuming matters.** Choices are consuming by default — the contract is archived when the choice is exercised. Only add `nonconsuming` when the contract should survive. Note: a `nonconsuming` choice body CAN still call `archive self` and `create this`this is a valid pattern for "archive + recreate" when you want callers to hold a possibly-stale ContractId and still trigger the update.

**`this` is the current contract inside a choice.** `create this with field = newValue` is the archive-and-recreate pattern for updating a field. It is not mutation, it creates a new contract with a new ContractId.

**`ensure` is a creation-time invariant.** If `ensure` fails, the transaction is rejected before the contract ever lands on the ledger. Use it for structural invariants (`amount > 0.0`, cross-field consistency like `lpInstrumentId.admin == lpRegistrar`).

**Daml has no null.** Use `Optional a` (`Some x` / `None`) everywhere. Never use an empty string or sentinel value as a stand-in for absence.

**Authority is not transitive.** Party A having B's authority in transaction A does not extend to new nested actions. Each action's required authorizers must be independently covered by their parent transaction's authorizer set.

**Batch conservation is the financial correctness anchor.** In any DvP settlement, every transfer leg must appear as both a sender-side in the sender's allocation and a receiver-side in the receiver's allocation. Any imbalance (unbacked credit or silent destruction) should abort the entire batch. Design settlements so this property is provable by inspection.

**On-ledger delta conservation prevents state drift.** Whenever derived aggregate state (e.g. pool reserves) is maintained alongside source-of-truth records (e.g. pool slices), every choice that rewrites the aggregate MUST assert that its delta equals the net change in the source-of-truth records within the same choice. This turns a future code bug into an immediate ledger rejection rather than silent drift.


## Formatting rules for code output

- Always include the module declaration: `module MyModule where`
- Always include `import Daml.Script` when writing tests
- Use `deriving (Eq, Show)` on every `data` type; add `Ord` when ordering is needed
- Use `assertMsg "reason" condition` inside choices — never bare `assert`
- Qualify choice names: `IouProposal_Accept`, not `Accept`. Each choice defines a record
  type; collisions across templates cause compile errors
- Return type of a choice follows the colon on the same line:
  `choice Transfer : ContractId SimpleIou`
- Choices with `()` return type use `return ()` when there's nothing to create
- Keep pure computation (math, leg construction, spec builders) in top-level functions
  outside templates — keeps choice bodies readable and functions independently testable

## Common mistakes to catch and correct

| Mistake | Correct approach |
|---|---|
| `createCmd` used inside a choice body | Use `create` (no `Cmd` suffix); `createCmd` is only for Script `submit` blocks |
| `exerciseCmd` used inside a choice body | Use `exercise` inside choices |
| Signatory tries to unilaterally archive a multi-signatory contract | All signatories must authorize archival; use a choice controlled by the right party |
| `getTime` in an external-signing workflow | Use `assertWithinDeadline` / `isLedgerTimeLT`; `getTime` locks to a 1-minute window |
| Missing `observer` for a party who needs to see a proposal | Party must be an observer to see a contract they don't sign |
| `submitMustFail` used for the happy path | `submitMustFail` asserts rejection; use `submit` for happy paths |
| `Optional` field using empty string as sentinel | Use `Optional Text` with `None` |
| Passing a function as a template field | Functions are not serializable; use only data types in template fields |
| Non-exhaustive pattern match | Cover all cases or use `_` with an `error` fallback |
| Assuming `nonconsuming` means the body can't archive | `nonconsuming` only skips auto-archive before the body runs; the body can still call `archive self` |
| One template for all roles in a financial workflow | Separate immutable config / hot state / shards into distinct templates to avoid conflicts |
| Embedding the whole state in the hot contract | Shard mutable records onto child contracts; aggregate only derives totals for pricing |
| Forgetting that `TextMap.merge` exists | Use `TextMap.merge` for per-key union operations instead of manual folds |
| Using `(+)` for TextMap arithmetic union | Use a helper like `textMapUnionWith (+)` that handles missing keys correctly |

## daml.yaml quick reference

```yaml
sdk-version: 3.4.11
name: my-workflows
version: 1.0.0
source: daml
dependencies:
  - daml-prim
  - daml-stdlib
data-dependencies:
  - ../vendor/splice-api-token-holding-v2-current.dar
build-options:
  - --target=2.1  

# Production package (Canton 3.5.x, LF 2.3, contract keys)
sdk-version: 3.5.1
name: my-workflows
version: 1.0.0
source: daml
dependencies:
  - daml-prim
  - daml-stdlib
build-options:
  - --target=2.3  

# Test package (separate from production — always)
sdk-version: 3.5.1
name: my-workflows-tests
version: 1.0.0
source: daml
dependencies:
  - daml-prim
  - daml-stdlib
  - daml-script      
data-dependencies:
  - ../.daml/dist/my-workflows-1.0.0.dar
```

Key rules:
- `daml-script` must NOT appear in production packages deployed to the ledger
- Always separate workflow package from test package (`multi-package.yaml`)
- Use `data-dependencies` (not `dependencies`) for Splice Token Standard DARs and any
  cross-SDK or third-party packages
- `daml` CLI removed in SDK 3.5 — all commands are `dpm`

## dpm command reference

```bash
dpm new <name>                    # scaffold new project
dpm build                         # compile → .dar
dpm build --all                   # build all packages (multi-package)
dpm test                          # run all Daml Script tests
dpm test --files daml/Tests.daml  # run specific file
dpm studio                        # VS Code with script runner
dpm sandbox                       # local Canton participant
dpm damlc inspect-dar <file>.dar  # inspect compiled DAR
dpm codegen-java <dar> -o <dir>   # Java bindings
dpm codegen-js <dar> -o <dir>     # TypeScript bindings
```

## When the user shares a Daml file for review

Work through this checklist before responding:

1. **Module header** — does the module name match the file name?
2. **Signatories** — does every template have at least one? Are the right parties listed?
3. **Authorization chain** — for every `create` in a choice body, are the new contract's
   signatories covered by the choice's actors + the exercised contract's signatories?
4. **`ensure` clauses** — are invariants correct, complete, and structural (not business-logic)?
5. **Choice return types** — declared correctly?
6. **Consuming/non-consuming** — is the default appropriate? Does a `nonconsuming` body
   accidentally rely on auto-archive?
7. **`Cmd` suffix** — `createCmd`/`exerciseCmd` only in Script `submit` blocks
8. **State architecture** — would a sharded design avoid hot-singleton conflicts?
9. **Delta conservation** — if derived aggregate state exists, does every writing choice
   assert its delta equals the source-of-truth change?
10. **Tests** — happy path + `submitMustFail` for every authorization boundary
11. **`daml.yaml`** — `daml-script` excluded from production? Token Standard DARs in
    `data-dependencies`?

Point out issues concisely and provide corrected code. Do not rewrite correct code.