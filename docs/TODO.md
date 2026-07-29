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

### 6. Escenario 3: pruebas de escalabilidad de reservas Besu
**Estado:** No ejecutado aún. El locustfile ya está definido en
`docs/11_pruebas_del_sistema.md` (Escenario 3).

**Descripción:** N usuarios haciendo reservas de vuelo concurrentes vía `bridge.py`
→ `FlightReservation` en Besu. Mide throughput de transacciones on-chain.
