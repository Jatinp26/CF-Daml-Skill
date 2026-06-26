# Daml Design Patterns

Canonical patterns for multi-party workflows and financial settlement on Canton Network. Each entry covers when to use it, the key structural insight, and a minimal code skeleton.

## 1. Propose-Accept

**When to use:** Two parties need to agree on a shared contract. One proposes; the other accepts, rejects, or lets it expire. The most fundamental Canton pattern: use it whenever you can't unilaterally impose a contract on another party.

**Key insight:** The proposal contract carries the proposer's authority as signatory. When the recipient exercises `Accept`, their authority is added and the resulting contract can have both as signatories — mutual consent without any off-chain coordination.

```daml
template IouProposal
  with
    issuer    : Party
    recipient : Party
    cash      : Cash
  where
    signatory issuer
    observer  recipient

    choice IouProposal_Accept : ContractId MutualIou
      controller recipient
      do create MutualIou with issuer; owner = recipient; cash

    choice IouProposal_Reject : ()
      controller recipient
      do return ()

    choice IouProposal_Withdraw : ()   -- issuer cancels before recipient acts
      controller issuer
      do return ()

template MutualIou
  with
    issuer : Party
    owner  : Party
    cash   : Cash
  where
    signatory issuer, owner

    choice MutualIou_Settle : ()
      controller owner
      do return ()
```

**Always include Accept + Reject + Withdraw.** Without Withdraw, the proposer can never escape an ignored proposal. Qualify all choice names to avoid record-type collisions.

## 2. Delegation (Role contracts)

**When to use:** One party grants another ongoing permission to act on their behalf without requiring fresh consent for each action. Models custodian relationships, power-of-attorney, pre-authorized agents.

**Key insight:** The principal creates the role contract (they sign). The agent observes it. The agent's choices on the role contract can then exercise choices on the principal's assets because the role contract's signatories are in scope as authorizers for its consequences.

```daml
template IouSender
  with
    sender   : Party    -- the agent
    receiver : Party    -- the principal granting permission
  where
    signatory receiver   -- RECEIVER signs — they are granting permission
    observer  sender

    nonconsuming choice IouSender_Send : ContractId Iou
      with iouCid : ContractId Iou
      controller sender
      do
        iou <- fetch iouCid
        assertMsg "sender must be the iou owner" (sender == iou.owner)
        exercise iouCid Iou_MutualTransfer with newOwner = receiver
```

Always `nonconsuming` — role contracts are meant to be used repeatedly.

## 3. Authorization token (on-ledger whitelist)

**When to use:** A controller needs to verify a party has been approved before they can take an action. Common for KYC/AML, accredited-investor checks, approved-counterparty lists.

**Key insight:** The token is signed by the authority. Requiring it as a choice argument makes the ledger verify it exists and is authentic. No off-chain database or API call needed.

```daml
template KycApproval
  with
    issuer    : Party
    recipient : Party
  where
    signatory issuer
    observer  recipient

    choice KycApproval_Revoke : ()
      controller issuer
      do return ()

-- In any template that needs KYC before transfer:
    choice Asset_Transfer : ContractId Asset
      with
        newOwner    : Party
        kycApproval : ContractId KycApproval
      controller owner
      do
        approval <- fetch kycApproval
        assertMsg "KYC issuer mismatch"     (approval.issuer    == trustedKycIssuer)
        assertMsg "KYC recipient mismatch"  (approval.recipient == newOwner)
        create this with owner = newOwner
```

## 4. Locking (state-change variant)

**When to use:** An asset must be frozen during settlement or clearing. A third party needs to prevent transfer until they explicitly release it.

**Key insight:** Add a `locker : Party` field. When `locker == owner`, the asset is unlocked. When they differ, the locker controls unlock. Enforced in the `Transfer` choice.

```daml
template LockableAsset
  with
    owner  : Party
    issuer : Party
    locker : Party    -- equals owner when unlocked
    amount : Decimal
  where
    signatory issuer, owner
    observer  locker
    ensure amount > 0.0

    choice LockableAsset_Transfer : ContractId TransferProposal
      with newOwner : Party
      controller owner
      do
        assertMsg "Cannot transfer a locked asset" (locker == owner)
        create TransferProposal with asset = this; newOwner

    choice LockableAsset_Lock : ContractId LockableAsset
      with newLocker : Party
      controller owner
      do
        assertMsg "Cannot lock to self" (newLocker /= owner)
        create this with locker = newLocker

    choice LockableAsset_Unlock : ContractId LockableAsset
      controller locker
      do
        assertMsg "Already unlocked" (locker /= owner)
        create this with locker = owner
```

## 5. Multiple party agreement

**When to use:** Three or more parties must all sign a contract before it becomes binding. Each signs separately and asynchronously, in any order.

**Key insight:** A `Pending` wrapper tracks who has signed so far. It is observable by all required signatories. Each `Sign` exercise adds a party. When complete, any signatory calls `Finalize` to create the final contract.

```daml
template Agreement
  with
    signatories : [Party]
    content     : Text
  where
    signatory signatories
    ensure unique signatories

template Pending
  with
    finalContract : Agreement
    alreadySigned : [Party]
  where
    signatory alreadySigned
    observer  finalContract.signatories
    ensure unique alreadySigned

    choice Pending_Sign : ContractId Pending
      with signer : Party
      controller signer
      do
        let remaining = filter (`notElem` alreadySigned) finalContract.signatories
        assertMsg "Not a required signatory" (signer `elem` remaining)
        create this with alreadySigned = signer :: alreadySigned

    choice Pending_Finalize : ContractId Agreement
      with signer : Party
      controller signer
      do
        assertMsg "Not all parties have signed"
          (sort alreadySigned == sort finalContract.signatories)
        create finalContract
```

## 6. Sharded hot-singleton (config / state / slices)

**When to use:** A shared resource (pool, registry, order book) has aggregate state that every operation must update, but the resource has many individual records that operations touch independently. A naive single contract becomes a contention bottleneck.

**Key insight:** Split into three layers:
- **Config** — immutable after creation, never conflicts
- **State** — the unavoidable single serialization point (totals only)
- **Slices/records** — each operation creates/touches only the slices it needs

Add/remove creates new slices (conflicts with nothing). Operations only touch the slices they source. The state tracks aggregate totals purely for pricing; slices are the source of truth for the underlying funds.

```daml
-- Immutable config — stable identifier, never conflicts
template PoolConfig
  with
    poolId     : Text
    operator   : Party
    feeBps     : Int
    baseToken  : Text
    quoteToken : Text
  where
    signatory operator

-- Hot state — one per pool, every swap rewrites it once
template PoolState
  with
    poolId       : Text
    operator     : Party
    baseReserve  : Decimal   -- sum of all base slice amounts
    quoteReserve : Decimal   -- sum of all quote slice amounts
    status       : PoolStatus
  where
    signatory operator

-- Sharded record — one per LP deposit; swaps touch only affected slices
template PoolSlice
  with
    poolId        : Text
    operator      : Party
    side          : Side
    allocationCid : ContractId SomeAllocation
    amount        : Decimal   -- cached; always reconciled against allocation
  where
    signatory operator
```

**Delta conservation rule:** Every choice that rewrites `PoolState.baseReserve` (or `quoteReserve`) must assert that the reserve delta equals the net slice-amount change performed in the same choice. This turns silent drift into an immediate ledger rejection.

```daml
-- Inside a swap choice, after computing newBaseReserve and newQuoteReserve:
assertMsg "swap: base reserve delta must equal net base slice delta"
  (newBaseReserve - state.baseReserve == inputSliceDelta + outputSliceDelta)
```

## 7. Acceptance evidence receipt

**When to use:** A request contract is consumed by a third-party action (e.g. a wallet calling `AllocationRequest_Accept`) before your settlement choice runs. Your settle choice needs the original request terms for validation, but the contract is gone.

**Key insight:** The `Accept` implementation creates an evidence receipt carrying the same validation terms immediately before consuming the request. The settle binds to whichever is available: the live request OR the receipt.

```daml
-- The accept implementation (on LiquidityAllocationRequest) creates this
-- before archiving itself:
template LiquidityAllocationAcceptance
  with
    operator             : Party
    lp                   : Party
    settlement           : SettlementInfo
    allocations          : [AllocationSpecification]
    settleAt             : Optional Time
    acceptedAt           : Time
    originalRequestCid   : ContractId LiquidityAllocationRequest
      -- ^ Correlation key: discover this by the consumed request's cid
  where
    signatory operator
    observer  lp

-- Settlement choice resolves whichever source is present:
resolveBinding
  : Optional (ContractId LiquidityAllocationRequest)
  -> Optional (ContractId LiquidityAllocationAcceptance)
  -> Update LiquidityBinding
resolveBinding mReq mAcc = case (mReq, mAcc) of
  (Some r, _) -> fetchLiveRequest r
  (None, Some a) -> fetchEvidence a
  (None, None) -> abort "settle requires a live request or acceptance evidence"
```

## 8. Prefunded / iterated allocation

**When to use:** An agent (e.g. an order) needs to lock funds at authorization time, but the exact transfer legs are not known until a later event (e.g. a match at an unknown price).

**Key insight:** The allocation is created with `nextIterationFunding` set (the locked budget) and no concrete transfer legs. At settlement time, the executor supplies the actual legs via `extraTransferLegSides` on `FinalizedAllocation`. If only partially consumed, the remainder rolls forward into a new allocation.

```daml
-- At order placement: lock budget, no legs yet
let prefundedSpec = AllocationSpecification with
      admin
      authorizer = traderAccount
      transferLegSides = []      -- no legs at placement time
      nextIterationFunding = Some (TextMap.fromList [(lockInstrument, lockAmount)])
      committed = True
      settlementDeadline = expiry
      meta = emptyMetadata

-- At match time: supply actual legs on the FinalizedAllocation
let finalized = FinalizedAllocation with
      allocationCid = existingAllocationCid
      extraTransferLegSides = legsToSides traderAccount matchLegs
      nextIterationFunding = remainderFunding   -- None = fully settled
```

The remainder calculation: `remainingBudget = lockedBudget - amountSpent`. Signal `None` (no next iteration) when the budget is fully consumed.

## 9. Capability contract (pure nonconsuming rules)

**When to use:** A set of operations share authorization (one operator deploys them once) but none of them modify the rules contract itself. The contract is a pure permission grant.

**Key insight:** Make every choice `nonconsuming`. The contract never moves — it is the stable reference point for all operations. All state lives elsewhere (on Config, State, and Slice contracts). One rules contract serves the entire lifetime of the system.

```daml
template PoolRules
  with
    operator : Party
  where
    signatory operator
    -- No mutable state here at all

    nonconsuming choice PoolRules_Swap : SwapResult
      with ...
      controller operator
      do
        -- fetch pool config + state by cid
        -- assert pre-conditions
        -- archive + recreate state
        -- archive + create slices
        -- return result

    nonconsuming choice PoolRules_Pause : ContractId PoolState
      with ...
      controller operator
      do ...

    nonconsuming choice PoolRules_ReconcileState : ReconcileResult
      with sliceCids : [ContractId PoolSlice]
      controller operator
      do ...  -- pure audit; touches no state
```

## 10. Multiple interfaces on one template

**When to use:** A single registry or factory contract needs to participate in several interface-defined protocols (e.g. Token Standard V2's AllocationFactory + SettlementFactory + TransferFactory + EventLog).

**Key insight:** A single template can implement multiple interface instances. Each `toInterfaceContractId` call is a safe upcast of the same underlying ContractId.

```daml
template Registry
  with
    admin : Party
    users : [Party]
  where
    signatory admin
    observer  users

    interface instance TransferFactory   for Registry where ...
    interface instance AllocationFactory for Registry where ...
    interface instance SettlementFactory for Registry where ...
    interface instance EventLog          for Registry where ...

-- Usage: one create, four typed references
createRegistry admin users = do
  cid <- create Registry with admin; users
  pure
    ( toInterfaceContractId cid   -- ContractId TransferFactory
    , toInterfaceContractId cid   -- ContractId AllocationFactory
    , toInterfaceContractId cid   -- ContractId SettlementFactory
    )
```

## Pattern selection guide

| Situation | Pattern |
|---|---|
| Two parties need to agree on a new contract | Propose-Accept (#1) |
| Party grants another ongoing permission | Delegation / Role contract (#2) |
| Verify a party is whitelisted before action | Authorization token (#3) |
| Asset must be frozen during settlement | Locking (#4) |
| 3+ parties must all sign before contract lives | Multiple party agreement (#5) |
| Shared resource suffers from contention hot-spot | Sharded hot-singleton (#6) |
| Request consumed by wallet before settle runs | Acceptance evidence receipt (#7) |
| Lock funds at authorization, reveal legs later | Prefunded / iterated allocation (#8) |
| Shared permissions, no state on the rules contract | Capability contract (#9) |
| One contract must satisfy multiple protocol interfaces | Multiple interface instances (#10) |
| Issuer needs ongoing control over who holds assets | Transfer proposal (see Propose-Accept variant) |
| Need an immutable audit record visible to regulator | Observable event (no-choice template, observer = regulator) |
| Party needs to counter-propose different terms | Propose-Accept with `Counter` choice |

## Composing patterns

Patterns compose naturally. A real DEX workflow combines:

1. **Propose-Accept** — trader proposes order funding, operator binds it
2. **Prefunded allocation** — trader locks budget at placement, legs revealed at match
3. **Sharded hot-singleton** — pool config / state / slices avoid AMM contention
4. **Capability contract** — `PoolRules` is the stable permission reference
5. **Acceptance evidence receipt** — wallet accepts liquidity request before settle runs
6. **Multiple interface instances** — one Registry serves AllocationFactory + SettlementFactory
7. **Observable event** — `PolicyReceipt` on `MatchedTrade` provides on-ledger RFQ audit trail

Each contract in the chain carries the minimum authority needed for its role.