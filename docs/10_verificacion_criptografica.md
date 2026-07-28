# Verificación criptográfica de credenciales SSI on-chain

Este documento describe el estado actual de la verificación de credenciales en los contratos Besu,
la solución implementada (Opción A), y la hoja de ruta hacia una verificación completamente
descentralizada (Opciones B y C).

---

## Estado actual — Opción A: Trusted Verifier ✅ IMPLEMENTADO

### Qué hace

Django actúa como **verificador de confianza**: cuando el bridge necesita registrar una entidad
en Besu, primero le pide a Django que firme una **atestación Ethereum** (EIP-191) que dice:
"yo, el sistema SSI, he verificado esta credencial y confirmo sus datos".

```
ACA-Py verifica credencial SSI (off-chain)
           ↓
Bridge llama POST /api/attest/<tipo>/
           ↓
Django firma con clave secp256k1 (eth_account)
           ↓
Bridge pasa la firma al contrato Besu
           ↓
Contrato usa ecrecover() para verificar la firma
           ↓
Solo acepta si la firma proviene de TRUSTED_VERIFIER_ADDRESS
```

### Dónde está el código

| Capa | Archivo | Qué hace |
|---|---|---|
| Django | `auth_app/services/attestation_service.py` | Genera firmas EIP-191 con `eth_account` |
| Django | `auth_app/views/attestation_views.py` | Endpoints `POST /api/attest/<tipo>/` |
| Solidity | `UserVerification.sol` · `VertiportManagement.sol` · `EVTOLManagement.sol` | `_verifyAttestation()` con `ecrecover()` |
| Bridge | `bridge/bridge.py` | `get_user/vertiport/evtol_attestation()` |

### Qué verifica on-chain

| Función del contrato | Mensaje firmado (keccak256 de...) |
|---|---|
| `UserVerification.setRiderPermission(user, canRide, sig)` | `"user" \|\| user_address \|\| can_ride` |
| `VertiportManagement.registerVertiport(..., sig)` | `"vertiport" \|\| vertiport_id` |
| `EVTOLManagement.registerEVTOL(..., sig)` | `"evtol" \|\| evtol_id` |

### Limitación

El punto de confianza es **centralizado**: quien controla la clave `TRUSTED_VERIFIER_KEY`
puede firmar cualquier atestación, incluso para entidades ficticias. El contrato no
puede distinguir entre una credencial SSI real y una inventada — solo verifica que
Django la haya "aprobado". En producción, esta clave debería vivir en un HSM (Hardware
Security Module) y firmarse solo después de que ACA-Py complete la presentación de prueba.

En el prototipo actual, la clave de despliegue y la clave del Trusted Verifier son la
misma (`0x8f2a55...`). En producción deben ser claves separadas con roles distintos.

---

## Trabajo futuro — Opción B: ZK-SNARK

### Qué resuelve

Con ZK-SNARKs, el bridge genera una **prueba matemática cero-conocimiento** que demuestra:
"tengo una credencial SSI válida firmada por el issuer con DID X, con `can_ride=true`, sin
revelar ningún otro dato de la credencial". El contrato Besu verifica esta prueba en
microsegundos usando precompilados EVM (BN254 o BLS12-381).

### Por qué es difícil

Las credenciales AnonCreds (Indy) usan **firmas CL** (Camenisch-Lysyanskaar), que son
incompatibles directamente con los circuitos ZK-SNARK estándar. Implementar la Opción B
requeriría:

1. Definir un circuito ZK que modele la verificación de una firma CL (o migrar a BBS+)
2. Usar herramientas como **Circom** + **snarkjs** o **gnark** (Go)
3. Generar la clave de prueba (trusted setup o PLONK universal)
4. Desplegar el verificador Solidity generado automáticamente
5. Integrar la generación de pruebas en el bridge

**Esfuerzo estimado:** 3-6 meses de trabajo especializado en ZK.

**Proyectos de referencia:** Polygon ID, Iden3, zkCreds, Sismo Protocol.

---

## Trabajo futuro — Opción C: Credenciales con secp256k1 (BBS+ o W3C JWT)

### Qué resuelve

En vez de usar firmas CL (que son incompatibles con EVM), se emitiría la credencial SSI
con una curva criptográfica que Ethereum ya soporta nativamente: **secp256k1** o **BLS12-381**.

**Opción C1 — JWT/W3C VC con secp256k1:**
ACA-Py puede emitir credenciales en formato W3C VC firmadas como JWTs con `ES256K`
(secp256k1). El bridge extraería el encabezado + payload + firma, y el contrato usaría
`ecrecover()` directamente sobre la firma del JWT para verificar que el issuer DID firmó
la credencial. **Sin intermediario.**

**Opción C2 — BBS+ signatures:**
BBS+ permite firmas de múltiples mensajes + presentaciones selectivas (zero-knowledge
parcial). La EVM puede verificar BBS+ con el precompilado BLS12-381 (EIP-2537). AnonCreds v2
(Hyperledger) ya migra hacia BBS+.

### Por qué es más alcanzable que la Opción B

- No requiere circuitos ZK ni trusted setup
- `ecrecover()` ya existe en todos los nodos EVM
- Solo requiere cambiar el formato de credencial en ACA-Py (W3C VC en vez de AnonCreds)
- La verificación on-chain es O(1) en gas, igual que una verificación de firma normal

**Esfuerzo estimado Opción C1:** 3-6 semanas. Requiere configurar ACA-Py para emitir W3C VC
con DID key (did:key o did:web), modificar el bridge para parsear el JWT, y adaptar los
contratos Solidity para verificar la firma del issuer.

---

## Comparativa de opciones

| | Opción A (actual) | Opción B (ZK) | Opción C (secp256k1) |
|---|---|---|---|
| **Verificación on-chain** | Firma del backend | Prueba ZK | Firma del issuer directamente |
| **Punto de confianza** | Backend centralizado | Ninguno (trustless) | Issuer DID |
| **Privacidad** | Baja | Alta (ZK) | Media (C1) / Alta (C2 BBS+) |
| **Complejidad** | Baja | Muy alta | Media |
| **Gas cost** | Bajo (~3k gas, ecrecover) | Medio (~300k gas, verificador ZK) | Bajo (~3k gas, ecrecover) |
| **Compatibilidad con AnonCreds** | ✅ | Requiere adaptación | Requiere migrar a W3C VC |
| **Estado en TI3** | ✅ Implementado | Trabajo futuro | Trabajo futuro |

---

## Recomendación para siguientes versiones

La secuencia natural de evolución es:

```
Opción A (ahora) → Opción C1 (próximo sprint) → Opción C2/B (investigación futura)
```

La Opción C1 (W3C VC + JWT + ecrecover) es el siguiente paso más alcanzable y elimina
la necesidad de un backend intermediario para la verificación, manteniendo la
arquitectura actual del sistema casi intacta.
