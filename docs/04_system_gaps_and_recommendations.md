# System Gaps, Current Status, and Recommendations

Last updated: April 2026

---

## Current Status by Component

### VON Network (`SSI_Project/`)
**Status: Fully operational.**
- 4-node Indy network, ledger browser on port 9000.
- `./manage start --wait` brings up all nodes cleanly.
- Genesis file served at `http://localhost:9000/genesis`.

### ACA-Py Agents — `credential_test/` path
**Status: Fully operational.**
- `credential_test/docker-compose-agents.yml`: issuer (8031) + holder (8041).
- `start_test.sh` downloads genesis and waits for both agents.
- `connect_and_issue.py` completes the full eVTOL credential issuance flow.
- This is the **canonical, working path** for SSI operations.

### ACA-Py Agents — `issuer/` path (ports 9040/9041)
**Status: Incomplete — do not use.**
- Missing `--auto-accept-invites`, `--auto-accept-requests`, `--auto-store-credential`.
- Cannot complete an issuance flow without manual intervention.
- Origin of the port mismatch confusion (`8031` vs `9041`).

### ACA-Py Agents — `aries_client/` path (multi-tenant + PostgreSQL)
**Status: Separate setup, not used by test scripts.**
- Launched by `start_aries.sh`.
- Uses multi-tenant mode with JWT authentication.
- Not referenced by `connect_and_issue.py` or `credential_test/`.
- Intended for the Django integration layer (deferred).

### Besu Smart Contracts (`BESU_project/`)
**Status: Fully operational.**
- 4 contracts: `UserVerification`, `VertiportManagement`, `EVTOLManagement`, `FlightReservation`.
- 7-step trip lifecycle smoke test passes.
- SSI credential params accepted but always verified as `true` (explicit mock).
- Contract addresses in `smoke_test_full_system.js` are **hardcoded from a previous deployment** — must be updated after each `./run.sh` + `deploy_system.js`.

### SSI–Besu Bridge
**Status: Does not exist.**
- `SSI_App/test/smart_contracts/bridge_service/index.js` is a skeleton with placeholder verification.
- `SSI_App/ssi-credential-system/` is an empty directory.
- See `02_bridge_design.md` for the full implementation plan.

### Django Application (`SSI_App/`)
**Status: Partially implemented — intentionally deferred.**
- Only `POST /api/user/` is active.
- All credential endpoints commented out in `wallet/urls.py`.
- `credential_service.py` uses DNI schema (does not match paper's UAM schemas).
- `Vertiport` model missing from `auth_app/models.py`.
- `dni_service.py` line 24 has a runtime bug (`user=user` should be `wallet=wallet`).

---

## Gap List

| # | Gap | Severity | Component |
|---|---|---|---|
| G1 | No SSI→Besu bridge | Critical | Architecture |
| G2 | Schema names/versions inconsistent across codebase | High | SSI_App |
| G3 | Only eVTOL credential tested; user_credential and vertiport_credential have no test scripts | High | credential_test |
| G4 | `connect_and_issue.py` uses Issue Credential v1.0; paper states v2.0 | Medium | credential_test |
| G5 | Hardcoded contract addresses in smoke test | Medium | BESU_project |
| G6 | Three conflicting ACA-Py agent setups (no canonical config) | Medium | SSI_App |
| G7 | Smoke test credential bytes are meaningless (`0x01`) | Medium | BESU_project |
| G8 | No credential revocation | Low (future work) | SSI_App |
| G9 | No access control beyond single-admin on Besu | Low (future work) | BESU_project |

---

## Specific Recommendations

### G1 — Implement `bridge.py` (closes the core gap)
See `02_bridge_design.md` for full design. Short version:
- Phase 1: issue all 3 credential types via ACA-Py (`/issue-credential/v2.0/`)
- Phase 2: poll each holder `GET /credentials` to confirm storage
- Phase 3: call `setRiderPermission`, `registerVertiport`, `registerEVTOL` via web3.py
- New dependency: `web3>=6.0.0`

### G2 — Align schema names and versions everywhere
All three locations must use identical values:
```
user_credential      v2.0  { first_name, last_name, can_ride }
evtol_credential     v3.0  { name, id_puerto, updates, status }
vertiport_credential v4.0  { n_airstrip, m_parkings, coord_lon, coord_lat }
```
Locations: `connect_and_issue.py`, any new `issue_all_entities.py`, Solidity mock comments.

### G3 — Write `credential_test/issue_all_entities.py`
Single script that issues all three credential types to their respective holders. This validates the SSI side for all entity types and provides the direct input to Phase 1 of the bridge.

### G4 — Upgrade to Issue Credential v2.0
In `connect_and_issue.py`, change:
```python
# Before
"/issue-credential/send-offer"
"@type": "issue-credential/1.0/credential-preview"

# After
"/issue-credential/v2.0/send-offer"
"@type": "issue-credential/2.0/credential-preview"
```

### G5 — Auto-write deployed addresses
Add to end of `deploy_system.js`:
```js
fs.writeFileSync(
  path.resolve(__dirname, "../deployed_addresses.json"),
  JSON.stringify({ UserVerification: uv, VertiportManagement: vm,
                   EVTOLManagement: em, FlightReservation: fr }, null, 2)
);
```
Read from this file in `smoke_test_full_system.js` and `bridge.py`.

### G6 — Consolidate to one canonical agent setup
Keep `credential_test/docker-compose-agents.yml` as the working base. Extend it with the additional agents for the bridge test. Document `issuer/` as deprecated. `aries_client/` is kept for future Django integration but clearly scoped.

### G7 — Pass real credential bytes from bridge
Use `json.dumps(attrs).encode("utf-8")` instead of `"0x01"`. See `02_bridge_design.md:attrs_to_bytes()`.

---

## What Is NOT Changed (Deferred)

- Django application (`SSI_App/` — `auth_app/`, `services/`, `wallet/`) — no modifications.
- Django credential schemas or model changes.
- Multi-tenant ACA-Py setup (`aries_client/`) — left for future Django integration phase.

---

## Startup Order (Full System)

```bash
# 1. Indy ledger
cd SSI_Project && ./manage start --wait

# 2. ACA-Py agents (credential_test canonical setup)
cd SSI_App/credential_test && bash start_test.sh --url http://localhost:9000/genesis

# 3. Besu network
cd BESU_project && ./run.sh

# 4. Deploy contracts (writes deployed_addresses.json)
cd BESU_project/smart_contracts && node scripts/deploy_system.js

# 5. Bridge — issue VCs + register on Besu
python bridge.py

# 6. Smoke test (reads deployed_addresses.json)
node scripts/smoke_test_full_system.js
```
