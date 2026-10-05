# Resultados del experimento C: Indy-Besu como registro SSI

## Alcance

Este experimento evalúa Indy-Besu como registro de objetos SSI: DID, schema, credential definition y estados de revocación. No implementa ni mide la verificación criptográfica CL de una presentación dentro de un contrato Besu.

La ejecución reproducible se conserva en `indy-besu/smart_contracts/demos/experiment-c.ts`; los datos crudos están en `indy-besu/smart_contracts/results/experiment_c/paper-c-20261005T154000Z/raw.json` y el resumen en `summary.json`.

## Entorno

- RPC: `http://127.0.0.1:8545`.
- Chain ID: `1337`.
- Consenso: QBFT.
- Contratos desplegados de Indy-Besu para DID, schema, credential definition y revocación.
- 30 repeticiones secuenciales por C1--C4.
- Cada escritura espera su recibo de confirmación; cada lectura se ejecuta después de esa confirmación.

Los identificadores creados incluyen el sufijo `paper-c-20261005T154000Z`, por lo que no sobrescriben registros previos.

## Correctitud funcional

| Prueba | Escrituras correctas | Lecturas y comparación correctas |
|---|---:|---:|
| C1: DID | 30/30 | 30/30 |
| C2: schema `user_credential` 2.0 | 30/30 | 30/30 |
| C3: credential definition CL | 30/30 | 30/30 |
| C4: definición de revocación | 30/30 | -- |
| C4: estado activo | 30/30 | 30/30 |
| C4: estado revocado | 30/30 | 30/30 |

La comparación de lecturas fue de igualdad exacta del JSON enviado y recuperado. En C4, la lectura posterior al estado activo recuperó dicho estado y la lectura posterior al estado revocado recuperó el nuevo estado publicado.

## Métricas

Las latencias son milisegundos. Gas es una unidad de ejecución de la EVM y no una medida directa de energía.

| Operación | Confirmación p50 | Lectura p50 | Gas p50 | Calldata p50 (B) | JSON p50 (B) |
|---|---:|---:|---:|---:|---:|
| C1, crear DID | 4144.60 | 14.20 | 628543 | 772 | -- |
| C2, registrar schema | 4156.38 | 13.19 | 435117 | 580 | 308 |
| C3, registrar cred. definition CL | 4242.72 | 19.04 | 4380324 | 6116 | 5799 |
| C4, crear definición de revocación | 4195.69 | -- | 1152257 | 900 | 590 |
| C4, publicar estado activo | 4150.23 | 25.66 | 202137 | 484 | 209 |
| C4, publicar estado revocado | 4149.42 | 38.23 | 185037 | 484 | 209 |

La resolución de un estado de revocación ya actualizado tarda más que la resolución tras la primera entrada porque el cliente recupera la secuencia de entradas del registro. Esto mide el mecanismo de almacenamiento y recuperación del registro, no una prueba de no revocación en el contrato de reservas.

## C3 y compatibilidad AnonCreds

La carga de C3 fue la credential definition CL real del entorno ACA-Py/VON (`5rtvexuq6mGGkvikxen2qK:3:CL:10:case-b`), preservada byte a byte dentro del registro Indy-Besu. Las 30 recuperaciones conservaron el contenido y la clave pública primaria CL.

La carga directa con la biblioteca AnonCreds actual falló porque el objeto emitido por la VON está en formato legado y no contiene `issuerId`. Al añadir únicamente ese campo, derivable del DID del issuer, `CredentialDefinition.load` terminó correctamente. Esto prueba compatibilidad estructural después de una normalización mínima; no prueba una verificación extremo a extremo de las presentaciones existentes. Estas siguen referenciando identificadores de la VON y requieren un adaptador VDR y una emisión/migración coherente para resolverse desde Indy-Besu.

## C5: disponibilidad

Se ejecutaron artefactos diagnósticos C5 con 10 lecturas y 10 escrituras, tres repeticiones por escenario. Durante esa verificación se determinó que la red local tiene cinco contenedores, pero **solo cuatro direcciones en el conjunto QBFT activo**. El quinto contenedor es un nodo adicional y no era un validador del quórum inicial.

Por ello, los ficheros `c5-*.json` no se usarán como evidencia de tolerancia a fallos de una red de cinco validadores. C5 debe repetirse seleccionando fallos entre los cuatro miembros devueltos por `qbft_getValidatorsByBlockNumber`, o incorporando formalmente el quinto nodo al conjunto QBFT antes de medir.

Al cerrar esta ejecución, los cinco contenedores estaban restaurados y el conjunto QBFT activo continuaba con cuatro validadores. Esta observación evita atribuir indebidamente los timeouts observados a la tolerancia Byzantine de una red de cinco miembros.

### Repetición corregida sobre el quórum QBFT real

La red local tiene cuatro validadores QBFT en el génesis (`validator1` a `validator4`). C5 se repitió usando únicamente esos miembros, con tres repeticiones de diez lecturas y diez escrituras por escenario. Cada escritura usó una identidad endorser distinta; así, las transacciones pendientes de una prueba sin quórum no bloquearon el nonce de otra escritura.

| Escenario | Validadores QBFT activos | Lecturas correctas | Escrituras confirmadas en 10 s | p50 lectura (ms) | p50 escritura (ms) |
|---|---:|---:|---:|---:|---:|
| C5Q4-1 | 4/4 | 30/30 | 30/30 | 16.45 | 4157.62 |
| C5Q4-2 | 3/4 | 30/30 | 21/30 | 14.59 | 8157.23 |
| C5Q4-3 | 2/4 | 30/30 | 0/30 | 15.50 | 10002.0 (límite) |
| C5Q4-4 | 4/4 recuperados | 30/30 | 30/30 | 14.62 | 4212.23 |

Con tres de cuatro validadores, el quórum siguió produciendo bloques, pero nueve escrituras excedieron el límite de confirmación de diez segundos. Con dos de cuatro no se confirmó ninguna escritura dentro del límite, mientras las lecturas del último estado confirmado siguieron disponibles mediante RPC. Tras recuperar ambos validadores, todas las operaciones volvieron a confirmar.

Los artefactos de esta repetición son `indy-besu/smart_contracts/results/experiment_c/paper-c-qbft4-final-20261005T133000Z/` y `paper-c-recovery-clean-20261005T161500Z/`.

## Comparación VON Indy: C1--C3

La VON estuvo disponible en `http://localhost:9000` y el issuer ACA-Py en el puerto 8131. Se ejecutaron 30 registros y 30 resoluciones por tipo. Los resultados están en `SSI_App/agents/case_c/results/paper-von-c-20261005T171000Z/`.

| Operación VON | Escrituras | Lecturas | p50 escritura (ms) | p50 lectura (ms) |
|---|---:|---:|---:|---:|
| C1, DID | 30/30 | 30/30 | 2980.85 | 27.49 |
| C2, schema `user_credential` 2.0 | 30/30 | 30/30 | 2992.74 | 16.45 |
| C3, credential definition CL | 30/30 | 30/30 | 2999.51 | 12.99 |

En C2, la VON/ACA-Py devolvió los mismos cuatro atributos, pero con un orden normalizado. La validación considera el conjunto de atributos, que es la semántica del schema AnonCreds; por tanto las 30 resoluciones son correctas.

C4 VON no se declara completado con estos datos. Una revocación efectiva exige emitir una credencial revocable, obtener su `cred_rev_id`, publicar la revocación y resolver la entrada resultante. Cambiar administrativamente el estado de un registro no demostraría ese flujo criptográfico.

## Interpretación

Indy-Besu registró y resolvió todos los objetos SSI evaluados de manera consistente. Esto apoya su función como sustituto experimental del ledger de registro de Indy y como base para que los contratos Besu y los clientes SSI compartan infraestructura.

Los resultados no muestran que Indy-Besu elimine el trusted verifier. El registro pone a disposición DID, schemas, credential definitions y entradas de revocación; todavía hace falta un verificador de presentaciones, ya sea off-chain con quórum, CL/ZK on-chain o una alternativa como VC/JWT con `secp256k1`.

## C6: integración SSI--Besu en una sola infraestructura

### Pregunta y criterio

C1--C4 demuestran que Indy-Besu puede almacenar objetos SSI; no responden por sí solos qué cambia cuando una operación de un contrato necesita esos objetos. C6 evalúa exclusivamente el criterio arquitectónico definido para esta comparación: **la integración de SSI y Besu dentro de una misma infraestructura**.

Se contrastaron dos recorridos que consumen el mismo tipo de objeto, un schema AnonCreds previamente registrado:

1. **VON + puente + Besu.** Un servicio externo consulta el schema en ACA-Py/VON mediante HTTP (`GET /schemas/{id}`), serializa su respuesta, calcula `keccak256` y envía ese hash a Besu. El contrato `IntegrationRegistryBenchmark` solo registra el hash recibido; no puede comprobar por sí mismo que proviene de la VON.
2. **Indy-Besu directo.** El contrato `IntegrationRegistryBenchmark` invoca en la misma transacción al contrato Indy-Besu `SchemaRegistry` (`0x0000000000000000000000000000000000005555`), recupera el schema por su identificador, calcula `keccak256` de los bytes recuperados y exige igualdad con el hash esperado. No hay consulta HTTP a ACA-Py/VON ni proceso puente que suministre el contenido al contrato.

Ambos recorridos terminan en una transacción Besu exitosa y usan schemas reales obtenidos de C2. Por tanto son comparables para dependencia externa, composición contrato--registro, gas y tiempo de operación. No son una prueba de verificación CL: ninguno verifica una firma AnonCreds, una presentación ni una prueba de no revocación.

### Protocolo reproducible

El ejecutor es `indy-besu/smart_contracts/demos/benchmark-integration-c.ts` y el contrato auxiliar es `indy-besu/smart_contracts/contracts/IntegrationRegistryBenchmark.sol`.

- Se usaron los schemas ya creados en C2 de Indy-Besu y VON, evitando modificar los schemas de la evaluación previa.
- Se hicieron 30 operaciones correctas por ruta, divididas en ocho lotes de 4, 4, 4, 3, 4, 4, 4 y 3 operaciones.
- En cada lote las transacciones se enviaron con nonces consecutivos y se confirmaron en el mismo bloque cuando el planificador de QBFT lo permitió. Este diseño se adoptó porque Besu limita los nonces futuros de una cuenta; la ejecución estrictamente una por bloque no cabe en el límite temporal del entorno.
- Se registraron: éxito del recibo, tiempo de consulta HTTP (solo VON), tiempo desde el envío hasta el recibo, tiempo de extremo a extremo, gas, hash de transacción y bloque.
- Los resultados agregados están en `indy-besu/smart_contracts/results/experiment_c/integration-c-20261005T185000Z-aggregate/`. Sus fuentes son los ocho directorios `...-p1` a `...-p8`.

### Resultados

| Ruta | Operaciones correctas | Consulta SSI externa p50 (ms) | Confirmación p50 (ms) | Extremo a extremo p50 (ms) | Gas p50 |
|---|---:|---:|---:|---:|---:|
| VON + puente + Besu | 30/30 | 22.35 | 4151.41 | 4297.43 | 23246 |
| Indy-Besu directo | 30/30 | -- | 4163.69 | 4163.69 | 62901 |

La confirmación depende de la posición de cada transacción en el intervalo de bloque QBFT. Por ello existen muestras de menos de un segundo cuando la transacción llegó justo antes de la propuesta de bloque; los percentiles de confirmación no deben interpretarse como una ventaja de rendimiento de una arquitectura sobre la otra. El resultado consistente de latencia es que la ruta VON añade una dependencia HTTP externa de 19.17 a 30.99 ms (p50 22.35 ms), mientras la resolución Indy-Besu se ejecuta completamente dentro de la transacción.

La composición directa cuesta 62,901 gas, 39,655 gas más que registrar una referencia ya resuelta por el puente (23,246 gas), aproximadamente 2.71 veces. Ese coste adicional corresponde a leer el registro y calcular/verificar el hash en la EVM. El resultado expresa un intercambio concreto: Indy-Besu desplaza la comprobación de disponibilidad e integridad del schema al contrato y elimina el puente para esa resolución, pero no hace gratuita la integración on-chain.

### Interpretación y límite de la conclusión

C6 aporta evidencia a favor de Indy-Besu bajo el criterio solicitado: un contrato Besu puede recuperar y comprobar un objeto SSI sin confiar en un proceso que le transmita una referencia externa. En VON, Besu solo ve un hash aportado por el puente; un contrato no puede distinguir si el valor fue resuelto correctamente, si el puente estaba actualizado o si consultó la red SSI esperada. La diferencia es una reducción de una frontera de confianza y de una dependencia operativa, no una reducción demostrada de la latencia de confirmación ni del gas.

Por ello, los datos permiten afirmar que Indy-Besu ofrece **integración directa y verificable del registro SSI con contratos Besu**, a costa de mayor gas para la resolución on-chain. No permiten afirmar que Indy-Besu sea globalmente mejor que VON: VON tuvo registros C1--C3 más rápidos y ya tiene compatibilidad operativa inmediata con ACA-Py. Tampoco permiten concluir que el trusted verifier desaparece: para autorizar una reserva de vuelo aún falta verificar criptográficamente la presentación del titular y su revocación. C6 sirve como evidencia experimental del habilitador arquitectónico que puede combinarse en el futuro con CL/ZK on-chain o con el quórum de verificadores.
