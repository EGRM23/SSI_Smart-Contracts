# Sesión 2026-06-02 — Pasos 5 y 6 de Architecture A

## Resumen

Sesión de implementación y estudio de los Pasos 5 y 6 del guía `07_architecture_a_implementation_guide.md`.

---

## Problemas resueltos

### Error al correr migraciones
**Síntoma:** `ModuleNotFoundError: No module named 'django'`

**Causa:** El prompt mostraba `(base)` — se estaba usando el entorno conda base en lugar del venv del proyecto.

**Fix:**
```bash
source /home/egrm23/Documentos/TI3/SSI_App/venv/bin/activate
# El prompt cambia a (venv), luego:
python manage.py makemigrations auth_app
python manage.py migrate
```

**Nota:** Las migraciones ya estaban aplicadas hasta `0004_rename_wallet_key_wallet_agent_admin_url.py`. El modelo `Wallet` ya tiene `agent_admin_url`.

---

## Paso 5 — `credential_service.py`

Completado. Ver `07_architecture_a_implementation_guide.md` para el código completo.

### Pregunta sobre variables globales

Las constantes `SCHEMA_NAME`, `SCHEMA_VERSION`, `SCHEMA_ATTRS` en `credential_service.py` están hardcodeadas como globales porque el Paso 5 solo cubre el esquema de usuarios. Es una simplificación intencional.

**El refactor a multi-esquema llega en el Paso 7**, donde se definen los tres esquemas canónicos:

| Entidad | `schema_name` | `schema_version` |
|---|---|---|
| Usuario | `user_credential` | `2.0` |
| eVTOL | `evtol_credential` | `3.0` |
| Vertiport | `vertiport_credential` | `4.0` |

El patrón correcto para Paso 7:
```python
SCHEMAS = {
    'user':      ('user_credential',      '2.0', ['nombres', 'apellidos', 'fecha_nacimiento', 'can_ride']),
    'evtol':     ('evtol_credential',     '3.0', ['id_puerto', 'state', 'version', 'name', 'can_fly']),
    'vertiport': ('vertiport_credential', '4.0', ['location', 'capacity', 'state', 'is_active']),
}

class CredentialService:
    def __init__(self, schema_type: str = 'user'):
        self.schema_name, self.schema_version, self.schema_attrs = SCHEMAS[schema_type]
```

---

## Paso 6 — Reescribir `dni_service.py` → renombrado a `user_credential_service.py`

### Decisión de renombrado

`DNIService` se renombró a `UserCredentialService` porque:
- `DNI` es un concepto local peruano (Documento Nacional de Identidad)
- El nuevo nombre es consistente con la convención del sistema: `UserCredentialService`, `EVTOLCredentialService`, `VertiportCredentialService`

### Archivos afectados

| Archivo | Cambio |
|---|---|
| `services/user_credential_service.py` | Creado (reemplaza `dni_service.py`) |
| `auth_app/views/credential_views.py` | Referencias comentadas actualizadas |

`services/dni_service.py` queda obsoleto — puede borrarse, nadie lo importa.

### Los 3 bugs corregidos

| Bug | Archivo original | Fix |
|---|---|---|
| `issue_dni_credential()` no existe | `dni_service.py:18` | Cambiado a `issue_credential()` |
| `user=user` en `CredentialIssuance.objects.create()` | `dni_service.py:24` | Cambiado a `wallet=wallet` |
| `wallet_service.create_connection_for_user()` no existe | `dni_service.py:50` | Reemplazado por `_establish_connection(wallet)` |

### Descripción de cada método en `UserCredentialService`

| Método | Tipo | Qué hace |
|---|---|---|
| `__init__` | constructor | Instancia `CredentialService` |
| `issue_credential_to_user(user, user_data)` | público | Orquesta el flujo completo: wallet → conexión → emisión → guardado en BD |
| `_get_user_wallet(user)` | privado | Lee BD, devuelve el `Wallet` del usuario. Lanza `ValueError` si no tiene. |
| `_establish_connection(holder_wallet)` | privado | Handshake DIDComm: reutiliza conexión `complete` existente o crea una nueva (issuer invita, holder acepta) |
| `_wait_for_active(client, conn_id)` | privado | Polling cada 2s hasta que la conexión llega a `active` (timeout: 60s) |
| `get_user_credentials(user, state)` | público | Consulta BD, devuelve `CredentialIssuance` del usuario, filtrables por estado |

### Diseño: por qué un orquestador

La alternativa sería meter todo el flujo en la vista Django directamente. El problema:
- Mezcla lógica de negocio con HTTP
- No reutilizable cuando se agreguen eVTOL/Vertiport
- No testeable sin levantar Django

Con el patrón orquestador, emitir credencial de eVTOL en el futuro es simplemente:
```python
EVTOLCredentialService().issue_credential_to_evtol(evtol, evtol_data)
```

### Smoke test (Paso 8 adelantado)

```bash
cd SSI_App
source venv/bin/activate
DJANGO_SETTINGS_MODULE=wallet.settings python -c "
import django; django.setup()
from aca_py.client import ACApyClient; print('client OK')
from services.wallet_service import WalletService; print('wallet_service OK')
from services.credential_service import CredentialService; print('credential_service OK')
from services.user_credential_service import UserCredentialService; print('user_credential_service OK')
"
```

Resultado esperado: los 4 `OK`.

> **Nota:** El smoke test de la guía usa `python -c "from services.dni_service import DNIService"` — reemplazar por `user_credential_service` como se muestra arriba.

---

## Estado del guía al final de la sesión

- [x] Paso 1 — `wallet/settings.py` actualizado
- [x] Paso 2 — Campo `agent_admin_url` en modelo `Wallet` + migración aplicada
- [x] Paso 3 — `aca_py/client.py` reescrito
- [x] Paso 4 — `services/wallet_service.py` reescrito
- [x] Paso 5 — `services/credential_service.py` reescrito
- [x] Paso 6 — `services/user_credential_service.py` (ex `dni_service.py`) reescrito y renombrado
- [ ] Paso 7 — Nombres canónicos de schemas en `credential_test/connect_and_issue.py`
- [ ] Paso 8 — Smoke test de imports (parcialmente verificado en esta sesión)
