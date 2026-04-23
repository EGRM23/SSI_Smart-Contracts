# Prioritized Remaining Functions — TI3 System

**Date:** 2026-04-21
**Source:** Analysis of TI_Examen_3.pdf against current codebase state

---

## Question

Based on the proposal described in TI3.pdf and considering what has already been implemented in the existing project folders, identify and prioritize the remaining functions that need to be developed. List them from highest urgency to lowest priority, focusing primarily on core system functionality, architecture, and backend logic rather than user interface aspects. For each function, include a brief explanation of its purpose, how it relates to the current implementation, and the reason for its prioritization.

---

## Answer

### 1. Fix the ACApyClient core layer (BLOCKING)

**Purpose:** The `aca_py/client.py` file has only 3 methods (`create_wallet`, `create_invitation`, `send_message`) and no `_make_request()` base method. Every service in `SSI_App` calls `self.client._make_request()` which does not exist.

**Relation to current implementation:** `credential_service.py`, `dni_service.py`, and `wallet_service.py` all depend on this missing method. Nothing in the credential pipeline runs. The fix requires implementing `_make_request()` as a generic HTTP wrapper (GET/POST/DELETE against the ACA-Py Admin API), then adding the missing endpoint methods: schema creation, credential definition creation, proof request, credential offer, credential status polling.

**Why first:** Every other SSI_App function is dead until this works. This is a structural prerequisite, not a feature.

---

### 2. Fix critical bugs in the service layer (BLOCKING)

**Purpose:** Three concrete bugs block all credential operations independently of the client issue:

- `credential_service.py:2` — `from .wallet_service import ACApyClient` (wrong module, fails at import)
- `dni_service.py:24` — `user=user` passed to `CredentialIssuance.objects.create()` (wrong FK; model expects `wallet=`)
- `wallet_service.py` / `dni_service.py:50` — calls `create_connection_for_user()` which is not defined anywhere

**Relation to current implementation:** These are not design gaps but concrete runtime errors in already-written code. Django user registration partially works (`user_views.py` + wallet creation), but the moment any credential operation is triggered, all three bugs fire independently.

**Why second:** These are fast fixes with immediate unblocking effect. Fixing them makes the existing partial implementation actually executable.

---

### 3. Unify credential schema definitions across all components (BLOCKING for integration)

**Purpose:** The system currently uses inconsistent schema names and versions in four different places:

| Location | User credential | eVTOL credential | Vertiport credential |
|---|---|---|---|
| Paper / bridge design | `user_credential` v2.0 | `evtol_credential` v3.0 | `vertiport_credential` v4.0 |
| `credential_test/` | — | `eVTOL_Credential` v1.0 | — |
| `credential_service.py` | `DNI` v1.0 | — | — |
| Solidity comments | `Rider_Credential` | `Evtol_Crede` | `Port_Credential` |

**Relation to current implementation:** The `credential_test/` flow works for eVTOL only because it's isolated. As soon as the bridge needs to match credentials from ACA-Py to Besu contract parameters, attribute names must be consistent. A `can_ride` attribute in the contract but `can_use` in the schema causes silent failures.

**Why third:** This is a design decision that must be settled before writing the bridge, because the bridge depends on stable schema/attribute names. Define the canonical schemas once and update all references.

---

### 4. Implement the SSI–Besu bridge (`bridge.py`) (CORE INTEGRATION)

**Purpose:** The central missing piece identified by the paper. A `bridge.py` module that orchestrates the three-phase integration: (1) issues credentials to all entities (user, eVTOL, vertiport) via ACA-Py issuer; (2) polls each holder agent to confirm credential storage and extract attributes; (3) calls Besu smart contracts via `web3.py` to register each entity on-chain with the extracted attribute values.

**Relation to current implementation:** A design document exists at `docs/02_bridge_design.md` with the exact multi-agent topology (6 agents: issuer, 2 users, 1 eVTOL, 2 vertiports) and the three-phase flow. The `credential_test/connect_and_issue.py` script is the working proof-of-concept for phases 1 and 2 with eVTOL only. What's needed is: `web3.py` added to dependencies, a generalized version of `connect_and_issue.py` covering all three entity types, and the web3 call layer for phase 3. The smart contracts are ready to receive this data — they already accept credential bytes in every registration function.

**Why fourth:** This is the paper's primary architectural claim. It transforms the system from two independent working subsystems into the integrated SSI+blockchain framework described. Everything before this was clearing blockers; this is the actual deliverable.

---

### 5. Add the multi-agent Docker Compose for all entity types (INFRASTRUCTURE FOR BRIDGE)

**Purpose:** The bridge requires 6 ACA-Py agent instances (issuer + user × 2 + eVTOL × 1 + vertiport × 2). Currently only the `credential_test/` setup provides a working 2-agent configuration (issuer + holder, eVTOL only). The `issuer/` and `holder1/` directories are broken (missing `--auto-accept-*` flags). The `aries_client/` multi-tenant setup is unused.

**Relation to current implementation:** The `credential_test/docker-compose-agents.yml` is the template. It needs to be extended to 6 containers with correct port assignments and genesis URL injection. The `start_test.sh` pattern (wait for readiness, download genesis) is already proven and should be replicated.

**Why fifth:** The bridge code cannot be run or tested without the agent infrastructure being stable. This is the runtime environment for function 4.

---

### 6. Implement the `Vertiport` model and its wallet integration (DATA MODEL GAP)

**Purpose:** `Vertiport` is referenced throughout the system (Besu contracts manage vertiport state, the bridge needs to issue vertiport credentials, the paper lists it as a first-class entity) but the Django model does not exist. The `Wallet` model supports polymorphic ownership via `GenericForeignKey`, so adding a `Vertiport` model follows the same pattern already used for `User` and `EVTOL`.

**Relation to current implementation:** `EVTOL` model exists but also has no wallet relationship yet. Both need to be completed: `content_type` + `object_id` pointing at their respective wallets. The bridge phase 1 needs these models to exist before it can create wallets and issue credentials to them.

**Why sixth:** Required for the bridge to manage vertiport and eVTOL identities through Django. Without this, the bridge must manage these identities entirely outside Django, which undermines the multi-tenant architecture.

---

### 7. Uncomment and fix the credential views and URL routes (API LAYER)

**Purpose:** `credential_views.py` has four endpoints fully written but entirely commented out: `issue_dni_credential`, `get_my_credentials`, `get_credential_status`, and `credential_webhook`. The corresponding URL routes in `wallet/urls.py` are also commented out. The webhook endpoint is particularly important — it is the ACA-Py callback that notifies Django when a credential state changes (offer accepted, credential stored), enabling event-driven updates instead of polling.

**Relation to current implementation:** Once bugs 1–3 are fixed, these views become functional with minimal changes. The webhook needs a route registered and the ACA-Py agents need to be configured with `--webhook-url` pointing to Django.

**Why seventh:** Necessary to make Django the controller of credential lifecycle rather than an external script. The bridge can work without Django views initially, but production-grade behavior requires Django to receive ACA-Py webhook events.

---

### 8. Make smoke test contract addresses dynamic (OPERATIONAL CORRECTNESS)

**Purpose:** `smoke_test_full_system.js` hardcodes contract addresses from a specific previous deployment (lines 9–12). Every time `deploy_system.js` runs, it produces new addresses, but the smoke test is never updated. The fix is to have `deploy_system.js` write a `deployed_addresses.json` file and have the smoke test read from it.

**Relation to current implementation:** The deployment script already outputs addresses to console. This is a small change that eliminates a recurring manual error. The paper's Fig. 7 shows the smoke test output as a validation result — it should be reliably reproducible without manual intervention.

**Why eighth:** Low effort, high reliability impact. Every developer running the system hits this issue on the second deployment.

---

### 9. Implement credential revocation (PAPER-STATED FUTURE WORK)

**Purpose:** The paper explicitly identifies revocation as a critical open challenge. On the Indy side: create a revocation registry per credential definition and call ACA-Py's `POST /revocation/revoke`. On the Besu side: update `UserVerification`, `EVTOLManagement`, and `VertiportManagement` to reject non-revocation proofs, and emit an event when an entity's authorization is revoked.

**Relation to current implementation:** None — this is entirely missing. The contracts currently accept any credential bytes without checking revocation status. A revoked eVTOL or suspended user would still be authorized on-chain with the current implementation.

**Why ninth:** Operationally necessary for the UAM safety model but requires the bridge (function 4) to be working first, since revocation is only meaningful once real credential verification is in place.

---

### 10. Replace `verifyCredentialSignature()` mocks in Solidity contracts (CRYPTOGRAPHIC INTEGRITY)

**Purpose:** All three verification contracts (`UserVerification`, `VertiportManagement`, `EVTOLManagement`) contain:
```solidity
// MOCK: siempre true
return true;
```
The eventual implementation should verify a hash or zero-knowledge proof derived from the Indy credential against a trusted issuer public key anchored on-chain, or delegate verification to an oracle pattern where the ACA-Py bridge pre-validates and submits a signed attestation.

**Relation to current implementation:** The contracts are architecturally ready — they already accept `bytes memory credential` parameters in every registration function. The bridge design proposes passing `abi.encode(attribute_value, issuer_did)` as a short-term step before full ZK proof integration.

**Why last:** Requires the bridge to be fully operational first. It also involves the deepest cryptographic design decision in the system and is explicitly deferred to future work in the paper.

---

## Summary Table

| # | Function | Layer | Current state | Estimated scope |
|---|---|---|---|---|
| 1 | Fix `ACApyClient` / add `_make_request()` | SSI_App | Broken (missing method) | Small |
| 2 | Fix 3 runtime bugs in service layer | SSI_App | Broken (wrong FK, wrong import, missing method) | Small |
| 3 | Unify credential schema names | All layers | Inconsistent across 4 locations | Small (decision + search/replace) |
| 4 | Implement `bridge.py` (3-phase integration) | Integration | Design exists, 0% code | Large |
| 5 | Multi-agent Docker Compose (6 agents) | Infrastructure | Template exists (2-agent) | Medium |
| 6 | `Vertiport` model + wallet relationship | SSI_App | Model missing entirely | Small |
| 7 | Uncomment credential views + webhook | SSI_App | Code exists, commented out | Small |
| 8 | Dynamic contract addresses in smoke test | BESU | Hardcoded, breaks on redeploy | Small |
| 9 | Credential revocation | SSI + Besu | Missing entirely | Large |
| 10 | Replace `return true` in Solidity mocks | BESU | Intentional mock | Large (research-level) |
