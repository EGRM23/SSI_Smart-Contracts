# Sesión de Análisis — TI3 Sistema UAM con SSI + Blockchain
**Fecha:** 2026-04-16  
**Participantes:** Eduardo G. Ruiz Mamani, Claude (Sonnet 4.6)

---

## PARTE 1 — Análisis del Paper: `TI_Examen_3.pdf`

**Título:** *Identidad soberana y contratos inteligentes para la gestión de vehículos autónomos*  
**Autores:** Víctor A. Quicaño Miranda, Eduardo G. Ruiz Mamani  
**Institución:** UNSA, Enero 2026

---

### 1.1 Objetivo Principal

Proponer y validar un marco arquitectónico que integra **Self-Sovereign Identity (SSI)** y **contratos inteligentes** para la gestión de identidades e interacciones operacionales en escenarios de **Movilidad Aérea Urbana (UAM)**, combinando:

- **Hyperledger Indy** — ledger raíz de confianza (DIDs, schemas, cred-defs)
- **Hyperledger Aries / ACA-Py** — middleware de agentes, wallets, emisión de VCs
- **Hyperledger Besu** — blockchain permisionada, contratos inteligentes

**Problema raíz:** Los sistemas de identidad centralizados no escalan ni son interoperables en un entorno dinámico como el transporte aéreo urbano.

---

### 1.2 Arquitectura General

```
[Browser / Cliente]
       │
       ▼
Django REST API (SSI_App, puerto 8000)
  └── WalletService / CredentialService / DNIService
        └── ACApyClient → Agentes ACA-Py (issuer:9041, holder:9031)
              └── indy_vdr → Nodos Indy (SSI_Project, puertos 9701–9708)
                              ↑ Ledger browser en puerto 9000

[On-chain]
FlightReservation (BESU_project)
  └── llama a UserVerification, VertiportManagement, EVTOLManagement
        (verificación SSI aceptada como parámetro pero mockeada con return true)
RPC Besu: localhost:8545
```

---

### 1.3 Componentes del Paper

#### Capa 1 — Hyperledger Indy (Ledger de Identidad)
- Almacena DIDs, schemas de credenciales y Credential Definitions.
- Habilita minimización de datos, divulgación selectiva y relaciones pairwise.
- Implementado con `von-network`: red de 4 nodos + explorador web (puerto 9000).

#### Capa 2 — Hyperledger Aries / ACA-Py (Agentes y Comunicación)
- Gestiona wallets independientes por entidad (modelo multitenant).
- Emite credenciales via protocolo **Issue Credential 2.0**.
- Comunicación entre agentes via **DIDComm**.
- Cada entidad opera con token de autenticación propio.

**Esquemas de credenciales del paper:**

| Entidad     | Credencial            | Atributos clave                                      |
|-------------|-----------------------|------------------------------------------------------|
| Usuario     | `User_Cred v1.0`      | `first_name`, `last_name`, `can_ride`                |
| Aeronave    | `Evtol_Crede v3.0`    | `id_puerto`, `status`, `updates`                     |
| Vertipuerto | `Port_Credential v4.0`| `last_name`, `n_airstrip`, `n_parkings`, `coord_lon`, `coord_lat` |

#### Capa 3 — Hyperledger Besu (Contratos Inteligentes)
Red permisionada IBFT2/QBFT. Cuatro contratos en orden de dependencia:

| Contrato               | Responsabilidad                                                  |
|------------------------|------------------------------------------------------------------|
| `UserVerification`     | Mapa `address → canRide bool`; setter por issuer autorizado      |
| `VertiportManagement`  | Capacidad de pistas y parkings; delta-based updates              |
| `EVTOLManagement`      | Máquina de estados: `PARKED → EXPECTING → IN_USE → PARKED`       |
| `FlightReservation`    | Orquestador único; lifecyle de viaje `REQUESTED→CONFIRMED→IN_PROGRESS→COMPLETED` |

---

### 1.4 Evaluación de Viabilidad

#### Técnica — CONFIRMADA con reservas
| Aspecto                              | Estado                        |
|--------------------------------------|-------------------------------|
| Red Indy + ACA-Py multitenant        | Funcional                     |
| Emisión de credenciales              | Funcional (v1.0, no v2.0)     |
| Contratos inteligentes Besu          | Funcionales                   |
| **Integración criptográfica SSI↔Besu** | **NO implementada — mock**   |
| Revocación de credenciales           | No implementada               |

#### Económica — No evaluada
- Red Besu privada: sin costos de gas reales.
- `von-network` Docker: adecuado para prototipo, no para producción.

#### Operacional — Parcial
- Onboarding automatizado de entidades.
- Dependencia del orden de arranque (4 pasos secuenciales).
- Integración SSI↔Besu manual/mockeada.

---

### 1.5 Limitaciones y Trabajo Futuro (del Paper)
1. Verificación criptográfica directa de credenciales en contratos: pendiente.
2. Revocación de credenciales: no implementada.
3. Escalabilidad y latencia: solo probado localmente.
4. Bridge Aries/Indy → Besu: línea futura principal.

---

## PARTE 2 — Comparación Paper vs. Implementación Real

### 2.1 Estado por Sub-proyecto

#### SSI_Project/ — Hyperledger Indy
**Estado: Completo y funcional.**
- Red 4 nodos, ledger browser, scripts CLI.
- `server/anchor.py` (942 líneas): pool, registro NYM, TAA, transacciones.
- Equivale exactamente a lo descrito como "VON Network" en Fig. 2 del paper.

#### SSI_App/ — Aries + Django

**Lo que SÍ funciona (directo a ACA-Py, bypasa Django):**
- `credential_test/connect_and_issue.py`: emite `Evtol_Crede` de issuer a holder ✅
- `multitenant_test.ipynb`: múltiples wallets, DIDs, credenciales simultáneas ✅
- `test_credentials.ipynb`: registro de schemas/cred-defs en Indy ✅
- Los resultados de Figuras 4, 5 y 6 del paper se generaron desde aquí.

**Lo que NO funciona (la app Django):**
- `credential_views.py`: todo el código comentado (líneas 1–101)
- `wallet/urls.py`: solo activo `POST /api/user/`; rutas de credenciales comentadas
- `credential_service.py`: llama a `_make_request()` que no existe en `ACApyClient`
- `dni_service.py` línea 24: referencia a campo `user` que no existe en `CredentialIssuance`
- Modelo `Vertiport`: no existe en `models.py`
- Schema hardcodeado como `"DNI"` en lugar de los schemas del paper

**Conflicto menor:**
- `docker-compose-agents.yml`: vertiport1 y vertiport2 comparten puertos 8050/8051

#### BESU_project/ — Contratos Inteligentes
**Estado: Completo y funcional.**
- 4 contratos implementados y probados.
- `smoke_test_full_system.js`: 7 pasos end-to-end ejecutan correctamente.
- Mock documentado explícitamente: `return true` en todas las funciones de verificación.
- **Problema menor:** Direcciones de contratos hardcodeadas en líneas 9–12 del smoke test.

---

### 2.2 Matriz de Afirmaciones del Paper vs. Realidad

| Afirmación del Paper                         | Implementado | Dónde                          |
|----------------------------------------------|:------------:|-------------------------------|
| Red Indy de 4 nodos                          | ✅ Completo  | `SSI_Project/`                |
| Wallets independientes por entidad           | ✅ Completo  | `credential_test/` + `models.py` |
| Registro de DIDs en ledger (NYM)             | ✅ Completo  | `credential_test/` (Fig. 4)   |
| Schema eVTOL emitido                         | ✅           | `connect_and_issue.py`        |
| Schemas usuario y vertipuerto                | ❌           | Solo definidos, no emitidos   |
| Issue Credential **v2.0**                    | ⚠️ v1.0      | `credential_test/`            |
| Sistema multitenant                          | ✅ Funcional | `multitenant_test.ipynb`      |
| 4 contratos inteligentes en Besu             | ✅ Completo  | `BESU_project/smart_contracts/` |
| Flujo end-to-end de viaje (Fig. 7)           | ✅ Con mock  | `smoke_test_full_system.js`   |
| Django REST API con credenciales             | ❌ Roto      | Endpoints comentados          |
| Integración criptográfica SSI↔Besu           | ❌ No existe | `ssi-credential-system/` vacío |

**Avance estimado: ~70–75% de lo descrito en el paper.**

---

## PARTE 3 — Análisis del Sistema y Escenario Operacional

### 3.1 Funciones Detalladas por Componente

#### Hyperledger Indy
El ancla de confianza. Almacena solo material criptográfico público.

| Almacena                      | NO almacena              |
|-------------------------------|--------------------------|
| DIDs                          | Información personal     |
| Schemas (nombres de atributos)| Valores de credenciales  |
| Credential Definitions        | Claves privadas          |
| Registros de revocación       | Registros de vuelos      |

#### Hyperledger Aries / ACA-Py
Capa operacional de SSI.
- **Issuer Agent** (9041): emite credenciales a usuarios, eVTOLs, vertipuertos.
- **Holder Agents** (multitenant): almacenan credenciales recibidas, generan Verifiable Presentations.
- Ejecuta protocolos DIDComm y gestión de wallets/claves.

#### Contratos Inteligentes Besu

```
FlightReservation.sol          ← único punto de entrada
├── UserVerification.sol       ← address → canRide bool
├── EVTOLManagement.sol        ← máquina de estados por aeronave
└── VertiportManagement.sol    ← capacidad delta-based por vertipuerto
```

`createReservation()` ejecuta 3 checks atómicos:
1. `userVerification.canUserRide(rider)` → autorización
2. `vertiportManagement.checkLandingAvailability(origin)` → capacidad física
3. `evtolManagement.isAvailable(evtolId)` → disponibilidad de aeronave

Si cualquier check falla, la tx revierte sin cambios de estado.

---

### 3.2 Escenario: 3 Vertipuertos × 2 Pistas × 3 eVTOLs

#### Configuración Inicial

```
Vertipuertos:
  VP-A: 2 pistas, 4 parkings
  VP-B: 1 pista,  3 parkings
  VP-C: 2 pistas, 2 parkings

eVTOLs:
  E1 → PARKED en VP-A
  E2 → PARKED en VP-A
  E3 → PARKED en VP-B

Usuarios:
  Alice → canRide: true
  Bob   → canRide: true
  Carol → canRide: false
```

#### Viaje 1: Alice VP-A → VP-C en E1

```
createReservation("TRIP-001", Alice, VP-A, VP-C, E1)
  ① canUserRide(Alice) → true ✓
  ② checkLandingAvailability(VP-A) → true ✓
  ③ isAvailable(E1) → true ✓
  → E1: PARKED → EXPECTING
  → TRIP-001: CONFIRMED

[ajuste manual pre-vuelo]
  updateVertiportState(VP-A, 0, -1)
  → VP-A: parkings 3/4 libres

startTrip("TRIP-001")
  → E1: EXPECTING → IN_USE
  → VP-A parkingDelta = +1  (E1 despegó)
  → VP-A: parkings 4/4 libres
  → TRIP-001: IN_PROGRESS

completeTrip("TRIP-001")
  → E1: IN_USE → PARKED en VP-C
  → VP-C parkingDelta = -1  (E1 aterrizó)
  → VP-C: parkings 1/2 libres
  → TRIP-001: COMPLETED
```

#### Viaje 2: Bob VP-A → VP-B en E2 (concurrente con Viaje 1)

```
E2 es independiente → ejecuta sin conflicto:
  TRIP-002: CONFIRMED → IN_PROGRESS → COMPLETED
  E2: PARKED en VP-B
  VP-B: parkings 2/3 libres
```

#### Carol intenta reservar (falla esperada)

```
createReservation("TRIP-003", Carol, VP-B, VP-A, E3)
  ① canUserRide(Carol) → false ✗
  → REVERT: "Usuario no autorizado para reservar"
  → Sin cambios de estado en ningún contrato
```

#### Capacidad agotada en VP-C

```
[VP-C: parkings 0/2 libres — ambos ocupados]

createReservation("TRIP-004", Alice, VP-A, VP-C, ...)
  ② checkLandingAvailability(VP-C)
     → n_free_airstrip > 0 AND n_parkings_free > 0
     → 2 > 0 AND 0 > 0  →  false ✗
  → REVERT: "Sin capacidad en vertiport origen"
```

#### Estado Final

```
VP-A: pistas 2/2, parkings 2/4 libres  (E1 y E2 partieron)
VP-B: pistas 1/1, parkings 2/3 libres  (E2 llegó)
VP-C: pistas 2/2, parkings 1/2 libres  (E1 llegó)

E1: PARKED en VP-C
E2: PARKED en VP-B
E3: PARKED en VP-B

Todos los viajes registrados inmutablemente en Besu con timestamps y eventos.
```

---

## PARTE 4 — Gaps y Lista de Tareas Pendientes

### TIER 1 — Bloqueantes: El Sistema No Funciona End-to-End

**TASK-01: Implementar el Bridge SSI → Besu**
- Directorio `ssi-credential-system/` existe pero está vacío.
- Este servicio debe: recibir VP de Aries → verificar contra Indy → extraer atributos → llamar `setRiderPermission(address, bool)` en Besu.
- Sin esto, SSI y Besu son sistemas aislados que nunca se comunican.
- **Esfuerzo:** Alto.

**TASK-02: Corregir `credential_service.py`**
- `_make_request()` no existe en `ACApyClient` → reemplazar con `get()`/`post()`.
- Schema hardcodeado como `"DNI"` → parametrizar para los 3 tipos de entidad.
- Falta campo `tag` en la creación de Credential Definitions.
- **Esfuerzo:** Bajo.

**TASK-03: Corregir `dni_service.py` línea 24**
- `CredentialIssuance.objects.create(user=user, ...)` → debe ser `wallet=user.wallet`.
- `create_connection_for_user()` no existe en `WalletService`.
- **Esfuerzo:** Bajo.

**TASK-04: Descomentar y conectar endpoints de credenciales**
- `wallet/urls.py` líneas 10–13: descomentar rutas de credenciales.
- Conectar a la capa de servicios corregida.
- **Esfuerzo:** Bajo.

---

### TIER 2 — Gaps Funcionales

**TASK-05: Agregar modelo `Vertiport` en Django**
- No existe en `models.py` pese a ser requerido como entidad con wallet.
- Seguir el patrón del modelo `EVTOL` existente.
- **Esfuerzo:** Bajo.

**TASK-06: Implementar emisión de credenciales de Usuario y Vertipuerto**
- `connect_and_issue.py` solo cubre `Evtol_Crede`.
- Crear scripts equivalentes para `User_Cred v1.0` y `Port_Credential v4.0`.
- **Esfuerzo:** Medio.

**TASK-07: Persistir direcciones de contratos desplegados**
- `smoke_test_full_system.js` líneas 9–12: direcciones hardcodeadas que caducan con cada redeploy.
- `deploy_system.js` debe escribir un `deployed_addresses.json` que todos los scripts lean.
- **Esfuerzo:** Bajo.

**TASK-08: Corregir conflicto de puertos en `docker-compose-agents.yml`**
- `vertiport1` y `vertiport2` comparten puertos 8050/8051.
- Asignar puertos únicos a cada agente.
- **Esfuerzo:** Bajo.

---

### TIER 3 — Correcciones de Protocolo y Correctitud

**TASK-09: Actualizar a Issue Credential v2.0**
- El paper especifica v2.0; el código usa v1.0.
- Cambiar endpoints a `/issue-credential-2.0/send-offer`.
- **Esfuerzo:** Medio.

**TASK-10: Corregir lifecycle de pistas (airstrip) en contratos**
- `startTrip()` pasa `airstripDelta = 0` → las pistas nunca cambian de estado.
- Opciones: usar solo parkings para `checkLandingAvailability`, o agregar delta `-1/+1` en start/complete.
- **Esfuerzo:** Bajo.

**TASK-11: Agregar control de acceso en `EVTOLManagement` y `VertiportManagement`**
- `registerEVTOL()`, `assignToTrip()`, `registerVertiport()`, etc. no tienen `onlyAuthorized`.
- Cualquier address puede llamarlos directamente.
- Agregar modifier que solo permita llamadas desde `FlightReservation`.
- **Esfuerzo:** Medio.

---

### TIER 4 — Trabajo Futuro (Reconocido en el Paper)

**TASK-12: Verificación real de credenciales on-chain**
- Reemplazar `return true` en contratos.
- Opciones: patrón Oracle, hash commitment, o ZK-proof.
- **Esfuerzo:** Muy alto.

**TASK-13: Revocación de credenciales**
- `support_revocation: false` en todas las cred-defs actuales.
- Requiere registros de revocación en Indy + verificación en el bridge.
- **Esfuerzo:** Alto.

**TASK-14: Limpieza de recursos en `cancelTrip()`**
- El comentario en el contrato dice explícitamente "no tocamos recursos físicos aquí".
- Un viaje cancelado en estado CONFIRMED debe liberar el eVTOL (EXPECTING → PARKED) y restaurar parkings.
- **Esfuerzo:** Bajo.

---

### Resumen de Tareas

| Tarea    | Tier | Componente    | Esfuerzo  |
|----------|------|---------------|-----------|
| TASK-01  | 1    | Nuevo servicio | Alto      |
| TASK-02  | 1    | SSI_App        | Bajo      |
| TASK-03  | 1    | SSI_App        | Bajo      |
| TASK-04  | 1    | SSI_App        | Bajo      |
| TASK-05  | 2    | SSI_App        | Bajo      |
| TASK-06  | 2    | SSI_App        | Medio     |
| TASK-07  | 2    | BESU_project   | Bajo      |
| TASK-08  | 2    | SSI_App        | Bajo      |
| TASK-09  | 3    | SSI_App        | Medio     |
| TASK-10  | 3    | BESU_project   | Bajo      |
| TASK-11  | 3    | BESU_project   | Medio     |
| TASK-12  | 4    | Ambos          | Muy alto  |
| TASK-13  | 4    | Ambos          | Alto      |
| TASK-14  | 4    | BESU_project   | Bajo      |

---

*Archivo generado automáticamente desde la sesión de análisis del 2026-04-16.*
