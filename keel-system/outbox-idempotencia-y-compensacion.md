# Outbox, idempotencia y compensación: el plano de ejecución

> **Qué responde esta nota.** Qué tablas crea el servidor que genera `keel-spring`, qué función cumple cada
> una, cómo se purgan, y qué pasa cuando el servicio corre con varias réplicas a la vez.
>
> **Alcance.** Es el plano del **código en ejecución**. Del DSL solo se repite lo imprescindible para
> entender por qué existe cada pieza; el plano del diseño —schemas completos, reglas de `keel validate`,
> cómo se declara cada cosa en YAML— está en [`idempotencia-y-compensaciones.md`](./idempotencia-y-compensaciones.md).
>
> Todas las rutas son relativas a `packages/keel-spring/` del repo `keel-system`, salvo que se diga otra cosa.
> Fecha: 2026-08-10.

---

## 0. El mapa de una página

Keel distingue **cinco mecanismos** de repetición y compensación. Confundirlos es el error caro: cada uno
tapa un agujero distinto, y ninguno sustituye a otro. Encima de los cinco, transversal, está el **outbox**,
que es la garantía de entrega de los eventos que este servicio publica.

| # | Mecanismo | Qué problema tapa | Se declara en | Artefacto persistente | Quién arbitra entre réplicas |
|---|---|---|---|---|---|
| — | **Outbox** | El evento se pierde si el broker está caído en el instante del commit | `messaging.publishing.reliability: outbox` | **`outbox_event`** | Lock de fila (`SKIP LOCKED`) o `claimed_at` con caducidad |
| 1 | Repetición del llamante HTTP | El cliente reintenta un `POST` con timeout y crea dos recursos | `use-cases.<op>.idempotency` | **`idempotency_record`** | PK `(operation_scope, idempotency_key)` |
| 2 | Reentrega del broker | El broker entrega dos veces el mismo mensaje | `subscriptions.<E>.contract.messageId` (o `envelope: keel`) | **`processed_event`** | PK `(handler_id, event_id)` |
| 3 | Nuestro reintento contra un proveedor | *Nuestro* `@Retry` cobra dos veces al otro lado | `http-clients.calls.<x>.idempotency` | ninguno | El proveedor |
| 4 | **Compensación** | Hay que deshacer trabajo ya encargado a otro servidor | `dependencies.<d>.compensations` | ninguno | Ninguno propio: **prestado** |
| 5 | **Reconciliación** | El desenlace **nunca llega** y nadie lo nota | `activations.<a>.reconciledBy` + una operación con `schedule` | ninguno | **Ninguno**: lo escribe el agente |

Tres tablas en total. Y una asimetría que conviene tener presente desde el principio: **los cuatro primeros
tienen un árbitro que no está en la JVM** (la base de datos, o el proveedor). Los dos últimos no tienen
ninguno propio, y ahí es donde el servidor generado puede fallar en silencio.

### Quién escribe qué

`keel-spring build` genera el **mecanismo**; el agente de código escribe el **uso**.

| Mecanismo | Lo genera `build` | Lo escribe el agente |
|---|---|---|
| Outbox | Tabla, repositorio, relay, purga, puerto `OutboxDispatcher`, fallback | La implementación del dispatcher (envío al broker) |
| Idempotencia de petición | Tabla, `IdempotencyStore`, `CommandSignature`, filtro, contexto, purga | La consulta del store en el handler |
| Idempotencia de consumo | Tabla, `ProcessedEventWriter`, `IdempotencyGuard`, purga | El listener que llama al guard, en el orden que el javadoc prescribe |
| Idempotencia saliente | `OutboundIdempotency` **y la cabecera ya cableada** | Nada |
| Compensación | **Nada.** Solo javadoc y notas en el stub | Todo |
| Reconciliación | El `@Scheduled` con su cron y la nota de qué barrer | La consulta de candidatos, el umbral y la decisión |

Ese reparto es exactamente lo que verifica `infra/check-idempotency.sh` (§8).

---

## 1. Outbox

### 1.1 Por qué existe

Publicar un evento después de commitear tiene una ventana: si el broker está caído justo ahí, el cambio de
estado quedó guardado y el hecho no salió nunca. Nadie se entera — no hay excepción que capturar, porque la
transacción de negocio fue perfecta.

El outbox cierra esa ventana escribiendo el evento **como una fila más, en la misma transacción que el
agregado**. O commitean los dos o no commitea ninguno. La entrega al broker pasa a ser un problema
posterior y reintentable.

Coste: la entrega deja de ser exactly-once y pasa a ser **at-least-once**. El consumidor tiene que
deduplicar (§3). El outbox no elimina duplicados: los cambia por una garantía de no-pérdida.

### 1.2 La escritura: `@EventListener` síncrono, y no es un descuido

El detalle que decide si el outbox funciona está en una sola línea de `src/scaffold/messaging.js:208`:

```js
const listener = outbox ? '@EventListener' : '@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)';
```

En modo `best-effort` el bridge escucha **después del commit** — es lo correcto: publicar dentro de una
transacción que puede revertir sacaría eventos de cambios que nunca ocurrieron. En modo outbox es al revés:
el listener tiene que ser **síncrono y dentro de la transacción**, porque lo que hace no es publicar, es
insertar una fila que debe compartir destino con el agregado.

El `append` que genera (`messaging.js:231-251`):

```java
private void append(String routingKey, String eventType, EventEnvelope<?> envelope) {
    try {
        outboxRepository.save(new OutboxEventJpa(
                UUID.randomUUID(),
                destination,
                routingKey,
                eventType,
                objectMapper.writeValueAsString(envelope),
                Instant.now(),
                null,   // publishedAt: null = pendiente
                0,      // attempts
                null,   // lastError
                null)); // nextAttemptAt: null = elegible ya
    } catch (JsonProcessingException ex) {
        // Serializar un evento propio no puede fallar: si falla, el diseño
        // del payload está roto y la transacción debe revertir.
        throw new IllegalStateException("No se pudo serializar el evento " + eventType, ex);
    }
}
```

Lo que se guarda es la **`EventEnvelope` ya serializada**: el relay no vuelve a tocarla, solo la manda tal
cual. Y el `metadata.eventId` de esa envoltura es el que estampó el agregado en su `raise` — no se
regenera nunca, porque es la clave con la que el consumidor deduplica.

Con outbox **no se generan** los puertos `<Evento>Publisher` ni sus stubs (`messaging.js:261-262`): la
entrega es del relay, y tener dos caminos de salida sería tener uno sin garantía.

### 1.3 La tabla `outbox_event`

`src/scaffold/outbox.js:49-97`:

```java
@Entity
@Table(name = "outbox_event", indexes = {
        @Index(name = "ix_outbox_event_pending", columnList = "published_at, created_at")
})
public class OutboxEventJpa {

    @Id
    private UUID id;

    /** Exchange / topic destino, tal como lo resolvió el bridge. */
    @Column(name = "destination", nullable = false)
    private String destination;

    /** Clave de enrutado dentro del destino. */
    @Column(name = "routing_key", nullable = false)
    private String routingKey;

    /** Tipo del evento de integración serializado (trazabilidad y filtros). */
    @Column(name = "event_type", nullable = false)
    private String eventType;

    /** EventEnvelope serializada a JSON: lo que viaja tal cual al broker. */
    @Column(name = "payload", nullable = false, columnDefinition = "text")
    private String payload;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    /** Null mientras esté pendiente; es lo que distingue una fila por entregar. */
    @Column(name = "published_at")
    private Instant publishedAt;

    @Column(name = "attempts", nullable = false)
    private int attempts;

    /** Momento a partir del cual la fila vuelve a ser elegible; null = ya. */
    @Column(name = "next_attempt_at")
    private Instant nextAttemptAt;

    @Column(name = "last_error", length = 1024)
    private String lastError;
```

| Columna | Tipo | Función |
|---|---|---|
| `id` | `UUID` (PK) | Identidad de la fila. **No** es el `eventId` del evento |
| `destination` | `varchar` | Exchange / topic, resuelto por el bridge |
| `routing_key` | `varchar` | Clave de enrutado dentro del destino |
| `event_type` | `varchar` | Nombre del evento de integración; trazabilidad y filtros |
| `payload` | `text` | La `EventEnvelope` en JSON, lista para salir |
| `created_at` | `timestamp` | Orden de entrega (FIFO por antigüedad) |
| `published_at` | `timestamp` nullable | **Null = pendiente.** Es el estado de la fila |
| `attempts` | `int` | Intentos consumidos. Al llegar a `max-attempts`, dead-letter |
| `next_attempt_at` | `timestamp` nullable | Backoff. Null = elegible ya |
| `last_error` | `varchar(1024)` | Último mensaje de error, truncado |

Un único índice, `ix_outbox_event_pending (published_at, created_at)`, que es exactamente el orden de
claves con el que consulta el relay: pendientes primero, y dentro de ellas por antigüedad.

**No hay restricción de unicidad.** Nada impide dos filas con el mismo `eventId` dentro del payload — y de
hecho el diseño lo permite a propósito: si el proceso muere entre `dispatch` y commit, la fila sigue
pendiente y se reenvía. Quien resuelve eso es el consumidor.

#### Espejo documental

Con `persistence.default.model: document` la entidad es `OutboxEventDocument`
(`outbox.js:258-311`), con los mismos campos como `@Field(name = ...)` en snake_case y **uno más**:

```java
/**
 * Marca de reclamo: la estampa el relay que se lleva la fila, y es lo que evita
 * que dos réplicas entreguen el mismo evento. Caduca (ver la cabecera del
 * generador): una réplica que muera con filas reclamadas no las retiene.
 */
@Field(name = "claimed_at")
private Instant claimedAt;
```

Ese campo es el sustituto del lock de fila, y se explica en §7.2.

En Mongo el índice no sale de una anotación: la creación automática está apagada, así que lo pide
explícitamente `src/scaffold/document-indexes.js:145-155` y lo materializa `MongoIndexConfig` como
`ApplicationRunner`. Ninguno de los tres índices de infraestructura es único: en `processed_event` y en
`idempotency_record` la unicidad ya la da el `_id`, que Mongo indexa siempre.

### 1.4 Aviso: el DDL no está escrito en ninguna parte

En la rama relacional **no hay ninguna migración Flyway con el `CREATE TABLE` de estas tres tablas**.
`src/main/resources/db/migration/` nace vacío (solo un `README.md`), y el baseline lo **exporta el agente
de calidad** desde las entidades JPA con `infra/export-schema.sh`, verificando después que el DDL conserva
los nombres de constraint del diseño.

Consecuencia práctica: **la definición autoritativa del esquema son las anotaciones JPA de arriba.** Si
alguien quiere saber qué columnas tiene `outbox_event` en producción, lo que manda es `OutboxEventJpa`, no
un `.sql` que no existe hasta que alguien lo genera.

### 1.5 El relay

`src/scaffold/outbox.js:746-769`:

```java
@Scheduled(fixedDelayString = "${outbox.relay.fixed-delay-ms:1000}")
@Transactional
public void relay() {
    List<OutboxEventJpa> pending = outboxRepository.findPending(maxAttempts, Instant.now(), PageRequest.of(0, batchSize));
    for (OutboxEventJpa row : pending) {
        try {
            dispatcher.dispatch(row.getDestination(), row.getRoutingKey(), row.getEventType(), row.getPayload());
            row.markPublished(Instant.now());
        } catch (RuntimeException ex) {
            row.markFailed(truncate(ex.getMessage()));
            if (row.getAttempts() >= maxAttempts) {
                // Dead-letter: agotó los reintentos. Queda parada (fuera de futuros
                // polls) para inspección manual; no se borra ni bloquea al resto.
                log.error("Outbox: {} agotó {} reintentos y queda como dead-letter: {}",
                        row.getId(), maxAttempts, ex.getMessage());
            } else {
                row.scheduleNextAttempt(Instant.now().plusMillis(backoffDelayMs(row.getAttempts())));
                log.warn("Outbox: fallo entregando {} (intento {}): {}", row.getId(), row.getAttempts(), ex.getMessage());
            }
        }
    }
}
```

Cuatro cosas que no son estilo:

1. **Un fallo de entrega no revierte nada.** El `catch` está dentro del bucle: una fila que falla no
   arrastra al resto del lote, y el incremento de `attempts` commitea igual.
2. **La dead-letter no se borra ni bloquea.** Deja de aparecer en `findPending` (`attempts < :maxAttempts`)
   y se reporta a `ERROR`. Queda en la tabla para inspección — y no la toca ninguna purga (§6).
3. **Backoff exponencial con guarda de desbordamiento** (`outbox.js:773-780`):

   ```java
   private long backoffDelayMs(int attempts) {
       int shift = Math.min(attempts - 1, 62);
       long delay = backoffInitialMs << shift;
       if (delay < 0 || delay > backoffMaxMs) {
           return backoffMaxMs;
       }
       return delay;
   }
   ```

   El `Math.min(…, 62)` y el `delay < 0` son la misma precaución vista dos veces: un desplazamiento de 64
   bits da resultados absurdos, y sin la guarda un `attempts` alto produciría un retraso negativo, es
   decir, reintento inmediato en bucle apretado justo cuando el broker está caído.
4. **Todo sale de `parameters/`, nunca del código**: cadencia, tamaño de lote, tope de intentos, backoff y
   retención (§6.3).

La consulta que alimenta el bucle es la pieza clave para multi-instancia y se detalla en §7.1.

### 1.6 El puerto y su fallback

La única frontera con el broker en todo el patrón (`outbox.js:609-624`):

```java
public interface OutboxDispatcher {
    /**
     * Envía el payload al destino indicado. Debe lanzar excepción si la entrega
     * no se confirma: el relay cuenta el intento y reintenta en la pasada siguiente.
     */
    void dispatch(String destination, String routingKey, String eventType, String payload);
}
```

Y su fallback, que merece leerse entero porque encapsula el fallo más peligroso del mecanismo
(`outbox.js:648-684`):

```java
@Configuration
public class OutboxDispatcherFallbackConfig {

    /** Perfiles en los que arrancar sin broker es legítimo. */
    private static final Set<String> TOLERATED = Set.of("local", "test");

    @Bean
    @ConditionalOnMissingBean(OutboxDispatcher.class)
    public OutboxDispatcher outboxDispatcherStub(Environment environment) {
        List<String> active = List.of(environment.getActiveProfiles());
        if (!active.isEmpty() && active.stream().noneMatch(TOLERATED::contains)) {
            throw new IllegalStateException(
                "No hay implementación de OutboxDispatcher y el perfil activo es " + active
                    + ": el relay marcaría como publicados eventos que nunca salen del proceso. …");
        }
        log.warn("OutboxDispatcher sin implementar: los eventos NO salen del proceso (perfil {})", active);
        return (destination, routingKey, eventType, payload) -> {
            // TODO (agente): sustituir este stub por el dispatcher real del broker …
            log.warn("OutboxDispatcher no implementado: {} no salió a {}/{}", eventType, destination, routingKey);
        };
    }
}
```

Las tres decisiones, ninguna de estilo:

- **`dispatch` no lanza.** Si lanzara, el relay lo contaría como fallo de entrega y todas las filas se
  irían acumulando hasta la dead-letter.
- **Es `@Bean` + `@ConditionalOnMissingBean`, no `@Component`.** El dispatcher real lo desplaza solo. Con
  dos `@Component` del mismo puerto el contexto no arranca, y el camino de menor resistencia sería *borrar
  este archivo* — y entonces nada quedaría avisando.
- **Falla al arrancar fuera de `local`/`test`.** Es la consecuencia del primer punto: si `dispatch` no
  lanza, el relay marca como publicadas filas que nunca salieron, y `reliability: outbox` —elegido
  precisamente para no perder ningún evento— se convierte en perderlos **todos** sin un solo error.

### 1.7 La garantía real

**At-least-once, no exactly-once.** Si el proceso muere entre el `dispatch` que ya salió y el commit que lo
marcaría publicado, la fila sigue pendiente y la siguiente pasada la reenvía. Eso no es un defecto del
generador: es el teorema. El duplicado lo absorbe el `processed_event` del consumidor (§3), y por eso los
dos mecanismos son la misma historia contada desde los dos extremos del cable.

---

## 2. Idempotencia de petición — `idempotency_record`

### 2.1 Qué contrato implementa

No es «rechazar la repetición»: es **reproducirla**. La segunda llamada con la misma clave y el mismo
contenido devuelve la respuesta de la primera sin volver a ejecutar nada. De ahí las dos columnas que
importan: un `resource_id` para reconstruir la respuesta, y una `signature` para detectar que alguien
reutilizó su clave con otro cuerpo.

### 2.2 La tabla

`src/scaffold/http-idempotency.js:383-452`:

```java
@Entity
@Table(name = "idempotency_record", indexes = {
        @Index(name = "ix_idempotency_record_expires_at", columnList = "expires_at")
})
public class IdempotencyRecordJpa {

    @EmbeddedId
    private IdempotencyRecordId id;

    /** Representación determinista del contenido con el que se usó la clave. */
    @Column(name = "signature", nullable = false, length = 128)
    private String signature;

    @Column(name = "resource_id", length = 255)
    private String resourceId;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @Column(name = "expires_at", nullable = false)
    private Instant expiresAt;

    @Embeddable
    public static class IdempotencyRecordId implements Serializable {

        /** Nombre de la operación del diseño. La columna no se llama "scope": lo es en SQL estándar. */
        @Column(name = "operation_scope", nullable = false, length = 128)
        private String scope;

        @Column(name = "idempotency_key", nullable = false, length = 255)
        private String idempotencyKey;
```

| Columna | Función |
|---|---|
| `operation_scope` (PK) | Nombre de la operación. La misma cabecera en dos operaciones distintas **no** colisiona |
| `idempotency_key` (PK) | El valor de `Idempotency-Key`, o la firma del payload con `keySource: payload-hash` |
| `signature` | Firma canónica del contenido; detecta la clave reutilizada con otro cuerpo |
| `resource_id` | Id del recurso creado, para reconstruir la respuesta. Null si la operación no crea nada |
| `created_at` | Traza |
| `expires_at` | **Caducidad ya calculada.** No se deduce del TTL al consultar |

Dos decisiones con motivo:

- **La columna se llama `operation_scope`.** `scope` es palabra reservada en SQL estándar.
- **`expires_at` se guarda calculado**, no derivado del `ttlSeconds` al leer: el TTL del diseño puede
  cambiar entre despliegues, y las filas ya escritas deben conservar la ventana con la que se registraron.

El índice `ix_idempotency_record_expires_at` es **de la purga**, no de la lectura: esta va siempre por
clave primaria. Sin él, cada réplica recorrería la tabla entera a la misma hora.

### 2.3 La cadena completa

```
Idempotency-Key (cabecera HTTP)
      ↓  IdempotencyKeyFilter   (@Order(HIGHEST_PRECEDENCE + 11); set() en try, clear() en finally)
   IdempotencyContext           (ThreadLocal; get() → Optional<String>)
      ↓
   <Operacion>Handler           ← lo escribe el AGENTE
      ↓  IdempotencyStore (puerto, domain.idempotency)
   JpaIdempotencyStore / MongoIdempotencyStore
      ↓
   idempotency_record
```

El filtro y el contexto **solo se generan si alguna operación declara `keySource: client-key`**
(`http-idempotency.js:80-84`). Con `payload-hash` no hay cabecera que transportar: la clave **es**
`CommandSignature.of(command)`.

El `ThreadLocal` con `clear()` en `finally` no es ceremonia: los hilos son de un pool y se reutilizan. Un
contexto sin cerrar hace que la siguiente petición atendida por ese hilo herede una clave ajena y se salte
su propia ejecución.

### 2.4 `CommandSignature`: por qué está generada

`http-idempotency.js:262-337`. La firma se compara contra otra guardada en **otro despliegue**. Si dos
handlers la calculan distinto —o el mismo la calcula distinto tras un refactor— la comparación deja de
significar nada y **nada lo delata**: el sistema simplemente deja de deduplicar.

```java
/** @return SHA-256 en hexadecimal de la forma canónica del command */
public static String of(Object command) {
    return HexFormat.of().formatHex(sha256(canonical(command).getBytes(StandardCharsets.UTF_8)));
}
```

Reglas de la forma canónica, cada una tapando un fallo concreto:

| Regla | Qué evita |
|---|---|
| Componentes de record **ordenados por nombre** | Que el orden de declaración cambie la firma tras un refactor |
| Escalares con **prefijo de longitud** (`raw.length() + ":" + raw`) | Que un contenido pueda imitar un separador |
| `null` → `"~"` (marca propia, no omisión) | Colapsar «ausente» y «nulo», que el contrato distingue |
| `BigDecimal.stripTrailingZeros().toPlainString()` | Que `1.50` y `1.5` den firmas distintas siendo el mismo importe |
| `byte[]` → su digest | Que entre la **identidad del objeto** del array, distinta en cada ejecución |
| `Collection` conserva el orden; `Map` se ordena | Dos listas con distinto orden son dos peticiones distintas; un mapa, no |
| **No usa Jackson, a propósito** | Que un cambio del `ObjectMapper` (precisión temporal, `@JsonInclude`) mueva en silencio firmas ya almacenadas |

### 2.5 Transaccionalidad: `REQUIRED`, al revés que el guard de consumo

`http-idempotency.js:582-599`:

```java
@Override
@Transactional   // ← propagación por defecto: se UNE a la transacción del caso de uso
public void save(String scope, String idempotencyKey, String signature, String resourceId, long ttlSeconds) {
    Instant now = Instant.now();
    try {
        // saveAndFlush y no save: JPA difiere el INSERT hasta el commit, y ahí la
        // violación de clave ya no la ve este método …
        repository.saveAndFlush(new IdempotencyRecordJpa(
                new IdempotencyRecordJpa.IdempotencyRecordId(scope, idempotencyKey),
                signature, resourceId, now, now.plusSeconds(ttlSeconds)));
    } catch (DataIntegrityViolationException concurrent) {
        throw new IdempotencyConflictException(scope, idempotencyKey, concurrent);
    }
}
```

La diferencia con el `IdempotencyGuard` de consumo (que usa `REQUIRES_NEW`) es deliberada y vale la pena
entenderla, porque es el eje que separa los dos mecanismos:

| | Idempotencia de petición | Idempotencia de consumo |
|---|---|---|
| Propagación | `REQUIRED` — commitea **con** el recurso | `REQUIRES_NEW` — sobrevive al fallo del handler |
| Por qué | Una clave marcada sin recurso detrás haría que el reintento de una operación fallida devolviese una respuesta que nunca existió | El mensaje ya se consumió; el registro no puede morir con el rollback del negocio |

`find` filtra la caducidad en memoria, no en la consulta:

```java
.filter(stored -> stored.getExpiresAt().isAfter(now))
```

Es decir: **la ventana de deduplicación la fija `expires_at`, no la purga.** Una fila caducada pero aún no
barrida es, a todos los efectos, como si no estuviera.

### 2.6 La carrera: `409 IDEMPOTENCY_KEY_IN_PROGRESS`

Dos peticiones simultáneas con la misma clave insertan la misma PK; la base de datos arbitra y quien pierde
**revierte entera**, así que de dos peticiones idénticas se ejecutó exactamente una.

El desenlace tiene código propio (`http-idempotency.js:94-122`) y no es cosmética: sin él, la petición que
pierde llegaría al `ApiExceptionHandler` como una violación de integridad cualquiera y el cliente recibiría
un `409` anónimo, **indistinguible del conflicto de negocio que la misma operación puede devolver**. Un
escenario de validación no puede afirmar nada sobre eso.

```java
super("Otra petición con la misma clave de idempotencia está en curso: " + scope + "/" + idempotencyKey,
        "IDEMPOTENCY_KEY_IN_PROGRESS",
        409,
        new Object[] {scope, idempotencyKey});
initCause(cause);
```

Y por qué la respuesta honesta es «vuelve a intentarlo» y no la respuesta original: **el registro guarda el
id del recurso, no el cuerpo.** Hasta que la ganadora no commitea, ese id no existe. Reproducir una
respuesta = releer el recurso, y no hay recurso todavía.

---

## 3. Idempotencia de consumo — `processed_event`

### 3.1 La tabla

`src/scaffold/idempotency.js:178-228`:

```java
@Entity
@Table(name = "processed_event", indexes = {
        @Index(name = "ix_processed_event_processed_at", columnList = "processed_at")
})
public class ProcessedEventJpa {

    @EmbeddedId
    private ProcessedEventId id;

    @Column(name = "processed_at", nullable = false)
    private Instant processedAt;

    @Embeddable
    public static class ProcessedEventId implements Serializable {

        /** Identifica al consumidor; convención: nombre simple de la clase del listener. */
        @Column(name = "handler_id", nullable = false, length = 128)
        private String handlerId;

        @Column(name = "event_id", nullable = false, length = 255)
        private String eventId;
```

| Columna | Función |
|---|---|
| `handler_id` (PK) | Qué consumidor lo procesó. Dos listeners del mismo servicio deduplican por separado |
| `event_id` (PK) | El `messageId` que declara el diseño, o `metadata.eventId` de la envoltura Keel |
| `processed_at` | Solo para la purga |

El `length = 255` de `event_id` lleva su propia justificación en el generador, y es un buen ejemplo de fallo
no obvio:

> «255 y no 64: el id lo elige quien publica, y no siempre es un uuid — un id compuesto o un
> `MessageDeduplicationId` de SQS FIFO se pasan de 64 con facilidad. Y quedarse corto aquí no da un error de
> validación: da una **violación de longitud al insertar**, es decir, un mensaje que se va a la DLQ por no
> caber en la tabla que existe para no procesarlo dos veces.»

De dónde sale la clave (`messaging.js:399-409`):

- Con `contract.messageId` declarado → ese header/campo de la fuente.
- Con `envelope: keel` → `metadata.eventId`, que estampó el emisor en su `raise` y viaja intacto. **No se
  declara nada**: apuntar a un metadato nativo del broker sería apuntar a algo que ningún emisor Keel
  escribe.
- Con `none` o `wrapped` sin `messageId` → no hay clave, y por tanto no hay orden que prescribir.

### 3.2 `ProcessedEventWriter`: por qué es un bean aparte

`idempotency.js:324-374`. Las dos razones son la misma vista dos veces — **el proxy de Spring**:

```java
@Transactional(readOnly = true, propagation = Propagation.REQUIRES_NEW)
boolean exists(ProcessedEventJpa.ProcessedEventId key) {
    return processedEventRepository.existsById(key);
}

/** Inserta o lanza. La violación de clave es el resultado esperado de una carrera. */
@Transactional(propagation = Propagation.REQUIRES_NEW)
void insert(ProcessedEventJpa.ProcessedEventId key) {
    processedEventRepository.saveAndFlush(new ProcessedEventJpa(key, Instant.now()));
}
```

1. **Propagación.** Un método `@Transactional` invocado desde otro de la *misma clase* no pasa por el proxy,
   así que su `REQUIRES_NEW` no se aplica. Con el registro dentro del guard, `tryRecord()` llamando a
   `record()` heredaría la transacción del handler y el registro moriría con su rollback — justo lo que el
   javadoc promete que no pasa.
2. **Captura.** Una violación de clave deja la sesión de Hibernate inservible y marca la transacción
   *rollback-only*. Capturarla **dentro** de esa misma transacción no salva nada: el `return false` acaba en
   `UnexpectedRollbackException` al commitear. Aquí la excepción se **lanza**, y quien la captura está fuera.

Dos detalles de escritura que deciden el destino del mensaje:

- **`saveAndFlush` y no `save`** en JPA: `save` difiere el `INSERT` al commit, donde la violación ya no la ve
  este método.
- **`insert` y no `save`** en Mongo: `save` con un `_id` ya presente es un **reemplazo silencioso**, no un
  error — la carrera se resolvería sobrescribiendo y el duplicado se procesaría otra vez.

### 3.3 `IdempotencyGuard` y los dos órdenes

`idempotency.js:474-512`:

```java
public boolean alreadyProcessed(String handlerId, String eventId) {
    return writer.exists(new ProcessedEventJpa.ProcessedEventId(handlerId, eventId));
}

public boolean record(String handlerId, String eventId) {
    try {
        writer.insert(new ProcessedEventJpa.ProcessedEventId(handlerId, eventId));
        return true;
    } catch (DataIntegrityViolationException duplicate) {
        // Dos entregas del mismo mensaje procesándose a la vez, normalmente en dos
        // réplicas distintas: la clave primaria es el árbitro y esta pierde.
        log.debug("Idempotencia: {} ya registrado por {} (carrera resuelta en la clave)", eventId, handlerId);
        return false;
    }
}

/**
 * … Es la MISMA inserción que {@link #record}: lo que cambia es cuándo se llama,
 * no cómo se escribe. No hay consulta previa a propósito — preguntar antes de
 * insertar no cierra ninguna ventana que la clave no cierre ya …
 */
public boolean tryRecord(String handlerId, String eventId) {
    return record(handlerId, eventId);
}
```

**Los dos órdenes no son intercambiables**, y la elección no es del agente:

| | Orden A: procesar → registrar | Orden B: registrar → procesar |
|---|---|---|
| Llamadas | `alreadyProcessed(...)` antes, `record(...)` después de que el handler termine bien | `tryRecord(...)` antes de despachar |
| Cuándo | La operación declara `transitions` | La operación **no** declara `transitions` |
| Fallo transitorio | El mensaje queda sin marcar, el broker lo reentrega, se reintenta ✅ | El mensaje queda marcado y **perdido** ❌ |
| Ventana de duplicado | Abierta entre las dos llamadas — la cierra la transición del agregado | Cerrada ✅ |
| Es el predeterminado | Sí | Solo cuando reprocesar es inaceptable **y** perder es tolerable |

Quién decide: `sub.triggerHasDomainGuard` (`src/lib/model.js:1585`, que es
`(triggerOp?.transitions ?? []).length > 0`), y se escribe literalmente en el javadoc del `<Evento>Message`
que build genera (`messaging.js:412-416`):

> «Orden: `IdempotencyGuard.alreadyProcessed(...)` antes de despachar y `record(...)` **después** de que el
> handler termine bien. La operación declara transiciones, así que la repetición la frena el agregado y lo
> que no puede perderse es el mensaje.»

o, en el otro caso:

> «Orden: `IdempotencyGuard.tryRecord(...)` antes de despachar … Ojo: un fallo del handler deja el mensaje
> marcado y perdido; si eso no es tolerable, lo que falta es **la guarda de dominio en el diseño**.»

Que el gate estático verifica: cruzar los órdenes es un `forbid` explícito en `check-idempotency.sh` (§8).

---

## 4. Idempotencia saliente — el mecanismo sin tabla

Aquí no hay nada que guardar porque **quien deduplica es el proveedor**. Nosotros solo mandamos una clave
estable, y solo sirve si su contrato dice que la honra.

`src/scaffold/http-clients.js:332-367`:

```java
public final class OutboundIdempotency {

    /** Clave por CONTENIDO: dos peticiones idénticas son la misma intención. */
    public static String fromPayload(String call, Object payload) {
        return CommandSignature.of(Arrays.asList(call, payload));
    }

    /**
     * Clave por CORRELACIÓN: el proveedor deduplica por intención de negocio, no por
     * contenido — dos peticiones idénticas de dos ejecuciones distintas sí deben
     * ejecutarse las dos.
     */
    public static String correlated(String call, Object payload) {
        String correlationId = CorrelationContext.get();
        return correlationId == null
                ? fromPayload(call, payload)
                : CommandSignature.of(Arrays.asList(call, correlationId));
    }
}
```

Lo que la hace útil es que **el reintento produzca la misma clave**: resilience4j vuelve a invocar el método
entero, así que cualquier cosa aleatoria o dependiente del instante pediría una ejecución nueva en cada
intento — exactamente lo que se quiere evitar.

Y es lo único de los cinco mecanismos que `build` cablea **entero**: el agente no escribe nada. En el
adaptador RestClient (`http-clients.js:414-446`), el request se hoista a variable para que la firma sea la
del mismo objeto que se envía:

```js
// Con clave saliente el wire request se hoista a variable: la firma tiene que
// ser la del MISMO objeto que se envía, y calcularla dos veces (una para el
// body, otra para la cabecera) es la forma de que un día dejen de coincidir.
```

```java
Response response = restClient.post()
        .uri("/withdrawals")
        .header("Idempotency-Key", OutboundIdempotency.fromPayload("recordWithdrawal", request))
        .body(request)
        .retrieve()
        .body(Response.class);
```

Sin cuerpo (un `DELETE`), el payload pasa a ser `Arrays.asList(<params>)` — y `Arrays.asList` y no
`List.of` porque un parámetro opcional puede venir nulo y `List.of` lo rechazaría en tiempo de ejecución.

Una degradación que conviene conocer (`src/lib/model.js:1128-1138`): `keyFrom: correlation` sin capa `api`
ni `messaging` no tiene de dónde sacar la correlación, así que se degrada a `payload-hash` con un aviso.

---

## 5. Compensación y reconciliación — cero clases

Los dos únicos mecanismos que **no generan ninguna tabla ni ninguna clase**. Son lógica de negocio y salen
enteros de la mano del agente. Lo que `build` hace es no dejarlo a la intuición.

### 5.1 Compensación

Una compensación es deshacer trabajo ya encargado a otro servidor. Regla de propiedad: **quien encarga el
trabajo es quien lo deshace** — `compensations` cuelga de `dependencies.<proveedor>` en el diseño del que
llama, nunca en el del proveedor.

Mecánicamente **es una suscripción normal**: un evento llega, dispara una operación. Lo que cambia es lo
que esa operación tiene que hacer, y eso no se ve leyendo la operación sola.

`build` resuelve el enlace en `src/lib/model.js:1360-1387`, calculando `moves` — las entidades cuyo
lifecycle movió la activación que se deshace — y emite dos textos:

**(a)** en el javadoc del `<Evento>Message` (`messaging.js:430-440`):

> «Compensa la dependencia de *inventory* deshaciendo la activación *'reserveStock'*. Ese trabajo movió el
> lifecycle de *Reservation*: la operación que despacha este listener tiene que **devolver ese estado**, no
> solo avisar al proveedor.»

**(b)** en la nota del stub del handler (`services.js:349-365`), que además **elige la guarda**:

| Situación | Guarda que prescribe |
|---|---|
| La operación declara `transitions` | La del agregado: rechaza la segunda desde un estado que ya no está en `from`. **Va en el dominio**, no en el handler |
| No, pero hay `messageId` / `envelope: keel` | La deduplicación del listener (`IdempotencyGuard`). El handler no añade ninguna otra |
| Ninguna de las dos | `designGap` — se reporta, no se inventa |

Y una ventana que **no se cierra con código**: transición aplicada → llamada al proveedor hecha → commit que
falla. Eso es exactamente lo que una `compensations` declarada en el diseño existe para cubrir. Si el diseño
no la declara, es hueco de diseño y va al reporte, no un `try/catch` improvisado.

### 5.2 Reconciliación

Es el único mecanismo que **detecta lo que NO ha pasado**. Un encargo se hizo, el desenlace nunca llegó —ni
confirmación ni rechazo— y la entidad se queda esperando para siempre. No hay excepción, no hay log de
error, no hay alarma: una ausencia no produce ningún hecho al que reaccionar.

`build` genera el disparador (`services.js:493`): un `<Servicio>Scheduler` en `infrastructure.scheduling`
con `@Scheduled(cron = "0 <cron del DSL>")` — el `0` inicial es el campo de segundos que Spring exige y el
DSL no declara.

Y genera la nota más prescriptiva de todo el generador (`services.js:330-344`), que vale la pena resumir
porque es un catálogo de errores reales:

| Punto | La regla |
|---|---|
| **Desde cuándo** | El estado dice que espera, no **cuánto** lleva. Hace falta una marca temporal propia, nombrada a partir de la activación: **`<activacion>AwaitingSince`** (`reserveStockAwaitingSince`). **No** `createdAt` (es cuándo nació la entidad) ni un `updatedAt` de auditoría, que rejuvenece con cualquier escritura y deja la entidad **invisible al barrido para siempre** |
| **Por qué el prefijo** | Una entidad puede quedar esperando **dos desenlaces distintos** —dos activaciones, cada una con su `reconciledBy`—. Con un `awaitingSince` a secas el segundo encargo pisa la marca del primero: cada barrido ve candidatos del otro y su umbral mide una espera que no es la suya |
| **El índice** | El par (estado, marca) es un predicado que corre **cada N minutos** sobre una tabla de negocio, así que quiere su índice compuesto en `persistence: entities.<E>.indexes`, con la **igualdad primero y el rango después** — lo único que un B-tree aprovecha entero. Es la misma disciplina que §6.1 describe para las purgas. Con el índice solo sobre el estado, la base filtra por estado y evalúa la marca fila a fila: invisible mientras la espera esté poco poblada, un recorrido de tabla cuando acumula |
| **El umbral** | No lo declara el diseño: sale de `parameters/` con `@Value` y default explícito, nunca de una constante |
| **Concurrencia** | Corre en **todas** las réplicas: la consulta tiene que **reclamar** los candidatos, no solo leerlos. El patrón está en `OutboxRelay` |
| **La transición no basta** | Las réplicas leen antes de que ninguna confirme: todas pasan el guard y todas actúan. Lo único que absorbe las llamadas repetidas al proveedor es la idempotencia saliente |
| **Reencargar publicando** | No lo absorbe **nada**: cada réplica hace su propio `raise` con distinto `metadata.eventId`, así que para el consumidor son N hechos y su `processed_event` no los deduplica |
| **El orden** | Reclamar → actuar fuera → confirmar. Al revés (confirmar y luego actuar), morir en medio deja la entidad resuelta y el trabajo vivo en el proveedor: un huérfano que no detecta nadie |
| **La carrera feliz** | Mientras barres puede llegar el desenlace. Encontrar el candidato ya fuera del estado de espera es la carrera resuelta, **no** un fallo |

#### Qué hacer con lo que encuentra

La nota del stub dice «reintentar el encargo o disparar la compensación», y esa disyuntiva se queda corta:
son **tres** caminos, no dos, y elegir mal tiene consecuencias distintas en cada uno.

El punto de partida es que **el silencio es ambiguo**. No distingue dos situaciones que exigen respuestas
opuestas: que el encargo no llegara nunca, o que llegara, el proveedor lo hiciera y se perdiera la
respuesta. Reintentar y rendirse son dos **apuestas** sobre esa ambigüedad; solo el primer camino la elimina.

| Alternativa | Cuándo aplica | Qué resuelve |
|---|---|---|
| **1. Preguntar** al proveedor | Solo si expone consulta de estado | Deja de apostar: averigua qué pasó de verdad y aplica el desenlace que corresponda |
| **2. Rendirse con cancelación explícita** | El defecto sensato | Cierra los **dos** lados: si el proveedor no hizo nada la cancelación es un no-op; si lo hizo, se deshace |
| **3. Reintentar el encargo** | Solo con las tres condiciones de abajo | Recupera el camino feliz cuando el trabajo sigue siendo deseable |

**1. Preguntar.** La mejor cuando existe, y la que menos se considera. Si el proveedor tiene un
`GET /stock/reservations/{orderId}`, el barrido consulta y ya no decide a ciegas. El coste es que exige un
endpoint de consulta que muchos proveedores no ofrecen y que hay que declarar como `call` en `http-clients`
— por eso no es el caso común, no por ser peor.

**2. Rendirse con cancelación explícita.** Rendirse **no es** mover el estado propio: es moverlo *y
decírselo al proveedor*. Es lo que hace la fixture del repo, donde el barrido `reconcileReservations`
dispara la activación de vuelta `cancelStock`
(`test/fixtures/stock-reservation/dependencies.keel.yaml:24-36`), y lo que verifica la cláusula 2 de
`FL-REC-001` (`test/fixtures/stock-reservation/validation-scenarios.md`):

> «El proveedor recibió **exactamente un** `DELETE /stock/reservations/{o5}`. Rendirse no es solo mover el
> estado propio: es decírselo al almacén, que pudo bloquear el stock sin que su respuesta llegara nunca. Un
> barrido que solo cambia el estado deja stock bloqueado para siempre y ninguna otra cláusula lo vería.»

Es el defecto porque **es la única que sale bien en las dos ramas de la ambigüedad**, y porque le da al
cliente un desenlace —fallido, pero desenlace— en vez de dejarlo colgado.

**3. Reintentar.** Correcto solo si se cumplen las tres, no dos de tres:

- **La llamada es idempotente y está declarada** (`http-clients.calls.<x>.idempotency` → `OutboundIdempotency`,
  §4). Sin eso, la rama «sí llegó pero se perdió la respuesta» produce **dos reservas o dos cobros**. Es el
  error más caro que puede cometer un barrido, y el más difícil de atribuir después.
- **El trabajo sigue siendo deseable.** Han pasado el umbral y quizá horas: a veces el encargo ya no tiene
  sentido porque el caso de negocio cambió mientras tanto.
- **Hay un tope.** Sin contador, un encargo que nunca puede completarse se reintenta cada N minutos **para
  siempre**, contra un tercero, sin que nada lo delate. El tope convierte «reintentar» en «reintentar N
  veces y luego rendirse», que es la política completa.

**El antipatrón: restablecer el estado a secas.** Mover el lifecycle sin tocar al proveedor es la opción más
fácil de escribir y la peor de todas: tu sistema queda coherente **consigo mismo** y deja trabajo huérfano al
otro lado que no detecta nadie —ni tú, porque tu entidad ya salió del estado de espera, ni él, porque para él
el encargo era legítimo—. No produce ningún síntoma; solo un descuadre que aparece semanas después en un
inventario.

Y el DSL **obliga a elegir**, que es lo que impide dejarlo a medias: la operación del barrido tiene que
aparecer en el `triggeredBy` de alguna activación —la reconciliada (reintentar) o la de vuelta (compensar)—.
Un barrido que solo mueve el lifecycle sin aparecer en ningún `triggeredBy` es exactamente el antipatrón, y
el diseño lo deja a la vista.

---

## 6. Purga y retención

### 6.1 El principio: crons, no TTL

**No hay ningún índice TTL de Mongo en todo el servidor generado.** Las tres tablas se purgan con
`@Scheduled(cron = ...)` y borrado por rango, y cada una lleva su índice sobre la columna del barrido
precisamente para eso: sin él, cada réplica haría un recorrido completo de la tabla a la misma hora.

Prerrequisito silencioso: sin `@EnableScheduling` no corre **ni el relay ni ninguna purga**. Se añade en
`src/scaffold/application.js:38-41` cuando el diseño usa outbox, idempotencia (de cualquiera de los dos
tipos) o tiene operaciones con `schedule`.

### 6.2 Las tres purgas

| Tabla | Método | Cron por defecto | Retención por defecto | Fuente |
|---|---|---|---|---|
| `outbox_event` | `OutboxRelay.purge()` | `0 0 3 * * *` (03:00) | 7 días | `outbox.js:782-790` (JPA) / `:565-572` (Mongo) |
| `processed_event` | `IdempotencyGuard.purge()` | `0 0 4 * * *` (04:00) | 14 días | `idempotency.js:568-576` |
| `idempotency_record` | `JpaIdempotencyStore.purge()` / `MongoIdempotencyStore.purge()` | `0 30 4 * * *` (04:30) | **por fila**, vía `expires_at` | `http-idempotency.js:663-670` / `:914-921` |

```java
// outbox_event
@Scheduled(cron = "${outbox.purge.cron:0 0 3 * * *}")
@Transactional
public void purge() {
    Instant cutoff = Instant.now().minus(retentionDays, ChronoUnit.DAYS);
    int deleted = outboxRepository.deletePublishedBefore(cutoff);
    if (deleted > 0) {
        log.info("Outbox: purgadas {} filas publicadas antes de {}", deleted, cutoff);
    }
}
```

Consultas de borrado, por rama:

| Tabla | Relacional | Documental |
|---|---|---|
| `outbox_event` | `delete from OutboxEventJpa o where o.publishedAt is not null and o.publishedAt < :cutoff` | `deleteByPublishedAtNotNullAndPublishedAtBefore(cutoff)` |
| `processed_event` | `delete from ProcessedEventJpa p where p.processedAt < :cutoff` | `deleteByProcessedAtBefore(cutoff)` |
| `idempotency_record` | `delete from IdempotencyRecordJpa r where r.expiresAt < :now` | `deleteByExpiresAtBefore(now)` |

### 6.3 Cinco cosas que hay que saber de la purga

1. **La dead-letter nunca se purga.** `deletePublishedBefore` exige `published_at is not null`. Una fila que
   agotó los reintentos no está publicada, así que sobrevive a todos los barridos — a propósito: es
   evidencia, no basura. También significa que **crece sin límite si nadie la mira**.
2. **La retención de `processed_event` es el techo real de la deduplicación.** Solo tiene que cubrir la
   ventana en la que el broker puede reentregar. Una reentrega más tardía que los 14 días **sí se
   reprocesa**, porque la fila que la habría frenado ya no existe.
3. **`idempotency_record` no parametriza la retención**, solo la cadencia: cada fila lleva su propia
   caducidad calculada con el `ttlSeconds` que el diseño declara para esa operación. La purga es higiene,
   no semántica — lo que decide si una clave sigue viva es `expires_at`, y eso lo comprueba `find` en
   memoria.
4. **Las purgas son idempotentes por forma**, y por eso no necesitan reclamo aunque corran en las N
   réplicas a la vez: borrar lo caducado dos veces da el mismo resultado que borrarlo una. El javadoc del
   `<Servicio>Scheduler` lo dice explícitamente para que nadie extrapole esa tranquilidad a los barridos que
   escribe el agente.
5. **Todo sale de `parameters/<perfil>/`** (`src/scaffold/config.js:429-497`), nunca del código. Con un
   detalle que sorprende al leer el YAML generado: `OUTBOX_RELAY_BACKOFF_MAX_MS` vale **2000 en `local`** y
   60000 en el resto —

   > «El tope se acorta en `local` —el perfil con el que corre la suite de integración— porque ahí el broker
   > caído no es una avería sino un **paso** del escenario de outbox: el flujo lo detiene, comprueba que la
   > API responde igual y lo vuelve a levantar. Con el tope de producción, la entrega tras la recuperación
   > llega decenas de segundos después: el escenario tendría que esperar más de lo que ninguna suite tolera,
   > o saldría intermitente.»

---

## 7. Multi-instancia

### 7.1 La premisa

El servidor generado se despliega **replicado**. Dos hechos que gobiernan todo lo demás:

- **`@Scheduled` es «una vez por instancia», no «una vez en el clúster».** Con tres réplicas, el relay corre
  tres veces por segundo, no una.
- **Lo único que coordina réplicas es el almacén.** Ninguna guarda que viva en la JVM —un `synchronized`, un
  `AtomicBoolean`, un caché local— vale nada entre procesos.

Tabla de árbitros (`assets/generators/spring/conventions/concurrency.md:18-25`):

| Mecanismo | Árbitro | Qué hay que hacer |
|---|---|---|
| Reentrega del broker | PK `(handler_id, event_id)` / `_id` del documento | Llamar al guard en el orden del javadoc |
| Idempotencia de comando | PK `(operation_scope, idempotency_key)` | Consultar el store antes; guardar en la misma transacción |
| Entrega del outbox | Lock de fila (`SKIP LOCKED`) o `claimed_at` con caducidad | Nada: el relay es de `build` |
| Reintento contra proveedor | El proveedor | Nada: la cabecera ya va cableada |
| **Compensación** | **Ninguno propio**: prestado de la transición o del `processed_event` | Comprobar cuál toca |
| **Reconciliación** | **Ninguno**: el reclamo lo escribes tú | Reclamar, no leer |

Las dos últimas filas son las que hay que mirar dos veces.

### 7.2 El outbox con varias réplicas

**Rama relacional: lock pesimista con SKIP LOCKED** (`outbox.js:210-213`):

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "-2"))
@Query("select o from OutboxEventJpa o where o.publishedAt is null and o.attempts < :maxAttempts and (o.nextAttemptAt is null or o.nextAttemptAt <= :now) order by o.createdAt asc")
List<OutboxEventJpa> findPending(@Param("maxAttempts") int maxAttempts, @Param("now") Instant now, Pageable pageable);
```

El `lock.timeout = -2` es el valor con el que Hibernate pide **SKIP LOCKED**: en vez de esperar a que se
libere una fila bloqueada, la salta. Efecto: cada réplica se lleva un **lote disjunto** en vez de competir
por las mismas filas. El lock se sostiene hasta el commit de la transacción del relay — de ahí el
`@Transactional` sobre `relay()`.

Dos advertencias:

- **En H2 (perfil `test`) SKIP LOCKED puede degradarse a lock normal.** No afecta a la validación, que corre
  contra la base de datos real de `infra/`, pero explica por qué una prueba con H2 no demuestra nada sobre
  este punto.
- El lock se suelta solo cuando muere la conexión, así que una réplica que se caiga no retiene nada más allá
  de eso. Es la diferencia con la rama documental.

**Rama documental: `findAndModify` + `claimed_at` con caducidad** (`outbox.js:527-552`). En Mongo no hay lock
de fila, así que el reclamo se hace estampando una marca, de forma atómica por documento:

```java
private List<OutboxEventDocument> claimPending() {
    Instant now = Instant.now();
    Instant claimCutoff = now.minusMillis(claimTimeoutMs);
    Query query = new Query(new Criteria().andOperator(
                    Criteria.where("published_at").is(null),
                    Criteria.where("attempts").lt(maxAttempts),
                    new Criteria().orOperator(
                            Criteria.where("next_attempt_at").is(null),
                            Criteria.where("next_attempt_at").lte(now)),
                    new Criteria().orOperator(
                            Criteria.where("claimed_at").is(null),
                            Criteria.where("claimed_at").lte(claimCutoff))))
            .with(Sort.by(Sort.Direction.ASC, "created_at"));
    Update claim = new Update().set("claimed_at", now);
    FindAndModifyOptions options = FindAndModifyOptions.options().returnNew(true);

    List<OutboxEventDocument> claimed = new ArrayList<>();
    for (int i = 0; i < batchSize; i++) {
        OutboxEventDocument row = mongoTemplate.findAndModify(query, claim, options, OutboxEventDocument.class);
        if (row == null) {
            break;
        }
        claimed.add(row);
    }
    return claimed;
}
```

Dos diferencias de fondo con la rama relacional:

1. **El relay documental no es `@Transactional`, y es deliberado.** En JPA la transacción existe para
   sostener el lock hasta el commit; aquí el reclamo ya es atómico por documento y cada actualización
   también, así que abrir una transacción solo serviría para **mantenerla abierta durante la entrega al
   broker** — I/O externo dentro de una transacción de base de datos.
2. **`claimed_at` caduca, y ese parámetro es peligroso.** Un lock se suelta cuando muere la conexión; una
   marca en un documento no. Sin caducidad, una réplica que muriera entre el reclamo y la entrega retendría
   la fila para siempre. Con caducidad, el riesgo se invierte:

   > «Debe superar con holgura la latencia peor del broker; **por debajo, dos réplicas entregan lo mismo**.»
   > — `config.js:470-473`

   Por defecto son 60 s (`OUTBOX_RELAY_CLAIM_TIMEOUT_MS`). Es el único parámetro del outbox cuyo valor
   incorrecto produce duplicados *adicionales* a los que el modelo ya admite.

Tanto en éxito como en fallo, el relay llama a `row.releaseClaim()` antes de guardar: la marca se suelta en
cuanto el trabajo termina, no se espera a la caducidad.

### 7.3 Los guards de idempotencia con varias réplicas

Nada en la JVM coordina, y no hace falta. Las dos carreras y su desenlace:

| Carrera | Quién arbitra | Qué recibe el perdedor |
|---|---|---|
| Dos entregas del mismo mensaje, en dos réplicas | PK `(handler_id, event_id)` (o el `_id`) | `record()` devuelve **`false`** — no es un error, es la carrera resuelta |
| Dos peticiones con la misma `Idempotency-Key` | PK `(operation_scope, idempotency_key)` | `409 IDEMPOTENCY_KEY_IN_PROGRESS`, y su transacción **revierte entera** |

Y por eso `tryRecord` no tiene consulta previa: *preguntar antes de insertar no cierra ninguna ventana que la
clave no cierre ya*, y añade una consulta por mensaje.

### 7.4 Lo que no existe

**No hay ShedLock. No hay elección de líder. No hay particionado.** Es una decisión, no una omisión: la
doctrina del método es *reclamar, no bloquear*. Un lock distribuido serializa todo el barrido en una réplica
—desperdiciando las demás y creando un punto único de fallo— para resolver un problema que la propia
consulta puede resolver repartiendo lotes disjuntos.

### 7.5 Leer no es reclamar

«Reclamar, no leer» aparece en el método como consigna —en la nota del stub, en la tabla de árbitros de
§7.1, en `conventions/dependencies.md`— pero conviene desarmarla, porque quien la lee sabe que tiene que
reclamar y **no siempre reconoce si lo que escribió es un reclamo**.

Reclamar es **tomar posesión exclusiva de las filas en el mismo acto en que se seleccionan**. Las dos mitades
de esa frase —posesión y *en el mismo acto*— son el contenido entero.

**El contraste.** Esto no reclama:

```java
List<Order> stale = repo.findByStatusAndAwaitingSinceBefore(AWAITING_STOCK, cutoff);
```

Ocho réplicas ejecutan ese `SELECT` a la misma hora y las ocho reciben **los mismos 100 candidatos**. La
consulta no deja ninguna huella de que alguien se los llevó, así que las ocho actúan sobre ellos — y actuar,
aquí, es llamar a otro servidor.

**Y esto tampoco**, que es la trampa que hay que nombrar porque parece el arreglo natural:

```java
List<Order> stale = repo.findByStatus(...);      // ①
for (Order o : stale) { o.setStatus(SWEEPING); } // ②
```

Entre ① de la réplica A y ② de la réplica A cabe perfectamente ① de la réplica B. La ventana se **estrecha**,
no desaparece. Y una ventana estrecha es peor que una ancha: desaparece de las pruebas y solo se manifiesta
bajo carga, que es cuando hay tráfico real que duplicar.

**Las tres formas válidas** son exactamente las que acepta el gate, y las dos primeras ya están en este
documento (§7.2), escritas por `build` en el `OutboxRelay`:

| Forma | Cómo se apropia | Dónde verla |
|---|---|---|
| `SELECT … FOR UPDATE SKIP LOCKED` | Lock de fila; quien llega segundo **salta** las bloqueadas en vez de esperar | `outbox.js:210-213` |
| `findAndModify` con marca | Atómico por documento: lo selecciona **y** le estampa `claimed_at`, devolviendo el resultado | `outbox.js:527-552` |
| `UPDATE … RETURNING` / `@Modifying` | Marca y recupera en una sola sentencia, sin viaje intermedio | — |

Las tres tienen la misma propiedad: **no hay instante** en el que una fila esté seleccionada por A y todavía
disponible para B.

**Las dos mitades que el gate comprueba por separado** (`src/scaffold/idempotency-check.js:296-307`):

- **Exclusividad** (`claim`): el lock, la marca o el `RETURNING`.
- **Cota** (`bound`): `Pageable`, `PageRequest`, `LIMIT`, `Top100`. Sin ella el reclamo se lleva la tabla
  entera y deja de ser un lote — una réplica lo toma todo y las demás no trabajan, o esperan.

Ese check lleva además un `exclude` de las clases donde `build` ya puso el patrón (`OutboxRelay`,
`OutboxEventJpaRepository`, los repositorios de `processed_event` e `idempotency_record`). El motivo es
preciso: encontrarlas probaría **lo que hizo `build`**, no lo que el agente tenía que escribir.

**Cómo se suelta el reclamo** difiere por rama, y está desarrollado en §7.2: el lock lo libera la conexión al
morir, así que una réplica caída no retiene nada; la marca `claimed_at` no se entera de que el proceso murió
y necesita caducidad —con el riesgo invertido que allí se explica—.

**Por qué esto pesa más en un barrido que en un listado.** Un `SELECT` duplicado en una consulta de lectura
es ruido: devuelve las mismas filas dos veces y no pasa nada. En un barrido de reconciliación, el resultado
de la consulta se convierte en **llamadas a otro servidor o en eventos publicados**. De las tres cosas que un
barrido puede hacer con lo que encuentra, solo una tiene red aguas abajo (§7.6), y ninguna la tiene si el
problema es que ocho réplicas se llevaron los mismos candidatos.

Por eso el reclamo no es una optimización: es lo que hace que el barrido sea correcto con más de una
instancia. La transición del agregado **no lo sustituye** —las réplicas leen antes de que ninguna confirme,
así que todas pasan el guard—: el guard llega tarde, el reclamo llega a tiempo.

### 7.6 El punto peligroso: los `@Scheduled` que escribe el agente

Todo lo anterior lo genera `build`. Lo que el agente escriba **no hereda nada de eso**, y el javadoc del
`<Servicio>Scheduler` lo dice en el propio código (`services.js:519-528`):

```java
/**
 * Disparadores por reloj de las operaciones que declaran `schedule`.
 *
 * <p><strong>Cada método corre en TODAS las réplicas del servicio</strong>: `@Scheduled` es
 * "una vez por instancia", no "una vez en el clúster". El handler de una operación que
 * ACTÚA sobre lo que encuentra (un barrido de reconciliación) tiene que reclamar sus
 * candidatos en vez de solo leerlos — el patrón está en {@code OutboxRelay} y en
 * docs/keel/conventions/dependencies.md. Las purgas generadas no lo necesitan: borrar lo
 * caducado es idempotente por forma.
 */
```

Qué frena cada repetición cuando N réplicas barren a la vez:

| Lo que hace el barrido | Qué lo frena |
|---|---|
| Mover el estado del agregado | Nada útil: las réplicas leen antes de que ninguna confirme, así que **todas** pasan el guard. Es una carrera, no una serialización |
| Llamar al proveedor por HTTP | **Solo** `OutboundIdempotency`, si el proveedor la honra |
| Reencargar publicando un evento | **Nada.** Cada réplica hace su `raise` con distinto `metadata.eventId`: para el consumidor son N hechos distintos y su `processed_event` no los deduplica |

Por eso el reclamo no es opcional, y por eso es lo único que `check-idempotency.sh` comprueba **una vez por
diseño** en vez de por operación.

### 7.7 Lo que ningún gate cubre

Con honestidad, y así lo admite `conventions/concurrency.md:71-90`: **la carrera relay-contra-relay entre
réplicas no la ejercita ningún test.** La suite de integración corre contra una sola instancia. El único
gate sobre este tramo es **estructural** (§8): comprueba que el patrón de reclamo esté escrito, no que se
comporte bien bajo carga concurrente real.

---

## 8. El gate: `infra/check-idempotency.sh`

### 8.1 Por qué existe

`build` genera los mecanismos; el agente escribe el uso. **Ese tramo falla en silencio**: un listener sin
guard, un handler que ignora el store, un `@Scheduled` que sigue lanzando — todo compila, todo arranca, y el
camino feliz pasa en verde. Solo se nota en la primera repetición, que es justo cuando algo ya iba mal.

Y hay dos familias que **ningún escenario de integración puede cubrir**:

- **Reconciliación**: el arnés es caja negra y un cron no se alcanza desde fuera. Este script es su *único*
  gate.
- **Entrega del outbox**: sí tiene un escenario detrás (el de canal indisponible), pero llega tarde — corre
  con la aplicación arrancada, y si el dispatcher es el fallback, la suite entera ya se ejecutó contra un
  servidor que descartaba eventos en silencio.

La matriz la precomputa `build` desde el diseño (`src/scaffold/idempotency-check.js`), porque build sabe qué
listener toca qué orden y qué operación barre qué activación. El script solo la contrasta contra el árbol
final. Lo ejecuta el agente de calidad.

### 8.2 Las cinco familias

| Familia | Sujeto | Exige | Prohíbe |
|---|---|---|---|
| **`dedupe`** | Cada `<Evento>Listener` | `IdempotencyGuard`; que el retorno de `alreadyProcessed`/`tryRecord` **gobierne una rama**; y el método del orden correcto | El método del orden **cruzado**; `UUID.randomUUID()` |
| **`commandIdempotency`** | Cada handler con `idempotency` | `IdempotencyStore`, `CommandSignature.of(` | `.hashCode()`; con `payload-hash`, también `IdempotencyContext` |
| **`compensation`** | El handler que compensa | El `<C>Client` de la activación de vuelta, si el diseño la declara | `TODO` \| `UnsupportedOperationException` |
| **`reconciliation`** | El handler del barrido + el `<Servicio>Scheduler` + **un check global de reclamo** | `@Value` (umbral); `@Scheduled`; y un patrón de reclamo **acotado** | `TODO` \| `UnsupportedOperationException` |
| **`outboxDelivery`** | `OutboxDispatcher` | Que exista un implementador **distinto del fallback** | — |

Detalles que explican bien el diseño del gate:

- El `require` de `dedupe` no se conforma con que el guard aparezca. Exige la forma
  `(if|return|while|&&|\|\||!)[^;]*\.?(alreadyProcessed|tryRecord)\s*\(` — porque *referenciar el guard sin
  mirar su respuesta no deduplica nada*.
- `UUID.randomUUID()` está prohibido en un listener porque una clave inventada compila, pasa el camino feliz
  y deduplica cero.
- `.hashCode()` está prohibido en un handler idempotente porque **ni siquiera es estable entre arranques**, y
  la firma se compara contra otra guardada en otro despliegue.
- El check de **reclamo del barrido** exige que el patrón de reclamo (`PESSIMISTIC_WRITE|SKIP LOCKED|
  findAndModify|@Modifying`) y el de cota (`Pageable|PageRequest|limit|firstN|topN`) estén en el **mismo
  archivo** —un reclamo aquí y un `Pageable` en otro listado cualquiera no es un lote— y **excluye** las
  clases que build ya genera con ese patrón (`OutboxRelay`, los repositorios de outbox/processed/
  idempotency): encontrarlas probaría lo que build hizo, no lo que el agente tenía que escribir.

### 8.3 Los tres tipos de comprobación

| Tipo | Qué hace |
|---|---|
| `unit` | Localiza el archivo **por nombre** (`find -name "$1.java"`, no por ruta: dónde lo ponga el agente es suyo) y aplica los `require`/`forbid` |
| `impl` | Busca un implementador de un puerto **distinto de la clase excluida** (el fallback) |
| `claim` | Busca en todo el árbol un archivo donde coincidan reclamo **y** cota |

Y el detalle que los tres comparten, que es el más fino del script:

```bash
code="$(sed -e 's://.*::' -e '/^[[:space:]]*\*/d' -e '/^[[:space:]]*\/\*/d' "$file")"
```

**Se borran los comentarios antes de mirar.** Sin eso, las propias notas que `build` deja en los stubs
—que nombran los patrones que faltan para explicarlos— harían salir el gate verde **por su propia prosa**.
Es también la razón por la que el script debe salir **rojo** recién generado: un gate que siempre sale verde
no distingue «correcto» de «no mira».

Salida: una línea `OK|KO` por familia, `exit 1` si hay hallazgos, y el detalle de qué falta con el porqué.

---

## 9. Resumen operativo

**Lo que se crea en la base de datos:**

| Tabla / colección | Existe si | Filas que crecen | Se purga | Índice de la purga |
|---|---|---|---|---|
| `outbox_event` | `reliability: outbox` | Una por evento publicado | Publicadas > 7 días | `ix_outbox_event_pending` (compartido con el relay) |
| `processed_event` | Hay `subscriptions` | Una por mensaje consumido | Procesadas > 14 días | `ix_processed_event_processed_at` |
| `idempotency_record` | Alguna operación declara `idempotency` | Una por clave de petición | Caducadas (`expires_at`) | `ix_idempotency_record_expires_at` |

Más `flyway_schema_history` en la rama relacional, que la crea Flyway.

**Lo que hay que vigilar en producción:**

1. **Filas de `outbox_event` con `attempts >= max-attempts`** — dead-letter. No se purgan y no se
   reintentan: si nadie las mira, crecen y los eventos que representan nunca llegaron.
2. **Filas pendientes con `created_at` antiguo** — el relay no está corriendo, o el dispatcher no confirma.
3. **`claim-timeout-ms` en la rama documental** — por debajo de la latencia peor del broker, se producen
   duplicados evitables.
4. **La retención de `processed_event`** — tiene que cubrir la ventana de reentrega del broker con holgura.
5. **`@EnableScheduling`** — sin él no corre nada de todo esto, y no lo dice ningún error.
6. **El estado de espera de una reconciliación** — dos cosas a la vez: que no **acumule** (si crece sin
   techo, el desenlace no está llegando por la vía normal y el barrido solo tapa el síntoma) y que su
   consulta tenga **índice compuesto** `(estado, <activacion>AwaitingSince)`. Sin él, el barrido recorre una
   tabla de negocio cada N minutos compitiendo con el tráfico real — y en pruebas, con diez filas, no se ve.

**El orden mental que evita la mayoría de los errores:** el outbox garantiza que el evento *sale*; el
`processed_event` garantiza que llega *una vez*; el `idempotency_record` garantiza que el cliente no crea
*dos recursos*; la `OutboundIdempotency` garantiza que *nuestro* reintento no cobra dos veces. La
compensación deshace lo que ya se hizo, y la reconciliación es lo único que detecta lo que **no** pasó.

---

## Fuentes

Todo lo citado está en el repo `keel-system`. Rutas relativas a `packages/keel-spring/` salvo las marcadas.

**Generadores (el código que produce el Java):**
- `src/scaffold/outbox.js` — tabla, repositorio, relay, puerto y fallback del outbox (ambas ramas)
- `src/scaffold/idempotency.js` — `processed_event`, `ProcessedEventWriter`, `IdempotencyGuard`
- `src/scaffold/http-idempotency.js` — `idempotency_record`, `IdempotencyStore`, `CommandSignature`, filtro y contexto
- `src/scaffold/http-clients.js` — `OutboundIdempotency` y su cableado en el adaptador RestClient
- `src/scaffold/messaging.js` — `DomainEventBridge`, `EventEnvelope`, javadoc de los `<Evento>Message`
- `src/scaffold/services.js` — notas de los stubs (idempotencia, compensación, reconciliación) y `<Servicio>Scheduler`
- `src/scaffold/document-indexes.js` — índices de infraestructura en la rama documental
- `src/scaffold/config.js` — fragmentos `parameters/<perfil>/` de purgas y relay
- `src/scaffold/application.js` — `@EnableScheduling`
- `src/scaffold/migrations.js` — `db/migration/` vacío y `export-schema.sh`
- `src/scaffold/idempotency-check.js` — la matriz y el script del gate

**Modelo:**
- `src/lib/model.js` — resolución de `compensations` (`:1355-1390`), `reconciledBy` (`:1305-1350`), `triggerHasDomainGuard` (`:1585`), degradación de `keyFrom: correlation` (`:1128-1138`)
- `src/lib/supported-features.js` — declaración explícita de lo que el generador **no** cubre

**Convenciones (viajan al proyecto generado en `docs/keel/conventions/`):**
- `assets/generators/spring/conventions/concurrency.md` — tabla de árbitros, `@Scheduled` replicado, lo que ningún gate cubre
- `assets/generators/spring/conventions/dependencies.md` — el barrido en todas las réplicas, orden reclamar/actuar/confirmar
- `assets/generators/spring/conventions/integration-tests.md` — escenario del canal indisponible

**Diseño (en `packages/keel-core/`):**
- `assets/core/docs/dsl-reference.md` — la tabla de los ejes de repetición
- `assets/core/docs/dsl/messaging.md`, `dsl/dependencies.md`, `dsl/http-clients.md`, `dsl/use-cases.md`
- `assets/core/schema/{messaging,use-cases,dependencies,common}.schema.json`

**Fixtures con el ciclo completo:**
- `test/fixtures/stock-reservation/` — outbox + compensación + reconciliación + los tres tipos de idempotencia
- `test/fixtures/catalog-extended/` — compensación por llamada HTTP síncrona
- `test/fixtures/metering-digest/` — contrato ajeno (`envelope: wrapped` + `messageId`)

**Nota hermana:** [`idempotencia-y-compensaciones.md`](./idempotencia-y-compensaciones.md), para el plano del
DSL: schemas completos, YAML de ejemplo, reglas de `keel validate` y el árbol de decisión de qué mecanismo
declarar.
