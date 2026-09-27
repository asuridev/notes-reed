# Servicio de notificación por correo y gestión de plantillas

**Respuesta corta:** un servicio de correo que sirve a muchos sistemas se sostiene sobre una sola regla — **su contrato pertenece al dominio «notificación», nunca al dominio de quien lo llama**. El llamante manda una clave de plantilla y unos datos; nunca manda HTML.

Y sobre las plantillas hay una confusión que conviene disolver antes de nada: **en ejecución viven siempre en la base de datos del servicio de notificación**. Eso no se decide. Lo único que se decide es dónde está su **original**:

- **Modelo A** — la plantilla nace y vive en el servicio de notificación. La base de datos es el único sitio del mundo donde está.
- **Modelo B** — el original está en el repositorio del sistema consumidor, en git, y el pipeline de despliegue la empuja al servicio de notificación.

El modelo A es más simple y es la elección correcta para empezar. Su única desventaja de verdad es que la plantilla y el código que la usa pueden desincronizarse, y todo lo demás deriva de ahí.

---

## 1. El problema: por qué estos servicios degeneran

Un servicio de notificación compartido casi siempre muere por la misma causa: **acaba conociendo el dominio de sus llamantes**.

Empieza bien. Alguien necesita mandar la confirmación de un pedido y monta un servicio que se suscribe a `OrderPlaced`, con el HTML de la confirmación en su propio repositorio. Funciona.

Llega el segundo sistema. Ahora hay que suscribirse también a `UserRegistered` y añadir otra plantilla al repo. Se edita el servicio, se despliega. Llega el tercero. Y el quinto. Para entonces el servicio de notificación es un cuello de botella con lista de espera: cada equipo que quiere mandar un correo tiene que abrir un ticket al equipo que lo mantiene, y ese equipo despliega para cada alta.

> **Idea clave:** un servicio compartido deja de ser reutilizable en el momento en que dar de alta un consumidor nuevo requiere **editar su código**. Todo lo que sigue está ordenado para que un alta sea una operación de datos, no un despliegue.

---

## 2. Los cinco principios

### 2.1 El contrato es del dominio «notificación»

El servicio no acepta «un pedido». Acepta **un envío**:

```json
{
  "templateKey": "order-confirmed",
  "locale": "es-ES",
  "to": ["cliente@ejemplo.com"],
  "variables": {
    "orderNumber": "A-1042",
    "customerName": "Ana",
    "total": "89,90 €"
  }
}
```

No hay nada de comercio ahí. `orderNumber` es una variable de una plantilla, no un concepto que el servicio entienda. Por eso el mismo endpoint sirve para confirmar un pedido, para restablecer una contraseña o para avisar del vencimiento de una póliza.

### 2.2 Una sola puerta de entrada por eventos, y genérica

Si además del HTTP se quiere ofrecer entrada asíncrona, se declara **una** suscripción, `NotificationRequested`, sobre un canal genérico. **No una por sistema.**

Es exactamente aquí donde se decide si el servicio es reutilizable. Con `OrderPlaced`, `UserRegistered`, `InvoiceIssued`… cada alta es una edición del diseño. Con un único evento genérico, el sistema número doce entra sin tocar nada.

Un canal genérico exige, eso sí, que **nadie publique en él sin autenticarse**: sin token delante, de dónde sale la aplicación deja de ser obvio. Cómo se resuelve el inquilino en este camino está en §6.8.

### 2.3 Las plantillas son dato, no código

`Template` es una entidad del dominio del servicio: clave, idioma, versión, asunto, cuerpo y ciclo de vida. Dar de alta un correo nuevo es una llamada a la API, no un despliegue.

Es el principio que hace posible el 2.2. Un canal genérico no sirve de nada si el contenido sigue estando compilado dentro del servicio.

### 2.4 La identidad sale del token, no del payload

Cada sistema consumidor es un cliente máquina con sus credenciales `client_credentials`. La aplicación a la que pertenece la petición se deriva del token.

Eso aísla plantillas, remitente por defecto, supresiones y trazas — y nadie puede enviar en nombre de otro, cosa que sí ocurriría con un `applicationId` viajando en el cuerpo, donde cualquier cliente autenticado podría poner el de otro.

### 2.5 Aceptar y rastrear, no enviar en línea

`POST /v1/notifications` responde **`202 Accepted`** con un `notificationId`. El resultado viaja después como evento (`NotificationSent`, `NotificationFailed`), y quien no quiera escuchar eventos consulta `GET /v1/notifications/{id}`.

La razón no es de rendimiento, es de acoplamiento: **la disponibilidad del proveedor SMTP no debe entrar en la transacción de quien llama**. Un proveedor lento no puede hacer que fallen los pedidos.

### 2.6 Y dos piezas que solo son obligatorias por ser compartido

**Idempotencia.** Con N sistemas reintentando —y con un consumidor de eventos que es *at-least-once* por definición— sin clave de deduplicación se mandan correos duplicados a clientes reales. Se implementa como una restricción de unicidad, no como una comprobación previa: dos peticiones simultáneas con la misma clave llegando a dos instancias distintas tienen que colisionar en la base de datos.

**Lista de supresión.** Un rebote duro o una queja de spam de *un* sistema quema la reputación del remitente de **todos**. El servicio ingiere el webhook del proveedor y un rebote duro crea automáticamente una supresión que bloquea envíos futuros a esa dirección. Las supresiones son por aplicación; el daño que previenen es compartido, y por eso no puede quedar en manos de cada llamante.

---

## 3. Recorrido de un envío

```
orders-service                          notifications-email
─────────────                           ───────────────────
POST /v1/notifications  ─────────────▶  1. resuelve la Application desde el token
Authorization: Bearer <token M2M>       2. busca la plantilla activa
Idempotency-Key: order-A-1042-confirm      (application + key + locale + status=active)
{ templateKey: "order-confirmed",       3. valida las variables obligatorias
  locale: "es-ES",                      4. comprueba la lista de supresión
  to: ["cliente@ejemplo.com"],          5. persiste la notificación
  variables: { ... } }                  6. responde
                        ◀─────────────  202 { "notificationId": "3d2e…" }

                                        ── en segundo plano ──
                                        7. renderiza asunto y cuerpos
                                        8. entrega por SMTP / API del proveedor
                                        9. registra el intento
                                       10. publica NotificationSent
        ◀───── evento ────────────────
```

Los pasos 1 a 5 son síncronos y baratos. Todo lo que puede tardar o fallar por causas externas está después del `202`.

Y los errores que sí son síncronos son los que el llamante puede arreglar (esto es el camino HTTP; por el canal de eventos no hay a quién responder, ver §6.8):

| Código | Cuándo | HTTP |
|---|---|---|
| `TEMPLATE_NOT_FOUND` | No hay plantilla activa con esa clave e idioma para esta aplicación | 422 |
| `MISSING_TEMPLATE_VARIABLE` | Falta una variable que la plantilla declara obligatoria | 422 |
| `RECIPIENT_SUPPRESSED` | La dirección está en la lista de supresión | 422 |
| `APPLICATION_INACTIVE` | La aplicación está dada de baja | 403 |

> **Idea clave:** validar contra las variables **declaradas** es lo que evita el fallo más caro de todos — mandar un correo que dice «Tu pedido por  € está confirmado». Ese fallo, sin la validación, se descubre por la reclamación del cliente.

---

## 4. Anatomía de una plantilla

Tres piezas: asunto, cuerpo HTML y cuerpo en texto plano.

### 4.1 El asunto

```handlebars
Tu pedido {{orderNumber}} está confirmado
```

### 4.2 El cuerpo HTML

Solo el cuerpo: la cabecera, el pie, los estilos y la marca los pone la **maquetación de la aplicación**.

```handlebars
<h1>Gracias por tu compra, {{customerName}}</h1>

<p>Hemos confirmado tu pedido <strong>{{orderNumber}}</strong>.</p>

<table role="presentation" width="100%">
  {{#each lines}}
  <tr>
    <td>{{this.description}}</td>
    <td align="right">{{this.quantity}} × {{this.unitPrice}}</td>
  </tr>
  {{/each}}
  <tr>
    <td><strong>Total</strong></td>
    <td align="right"><strong>{{total}}</strong></td>
  </tr>
</table>

{{#if trackingUrl}}
  <p><a href="{{trackingUrl}}">Sigue tu envío</a></p>
{{/if}}

<p>Si no reconoces esta compra, responde a este correo.</p>
```

### 4.3 El cuerpo en texto

```handlebars
Gracias por tu compra, {{customerName}}.

Hemos confirmado tu pedido {{orderNumber}}.

{{#each lines}}- {{this.description}}: {{this.quantity}} x {{this.unitPrice}}
{{/each}}
Total: {{total}}
{{#if trackingUrl}}
Sigue tu envío: {{trackingUrl}}
{{/if}}
```

No es por los clientes de correo de texto —quedan pocos— sino porque los filtros antispam desconfían de un correo HTML sin alternativa textual. Sale como `multipart/alternative` con las dos partes.

### 4.4 Las variables declaradas

```yaml
key: order-confirmed
locales: [es-ES, en-GB]
variables:
  - { name: orderNumber,  required: true,  description: Número visible del pedido. }
  - { name: customerName, required: true,  description: Nombre de pila del cliente. }
  - { name: total,        required: true,  description: Importe total ya formateado con moneda. }
  - { name: lines,        required: true,  description: Líneas del pedido - description, quantity, unitPrice. }
  - { name: trackingUrl,  required: false, description: Enlace de seguimiento, si ya existe. }
```

### 4.5 El motor de renderizado, y una trampa importante

Fíjate en lo que **no** hay en la plantilla: ni formateo de importes, ni conversión de fechas, ni condiciones sobre el estado del pedido. `total` llega ya como `"89,90 €"`.

Eso es deliberado, y viene de la elección de motor.

En Spring lo natural sería **Thymeleaf**, pero Thymeleaf está pensado para plantillas *que escribes tú*: evalúa expresiones SpEL, y SpEL puede invocar métodos arbitrarios. Si las plantillas entran por una API que rellenan equipos ajenos, eso es una ejecución remota de código esperando a suceder.

Lo correcto para plantillas de origen externo es un motor **sin lógica arbitraria** — Handlebars o Mustache. Solo sustituyen variables, recorren listas y evalúan condiciones simples. No hay forma de llamar a nada.

Pagas en expresividad, y es más una virtud que un defecto: el formato de un importe tiene reglas de locale y se prueba mucho mejor en el sistema llamante que dentro de una cadena de texto guardada en una fila.

### 4.6 Dos escapes que hay que cerrar

**Escapado HTML por defecto en las variables.** Si un `orderNumber` llega con `<script>`, se escribe como texto. El XSS en correo es real: hay clientes que ejecutan.

**Saneado del asunto.** Un salto de línea dentro de una variable interpolada en el `Subject:` permite **inyectar cabeceras SMTP** — añadir un `Bcc:` que tú no pusiste. Se eliminan `\r` y `\n` antes de componer el mensaje.

---

## 5. Cómo se almacena

```sql
CREATE TABLE application (
    id              UUID PRIMARY KEY,
    key             VARCHAR(64)  NOT NULL UNIQUE,   -- casa con el client_id del token
    name            VARCHAR(120) NOT NULL,
    default_sender  VARCHAR(320) NOT NULL,
    default_locale  VARCHAR(10)  NOT NULL,
    active          BOOLEAN      NOT NULL DEFAULT TRUE
);

CREATE TABLE template (
    id              UUID PRIMARY KEY,
    application_id  UUID         NOT NULL REFERENCES application(id),
    key             VARCHAR(64)  NOT NULL,
    locale          VARCHAR(10)  NOT NULL,
    version         INTEGER      NOT NULL,
    subject         VARCHAR(200) NOT NULL,
    body_html       TEXT         NOT NULL,
    body_text       TEXT         NOT NULL,
    status          VARCHAR(16)  NOT NULL DEFAULT 'draft',
    created_at      TIMESTAMPTZ  NOT NULL,
    published_at    TIMESTAMPTZ,
    CONSTRAINT uk_template_version UNIQUE (application_id, key, locale, version)
);

-- "Como máximo una activa" no es una UNIQUE normal: es un índice parcial.
CREATE UNIQUE INDEX uk_template_active
    ON template (application_id, key, locale)
    WHERE status = 'active';

CREATE TABLE template_variable (
    id           UUID PRIMARY KEY,
    template_id  UUID        NOT NULL REFERENCES template(id) ON DELETE CASCADE,
    name         VARCHAR(64) NOT NULL,
    required     BOOLEAN     NOT NULL DEFAULT FALSE,
    description  VARCHAR(255),
    CONSTRAINT uk_template_variable UNIQUE (template_id, name)
);

CREATE TABLE notification (
    id                  UUID PRIMARY KEY,
    application_id      UUID         NOT NULL REFERENCES application(id),
    template_id         UUID         NOT NULL REFERENCES template(id),
    template_version    INTEGER      NOT NULL,      -- congelada
    locale              VARCHAR(10)  NOT NULL,
    variables           JSONB        NOT NULL,      -- lo que mandó el llamante
    rendered_subject    VARCHAR(200) NOT NULL,      -- ya interpolado
    status              VARCHAR(16)  NOT NULL,
    dedupe_key          VARCHAR(128) NOT NULL,
    correlation_id      VARCHAR(64),
    requested_at        TIMESTAMPTZ  NOT NULL,
    sent_at             TIMESTAMPTZ,
    provider_message_id VARCHAR(255),
    CONSTRAINT uk_notification_dedupe UNIQUE (application_id, dedupe_key)
);
```

Y las filas de la plantilla del ejemplo:

| Tabla | Contenido |
|---|---|
| `application` | `key='orders-service'`, `default_sender='pedidos@tutienda.com'`, `default_locale='es-ES'` |
| `template` | `key='order-confirmed'`, `locale='es-ES'`, `version=1`, `status='active'`, `subject='Tu pedido {{orderNumber}} está confirmado'` |
| `template_variable` | cinco filas: `orderNumber`/true, `customerName`/true, `total`/true, `lines`/true, `trackingUrl`/false |

### Cinco decisiones de almacenamiento y su porqué

**El cuerpo se guarda literal, con las llaves dentro.** No se preprocesa ni se compila al guardar. Lo que hay en la columna es exactamente lo que escribió quien la registró; el motor lo compila al renderizar, con una caché en memoria por identificador de plantilla.

**`TEXT`, no un bucket.** El cuerpo es pequeño (decenas de KB), está versionado, y quieres que viaje en la misma transacción, el mismo backup y la misma migración que el resto. Al bucket van las imágenes de la maquetación — y esas ni siquiera se adjuntan: se referencian por URL absoluta desde el HTML.

**Tabla hija para las variables declaradas, JSONB para las de la ocurrencia.** Las declaradas son **contrato**: se consultan en cada petición para validar, se muestran en la documentación de la plantilla y quieres poder responder «¿qué plantillas usan `trackingUrl`?». Las de la notificación son datos opacos de un envío concreto: JSONB es exactamente lo que hacen falta.

**La notificación congela `template_version` y `rendered_subject`.** Dentro de seis meses, con la plantilla ya en la v4, la traza sigue diciendo qué se envió de verdad. Sin eso, el histórico se reescribe solo cada vez que alguien publica un cambio, y una traza que decía «se envió esto» pasa a decir «hoy se enviaría esto otro» — que no es lo mismo, sobre todo cuando alguien reclama.

**El índice parcial.** «Como máximo una activa» no es una unicidad de columnas: es una unicidad *condicionada al estado*. Con `UNIQUE (application_id, key, locale)` a secas no podrías tener nunca dos versiones. PostgreSQL lo resuelve con `WHERE status = 'active'`; MySQL y SQL Server, que no tienen índices parciales, lo emulan con una columna generada o lo sostienen con un bloqueo en el caso de uso de publicación.

---

## 6. La tabla `application`: quién puede enviar y cómo se resuelve

Esta tabla es el eje del servicio y merece su propia sección: es lo que convierte un relay de correo compartido en un servicio multi-sistema.

### 6.1 Qué almacena

**Un registro de sistemas consumidores.** Una fila por cada sistema autorizado a mandar correos.

| id | key | name | default_sender | default_locale | active |
|---|---|---|---|---|---|
| `a1b2…` | `orders-service` | Pedidos de la tienda | `pedidos@tutienda.com` | `es-ES` | true |
| `c3d4…` | `identity-service` | Cuentas y acceso | `no-reply@tutienda.com` | `es-ES` | true |
| `e5f6…` | `billing-service` | Facturación | `facturacion@tutienda.com` | `es-ES` | false |

| Columna | Para qué |
|---|---|
| `key` | El identificador estable del sistema, y **la bisagra con la autenticación**: coincide con el `client_id` del token máquina |
| `default_sender` | Desde qué dirección salen sus correos. Cada sistema puede tener la suya, y es una dirección **verificada ante el proveedor** |
| `default_locale` | A qué idioma recurrir cuando la petición no especifica uno, o no existe plantilla en el que pidió |
| `active` | El interruptor. Poner `false` corta los envíos de ese sistema al instante, sin revocar credenciales ni desplegar |

Ese `active` es lo que respondes cuando un sistema con un bug empieza a mandar diez mil correos a la misma persona a las tres de la mañana.

**No son usuarios.** Ni los del servicio de notificación ni los de los sistemas consumidores. `cliente@ejemplo.com` no tiene fila en ningún sitio: es un dato dentro de la notificación, no una entidad del servicio. Una `application` es **un sistema**. Si tienes doce sistemas mandando correos, esta tabla tiene doce filas y crece una vez al trimestre.

### 6.2 Cómo se resuelve el inquilino

En el flujo `client_credentials` no hay usuario: el proveedor de identidad pone el `client_id` en el token. Keycloak lo deja en `sub`, y además en `azp` (*authorized party*); otros proveedores usan un claim `client_id`.

```json
{
  "sub": "orders-service",
  "azp": "orders-service",
  "scope": "notification:send notification:read"
}
```

```sql
SELECT * FROM application WHERE key = 'orders-service';
```

Ese es todo el mecanismo. Ni cabecera, ni campo en el cuerpo, ni configuración por consumidor.

> **Idea clave:** `applicationId` **no viaja nunca en la petición**. Si viajara, cualquier cliente autenticado podría poner el de otro y enviar en su nombre, con su remitente verificado. La identidad se deriva de la credencial o no es identidad.

Detalle de implementación: **qué claim se lee debe ser configurable, no cableado.** Cambia si algún día se cambia de proveedor de identidad.

Y «todo el mecanismo» lo es de **esta** puerta. Por el canal de eventos no llega ningún token, así que la resolución es otra: §6.8.

### 6.3 Un sistema no registrado no puede enviar: cuatro puertas

| Situación | Qué pasa | Respuesta |
|---|---|---|
| No tiene cliente máquina dado de alta | No consigue token; no llega a tocar el servicio | `401` |
| Token válido, pero no hay fila en `application` | Se rechaza | `403 APPLICATION_NOT_REGISTERED` |
| Fila existe con `active = false` | Se rechaza | `403 APPLICATION_INACTIVE` |
| Fallo lógico que dejara pasar cualquiera de los anteriores | `notification.application_id NOT NULL REFERENCES application(id)` impide insertar la fila | error de integridad |

El segundo caso **ocurre de verdad**: el proveedor de identidad y la base de datos del servicio son dos registros distintos y se desincronizan — alguien crea el cliente en Keycloak y olvida la fila.

Y ahí hay una tentación que conviene nombrar para descartarla: *«tiene token válido, le creo la fila al vuelo»*. **No.** El alta de una aplicación implica verificar el dominio de envío y fijar un remitente autorizado; auto-aprovisionar significa que ese sistema enviaría desde un remitente por defecto que nadie verificó. Se falla cerrado.

La cuarta puerta es la más sólida porque no es una comprobación que alguien pueda olvidarse de escribir: es una **restricción de integridad**. No existen notificaciones ni plantillas huérfanas, aunque la lógica fallara.

**Matiz de la entrada por eventos.** Si además ofreces el canal genérico, un mensaje de un sistema no registrado es un fallo **permanente**, no transitorio: reintentarlo cinco veces con backoff exponencial no va a cambiar nada, porque la aplicación seguirá sin existir en el intento cinco. Va a la cola de descartados **inmediatamente, sin reintentos**. La política de reintentos por defecto trata todo fallo como transitorio, y aquí eso solo retrasa el descubrimiento del problema y llena los logs.

### 6.4 Identidad y permiso son cosas distintas

El token dice **quién eres** (`sub`) y **qué puedes hacer** (`scope`). La fila de `application` no concede permisos —eso lo hace el proveedor de identidad— sino que define **sobre qué datos operas**.

Un token con `template:write` cuya identidad no resuelve a ninguna aplicación no puede escribir nada: tiene el permiso, pero no tiene inquilino.

Por eso la tabla es la raíz de la que cuelga todo lo demás:

```
application
   ├── template          (application_id, key, locale, version)
   ├── notification      (application_id, ...)
   └── suppression       (application_id, address, ...)
```

Y de ahí salen cuatro propiedades que no se pueden tener sin ella:

- `orders-service` e `identity-service` pueden tener **ambos** una plantilla `welcome` sin pisarse — la unicidad es `(application_id, key, locale, version)`, no `key` a secas.
- Un editor de contenido de pedidos no puede tocar las plantillas de facturación.
- La deduplicación es por aplicación: dos sistemas pueden usar la misma `Idempotency-Key` sin bloquearse.
- Un rebote se registra contra quien lo provocó, y las métricas se pueden repartir.

### 6.5 La arruga: un sistema con dos credenciales

En el modelo B cada sistema consumidor tiene **dos** clientes —el de runtime y el del pipeline— y ambos tienen que resolver a la **misma** fila de `application`. Con `key` como columna única eso no cuadra. Dos salidas:

**Por convención.** `application.key` es el cliente de runtime y el servicio quita el sufijo `-ci` antes de buscar. Simple, sin tablas extra, pero es una regla implícita: el día que alguien nombre el cliente `orders-ci` en vez de `orders-service-ci`, deja de funcionar y el error no explica por qué.

**Explícita, con una tabla de credenciales.**

```sql
CREATE TABLE application_client (
    id              UUID PRIMARY KEY,
    application_id  UUID        NOT NULL REFERENCES application(id),
    client_id       VARCHAR(64) NOT NULL UNIQUE,
    purpose         VARCHAR(16) NOT NULL,   -- runtime | ci
    active          BOOLEAN     NOT NULL DEFAULT TRUE
);
```

| application | client_id | purpose |
|---|---|---|
| orders-service | `orders-service` | runtime |
| orders-service | `orders-service-ci` | ci |

La resolución pasa a ser `application_client → application`, sin convenciones de nombres. Y regala dos cosas que acabarás necesitando:

- **Rotación de credenciales sin ventana de caída**: das de alta el cliente nuevo, conviven los dos apuntando a la misma aplicación, migras el despliegue, desactivas el viejo. Con `key` única, eso obliga a un corte.
- **Varios consumidores legítimos de la misma aplicación**: un back-office y su API, o una función serverless del mismo dominio de negocio.

**Criterio:** con un solo cliente por sistema (modelo A, sin pipeline), la columna `key` basta y no añadas la tabla. En cuanto aparezca el segundo cliente, mete `application_client` — es una migración pequeña, y el `purpose` te deja además exigir que las operaciones de plantilla vengan de un cliente `ci` y los envíos de uno `runtime`, que es mínimo privilegio de verdad y no solo scopes bien puestos.

### 6.6 Quién escribe esta tabla

El **administrador de plataforma**, y es el único punto del diseño que deliberadamente no es autoservicio: dar de alta una aplicación implica verificar el dominio de envío ante el proveedor, publicar SPF y DKIM si el dominio es nuevo, y crear las credenciales máquina. Toca la reputación de envío compartida, así que pasa por alguien.

Todo lo demás —plantillas, envíos— sí es autoservicio una vez existe la fila.

### 6.7 Lo que suele añadírsele después

Dos columnas que no están en el esquema mínimo pero aparecen a los pocos meses:

- **`reply_to`** — dirección de respuesta distinta del remitente: envías desde `no-reply@` pero quieres que respondan a `soporte@`.
- **`layout_html`** — la maquetación compartida de esa aplicación: cabecera, pie, estilos y marca. Si cada plantilla lleva su HTML completo, cambiar el logo son cuarenta ediciones.

Y una tercera si el servicio crece: **`monthly_quota`**, para que un bucle infinito en un sistema no consuma el presupuesto de correo de todos. Aunque eso suele empezar siendo una alerta antes que un límite.

### 6.8 Resolución del inquilino en la entrada por eventos

Todo lo de §6.2 supone una petición HTTP con un token delante. En el canal genérico de §2.2 no hay ninguna de las dos cosas: en `client_credentials` no hay usuario pero sí token; en un broker no hay ni usuario ni token. Hay que decir qué lo sustituye, porque si no se dice cada quien supondrá una cosa distinta.

**La decisión: el inquilino sale de `metadata.source`, el campo de la envoltura del mensaje que nombra al servicio emisor.**

```json
{
  "metadata": {
    "eventId": "9f1c3b6e-2d4a-4a91-b0f2-5c7d8e0a1b23",
    "eventType": "NotificationRequested",
    "source": "orders-service",          ← el inquilino
    "occurredAt": "2026-03-14T09:21:07.482Z",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1"
  },
  "data": { "templateKey": "order-confirmed", "to": ["cliente@ejemplo.com"], "variables": { } }
}
```

```sql
SELECT * FROM application WHERE key = 'orders-service';
```

Y se sostiene sobre **dos asunciones que conviene enunciar como tales**, porque no son del mismo tipo:

1. **El emisor está autenticado ante el broker** por algún mecanismo válido. No se admite publicación anónima: el conjunto de quienes pueden poner un mensaje en ese canal son los sistemas registrados de la organización.
2. **El `source` de un emisor es siempre el nombre de su propio servicio.** Esto es una **política de desarrollo, no un control técnico**: nada en el camino lo comprueba.

#### Qué compra cada asunción y qué no

| | HTTP (§6.2) | Eventos (§6.8) |
|---|---|---|
| De dónde sale la identidad | Un claim del token (`azp`) | `metadata.source`, dentro del cuerpo |
| Quién la garantiza | El proveedor de identidad, con su firma | Una política de desarrollo |
| Qué hace falta para suplantar | Robar un token vivo | Escribir otro nombre en un campo |
| Riesgo residual | Token filtrado | Un emisor **registrado** enviando en nombre de otro |

La asunción 1 hace el trabajo pesado: acota el riesgo a quienes ya están dentro. Un atacante externo no llega al canal. La asunción 2 **no elimina lo que queda** — un sistema propio con un bug, o comprometido, puede pedir un envío en nombre de otra aplicación y salir desde su remitente verificado. Se acepta a sabiendas, y es una decisión razonable mientras todos los emisores sean sistemas nuestros en la misma malla de confianza.

Lo que **no** cambia: `applicationId` sigue sin viajar en el payload. El dato de identidad es uno solo y está en `metadata`. Ponerlo en dos sitios son dos versiones de la verdad, y tarde o temprano una de las dos deja de validarse.

#### Las cuatro puertas de §6.3 siguen aplicando

Sobre el `source` ya resuelto, y con un matiz: **aquí no hay a quién devolverle un `403`**. El llamante se fue en el momento de publicar. Cada rechazo es un descarte y una alerta.

| Situación | Qué pasa |
|---|---|
| No hay fila en `application` con ese `key` | Descarte **inmediato, sin reintentos** + alerta |
| Fila con `active = false` | Descarte + alerta |
| Falta `metadata.source`, o el mensaje no trae envoltura | Descarte: no hay inquilino que resolver |
| Fallo lógico que dejara pasar los anteriores | La FK `notification.application_id` impide insertar |

Los tres primeros son fallos **permanentes**, no transitorios: reintentar cinco veces con backoff no va a hacer aparecer una fila que no existe. Y la cuarta puerta sigue siendo la que más aguanta, porque no es una comprobación que alguien pueda olvidarse de escribir.

#### La costura, que es lo que hace barato cambiar de opinión

La resolución vive en **un único punto** —llámese `PublisherIdentityResolver`—, que recibe el mensaje y devuelve la `Application`. El caso de uso recibe la aplicación **ya resuelta**, nunca el mensaje crudo, y en ningún otro sitio del servicio se vuelve a leer `metadata.source`.

Eso importa porque hay tres mecanismos que sí dan una identidad **verificada por el broker**, y cada uno es una implementación distinta de esa misma pieza:

| Broker | De dónde saldría el inquilino | Quién lo garantiza |
|---|---|---|
| RabbitMQ | La propiedad `user_id` del mensaje | El broker la contrasta contra el usuario de la conexión y rechaza el `publish` si no coincide |
| SQS | El atributo `SenderId` | Lo estampa AWS con el principal IAM del emisor; el cliente no puede escribirlo |
| Kafka | El sufijo del topic (`notifications.requests.<app>`) | El ACL: solo ese principal puede escribir en ese topic |

Cambiar a cualquiera de ellos es sustituir el resolver y su configuración — no toca el dominio, ni los casos de uso, ni el esquema. Igual que §6.2 pide que el claim leído sea configurable, aquí lo configurable es **de dónde se toma la identidad**.

**La señal para activarla es inconfundible**: el día que un emisor deje de estar dentro del perímetro de confianza —un tercero, un sistema de otra organización, un cliente que integramos— o que alguien pida no-repudio del origen para una auditoría. Mientras todos los emisores sean sistemas propios, la política se sostiene sola.

#### Dos cosas más que arrastra el canal asíncrono

**La idempotencia cambia de fuente.** No hay cabecera `Idempotency-Key` que valga. El `dedupe_key` sale del identificador del mensaje —`metadata.eventId` con la envoltura estándar—, que el emisor estampa una vez y una reentrega repite intacto. El contrato del evento tiene que declararlo obligatorio: sin él, §2.6 promete una deduplicación que este camino no tiene, y un consumidor de eventos es *at-least-once* por definición.

**Los errores de §3 dejan de ser síncronos.** `TEMPLATE_NOT_FOUND`, `MISSING_TEMPLATE_VARIABLE` y `RECIPIENT_SUPPRESSED` no pueden devolverse: se convierten en `NotificationFailed` y en un descarte. Y son **permanentes**, por el mismo razonamiento que la aplicación no registrada — la plantilla no va a aparecer en el intento cinco.

---

## 7. Cómo un servicio usa una plantilla concreta

El recorrido completo, con `orders-service` y `order-confirmed`.

### Paso 1 — la plantilla existe y está publicada

```http
PUT /v1/templates/order-confirmed/es-ES
Authorization: Bearer <token del cliente de CI de orders>
```
```json
{
  "subject": "Tu pedido {{orderNumber}} está confirmado",
  "bodyHtml": "<h1>Gracias por tu compra, {{customerName}}</h1>...",
  "bodyText": "Gracias por tu compra, {{customerName}}...",
  "variables": [
    { "name": "orderNumber",  "required": true },
    { "name": "customerName", "required": true },
    { "name": "total",        "required": true },
    { "name": "lines",        "required": true },
    { "name": "trackingUrl",  "required": false }
  ],
  "publish": true
}
```

La aplicación no viaja en el cuerpo: sale del token.

### Paso 2 — el pedido se confirma y se pide el envío

```http
POST /v1/notifications
Authorization: Bearer <token de runtime de orders>
Idempotency-Key: order-A-1042-confirmed
```
```json
{
  "templateKey": "order-confirmed",
  "locale": "es-ES",
  "to": ["cliente@ejemplo.com"],
  "variables": {
    "orderNumber": "A-1042",
    "customerName": "Ana",
    "total": "89,90 €",
    "lines": [
      { "description": "Teclado mecánico", "quantity": 1, "unitPrice": "89,90 €" }
    ]
  }
}
```

```json
202 Accepted
{ "notificationId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9", "status": "accepted" }
```

`trackingUrl` no viaja: es opcional, y el `{{#if}}` de la plantilla simplemente no pinta ese párrafo.

Si faltara `total`, la respuesta sería `422 MISSING_TEMPLATE_VARIABLE` y no se enviaría nada.

### Paso 3 — el correo sale

El servicio busca la versión activa, la renderiza con esas variables, la envuelve en la maquetación de la aplicación, la envía desde `pedidos@tutienda.com` y publica `NotificationSent`. `orders-service` no ha visto una sola etiqueta HTML en todo el recorrido.

---

## 8. Las dos estrategias de gestión de plantillas

Aquí está la decisión de verdad. Y empieza por deshacer una confusión frecuente: **en los dos modelos la plantilla se ejecuta desde la base de datos del servicio de notificación**. El sistema llamante nunca manda HTML en ninguno de los dos casos.

|  | **Modelo A** | **Modelo B** |
|---|---|---|
| **Dónde se ejecuta** | BD del servicio de notificación | BD del servicio de notificación |
| **Dónde está el original** | *también ahí* — no existe en ningún otro sitio | en el repositorio del sistema consumidor, en git |
| **Quién la crea** | una persona, por API o back-office | un desarrollador, en un commit |
| **Cómo llega a la BD** | directamente, al crearla | la empuja el pipeline en cada despliegue |
| **Quién manda si hay conflicto** | la base de datos | git |

> **Idea clave:** la analogía exacta es la de las migraciones de base de datos. El esquema real vive en el motor, en ejecución, y nadie lo discute — pero la verdad está en los archivos de migración en git, y el motor solo refleja lo que dicen. El modelo B es eso aplicado al contenido de los correos. El modelo A es no tener migraciones y modificar el esquema a mano.

### 8.1 Modelo A: nace y vive en el servicio de notificación

Alguien la crea por API o desde un back-office, se edita ahí, y la base de datos **es** la fuente de verdad. Git no sabe nada.

Es más simple, y para arrancar suele ser lo correcto. Es la elección natural cuando:

- **El contenido lo posee negocio** y cambia por razones que no tienen nada que ver con el código: un aviso, una comunicación comercial, un texto que se ajusta según cómo responde la gente.
- **Las variables son estables.** Si `password-reset` usa `{{userName}}` y `{{resetUrl}}` y eso no va a cambiar nunca, el riesgo que justifica el modelo B es prácticamente cero.
- **Son pocas plantillas y un equipo.** La ceremonia de B no se paga sola con cuatro correos.

### 8.2 Modelo B: el original en el repositorio del consumidor

```
orders-service/
├── src/main/java/...
└── emails/
    ├── order-confirmed/
    │   ├── template.yaml        # metadatos: variables declaradas
    │   ├── subject.hbs
    │   ├── body.html.hbs
    │   └── body.txt.hbs
    └── order-cancelled/
        └── ...
```

El pipeline recorre `emails/`, pide su token máquina y empuja cada plantilla. Hay seis detalles que lo hacen funcionar o lo convierten en un dolor.

**a) No es un `INSERT`, es un «asegúrate de que existe».**
El pipeline corre en **cada** despliegue. Si la operación fuera «crea una plantilla», tendrías la versión 87 a los tres meses. Lo que necesita es un **upsert idempotente por contenido**: si el contenido es idéntico al de la versión activa, no hace nada; si difiere, crea la versión siguiente y la publica. De ahí el `"publish": true` del ejemplo — el camino de dos pasos (`draft` → publicar) sigue existiendo para el back-office humano, que sí quiere revisar antes.

**b) Va antes que el código, como una migración.**
Si despliegas el código nuevo y **después** empujas la plantilla, los correos que salgan en esa ventana fallan con `TEMPLATE_NOT_FOUND`. El empuje es un paso previo del despliegue, no un post-deploy. Y por la misma razón: **si el empuje falla, el despliegue falla**. Continuar significa poner en producción código que manda correos que no existen.

**c) El proceso en ejecución no necesita `template:write`.**
Son dos clientes máquina distintos:

| Cliente | Scopes | Quién lo usa |
|---|---|---|
| `orders-service` | `notification:send`, `notification:read` | la aplicación en ejecución |
| `orders-service-ci` | `template:write`, `template:publish` | el pipeline de despliegue |

Si le das `template:write` al proceso que corre en producción, una vulnerabilidad en `orders-service` permite reescribir el contenido de sus correos: una plataforma de phishing servida desde tu propio dominio verificado.

**d) Una copia por ambiente.**
Git tiene el original; cada instancia del servicio de notificación (desarrollo, preproducción, producción) tiene su propia copia, empujada por el pipeline de ese ambiente. No es duplicación: es despliegue. Con una ventaja práctica — en preproducción puedes trastear una plantilla a mano y el siguiente despliegue la devuelve a lo que dice git.

**e) La vuelta atrás es publicar, no borrar.**
Revertir el commit hace que el siguiente despliegue empuje el contenido antiguo, que nace como una versión **nueva** (la 5 con el contenido de la 3). No se borran versiones nunca: el histórico de qué se envió tiene que seguir siendo legible. Y para una urgencia sin esperar al pipeline, el back-office puede republicar la versión anterior en segundos.

**f) La trampa: negocio editando lo que git va a pisar.**
Este es el que muerde. Si `orders-service` está en modelo B y alguien de marketing edita el texto desde el back-office, **el siguiente despliegue lo borra sin avisar**. Nadie lo relaciona: el correo cambió el martes y volvió atrás el jueves, cuando se desplegó otra cosa. Tres salidas:

- **Git manda**: la aplicación se marca como gestionada por pipeline y el back-office deja esas plantillas en solo lectura. Limpio y sin sorpresas.
- **Detección de deriva**: el pipeline compara antes de empujar y **falla** si la versión activa no coincide con la que empujó él la última vez. Alguien resuelve el conflicto a mano, como en un merge.
- **Convivencia por plantilla**: unas gestionadas por git, otras editables. Funciona, pero exige que la marca esté en la propia plantilla y se vea en la interfaz.

De entrada, la primera. La segunda solo cuando alguien pida de verdad editar sin desplegar.

### 8.3 Las desventajas del modelo A

Casi todas se concentran en una.

**La plantilla y el código se desincronizan.** Añades `trackingUrl` al correo de confirmación. En el modelo B es un commit: el código que manda la variable nueva y la plantilla que la usa viajan juntos, se revisan juntos, se despliegan juntos y se revierten juntos.

En el modelo A son **dos acciones separadas, en dos sitios, sin nada que las coordine**. Despliegas el viernes y editas la plantilla el lunes: todo el fin de semana los clientes reciben un correo sin el enlace. O al revés — publicas la plantilla antes del despliegue y durante dos horas el correo dice «Sigue tu envío» con un enlace vacío, porque el código todavía no manda esa variable. Nada lo detecta.

De ahí salen las otras cuatro:

| Desventaja | Qué significa |
|---|---|
| **Sin revisión por pares** | No hay pull request, ni diff, ni segunda persona. Alguien edita directamente el contenido que van a leer clientes reales. Para un correo con importes o texto legal, es un cambio sin control |
| **Ambientes no reproducibles** | Producción tiene plantillas que desarrollo no tiene. Montar un entorno nuevo es volver a teclearlas. No puedes reconstruir el sistema desde git: la copia de seguridad de la BD pasa a ser la única custodia de un contenido que costó semanas |
| **Trazabilidad reducida** | En B tienes `git blame`, el mensaje del commit, el ticket y el revisor. En A todo eso hay que construirlo como registro de auditoría dentro del servicio, y aun así te quedas lejos |
| **Los tests del llamante no verifican el correo** | En B, un test de integración de `orders-service` puede sembrar la plantilla en un servicio de notificación local y afirmar sobre el HTML renderizado. En A la plantilla no está en el repositorio: no hay nada que sembrar, y ese escenario deja de ser comprobable |

---

## 9. Quién crea y quién publica

Un correo transaccional tiene dos dueños que no son la misma persona: **el sistema** decide cuándo se manda y qué datos lleva; **alguien de negocio** decide qué dice y cómo se ve. Si solo contemplas al primero, cambiar una coma es un despliegue. Si solo contemplas al segundo, dar de alta un sistema pasa por una persona y vuelve el cuello de botella.

En la práctica conviven tres actores:

| Actor | Cómo se autentica | Permisos | Qué hace |
|---|---|---|---|
| **Admin de plataforma** | persona, rol admin | `application:write` y el resto | Da de alta aplicaciones, verifica el dominio de envío, crea credenciales, fija el remitente por defecto. Una vez por sistema consumidor |
| **El pipeline del consumidor** | cliente máquina de CI | `template:write`, `template:publish` | Empuja las plantillas del repositorio en cada despliegue (modelo B) |
| **Editor de contenido** | persona, adscrita a aplicaciones | `template:read`, `template:write`, `template:publish` según política | Ajusta textos sin desplegar (modelo A) |
| **La aplicación en ejecución** | cliente máquina de runtime | `notification:send`, `notification:read` | Solo pide envíos. **Nunca escribe plantillas** |

Dos observaciones que cuestan poco ahora y mucho después:

**Crear y publicar no son el mismo acto.** Registrar una plantilla no hace nada: nace en `draft`. Publicarla la pone delante de clientes reales, inmediatamente. Separar `template:write` de `template:publish` permite que la organización decida más adelante si el mismo actor hace las dos cosas (lo normal al principio) o si exige cuatro ojos para el paso en vivo (lo normal cuando el correo lleva importes o condiciones legales). Separarlos hoy cuesta una línea en el catálogo de permisos; separarlos después es un cambio incompatible, porque los clientes ya tienen concedido un scope que pasaría a significar menos.

**El aislamiento entre aplicaciones no lo da el rol.** Quien edita las plantillas de pedidos no puede poder tocar las de recuperación de contraseña de otro sistema. Los roles y permisos son globales al servicio; «un editor solo opera sobre las aplicaciones a las que está adscrito» es una **invariante de dominio**, y la hace cumplir el caso de uso. Conviene verlo claro, porque es de esas cosas que se asumen resueltas por el token y no lo están.

**Y lo que hace seguro editar**, en cualquiera de los dos modelos: una operación de **previsualización** que renderiza con variables de muestra y devuelve el HTML sin enviar, y un envío de prueba a una dirección propia antes de publicar. Es donde se descubre que Outlook rompe la tabla.

---

## 10. Desarrollo local: probar los correos sin mandar ninguno

Todo lo anterior es diseñable sin contratar nada. La pieza que lo permite es una imagen de Docker.

### 10.1 Mailpit

```yaml
# docker-compose.yaml
services:
  mailpit:
    image: axllent/mailpit:latest
    ports:
      - "1025:1025"    # SMTP  — aquí apunta la aplicación
      - "8025:8025"    # HTTP  — interfaz web y API REST
    environment:
      MP_MAX_MESSAGES: 500
```

**Es un servidor SMTP falso.** Se comporta como uno real —acepta la conexión, la autenticación y el mensaje— pero **no entrega nada a nadie**: se lo queda y te lo enseña. Es el sucesor de MailHog, que hacía lo mismo pero está prácticamente abandonado.

### 10.2 Qué función cumple

| Función | Por qué importa |
|---|---|
| **Sustituye al proveedor** | Desarrollas, generas y demuestras el servicio completo sin contratar SES, Brevo ni nada. Coste cero |
| **Muestra el HTML renderizado** | La interfaz de `:8025` enseña el correo tal cual llegaría, con sus cabeceras, sus dos partes (HTML y texto) y sus adjuntos. Es donde se itera una plantilla |
| **Expone una API REST** | Lo verdaderamente valioso: convierte «se envía el correo correcto» en algo que un test automático puede afirmar |
| **Evita accidentes** | Ningún correo de pruebas puede llegar a una dirección real. Con un proveedor de verdad en desarrollo, eso pasa el primer día |
| **Puntúa spam y enlaces** | Avisa si el HTML tiene pinta de acabar en la carpeta gris, y detecta enlaces rotos |

> **Idea clave:** la API de `:8025` es lo que separa «lo he mirado y se ve bien» de una prueba de regresión. Sin ella, la verificación del correo es siempre manual.

### 10.3 Cómo se verifica un escenario

```
Dado    una plantilla `order-confirmed` activa en es-ES
Cuando  se solicita una notificación con orderNumber=A-1042
Entonces Mailpit tiene un mensaje para cliente@ejemplo.com
         cuyo asunto es "Tu pedido A-1042 está confirmado"
```

El test consulta `GET http://localhost:8025/api/v1/messages`, encuentra el mensaje y comprueba asunto y cuerpo. **Correo real por SMTP real, aserción real**, sin salir a Internet.

Y es lo que hace comprobable el escenario del modelo B descrito en 8.3: el test del sistema consumidor siembra su plantilla en un servicio de notificación local y afirma sobre el HTML renderizado.

### 10.4 De local a producción: solo variables de entorno

La configuración sigue el mismo gradiente que el resto del servicio — literal en local, variable obligatoria en producción:

```yaml
# parameters/local/mail.yaml — apunta al contenedor
spring:
  mail:
    host: localhost
    port: 1025
    properties:
      mail.smtp.auth: false
      mail.smtp.starttls.enable: false

# parameters/production/mail.yaml — apunta al proveedor contratado
spring:
  mail:
    host: ${MAIL_HOST}
    port: ${MAIL_PORT}
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
    properties:
      mail.smtp.auth: true
      mail.smtp.starttls.enable: true
```

Cambiar de Brevo a SES en producción es cambiar cuatro variables y reiniciar. Mismo binario, sin recompilar. En producción esas variables no se escriben a mano: las inyecta un Secret de Kubernetes, Vault o un gestor de secretos — pero el consumo sigue siendo por variable de entorno, así que el patrón no cambia.

### 10.5 Lo que Mailpit no cubre

Tres cosas que solo se prueban contra un proveedor real, y conviene no descubrirlas tarde:

- **El webhook de rebotes y quejas.** Mailpit no rebota nada, así que la lista de supresión y el `recordDeliveryFeedback` no se ejercitan. Hay que simularlos invocando el endpoint con un payload de ejemplo.
- **La entregabilidad.** Que el correo salga no es que llegue a la bandeja de entrada. SPF, DKIM, DMARC y la reputación del remitente son trabajo de DNS y de proveedor, no de código, y no tienen equivalente local.
- **Los límites del proveedor.** Tamaño máximo del mensaje (10–25 MB según cuál, y los adjuntos viajan en base64, que infla un 33 %) y máximo de envíos por segundo. Mailpit lo acepta todo.

---

## 11. Cómo elegir

**Empieza por el modelo A.** Es más simple, no exige pipeline, y las desventajas solo se materializan cuando las plantillas empiezan a depender de la forma del código.

**La señal para pasar a B es inconfundible:** la primera vez que un correo se rompe porque alguien desplegó código y olvidó la plantilla. Esa aplicación pasa a B; las demás siguen donde estén.

Y como criterio de fondo, para no dudar en cada caso:

| Si el correo… | Modelo |
|---|---|
| cambia cuando cambia la funcionalidad que lo dispara | **B** — que viajen en el mismo commit |
| lleva importes, condiciones legales o cualquier cosa que exija revisión | **B** — el pull request es la revisión |
| lo escribe y ajusta negocio, con su propio ciclo | **A** — obligarles a un despliegue es absurdo |
| usa dos variables que no van a cambiar nunca | **A** — B no compra nada aquí |

Lo que quita hierro a toda la decisión son dos hechos:

**El servicio es idéntico en ambos modelos.** Misma API, mismas tablas, mismo ciclo de vida. La diferencia está entera fuera: en si existe o no un pipeline que empuja. No estás eligiendo una arquitectura, estás eligiendo una práctica de trabajo.

**Y migrar de A a B cuesta una tarde.** Exportas las plantillas que ya existen, las metes en el repositorio del sistema que las usa, enciendes el paso del pipeline. No hay ninguna puerta que se cierre.

> **Idea clave:** puedes tener `orders-service` en modelo B e `identity-service` en modelo A al mismo tiempo. No es una política global del servicio de notificación — es una convención por aplicación.
