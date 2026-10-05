# Plan reproducible del paper

## Evaluación de patrones de integración entre SSI y Besu para reservar vuelos UAM

## 1. Propósito del documento

Este documento define el problema, las alternativas, las hipótesis y el protocolo experimental del paper. Está escrito para que cualquiera pueda reproducir las pruebas sin tener que decidir parámetros durante la ejecución.

La operación común de todas las pruebas será la misma:

> Un usuario intenta reservar un vuelo eVTOL. El sistema debe comprobar que posee una credencial válida, que `can_ride = true`, que no está revocada y que el contrato Besu acepta o rechaza la reserva.

Las alternativas comparadas serán:

1. Django distribuido con quorum 3-de-5.
2. Verificación directa de presentaciones AnonCreds basadas en firmas CL y pruebas de conocimiento cero.
3. Indy-Besu como registro SSI alternativo, evaluado experimentalmente sin combinarlo todavía con verificación directa.
4. W3C VC/JWT con firmas `secp256k1`.

La comparación no tratará estas alternativas como si fueran componentes equivalentes. Django y JWT son mecanismos de verificación; Indy-Besu es principalmente una alternativa de registro e infraestructura; CL/ZK es una alternativa de verificación criptográfica directa.

---

## 2. Problema de investigación

El sistema actual utiliza un Trusted Verifier en Django:

```text
Holder presenta una credencial AnonCreds
        ↓
ACA-Py/Django verifica fuera de la cadena
        ↓
Django firma una atestación
        ↓
Besu verifica la firma con ecrecover()
        ↓
Se crea la reserva
```

Este diseño es compatible con el sistema actual, pero concentra la confianza en Django y en su clave privada. El contrato Besu no verifica la presentación AnonCreds original ni puede auditar por sí mismo cómo se obtuvo la decisión.

### Pregunta principal

> En un sistema UAM basado en ACA-Py, AnonCreds y Besu, ¿qué patrón de integración para autorizar reservas de vuelo ofrece el mejor compromiso entre seguridad criptográfica, privacidad, tolerancia a fallos, rendimiento, compatibilidad con SSI y esfuerzo de migración?

### Preguntas secundarias

1. ¿Cuánto mejora un quorum 3-de-5 la disponibilidad y la resistencia al fallo de un Trusted Verifier único?
2. ¿Qué parte de la verificación AnonCreds puede trasladarse directamente a Besu mediante pruebas CL/ZK?
3. ¿Qué coste y complejidad introduce Indy-Besu como reemplazo del registro VON Indy?
4. ¿Qué ventajas de rendimiento e integración ofrece W3C VC/JWT con `secp256k1`?
5. ¿Qué problemas permanecen sin resolver en cada alternativa, especialmente revocación, procedencia de claves y dependencia del issuer?

---

## 3. Descomposición del problema

| Código | Subproblema | Pregunta experimental |
|---|---|---|
| R1 | Registro SSI | ¿Dónde se resuelven DIDs, schemas, credential definitions y revocación? |
| R2 | Verificación criptográfica | ¿Quién verifica la firma o prueba de la credencial? |
| R3 | Política | ¿Quién comprueba `can_ride = true` y no revocación? |
| R4 | Comunicación | ¿Cómo llega el resultado o la prueba al contrato? |
| R5 | Confianza | ¿Cuántas entidades deben comprometerse para autorizar una reserva falsa? |
| R6 | Disponibilidad | ¿Qué ocurre cuando fallan verificadores, validadores o el ledger? |
| R7 | Privacidad | ¿Qué datos observa cada componente y qué queda en Besu? |
| R8 | Operación | ¿Cuál es el coste de implementación, configuración y migración? |

---

## 4. Hipótesis

### H1: Django distribuido

Un quorum 3-de-5 mejora la disponibilidad y elimina el punto único de fallo de Django, pero mantiene la confianza en los operadores y no proporciona verificación on-chain de la presentación AnonCreds.

### H2: CL/ZK directo

La verificación directa de una presentación AnonCreds ofrece mejores garantías de privacidad y reduce la dependencia de verificadores externos, pero presenta mayor complejidad, coste computacional y dificultad de integración con la EVM.

### H3: Indy-Besu

Indy-Besu reduce la separación entre el registro SSI y la red Besu, pero no sustituye por sí mismo la verificación de presentaciones ni elimina automáticamente la capa de integración con ACA-Py.

### H4: W3C VC/JWT

W3C VC/JWT con `secp256k1` proporciona verificación directa compatible con `ecrecover` y menor esfuerzo de integración que CL/ZK, a cambio de menor privacidad y de una migración desde AnonCreds.

---

## 5. Entorno común obligatorio

Todas las pruebas deben ejecutarse en el mismo equipo y con la misma configuración base.

### Componentes

- `SSI_Project`: VON Indy, si se usa como registro de referencia.
- `SSI_App`: Django, ACA-Py, bridge y scripts de prueba.
- `BESU_project`: red Besu de negocio y contratos UAM.
- `indy-besu`: red alternativa de registro SSI.
- Python 3.12 o la versión declarada por el entorno.
- Node.js y npm de las versiones instaladas en el repositorio.
- Docker y Docker Compose.

### Caso de negocio fijo

Usar siempre los siguientes datos lógicos:

```json
{
  "rider_id": "rider-001",
  "can_ride": true,
  "credential_schema": "user_credential",
  "credential_version": "2.0",
  "origin": "VP-ORIGIN",
  "destination": "VP-DESTINATION",
  "evtol_id": 1,
  "trip_prefix": "PAPER-TRIP"
}
```

Cada ejecución debe generar un `trip_id` único con el formato:

```text
PAPER-TRIP-<timestamp>-<run_number>
```

No se deben cambiar el schema, los atributos, los vertiports, el eVTOL ni la política entre alternativas.

### Estado inicial

Antes de cada bloque de pruebas:

1. El usuario no debe tener una reserva activa.
2. El eVTOL debe estar en estado `PARKED`.
3. El vertiport de origen debe tener al menos un parking libre.
4. El vertiport destino debe estar registrado.
5. El contrato de reservas debe estar desplegado.
6. El registro de credenciales debe contener el schema y la credential definition necesarias.

Si una prueba modifica el estado de forma irreversible, se debe reiniciar el escenario antes de la siguiente repetición.

### Registro obligatorio por ejecución

Cada ejecución debe registrar en CSV o JSON:

```text
experiment_id
alternative
run_id
timestamp_start
timestamp_end
validity_case
concurrency
verifier_count
quorum
credential_schema
credential_id_or_hash
trip_id
rpc_endpoint
chain_id
transaction_hash
transaction_status
block_number
gas_used
calldata_bytes
verification_result
reservation_result
error_type
error_message
```

También deben conservarse los logs de Django, ACA-Py, Besu y los verificadores.

---

## 6. Métricas comunes

### Correctitud

- `true positive`: credencial válida y reserva aceptada.
- `true negative`: credencial inválida/revocada y reserva rechazada.
- `false positive`: credencial inválida y reserva aceptada.
- `false negative`: credencial válida y reserva rechazada.

### Rendimiento

- Latencia end-to-end: desde la solicitud de reserva hasta la confirmación de la transacción.
- Latencia de verificación: desde la recepción de la presentación hasta la decisión.
- Latencia de confirmación de Besu.
- Percentiles p50, p95 y p99.
- Throughput: reservas completadas por segundo.
- CPU y memoria del verificador.
- CPU y memoria del nodo RPC.

### Coste blockchain

- `gas_used` por transacción.
- Tamaño de calldata.
- Número de transacciones adicionales.
- Número de lecturas del registro SSI.
- Número de firmas enviadas al contrato.

### Disponibilidad

- Tasa de solicitudes completadas.
- Tiempo de recuperación después de fallos.
- Número de fallos tolerados antes de rechazar una reserva válida.

### Seguridad y privacidad

- Número de entidades que deben coludirse para autorizar una reserva falsa.
- Datos visibles para los verificadores.
- Datos almacenados en eventos o estado de Besu.
- Exposición de identificadores correlacionables.
- Protección contra replay, expiración y sustitución de atributos.

---

## 7. Experimento A: Django distribuido con quorum 3-de-5

### Objetivo

Evaluar si cinco verificadores simulados con quorum 3-de-5 mejoran la disponibilidad y la tolerancia a fallos frente al Trusted Verifier único actual.

### Arquitectura

Crear cinco procesos independientes:

```text
verifier-1
verifier-2
verifier-3
verifier-4
verifier-5
```

Cada proceso debe tener:

- una clave privada propia;
- un endpoint HTTP propio;
- un identificador propio;
- logs propios;
- la misma lógica de validación;
- acceso al mismo caso de credencial de prueba.

En esta fase los procesos pueden ejecutarse en la misma máquina. El paper debe indicar que esto mide tolerancia técnica, no independencia institucional.

### Decisión de quorum

Aceptar la reserva únicamente si se obtienen al menos 3 respuestas válidas de 5.

El contrato debe verificar tres firmas válidas de verificadores autorizados o una firma agregada equivalente. No se debe aceptar una respuesta basada en el número de respuestas HTTP solamente.

La firma debe incluir como mínimo:

```text
domain = "UAM_RESERVATION"
chain_id
rider
can_ride
credential_hash
trip_id
nonce
expiration
```

### Casos funcionales

Ejecutar exactamente 10 repeticiones por caso, reiniciando el escenario después de cada repetición.

| Caso | Verificadores | Respuesta esperada |
|---|---:|---|
| A1: todos aprueban | 5 válidos | Reserva aceptada |
| A2: un verificador caído | 4 disponibles | Reserva aceptada |
| A3: dos verificadores caídos | 3 disponibles | Reserva aceptada |
| A4: tres verificadores caídos | 2 disponibles | Reserva rechazada |
| A5: un verificador devuelve rechazo | 4 aprobaciones, 1 rechazo | Reserva aceptada |
| A6: dos verificadores devuelven rechazo | 3 aprobaciones, 2 rechazos | Reserva aceptada |
| A7: tres verificadores devuelven rechazo | 2 aprobaciones, 3 rechazos | Reserva rechazada |
| A8: una firma inválida | 4 válidas, 1 inválida | Reserva aceptada |
| A9: tres firmas inválidas | 2 válidas, 3 inválidas | Reserva rechazada |
| A10: nonce repetido | firma ya utilizada | Reserva rechazada |
| A11: atestación expirada | timestamp vencido | Reserva rechazada |
| A12: atributo alterado | firma para `false`, solicitud `true` | Reserva rechazada |

Total funcional: 120 ejecuciones.

### Prueba de rendimiento

Usar únicamente el caso A1, con cinco verificadores activos y respuesta válida.

Ejecutar tres niveles de concurrencia:

| Nivel | Solicitudes simultáneas | Repeticiones del bloque |
|---|---:|---:|
| A-P1 | 1 | 3 bloques |
| A-P2 | 10 | 3 bloques |
| A-P3 | 25 | 3 bloques |
| A-P4 | 50 | 3 bloques |

Cada bloque debe procesar 100 reservas, usando eVTOLs y `trip_id` distintos. Entre bloques se debe reiniciar el estado o utilizar un conjunto limpio de entidades.

Para evitar que el registro de entidades contamine la medición:

1. Antes de iniciar el cronómetro, preparar 100 fixtures independientes.
2. El fixture `i` debe usar `EVTOL_ID = 1000 + i`, `ORIGIN = VP-ORIGIN-<block>-<i>` y `DESTINATION = VP-DEST-<block>-<i>`.
3. Registrar los 100 vertiports y los 100 eVTOLs antes de comenzar el bloque.
4. Emitir o cargar las atestaciones antes de comenzar el bloque.
5. Iniciar el cronómetro únicamente al enviar la solicitud de reserva.
6. No reutilizar un eVTOL, vertiport o `trip_id` dentro del mismo bloque.
7. Registrar por separado el tiempo de preparación de fixtures, pero no incluirlo en la latencia de la reserva.

Total de rendimiento: 1.200 reservas.

Reportar p50, p95, p99, throughput, errores, gas medio y desviación estándar.

### Comparación contra baseline

Repetir A-P1, A-P2, A-P3 y A-P4 con el Trusted Verifier único actual. El caso de negocio, hardware, carga y contrato deben ser iguales.

La comparación principal será:

```text
Trusted Verifier único
vs.
quorum 3-de-5
```

---

## 8. Experimento B: AnonCreds CL/ZK

### Objetivo

Medir la generación y verificación de presentaciones AnonCreds y documentar con evidencia técnica qué impide trasladar toda la verificación al contrato Besu.

La verificación directa completa puede quedar como evaluación bibliográfica y análisis de PoC si no existe un verificador CL/ZK integrado en Solidity.

### Preparación

Usar el schema de usuario actual:

```text
name: user_credential
version: 2.0
attributes: nombres, apellidos, fecha_nacimiento, can_ride
```

Emitir una credencial con:

```json
{
  "nombres": "Usuario",
  "apellidos": "Prueba",
  "fecha_nacimiento": "1990-01-01",
  "can_ride": "true"
}
```

La credencial debe ser emitida por el issuer del entorno y almacenada por el holder.

### Casos de presentación

Ejecutar 30 repeticiones independientes por caso.

| Caso | Presentación | Resultado esperado |
|---|---|---|
| B1 | Revela únicamente `can_ride=true` | Válida |
| B2 | Revela `can_ride=false` | Rechazada por política |
| B3 | Demuestra `can_ride=true` y oculta los demás atributos | Válida |
| B4 | Presentación modificada | Inválida |
| B5 | Credential definition incorrecta | Inválida |
| B6 | Schema incorrecto | Inválida |
| B7 | Credencial revocada | Rechazada |
| B8 | Prueba de no revocación válida | Válida |

Total: 240 presentaciones.

### Métricas off-chain

Para cada presentación registrar:

- tiempo de generación;
- tiempo de verificación;
- tamaño en bytes;
- número de atributos revelados;
- número de predicados;
- tiempo de consulta del schema;
- tiempo de consulta de la credential definition;
- tiempo de consulta de revocación;
- CPU y memoria del holder;
- CPU y memoria del verifier.

### Evaluación de integración on-chain

Documentar, sin inventar una implementación, los componentes que tendría que ejecutar Besu:

1. Resolver la credential definition.
2. Obtener la clave pública CL del issuer.
3. Verificar la prueba primaria.
4. Verificar predicados.
5. Verificar no revocación.
6. Validar dominio, nonce y expiración.
7. Ejecutar la reserva.

Comparar estas operaciones con la implementación `ecrecover` actual.

### Evidencia bibliográfica

Usar el trabajo de Muth et al. como evidencia principal. El paper debe reportar separadamente:

- resultados medidos en el entorno local para generación/verificación off-chain;
- resultados publicados para verificación CL on-chain;
- limitaciones que impiden afirmar que la implementación local es una verificación CL on-chain completa.

No se debe llamar “zkSNARK implementado” a una prueba que solo demuestra un atributo con un circuito simplificado.

---

## 9. Experimento C: Indy-Besu como registro SSI

### Objetivo

Evaluar experimentalmente Indy-Besu como reemplazo del ledger VON Indy para registrar y resolver objetos SSI. La combinación Indy-Besu + verificación directa CL/ZK queda como trabajo futuro.

### Estado conocido del entorno

La red local levantada tiene:

- cinco validadores Besu;
- RPC en `http://localhost:8545`;
- chain ID `1337`;
- contratos de DID, schema, credential definition y revocación desplegados.

### Objetos de prueba

Usar los mismos objetos lógicos del entorno actual:

1. Un DID del issuer.
2. Un schema `user_credential` versión `2.0`.
3. Una credential definition CL asociada al schema.
4. Una revocation registry.

Los identificadores deben incluir un sufijo de ejecución, por ejemplo:

```text
paper-run-001
paper-run-002
```

### Prueba C1: creación y resolución de DID

Ejecutar 30 creaciones y 30 resoluciones.

Para cada DID:

1. Crear el objeto válido.
2. Enviarlo al registro Indy-Besu.
3. Esperar confirmación de transacción.
4. Registrar hash, bloque, gas y latencia.
5. Resolverlo mediante el cliente/VDR.
6. Comparar el objeto recuperado con el objeto original.

Resultado esperado: la resolución debe devolver el objeto original o una representación equivalente documentada por el VDR.

### Prueba C2: schema

Ejecutar 30 registros y 30 resoluciones del schema `user_credential`.

Registrar:

- gas de creación;
- latencia de confirmación;
- latencia de resolución;
- tamaño del JSON;
- tamaño de calldata;
- resultado de comparación del contenido.

### Prueba C3: credential definition

Ejecutar 30 registros y 30 resoluciones de credential definitions CL.

Verificar:

- que la credential definition referencia el schema correcto;
- que el issuer DID existe;
- que la resolución devuelve la clave pública y datos esperados;
- que el objeto se puede usar como entrada de una verificación AnonCreds off-chain.

### Prueba C4: revocación

Ejecutar 30 ciclos:

1. Crear una revocation registry.
2. Publicar un estado no revocado.
3. Resolverlo.
4. Publicar un estado revocado.
5. Resolver el nuevo estado.
6. Registrar latencias y hashes de bloque.

No afirmar que esta prueba demuestra verificación completa de no revocación dentro del contrato de reservas. Solo demuestra almacenamiento y resolución del registro de revocación.

### Prueba C5: disponibilidad de validadores

Ejecutar tres repeticiones por escenario:

| Escenario | Acción |
|---|---|
| C5-1 | Cinco validadores activos |
| C5-2 | Un validador detenido |
| C5-3 | Dos validadores detenidos |
| C5-4 | Recuperación del validador detenido |

Durante cada escenario realizar 10 lecturas y 10 escrituras de objetos de prueba.

Registrar:

- éxito o fallo;
- latencia;
- altura de bloque;
- tiempo de recuperación;
- errores RPC.

### Comparación con VON Indy

Si VON Indy está disponible, repetir C1-C4 con el mismo número de operaciones y objetos equivalentes.

La comparación será sobre:

- tiempo de escritura;
- tiempo de lectura;
- coste de operación;
- disponibilidad;
- recursos de infraestructura;
- esfuerzo de adaptación del cliente.

No comparar directamente gas de Indy y gas de Besu como si fueran la misma unidad económica. Reportar gas como coste computacional dentro de Besu y reportar métricas de tiempo/recursos para VON Indy.

---

## 10. Experimento D: W3C VC/JWT con secp256k1

### Objetivo

Evaluar una credencial alternativa que pueda verificarse directamente en Besu mediante `ecrecover`.

### Credencial de prueba

Emitir un JWT con los siguientes claims mínimos:

```json
{
  "iss": "did:example:issuer",
  "sub": "rider-001",
  "can_ride": true,
  "schema": "user_credential",
  "schema_version": "2.0",
  "iat": 1700000000,
  "exp": 4102444800,
  "aud": "uam-flight-reservation",
  "nonce": "unique-per-reservation"
}
```

La firma debe producirse con una clave `secp256k1` asociada al issuer.

### Casos funcionales

Ejecutar 30 repeticiones por caso.

| Caso | Modificación | Resultado esperado |
|---|---|---|
| D1 | JWT válido | Reserva aceptada |
| D2 | Firma alterada | Rechazada |
| D3 | `can_ride=false` | Rechazada |
| D4 | `iss` no autorizado | Rechazada |
| D5 | JWT expirado | Rechazada |
| D6 | `aud` incorrecta | Rechazada |
| D7 | `nonce` repetido | Rechazada |
| D8 | Payload modificado | Rechazada |
| D9 | Schema incorrecto | Rechazada |
| D10 | JWT válido con atributos extra | Aceptada si la política no los prohíbe |

Total: 300 ejecuciones.

### Prueba de rendimiento

Usar únicamente D1.

| Nivel | Concurrencia | Bloques | Solicitudes por bloque |
|---|---:|---:|---:|
| D-P1 | 1 | 3 | 100 |
| D-P2 | 10 | 3 | 100 |
| D-P3 | 25 | 3 | 100 |
| D-P4 | 50 | 3 | 100 |

Para estos bloques usar fixtures independientes preparados antes de la medición:

```text
EVTOL_ID = 2000 + i
ORIGIN = JWT-ORIGIN-<block>-<i>
DESTINATION = JWT-DEST-<block>-<i>
TRIP_ID = JWT-TRIP-<block>-<i>
```

Los 100 fixtures deben estar registrados antes de iniciar cada bloque. La preparación se mide por separado y se excluye de la latencia de la reserva.

Registrar latencia, throughput, gas, calldata, CPU, memoria y errores.

### Comparación directa

Comparar D-P1 a D-P4 con la misma prueba de Django quorum, manteniendo:

- mismo contrato de reserva;
- mismo rider;
- misma política;
- mismo número de solicitudes;
- mismo endpoint RPC;
- mismo hardware.

---

## 11. Tabla de resultados esperada

La tabla final debe contener una fila por alternativa y columnas separadas por evidencia.

| Alternativa | Correctitud | p50 | p95 | Throughput | Gas | Privacidad | Fallos tolerados | Compatibilidad | Migración | Evidencia |
|---|---:|---:|---:|---:|---:|---|---:|---|---|---|
| Django 3-de-5 | | | | | | | | | | Experimental |
| CL/ZK directo | | | | | | | | | | Experimental + bibliográfica |
| Indy-Besu | | | | | | | | | | Experimental de registro |
| JWT/secp256k1 | | | | | | | | | | Experimental |

Para CL/ZK, no rellenar métricas on-chain como si hubieran sido medidas localmente. Usar columnas separadas:

- `medido localmente`;
- `reportado en bibliografía`;
- `estimado/no disponible`.

---

## 12. Interpretación prevista

No se debe buscar un ganador absoluto. Se debe identificar la frontera de compromiso:

- Django 3-de-5: mejor compatibilidad y disponibilidad inmediata.
- CL/ZK directo: mejor privacidad y menor confianza en servidores, con mayor complejidad y coste.
- Indy-Besu: mejor integración del registro SSI, pero no sustituye la verificación.
- JWT/secp256k1: mejor equilibrio entre verificación on-chain y esfuerzo de migración.

La evaluación debe declarar que cinco procesos locales no demuestran independencia institucional. Demuestran comportamiento del protocolo bajo fallos simulados.

---

## 13. Amenazas a la validez

### Validez interna

- La carga puede verse afectada por otros contenedores activos.
- Besu puede estar ejecutándose en modo de desarrollo.
- Los verificadores simulados pueden compartir recursos.
- El contrato actual puede no representar todas las reglas de producción.

### Validez externa

- Los resultados no representan automáticamente una red UAM real.
- El quorum 3-de-5 solo representa descentralización organizacional si los nodos pertenecen a entidades independientes.
- Los resultados de CL/ZK dependen de la implementación criptográfica y de la curva utilizada.
- Indy-Besu puede cambiar de API o madurez durante su evolución.

### Mitigación

- Publicar versiones de código y configuración.
- Registrar hardware y versiones.
- Repetir cada caso con el número indicado.
- Informar media, desviación estándar y percentiles.
- Conservar logs y hashes de transacciones.
- Separar resultados medidos de resultados bibliográficos.

---

## 14. Resultado esperado del paper

El resultado no debe afirmar que una alternativa resuelve todos los subproblemas.

La conclusión esperada es:

> El quorum 3-de-5 es la alternativa más viable a corto plazo porque mantiene AnonCreds y ACA-Py, reduce el punto único de fallo y puede evaluarse completamente. La verificación directa CL/ZK proporciona las mejores garantías de privacidad y confianza criptográfica, pero presenta barreras de coste, revocación e integración con la EVM documentadas en la literatura. Indy-Besu mejora el registro y la integración de infraestructura, pero no reemplaza el mecanismo de verificación. W3C VC/JWT con `secp256k1` ofrece una alternativa de transición con verificación directa en Besu y menor esfuerzo que CL/ZK.

La combinación futura más completa sería:

```text
Indy-Besu como registro
        +
CL/ZK o una prueba compatible con EVM
        +
revocación publicada y verificable
```

Esta combinación queda explícitamente fuera del alcance experimental principal y se presenta como trabajo futuro.
