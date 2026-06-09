# Architecture A — Guía de Implementación (Tasks 1–8)

**Date:** 2026-05-05
**Decisión:** Se elige Architecture A (un agente ACA-Py por entidad, cada uno en su propio puerto/contenedor Docker).

---

## Fundamento de Architecture A

En Architecture A cada entidad tiene **su propio proceso ACA-Py corriendo en su propio puerto**. Tres consecuencias de diseño que guían toda la implementación:

1. **"Crear wallet" en runtime no existe** — el wallet ya existe, arrancó con Docker. Lo que se hace en runtime es crear un DID dentro de ese wallet. El método `create_wallet` en `client.py` tiene el nombre equivocado.
2. **`ACApyClient` no puede tener una URL global** — el issuer está en un puerto, cada holder en otro. El cliente debe recibir la URL como parámetro.
3. **Django necesita saber a qué agente hablarle por entidad** — el modelo `Wallet` debe guardar `agent_admin_url` para que los servicios instancien el cliente correcto.

---

## Estado de avance

- [x] Paso 1 — `wallet/settings.py` actualizado
- [x] Paso 2 — Campo `agent_admin_url` en modelo `Wallet` + migración aplicada
- [ ] Paso 3 — Reescribir `aca_py/client.py`
- [ ] Paso 4 — Reescribir `services/wallet_service.py`
- [ ] Paso 5 — Reescribir `services/credential_service.py`
- [ ] Paso 6 — Reescribir `services/dni_service.py`
- [ ] Paso 7 — Nombres canónicos de schemas (Task 3)
- [ ] Paso 8 — Smoke test de imports

---

## Paso 1 — `wallet/settings.py` ✅

**Archivo:** `SSI_App/wallet/settings.py`

Reemplazar `ACA_PY_CONFIG` por `ACA_PY_AGENTS`:

```python
ACA_PY_AGENTS = {
    'issuer':      os.getenv('ISSUER_ADMIN_URL',      'http://localhost:9041'),
    'user_1':      os.getenv('USER_1_ADMIN_URL',      'http://localhost:9031'),
    'user_2':      os.getenv('USER_2_ADMIN_URL',      'http://localhost:9033'),
    'evtol_1':     os.getenv('EVTOL_1_ADMIN_URL',     'http://localhost:9035'),
    'vertiport_1': os.getenv('VERTIPORT_1_ADMIN_URL', 'http://localhost:9037'),
    'vertiport_2': os.getenv('VERTIPORT_2_ADMIN_URL', 'http://localhost:9039'),
}
```

**Por qué:** Es la fuente de verdad de qué agentes existen. Todo el código posterior lo leerá.

---

## Paso 2 — Modelo `Wallet` + migración ✅

**Archivo:** `SSI_App/auth_app/models.py`

```python
class Wallet(models.Model):
    wallet_id       = models.CharField(max_length=255)
    wallet_key      = models.CharField(max_length=255, blank=True)  # no necesario en Arq. A
    public_did      = models.CharField(max_length=255, blank=True, null=True)
    agent_admin_url = models.CharField(max_length=255)              # ← campo nuevo
    created_at      = models.DateTimeField(auto_now_add=True)

    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id    = models.PositiveIntegerField()
    owner        = GenericForeignKey('content_type', 'object_id')
```

Django renombró `wallet_key` → `agent_admin_url` via migración `0004`. Resultado:

| Campo | Propósito |
|---|---|
| `wallet_id` | Nombre del agente (ej: `'user_1'`) |
| `agent_admin_url` | URL Admin API del agente (ej: `http://localhost:9031`) |
| `public_did` | DID registrado en el ledger Indy |
| `owner` | GenericForeignKey → User, EVTOL, Vertiport |

```bash
# Comandos ejecutados
python manage.py makemigrations auth_app
python manage.py migrate
```

---

## Paso 3 — Reescribir `aca_py/client.py`

**Archivo:** `SSI_App/aca_py/client.py`

Reemplazar el archivo completo:

```python
# aca_py/client.py
import requests
from django.conf import settings


class ACApyClient:

    def __init__(self, admin_url: str):
        self.admin_url = admin_url.rstrip('/')
        self.headers = {
            'Content-Type': 'application/json',
            'Accept': 'application/json',
        }

    # ── base ──────────────────────────────────────────────────────────────

    def _make_request(self, method: str, path: str, payload: dict = None) -> dict:
        url = f"{self.admin_url}{path}"
        method = method.upper()
        if method == 'GET':
            r = requests.get(url, headers=self.headers, timeout=30)
        elif method == 'POST':
            r = requests.post(url, json=payload or {}, headers=self.headers, timeout=30)
        elif method == 'DELETE':
            r = requests.delete(url, headers=self.headers, timeout=30)
        else:
            raise ValueError(f"Unsupported method: {method}")
        r.raise_for_status()
        return r.json()

    # ── DID / wallet ──────────────────────────────────────────────────────

    def create_did(self) -> dict:
        """Crea un DID local dentro del wallet del agente."""
        return self._make_request('POST', '/wallet/did/create', {
            "method": "sov",
            "options": {"key_type": "ed25519"}
        })

    # ── conexiones ────────────────────────────────────────────────────────

    def create_invitation(self, alias: str = None) -> dict:
        payload = {"alias": alias} if alias else {}
        return self._make_request('POST', '/connections/create-invitation', payload)

    def receive_invitation(self, invitation: dict) -> dict:
        return self._make_request('POST', '/connections/receive-invitation', invitation)

    def get_connection(self, connection_id: str) -> dict:
        return self._make_request('GET', f"/connections/{connection_id}")

    def get_connections(self) -> dict:
        return self._make_request('GET', '/connections')

    def send_message(self, connection_id: str, content: str) -> dict:
        return self._make_request('POST', f"/connections/{connection_id}/send-message",
                                  {"content": content})

    # ── schemas ───────────────────────────────────────────────────────────

    def get_schemas_created(self) -> dict:
        return self._make_request('GET', '/schemas/created')

    def create_schema(self, name: str, version: str, attributes: list) -> dict:
        return self._make_request('POST', '/schemas', {
            "schema_name": name,
            "schema_version": version,
            "attributes": attributes,
        })

    # ── credential definitions ────────────────────────────────────────────

    def get_credential_definitions_created(self) -> dict:
        return self._make_request('GET', '/credential-definitions/created')

    def create_credential_definition(self, schema_id: str, tag: str = "default",
                                      support_revocation: bool = False) -> dict:
        return self._make_request('POST', '/credential-definitions', {
            "schema_id": schema_id,
            "tag": tag,
            "support_revocation": support_revocation,
        })

    # ── emisión de credenciales ───────────────────────────────────────────

    def send_credential_offer(self, connection_id: str, cred_def_id: str,
                               attributes: list) -> dict:
        return self._make_request('POST', '/issue-credential/send-offer', {
            "connection_id": connection_id,
            "cred_def_id": cred_def_id,
            "credential_preview": {
                "@type": "issue-credential/1.0/credential-preview",
                "attributes": attributes,
            },
            "auto_issue": True,
            "auto_remove": False,
        })

    def get_issue_credential_records(self, connection_id: str = None,
                                      state: str = None) -> dict:
        path = '/issue-credential/records'
        params = []
        if connection_id:
            params.append(f"connection_id={connection_id}")
        if state:
            params.append(f"state={state}")
        if params:
            path += '?' + '&'.join(params)
        return self._make_request('GET', path)

    def get_holder_credentials(self) -> dict:
        """Credenciales ya almacenadas en el wallet del holder."""
        return self._make_request('GET', '/credentials')
```

**Cambios clave:**

| Antes | Ahora | Por qué |
|---|---|---|
| `__init__(self)` — URL de settings | `__init__(self, admin_url)` | Cada servicio apunta al agente correcto |
| `create_wallet(wallet_name, wallet_key)` | `create_did()` sin parámetros | Nombra lo que hace; la clave es de Docker |
| Sin `_make_request` | Con `_make_request` | Todos los métodos convergen aquí |
| 3 métodos | 15 métodos | Cubre todos los endpoints que los servicios necesitan |

---

## Paso 4 — Reescribir `services/wallet_service.py`

**Archivo:** `SSI_App/services/wallet_service.py`

```python
# services/wallet_service.py
from aca_py.client import ACApyClient
from auth_app.models import Wallet, Connection
from django.conf import settings


class WalletService:

    def __init__(self, agent_key: str = 'issuer'):
        admin_url = settings.ACA_PY_AGENTS[agent_key]
        self.client = ACApyClient(admin_url)
        self.agent_admin_url = admin_url

    def create_wallet(self, owner_model, agent_key: str) -> Wallet:
        """
        Registra en Django la asociación entre una entidad y su agente ACA-Py.
        Crea un DID en ese agente y lo guarda como public_did.
        """
        admin_url = settings.ACA_PY_AGENTS[agent_key]
        client = ACApyClient(admin_url)

        did_data = client.create_did()
        did = did_data.get('result', {}).get('did')

        wallet = Wallet.objects.create(
            owner=owner_model,
            wallet_id=agent_key,
            agent_admin_url=admin_url,
            public_did=did,
        )
        return wallet

    def create_connection_invitation(self, wallet: Wallet, alias: str = None) -> Connection:
        client = ACApyClient(wallet.agent_admin_url)
        inv_data = client.create_invitation(alias)

        return Connection.objects.create(
            wallet=wallet,
            connection_id=inv_data['connection_id'],
            their_label=alias or '',
            state='invited',
            invitation_url=inv_data.get('invitation_url', ''),
        )
```

**Por qué cambia:** `create_wallet` recibe `agent_key` (ej: `'user_1'`) en lugar de generar una clave aleatoria inútil. `create_connection_invitation` lee `wallet.agent_admin_url` para hablar con el agente correcto — la invitación la crea el agente del holder, no el issuer.

---

## Paso 5 — Reescribir `services/credential_service.py`

**Archivo:** `SSI_App/services/credential_service.py`

```python
# services/credential_service.py
from aca_py.client import ACApyClient          # ← import correcto (Task 2, bug 1)
from django.conf import settings


SCHEMA_NAME    = 'user_credential'
SCHEMA_VERSION = '2.0'
SCHEMA_ATTRS   = ['nombres', 'apellidos', 'fecha_nacimiento', 'can_ride']


class CredentialService:

    def __init__(self):
        issuer_url = settings.ACA_PY_AGENTS['issuer']
        self.client = ACApyClient(issuer_url)

    def get_or_create_schema(self) -> str:
        resp = self.client.get_schemas_created()
        matching = [s for s in resp.get('schema_ids', []) if SCHEMA_NAME in s]
        if matching:
            return matching[0]
        result = self.client.create_schema(SCHEMA_NAME, SCHEMA_VERSION, SCHEMA_ATTRS)
        return result.get('schema_id') or result.get('sent', {}).get('schema_id')

    def get_or_create_credential_definition(self) -> str:
        resp = self.client.get_credential_definitions_created()
        matching = [c for c in resp.get('credential_definition_ids', [])
                    if SCHEMA_NAME in c]
        if matching:
            return matching[0]
        schema_id = self.get_or_create_schema()
        result = self.client.create_credential_definition(schema_id, tag='default')
        return (result.get('credential_definition_id') or
                result.get('sent', {}).get('credential_definition_id'))

    def issue_credential(self, connection_id: str, user_data: dict) -> dict:
        cred_def_id = self.get_or_create_credential_definition()
        attributes = [{"name": k, "value": str(v)} for k, v in user_data.items()]
        return self.client.send_credential_offer(connection_id, cred_def_id, attributes)

    def get_credential_records(self, connection_id: str = None,
                                state: str = None) -> dict:
        return self.client.get_issue_credential_records(connection_id, state)
```

---

## Paso 6 — Reescribir `services/dni_service.py`

**Archivo:** `SSI_App/services/dni_service.py`

Corrige los tres bugs de Task 2:

```python
# services/dni_service.py
import time
from .credential_service import CredentialService
from .wallet_service import WalletService
from aca_py.client import ACApyClient
from auth_app.models import CredentialIssuance, Connection, Wallet


POLL_INTERVAL = 2
POLL_TIMEOUT  = 60


class DNIService:

    def __init__(self):
        self.credential_service = CredentialService()

    def issue_dni_to_user(self, user, user_data: dict) -> CredentialIssuance:
        wallet = self._get_user_wallet(user)
        connection = self._establish_connection(wallet)

        result = self.credential_service.issue_credential(
            connection.connection_id, user_data
        )

        # Bug 2 corregido: wallet= en lugar de user=
        record = CredentialIssuance.objects.create(
            wallet=wallet,
            connection=connection,
            credential_exchange_id=result['credential_exchange_id'],
            credential_definition_id=result['credential_definition_id'],
            state=result['state'],
            attributes=user_data,
        )
        return record

    def _get_user_wallet(self, user) -> Wallet:
        wallets = user.wallets.all()
        if not wallets.exists():
            raise ValueError(f"El usuario {user} no tiene wallet registrado.")
        return wallets.first()

    def _establish_connection(self, holder_wallet: Wallet) -> Connection:
        """
        Flujo completo issuer → holder:
        1. Issuer crea invitación
        2. Holder la recibe
        3. Se espera que la conexión esté activa
        """
        existing = Connection.objects.filter(
            wallet=holder_wallet, state='complete'
        ).first()
        if existing:
            return existing

        from django.conf import settings
        issuer_client = ACApyClient(settings.ACA_PY_AGENTS['issuer'])
        inv_data = issuer_client.create_invitation(alias=f"holder-{holder_wallet.wallet_id}")
        issuer_conn_id = inv_data['connection_id']

        # Bug 3 corregido: ya no llama create_connection_for_user (inexistente)
        holder_client = ACApyClient(holder_wallet.agent_admin_url)
        holder_client.receive_invitation(inv_data['invitation'])

        connection = Connection.objects.create(
            wallet=holder_wallet,
            connection_id=issuer_conn_id,
            their_label='issuer',
            state='invited',
            invitation_url=inv_data.get('invitation_url', ''),
        )
        self._wait_for_active(issuer_client, issuer_conn_id)
        connection.state = 'complete'
        connection.save()
        return connection

    def _wait_for_active(self, client: ACApyClient, conn_id: str):
        start = time.time()
        while time.time() - start < POLL_TIMEOUT:
            data = client.get_connection(conn_id)
            if data.get('state') == 'active':
                return
            time.sleep(POLL_INTERVAL)
        raise TimeoutError(f"Conexión {conn_id} no llegó a 'active' en {POLL_TIMEOUT}s")

    def get_user_credentials(self, user, state: str = None):
        wallet = self._get_user_wallet(user)
        qs = CredentialIssuance.objects.filter(wallet=wallet)
        if state:
            qs = qs.filter(state=state)
        return qs
```

**Los tres bugs de Task 2 corregidos:**

| Bug | Archivo original | Fix |
|---|---|---|
| `from .wallet_service import ACApyClient` | `credential_service.py:2` | `from aca_py.client import ACApyClient` (Paso 5) |
| `user=user` en `CredentialIssuance.objects.create()` | `dni_service.py:24` | `wallet=wallet` |
| `create_connection_for_user()` no existe | `dni_service.py:50` | Reemplazado por `_establish_connection()` |

---

## Paso 7 — Nombres canónicos de schemas (Task 3)

**Archivos:** `services/credential_service.py` (ya aplicado en Paso 5) + `credential_test/connect_and_issue.py`

Tabla canónica que debe ser consistente en todo el sistema:

| Entidad | `schema_name` | `schema_version` | Atributos clave |
|---|---|---|---|
| Usuario | `user_credential` | `2.0` | `nombres`, `apellidos`, `fecha_nacimiento`, `can_ride` |
| eVTOL | `evtol_credential` | `3.0` | `id_puerto`, `state`, `version`, `name`, `can_fly` |
| Vertiport | `vertiport_credential` | `4.0` | `location`, `capacity`, `state`, `is_active` |

El atributo `can_ride`/`can_fly`/`is_active` es el flag que los contratos Solidity leen. Si el nombre no coincide entre schema y contrato, el bridge pasa el valor incorrecto en silencio.

Buscar y actualizar en:
- `credential_test/connect_and_issue.py` — cambiar `"eVTOL_Credential"` / `"eVTOL_Cred"` al nombre canónico `evtol_credential`
- Comentarios en `BESU_project/smart_contracts/` — actualizar para consistencia

---

## Paso 8 — Smoke test de imports

```bash
cd SSI_App
source venv/bin/activate
python -c "from aca_py.client import ACApyClient; print('client OK')"
python -c "from services.wallet_service import WalletService; print('wallet_service OK')"
python -c "from services.credential_service import CredentialService; print('credential_service OK')"
python -c "from services.dni_service import DNIService; print('dni_service OK')"
```

Los cuatro deben imprimir `OK`.

---

## Roadmap Tasks 4–8

| Task | Qué hacer en Architecture A |
|---|---|
| **4 — bridge.py** | Crear `SSI_App/services/bridge.py`. Fase 1: `WalletService.create_wallet(entity, agent_key)` para cada entidad. Fase 2: `DNIService._establish_connection()` generalizado. Fase 3: `web3.py` llama contratos Besu con DID/atributos. |
| **5 — Docker Compose 6 agentes** | Extender `credential_test/docker-compose-agents.yml` con 6 servicios. Cada uno con su par de puertos y `--auto-accept-invites --auto-accept-requests`. Template ya probado en 2 agentes. |
| **6 — Modelo Vertiport** | Agregar `Vertiport` a `auth_app/models.py` siguiendo patrón de `EVTOL`. Agregar `GenericRelation` para wallets. Correr migration. |
| **7 — Credential views** | Descomentar `credential_views.py` y rutas en `wallet/urls.py`. Configurar `--webhook-url http://localhost:8000/api/webhook/` en docker-compose de agentes. |
| **8 — Contract addresses dinámicos** | `deploy_system.js` escribe `deployed_addresses.json`. `smoke_test_full_system.js` lo lee con `require('./deployed_addresses.json')`. |

---

## Orden de arranque del sistema completo

```
1. VON Network:      cd SSI_Project && ./manage start --wait
2. 6 agentes ACA-Py: cd SSI_App/credential_test && docker-compose -f docker-compose-agents.yml up
3. Django:           cd SSI_App && source venv/bin/activate && python manage.py runserver
4. Besu network:     cd BESU_project && ./run.sh
5. Deploy contratos: cd BESU_project/smart_contracts && node scripts/deploy_system.js
```

## Nota de entorno

El venv está en `SSI_App/venv/`. Siempre activar antes de correr comandos Django:

```bash
source /home/egrm23/Documentos/TI3/SSI_App/venv/bin/activate
```

Paquetes instalados: `Django==5.2.7`, `djangorestframework==3.16.1`, `python-dotenv==1.1.1`, `requests==2.32.5`, `Pillow==12.0.0`.
Los paquetes ACA-Py (`aries-askar`, `indy-credx`, `indy-vdr`) corren dentro de los contenedores Docker — no necesitan estar instalados en el venv local.
