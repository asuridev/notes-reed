# Declarar la dependencia de un servidor con otro (método Keel)

**Respuesta corta:** en Keel un servidor solo puede depender de otro de **dos maneras**, y la elección entre ellas no es terminológica:

- **`needs` — me llevo un dato suyo.** Necesito información que no es mía para poder decidir.
- **`activations` — le dejo un trabajo.** Hay una parte de mi operación que no es responsabilidad mía y se la encargo.

Y esa dependencia se escribe en **tres planos distintos**, cada uno con su propia pregunta:

| Plano | Archivo | Pregunta que responde |
|---|---|---|
| **El mapa** | `system.yaml` | ¿quién depende de quién y quién se construye antes? |
| **La razón** | `specs/<servicio>/dependencies.keel.yaml` | ¿por qué existe esa integración y qué caso de uso la necesita? |
| **El canal** | `http-clients.keel.yaml` · `messaging.keel.yaml` | ¿por dónde viaja y con qué política? |

> **Idea clave:** el canal (un `GET`, un topic) **no dice de qué dependes**. Un `POST` puede ser tanto una lectura como un encargo. Lo que declara la naturaleza de la dependencia es la capa `dependencies`, y de ella salen el orden de construcción del sistema y el lado hacia el que cae el acoplamiento.

---

## 1. Los tres planos, y por qué son tres

| Plano | Quién lo escribe | Qué lo valida | Grano |
|---|---|---|---|
| `system.yaml` | `/keel-decompose` | `keel system check` | El sistema entero |
| `dependencies.keel.yaml` | `/keel-consume` (dentro de `/keel-design`) | `keel validate` + `/keel-validate` | Un servicio |
| `http-clients` / `messaging` | `/keel-consume`, a la vez que la anterior | `keel validate` | Un servicio |

El recorrido, de arriba abajo:

```
system.yaml            booking  ──consumes──>  flight-catalog     ¿quién va antes?
                       booking  ──invokes───>  notifications
                          │
                          ▼
dependencies.keel.yaml  needs.seatFare       (usedBy: quoteTrip)   ¿por qué existe?
                        activations.sendTicket (triggeredBy: confirmBooking)
                          │
                          ▼
http-clients / messaging  clients.flight-catalog.calls.getFare     ¿por dónde viaja?
                          publishing.events.DeliveryRequested
```

Los tres planos existen porque responden a preguntas que se toman en momentos distintos y las decide gente distinta. El mapa se cierra antes de diseñar nada; la razón sale de la entrevista de casos de uso; el canal y su resiliencia salen del contrato del proveedor.

Y hay una **regla de oro de la capa `dependencies`**: *referencia, nunca redeclara*. Todo lo que ya vive en otra capa se cita por nombre y no se repite. Ni un método HTTP, ni un payload, ni un timeout.

---

## 2. El corte fundamental: leer vs. activar

|  | `needs` — **leer** | `activations` — **activar** |
|---|---|---|
| Qué obtengo | Un **dato** que no es mío | **Trabajo** que hace otro |
| La pregunta | ¿Qué dato ajeno necesita esta operación **para decidir**? | ¿Qué parte de esto **no es mía** y se la pido a otro? |
| Del proveedor me acopla | Su **salida**: la forma del dato que devuelve o publica | Su **entrada**: la firma exacta que hay que mandarle para que actúe |
| ¿El proveedor sabe que existo? | **No** | **Sí**: le he elegido para el trabajo |
| Quién dibuja la arista en el mapa | Quien lee (`consumes`) | Quien pide (`invokes`) |
| Ejemplo | `orders` lee de `catalog` el precio vigente para cotizar | `orders` le pide a `notifications` que mande el correo |

**La pregunta de corte, en una línea:** *¿me llevo un dato suyo, o le dejo un trabajo?*

Si lo que escribirías como descripción es **una acción** — «envío del correo», «inicio del cobro», «reindexado del producto» —, es una **activación**, por mucho que técnicamente sea un `GET`.

### Por qué la distinción no es cosmética

Al **leer**, el proveedor no sabe que existo, y esa ignorancia es exactamente la propiedad que le permite desplegarse solo. Al **activarlo**, soy yo quien tiene que conocer su firma de entrada — y por eso necesito su `INTEGRATION.md` **antes** de poder diseñarme.

Eso cambia el orden de construcción del sistema. Escribir una activación disfrazada de `need` (un `strategy: on-demand` cuyo `fetchedFrom` apunta a un `POST` que no lee nada) **valida**, pero miente: dice que el dato vive en el proveedor cuando lo que vive allí es el trabajo, y deja el acoplamiento fuera del mapa.

Un mismo proveedor puede aparecer de las dos formas: se le leen datos **y** se le encarga trabajo. Es un solo bloque con `needs` y `activations`.

### El esqueleto del archivo

```yaml
dependencies:
  <nombre-del-proveedor>:              # kebab-case; el mismo nombre que usa como `source` de sus eventos
    description: Para qué existe esta dependencia.
    contract:                          # opcional, pero muy recomendable
      version: "2.1.0"                 # a qué versión de SU contrato me acoplo
      source: contracts/<proveedor>/INTEGRATION.md
    needs:            { ... }          # lo que le leo        (al menos uno de los dos)
    activations:      { ... }          # lo que le encargo
    compensations:    [ ... ]          # lo que deshago si algo se tuerce
```

Sobre `contract`:

- `version` registra **a qué versión del contrato publicado se acopla este diseño**. Romperla rompe este servicio: es un hecho de arquitectura, no un detalle administrativo.
- Con `needs` me acopla a su contrato de **salida**; con `activations`, al de **entrada** — y ese es más frágil: un campo nuevo obligatorio en su firma me rompe sin que yo toque una línea.
- `source` es la procedencia (ruta o URL) y es puramente informativa: nadie la resuelve.
- El bloque entero es opcional — hay proveedores que aún no publican contrato. Declarar la dependencia sin contrato es correcto; **inventarse el contrato, no**.

---

## 3. Forma 1 — `needs`: leer un dato ajeno

Un `need` es **un dato ajeno concreto**, no un endpoint del proveedor. Se descubre recorriendo *mis* casos de uso y preguntando: *"¿qué dato que no es mío necesita esta operación para decidir?"*. Nunca al revés: empezar por lo que el proveedor ofrece produce integraciones que nadie pidió.

| Campo | Oblig. | Qué declara |
|---|---|---|
| `usedBy` | ✅ | Operaciones de `use-cases` que necesitan el dato. **Es el único enlace del DSL entre un caso de uso y su integración** |
| `strategy` | ✅ | `on-demand` o `replicated` |
| `fetchedFrom` | según estrategia | La llamada de `http-clients` que resuelve el dato: `{ client, call }` |
| `replica` | con `replicated` | La copia local |

### 3.1 `strategy: on-demand` — se pide en el momento

```yaml
dependencies:
  pricing:
    description: Servicio de precios, dueño del precio vigente.
    contract:
      version: "2.1.0"
    needs:
      currentPrice:
        description: Precio vigente que acompaña a la ficha pública del producto.
        strategy: on-demand
        usedBy: [getProductBySlug]
        fetchedFrom:
          client: pricing
          call: getPrice
```

Y el canal que lo sostiene, en `http-clients.keel.yaml` — **aquí y solo aquí** viven método, ruta, contrato wire y resiliencia:

```yaml
clients:
  pricing:
    purpose: Precio vigente del producto, que es propiedad del servicio de precios.
    auth:
      type: api-key
      headerName: X-Api-Key
    calls:
      getPrice:
        contract: GET /prices/{sku} devuelve { amount, currency }.
        method: GET
        path: /prices/{sku}
        request:
          pathParams:
            sku: { type: string, required: true }
        response:
          fields:
            amount:   { type: decimal, required: true, constraints: { scale: 2 } }
            currency: { type: string, required: true }
        timeoutMs: 2000
        retry: { maxAttempts: 3, initialDelayMs: 200, maxDelayMs: 4000, retryOn: [timeout, 5xx] }
        circuitBreaker: { failureRateThreshold: 50, slidingWindowSize: 20, waitDurationMs: 30000 }
        fallback: Devolver el último precio conocido en caché.
```

Con `on-demand`, `fetchedFrom` es **obligatorio** y `replica` está **prohibido** (lo impone el propio schema).

> El `timeoutMs` sale del presupuesto de latencia de **mi** operación, nunca del SLA ajeno. Y un `fallback` que produce datos plausibles pero falsos es peor que el error que evita.

### 3.2 `strategy: replicated` — se mantiene una copia local

```yaml
    needs:
      supplierPrice:
        description: Precio de proveedor, que se consulta en cada listado y no puede pagar una llamada por elemento.
        strategy: replicated
        usedBy: [listProducts, getProductsByIds]
        fetchedFrom:                     # vía de rescate, porque onMiss es `fetch`
          client: pricing
          call: getPrice
        replica:
          entity: SupplierPrice
          keyField: sku
          fedBy: [SupplierPriceChanged]
          freshness: Vale con que refleje el precio del último cambio publicado; unos segundos de retraso son aceptables.
          onMiss:
            action: fetch
```

La réplica es una entidad **normal** del diseño repartida en tres capas, y `dependencies` solo declara **que esa entidad es una copia, de quién y cómo se mantiene**:

| Dónde | Qué se declara allí |
|---|---|
| `domain` | La entidad `SupplierPrice` y sus campos |
| `persistence` | Dónde se guarda y con qué índices |
| `dependencies` | Que es una réplica de `pricing`, quién la alimenta y qué pasa si falta el dato |
| `messaging` | La suscripción que la alimenta |

```yaml
# messaging.keel.yaml — la suscripción citada en `fedBy`
subscriptions:
  SupplierPriceChanged:
    description: El servicio de precios cambió el precio de proveedor de un producto.
    source: pricing                    # coincide con la clave de la dependencia
    payload:
      sku:        { type: string, required: true }
      amount:     { type: decimal, required: true, constraints: { scale: 2 } }
      currency:   { type: string, required: true }
      occurredAt: { type: timestamp, required: true }
    triggers: projectSupplierPrice
```

Campo a campo de `replica`:

- **`entity`** — la entidad de `domain` que materializa la copia. **No es fuente de verdad**: nunca se expone tal cual como recurso propio ni se le atribuyen invariantes de negocio.
- **`keyField`** — el campo que correlaciona la copia con el identificador del proveedor. Debería ser `unique`: sin unicidad, una reentrega duplica la copia (`keel validate` lo avisa).
- **`fedBy`** — las suscripciones que la mantienen al día. **Deben cubrir todas las vías de cambio, incluidas bajas y retiradas.** Si el proveedor retira un producto y ese evento no se consume, la copia se queda rancia para siempre y nadie se entera.
- **`freshness`** — la tolerancia de negocio a leer un dato viejo, **en prosa**. Nunca un número: un `maxStalenessSeconds` es una decisión de implementación, no un hecho del dominio.
- **Copia solo los campos que este servicio lee.** Replicar el agregado ajeno entero acopla tu diseño a decisiones que no controlas.

### 3.3 Elegir estrategia

|  | `on-demand` | `replicated` |
|---|---|---|
| Dónde está el dato al decidir | Se pide al proveedor en el momento | En una copia local alimentada por sus eventos |
| Con el proveedor caído | **No** se puede operar | Se sigue operando |
| Frescura | Siempre vigente | Eventual: la copia va por detrás |
| Coste por petición | Una llamada de red (N si es un listado) | Ninguno |
| Complejidad | Un cliente | Entidad + tabla + listener + `onMiss` |

Tres preguntas la deciden:

1. **Corrección** — ¿la decisión exige el valor vigente **en ese instante**, o vale una copia reciente? Cobrar exige lo primero; pintar un catálogo, no.
2. **Disponibilidad** — ¿puede este servicio seguir operando con el proveedor caído, y con qué consecuencia de negocio?
3. **Volumen** — ¿es una consulta por petición, o un listado que exigiría N llamadas?

> Es una decisión de **negocio**, no de rendimiento: cambia lo que el servicio puede prometerle a sus clientes.

En el ejemplo de arriba, el mismo dato del mismo proveedor se declara de las dos formas porque las operaciones son distintas: la ficha individual (`on-demand`, una llamada, precio vigente) y el listado (`replicated`, porque una llamada por elemento no es viable).

### 3.4 `onMiss` — qué pasa cuando la copia no tiene el dato

Obligatorio en toda réplica. Es el hueco más caro de dejar implícito, porque **siempre ocurre**: arranque en frío, evento aún no llegado, alta recién creada en el proveedor.

| `action` | Qué exige | Qué observa el cliente |
|---|---|---|
| `fetch` | `fetchedFrom` en el `need` | Nada: la petición tarda un poco más y el dato llega |
| `fail` | `error`, declarado por alguna operación de `use-cases` | El error de negocio declarado, con su status |
| `degrade` | `degradedTo` en prosa | Un resultado parcial o conservador, descrito ahí |

Cada acción **obliga a declarar su consecuencia de negocio**, y eso es lo que la convierte en un hecho del dominio y no en un botón de configuración.

`degrade` es la opción peligrosa: un resultado degradado que produce datos plausibles pero falsos es peor que fallar.

---

## 4. Forma 2 — `activations`: encargar trabajo

Una `activation` es **un trabajo concreto que hace otro servicio**. Se descubre igual que un `need`, recorriendo mis casos de uso, pero con la otra pregunta: *"¿qué parte de esta operación no es responsabilidad mía?"*.

| Campo | Oblig. | Qué declara |
|---|---|---|
| `triggeredBy` | ✅ | Operaciones de `use-cases` que la disparan. Es el espejo de `usedBy` |
| `via` | ✅ | El canal: `{ client, call }` de `http-clients`, o `{ publishes: <Evento> }` de `messaging` |
| `effect` | ✅ | Qué hace el proveedor al recibirla, **en lenguaje de negocio** |
| `awaits` | — | `outcome` · `acknowledgement` (por defecto) · `nothing` |
| `onFailure` | con `via` HTTP | Qué hace mi operación si el encargo no sale |

`effect` no es decorativo: **sin efecto declarado, una activación es una llamada que nadie pidió.**

### 4.1 `via: { client, call }` — activación síncrona

```yaml
  compliance:
    description: Registro regulatorio de retiradas, obligatorio antes de retirar un producto.
    contract:
      version: "1.0.0"
    activations:
      recordWithdrawal:
        description: Alta de la retirada en el registro regulatorio.
        triggeredBy: [retireProduct]
        via:
          client: compliance
          call: recordWithdrawal
        effect: La retirada queda inscrita en el registro regulatorio con su motivo.
        awaits: outcome                 # sin su confirmación, la retirada no es válida
        onFailure:
          action: fail
          error: COMPLIANCE_UNAVAILABLE # lo declara alguna operación de use-cases
```

Y el mismo patrón con una consecuencia mucho más laxa — el buscador se pone al día solo:

```yaml
  search-index:
    description: Servicio de búsqueda, que mantiene el índice del catálogo.
    activations:
      indexProduct:
        description: Reindexado del producto tras darlo de alta o cambiarlo.
        triggeredBy: [createProduct, updateProduct]
        via:
          client: search-index
          call: reindexProduct
        effect: El producto queda buscable con sus datos nuevos.
        awaits: acknowledgement
        onFailure:
          action: ignore
```

La **firma** que viaja por ese canal (el `request` del `call` en `http-clients`) **la fija el proveedor, no yo**. Por eso una activación sin `contract.version` es una integración contra un contrato que nadie ha fijado.

### 4.2 `via: { publishes: <Evento> }` — el evento-comando

```yaml
  notifications:
    description: Servicio de avisos. No le leo ningún dato, solo le pido que envíe.
    contract:
      version: "1.2.0"
    activations:
      announceNewProduct:
        description: Aviso a los suscriptores de la categoría.
        triggeredBy: [createProduct]
        via:
          publishes: ProductCreated     # evento propio, declarado en messaging: publishing.events
        effect: Sale un aviso hacia los suscriptores de la categoría del producto.
        awaits: nothing
```

Esta es la vía que suele quedarse sin declarar, y es justo la que más conviene declarar: **el evento se publica *para* que alguien concreto actúe**, y su payload existe para cumplir lo que ese alguien exige.

Un evento-comando **no es ilegítimo por serlo**: un servicio genérico (avisos, auditoría, facturación) existe precisamente para que le encarguen trabajo, y su puerta de entrada puede ser un mensaje. Lo ilegítimo es no declararlo, porque entonces el acoplamiento existe y nadie lo ve.

> **El par que lo cierra:** el proveedor tiene que aceptar el encargo declarándolo en su capa `messaging` como `subscriptions.<Evento>` con **`nature: request`**. Si lo consume como `fact`, nadie se ha comprometido a atenderlo — y `keel system check` lo reporta.

### 4.3 `awaits` — qué necesita saber mi operación

|  | Qué significa | Consecuencia |
|---|---|---|
| `outcome` | Necesito el resultado del trabajo para continuar | **Exige canal síncrono**: publicar no devuelve nada |
| `acknowledgement` (por defecto) | Me basta con que el proveedor lo aceptara | El trabajo puede fallar después sin que me entere |
| `nothing` | Delego y sigo | El fire-and-forget honesto |

Es una decisión de **negocio**: `acknowledgement` y `nothing` significan que el trabajo puede no llegar a hacerse y que mi operación dio el visto bueno igualmente. Si eso no es aceptable (un cobro, una inscripción regulatoria), el `awaits` es `outcome`.

`awaits: outcome` sobre un `via: { publishes }` es **error de validación**: publicar un evento no devuelve resultado.

### 4.4 `onFailure` — qué pasa si el encargo no sale

Obligatorio con `via` HTTP, por el mismo motivo que `onMiss` en una réplica: siempre ocurre, y es **comportamiento observable en mi propia API**.

| `action` | Qué exige | Cuándo es honesto |
|---|---|---|
| `fail` | `error` declarado en `use-cases` | Sin ese trabajo, mi operación no es válida |
| `degrade` | `degradedTo` en prosa | Hay un resultado parcial que el negocio acepta |
| `ignore` | nada | El negocio **de verdad** no cuenta con ese trabajo |

Con `via: { publishes }`, `onFailure` está **prohibido** por el schema: ahí la entrega la garantiza `publishing.reliability: outbox` — «ningún evento se pierde si la transacción confirma».

---

## 5. La cara del proveedor: qué escribe el otro lado

Depender es asimétrico, y conviene ver las dos caras:

| Yo declaro… | El proveedor declara… |
|---|---|
| `needs` (leo su dato) | **nada sobre mí** — ni sabe que existo |
| `activations` con `via` HTTP | `api: audience: services` (o `both`) + `security: serviceClients.<yo>` con sus scopes |
| `activations` con `via: { publishes }` | `messaging: subscriptions.<Evento>` con **`nature: request`** |

`security: serviceClients` es el **inverso exacto** de `dependencies`: cataloga a quienes me consumen a mí, con el mínimo privilegio de cada uno.

```yaml
# security.keel.yaml del proveedor
serviceClients:
  billing-service: { description: Consulta precios para facturar., scopes: [product:read] }
```

### `fact` vs. `request`: dos mensajes idénticos en el cable, significados opuestos

|  | `fact` (por defecto) | `request` |
|---|---|---|
| Qué es | Algo que **pasó** en el origen | Algo que me **piden** hacer |
| Quién decide el payload | El emisor, para describir su hecho | **Yo**: es mi firma de entrada |
| Quién se acopla | Yo al emisor | El emisor a mí |
| ¿El emisor sabe que existo? | No | Sí: me eligió para el trabajo |
| ¿Es dependencia mía? | **Sí** → va en mi `dependencies` | **No**: es una puerta de entrada |
| Dónde se publica el contrato | En el diseño del emisor | En **mi** `INTEGRATION.md` §Suscripciones |

`OrderPlaced` es un `fact`: `orders` lo publica porque pasó, y quien quiera reaccionar lo hace por su cuenta. `DeliveryRequested` es un `request`: quien lo emite me está encargando un envío, con los campos que **yo** exijo para poder hacerlo.

Consecuencia práctica: **un `request` no obliga a declarar su `source` como dependencia** — no dependo de quien me da trabajo — y su payload es contrato público que no puedo cambiar sin avisar.

---

## 6. El plano del sistema — `system.yaml`

El mapa dibuja las mismas dos formas de depender, pero **en sentidos opuestos**:

|  | `consumes` — leer | `invokes` — activar |
|---|---|---|
| Quién dibuja la arista | Quien **lee** | Quien **pide** |
| Qué contrato hace falta | La **salida** del proveedor | La **entrada** del proveedor |
| En la capa `dependencies` | `needs` | `activations` |

```yaml
services:
  booking:
    summary: Reserva y emisión de billetes.
    responsibility: Única fuente de verdad de la reserva de un pasajero.
    consumes:
      - from: flight-catalog
        kind: http                 # http | events
        what: [tarifa vigente del tramo]
        strategy: on-demand        # preacuerda la estrategia a nivel de sistema
        why: Cotizar exige la tarifa vigente en el instante; una copia no sirve para cobrar.
        blocking: true             # ¿necesito su INTEGRATION.md para diseñarme?
    invokes:
      - to: notifications
        kind: events
        what: [enviar el billete al pasajero]
        events: [DeliveryRequested]
        why: El aviso al pasajero es trabajo de notifications; booking solo dice a quién y con qué.
```

Campos que más pesan:

- **`kind`** — en `consumes`: `http` si **no puedo completar mi operación** sin el dato en el instante, `events` si solo necesito reaccionar o mantener una copia. En `invokes`: `http` si espero respuesta, `events` si mando el encargo y sigo.
- **`what`** — en lenguaje de negocio. Se convierte en los `needs` / `activations` al diseñar. **Si lo que escribes ahí es una acción, la arista es un `invokes`.**
- **`why`** — obligatorio. *Una arista sin porqué es una integración que nadie pidió.*
- **`blocking`** (por defecto `true`) — **es lo que fija el orden de construcción**.

### Las olas se calculan, no se declaran

No hay campo `wave`. Las olas son un orden topológico sobre las aristas **bloqueantes** de los dos tipos: un `consumes` pone al lector después del proveedor, y un `invokes` pone al que pide después de quien hace el trabajo.

De ahí el coste concreto de dibujar la flecha al revés: si `booking` necesita el `INTEGRATION.md` de `notifications` para saber qué mandarle, y esa dependencia se apunta como un `consumes` de `notifications` hacia `booking`, el mapa programa `booking` primero — justo al revés de lo que hace falta.

Lo que no entra en ninguna ola es un **ciclo**. Antes de romperlo, comprueba si alguna arista está al revés: el caso más común es un proveedor que no necesita conocer a su consumidor (le basta recibir su identificador como dato opaco). Si el ciclo es real, se elige qué lado se diseña en modo degradado y esa arista se marca `blocking: false`.

---

## 7. `compensations` — deshacer lo hecho contra el proveedor

### 7.1 Los dos modos de fallo, y la línea que los separa

Casi todo el mundo llega a `compensations` buscando *"¿qué pasa si el pago falla?"* — y la respuesta es que **hay dos preguntas ahí dentro**, con mecanismos distintos:

| | El encargo **no sale** | El encargo salió y **falla después** |
|---|---|---|
| Cuándo se sabe | En el acto (timeout, 5xx, circuito abierto) | Minutos u horas más tarde |
| Qué se sabe | Que no llegó a empezar | Que empezó y acabó mal |
| Cómo se declara | **`onFailure`** de la activación | **`compensations`** de la dependencia |
| Dónde vive en el código | El fallback del adaptador del cliente | Un listener + una operación de reverso |
| ¿Hay algo que deshacer allí? | No | Sí, y también aquí |

> **La línea:** `onFailure` cubre *«no pude encargarlo»*. `compensations` cubre *«lo encargué, se aceptó, y salió mal»*. Un cobro con `awaits: acknowledgement` necesita **las dos**: la pasarela puede estar caída (`onFailure`) o aceptar el cargo y que el banco lo rechace 40 segundos después (`compensations`).

### 7.2 Cómo se propaga una mala noticia tardía

Solo hay una vía: **un evento del proveedor**. No hay callback, ni polling declarado, ni transacción distribuida. El proveedor publica un hecho suyo (`fact`), y quien le encargó el trabajo lo consume y dispara una operación de reverso.

```
payments                                     orders
────────                                     ──────
confirmOrder  ─────POST /charges────────────►  (activación chargeOrder, awaits: acknowledgement)
                                    ◄──202───  el pedido responde 201 al cliente
        …el banco rechaza 40 s después…
publishing.events.PaymentFailed ──broker──►  subscriptions.PaymentFailed (fact)
                                                    │
                                             Listener → IdempotencyGuard → UseCaseMediator
                                                    │
                                             failOrderPayment: el pedido pasa a PAYMENT_FAILED
                                             y se libera la reserva de stock
```

### 7.3 Los seis artefactos

Una compensación no se declara: **se hace declarable**. Con `orders` cobrando contra `payments`, esto es lo que hay que escribir, y ninguna pieza sobra.

**Lado proveedor (`payments`)** — tiene que existir primero, o no hay nada que consumir:

```yaml
# payments/messaging.keel.yaml
publishing:
  reliability: outbox                    # el fallo no se puede perder
  events:
    PaymentFailed:
      description: El cobro aceptado no pudo completarse.
      channel: paymentEvents
      payload:
        chargeId: { type: uuid, required: true }
        orderId:  { type: uuid, required: true }
        reason:   { type: string, required: true }
```

Y en el mapa, `services.payments.publishes: [PaymentAuthorized, PaymentFailed]`. Sin eso, la arista del consumidor no es verificable.

**Lado consumidor (`orders`)** — cuatro piezas más:

```yaml
# 1 · messaging — el canal de entrada
subscriptions:
  PaymentFailed:                         # ⚠ nombre EXACTO al del proveedor: el chequeo cruza por nombre
    description: El cobro aceptado no pudo completarse.
    source: payments
    nature: fact                         # es un hecho suyo, no un encargo nuestro
    channel: paymentEvents
    contract:
      envelope: keel                     # fuente Keel: la identidad del mensaje ya es
                                         # metadata.eventId y de ahí deduplica el listener.
                                         # Declarar un messageId propio aquí es un aviso de
                                         # keel validate: apuntaría a un metadato del broker
                                         # que ningún emisor Keel escribe.
    payload:
      orderId: { type: uuid, required: true }
      reason:  { type: string, required: true }
    triggers: failOrderPayment
    onFailure:
      retry: { maxAttempts: 5, backoff: exponential, initialDelayMs: 1000 }
      deadLetter: true
```

```yaml
# 2 · use-cases — el reverso es una operación NORMAL, no un mecanismo aparte
operations:
  failOrderPayment:
    description: Marca el pedido como impagado y libera lo que se había reservado.
    kind: command
    internal: true                       # no la expone la API: entra por el listener
    idempotency: { key: orderId }        # la entrega es at-least-once
    input:
      fields:
        orderId: { type: uuid, required: true }
        reason:  { type: string, required: true }
```

```yaml
# 3 · domain — la transición de lifecycle (el pedido no "vuelve atrás": avanza)
#     CONFIRMED --failOrderPayment--> PAYMENT_FAILED
```

```yaml
# 4 · dependencies — el nudo que ata las tres anteriores
dependencies:
  payments:
    description: Cobro del pedido, del que solo esperamos el desenlace.
    contract:
      version: "3.0.0"
    activations:
      chargeOrder:
        triggeredBy: [confirmOrder]
        via: { client: payments, call: createCharge }
        effect: Se inicia el cobro contra el medio de pago del comprador.
        awaits: acknowledgement          # acepta el cargo; NO dice que se haya cobrado
        onFailure: { action: fail, error: PAYMENT_UNAVAILABLE }
    compensations:
      - onEvent: PaymentFailed
        undoes: chargeOrder
        description: El cobro se rechazó a posteriori; se libera la reserva y el pedido queda impagado.
```

**El bloque `compensations` es lo que *justifica* que exista esa suscripción.** El checklist de `/keel-consume` lo dice literal: *«toda suscripción escrita alimenta una réplica o es una compensación declarada»*. Sin él, `PaymentFailed` es una suscripción huérfana que nadie sabe por qué está ahí — y `keel validate` avisa de una suscripción `fact` cuyo `source` no está declarado como dependencia.

### 7.4 En el mapa: dos aristas hacia el mismo proveedor

La parte que se olvida. La comprobación de suscripciones busca específicamente una arista **`consumes` con `kind: events`**; una `invokes` no la satisface. Así que describir bien la integración exige las dos:

```yaml
  orders:
    invokes:
      - to: payments
        kind: http
        what: [inicio del cobro del pedido]
        why: Cobrar no es responsabilidad de orders; payments es dueño del medio de pago.
    consumes:
      - from: payments
        kind: events
        what: [desenlace del cobro]
        events: [PaymentFailed, PaymentAuthorized]
        why: El resultado real del cobro llega después; sin él orders no puede compensar.
        blocking: false
```

Dos aclaraciones, porque las dos parecen problemas y no lo son:

- **No es contradicción de dirección.** Esa comprobación salta cuando *el otro* declara consumir de ti (`payments.consumes ← orders`), no cuando tú declaras ambas cosas hacia él.
- **No fabrica un ciclo.** Las dos aristas apuntan en el mismo sentido: `orders` va después de `payments` en las dos lecturas.

> Ojo con la versión de la CLI: hasta hace poco esto era un catch-22 — declarar el `consumes` disparaba *«el mapa planifica consumir 'payments' y su capa dependencies no declara ningún need suyo»*, porque solo un `need` contaba como lectura. Hoy una **compensación** también justifica la arista.

### 7.5 Es coreografía, no orquestación

Y por decisión del método: `system-decomposition.md` lista **«un orquestador sin dominio propio»** entre los antipatrones de frontera. No es que falte el coordinador; es que se rechaza que exista.

- **Cada servicio decide lo suyo.** El pedido decide pasar a `PAYMENT_FAILED`. Nadie se lo ordena: reacciona a un hecho ajeno.
- **El acoplamiento va en un solo sentido.** `payments` publica que un cobro falló y no sabe quién reacciona. Por eso es un `fact` y no un `request`: si fuera `request`, `payments` estaría orquestando a `orders`.
- **El DSL declara los nudos, no el flujo.** La saga completa no vive en ningún artefacto — emerge de la activación, la compensación y el `lifecycle`. Esa es justamente la propiedad de una coreografía.

El precio, que conviene tener escrito: **nadie tiene la vista global de la saga en curso**. De ahí que el `correlationId` de la envoltura Keel no sea un detalle de observabilidad, sino la única forma de reconstruirla a posteriori.

### 7.6 Idempotencia y `lifecycle`: los dos que se olvidan

- **La entrega es at-least-once.** Si la suscripción reintenta, el mismo `PaymentFailed` puede llegar dos veces, y sin nada que lo frene el reverso se aplica dos veces — se libera stock que ya se había liberado. Los mecanismos válidos son **dos**: la clave de deduplicación del listener (`metadata.eventId` si la fuente es Keel, `contract.messageId` si es ajena) o una `transitions` irrepetible en la operación disparada. `keel validate` lo marca como **error** cuando `retry.maxAttempts > 1` y no hay ninguno de los dos. **La `idempotency` de la operación no cuenta**: su clave llega por una cabecera HTTP que el broker no manda.
- **El reverso es un estado nuevo, no una vuelta atrás.** El pedido no regresa a `PENDING` como si nada hubiera pasado: avanza a `PAYMENT_FAILED`. Si el `lifecycle` de `domain` no contempla esa transición, la compensación no se puede implementar sin inventarse el dominio.

### 7.7 Límites y antipatrones

| Límite | Qué significa en la práctica |
|---|---|
| `undoes` es **intra-dependencia** | Solo cita activaciones del mismo proveedor. Compensar el trabajo de tres servidores son tres bloques |
| **No hay `deadline` de saga** | Si el proveedor nunca publica el fallo, nada lo detecta. Eso se resuelve con un estado intermedio en el `lifecycle` y una operación programada, todo declarado a mano |
| **No hay motor de sagas** | `compensations` genera **cero código**: el mapeo del generador es *«nada nuevo: es una suscripción normal, solo etiquetada como compensación»*. Lo que aporta es que el nudo quede declarado y verificable |
| **Lo irreversible no se compensa** | Un correo enviado no se des-envía. Por eso el orden importa: encarga lo irreversible **al final**, cuando ya no queda nada que pueda fallar detrás |

Y el error de diseño más caro: **una activación por evento sin compensación cuando la operación propia puede fallar después de haber encargado el trabajo**. `/keel-validate` lo marca como aviso fuerte, y con razón — queda trabajo hecho en otro servicio que nadie va a deshacer nunca.

---

## 8. Ejemplo completo: un servicio, cuatro caminos

Un catálogo de productos que depende de cuatro proveedores por los cuatro caminos posibles. Es la forma canónica del archivo completo:

```yaml
dependencies:

  # ── LEER (needs), las dos estrategias sobre el mismo proveedor ──────────────
  pricing:
    description: Servicio de precios, dueño del precio vigente y del precio de proveedor.
    contract:
      version: "2.1.0"
    needs:
      currentPrice:                       # on-demand: una ficha, una llamada
        description: Precio vigente que acompaña a la ficha pública del producto.
        strategy: on-demand
        usedBy: [getProductBySlug]
        fetchedFrom: { client: pricing, call: getPrice }

      supplierPrice:                      # replicated: un listado no paga N llamadas
        description: Precio de proveedor, que se consulta en cada listado.
        strategy: replicated
        usedBy: [listProducts, getProductsByIds]
        fetchedFrom: { client: pricing, call: getPrice }
        replica:
          entity: SupplierPrice
          keyField: sku
          fedBy: [SupplierPriceChanged]
          freshness: Vale con que refleje el último cambio publicado; unos segundos de retraso son aceptables.
          onMiss: { action: fetch }

  # ── ACTIVAR por HTTP, fallo tolerable ──────────────────────────────────────
  search-index:
    description: Servicio de búsqueda, que mantiene el índice del catálogo.
    activations:
      indexProduct:
        triggeredBy: [createProduct, updateProduct]
        via: { client: search-index, call: reindexProduct }
        effect: El producto queda buscable con sus datos nuevos.
        awaits: acknowledgement
        onFailure: { action: ignore }

  # ── ACTIVAR por evento (evento-comando) ────────────────────────────────────
  notifications:
    description: Servicio de avisos, que comunica a los suscriptores las novedades.
    activations:
      announceNewProduct:
        triggeredBy: [createProduct]
        via: { publishes: ProductCreated }
        effect: Sale un aviso hacia los suscriptores de la categoría del producto.
        awaits: nothing

  # ── ACTIVAR por HTTP, sin él la operación no es válida, + compensación ─────
  compliance:
    description: Registro regulatorio de retiradas, obligatorio antes de retirar un producto.
    contract:
      version: "1.0.0"
    activations:
      recordWithdrawal:
        triggeredBy: [retireProduct]
        via: { client: compliance, call: recordWithdrawal }
        effect: La retirada queda inscrita en el registro regulatorio con su motivo.
        awaits: outcome
        onFailure: { action: fail, error: COMPLIANCE_UNAVAILABLE }
    compensations:
      - onEvent: WithdrawalRejected
        undoes: recordWithdrawal
        description: El registro rechazó la retirada a posteriori; el producto vuelve a estado activo.
```

Fíjate en lo que **no** aparece por ninguna parte: ni una ruta, ni un método, ni un timeout, ni un payload, ni una credencial. Todo eso vive en `http-clients`, `messaging`, `domain` y `persistence`, y aquí solo se cita por nombre.

---

## 9. Matriz de decisión

| Quiero… | Declaro |
|---|---|
| Un dato ajeno **vigente en el instante** de decidir | `needs` · `strategy: on-demand` + `fetchedFrom` → `http-clients` |
| Un dato ajeno **en un listado** (N elementos) | `needs` · `strategy: replicated` + `replica` → `domain` + `persistence` + suscripción `fact` |
| **Seguir operando** con el proveedor caído | `needs` · `strategy: replicated` (y un `onMiss` honesto) |
| Que otro haga un trabajo y **necesito su resultado** | `activations` · `via: { client, call }` + `awaits: outcome` + `onFailure` |
| Que otro haga un trabajo y **me basta con delegarlo** | `activations` · `via: { publishes }` + `awaits: nothing` (+ `reliability: outbox`) |
| **Reaccionar** a algo que pasó en otro servicio | `messaging: subscriptions` con `nature: fact` + la entrada en `dependencies` que la justifica |
| **Recibir encargos** de otros servicios | `messaging: subscriptions` con `nature: request` — **no** va en mi `dependencies` |
| Cubrir que el encargo **no salga** | `onFailure` en la activación (`fail` \| `degrade` \| `ignore`) |
| Cubrir que el encargo salga y **falle después** | `compensations` con `onEvent` + `undoes`, más la suscripción `fact` y la operación de reverso |

---

## 10. Errores típicos

| Error | Por qué duele |
|---|---|
| **Activación disfrazada de `need`** (`on-demand` sobre un `POST` que no lee nada) | Valida, pero el acoplamiento a la firma de entrada del proveedor queda fuera del mapa y el orden de construcción sale mal |
| **Arista dibujada al revés** en `system.yaml` | El mapa programa al consumidor antes que a su proveedor; se diseña contra un contrato inventado |
| **`fedBy` sin la baja** (solo altas y cambios) | La copia se queda rancia para siempre y **nadie se entera**: no hay síntoma |
| **`onFailure: ignore`** sobre trabajo con el que el negocio sí cuenta | El cliente ve un 200 y el trabajo no se hizo. Es la forma más silenciosa de perder dinero |
| **`degrade`** que fabrica datos plausibles pero falsos | Peor que fallar: nadie detecta que la respuesta es mentira |
| **Réplica expuesta como recurso propio** | Estás publicando como fuente de verdad un dato del que no eres dueño |
| **Réplica del agregado ajeno entero** | Te acoplas a campos que no lees y a decisiones que no controlas |
| **`freshness` como número** (`maxStalenessSeconds: 300`) | Es implementación disfrazada de diseño; el hecho de negocio es la frase, no el número |
| **Activación sin `contract.version`** | Integración contra un contrato de entrada que nadie ha fijado |
| **Suscripción `request` declarada como `fact`** | El emisor cree haber delegado un trabajo; el receptor solo cree estar reaccionando por su cuenta |
| **Confundir `onFailure` con `compensations`** | Cubres «no pude encargarlo» y te queda descubierto «lo encargué y salió mal», que es el caso que de verdad deja rastro en otro servicio |
| **Compensación sin `idempotency`** | La entrega es at-least-once: el reverso se aplica dos veces y libera stock que ya estaba libre |
| **Encargar lo irreversible antes que lo que puede fallar** | El correo ya salió cuando el cobro se rechaza. El orden de las activaciones es parte del diseño |

---

## 11. Qué comprueba cada puerta

| Puerta | Alcance | Qué caza |
|---|---|---|
| `keel validate` | **Intra-servicio** (entre capas de un mismo diseño) | `usedBy`/`triggeredBy` a operaciones inexistentes · `fetchedFrom`/`via` a clientes o llamadas que no existen · `via.publishes` a un evento no publicado · `awaits: outcome` sobre un evento · `replica.entity` o `keyField` inexistentes · `fedBy`/`onEvent` a suscripciones que no existen · `undoes` a una activación inexistente · `onMiss.error`/`onFailure.error` que nadie declara |
| `keel system check` | **Cross-servicio** — la única del método | Aristas a servicios no declarados · ciclos bloqueantes · eventos suscritos que el proveedor no publica · contradicciones de dirección · dependencias que un diseño declara y el mapa no conoce · suscripciones `fact` a fuentes sin arista (una arista `consumes` la justifica un `need` **o una compensación**) · que quien recibe un encargo lo consuma con `nature: request` |
| `/keel-validate` | **Semántica** (lo que ninguna regla mecánica ve) | Si `fedBy` cubre la baja · si el `degradedTo` es aceptable · si un `on-demand` o una activación ocurren dentro de una transacción de escritura · si la réplica copia campos que nadie lee · si un `ignore` o un `nothing` dan por hecho un trabajo con el que el negocio sí cuenta · **si una activación por evento no tiene compensación pudiendo fallar la operación después** |

```bash
keel validate specs/<servicio>     # capas 0/1/2 de un diseño
keel system                        # olas de construcción y estado
keel system check                  # el mapa contra los diseños reales — puerta de CI
```

`keel validate` **no puede ver más allá de un servicio**: su motor de referencias cruzadas solo recibe las capas de un diseño. Que el `source: pricing` de una suscripción exista de verdad, y que ese servicio publique ese evento, lo comprueba únicamente `keel system check`.

Y con una **asimetría deliberada**: lo que el mapa planificó y un diseño aún no implementa solo es deriva cuando ese diseño está **cerrado**; una dependencia que un diseño declara y el mapa no conoce es deriva **siempre**.

---

## 12. El flujo de trabajo

```
/keel-decompose  ──>  system.yaml (consumes / invokes) + briefs/<servicio>.md
                          │
      ola 1               ▼
  /keel-design  ──>  use-cases cerrados
                          │
                          ├── /keel-consume  <── INTEGRATION.md del proveedor
                          │      escribe dependencies + http-clients + messaging
                          │      de una sola pasada y coherentes entre sí
                          ▼
  /keel-integrate  ──>  docs/<servicio>/INTEGRATION.md   ─────┐
                          │                                   │
                          ▼                                   ▼
                    keel system check                 desbloquea la ola 2
```

`/keel-consume` e `/keel-integrate` son **simétricas**: una ingiere el contrato de otro, la otra publica el propio. Se ejecuta `/keel-consume` desde dentro de `/keel-design` (tras cerrar `use-cases`) y a mano en dos casos: una dependencia que aparece con el diseño ya cerrado, y una versión nueva del contrato del proveedor.

> **La regla de oro de `/keel-consume`:** *preguntar y declarar huecos, nunca inferir.* Un hueco declarado es un diseño correcto e incompleto; un contrato inventado es un diseño incorrecto que **parece** completo.

---

## 13. Qué NO se declara en `dependencies`

| Cosa | Dónde va de verdad |
|---|---|
| Método, ruta, timeout, retry, circuit breaker, auth de la llamada | `http-clients` |
| Payload, `envelope`, `messageId`, `discriminator`, retry, DLQ del evento | `messaging` |
| Los campos de la copia, sus tipos y constraints | `domain` |
| Dónde se guarda la copia, sus índices | `persistence` |
| El error de negocio en sí (su `when`, su `http`) | `use-cases: errors` |
| Quién me consume **a mí** | `security: serviceClients` + `INTEGRATION.md` |
| TTL de refresco, tamaño de lote, cron de rehidratación | **El generador** — son decisiones de solución |
| Nombre físico del topic/cola, consumer group, nº de consumidores | Generación y despliegue |
| Credenciales, URLs de entorno, `basePath` del proveedor | Configuración del servicio generado. **Nunca en el diseño** |

---

## Resumen en seis frases

1. Solo se puede depender de dos maneras: **leer un dato** (`needs`) o **encargar un trabajo** (`activations`).
2. El corte se decide con una pregunta — *¿me llevo un dato o le dejo un trabajo?* — y de él salen el orden de construcción y el lado del acoplamiento.
3. Leer se resuelve **en el momento** (`on-demand`) o **con una copia local** (`replicated`), y esa es una decisión de negocio, no de rendimiento.
4. Activar obliga a declarar el **efecto**, qué se espera (`awaits`) y qué pasa si no sale (`onFailure`) — y, si es por evento, obliga al proveedor a aceptarlo con `nature: request`.
5. Un trabajo que se acepta y **falla después** no lo cubre `onFailure` sino `compensations`, que es coreografía pura: un evento del proveedor, una suscripción `fact` y una operación de reverso idempotente.
6. Todo lo que ya vive en otra capa **se cita por nombre y no se repite**: `dependencies` declara la razón, nunca el canal.
