# Flujo completo: desde el arranque hasta la credencial almacenada

**Última actualización:** 2026-06-09

Este documento describe cada paso que ocurre desde que se levanta el sistema hasta que una credencial queda guardada en el wallet del holder, incluyendo qué archivo ejecuta cada acción y dónde vive cada pieza de datos.

---

## Mapa general

```
[Terminal 1]           [Terminal 2]              [Terminal 3]
SSI_Project/           credential_test/           SSI_App/
./manage start         docker compose up          python connect_and_issue.py
      │                      │                           │
      ▼                      ▼                           ▼
  VON Network            Agentes ACA-Py            Script de prueba
  (ledger Indy)          (issuer, holders)         (o Django en producción)
```

---

## Fase 1 — Arranque de la VON Network

**Comando:** `cd SSI_Project && ./manage start --wait`
**Archivos involucrados:**
- `SSI_Project/manage` — script bash que orquesta Docker Compose
- `SSI_Project/docker-compose.yml` — define los 4 nodos Indy + webserver
- `SSI_Project/genesis/` — generado automáticamente al crear la red

**Qué pasa:**
1. Docker levanta 5 contenedores: `von-node1`, `von-node2`, `von-node3`, `von-node4`, `von-webserver`
2. Los 4 nodos Indy se sincronizan usando BFT (Byzantine Fault Tolerance) — necesitan que al menos 3 de 4 estén activos para aceptar escrituras
3. El webserver queda escuchando en el puerto 9000 con:
   - `/genesis` → devuelve las transacciones iniciales del pool
   - `/register` → permite registrar DIDs (solo en desarrollo)
   - `/browse/domain` → explorador visual del ledger

**Dónde viven los datos:**
- Volúmenes Docker `von_node1-data` ... `von_node4-data` (RocksDB dentro de cada contenedor)
- Se pierden con `./manage down`, se conservan con `./manage stop`

---

## Fase 2 — Registro del DID del issuer en el ledger

> **⚠ Esta fase NO está automatizada en ningún archivo del proyecto.**
> Es un paso manual que debe ejecutarse después de cada `./manage down`.

### Por qué es necesario

El issuer arranca con `--seed issuer00000000000000000000000001` en el docker-compose. Ese seed genera localmente un par de claves ed25519:

```
seed (32 bytes) ──► clave privada  → queda en wallet Askar del contenedor
                └─► clave pública  → DID: Utwqp5cpEATQpGZL5WSQZJ
                                      Verkey: GCqSzwWVsRumDY67zkBACHs92zQJpg71WKmFEHnTreUQ
```

El DID existe localmente pero el ledger no lo conoce. Para escribir schemas y credential definitions en el ledger, el issuer necesita el rol **TRUST_ANCHOR** (Endorser). Sin este registro, cualquier intento de escritura falla con `could not authenticate`.

### La jerarquía de permisos de Indy

```
TRUSTEE  →  puede crear Stewards
  └── STEWARD  →  puede registrar Trust Anchors
        └── TRUST_ANCHOR / ENDORSER  →  puede escribir schemas y cred defs
              └── (sin rol)  →  solo lectura
```

Los Stewards están pre-configurados en el genesis. En VON Network de desarrollo hay 4, con seeds conocidos (ej. `000000000000000000000000Steward1` → DID `Th7MpTaRZVRYnPiabds81Y`). El webserver tiene acceso a sus claves privadas.

### Qué hace el endpoint `/register`

```
curl POST http://localhost:9000/register
  {"did": "...", "verkey": "...", "role": "TRUST_ANCHOR"}
        │
        ▼
  von-webserver-1
  ├── toma la clave privada de Steward1
  ├── construye transacción NYM:
  │     type=NYM, dest=<DID issuer>, verkey=<verkey>, role=101 (TRUST_ANCHOR)
  ├── firma la transacción con Steward1
  └── envía a los nodos Indy
              │
              ▼
        3 de 4 nodos validan y confirman
        Transacción queda permanentemente en el domain ledger
```

### Dónde queda guardado

| Lugar | Qué guarda | Se pierde con |
|---|---|---|
| Domain ledger (volúmenes von_node*) | DID + verkey + rol | `./manage down` |
| Wallet Askar del issuer (volumen del contenedor acapy-issuer) | Clave privada | `docker compose down` |

### Los holders NO necesitan registrarse

user1, evtol1, vertiport1, vertiport2 solo leen del ledger y reciben mensajes DIDComm. Sus DIDs son locales y no requieren ningún rol.

### Brecha actual y solución propuesta

Hoy hay que ejecutar el curl manualmente. Debería automatizarse en `start_test.sh` verificando si el DID ya está registrado antes de arrancar los agentes:

```bash
# verificar si ya está registrado
REGISTERED=$(curl -s "http://localhost:9000/browse/domain?query=Utwqp5cpEATQpGZL5WSQZJ&type=1" | grep -c seqNo)
if [ "$REGISTERED" -eq 0 ]; then
  curl -X POST http://localhost:9000/register \
    -H "Content-Type: application/json" \
    -d '{"did":"Utwqp5cpEATQpGZL5WSQZJ","verkey":"GCqSzwWVsRumDY67zkBACHs92zQJpg71WKmFEHnTreUQ","alias":"issuer","role":"TRUST_ANCHOR"}'
fi
```

---

## Fase 3 — Arranque de los agentes ACA-Py

**Comando:** `docker compose -f docker-compose-agents.yml up issuer user1`
**Archivos involucrados:**
- `credential_test/docker-compose-agents.yml` — define cada agente con su puerto y configuración
- `credential_test/genesis/genesis.txn` — montado como volumen en cada contenedor

**Qué pasa por cada agente:**

```
docker-compose-agents.yml
    └── acapy-issuer (imagen: bcgovimages/aries-cloudagent:py3.12_1.2.0)
          ├── --seed issuer000...  → genera DID Utwqp5cpEATQpGZL5WSQZJ (siempre igual)
          ├── --wallet-name issuer_wallet  → crea/abre base Askar en /home/aries/.aries_cloudagent
          ├── --genesis-file /home/indy/genesis.txn  → conecta a la VON Network
          ├── --admin 0.0.0.0 8031  → expone Admin API en puerto 8031
          ├── --auto-accept-invites/requests  → acepta conexiones automáticamente
          └── --auto-ping-connection  → mantiene conexiones activas

    └── acapy-user1 (holder)
          ├── sin --seed  → DID aleatorio en cada arranque desde cero
          ├── --auto-respond-credential-offer  → acepta ofertas automáticamente
          └── --auto-store-credential  → almacena credenciales al recibirlas
```

Los agentes están listos cuando imprimen `Listening...` sin `Shutting down`.

---

## Fase 4 — Registro del schema en el ledger

**Disparado por:** `connect_and_issue.py` paso 4 / `CredentialService.get_or_create_schema()`
**Archivos:** `services/credential_service.py:16` → `aca_py/client.py:63`

**Qué pasa:**
```
CredentialService.get_or_create_schema()
  └── ACApyClient.create_schema("evtol_credential", "3.0", [...])
        └── POST http://localhost:8031/schemas
              └── ACA-Py del issuer firma transacción SCHEMA con su clave privada
                    └── Se escribe en el ledger Indy
                          └── schema_id = "Utwqp5cpEATQpGZL5WSQZJ:2:evtol_credential:3.0"
```

El `schema_id` sigue el formato `<DID_issuer>:2:<nombre>:<versión>`. Queda permanentemente en el ledger — cualquier agente en la red puede leerlo para verificar credenciales. La versión `credential_service.py` verifica primero si ya existe para no intentar crearlo dos veces.

---

## Fase 5 — Registro de la credential definition en el ledger

**Disparado por:** `connect_and_issue.py` paso 5 / `CredentialService.get_or_create_credential_definition()`
**Archivos:** `services/credential_service.py:24` → `aca_py/client.py:75`

**Qué pasa:**
```
CredentialService.get_or_create_credential_definition()
  └── ACApyClient.create_credential_definition(schema_id, tag="default")
        └── POST http://localhost:8031/credential-definitions
              └── ACA-Py genera un par de claves CL (Camenisch-Lysyanskaya):
                    ├── Clave pública  → se escribe en el ledger
                    └── Clave privada  → queda en wallet Askar del issuer (NUNCA sale)
                          └── cred_def_id = "Utwqp5cpEATQpGZL5WSQZJ:3:CL:8:default"
```

La clave privada CL es lo que permite al issuer firmar credenciales. Si se pierde (por `docker compose down`), el issuer no puede emitir contra esa cred def aunque el schema siga en el ledger. La cred def tendría que recrearse con un tag distinto.

---

## Fase 6 — Establecimiento de conexión DIDComm

**Disparado por:** `connect_and_issue.py` pasos 1-3 / `UserCredentialService._establish_connection()`
**Archivos:** `services/user_credential_service.py:39` → `aca_py/client.py:42-47`

**Qué pasa (protocolo RFC 0160 — pendiente migrar a RFC 0023):**

```
Issuer (8031)                              Holder (8041)
     │                                          │
     │  POST /connections/create-invitation     │
     │  ACApyClient.create_invitation()         │
     │  genera: connection_id + clave efímera   │
     │  devuelve: invitation JSON               │
     │                                          │
     │  [script/Django pasa el invitation JSON] │
     │                                          │
     │             POST /connections/receive-invitation
     │             ACApyClient.receive_invitation()
     │             Holder genera su DID para esta conexión
     │◄─── connection REQUEST ─────────────────│
     │                                          │
     │  --auto-accept-requests                  │
     │──── connection RESPONSE ────────────────►│
     │                    [estado: active] ✅   │
```

**En Django** (`user_credential_service.py`): este proceso crea un registro `Connection` en la base de datos SQLite con el `connection_id`, estado y `invitation_url`.

---

## Fase 7 — Emisión de la credencial (protocolo de 3 mensajes)

**Disparado por:** `connect_and_issue.py` pasos 6-7 / `UserCredentialService.issue_credential_to_user()`
**Archivos:** `services/user_credential_service.py:19` → `services/credential_service.py:35` → `aca_py/client.py:84`

**Qué pasa (RFC 0036 — pendiente migrar a RFC 0453):**

```
Issuer                                         Holder
     │                                              │
     │  ACApyClient.send_credential_offer()         │
     │  POST /issue-credential/send-offer           │
     │  {cred_def_id, atributos, auto_issue:true}   │
     │──── OFFER ──────────────────────────────────►│
     │                           [offer_received]   │
     │                                              │
     │                    --auto-respond-credential-offer
     │                    Holder genera blind link secret
     │                    (prueba de que posee el wallet)
     │◄─── REQUEST ────────────────────────────────│
     │  [request_received]                          │
     │                                              │
     │  auto_issue: true                            │
     │  Issuer firma credencial con clave CL privada│
     │──── ISSUE ───────────────────────────────────►│
     │                           [credential_received]
     │                    --auto-store-credential   │
     │                    verifica firma contra ledger
     │                    almacena en wallet Askar  │
     │                           [done] ✅          │
```

**En Django** (`user_credential_service.py:29-33`): crea un registro `CredentialIssuance` con el `credential_exchange_id`, estado y atributos.

---

## Mapa completo de archivos

```
SSI_Project/
  manage                      ← arranca/detiene VON Network
  docker-compose.yml          ← define los 4 nodos + webserver

SSI_App/
  credential_test/
    docker-compose-agents.yml ← define qué agentes existen y sus puertos/flags
    genesis/genesis.txn       ← conecta los agentes al ledger correcto
    connect_and_issue.py      ← script standalone que ejecuta el flujo completo
    start_test.sh             ← arranca agentes y espera que estén listos
                                 (⚠ falta: automatizar registro del DID del issuer)

  aca_py/client.py            ← HTTP puro: un método por endpoint Admin API
                                 no tiene lógica de negocio

  services/
    credential_service.py     ← maneja schema, cred def y envío de oferta
                                 (⚠ schema hardcodeado a user_credential)
    wallet_service.py         ← registra en Django qué agente corresponde
                                 a cada entidad; crea conexiones
    user_credential_service.py ← orquestador: wallet + conexión + credencial + BD

  auth_app/models.py          ← Wallet, Connection, CredentialIssuance
                                 (estado persistido en SQLite)
  wallet/settings.py          ← ACA_PY_AGENTS: mapa agent_key → URL Admin API
```

---

## Dónde vive cada dato

| Dato | Dónde | Se pierde con |
|---|---|---|
| Transacciones del ledger (schemas, cred defs, DIDs) | Volúmenes `von_node*` | `./manage down` |
| Clave privada del issuer + DID local | Volumen del contenedor `acapy-issuer` | `docker compose down` |
| Wallets de los holders | Volúmenes de cada contenedor holder | `docker compose down` |
| Registros Django (Wallet, Connection, CredentialIssuance) | `SSI_App/db.sqlite3` | Si se borra el archivo |
| Credenciales almacenadas en holder | Wallet Askar del contenedor holder | `docker compose down` |

---

## Pendientes identificados

| # | Qué | Archivo |
|---|---|---|
| 1 | Automatizar registro del DID del issuer en `start_test.sh` | `credential_test/start_test.sh` |
| 2 | Migrar RFC 0160 → RFC 0023 (DID Exchange) | `connect_and_issue.py`, `client.py` |
| 3 | Migrar RFC 0036 → RFC 0453 (Issue Credential 2.0) | `connect_and_issue.py`, `client.py` |
| 4 | Refactorizar `credential_service.py` para soportar múltiples schemas | `services/credential_service.py` |
| 5 | Probar flujo con `user_credential`, `evtol_credential`, `vertiport_credential` | scripts nuevos |
