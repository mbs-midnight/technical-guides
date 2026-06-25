# From Solana to Midnight — Developer Field Guide

*You already understand parallelism and stateless programs. The rest is a privacy upgrade.*

**Midnight Network Foundation | June 2026**

> **Version note:** Midnight mainnet launched March 2026. Compact targets `language_version >= 0.23`. Cross-contract calls are reserved in the language spec but not yet implemented in Compact 1.0 — see §4. Always verify against [docs.midnight.network](https://docs.midnight.network) before shipping.

---

## 1. Why Midnight? The Pitch for Solana Devs

Solana solved throughput. Tens of thousands of TPS, sub-cent fees, parallel execution via Sealevel — impressive engineering. But every single transaction is still completely public. Every wallet balance, every trade, every on-chain interaction is readable by anyone, forever. That's fine for a DEX order book. It's a showstopper for regulated finance, healthcare data, enterprise business logic, or anything involving real-world compliance requirements where you can't just dump everyone's financials on a globally readable ledger.

Midnight is built specifically for the cases where you need programmable logic on a blockchain but cannot expose the underlying data. Privacy is cryptographic — zero-knowledge proofs — not policy-based ("trust us, the data is off-chain somewhere").

**One-line pitch:** Midnight is a privacy-first blockchain where smart contract logic is proved correct off-chain using ZK proofs, private state never hits the chain, and public verifiability is preserved without public visibility.

### What Carries Over from Solana

- **Stateless program logic** — Compact contracts don't hold state internally, similar to how Solana programs are stateless executors
- **Parallel transaction processing by design** — UTXO independence mirrors Solana's account-disjoint parallelism (with an important caveat — see §3)
- **TypeScript-heavy off-chain development** — Midnight's SDK is TypeScript; Solana's `@solana/kit` is TypeScript
- **Separation of code and data** — you'll feel at home thinking about where state actually lives
- **Native token standards with per-UTXO ownership** — conceptually closer to SPL Token Accounts than ERC-20 mappings

### What Needs Rewiring

- **Rust / Anchor** — Midnight's contract language is Compact, TypeScript-inspired, compiled to ZK circuits
- **Account passing** — you don't declare account lists in transactions; UTXO ownership is proven via ZK proof
- **PDAs** — no equivalent; program-controlled state works differently in Compact
- **CPI (Cross-Program Invocation)** — cross-contract calls are **not yet implemented** in Compact 1.0
- **Everything is public by default on Solana** — on Midnight, everything is private by default
- **SOL / compute units / rent** — Midnight uses NIGHT + DUST, a fundamentally different fee model with no rent concept

---

## 2. Execution Model: The Biggest Conceptual Shift

Solana's execution model: the runtime invokes your program on-chain, your program reads and writes accounts passed to it, Sealevel runs non-overlapping transactions in parallel. Fast, predictable, fully transparent.

Midnight's execution model differs in one fundamental respect: **the contract logic doesn't run on-chain at all.** It runs on the user's machine. The chain receives a cryptographic proof that the logic ran correctly, verifies it, and updates state. Validators never see the private inputs and never re-execute your code.

| Aspect | Solana | Midnight |
|---|---|---|
| Where logic runs | On-chain, all validators re-execute | Off-chain, on user's machine |
| What hits the chain | Signed transaction + instruction data | Transaction + ZK proof of correct execution |
| Private inputs | Not possible — all data visible | Witness functions — local, never on-chain |
| Validator role | Execute + verify | Verify proof only (faster, no re-execution) |
| Parallelism basis | Non-overlapping account write sets | Independent UTXOs (no shared mutable state) |
| State mutability | Accounts mutated in-place | UTXOs consumed; new ones created |
| Language | Rust (usually Anchor) | Compact (TypeScript-inspired DSL) |
| Privacy default | Everything public | Everything private unless explicitly disclosed |

### The ZK Proof Pipeline

When a user invokes a Midnight contract circuit, here's what actually happens:

1. **Witness execution:** Private functions (witnesses) run locally, consuming the user's private data — spending keys, off-chain records, anything sensitive.
2. **Circuit execution:** The contract circuit runs locally using witness outputs as private inputs. This is the business logic.
3. **Proof generation:** The Kachina proving system generates a ZK proof that the circuit ran correctly. This happens client-side via a **proof server process** (a separate daemon you run alongside your dApp) and takes real time — currently seconds for typical circuits.
4. **Transaction submission:** The proof, plus any public state changes and UTXO outputs, is submitted to the network.
5. **Validation:** Validators verify the proof (fast), check nullifiers for double-spends, and update the public ledger.

> **The UX-critical path is proof generation, not confirmation time.** On Solana you design around network confirmation latency (~400ms). On Midnight you're designing around proof generation time (seconds, client-side). Keep circuits lean. Don't let constraint count creep up.

> **The proof server is a process, not a library.** Unlike Solana where your dApp talks directly to an RPC endpoint, Midnight requires a running proof server daemon. Your Midnight.js SDK calls connect to it locally. Budget for this in your dev setup and deployment architecture.

---

## 3. State Model: From Accounts to UTXOs

This is the most structurally significant difference for Solana developers. Solana's everything-is-an-account model is one of its defining innovations. Midnight's primary state model is **UTXO-based** — closer to Bitcoin or Cardano in lineage, extended with ZK-native privacy features.

### Solana Accounts: A Refresher

In Solana, everything is an account: wallets, programs, token balances, program state. Each account has an address (public key), lamports, a data buffer, an owner program, and an executable flag. Programs are stateless — they only touch accounts explicitly passed in each instruction.

```rust
// Solana account structure
pub struct Account {
    pub lamports:   u64,      // balance in lamports
    pub data:       Vec<u8>,  // arbitrary data buffer
    pub owner:      Pubkey,   // program that can modify this
    pub executable: bool,     // is this a program?
    pub rent_epoch: Epoch,    // legacy rent tracking
}

// Program state lives in a PDA (data account) passed per-instruction
// Programs are stateless — state is in accounts, not in the program itself
```

### Midnight UTXOs: What Changes

Midnight has no mutable accounts. Instead, value and state are held in UTXOs — discrete, immutable objects. Spending a UTXO consumes it entirely and creates new outputs. There is no "update this account's data buffer." Every state change is a create-and-consume operation.

```
// Midnight UTXO mental model (pseudocode — not Compact syntax)
UTXO_1 = { value: 100 NIGHT, owner: AlicePublicKey }

// To send 60 NIGHT to Bob:
Transaction {
  inputs:  [UTXO_1],                        // consume entirely — UTXO_1 ceases to exist
  outputs: [
    { value: 60, owner: Bob   },            // payment
    { value: 40, owner: Alice },            // change back to Alice
  ]
}
// Result: UTXO_1 is nullified; two new UTXOs exist
```

### The Nullifier Set: No Solana Equivalent

Solana prevents double-spends by checking live account state — you can't spend SOL you don't have because the balance is a mutable value the runtime checks. Midnight uses a **nullifier set**: when a UTXO is spent, a nullifier (a hash derived from the UTXO and the owner's private key) is added to a global set. Future transactions check this set before accepting a spend.

The privacy insight: a nullifier can be validated without revealing which UTXO it corresponds to. Double-spend prevention works even on shielded UTXOs where the chain doesn't know the value or owner.

### UTXO Parallelism — and the Contention Gotcha

Solana's parallelism comes from disjoint write sets: if two transactions don't overlap on writable accounts, they execute concurrently. Midnight's parallelism comes from UTXO independence: each UTXO is a separate object, and two transactions spending different UTXOs never conflict.

**But watch out for UTXO contention.** If two transactions try to spend the same UTXO simultaneously (a race that's easy to hit in high-throughput dApps or wallets with few large UTXOs), only one succeeds and the other fails. Solana devs are used to hot accounts being the contention point; on Midnight, contention is about which UTXO gets nullified first. The mitigation: keep wallets with many small UTXOs rather than few large ones, and design dApp flows to avoid concurrent spending of the same UTXO.

### Program Accounts vs. Compact Ledger State

In Solana, your program stores state in data accounts it owns — usually PDAs. You size and allocate those accounts upfront and pay rent-exempt deposits for their storage.

In Midnight, on-chain state for a contract is declared directly in the contract source with `export ledger` declarations. It lives on the public ledger automatically. There are no separate account allocations, no rent deposits, and no account size limits to pre-calculate.

| Concept | Solana | Midnight |
|---|---|---|
| Where contract state lives | Data accounts (PDAs) passed per-instruction | `export ledger` fields, on-chain automatically |
| State initialization | Create account, pay rent-exempt deposit | Deployed via constructor — no rent concept |
| State access pattern | Passed explicitly in instruction account list | Accessed directly in circuit body |
| Account size limits | Must allocate upfront; reallocating is complex | No per-contract size pre-allocation |
| Program-controlled addresses | PDA (seeds + program ID + bump) | Contract address derived at deployment |
| State mutability | Mutable data buffer in-place | UTXO consumed + new one created |

---

## 4. Compact vs. Rust/Anchor: The Language Comparison

This is the steepest part of the learning curve — but also where Midnight surprises you. Compact is TypeScript-inspired, not Rust. If you've been grinding through Anchor's account validation macros and Borsh serialization, Compact's syntax will feel like a vacation.

The tradeoff: Compact has constraints Rust doesn't. It compiles to ZK circuits, which means no dynamic loops with runtime-determined bounds, no heap allocation, no I/O, and — critically — **no cross-contract calls in the current release**. What you give up in generality, you gain in privacy guarantees the compiler enforces automatically.

### Anatomy of a Compact Contract

A Compact contract has three execution contexts:

- **Public ledger:** On-chain state declared with `export ledger`. Visible to everyone. The equivalent of your Solana PDA data.
- **Circuits:** Functions compiled to ZK circuits. Run off-chain on the user's machine. These are your program entry points — equivalent to Anchor instruction handlers.
- **Witnesses:** Functions that run locally with access to private data. Never go on-chain. No Solana equivalent — this is the genuinely new concept.

### Basic Contract Structure

```compact
pragma language_version >= 0.23;
import CompactStandardLibrary;

// On-chain state — each field declared individually (NOT a ledger { } block)
export ledger owner: Bytes<32>;
export ledger value: Uint<64>;
export ledger round: Counter;   // always include a round counter — see §7

// Witness: reads private key locally — never leaves the user's machine
witness getSpendingKey(): Bytes<32>;

constructor(initialOwner: Bytes<32>) {
    owner = disclose(initialOwner);
    round.increment(1);
}

// Circuit: exported entry point, equivalent to an Anchor instruction handler
export circuit setValue(newValue: Uint<64>): [] {
    // Access control: prove knowledge of the owner's private key
    assert(persistentHash(getSpendingKey()) == owner, "Not authorized");
    value = disclose(newValue);
    round.increment(1);
}

export circuit getValue(): Uint<64> {
    return value;
}
```

**Ledger syntax note:** Each ledger field is declared with its own `export ledger fieldName: Type;` statement. There is no block `ledger { }` syntax in current Compact — that appeared in earlier pre-mainnet versions. The compiler will reject the block form.

### Anchor vs. Compact: Side-by-Side

| Concept | Anchor (Rust) | Compact |
|---|---|---|
| Program structure | `#[program]` mod with instruction fns | File IS the contract; circuits are entry points |
| State storage | `#[account]` struct in a PDA | `export ledger field: Type;` declarations |
| Account validation | `#[derive(Accounts)]` struct + constraints | No account list — ownership proved via ZK |
| Access control | Signer checks in Accounts struct | `assert(persistentHash(witness()) == owner)` |
| Private inputs | Not possible | `witness` functions (off-chain, local) |
| Explicit disclosure | Not needed (everything public) | `disclose()` required for private → public |
| Instruction handler | `pub fn my_ix(ctx: Context<MyAccounts>, ...)` | `export circuit myCircuit(...): []` |
| Error handling | `return err!(MyError::Unauthorized)` | `assert(condition, "message")` — proof fails |
| CPI / cross-contract | `invoke()` / `invoke_signed()` | **Not yet implemented in Compact 1.0** |
| Build tool | `anchor build` | `compactc` (Compact compiler) |
| Bounded loops | No restriction (runtime compute-metered) | Bounds must be compile-time constants |
| Serialization | Borsh (manual derive) | Automatic — Compact types serialize natively |
| Deploy | `anchor deploy` | Midnight.js SDK deploy |

### A Note on Cross-Contract Calls

This is a hard wall Solana devs will hit immediately. Solana's CPI model is fundamental to how programs compose — SPL Token, Metaplex, and virtually every protocol depend on programs calling other programs mid-instruction.

**In Compact 1.0, this does not exist.** The `contract` keyword is reserved in the language spec for future cross-contract call support, but it is not currently implemented. Circuits cannot call circuits in other deployed contracts at runtime.

The workaround for now: **module imports** (sharing circuit logic at compile time, not at runtime). You can import and reuse Compact modules from libraries like `@openzeppelin/compact-contracts` within a single compiled contract. But two separately deployed contracts cannot call each other. Design your contracts accordingly — monolithic where Solana would be composable, for now.

The OZ Ownable pattern on Midnight (since you can't use standard CPI-based ownership transfer) reflects this: it explicitly notes that contract-to-contract ownership transfer is unsupported until cross-contract calls land.

### Access Control Without Signer Checks

Solana's access control is explicit: you declare which accounts must be signers in your Accounts struct, and the runtime enforces it. There is no `msg.sender`, but the signer pattern is clear.

Midnight has no signer list. Instead, access control is **proven via knowledge of a secret**. The contract stores a commitment (hash) of the owner's identity on-chain; the caller proves they hold the corresponding private key through a witness function. The proof verifies this without revealing the key.

```rust
// Solana / Anchor pattern
#[derive(Accounts)]
pub struct UpdateValue<'info> {
    #[account(mut, has_one = owner)]
    pub state:  Account<'info, MyState>,
    pub owner:  Signer<'info>,   // runtime enforces this must sign
}
```

```compact
// Compact equivalent — no signer list; ZK proof of key knowledge instead
witness getSpendingKey(): Bytes<32>;

export circuit updateValue(newValue: Uint<64>): [] {
    // Prove caller knows the private key behind the stored commitment
    assert(persistentHash(getSpendingKey()) == owner, "Not authorized");
    value = disclose(newValue);
    round.increment(1);
}
```

This pattern — store a commitment to a public key or secret, prove knowledge via witness — is the Midnight equivalent of Anchor's `Signer` constraint. It's used everywhere: token ownership, governance, role-based access. Get comfortable with it early.

### The TS SDK as Transaction Orchestrator: Replacing CPI

On Solana, CPI lets one program call another mid-execution — a single transaction can chain program logic atomically. On Midnight, circuits are closed computations. They cannot call other contracts at runtime.

The equivalent composability pattern moves to the **TypeScript SDK layer**, which assembles multi-step flows before anything hits the chain:

```typescript
// Solana mental model: Program A calls Program B via CPI mid-execution
// (atomic, single transaction, on-chain)

// Midnight mental model: SDK orchestrates multiple circuit calls,
// bundled into a single transaction with multiple proofs

const tx = await sdk.buildTransaction([
  // Step 1: prove eligibility (circuit call 1)
  await credentialContract.proveEligibility({
    witnesses: { getCreditScore: () => wallet.getCreditScore() }
  }),

  // Step 2: execute the action that depends on eligibility (circuit call 2)
  await vaultContract.withdraw({
    witnesses: { secretKey: () => wallet.getSecretKey() },
    args: { amount: 1000n }
  }),

  // Both proofs + their UTXO flows bundled into one atomic transaction
]);

await wallet.submitTransaction(tx);
```

**The key difference from CPI:** the SDK assembles both circuit calls *before* submission. The transaction is atomic — both succeed or neither applies — but the composition happens off-chain in TypeScript, not mid-execution on-chain. This means:

- Contract B cannot read or modify state based on what Contract A just computed in the same execution context
- Cross-contract *data* dependencies must be expressed through shared ledger state that was committed in a prior transaction
- The TS layer is where your protocol's sequencing logic lives, not the circuit

For Solana developers: think of it less like CPI and more like building a versioned transaction with multiple instructions — except the "instructions" are ZK proofs, and the sequencing guarantee is enforced by the transaction's well-formedness check rather than the runtime.

---

## 5. The Privacy Model: Private by Default

Everything on Solana is public. Every account, every instruction argument, every PDA content. This is not a flaw — it's the design. Sealevel's parallelism requires every account to be declared upfront, making every account visible in the transaction.

Midnight inverts this. **Private by default** means any value derived from a witness function is considered private. The compiler tracks data flow and requires you to explicitly use `disclose()` before any private-origin data can appear on-chain. Skip it and the compiler rejects the contract — this is a compile-time privacy guarantee, not a runtime check.

### The `disclose()` Pattern

```compact
witness _secretBalance(): Uint<64>;   // underscore = private by convention

export circuit reportBalance(): [] {
    const bal = _secretBalance();

    // ❌ COMPILE ERROR: witness-derived value cannot land on-chain directly
    ledger.publicTotal = ledger.publicTotal + bal;

    // ✅ CORRECT: explicitly declare intent to make this value public
    ledger.publicTotal = ledger.publicTotal + disclose(bal);
}
```

`disclose()` is not a cast — it's a compiler annotation. It signals "I know this value is leaving the private domain; I accept responsibility." The value only appears on-chain if it's stored in ledger state or returned from an exported circuit. Think of it as intentionality enforcement — the compiler makes you say out loud that you meant to do this.

### Three Privacy States for Data

| State | Where it lives | Who can see it | Solana equivalent |
|---|---|---|---|
| Witness data | User's local machine only | Only the user | No equivalent — Solana has none |
| Private circuit computation | In-flight during proof generation | Nobody, not even the chain | No equivalent |
| Disclosed / ledger state | Public ledger on-chain | Everyone | PDA data / account state |

### Shielded vs. Unshielded UTXOs

Token UTXOs in Midnight can be shielded or unshielded. This is a per-UTXO choice:

- **Unshielded:** Value and owner visible on-chain. Same transparency as a Solana token account balance.
- **Shielded:** Value and owner hidden behind cryptographic commitments. The network sees a commitment hash; only the owner knows the contents.

A single transaction can mix shielded and unshielded UTXOs. A regulated asset might be shielded for user-to-user transfers but require unshielded disclosure to a compliance auditor via viewing key — both in the same transaction.

---

## 6. Tokens: From SPL to Midnight's UTXO Token Model

### NIGHT and DUST: The Protocol Tokens

Midnight has two native protocol-level components:

**NIGHT** is the transferable utility token. Used for staking, governance, and generating DUST. Think of it like SOL's role in staking — but NIGHT is not directly used for transaction fees. All NIGHT transfers are publicly visible on-chain. It is deliberately not a privacy coin.

**DUST** is a non-transferable resource generated passively by holding NIGHT. It's used to pay transaction fees. It decays if unused.

> **Analogy:** SOL on Solana is both your staking asset and your gas token — when demand spikes, fee volatility hits validators and users simultaneously. NIGHT and DUST decouple these roles. Fees stay predictable regardless of NIGHT's market price because DUST isn't traded — it's generated proportionally to NIGHT holdings.

Key DUST properties:
- **Shielded.** Fee payment doesn't expose your wallet address.
- **Non-transferable.** Cannot be sent between wallets — but can be *delegated* to sponsor others' transactions.
- **Decaying.** Unused DUST bleeds off; move your NIGHT and it decays to zero.
- **Renewable.** Consume it and it refills over time based on your NIGHT holdings.

**The cold-start problem.** New wallets have no DUST. Without DUST, they cannot pay fees and cannot transact. The solution is **DUST sponsorship**: a sponsor wallet (typically a dApp holding NIGHT) pays the DUST fees for a new user's transaction. The Midnight SDK exposes this via a two-phase balancing model: the user balances their token flows, the sponsor covers the DUST portion. This is the Midnight equivalent of Solana's compute budget fee payer pattern, but at a deeper level — design sponsored flows into your dApp from day one if you want users who don't hold NIGHT.

**Sponsored transactions:** A dApp holds NIGHT, generates DUST, and delegates it to cover users' fees — making the app feel gasless at the point of use. Conceptually similar to Solana's fee payer account pattern, but structural rather than an SDK workaround.

### Custom Tokens: SPL vs. Midnight UTXO Tokens

| Property | SPL Token (Solana) | Custom Token (Midnight) |
|---|---|---|
| Token definition | Mint Account (one per token type) | 256-bit token type ID derived from contract address + domain separator |
| Balance storage | Token Account (one per owner per mint) | UTXOs on the ledger (shielded or unshielded) |
| Transfer mechanism | Token Program mutates Token Account data | Consume UTXO input, create UTXO output with ZK proof |
| Spend authorization | Account signer check at runtime | ZK proof of UTXO ownership |
| Privacy option | Token-2022 confidential transfers (opt-in, complex) | Shielded UTXOs native to the model |
| Atomic multi-token swap | Multiple instructions, multiple programs | Native via Zswap protocol |
| Approve / delegate pattern | Required before delegated spend | No approve — ZK proof owns the spend |
| ATA equivalent | Associated Token Account (PDA-derived) | No equivalent; UTXOs self-describe ownership |

### Token-2022 vs. Midnight: The Confidential Transfer Comparison

If you've worked with Token-2022 confidential transfers (PYUSD uses them), you've touched the edge of what Midnight does natively — and you've seen how much complexity the extension model adds.

| Aspect | Token-2022 Confidential Transfers | Midnight Shielded Tokens |
|---|---|---|
| Privacy scope | Balance amounts only | Amounts + ownership + metadata |
| Implementation | Extension bolted onto SPL Token Program | Native UTXO property |
| Proof type | Bulletproofs (range proofs) | zk-SNARKs (Kachina/Zswap) |
| Auditor support | Auditor ElGamal key in extension state | Viewing key architecture |
| Developer overhead | High — pending balances, apply-pending flows | Lower — shielding is default UTXO behavior |
| Regulatory compliance | Possible but complex | Designed-in (viewing keys, selective disclosure) |

### Zswap: Atomic Multi-Asset Swaps

Zswap is Midnight's native atomic swap protocol, built into the transaction layer — not a DEX program on top. Where Solana DEXes (Jupiter, Raydium, Orca) are programs composing token transfers, Zswap is a protocol-level primitive.

The core concept is an **offer** — a bundle specifying input UTXOs consumed and output UTXOs created, with a balance vector tracking net value flow per token type. Two parties' offers can be merged non-interactively by an off-chain matcher. The matcher verifies that the combined balance vector nets to zero without seeing the amounts. Neither party's trade size is visible pre-broadcast.

```
// Conceptual Zswap offer structure (pseudocode)
AliceOffer {
    inputs:  [shielded_NIGHT_100],       // Alice spends 100 NIGHT
    outputs: [shielded_USDC_95],         // Alice receives 95 USDC
    balance: { NIGHT: -100, USDC: +95 }
}

BobOffer {
    inputs:  [shielded_USDC_95],         // Bob spends 95 USDC
    outputs: [shielded_NIGHT_100],       // Bob receives 100 NIGHT
    balance: { NIGHT: +100, USDC: -95 }
}

// Matcher merges: combined balance = { NIGHT: 0, USDC: 0 } ✓
// Neither party's amounts visible to matcher
```

No `approve()` transaction. No front-running window. No mempool-visible order size. Token type identity is cryptographically derived from the issuing contract's address — no global registry needed, no token address collisions possible.

> **Zswap implementation note:** The official docs flag this layer as not yet performance-optimized and subject to further revision. The conceptual model is stable; verify specific API surface and proof costs against current docs before shipping production integrations.

---

## 7. Transactions and the Two-Phase Execution Model

Solana transactions are a precisely engineered structure: an account list (with read/write/signer flags), a recent blockhash, and a set of instructions. The entire account list must be declared upfront so Sealevel can determine which transactions conflict. Fees are base fee (5,000 lamports/signature) plus optional priority fee (CU price × CU limit).

Midnight transactions are structured around UTXOs and proofs, not account lists.

### What a Midnight Transaction Contains

- One or more circuit invocations (the contract calls)
- ZK proofs for each circuit invocation
- UTXO inputs being consumed (shielded or unshielded)
- UTXO outputs being created (payments, change, new token UTXOs)
- Nullifiers for spent UTXOs (prevent double-spend)
- DUST fee payment

### No Account List Declaration

One of the most significant ergonomic differences: **you do not declare an account list in Midnight transactions.** On Solana, every account your instruction will touch must be listed upfront — this is how Sealevel determines which transactions can run in parallel. On Midnight, UTXO ownership is embedded in the ZK proof itself; the runtime doesn't need a pre-declared conflict graph.

The practical payoff: no more tracking down which accounts to pass, no "account X missing from instruction" errors, no `ComputeBudget` instructions. The circuit either proves it has the right to spend the inputs, or the proof fails.

Solana devs know the pain of "Transaction too large" when packing too many accounts into the 1,232-byte payload limit. Midnight transactions don't have this shape. The constraint is ZK proof size and proving time, not account list byte length.

### The Two-Phase Execution Model

**This is the biggest transaction-semantic difference from both Solana and Ethereum, and it's absent from most Midnight documentation — read this carefully.**

Solana transactions are all-or-nothing: they fully succeed or the entire transaction reverts (though fees are still charged on failure). Midnight splits execution into three stages:

**Stage 1 — Well-formedness check:** Runs against no state. Verifies all ZK proofs, checks that offers are balanced (accounting for fees and mints). A bad proof or unbalanced offer → rejected outright, not included in the ledger.

**Stage 2 — Guaranteed phase:** Runs against ledger state. **DUST fees are collected here.** If this phase fails, the transaction is *not included in the ledger at all* — similar to Solana's pre-execution checks.

**Stage 3 — Fallible phase:** Also runs against ledger state. If this phase fails, the effects of the guaranteed phase **still apply**, and the transaction is recorded as a **partial success**. Fees are forfeited even if the fallible phase fails.

**Why this matters for Solana devs:** Unlike Solana where a failed transaction always reverts cleanly (minus the base fee), a Midnight transaction can land in a partially-applied state. Design your circuits so that a partial success is always a safe state to be in. Put DUST payment and must-succeed invariants in the guaranteed phase. Put speculative or best-effort effects in the fallible phase — and never put a step in the fallible phase that a guaranteed-phase commitment depends on.

**Fees on failed transactions:** The guaranteed phase always collects DUST. If the transaction fails in the guaranteed phase, it's not included and no DUST is spent. If it fails in the fallible phase, the guaranteed-phase DUST is forfeited. The "TBD" on failed transaction fees is answered by the two-phase model.

### The Fee Model: DUST vs. Compute Units

| Aspect | Solana | Midnight |
|---|---|---|
| Fee currency | SOL (lamports) | DUST (generated by holding NIGHT) |
| Base fee | 5,000 lamports per signature | DUST consumed per transaction |
| Fee market | Priority fee = CU price × CU limit | Dynamic based on network usage — no competitive mempool |
| Fee predictability | Volatile under congestion (hot accounts) | Predictable — DUST not traded, generated by NIGHT |
| Account rent | Rent-exempt deposit required (refundable) | No rent concept |
| Failed tx fee | Base fee charged even on failure | Guaranteed-phase DUST forfeited on fallible failure |
| Fee optimization | Simulate CUs, set tight CU limit | Keep circuits lean to reduce proof time |

---

## 8. Toolchain and Development Environment

If you've used Anchor's CLI, you'll find Midnight's toolchain conceptually familiar — even though the tools themselves are different. The big relief: you're writing TypeScript, not Rust. The main adjustment: there's a ZK compilation step with no equivalent in your current stack, plus a separate proof server process.

### Tool Mapping: Solana → Midnight

| Solana Tool | Midnight Equivalent | Notes |
|---|---|---|
| Rust + Anchor | Compact (`compactc`) | TypeScript-like DSL compiled to ZK circuits |
| `@solana/kit` / web3.js | Midnight.js SDK | TypeScript SDK for dApp + transaction building |
| Anchor IDL | Compiler-generated TypeScript stubs | Same idea — typed interfaces to your contract |
| Anchor test framework | Compact test suite + DevNet | Tests run against a local DevNet node |
| `solana-test-validator` | Midnight DevNet | Local chain for development |
| Solana CLI | `compactc` CLI | Compile, manage contracts |
| Phantom / Backpack | Lace wallet (Midnight-enabled) | Manages NIGHT, DUST, shielded UTXOs |
| Helius / QuickNode RPC | Midnight node RPC | Standard JSON-RPC interface |
| Anchor PDA utilities | No direct equivalent | PDAs don't exist; contract addresses derived differently |
| *(no equivalent)* | Proof server daemon | Separate process for client-side ZK proof generation |
| OpenZeppelin Solidity | `@openzeppelin/compact-contracts` | Available, actively maintained, flagged experimental |

### Project Structure

```
my-midnight-dapp/
├── contracts/
│   ├── my_contract.compact          # Compact contract source
│   └── managed/                     # Compiler output — do not edit
│       └── my_contract/
│           ├── contract/index.cjs   # Generated TypeScript stubs (your typed ABI)
│           ├── keys/                # ZK proving/verifying keys
│           └── zkir/                # ZK intermediate representation
└── src/
    ├── index.ts                     # dApp logic using Midnight.js SDK
    └── witnesses.ts                 # Implement witness functions here
                                     # (private key access, off-chain data, etc.)
```

### Development Workflow

1. **Write contract:** Author your `.compact` file with ledger state, witness interfaces, and exported circuits.
2. **Compile:** Run `compactc` to generate ZK circuits and TypeScript stubs. Fix compile errors — the compiler is strict about `disclose()` usage and circuit bounds.
3. **Start proof server:** Run the proof server daemon. Your dApp's SDK calls route through it for proof generation.
4. **Implement witnesses:** The compiler generates witness interfaces. You implement the bodies in TypeScript — this is where private key handling and off-chain data access lives.
5. **Test locally:** Run against a local DevNet. Test happy paths and proof failures (failed asserts).
6. **Deploy:** Use the Midnight.js SDK to deploy. The deployer's wallet pays DUST for the deployment transaction.
7. **Build dApp:** Use compiler-generated TypeScript stubs as typed interfaces to your circuits.

---

## 9. Security: What Changes, What Disappears, What's New

Solana has a well-documented set of security pitfalls: missing signer checks, missing ownership checks, arithmetic overflow, account data length mismatches, CPI privilege escalation, PDA substitution attacks. Midnight eliminates some of these structurally; others require new patterns.

### Solana Vulnerabilities That Don't Exist on Midnight

- **Missing signer checks:** Replaced by ZK proof of key knowledge. You can't fake ownership — the math won't verify.
- **Account ownership checks:** UTXO ownership is cryptographic, not a field you can forget to check.
- **CPI privilege escalation:** No CPI model in Compact 1.0. No cross-contract attack surface.
- **Reentrancy via CPI:** Circuits are bounded computations with no external calls mid-execution. No reentrant path exists.
- **Account data length mismatches:** No manual buffer management; ledger fields are typed, not raw byte arrays.
- **PDA substitution attacks:** No PDA model; contract addressing works differently.
- **Front-running via mempool:** Shielded transactions don't expose amounts or intent pre-broadcast.

### Security Patterns That Still Apply

- **`assert()` for invariants:** Same concept as Anchor's `require!()`. Proof fails if violated — equivalent to a transaction abort.
- **Arithmetic overflow:** Midnight uses fixed-size integer types (`Uint<64>`, `Uint<128>`). Overflow behavior is defined at the type level. Audit your arithmetic.
- **Access control completeness:** Every exported circuit that should be restricted needs its own `assert(persistentHash(witness()) == owner)`. No global authorization — check it everywhere it's needed.
- **Round counters:** Always include a `round: Counter` field in ledger state and increment it in every circuit that modifies state. Without it, transaction correlation analysis becomes easier. The Compact standard library's `Counter` type makes this one line.

### New Attack Surfaces

**Disclosure mistakes** are the most common Compact bug class. Using `disclose()` on data you didn't mean to make public, or storing private-derived values in ledger state without realizing what you've disclosed. The compiler catches *accidental* leakage of witness data, but can't catch *intentional* disclosure you didn't think through. Audit every `disclose()` call.

**Commitment brute-forcing:** If you commit to a small-domain value (a boolean, a small integer, a role enum), an adversary can enumerate the domain and reverse your commitment. Use `persistentCommit()` with a random salt whenever the domain is small.

**Witness implementation bugs:** The Compact compiler validates the contract logic, not your TypeScript witness implementations. A buggy witness that leaks private data or produces incorrect values won't be caught at compile time. Treat witness code as security-critical. The OZ Compact library explicitly ships test witnesses only — not production witnesses — because of this. If you consume it, you must author and audit your own witnesses.

**Untrusted witness values:** Witness implementations are provided by the executing user and run off-chain. Contract logic must never trust a witness value without validating it in-circuit. Treat every witness result as untrusted user input — because that's exactly what it is.

**Partial-success states:** Given the two-phase execution model (§7), a fallible-phase failure can leave your contract in a partially-applied state. Design circuit boundaries so partial success is always safe. Don't leave invariants dependent on fallible-phase steps that a guaranteed-phase commitment already assumed would complete.

**UTXO contention as a liveness attack:** An adversary who knows which UTXOs a victim wallet holds could front-run UTXO consumption with conflicting transactions, forcing repeated failures. Mitigation: maintain many small UTXOs, and consider building retry logic into latency-tolerant flows.

---

## 10. Quick Reference: Anchor/Rust → Compact Cheat Sheet

| Anchor / Solana | Compact / Midnight | Notes |
|---|---|---|
| `declare_id!("...");` | Derived at deployment | No hardcoded program ID |
| `#[program] mod` | File IS the contract | One contract per `.compact` file |
| `#[account] MyState` | `export ledger field: Type;` | Ledger = on-chain state |
| PDA with seeds + bump | Contract address derivation | Different model; no seeds/bump concept |
| `#[derive(Accounts)]` | No equivalent | UTXO ownership proved via ZK proof |
| `ctx.accounts.state.field` | `fieldName` directly | Direct ledger access in circuit body |
| `pub fn my_ix(ctx)` | `export circuit myCircuit()` | `export` = callable in transaction |
| `Signer<'info>` | `assert(persistentHash(witness()) == owner)` | Prove key knowledge instead |
| `require!(cond, err)` | `assert(cond, "message")` | Proof fails on violation |
| `msg.sender` / signer | No equivalent | Use `persistentHash(getSpendingKey())` pattern |
| SPL Token transfer | Consume UTXO, create UTXO | UTXO model replaces token accounts |
| ATA (Associated Token Account) | No equivalent | UTXOs self-describe ownership |
| `invoke()` / CPI | **Not implemented in Compact 1.0** | `contract` keyword reserved, not functional |
| `invoke_signed()` / PDA signer | No equivalent | Contracts sign differently |
| lamports / SOL fee | DUST fee | Generated by holding NIGHT |
| `ComputeBudget` instruction | No equivalent | Circuit bounds are compile-time fixed |
| Compute unit limit | Circuit constraint budget | Fixed at compile time, not runtime |
| Rent-exempt deposit | No rent concept | State lives on-chain automatically |
| `account.data: Vec<u8>` | Typed ledger fields | No raw byte buffers |
| Borsh serialize / deserialize | Not needed | Compact types serialize automatically |
| `keccak256(data)` | `transientHash(value)` or `persistentHash(value)` | ZK-optimized hash functions |
| Commitment with salt | `transientCommit(v, salt)` / `persistentCommit(v, rand)` | Use persistent* for ledger state |
| `#[tokio::test]` Anchor test | Compact test suite | Against local DevNet |
| Proof of history / fast finality | PoS via Cardano SPOs | ~4-6s block time; proving time dominates latency |

---

## 11. Suggested Learning Path

1. **Install the toolchain** → start the proof server → clone `example-hello-world` → deploy to local DevNet. Get the compile → proving-keys → TypeScript-driver loop in your fingers before writing real logic. ([docs.midnight.network/getting-started](https://docs.midnight.network/getting-started))
2. **Work the bulletin-board tutorial.** It introduces witnesses, commitments, `disclose()`, the round-counter replay pattern, and the compile → prove → submit flow in a realistic contract.
3. **Read the Compact language reference and standard library** ([docs.midnight.network/compact](https://docs.midnight.network/compact)). Pay attention to ledger types: `Counter`, `Map`, `MerkleTree`, `Maybe`.
4. **Read the "How Midnight Works" concept pages** — transaction semantics (especially guaranteed/fallible phases), Zswap, DUST architecture, Kachina. Protocol-level reasoning lives here.
5. **Build something with a native token** to feel the Zswap UTXO model, then add selective disclosure to feel the compliance angle.
6. **Explore `@openzeppelin/compact-contracts`** for `Ownable`, `Pausable`, and `FungibleToken` patterns — noting it's actively maintained but flagged experimental. Author your own witnesses; don't reuse the test witnesses.

---

## Resources

| Resource | URL |
|---|---|
| Official Docs | https://docs.midnight.network |
| Compact Language Reference | https://docs.midnight.network/compact |
| Smart Contract Security | https://docs.midnight.network/compact/smart-contract-security |
| Transaction Semantics | https://docs.midnight.network/concepts/how-midnight-works/semantics |
| Zswap | https://docs.midnight.network/concepts/how-midnight-works/zswap |
| DUST Architecture | https://docs.midnight.network/concepts/dust-architecture |
| UTXO Model | https://docs.midnight.network/concepts/utxo |
| OpenZeppelin for Compact | https://docs.openzeppelin.com/contracts-compact |
| Midnight Discord | https://discord.com/invite/midnightnetwork |
| Midnight MCP (AI-assisted dev) | https://docs.midnight.network/blog/midnight-mcp-ai-assisted-development |

---

*Sourced from official Midnight documentation and the Midnight tokenomics whitepaper. Current as of June 2026. Compact 1.0 does not yet support cross-contract calls. Zswap/coin mechanics are flagged by the official docs as not yet performance-optimized — verify against current docs before shipping production integrations.*
