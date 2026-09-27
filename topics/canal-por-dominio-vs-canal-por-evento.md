# Granularidad de canales: un canal por dominio vs. un canal por evento

**Respuesta corta:** la decisión responde a **una sola pregunta**: *¿quién filtra, el broker o el consumidor?*

- **Un canal por dominio** → filtra el **consumidor**. Recibe todo el flujo del dominio y descarta lo que no le toca.
- **Un canal por evento** → filtra el **broker**. El consumidor se suscribe exactamente al tipo que quiere y no ve nada más.

Todo lo demás —orden, retención, permisos, escalado, coste operativo— se deriva de esa única pregunta.

Y hay un sesgo de partida que conviene declarar ya: **empieza por dominio**. Separar después es barato; unir después no lo es, porque para entonces los consumidores ya se suscribieron a los canales que separaste.

---

## 1. Por qué esta decisión confunde

La confusión típica es tratar el canal como si fuera una unidad de **organización de código**: *"una cosa, un sitio; si tengo tres eventos, tres canales; así está más limpio"*.

Un canal no es una clase. Un canal es una unidad **operativa**, y lo que agrupas dentro de él comparte cuatro cosas, quieras o no:

| Un canal es la unidad de… | Qué significa |
|---|---|
| **Orden** | Los mensajes de un mismo canal (y misma clave de partición) llegan al consumidor en el orden en que se publicaron. Entre canales distintos **no hay ninguna garantía de orden** |
| **Retención** | Cuánto tiempo se guardan los mensajes es una política del canal entero, no del tipo de evento |
| **Permisos** | Quién puede leer y quién puede escribir se concede sobre el canal. Todo lo que metas dentro es visible para quien tenga lectura |
| **Coste operativo** | Cada canal es un objeto que se crea, se dimensiona, se monitoriza, se alerta y se documenta. Diez canales cuestan diez veces eso |

> **Idea clave:** no agrupas eventos "por parecido semántico". Agrupas por **qué política operativa comparten** y por **qué secuencia describen**.

Y el corolario que más gente pasa por alto: **partir un canal en dos rompe el orden entre lo que había dentro.** Si `ProductCreated` y `ProductRetired` viajan por canales distintos, nada impide que el consumidor vea la baja antes que el alta.

---

## 2. Las dos opciones en detalle

### Un canal por dominio

Un canal transporta **todos los eventos de un dominio**: `productEvents` lleva `ProductCreated`, `ProductPriceChanged` y `ProductRetired`.

```yaml
channels:
  productEvents:
    description: Ciclo de vida del producto que publica este servicio.

publishing:
  events:
    ProductCreated:      { channel: productEvents, ... }
    ProductPriceChanged: { channel: productEvents, ... }
    ProductRetired:      { channel: productEvents, ... }
```

- **Garantía que compras**: **orden**. Con la clave de partición puesta en la identidad de la entidad (`productId`), todo lo que le pasa a un producto llega en el orden en que ocurrió. Gratis, sin código.
- **Precio 1 — tráfico inútil**: un consumidor que solo quiere altas recibe también los cambios de precio y descarta el 90 % de lo que lee. Se paga en ancho de banda y en CPU de deserialización, no en corrección.
- **Precio 2 — hace falta discriminador**: si el canal transporta varias formas, el consumidor necesita un campo por el que saber qué acaba de recibir (típicamente `eventType` en la cabecera o en la envoltura). Sin él, el canal por dominio es inmanejable.
- **Precio 3 — políticas al peor común denominador**: la retención es la del evento que más necesita, y el permiso de lectura es el del evento más laxo. Todo el que lea el canal lee **todo** el canal.

### Un canal por evento

Cada tipo de evento tiene el suyo: `productCreatedEvents`, `productRetiredEvents`.

- **Garantía que compras**: **suscripción precisa**. El filtrado ocurre en el broker; el consumidor recibe exactamente lo suyo. Y con ello vienen tres cosas que solo se pueden dar por canal: permisos por evento, retención por evento y escalado (particiones, consumidores) por evento.
- **Precio 1 — se pierde el orden**: es el precio caro y el que se paga sin darse cuenta. Dos canales son dos particiones distintas; no hay ninguna garantía de orden relativo entre ellas. Las secuencias de negocio se vuelven **carreras**, y el consumidor tiene que tolerar llegadas fuera de orden explícitamente.
- **Precio 2 — proliferación**: cada evento nuevo es un objeto de infraestructura nuevo, con su alta, su dimensionado, su alerta y su documentación. Un servicio con doce eventos son doce canales que operar.
- **Precio 3 — el catálogo se vuelve el mapa**: con muchos canales, entender "qué publica este servicio" pasa por leer la lista de topics en vez de leer el diseño.

### Lo que **no** cambia con esta decisión

Conviene aislarlo porque se usa mucho como argumento falso:

- **El acoplamiento de contrato es el mismo.** Un consumidor depende del *payload* que lee, no del canal por el que le llega. Separar canales no desacopla nada; solo cambia dónde ocurre el filtrado.
- **La necesidad de idempotencia es la misma.** Con entrega *at-least-once* hay reentregas en las dos opciones.
- **La versión del contrato es un eje aparte.** Publicar la v2 de un evento no es "otro canal" por definición (ver §9).

---

## 3. Ejemplo 1 — Ciclo de vida de una entidad (aquí gana el canal por dominio)

Servicio `product-service`. Publica tres eventos sobre la misma entidad:

```
ProductCreated  →  ProductPriceChanged  →  ProductRetired
```

Un consumidor —`search-service`— mantiene una copia local del catálogo para indexarlo.

### Con un canal por evento

Tres canales. `search-service` se suscribe a los tres, y cada suscripción avanza a su ritmo. Entonces ocurre esto:

```
Realidad (t):    ProductCreated(P1)  ──  ProductPriceChanged(P1)  ──  ProductRetired(P1)
Lo que llega:    ProductRetired(P1)  ──  ProductCreated(P1)       ──  ProductPriceChanged(P1)
```

**No es un fallo del broker. Es lo que garantiza el broker:** nada, entre canales distintos. Basta con que el canal de bajas vaya menos cargado, o con que el de altas esté reintentando un mensaje anterior.

Y ahora el consumidor tiene que resolverlo:

- procesar `ProductRetired` de un producto que no tiene en su índice — ¿lo ignora? ¿lo guarda como "baja pendiente" esperando el alta?;
- procesar después `ProductCreated` — ¿lo indexa, resucitando un producto que ya está de baja?;
- llevar una **versión** o un `occurredAt` en cada registro y descartar lo que llegue viejo, que es la solución correcta pero es **trabajo, código y bugs**.

Eso es lo que cuesta de verdad separar por evento aquí: no la infraestructura, sino **meter reordenación en la lógica de todos los consumidores**.

### Con un canal por dominio

Un solo `productEvents`, con clave de partición = `productId`. Los tres eventos de P1 caen en la misma partición y llegan **en orden**. El consumidor aplica cada uno tal cual llega y se acabó.

Lo que necesita a cambio es discriminar. Con una envoltura estándar que lleve el tipo:

```json
{
  "metadata": { "eventType": "ProductPriceChanged", "eventId": "...", "occurredAt": "..." },
  "data":     { "productId": "...", "price": 19.90 }
}
```

el consumidor hace un despacho trivial:

```java
switch (envelope.metadata().eventType()) {
    case "ProductCreated"      -> index.upsert(...);
    case "ProductPriceChanged" -> index.updatePrice(...);
    case "ProductRetired"      -> index.remove(...);
    default                    -> { /* tipo que no me interesa: ack y fuera */ }
}
```

**Esta es toda la diferencia:** el canal por dominio mueve el filtrado a un `switch` de diez líneas y te regala el orden. El canal por evento te quita el `switch` y te cobra reordenar a mano en cada consumidor.

---

## 4. Ejemplo 2 — Eventos con políticas incompatibles (aquí gana el canal por evento)

Servicio `order-service`. También publica tres eventos, y también son "del mismo dominio". Pero mira lo que hay debajo:

| Evento | Volumen | Retención que necesita | Quién puede leerlo |
|---|---|---|---|
| `OrderPlaced` | ~10 k/día | 7 días (reproceso de consumidores) | Media empresa |
| `OrderTrackingPing` | ~5 M/día (posición del repartidor cada 10 s) | 30 minutos | El servicio de mapas |
| `OrderPaymentFailed` | ~200/día | 30 días (auditoría) | **Solo facturación** — lleva datos de pago |

Meterlos en un único `orderEvents` fuerza el **peor común denominador en los tres ejes a la vez**:

- **Retención**: 30 días, porque `OrderPaymentFailed` los necesita. Eso son 150 millones de pings guardados un mes para nada.
- **Permisos**: quien lea el canal para consumir `OrderPlaced` —media empresa— lee también los fallos de pago. No hay forma de conceder lectura "de un tipo": el permiso es del canal.
- **Escalado**: el canal se dimensiona para 5 M/día. `OrderPlaced` queda enterrado detrás del volumen del tracking, y un pico de repartidores retrasa el procesamiento de pedidos.

Y lo decisivo: **estos tres eventos no describen la misma secuencia**. Que un ping llegue antes o después de un `OrderPlaced` no significa nada para nadie. **No hay orden que perder**, así que el precio principal del canal por evento aquí es cero.

```yaml
channels:
  orderEvents:          # OrderPlaced, OrderConfirmed, OrderCancelled — misma secuencia, mismas políticas
    description: Ciclo de vida del pedido.
  orderTrackingEvents:  # volumen enorme, retención de minutos
    description: Posición del reparto en curso.
  orderBillingEvents:   # datos sensibles, lectura restringida, retención larga
    description: Incidencias de cobro del pedido.
```

Fíjate en que la respuesta no ha sido "un canal por cada uno de los diez eventos": ha sido **tres grupos**, y cada grupo es un conjunto de eventos que comparte secuencia y política. Que es exactamente el punto medio del que va la §8.

---

## 5. Ejemplo 3 — El caso donde da exactamente igual

Un servicio que publica **un solo evento**. Las dos opciones producen literalmente el mismo topic. La decisión no cambia nada… **hoy**.

Por eso la regla práctica es asimétrica, y merece la pena entender por qué:

- **De un canal a dos** (separar): publicas el evento nuevo en el canal nuevo, avisas a quien lo consuma, y ya. El cambio es aditivo.
- **De dos canales a uno** (unir): todos los consumidores existentes están suscritos a los canales viejos. Hay que publicar en ambos durante un tiempo, migrar consumidor por consumidor, y solo entonces apagar los viejos. Es una migración coordinada entre equipos.

**Empieza agrupado.** Sale más barato el día que te equivoques.

---

## 6. Cómo decidir: el procedimiento

### Paso 1 — ¿Describen la misma secuencia sobre la misma entidad?

Coge dos eventos y pregúntate: *¿pueden ocurrirle los dos a la misma instancia, y el orden entre ellos significa algo?*

- **Sí** (`ProductCreated` y `ProductRetired` del mismo producto) → **mismo canal**, salvo que el paso 2 dé una razón muy fuerte. Fin del análisis.
- **No** (un `OrderPlaced` y un `OrderTrackingPing`) → sigue al paso 2, donde no arriesgas nada al separar.

### Paso 2 — La pregunta decisiva

> ¿Hay algún eje **operativo** en el que estos eventos exijan políticas distintas?

Los ejes son cuatro: **volumen/retención**, **permisos y sensibilidad de los datos**, **SLA y escalado**, y **ciclo de vida** (uno cambia cada mes, otro lleva dos años estable).

- **Ninguno** → un canal. Separar solo añade objetos que operar, sin comprar nada.
- **Alguno, y es real** (no "es que quedaría más limpio") → separa **por ese eje**, agrupando lo que comparte política.

### Paso 3 — Contrastar con la tabla

| Eje | Pregunta | Respuesta → decisión |
|---|---|---|
| **Orden** | ¿El consumidor necesita verlos en el orden en que ocurrieron? | Sí → **un canal**. Es la razón más fuerte de todas |
| **Volumen** | ¿Hay uno que sea 100× los demás? | Sí → **sepáralo**, o ahogará a los otros |
| **Retención** | ¿Necesitan tiempos de vida muy distintos? | Sí → **separa**: la retención es del canal |
| **Permisos / PII** | ¿Alguno lleva datos que no todo consumidor debe ver? | Sí → **sepáralo**. Es un requisito, no una preferencia |
| **Fan-out** | ¿Los consumen los mismos o conjuntos disjuntos? | Los mismos → **un canal**. Disjuntos → separar tiene sentido |
| **Coste operativo** | ¿Cuántos canales estás dispuesto a operar y documentar? | Pocos → **agrupa**. Es un coste recurrente, no de una vez |

Si los ejes empujan en direcciones opuestas, **el orden gana**. Es el único que no puedes recuperar con código barato en el consumidor.

---

## 7. Un matiz importante: qué es "un dominio"

"Un canal por dominio" se malinterpreta con facilidad hacia arriba. Dominio **no** es:

- **la empresa entera** — un `domainEvents` global donde publica todo el mundo. Es el antipatrón de §9;
- **el servicio entero**, sin más. Un servicio grande puede publicar cosas que no tienen nada que ver entre sí (es justo el ejemplo 2).

La unidad que funciona en la práctica es **el agregado o el contexto acotado**: el conjunto de eventos que le ocurren a **la misma cosa** y que por tanto forman **una secuencia**. `orderEvents` (el ciclo de vida del pedido) y `orderTrackingEvents` (la telemetría del reparto) están en el mismo servicio y en el mismo "dominio de negocio" en sentido laxo — pero no le pasan a la misma cosa ni comparten política, así que son dos canales.

---

## 8. El punto medio, que suele ser la respuesta real

Casi nunca eliges entre los dos extremos puros. La respuesta habitual es:

> **Canal por agregado, con discriminador, y filtrado en el consumidor.**

Es decir: agrupa lo que comparte secuencia, mete el tipo de evento en la envoltura, y deja que cada consumidor descarte lo que no le sirve. Ganas el orden, mantienes pocos canales que operar, y el precio es ancho de banda —el recurso más barato de los que están en juego.

Y sobre ese punto medio, dos escapes que evitan tener que ir al extremo:

- **Filtrado en el broker sin partir el canal.** Varios brokers permiten enrutar por cabecera o por *routing key* sin crear un canal por tipo (los *topic exchange* de RabbitMQ son exactamente eso; en Kafka lo suele resolver un consumidor que filtra, o un stream derivado). Si tu problema era solo "no quiero leer lo que no me toca", esto lo resuelve sin romper el orden.
- **Canal derivado para el consumidor exigente.** Si un consumidor concreto necesita un flujo filtrado y de alto volumen, se le puede materializar un canal derivado a partir del canal principal. El canal de dominio sigue siendo la fuente de verdad ordenada; el derivado es una vista.

---

## 9. Trampas

### Separar "por limpieza" y romper una secuencia sin enterarte

Es la trampa número uno y no da la cara en desarrollo: con poco tráfico, los mensajes llegan casi siempre en orden. El bug aparece en producción, bajo carga, y se manifiesta como "a veces un producto se queda sin indexar". Si separas eventos que le ocurren a la misma entidad, **eso es una decisión de diseño con consecuencias en el código de todos los consumidores**, y hay que escribirla como tal.

### El `domainEvents` global de la empresa

El extremo opuesto y también real: un único canal donde publica todo el mundo. La retención es imposible de fijar (alguien siempre necesita más), los permisos son "todos leen todo", y cualquier consumidor deserializa el tráfico de veinte servicios para quedarse con el 0,5 %. La agrupación tiene un límite superior y es el **contexto acotado**, no la organización.

### Confundir canal con versión

Publicar la v2 de un evento con cambios incompatibles **no es por defecto "otro canal"**. Son dos decisiones distintas: la granularidad responde a *cómo agrupo eventos*; el versionado responde a *cómo convivo con consumidores viejos*. Lo habitual es versionar dentro del mismo canal (un `eventVersion` en la envoltura, consumidores que ignoran lo que no entienden) y reservar el canal aparte para roturas grandes con migración coordinada.

### Creer que el canal por evento elimina el discriminador

No lo elimina: lo **aplaza**. El día que ese canal transporte dos formas del mismo evento (una v1 y una v2, típicamente), el consumidor necesita distinguirlas igual. Estampar el tipo y la versión en la envoltura es barato y hay que hacerlo en las dos opciones.

### Dejar que el nombre físico del topic se cuele en el diseño

`prod-eu-west-1.product.events.v2` no es un nombre de diseño: es un nombre de despliegue, con entorno y región dentro. El diseño declara un canal **lógico** (`productEvents`) y su propósito; el nombre real se resuelve al desplegar. Si el spec lleva el nombre físico, no se puede desplegar dos veces en sitios distintos.

### Olvidar que la clave de partición es parte de la decisión

Un canal por dominio **no garantiza orden por sí solo**: lo garantiza dentro de una partición. Si publicas sin clave, o con clave aleatoria, tienes el canal agrupado y el orden roto de todas formas — lo peor de las dos opciones. La clave debe ser la identidad de la entidad de la que va la secuencia (`productId`, `orderId`).

---

## 10. Cómo lo trata Keel

**Dónde se declara** — capa `messaging`:

```yaml
channels:
  productEvents:
    description: Ciclo de vida del producto que publica este servicio.
  inventoryEvents:
    description: Eventos de inventory-service que este servicio consume.
    external: true          # el canal lo posee otro sistema
```

Y cada evento referencia el suyo por nombre desde `publishing.events.<Evento>.channel` (o desde `subscriptions.<Evento>.channel`, al consumir).

**El canal es lógico, no físico.** En el DSL solo se declara el nombre y el propósito; que eso se materialice en un *topic* de Kafka o en una cola/exchange de RabbitMQ —y con qué nombre real— se decide al **generar** y al desplegar, nunca en el spec. En un canal `external: true`, el nombre físico ya existe fuera y entra como parámetro de despliegue.

**La envoltura estándar es lo que hace viable el canal por dominio.** Todo evento que publica un servicio Keel sale envuelto en `{ metadata, data }`, y `metadata.eventType` lleva el nombre del evento. Ese campo **es** el discriminador de §3: es lo que permite que `productEvents` transporte varios tipos sin ambigüedad. Su contraparte al consumir sistemas ajenos es el bloque `contract` de la suscripción (`envelope`, `payloadPath`, `discriminator`), donde se declara cómo reconocer el evento en el cable de un emisor que no es Keel.

**`channel` es opcional.** Un diseño puede omitirlo y dejar el enrutado a convención del generador. Es aceptable mientras nadie te consuma; es mal negocio en cuanto el servicio se integra con otros, porque el canal declarado *es* parte del contrato de integración (por dónde emites, de dónde consumes).

**Qué valida `keel validate`** — que el canal referenciado exista (referencia cruzada) y avisa de canales declarados que no usa nadie (canal huérfano). Lo que **no** puede validar es si la agrupación es la correcta: eso es semántico, y lo revisa `/keel-validate`.

**La decisión es del diseñador, no del agente.** Es observable por los consumidores, es cara de deshacer, y —a través del orden— cambia el código que tienen que escribir. Ningún generador puede tomarla por ti; a lo sumo aplica una convención cuando no la has tomado.

---

## En resumen

Un canal no organiza tu código: organiza **orden, retención y permisos**. Por eso la pregunta no es "¿cuántos eventos tengo?" sino "¿qué comparten estos eventos?".

**Agrupa lo que comparte secuencia** —lo que le ocurre a la misma entidad, donde el orden significa algo— y **separa solo cuando un eje operativo lo exija**: volumen desproporcionado, datos sensibles, retenciones incompatibles. En la duda, agrupa: separar más tarde es un cambio aditivo, y unir más tarde es una migración entre equipos.
