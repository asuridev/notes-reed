# Las capas `http-clients` y `messaging`: cómo se declara el canal (método Keel)

**Respuesta corta:** estas dos capas declaran **por dónde viaja** una integración y **con qué política**, nunca **por qué existe**. Son los dos únicos canales del método:

- **`http-clients` — pido algo y espero la respuesta.** Síncrono. El otro tiene que estar vivo *ahora*.
- **`messaging` — suelto un hecho, o reacciono a uno ajeno.** Asíncrono. El otro puede estar caído y no pasa nada… si lo has declarado bien.

Y el canal es solo **un plano de tres**:

| Plano | Archivo | Pregunta que responde |
|---|---|---|
| **El mapa** | `system.yaml` | ¿quién depende de quién y quién se construye antes? |
| **La razón** | `specs/<servicio>/dependencies.keel.yaml` | ¿por qué existe esa integración y qué caso de uso la necesita? |
| **El canal** | `http-clients.keel.yaml` · `messaging.keel.yaml` | **¿por dónde viaja y con qué política?** ← *esta nota* |

> **Idea clave:** un `POST` no te dice si estás **leyendo un dato** o **encargando un trabajo**, y un evento publicado tampoco. La naturaleza de la dependencia vive en `dependencies`; aquí solo vive el cable. Por eso estas dos capas se escriben **a la vez** que `dependencies` y nunca antes: el canal sin razón es una integración que nadie pidió.

Nota hermana, para el plano de la razón: [`declarar-dependencias-entre-servidores.md`](declarar-dependencias-entre-servidores.md).

---

## 1. Las dos capas de un vistazo

|  | `http-clients` | `messaging` |
|---|---|---|
| Forma del intercambio | Petición → respuesta | Publicar / suscribirse |
| Acoplamiento | **En el tiempo y en la forma**: el otro debe responder ahora | **Solo en la forma**: el mensaje espera |
| Quién conoce a quién | Yo conozco su URL, su firma y su auth | El emisor no sabe quién le escucha (salvo `nature: request`) |
| Dónde vive la resiliencia | Por llamada: `timeoutMs`, `retry`, `circuitBreaker`, `fallback` | Por suscripción: `onFailure.retry` + `deadLetter`; y `reliability` al publicar |
| Si el otro está caído | Mi operación **lo nota**: falla, degrada o tira del `fallback` | El mensaje se acumula; mi operación **ya terminó** |
| Puedo saber el resultado | Sí, es lo natural | No: publicar no devuelve nada |
| Qué me llega si me equivoco | Un error visible en la petición | Silencio: nadie procesa y nadie se entera |
| Capa obligatoria | No | No |

Las dos son **opcionales**. Se declaran en el manifiesto:

```yaml
# specs/<servicio>/service.keel.yaml
layers:
  http-clients: http-clients.keel.yaml
  messaging:    messaging.keel.yaml
```

---

# Parte I — `http-clients`: el canal saliente síncrono

## 2. Anatomía de la capa

Dos niveles y nada más: **clientes** (un sistema externo) que contienen **llamadas** (una operación suya).

```yaml
clients:
  <client-id>:                      # kebab-case: pricing, search-index
    purpose: Para qué existe este cliente.     # obligatorio
    auth: { ... }                              # opcional, por cliente
    calls:
      <callName>:                   # camelCase: getPrice, reindexProduct
        contract: ...               # obligatorio
        ...
```

| Nivel | Convención | Obligatorio |
|---|---|---|
| `clients.<id>` | kebab-case, normalmente el nombre del servicio proveedor | `purpose` + `calls` |
| `calls.<call>` | camelCase, verbo en presente: es *su* operación, no un evento | `contract` |

> **Idea clave:** el cliente se nombra igual que el proveedor en `dependencies` y que el `source` de sus eventos en `messaging`. `keel validate` cruza esos nombres, y llamarlo distinto en cada capa rompe la trazabilidad.

Y el sentido de la referencia va **siempre de `dependencies` hacia aquí**:

```yaml
# dependencies.keel.yaml — la razón
needs:
  currentPrice:
    usedBy: [getProductBySlug]
    fetchedFrom: { client: pricing, call: getPrice }   # ← referencia, nunca redeclaración
```

El método, la ruta, el timeout y el fallback viven **solo** en `http-clients`. Si aparecen también en `dependencies`, es un error de diseño: dos fuentes de verdad que divergirán.

---

## 3. El contrato: prosa siempre, estructura cuando importa

`contract` es el único campo obligatorio de una llamada y es **prosa**. No pretende ser un OpenAPI: es el mínimo que un integrador humano necesita para entender qué pide y qué recibe.

Pero en cuanto conoces el contrato real del tercero, **declara la forma estructurada**:

```yaml
calls:
  getPrice:
    contract: Precio vigente de un SKU con su moneda.
    method: GET
    path: /prices/{sku}
    request:
      pathParams:
        sku: { type: string, required: true }
      queryParams:
        currency: { type: string }
    response:
      fields:
        amount:   { type: decimal, required: true }
        currency: { type: string, required: true }
```

Qué cambia según lo que declares:

| Declaras | Qué obtienes |
|---|---|
| Solo `contract` (prosa) | El generador deja records **vacíos** y el mapper como stub. El agente tiene que deducir la ruta y los campos de tu frase |
| `method` + `path` + `request` + `response` | Parámetros y records **tipados**, mapper completo, y `keel validate` cruza los tipos contra el dominio |

### Reglas del schema que conviene conocer de memoria

| Regla | Por qué |
|---|---|
| `method` y `path` van **juntos** o no van | Media ruta no sirve para generar nada |
| `request` exige `method` | No se puede tipar una petición sin saber el verbo |
| `GET`/`DELETE` **no admiten** `request.body` | Un GET con cuerpo es un error de diseño, no una opción |
| `list: true` **prohibido** en `pathParams` | Una variable de ruta es un solo valor |
| Toda `{var}` de `path` debe estar en `request.pathParams`, **y viceversa** | Comprobación bidireccional: un pathParam que no aparece en la ruta es un campo muerto |
| Los tipos son los del DSL | Base types, value types de `domain: types`, o `enum` inline — pero **prefiere el enum nominal del dominio** |

### `response.fields` es el contrato del cable, no tu dominio

Esta es la distinción que más se falla. `response.fields` describe **lo que devuelve el sistema externo, tal cual lo devuelve**, con sus nombres y su granularidad. No es tu modelo.

```yaml
    response:
      fields:
        grossAmount:  { type: decimal, required: true, wireName: gross_amount }
        currencyCode: { type: string, required: true, wireName: currency_code }
```

El generador lo aísla del dominio con una **capa de anticorrupción**: si el tercero cambia su respuesta, cambian el DTO wire, el adaptador y el mapper — y nada más. Ni el dominio ni los casos de uso se enteran.

`wireName` guarda el nombre real del cable cuando no coincide con el tuyo (`product_id`, `numero_documento`). Es **exclusivo de contratos externos**: solo vale en `http-clients` y en `messaging.subscriptions`. Si aparece en `domain`, `api` o en un evento que **publicas**, `keel validate` da error — y con razón: ahí el nombre lo pones tú.

---

## 4. `auth`: el mecanismo, jamás la credencial

Se declara **por cliente**, no por llamada:

```yaml
clients:
  pricing:
    purpose: Obtener el precio vigente de un producto.
    auth:
      type: api-key
      headerName: X-Api-Key
```

| `type` | Qué declara | Campos extra |
|---|---|---|
| `none` | Sin autenticación | — |
| `api-key` | Clave en cabecera | `headerName` (por defecto `X-Api-Key`) |
| `bearer-static` | Token fijo en `Authorization` | — |
| `basic` | Usuario y contraseña | — |
| `oauth2-client-credentials` | Token de máquina emitido por el proveedor | `tokenUrl` (**obligatorio**), `scopes` |

El schema es cerrado en las dos direcciones: `headerName` **solo** con `api-key`; `tokenUrl` y `scopes` **solo** con oauth2, y `tokenUrl` obligatorio ahí. No puedes escribir un `api-key` con `tokenUrl`.

> **Las credenciales jamás van en el diseño.** Aquí se declara el *mecanismo*; los valores llegan por variable de entorno al servicio generado. Una API key en un `.keel.yaml` es una API key en el historial de git para siempre.

---

## 5. Resiliencia: la decide el diseñador, no el agente

Se declara **por llamada**, porque es política del canal saliente: la comparten todos los casos de uso que pasen por ahí.

```yaml
        timeoutMs: 2000
        retry:
          maxAttempts: 3
          backoff: exponential          # fixed | exponential (default: exponential)
          initialDelayMs: 200
          maxDelayMs: 4000
          retryOn: [timeout, 5xx]
        circuitBreaker:
          failureRateThreshold: 50      # % de fallos que abre el circuito
          slidingWindowSize: 20
          waitDurationMs: 30000         # cuánto espera abierto antes de reintentar
        fallback: Devolver el último precio conocido en caché; si no existe, error PRICE_UNAVAILABLE.
```

| Campo | La pregunta que responde |
|---|---|
| `timeoutMs` | ¿Cuánto puede esperar **mi** operación? Sale de **mi** presupuesto de latencia, **nunca del SLA ajeno** |
| `retry` | ¿El fallo puede ser transitorio? `initialDelayMs` es la espera antes del primer reintento; `maxDelayMs` topa el crecimiento y **solo tiene efecto con `backoff: exponential`** |
| `retryOn` | ¿Qué fallo merece reintento? `timeout`, `5xx`, `connection` |
| `circuitBreaker` | ¿Qué hago si está caído de forma sostenida? Dejo de martillearle |
| `fallback` | **¿Qué hace mi servicio cuando la llamada falla definitivamente?** |

Tres reglas que no son opinables:

1. **Nunca se reintenta un 4xx.** No es un accidente: el enum de `retryOn` **no incluye 4xx**, así que la prohibición está codificada en el propio schema. Un 400 reintentado tres veces sigue siendo un 400, y un 409 reintentado puede duplicar un efecto.
2. **Todo `circuitBreaker` debería tener `fallback`.** `keel validate` avisa si falta. Un circuito abierto sin fallback es una excepción de infraestructura escapándose hacia el usuario.
3. **Un `fallback` que produce datos plausibles pero falsos es peor que el error que evita.** Un precio de ayer que el cliente no puede distinguir de uno de hoy no es degradación: es una mentira silenciosa. Si degradas, el resultado tiene que ser **distinguible** por quien lo recibe.

---

## 6. Ejemplo completo: tres clientes, tres papeles distintos

Estos tres clientes vienen del fixture `catalog-extended` del propio Keel, y están elegidos porque cada uno ilustra un papel diferente en `dependencies`.

```yaml
clients:
  # 1) Proveedor del que se LEE un dato ajeno (needs · on-demand): el precio vigente
  #    no es del catálogo y no se replica — se pide al consultar la ficha.
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
        retry:
          maxAttempts: 3
          initialDelayMs: 200
          maxDelayMs: 4000
          retryOn: [timeout, 5xx]
        circuitBreaker:
          failureRateThreshold: 50
          slidingWindowSize: 20
          waitDurationMs: 30000
        fallback: Devolver el último precio conocido en caché.

  # 2) Proveedor al que se le PIDE trabajo (activations): la reindexación del
  #    buscador no la hace el catálogo, y su fallo no puede tumbar la operación.
  search-index:
    purpose: Índice de búsqueda del catálogo, que mantiene el servicio de búsqueda.
    calls:
      reindexProduct:
        contract: PUT /index/products/{productId} reindexa el producto y devuelve { indexedAt }.
        method: PUT
        path: /index/products/{productId}
        request:
          pathParams:
            productId: { type: uuid, required: true }
          body:
            sku:  { type: string, required: true }
            name: { type: string, required: true }
        response:
          fields:
            indexedAt: { type: timestamp, required: true }
        timeoutMs: 3000
        circuitBreaker:
          failureRateThreshold: 60
          slidingWindowSize: 10
          waitDurationMs: 20000
        fallback: Seguir sin reindexar; el buscador se pone al día en su siguiente barrido.

  # 3) Activación con onFailure: fail — sin él la retirada del producto
  #    no puede darse por buena.
  compliance:
    purpose: Registro regulatorio de retiradas de producto, obligatorio antes de retirar.
    auth:
      type: bearer-static
    calls:
      recordWithdrawal:
        contract: POST /withdrawals registra la retirada y devuelve { recordId }.
        method: POST
        path: /withdrawals
        request:
          body:
            productId: { type: uuid, required: true }
            reason:    { type: string, required: true }
        response:
          fields:
            recordId: { type: uuid, required: true }
        timeoutMs: 4000
        circuitBreaker:
          failureRateThreshold: 50
          slidingWindowSize: 10
          waitDurationMs: 30000
        fallback: Rechazar la retirada; sin inscripción regulatoria no puede darse por buena.
```

Y el `dependencies.keel.yaml` que los justifica — **léelos en paralelo**, porque la política del `fallback` de arriba y el `onFailure` de abajo son la misma decisión escrita dos veces con distinto propósito:

```yaml
dependencies:
  pricing:
    needs:
      currentPrice:
        strategy: on-demand
        usedBy: [getProductBySlug]
        fetchedFrom: { client: pricing, call: getPrice }

  search-index:
    activations:
      indexProduct:
        triggeredBy: [createProduct, updateProduct]
        via: { client: search-index, call: reindexProduct }
        awaits: acknowledgement
        onFailure: { action: ignore }         # → el fallback devuelve neutro y loguea

  compliance:
    activations:
      recordWithdrawal:
        triggeredBy: [retireProduct]
        via: { client: compliance, call: recordWithdrawal }
        awaits: outcome
        onFailure:
          action: fail
          error: COMPLIANCE_UNAVAILABLE        # → el fallback lanza esta excepción del catálogo
```

Fíjate en el mismo cliente `pricing` con dos usos distintos (`on-demand` y una réplica alimentada por eventos): **un cliente puede servir a varios `needs`**, y la resiliencia se declara una sola vez, en la llamada.

---

# Parte II — `messaging`: el canal asíncrono

## 7. Anatomía de la capa

Tres bloques, de los que **al menos uno de los dos últimos** debe existir (una capa `messaging` vacía es inválida):

```yaml
channels:        # opcional — los canales lógicos
publishing:      # lo que emito
subscriptions:   # lo que consumo
```

### 7.1 `channels`: lógicos, nunca topics

```yaml
channels:
  productEvents:
    description: Ciclo de vida del producto que publica este servicio.
  inventoryEvents:
    description: Eventos de inventory-service que este servicio consume.
    external: true                 # el canal lo posee otro sistema
```

Un canal es un concepto **agnóstico del broker**: al generar se materializa en un topic (Kafka), una cola o exchange (RabbitMQ), un topic SNS… igual que un `bucket` de la capa `storage` acaba siendo S3 o MinIO. El nombre físico del topic **no se decide en el diseño**.

`external: true` significa que el canal es de otro: el generador no lo crea, no asume la envoltura Keel sobre él, y su nombre físico se resuelve como parámetro de despliegue. Publicar en un canal externo se puede, pero `keel validate` avisa — exige acuerdo con su dueño.

`channel` es **opcional** en cada evento y suscripción. Declararlo deja plasmado el contrato de integración; omitirlo deja el enrutado a convención.

> Para decidir **cuántos canales** y con qué grano, la nota hermana: [`canal-por-dominio-vs-canal-por-evento.md`](canal-por-dominio-vs-canal-por-evento.md).

### 7.2 `publishing`: lo que emito

```yaml
publishing:
  reliability: outbox            # outbox | best-effort (default: best-effort)
  events:
    ProductCreated:
      description: Se emitió un alta de producto.
      channel: productEvents
      payload:
        productId: { type: uuid, required: true }
        sku:       { type: SKU, required: true }
```

Reglas:

- Los eventos van en **PascalCase y en pasado**: `ProductCreated`, nunca `CreateProduct`. Un evento es un hecho consumado; un imperativo es un comando y no se publica.
- Todo evento que una operación declare en `emits` (capa `use-cases`) **debe** estar aquí. Y al revés: un evento declarado que nadie emite es un aviso de `keel validate` — nada lo publicaría.
- Un campo del payload puede ser colección: `list: true` con `constraints: { minItems, maxItems }`.
- Los eventos que **publicas** no admiten `wireName`: el nombre del cable lo pones tú.

**`reliability` es una decisión del diseñador, no del agente:**

| | `best-effort` (por defecto) | `outbox` |
|---|---|---|
| El contrato | Se intenta publicar tras confirmar | **Ningún evento se pierde si la transacción confirma** |
| Qué pasa si el broker está caído justo al confirmar | El evento **se pierde**, y nadie se entera | El evento queda persistido y sale después |
| Qué arrastra | Nada | La capa `persistence`: exige frontera transaccional |
| Cuándo elegirlo | El hecho es reproducible o su pérdida es tolerable | El hecho dispara dinero, obligaciones o estado ajeno |

La pregunta exacta que hay que hacerse es esa: *¿qué se pierde si el broker está caído en el instante en que mi transacción confirma?* Si la respuesta te incomoda, es `outbox`. El **mecanismo** (tabla + relay, CDC) lo decide el generador; el **contrato**, tú.

---

## 8. La envoltura Keel: ningún evento viaja desnudo

Todo evento publicado sale con dos claves de primer nivel. Tu `payload` ocupa `data`; `metadata` es transversal — la misma para todos los eventos, **no se declara en el spec** — y la estampa el servicio al emitir.

```json
{
  "metadata": {
    "eventId": "9f1c3b6e-2d4a-4a91-b0f2-5c7d8e0a1b23",
    "eventType": "ProductCreated",
    "eventVersion": 1,
    "occurredAt": "2026-03-14T09:21:07.482Z",
    "source": "product-service",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1"
  },
  "data": {
    "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
    "sku": "SKU-10493"
  }
}
```

| Campo | Qué es |
|---|---|
| `eventId` | Id único de **esta ocurrencia**. Es la clave de deduplicación del consumidor: se estampa al emitir y **nunca se regenera aguas abajo** — una reentrega repite el mismo `eventId` |
| `eventType` | Nombre del evento en el diseño. Discriminador cuando un canal transporta varios tipos |
| `eventVersion` | Versión del contrato del payload. Arranca en `1` y solo sube al romper compatibilidad |
| `occurredAt` | Instante ISO-8601 UTC en que **ocurrió el hecho**, no el del envío — con `outbox` pueden distar |
| `source` | El `service.name` del manifiesto |
| `correlationId` | Correlación de la petición que originó el hecho; hila la traza end-to-end. `null` si no hubo contexto (un job programado, por ejemplo) |
| `data` | El `payload` que declaraste |

> **Idea clave:** esta forma es **contrato público** y la emite todo generador, sea cual sea la tecnología. Es lo que permite que un servicio Keel en Java y otro en Go se consuman entre sí sin traductor. Cómo se materializa en cada stack (nombres de clase, serializador) lo decide el generador; **la forma del cable, no**.

Que el `eventId` no se regenere es lo que hace posible la deduplicación extremo a extremo: el consumidor puede afirmar «este hecho ya lo procesé» aunque el broker se lo entregue tres veces.

---

## 9. `subscriptions`: lo que consumo

```yaml
subscriptions:
  StockDepleted:
    source: inventory-service        # obligatorio: quién lo origina
    channel: inventoryEvents
    payload:                         # obligatorio: qué campos me interesan
      productId: { type: uuid, required: true, wireName: product_id }
    triggers: retireProduct          # obligatorio: qué operación mía se ejecuta
    input:                           # campo del input de la operación → campo del payload
      productId: productId
    onFailure:
      retry: { maxAttempts: 5, backoff: exponential, initialDelayMs: 1000 }
      deadLetter: true
```

### `input` y sus tres comprobaciones

`input` mapea el input de la operación disparada contra el payload del mensaje. Si los nombres coinciden, puedes omitirlo: se resuelve por identidad. `keel validate` comprueba tres cosas:

| Comprobación | Severidad |
|---|---|
| Todo campo `required` del input de `triggers` (que no sea `generated` ni `computed`) llega en el payload, directamente o vía `input` | **Error** |
| Las claves de `input` existen en el input de la operación, y sus valores existen en el payload | **Error** |
| Todo campo del payload alimenta algo | **Aviso** — un campo que no alimenta nada es contrato que declaras y no usas |

### `onFailure`: la trampa del retry sin DLQ

```yaml
    onFailure:
      retry: { maxAttempts: 5, backoff: exponential, initialDelayMs: 1000, maxDelayMs: 30000 }
      deadLetter: true
```

Mismo juego de campos que el `retry` de `http-clients`. Y una regla:

> **`retry` sin `deadLetter` no produce un fallo visible: produce un consumidor que deja de avanzar.** El mensaje envenenado se reintenta, agota, vuelve, y la cola se atasca detrás de él. Nadie recibe una alerta porque técnicamente nada ha «fallado».

Y su complemento: **si una suscripción reintenta (`maxAttempts > 1`), la operación disparada debería declarar `idempotency`**. La entrega es *at-least-once*; procesar dos veces el mismo mensaje tiene que ser inofensivo. La skill `/keel-validate` lo comprueba.

---

## 10. `contract`: la forma real del mensaje que emite otro

Cuando consumes de un sistema que **no es Keel**, su mensaje no viene con la envoltura Keel: viene como a él le dio la gana. `contract` es donde plasmas esa realidad, y cada campo responde a una pregunta concreta que tienes que hacerle al emisor.

| Pregunta al emisor | Dónde se plasma |
|---|---|
| ¿El mensaje viene envuelto? ¿Dónde está el payload dentro? | `envelope` (`keel` \| `wrapped` \| `none`) + `payloadPath` |
| ¿En qué formato serializa? ¿Hay schema registrado? | `format` (`json` \| `avro` \| `protobuf` \| `xml`) + `schemaRef` |
| ¿El canal transporta varios tipos? ¿Cómo se reconoce este? | `discriminator` (`location: header\|field`, `name`, `value`) |
| ¿Qué dato identifica el mensaje para no procesarlo dos veces? | `messageId` (`location`, `name`) |
| ¿Los campos llegan con otro nombre? | `wireName` en cada campo del `payload` |
| ¿Qué hago con campos que envía y yo no declaro? | `unknownFields` (`ignore` \| `fail`) |

Detalles que ahorran tiempo:

- `envelope` por defecto es `keel` si el canal **no** es external, y `none` si lo es.
- `payloadPath` es **obligatorio y exclusivo** de `envelope: wrapped`. Es un dot-path.
- `discriminator` **exige** `value` (es lo que identifica *este* evento); `messageId` **prohíbe** `value` (es una clave, no una constante).
- `format` es un dato del emisor, no una elección tuya de generación.
- `schemaRef` es **informativo**: ni `keel validate` ni los generadores lo resuelven. Es un puntero para humanos.
- Consumir de un canal `external: true` **sin** `contract` produce un aviso: el generador tendría que suponer la envoltura, el formato, el discriminador y el id de deduplicación. Suponer cualquiera de los cuatro sale caro.

---

## 11. `nature`: la distinción que decide hacia dónde cae el acoplamiento

Este es el campo más importante de `subscriptions` y el que más se ignora. Vale `fact` (por defecto) o `request`.

|  | `fact` (por defecto) | `request` |
|---|---|---|
| Qué es | Algo que **pasó** en el origen | Algo que me **piden** hacer |
| Quién decide el payload | El emisor, para describir su hecho | **Yo**: es mi firma de entrada |
| Quién se acopla a quién | Yo al emisor | El emisor **a mí** |
| ¿El emisor sabe que existo? | No | Sí: me eligió para el trabajo |
| ¿Es una dependencia mía? | **Sí** → va en `dependencies` | **No**: es una puerta de entrada |
| Dónde se publica el contrato | En mi diseño, para mí | En **mi `INTEGRATION.md`**, §Suscripciones |

Un `fact` es `OrderPlaced`: alguien hizo un pedido y yo, por mi cuenta y sin que él lo sepa, reacciono. Un `request` es `DeliveryRequested`: quien lo emite me está **encargando** un envío, con los campos que **yo** exijo.

> **Idea clave:** el otro lado de un `request` lo declara con una `activation` en `dependencies` y una arista `invokes` en `system.yaml`. **`keel system check` comprueba que las dos lecturas coincidan**: si alguien te manda trabajo y tú lo tratas como `fact`, nadie se ha comprometido a atenderlo. Compila, valida por separado, y en producción el mensaje se publica y no lo procesa nadie.

Consecuencia práctica de la fila «¿es una dependencia mía?»: `keel validate` avisa si el `source` de una suscripción `fact` no está declarado en `dependencies` — pero **no** lo exige para un `request`, porque hacerlo invertiría el sentido del acoplamiento.

---

## 12. Ejemplo completo: dos diseños complementarios

Los fixtures `catalog-extended` y `metering-digest` de Keel están construidos como **par pedagógico**: cada uno tiene lo que al otro le falta.

### `catalog-extended` — sin canales, `outbox`, integración Keel↔Keel

```yaml
publishing:
  reliability: outbox
  events:
    ProductCreated:
      description: Se publicó un producto nuevo en el catálogo.
      payload:
        productId:  { type: uuid, required: true }
        sku:        { type: string, required: true }
        occurredAt: { type: timestamp, required: true }
    ProductUpdated:
      description: Cambió el estado o los datos de un producto.
      payload:
        productId: { type: uuid, required: true }
        status:    { type: ProductStatus, required: true }   # ← value type del dominio

subscriptions:
  # Alimenta la copia local del precio de proveedor (dependencies: pricing.supplierPrice).
  SupplierPriceChanged:
    description: El servicio de precios cambió el precio de proveedor de un producto.
    source: pricing
    payload:
      sku:        { type: string, required: true }
      amount:     { type: decimal, required: true, constraints: { scale: 2 } }
      currency:   { type: string, required: true }
      occurredAt: { type: timestamp, required: true }
    triggers: projectSupplierPrice

  # Compensación: deshace a posteriori el trabajo que se delegó en compliance.
  WithdrawalRejected:
    description: El registro regulatorio rechazó una retirada ya inscrita.
    source: compliance
    payload:
      productId: { type: uuid, required: true }
      reason:    { type: string, required: true }
    triggers: reactivateWithdrawnProduct
```

Qué enseña: sin `channels` (enrutado por convención), sin `contract` (ambas fuentes son Keel, se asume `envelope: keel`), sin `wireName`, y **dos suscripciones con papeles muy distintos** — una alimenta una réplica local, la otra compensa una activación que salió mal a posteriori.

### `metering-digest` — canales, `best-effort`, y un emisor ajeno

```yaml
channels:
  meterTelemetry:
    description: Canal del sistema de medición, que no es nuestro.
    external: true
  digests:
    description: Canal propio por el que se anuncian los resúmenes cerrados.

publishing:
  reliability: best-effort
  events:
    DailyDigestClosed:
      description: Se cerró el resumen diario de un contador.
      channel: digests
      payload:
        meterId:  { type: uuid, required: true }
        day:      { type: date, required: true }
        totalKwh: { type: decimal, required: true, constraints: { scale: 3 } }

subscriptions:
  MeterReadingCaptured:
    description: El sistema de medición publicó una lectura nueva.
    source: metering-gateway
    channel: meterTelemetry
    contract:
      envelope: wrapped
      payloadPath: data
      format: json
      discriminator: { location: header, name: eventType, value: MeterReadingCaptured }
      messageId: { location: header, name: messageId }
      unknownFields: ignore
    payload:
      meterId:        { type: uuid, required: true, wireName: meter_id }
      readAt:         { type: timestamp, required: true, wireName: read_at }
      consumptionKwh: { type: decimal, required: true, wireName: consumption_kwh }
      source:         { type: ReadingSource, required: true }
    triggers: recordReading
    onFailure:
      retry: { maxAttempts: 5, backoff: exponential, initialDelayMs: 200, maxDelayMs: 10000 }
      deadLetter: true
```

Qué enseña: canal `external`, `contract` completo (el emisor no es Keel), `wireName` en tres de los cuatro campos, y `best-effort` — el resumen diario se recalcula, perderlo no es dramático.

---

# Parte III — Cómo encajan con todo lo demás

## 13. Qué comprueba `keel validate` (dentro de un servicio)

Estas dos capas son de las más cruzadas del DSL. Las reglas mecánicas, agrupadas:

**Hacia `http-clients`:**

| Regla | Severidad |
|---|---|
| `fetchedFrom` / `via` de `dependencies` apuntan a un `client` y una `call` que existen | Error |
| Una variable `{v}` de `path` no declarada en `pathParams`, o al revés | Error |
| Los tipos de `request`/`response` existen (base types, `domain: types`, enums) | Error |
| `wireName` en una capa interna | Error |
| `path` con variables pero sin `request.pathParams` | Aviso |
| `request`/`response` tipados sin `method` + `path` | Aviso |
| `circuitBreaker` sin `fallback` | Aviso |
| **Un cliente que ningún `need` ni activación de `dependencies` usa** | Aviso — *¿de qué dependencia forma parte?* |

**Hacia `messaging`:**

| Regla | Severidad |
|---|---|
| Todo evento de `use-cases: emits` existe en `publishing.events` | Error |
| El `channel` referenciado existe en `channels` | Error |
| `triggers` apunta a una operación que existe en `use-cases` | Error |
| Cobertura del `input` (las tres comprobaciones del §9) | Error |
| El evento de `cache.invalidatedBy` existe en messaging | Error |
| Un evento publicado que **ninguna operación emite** | Aviso — nada lo publicaría |
| Un canal declarado que nadie referencia | Aviso |
| El `source` de una suscripción `fact` no está en `dependencies` | Aviso |
| Publicar en un canal `external: true` | Aviso |
| Un campo del payload que no alimenta nada del input | Aviso |

> Fíjate en las dos reglas **inversas** (cliente sin usar, evento sin emisor). Son las que cazan el canal huérfano: código que se generaría, se desplegaría y no serviría a nadie.

## 14. Qué comprueba `keel system check` (entre servicios)

Es la **única validación cross-servicio** del método, y es donde `messaging` se juega de verdad la integración. Cruza el mapa con los diseños reales:

| Arista en `system.yaml` | Qué exige |
|---|---|
| `consumes` (kind `events`) | Que el proveedor **declare en el mapa** que publica ese evento… y, si su diseño ya está cerrado, que su capa `messaging` **lo publique de verdad** |
| `invokes` (kind `events`) | Que el destinatario **lo consuma**… y que lo consuma con **`nature: request`** |

Los dos mensajes de error merecen leerse tal cual, porque explican el fallo mejor que cualquier resumen:

> «el mapa dice que publica `X` para `Y`, pero su capa `messaging` no lo publica. Uno de los dos está mal, y el consumidor se integrará contra un evento que no existe.»

> «consume `X` como `fact`, pero `Y` lo emite para pedirle trabajo. Un `fact` es un hecho al que se reacciona por cuenta propia: nadie se compromete a atenderlo, y su payload no es contrato de entrada.»

Y detecta el **ciclo imposible**: si A declara `invokes: B` y B declara `consumes: A` por el mismo tipo de arista, alguien ha entendido mal quién le está pidiendo qué a quién.

---

## 15. Elegir canal para la misma dependencia

Un mismo encargo puede salir por los dos canales. En `catalog-extended` conviven los dos casos:

```yaml
  # Por HTTP: espero el resultado
  compliance:
    activations:
      recordWithdrawal:
        via: { client: compliance, call: recordWithdrawal }
        awaits: outcome
        onFailure: { action: fail, error: COMPLIANCE_UNAVAILABLE }

  # Por evento: lo suelto y sigo
  notifications:
    activations:
      announceNewProduct:
        via: { publishes: ProductCreated }
        awaits: nothing
```

|  | Por `http-clients` | Por `messaging` |
|---|---|---|
| Sé si salió bien | **Sí** | No |
| Mi operación falla si el otro está caído | Sí (salvo `fallback`) | No |
| Acoplamiento en el tiempo | Sí | No |
| Tengo que conocer su URL y su auth | Sí | No |
| Puedo tener varios destinatarios | No: uno por llamada | **Sí**: quien se suscriba |
| El otro tiene que aceptarme explícitamente | Sí, me da credenciales | Solo si es `nature: request` |

**La regla que lo decide, y está codificada:** `awaits: outcome` **exige un canal síncrono**. Publicar un evento no devuelve el resultado del trabajo, y `keel validate` lo rechaza:

> «`awaits: 'outcome'` exige un canal síncrono — publicar `X` no devuelve el resultado del trabajo»

Así que la pregunta de corte es: *¿necesito el resultado para decidir lo siguiente en esta misma operación?* Si sí, HTTP. Si no, evento — y ganas desacoplamiento en el tiempo y varios destinatarios gratis.

Y si el trabajo se acepta pero **falla después**, ninguno de los dos canales lo cubre: eso es una **compensación** (un evento del proveedor, una suscripción `fact` y una operación de reverso idempotente), como el `WithdrawalRejected` del ejemplo.

---

## 16. Qué sale de esto en Spring

Vale la pena conocerlo, porque enseña qué consecuencia tiene cada campo que escribes.

### 16.1 De `http-clients`

```
domain/clients/                       ← el dominio solo ve esto
  PricingClient          (interfaz)     el puerto: getPrice(String sku) → GetPriceResult
  GetPriceResult         (record)       tipos de MI dominio

infrastructure/http/                  ← aquí vive el mundo exterior
  PricingClientConfig                   bean RestClient: base-url, timeouts, auth
  PricingHttpAdapter                    implementa el puerto; @Retry / @CircuitBreaker
  PricingMapper                         ANTICORRUPCIÓN: GetPriceResponse → GetPriceResult
  GetPriceResponse       (record)       el contrato del tercero, TAL CUAL
  GetPriceRequest        (record)       solo si la llamada tiene body
```

- **Los casos de uso inyectan solo el puerto.** `application` y `domain` jamás importan `RestClient` ni una excepción de Spring web. Si el tercero cambia su respuesta, cambian los tres archivos de `infrastructure/http` y nada más.
- Las anotaciones de resilience4j usan como nombre de instancia **`<cliente>-<llamada>`** (`pricing-get-price`), y la config generada declara esa misma instancia. Si los nombres no cuadran, resilience4j aplica su config por defecto **en silencio**.
- `retryOn` se traduce a excepciones concretas, y `HttpClientErrorException` (los 4xx) entra siempre en `ignore-exceptions`: la prohibición del DSL acaba siendo config real.
- La `base-url` **no lleva default fuera del perfil `local`**: un despliegue sin configurar falla al arrancar, en vez de llamarse a sí mismo en silencio.
- En `local`, las `base-url` apuntan a un **WireMock** que `build` levanta en `infra/`. No es un doble prohibido de los que veta la convención de tests de integración (`@MockBean`, `@EmbeddedKafka` sustituyen la fontanería *dentro* de la JVM): es un proceso aparte que habla HTTP por el mismo socket, exactamente igual que LocalStack sustituye a SNS/SQS. Sin él, ningún escenario de validación que atraviese un cliente saliente es puntuable — falla por conexión rechazada, que no dice nada sobre tu código.

**Lo más notable:** el `onFailure` de una activación **deja de ser prosa y se convierte en el cuerpo del fallback**.

| `onFailure.action` | Qué genera `build` |
|---|---|
| `ignore` | Código real: `log.warn(...)` + resultado neutro. Nada de propagar la excepción |
| `fail` | `throw new <ERROR_DECLARADO>Exception(...)` con el `code` exacto del catálogo de `use-cases` |
| `degrade` | Un `TODO` con la prosa citada — el resultado degradado **es lógica de negocio** y lo decide el agente |

Si dos activaciones salen por la misma llamada con políticas distintas, `build` **no elige por ti**: las enumera como conflicto del diseño.

Lo que queda para el agente: los `*Fallback`, la traducción de errores wire→dominio (`.onStatus(...)`), y tipar lo que dejaste solo en prosa.

### 16.2 De `messaging`

La cadena completa, de la regla de negocio al broker:

```
Agregado                raise(ProductCreatedEvent.of(...))        ← lo escribe el AGENTE
   │                    (buffer interno de eventos de dominio)
   ▼
XxxRepositoryImpl       save() drena pullDomainEvents()           dentro de la @Transactional
   │
   ▼
XxxDomainEventBridge    DomainEvent → <E>IntegrationEvent
   │                    + EventEnvelope(metadata, data, correlationId)
   ▼
  ┌─ reliability: outbox ──────────────┐   ┌─ best-effort ────────┐
  │  tabla outbox_event (misma tx)     │   │  <E>Publisher        │
  │  OutboxRelay (@Scheduled, backoff) │   │  (@TransactionalEvent│
  │  OutboxDispatcher  ← el ÚNICO      │   │   Listener AFTER_    │
  │                      acoplado al   │   │   COMMIT)            │
  │                      broker        │   └──────────────────────┘
  └────────────────────────────────────┘
```

Y del lado consumidor:

```
<E>Listener  →  IdempotencyGuard (tabla processed_event)  →  UseCaseMediator  →  operación de triggers
```

Piezas que conviene tener en la cabeza:

- **`OutboxDispatcher` es lo único acoplado al broker de todo el patrón.** Un puerto de un método. Cambiar de Kafka a SNS toca ese archivo y su config; nada más.
- El **`OutboxRelay`** hace polling con `SKIP LOCKED` (varias instancias no se pisan), backoff exponencial con tope, y al agotar intentos deja la fila **sin borrar** con un log de error: el evento no publicado sigue ahí para inspeccionarlo.
- El **`IdempotencyGuard`** deduplica por `(handlerId, eventId)` y **deja que la clave primaria arbitre la carrera** entre dos entregas simultáneas, en lugar de consultar antes de insertar. Su javadoc lleva la advertencia que más importa: registra **después** de procesar si la operación puede fallar y reintentarse — registrar antes convierte un fallo transitorio en un mensaje perdido para siempre.
- El **`CorrelationContext`** es lo que hace que `metadata.correlationId` no sea siempre `null`: lo abre el filtro HTTP en la petición entrante y lo lee el bridge al emitir. En los listeners lo abre el propio listener con `runWith(...)` — los hilos son de un pool y se reutilizan, así que quien lo abre siempre lo cierra.
- El **javadoc de cada record de suscripción es la especificación del listener** para el agente: qué envoltura llega, por qué header se discrimina, por qué campo se deduplica, y qué comando despachar.

Lo que queda para el agente: el `raise(...)` en el método de negocio del agregado (el `TODO` ya está sembrado en el sitio exacto), la implementación del `OutboxDispatcher` o del `<E>Publisher`, y los `<E>Listener` de cada suscripción.

---

## 17. Trampas

**Repetir la resiliencia en el adaptador.** El `@Retry` ya está puesto por la anotación; un bucle de reintentos escrito a mano dentro del método multiplica los intentos y hace que el circuit breaker cuente mal. La resiliencia se declara una vez, en el diseño.

**Reintentar un 4xx.** El enum no te deja declararlo, pero sí puedes colarlo escribiendo el retry a mano. Un 400 reintentado sigue siendo un 400; un 409 reintentado puede duplicar un efecto.

**`circuitBreaker` sin `fallback`.** El circuito abierto tiene que hacer *algo*. Si no lo declaras, ese algo lo improvisa el agente — y no es su decisión.

**Un fallback que miente.** Devolver el precio de ayer sin que el cliente pueda distinguirlo del de hoy no es degradar: es corromper datos con buena intención.

**Confundir la doble envoltura del broker con `envelope: wrapped`.** SNS→SQS sin *raw message delivery* mete tu mensaje dentro de su propio sobre. Eso es **infraestructura**, no diseño. `envelope: wrapped` describe cómo envuelve **el emisor**, no cómo envuelve el transporte.

**Tratar como `fact` algo que te están pidiendo.** Valida por separado, y en producción el mensaje se publica y nadie lo atiende. Es exactamente lo que `keel system check` existe para cazar.

**`retry` sin `deadLetter`.** No produce un fallo visible; produce un consumidor atascado detrás de un mensaje envenenado, sin alerta.

**Reintentar sin idempotencia.** `maxAttempts: 5` sobre una operación no idempotente son cinco cobros.

**Registrar la idempotencia antes de procesar.** Convierte cualquier fallo transitorio en un mensaje perdido: quedó marcado como procesado y no volverá.

**Meter credenciales, URLs de entorno o `basePath` en el diseño.** El diseño declara el mecanismo y el contrato. Los valores son configuración del servicio generado.

**Dejar que el nombre físico del topic se cuele.** `channels` son lógicos. `product-service-events-prod-v2` no es un canal: es un detalle de despliegue disfrazado.

---

## 18. Qué NO va en estas capas

| Cosa | Dónde va de verdad |
|---|---|
| **Por qué** existe la llamada o la suscripción | `dependencies` (`needs` / `activations`) |
| Qué caso de uso la necesita | `dependencies: usedBy` / `triggeredBy` |
| Qué operación emite cada evento | `use-cases: emits` |
| El error de negocio que dispara el fallback (su `when`, su `http`) | `use-cases: errors` |
| La frontera transaccional que sostiene el `outbox` | `persistence` (`consistency`) |
| Los campos de una réplica local, sus tipos y sus índices | `domain` + `persistence` |
| La idempotencia de la operación disparada | `use-cases: idempotency` |
| Quién me consume a mí | `security: serviceClients` + `INTEGRATION.md` |
| El broker concreto (Kafka, RabbitMQ, SNS/SQS) | Se decide **al generar** |
| El nombre físico del topic o de la cola, consumer group, nº de consumidores | Generación y despliegue |
| Credenciales, tokens, URLs de entorno | Configuración del servicio generado. **Nunca en el diseño** |

---

## Resumen en ocho frases

1. Estas dos capas declaran **el canal y su política**, nunca la razón: el porqué vive en `dependencies` y se cita por nombre desde allí hacia aquí.
2. En `http-clients`, la prosa del `contract` siempre; la forma estructurada en cuanto conozcas el contrato real — es la diferencia entre código tipado y records vacíos.
3. `response.fields` es el **contrato del cable**, y el generador lo aísla del dominio con una capa de anticorrupción: si el tercero cambia, cambia solo `infrastructure/http`.
4. La resiliencia la decide el diseñador: el `timeoutMs` sale de **tu** presupuesto de latencia, los 4xx no se reintentan nunca, y un `fallback` que miente es peor que el error que evita.
5. `reliability: outbox | best-effort` responde a una sola pregunta: *¿qué se pierde si el broker está caído justo cuando mi transacción confirma?*
6. Todo evento Keel viaja con la envoltura `metadata` + `data`, y el `eventId` **nunca se regenera** — por eso sirve de clave de deduplicación extremo a extremo.
7. `nature: fact | request` decide hacia dónde cae el acoplamiento, y `keel system check` comprueba que las dos partes lo lean igual: un encargo tratado como hecho es un mensaje que nadie atiende.
8. Un `retry` sin `deadLetter` no produce un fallo visible, y un canal que nadie usa ni un evento que nadie emite son avisos de `keel validate` que conviene tratar como errores.

---

## Notas relacionadas

- [`declarar-dependencias-entre-servidores.md`](declarar-dependencias-entre-servidores.md) — el plano de la **razón**: `needs` vs. `activations`, `awaits`, `onFailure` y compensaciones.
- [`canal-por-dominio-vs-canal-por-evento.md`](canal-por-dominio-vs-canal-por-evento.md) — cuántos canales declarar y con qué grano.
- [`aws-snssqs-guia.md`](aws-snssqs-guia.md) — cómo aterriza todo esto en SNS/SQS: raw delivery, filter policies, redrive.
- [`frontera-transaccional-per-operation-vs-per-aggregate.md`](frontera-transaccional-per-operation-vs-per-aggregate.md) — la frontera que sostiene el `outbox`.
