# Daml Language Reference

## Table of contents

1. Native types
2. Data types (records, variants, Optional)
3. Functions and control flow
4. Templates — structure
5. Signatories and observers
6. Choices — full anatomy
7. Daml Script — testing API
8. Interfaces and interface instances
9. Contract keys (Canton 3.5+ / LF 2.3)
10. Standard library essentials
11. TextMap patterns (financial math)
12. Advanced idioms from production code

## 1. Native types

| Type | Notes |
|---|---|
| `Party` | Opaque identity — cannot be forged; do not parse or construct manually |
| `Text` | Unicode string |
| `Int` | Signed 64-bit integer |
| `Decimal` | Fixed-point: 28 digits before, 10 after decimal point. Alias for `Numeric 10` |
| `Bool` | `True` / `False` |
| `Date` | Calendar date |
| `Time` | Absolute UTC time |
| `RelTime` | Relative time difference |
| `ContractId a` | Typed ledger reference; `a` is the template or interface type |
| `Optional a` | `Some a` or `None` — Daml's null replacement |
| `AnyContractId` | Untyped cid; use `coerceContractId` to retype |

## 2. Data types

### Records

```daml
data Cash = Cash with
    currency : Text
    amount   : Decimal
  deriving (Eq, Show)
```

- `deriving (Eq, Show)` — always include; required for `==` and debug output
- Add `Ord` when you need `<`, `>`, sorting
- Field access: `cash.amount`; record update: `cash with amount = cash.amount * 2.0`
- Short-form field binding: `Cash with ..` in patterns binds all fields as local vars

### Variants (sum types / ADTs)

```daml
data PoolStatus
  = PS_Unfunded   -- no payload
  | PS_Active
  | PS_Paused
  deriving (Eq, Show)

data OrderStatus
  = OS_Pending
  | OS_Funded
  | OS_PartiallyFilled
  deriving (Eq, Show)
```

- Prefix variant constructors with their type name (`PS_`, `OS_`) to avoid ambiguity
- Prefer ADT status over `Text` — typos become compile errors
- Carry data in variants: `Settled Text` — the `Text` is extractable in case expressions

### Optional

```daml
findBySymbol : [Asset] -> Text -> Optional Asset
findBySymbol [] _ = None
findBySymbol (x::xs) sym =
  if x.symbol == sym then Some x else findBySymbol xs sym

-- Pattern-match in do-blocks:
case optValue of
  None    -> abort "not found"
  Some v  -> create Contract with field = v

-- Conditional action on Optional — clean idiom:
forA_ mOptionalCid archive          -- archive if Some, skip if None
forA_ mOptionalCid $ \cid -> do ... -- do something if Some
```

### Tuples

- `(Party, Text)` — 2-tuple; access with `._1`, `._2`
- Prefer named records for anything stored in templates or returned from choices
- Tuples fine for local function returns or intermediate grouping

## 3. Functions and control flow

### Function signatures

```daml
-- Always annotate top-level functions
constantProductOut : Decimal -> Decimal -> Int -> Decimal -> Decimal
constantProductOut reserveIn reserveOut feeBps inputAmount =
  let feeMultiplier = (10000.0 - intToDecimal feeBps) / 10000.0
      amountInAfterFee = inputAmount * feeMultiplier
  in floorDecimal10 ((amountInAfterFee * reserveOut) / (reserveIn + amountInAfterFee))
```

### Guards (readable multi-branch)

```daml
tier : Decimal -> Text
tier amount
  | amount < 0.0    = "Invalid"
  | amount == 0.0   = "Zero"
  | amount < 1000.0 = "Retail"
  | otherwise       = "Institutional"
```

### Case expression

```daml
describeStatus : PoolStatus -> Text
describeStatus s = case s of
  PS_Unfunded -> "Unfunded"
  PS_Active   -> "Active"
  PS_Paused   -> "Paused"
```

### `when` and `unless` — conditional actions

```daml
when   condition (void (create SomeContract with ...))
unless condition (abort "condition must be false")
```

### `void` — discard a return value

```daml
void (exercise cid SomeChoice with ...)
```

### `forA_` — mapped side effects, result discarded

```daml
forA_ sliceCids archive          -- archive every slice
forA_ items $ \item -> do ...    -- run action for each
```

### `forA` — mapped side effects, collect results

```daml
results <- forA itemCids $ \cid -> do
  item <- fetch cid
  pure item.amount
```

### `let` in do-blocks

```daml
do
  let fee = amount * 0.001
      net  = amount - fee
  create Payment with ...
```

### Recursion (tail-recursive accumulator)

```daml
-- Daml only supports recursion for top-level functions
drawFromSlicesAcc : Decimal -> [(ContractId Slice, Slice)] -> [...] -> SliceDraw
drawFromSlicesAcc remaining xs acc
  | remaining <= 0.0 = SliceDraw with fullyConsumed = reverse acc; boundary = None
  | otherwise = case xs of
      [] -> error "slices cannot cover the required amount"
      (cid, s) :: rest
        | remaining >= s.amount ->
            drawFromSlicesAcc (remaining - s.amount) rest ((cid, s) :: acc)
        | otherwise ->
            SliceDraw with
              fullyConsumed = reverse acc
              boundary = Some (cid, s, remaining, s.amount - remaining)
```

### List comprehensions

```daml
-- Collect pairs from a list:
legPairs = [(leg.instrumentId, leg.amount) | leg <- legs, leg.sender == account]
```

## 4. Templates — structure

```daml
template MyContract
  with
    issuer    : Party
    owner     : Party
    info      : MyRecord
    observers : [Party]
  where
    signatory issuer, owner
    observer  observers
    ensure info.quantity > 0.0
    -- Choices below
```

Key rules:

- `with` = data schema; `where` = rules and behavior
- `this` inside `where` = current contract's field values (full record)
- `self` inside a choice = current contract's `ContractId`
- Template fields are in scope throughout the `where` block

## 5. Signatories and observers

### Signatories

```daml
signatory issuer                      -- single
signatory issuer, owner               -- multiple — all must consent to create/archive
signatory [party1, party2]            -- from a list
signatory (signatory nestedContract)  -- extract from a nested record
signatory admin :: optional [] (\o -> [o]) account.owner  -- conditional (mint/burn accounts)
```

### Observers

```daml
observer counterparty
observer recipients                   -- [Party] list
observer admin :: optional [] identity publicReaders  -- conditional list
```

### Authorization model (formal)

Every action has **required authorizers**; every transaction has **authorizers**.

Rule: required authorizers of every action ⊆ authorizers of the parent transaction.

- **Create action**: required = signatories of the new contract
- **Exercise action**: required = controllers of the choice
- **Archive action**: required = signatories (implicit Archive choice)

Consequences of an exercise are authorized by:

- Actors of that choice (controllers who exercised it)
- PLUS signatories of the contract on which the choice was exercised

**Authority is NOT transitive.** B having A's authority in transaction X does not give BA's authority in a new nested action unless the new action's parent is authorized by A.

## 6. Choices — full anatomy

```daml
-- Consuming (default): contract archived before body runs
choice Transfer : ContractId SimpleIou
  with
    newOwner : Party
  controller owner
  do
    create this with owner = newOwner

-- Non-consuming: contract NOT auto-archived
-- The body CAN still call `archive self` and `create this` explicitly
nonconsuming choice DexPair_RecordTrade : ContractId DexPair
  with legNotionals : TextMap Decimal
  controller operator
  do
    archive self                    -- explicit archive in nonconsuming body
    create this with                -- create updated version
      accumulatedFees = bumped

-- Multiple controllers: ALL must authorize
choice MutualAction : ()
  controller owner, counterparty
  do return ()
```

### Inside a choice `do` block

| Expression | Use |
|---|---|
| `create T with ...` | Create a contract (NOT `createCmd`) |
| `exercise cid Choice with ...` | Exercise a choice (NOT `exerciseCmd`) |
| `fetch cid` | Read contract fields |
| `archive cid` | Shorthand for `exercise cid Archive` |
| `assertMsg "reason" cond` | Abort with message if False |
| `return v` / `pure v` | Return a value without side effects |
| `this` | Current contract's field record |
| `self` | Current contract's `ContractId` |
| `view <$> fetch ifaceCid` | Fetch and extract the view from an interface cid |

### `createCmd` vs `create` — the most common mistake

| | Where | Context |
|---|---|---|
| `createCmd` | Inside `submit` in Daml Script | Script/client side only |
| `create` | Inside choice `do` blocks | Ledger/server side |
| `exerciseCmd` | Inside `submit` in Daml Script | Script/client side only |
| `exercise` | Inside choice `do` blocks | Ledger/server side |

## 7. Daml Script — testing API

```daml
import Daml.Script

myTest : Script ()
myTest = script do
  alice <- allocateParty "Alice"
  bob   <- allocateParty "Bob"

  -- Single-party submit
  cid <- submit alice do
    createCmd MyContract with owner = alice; ...

  -- Multi-party submit (when multiple signatories required at creation)
  cid2 <- submitMulti [alice, bob] [] do
    createCmd MutualContract with party1 = alice; party2 = bob

  -- Exercise a choice
  result <- submit alice do
    exerciseCmd cid MyChoice with arg1 = "value"

  -- Assert a transaction MUST be rejected (negative test)
  submitMustFail bob do
    exerciseCmd cid RestrictedChoice

  -- Query
  Some contract <- queryContractId alice cid
  assert (contract.owner == alice)

  -- Time
  passTime (days 7)
  setTime (time (date 2026 Jan 1) 9 0 0)

  pure ()
```

### Key Script functions

| Function | Purpose |
|---|---|
| `allocateParty "Name"` | Create test party |
| `submit party do ...` | Submit as one party |
| `submitMulti [p1,p2] [] do ...` | Submit as multiple parties |
| `submitMustFail party do ...` | Assert transaction fails |
| `queryContractId party cid` | Fetch by ID → Optional |
| `queryFilter @T party pred` | Filter ACS |
| `assert cond` / `assertMsg "msg" cond` | Fail test if False |
| `passTime relTime` / `setTime t` | Advance or set ledger time |
| `getTime` | Current ledger time (1-min window constraint) |

## 8. Interfaces and interface instances

```daml
-- Define interface
data TokenView = TokenView with
    owner    : Party
    symbol   : Text
    quantity : Decimal
  deriving (Eq, Show)

interface IToken where
  viewtype TokenView
  getOwner    : Party
  getQuantity : Decimal

  choice IToken_Transfer : ContractId IToken
    with newOwner : Party
    controller getOwner this
    do ...

-- Implement on a template
template MyToken
  with
    owner    : Party
    symbol   : Text
    quantity : Decimal
  where
    signatory owner

    interface instance IToken for MyToken where
      view       = TokenView with owner; symbol; quantity
      getOwner   = owner
      getQuantity = quantity
```

### One template, multiple interface instances

A single template can satisfy multiple interfaces simultaneously:

```daml
template Registry with admin : Party; users : [Party]
  where
    signatory admin; observer users

    interface instance TransferFactory   for Registry where ...
    interface instance AllocationFactory for Registry where ...
    interface instance SettlementFactory for Registry where ...

-- All point to the same underlying contract:
cid <- create Registry with admin; users
let transferFac  = toInterfaceContractId @TransferFactory cid
    allocFac     = toInterfaceContractId @AllocationFactory cid
    settleFac    = toInterfaceContractId @SettlementFactory cid
```

### Interface conversion functions

| Function | Purpose |
|---|---|
| `toInterfaceContractId @I cid` | `ContractId T` → `ContractId I` (safe upcast) |
| `fromInterfaceContractId @T cid` | `ContractId I` → `ContractId T` (unsafe — use only when certain) |
| `toInterface @I value` | Template value → interface value |
| `fromInterface @T iface` | Interface value → `Optional T` |
| `fetchFromInterface @T cid` | Fetch + convert safely → `Optional (ContractId T, T)` |
| `coerceContractId cid` | `ContractId A` → `ContractId B` without fetch (use carefully) |
| `view <$> fetch ifaceCid` | Extract the viewtype from an interface contract |

Canton Network standard interfaces:

- **CIP-0056 / Token Standard V2** — implement for any fungible token or registry
- **CIP-0103** — dApp interoperability standard

## 9. Contract keys (Canton 3.5+ / LF 2.3)

Requires `--target=2.3` in `build-options`.

```daml
type AccountKey = (Party, Text)

template Account
  with
    bank    : Party
    number  : Text
    owner   : Party
    balance : Decimal
  where
    signatory bank, owner

    key (bank, number) : AccountKey
    maintainer key._1   -- must be expressed in terms of `key`; must be a signatory
```

Rules:

- Keys are **not unique by default** in Canton 3.5 — multiple contracts can share a key
- Key uniqueness is the app's responsibility
- Cannot add/remove a key definition from a template after it has been deployed (SCU constraint)

### Key operations

```daml
-- Inside a choice (Update context):
optCid  <- lookupByKey @Account (bank, "ACC-001")   -- Optional ContractId
(cid, a) <- fetchByKey @Account (bank, "ACC-001")
exerciseByKey @Account (bank, "ACC-001") Deposit with amount = 100.0

-- In Daml Script:
optResult <- queryByKey @Account alice (bank, "ACC-001")
result <- alice `submit` do
  exerciseByKeyCmd @Account (bank, "ACC-001") Deposit with amount = 100.0

-- Multi-contract lookups (import DA.ContractKeys):
results <- lookupNByKey @Account 5 (bank, "ACC-001")
```

## 10. Standard library essentials

### DA.List

```daml
import DA.List
head xs         -- first element (partial — fails on [])
null []         -- True
length xs       -- Int
zip xs ys       -- [(a,b)]
sortOn f xs     -- sort by field: sortOn (.amount) items
groupOn f xs    -- group by field
dedup xs        -- remove duplicates (requires Eq)
find pred xs    -- Optional a
any pred xs     -- Bool
all pred xs     -- Bool
```

### DA.Optional

```daml
import DA.Optional
fromOptional def None    -- def
fromOptional def (Some x) -- x
fromSome (Some x)        -- x (partial)
isSome / isNone          -- Bool
optional def f optVal    -- apply f if Some, else def
mapOptional f xs         -- [b], applying f :: a -> Optional b, filtering Nones
```

### DA.TextMap

```daml
import DA.TextMap qualified as TextMap
TextMap.empty                           -- TextMap a
TextMap.fromList [("k", v)]             -- TextMap a
TextMap.toList m                        -- [(Text, a)]
TextMap.lookup "k" m                    -- Optional a
TextMap.insert "k" v m                  -- TextMap a
TextMap.fromListWithR (+) pairs         -- merge with combinator
TextMap.merge leftFn rightFn bothFn l r -- full outer merge
```

### DA.Map (keyed by any Ord type)

```daml
import DA.Map qualified as Map
Map.fromList [(party, legs)]
Map.toList m
Map.fromListWithR (++) pairs   -- merge lists
```

### DA.Time / DA.Date

```daml
import DA.Time; import DA.Date
date 2026 Jan 15
time (date 2026 Jan 15) 9 0 0
days 7; hours 2; minutes 30
addRelTime t (hours 1)
```

## 11. TextMap patterns for financial math

The `TextMap Text Decimal` (instrument → amount) pattern is used throughout Token Standard V2. These helpers come up constantly:

```daml
type Funding = TextMap.TextMap Decimal

-- Union two funding maps, combining shared keys:
textMapUnionWith : (a -> a -> a) -> TextMap.TextMap a -> TextMap.TextMap a -> TextMap.TextMap a
textMapUnionWith f = TextMap.merge
  (\_ v1 -> Some v1)
  (\_ v2 -> Some v2)
  (\_ v1 v2 -> Some (f v1 v2))

-- Sum all sender outflows from a list of transfer legs:
sumSides : Side -> [TransferLegSide] -> TextMap.TextMap Decimal
sumSides side allSides = TextMap.fromListWithR (+)
  [ (s.instrumentId, s.amount) | s <- allSides, s.side == side ]

-- Remove negative and zero entries; return None if any instrument overspent:
normalizeFunding : Funding -> Optional Funding
normalizeFunding funding
  | any (\(_, amt) -> amt < 0.0) entries = None
  | otherwise = Some $ TextMap.fromList
      [(iid, amt) | (iid, amt) <- entries, amt > 0.0]
  where entries = TextMap.toList funding

-- Safe default lookup:
lookup0 : TextMap.TextMap Decimal -> Text -> Decimal
lookup0 m iid = fromOptional 0.0 (TextMap.lookup iid m)
```

## 12. Advanced idioms from production code

### Optional constraint with `forA_`

```daml
-- Cleaner than case ... of None -> pure (); Some x -> assertMsg ...
forA_ supplyCap $ \cap ->
  assertMsg ("exceeds cap " <> show cap) (next <= cap)
```

### Conditional signatory list

```daml
-- Mint/burn accounts have no owner; normal accounts have one:
signatory admin :: optional [] (\o -> [o]) spec.authorizer.owner
```

### `coerceContractId` for interface cross-referencing

```daml
-- Store a typed cid as a correlation key across an interface boundary:
create EvidenceReceipt with
  originalRequestCid = coerceContractId self   -- ContractId I -> ContractId T
```

### Asserting positional results from batch settle

```daml
-- SettleBatch returns results in allocation submission order.
-- Guard against future reordering by asserting the count:
let nextIterCids = nextIterationAllocationCids settleResult
assertMsg "unexpected next-iteration count"
  (length nextIterCids == 2 + length draw.fullyConsumed + boundaryCount)
let inputNextCid = case nextIterCids of
      (_ :: Some inNext :: _) -> inNext
      _ -> error "input slice must produce a next-iteration allocation"
```

### Pure constructor modules

Keep complex record construction in separate pure top-level functions outside templates. Choice bodies call these functions rather than inlining record construction:

```daml
-- In WorkflowConstructors.daml (no `create` or `exercise` anywhere):
mkPrefundedAllocationFactoryAllocate
  : Party -> Account -> SettlementInfo -> Optional Time -> Funding
  -> Time -> [ContractId Holding] -> [Party] -> ExtraArgs
  -> AllocationFactory_Allocate
mkPrefundedAllocationFactoryAllocate admin authorizer settlement deadline funding ...=
  AllocationFactory_Allocate with
    settlement
    allocation = mkPrefundedAllocationSpecification admin authorizer deadline funding
    ...

-- In the choice body — clean, readable:
allocResult <- exercise factoryCid
  (WC.mkPrefundedAllocationFactoryAllocate admin account settlement expiry funding now [] [operator] extraArgs)
```

### Floor/truncation for pool payouts

Always floor pool payouts (output amounts) so the pool never pays out more than the exact constant-product result:

```daml
floorDecimal10 : Decimal -> Decimal
floorDecimal10 x = intToDecimal (floor (x * 1e10)) / 1e10
```

Apply `floorDecimal10` to every amount that sources from the pool (swap output, pro-rata redemption). Apply `min` for LP token quantities to prevent over-minting.