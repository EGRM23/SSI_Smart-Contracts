# Paper vs. Implementation Assessment

Source: Analysis of `TI_Examen_3.pdf` — "Identidad soberana y contratos inteligentes para la gestión de vehículos autónomos" (Quicaño Miranda & Ruiz Mamani, UNSA, January 2026)

---

## What the Paper Claims

The paper proposes and validates a three-layer architecture for UAM (Urban Air Mobility):

| Layer | Technology | Purpose |
|---|---|---|
| SSI / Ledger | Hyperledger Indy (VON Network) | DID and VC trust registry |
| Agent | Hyperledger Aries (ACA-Py, multi-tenant) | Wallet, credential issuance/verification |
| Business Logic | Hyperledger Besu (smart contracts) | On-chain trip orchestration |

**Reported results:** multi-tenant wallets per entity, credential schema registration, issuance and verification (Figs. 4–6), end-to-end trip execution (Fig. 7). SSI–Besu cryptographic integration is explicitly acknowledged as **mocked**.

---

## Credential Schemas Defined in the Paper (Fig. 5)

```json
evtol_credential    v3.0  { name, id_puerto, updates, status }
user_credential     v2.0  { first_name, last_name, can_ride }
vertiport_credential v4.0 { n_airstrip, m_parkings, coord_lon, coord_lat, ... }
```

These must be used consistently in: ACA-Py schema registration, bridge script, and Solidity mock comments.

---

## Strengths (What Is Solid)

**1. Smart contract layer is complete and well-engineered.**
All four contracts exist, compile, deploy, and pass the smoke test. Interface segregation (`IUserVerification`, `IEVTOLManagement`, `IVertiportManagement`) is clean. State machines (`PARKED→EXPECTING→IN_USE→PARKED`, `REQUESTED→CONFIRMED→IN_PROGRESS→COMPLETED`) are correctly enforced. Mock boundary properly documented in code comments.

**2. VON Network is fully functional.**
4-node Indy network correctly containerized. `./manage` wrapper handles full lifecycle.

**3. Django data model is architecturally correct.**
`Wallet` using `GenericForeignKey` is the right design for polymorphic wallet ownership. `CredentialIssuance` tracks the full ACA-Py state machine. `Connection` tracks DIDComm state.

**4. Credential test suite validates the SSI layer independently.**
`credential_test/connect_and_issue.py`, `test_credentials.ipynb`, and `multitenant_test.ipynb` demonstrate that wallet creation, DID registration, schema/cred-def creation, and Issue Credential 2.0 flow all work when called directly against ACA-Py. These scripts are the basis for Figs. 4–6 in the paper.

---

## Critical Discrepancies and Weaknesses

### 1. Django credential schemas do not match the paper's UAM schemas
`services/credential_service.py` only implements a generic **DNI** schema (`nombres, apellidos, fecha_nacimiento`). This schema has nothing to do with UAM or eVTOLs. The Django service cannot reproduce the paper's credential results.

### 2. All credential API endpoints are disabled
`wallet/urls.py` has every credential route commented out. The only reachable endpoint is `POST /api/user/`. The issuance pipeline (`issue_dni_credential`, `get_my_credentials`, `credential_webhook`) exists as dead code.

### 3. `Vertiport` model is missing from Django
The paper states vertiports also receive wallets and vertiport credentials. `auth_app/models.py` only defines `User` and `EVTOL`. There is no `Vertiport` model.

> **Note:** Django modifications are deferred. These are documented for completeness.

### 4. `dni_service.py` has a runtime bug
```python
# Line 24 — wrong field name
CredentialIssuance.objects.create(user=user, ...)
# The model has a `wallet` FK, not `user`
```

### 5. The SSI–Besu bridge does not exist
There is no bridge service, no connector, and no intermediate component that reads an ACA-Py credential verification result and calls a Besu contract. `ssi-credential-system/` directory is empty.

### 6. Hardcoded contract addresses in the smoke test
`smoke_test_full_system.js` lines 9–12 contain addresses from a previous deployment. On each redeploy the test silently uses stale addresses.

---

## Root Cause of the Gap

The paper's experimental results (Figs. 4–6: wallet creation, schemas, credential verification) were obtained by running `credential_test/` scripts and Jupyter notebooks **directly against ACA-Py admin APIs**, completely bypassing the Django application. The Django layer was developed in parallel but never wired up to match the paper's use-case schemas, and its HTTP interface was left disabled.

---

## Priority Action Items

### Immediate — SSI layer alignment (no Django changes)
1. Align schema names/versions across all scripts to match paper exactly:
   - `evtol_credential v3.0`, `user_credential v2.0`, `vertiport_credential v4.0`
2. Write `credential_test/issue_all_entities.py` — issue all three credential types to their respective holder agents (user, eVTOL, vertiport).
3. Upgrade `connect_and_issue.py` to Issue Credential **v2.0** (`/issue-credential/v2.0/`).

### Short term — Besu
4. Replace hardcoded addresses in `smoke_test_full_system.js` with a `deployed_addresses.json` written by `deploy_system.js`.

### Core gap
5. Implement `bridge.py` — see `02_bridge_design.md` for full specification.
