# Pruebas del sistema — TI3

Catálogo de todas las pruebas que se pueden ejecutar sobre el sistema, organizadas
por tipo. Cada sección describe qué se verifica, cómo ejecutarlo y qué resultado
esperar.

El sistema completo tiene que estar corriendo para las pruebas de integración y
de seguridad. Las pruebas unitarias no necesitan nada externo.

---

## 1. Pruebas unitarias (Django) — sin infraestructura

**Qué verifican:** que la lógica de la API Django funciona correctamente de forma
aislada, sin ACA-Py ni VON Network. Usan mocks para simular los agentes.

**Cómo ejecutar:**

```bash
cd ~/Documentos/TI3/SSI_App
source venv/bin/activate
python manage.py test auth_app
```

**Qué cubre (22 tests):**

| Test class | Qué verifica |
|---|---|
| `UserModelTest` | Creación de usuarios en la BD |
| `EvtolModelTest` | Creación de eVTOLs en la BD |
| `VertiportModelTest` | Creación de vertiports en la BD |
| `WalletPolymorphicTest` | Un wallet puede pertenecer a User, EVTOL o Vertiport |
| `UserEndpointTest` | `POST /api/user/` crea usuario + wallet (mockeado) |
| `EvtolEndpointTest` | `POST /api/evtol/` crea eVTOL + wallet (mockeado) |
| `VertiportEndpointTest` | `POST /api/vertiport/` crea vertiport + wallet (mockeado) |
| `LoginEndpointTest` | `POST /api/auth/login/` autentica correctamente |

**Resultado esperado:**

```
Ran 22 tests in X.XXXs
OK
```

---

## 2. Prueba de integración SSI — flujo completo de credenciales

**Qué verifica:** que los 4 agentes ACA-Py reciben sus credenciales correctamente
a través de la API Django (registro + emisión real con el ledger Indy).

**Prerequisitos:** VON Network + ACA-Py agents + Django corriendo.

**Cómo ejecutar:**

```bash
cd ~/Documentos/TI3/SSI_App
source venv/bin/activate
python agents/scripts/test_all.py
```

**Qué hace:**

1. Registra un usuario por `POST /api/user/` → emite `user_credential` → wallet user1
2. Registra un eVTOL por `POST /api/evtol/` → emite `evtol_credential` → wallet evtol1
3. Registra vertiport1 por `POST /api/vertiport/` → emite `vertiport_credential`
4. Registra vertiport2 por `POST /api/vertiport/` → emite `vertiport_credential`

**Resultado esperado:**

```
Holder              Schema   Estado
user1 (Django)      2.0      ✅
evtol1              3.0      ✅
vertiport1          4.0      ✅
vertiport2          4.0      ✅

Sistema SSI operativo — todos los agentes tienen credenciales.
```

---

## 3. Prueba de integración bridge SSI → Besu — flujo extremo a extremo

**Qué verifica:** el flujo completo desde credenciales SSI hasta un viaje registrado
en la blockchain, incluyendo la verificación criptográfica con `ecrecover`.

**Prerequisitos:** todo el sistema corriendo (VON Network + ACA-Py + Django + Besu)
y los contratos desplegados. Los 4 agentes deben tener credenciales (ejecutar primero
`test_all.py`).

**Cómo ejecutar:**

```bash
# Desplegar contratos si aún no se hizo
cd ~/Documentos/TI3/BESU_project/smart_contracts
node scripts/deploy_system.js

# Ejecutar el bridge
cd ~/Documentos/TI3/SSI_App
source venv/bin/activate
python bridge/bridge.py
```

**Qué verifica:**

- Fase 1: credenciales leídas de los 4 wallets ACA-Py
- Fase 2: atestaciones firmadas por Django y aceptadas por los contratos (`ecrecover`)
- Fase 3: viaje completo CONFIRMED → IN_PROGRESS → COMPLETED en Besu

**Resultado esperado:** `Estado: COMPLETED ✅`

---

## 4. Pruebas de seguridad — contratos Besu

Verifican que los contratos rechazan operaciones inválidas o no autorizadas.
No requieren un framework especial — son scripts que intentan hacer algo incorrecto
y comprueban que la transacción revierte.

**Prerequisitos:** Besu corriendo + contratos desplegados.

### 4.1 Registro sin atestación (debe revertir)

Intenta registrar un vertiport sin la firma del Trusted Verifier. El contrato debe rechazarlo.

```bash
cd ~/Documentos/TI3/BESU_project/smart_contracts
node - <<'EOF'
const Web3 = require("web3");
const fs = require("fs");
const path = require("path");

const w3 = new Web3("http://localhost:8545");
const KEY = "0x8f2a55949038a9610f50fb23b5883af3b4ecb3c3bb792cbcefbd1542c692be63";
const account = w3.eth.accounts.privateKeyToAccount(KEY);

const addresses = JSON.parse(fs.readFileSync(
  path.resolve(__dirname, "deployed_addresses.json")
));
const abi = JSON.parse(fs.readFileSync(
  path.resolve(__dirname, "contracts/VertiportManagement.json")
)).abi;

const contract = new w3.eth.Contract(abi, addresses.VertiportManagement);

// Firma vacía — debe revertir
contract.methods.registerVertiport("vp-test", 1, 5, "0x", "0x").send({
  from: account.address, gas: 200000
}).then(() => {
  console.log("❌ ERROR: debió revertir y no lo hizo");
}).catch(e => {
  console.log("✅ Correcto: transacción revirtió —", e.message.slice(0, 80));
});
EOF
```

**Resultado esperado:** `✅ Correcto: transacción revirtió`

### 4.2 Atestación con firma incorrecta (debe revertir)

```bash
cd ~/Documentos/TI3/BESU_project/smart_contracts
node - <<'EOF'
const Web3 = require("web3");
const fs = require("fs"), path = require("path");

const w3 = new Web3("http://localhost:8545");
const KEY = "0x8f2a55949038a9610f50fb23b5883af3b4ecb3c3bb792cbcefbd1542c692be63";
const account = w3.eth.accounts.privateKeyToAccount(KEY);

const addresses = JSON.parse(fs.readFileSync(path.resolve(__dirname, "deployed_addresses.json")));
const abi = JSON.parse(fs.readFileSync(path.resolve(__dirname, "contracts/VertiportManagement.json"))).abi;
const contract = new w3.eth.Contract(abi, addresses.VertiportManagement);

// Firma de 65 bytes pero inválida (basura)
const fakeSig = "0x" + "ab".repeat(65);

contract.methods.registerVertiport("vp-fake", 1, 5, "0x", fakeSig).send({
  from: account.address, gas: 200000
}).then(() => {
  console.log("❌ ERROR: aceptó una firma falsa");
}).catch(e => {
  console.log("✅ Correcto: firma inválida rechazada —", e.message.slice(0, 80));
});
EOF
```

**Resultado esperado:** `✅ Correcto: firma inválida rechazada`

### 4.3 Doble reserva del mismo eVTOL (debe revertir)

El eVTOL pasa a estado `EXPECTING` al asignarle un viaje. Un segundo
`createReservation` con el mismo eVTOL debe fallar porque `isAvailable()` devuelve false.

```python
# Ejecutar en un script Python con el bridge ya corrido una vez
# y el eVTOL en estado CONFIRMED (no iniciar el viaje aún)

from web3 import Web3
from web3.middleware import geth_poa_middleware
import json
from pathlib import Path

BASE = Path("~/Documentos/TI3").expanduser()
addresses = json.loads((BASE / "BESU_project/smart_contracts/deployed_addresses.json").read_text())
abi = json.loads((BASE / "BESU_project/smart_contracts/contracts/FlightReservation.json").read_text())["abi"]

w3 = Web3(Web3.HTTPProvider("http://localhost:8545"))
w3.middleware_onion.inject(geth_poa_middleware, layer=0)
account = w3.eth.account.from_key("0x8f2a55949038a9610f50fb23b5883af3b4ecb3c3bb792cbcefbd1542c692be63")
fr = w3.eth.contract(address=Web3.to_checksum_address(addresses["FlightReservation"]), abi=abi)

try:
    nonce = w3.eth.get_transaction_count(account.address)
    tx = fr.functions.createReservation(
        "TRIP-DOUBLE-TEST", account.address,
        "vp1-3687", "vp2-4142", 1,
        b"user", b"origin", b"evtol"
    ).build_transaction({"from": account.address, "nonce": nonce, "gas": 300000, "gasPrice": w3.eth.gas_price})
    signed = account.sign_transaction(tx)
    txh = w3.eth.send_raw_transaction(signed.rawTransaction)
    receipt = w3.eth.wait_for_transaction_receipt(txh)
    if receipt.status == 0:
        print("✅ Correcto: doble reserva rechazada (revertió)")
    else:
        print("❌ ERROR: doble reserva aceptada — revisar lógica del contrato")
except Exception as e:
    print("✅ Correcto: doble reserva rechazada —", str(e)[:80])
```

### 4.4 Viaje sin autorización de usuario (debe revertir)

`createReservation` verifica internamente `canUserRide(rider)`. Si el rider no tiene
permiso, la transacción revierte.

```bash
# Usar una dirección cualquiera que no esté autorizada
python3 - <<'EOF'
from web3 import Web3
from web3.middleware import geth_poa_middleware
import json
from pathlib import Path

BASE = Path("~/Documentos/TI3").expanduser()
addresses = json.loads((BASE / "BESU_project/smart_contracts/deployed_addresses.json").read_text())
abi_fr = json.loads((BASE / "BESU_project/smart_contracts/contracts/FlightReservation.json").read_text())["abi"]

w3 = Web3(Web3.HTTPProvider("http://localhost:8545"))
w3.middleware_onion.inject(geth_poa_middleware, layer=0)
account = w3.eth.account.from_key("0x8f2a55949038a9610f50fb23b5883af3b4ecb3c3bb792cbcefbd1542c692be63")
fr = w3.eth.contract(address=Web3.to_checksum_address(addresses["FlightReservation"]), abi=abi_fr)

UNAUTHORIZED = "0x0000000000000000000000000000000000000001"  # nunca autorizado

try:
    nonce = w3.eth.get_transaction_count(account.address)
    tx = fr.functions.createReservation(
        "TRIP-UNAUTH", UNAUTHORIZED,
        "vp1-3687", "vp2-4142", 1, b"u", b"o", b"e"
    ).build_transaction({"from": account.address, "nonce": nonce, "gas": 300000, "gasPrice": w3.eth.gas_price})
    signed = account.sign_transaction(tx)
    txh = w3.eth.send_raw_transaction(signed.rawTransaction)
    receipt = w3.eth.wait_for_transaction_receipt(txh)
    if receipt.status == 0:
        print("✅ Correcto: usuario no autorizado rechazado")
    else:
        print("❌ ERROR: aceptó un rider sin permiso")
except Exception as e:
    print("✅ Correcto:", str(e)[:80])
EOF
```

---

## 5. Pruebas de tolerancia a fallas

Verifican que el sistema sigue funcionando cuando un componente falla.
La red Besu usa QBFT: con 4 validadores tolera 1 caído. La red Indy también tolera 1 nodo caído de 4.

### 5.1 Caída de un nodo Besu validator

```bash
# Derribar un validador
docker stop besu_project-validator1-1

# Verificar que la red sigue procesando bloques
sleep 5
curl -s -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
# El número de bloque debe seguir incrementando

# Ejecutar el bridge para confirmar que las transacciones siguen funcionando
cd ~/Documentos/TI3/SSI_App && python bridge/bridge.py

# Restaurar el nodo
docker start besu_project-validator1-1
```

**Resultado esperado:** el bridge completa con ✅ incluso con un validator caído.

**Límite:** si caen 2 de 4 validadores, el consenso QBFT se detiene (necesita ≥ 3 nodos).

### 5.2 Caída de un nodo Indy

```bash
# Derribar un nodo del ledger
docker stop von-node1-1

# Emitir una credencial (debe seguir funcionando con 3 de 4 nodos)
cd ~/Documentos/TI3/SSI_App
python agents/scripts/connect_and_issue.py \
  --holder http://localhost:8041 \
  --schema-name user_credential --schema-version 2.0 \
  --attributes '{"nombres":"Test","apellidos":"Falla","fecha_nacimiento":"2000-01-01","can_ride":"true"}'

# Restaurar
docker start von-node1-1
```

**Resultado esperado:** la emisión de credenciales funciona con 3 nodos Indy.

### 5.3 Caída del agente issuer durante la emisión

```bash
# Iniciar emisión en background y derribar el issuer a mitad
python agents/scripts/test_all.py &
sleep 5
docker stop acapy-issuer

# Observar el error en test_all.py (timeout esperado)
wait

# Verificar qué credentials quedaron incompletas
curl -s http://localhost:8041/credentials | python3 -m json.tool

# Restaurar
docker start acapy-issuer
```

**Resultado esperado:** `test_all.py` falla con timeout. Al reiniciar el issuer, se puede re-ejecutar y completar la emisión.

### 5.4 Caída de Django mientras el bridge está corriendo

```bash
# Correr el bridge en background
python bridge/bridge.py &
BRIDGE_PID=$!

# Matar Django después de Fase 1 (cuando el bridge pide atestaciones)
sleep 8 && kill $(lsof -ti:8000)

# Observar el error del bridge
wait $BRIDGE_PID

# Resultado esperado: error claro en la fase de atestación,
# ningún contrato queda en estado inconsistente (las transacciones
# Besu son atómicas — o se completan o no ocurren)
```

---

## 6. Pruebas de escalabilidad (Locust)

Miden cuántas operaciones por segundo puede procesar el sistema antes de degradarse.

### Instalación

```bash
cd ~/Documentos/TI3/SSI_App
source venv/bin/activate
pip install locust
```

### Escenario 1 — Registro masivo de usuarios en Django

Crea el archivo `locustfile_users.py`:

```python
from locust import HttpUser, task, between
import random, string

class RegistroUsuario(HttpUser):
    wait_time = between(0.5, 1.5)

    @task
    def registrar_usuario(self):
        sufijo = ''.join(random.choices(string.ascii_lowercase + string.digits, k=8))
        self.client.post("/api/user/", json={
            "username":         f"load_{sufijo}",
            "email":            f"load_{sufijo}@test.com",
            "password":         "test1234",
            "password_confirm": "test1234",
        })
```

```bash
locust -f locustfile_users.py --host=http://localhost:8000 \
       --users 50 --spawn-rate 5 --run-time 60s --headless \
       --html reporte_usuarios.html
```

**Qué medir:** throughput (req/s), latencia p50/p95/p99, tasa de error.
**Cuello de botella esperado:** Django + SQLite (single-writer lock).

### Escenario 2 — Emisión masiva de credenciales

```python
from locust import HttpUser, task, between
import random, string

class EmisionCredencial(HttpUser):
    wait_time = between(1, 3)

    def on_start(self):
        # Registrar y autenticar un usuario al inicio de cada worker
        sufijo = ''.join(random.choices(string.ascii_lowercase, k=6))
        self.client.post("/api/user/", json={
            "username": f"lc_{sufijo}", "email": f"lc_{sufijo}@t.com",
            "password": "test1234", "password_confirm": "test1234",
        })
        self.client.post("/api/auth/login/", json={
            "username": f"lc_{sufijo}", "password": "test1234"
        })

    @task
    def emitir_credencial(self):
        self.client.post("/api/credentials/issue/")
```

```bash
locust -f locustfile_credentials.py --host=http://localhost:8000 \
       --users 10 --spawn-rate 2 --run-time 60s --headless \
       --html reporte_credenciales.html
```

**Cuello de botella esperado:** ACA-Py (procesa una conexión OOB a la vez por agente)
y el ledger Indy (escritura de schema/cred_def toma 1-2 segundos).

### Escenario 3 — Reservas concurrentes en Besu

```python
# locustfile_besu.py
# Requiere: contratos desplegados, entidades ya registradas
from locust import User, task, between, events
from web3 import Web3
from web3.middleware import geth_poa_middleware
import json, uuid, time
from pathlib import Path

BASE = Path("~/Documentos/TI3").expanduser()
addresses = json.loads((BASE / "BESU_project/smart_contracts/deployed_addresses.json").read_text())
abi = json.loads((BASE / "BESU_project/smart_contracts/contracts/FlightReservation.json").read_text())["abi"]
KEY = "0x8f2a55949038a9610f50fb23b5883af3b4ecb3c3bb792cbcefbd1542c692be63"

class BesuUser(User):
    wait_time = between(0.1, 0.5)

    def on_start(self):
        self.w3 = Web3(Web3.HTTPProvider("http://localhost:8545"))
        self.w3.middleware_onion.inject(geth_poa_middleware, layer=0)
        self.account = self.w3.eth.account.from_key(KEY)
        self.fr = self.w3.eth.contract(
            address=Web3.to_checksum_address(addresses["FlightReservation"]), abi=abi
        )

    @task
    def crear_reserva(self):
        trip_id = f"LOAD-{uuid.uuid4().hex[:8].upper()}"
        start = time.time()
        try:
            nonce = self.w3.eth.get_transaction_count(self.account.address)
            tx = self.fr.functions.createReservation(
                trip_id, self.account.address,
                "vp1-3687", "vp2-4142", 1, b"u", b"o", b"e"
            ).build_transaction({
                "from": self.account.address, "nonce": nonce,
                "gas": 300000, "gasPrice": self.w3.eth.gas_price
            })
            signed = self.account.sign_transaction(tx)
            txh = self.w3.eth.send_raw_transaction(signed.rawTransaction)
            receipt = self.w3.eth.wait_for_transaction_receipt(txh)
            elapsed = int((time.time() - start) * 1000)
            status = "success" if receipt.status == 1 else "failure"
            events.request.fire(
                request_type="ETH", name="createReservation",
                response_time=elapsed, response_length=0,
                exception=None if status == "success" else Exception("revert")
            )
        except Exception as e:
            elapsed = int((time.time() - start) * 1000)
            events.request.fire(
                request_type="ETH", name="createReservation",
                response_time=elapsed, response_length=0, exception=e
            )
```

```bash
locust -f locustfile_besu.py \
       --users 20 --spawn-rate 5 --run-time 30s --headless \
       --html reporte_besu.html
```

**Cuello de botella esperado:** Besu tiene un block time de ~2s, pero el nonce
debe ser único por transacción desde la misma cuenta. Las reservas concurrentes
desde la misma cuenta necesitarán nonces secuenciales.

---

## 7. Resumen — qué prueba qué

| Prueba | Herramienta | Requiere infraestructura | Tiempo estimado |
|---|---|---|---|
| Tests unitarios Django | `manage.py test` | No | < 30 s |
| Integración SSI (test_all.py) | Script Python | SSI completo | 1-2 min |
| Integración bridge | Script Python | Sistema completo | 1-2 min |
| Seguridad — sin atestación | Script Node.js | Besu | < 1 min |
| Seguridad — firma falsa | Script Node.js | Besu | < 1 min |
| Seguridad — doble reserva | Script Python | Besu | < 1 min |
| Seguridad — sin autorización | Script Python | Besu | < 1 min |
| Tolerancia — validator caído | Docker + bridge | Sistema completo | 5 min |
| Tolerancia — nodo Indy caído | Docker + script | SSI completo | 5 min |
| Escalabilidad — registro usuarios | Locust | Django | 5-10 min |
| Escalabilidad — credenciales | Locust | SSI completo | 10-15 min |
| Escalabilidad — reservas Besu | Locust | Besu | 5-10 min |
