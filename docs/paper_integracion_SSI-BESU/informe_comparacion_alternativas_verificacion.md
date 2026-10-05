# Informe comparativo de alternativas de integración SSI–Besu

## Evaluación para autorización de reservas de vuelos UAM

## Resumen ejecutivo

El sistema actual verifica las credenciales AnonCreds fuera de la cadena mediante ACA-Py y Django. Después, Django firma una atestación que el contrato Besu acepta mediante `ecrecover`. Este patrón funciona, pero concentra la decisión en un único Trusted Verifier.

Las alternativas evaluadas son:

1. Django distribuido con quorum 3-de-5.
2. Verificación directa de presentaciones AnonCreds basadas en firmas CL y pruebas de conocimiento cero.
3. Indy-Besu como registro SSI alternativo.
4. W3C VC/JWT con firmas `secp256k1`.

No son cuatro mecanismos equivalentes. Cada uno actúa en una parte distinta de la arquitectura:

```text
Registro SSI       → Indy-Besu
Verificación       → Django quorum, CL/ZK o JWT
Autorización       → contrato de reservas
Transporte         → bridge, llamadas RPC o transacción on-chain
```

La comparación correcta es entre **patrones completos de integración** para una misma operación UAM, no entre tecnologías aisladas.

La conclusión principal es:

> Django 3-de-5 es la alternativa más viable a corto plazo; CL/ZK ofrece las mejores garantías potenciales de privacidad y confianza; Indy-Besu mejora principalmente el registro y la infraestructura; W3C VC/JWT ofrece la mejor ruta intermedia para verificar directamente en Besu con un coste de migración moderado.

---

## 1. Problema que se compara

La operación de referencia es una reserva de vuelo.

El usuario intenta ejecutar:

```text
createReservation(user, origin, destination, evtol)
```

La autorización debe comprobar que:

1. la credencial fue emitida por un issuer aceptado;
2. la credencial corresponde al usuario;
3. `can_ride = true`;
4. la credencial no está revocada;
5. la prueba o decisión no puede reutilizarse;
6. Besu puede ejecutar la reserva de manera verificable.

El problema no es solamente verificar una firma. También hay que resolver:

- dónde viven DIDs, schemas y credential definitions;
- quién interpreta la política de autorización;
- cómo se gestiona la revocación;
- cómo se comunica el resultado a Besu;
- quién puede modificar o falsificar la decisión;
- qué información se revela;
- qué ocurre cuando fallan los componentes.

---

## 2. Capas funcionales del problema

| Capa | Función | Pregunta que debe responder |
|---|---|---|
| P1. Registro | DIDs, schemas, cred defs y revocación | ¿Dónde se encuentran los datos públicos necesarios? |
| P2. Verificación | Firma o prueba de posesión | ¿Quién comprueba que la credencial es válida? |
| P3. Política | `can_ride`, expiración y revocación | ¿Quién decide si el usuario cumple la política? |
| P4. Integración | Resultado hacia Besu | ¿Cómo se transforma la validación en una autorización? |
| P5. Confianza | Emisor, verificadores y operadores | ¿Cuántas entidades deben comprometerse para atacar el sistema? |
| P6. Disponibilidad | Fallos y recuperación | ¿Puede seguir funcionando si fallan componentes? |
| P7. Privacidad | Exposición y correlación | ¿Qué información recibe cada participante? |
| P8. Operación | Complejidad y mantenimiento | ¿Qué esfuerzo requiere implementar y operar la solución? |

---

## 3. Naturaleza de las alternativas

### 3.1 Django distribuido con quorum 3-de-5

Es un mecanismo de verificación y autorización externa.

```text
Holder
  ↓
ACA-Py verifica AnonCreds
  ↓
Cinco verificadores independientes
  ↓
Tres aprobaciones mínimas
  ↓
Firma múltiple o atestación de quorum
  ↓
Besu
```

El quorum debe configurarse como 3 de 5. Tres respuestas válidas autorizan; dos o menos no autorizan.

#### Qué cubre

- P2: sí, pero fuera de la cadena.
- P3: sí, fuera de la cadena.
- P4: sí, mediante firmas o atestaciones.
- P5: parcialmente, porque requiere comprometer tres verificadores.
- P6: sí, tolera hasta dos verificadores caídos si los tres restantes responden.

#### Qué no cubre por sí solo

- P1: sigue dependiendo de Indy/VON u otro registro.
- P7: los verificadores pueden observar la presentación.
- P5 completamente: la independencia real depende de operadores, claves y datos independientes.

#### Modelo de seguridad

La seguridad depende de:

- independencia de las cinco claves;
- independencia operativa de los verificadores;
- correcta validación de la credencial en cada nodo;
- prevención de replay;
- control de las claves autorizadas en Besu;
- quorum correctamente implementado.

Si los cinco procesos están en el mismo servidor y comparten la misma base de datos, el experimento demuestra tolerancia de software, pero no descentralización institucional.

#### Evaluación

Es la opción más compatible con el proyecto actual y la que tiene menor esfuerzo de migración. No elimina la confianza en verificadores, pero la distribuye.

---

### 3.2 Verificación directa AnonCreds CL/ZK

Es un mecanismo de verificación criptográfica on-chain.

```text
Issuer emite AnonCreds
  ↓
Holder genera una presentación ZK
  ↓
Besu verifica la prueba
  ↓
El contrato ejecuta la reserva
```

AnonCreds utiliza firmas Camenisch-Lysyanskaya y pruebas de conocimiento cero para divulgación selectiva, predicados y no revocación.

#### Qué cubre

- P2: sí, dentro del contrato o de un verificador on-chain.
- P3: sí, si la política está incluida en la prueba o en el contrato.
- P4: sí, porque la propia prueba llega al contrato.
- P5: mejora sustancialmente la confianza, ya que no depende de un verificador que firme el resultado.
- P7: ofrece la mejor privacidad potencial.

#### Qué no cubre por sí solo

- P1: necesita resolver el schema, la credential definition y la clave del issuer.
- Revocación: requiere integrar y actualizar información de revocación.
- Disponibilidad: depende del ledger y de la generación de la prueba.
- Correctitud del circuito o verificador.
- Procedencia de las claves públicas del issuer.

#### Viabilidad técnica

La verificación de credenciales CL dentro de contratos EVM ha sido estudiada y prototipada. El artículo de Muth et al. demuestra viabilidad técnica, pero documenta barreras de gas, operaciones criptográficas, revocación y producción:

[Towards Smart Contract-based Verification of Anonymous Credentials](https://eprint.iacr.org/2022/492.pdf)

La especificación oficial de AnonCreds documenta las pruebas ZK basadas en firmas CL:

[AnonCreds Specification](https://hyperledger.github.io/anoncreds-spec/)

#### Evaluación

Es la alternativa con mejores propiedades de privacidad y menor dependencia de servidores intermediarios, pero también la más compleja. En el alcance actual debe evaluarse con una combinación de:

- mediciones reales de generación y verificación AnonCreds;
- análisis del coste de transportar y verificar la prueba;
- reproducción o estudio de una implementación CL on-chain publicada;
- identificación explícita de los componentes aún no implementados.

No debe confundirse la prueba ZK nativa de AnonCreds con un zkSNARK genérico. Un circuito Groth16 o PLONK que demuestre la verificación CL sería una capa adicional.

---

### 3.3 Indy-Besu como registro SSI

Indy-Besu es una alternativa de infraestructura y registro, no un mecanismo completo de autorización.

```text
Indy-Besu
  ├── DIDs
  ├── schemas
  ├── credential definitions
  ├── revocación
  └── contratos de permisos y registro

ACA-Py/VDR
  └── consulta y uso de los objetos SSI

Besu UAM
  └── reservas y estados de vuelos
```

El proyecto oficial define Indy-Besu como sustituto del ledger Indy/Plenum con soporte para DIDs y registros AnonCreds:

[Hyperledger Indy-Besu](https://github.com/hyperledger-indy/indy-besu)

#### Qué cubre

- P1: sí, proporcionando registros SSI en una red EVM permissioned.
- Parte de P4: puede reducir la separación entre las redes SSI y de negocio.
- P6: depende del consenso y de la distribución de validadores.
- P8: puede simplificar la infraestructura si se adopta como plataforma principal.

#### Qué no cubre por sí solo

- P2: no verifica automáticamente la presentación AnonCreds.
- P3: no decide `can_ride` ni la política de reserva por sí mismo.
- P4 completamente: todavía se necesita adaptar ACA-Py/VDR y los contratos.
- P5 completamente: una red con varios validadores operados por una organización sigue siendo centralizada institucionalmente.
- P7: depende del formato de credencial y del mecanismo de verificación.

#### Evaluación

Indy-Besu debe compararse como reemplazo del registro VON Indy, manteniendo constante el mecanismo de verificación:

```text
VON Indy + Django 3-de-5 + Besu UAM
```

contra:

```text
Indy-Besu + Django 3-de-5 + Besu UAM
```

De esta forma se mide la influencia real del ledger en:

- latencia de resolución;
- coste de registro;
- disponibilidad;
- complejidad operativa;
- adaptación del cliente;
- propagación de objetos SSI.

La combinación Indy-Besu + CL/ZK queda como trabajo futuro porque requiere resolver primero la integración completa entre el VDR, las credential definitions y el verificador on-chain.

---

### 3.4 W3C VC/JWT con `secp256k1`

Es una alternativa de formato y verificación criptográfica directamente compatible con la EVM.

```text
Issuer firma una W3C VC/JWT
  ↓
Holder presenta el JWT
  ↓
Besu recupera el signer con ecrecover
  ↓
El contrato aplica la política
  ↓
Se crea la reserva
```

#### Qué cubre

- P2: sí, on-chain mediante la firma del issuer.
- P3: sí, mediante claims y lógica del contrato.
- P4: sí, con una llamada directa al contrato.
- P8: mejor que CL/ZK porque utiliza primitivas conocidas por la EVM.

#### Qué no cubre por sí solo

- P1: requiere resolver el issuer y el estado de revocación.
- P7: revela más información si se envía el JWT completo.
- Revocación: necesita listas, registros o estados externos.
- Divulgación selectiva avanzada: no es equivalente a AnonCreds.
- P5: el issuer sigue siendo una autoridad de confianza.

#### Evaluación

Es una alternativa de transición atractiva. Reduce la dependencia de Django como firmante intermedio y evita implementar verificación CL, pero exige cambiar el formato de credencial y diseñar cuidadosamente:

- `iss`;
- `aud`;
- `exp`;
- `iat`;
- `nonce`;
- identificación del issuer autorizado;
- revocación;
- prevención de replay.

---

## 4. Matriz de cobertura

| Propiedad | Django 3-de-5 | CL/ZK directo | Indy-Besu | JWT/secp256k1 |
|---|---|---|---|---|
| Registro de DIDs | Requiere ledger externo | Requiere ledger externo | Proporciona registro | Requiere registro o issuer resoluble |
| Registro de schemas | Requiere ledger externo | Requiere ledger externo | Proporciona registro | Opcional, según diseño |
| Registro de cred defs | Requiere ledger externo | Necesario para verificar CL | Proporciona registro | No necesariamente necesario para JWT |
| Verificación de issuer | Off-chain | On-chain o verificador ZK | No automática | On-chain |
| Aplicación de `can_ride` | Verificadores | Circuito/contrato | No automática | Contrato |
| Verificación de revocación | Off-chain o atestación | Compleja, puede ser on-chain | Registra estado, no decide por sí sola | Externa o registro adicional |
| Privacidad | Baja/media | Alta | Depende del verificador | Media |
| Dependencia de oráculo | Sí, aunque distribuida | No para la prueba | Puede seguir existiendo | No necesariamente para firma, sí para revocación |
| Tolerancia a fallos | Hasta dos verificadores | Depende de ledger/prover | Depende de validadores | Depende de issuer/registro |
| Verificación EVM nativa | Firma de quorum | No trivial | No por sí sola | Sí |
| Compatibilidad ACA-Py | Alta | Alta en emisión; difícil on-chain | Requiere adaptador | Requiere migración de formato |
| Esfuerzo de migración | Bajo/medio | Muy alto | Alto | Medio |
| Madurez para el proyecto | Alta | Baja/media | Experimental | Media |

---

## 5. Qué aspectos son realmente comparables

### 5.1 Correctitud

Todas las alternativas deben aceptar una credencial válida y rechazar una credencial modificada, falsa, revocada o incompatible con la política.

Casos comunes:

1. `can_ride = true`, issuer válido, no revocada.
2. `can_ride = false`.
3. firma modificada.
4. issuer no autorizado.
5. credencial revocada.
6. nonce repetido.
7. credencial expirada.

### 5.2 Seguridad

Comparar:

- número de entidades que deben comprometerse;
- dependencia de claves privadas;
- posibilidad de falsificar una autorización;
- resistencia a replay;
- trazabilidad de la decisión;
- dependencia de un issuer central.

### 5.3 Privacidad

Comparar qué observa:

- el holder;
- ACA-Py;
- Django o verificadores;
- el bridge;
- el nodo RPC;
- los validadores Besu.

CL/ZK puede revelar solo una afirmación. JWT normalmente revela los claims incluidos. Django quorum puede recibir la presentación completa.

### 5.4 Rendimiento

Medir en todos los casos comparables:

- generación de la prueba o firma;
- verificación;
- resolución del registro;
- coordinación entre verificadores;
- envío de transacción;
- confirmación de Besu;
- CPU y memoria;
- throughput.

### 5.5 Coste on-chain

Medir:

- gas usado;
- bytes de calldata;
- número de firmas;
- número de transacciones;
- almacenamiento persistente;
- lecturas de contratos de registro.

No mezclar directamente gas de Besu con costes de Indy/VON. En VON se deben reportar tiempo y recursos; en Besu, gas y recursos.

### 5.6 Tolerancia a fallos

Comparar fallos equivalentes:

- un verificador caído;
- dos verificadores caídos;
- un validador caído;
- dos validadores caídos;
- issuer no disponible;
- ledger no disponible;
- RPC no disponible.

### 5.7 Esfuerzo de migración

Medir o documentar:

- número de componentes nuevos;
- archivos modificados;
- cambios en schemas;
- cambios en contratos;
- cambios en ACA-Py;
- nuevas dependencias;
- nuevos procesos operativos;
- necesidad de cambiar la wallet o el formato de credencial.

---

## 6. Cómo hacer una comparación justa

### 6.1 Mantener constante el caso de negocio

Todas las alternativas deben autorizar la misma operación:

```text
Usuario rider-001
can_ride = true
Origen VP-ORIGIN
Destino VP-DESTINATION
EVTOL_ID = 1
```

El contrato de reserva, la red Besu, el hardware y la política deben mantenerse constantes.

### 6.2 Cambiar una dimensión por vez

Para evaluar Indy-Besu:

```text
VON Indy + quorum + Besu UAM
```

contra:

```text
Indy-Besu + quorum + Besu UAM
```

Para evaluar JWT:

```text
JWT + Besu UAM
```

contra:

```text
AnonCreds + quorum + Besu UAM
```

Para evaluar CL/ZK:

```text
AnonCreds verificado off-chain
```

contra:

```text
AnonCreds con prueba verificada on-chain
```

Si la segunda variante no está implementada, debe compararse mediante resultados bibliográficos y un análisis de integración, no mediante métricas inventadas.

### 6.3 Separar evidencia experimental y bibliográfica

Usar tres categorías:

| Categoría | Significado |
|---|---|
| Medido | Resultado obtenido ejecutando el proyecto |
| Reproducido | Resultado obtenido siguiendo una implementación externa o referencia publicada |
| Bibliográfico | Resultado tomado de literatura técnica sin reproducción local |

CL/ZK probablemente tendrá resultados medidos para generación/verificación off-chain y resultados bibliográficos para verificación CL on-chain.

---

## 7. Evaluación resumida por alternativa

### Django distribuido

#### Fortalezas

- Mayor compatibilidad con el sistema existente.
- Se puede implementar completamente.
- Permite medir tolerancia a fallos real.
- Mantiene las credenciales AnonCreds actuales.
- Migración gradual.

#### Debilidades

- Los verificadores siguen siendo confiables.
- La presentación se verifica off-chain.
- Privacidad limitada frente a CL/ZK.
- La independencia depende de operadores diferentes.

#### Papel en el paper

Debe ser la alternativa de referencia práctica y el baseline de comparación.

### CL/ZK directo

#### Fortalezas

- Privacidad fuerte.
- Verificación matemáticamente auditable.
- Menor dependencia de oráculos.
- La política puede quedar ligada a la prueba.

#### Debilidades

- Verificación costosa.
- Integración compleja con EVM.
- Revocación difícil.
- Dependencia de parámetros criptográficos, circuitos y claves públicas.
- Implementación completa fuera del alcance inmediato.

#### Papel en el paper

Debe representar el límite de descentralización y privacidad, respaldado por pruebas off-chain, literatura y análisis de viabilidad on-chain.

### Indy-Besu

#### Fortalezas

- Registro SSI sobre infraestructura EVM.
- Menor separación entre ledger SSI y Besu.
- Contratos específicos para DID, schema, credential definition y revocación.
- Experimentos reales disponibles en el repositorio local.

#### Debilidades

- No verifica automáticamente las presentaciones.
- Requiere adaptar VDR/ACA-Py.
- No elimina por sí solo el bridge.
- Su madurez y compatibilidad deben evaluarse experimentalmente.

#### Papel en el paper

Debe presentarse como alternativa de registro e infraestructura, no como sustituto completo del mecanismo de verificación.

### W3C VC/JWT

#### Fortalezas

- Verificación directa con `ecrecover`.
- Menor coste de integración con contratos.
- Menor complejidad que CL/ZK.
- Puede reducir la dependencia de Django.

#### Debilidades

- Requiere cambiar el formato de credencial.
- Menor privacidad si se revelan todos los claims.
- Revocación no resuelta automáticamente.
- Confianza permanente en el issuer.

#### Papel en el paper

Debe representar una alternativa intermedia entre el oráculo distribuido y la verificación ZK.

---

## 8. Interpretación de los resultados

No se debe elegir una alternativa únicamente por la menor latencia o el menor gas.

Una alternativa puede ser rápida y poco privada. Otra puede ser segura y demasiado costosa. Otra puede simplificar el ledger, pero no mejorar la verificación.

Se recomienda presentar:

1. una tabla de resultados experimentales;
2. una matriz cualitativa de seguridad y privacidad;
3. una frontera de Pareto;
4. una matriz ponderada solo si los pesos se justifican antes del cálculo.

### Ejemplo de pesos

| Criterio | Peso recomendado |
|---|---:|
| Seguridad criptográfica | 25 % |
| Privacidad | 20 % |
| Tolerancia a fallos | 15 % |
| Rendimiento | 15 % |
| Compatibilidad ACA-Py | 15 % |
| Esfuerzo de migración | 10 % |

Los pesos no deben modificarse después de observar qué alternativa obtiene el mejor resultado.

---

## 9. Conclusiones

### Conclusión 1: no existe una solución única dominante

Las alternativas optimizan objetivos diferentes. Compararlas como si todas resolvieran el mismo componente llevaría a conclusiones incorrectas.

### Conclusión 2: Django 3-de-5 es la solución inmediata más viable

Mantiene AnonCreds, ACA-Py y la arquitectura actual. Reduce el punto único de fallo y permite pruebas reproducibles de disponibilidad y tolerancia a fallos.

Su limitación principal es que continúa dependiendo de verificadores externos y no ofrece privacidad completa.

### Conclusión 3: CL/ZK tiene el mejor potencial de privacidad y confianza

La verificación directa puede eliminar el Trusted Verifier como intermediario y permitir que Besu valide una afirmación criptográfica. Sin embargo, los costes, la revocación y la integración con la EVM hacen que sea una línea de investigación avanzada, no la solución inmediata del sistema.

### Conclusión 4: Indy-Besu es un habilitador de infraestructura

Indy-Besu puede simplificar el registro SSI y acercarlo a la infraestructura Besu, pero no resuelve por sí mismo la verificación de AnonCreds ni elimina automáticamente el bridge.

La evaluación experimental debe centrarse en registro, resolución, latencia, disponibilidad, coste y adaptación del cliente.

### Conclusión 5: JWT/secp256k1 es la transición más práctica hacia verificación on-chain

Permite aprovechar `ecrecover` y reduce el coste de integración frente a CL/ZK. El precio es una menor privacidad y la necesidad de migrar desde AnonCreds o mantener dos formatos.

### Conclusión 6: la arquitectura futura más completa es compuesta

La solución de largo plazo podría combinar:

```text
Indy-Besu para el registro SSI
        +
CL/ZK o una prueba compatible con EVM
        +
revocación verificable on-chain
        +
contratos UAM con política explícita
```

Esta combinación no debe presentarse como implementada. Debe quedar como arquitectura futura derivada de la evaluación.

---

## 10. Resultado que debe defender el paper

El paper debe defender una conclusión condicional:

> Para una implementación inmediata y compatible con el sistema existente, el quorum Django 3-de-5 ofrece el mejor equilibrio entre esfuerzo, compatibilidad y tolerancia a fallos. Para una arquitectura con máxima privacidad y menor dependencia de servidores, la verificación directa CL/ZK es conceptualmente superior, pero actualmente tiene mayor coste y complejidad. Indy-Besu mejora la ubicación y operación del registro SSI, pero debe combinarse con un mecanismo de verificación. W3C VC/JWT con `secp256k1` constituye una alternativa intermedia con verificación on-chain más sencilla y una migración más accesible que CL/ZK.

La afirmación central no es que una tecnología gane en todos los criterios. Es que la mejor arquitectura depende del peso que el sistema UAM otorgue a privacidad, confianza, disponibilidad, rendimiento y esfuerzo de adopción.

