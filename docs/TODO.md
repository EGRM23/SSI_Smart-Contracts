# TODO — Trabajo futuro identificado

Problemas y mejoras identificados durante el desarrollo e integración del sistema TI3.
Ordenados por área y prioridad aproximada.

---

## Escalabilidad de ACA-Py

### 1. Migrar ACA-Py de SQLite a PostgreSQL
**Problema identificado:** El agente ACA-Py usa SQLite como backend de wallet. Con ≥15 usuarios
concurrentes emitiendo credenciales, el SQLite interno de ACA-Py se bloquea (`database is locked`)
y el sistema colapsa (96% de fallos a 25 usuarios). Tras el colapso, ACA-Py no se recupera solo —
requiere reinicio manual.

**Impacto medido:** Zona estable ≤10 usuarios concurrentes (0% fallos). Zona de colapso ≥15 usuarios.

**Solución:** Configurar ACA-Py con wallet PostgreSQL usando el plugin
`aries-askar` + PostgreSQL en el `docker-compose.yml` de los agentes:
```yaml
environment:
  - ACAPY_WALLET_TYPE=askar
  - ACAPY_WALLET_STORAGE_TYPE=postgres_storage
  - ACAPY_WALLET_STORAGE_CONFIG={"url":"postgres:5432"}
```

### 2. Un agente ACA-Py por usuario (arquitectura multiagente)
**Problema identificado:** Todos los usuarios comparten el mismo agente `acapy-user1`.
Las solicitudes concurrentes de conexión OOB y emisión de credenciales se serializan en ese
único agente, creando un cuello de botella estructural.

**Solución:** Crear un agente ACA-Py dedicado por usuario al momento del registro
(`POST /api/user/`). El `WalletService` de Django levantaría un contenedor dinámicamente
o usaría un pool de agentes pre-creados. Esto es el diseño correcto para producción.

**Complejidad:** Alta — requiere orquestación dinámica de contenedores (Kubernetes o Docker API).

---

## Bridge SSI → Besu

### 3. Verificación de firmas en el bridge (ecrecover on-chain)
**Estado actual:** Los contratos Solidity aceptan parámetros de credenciales SSI pero la
verificación está mockeada (`return true` en `UserVerification`, `EVTOLManagement`, etc.).

**Trabajo pendiente:**
- Implementar firma del hash de credencial en el lado Python (`bridge.py`) usando la
  clave privada del Trusted Verifier (`eth_account.sign_message`)
- Implementar `ecrecover` en los contratos Solidity para verificar que la firma proviene
  del Trusted Verifier autorizado
- La arquitectura Opción A ("Trusted Verifier") ya está documentada en
  `docs/10_verificacion_criptografica.md`

**Impacto:** Es el paso que convierte el sistema de prototipo a producción real —
sin esto, cualquiera podría llamar a los contratos con credenciales falsas.

---

## Django / Base de datos

### 4. Migrar Django de SQLite a PostgreSQL
**Problema identificado:** SQLite bloquea escrituras concurrentes en Django. El Escenario 1
de escalabilidad mostró primera degradación a 75 usuarios simultáneos registrándose
(1.6% de fallos) y colapso a 200 usuarios (47.9% de fallos).

**Solución:** Reemplazar `db.sqlite3` por PostgreSQL. Cambio mínimo en `settings.py`
y `docker-compose.yml` del SSI_App.

---

## Besu / Smart contracts

### 5. Actualizar addresses en smoke_test_full_system.js
**Estado:** `smoke_test_full_system.js` tiene addresses de contratos hardcodeadas.
Cada vez que se hace `node scripts/deploy_system.js`, las addresses cambian y el
smoke test falla hasta actualizarlas manualmente.

**Solución:** El script ya genera `deployed_addresses.json` automáticamente —
hacer que `smoke_test_full_system.js` lo lea en lugar de usar addresses hardcodeadas.

---

## Documentación / Testing

### 8. Pruebas de seguridad — contratos Besu
**Estado:** ✅ Ejecutado. Los 4 tests pasan.

| Test | Ataque intentado | Resultado |
|------|-----------------|-----------|
| 4.1 | `registerVertiport` con firma vacía (`b""`) | ✅ revert — ecrecover rechaza 0 bytes |
| 4.2 | `registerVertiport` con firma basura (65 bytes `0xabab…`) | ✅ revert — dirección recuperada ≠ trustedVerifier |
| 4.3 | `createReservation` dos veces con el mismo eVTOL | ✅ revert — `isAvailable()` = false (state machine) |
| 4.4 | `createReservation` con rider no autorizado (`0x0000…0001`) | ✅ revert — `canUserRide()` = false |

**Nota:** Los contratos implementan `ecrecover` real (no mock). El comentario en CLAUDE.md
("verificación mockeada") corresponde a una versión anterior reemplazada en la Opción A
del Trusted Verifier (doc 10).

---

### 6. Escenario 3: pruebas de escalabilidad de reservas Besu
**Estado:** ✅ Ejecutado. Resultados en `SSI_App/load_tests/resultados/besu_*.html`
y gráficos `grafico_07/08/09_besu_*.png`.

**Hallazgo:** El contrato `FlightReservation` sólo permite un viaje a la vez (state machine
del EVTOL: PARKED → EXPECTING → IN_USE → PARKED). Con N usuarios concurrentes compartiendo
una sola cuenta Besu (nonce centralizado) y un solo EVTOL:
- **1u**: 6 viajes completos/90 s, 0% fallos, ~5 s latencia (tiempo de bloque QBFT)
- **3u**: 1 viaje/90 s, **88% fallos** (txs revertan on-chain por EVTOL EXPECTING)
- **5u**: 1 viaje/90 s, **96% fallos** (misma causa)
- **10u**: 0 viajes/90 s, **~100% fallos** (Besu rechaza con "nonce too distant", ~42 ms)

**Causa raíz:** Los N usuarios generan N nonces por ciclo pero el EVTOL solo acepta 1
createReservation por vez → N-1 txs consumen nonces inútilmente → el nonce de startTrip
queda bloqueado detrás de N-1 txs fallidas por bloque.

**Solución para producción:** Flota de múltiples EVTOLs (1 por par de vertiports) o
mecanismo de cola on-chain para serializar las reservas correctamente.

---

### 7. Pruebas de tolerancia a fallos
**Estado:** ✅ Ejecutado. Cuatro escenarios validados.

#### 7.1 Besu QBFT — caída de nodos validator
- **1/4 caído (3/4 activos):** consenso continúa, throughput sin cambio (0.20 blq/s), latencia 4–9 s.
- **2/4 caídos (quórum roto):** bloque congelado, txs aceptadas en mempool pero nunca minadas.
- **Recovery:** ~5 min para re-sync QBFT; la tx pendiente se incluye en el primer bloque nuevo.

#### 7.2 Indy Plenum — caída de nodos ledger
- **1/4 caído (3/4):** emisión de credenciales sin degradación (~2 s).
- **2/4 caídos (quórum roto):** re-emisión de cred_def cacheada **continúa** (ACA-Py cachea en wallet local); creación de schema **nuevo** falla con timeout (requiere write al ledger).
- **Recovery:** inmediato al restaurar nodos.

#### 7.3 ACA-Py issuer — crash durante emisión
- Fallo limpio con `ConnectionRefused` en :8031. Ningun estado inconsistente en Besu.
- Recovery inmediato con `docker start acapy-issuer`.

#### 7.4 Django — caída durante bridge
- Bridge falla en Fase 2a (`/api/attest/user/`) con `ConnectionRefused` en :8000.
- Besu no recibe ninguna tx → estado on-chain consistente.
- Recovery: reiniciar Django + re-ejecutar bridge (operaciones register son idempotentes).
