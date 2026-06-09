# Prueba issuer → holder: lecciones aprendidas y pendientes

**Fecha:** 2026-06-09
**Resultado:** Credencial `evtol_credential:3.0` emitida y almacenada exitosamente en el wallet del holder.
**Instrucciones de ejecución:** ver `08_como_ejecutar_prueba_ssi.md`

---

## Por qué hubo tantos problemas en la primera ejecución

Los problemas no fueron del código — fueron de infraestructura y bootstrapping. Se agrupan en cuatro causas:

### 1. Disco lleno (causa raíz de varios fallos)
El disco estaba al 99% de capacidad. RocksDB (la base de datos que usan los nodos Indy) no puede operar sin espacio para escribir logs de transacciones. Los nodos 2 y 4 crashearon durante el arranque dejando datos corruptos. El síntoma visible fue el loop de restart con:
```
rocksdb.errors.RocksIOError: No space left on device
```
**Lección:** Liberar espacio **antes** de arrancar la VON Network. Con menos del 5% libre, los nodos son inestables.

### 2. DID del issuer no registrado en el ledger
El issuer arranca con `--seed issuer00000000000000000000000001`, que genera su DID localmente, pero ese DID no existe en el ledger Indy hasta que alguien lo registre como Trust Anchor (o Endorser). Sin ese registro, el issuer no puede escribir schemas ni credential definitions. El error fue:
```
could not authenticate, verkey for Utwqp5cpEATQpGZL5WSQZJ cannot be found
```
El registro requiere que un **Steward** (cuenta con permisos en el ledger) firme una transacción NYM. En VON Network de desarrollo esto se hace vía el endpoint `POST /register` del webserver en el puerto 9000.

**Truco importante:** el endpoint solo acepta JSON (no form-data), y solo responde bien si el ledger tiene los 4 nodos activos. Cualquier intento en disco lleno o con nodos caídos devuelve 500.

### 3. Volúmenes Docker corruptos
Después del crash por disco lleno, simplemente reiniciar la VON Network (`./manage stop` + `./manage start`) no es suficiente — los volúmenes de los nodos caídos conservan datos RocksDB corruptos. Se requiere `./manage down` (que elimina contenedores **y** volúmenes) para empezar limpio.

### 4. Flags faltantes y script no idempotente
- El holder necesitaba `--auto-respond-credential-offer` para aceptar automáticamente las ofertas de credencial. Sin este flag, el flujo se congela en `offer_received`.
- El script `connect_and_issue.py` fallaba en la segunda ejecución porque intentaba crear el schema y la cred def aunque ya existieran (error 400). Ahora el script verifica si existen antes de crearlos.

---

## Cómo ejecutar la prueba

### Requisitos previos
- Docker y Docker Compose instalados
- Al menos 5 GB libres en disco: `df -h /`
- El venv de SSI_App activado: `source ~/Documentos/TI3/SSI_App/venv/bin/activate`

### Paso 1 — Arrancar la VON Network (Terminal 1)

```bash
cd ~/Documentos/TI3/SSI_Project
./manage start --wait
```

Esperar el mensaje `Ledger started`. Verificar que los 4 nodos estén corriendo:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | grep von
```

Debe mostrar `von-node1-1`, `von-node2-1`, `von-node3-1`, `von-node4-1` todos en `Up`.

### Paso 2 — Registrar el DID del issuer en el ledger

Este paso solo es necesario la **primera vez** después de un `./manage down`, o si el ledger fue reiniciado desde cero.

```bash
curl -X POST http://localhost:9000/register \
  -H "Content-Type: application/json" \
  -d '{"did":"Utwqp5cpEATQpGZL5WSQZJ","verkey":"GCqSzwWVsRumDY67zkBACHs92zQJpg71WKmFEHnTreUQ","alias":"issuer","role":"TRUST_ANCHOR"}'
```

Respuesta esperada:
```json
{"did": "Utwqp5cpEATQpGZL5WSQZJ", "seed": null, "verkey": "GCqSzwWVsRumDY67zkBACHs92zQJpg71WKmFEHnTreUQ"}
```

> **Nota:** El DID y verkey son deterministas — siempre los mismos para el seed `issuer00000000000000000000000001`. No cambian entre reinicios del ledger.

Si el registro ya se hizo (ledger no fue reiniciado), este paso puede omitirse. Para verificar:
```bash
curl -s "http://localhost:9000/browse/domain?page=1&query=Utwqp5cpEATQpGZL5WSQZJ&type=1" | grep -c seqNo
```
Si retorna `1` o más, ya está registrado.

### Paso 3 — Arrancar los agentes ACA-Py (Terminal 2)

```bash
cd ~/Documentos/TI3/SSI_App/credential_test
docker compose -f docker-compose-agents.yml up issuer user1
```

Esperar que ambos impriman `Listening...` sin `Shutting down`. El issuer debe mostrar su DID público:
```
:: Public DID Information:
::   - DID: Utwqp5cpEATQpGZL5WSQZJ
```

### Paso 4 — Ejecutar el script de prueba (Terminal 3)

```bash
cd ~/Documentos/TI3/SSI_App/credential_test
source ../venv/bin/activate
python connect_and_issue.py
```

#### Resultado esperado (paso 7):
```
Credenciales guardadas en Holder:
[
  {
    "referent": "...",
    "schema_id": "Utwqp5cpEATQpGZL5WSQZJ:2:evtol_credential:3.0",
    "cred_def_id": "Utwqp5cpEATQpGZL5WSQZJ:3:CL:8:default",
    "attrs": {
      "can_fly": "true",
      "id_puerto": "puerto_42",
      "version": "v1",
      "state": "ACTIVE",
      "name": "EVTOL-Alpha"
    }
  }
]
```

### Cómo verificar sin el script

Además del script, puedes verificar el estado directamente via la Admin API:

```bash
# Credenciales en el wallet del holder
curl -s http://localhost:8041/credentials | python3 -m json.tool

# Schemas registrados por el issuer
curl -s http://localhost:8031/schemas/created | python3 -m json.tool

# Conexiones activas del issuer
curl -s http://localhost:8031/connections | python3 -m json.tool

# Ledger browser (interfaz web)
# Abrir en el navegador: http://localhost:9000/browse/domain
```

---

## Cómo detener el sistema

### Detener solo los agentes ACA-Py
```bash
# En el terminal donde corren, Ctrl+C, luego:
cd ~/Documentos/TI3/SSI_App/credential_test
docker compose -f docker-compose-agents.yml down
```

### Detener la VON Network (conservando el estado del ledger)
```bash
cd ~/Documentos/TI3/SSI_Project
./manage stop
```
Los datos del ledger (DIDs, schemas, cred defs) se conservan. La próxima vez basta con `./manage start --wait` sin registrar el DID de nuevo.

### Detener y borrar todo (reset completo)
```bash
cd ~/Documentos/TI3/SSI_Project
./manage down
```
Elimina contenedores y volúmenes. La próxima vez hay que registrar el DID del issuer nuevamente (Paso 2).

---

## Architecture A vs. Multitenant: diferencias

### Multitenant (enfoque de los notebooks anteriores)

```
Un proceso ACA-Py
└── Wallet A (issuer)   ← creado en runtime vía POST /multitenancy/wallet
└── Wallet B (holder)   ← creado en runtime vía POST /multitenancy/wallet
└── Wallet C (otro)     ← creado en runtime
```

- Un solo contenedor Docker, un solo puerto admin
- Los wallets se crean y destruyen durante la ejecución del script
- Más simple de arrancar
- Si el proceso cae, caen todos los wallets

### Architecture A (enfoque actual del proyecto)

```
acapy-issuer (puerto 8031)   ← proceso propio, wallet propio en Askar
acapy-user1  (puerto 8041)   ← proceso propio, wallet propio en Askar
acapy-evtol1 (puerto 8051)   ← proceso propio, wallet propio en Askar
...
```

- N contenedores Docker, N puertos admin
- Los wallets existen desde que Docker arranca (`--auto-provision`)
- Un agente caído no afecta a los demás
- Cada entidad del sistema (usuario, eVTOL, vertiport) tiene su agente propio
- Es lo que describe el paper del proyecto

### Comparación

| | Multitenant | Architecture A |
|---|---|---|
| Arranque | Un contenedor | N contenedores |
| Creación de wallets | Runtime via API | Al arrancar Docker |
| Aislamiento | Bajo (proceso compartido) | Alto (proceso independiente) |
| Complejidad operativa | Baja | Media |
| Corresponde al paper | No | Sí |
| Útil para | Pruebas rápidas en notebook | Sistema completo del proyecto |

---

## ¿Tendría los mismos problemas en un sistema real?

### Problemas que desaparecen en producción

| Problema hoy | Por qué no existe en producción |
|---|---|
| Disco lleno crashea nodos | Infraestructura monitoreada con alertas de disco |
| Registro manual del issuer DID | El DID se registra una sola vez en el onboarding del sistema, no en cada deploy |
| VON Network de desarrollo | Se usa una red Indy real (Sovrin MainNet, Indicio, IDunion) — no gestionas nodos tú mismo |
| Volúmenes corruptos tras crash | Backups, health checks y reinicio automático gestionados por Kubernetes o similar |
| `--auto-respond-credential-offer` | En una app real, el usuario toca "Aceptar" en su billetera móvil (ej. Lissi, Bifold) |

### Problemas que sí persisten y requieren diseño

| Problema | Solución en producción |
|---|---|
| Schema ya existe en re-deploy | Igual que ahora: verificar antes de crear (ya aplicado) |
| El issuer debe tener rol de Endorser en la red | Proceso de onboarding con la red Indy elegida. En Sovrin cuesta dinero; en redes de test es gratuito |
| `connect_and_issue.py` usa RFC 0036 (deprecated) | Migrar a RFC 0453 (Issue Credential 2.0) antes de producción |

### Diferencia más importante

En producción **no gestionas los nodos Indy** — te conectas a una red pública o permisionada que ya existe. El equivalente del Paso 2 (registrar el DID del issuer) ocurre una sola vez cuando la organización se une a esa red, no en cada arranque del sistema.

---

## Pendientes identificados en esta sesión

El bridge (SSI → Besu) queda para después. Estas tareas son las prioritarias ahora:

### Limpieza y orden de archivos

- [ ] Eliminar `SSI_App/services/dni_service.py` — obsoleto, reemplazado por `user_credential_service.py`
- [ ] Actualizar `SSI_App/CLAUDE.md` — todavía referencia `dni_service.py`, puertos viejos (9040/9041) y `ACA_PY_CONFIG`
- [ ] Eliminar el atributo `version` obsoleto del `docker-compose-agents.yml` (Docker Compose lo ignora pero genera warnings)
- [ ] Revisar si `SSI_App/ssi-credential-system/` tiene contenido útil o puede eliminarse

### Deprecaciones de ACA-Py a actualizar

Los agentes muestran estos warnings en cada arranque — indican que el código actual usa protocolos que serán eliminados:

| Deprecated | Reemplazar por |
|---|---|
| RFC 0160: Connection Protocol | RFC 0023: DID Exchange |
| RFC 0036: Issue Credential 1.0 | RFC 0453: Issue Credential 2.0 |
| RFC 0037: Present Proof 1.0 | RFC 0454: Present Proof 2.0 |
| Prefix `did:sov:BzCbsNYhMrjHiqZDTUASHg;spec` | Prefix `https://didcomm.org/` |

Afecta principalmente a `connect_and_issue.py` (usa `issue-credential/1.0/credential-preview`).

### Pruebas con los demás credentials

Solo se probó `evtol_credential` con `issuer → user1`. Falta probar:

- [ ] `user_credential:2.0` — emitir al agente `user1` (el que más importa para el bridge)
- [ ] `evtol_credential:3.0` — emitir al agente `evtol1` (puerto 8051)
- [ ] `vertiport_credential:4.0` — emitir a `vertiport1` (8061) y `vertiport2` (8071)

Para cada uno habrá que registrar sus DIDs en el ledger (misma mecánica que el issuer, pero como holders no necesitan rol TRUST_ANCHOR — pueden quedar sin rol).

### Otros

- [ ] Descomentar y probar las vistas de credenciales en `credential_views.py` + `wallet/urls.py`
- [ ] Verificar que `UserCredentialService` (Paso 6) funciona end-to-end desde Django contra los agentes reales
