# Fix the ACApyClient Core Layer — Implementation Guide

**Date:** 2026-04-21
**Source:** Task 1 from `05_prioritized_remaining_functions.md`

---

## Question

The `aca_py/client.py` file has only 3 methods (`create_wallet`, `create_invitation`, `send_message`) and no `_make_request()` base method. Every service in `SSI_App` calls `self.client._make_request()` which does not exist. How should this be implemented step by step?

---

## Answer

### Step 1 — Understand what callers expect from `_make_request()`

Read each call site across the three services:

| Caller | Call | What it expects back |
|---|---|---|
| `credential_service.py:22` | `_make_request('POST', '/schemas', payload)` | `{'schema_id': ...}` |
| `credential_service.py:32` | `_make_request('POST', '/credential-definitions', payload)` | `{'credential_definition_id': ...}` |
| `credential_service.py:53` | `_make_request('POST', '/issue-credential/send', payload)` | `{'credential_exchange_id': ..., 'state': ...}` |
| `credential_service.py:58` | `_make_request('GET', '/schemas/created')` | `{'schema_ids': [...]}` |
| `credential_service.py:69` | `_make_request('GET', '/credential-definitions/created')` | `{'credential_definition_ids': [...]}` |
| `credential_service.py:92` | `_make_request('GET', '/issue-credential/records?...')` | `{'results': [...]}` |

The signature must be: `_make_request(self, method: str, path: str, payload: dict = None) -> dict`.

---

### Step 2 — Add `_make_request()` to `client.py`

Open `SSI_App/aca_py/client.py`. Add this method **inside** the `ACApyClient` class, before `create_wallet`:

```python
def _make_request(self, method: str, path: str, payload: dict = None) -> dict:
    url = f"{self.admin_url}{path}"
    method = method.upper()

    if method == 'GET':
        response = requests.get(url, headers=self.headers, timeout=30)
    elif method == 'POST':
        response = requests.post(url, json=payload or {}, headers=self.headers, timeout=30)
    elif method == 'DELETE':
        response = requests.delete(url, headers=self.headers, timeout=30)
    else:
        raise ValueError(f"Unsupported HTTP method: {method}")

    response.raise_for_status()
    return response.json()
```

**Why `raise_for_status()`:** If ACA-Py returns a 4xx/5xx, you want a `requests.HTTPError` with the status code, not a silent `None` that crashes three frames up the stack with a confusing `AttributeError`.

**Why `timeout=30`:** ACA-Py is local but ledger writes can block waiting for Indy node consensus. 30 s is safe for dev; increase for schema/cred-def creation which involves a ledger transaction.

---

### Step 3 — Refactor the three existing methods to use `_make_request()`

The three existing methods (`create_wallet`, `create_invitation`, `send_message`) bypass `_make_request()` with inline `requests.post()` calls. Rewrite them to go through `_make_request()` so error handling is consistent:

```python
def create_wallet(self, wallet_name, wallet_key):
    payload = {"method": "sov", "options": {"key_type": "ed25519"}}
    return self._make_request('POST', '/wallet/did/create', payload)

def create_invitation(self, alias=None):
    payload = {"alias": alias} if alias else {}
    return self._make_request('POST', '/connections/create-invitation', payload)

def send_message(self, connection_id, content):
    payload = {"content": content}
    return self._make_request('POST', f"/connections/{connection_id}/send-message", payload)
```

Delete the old versions.

---

### Step 4 — Add the missing endpoint methods

`credential_service.py` calls `_make_request()` directly with raw paths. That is workable but breaks the abstraction: each service is aware of ACA-Py URL structure. The better design is named methods. Add these to `ACApyClient`:

```python
def get_schemas_created(self) -> dict:
    return self._make_request('GET', '/schemas/created')

def create_schema(self, name: str, version: str, attributes: list) -> dict:
    payload = {
        "schema_name": name,
        "schema_version": version,
        "attributes": attributes,
    }
    return self._make_request('POST', '/schemas', payload)

def get_credential_definitions_created(self) -> dict:
    return self._make_request('GET', '/credential-definitions/created')

def create_credential_definition(self, schema_id: str, tag: str = "default",
                                  support_revocation: bool = False) -> dict:
    payload = {
        "schema_id": schema_id,
        "tag": tag,
        "support_revocation": support_revocation,
    }
    return self._make_request('POST', '/credential-definitions', payload)

def send_credential_offer(self, connection_id: str, cred_def_id: str,
                           attributes: list) -> dict:
    payload = {
        "connection_id": connection_id,
        "cred_def_id": cred_def_id,
        "credential_preview": {
            "@type": "issue-credential/1.0/credential-preview",
            "attributes": attributes,
        },
        "auto_issue": True,
        "auto_remove": False,
    }
    return self._make_request('POST', '/issue-credential/send-offer', payload)

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

def receive_invitation(self, invitation: dict) -> dict:
    return self._make_request('POST', '/connections/receive-invitation', invitation)

def get_connection(self, connection_id: str) -> dict:
    return self._make_request('GET', f"/connections/{connection_id}")

def get_connections(self) -> dict:
    return self._make_request('GET', '/connections')
```

**Why `send-offer` instead of `send`:** `connect_and_issue.py` (the only proven working script) uses `POST /issue-credential/send-offer` with `auto_issue: True`. The `POST /issue-credential/send` endpoint bypasses the offer/request handshake and requires auto-accept flags on the holder agent. Using `send-offer` is more robust and matches the working test.

---

### Step 5 — Fix the wrong import in `credential_service.py`

Line 2 of `credential_service.py` reads:

```python
from .wallet_service import ACApyClient   # WRONG — ACApyClient is not in wallet_service
```

Change it to:

```python
from aca_py.client import ACApyClient
```

This is a direct import from the module where `ACApyClient` lives, matching the pattern already used correctly in `wallet_service.py:2`.

---

### Step 6 — Update `credential_service.py` to call named methods instead of raw `_make_request()`

Now that the named methods exist, update `credential_service.py` to call them. This avoids exposing ACA-Py URL paths in business logic:

- `self.client._make_request('POST', '/schemas', payload)` → `self.client.create_schema(schema_name, schema_version, attributes)`
- `self.client._make_request('POST', '/credential-definitions', payload)` → `self.client.create_credential_definition(schema_id)`
- `self.client._make_request('GET', '/schemas/created')` → `self.client.get_schemas_created()`
- `self.client._make_request('GET', '/credential-definitions/created')` → `self.client.get_credential_definitions_created()`
- `self.client._make_request('POST', '/issue-credential/send', payload)` → `self.client.send_credential_offer(connection_id, credential_definition_id, attributes)`
- `self.client._make_request('GET', endpoint)` in `get_credential_records` → `self.client.get_issue_credential_records(connection_id, state)`

This cleans up `credential_service.py` considerably and makes the service methods readable without knowing ACA-Py's API surface.

---

### Step 7 — Verify with a smoke test (no running agent needed)

Before starting any Docker container, confirm there are no import errors:

```bash
cd SSI_App
python -c "from aca_py.client import ACApyClient; print('import OK')"
python -c "from services.credential_service import CredentialService; print('import OK')"
python -c "from services.wallet_service import WalletService; print('import OK')"
python -c "from services.dni_service import DNIService; print('import OK')"
```

All four should print `import OK`. If any raises `ImportError` or `AttributeError`, that pinpoints the remaining issue before touching any running system.

---

## What This Unlocks

After these steps:
- The entire service layer becomes importable and instantiable.
- `WalletService.create_wallet()` can reach ACA-Py.
- `CredentialService.get_or_create_dni_credential_def()` can call ACA-Py to check/create schemas and cred-defs.
- The Django user registration endpoint (`POST /api/user/`) can complete the full wallet creation path without crashing at `_make_request`.

Tasks 2 and 3 from `05_prioritized_remaining_functions.md` (the three runtime bugs and schema unification) must be fixed next before the credential issuance pipeline produces correct database records.
