# From Cardano to Midnight — Developer Field Guide

*The eUTXO model you know is the best possible foundation. Here's what changes — and what to watch out for.*

**Midnight Network Foundation | June 2026**

> **Version note:** Midnight mainnet launched March 31, 2026 in a federated phase — a curated set of trusted node operators (IOG, Google Cloud, Vodafone) runs the network while decentralization via Cardano SPO onboarding is completed in the Mōhalu phase (Q2–Q3 2026). Compact targets `language_version >= 0.23`. Cross-contract calls are reserved in the language spec but not yet implemented in Compact 1.0. Always verify against [docs.midnight.network](https://docs.midnight.network) before shipping.

---

## 1. Why Midnight? The Pitch for Cardano Developers

Of all the blockchain developer communities, Cardano developers are the most naturally prepared for Midnight. You already think in UTXOs. You already reason about determinism, off-chain transaction building, and on-chain validation as a pure function. You've already internalized the insight that smart contracts should *approve or deny*, not execute imperatively. These instincts are directly transferable.

What Cardano doesn't have — and what Midnight is built around — is privacy. Every Cardano UTXO is public: value, datum, address, minting policy, all of it. That's fine for many applications. It's a blocker for regulated financial instruments, enterprise data sharing, healthcare applications, and anything where "everyone can read your ledger" is a compliance violation rather than a feature.

**One-line pitch:** Midnight is what Cardano's eUTXO model would look like if it had been extended with zero-knowledge proofs from the start. The UTXO lineage is shared; the privacy layer is what's new.

### What Carries Over Directly

- **UTXO model:** consume inputs, create outputs, all-or-nothing atomicity
- **Deterministic transaction validation:** outcomes are knowable off-chain before submission
- **Separation of on-chain logic from off-chain transaction building**
- **Native multi-asset support:** tokens live in UTXOs, not in contract storage
- **No global mutable state:** contracts validate, they don't imperatively execute
- **Substrate under the hood:** Midnight is built on the Polkadot SDK using AURA block production and GRANDPA finality extended with Cardano partner chain components; node infrastructure concepts are recognizable
- **TypeScript off-chain tooling:** Midnight.js SDK is TypeScript, familiar to Lucid/Mesh/Blaze users
- **You already hold NIGHT:** if you're an ADA holder, NIGHT was airdropped to Cardano wallets in the Glacier Drop. You may have more skin in this game than you think.

### What Needs Rewiring

- **Aiken/Plutus → Compact:** different language, same functional intuitions, new compiler target (ZK circuits not UPLC/Wasm)
- **Datum/redeemer pattern → circuit/witness split:** state and access patterns work differently
- **Script addresses → contract addresses:** scripts don't live at UTXO addresses in Midnight
- **Collateral → no collateral:** Midnight has no Phase 2 validation failure risk
- **minUTXO requirement → no equivalent:** Midnight UTXOs don't require a minimum token deposit
- **On-chain validators see full transaction context → Midnight circuits see only their parameters, witnesses, and ledger state**
- **Everything public by default → everything private by default**
- **cNIGHT on Cardano ≠ immediate DUST on Midnight:** there is a bridge and a registration step (see §6)

---

## 2. eUTXO vs. Midnight UTXO: Closer Than You Think, Different Where It Counts

Most Cardano developers expect this section to be simple. It's not, because the extensions Cardano and Midnight each added to base UTXO go in fundamentally different directions. Cardano extended UTXO with datums, redeemers, and on-chain script validation. Midnight extended UTXO with zero-knowledge proofs, private witnesses, and shielded commitments.

### The Cardano eUTXO Model: A Refresher

Each UTXO carries: a value (ADA + native tokens), an address (public key or script hash), and an optional datum. To spend a UTXO locked at a script address, you provide a redeemer and the validator runs on-chain, receiving the datum, redeemer, and full transaction context (`ctx: Transaction`). If it returns `True`, the spend is valid.

```aiken
validator my_contract {
  spend(
    datum:    Option<MyDatum>,
    redeemer: MyRedeemer,
    ctx:      Transaction,   // full tx context: inputs, outputs, signatories, etc.
  ) {
    // Validator sees ALL transaction inputs and outputs
    // Returns True (valid) or False (invalid) — pure function
    expect Some(d) = datum
    list.has(ctx.transaction.extra_signatories, d.owner) &&
    verify_output_datum(ctx.transaction.outputs, redeemer.new_value)
  }
}
```

### The Midnight UTXO Model: Key Structural Differences

Midnight UTXOs carry value and a cryptographic commitment to ownership — but no datum in the Cardano sense. Contract state lives in `export ledger` fields on the public ledger, not attached to individual UTXOs. Spending a UTXO doesn't invoke an on-chain validator that sees the full transaction context; the user generates a ZK proof off-chain that the spend is valid, and the chain verifies the proof.

| Aspect | Cardano eUTXO | Midnight UTXO |
|---|---|---|
| UTXO carries | Value + address + datum | Value + cryptographic commitment |
| Contract state | Datum attached to UTXO at script address | `export ledger` fields on public ledger |
| Spending a UTXO | Provide redeemer; validator runs on-chain | Generate ZK proof; verifier checks proof |
| Validator sees | Full tx context: all inputs, outputs, sigs | Only circuit parameters + ledger state + witness outputs |
| Privacy default | Everything public | Everything private unless disclosed |
| On-chain execution | Script runs on every validating node | Proof verified; no re-execution |
| Script address | Derived from script hash | Contract address at deployment |
| Collateral | Required for Phase 2 script execution | No collateral mechanism |
| minADA requirement | Every output must carry minimum ADA | No minimum token deposit on UTXOs |
| Cross-contract calls | Reference inputs + validator composition | **Not implemented in Compact 1.0** |

> **The biggest trap for Cardano developers:** you cannot read other UTXOs in a Midnight circuit the way you can read transaction inputs in an Aiken validator. A Cardano validator receives `ctx.transaction` with all inputs and outputs. A Midnight circuit sees only its own parameters, its witness function outputs, and the ledger state. If your Cardano contract logic depends on reading reference inputs or inspecting all transaction outputs, you need to rethink the architecture.

### The Datum/Redeemer Pattern vs. the Circuit/Witness Split

| Cardano concept | Midnight equivalent | Notes |
|---|---|---|
| Datum (state on UTXO) | `export ledger` fields (state on public ledger) | State lives globally, not per-UTXO |
| Redeemer (spend argument) | Circuit parameters | Arguments passed to an exported circuit |
| Validator script | Compact circuit | Compiled to ZK circuit, runs off-chain |
| Script context (tx view) | No equivalent | Circuits don't see the full transaction |
| Datum witness (inline) | `witness` function | Private data supplied off-chain by the user |
| Minting policy | Token type derivation from contract | Token identity cryptographically bound to contract |
| Reference inputs | No equivalent | Circuits cannot read external UTXOs |

---

## 3. Compact vs. Aiken: The Language Comparison

Aiken gave Cardano developers a clean Rust-like syntax targeting Untyped Plutus Core. Compact gives Midnight developers a TypeScript-like syntax targeting ZK arithmetic circuits. Both are functional in spirit — pure functions, no side effects, immutable values — but the runtime targets are completely different and the constraints differ in important ways.

### What's Shared Philosophically

- Pure functions: given the same inputs, always produce the same result
- No implicit global state: all inputs to logic must be explicit
- Fail loudly: validation failure aborts everything (`assert` vs `expect`)
- Compile-time correctness: both compilers catch errors before deployment
- Off-chain transaction building: both require TypeScript off-chain tooling

### Anatomy of a Compact Contract

A Compact contract has three execution contexts. Aiken developers will recognize the on-chain/off-chain split but witnesses have no Aiken equivalent:

- **`export ledger` fields:** On-chain state. Public, globally visible. Equivalent to the state you'd encode in datums across a series of Cardano UTXOs — but stored directly on the ledger rather than attached to individual UTXOs.
- **Circuits (`export circuit`):** The entry points — equivalent to Aiken's `spend`/`mint`/`withdraw` handlers. Compiled to ZK circuits and run off-chain on the user's machine. The proof is what hits the chain.
- **Witnesses (`witness`):** Private functions that run locally with access to data the user doesn't want on-chain. Spending keys, off-chain database queries, anything sensitive. No Cardano equivalent.

### Side-by-Side: Aiken Validator vs. Compact Circuit

```aiken
// ── AIKEN (Cardano) ───────────────────────────────────────────────
pub type Datum   { owner: ByteArray }
pub type Redeemer { new_value: Int }

validator my_contract {
  spend(datum: Option<Datum>, redeemer: Redeemer, ctx: Transaction) {
    expect Some(d) = datum
    // Access control via signatories in tx context
    list.has(ctx.transaction.extra_signatories, d.owner) &&
    // Inspect outputs to enforce state transition
    find_output_with_value(ctx.transaction.outputs, redeemer.new_value)
  }
}
```

```compact
// ── COMPACT (Midnight) ────────────────────────────────────────────
pragma language_version >= 0.23;
import CompactStandardLibrary;

// On-chain state — each field declared individually (NOT a ledger { } block)
export ledger authority: Bytes<32>;   // stored identity commitment
export ledger value: Uint<64>;
export ledger round: Counter;          // anonymity counter — always include

// Private input: spending key from user's wallet, never on-chain
witness secretKey(): Bytes<32>;

// Helper circuit: derives a tamper-proof identity commitment
// Uses domain separation + the round counter for unlinkability between sessions
// NEVER use ownPublicKey() or a bare publicKey() stdlib call here — see §5
circuit ownerKey(sk: Bytes<32>): Bytes<32> {
    return persistentHash<Vector<3, Bytes<32>>>(
        [pad(32, "myapp:owner:pk:"), round as Bytes<32>, sk]
    );
}

constructor(deployerSk: Bytes<32>) {
    authority = disclose(ownerKey(deployerSk));
    round.increment(1);
}

// Entry point: equivalent to Aiken's spend handler
export circuit setValue(newValue: Uint<64>): [] {
    // Access control: prove knowledge of private key matching authority
    // No ctx.extra_signatories — ZK proof replaces the signatories list
    assert(ownerKey(secretKey()) == authority, "Not authorized");

    // State update: direct ledger mutation (no continuing output required)
    value = disclose(newValue);
    round.increment(1);
}
```

**Critical syntax notes:**
- Each ledger field is declared with its own `export ledger fieldName: Type;` statement. The `ledger { }` block syntax from older pre-mainnet docs will fail to compile.
- `round: Counter` uses `round.increment(1)`, not `round = round + 1` — the `Counter` type exposes an increment method; arithmetic assignment is a type error.
- The `ownerKey` helper is a **circuit**, not a stdlib function call. This matters for security — see §5.

### Key Syntax and Concept Mapping

| Aiken / Plutus | Compact | Notes |
|---|---|---|
| `validator { spend(...) }` | `export circuit myCircuit(...)` | `export` = callable in a transaction |
| `validator { mint(...) }` | `export circuit mint(...)` | Minting logic is a circuit |
| `datum: Option<MyDatum>` | `export ledger` fields | State in ledger, not per-UTXO datum |
| `redeemer: MyRedeemer` | Circuit parameters | Arguments passed directly to the circuit |
| `ctx: Transaction` | No equivalent | Circuits don't see full tx context |
| `ctx.extra_signatories` | Custom `persistentHash` circuit + witness | See §5 — `ownPublicKey()` is unsafe for this |
| `expect Some(x) = opt` | `assert(condition, "msg")` | Both abort on failure; proof fails in Compact |
| `trace("msg", False)` | `assert(false, "msg")` | Explicit failure with message |
| Continuing output pattern | `ledger.field = newValue` | Direct ledger mutation; no output to construct |
| Reference inputs | No equivalent | Cannot read external UTXOs in a circuit |
| `validity_range` / `POSIXTime` | `round: Counter` (ledger field) | No block timestamp — use round counter |
| Script hash | Contract address | Different derivation model |
| `aiken build` | `compactc` | The Compact compiler |
| `ByteArray` | `Bytes<N>` (fixed size) | Compile-time-known sizes |
| `Int` | `Uint<64>` / `Uint<128>` | Explicit fixed-width integers |
| `Bool` | `Boolean` | Same concept |
| `pub type MyType { field: Int }` | `struct MyType { field: Uint<64> }` | Custom types supported |
| `ExUnits` (execution budget) | Circuit constraint count | Budget is compile-time, not runtime-metered |
| Lucid / Mesh / Blaze | Midnight.js SDK | TypeScript off-chain SDK |

---

## 4. The Privacy Model: Private by Default

Cardano is entirely transparent. Every UTXO — its value, datum, and address — is readable by anyone. Privacy on Cardano requires off-chain coordination (keeping datum contents secret) or off-chain systems, and even then, token amounts and addresses are visible on-chain. Midnight inverts this at the protocol level.

### How the Compiler Enforces Privacy

The Compact compiler tracks where data originates. Any value produced by or derived from a witness function is classified as private. If that private-origin data attempts to reach the public ledger without explicit permission, the compiler rejects it — at compile time, not runtime.

```compact
witness _userBalance(): Uint<64>;   // private: underscore convention

export circuit updateBalance(): [] {
    const bal = _userBalance();   // private origin

    // ❌ COMPILE ERROR: witness-derived value reaching public ledger
    ledger.total = ledger.total + bal;

    // ✅ CORRECT: explicit disclosure annotation
    ledger.total = ledger.total + disclose(bal);

    // ✅ ALSO VALID: prove a property without disclosing the value
    // The balance stays private; only the fact it meets the threshold is public
    assert(bal >= 1000, "Balance below minimum");
    ledger.meetsThreshold = disclose(true);
}
```

`disclose()` is not a cast. It's a compiler annotation saying "I consciously accept that this value is leaving the private domain." The value only becomes public if it's actually stored in ledger state or returned from an exported circuit. Think of it as the mandatory acknowledgment before crossing a privacy boundary.

### The Three Privacy States

| State | Location | Visible to | Cardano equivalent |
|---|---|---|---|
| Witness data | User's local machine only | Only the user | Keeping datum content secret off-chain (fragile, no protocol guarantee) |
| Private circuit execution | In-flight during proof generation | Nobody | No equivalent |
| Disclosed / ledger state | Public ledger | Everyone | Datum at a script address, UTXO value |

### Shielded UTXOs: What Cardano Native Tokens Don't Have

Cardano native tokens are fully public — amount, policy ID, asset name, and both addresses are visible on-chain in plaintext. Midnight extends the UTXO model with shielded UTXOs where value and ownership are hidden behind cryptographic commitments. The network knows a UTXO exists; it cannot determine the amount or owner without the owner's private key.

Shielding is a per-UTXO property, not a network-wide mode. A single transaction can mix shielded and unshielded UTXOs. A regulated stablecoin might be shielded for user-to-user transfers but disclosed to a compliance auditor via a viewing key — both in the same transaction.

> **Wallet architecture changes significantly.** A Cardano wallet can reconstruct your balance by querying the chain for UTXOs at your addresses. A Midnight wallet must maintain a local database of the owner's shielded UTXOs, decrypt incoming commitments using the spending key, and track which have been nullified. There is no chain-side balance query for shielded state. Plan for this in your dApp's wallet integration.

### `sealed` Ledger Fields: Midnight's Minting Policy Analogy

Cardano developers familiar with one-shot minting policies — where a policy is valid only for a specific transaction and can never be used again — will appreciate `sealed` ledger fields:

```compact
export ledger sealed totalSupplyCap: Uint<64>;  // immutable after deployment
export ledger minted: Uint<64>;

constructor(cap: Uint<64>) {
    totalSupplyCap = disclose(cap);  // set once; immutable forever after
}

export circuit mint(amount: Uint<64>): [] {
    assert(ownerKey(secretKey()) == authority, "Not authorized");
    assert(minted + disclose(amount) <= totalSupplyCap, "Exceeds cap");
    minted = disclose(minted + disclose(amount));
    round.increment(1);
    // totalSupplyCap cannot be modified here — compiler enforces this
}
```

A `sealed` field can only be set by the constructor (or circuits called by the constructor). After initialization, no exported circuit can modify it, and the compiler enforces this at compile time. This is Midnight's equivalent of Cardano's parameterized scripts where certain parameters are locked at script creation.

---

## 5. Access Control: From `ctx.extra_signatories` to ZK-Proved Keys

In Aiken, access control flows through `ctx.transaction.extra_signatories` — a list of public keys that have signed the transaction. The validator checks membership. It's reliable, intuitive, and directly enforced by the Cardano ledger rules.

Midnight has no transaction signatories list visible to circuits. It doesn't exist because exposing it would link identities to contract interactions, undermining the privacy model. Instead, access control is proved via knowledge of a private key.

### A Critical Security Warning Before Any Code

**Never use `ownPublicKey()` for access control in Compact circuits.** This is officially documented in the Midnight security guide and confirmed in production audits.

The reason: `ownPublicKey()` is technically a witness function. In a ZK circuit, it compiles to an **unconstrained private input** — a value the prover sets freely. There is no cryptographic obligation for the prover to provide their actual public key. An attacker supplies the stored `authority` value directly as their "public key," the comparison passes, and access is granted with no knowledge of any secret.

The fix is a commitment scheme: the contract stores a cryptographic hash of the owner's secret key (the commitment). To prove ownership, the caller re-derives that same hash using their witness-provided secret. This is only possible if they know the original secret — which is exactly what the ZK proof then certifies.

The official `Writing a contract` tutorial and the bulletin board example both use this pattern correctly.

### The Correct Access Control Pattern

```aiken
// Cardano / Aiken: signatories-based access control
validator {
  spend(datum: Option<Datum>, redeemer: Redeemer, ctx: Transaction) {
    expect Some(d) = datum
    list.has(ctx.transaction.extra_signatories, d.owner)
  }
}
```

```compact
// Compact: ZK-proved key ownership (correct pattern from official docs)
export ledger authority: Bytes<32>;   // commitment stored at deployment
export ledger round: Counter;

witness secretKey(): Bytes<32>;

// Helper circuit: builds a cryptographically sound identity commitment
// Domain tag: prevents the same secret from being valid across different contracts
// Round counter: ensures each commitment is unique to this contract session,
//                preventing replay attacks across rounds
circuit ownerKey(sk: Bytes<32>): Bytes<32> {
    return persistentHash<Vector<3, Bytes<32>>>(
        [pad(32, "myapp:owner:pk:"), round as Bytes<32>, sk]
    );
}

// Registering ownership at deployment:
constructor(sk: Bytes<32>) {
    authority = disclose(ownerKey(sk));
    round.increment(1);
}

// Verifying ownership in a restricted circuit:
export circuit restrictedAction(param: Uint<64>): [] {
    // Prove: "I know the sk whose hash matches authority, for this round"
    // ZK proof certifies this without the chain ever seeing sk
    assert(ownerKey(secretKey()) == authority, "Not authorized");
    value = disclose(param);
    round.increment(1);
}
```

> **Every restricted circuit needs its own explicit `assert`.** There is no inherited validator context in Compact — no equivalent of a single Aiken validator handler covering all spend paths. If you add a new exported circuit and omit the `assert`, it's callable by anyone. Make access control the first line of every restricted circuit.

### Why the Round Counter in the Commitment

Cardano validators can detect which specific UTXO is being spent, giving natural replay protection. Midnight circuits don't see UTXOs in the same way. The round counter in the commitment ensures that even if an attacker observes a valid commitment in one round, it won't be valid in the next — because the commitment changes when the round increments. This provides transaction unlinkability: even the same owner appears to use a different identity commitment each round.

---

## 6. Tokens and the cNIGHT / NIGHT / DUST Model

This section is uniquely important for Cardano developers because you're not starting from zero — NIGHT already exists on Cardano, and the relationship between Cardano-native NIGHT and Midnight-native NIGHT is a live operational detail, not an abstraction.

### The Token Relationship: cNIGHT, mNIGHT, and DUST

NIGHT was launched in December 2025 as a **Cardano Native Asset** (`cNIGHT`, policy ID: `0691b2fecca1ac4f53cb6dfb00b7...4e49474854`). At Midnight mainnet launch, it became simultaneously a Midnight-native asset (`mNIGHT`). These are two representations of the same token, bridged across chains via a protocol-level mechanism.

**NIGHT** is the transferable utility token — publicly visible on both chains. It's used for staking (network security), governance, and generating DUST. NIGHT is not spent directly as a fee medium.

**DUST** is a non-transferable resource generated by holding NIGHT on Cardano. It's used exclusively to pay Midnight transaction fees. DUST expires if unused, cannot be transferred, and is not traded on markets — fee costs are therefore stable regardless of NIGHT's market price.

### How DUST Generation Actually Works (Cardano Developers: Read This)

This cross-chain mechanism is something you won't find in any of the other migration guides and it directly affects your infrastructure:

1. You register your **Cardano reward address + a DUST public key** on Cardano.
2. The `Native Token Observation Pallet` (`pallet_cnight_observation`) on Midnight monitors your cNIGHT UTXO activity on Cardano.
3. When cNIGHT is received (created) at your registered address, a corresponding DUST creation event fires on Midnight. When cNIGHT is sent (spent), a DUST destruction event fires.
4. Events are batched per block and executed on the Midnight ledger via a system transaction.
5. DUST supply on Midnight updates 1:1 with cNIGHT movements on Cardano.

The protocol-level bridge (MIP-020) for transferring cNIGHT to mNIGHT has approximately 12-hour finality — by design, to ensure Cardano transaction irreversibility before Midnight mints the equivalent. This is a meaningful UX consideration for applications that depend on NIGHT availability on Midnight.

**The cold-start problem.** New wallets have no DUST. Without DUST, they can't pay fees. The solution is **DUST sponsorship**: a dApp holding NIGHT can delegate its DUST to cover new users' transaction fees. The SDK exposes this as a two-phase transaction balancing model — the user covers token flows, the sponsor covers DUST. For consumer-facing apps where users may not yet hold NIGHT, design sponsored transactions from day one.

### DUST vs. ADA: The Fee Model Compared

| Aspect | Cardano | Midnight |
|---|---|---|
| Fee token | ADA (minFee formula) | DUST (generated by holding NIGHT) |
| Fee calculation | `minFee(a,b)` based on tx size | Dynamic based on network usage |
| Fee volatility | Stable in ADA terms; varies in USD | Stable — DUST not market-priced |
| Failed tx cost | Tx fee still paid | Guaranteed-phase DUST forfeited on fallible failure |
| Storage cost | minUTXO deposit per output | No minimum deposit |
| Sponsored fees | No native mechanism | DUST delegation via SDK |
| Account required | No — UTXOs are stateless | No — same UTXO model |

### Custom Tokens: Native Assets vs. Midnight UTXO Tokens

| Property | Cardano Native Token | Midnight Custom Token |
|---|---|---|
| Token identity | Policy ID + Asset Name | 256-bit hash of contract address + domain separator |
| Token storage | Inside UTXOs alongside ADA | UTXOs on the ledger (shielded or unshielded) |
| Minting authorization | Minting policy script validates | Circuit-based minting logic (ZK proof) |
| Transfer mechanism | Consume input UTXO, create output UTXO | Same — consume input, create output |
| Privacy option | None — amounts and addresses always public | Shielded UTXOs hide value and owner |
| minADA requirement | Yes — every output must carry minimum ADA | No — no minimum deposit on UTXOs |
| Atomic multi-asset | Multi-asset UTXO natively | Native via Zswap protocol |
| Approve pattern | Not needed (UTXO spend = authorization) | Not needed (ZK proof owns the spend) |

### Minting Policies vs. Circuit-Based Minting

On Cardano, token minting is controlled by a separate minting policy script. The policy ID is derived from the policy script hash. On Midnight, minting logic lives in a circuit — the token type is derived from the contract's address and a developer-chosen domain separator. No separate policy script; the circuit enforces minting conditions through ZK proof.

```compact
// Midnight: minting as a circuit
// Token type = Hash(contractAddress, "myToken") — no separate policy script
export circuit mintTokens(amount: Uint<64>): [] {
    assert(ownerKey(secretKey()) == minter, "Not authorized to mint");
    ledger.totalSupply = disclose(ledger.totalSupply + disclose(amount));
    round.increment(1);
    // The Zswap balance vector is adjusted by the runtime
    // when this circuit creates new token UTXOs as outputs
}
```

### Zswap: Native Atomic Swaps

Cardano's native multi-asset support means you can include multiple token types in a single UTXO — but swapping requires either careful transaction structure or a DEX batcher. Amounts are fully visible on-chain. Midnight includes Zswap at the protocol level — a ZK-SNARK-based atomic swap mechanism where offers from two parties merge non-interactively and the matcher never learns the trade amounts.

| Aspect | Cardano DEX / Native Multi-asset | Midnight Zswap |
|---|---|---|
| Trade amounts visible | Yes — fully public on-chain | No — shielded in offers |
| Atomic swap mechanism | Transaction structure + smart contract | Protocol-level offer merging |
| Matcher sees amounts | Yes — batcher reads all UTXO values | No — verifies balance vector only |
| Front-running exposure | Yes — pending tx mempool visible | Minimal — offers encrypted pre-broadcast |
| Approve pattern | Not needed (UTXO spend) | Not needed (ZK proof) |

> **Zswap implementation note:** The official docs flag this layer as not yet performance-optimized and subject to further revision. The conceptual model is stable; verify specific API surface before building production integrations.

---

## 7. State Management: From Datum Chains to Ledger State

State management is where experienced Cardano developers have the most accumulated patterns to unlearn — not because those patterns are wrong, but because the Midnight model makes them unnecessary.

### How Cardano Manages Contract State

Cardano contracts carry state in datums attached to UTXOs at script addresses. A state machine works by consuming the UTXO with the current datum and producing a new UTXO at the same script address with the updated datum. The "continuing output" pattern is the backbone of Cardano stateful contracts.

```aiken
validator counter {
  spend(datum: Option<Int>, redeemer: Void, ctx: Transaction) {
    expect Some(count) = datum
    let script_addr = find_script_input_address(ctx.transaction.inputs)
    ctx.transaction.outputs |> list.any(fn(out) {
      out.address == script_addr &&
      out.datum == InlineDatum(count + 1)
    })
  }
}
// Off-chain: build tx consuming old UTXO, producing new UTXO at script addr
// Datum on output = new state. Whole chain of datums = contract history.
```

### How Midnight Manages Contract State

In Midnight, contract state lives directly in `export ledger` fields. There is no continuing output to construct, no script address to target, no datum to serialize and attach to an output. Circuits mutate ledger fields directly and the runtime handles persistence.

```compact
pragma language_version >= 0.23;
import CompactStandardLibrary;

export ledger count: Counter;

export circuit increment(): [] {
    // No continuing output. No datum to construct. No script address targeting.
    count.increment(1);
}

// Off-chain: call increment() circuit, submit proof.
// The chain updates count. Done.
```

> **Relief for Cardano developers:** the continuing output pattern is a workaround for the fact that eUTXO state must live in UTXOs. Midnight's ledger fields are direct on-chain contract state storage — no workaround needed. If you've spent hours debugging scripts that fail because the wrong output went back to the script address with the wrong datum, this will feel like a significant relief.

### Concurrency: A Familiar Problem with a Different Shape

Cardano developers know the concurrency problem well: because all parties must consume the same script UTXO to interact with a contract, only one transaction can touch it per block. The solutions — batchers, order books, off-chain coordination — are a significant part of Cardano DeFi architecture.

Midnight doesn't have this specific problem in the same form. Ledger state updates at the contract level are serialized (only one circuit call can modify a contract's ledger per block), but UTXO-level token transfers are naturally parallel since each UTXO is independent. The contention concern shifts from script-UTXO throughput to contract-level update throughput — a narrower bottleneck in most designs.

Complex DeFi patterns that required batcher architecture on Cardano may benefit from redesign when porting to Midnight, since the root constraint is different.

---

## 8. Transactions, Fees, and the Two-Phase Model

Cardano transaction structure is one of the most carefully designed in the industry: inputs, outputs, mint field, validity range, required signers, reference inputs, collateral, metadata. Midnight transaction structure shares the UTXO input/output backbone but replaces everything that was about on-chain script execution with ZK proof equivalents.

### What a Midnight Transaction Contains

- Circuit invocations (the contract calls, with their arguments and ZK proofs)
- UTXO inputs being consumed (shielded or unshielded)
- UTXO outputs being created (payments, change, new token UTXOs)
- Nullifiers for any shielded UTXOs being spent
- DUST fee payment

### Cardano vs. Midnight: Transaction Field Comparison

| Cardano tx field | Midnight equivalent | Notes |
|---|---|---|
| `inputs` | UTXO inputs | Same concept — UTXOs consumed |
| `outputs` | UTXO outputs | Same concept — UTXOs created |
| `mint` | Zswap balance vector adjustment | Minting via circuit + balance vector |
| `collateral` | None | No Phase 2 failure risk; no collateral needed |
| `required_signers` | None | Signers proved via ZK, not explicit field |
| `validity_range` | None | No block timestamp; use round counter |
| `reference_inputs` | None | Circuits cannot read external UTXOs |
| `redeemers` | Circuit parameters + ZK proof | Logic proved, not executed on-chain |
| `scripts` | Included in proof (off-chain) | No script submission; proof carries correctness |
| `fee` (ADA) | DUST fee | Non-transferable, generated by NIGHT holdings |

### No Collateral — And Why

Cardano requires collateral for Phase 2 script execution because a validator might return `False` after consuming network resources. On Midnight, there's no on-chain script execution to fail mid-run. The ZK proof is either valid or it isn't, and proof verification is cheap and deterministic. If the proof is invalid, the transaction is rejected before any state change occurs, at negligible cost. This removes one of the more friction-filled aspects of Cardano dApp UX — users no longer need to set aside collateral UTXOs.

### The Two-Phase Execution Model

**This is absent from most Midnight documentation and from the Cardano mental model — read carefully.**

Unlike Cardano where a transaction either fully succeeds or fully reverts (minus fee), Midnight splits execution into three stages:

**Stage 1 — Well-formedness check:** Runs against no state. Verifies all ZK proofs and that offers are balanced (accounting for fees and mints). A bad proof or unbalanced offer → rejected outright, not included.

**Stage 2 — Guaranteed phase:** Runs against ledger state. **DUST fees are collected here.** If this phase fails, the transaction is *not included at all* — similar to a Cardano pre-submission script failure.

**Stage 3 — Fallible phase:** Also runs against ledger state. If this fails, the guaranteed phase effects **still apply**, and the transaction is recorded as a **partial success**. Fees are forfeited.

This means a Midnight transaction can land in a partially-applied state with no clean revert. Design circuits so partial success is always a safe state. Put must-succeed logic in the guaranteed phase; put speculative effects in the fallible phase and ensure they're always safe to leave incomplete.

**On fees for failed transactions:** Guaranteed-phase DUST is forfeited on fallible-phase failure. If the transaction fails in the well-formedness check or guaranteed phase, it's not included and no DUST is spent.

---

## 9. Toolchain and Development Environment

Cardano has a rich toolchain: Aiken, Plutus/Haskell, Helios, OpShin, and Plu-ts for on-chain, with Lucid/Mesh/Blaze/Cardano-js-sdk for off-chain. Midnight's toolchain is more consolidated. The biggest new addition with no Cardano equivalent: **the proof server daemon**, a separate process your dApp must run for ZK proof generation.

### Tool Mapping: Cardano → Midnight

| Cardano Tool | Midnight Equivalent | Notes |
|---|---|---|
| Aiken (on-chain language) | Compact (`compactc`) | TypeScript-like DSL; compiles to ZK circuits |
| Plutus / Haskell | Compact | Compact replaces all on-chain language options |
| `aiken build` | `compactc` | Compiles `.compact` → ZK circuits + TS stubs |
| Lucid / Mesh / Blaze | Midnight.js SDK | TypeScript SDK for tx building and submission |
| Blueprint (contract ABI) | Compiler-generated TS stubs | Typed interfaces to your circuits, auto-generated |
| ogmios / Kupo | Midnight node RPC | Standard JSON-RPC for chain queries |
| `cardano-node` | Midnight DevNet node | Local chain for development and testing |
| CIP-30 wallet connector | Midnight wallet SDK | Wallet integration for NIGHT, DUST, UTXOs |
| `aiken check` / `aiken test` | Compact test suite | Tests run against local DevNet |
| Eternl / Nami / Lace | Lace wallet (Midnight-enabled) | IOG's Lace supports Midnight natively |
| *(no equivalent)* | Proof server daemon | Separate process for client-side ZK proof generation |
| `@openzeppelin/compact-contracts` | Audited contract modules | `Ownable`, `Pausable`, `FungibleToken` — experimental |
| Midnight MCP | AI-assisted dev tool | Live Compact compiler integration for AI editors |

### Project Structure

```
my-midnight-dapp/
├── contracts/
│   ├── my_contract.compact          # Compact source (replaces src/validators/*.ak)
│   └── managed/                     # Compiler output — do not edit
│       └── my_contract/
│           ├── contract/index.cjs   # TypeScript stubs (replaces blueprint.json)
│           ├── keys/                # ZK proving/verifying keys
│           └── zkir/                # ZK intermediate representation
└── src/
    ├── index.ts                     # dApp logic using Midnight.js SDK
    └── witnesses.ts                 # Implement witness functions here
                                     # (secret key access, off-chain data)
```

### Development Workflow

1. **Write contract:** Author your `.compact` file. Declare ledger fields, witness function signatures, and exported circuits.
2. **Compile:** Run `compactc`. The compiler is strict — `disclose()` violations, non-constant loop bounds, and type mismatches are compile errors.
3. **Start proof server:** Run the proof server daemon. Your SDK calls route through it for proof generation. This is a separate process from your dApp; budget for it in development setup and deployment.
4. **Implement witnesses:** The compiler generates TypeScript interfaces for each witness function. You implement the bodies — secret key retrieval, off-chain data access.
5. **Test:** Run against local DevNet. Test circuit correctness, proof generation, and both happy paths and failing assertions.
6. **Deploy:** Midnight.js SDK. Deployer's wallet pays DUST. Contract is immutable post-deployment.
7. **Build dApp:** Use compiler-generated TypeScript stubs as typed circuit interfaces. Build and submit transactions via the SDK.

> **Proof generation is the new UX-critical latency.** On Cardano, the bottleneck is block confirmation time. On Midnight, it's proof generation — which happens client-side and currently takes seconds for typical circuits. Aiken's `ExUnits` budget gives you a cost model at compile time; Midnight's equivalent is circuit constraint count. Keep circuits lean.

---

## 10. Security: What Changes, What Disappears, What's New

Cardano has a well-understood set of security patterns and failure modes: double satisfaction, unbounded datum growth, concurrency attacks, policy ID confusion, missing required signer checks, and the subtleties of inline vs. hash datums. Midnight eliminates some of these structurally while introducing a distinct set of new concerns.

### Cardano Vulnerabilities That Don't Exist on Midnight

- **Double satisfaction:** The canonical Cardano bug — two inputs satisfied by one output. On Midnight, circuits don't construct or inspect output sets. State lives in the ledger; there's no continuing output to confuse.
- **Missing signer checks:** Replaced structurally — but see the warning below about incorrect replacement patterns.
- **Policy ID confusion:** Midnight token types are cryptographically derived from the contract address + domain separator. Two tokens with the same name but different contracts have different types by construction.
- **Unbounded datum growth:** No datum-per-UTXO pattern. Ledger fields are typed with compile-time-known sizes.
- **Collateral loss:** No collateral mechanism; no Phase 2 failure costs.
- **Reference input staleness:** Circuits don't read reference inputs; this class of bugs doesn't apply.

### Security Patterns That Still Apply

- **`assert()` for invariants:** Same concept as Aiken's `expect`. Proof fails on violation — equivalent to the validator returning `False`.
- **Access control completeness:** Every restricted circuit needs its own `assert(ownerKey(secretKey()) == authority)`. There is no inherited validator context. Check it explicitly in each circuit that requires it.
- **Round counter increment:** Always call `round.increment(1)` in every circuit that modifies state. Without it, transaction correlation analysis becomes easier — similar to reusing the same datum nonce.
- **Arithmetic bounds:** Compact uses fixed-size integers (`Uint<64>`, `Uint<128>`). Midnight doesn't have Cardano's arbitrary-precision `Int`. Large-value arithmetic needs care.

### New Failure Modes

**`ownPublicKey()` and unconstrained public key access** is the most dangerous footgun for Cardano developers moving to Midnight. The pattern that looks like Aiken's `ctx.extra_signatories` check — call something to get the caller's public key, compare it to a stored value — is broken in the ZK context. The official Midnight security guide states explicitly: *"Never use `ownPublicKey()` for verification of the caller."* Use the commitment scheme with `persistentHash` and domain separation as shown in §5.

**Disclosure mistakes** are the most common Compact bug class. Using `disclose()` on data you didn't intend to make public. The compiler catches accidental leakage of witness data but not intentional disclosure you didn't think through. Audit every `disclose()` call.

**Commitment brute-forcing:** If you commit to a small-domain value (a boolean, a small integer) without a random salt, the commitment is trivially invertible. Use `persistentCommit()` with randomness for small-domain secrets.

**Witness implementation bugs:** `compactc` validates the circuit interface and data flow. It cannot validate your TypeScript witness implementations. A buggy witness that leaks data or returns incorrect values won't be caught at compile time. Test witness logic independently. The `@openzeppelin/compact-contracts` library ships test witnesses only — not production witnesses. If you use it, author and audit your own.

**Partial-success states:** Given the two-phase execution model (§8), a fallible-phase failure can leave your contract in a partially-applied state with no clean revert. Design circuit boundaries so partial success is always safe.

---

## 11. Quick Reference: Aiken / Cardano → Compact / Midnight

| Cardano / Aiken | Compact / Midnight | Notes |
|---|---|---|
| `validator { spend(...) }` | `export circuit myCircuit(...)` | `export` = callable in a tx |
| `validator { mint(...) }` | `export circuit mint(...)` | Minting logic is a circuit |
| `datum: Option<MyDatum>` | `export ledger field: Type;` | No per-UTXO datum; state in ledger |
| `redeemer: MyRedeemer` | Circuit parameters | Args passed directly to circuit |
| `ctx: Transaction` | No equivalent | Circuits don't see full tx context |
| `ctx.extra_signatories` | Custom `persistentHash` circuit + witness | `ownPublicKey()` is unsafe — see §5 |
| `expect Some(x) = opt` | `assert(condition, "msg")` | Both abort on failure |
| `trace(..., False)` | `assert(false, "msg")` | Explicit failure |
| Continuing output pattern | `ledger.field = disclose(newValue)` | Direct ledger mutation; no output to construct |
| `ledger { }` block syntax | `export ledger field: Type;` | Flat declarations; block syntax fails compilation |
| Script address | Contract address | Different derivation model |
| Reference inputs | No equivalent | Circuits cannot read external UTXOs |
| `validity_range` / `POSIXTime` | `round: Counter` (ledger field) | No timestamps; use round counter |
| Collateral UTXO | None | No Phase 2 execution; no collateral needed |
| minADA on output | None | No minimum token deposit on UTXOs |
| Policy ID + Asset Name | `Hash(contractAddr, domainSeparator)` | Token type cryptographically bound to contract |
| Minting policy script | Circuit + Zswap balance vector | No separate policy script type |
| One-shot policy | `sealed` ledger field | Set in constructor; immutable afterward |
| Native multi-asset UTXO | Native UTXO token model | Both first-class; Midnight adds shielding |
| Concurrent UTXO contention | Contract ledger update serialization | Different bottleneck; different solutions |
| cNIGHT (Cardano native) | mNIGHT → DUST generation | Register reward address + DUST key on Cardano |
| `aiken build` | `compactc` | The Compact compiler |
| Lucid / Mesh / Blaze | Midnight.js SDK | TypeScript off-chain SDK |
| `blueprint.json` (ABI) | Compiler-generated TypeScript stubs | Typed circuit interfaces, auto-generated |
| `aiken check` | `compactc` (strict compiler) | Errors on `disclose()` violations, bound issues |
| `ByteArray` | `Bytes<32>` / `Bytes<N>` | Fixed-size; specify width at compile time |
| `Int` (arbitrary precision) | `Uint<64>` / `Uint<128>` | Fixed-width integers; audit overflow |
| `Bool` | `Boolean` | Same concept |
| `pub type Foo { field: Int }` | `struct Foo { field: Uint<64> }` | Custom types supported in Compact |
| `ExUnits` (execution budget) | Circuit constraint count | Budget is compile-time, not runtime-metered |

---

## 12. Suggested Learning Path

1. **Install the toolchain** → start the proof server → clone `example-hello-world` → deploy to local DevNet. Get the compile → proving-keys → TypeScript-driver loop in your fingers before writing real logic. ([docs.midnight.network/getting-started](https://docs.midnight.network/getting-started))
2. **Work the bulletin board tutorial** ([docs.midnight.network/tutorials/bboard](https://docs.midnight.network/tutorials/bboard)). Study the `publicKey` helper circuit carefully — it's the canonical correct access control pattern. Understand why the domain tag and round counter are in the commitment.
3. **Read the smart contract security page** ([docs.midnight.network/compact/smart-contract-security](https://docs.midnight.network/compact/smart-contract-security)). Specifically: the `ownPublicKey()` warning, `sealed` fields, hash-based authentication, and commitment brute-forcing.
4. **Read the `Writing a contract` tutorial** ([docs.midnight.network/compact/writing](https://docs.midnight.network/compact/writing)). It shows the full lifecycle of a stateful contract with access control — the closest thing to a canonical Compact reference.
5. **Read the DUST architecture page** ([docs.midnight.network/concepts/dust-architecture](https://docs.midnight.network/concepts/dust-architecture)). As a Cardano developer you need to understand the cNIGHT → DUST cross-chain flow in detail.
6. **Read the "How Midnight Works" concept pages** — transaction semantics (especially guaranteed/fallible phases), Zswap, Kachina.
7. **Explore `@openzeppelin/compact-contracts`** for `Ownable`, `Pausable`, and `FungibleToken` patterns — actively maintained but flagged experimental. Author your own witnesses; do not reuse the test witnesses shipped with the library.

---

## Resources

| Resource | URL |
|---|---|
| Official Docs | https://docs.midnight.network |
| Compact Language Reference | https://docs.midnight.network/compact |
| Writing a Contract (canonical tutorial) | https://docs.midnight.network/compact/writing |
| Bulletin Board Tutorial | https://docs.midnight.network/tutorials/bboard |
| Smart Contract Security | https://docs.midnight.network/compact/smart-contract-security |
| Transaction Semantics | https://docs.midnight.network/concepts/how-midnight-works/semantics |
| Zswap | https://docs.midnight.network/concepts/how-midnight-works/zswap |
| DUST Architecture | https://docs.midnight.network/concepts/dust-architecture |
| UTXO Model | https://docs.midnight.network/concepts/utxo |
| OpenZeppelin for Compact | https://docs.openzeppelin.com/contracts-compact |
| Midnight Discord | https://discord.com/invite/midnightnetwork |
| Midnight MCP (AI-assisted dev) | https://docs.midnight.network/blog/midnight-mcp-ai-assisted-development |

---

*Sourced from official Midnight documentation, the Midnight tokenomics whitepaper, and Midnight improvement proposals (MIP-020). Current as of June 2026. Compact 1.0 does not yet support cross-contract calls. `ownPublicKey()` must not be used for access control — confirmed against compiler v0.30.0 and documented in the official security guide. Midnight mainnet is in a federated phase with Cardano SPO onboarding in progress. Zswap/coin mechanics are flagged by the official docs as not yet performance-optimized — verify against current docs before shipping production integrations.*
