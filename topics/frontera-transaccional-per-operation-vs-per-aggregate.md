# Frontera transaccional: `per-operation` vs `per-aggregate`

**Respuesta corta:** la frontera transaccional responde a **una sola pregunta**: *¿puede una transacción abarcar más de un agregado?*

- `per-operation` → **sí**, sin límite. La transacción es la operación completa, toque las raíces que toque.
- `per-aggregate` → **no**. Máximo un agregado por transacción. Si una operación necesita tocar dos, van en transacciones separadas y aceptas que una confirme y la otra no.

Todo lo demás —contención, compensaciones, códigos de estado— se deriva de esa única pregunta.

---

## 1. Por qué esta decisión confunde

La confusión típica es esta: *"en las dos opciones me hablan de agregados; ¿entonces `per-operation` también usa agregados?"*.

La respuesta es que **un agregado es un concepto de tu modelo de dominio, no de la frontera transaccional**. Un agregado es un grupo de entidades que cambian siempre juntas y que tienen una **raíz** por la que se accede a todo el grupo: un `Order` con sus `OrderLine`, una `Invoice` con sus `InvoiceLine`. Ese grupo existe en tu dominio lo declares o no, y exista o no el campo `transactionalBoundary`.

Lo que hace `transactionalBoundary` es decidir si la transacción **puede cruzar** esa frontera:

| | ¿Existen agregados en el dominio? | ¿La transacción puede cruzarlos? |
|---|---|---|
| `per-operation` | Sí, siempre (los declares o no) | **Sí** |
| `per-aggregate` | Sí, y **tienes que declararlos** | **No** |

De ahí la asimetría en el DSL: el bloque `domain: aggregates` es **opcional**, pero `per-aggregate` lo vuelve **obligatorio**. Sin agregados declarados, la restricción "máximo uno por transacción" no se puede ni evaluar. `per-operation` no exige nada porque no restringe nada.

> **Idea clave:** `per-aggregate` no inventa los agregados. Los usa como **límite**.

---

## 2. Las dos opciones en detalle

### `per-operation`

La transacción es la operación entera. Si `placeOrder` escribe en tres tablas de tres agregados distintos, las tres escrituras confirman juntas o fallan juntas.

- **Garantía**: atomicidad total. Nunca existe un estado intermedio visible.
- **Coste**: los bloqueos de fila duran lo que dura toda la operación. A más filas tocadas y más tiempo, más contención bajo concurrencia.
- **En Spring (keel-spring)**: no hay que hacer nada especial. El `UseCaseMediator` ya abre una transacción por mensaje (las `Query` en modo `readOnly`, los `Command` en escritura), así que "la operación completa es la transacción" se cumple solo.

### `per-aggregate`

Cada transacción abarca **como máximo un agregado**: su raíz y sus entidades internas. Un command que necesita mutar dos raíces las muta en dos transacciones distintas.

- **Garantía**: atomicidad **dentro** de cada agregado. Entre agregados, consistencia **eventual**.
- **Ganancia**: bloqueos más cortos y más acotados.
- **Precio**: tienes que diseñar qué pasa cuando la primera transacción confirma y la segunda falla. Eso significa un **estado intermedio** en el modelo y una **compensación** o reconciliación declarada.
- **En Spring (keel-spring)**: el command solo puede tocar una raíz dentro de la transacción del mediator; nunca dos agregados en la misma. Es una regla inviolable del generador.

### Si no declaras nada

El campo no tiene valor por defecto en el schema, pero el generador Spring abre transacción por mensaje igualmente. **Omitirlo se comporta de facto como `per-operation`** — con la diferencia de que no queda escrito que lo hayas decidido. Si el servicio tiene agregados de verdad, decláralo.

---

## 3. Ejemplo 1 — Pedido + stock (aquí `per-aggregate` tiene sentido)

Servicio `orders`, dos agregados:

```yaml
# domain.keel.yaml
aggregates:
  Order:
    description: Un pedido y sus líneas cambian siempre juntos.
    root: Order
    members: [OrderLine]     # una línea nunca vive sin su pedido
  StockItem:
    description: El stock disponible de un producto.
    root: StockItem          # es su propia raíz: vive sin pedidos
```

La operación `placeOrder` crea el pedido **y** descuenta stock de 3 productos. Toca **dos** agregados.

### Con `per-operation`

Una sola transacción: `INSERT` en `order` + `INSERT` de las 3 `order_line` + `UPDATE` sobre 3 filas de `stock_item`.

- **Si el tercer producto no tiene stock:** falla todo. No hay pedido, no se descontó nada. El cliente recibe `409 INSUFFICIENT_STOCK` y el mundo queda exactamente como estaba.
- **Coste real:** mientras dura la operación completa —validaciones incluidas— esas 3 filas de `stock_item` están bloqueadas. Si son los productos estrella del Black Friday, todas las compras concurrentes de ese producto hacen cola detrás.

### Con `per-aggregate`

Dos transacciones, porque son dos agregados:

1. **TX1** — crea el `Order` en estado `PENDING_STOCK`.
2. **TX2** — reserva el stock de las 3 líneas.

- **Si TX2 falla, TX1 ya confirmó.** Existe un pedido en la base de datos sin stock reservado. Eso **no es un bug**: es el diseño que elegiste, y por eso necesitas la respuesta escrita de antemano. Las salidas legítimas son: pasar el pedido a `CANCELLED_NO_STOCK` y emitir `OrderCancelled`, o dejarlo en `PENDING_STOCK` para que un proceso lo reintente con una cota de tiempo.
- **Es observable en el contrato.** El cliente ya no recibe `201 Created` con un pedido confirmado: recibe `202 Accepted` con un pedido en estado pendiente, y se entera del resultado real después (por polling o por evento). Esto no es un detalle de implementación — cambia lo que el frontend tiene que pintar.
- **Ganancia:** el bloqueo sobre `stock_item` dura lo que dura TX2, no lo que dura toda la operación.

```yaml
# persistence.keel.yaml
consistency:
  transactionalBoundary: per-aggregate
  optimisticLocking: all
```

**Esta es toda la diferencia:** `per-operation` te da atomicidad y paga contención. `per-aggregate` te da menos contención y te cobra diseñar el estado intermedio.

---

## 4. Ejemplo 2 — Producto + marca + categoría (aquí `per-operation` es la respuesta)

Servicio `catalog`. La operación `createProduct` crea al vuelo la marca y la categoría si no existen, y después el producto. Tres raíces de agregado distintas (`Brand`, `Category` y `Product` viven cada una por su cuenta).

### Por qué `per-aggregate` sería mala elección aquí

Serían tres transacciones. Si la tercera falla —el SKU ya existe, una validación del producto no pasa— te quedas con **una marca y una categoría creadas que nadie pidió** y de las que ningún cliente sabe nada.

Eso es cualitativamente distinto del `PENDING_STOCK` del ejemplo anterior. Un pedido pendiente es un **estado intermedio con significado de negocio**: alguien lo puede consultar, cancelar, reintentar. Una marca huérfana no significa nada: es **basura**, y limpiarla exige un proceso de reconciliación que no aporta valor a nadie.

### Con `per-operation`

Una transacción: o sale todo o no sale nada. Que es exactamente lo que el negocio espera cuando dice "crear un producto".

```yaml
# persistence.keel.yaml
consistency:
  transactionalBoundary: per-operation
```

### El matiz que conviene no saltarse

Antes de fijar la frontera, **cuestiona la operación misma**. Un `createProduct` que crea la marca al vuelo es un *find-or-create* implícito, y eso tiene un coste propio: si el cliente escribe mal el nombre de la marca, en vez de un `404 BRAND_NOT_FOUND` te crea una marca duplicada y nadie se entera nunca. Alternativas habituales:

| Alternativa | Qué implica |
|---|---|
| **Referencias por id** (lo más común en catálogos maduros) | `createProduct` recibe `brandId` y `categoryId` que **ya existen**. La operación toca una sola raíz y la frontera deja de importar |
| **Find-or-create explícito** | La operación declara ese comportamiento en `rules`, con su error de colisión. Legítimo, pero **decidido**, no accidental |

Si tu caso es el primero, el problema desaparece solo. Si de verdad es el segundo —un alta guiada donde el usuario teclea marca y categoría nuevas en el mismo formulario— entonces sí: `per-operation`, por atomicidad, y así queda escrito.

---

## 5. Ejemplo 3 — El caso donde da exactamente igual

```yaml
updateProductPrice:
  kind: command
  # solo escribe la raíz Product
```

Si **todas** tus operaciones tocan una sola raíz, los dos valores generan **exactamente el mismo código**: una transacción, un agregado. La elección no cambia absolutamente nada del comportamiento.

Por eso la regla práctica es: si ninguna operación cruza agregados, pon `per-operation`. No renuncias a nada y no prometes una frontera que no estás usando. La decisión solo mueve la aguja el día que una operación cruza agregados y decides conscientemente que no sea atómica.

---

## 6. Cómo decidir: el procedimiento

### Paso 1 — Inventario

Recorre tus commands y apunta **cuántas raíces de agregado escribe cada uno** (escribe, no lee: leer otra raíz para validar una precondición no cuenta).

- **Todos escriben una sola raíz** → `per-operation`. Fin del análisis.
- **Alguno escribe dos o más** → sigue al paso 2.

### Paso 2 — La pregunta decisiva

> Si la segunda mitad falla, ¿es aceptable que la primera quede confirmada?

- **"No, sería un desastre"** (pedido sin descontar stock, cobro sin apunte contable, producto con marca huérfana) → `per-operation`.
- **"Sí, y así se arregla: ..."** — con el *"así se arregla"* **escrito**, no supuesto → `per-aggregate`.

Si tocas dos raíces y no sabes responder, la señal no es "elige una": es que **falta decidir el estado intermedio y la compensación** en el diseño.

### Paso 3 — Contrastar con los tres ejes

| Eje | Pregunta | Respuesta → decisión |
|---|---|---|
| **Atomicidad** | ¿Hay operaciones que tocan dos agregados y deben confirmar o fallar juntas? | "Sí, y deben ser atómicas" → `per-operation`. "Cada agregado por su cuenta" → `per-aggregate` |
| **Concurrencia** | ¿Cuánto contiende esta escritura? Una transacción por operación bloquea más filas y por más tiempo | Alta contención → `per-aggregate`, **si el negocio lo tolera** |
| **Consistencia aceptada** | Con `per-aggregate`, un cambio confirma y el otro no. ¿Qué se hace entonces? | Si no hay respuesta, la frontera está mal elegida o falta una compensación declarada |

---

## 7. Trampas y acoplamientos

### La trampa habitual

Elegir `per-aggregate` **por rendimiento** sin decidir qué pasa con la mitad que falló.

> La consistencia eventual es una **decisión de negocio**, no un ajuste de rendimiento.

Si el argumento para elegirla es "va más rápido" y no "el negocio tolera que el pedido quede pendiente unos segundos, y así se resuelve", la decisión está mal tomada. El coste no lo paga el diseño: lo paga producción, el día que aparece el primer estado intermedio que nadie modeló.

### La frontera es de **todo el servicio**, no por operación

`transactionalBoundary` vive en `persistence.consistency`, así que aplica a la capa entera. **No puedes mezclar.** Basta con que **una sola** operación cruce agregados y necesite atomicidad para que el servicio completo sea `per-operation`.

Si eso choca con otra operación de alta contención que querrías aflojar, la tensión **no se resuelve con este campo**: se resuelve rediseñando las fronteras de agregado, o partiendo el servicio.

### Acoplamiento con `outbox`

Si `messaging` declara `publishing.reliability: outbox`, **la escritura del evento comparte esta frontera**. Las dos decisiones se toman juntas, no por separado: la garantía de "el evento se publica si y solo si la operación confirmó" depende de cuál sea "la transacción" que confirma.

### No confundir con `optimisticLocking`

Son dos decisiones distintas y ortogonales, y es fácil mezclarlas porque viven en el mismo bloque `consistency`:

| Campo | Responde a | Escenario |
|---|---|---|
| `transactionalBoundary` | ¿La transacción puede abarcar **varios** agregados? | Una operación toca **raíces distintas** |
| `optimisticLocking` | ¿Qué pasa si dos escrituras concurrentes caen sobre la **misma** raíz? | Dos peticiones simultáneas sobre **la misma fila** |

`optimisticLocking: all` (por defecto) hace que la segunda escritura sobre una versión obsoleta reciba `409`; `none` es "último escritor gana". Es una decisión de concurrencia, no de frontera — pero también es de negocio y también es observable por el cliente.

---

## 8. Cómo lo trata Keel

**Dónde se declara** — capa `persistence`:

```yaml
consistency:
  transactionalBoundary: per-operation   # per-operation | per-aggregate
  optimisticLocking: all                 # all | declared | none
```

**Qué valida `keel validate`** — `per-aggregate` **exige** que `domain` declare `aggregates`. Si no los hay, es un **error duro, incluso con `--wip`**: no es un pendiente de diseño incompleto, es una contradicción entre capas.

`/keel-validate` además lo revisa al revés, como aviso semántico: si tienes agregados declarados cuyas operaciones tocan raíz + entidades internas y elegiste `per-operation`, te pregunta si no debería ser `per-aggregate`.

**Qué genera keel-spring:**

| Valor | Qué hace el generador |
|---|---|
| `per-operation` | Nada especial: la transacción por mensaje que abre el `UseCaseMediator` ya lo cumple |
| `per-aggregate` | El command debe tocar una sola raíz dentro de la transacción del mediator; nunca dos agregados en la misma |

En ambos casos, la transacción vive en el **mediator**, no en los handlers: los handlers no llevan `@Transactional` y no dependen de Spring. La única excepción documentada es `per-aggregate` con semántica especial (por ejemplo `REQUIRES_NEW`), y entonces se anota el handler y se documenta el porqué.

**La decisión es del diseñador, no del agente.** Con `per-aggregate` un cambio puede confirmar y el otro no; eso es consistencia eventual aceptada, y ningún generador puede tomar esa decisión por ti.

---

## En resumen

La frontera transaccional no decide si tienes agregados —los tienes igual—, decide si una transacción puede cruzarlos. Elige `per-operation` mientras ninguna operación necesite lo contrario, y reserva `per-aggregate` para cuando de verdad cruces agregados **y** tengas escrito qué pasa con la mitad que falla.
