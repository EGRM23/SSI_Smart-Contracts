# Cómo ejecutar la prueba SSI completa

Este documento describe cómo arrancar el sistema SSI y correr la prueba integral
que cubre los 4 agentes holders (usuario, eVTOL, vertiport×2).

---

## Requisitos previos

- Docker y Docker Compose instalados
- Python 3 + venv de `SSI_App/` activado
- Al menos 5 GB libres en disco: `df -h /`

```bash
source ~/Documentos/TI3/SSI_App/venv/bin/activate
```

---

## Arranque del sistema (3 terminales)

### Terminal 1 — VON Network (ledger Indy)

```bash
cd ~/Documentos/TI3/SSI_Project
./manage start --wait
```

Espera a que aparezca `Ledger started`. Verifica los 4 nodos activos:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | grep von
```

Ledger browser disponible en: http://localhost:9000

---

### Terminal 2 — Agentes ACA-Py

`start.sh` hace todo en un paso: registra el DID del issuer en el ledger
(idempotente — funciona tanto si ya estaba registrado como si no) y levanta los 5 agentes.

```bash
cd ~/Documentos/TI3/SSI_App/agents
bash start.sh
```

El script:
1. Descarga el genesis file si no existe
2. Llama a `POST /register` en la VON Network con el DID del issuer
3. Levanta los 5 contenedores ACA-Py con `docker compose up -d`
4. Espera a que los 5 admin APIs respondan antes de terminar

Verifica que todos estén `Up` al finalizar:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | grep acapy
```

Resultado esperado:

```
acapy-issuer       Up X seconds
acapy-user1        Up X seconds
acapy-evtol1       Up X seconds
acapy-vertiport1   Up X seconds
acapy-vertiport2   Up X seconds
```

Si alguno aparece en `Restarting`, el ledger aún no estaba listo cuando arrancó.
Solución: `docker restart <nombre-del-contenedor>`

---

### Terminal 3 — Django

```bash
cd ~/Documentos/TI3/SSI_App
source venv/bin/activate
python manage.py runserver
```

API disponible en: http://localhost:8000

---

## Ejecutar la prueba integral

Con los 3 componentes corriendo, desde cualquier terminal:

```bash
cd ~/Documentos/TI3/SSI_App
python agents/scripts/test_all.py
```

### Qué hace `test_all.py`

El script prueba el sistema en 4 secciones secuenciales:

**Sección 1 — Flujo Django completo (usuario)**
Pasa por toda la lógica real de la aplicación Django:
1. `POST /api/user/` → crea usuario en SQLite + registra su wallet en el agente user1
2. `POST /api/auth/login/` → autenticación por sesión (cookies)
3. `POST /api/credentials/issue/` → emite una `user_credential:2.0` al wallet del usuario

Cada corrida usa un usuario con sufijo aleatorio (`testuser_XXXXXX`), así que es repetible sin limpiar la BD.

**Sección 2 — Credencial eVTOL (directo a ACA-Py)**
Emite `evtol_credential:3.0` al agente `acapy-evtol1` (admin: 8051) usando
RFC 0023 (OOB) + RFC 0453 (Issue Credential 2.0). Bypasea Django porque
el sistema todavía no tiene endpoints REST para entidades eVTOL.

**Secciones 3 y 4 — Credenciales Vertiport (directo a ACA-Py)**
Emite `vertiport_credential:4.0` a `acapy-vertiport1` (8061) y `acapy-vertiport2` (8071).
Misma mecánica que la sección 2.

### Salida esperada (sistema OK)

```
  SECCIÓN 1 — Flujo Django: Registro + Credencial de Usuario (user1)
  Usuario: testuser_c88e4a
  ✅ Usuario creado (id=10)
  ✅ Login OK
  ✅ Credencial emitida (cred_ex_id=19da0b05...)

  SECCIÓN 2 — Credencial eVTOL → evtol1 (admin: 8051)
  ✅ cred_def_id=Utwqp5cpEATQpGZL5WSQZJ:3:CL:8:default

  SECCIÓN — Credencial Vertiport → vertiport1 (admin: 8061)
  ✅ cred_def_id=Utwqp5cpEATQpGZL5WSQZJ:3:CL:10:default

  SECCIÓN — Credencial Vertiport → vertiport2 (admin: 8071)
  ✅ cred_def_id=Utwqp5cpEATQpGZL5WSQZJ:3:CL:10:default

  RESUMEN
  Holder              Schema   Estado
  user1 (Django)      2.0      ✅
  evtol1              3.0      ✅
  vertiport1          4.0      ✅
  vertiport2          4.0      ✅

  Sistema SSI operativo — todos los agentes tienen credenciales.
```

---

## Bridge SSI → Besu (flujo completo extremo a extremo)

Una vez que `test_all.py` termina con ✅ en los 4 agentes, ejecuta el bridge para
registrar las entidades en los contratos Besu y realizar un viaje completo.

### Terminal 4 — Red Besu

```bash
cd ~/Documentos/TI3/BESU_project
./run.sh                   # levanta la red (puede tardar ~1 min)
```

Verifica que el RPC responde:

```bash
curl -s -X POST http://localhost:8545 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

### Deploy de contratos

```bash
cd ~/Documentos/TI3/BESU_project/smart_contracts
npm install        # solo la primera vez
npm run compile    # compilar .sol → .json
node scripts/deploy_system.js   # despliega y escribe deployed_addresses.json
```

### Ejecutar el bridge

```bash
cd ~/Documentos/TI3/SSI_App
source venv/bin/activate
python bridge/bridge.py
```

El bridge hace 3 fases:

1. **Fase 1** — Lee las credenciales almacenadas en los wallets ACA-Py (user1, evtol1, vertiport1, vertiport2)
2. **Fase 2** — Registra las entidades en Besu:
   - `UserVerification.setRiderPermission(rider, can_ride)` — autoriza al usuario
   - `VertiportManagement.registerVertiport(...)` — registra los 2 vertiports con su capacidad real
   - `EVTOLManagement.registerEVTOL(...)` — registra el eVTOL en su vertiport de origen
3. **Fase 3** — Ejecuta un viaje completo vía `FlightReservation`:
   - `createReservation(...)` → estado CONFIRMED
   - `startTrip(...)` → estado IN_PROGRESS
   - `completeTrip(...)` → estado COMPLETED

Salida esperada:
```
  FASE 1 — Leyendo credenciales de los agentes ACA-Py
  ✅ user1: can_ride=true
  ✅ evtol1: id_puerto=p1, name=Eagle-XXXXXX
  ✅ vertiport1: id=vp1-XXXX, capacity=10
  ✅ vertiport2: id=vp2-XXXX, capacity=8

  FASE 2 — Registrando entidades en Besu
  ✅ Usuario autorizado (can_ride=True)
  ✅ Vertiport1 registrado (vp1-XXXX)
  ✅ Vertiport2 registrado (vp2-XXXX)
  ✅ eVTOL registrado (id=1)
  ✅ Parking pre-ocupado en p1

  FASE 3 — Ejecutando viaje vía FlightReservation
  ✅ Reserva creada — status=1 (CONFIRMED)
  ✅ Viaje iniciado — status=2 (IN_PROGRESS)
  ✅ Viaje completado — status=3 (COMPLETED)

  RESULTADO
  Trip ID  : TRIP-BRIDGE-XXXXXXXX
  Estado   : COMPLETED ✅
```

> **Nota:** el bridge usa los atributos reales de las credenciales SSI como `bytes` en los
> contratos (`userCredential`, `evtolCredential`, `vertiportCredential`). La verificación
> on-chain sigue mockeada (`return true`), pero los bytes ya contienen datos reales —
> cuando se implemente la verificación real en Solidity, el bridge no cambia.

---

## Pruebas individuales por tipo de credencial

Si quieres emitir solo un tipo específico sin pasar por Django:

```bash
cd ~/Documentos/TI3/SSI_App

# Usuario (directo al agente, sin Django)
python agents/scripts/connect_and_issue.py \
  --holder http://localhost:8041 \
  --schema-name user_credential --schema-version 2.0 \
  --attributes '{"nombres":"Ana","apellidos":"López","fecha_nacimiento":"1995-05-01","can_ride":"true"}'

# eVTOL
python agents/scripts/connect_and_issue.py \
  --holder http://localhost:8051 \
  --schema-name evtol_credential --schema-version 3.0 \
  --attributes '{"id_puerto":"p1","state":"ACTIVE","version":"v1","name":"EVTOL-1","can_fly":"true"}'

# Vertiport 1
python agents/scripts/connect_and_issue.py \
  --holder http://localhost:8061 \
  --schema-name vertiport_credential --schema-version 4.0 \
  --attributes '{"id_vertiport":"vp1","name":"Vertiport Norte","location":"Lima","capacity":"10","state":"ACTIVE"}'

# Vertiport 2
python agents/scripts/connect_and_issue.py \
  --holder http://localhost:8071 \
  --schema-name vertiport_credential --schema-version 4.0 \
  --attributes '{"id_vertiport":"vp2","name":"Vertiport Sur","location":"Lima","capacity":"8","state":"ACTIVE"}'
```

---

## Verificación manual de wallets

```bash
# Credenciales almacenadas en cada agente
curl -s http://localhost:8041/credentials | python3 -m json.tool   # user1
curl -s http://localhost:8051/credentials | python3 -m json.tool   # evtol1
curl -s http://localhost:8061/credentials | python3 -m json.tool   # vertiport1
curl -s http://localhost:8071/credentials | python3 -m json.tool   # vertiport2

# Schemas y cred defs registrados por el issuer
curl -s http://localhost:8031/schemas/created | python3 -m json.tool
curl -s http://localhost:8031/credential-definitions/created | python3 -m json.tool
```

---

## Apagar el sistema

Elige una de las dos opciones según lo que necesites la próxima sesión.

---

> ⚠️ **Regla de oro:** el ledger Indy y los wallets de los agentes ACA-Py son un
> par inseparable. Si reseteas el ledger (`./manage down`) pero conservas los wallets,
> los agentes seguirán creyendo que sus schemas existen en el ledger anterior y fallarán
> al emitir credenciales (`Could not get schema from ledger for seq no X`).
> **Siempre resetea ambos juntos, o conserva ambos juntos.**

### Opción A — Conservar wallets (reinicio rápido)

Detiene los contenedores sin borrar nada. La próxima sesión arranca en segundos
y no hay que volver a registrar el DID del issuer ni los schemas.

```bash
# 1. Agentes ACA-Py
cd ~/Documentos/TI3/SSI_App/agents
docker compose stop

# 2. Ledger Indy
cd ~/Documentos/TI3/SSI_Project
./manage stop
```

**Cómo reanudar la próxima vez:**

```bash
# Ledger primero
cd ~/Documentos/TI3/SSI_Project
./manage start --wait

# Luego los agentes — usa 'start', NO 'bash start.sh'
cd ~/Documentos/TI3/SSI_App/agents
docker compose start

# Django (Terminal 3 de siempre)
cd ~/Documentos/TI3/SSI_App
source venv/bin/activate
python manage.py runserver
```

> **Por qué `docker compose start` y no `bash start.sh`:**
> `start.sh` intenta crear los contenedores de nuevo (`up -d`), pero como ya existen
> (están parados, no eliminados), Docker devuelve un error de conflicto de nombre.
> `docker compose start` los retoma directamente sin recrearlos.

---

### Opción B — Reset completo (sesión limpia)

Borra contenedores, volúmenes (wallets ACA-Py) y el ledger Indy.
La próxima sesión arranca desde cero como si fuera la primera vez.

```bash
# Agentes ACA-Py + wallets
cd ~/Documentos/TI3/SSI_App/agents
bash start.sh --down

# Ledger Indy + datos
cd ~/Documentos/TI3/SSI_Project
./manage down
```

**Cómo reanudar la próxima vez:** seguir el arranque normal desde el paso 1
(Terminal 1 → Terminal 2 con `bash start.sh` → Terminal 3).

Los schemas se recrean solos al primera emisión de credencial.

---

## Mapa de puertos

| Componente | Puerto inbound | Puerto admin/web |
|---|---|---|
| VON Network (ledger browser) | — | 9000 |
| ACA-Py issuer | 8030 | 8031 |
| ACA-Py user1 | 8040 | 8041 |
| ACA-Py evtol1 | 8050 | 8051 |
| ACA-Py vertiport1 | 8060 | 8061 |
| ACA-Py vertiport2 | 8070 | 8071 |
| Django REST API | — | 8000 |
| Besu RPC (cuando aplique) | — | 8545 |

---

## Errores comunes

| Error | Causa | Solución |
|---|---|---|
| `acapy-issuer` en `Restarting` | Ledger no estaba listo al arrancar | `docker restart acapy-issuer` |
| `Connection refused` en port 8000 | Django no está corriendo | `python manage.py runserver` |
| `Pool timeout` en logs del issuer | VON Network caída | `./manage start --wait` en SSI_Project |
| `password_confirm required` en POST /api/user/ | Falta campo en el request | Enviar `password_confirm` igual a `password` |
| Schema/cred_def no encontrado | DID del issuer no registrado | Ejecutar el curl de registro del paso 2a |
| `Conflict: container name "/acapy-X" already in use` | Sesión anterior cerrada con `stop` y se volvió a correr `bash start.sh` | Usar `docker compose start` en vez de `bash start.sh` (ver Opción A de apagado) |
| `Could not get schema from ledger for seq no X` | Ledger reseteado pero wallets de agentes conservados (desincronización) | Reset completo: `bash start.sh --down` + `./manage down`, luego arranque normal |
