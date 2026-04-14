# ACA-Py Replacement Analysis

## Question

Is it feasible to implement a custom server that handles entity registration and credential issuance/management, replacing ACA-Py entirely?

---

## What ACA-Py Is (vs. VON Network)

VON Network and ACA-Py are two entirely separate systems:

```
VON Network (SSI_Project)        ACA-Py Agent
─────────────────────────        ────────────────────────────────────
server/anchor.py + indy_vdr      Wallet (Askar backend)
→ Indy ledger nodes 9701-9708    DIDComm over HTTP
→ Public read/write of:          Issue Credential protocol (RFC 0453)
   NYM (DIDs)                    Present Proof protocol (RFC 0454)
   SCHEMA                        CL signature operations
   CRED_DEF                      (via anoncreds-rs / indy-credx)
```

VON Network provides the **public ledger**. ACA-Py is the **identity agent**. They are not interchangeable. Replacing ACA-Py means replacing the agent layer, not the ledger.

---

## What ACA-Py Does: Reimplementability by Layer

### Layer 1 — Key generation and DID derivation
ACA-Py generates Ed25519 keypairs and derives DIDs using `pynacl` + `base58`:
```python
vk = bytes(nacl.signing.SigningKey(seed).verify_key)
did = base58.b58encode(vk[:16]).decode("ascii")
```
`anchor.py:nacl_seed_to_did()` **already does this** in the project.
**Verdict: Reimplementable. Already in codebase.**

### Layer 2 — NYM transactions (DID registration)
Uses `indy_vdr.ledger.build_nym_request()` + `nacl.signing.SigningKey` + `pool.submit_request()`.
`anchor.py:register_did()` **already does this**.
**Verdict: Reimplementable. Already in codebase.**

### Layer 3 — SCHEMA transactions
Same `indy_vdr` pattern. JSON payload is straightforward.
**Verdict: Reimplementable with `indy_vdr` directly.**

### Layer 4 — CRED_DEF transactions ← Hard boundary
A Credential Definition is not just a data record. It contains a **CL (Camenisch-Lysyanskayer) public key** generated from a master secret. Internally ACA-Py calls `anoncreds-rs` (a Rust library) to:
- Generate the CL keypair (`CredentialDefinition.create()`)
- Compute `p_prime`, `q_prime`, `n`, `s`, `z`, `r` for each attribute
- Store the private key in the wallet
- Write the public key to the ledger as a CRED_DEF transaction

This is approximately 800 lines of audited Rust cryptography that has no pure Python equivalent.
**Verdict: NOT reimplementable without `anoncreds-rs`.**

### Layer 5 — Credential issuance (CL signature)
Issuing a credential means signing attribute values using the CL private key with a blinded master secret from the holder:
- `anoncreds.Credential.create()` — issuer side
- `anoncreds.CredentialRequest.create()` — holder side
- A two-step cryptographic exchange (offer → request → issue)

This is the core of Issue Credential protocol RFC 0453.
**Verdict: NOT reimplementable without `anoncreds-rs`.**

### Layer 6 — DIDComm connection protocol
Connection establishment uses DIDComm envelope encryption: each message is packed with the sender's DID key and recipient's verkey using `libsodium`'s `box` (X25519 ECDH + XSalsa20-Poly1305).
**Verdict: Reimplementable but high effort. Requires implementing the DIDComm v1 packing spec.**

### Layer 7 — Present Proof (ZKP)
Zero-knowledge proof requests and responses are the most complex operation. A holder proves attributes from a credential without revealing the credential itself, using a ZKP derived from the CL signature. Requires `anoncreds-rs` on both verifier and prover sides.
**Verdict: NOT reimplementable without `anoncreds-rs`.**

---

## Summary Table

| Responsibility | Reimplementable | Notes |
|---|---|---|
| Key generation / DID derivation | Yes | `pynacl` + `base58` — already in project |
| NYM transaction (DID to ledger) | Yes | `indy_vdr` — already in project |
| SCHEMA transaction | Yes | `indy_vdr` directly |
| CRED_DEF transaction | No | Requires `anoncreds-rs` (Rust) |
| Credential issuance (CL sign) | No | Requires `anoncreds-rs` |
| DIDComm connection protocol | Partial | High effort, DIDComm v1 spec |
| Present Proof (ZKP) | No | Requires `anoncreds-rs` on both sides |
| Wallet / key storage | Partial | Can use simple encrypted file; loses Askar security model |

---

## What the Codebase Already Shows

`SSI_App/test/smart_contracts/bridge_service/index.js` has `verifyVCWithIndy()` returning `true` as a placeholder — the architecture was already conceived as an orchestration layer calling ACA-Py, not a replacement.

`SSI_App/test/client.py` wraps ACA-Py's REST endpoints for schemas, cred-defs, invitations, and credential offers. This is the pattern that should be followed.

`SSI_App/test/smart_contracts/aries-quorum-agent.py` shows `verify_and_execute_transaction()` calling a bridge service — same concept in Python.

---

## Recommended Architecture: Orchestrator, Not Replacement

Build `bridge.py` as an **orchestration layer on top of ACA-Py**, not a replacement for it.

```
bridge.py
├── calls ACA-Py REST API  → handles all cryptography (schemas, cred-defs, issuance)
├── calls indy_vdr directly → only if direct ledger reads are needed (optional)
└── calls web3.py           → writes results to Besu contracts

New dependencies: web3.py only.
No new cryptographic implementations.
```

---

## Conclusion

**Replacing ACA-Py is not practical for this project.**

ACA-Py is not a convenience wrapper — it provides the only available implementation of CL signatures and ZKP proofs in the Hyperledger Anoncreds stack. Any replacement would still require `anoncreds-rs` as a dependency, meaning you would be rebuilding ACA-Py at significant cost for no net gain.

**Replacement would be advisable only if:**
1. You need a different programming language or runtime that cannot call ACA-Py.
2. You abandon ZKP proofs and use simple attribute disclosure (reducing the cryptographic scope significantly).
3. You have months available for protocol implementation and security review.
4. ACA-Py's HTTP overhead has been demonstrated as a real bottleneck (it has not been in this project).

**None of these conditions apply here.**

The correct action is to implement `bridge.py` as described in `02_bridge_design.md`. ACA-Py handles the cryptography; the bridge handles the orchestration logic between the SSI layer and the Besu contracts.
