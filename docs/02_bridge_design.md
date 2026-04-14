# Bridge Design: SSI ↔ Besu Integration

## Problem Statement

The SSI layer (ACA-Py agents + Indy ledger) and the Besu smart contract layer currently operate independently. The smart contracts accept credential parameters but mock all verification (`return true`). There is no component that reads a real ACA-Py credential and uses it to authorize an entity on Besu.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                  bridge.py  (Orchestrator)                      │
│                  Python + requests + web3.py                    │
│                                                                 │
│  PHASE 1: Issue credentials via ACA-Py                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Issuer Admin API  (localhost:8031)                       │  │
│  │  POST /schemas              → register 3 UAM schemas      │  │
│  │  POST /credential-definitions → create CL cred-defs       │  │
│  │  POST /connections/create-invitation                      │  │
│  │  POST /issue-credential/v2.0/send-offer                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  PHASE 2: Verify credentials stored in holders                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Holder Admin APIs  (localhost:8041, 8051, 8061, 8063)    │  │
│  │  GET /credentials  → confirm VC stored + extract attrs    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  PHASE 3: Register entities on Besu                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  web3.py  (localhost:8545)                                │  │
│  │  UserVerification.setRiderPermission(addr, can_ride)      │  │
│  │  VertiportManagement.registerVertiport(id, n_a, n_p, cred)│  │
│  │  EVTOLManagement.registerEVTOL(id, port_id, cred)         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

Tools: requests (already in ecosystem), web3.py (new dependency)
```

---

## Multi-Agent Docker Compose

Extend `credential_test/docker-compose-agents.yml` with one agent per entity type:

| Container | DIDComm port | Admin port | Wallet name | Role |
|---|---|---|---|---|
| `acapy-issuer` | 8030 | 8031 | `issuer_wallet` | Issues all credentials |
| `acapy-user1` | 8040 | 8041 | `user1_wallet` | Holds user_credential |
| `acapy-user2` | 8042 | 8043 | `user2_wallet` | Holds user_credential |
| `acapy-evtol1` | 8050 | 8051 | `evtol1_wallet` | Holds evtol_credential |
| `acapy-vertiport1` | 8060 | 8061 | `vertiport1_wallet` | Holds vertiport_credential |
| `acapy-vertiport2` | 8062 | 8063 | `vertiport2_wallet` | Holds vertiport_credential |

All agents use:
- `image: bcgovimages/aries-cloudagent:py3.12_1.2.0`
- `network_mode: "host"` (Linux — allows access to `localhost:9000` VON Network)
- `--auto-accept-invites --auto-accept-requests --auto-store-credential --auto-ping-connection`
- `--genesis-file /home/indy/genesis.txn` (mounted from `./genesis/genesis.txn`)
- `--wallet-type askar`

Agent template (repeat with different ports/names):
```yaml
  user1:
    image: bcgovimages/aries-cloudagent:py3.12_1.2.0
    container_name: acapy-user1
    network_mode: "host"
    volumes:
      - ./genesis/genesis.txn:/home/indy/genesis.txn
    command: >
      start --label "User1 Agent"
      --endpoint http://localhost:8040
      --inbound-transport http 0.0.0.0 8040
      --outbound-transport http
      --admin 0.0.0.0 8041 --admin-insecure-mode
      --wallet-type askar --wallet-name user1_wallet --wallet-key user1key123
      --auto-provision --auto-accept-invites --auto-accept-requests
      --auto-store-credential --auto-ping-connection
      --genesis-file /home/indy/genesis.txn
```

---

## Bridge Script Workflow (`bridge.py`)

### Phase 1: Issue Credentials

```python
# 1. Register schemas (issuer only needs to do this once)
schemas = {
    "user":       {"name": "user_credential",       "version": "2.0",
                   "attrs": ["first_name", "last_name", "can_ride"]},
    "evtol":      {"name": "evtol_credential",       "version": "3.0",
                   "attrs": ["name", "id_puerto", "updates", "status"]},
    "vertiport":  {"name": "vertiport_credential",   "version": "4.0",
                   "attrs": ["n_airstrip", "m_parkings", "coord_lon", "coord_lat"]},
}

# 2. Create credential definitions (one per schema)

# 3. For each entity: create-invitation → receive-invitation → send-offer
#    user1:      can_ride=true,  first_name=Ana, last_name=Lopez
#    evtol1:     name=Alpha, id_puerto=PORT_1, updates=0, status=ACTIVE
#    vertiport1: n_airstrip=2, m_parkings=4, coord_lon=-71.5, coord_lat=-16.4
```

### Phase 2: Verify Credential Storage

```python
# Poll each holder until credential appears
def get_stored_credential(holder_admin_url, schema_name, timeout=60):
    start = time.time()
    while time.time() - start < timeout:
        r = requests.get(f"{holder_admin_url}/credentials")
        for cred in r.json().get("results", []):
            if schema_name in cred.get("schema_id", ""):
                return cred["attrs"]   # dict of attribute name → value
        time.sleep(2)
    raise TimeoutError(f"Credential {schema_name} not found in {holder_admin_url}")
```

### Phase 3: Credential Bytes and Besu Calls

```python
# Serialize ACA-Py attrs as bytes — real data, not 0x01
def attrs_to_bytes(attrs: dict) -> bytes:
    return json.dumps(attrs, sort_keys=True).encode("utf-8")

# User authorization
user_attrs = get_stored_credential("http://localhost:8041", "user_credential")
can_ride = user_attrs.get("can_ride", "false").lower() == "true"
user_contract.functions.setRiderPermission(BESU_USER1_ADDR, can_ride).build_transaction(...)

# Vertiport registration
vp_attrs = get_stored_credential("http://localhost:8061", "vertiport_credential")
vp_cred_bytes = attrs_to_bytes(vp_attrs)
vp_contract.functions.registerVertiport(
    "VERTIPORT_1",
    int(vp_attrs["n_airstrip"]),
    int(vp_attrs["m_parkings"]),
    vp_cred_bytes
).build_transaction(...)

# eVTOL registration
ev_attrs = get_stored_credential("http://localhost:8051", "evtol_credential")
ev_cred_bytes = attrs_to_bytes(ev_attrs)
ev_contract.functions.registerEVTOL(
    EVTOL_ID, ev_attrs["id_puerto"], ev_cred_bytes
).build_transaction(...)
```

### Why `attrs_to_bytes()` Matters

The current smoke test passes `"0x01"` as a credential — a single byte with no content. Using the actual serialized attributes means:
- The `bytes memory credential` parameter contains real entity data
- When the Solidity mock is replaced, these bytes already carry the right payload
- Only the on-chain verification logic in Solidity needs to change — the bridge does not

---

## Contract Address Management

`deploy_system.js` should write addresses to a file after deployment:

```js
// At the end of deploy_system.js main()
const addresses = {
  UserVerification:    userVerificationAddress,
  VertiportManagement: vertiportManagementAddress,
  EVTOLManagement:     evtolManagementAddress,
  FlightReservation:   flightReservationAddress,
};
fs.writeFileSync(
  path.resolve(__dirname, "../deployed_addresses.json"),
  JSON.stringify(addresses, null, 2)
);
console.log("Addresses written to deployed_addresses.json");
```

`smoke_test_full_system.js` and `bridge.py` then read from this file instead of hardcoding.

---

## Full System Startup Order

```
1. SSI_Project:   ./manage start --wait          (Indy ledger, port 9000)
2. credential_test: bash start_test.sh            (issuer :8031, holder :8041)
   [extended]    add user2, evtol1, vertiport1/2
3. BESU_project:  ./run.sh                        (Besu RPC, port 8545)
4.                node scripts/deploy_system.js   (writes deployed_addresses.json)
5. bridge.py:     python bridge.py               (issues VCs, registers on Besu)
6.                node scripts/smoke_test_full_system.js
```

---

## Dependencies to Add

```
# bridge.py requirements
web3>=6.0.0       # Besu interaction
requests          # already in SSI_App/requirements.txt
```

No other new dependencies. ACA-Py and the Besu network are already running.

---

## Existing Bridge Skeleton

`SSI_App/test/smart_contracts/bridge_service/index.js` already implements the Besu side (`executeOnQuorum`) with `verifyVCWithIndy()` returning `true` as a placeholder. `SSI_App/test/smart_contracts/aries-quorum-agent.py` shows the same concept in Python. The `bridge.py` described here is the completion of this design using ACA-Py's REST API instead of the placeholder.
