# Observabilidad: guía de uso y operación

Esta guía es para **personas**: quien levanta el servicio, mira si funciona, investiga por qué algo tardó o falló, y decide a qué backend van las señales. Las reglas para quien escribe código están en otros dos documentos y no se solapan con esta:

- `conventions/observability.md` — qué instrumenta build y qué no se debe tocar.
- `conventions/logging.md` — qué loguea build, qué puede añadir el agente y qué no se loguea nunca.

Todo lo de aquí aplica a un servicio generado **con telemetría activada** (`telemetry: otel` en `keel-stack.json`). Si tu proyecto no la tiene, la primera sección explica cómo activarla.

---

## 1. Activarla (y qué aparece cuando lo haces)

La telemetría es una elección de **stack**, no del diseño: el mismo diseño se genera con ella o sin ella. Por defecto **no** se activa.

```bash
# Proyecto nuevo: el cuestionario de build pregunta por ella…
keel-spring build specs/<servicio>

# …o se elige directamente:
keel-spring build specs/<servicio> --telemetry otel

# Proyecto que ya existe: se activa y se propaga sin pisar el código del agente
keel-spring build specs/<servicio> --telemetry otel --refresh
```

Para quitarla: `--telemetry none --refresh`.

Con `--refresh`, los archivos nuevos aparecen y los que generó build se ponen al día; lo que haya tocado el agente se reporta como conflicto y no se pisa. Si el servicio ya está desplegado, recuerda que activar la telemetría **añade dependencias**: hay que reconstruir la imagen.

Lo que aparece en el proyecto:

| Pieza | Dónde | Qué hace |
|---|---|---|
| Dependencias | `build.gradle` | Puente de Micrometer Tracing a OpenTelemetry, exportadores OTLP y —en el modelo relacional— la instrumentación de JDBC |
| Configuración | `src/main/resources/parameters/<perfil>/telemetry.yaml` | A dónde se exporta, cuánto se muestrea y qué atributos identifican al servicio |
| Logs | `parameters/<perfil>/logging.yaml` | `traceId`/`spanId` en cada línea y JSON (ECS) fuera de `local` |
| Código | `infrastructure/telemetry/` | Lo que la autoconfiguración no hace sola: el appender de logs, el filtro anti-ruido, la traza a través de los mensajes y la instrumentación de los buckets, del correo y de la caché |
| **Panel y alertas** | `deploy/observability/` | Un panel de Grafana derivado de tu diseño y tres alertas, ya provisionados en el backend de prueba |
| Infraestructura de prueba | `deploy/otel/collector.yaml` + dos contenedores en `deploy/docker-compose.yaml` | El colector y un backend para mirarlo todo en local |
| Referencia de producción | `deploy/otel/collector-agent.example.yaml` y `collector-gateway.example.yaml` | Plantillas para desplegar de verdad; build no las despliega |

Y lo que se instrumenta **solo si tu diseño lo trae**: la caché (comandos de Redis), el almacenamiento (operaciones sobre los buckets) y el correo. Los tres se apagan por variable de entorno sin recompilar — § 6.

---

## 2. El modelo mental: tres señales y un colector en medio

Son tres cosas distintas y cada una responde a una pregunta:

| Señal | Pregunta que responde | Ejemplo |
|---|---|---|
| **Trazas** | ¿Por dónde pasó ESTA petición y dónde se fue el tiempo? | La confirmación tardó 900 ms: 850 fueron una consulta |
| **Métricas** | ¿Cómo se comporta el servicio en conjunto? | El p95 de `confirmReservation` subió esta semana |
| **Logs** | ¿Qué decidió el código en ese instante? | «Mensaje duplicado descartado por StockReservedListener» |

El servicio **no habla con el backend**. Habla con un **colector OpenTelemetry**, y lo único que sabe de él es su dirección. Con un matiz que conviene tener claro desde el principio: las trazas y los logs **salen** hacia el colector, pero las métricas **las viene a buscar él** a `/actuator/prometheus`:

```
   ┌─────────────────────┐   OTLP/HTTP    ┌───────────────┐   lo que tú decidas
   │   tu servicio       │ ─────────────▶ │   colector    │ ─────────────────▶  Tempo / Jaeger
   │                     │  trazas, logs  │               │                     Prometheus / Mimir
   │  /actuator/…    ◀───┼── lo scrapea ──│  recibe       │                     Loki / Elastic
   │                     │  cada 15 s     │  procesa      │                     un SaaS…
   │  consola (JSON) ────┼──▶ la recoge   │  reexporta    │
   └─────────────────────┘   la plataforma└───────────────┘
```

Esto es lo que hace que **cambiar de backend no toque el servicio**: se edita la configuración del colector y se reinicia el colector. Ni se recompila, ni se redespliega, ni se reinicia la aplicación.

**¿Por qué las métricas se scrapean y no se empujan?** Por los **exemplars**: el enlace que lleva de un punto de una métrica («el p95 subió») a una traza de ejemplo («esta petición concreta tardó eso»). Viajan pegados a los cubos del histograma en el formato de exposición de Prometheus, y el exportador OTLP de métricas de esta versión de Spring Boot no sabe emitirlos. Si tu plataforma no puede scrapear, `METRICS_EXPORT_OTLP=true` vuelve al push y lo único que se pierde son los exemplars — encender los dos a la vez duplica cada serie.

Los **logs tienen dos caminos** para llegar al backend, y conviene entenderlo desde el principio:

- **Camino A, la consola (el principal, siempre activo).** El servicio escribe cada línea en su salida estándar y **otro programa** la lee de ahí y la envía al backend: en Kubernetes, el agente de logs del clúster o el propio colector leyendo el fichero del contenedor; en AWS, CloudWatch; en otras plataformas, Fluent Bit o similares. El servicio no sabe adónde va el log. En `develop` y `production` cada línea es **JSON en formato ECS** (Elastic Common Schema: una convención abierta de nombres de campo, como `@timestamp`, `log.level` o `trace.id`, que no depende de ningún proveedor), para que quien la recoja pueda separar los campos sin expresiones regulares.
- **Camino B, OTLP (opcional, apagado por defecto).** El propio servicio envía cada línea **directamente al colector** por la red, igual que las trazas. Se enciende con `LOG_EXPORT_OTLP=true`.

**No enciendas los dos a la vez si la plataforma ya recoge la consola**: cada línea llegaría **dos veces** al backend, con el doble de almacenamiento y búsquedas confusas. Por eso el camino B viene apagado: lo normal en producción es que la plataforma recoja la consola.

**`deploy/` sí lo enciende**, a propósito: en el entorno de pruebas con compose **nadie lee la consola** de los contenedores, así que sin el camino B no llegaría ningún log a Loki y Grafana saldría vacío.

| Entorno | Camino A (consola) | Camino B (OTLP) | Por qué |
|---|---|---|---|
| `production`, `develop` | activo | apagado | la plataforma recoge la consola; con los dos, cada log saldría duplicado |
| `deploy/` | activo, pero nadie lo lee | **encendido** | es la única forma de que los logs lleguen a Loki |

> **No añadas un `logback-spring.xml` al proyecto.** Es la forma habitual de configurar los logs en Spring, pero aquí rompe el camino A **sin que nada falle**: cuando ese archivo existe, Spring Boot deja de configurar la consola por su cuenta, y el formato JSON ECS lo configura justo Boot. Los logs volverían a texto plano, el servicio arrancaría con normalidad y la plataforma recibiría texto donde espera JSON: no podría separar los campos y se perdería el enlace de cada log con su traza. Nadie se enteraría hasta buscar un log. Por eso el camino B no se configura con XML: su *appender* (el componente de la librería de logs que envía cada línea a un destino) lo añade **desde código** la clase generada `TelemetryConfig` al arrancar, sin tocar la consola, y solo si `LOG_EXPORT_OTLP` está encendido. Con el interruptor apagado, ni siquiera existe.

---

## 3. Levantarlo y mirarlo: el primer recorrido

```bash
bash deploy/up.sh
```

Levanta el servicio en contenedor con su infraestructura, el colector y un backend completo para mirar las tres señales (Grafana + Tempo + Loki + Prometheus, todo en un contenedor). Al terminar imprime las URLs; las que importan aquí:

| URL | Qué es |
|---|---|
| `http://localhost:3001` | **Grafana**: aquí se miran trazas, logs y métricas |
| `http://localhost:8080` | La API del servicio |
| `http://localhost:8080/actuator/health` | Estado del servicio |
| `http://localhost:4318` | El colector (OTLP/HTTP). No tiene interfaz: es a donde exporta la app |

Si el stack del servicio trae broker, caché, almacenamiento, correo o proveedor de identidad, `up.sh` imprime también sus consolas (Kafka UI, RabbitMQ, RedisInsight, MinIO, Mailpit, Keycloak). Todos los puertos salen de `deploy/.env` (`GRAFANA_PORT`, `APP_PORT`, `OTEL_HTTP_PORT`…): si alguno está ocupado en tu máquina, cámbialo ahí y vuelve a levantar.

### El recorrido, paso a paso

**1. Provoca algo.** Llama a una operación del servicio con una correlación tuya, que es un hilo fácil de seguir:

```bash
curl -X POST http://localhost:8080/<ruta-de-tu-operación> \
  -H 'Content-Type: application/json' \
  -H 'X-Correlation-Id: mi-prueba-1' \
  -d '{ … }'
```

**2. Busca la traza.** Abre `http://localhost:3001` — **no pide usuario ni contraseña**: el backend de pruebas viene con acceso anónimo, porque es de usar y tirar. Ve a **Explore** (la brújula del menú lateral) → origen de datos **Tempo** → pestaña **Search**: filtra por `Service Name` = el nombre de tu servicio y pulsa *Run query*. Verás la lista de trazas recientes; la última es la tuya.

> Los tres orígenes de datos ya están configurados y se llaman **Tempo** (trazas), **Loki** (logs) y **Prometheus** (métricas). No hay que dar de alta nada.

Si prefieres escribirlo, la pestaña **TraceQL** acepta:

```
{ resource.service.name = "<nombre-del-servicio>" }
```

y para encontrar exactamente la tuya, por la correlación que enviaste:

```
{ span.keel.correlation_id = "mi-prueba-1" }
```

**3. Léela.** Al abrir una traza se ve la cascada de spans: arriba la petición HTTP y, debajo y anidado, todo lo que provocó. Cada barra es tiempo real. Ahí se ve de un vistazo si el tiempo se fue en la base de datos, en una llamada saliente o en el propio código.

**4. Salta de la traza a los logs.** Los dos saltos vienen ya cableados en Grafana: desde un span hay un enlace a sus logs, y cada línea de log enseña un botón *Trace: …* que abre la traza. Si prefieres escribir la consulta: Explore → origen **Loki** →

```
{service_name="<nombre-del-servicio>"} | trace_id="<el trace id>"
```

o, si lo que tienes a mano es la correlación que enviaste:

```
{service_name="<nombre-del-servicio>"} | correlationId="mi-prueba-1"
```

> Ojo con un error fácil: **no** hace falta `| json`. Los logs llegan por OTLP con los campos ya separados (`trace_id`, `span_id`, `correlationId`, `severity_text`…), no como texto JSON que haya que parsear; añadir `| json` da un error de parseo.

**5. Mira las métricas.** Explore → origen **Prometheus**. Escribe `keel_` y el autocompletado enseña las métricas propias del servicio. El percentil 95 por operación, por ejemplo:

```
histogram_quantile(0.95, sum by (keel_operation) (rate(keel_use_case_milliseconds[10m])))
```

Dos detalles que ahorran un rato de desconcierto:

- **Los nombres cambian de puntuación**: lo que OpenTelemetry llama `keel.use-case` aquí es `keel_use_case_milliseconds`.
- **Son histogramas nativos**, así que no verás series `..._bucket` ni hace falta agrupar por `le`. Si tu backend no soporta histogramas nativos, la misma consulta se escribe en la forma clásica `histogram_quantile(0.95, sum by (le, …) (rate(..._bucket[10m])))`.
- Con poco tráfico el resultado es `NaN`: `rate` necesita varias muestras en la ventana. Lanza unas cuantas peticiones o amplía la ventana.

Cuando termines: `bash deploy/down.sh` (con `-v` borra también los datos).

---

## 4. Qué vas a ver exactamente

**Spans** de una petición que acaba publicando un evento y consumiéndolo:

| Span | De dónde sale |
|---|---|
| `http post /api/v1/…` | La petición entrante (es la raíz de la traza) |
| `<operación>-command` / `<operación>-query` | El caso de uso; lleva `keel.operation` y el `keel.correlation_id` |
| `connection`, `query`, `result-set` | Cada acceso a la base de datos relacional |
| `<colección>.find`, `<colección>.findAndModify` | Lo mismo en el modelo documental (MongoDB) |
| `keel.outbox.publish <Evento>` | La publicación del evento desde el outbox, **dentro de la traza que lo originó** aunque ocurra mucho después |
| `<destino> send` | El envío al broker |
| `keel.message.consume <Evento>` | El consumo de un mensaje, reenganchado a la traza del emisor |
| `keel.storage <operación>` | Cada operación sobre un bucket, con su nombre lógico del diseño |
| `keel.mail.send` | Cada entrega de correo al proveedor |
| `spring.data.redis` | Cada comando de Redis que ejecuta la caché |
| `<cliente> …` | Cada llamada HTTP saliente |

**Métricas** propias (además de las de Spring Boot y la JVM):

| Métrica | Para qué sirve |
|---|---|
| `keel.use-case` | Duración y número de ejecuciones por operación (con histograma: admite p95/p99) |
| `keel.message.consume` | Lo mismo para los mensajes consumidos |
| `keel.outbox.publish` | Lo mismo para los eventos publicados por el outbox |
| `keel.outbox.dead_lettered` | **Eventos que agotaron sus reintentos y no salieron.** Debería ser siempre 0 |
| `keel.storage` | Duración y desenlace por operación de bucket (`upload`, `download`, `delete`…) y por bucket |
| `keel.mail.send` | Lo mismo para cada entrega de correo al proveedor |
| `cache.gets` | Aciertos y fallos por caché, que es lo único que dice si la caché sirve para algo |
| `http.server.requests` | Latencia y códigos de respuesta por endpoint |

**Una línea de log** en `develop`/`production` (JSON ECS, recortada):

```json
{"@timestamp":"2026-09-19T20:23:35.60Z","log":{"level":"DEBUG","logger":"…UseCaseMediator"},
 "message":"Caso de uso ConfirmReservationCommand: ok",
 "trace.id":"f6aef85d84e48f9f0d8f3e70ecb3d409","span.id":"00cc3b8d80deecc3",
 "correlationId":"mi-prueba-1",
 "keel":{"operation":"ConfirmReservationCommand","outcome":"ok","duration_ms":31}}
```

Los tres identificadores son los que permiten cruzar señales: `trace.id` lleva a la traza, `correlationId` es el que recibió el cliente (viaja en la cabecera `X-Correlation-Id` de la respuesta) y `keel.operation` es el nombre de la operación del diseño.

En `local` el log es texto plano para leerlo en la terminal, con los mismos identificadores entre corchetes.

Las etiquetas de cada métrica, y cómo llega al backend, están en § 4 bis.

---

## 4 bis. Cómo llega una métrica al backend, y con qué etiquetas

### El recorrido

Una métrica **no sale sola del servicio**: el servicio la acumula y es el colector quien viene a buscarla. Paso a paso, con la configuración de `develop` (la de `deploy/`):

1. **El código registra un valor.** Por ejemplo, el mediator mide cuánto tardó `ConfirmReservationCommand` y si acabó bien. Lo hace con una *observación* de Micrometer, que produce a la vez el span de la traza y la métrica.
2. **Micrometer lo acumula en memoria**, dentro del servicio: cuántas ejecuciones, cuánto sumaron, en qué franja de duración cayó cada una.
3. **El servicio lo publica en `/actuator/prometheus`** como texto en formato Prometheus. Ahí no se envía nada: es una página que muestra el estado acumulado en ese momento.
4. **El colector la lee cada 15 segundos** (*scrape*). Al leerla añade dos etiquetas propias del scrape, `job` (el nombre del servicio) e `instance` (la dirección de la que la leyó).
5. **El colector la enriquece** con `service.name` y `deployment.environment` (procesador `resource/scrape` de `deploy/otel/collector.yaml`), porque un scrape no los trae, y la agrupa en lotes.
6. **El colector la envía al backend** por OTLP (en `deploy/`, al Prometheus del contenedor `lgtm`).
7. **El panel y las alertas la consultan** en el backend, filtrando por sus etiquetas.

Puedes ver lo mismo que ve el colector en el paso 4:

```bash
curl -s http://localhost:8080/actuator/prometheus | grep keel_use_case
```

Si la serie está aquí y no en el backend, el problema está en los pasos 4 a 6 (§ 9, «No llega nada»). **En `production` el paso 3 ocurre en otro puerto**: el actuator vive en un **puerto de gestión** aparte (`MANAGEMENT_PORT`, 8081 por defecto) que no se publica fuera del pod o del host, así que el colector lee `http://<servicio>:8081/actuator/prometheus` y por el puerto de negocio no sale ningún nombre interno (§ 8). Si al arrancar el servicio escribe el aviso «Las métricas no tienen por dónde salir», alguien desactivó los dos caminos por variables. Con el push por OTLP (`METRICS_EXPORT_OTLP=true`), los pasos 3 y 4 se sustituyen por un envío del servicio al colector cada `METRICS_EXPORT_STEP`, sin exemplars.

### Cómo se ve una métrica en el backend

Los nombres cambian por el camino: los puntos y guiones pasan a `_`, y a las duraciones se les añade la unidad (`_seconds`). Además, **una métrica de duración son varias series**:

| Serie | Qué es |
|---|---|
| `keel_use_case_seconds_count` | Cuántas ejecuciones hubo |
| `keel_use_case_seconds_sum` | Cuánto tiempo sumaron (con `_count`, da la media) |
| `keel_use_case_seconds_max` | La más lenta del último intervalo |
| `keel_use_case_seconds_bucket{le="…"}` | Cuántas duraron **menos de** `le` segundos: es el histograma del que salen el p95 y el p99, y donde viajan los exemplars |
| `keel_use_case_active_seconds_*` | Las ejecuciones **en curso** en este momento y cuánto llevan |

El histograma (`_bucket`) solo se publica para las métricas en las que interesa un percentil: `http.server.requests`, `http.client.requests`, `keel.use-case` y, si tu diseño los trae, `keel.storage` y `keel.mail.send` (`parameters/<perfil>/telemetry.yaml`, bloque `percentiles-histogram`). Las demás tienen `_count`, `_sum` y `_max`, pero no p95.

### Qué métricas se envían y con qué etiquetas

**Las propias del proyecto.** Las etiquetas son exactamente las que pone el código generado. Solo existen las de los subsistemas que trae tu diseño.

| Métrica (nombre en el backend) | Etiquetas | De dónde sale el valor de cada etiqueta |
|---|---|---|
| `keel_use_case_seconds_*` | `keel_operation`, `keel_outcome` | `keel_operation`: el nombre de la operación del diseño (p. ej. `ConfirmReservationCommand`). `keel_outcome`: `ok`, `rejected` (el dominio la rechazó, un 4xx esperado) o `error` (fallo) |
| `keel_outbox_publish_seconds_*` | `keel_event_type` | El nombre del evento del diseño |
| `keel_message_consume_seconds_*` | `keel_event_type` | Ídem, para los eventos consumidos |
| `keel_outbox_dead_lettered` | *(ninguna)* | Es un único número: los eventos que agotaron sus reintentos |
| `keel_storage_seconds_*` | `keel_storage_operation`, `keel_storage_bucket`, `keel_outcome` | La operación (`upload`, `download`, `delete`…), el nombre **lógico** del bucket en el diseño y `ok`/`error` |
| `keel_mail_send_seconds_*` | `keel_outcome` | `ok`/`error`. El destinatario **no** es etiqueta |

A todas las métricas de duración que salen de una observación, Micrometer les añade además la etiqueta `error`, con el nombre de la excepción o `none` si no la hubo.

**Las de Spring Boot y sus librerías.** Estas etiquetas no las decide el proyecto sino Spring Boot, así que conviene confirmarlas en tu versión con el `curl` de arriba:

| Métrica | Etiquetas principales |
|---|---|
| `http_server_requests_seconds_*` | `method`, `uri` (la **plantilla** de la ruta, `/products/{id}`, no la ruta con el id), `status`, `outcome` (`SUCCESS`, `CLIENT_ERROR`…), `exception` |
| `http_client_requests_seconds_*` | Las mismas, más `client_name` (a quién se llama) |
| `cache_gets_total` | `cache` (el nombre de la caché del diseño), `result` (`hit`/`miss`) |
| `jvm_memory_used_bytes`, `jvm_gc_pause_seconds_*`… | `area` (`heap`/`nonheap`), `id` (la zona de memoria), `gc`, `action`, `cause` |
| `hikaricp_connections_*` (relacional) | `pool` |
| `mongodb_driver_pool_*` (documental) | `cluster_id`, `server_address` |
| `kafka_consumer_fetch_manager_records_lag` (solo Kafka) | `client_id`, `topic`, `partition` |

**Las que se añaden por el camino**, en todas las series: `job` e `instance` (las pone el scrape), `service.name` y `deployment.environment` (las pone el colector). Cómo se muestran estas dos últimas depende del backend: unos las convierten en etiquetas de cada serie y otros las guardan aparte, como información del servicio.

**Lo que nunca es etiqueta**: el `correlationId`, los ids de negocio, emails, importes… Cada valor distinto de una etiqueta crea una serie nueva, así que un id como etiqueta multiplicaría las series por el número de ids. Esos datos van en el **span** de la traza, donde sirven para encontrar la petición concreta. Tampoco confundas las etiquetas con los campos de los logs: el log de frontera lleva `keel.error_code`, `keel.duration_ms` o `keel.event_id`, pero son campos del log y no etiquetas de ninguna métrica.

### ¿Se pueden añadir o quitar etiquetas?

Depende de qué se quiera cambiar y de quién lo haga.

**Desde el diseño: no hay dónde declarar etiquetas.** El DSL no tiene ningún campo de telemetría, y la telemetría es una elección de stack (§ 1). Lo que el diseño sí decide son los **valores**: al añadir una operación aparece un valor nuevo de `keel_operation`; un evento, de `keel_event_type`; un bucket, de `keel_storage_bucket`; una caché, de `cache`. Las **claves** (qué etiquetas existen) las fija el generador.

**Al desplegar, sin tocar código** (quien opera el servicio, con variables de entorno o en el colector):

| Qué quieres | Cómo | Ejemplo |
|---|---|---|
| **Añadir una etiqueta fija a todas las métricas** (equipo, región, *tenant* del despliegue…) | Propiedad `management.metrics.tags.<clave>` de Spring Boot, en el YAML del perfil o por variable de entorno (la clave queda en minúsculas; para una clave compuesta, mejor el YAML) | `MANAGEMENT_METRICS_TAGS_TEAM=pagos` → todas las series llevan `team="pagos"` |
| **Apagar una familia de métricas entera** | Propiedad `management.metrics.enable.<prefijo>=false` | `management.metrics.enable.jvm.gc=false` |
| **Apagar la instrumentación de caché, buckets o correo** | Los interruptores de § 6 | `TELEMETRY_INSTRUMENT_STORAGE=false` |
| **Quitar o renombrar una etiqueta concreta** | En el **colector**, con el procesador `transform` (viene en la imagen contrib que usa `deploy/`), añadido al pipeline de `metrics` | ver abajo |

```yaml
# deploy/otel/collector.yaml — quitar la etiqueta `exception` de todas las métricas
processors:
  transform/labels:
    metric_statements:
      - context: datapoint
        statements:
          - delete_key(attributes, "exception")
service:
  pipelines:
    metrics:
      processors: [memory_limiter, resource/scrape, resourcedetection, attributes/redact, transform/labels, batch]
```

> **Ojo al quitar etiquetas que usa el panel.** `keel_operation`, `keel_outcome`, `keel_event_type`, `le`, `result` o `area` aparecen en las consultas del panel y de las alertas. Si las quitas, **nada falla**: el panel se queda vacío y la alerta de tasa de error deja de poder dispararse. Quita solo lo que ninguna consulta de `deploy/observability/` nombre.

**Con código** (una clase nueva en el proyecto):

- **Quitar o renombrar una etiqueta dentro del servicio**, en vez de en el colector: un bean `MeterFilter` de Micrometer (`MeterFilter.ignoreTags(...)`, `MeterFilter.renameTag(...)`). Sirve para cualquier backend, sin depender de cómo esté configurado el colector.
- **Añadir una etiqueta nueva con un dato de negocio** (por ejemplo, el país del pedido en `keel_use_case`): **no está permitido en el proyecto generado**. `infra/check-telemetry.sh` solo acepta como claves de etiqueta las del vocabulario del generador (las de la primera tabla), y cualquier otra la marca como error. Es a propósito: es la forma más fácil de multiplicar las series sin que nada lo avise. Si hace falta una etiqueta nueva para **todos** los servicios, el cambio va en el generador (`keel-spring`), no en un proyecto. Para un dato de un solo servicio, lo correcto es ponerlo en el span con `addHighCardinalityKeyValue`, donde se puede buscar sin crear series.

---

## 4 ter. Cómo llega una traza al servidor

Aquí «servidor» es el **servidor de trazas** (el backend: Tempo en `deploy/`). Al contrario que las métricas, las trazas **sí salen del servicio**: el servicio las envía al colector, nadie viene a buscarlas.

### El recorrido

1. **Llega una petición y nace la traza.** Spring Boot abre el primer span, el de la petición HTTP. Si la petición trae la cabecera W3C `traceparent` (porque quien llama ya tenía una traza, por ejemplo un API gateway u otro servicio), **no nace una traza nueva**: el span se cuelga de la de quien llama. También se aceptan las cabeceras **B3** (`b3` o `X-B3-TraceId`…), que hablan muchas mallas de servicio y los sistemas tipo Zipkin; al llamar a otros, el servicio emite siempre W3C.
2. **Se decide si esta traza se guarda (muestreo).** Solo cuando la traza nace aquí; si viene de fuera, se respeta lo que decidió quien llama (muestreo *parent-based*). La proporción la fija `TRACING_SAMPLING_PROBABILITY`: 100 % en `local` y `develop`, 10 % en `production`. Una traza no muestreada se sigue ejecutando igual, pero no se envía.
3. **Cada trabajo dentro de la petición añade su span**, colgado del anterior: el caso de uso (con `keel.operation`, `keel.outcome` y el `keel.correlation_id`), cada consulta a la base de datos, cada llamada HTTP saliente, cada comando de Redis, cada operación sobre un bucket o envío de correo. Todos salen de *observaciones* (§ 4 bis).
4. **La traza sigue al salir del servicio.** En una llamada HTTP saliente, el cliente pone la cabecera `traceparent` y el servicio de destino continúa la misma traza. En un evento, el contexto viaja dentro del sobre del mensaje (§ 5).
5. **Se descarta el ruido.** Antes de enviar nada se descartan las peticiones al *actuator* (las sondas de salud), los ciclos de las tareas programadas y las consultas a la base de datos que no cuelgan de ningún trabajo (§ 9, «Ruido»).
6. **El servicio envía los spans terminados al colector.** No los envía uno a uno: los agrupa en memoria y los manda por lotes, comprimidos, por OTLP/HTTP a `${OTEL_EXPORTER_OTLP_ENDPOINT}/v1/traces`. Cada lote lleva los datos del servicio: `service.name`, `service.version` y `deployment.environment`. Si el colector no responde, el servicio **no falla**: reintenta, escribe un aviso en el log y, si la espera se alarga, descarta.
7. **El colector los procesa**: limita memoria, **borra las cabeceras de credenciales** que una instrumentación pudiera haber capturado, agrupa por lotes y reenvía al backend, con reintentos y cola si el backend está caído. El colector de `deploy/` borra `authorization`, `cookie`, `set-cookie` y `x-api-key`; la plantilla del gateway de producción, de momento, solo `authorization`: si tu servicio recibe credenciales en otras cabeceras, añádelas ahí.
8. **El backend reconstruye la traza** uniendo los spans por su `traceID`, aunque hayan llegado en momentos distintos, desde servicios distintos e incluso con horas de diferencia (la publicación desde el outbox).

En producción con Kubernetes, entre los pasos 6 y 8 hay dos colectores en lugar de uno: el agente del nodo recibe los spans y los reparte por `traceID` a un gateway, que decide qué trazas se guardan viéndolas enteras (muestreo por cola, § 8).

Para comprobar que llegan: en Grafana, **Explore → Tempo**, buscando por el nombre del servicio o por `{ span.keel.correlation_id = "…" }` (§ 3).

### ¿Qué se puede ajustar de una traza?

**Desde el diseño: la forma, no la configuración.** El DSL no tiene ningún campo de trazas, pero el diseño decide **qué spans existen y cómo se llaman**: cada operación da un span con su nombre, cada evento da un span de publicación y otro de consumo, cada bucket, cliente HTTP o correo declarado da los suyos. Y `reliability: outbox` decide que la publicación de un evento aparezca **más tarde pero dentro de la misma traza**.

**Al desplegar, sin tocar código** (variables de entorno o YAML del perfil):

| Qué quieres | Cómo | Ejemplo |
|---|---|---|
| **Guardar más o menos trazas** | `TRACING_SAMPLING_PROBABILITY` (de 0.0 a 1.0) | `TRACING_SAMPLING_PROBABILITY=1.0` para investigar un problema en producción; bájalo después |
| **No enviar ninguna traza** | `TELEMETRY_EXPORT_ENABLED=false` | Para silenciar un entorno sin quitar la telemetría |
| **Quitar los spans de un subsistema** (caché, buckets, correo) | Los interruptores de § 6 | `TELEMETRY_INSTRUMENT_CACHE=false` |
| **Quitar una familia de spans de Spring Boot o sus librerías** | Propiedad `management.observations.enable.<prefijo>=false`. Quita a la vez el span y la métrica de esa observación. Para prefijos con puntos, mejor en el YAML del perfil que por variable de entorno | `management.observations.enable.spring.security: false` quita los spans del filtro de seguridad |
| **Añadir un dato fijo a todas las trazas** (equipo, región…) | Atributos de recurso: `management.opentelemetry.resource-attributes.<clave>` en `parameters/<perfil>/telemetry.yaml`, o la variable estándar `OTEL_RESOURCE_ATTRIBUTES`, que no toca ningún archivo generado | `OTEL_RESOURCE_ATTRIBUTES=team=pagos,region=eu-west-1` |

Los atributos de recurso describen **al servicio**, no a cada span, y viajan con las trazas y los logs que salen por OTLP. A las métricas no llegan, porque esas las lee el colector (§ 4 bis); para ellas se usa `management.metrics.tags`.

**En el colector** (sin tocar el servicio, para todo lo que pase por él):

- **Decidir qué trazas se guardan en producción.** En el gateway (`collector-gateway.example.yaml`), el muestreo por cola guarda todas las que tienen error, todas las que tardan más de 1 s y un 10 % del resto. Se ajustan el umbral de lentitud, el porcentaje, o se añaden reglas; por ejemplo, guardar siempre las de una operación crítica:

  ```yaml
  # collector-gateway.example.yaml, dentro de tail_sampling.policies
  - name: operaciones-criticas
    type: string_attribute
    string_attribute:
      key: keel.operation
      values: [ConfirmReservationCommand, CancelReservationCommand]
  ```

  Recuerda que el gateway solo decide sobre lo que la aplicación dejó pasar: con muestreo por cola, pon `TRACING_SAMPLING_PROBABILITY=1.0` en la aplicación (§ 8).
- **Quitar, renombrar o anonimizar atributos** con los procesadores `attributes` o `transform`, igual que ya se hace con las cabeceras de credenciales.
- **Descartar spans concretos** con el procesador `filter`, por ejemplo los de una ruta que no interesa.

**Con código** (en el proyecto, respetando `conventions/observability.md`):

- **Añadir un dato de negocio al span del caso de uso** (el país de un pedido, un id): dentro del handler, sobre la observación en curso, **nunca abriendo un span nuevo**:

  ```java
  observationRegistry.getCurrentObservation()
          .highCardinalityKeyValue("keel.order.country", command.country());
  ```

  Al ser `highCardinality`, va **solo al span** y no crea series de métricas, así que no lo bloquea `check-telemetry.sh`. Nada de datos personales ni secretos: lo que entra en una traza se queda en el backend.
- **Añadir un atributo a todas las observaciones** del servicio: un bean `ObservationFilter`, que es como el proyecto pone el `keel.correlation_id` en cada span (`TelemetryConfig`).
- **Lo que no se debe hacer**: abrir spans a mano con el `Tracer`. Lo importante ya es una observación, y un span manual suele duplicar uno existente o quedar suelto, sin padre.

---

## 5. La traza a través de los eventos

Un evento rompe la traza dos veces si nadie hace nada: con `reliability: outbox` se publica **en otro hilo y más tarde**, y el consumidor es **otro proceso**.

Se conserva así:

1. **Al publicar**, la envoltura del evento lleva el contexto de traza W3C en `metadata.traceparent`. Es parte del contrato del evento, con telemetría o sin ella (sin ella viaja a `null`).
2. **El relay del outbox** lee ese contexto de la fila y publica dentro de la traza original, aunque hayan pasado horas.
3. **Al consumir**, el listener reabre el contexto. Con Kafka y RabbitMQ además viaja la cabecera nativa del broker; **con SNS/SQS no existe esa cabecera** y el `traceparent` del sobre es la única vía.

Para el punto 3 hace falta una línea en el listener, que es código del agente:

```java
CorrelationContext.runWith(envelope.metadata(), () -> { … });   // ✅ continúa la traza
CorrelationContext.runWith(envelope.metadata().correlationId(), () -> { … });  // ❌ la corta
```

Las dos compilan y las dos abren la correlación; solo la primera continúa la traza. Lo verifica `bash infra/check-idempotency.sh` (familia `inboundContext`).

**Cómo se ve que está bien:** una petición que publica un evento y lo consume produce **una sola traza**, con el span del consumo colgando del de la publicación. Si ves dos trazas separadas, el eslabón roto es ese.

---

## 6. Perfiles y variables

| Perfil | Cuándo se usa | Exporta | Muestreo | Endpoint |
|---|---|---|---|---|
| `local` | Desarrollo y suite de escenarios | **No** por defecto | 100 % | `http://localhost:4318` |
| `develop` | El contenedor de `deploy/` y entornos de prueba | Sí | 100 % | `OTEL_EXPORTER_OTLP_ENDPOINT` |
| `production` | Producción | Sí | 10 % de las raíces | `OTEL_EXPORTER_OTLP_ENDPOINT` **obligatoria** |
| `test` | Pruebas unitarias | No, y sin trazas | — | — |

`local` no exporta porque es el perfil con el que corre la suite de escenarios y ahí no hay colector: un exportador sin destino no rompe nada, pero llena la salida de avisos. Para mirar telemetría en local, levanta `deploy/` y arranca con `TELEMETRY_EXPORT_ENABLED=true`.

| Variable | Qué hace | Cuándo tocarla |
|---|---|---|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Dirección del colector | Siempre en producción; en `deploy/` ya viene puesta |
| `TELEMETRY_EXPORT_ENABLED` | Enciende o apaga la exportación de trazas | Para mirar telemetría en `local`, o para silenciar un entorno |
| `METRICS_EXPORT_PROMETHEUS` | Publica `/actuator/prometheus`, que es de donde el colector scrapea las métricas | Apagarlo solo si vuelves al push por OTLP |
| `METRICS_EXPORT_OTLP` | Vuelve al push de métricas por OTLP (pierde los exemplars) | Si tu plataforma no puede scrapear el endpoint |
| `MANAGEMENT_ENDPOINTS` | Qué endpoints del actuator se publican | Rara vez: con telemetría, `production` ya publica `health,info,prometheus` en el puerto de gestión |
| `MANAGEMENT_PORT` | Puerto del actuator en `production` (8081). Las sondas siguen también en el 8080 como `/livez` y `/readyz` | Si el 8081 choca con algo; **nunca** lo publiques fuera del pod o del host |
| `METRICS_OTLP_HISTOGRAM_FLAVOR` | Tipo de histograma del envío por OTLP (`base2_exponential_bucket_histogram`) | Si tu backend no acepta histogramas exponenciales: `explicit_bucket_histogram` |
| `SERVICE_INSTANCE_ID` | Identifica la réplica en trazas y logs (por defecto, `HOSTNAME`) | Donde `HOSTNAME` no sea el nombre de la instancia |
| `TELEMETRY_INSTRUMENT_STORAGE` | Instrumenta —o no— las operaciones sobre los buckets | Para quitar esa señal de un entorno sin recompilar |
| `TELEMETRY_INSTRUMENT_CACHE` | Instrumenta —o no— los comandos de Redis de la caché | Ídem |
| `TELEMETRY_INSTRUMENT_MAIL` | Instrumenta —o no— los envíos de correo | Ídem |
| `LOG_EXPORT_OTLP` | Manda **además** los logs por OTLP | Solo donde nada recoge la consola |
| `TRACING_SAMPLING_PROBABILITY` | Fracción de trazas que se conservan (1.0 = todas) | Subirlo para investigar; ponerlo a 1.0 si el colector hace muestreo por cola |
| `METRICS_EXPORT_STEP` | Cada cuánto se envían las métricas | Bajarlo para ver antes en una prueba |
| `LOG_FORMAT` | Formato de la consola fuera de `local` (`ecs` por defecto) | Si tu plataforma espera otro formato |
| `DEPLOYMENT_ENVIRONMENT` | Etiqueta el entorno en todas las señales | Para distinguir varios entornos en el mismo backend |

El **muestreo** es *parent-based*: si quien llama ya decidió que su traza se conserva, este servicio la respeta. Solo decide cuando la traza nace aquí.

El **`X-Correlation-Id`** que manda el cliente se acepta si tiene entre 1 y 64 caracteres de `A-Z`, `a-z`, `0-9`, `.`, `_` o `-` (un UUID lo cumple). Si no, el servicio lo sustituye por uno nuevo y devuelve ese en la respuesta: el valor acaba en cada línea de log, en las trazas y en los eventos, y no puede traer saltos de línea ni kilobytes.

### ¿Puedo decidir por variable de entorno si el bucket se instrumenta?

**Sí, y esa es exactamente la forma prevista.** `TELEMETRY_INSTRUMENT_STORAGE=false` en el entorno del proceso —una línea de `deploy/.env`, una variable del pod, lo que uses— y los spans y los timers de las operaciones sobre los buckets dejan de existir. Lo mismo con `TELEMETRY_INSTRUMENT_CACHE` y `TELEMETRY_INSTRUMENT_MAIL`.

Tres detalles que evitan una sorpresa:

- **No se recompila ni se despliega otro código.** Es la misma imagen; lo que cambia es qué beans construye el contexto al arrancar.
- **Cambia al REINICIAR, nunca en caliente.** La decisión se toma cuando se construye el contexto de Spring, así que la variable no se relee después.
- **Lo que se apaga es la OBSERVACIÓN, no el subsistema.** El bucket se sigue escribiendo, el correo se sigue enviando, la caché sigue cacheando. Lo único que desaparece es la instrumentación: un objeto menos en medio de cada llamada.

Las variables solo existen si tu diseño trae ese subsistema: en un servicio sin buckets no se emite `TELEMETRY_INSTRUMENT_STORAGE`, porque una palanca que no mueve nada es peor que no tenerla.

Y al revés, para el caso contrario —«esto emite demasiado y quiero bajar volumen»—: el orden barato es apagar primero estos tres interruptores, después subir `METRICS_EXPORT_STEP`, y solo al final tocar el muestreo.

---

## 7. El colector: cambiar o añadir un backend

El archivo de las pruebas manuales es `deploy/otel/collector.yaml` y tiene cuatro bloques:

```yaml
receivers:    # por dónde entran los datos (aquí, OTLP)
processors:   # qué se les hace antes de salir
exporters:    # a dónde van
service:
  pipelines:  # cómo se encadenan, una por señal (traces, metrics, logs)
```

Los procesadores **van en orden y el orden importa**:

| Procesador | Qué hace | Por qué ahí |
|---|---|---|
| `memory_limiter` | Rechaza si se queda sin memoria | **El primero**: si el backend se atasca, el colector rechaza en vez de morirse, y el servicio reintenta |
| `resourcedetection` | Completa de dónde viene el dato | Antes de exportar, sin pisar lo que ya trae la app |
| `attributes/redact` | Borra cabeceras de credenciales | Antes de salir: es la última red antes del backend |
| `batch` | Agrupa y comprime | **El último**: exportar dato a dato satura cualquier backend |

### Cambiar de backend

Tres pasos, siempre los mismos:

1. **Declara el exportador nuevo** en `exporters`.
2. **Ponlo en los pipelines** que quieras que vayan a él.
3. **Reinicia solo el colector.**

Ejemplos listos para copiar:

```yaml
# Trazas a un Tempo o Jaeger que ya tienes
exporters:
  otlp/tempo:
    endpoint: tempo.observabilidad.svc:4317
    tls:
      insecure: true            # quítalo si el destino tiene TLS

# Métricas a un Prometheus con remote-write
  prometheusremotewrite:
    endpoint: http://prometheus:9090/api/v1/write

# Cualquier proveedor SaaS que hable OTLP (la credencial, del ENTORNO del colector)
  otlphttp/proveedor:
    endpoint: https://otlp.ejemplo.com
    headers:
      api-key: ${env:MI_API_KEY}
```

y en los pipelines:

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, attributes/redact, batch]
      exporters: [otlp/tempo]            # ← antes decía otro
```

### Añadir un backend sin quitar el que hay

Un pipeline admite **varios exportadores**: el colector manda una copia a cada uno. Es la forma de migrar sin apagar nada.

```yaml
    traces:
      exporters: [otlphttp/backend, otlp/tempo]
```

### Comprobar y aplicar

Antes de reiniciar nada, valida el archivo (no arranca nada, solo lo lee):

```bash
podman run --rm -v "$PWD/deploy/otel:/cfg:ro" \
  otel/opentelemetry-collector-contrib:<versión> validate --config=/cfg/collector.yaml
```

Si hay un error —un exportador citado en un pipeline que no existe, una clave mal escrita— lo dice con el camino exacto. Después:

```bash
podman restart <proyecto>_otel-collector_1     # o el nombre que muestre `podman ps`
```

La aplicación no se toca. Si la editas mientras hay tráfico, lo que estuviera en la cola del colector se pierde; con `retry_on_failure` y `sending_queue` puestos (ya vienen), lo que ya había entrado se reintenta.

**Y si el backend está caído, el servicio no se entera.** Es fácil de comprobar y conviene haberlo visto una vez: apunta el exportador a un host que no existe, reinicia el colector y sigue llamando a la API. El servicio responde igual; quien acumula reintentos y avisa en sus logs es el colector. Al devolver el endpoint bueno, los datos vuelven a aparecer en el backend. Esa es, en una frase, la razón de tener un colector en medio.

**Las credenciales del backend van en el entorno del colector** (`${env:…}`), nunca dentro del archivo ni en la aplicación. La aplicación no tiene credenciales de observabilidad: esa es otra ventaja de tener un colector en medio.

---

## 7 bis. El panel y las alertas que vienen puestos

`build` genera un panel y las alertas **derivados de tu diseño**, y `deploy/up.sh` los provisiona en el Grafana de prueba. Al levantarlo, el script imprime el enlace directo al panel: si da 404, el provisionado no entró (es la forma más rápida de comprobarlo).

| Archivo | Qué es |
|---|---|
| `deploy/observability/dashboards/<servicio>.json` | El panel. Una fila por superficie que tu diseño tiene: casos de uso, HTTP, outbox, mensajes, caché, almacenamiento, correo, y siempre la de **runtime** |
| `deploy/observability/dashboards-provisioning.yaml` | Le dice a Grafana dónde buscar el panel |
| `deploy/observability/alerting/keel-alerts.yaml` | Las alertas, el **contacto** al que salen y la política que enruta a él, en el formato de provisioning de Grafana |
| `deploy/observability/prometheus-rules.example.yaml` | Las mismas reglas en formato Prometheus, para llevártelas a otro sitio. Nadie las monta: son una referencia |
| `deploy/observability/alertmanager.example.yaml` | La otra mitad de esa referencia: a quién se le cuenta. Una regla sin receptor no avisa a nadie |

**Las filas del panel solo aparecen si su subsistema existe.** Un panel con paneles vacíos diría «falta señal» donde lo que falta es el subsistema, y eso manda a buscar un problema que no hay.

Los paneles de percentiles llevan **exemplars activados**: cada punto enseña un diamante con el id de una traza de ejemplo. Es el salto que este panel existe para dar.

### La fila de runtime

Va siempre y va la última, porque no es lo que se mira primero: es a donde se baja cuando lo de arriba dice que algo va lento y no se sabe por qué. Enseña el **heap** sobre su máximo, el tiempo que se va en **pausas de GC** y las **conexiones del pool**.

El pool depende del modelo de persistencia de tu diseño: con `relational` es Hikari (en uso, libres, máximo y **esperando**), con `document` es el del driver de Mongo (en uso, tamaño y **cola de espera**). Son series distintas y por eso cada rama enseña la suya: una consulta a la serie de la otra no falla, deja media fila vacía justo donde vas a mirar.

### Las alertas

| Alerta | Cuándo dispara | Por qué esa |
|---|---|---|
| Eventos del outbox rendidos | `keel.outbox.dead_lettered > 0` durante 5 min | Es **pérdida de datos** en el mecanismo cuya única promesa es que no se pierde nada. Solo se emite si tu diseño usa `reliability: outbox` |
| Tasa de error de los casos de uso | más del 5 % con desenlace `error` durante 10 min | Cuenta solo `error`: un rechazo del dominio (`rejected`) es un 4xx esperado, no un fallo del servicio |
| p95 de los casos de uso | alguna operación pasa de 1 s durante 10 min | Degradación antes de que alguien se queje |
| Saturación del pool de conexiones | hay peticiones **esperando** una conexión durante 5 min | A partir de ahí la latencia la pone la cola, no el trabajo. Solo si tu diseño persiste algo |
| Presión de memoria en el heap | por encima del 90 % durante 10 min | Lo que viene después es GC continuo y, al final, un `OutOfMemoryError` — que sí se ve, pero cuando ya no hay nada que hacer |
| Retraso del consumidor | más de 1.000 mensajes por detrás durante 10 min | No lo dice ninguna otra: un consumidor que se queda atrás procesa igual de rápido, solo que cada vez más tarde. Solo con Kafka (§ 10) |

Las tres primeras dicen que algo ya va mal; las tres siguientes dicen **por qué**, y llegan antes. Los umbrales (5 %, 1 s, 1.000 mensajes) son un punto de partida: dependen del volumen de cada servicio y se ajustan con los primeros días de tráfico real.

**Sin datos no disparan.** Todas se provisionan con `noDataState: OK`, y no es un detalle: el valor por defecto de Grafana es notificar, así que un servicio recién desplegado —o de madrugada, sin tráfico— avisaría de todo a la vez sin que pase nada. Se midió en vivo, y era exactamente lo que hacía. Si lo que falta es el servicio entero, eso lo dice la plataforma (liveness), no una alerta de negocio. Un error de evaluación sí se ve, pero con nombre propio (`execErrState: Error`) para que no se confunda con «el p95 subió».

### A dónde salen

En `deploy/` las alertas **sí salen**: el mismo archivo provisiona un contacto de tipo webhook y la política que enruta a él. La URL sale de la variable `ALERT_WEBHOOK_URL`, que por defecto apunta a un **sumidero** —un contenedor que registra lo que recibe— para que «la alerta salió» se pueda leer en vez de suponerse:

```bash
# lo que el contacto ha recibido, con su cuerpo
curl -s http://localhost:${ALERT_SINK_PORT:-8091}/__admin/requests
```

Para recibirlas de verdad, cambia `ALERT_WEBHOOK_URL` en `deploy/.env` por la URL de tu canal (o la del puente que hable con él) y vuelve a levantar. Y si no usas Grafana, las dos mitades portables están en `prometheus-rules.example.yaml` (cuándo disparar) y `alertmanager.example.yaml` (a quién contárselo).

> **Ojo con llevarte `keel-alerts.yaml` a un Grafana compartido.** Provisionar `policies` **sustituye** el árbol de notificación por defecto de esa organización, y el árbol provisionado deja de ser editable desde la interfaz. En el Grafana de `deploy/` da igual; en uno de verdad, mira antes qué árbol tiene.

> **`build` REGENERA el panel.** Si lo editas en la interfaz de Grafana, el siguiente `keel-spring build` lo pisa. Para conservar un cambio: exporta el JSON desde Grafana y sustituye el archivo de `deploy/observability/dashboards/`, sabiendo que el próximo build también lo pisará — el camino sostenible es que el cambio valga para todos los servicios y entre en el generador.

---

## 8. Producción

`deploy/` es para probar a mano. Para producción, antes que nada: **el colector que necesitas depende de la infraestructura donde despliegues**, no del servicio.

Lo que no cambia es el servicio. Solo conoce la dirección de un colector (`OTEL_EXPORTER_OTLP_ENDPOINT`) y unas pocas variables: muestreo, qué expone el actuator y si envía logs por OTLP. El mismo jar sirve igual en Kubernetes, en máquinas virtuales, en ECS o con docker compose. Lo que sí cambia de una plataforma a otra es **cuántos colectores hacen falta, dónde van y cómo se configuran**, y eso lo deciden dos preguntas:

1. **¿Alguien tiene que leer la consola de los contenedores en cada máquina?** Si la plataforma ya recoge los logs (CloudWatch, un Fluent Bit o un Vector que ya tengas…), no. Si nadie los recoge, hace falta un colector en cada máquina.
2. **¿Vas a hacer muestreo por cola con más de un colector?** Si es así, todos los fragmentos de una traza tienen que llegar al mismo colector, y hay que repartirlos por `traceID`. Con un solo colector, o sin muestreo por cola, no hace falta.

| Infraestructura | Qué colector | De dónde partir |
|---|---|---|
| **Un servidor o pocas instancias** (docker compose, una VM) | **Uno solo**, que lo hace todo | `deploy/otel/collector.yaml`, cambiando el bloque `exporters` al backend real |
| **VMs o hosts con Docker, en varias máquinas** | Uno por host (servicio de systemd o contenedor) que lee los logs de Docker o de journald, y un gateway si hay muestreo por cola | Las dos plantillas de abajo, sustituyendo lo propio de Kubernetes: las rutas de logs, `k8sattributes` por `resourcedetection` y la lista de gateways por una estática o por DNS |
| **AWS ECS / Fargate** | Normalmente uno como sidecar en cada tarea (la distribución de AWS, ADOT). Los logs de consola los recoge la plataforma | La plantilla del gateway, si hay muestreo por cola; la del agente no hace falta |
| **Kubernetes** | Los dos: agente por nodo y gateway central | Las dos plantillas, tal cual están |

Las dos plantillas que genera `build` están escritas **para Kubernetes**: leen los logs de `/var/log/pods/`, añaden los metadatos con `k8sattributes` y localizan los gateways por el DNS de un servicio headless. Son la referencia más completa porque cubren las dos preguntas a la vez; en otra plataforma se adaptan, no se copian.

En Kubernetes:

| Archivo | Dónde va | Qué hace |
|---|---|---|
| `deploy/otel/collector-agent.example.yaml` | Uno por nodo (DaemonSet) | Recoge la **consola** de los contenedores (`filelog`), recibe trazas y métricas de los pods del nodo, añade los metadatos de Kubernetes y reparte las trazas por `traceID` |
| `deploy/otel/collector-gateway.example.yaml` | Un despliegue central | Decide **qué trazas se guardan** (`tail_sampling`) y es el único que conoce el backend |

Son dos y no uno por dos razones que no se pueden cumplir a la vez en el mismo sitio: recoger la consola exige estar **en cada nodo**, y decidir con la traza entera delante exige que **todos sus spans lleguen al mismo colector**.

**Y las métricas, ¿quién las scrapea en producción?** Sea cual sea la plataforma, no hay colector de `deploy/`, así que hay dos rutas soportadas y conviene elegir antes de desplegar:

| Ruta | Cómo | Qué poner |
|---|---|---|
| **Scrape** (por defecto: conserva los exemplars) | el colector más cercano al servicio lee `:8081/actuator/prometheus` (en Kubernetes, el agente del nodo; en una VM, el del host) | Nada en el servicio: en `production` el actuator ya vive en el **puerto de gestión** (`MANAGEMENT_PORT`, 8081). Lo que hay que cuidar es **no publicar ese puerto** fuera del pod o del host: los nombres de las métricas son nombres de negocio. Las plantillas de colector de `deploy/otel/` todavía no traen esa lectura: hay que añadir un receptor `prometheus` que apunte al 8081 |
| **Push por OTLP** (pierde los exemplars) | si tu plataforma no deja exponer el endpoint | `METRICS_EXPORT_OTLP=true` y `METRICS_EXPORT_PROMETHEUS=false` |

Cambiar de una a otra **no exige recompilar**: las dos dependencias van en la imagen justo por eso.

**Y las alertas, ¿a quién avisan allí?** El contacto que viene provisionado lee su URL de `ALERT_WEBHOOK_URL`, así que en un entorno de verdad basta con darle esa variable al proceso de Grafana —no hay que editar ningún archivo generado—. Si tu alerta la evalúa Prometheus y no Grafana, las dos mitades están en `prometheus-rules.example.yaml` y `alertmanager.example.yaml`, y conviene llevarse las dos: una regla sin receptor no avisa a nadie, que es justo el estado en el que suele quedarse esto.

El **muestreo por cola** del gateway se queda con todas las trazas que tienen un error, todas las lentas y una parte del resto. Es mucho mejor que muestrear a ciegas en el servicio: las trazas que te interesan son justo las raras. Si lo usas, pon `TRACING_SAMPLING_PROBABILITY=1.0` en la aplicación, o estarás muestreando dos veces.

---

## 9. Operación del día a día

### «No llega nada»

En orden, que es de lo más común a lo más raro:

1. **¿Está encendida la exportación?** `TELEMETRY_EXPORT_ENABLED` en el entorno de la app. En `local` está apagada a propósito.
2. **¿Apunta a donde crees?** `OTEL_EXPORTER_OTLP_ENDPOINT` tiene que ser alcanzable **desde dentro del contenedor de la app** — `localhost` ahí es la propia app, no tu máquina.
3. **¿Llega al colector?** Mira sus logs: `podman logs <colector>`. Si el servicio no alcanza el colector, la app escribe avisos de exportación fallida cada pocos segundos.
4. **¿Sale del colector?** Añade temporalmente el exportador `debug` al pipeline y reinícialo: imprime lo que recibe. Si ahí hay datos, el problema está entre el colector y el backend.
5. **¿Es el muestreo?** En `production` solo se conserva una de cada diez trazas por defecto. Sube `TRACING_SAMPLING_PROBABILITY` para una prueba.
6. **¿Es el ruido lo que esperabas ver?** Las peticiones al *actuator* y los ciclos de las tareas programadas se descartan a propósito (ver más abajo).
7. **¿Y los logs?** Si esperas verlos en el backend pero no los recoge nadie, te falta `LOG_EXPORT_OTLP=true` o un recolector de la consola.
8. **¿Faltan solo las MÉTRICAS?** Ahí el camino es otro: no salen, las viene a buscar el colector. Pregúntale al servicio directamente —`curl http://localhost:8080/actuator/prometheus` en `local` y `develop`, `curl http://localhost:8081/actuator/prometheus` en `production`, que usa el puerto de gestión— y mira si la serie está. Si responde 404, no está expuesto (`MANAGEMENT_ENDPOINTS`); si está y no llega al backend, el problema es el scrape del colector, no la aplicación.
9. **¿Y qué salió del colector?** `deploy/otel/out/` tiene, en crudo y en disco, lo que reexportó: `traces.json`, `metrics.json`, `logs.json`. Si ahí hay datos, el problema está entre el colector y el backend.

### «No veo los exemplars»

Lo primero, porque descarta casi todo: **el exemplar solo viaja en la exposición OpenMetrics**. Pídela a propósito:

```bash
curl -H 'Accept: application/openmetrics-text; version=1.0.0' \
  http://localhost:8080/actuator/prometheus | grep 'keel_use_case_seconds_bucket.*# {'
```

Con el `Accept` por defecto la respuesta trae las mismas series y las mismas etiquetas, y **ni un exemplar** — así que mirarlo sin pedir OpenMetrics lleva a concluir que no se emiten. Si aparecen ahí y no en el backend: el Prometheus del backend los acepta y los tira si no se arrancó con `--enable-feature=exemplar-storage` (el de `deploy/` ya lo lleva). Y si la traza no se muestreó, el punto se publica sin exemplar: es correcto, y es por qué en `production`, con el 10 %, la mayoría no lleva ninguno.

### «La traza está partida»

- Si el corte está entre **publicar y consumir**, casi siempre es el listener: tiene que usar `runWith(envelope.metadata(), …)` (§ 5).
- Si el corte está entre **dos servicios por HTTP**, quien llama tiene que propagar la cabecera `traceparent`. Los clientes que genera build lo hacen; un cliente escrito a mano puede no hacerlo.
- Si aparecen **spans sueltos sin padre**, suele ser trabajo lanzado a otro hilo sin propagar el contexto. Un hilo nuevo, virtual o no, empieza sin traza, sin MDC y sin `correlationId`. Los dos caminos que sí lo llevan son `ContextPropagatingExecutors` (`application/support`) y los métodos `@Async`, a cuyo executor build le aplica un `ContextPropagatingTaskDecorator`. Cualquier otra forma —`CompletableFuture.supplyAsync` sin executor, `parallelStream`, un `Thread` a mano, un executor de `Executors`— la marca `infra/check-logging.sh`.

### Ruido, y por qué no lo hay

El relay del outbox consulta la base de datos **cada segundo**. Sin filtro, eso serían miles de trazas vacías al día que además se comerían tu cuota de muestreo. Build descarta las peticiones al *actuator*, los ciclos de las tareas programadas y las consultas que no cuelgan de nada.

> Medido en una corrida real: media hora de servicio en marcha, con unas 1.800 pasadas del relay, produjo **16 trazas**, todas con contenido.

Lo que sí se ve: cada evento publicado (`keel.outbox.publish`) y cada barrido que hace trabajo, porque pasan por un caso de uso.

### Coste y cardinalidad

Lo que hace cara —y lenta— una plataforma de observabilidad es casi siempre la **cardinalidad**: una etiqueta con muchos valores distintos multiplica las series. Por eso las métricas que genera build solo llevan etiquetas de baja cardinalidad (nombre de operación, de evento, código de respuesta), y los identificadores (ids, correlación) van como **atributos del span**, no como etiquetas de métrica. Si añades métricas propias, respeta esa frontera.

Palancas para ajustar volumen, de menos a más invasivas: `METRICS_EXPORT_STEP`, el muestreo por cola en el gateway, `TRACING_SAMPLING_PROBABILITY` y, para los logs, el nivel (`LOG_LEVEL_ROOT`, `LOG_LEVEL_APP`).

Y desde ahora esa frontera **no es solo una convención**: `infra/check-telemetry.sh` (lo ejecuta el agente de calidad, y tú también puedes) veta una etiqueta de métrica cuya clave no esté en el vocabulario que estampa build. Es el único gate cuyo hallazgo no produce ningún síntoma —nada falla, nada se loguea, ningún escenario se pone rojo—, y la corrección no es quitar el dato sino moverlo: al span, con `addHighCardinalityKeyValue`, donde un identificador sirve para lo que sirve sin multiplicar ninguna serie.

### Las alertas

Ya vienen puestas y provisionadas: § 7 bis.

### Qué no sale en la telemetría, nunca

Ni datos personales, ni secretos, ni cuerpos de peticiones o mensajes. Se identifica por id. El colector borra además las cabeceras de credenciales por si acaso, pero eso es la última red, no la primera: lo que no se emite no se puede filtrar.

---

## 10. Límites conocidos

- **Solo se instrumenta lo que pasa por Micrometer** (peticiones HTTP, casos de uso, base de datos, clientes HTTP, mensajería, caché, buckets y correo). No se usa el agente Java de OpenTelemetry, así que una librería de terceros que hagas servir por tu cuenta no aparece sola.
- **Los exemplars no van por OTLP**, y de ahí que las métricas se scrapeen: el registro OTLP de Micrometer no sabe emitirlos hasta la versión 1.17, que llega con Spring Boot 4.1. Cuando el generador suba de Boot, el push por OTLP podrá llevarlos y el scrape dejará de ser necesario.
- **Un adaptador de un puerto instrumentado no puede ser `final`**: con el aspecto puesto, Spring lo proxya por CGLIB y el contexto no arranca. El mensaje de error habla de CGLIB, no de telemetría.
- **SNS/SQS no propaga cabeceras de traza** (su cliente no lo soporta todavía): ahí la traza viaja solo en el sobre del evento, y por eso la línea del listener es obligatoria.
- **El retraso del consumidor solo sale con Kafka.** Su cliente conoce el final de la partición, así que Micrometer puede restar y el panel lo enseña. Con RabbitMQ el consumidor no sabe cuántos mensajes quedan por detrás —no hay offset que restar—: eso se mira en el broker, por la profundidad de la cola (plugin de management o `rabbitmq_exporter`). Con SNS/SQS el dato es de CloudWatch (`ApproximateAgeOfOldestMessage`). En los dos casos el panel **no emite** ni la fila ni la alerta a propósito: una consulta a una serie que nadie publica no falla, se queda vacía, y su alerta no dispara nunca.
- **Los logs del arranque no salen por OTLP**: el appender se instala cuando el contexto de Spring ya existe. Sí están en la consola.
- **El muestreo por cola necesita** que todos los spans de una traza lleguen al mismo colector; por eso el agente de nodo los reparte por `traceID`.
- **`local` no exporta por defecto**, para no ensuciar la suite de escenarios.

---

## 11. Qué de esta guía está medido

Casi todo lo de aquí se ejecutó sobre una pila real (un servicio generado con `telemetry: otel`, PostgreSQL, Kafka, el colector y Grafana LGTM, con podman): el recorrido de Grafana, las consultas de ejemplo de las tres señales, el salto entre traza y logs, la traza que cruza el outbox y el consumo, el filtro de ruido, la validación del colector y la prueba del backend caído.

Y hay una parte que, además, es **repetible**: `npm run telemetry-check` (en el repo del generador) arranca la aplicación contra la infraestructura real y comprueba que el endpoint de scrape responde sin token, que la serie del caso de uso lleva su operación y su desenlace —que es lo que filtra la alerta de errores—, que el histograma trae exemplars, que el `correlationId` **no** es etiqueta de métrica, que las operaciones de bucket y los envíos de correo dejan su serie sin filtrar datos de nadie, que la caché cuenta aciertos y fallos de verdad, y que **apagar el interruptor de un subsistema hace desaparecer su serie dejando el subsistema funcionando**. Está falsado: rompiendo cada mecanismo por separado se comprobó que el caso correspondiente se pone rojo. Esa medición encontró dos cosas que nadie sabía —que un bean de exemplars propio era código muerto porque Boot ya lo autoconfigura, y que las métricas de caché se publican a cero sin estadísticas habilitadas, con lo que un panel podía enseñar un ratio inventado—.

Lo que **no** se ha ejercitado en vivo y se documenta por diseño: las dos plantillas de producción (`collector-agent`/`collector-gateway`), que dependen de un clúster de Kubernetes, y los exportadores de proveedores concretos, que dependen de sus credenciales. Están escritas siguiendo la configuración de referencia del proyecto OpenTelemetry y validadas con `otelcol validate`, que comprueba la forma, no el destino.

Esa red cubre además la fila de **runtime** —que la JVM publique el heap y las pausas de GC que el panel consulta, y que el **pool de tu modelo** publique las suyas, Hikari o el del driver de Mongo, incluida la serie sobre la que alerta— y el **retraso del consumidor** con Kafka, creando el consumidor a partir de la `ConsumerFactory` de la propia aplicación: quien publica esa serie no es Kafka, es el listener de Micrometer que Boot instala sobre esa factoría, así que un consumidor con propiedades propias habría medido una copia de sí mismo.

Y la otra mitad —la que hasta ahora no medía nadie— la cubre `npm run deploy-check`: levanta `deploy/` entero, con la **aplicación en un contenedor**, manda tráfico y le pregunta **al backend**. Ahí se comprueba lo que `telemetry-check` no puede: que el colector **alcance** a la aplicación por la red del compose y se traiga sus métricas, que el **exemplar sobreviva** el viaje, que con él se llegue a la **traza**, que los logs lleguen, que el **panel y las reglas provisionadas hayan entrado** —hasta ahora eso solo se veía mirando la interfaz— y que el **contacto entregue de verdad**, disparando su notificación de prueba y leyéndola en el sumidero. Esta última es la que más barata salía de perder: la URL del contacto viene de `$__env{ALERT_WEBHOOK_URL}`, y si esa variable no llega al proceso de Grafana el contacto se provisiona con el literal dentro, la entrega falla y **no hay ningún síntoma** — las alertas se siguen viendo en la interfaz.

Esa pasada ya se ha corrido, y cerró **8 de 8** sobre `job-dispatch` con PostgreSQL. Encontró dos cosas que nadie sabía: que las alertas generadas **disparaban sin datos** (arreglado arriba) y que un proyecto recién generado **no arranca en `deploy/`** —el perfil `develop` valida el esquema contra `db/migration/`, y el baseline lo exporta el agente de calidad—, así que el propio check le da esa precondición y lo dice en voz alta. `deploy-check` mide la ruta de la telemetría; no es una prueba de que un árbol sin agente se despliegue.

Lo que sigue **sin ejercitarse** son las dos plantillas de Kubernetes, por la razón de siempre: hacen falta un clúster y unas credenciales.

## 12. Dónde seguir

- `conventions/logging.md` — qué loguea build en cada frontera, qué puede añadir el agente y qué está prohibido.
- `conventions/observability.md` — las reglas que el código tiene que respetar para no romper nada de esto.
- `deploy/otel/collector.yaml` — el archivo que decides tú, comentado por dentro.
