# Idempotencia y compensaciones: los cinco mecanismos del método Keel

**Respuesta corta:** repetir y deshacer no son un problema, son **cinco problemas distintos**, y Keel tiene un mecanismo separado para cada uno. Confundirlos es el fallo más caro del método, porque el diseño valida en verde y el servidor se rompe la primera vez que algo va mal:

1. **Un cliente HTTP reintenta** → `use-cases.<op>.idempotency`
2. **El broker reentrega** → el id del mensaje (`metadata.eventId` si la fuente es Keel; `subscriptions.<E>.contract.messageId` si no) — o, mejor, una `transitions` irrepetible
3. **Nosotros reintentamos contra un proveedor** → `http-clients.calls.<x>.idempotency`
4. **Hay que deshacer trabajo ya encargado** → `dependencies.<d>.compensations`
5. **El desenlace no llega nunca** → `activations.<a>.reconciledBy`

> **Idea clave:** son cinco mecanismos con **cinco tablas o disparadores distintos**. Declarar el de un eje contra la repetición de otro no protege nada — y nada lo delata hasta que llega el duplicado.

Notas hermanas: [`declarar-dependencias-entre-servidores.md`](declarar-dependencias-entre-servidores.md) (el plano de la razón) y [`capas-http-clients-y-messaging.md`](capas-http-clients-y-messaging.md) (el plano del canal). Esta nota es el plano de la **garantía**.

---

## 0. La tabla maestra

| # | Qué repite / qué se deshace | Se declara en | Se materializa en | Naturaleza de la guarda |
|---|---|---|---|---|
| 1 | Reintento del llamante HTTP | `use-cases.<op>.idempotency` | tabla `idempotency_record` + `CommandSignature` | **de puerta** (el filtro HTTP) |
| 2 | Reentrega del broker | `metadata.eventId` de la envoltura Keel, o `subscriptions.<E>.contract.messageId` si la fuente es ajena | tabla `processed_event` + `IdempotencyGuard` | **de puerta** (el listener) |
| 2b | *(la alternativa que vale para todo)* | `use-cases.<op>.transitions` irrepetible | el propio agregado | **de dominio** |
| 3 | Nuestro reintento contra un proveedor | `http-clients.calls.<x>.idempotency` | `OutboundIdempotency` → cabecera saliente | **del otro lado** (la honra el proveedor) |
| 4 | Deshacer trabajo ya encargado | `dependencies.<d>.compensations` | **ninguna clase propia** | la del agregado |
| 5 | El desenlace que nunca llega | `activations.<a>.reconciledBy` | operación `@Scheduled` (un barrido) | — |

Y una sexta pieza que no es una guarda pero es la que hace que el problema 2 exista:

| 6 | Que un evento no se pierda al publicar | `messaging.reliability: outbox` | tabla `outbox_event` + `OutboxRelay` | — |

### El ejemplo que se usa en toda la nota

`stock-reservation`, la fixture del repo (`packages/keel-spring/test/fixtures/stock-reservation/`). Existe exactamente para esto — su manifiesto lo dice:

```yaml
# specs/stock-reservation/service.keel.yaml
keel: "2.6"
# Diseño MÍNIMO cuya única razón de ser es la cadena de compensación e idempotencia,
# que es lo que el resto de fixtures ejercita de refilón y ninguna prueba entera:
#
#   - idempotencia de PETICIÓN  → `createReservation` con `client-key`
#   - idempotencia SALIENTE     → `inventory.reserveStock` con retry + payload-hash
#   - idempotencia de CONSUMO   → `StockRejected` por el eventId de la envoltura Keel
#   - COMPENSACIÓN              → `releaseReservation`, con su guarda de dominio,
#                                 su estado de vuelta y su reconciliación
```

La historia: **reservamos stock para un pedido**. El bloqueo físico lo hace `inventory`, otro servidor. Confirmar la reserva le encarga el bloqueo; si el almacén rechaza ese bloqueo *después*, hay que liberar la reserva. Y si el almacén nunca dice nada, alguien tiene que darse cuenta.

---

# Parte I — El plano del diseño

## 1. Los tres ejes de repetición

Antes de elegir un mecanismo hay que responder una sola pregunta: **¿quién repite?** La doctrina está fijada en `structural-decisions.md` §3.2 de la skill `keel-design`:

| Quién repite | Mecanismo | Dónde se declara |
|---|---|---|
| Un llamante HTTP que reintenta (timeout, doble clic) | clave que **él** manda en una cabecera | `use-cases.<op>.idempotency` |
| El broker, que reentrega el mismo mensaje | id del mensaje, o irrepetibilidad en el dominio | `subscriptions.<E>.contract.messageId`, o `use-cases.<op>.transitions` |
| **Nosotros**, reintentando contra un proveedor | clave que **le mandamos a él** | `http-clients.calls.<x>.idempotency` |

> **Idea clave:** el tercer eje es el que más se olvida, *porque el reintento parece resiliencia y no repetición*. Un `retry: { maxAttempts: 3 }` sobre un `POST /charges` no es robustez: son tres cobros.

La pregunta que resuelve el tercero es literal: *«si esta llamada se manda dos veces, ¿el proveedor hace el trabajo dos veces?»*. Si la respuesta es sí y él ofrece cabecera de idempotencia, se declara; si no la ofrece, se escribe en el `contract` — **es la deuda que después tendrá que ir a limpiar una compensación**.

---

## 2. Eje 1 — `use-cases.<op>.idempotency`: el reintento del llamante

### El problema

Un cliente hace `POST /reservations`, se le agota el timeout, reintenta. Sin nada, hay dos reservas del mismo pedido.

> **Consecuencia observable de no declararla** (`structural-decisions.md` §3.2): *en una red real, con reintentos, el duplicado no es probable — es seguro. La pregunta es cuándo, no si.*

### El YAML

```yaml
# specs/stock-reservation/use-cases.keel.yaml
  createReservation:
    description: Registra la intención de reservar stock para un pedido.
    kind: command
    # El eje de repetición del LLAMANTE: un cliente con timeout reintenta el POST y
    # no puede acabar con dos reservas del mismo pedido. La clave la genera él.
    idempotency:
      keySource: client-key
      ttlSeconds: 3600
```

### El schema exacto

`common.schema.json` → `$defs.idempotency`:

| Campo | Tipo | Obligatorio | Valores |
|---|---|---|---|
| `keySource` | string | ✅ | `client-key` \| `payload-hash` |
| `ttlSeconds` | integer ≥ 1 | | sin default en el schema (el generador usa `86400` si falta) |

`additionalProperties: false`. No hay nada más.

### Cómo se elige `keySource`

| | Cuándo | Trampa |
|---|---|---|
| `client-key` | El llamante puede generar y **repetir** un identificador de intento | Ninguna: es el caso robusto |
| `payload-hash` | No puede, pero el mismo cuerpo significa la misma intención | **Un payload con `timestamp`, `requestId` o un uuid del cliente**: dos envíos idénticos dan hashes distintos y no deduplica nada |

En **ambos casos** la clave entra por la superficie HTTP, en la cabecera `Idempotency-Key`.

### Lo que exige `keel validate`

| Regla | Severidad | Por qué |
|---|---|---|
| La operación tiene que ser alcanzable por HTTP | **error** | La clave llega por una cabecera; una operación interna o disparada por evento no tiene esa puerta |
| Salida `list` / `paginated` con idempotencia | aviso | De la primera ejecución solo se guarda el id del recurso: una repetición no puede devolver la **misma** respuesta, porque una lista depende del estado del resto del sistema |

El mensaje de error de `crossrefs.js` es didáctico a propósito: si la operación la dispara una suscripción, dice que *la reentrega se ataja con `contract.messageId`*; si no la invoca nadie externo, dice que *lo que evita el efecto doble es la clave natural en `persistence` o una transición de lifecycle irrepetible*.

### Dos matices que se dan por sabidos y no lo son

**Declararla no la hace obligatoria.** Si el cliente no manda la cabecera, el generador **ejecuta la operación sin deduplicar** — no la rechaza. Rechazarla exigiría un `code` público y este campo no lo declara. Si el contrato es que la cabecera sea obligatoria, se declara como un error más:

```yaml
    errors:
      - code: IDEMPOTENCY_KEY_REQUIRED
        when: La petición no trae la cabecera Idempotency-Key.
        http: 400
```

**El contrato no es rechazar el duplicado: es devolver la respuesta original.** Mismo status, mismo cuerpo, sin segundo efecto. Lo que se guarda de la primera ejecución es el **id del recurso creado**, y con eso se reconstruye la ficha de una entidad. Por eso una lista no vale.

---

## 3. Eje 2 — `contract.messageId`: la reentrega del broker

### El problema

La entrega de **cualquier** broker es at-least-once. Y si la suscripción declara `retry`, se está pidiendo explícitamente que el mismo mensaje llegue más de una vez.

### Primero, deshacer un malentendido: aquí «header» no es HTTP

`messageId` se declara con un `location`, y el enum tiene exactamente dos valores porque un mensaje de broker tiene exactamente dos partes:

| `location` | Qué es | Ejemplos reales |
|---|---|---|
| `header` | **Metadato nativo del mensaje**, fuera del cuerpo | header de Kafka, atributo de mensaje de SQS, property de AMQP |
| `field` | Un campo **del cuerpo** (dot-path si está anidado) | `meta.id`, `messageId` |

No hay ninguna cabecera HTTP en este eje. La palabra está tomada del vocabulario de los brokers, no del de la web — y esa colisión es la causa número uno de que este campo se lea mal.

### Y segundo: con `envelope: keel` **no se declara nada**

Si la fuente es otro servicio Keel, el mensaje ya viene con la envoltura estándar (`metadata` + `data`) y **la identidad del mensaje es `metadata.eventId`**. Tiene una garantía escrita en la constitución del generador:

> La `EventMetadata` se estampa **una vez**, en el `raise`, y viaja intacta hasta el wire: su `eventId` es la clave de idempotencia del consumidor. Regenerarla aguas abajo rompe la deduplicación.

Es decir: **una reentrega repite el mismo `eventId`**, y el consumidor deduplica por él sin que el diseño diga nada. Las skills de broker se lo prescriben literalmente al agente (`keel-spring-kafka/references/implementation.md`): *«el `messageId` de la suscripción o, si no lo hay, `envelope.metadata().eventId()`»*.

Declararlo igualmente **no es redundante: es incorrecto**. Un emisor Keel no escribe ningún metadato nativo del broker —la envoltura entera, metadata incluida, viaja dentro del JSON—, así que `messageId: { location: header, name: messageId }` sobre una fuente Keel apunta a un header que nadie estampa. El generador le escribiría al agente «deduplica por header `messageId`» y el listener lo leería vacío. `keel validate` lo avisa desde que existe esta regla.

> **Idea clave:** `contract.messageId` es para cuando **no** hay envoltura Keel de la que tirar: `envelope: none`, `envelope: wrapped`, canales `external`, o una fuente que sí publica su id en una propiedad nativa del broker. Con fuente Keel, el campo correcto es *ninguno*.

### El YAML

Fuente Keel — no se declara identidad:

```yaml
# specs/stock-reservation/messaging.keel.yaml
subscriptions:
  StockRejected:
    description: El almacén rechazó a posteriori un bloqueo de stock ya confirmado.
    source: inventory
    payload:
      orderId: { type: uuid, required: true }
      reason:  { type: string, required: true }
    contract:
      # Sin `messageId`: la identidad ya es `metadata.eventId` de la envoltura, y es lo
      # que alimenta `processed_event`.
      envelope: keel
    triggers: releaseReservation
    input:
      orderId: orderId
      reason: reason
    # Una compensación puede llegar ANTES del hecho que compensa: entre que este
    # servicio confirma la reserva y que el almacén publica su rechazo no hay orden
    # garantizado. Los reintentos absorben esa carrera solos; la DLQ es la red.
    onFailure:
      retry: { maxAttempts: 5, backoff: exponential, initialDelayMs: 500, maxDelayMs: 15000 }
      deadLetter: true
```

Fuente ajena — ahí sí se declara (`metering-digest`, la fixture de la silueta de ingesta):

```yaml
# specs/metering-digest/messaging.keel.yaml
    contract:
      envelope: wrapped
      payloadPath: data
      discriminator: { location: header, name: eventType, value: MeterReadingCaptured }
      # La fuente no es Keel: no hay metadata de la que tirar, así que el id del mensaje
      # es un dato del contrato que hay que averiguar con su dueño.
      messageId: { location: header, name: messageId }
```

### El schema exacto

`messageId` es un `messageRef` (`common.schema.json`):

| Campo | Obligatorio | Valores |
|---|---|---|
| `location` | ✅ | `header` (metadato del mensaje: header Kafka, atributo SQS, property AMQP) \| `field` (campo del cuerpo, dot-path si está anidado) |
| `name` | ✅ | El nombre **real tal cual lo emite la fuente** |
| `value` | ❌ **prohibido aquí** | Solo aplica al `discriminator` |

Ese último punto es una restricción condicional del schema y tiene sentido: un `discriminator` identifica **un tipo** (por eso lleva `value`), un `messageId` identifica **una ocurrencia** (por eso no puede llevarlo).

### La regla dura

```
maxAttempts > 1  y  ninguna guarda  →  ERROR
```

Y `keel validate` solo acepta **dos** guardas — la función `redeliveryGuardsOf` de `crossrefs.js` es literalmente esa lista:

- una **clave de deduplicación en el listener**, que es `contract.messageId` cuando el contrato la declara **o** el `metadata.eventId` de la envoltura Keel cuando la fuente es un servicio Keel: las dos alimentan el mismo `processed_event`, así que valen igual; o
- una `transitions` irrepetible en la operación disparada.

> **Idea clave:** la `idempotency` de la operación **no cuenta**. Su clave llega por una cabecera HTTP que el broker no manda.

De ahí que el error solo aparezca cuando **no hay envoltura Keel**: `envelope: none`, `wrapped`, o un canal `external`. Con fuente Keel la clave existe siempre, y el mensaje del error lo dice para no empujar a declarar un `messageId` que no debería declararse.

Otros avisos del mismo bloque: un `messageId` con `location: field` cuyo `name` no está en el `payload` declarado; un canal `external: true` sin `contract` (sin envoltura Keel no hay id del que tirar); y el inverso de este §, un `messageId` declarado **habiendo** envoltura Keel.

---

## 4. Eje 2b — `transitions`: la guarda de dominio

Es el mecanismo menos evidente y el más importante, porque es **el único que no depende de por dónde entre la ejecución**.

### Qué lo hace irrepetible

```yaml
# specs/stock-reservation/domain.keel.yaml
    lifecycle:
      field: status
      transitions:
        # `released` es terminal y no aparece en ningún `from`: es lo que hace
        # IRREPETIBLE la compensación por construcción — aplicarla dos veces la
        # rechaza el propio agregado, sin depender de ninguna guarda del borde.
        # Y una reserva pendiente no tiene stock bloqueado todavía: no hay nada que
        # compensar, así que tampoco hay arista pending → released.
        pending: [confirmed]
        confirmed: [released]
        released: []
```

```yaml
# specs/stock-reservation/use-cases.keel.yaml
  releaseReservation:
    # El estado de vuelta: deshacer contra el almacén sin devolver esto dejaría la
    # reserva en `confirmed`, es decir, donde la puso un trabajo que ya no existe.
    transitions:
      - entity: Reservation
        from: [confirmed]
        to: released
```

La regla, en una línea: **una transición cuyo `to` no está entre sus propios `from` es irrepetible por construcción**. Al segundo intento la entidad ya está en el destino y el guard del agregado lo rechaza.

El generador deriva del `lifecycle` un guard que rechaza cualquier cambio no declarado. Eso tiene un corolario incómodo: una operación que necesita una arista que el `lifecycle` no tiene **no falla al generar — falla en cada ejecución**.

### Guarda de puerta vs. guarda de dominio

Esta es la distinción que más se pasa por alto:

| | Guarda de **puerta** | Guarda de **dominio** |
|---|---|---|
| Quiénes son | la clave del listener —`metadata.eventId` o `contract.messageId`— · `idempotency` HTTP (filtro) | `transitions` (agregado) |
| Dónde corta | En el borde, antes de llegar al dominio | Dentro del agregado, por debajo de las dos |
| Alcance | **Solo su camino** | **Cualquier camino** |
| Registro | Dos tablas distintas, con espacios de clave distintos | Ninguno: es el estado mismo |

Cada guarda de puerta cierra **su** puerta y **no sabe de la otra**. `processed_event` se indexa por (listener, id de mensaje); `idempotency_record`, por (operación, clave del cliente). Son dos universos.

> **Idea clave:** si una operación se puede alcanzar por dos caminos —el evento **y** un endpoint para que un operador la reejecute a mano— deduplicar el mensaje deja el otro camino abierto. `keel validate` lo da en **rojo**. Y añadir `idempotency` no lo arregla: sin cabecera se ejecuta sin deduplicar, y quien reejecuta a mano es precisamente el que no la manda.

Por eso la fixture marca `releaseReservation` como `internal: true`, con el porqué escrito:

```yaml
    # Interna: solo la dispara la suscripción. Exponerla por HTTP abriría un segundo
    # camino que la deduplicación del listener no cubre (keel validate lo da en rojo
    # si no hay guarda de dominio; aquí la hay, pero no hace falta el endpoint).
    internal: true
```

### Un matiz sobre canales externos

Sobre un canal `external: true`, el guard de lifecycle **a solas** es aviso: sin envoltura Keel no hay id de mensaje con el que deduplicar antes, así que **cada reentrega normal llega al dominio, sale rechazada y acaba en la cola de descartes**. Funciona, pero convierte lo normal en ruido.

---

## 5. Eje 3 — `http-clients.calls.<x>.idempotency`: nuestro reintento

### El problema

> Reintentar es ejecutar otra vez. En una lectura da igual; en una escritura ajena —cobrar, reservar, inscribir— el reintento **duplica el efecto al otro lado**, y un timeout no distingue «no llegó» de «llegó y se hizo».

### El YAML

```yaml
# specs/stock-reservation/http-clients.keel.yaml
      reserveStock:
        method: POST
        path: /stock/reservations
        timeoutMs: 3000
        # La cara SALIENTE de la idempotencia. Sin ella, un timeout de red haría que
        # el reintento bloquease el stock dos veces en el almacén — y esa es
        # exactamente la deuda que la compensación de abajo tendría que ir a limpiar.
        idempotency:
          keyFrom: payload-hash
        retry:
          maxAttempts: 3
          backoff: exponential
          initialDelayMs: 200
          retryOn: [timeout, connection]
```

### El schema exacto — y el detalle que descoloca

`common.schema.json` → `$defs.outboundIdempotency`:

| Campo | Obligatorio | Valores | Default |
|---|---|---|---|
| `keyFrom` | ✅ | `payload-hash` \| `correlation` | — |
| `header` | | cualquier nombre de cabecera | `Idempotency-Key` |

> **Idea clave:** el enum **no es el mismo** que el de entrada. Entrante: `client-key` \| `payload-hash`. Saliente: `payload-hash` \| `correlation`. Y tiene sentido: hacia dentro la pregunta es *¿puede el cliente repetir una clave?*; hacia fuera la pregunta es *¿el proveedor deduplica por contenido o por intención?*.

| `keyFrom` | Cuándo |
|---|---|
| `payload-hash` | Firma determinista del contenido. El reintento manda lo mismo, luego repite clave. **Es el caso normal.** |
| `correlation` | El proveedor deduplica por **intención de negocio** y no por contenido: dos peticiones idénticas de dos ejecuciones distintas **sí** deben ejecutarse las dos |

### Lo que exige `keel validate`

| Regla | Severidad |
|---|---|
| `retry` con `maxAttempts > 1` sobre método no seguro (`POST`/`PUT`/`PATCH`/`DELETE`) sin `idempotency` | aviso |
| `idempotency` sobre un `GET` | aviso (no aporta nada) |
| `circuitBreaker` sin `fallback` | aviso |

### La condición que el DSL no puede comprobar

**Solo sirve si el proveedor la honra.** Eso es parte de *su* contrato, no una decisión nuestra. Si no la honra, hay que decirlo en el `contract` de la llamada: que reintentar duplica es información que el siguiente que lea el diseño necesita.

---

## 6. Mecanismo 4 — `compensations`: deshacer trabajo ya encargado

### Qué es

Eventos ante los que este servicio **deshace lo que hizo contra el proveedor**. Aquí solo se declara **el hecho y contra quién**: la operación que se ejecuta vive en `messaging: subscriptions.<evento>.triggers` y no se repite.

```yaml
# specs/stock-reservation/dependencies.keel.yaml
dependencies:
  inventory:
    description: Almacén, dueño del stock físico y de su bloqueo.
    contract:
      version: 1.0.0
    activations:
      reserveStock:
        description: Bloqueo del stock del pedido en el almacén.
        triggeredBy: [confirmReservation]
        via:
          client: inventory
          call: reserveStock
        effect: El stock del pedido queda bloqueado y deja de estar disponible para otros.
        awaits: outcome
        # Sin esto, toda la compensación depende de que el almacén se acuerde de
        # publicar su rechazo. Cuando no lo hace, nada lo detecta.
        reconciledBy: reconcileReservations
        onFailure:
          action: fail
          error: STOCK_UNAVAILABLE
    compensations:
      - onEvent: StockRejected
        undoes: reserveStock
        description: El almacén rechazó el bloqueo a posteriori; la reserva se libera.
```

### El schema exacto

| Campo | Obligatorio | Qué es |
|---|---|---|
| `onEvent` | ✅ | Evento consumido; **debe existir** en `messaging: subscriptions` |
| `undoes` | | Activación **de este mismo proveedor** que se deshace |
| `description` | | Prosa |

`compensations` es hermano de `needs` y `activations`, pero **no basta por sí solo** para declarar una dependencia: el schema exige `anyOf: [needs, activations]`.

`undoes` solo puede citar una activación del mismo proveedor: *compensar el trabajo de tres servidores son tres bloques*.

### `onFailure` no es una compensación

Se confunden constantemente:

| | Cubre |
|---|---|
| `activations.<a>.onFailure` | Que el encargo **no salga** |
| `compensations` | Que el encargo **sí salió** y luego dejó de valer |
| `activations.<a>.reconciledBy` | Que no llegue **ninguna noticia** |

Son los tres desenlaces posibles de un encargo, y ninguno cubre a los otros.

### Las dos obligaciones que exige `keel validate`

> Una compensación es el punto del diseño donde un fallo silencioso cuesta más caro: se ejecuta ante un evento de fallo, por un canal **at-least-once**, y deshace trabajo real.

**(a) No poder aplicarse dos veces — error.** *Deshacer dos veces el mismo trabajo no es deshacerlo: es liberar el stock de otro o reembolsar dos veces.*

Vale cualquiera de los dos mecanismos del eje de eventos (la clave del listener —`metadata.eventId` o `contract.messageId`— o `transitions`), pero **no son equivalentes**: si además hay endpoint HTTP, tiene que ser `transitions`.

**(b) Devolver el estado propio — aviso fuerte.** Si las operaciones que disparan la activación deshecha mueven el `lifecycle` de una entidad y la operación compensadora no declara ninguna `transition` sobre ella, *el trabajo se deshace contra el proveedor y la entidad se queda en el estado que le puso un trabajo que ya no existe*.

La pregunta que hay que responder es literal: **¿a qué estado vuelve?**

> **Trampa habitual, y la razón de que este eje exista:** declarar la compensación y olvidar la arista de vuelta en el `lifecycle`. El diseño valida en verde, el guard del generador rechaza la compensación **en cada ejecución** — el peor sitio posible para un fallo, porque solo se ejecuta cuando algo ya había salido mal.

### El catálogo completo de reglas

**Errores:**

| Regla |
|---|
| `onEvent` que no está en `messaging: subscriptions` |
| `undoes` hacia una activación inexistente |
| La operación disparada es `kind: query` — *una lectura no deshace nada* |
| Ninguno de los dos mecanismos que impiden aplicarla dos veces |
| También expuesta por HTTP y solo tiene la clave del listener (`metadata.eventId` o `contract.messageId`) — una guarda de puerta no cubre el otro camino |
| La suscripción no reintenta ni tiene `deadLetter` — una llegada fuera de orden se pierde |
| `reconciledBy` hacia una operación inexistente, sin `schedule`, o `kind: query` |

**Avisos:**

| Regla |
|---|
| No devuelve el estado que movió el trabajo encargado |
| Sin `undoes` habiendo activaciones en esa dependencia |
| Activación compensada **sin `reconciledBy`** |
| Encarga trabajo a **varios** proveedores y solo compensa a algunos → **la saga incompleta** |
| `deadLetter` cuya operación no se expone por HTTP ni se reconcilia → la DLQ sin vía de reejecución |
| `deadLetter` pero sin reintentos |
| Canal `external` cuya única protección es el guard de lifecycle |
| Disparada por un evento de un tercero, y la operación no tiene por dónde avisar al proveedor |

### El orden de llegada: la carrera que no es exótica

> Entre que este servicio confirma su trabajo y que el proveedor publica su fallo **no hay ninguna garantía de orden**: el evento de compensación puede llegar **antes** del hecho que compensa.

Y entonces se rechaza — la transición no sale de un estado al que todavía no se ha llegado. Lo que decide el desenlace es la política de la suscripción:

- `onFailure.retry` **absorbe la carrera solo**, sin que nadie intervenga.
- `deadLetter` es la red por si no se resuelve.
- Sin ninguno de los dos, el mensaje se pierde en silencio — y lo que se pierde es justo lo que deshace trabajo real contra otro servidor. Por eso es **error**.

*No es un caso exótico: es el orden normal de dos hechos concurrentes.*

### En el mapa del sistema

Una compensación son **dos aristas hacia el mismo proveedor**: `invokes` por la activación, y `consumes` con `kind: events` por el evento que la deshace. Las dos apuntan en el mismo sentido, así que no fabrican un ciclo. Sin la segunda, `keel system check` reporta la suscripción como una fuente que el mapa no contempla.

---

## 7. Mecanismo 5 — `reconciledBy`: el desenlace en el que no pasa nada

### El problema

> Falta el tercer desenlace, que es el único que no produce ningún hecho: el proveedor acepta el encargo y luego cae, pierde el mensaje, o ni siquiera se entera de que hay que deshacerlo. Entonces no llega ningún evento, nada se dispara, y el encargo queda hecho con nuestra entidad esperando un desenlace que no va a venir. **El sistema no está roto: está callado.**

En el eje de la compensación, la pregunta se formula así: *«Toda la compensación cuelga de que llegue un aviso. ¿Y si no llega ninguno? ¿Quién se entera, y cuándo?»* — y **«alguien lo verá» no es una respuesta**.

### El YAML

```yaml
# dependencies.keel.yaml
        reconciledBy: reconcileReservations
```

```yaml
# use-cases.keel.yaml
  reconcileReservations:
    description: Revisa las reservas confirmadas que siguen sin desenlace del almacén.
    kind: command
    internal: true
    input: "void"
    output: "void"
    # La pata del SILENCIO: si el almacén nunca publica su rechazo —cae, pierde el
    # mensaje, o ni sabe que hay que deshacerlo— no llega ningún evento y la reserva
    # se queda confirmada para siempre. Lo que no pasa solo lo detecta un barrido.
    schedule:
      cron: "*/15 * * * *"
```

### Las tres reglas duras

`reconciledBy` es un **string** (nombre de operación), no un objeto. Y la operación citada tiene que cumplir tres cosas, todas **error** si no:

| Regla | Por qué |
|---|---|
| Existe en `use-cases` | Obvio |
| Declara `schedule` | *Una reconciliación que hay que disparar a mano no reconcilia nada.* Lo que detecta **lo que no ha pasado** solo puede dispararlo el reloj |
| No es `kind: query` | *Reconciliar es corregir el estado, no leerlo* |

El `cron` del DSL es de **exactamente 5 campos** (`^\S+( +\S+){4}$`), agnóstico del scheduler. El campo de segundos lo añade cada generador; declararlo aquí produce una expresión que el servicio generado **rechaza al arrancar**, lejísimos de donde está la causa.

### Lo que el DSL deja abierto a propósito

| Decisión | Dónde vive |
|---|---|
| Cuánto tiempo es «demasiado tiempo» | **Configuración** del servicio generado, nunca una constante en el código |
| Qué hacer con lo que encuentra (reintentar el encargo o disparar la compensación) | Decisión de negocio; si el diseño no lo dice, el agente lo reporta como `designGap` |

### La DLQ también necesita salida

Hay un aviso hermano: si la suscripción manda a la DLQ lo que no logra procesar, **lo que caiga ahí necesita una vía declarada de reejecución** — el endpoint HTTP de la operación (con su guarda de dominio) o el barrido de reconciliación. Sin ninguno de los dos, *el final de ese mensaje es que alguien abra la base de datos a mano*.

---

## 8. Por qué `idempotency` no protege la reentrega

Es el error conceptual más caro del método, y merece su propia sección.

La tentación es razonable: «esta operación ya declara `idempotency`, luego está protegida contra repeticiones». **No lo está.**

| | Eje HTTP | Eje de eventos |
|---|---|---|
| Se declara en | `use-cases.<op>.idempotency` | Nada, si la fuente es Keel; `subscriptions.<E>.contract.messageId` si es ajena |
| La clave viaja en | La cabecera `Idempotency-Key` | `metadata.eventId` de la envoltura, o el header/campo del mensaje que diga el contrato |
| ¿La manda el broker? | **No** | — |
| Quién escribe el registro | La superficie HTTP | El consumidor de mensajes |
| Tabla | `idempotency_record` | `processed_event` |
| Clave del registro | (operación, clave del cliente) | (listener, id de mensaje) |

> **Idea clave:** no es un matiz de implementación. Son **dos tablas distintas con espacios de clave distintos**, y por eso `keel validate` no acepta la una como prueba de la otra. Declarar `idempotency` en una operación sin endpoint es directamente error: *la clave llega por una puerta que esa operación no tiene*.

El corolario práctico: **la única guarda que cubre los dos caminos a la vez es `transitions`**, porque no vive en ninguna puerta — vive dentro del agregado, por debajo de las dos.

---

# Parte II — El plano del código generado (keel-spring)

La regla que estructura toda esta parte:

> **`build` genera los mecanismos; quien los usa es el agente de código.** Ese es el único tramo de toda la cadena que no está garantizado por construcción, y falla en silencio: un listener sin guard, o un handler que ignora el `IdempotencyStore`, funcionan perfectamente hasta la primera repetición — que es justo cuando algo ya iba mal.

---

## 9. `idempotency_record` + `CommandSignature`

Módulo: `packages/keel-spring/src/scaffold/http-idempotency.js`.

### Qué se genera

| Clase | Paquete | Cuándo |
|---|---|---|
| `CommandSignature` | `application.support` | Hay idempotencia entrante **o** saliente |
| `IdempotencyStore` (puerto) | `domain.idempotency` | Hay idempotencia entrante + capa `persistence` |
| `IdempotencyRecordJpa` / `…Document` + repositorio | `infrastructure.persistence.idempotency` | según el modelo de persistencia |
| `JpaIdempotencyStore` / `MongoIdempotencyStore` | idem | idem |
| `IdempotencyContext` | `application.support` | **solo** con `keySource: client-key` |
| `IdempotencyKeyFilter` | `infrastructure.web` | `client-key` **y** capa `api` |

Las dos últimas son condicionales por una razón que está escrita en el generador: *el contexto y el filtro son el camino de la CABECERA; con `payload-hash` la clave no viaja por transporte —sale del propio contenido—, así que no hay nada que transportar.*

### La tabla

```java
@Entity
@Table(name = "idempotency_record")
public class IdempotencyRecordJpa {
    @EmbeddedId private IdempotencyRecordId id;   // (operation_scope, idempotency_key)
```

| Columna | Tipo | Notas |
|---|---|---|
| `operation_scope` | `String(128)` | PK. *«La columna no se llama "scope": lo es en SQL estándar.»* |
| `idempotency_key` | `String(255)` | PK |
| `signature` | `String(128)` | Firma determinista del contenido |
| `resource_id` | `String(255)`, nullable | Id del recurso creado; **es lo que reconstruye la respuesta** |
| `created_at` | `Instant` | |
| `expires_at` | `Instant` | Caducidad **calculada** |

Dos decisiones con su porqué en el javadoc generado:

> La unicidad la impone la clave primaria, **no una consulta previa**: es la BD la que arbitra la carrera entre dos reintentos simultáneos, y quien la pierde revierte.

> La caducidad se guarda calculada (`expires_at`) en vez de deducirla del TTL al consultar: el `ttlSeconds` del diseño puede cambiar entre despliegues y **las filas ya escritas conservan la ventana con la que se registraron**.

### El adaptador: tres detalles que no son estilo

**Una fila caducada es como si no estuviera.**

```java
return repository.findById(new IdempotencyRecordJpa.IdempotencyRecordId(scope, idempotencyKey))
        // Una fila caducada es como si no estuviera: la ventana de
        // deduplicación la fija el diseño, no la purga (que va por lotes).
        .filter(stored -> stored.getExpiresAt().isAfter(now))
        .map(stored -> new StoredRequest(stored.getSignature(), stored.getResourceId()));
```

**`save` va en `REQUIRED`, al revés que el guard de mensajes.** Del javadoc:

> `save` usa la propagación por defecto (REQUIRED), es decir, se une a la transacción del caso de uso — **al revés que `IdempotencyGuard`, que registra en REQUIRES_NEW**. La diferencia es deliberada: allí el registro debe sobrevivir al fallo del handler (el mensaje ya se consumió); aquí el registro y el recurso creado tienen que commitear juntos, porque una clave marcada sin recurso detrás haría que el reintento de una operación fallida devolviese una respuesta que nunca existió.

**En Mongo se usa `insert`, no `save`.** Un `save` con el `_id` ya presente **reemplaza en silencio**, y la segunda petición pisaría el registro de la primera en vez de perder la carrera.

### `CommandSignature`: la firma canónica

`SHA-256` hexadecimal de una forma canónica calculada a mano. Por qué está generada y no la escribe cada handler:

> La firma se compara contra una guardada en otro despliegue. Si dos handlers la calculan distinto —o el mismo la calcula distinto tras un refactor— la comparación deja de significar nada y **no hay nada que lo delate**: el sistema simplemente deja de deduplicar.

Las reglas de canonicalización, cada una con su motivo:

| Valor | Se codifica como | Por qué |
|---|---|---|
| `null` / `Optional` vacío | `~` | *«Ausencia vs. nulo» es una convención del diseño; colapsarlos daría la misma firma a dos peticiones que el contrato distingue* |
| `BigDecimal` | `stripTrailingZeros().toPlainString()` | `1.50` y `1.5` son el mismo importe |
| `byte[]` | Su digest SHA-256 | Lo que devolvería `String.valueOf` de un array es **la identidad del objeto**, distinta en cada ejecución |
| `Map` | Entradas **ordenadas** | El orden de un mapa no es contenido |
| `Collection` | Orden **conservado** | *Dos listas con los mismos elementos en distinto orden son dos peticiones distintas* |
| `record` | Componentes ordenados por nombre | Estable frente a reordenaciones del código |
| Escalar | `longitud + ":" + valor` | *Prefijo de longitud: hace imposible que un contenido imite un separador* |

Y la decisión de fondo:

> No usa Jackson a propósito: el `ObjectMapper` de la aplicación lo configuran la serialización de la API y el broker, y un cambio ahí —una precisión temporal, un `@JsonInclude`— **movería en silencio firmas ya almacenadas**.

### Qué le queda al agente

El uso dentro del handler. `build` lo deja escrito como nota en el stub, y el esqueleto **cambia según `keySource`**:

| `keySource` | Esqueleto |
|---|---|
| `client-key` | La clave sale de `IdempotencyContext.get()`. **Vacío = el cliente no mandó la cabecera: ejecuta sin deduplicar, no rechaces.** La firma es `CommandSignature.of(command)` |
| `payload-hash` | La clave **es** `CommandSignature.of(command)`, que también es la firma. No hay cabecera, no hay `IdempotencyContext`, y **por tanto tampoco existe el caso «sin clave»** |

El resto es común: `find(scope, clave)` → si hay registro con la **misma** firma, reconstruye la respuesta desde su `resourceId` **sin re-ejecutar nada** (ni escrituras ni eventos); si la firma difiere, lanza el error que el diseño declare; si no hay registro, ejecuta y llama a `save(...)` dentro de la misma transacción del comando.

> **Idea clave:** con `payload-hash`, escribir un `if (key.isPresent())` es *el defecto exacto que hace que la operación no deduplique nunca sin que nada lo delate*. El agente de calidad lo busca específicamente.

### Purga

`@Scheduled(cron = "${idempotency-record.purge.cron:0 30 4 * * *}")` borra lo caducado. **No hay parámetro de retención**: cada fila lleva su propia caducidad.

---

## 10. `processed_event` + `IdempotencyGuard`

Módulo: `packages/keel-spring/src/scaffold/idempotency.js`. Su cabecera enmarca todo el capítulo:

> Idempotencia de consumo: la cara simétrica del outbox. El outbox garantiza que un evento **no se pierde** (a costa de poder entregarlo dos veces); esto garantiza que entregarlo dos veces **no lo procese dos veces**.

### La tabla

| Columna | Tipo | Notas |
|---|---|---|
| `handler_id` | `String(128)` | PK. *Convención: nombre simple de la clase del listener* |
| `event_id` | `String(255)` | PK. El `messageId` del diseño o `metadata.eventId` |
| `processed_at` | `Instant` | Para la purga |

El javadoc del `255` es un ejemplo perfecto del nivel de detalle del generador:

> 255 y no 64: el id lo elige quien publica, y no siempre es un uuid — un id compuesto o un `MessageDeduplicationId` de SQS FIFO se pasan de 64 con facilidad. Y quedarse corto aquí no da un error de validación: da una violación de longitud al insertar, es decir, **un mensaje que se va a la DLQ por no caber en la tabla que existe para no procesarlo dos veces**.

### Los dos órdenes

Esta es la sección central de toda la nota. El `IdempotencyGuard` expone tres métodos y **el orden en que se usan no es intercambiable**:

```java
@Transactional(readOnly = true, propagation = Propagation.REQUIRES_NEW)
public boolean alreadyProcessed(String handlerId, String eventId) { … }

@Transactional(propagation = Propagation.REQUIRES_NEW)
public boolean record(String handlerId, String eventId) { … }

@Transactional(propagation = Propagation.REQUIRES_NEW)
public boolean tryRecord(String handlerId, String eventId) { … }   // alreadyProcessed + record
```

El registro va en **su propia transacción** (`REQUIRES_NEW`), así que **sobrevive al fallo del handler** — y eso es justo lo que hace que el orden importe:

| Orden | Cómo se usa | Qué gana | Qué pierde |
|---|---|---|---|
| **Procesar y luego registrar** *(predeterminado)* | `alreadyProcessed(...)` antes, `record(...)` **después** de que el handler termine bien | Un fallo transitorio deja el mensaje sin marcar, el broker lo reentrega y se vuelve a intentar | Si el proceso muere entre el commit del negocio y el registro, la reentrega se procesa **dos veces** |
| **Registrar y luego procesar** | `tryRecord(...)` antes de despachar (atómico) | Cierra la ventana del duplicado | Convierte cualquier fallo transitorio en un mensaje **perdido**: quedó marcado como procesado y nunca se ejecutó |

Por eso el primer orden **pide una guarda de dominio detrás**: es la única que protege de verdad. Y el segundo solo vale *cuando reprocesar es inaceptable **y** perder es tolerable*.

### Quién elige el orden: el diseño, no el agente

El generador lo calcula en el modelo:

```js
// ¿Hay una guarda EN EL DOMINIO detrás de este listener? Es lo que decide el
// orden del registro de idempotencia: con ella, procesar y luego registrar (un
// fallo transitorio se reintenta); sin ella, la única forma de no repetir el
// efecto es reclamar antes, al precio de perder el mensaje si el handler falla.
triggerHasDomainGuard: (triggerOp?.transitions ?? []).length > 0,
```

Y lo escribe como javadoc del record `<Evento>Message`, que es lo que el agente lee:

| Si la operación de `triggers`… | El javadoc prescribe |
|---|---|
| **declara `transitions`** | *«Orden: `alreadyProcessed(...)` antes de despachar y `record(...)` DESPUÉS de que el handler termine bien. La operación declara transiciones, así que la repetición la frena el agregado y lo que no puede perderse es el mensaje.»* |
| **no las declara** | *«Orden: `tryRecord(...)` antes de despachar […]. Ojo: un fallo del handler deja el mensaje marcado y perdido; si eso no es tolerable, lo que falta es la guarda de dominio en el diseño.»* |

En `stock-reservation`, `releaseReservation` declara `transitions` → primer orden.

> **Idea clave:** *el error caro es el cruzado.* `tryRecord` en un handler reintentable marca como procesado un mensaje que falló y **lo pierde**. El agente de calidad lo reporta como `dedupe: KO` aunque el guard esté llamado.

Y hay una preferencia explícita para el caso que nos ocupa:

> Deshacer trabajo (una compensación) va casi siempre por el primer orden: lo que no se puede perder es el mensaje que revierte algo real, y el guard del agregado absorbe la repetición.

### Dos detalles de escritura que deciden el destino del mensaje

```js
const write = model.persistenceKind === 'document'
  ? `            ${field}.insert(new ${entity}(key, Instant.now()));`
  : `            ${field}.saveAndFlush(new ${entity}(key, Instant.now()));`;
```

- **Mongo: `insert`, no `save`.** Un `save` con el `_id` presente es un **reemplazo**, no un error: la carrera se resolvería sobrescribiendo y el duplicado se procesaría otra vez.
- **JPA: `saveAndFlush`, no `save`.** JPA difiere el `INSERT` hasta el flush, así que con `save` la violación de clave **no salta dentro del `try`** — salta al commit, fuera del `catch`, y el listener ve un error en vez de un duplicado. *La diferencia es que el mensaje acabe confirmado o en la DLQ.*

### Purga

Aquí sí hay retención, porque no hay caducidad por fila:

```java
@Value("\${processed-event.purge.retention-days:14}")
@Scheduled(cron = "\${processed-event.purge.cron:0 0 4 * * *}")
```

14 días por defecto, con el criterio escrito en el YAML generado: *la retención solo tiene que cubrir la ventana en la que el broker puede reentregar un mensaje*.

### Qué le queda al agente

El `<Evento>Listener` **completo** — lo escribe siguiendo la skill del broker (`keel-spring-kafka`, `keel-spring-rabbitmq`, `keel-spring-snssqs`) — incluida la llamada al guard en el orden prescrito.

---

## 11. `OutboundIdempotency`

Módulo: `packages/keel-spring/src/scaffold/http-clients.js`. Es el mecanismo **más pequeño y el único que no deja nada al agente**.

```java
public final class OutboundIdempotency {

    /** Clave por CONTENIDO: dos peticiones idénticas son la misma intención. */
    public static String fromPayload(String call, Object payload) {
        return CommandSignature.of(Arrays.asList(call, payload));
    }

    public static String correlated(String call, Object payload) {
        String correlationId = CorrelationContext.get();
        return correlationId == null
                ? fromPayload(call, payload)
                : CommandSignature.of(Arrays.asList(call, correlationId));
    }
}
```

Lo que la hace útil está en su javadoc:

> Lo que la hace útil es que el reintento produzca la MISMA clave: resilience4j vuelve a invocar el método entero, así que cualquier cosa aleatoria o dependiente del instante pediría una ejecución nueva al proveedor en cada intento, que es exactamente lo que se quiere evitar.

Y sin correlación abierta (un hilo de fondo, un arranque) `correlated` **cae a la firma del contenido**: *es lo único estable que queda, y una clave aleatoria sería peor que no mandar ninguna*.

### El cableado

La cabecera se inserta en el builder del `RestClient`, generada al 100 %:

```java
.header("Idempotency-Key", OutboundIdempotency.fromPayload("reserveStock", request))
```

Con un detalle que evita un bug futuro: cuando hay clave saliente, el request se **hoista a variable**.

```js
// Con clave saliente el wire request se hoista a variable: la firma tiene que
// ser la del MISMO objeto que se envía, y calcularla dos veces (una para el
// body, otra para la cabecera) es la forma de que un día dejen de coincidir.
```

Sin cuerpo, el payload son los parámetros vía `Arrays.asList(...)` — y no `List.of(...)`, *porque un parámetro opcional puede venir nulo y `List.of` lo rechazaría en tiempo de ejecución*.

### Una degradación que conviene conocer

`keyFrom: correlation` **sin capa `api` ni `messaging`** se degrada a `payload-hash` con aviso: nadie abre el contexto de correlación, así que no habría nada que leer.

---

## 12. Compensaciones: cero clases, dos notas

Esta es la parte que más sorprende al leer el generador: **una compensación no produce ninguna clase propia**. El motivo está en el modelo:

```js
// Compensaciones. No generan código propio —son una suscripción normal—, pero sí
// cambian lo que el agente tiene que escribir en el handler que dispara la
// suscripción: deshacer trabajo encargado no es aplicar un cambio más. `undoes`
// es el único dato que dice QUÉ encargo se deshace, y con él, qué entidades movió
// ese encargo y por tanto a qué estado hay que devolverlas. Sin llevarlo hasta el
// stub, el agente lee un handler indistinguible de cualquier otro.
```

El generador calcula `moves` —las entidades cuyo `lifecycle` movieron las operaciones en el `triggeredBy` de la activación deshecha— y lo estampa en dos sitios.

### (a) Javadoc del `<Evento>Message`

> *Compensa la dependencia de `inventory` deshaciendo la activación 'reserveStock': El almacén rechazó el bloqueo a posteriori; la reserva se libera.*
> *Ese trabajo movió el lifecycle de `Reservation`: la operación que despacha este listener tiene que devolver ese estado, no solo avisar al proveedor.*

### (b) Nota en el stub del handler

Con la guarda resuelta en **tres ramas**:

| Situación | Nota generada |
|---|---|
| La operación declara `transitions` | *«la guarda es la transición del agregado, que rechaza la segunda desde un estado que ya no está en `from`. **Va en el DOMINIO, no en el handler**»* |
| Solo hay `messageId` | *«la guarda es la deduplicación del listener por messageId (IdempotencyGuard). **El handler no añade ninguna otra**»* |
| Ninguna de las dos | *«el diseño no declara guarda: repórtalo como `designGap`»* |

Y si `moves` contiene entidades sobre las que la operación **no** declara transición, la nota lo dice explícitamente y pide reportarlo como `designGap` *en vez de inventar el estado destino*.

### La nota de ORDEN: el defecto que crea la deuda

Cuando una operación tiene a la vez `transitions` y una activación HTTP saliente, `build` escribe esto en su stub:

> **ORDEN de los efectos:** aplica PRIMERO la transición de estado (`Reservation: pending → confirmed`) y solo después llama a `inventory.reserveStock`. La llamada saliente **no es transaccional**: si sale antes y la guarda del agregado rechaza el cambio, el rollback revierte la fila pero **el trabajo ya está encargado en el otro servidor y nadie lo deshace**.

> **Idea clave:** este es el defecto que convierte una reentrega inocente en un doble efecto real contra otro servidor. Y es exactamente la deuda que las compensaciones existen para limpiar — con la diferencia de que aquí es evitable de balde, solo ordenando dos líneas.

Aun así queda una ventana irreducible —transición aplicada, llamada hecha, commit que falla— y esa **no se cierra con código**: es literalmente lo que una `compensations` del diseño existe para cubrir.

---

## 13. `reconciledBy` y `onFailure` en el código

### El barrido

La operación reconciliadora **no aparece en ningún `triggeredBy`** —no la dispara un caso de uso, la dispara el reloj—, así que sin enlace explícito su stub sería un `@Scheduled` vacío. `build` le escribe:

> *Reconciliación de `inventory.reserveStock`: barre los encargos que nunca recibieron desenlace — el evento que los cerraría puede no llegar nunca. Los candidatos son `Reservation en confirmed` que llevan demasiado tiempo ahí. El umbral de "demasiado tiempo" **NO lo declara el diseño**: sácalo de `parameters/` con un default explícito, nunca de una constante en el código. Y decide qué hace con cada uno según el efecto declarado ("El stock del pedido queda bloqueado…"): reintentar el encargo o disparar la compensación. Si el diseño no lo dice, es `designGap`.*

Generado: el `@Scheduled` y la nota. Del agente: el barrido entero.

### `onFailure` → el cuerpo del fallback del circuit breaker

Aquí el DSL deja de ser prosa. El `onFailure` de la activación que sale por una llamada **se convierte en el cuerpo del método de fallback**:

| `onFailure.action` | Qué genera `build` |
|---|---|
| `ignore` | **Completo**: resultado neutro (`null` / `List.of()` por campo), `log.warn`, sin propagar. *«El llamante sigue adelante: no propagues la excepción ni inventes datos del proveedor»* |
| `fail` | **Completo** si la excepción existe: `throw new StockUnavailableException("inventory no está disponible para reserveStock")`. Si ninguna operación de `use-cases` declara ese `code`, la clase no existe → TODO |
| `degrade` | **Siempre TODO.** *«El resultado degradado es lógica de negocio y debe ser distinguible por el cliente de una respuesta normal — un dato plausible pero falso es peor que fallar»*, con la prosa del `degradedTo` citada |
| **≥2 activaciones por la misma llamada** | TODO **enumerando el conflicto**: *«el diseño no puede darles políticas distintas sobre un único método»* |
| Sin `onFailure` | TODO |

En todas las ramas se emite la traza y la línea de procedencia:

```java
LoggerFactory.getLogger(InventoryHttpAdapter.class)
        .warn("Fallback de inventory.reserveStock", throwable);
// Política declarada por la activación inventory.reserveStock (onFailure: fail).
throw new StockUnavailableException("inventory no está disponible para reserveStock");
```

La causa se registra **siempre**, gane la política que gane: sin el log, *un fallo de integración —una negociación HTTP rota, un cuerpo que no deserializa— queda indistinguible de «el proveedor está caído»*.

---

## 14. Outbox: la pieza que hace que la reentrega exista

Módulo: `packages/keel-spring/src/scaffold/outbox.js`. Se activa con `messaging.reliability: outbox` + capa `persistence`.

`reliability: outbox` es el contrato **«ningún evento se pierde si la transacción confirma»**. Su precio es exactamente el problema del §10: el relay reintenta, y por tanto puede entregar dos veces.

> **Consecuencia observable de `best-effort`** (`structural-decisions.md` §3.1): *el estado local cambió y nadie aguas abajo se enteró; no hay error, no hay reintento, no hay traza. Se descubre semanas después por descuadre.*
> **Trampa habitual:** «el broker no se cae». El despliegue del broker es la caída más frecuente y la más segura de todas: **está en el calendario**.

### La tabla

```java
@Entity
@Table(name = "outbox_event", indexes = {
        @Index(name = "ix_outbox_event_pending", columnList = "published_at, created_at")
})
```

| Columna | Notas |
|---|---|
| `id` (`UUID`) | PK |
| `destination`, `routing_key` | Tal como los resolvió el bridge |
| `event_type` | Trazabilidad y filtros |
| `payload` (`text`) | La `EventEnvelope` serializada: **lo que viaja tal cual al broker** |
| `created_at` | |
| `published_at` | Null mientras esté pendiente: **es lo que distingue una fila por entregar** |
| `attempts`, `next_attempt_at`, `last_error` | Backoff y diagnóstico |
| `claimed_at` | **Solo en Mongo** |

### Dos reclamos distintos según el motor

| | Relacional | Documental |
|---|---|---|
| Reclamo del lote | `SELECT … FOR UPDATE SKIP LOCKED` (`@Lock(PESSIMISTIC_WRITE)` + hint `-2`) | `findAndModify` estampando `claimed_at`, atómico por documento |
| Liberación | Al commit / al morir la conexión | **Por caducidad**: `outbox.relay.claim-timeout-ms` (60 s) |
| `@Transactional` en el relay | Sí — sostiene el lock | **No, a propósito**: abrir una transacción solo serviría para mantenerla abierta durante la entrega al broker, *I/O externo dentro de una transacción de base de datos* |

### El relay

```java
@Scheduled(fixedDelayString = "\${outbox.relay.fixed-delay-ms:1000}")
```

Backoff exponencial con tope (`backoff.initial-ms:1000` → `backoff.max-ms:60000`, con guarda de desbordamiento en el desplazamiento) y **dead-letter propio**: agotados los `max-attempts:10`, la fila *queda parada (fuera de futuros polls) para inspección manual; no se borra ni bloquea al resto*. Purga a `retention-days:7` de lo ya publicado.

### El único TODO literal

```java
@Component
public class OutboxDispatcherStub implements OutboxDispatcher {
    @Override
    public void dispatch(String destination, String routingKey, String eventType, String payload) {
        // TODO (agente): sustituir este stub por el dispatcher real del broker
        //   elegido en keel-stack.json (skill keel-spring-<broker>):
        //   enviar el payload tal cual (ya es la EventEnvelope serializada) al
        //   destino/routing key indicados, con content-type application/json.
        log.warn("OutboxDispatcher no implementado: {} no salió a {}/{}", eventType, destination, routingKey);
    }
}
```

El puerto `OutboxDispatcher` *es lo ÚNICO acoplado al broker en todo el patrón*. Y el stub **deliberadamente no lanza**: el relay lo interpretaría como fallo de entrega y las filas se acumularían con `attempts` creciendo.

### El bridge

Con outbox, el listener del bridge es **síncrono dentro de la transacción** (`@EventListener`), no `AFTER_COMMIT`. Es lo que garantiza que la fila del outbox y el cambio del agregado commiteen juntos. Y si serializar falla, revierte:

```java
} catch (JsonProcessingException ex) {
    // Serializar un evento propio no puede fallar: si falla, el diseño
    // del payload está roto y la transacción debe revertir.
    throw new IllegalStateException("No se pudo serializar el evento " + eventType, ex);
}
```

---

## 15. Resumen: qué genera `build` y qué escribe el agente

| Mecanismo | 100 % `build` | Del agente |
|---|---|---|
| `idempotency_record` / `CommandSignature` | Entidad, repo, puerto `IdempotencyStore`, adaptador (`find` filtrando caducados + `save` transaccional + purga), `CommandSignature`, `IdempotencyContext`, `IdempotencyKeyFilter`, YAML | El uso en el handler: `find` → comparar firma → reconstruir o ejecutar → `save` |
| `processed_event` / `IdempotencyGuard` | Entidad, repo, guard entero (los tres métodos + purga), javadoc del `<Evento>Message` con **el orden ya elegido**, YAML, índice Mongo | El `<Evento>Listener` completo, incluida la llamada al guard en ese orden |
| `OutboundIdempotency` | La clase **y** el `.header(...)` cableado en cada método del adaptador | **Nada** |
| Compensaciones | **Cero clases.** Javadoc del `<Evento>Message` + nota en el stub con `undoes`, `moves` y la guarda | La transición de vuelta en el agregado + el aviso al proveedor |
| `reconciledBy` | El `@Scheduled` y la nota con los candidatos | El barrido entero, el umbral desde `parameters/`, la decisión reintentar-vs-compensar |
| `onFailure` (fallback) | Completo para `ignore` y `fail`; traza y procedencia siempre | `degrade`; `fail` sin excepción declarada; conflicto de ≥2 activaciones |
| Outbox | Tabla + índice, repo con `SKIP LOCKED`/`findAndModify`, `OutboxRelay` completo, puerto, `append` en el bridge, YAML | **Solo** `OutboxDispatcherStub.dispatch` |

---

# Parte III — Cómo se prueba y cómo se verifica

## 16. Los escenarios `FL-*`

Los dos ejes **se ejercitan de forma distinta**, y eso no es un detalle de estilo:

| Eje | Cómo se escribe el escenario |
|---|---|
| Reintento del llamante HTTP | **Contra la API.** Qué clave se envía en la cabecera, qué devuelve el reintento con la **misma** clave (mismo status y mismo cuerpo, sin segundo efecto), y qué ocurre con clave distinta y mismo contenido |
| Reentrega de un evento | **Contra el canal.** No hay cabecera que enviar: se reentrega el mismo mensaje y se afirma que no hay segundo efecto observable |

Y para compensaciones la regla es tajante:

> Si el diseño declara `dependencies: compensations`, **cada compensación tiene dos escenarios, y ninguno es opcional**: (1) llega el evento de fallo y se verifica el efecto completo; (2) **el mismo evento se reentrega** y el `Then` afirma que no hay segundo efecto. Una compensación es lo que se ejecuta cuando algo ya salió mal, por un canal que reentrega: es la parte del servicio con **menos probabilidad de ejercitarse a mano y más coste si está rota**.

## 17. La trampa que anula el escenario

`build` genera un helper `deliver<Evento>(messageId, payloadJson)` por cada suscripción. La reentrega se escribe llamando **dos veces con el mismo `messageId`**:

```java
String mid = UUID.randomUUID().toString();
deliverStockRejected(mid, "{\"orderId\":\"" + orderId + "\",\"reason\":\"out-of-stock\"}");
await(Duration.ofSeconds(15), () -> "released".equals(jsonPath(get("/api/reservations/" + id), "$.status")));

deliverStockRejected(mid, "{\"orderId\":\"" + orderId + "\",\"reason\":\"out-of-stock\"}");   // MISMO mid: reentrega
// El Then afirma que NO hay segundo efecto.
```

> **Idea clave:** con `messageId` **distintos** son dos hechos distintos, no una reentrega — *un escenario así pasa en verde contra un consumidor que no deduplica nada, que es exactamente el fallo que se pretendía cazar*.

Dos consecuencias prácticas: el efecto es **asíncrono** (se afirma con `await(...)` sobre una lectura por la API, nunca en la línea siguiente), y una compensación siempre lleva **los dos** escenarios.

## 18. El pase de calidad

El agente `keel-spring-quality` hace tres comprobaciones **estáticas** sobre código que no puede tocar, porque:

> Un listener sin guard, o un handler que ignora el `IdempotencyStore`, funcionan perfectamente **hasta la primera repetición** — que es justo cuando algo ya iba mal.

| Campo del reporte | Qué verifica |
|---|---|
| `dedupe` | Las **dos mitades** (consultar el guard **y** descartar el mensaje con ack sin despachar — *referenciar el guard sin actuar sobre su respuesta no deduplica nada*), el **orden** correcto según `transitions`, y que la clave sea la del diseño. Un `UUID.randomUUID()` como `eventId` *compila, pasa cualquier prueba de camino feliz y deduplica **cero*** |
| `commandIdempotency` | Usa el `IdempotencyStore` generado (ni tabla propia, ni `SET NX`, ni un flag), firma con `CommandSignature.of(command)` (una firma a mano o un `hashCode()` es KO: *ni siquiera es estable entre arranques*), y con `payload-hash` **no** hay rama «sin clave» |
| `compensation` | Ejecuta la transición de vuelta **por el método de negocio del agregado** (un `// TODO` vivo ahí es KO), deshace también contra el proveedor si el diseño declara la activación de vuelta, y **no añade su propia guarda** — la que vale es la del agregado |

Cualquiera de los tres en `KO` **revierte el pase de calidad** en la fase 3 del pipeline.

Nota de alcance: el gate **conductual** son los escenarios `FL-*`, que corren antes. Estas comprobaciones existen porque un diseño puede no tener esos escenarios todavía, y porque *leer el código dice por qué falla, no solo que falla*.

---

# Guía de decisión

## 19. El árbol

```
¿Alguien puede ejecutar esto dos veces?
│
├─ Un cliente HTTP (timeout, doble clic, reproceso manual)
│   └─→ use-cases.<op>.idempotency
│       ├─ ¿Puede el cliente repetir un identificador?  → client-key
│       └─ ¿El mismo cuerpo = la misma intención?       → payload-hash
│           ⚠ ¿El payload lleva timestamp/uuid? → NO sirve: usa client-key
│
├─ El broker (retry, DLQ, at-least-once)
│   ├─ ¿La operación tiene un `to` fuera de sus `from`?  → transitions (basta, y es lo mejor)
│   ├─ ¿No, y la fuente es un servicio Keel?            → nada que declarar: ya deduplica por metadata.eventId
│   └─ ¿No, y la fuente es ajena (none/wrapped/external)? → contract.messageId
│       ⚠ ¿La operación TAMBIÉN se expone por HTTP?     → la clave del listener NO basta: exige transitions
│
└─ Nosotros, contra un proveedor (retry sobre POST/PUT/PATCH/DELETE)
    └─→ http-clients.calls.<x>.idempotency
        ├─ ¿Deduplica por contenido?          → payload-hash
        └─ ¿Por intención de negocio?         → correlation
            ⚠ ¿El proveedor la honra? Si no, dilo en el `contract`

¿Le encargamos trabajo a otro que puede dejar de valer después?
│
├─ ¿Qué queda hecho que nadie va a deshacer?
│   └─→ dependencies.<d>.compensations (con undoes)
│       ├─ ¿A qué estado vuelve la entidad?  → transitions en la operación compensadora
│       │                                       + la arista en domain: lifecycle
│       ├─ ¿Y si llega antes que el hecho?   → onFailure.retry + deadLetter
│       └─ ¿Y si acaba en la DLQ?            → endpoint con guarda de dominio, o el barrido
│
└─ ¿Y si no llega ningún aviso?
    └─→ activations.<a>.reconciledBy → operación con schedule (cron de 5 campos)
```

## 20. Síntomas → mecanismo que falta

Para diagnosticar diseños ya escritos:

| Síntoma | Qué falta |
|---|---|
| Dos recursos idénticos creados con segundos de diferencia | `idempotency` en la operación expuesta |
| El efecto de un evento se aplica dos veces tras un corte del broker | El listener no consulta el guard (la clave existe: `metadata.eventId`), o falta `contract.messageId` si la fuente es ajena — y mejor aún, `transitions` |
| El proveedor tiene dos cargos/reservas y nosotros uno | `idempotency` en la llamada saliente (o el orden de efectos invertido) |
| Estado local coherente, proveedor con trabajo huérfano | `compensations` |
| La entidad se queda en el estado que le puso un trabajo que ya no existe | Falta la `transitions` de vuelta en la operación compensadora |
| La compensación falla **en cada ejecución** | La arista de vuelta no existe en `domain: lifecycle.transitions` |
| Encargos que quedan colgados para siempre y nadie se entera | `reconciledBy` |
| El consumidor deja de avanzar sin ningún error visible | `retry` sin `deadLetter` |
| «Se descubre semanas después por descuadre» | `reliability: outbox` (estaba en `best-effort`) |
| Mensajes en la DLQ que nadie sabe cómo reejecutar | Endpoint con guarda de dominio, o el barrido de reconciliación |
| Compensamos a un proveedor de tres | La **saga incompleta** |

## 21. Los cinco no se sustituyen entre sí

Recapitulando lo que más se confunde:

- **`idempotency` no protege la reentrega** → §8. Dos tablas distintas.
- **`onFailure` no es una compensación** → §6. Uno cubre que el encargo no salga; la otra, que sí saliera y dejara de valer.
- **`compensations` no cubre el silencio** → §7. Deshacer necesita un aviso; si no llega ninguno, hace falta un barrido.
- **`messageId` no cubre el segundo camino** → §4. Es guarda de puerta; solo `transitions` está por debajo de todas.
- **El outbox no evita el duplicado: lo garantiza** → §14. Es la razón de que el §10 exista.

---

## Anexo — Tres inconsistencias detectadas al escribir esta nota (corregidas el 2026-08-07)

Contrastar toda la prosa del repo contra la validación y el código realmente generado destapó tres assets que se habían quedado atrás. **Ninguno era un error del método** — la doctrina en `crossrefs.js`, `docs/dsl/*.md`, `mapping.md` y los agentes era correcta; lo que decía otra cosa era la prosa que la describe. Las tres están corregidas; se dejan aquí porque explican bien dónde se agrieta este tema.

**1. La plantilla enumeraba tres guardas donde solo hay dos.** `keel-core/assets/core/templates/service/dependencies.keel.yaml` decía *«(contract.messageId, idempotency, o una transición de lifecycle irrepetible…)»*, pero `redeliveryGuardsOf()` de `crossrefs.js` solo acepta **dos**. Copiar ese comentario contradecía la validación. → Ahora dice explícitamente que `idempotency` **no** vale ahí, y por qué.

**2. La plantilla de `activations` omitía `reconciledBy`.** Cubría `triggeredBy`, `via`, `effect`, `awaits` y `onFailure`, pero no `reconciledBy` — el único de los cinco mecanismos ausente de lo que siembra `keel new`, y precisamente el que más fácil es no echar de menos: **nadie extraña un desenlace que nunca llega**. → Añadido comentado, con la salvedad de que no aplica a un `awaits: nothing`.

**3. La grave: las tres skills de broker prescribían el orden equivocado.** `keel-spring-{kafka,rabbitmq,snssqs}/references/implementation.md` mandaban `tryRecord(...)` **sin condición**, y encima cerraban con *«si la operación puede fallar de forma transitoria, llama a `tryRecord` después de despachar»* — que es imposible: `tryRecord` **es** la reclamación atómica *antes*; lo que va después es `record`. Llamarlo después significa no comprobar nada antes de despachar, o sea, procesar siempre el duplicado.

Es la que producía código malo, no solo prosa desactualizada: son los archivos que el agente `keel-spring-code` lee para escribir cada `<Evento>Listener`, y le dictaban el orden equivocado en el caso más común (§10). Después, el gate `dedupe` del agente de calidad marcaba `KO` exactamente lo que la skill había mandado escribir. → Las tres remiten ahora al javadoc del `<Evento>Message`, que es quien prescribe el orden, con las dos variantes y su porqué.

> **Idea clave del anexo:** las tres inconsistencias son del mismo tipo — prosa que envejeció respecto al mecanismo que describe. Y las tres estaban en el **único tramo de la cadena que no está garantizado por construcción**. No es casualidad: donde el código no obliga, la única red es que la documentación no mienta.

---

## Fuentes

Todo lo citado aquí sale del repo `keel-system`, sin reinterpretar nada:

| Plano | Dónde |
|---|---|
| Los tres ejes, las preguntas y las consecuencias observables | `keel-core/assets/skills/keel-design/references/structural-decisions.md` §3.1, §3.2, §3.5, §3.11 |
| Las reglas duras de `keel validate` | `keel-core/assets/core/docs/dsl/dependencies.md`, `messaging.md`, `use-cases.md`, `http-clients.md`; `keel-core/src/lib/crossrefs.js` |
| Los schemas | `keel-core/assets/core/schema/common.schema.json`, `messaging.schema.json`, `dependencies.schema.json` |
| La tabla transversal de decisiones | `keel-core/assets/core/docs/dsl-reference.md` § «Dónde vive cada decisión transversal» |
| El código generado | `keel-spring/src/scaffold/{http-idempotency,idempotency,outbox,http-clients,services,messaging}.js`, `src/lib/model.js` |
| Cómo se prueba | `keel-core/assets/core/docs/validation-scenarios.md`; `keel-spring/assets/generators/spring/conventions/integration-tests.md` |
| Cómo se verifica | `keel-spring/assets/agents/keel-spring-quality.md` § «La cadena de idempotencia y compensación» |
| El ejemplo completo | `keel-spring/test/fixtures/stock-reservation/` |
