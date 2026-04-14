# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is a **TI3 (Taller de Integración 3) project** that prototypes an **eVTOL travel system backed by blockchain and Self-Sovereign Identity (SSI)**. The three sub-projects work together:

1. **`SSI_Project/`** — Local Hyperledger Indy network (VON Network). Provides the DID ledger that SSI agents write schemas, credential definitions, and DIDs to.
2. **`SSI_App/`** — Django application + ACA-Py agents. Manages user wallets and Verifiable Credential issuance/verification using the Indy ledger above.
3. **`BESU_project/`** — Hyperledger Besu blockchain network + Solidity smart contracts. Runs the on-chain eVTOL trip logic (user authorization, vertiport capacity, flight reservations).

The integration point: SSI credentials (user authorization, eVTOL, vertiport) are verified **off-chain** by the ACA-Py agents, then the result is passed to the Besu smart contracts **on-chain** as flags/identifiers. On-chain verification is currently mocked (`return true` in contracts).

## Startup Order

To run the full system, start components in this order:
1. `SSI_Project/` (Indy ledger — port 9000)
2. `SSI_App/` agents (ACA-Py issuer on 9040/9041, holder on 9030/9031)
3. `SSI_App/` Django server (port 8000)
4. `BESU_project/` network (Besu RPC on 8545/8546)
5. Deploy smart contracts via `BESU_project/smart_contracts/scripts/deploy_system.js`

## Sub-project Commands

Each sub-project has its own `CLAUDE.md` with detailed commands. Key entry points:

### SSI_Project (VON Network / Indy ledger)
```bash
cd SSI_Project
./manage build
./manage start --wait        # starts 4 Indy nodes + ledger browser on port 9000
./manage stop
./manage down                # removes containers + volumes
./manage cli                 # interactive Indy CLI
```

### SSI_App (Django + ACA-Py)
```bash
cd SSI_App
bash start_aries.sh --url http://localhost:9000/genesis --rebuild   # start ACA-Py agents
python manage.py migrate
python manage.py runserver                                           # Django on port 8000
```

### BESU_project (Besu network + contracts)
```bash
cd BESU_project
./run.sh                                    # start Besu network (Docker Compose)
cd smart_contracts && npm install
npm run compile                             # compile .sol → JSON artifacts
node scripts/deploy_system.js               # deploy all 4 contracts
node scripts/smoke_test_full_system.js      # end-to-end trip smoke test
./stop.sh / ./resume.sh / ./remove.sh
```

## Architecture: How the Three Pieces Fit

```
[Browser / Client]
       │
       ▼
Django REST API (SSI_App, port 8000)
  └── WalletService / CredentialService / DNIService
        └── ACApyClient → ACA-Py Agents (issuer:9041, holder:9031)
              └── indy_vdr → Indy Nodes (SSI_Project, ports 9701–9708)
                              ↑ Ledger browser at port 9000

[On-chain]
FlightReservation contract (BESU_project)
  └── calls UserVerification, VertiportManagement, EVTOLManagement
        (SSI credential params accepted but verification mocked)
Besu RPC node: localhost:8545
```

## Key Design Decisions

- **On-chain credential verification is mocked**: The Solidity contracts accept credential parameters but always return `true`. Real verification requires completing the Aries/Indy integration.
- **`FlightReservation` is the single Besu entry point**: All trip operations go through this contract; it calls the other three internally.
- **Wallet polymorphism in Django**: `Wallet` uses `GenericForeignKey` so any model (User, EVTOL, etc.) can own an ACA-Py wallet. Use Django's `ContentType` framework when querying ownership.
- **Deploy addresses must be updated after each deployment**: `smoke_test_full_system.js` hardcodes contract addresses — update them after running `deploy_system.js`.
- **Well-known Besu dev key**: The deploy account (`0x8f2a55...`) is a public test key with no real value. Local dev only.
