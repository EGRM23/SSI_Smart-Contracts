# Cómo ejecutar y detener la prueba SSI (issuer → holder)

## Requisitos previos

- Docker y Docker Compose instalados
- Al menos **5 GB libres** en disco: `df -h /`
- Estar en el directorio correcto con el venv activado

```bash
source ~/Documentos/TI3/SSI_App/venv/bin/activate
```

---

## Arranque (3 terminales)

### Terminal 1 — VON Network (ledger Indy)

```bash
cd ~/Documentos/TI3/SSI_Project
./manage start --wait
```

Esperar `Ledger started`. Verificar que los 4 nodos estén en `Up`:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | grep von
```

### Terminal 2 — Registrar DID del issuer *(solo después de un reset del ledger)*

> Omitir este paso si el ledger no fue reiniciado desde cero (`./manage down`).

```bash
curl -X POST http://localhost:9000/register \
  -H "Content-Type: application/json" \
  -d '{"did":"Utwqp5cpEATQpGZL5WSQZJ","verkey":"GCqSzwWVsRumDY67zkBACHs92zQJpg71WKmFEHnTreUQ","alias":"issuer","role":"TRUST_ANCHOR"}'
```

Respuesta esperada: `{"did": "Utwqp5cpEATQpGZL5WSQZJ", ...}`

Para verificar si ya está registrado:
```bash
curl -s "http://localhost:9000/browse/domain?page=1&query=Utwqp5cpEATQpGZL5WSQZJ&type=1" | grep -c seqNo
# retorna 1 → ya registrado, retorna 0 → hay que registrar
```

### Terminal 2 — Agentes ACA-Py

```bash
cd ~/Documentos/TI3/SSI_App/credential_test
docker compose -f docker-compose-agents.yml up issuer user1
```

Esperar que ambos impriman `Listening...` sin `Shutting down`.
El issuer debe mostrar `DID: Utwqp5cpEATQpGZL5WSQZJ`.

### Terminal 3 — Script de prueba

```bash
cd ~/Documentos/TI3/SSI_App/credential_test
source ../venv/bin/activate
python connect_and_issue.py
```

#### Resultado exitoso (paso 7):

```json
[
  {
    "schema_id": "Utwqp5cpEATQpGZL5WSQZJ:2:evtol_credential:3.0",
    "attrs": {
      "can_fly": "true",
      "id_puerto": "puerto_42",
      "state": "ACTIVE",
      "name": "EVTOL-Alpha",
      "version": "v1"
    }
  }
]
```

---

## Verificación manual (sin el script)

```bash
# Credenciales en el wallet del holder
curl -s http://localhost:8041/credentials | python3 -m json.tool

# Schemas creados por el issuer
curl -s http://localhost:8031/schemas/created | python3 -m json.tool

# Ledger browser (web)
# http://localhost:9000/browse/domain
```

---

## Detener el sistema

### Detener agentes ACA-Py (conserva wallets)
```bash
cd ~/Documentos/TI3/SSI_App/credential_test
docker compose -f docker-compose-agents.yml down
```

### Detener VON Network conservando el ledger
```bash
cd ~/Documentos/TI3/SSI_Project
./manage stop
```
Los DIDs, schemas y cred defs se conservan. La próxima vez no hay que registrar el DID del issuer.

### Reset completo (borra todo el ledger)
```bash
cd ~/Documentos/TI3/SSI_Project
./manage down
```
La próxima vez hay que ejecutar el paso de registro del DID del issuer nuevamente.

---

## Mapa de puertos

| Componente | Puerto inbound | Puerto admin |
|---|---|---|
| VON Network browser | — | 9000 |
| ACA-Py issuer | 8030 | 8031 |
| ACA-Py user1 | 8040 | 8041 |
| ACA-Py evtol1 | 8050 | 8051 |
| ACA-Py vertiport1 | 8060 | 8061 |
| ACA-Py vertiport2 | 8070 | 8071 |
| Django | — | 8000 |
| Besu RPC | — | 8545 |
