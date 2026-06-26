# Midnight for AI Builders — A Practical Integration Guide

*You build systems that reason, decide, and act. Midnight gives those systems cryptographic proof that they did so correctly — without exposing what they know.*

**Midnight Network Foundation | June 2026**

> **Version note:** Midnight mainnet launched March 31, 2026 in a federated phase. This guide assumes familiarity with AI systems, agent frameworks, and TypeScript/Python. It does not assume blockchain experience. Compact examples target `language_version >= 0.23`. Always verify against [docs.midnight.network](https://docs.midnight.network). For known compiler quirks affecting the code examples, see the [Appendix](#appendix-compact-syntax-verification-notes-june-2026).

---

## 1. Why This Guide Exists

AI systems in 2026 face a trust problem that getting better at AI doesn't solve. An agent can reason correctly, produce accurate outputs, follow its policy, and use only authorized data — and there is still no way for anyone outside the system to verify any of that without seeing everything inside it. The choice has been binary: full transparency (expose your model, your data, your prompts) or trust me (institutional assurance, audit logs, regulatory compliance theater).

Zero-knowledge proofs break that binary. They let a system prove specific facts about its behavior — "this decision used only authorized data," "this agent holds a valid credential," "this output came from a model with these properties" — without revealing anything else.

Midnight is a blockchain built around this capability. It's not the right tool for every AI problem. But for a specific and growing set of problems — verifiable decisions, private data provenance, agent-to-agent credentialing, compliance without disclosure — it's purpose-built infrastructure that nothing else currently provides.

**What this guide is:** A practical map of where Midnight fits in an AI architecture, with integration patterns and code. TypeScript-centric because that's the Midnight SDK layer and increasingly the glue layer for agent systems.

**What this guide is not:** A Compact language tutorial (the existing guides cover that), a blockchain primer (same), or a pitch for putting your LLM inference on-chain (don't — see §2).

### Which pattern should I read?

| If you need to… | Read |
|---|---|
| Prove an AI decision followed a declared policy without exposing inputs | [Pattern 1 — Verifiable Decisions](#pattern-1-verifiable-decisions) |
| Prove a dataset meets compliance standards without exposing the data | [Pattern 2 — Private Data Provenance](#pattern-2-private-data-provenance) |
| Create an auditable compliance record without putting sensitive data on-chain | [Pattern 3 — Compliance Without Full Disclosure](#pattern-3-compliance-without-full-disclosure) |
| Let AI agents verify each other's credentials across trust domains | [Pattern 4 — Agent-to-Agent Credentialing](#pattern-4-agent-to-agent-credentialing) |

---

## 2. The Honest Framing: What ZK Proofs Can and Cannot Do for AI

Before the use cases, the most important thing to understand clearly.

### What ZK proofs prove

A ZK proof certifies that a **specific computation ran correctly on specific inputs**. In Midnight's model, a circuit encodes the computation; a proof certifies the circuit ran as written with the given (possibly private) inputs; the chain verifies the proof.

### What ZK proofs do not prove

ZK proofs say nothing about whether the output is *correct in the world*. An LLM can produce a confident, internally consistent hallucination and the ZK proof verifies perfectly — because the model *did* run correctly. It just produced a wrong answer. **Cryptographic correctness is not semantic correctness.**

This distinction matters because the most intuitive AI use case — "prove my LLM gave the right answer" — is not what ZK proofs do. Don't build toward that.

### The latency reality for full inference proofs

Proving an entire LLM inference computation in ZK is an active research area and not yet practical for interactive use. As of mid-2026: zkLLM-style end-to-end proofs are thousands of times slower than native inference; even optimized zkML systems like Lagrange's DeepProve achieve practical proving times only for models up to roughly 100M parameters with sub-minute latency. For real-time agent interactions, full inference proofs are not viable tooling yet.

### Where ZK proofs *do* fit in AI systems

The practical fit is not proving the inference — it's proving the **surrounding structure**:

- That a decision was made according to a declared policy
- That a data source meets a compliance standard without exposing the data
- That an agent holds a valid credential without revealing credential details
- That an output was produced under specific constraints
- That a model with certain properties (not necessarily specific weights) was used

These are bounded, structured computations — the kind Midnight circuits handle well. The AI model does the hard reasoning work; Midnight proves facts about the inputs, outputs, and conditions surrounding that work.

### How Midnight fits into your stack

Midnight doesn't replace your AI infrastructure. It adds a cryptographic trust layer around specific operations:

```
┌─────────────────────────────────────────────────────┐
│                  Your AI System                     │
│                                                     │
│  Models · Agents · Pipelines · Data · APIs          │
│                                                     │
│  (Python, any ML framework, any agent SDK)          │
└──────────────────────────┬──────────────────────────┘
                           │ outputs: decisions, features,
                           │ compliance results, credentials
                           ▼
┌─────────────────────────────────────────────────────┐
│              Witness Layer (TypeScript)              │
│                                                     │
│  Translates AI outputs → private circuit inputs     │
│  Runs locally — never submitted to chain            │
│  Implemented in witnesses.ts                        │
└──────────────────────────┬──────────────────────────┘
                           │ private inputs into circuits
                           ▼
┌─────────────────────────────────────────────────────┐
│              Compact Circuits + Proof Server         │
│                                                     │
│  Encodes the policy / compliance check / credential │
│  Proof server runs locally as a separate process    │
│  Proof generation: seconds for typical circuits     │
└──────────────────────────┬──────────────────────────┘
                           │ ZK proof + public outputs
                           ▼
┌─────────────────────────────────────────────────────┐
│                  Midnight Chain                      │
│                                                     │
│  Verifies proof · Updates ledger state              │
│  Public record: compliance facts, credentials,      │
│  decisions — not the underlying data                │
└─────────────────────────────────────────────────────┘
```

The witness layer is your primary integration surface — where your ML pipeline outputs become Midnight circuit inputs. More on this in §4.

---

## 3. The Four Integration Patterns

### Pattern 1: Verifiable Decisions

**The problem:** Your AI system makes consequential decisions — loan approvals, content moderation, medical triage flags, access grants. Regulators, auditors, or users may need to verify that the decision followed the declared policy. But the policy, the input data, or the model itself may be proprietary or privacy-sensitive.

**What Midnight enables:** Prove that a decision followed a declared policy, given inputs that meet declared criteria — without exposing the inputs, the policy details, or the model.

**Architecture:**

```
AI Model (Python/any)
    │
    ├─ Produces: decision + supporting features
    │
    └─► Midnight witness layer (TypeScript)
            │ features, model output → private inputs
            └─► Compact circuit
                    │ encodes decision policy as assertions
                    └─► ZK proof on-chain: "decision followed policy"
```

**Compact circuit (credit decisioning example):**

```compact
pragma language_version >= 0.23;
import CompactStandardLibrary;

// On-chain: only the decision outcome and policy version are public
export ledger sealed policyVersion: Uint<32>;  // verify sealed syntax — see appendix
export ledger decisionCount: Counter;
export ledger round: Counter;
export ledger operatorAuthority: Bytes<32>;

// Witnesses: private inputs from your AI system — never on-chain
witness applicantScore(): Uint<64>;       // model output
witness debtToIncomeRatio(): Uint<64>;    // input feature (×100 for integer math)
witness getAuthorityKey(): Bytes<32>;     // operator authorization

circuit operatorKey(sk: Bytes<32>): Bytes<32> {
    return persistentHash<Vector<3, Bytes<32>>>(
        [pad(32, "creditdecision:operator:"), round as Bytes<32>, sk]
    );
}

export circuit recordDecision(approved: Boolean): [] {
    // Verify operator is authorized to submit decisions
    assert(operatorKey(getAuthorityKey()) == operatorAuthority, "Not authorized");

    // Enforce policy constraints — these are what the proof certifies
    const score = applicantScore();
    const dti = debtToIncomeRatio();

    if approved {
        // Prove: an approved decision met minimum thresholds
        assert(score >= 650, "Score below approval threshold");
        assert(dti <= 43_00, "DTI above approval threshold");  // 43.00%
    } else {
        // Prove: a denial had at least one disqualifying factor
        assert(score < 650 || dti > 43_00, "Denial policy not met");
    }

    // Only the outcome and policy version hit the chain
    // Score, DTI, and applicant identity stay private
    decisionCount.increment(1);
    round.increment(1);
}
```

**TypeScript witness implementation:**

```typescript
import { ModelOutput, ApplicantFeatures } from './your-ml-pipeline';

export function buildDecisionWitnesses(
  output: ModelOutput,
  features: ApplicantFeatures,
  operatorKey: Uint8Array
) {
  return {
    // These values are consumed by the circuit but never submitted to the chain
    applicantScore:      () => BigInt(output.creditScore),
    debtToIncomeRatio:   () => BigInt(features.dtiScaled),  // pre-rounded on Python side
    getAuthorityKey:     () => operatorKey,
  };
}
```

**What the proof certifies:** A specific policy version was applied. An approved decision met the declared thresholds. A denied decision had a qualifying disqualifying factor. Nothing about the specific applicant, their data, or the model internals is disclosed.

---

### Pattern 2: Private Data Provenance

**The problem:** Your AI system was trained on or processes sensitive data. You need to prove to a regulator, customer, or auditor that the data meets certain standards — consent obtained, PII excluded, geographic restrictions honored — without handing over the data itself.

**What Midnight enables:** Prove compliance properties of a dataset using the oracle-via-witness pattern. An authorized data auditor signs attestations off-chain; your circuit verifies the signature and records the compliance fact on-chain. The dataset never touches the chain.

**Architecture:**

```
Data Auditor / Certifier (off-chain)
    │
    ├─ Audits dataset against compliance criteria
    ├─ Signs attestation: hash(datasetId, criteria, result)
    └─► Publishes: attestation + signature (off-chain endpoint)

AI System (your code)
    │
    ├─ Fetches attestation + signature via witness
    └─► Compact circuit
            │ verifies signature against auditor's public key
            └─► ZK proof on-chain: "dataset meets [criteria]"
```

**Compact circuit:**

```compact
pragma language_version >= 0.23;
import CompactStandardLibrary;

// Auditor's public key locked at deployment — immutable
export ledger sealed auditorPublicKey: Bytes<32>;  // verify sealed syntax — see appendix
export ledger round: Counter;

// The compliance result is public; the dataset contents are not
export ledger datasetId: Bytes<32>;
export ledger complianceStatus: Boolean;
export ledger criteriaVersion: Uint<32>;

// Witnesses: fetched from the auditor's off-chain endpoint
witness getDatasetId(): Bytes<32>;
witness getAttestation(): Bytes<64>;   // auditor's signed attestation
witness getSignature(): Bytes<64>;     // signature over attestation

export circuit recordCompliance(criteria: Uint<32>): [] {
    const id  = getDatasetId();
    const att = getAttestation();
    const sig = getSignature();

    // Verify the auditor signed this attestation
    assert(verifySignature(auditorPublicKey, att, sig), "Invalid auditor signature");

    // Record compliance fact — dataset ID and result are public, contents are not
    datasetId        = disclose(id);
    complianceStatus = disclose(true);
    criteriaVersion  = disclose(criteria);
    round.increment(1);
}
```

**What the proof certifies:** A specific dataset (identified by hash) was attested as compliant with a specific criteria version by a specific authorized auditor. The dataset contents, the audit details, and the attestation details are not on-chain.

**Practical note:** This pattern requires an off-chain auditor/certifier service that signs attestations. That's a real infrastructure component — but it's the same trust model as any compliance certification, except the verification is now cryptographic rather than institutional.

---

### Pattern 3: Compliance Without Full Disclosure

**The problem:** Your AI system processes regulated data — health records, financial data, personal information. Regulators require audit trails. But the audit trail itself may contain information that can't be shared broadly. You need to prove compliance without creating a second privacy problem in the audit record itself.

**What Midnight enables:** Prove that data meets regulatory criteria as it is processed — without recording the data on-chain. The compliance fact is public; the underlying record stays entirely private.

```compact
pragma language_version >= 0.23;
import CompactStandardLibrary;

// HIPAA / healthcare audit example
export ledger sealed regulationVersion: Uint<32>;  // verify sealed syntax — see appendix
export ledger auditCount: Counter;
export ledger round: Counter;
export ledger operatorAuthority: Bytes<32>;

// Witnesses: patient data processed by the AI system — never on-chain
witness getPatientAge(): Uint<64>;
witness getDataConsentFlag(): Boolean;
witness getDataResidencyCode(): Uint<32>;    // jurisdiction code
witness getProcessingPurposeCode(): Uint<32>;
witness getOperatorKey(): Bytes<32>;

circuit operatorKey(sk: Bytes<32>): Bytes<32> {
    return persistentHash<Vector<3, Bytes<32>>>(
        [pad(32, "hipaa:operator:"), round as Bytes<32>, sk]
    );
}

export circuit recordCompliantProcessing(recordId: Bytes<32>): [] {
    assert(
        operatorKey(getOperatorKey()) == operatorAuthority,
        "Not authorized"
    );

    // Assert compliance conditions — these are what the proof verifies
    assert(getDataConsentFlag(),           "No consent on record");
    assert(getDataResidencyCode() == 840,  "Data not US-resident");  // 840 = US
    assert(getProcessingPurposeCode() == 1, "Invalid processing purpose");

    // Pediatric check: flag if minor (separate handling required)
    const age = getPatientAge();
    assert(age >= 18, "Minor patient — requires additional authorization");

    // On-chain record: a specific record ID was processed compliantly
    // No age, no consent details, no patient identity
    auditCount.increment(1);
    round.increment(1);
}
```

**What the proof certifies:** A specific record (identified by opaque ID) was processed by an authorized operator in compliance with the declared regulation version. No patient data of any kind is on-chain.

---

### Pattern 4: Agent-to-Agent Credentialing

**The problem:** Multi-agent systems need agents to verify each other's capabilities and permissions before cooperating — especially across organizational boundaries. API keys are opaque. OAuth tokens require a central authority. You need agents to prove specific properties about themselves without a central trust broker at runtime.

**What Midnight enables:** Agents hold cryptographic credentials stored as commitments on-chain. An agent proves it holds a valid credential for a specific capability without revealing its private key or credential details. No central authority needed after initial issuance.

This is the pattern the AI credentialing research community (AIP, Indicio ProvenAI, W3C DID-based agent identity) is converging on — Midnight provides the ZK-native implementation.

**Architecture:**

```
Credential Issuer (your org or a trust authority)
    │
    ├─ Issues capability credential to Agent A
    └─► Registers credential commitment on Midnight

Agent A (wants to call Agent B's service)
    │
    ├─ Holds credential + private key locally
    └─► Builds ZK proof: "I hold a valid credential for [capability]"
            │
            └─► Agent B verifies proof on-chain before responding
```

**Compact circuit (capability credential verification):**

```compact
pragma language_version >= 0.23;
import CompactStandardLibrary;

// Registry of issued credential commitments
// Note: Map key type constraints — verify Bytes<32> is a valid key type (see appendix)
export ledger credentialRegistry: Map<Bytes<32>, Bytes<32>>;
export ledger sealed issuerPublicKey: Bytes<32>;  // verify sealed syntax — see appendix
export ledger round: Counter;
export ledger verifiedCount: Counter;

witness getAgentSecretKey(): Bytes<32>;
witness getCredentialId(): Bytes<32>;
witness getCredentialData(): Bytes<64>;

circuit credentialCommitment(sk: Bytes<32>, data: Bytes<64>): Bytes<32> {
    return persistentHash<Vector<3, Bytes<32>>>(
        [pad(32, "agent:credential:"), sk, data]
    );
}

// Called by the issuer to register a new credential
export circuit issueCredential(
    credentialId: Bytes<32>,
    agentCommitment: Bytes<32>
): [] {
    credentialRegistry.insert(credentialId, agentCommitment);
    round.increment(1);
}

// Called by an agent to prove it holds a valid credential
// Note: verify .lookup() method name against current stdlib (see appendix)
export circuit proveCapability(requiredCredentialId: Bytes<32>): [] {
    const sk   = getAgentSecretKey();
    const id   = getCredentialId();
    const data = getCredentialData();

    assert(id == requiredCredentialId, "Wrong credential");

    const computed = credentialCommitment(sk, data);
    assert(
        credentialRegistry.lookup(id) == computed,
        "Credential not valid"
    );

    // Proof certifies: "this agent holds a valid credential for [id]"
    // The agent's secret key and credential data never go on-chain
    verifiedCount.increment(1);
    round.increment(1);
}
```

**TypeScript integration for an agent using any framework:**

```typescript
import { MidnightClient } from '@midnight-ntwrk/midnight-js';

class CredentialedAgent {
  private midnight: MidnightClient;
  private secretKey: Uint8Array;
  private credentialData: Uint8Array;
  private credentialId: Uint8Array;

  async proveCapabilityTo(
    targetAgent: string,
    requiredCapability: Uint8Array
  ): Promise<string> {
    const witnesses = {
      getAgentSecretKey:  () => this.secretKey,
      getCredentialId:    () => this.credentialId,
      getCredentialData:  () => this.credentialData,
    };

    // ZK proof generated locally, submitted on-chain
    const proof = await this.midnight.buildTransaction(
      credentialContract.proveCapability(requiredCapability, { witnesses })
    );

    await this.midnight.submitTransaction(proof);

    // Target agent verifies on-chain using transaction ID
    return proof.transactionId;
  }
}
```

**What the proof certifies:** A specific agent holds a valid credential for a specific capability, as registered by the issuer. The agent's secret key, full credential data, and which agent is proving are not disclosed.

---

## 4. Integration Architecture: The Witness Layer in Depth

The witness layer is where most integration friction lives. Your ML pipeline speaks floats, dicts, and numpy arrays. Compact circuits speak `Uint<64>`, `Bytes<32>`, and `Boolean`. The witness layer is where you cross that boundary — and where silent correctness bugs are most likely to hide.

### Type mapping

| Python/ML type | Witness TypeScript type | Notes |
|---|---|---|
| `float` (0.0–1.0) | `BigInt` (integer × scale factor) | Fix precision on Python side before serializing — see below |
| `int` | `BigInt` | Straightforward; watch overflow for values above 2^64 |
| `bool` | `boolean` | Direct |
| `str` (UUID, record ID) | `Uint8Array` (32 bytes) | Hash to fixed size if variable length |
| `np.ndarray` | Not directly usable | Extract scalar features individually |
| `None` / `Optional[T]` | Handle before witness layer | Circuits have no null; assert presence before passing |

### The float precision trap

This is the most common silent correctness bug in AI integrations. If your Python model outputs `0.4299999999999999` (a common IEEE 754 artifact) and you convert it in TypeScript with `Math.round(0.4299999... * 100)`, you get `42` — and a compliant applicant fails a circuit assertion with no proof error, just a silently wrong value.

Fix: round and convert to integer on the Python side using `decimal` for controlled precision, before serialization:

```python
# Python — serialize before sending to the witness layer
import decimal

def serialize_ratio(value: float, scale: int = 100) -> int:
    """Convert a float ratio to scaled integer with controlled precision.
    0.43 -> 43, 0.4350 -> 44 (rounds half-up), not 0.4299999... -> 42
    """
    return int(
        decimal.Decimal(str(value))
        .quantize(decimal.Decimal('0.01'), rounding=decimal.ROUND_HALF_UP)
        * scale
    )
```

```typescript
// TypeScript witness — receives already-precise integer, no float math here
debtToIncomeRatio: () => BigInt(serializedFeatures.dtiScaled),
```

### Handling optionals

Compact ledger fields are always initialized — there is no null. If your ML pipeline produces optional outputs, handle the absent case before it reaches the witness layer. Passing `undefined` or `null` into a `BigInt()` constructor throws at runtime, not as a graceful proof error:

```typescript
function buildWitnesses(output: ModelOutput): WitnessMap {
  if (output.creditScore == null) {
    throw new Error('Score absent — route to manual review before proof generation');
  }
  if (output.dtiScaled == null) {
    throw new Error('DTI feature missing — cannot build proof');
  }
  return {
    applicantScore:    () => BigInt(output.creditScore),
    debtToIncomeRatio: () => BigInt(output.dtiScaled),
    getAuthorityKey:   () => operatorKey,
  };
}
```

### Encoding string IDs as `Bytes<32>`

Your system likely uses UUID or string record identifiers. Hash them to fixed-size `Uint8Array` for use as `Bytes<32>` witness inputs:

```typescript
import { createHash } from 'crypto';

function idToBytes32(id: string): Uint8Array {
  return new Uint8Array(createHash('sha256').update(id).digest());
}

// Usage in witness
recordId: () => idToBytes32(decision.applicantId),
```

### Keep witnesses thin

The witness layer should translate and validate, not contain business logic. If you find yourself writing conditional logic that determines *what* value to pass based on model outputs, that logic belongs in your ML pipeline (Python side) or in the circuit assertions (Compact side). Witnesses that contain policy logic are harder to audit and easier to get wrong.

### The proof server

Unlike an RPC endpoint, Midnight requires a running proof server daemon alongside your application. In a containerized deployment, treat it as a sidecar. Proof generation for typical policy-checking circuits takes low single-digit seconds — fine for batch decisions and audit records, not suitable for synchronous inference-time operations where users expect sub-100ms response. Design accordingly: generate proofs asynchronously, or buffer decisions and prove in batch.

### DUST for fees

Your integration service needs a NIGHT-holding wallet to generate DUST for transaction fees. DUST generation is proportional to NIGHT holdings — plan your NIGHT allocation based on expected transaction volume. You can delegate DUST to cover fees for agents or users who don't hold NIGHT.

---

## 5. Honest Assessment: When Midnight Is and Isn't the Right Tool

### Good fit

- **Regulated AI decisions** requiring auditable compliance without exposing sensitive inputs (lending, healthcare, content moderation with legal liability)
- **Multi-organization AI pipelines** where parties need to verify each other's data quality or model properties without sharing proprietary details
- **Agent credentialing** in multi-agent systems that span trust domains or organizations
- **Data marketplace provenance** — proving training data meets quality or consent standards before purchase
- **Compliance-as-proof** for EU AI Act, HIPAA, or similar frameworks requiring auditable records of AI system behavior

### Not a good fit (right now)

- **Real-time inference verification** — full LLM inference proofs are minutes per query; not interactive
- **Sub-second latency requirements** — proof generation takes seconds; budget for async flows
- **Semantic correctness proofs** — ZK proves computation ran correctly, not that the output is factually right. A ZK-verified hallucination is still a hallucination.
- **Simple audit logging** — if your requirements are met by a tamper-evident log (immutable database, signed audit trail), that's much simpler. Midnight's value is privacy-preserving verifiability, not just immutability.
- **Replacing your AI system's core reasoning** — Midnight is trust infrastructure for AI, not AI infrastructure itself

---

## 6. Getting Started

### Prerequisites

You need:
- Node.js 18+ and TypeScript
- A running Midnight DevNet or testnet node
- The proof server daemon (runs alongside your application)
- A Midnight wallet with NIGHT for DUST generation

You do not need:
- Prior blockchain experience
- Knowledge of cryptography beyond the conceptual level
- To learn Rust or any new ML tooling

### Step 1: Install the Midnight.js SDK

```bash
npm install @midnight-ntwrk/midnight-js
```

### Step 2: Write your Compact contract

Start with Pattern 1 (Verifiable Decisions) as the template. Define your policy as circuit assertions. Keep the circuit focused on one decision type per contract. Read the [Writing a Contract tutorial](https://docs.midnight.network/compact/writing) before your first circuit.

### Step 3: Implement your witness layer

The witnesses bridge your AI system and the circuit. They run locally. Serialize your ML outputs on the Python side first (see the float precision guidance in §4), then pass the clean integers and byte arrays through:

```typescript
// witnesses.ts
export function buildWitnesses(mlOutput: SerializedModelOutput) {
  return {
    applicantScore:    () => BigInt(mlOutput.score),
    debtToIncomeRatio: () => BigInt(mlOutput.dtiScaled),  // already rounded
    getAuthorityKey:   () => yourOperatorKey,
  };
}
```

### Step 4: Start the proof server and submit your first proof

```typescript
import { createMidnightClient } from '@midnight-ntwrk/midnight-js';

const client = await createMidnightClient({
  proofServerUrl: 'http://localhost:6300',  // proof server sidecar
  networkId: 'testnet',
});

const tx = await client.buildTransaction(
  decisionContract.recordDecision(approved, {
    witnesses: buildWitnesses(mlOutput)
  })
);

await client.submitTransaction(tx);
```

### Learning path

1. [Midnight Getting Started](https://docs.midnight.network/getting-started) — toolchain setup and your first contract
2. [Writing a Contract tutorial](https://docs.midnight.network/compact/writing) — the canonical Compact reference
3. [Bulletin Board tutorial](https://docs.midnight.network/tutorials/bboard) — the access control pattern used throughout this guide
4. [Smart Contract Security](https://docs.midnight.network/compact/smart-contract-security) — specifically the `ownPublicKey()` warning; affects every pattern in this guide
5. [DUST Architecture](https://docs.midnight.network/concepts/dust-architecture) — fee planning for production integrations

---

## 7. The Security Rules That Apply to AI Integrations

All the standard Midnight security patterns apply, with a few AI-specific considerations.

**Never use `ownPublicKey()` or naive public key comparisons for access control.** This applies whether the caller is a human user or an AI agent. In a Compact circuit, calling a stdlib function to retrieve the caller's public key compiles to an **unconstrained private input** — a value the prover can set freely to any value they choose, including the stored authority commitment itself, without knowing any secret. There is no cryptographic obligation. An attacker sets their "public key" to whatever is stored in the authority field, the comparison passes, and the check is bypassed with no valid secret required. Always use the `persistentHash` commitment scheme shown in §3: the contract stores a hash of the secret key, and the circuit proves the caller knows the preimage. The ZK proof makes this cryptographically binding — it cannot be faked.

**Validate all witness inputs in-circuit.** Your witness layer takes outputs from your ML pipeline — a Python process you don't fully control. Treat every witness value as untrusted. The circuit is responsible for asserting that witness values meet required constraints before acting on them. A compromised ML pipeline that feeds bad values into witnesses should fail circuit assertions, not silently produce a valid proof of incorrect behavior.

**Design for partial success.** The two-phase transaction model (guaranteed + fallible phases) means your AI audit record can be partially applied. In practice: put the DUST fee payment in the guaranteed phase, put the policy assertion in the fallible phase. A failed policy assertion (because the ML output changed between proof generation and submission) results in a failed fallible phase, not a false compliance record.

**Proofs certify computation, not correctness.** Repeat this to yourself before every design decision. A ZK proof that a credit decision followed your declared policy does not certify the policy is fair, the model is unbiased, or the outcome is correct. ZK handles the integrity layer; your AI system still owns the quality layer.

---

## Resources

| Resource | URL |
|---|---|
| Midnight Docs | https://docs.midnight.network |
| Writing a Contract | https://docs.midnight.network/compact/writing |
| Bulletin Board Tutorial | https://docs.midnight.network/tutorials/bboard |
| Smart Contract Security | https://docs.midnight.network/compact/smart-contract-security |
| DUST Architecture | https://docs.midnight.network/concepts/dust-architecture |
| Transaction Semantics | https://docs.midnight.network/concepts/how-midnight-works/semantics |
| Midnight.js SDK | https://docs.midnight.network/develop/midnight-js |
| Midnight MCP (AI-assisted dev) | https://docs.midnight.network/blog/midnight-mcp-ai-assisted-development |
| Discord | https://discord.com/invite/midnightnetwork |

---

## Appendix: Compact Syntax Verification Notes (June 2026)

The following items in the code examples above should be confirmed against the current compiler before production use. Run examples through `compactc` or the [Midnight MCP](https://docs.midnight.network/blog/midnight-mcp-ai-assisted-development) before publishing.

**`export ledger sealed` modifier.** The `sealed` keyword is used in several patterns to mark fields as immutable after constructor initialization. Verify that this modifier syntax is current in `language_version >= 0.23`. If not supported, the equivalent behavior may require a different pattern — consult the Compact language reference for immutable field semantics.

**`Map<Bytes<32>, Bytes<32>>` key types.** Compact's `Map` and collection types have constraints on what types can be used as keys. Non-scalar key types like `Bytes<32>` may not be supported. If `Bytes<32>` keys are invalid, an alternative is to use a hash of the key (already a scalar) or a different collection type. Verify against the current stdlib reference.

**`.lookup()` method name.** The credential registry pattern uses `credentialRegistry.lookup(id)` to retrieve a value. Confirm the method name is `.lookup()` and not `.member()`, `.get()`, or something else in the current `Map` stdlib type.

**`pad()` function signature.** Several circuits use `pad(32, "string-literal")` to create fixed-size byte arrays from domain separator strings. Verify the current function name, argument order, and whether string literal syntax is supported directly.

**`Counter` type with `.increment()`.** This appears consistently across all five migration guides in this series and is the item most likely to be correct. Still worth a quick compile check.

---

*Current as of June 2026. Full LLM inference proofs (zkLLM-style) are not yet practical for interactive use — proving times remain in the minutes-to-hours range for large models. The patterns in this guide target the surrounding infrastructure (decisions, credentials, provenance, compliance) where Midnight is deployable today. Compact 1.0 does not yet support cross-contract calls. Midnight mainnet is in a federated phase with decentralization in progress.*
