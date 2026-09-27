# Seguridad y autorización: roles, permisos y scopes (método Keel)

**Respuesta corta**, y si te quedas solo con esto ya has desactivado el 80% de la confusión:

- Un **permiso** es una **capacidad**: `product:write`. Es lo único que el código comprueba de verdad.
- Un **rol** es un **nombre para un paquete de permisos**: `catalog-admin`. No tiene contenido propio; el contenido está en la lista de permisos que se le otorgan, y esa lista vive **fuera** del rol.
- Un **scope** **no es un tercer concepto**: es *el mismo permiso*, exigido a una **credencial de máquina** en vez de a una persona.

Keel llega a esa conclusión y la vuelve estructura: en la capa `security` hay **un solo catálogo, `permissions`**, y las dos ramas —humana y máquina— tiran de él.

> **Idea clave:** la pregunta "¿esto es un rol, un permiso o un scope?" casi siempre está mal planteada. Las preguntas útiles son otras dos: **¿qué capacidad es?** (eso es siempre un permiso) y **¿a quién se la exijo, a una persona o a un proceso?** (eso decide si la escribes en `permissions:` o en `scopes:`).

Los tres planos en los que vive la seguridad de un servicio Keel:

| Plano | Dónde | Pregunta que responde |
|---|---|---|
| **El diseño** | `specs/<servicio>/security.keel.yaml` | ¿qué capacidades existen y quién puede invocar cada operación? |
| **El código** | `SecurityConfig`, `JwtAuthConverter`… (los genera `keel-spring build`) | ¿cómo se comprueba eso en cada petición? |
| **El entorno** | `infra/init-keycloak.sh`, `infra/test-credentials.env`, `deploy/keycloak/realm-export.json` | ¿de dónde salen los tokens con los que se prueba? |

Los tres salen del mismo archivo. Ese es el argumento entero de esta nota.

Notas hermanas: [`capas-http-clients-y-messaging.md`](capas-http-clients-y-messaging.md) para el canal saliente (la autenticación **de salida**, que es otra capa y otro problema) y [`../keycloak.md`](../keycloak.md) para el detalle del proveedor.

---

# Parte I — Los conceptos, desde cero

## 1. Autenticar no es autorizar

Son dos preguntas distintas y en ese orden:

| | Pregunta | Respuesta si falla | Quién la contesta |
|---|---|---|---|
| **Autenticación** (*authn*) | ¿**Quién** eres? | `401 Unauthorized` | El **proveedor de identidad** (IdP): Keycloak, Cognito, Auth0… |
| **Autorización** (*authz*) | ¿**Qué puedes** hacer? | `403 Forbidden` | **Tu servicio** |

El nombre HTTP de `401` es un accidente histórico desafortunado: dice "Unauthorized" pero significa "no autenticado". Ténlo interiorizado porque es la fuente de la mitad de los bugs de seguridad mal diagnosticados:

- **`401`** = no sé quién eres. No mandaste token, o el token está caducado, o la firma no cuadra.
- **`403`** = sé perfectamente quién eres, y **no puedes**.

> **Idea clave:** si estás depurando un `403`, el token es válido. No toques la validación del token: el problema está en los permisos. Y al revés: un `401` nunca se arregla dando más permisos.

Keel toma esta distinción tan en serio que la convierte en regla de arquitectura del código generado. Verás en la Parte III un filtro entero (`AudienceAuthorizationFilter`) que existe **solo** para que un caso concreto devuelva `403` y no `401`, aunque técnicamente pudiera implementarse de las dos formas.

## 2. El token: quién emite y quién decide

El flujo, con los nombres de los actores:

```
                      (1) credenciales
   Usuario/Servicio ─────────────────────► Proveedor de identidad (Keycloak)
        │                                            │
        │  ◄──────────────────────────────────────────┘
        │        (2) access token (JWT firmado)
        │
        │  (3) GET /api/v1/products
        │      Authorization: Bearer eyJhbGc...
        ▼
   Tu servicio  ──(4) verifica la firma con la clave pública del IdP
                ──(5) lee los claims del token
                ──(6) decide: 200 / 403
```

Un **JWT** (JSON Web Token) es, sin misterio, tres bloques en base64 separados por puntos: cabecera, **payload** y firma. El payload es un JSON con **claims** (afirmaciones). Algo así, simplificado:

```json
{
  "sub": "9f1c...",
  "preferred_username": "catalog-admin",
  "iss": "http://localhost:8180/realms/product-service",
  "aud": ["product-service"],
  "exp": 1755100000,
  "realm_access": { "roles": ["catalog-admin"] },
  "scope": "product:read"
}
```

Dos cosas que hay que tener muy claras:

**(a) El token no está cifrado, está firmado.** Cualquiera puede leer su contenido (pégalo en jwt.io y lo verás en claro). Lo que la firma garantiza es que **nadie lo ha modificado** y que lo emitió quien dice. Corolario: nunca metas secretos en un JWT.

**(b) Tu servicio no llama al IdP en cada petición.** Sería un cuello de botella y un punto único de fallo. Lo que hace es descargar **una vez** (y cachear) el juego de claves públicas del IdP —el **JWKS**, que se publica en una URL tipo `/protocol/openid-connect/certs`— y con eso verifica la firma **localmente**, en microsegundos. Por eso un token robado sigue valiendo hasta que caduca: no hay nadie a quien preguntar "¿este sigue siendo válido?".

Eso explica por qué los access tokens duran poco (minutos, no días) y por qué el refresh token existe.

> **Idea clave:** el IdP **afirma**; tu servicio **decide**. El IdP dice "este usuario tiene el rol `catalog-admin`". Es tu servicio el que decide si `catalog-admin` puede borrar un producto. Esa frontera es exactamente la que la capa `security` de Keel materializa: los claims son del IdP, la tabla de decisiones es del diseño.

## 3. Rol, permiso y scope, con una analogía sostenida

Imagina un edificio de oficinas.

**Un permiso es una puerta concreta.** "Abrir el almacén." "Entrar al servidor." "Firmar pedidos." Es atómico, es literal, y es lo que el lector de la puerta comprueba. En Keel se escribe siempre en formato `recurso:accion`:

```yaml
permissions:
  product:write: { description: Crear y mutar productos. }
  product:read:  { description: Leer productos. }
```

**Un rol es un modelo de tarjeta.** "Tarjeta de encargado de almacén." La tarjeta en sí no abre nada: lo que abre puertas es la **lista de puertas asociada a ese modelo de tarjeta**. Y esa lista es una decisión administrativa, no una propiedad de la tarjeta:

```yaml
roles:
  catalog-admin:  { description: Gestiona el catálogo completo. }
  catalog-reader: { description: Solo consulta el catálogo. }

roleGrants:                                # ← la lista de puertas por modelo de tarjeta
  catalog-admin: [product:write, product:read]
  catalog-reader: [product:read]
```

Fíjate en algo que en Keel es literal y muy deliberado: **el rol no contiene sus permisos**. En el schema, un rol solo admite `description`, nada más. La conexión rol→permisos vive en un bloque aparte, `roleGrants`. No es capricho: si un día `catalog-reader` necesita también `product:export`, cambias **una línea de `roleGrants`** y no tocas ni el catálogo de roles ni ninguna regla de acceso. El rol es una etiqueta estable; el paquete que representa es lo que evoluciona.

**Un scope es un permiso concedido a una llave, no a una persona.** Aquí es donde la analogía se afina. Imagina que contratas a una empresa de limpieza y le das una llave que abre **solo** el pasillo y los baños, a las 3 de la mañana. Esa llave no tiene "rol": no es encargado de nada. Tiene un conjunto **explícito y mínimo** de puertas, y punto.

Eso es un scope: cuando el que llama no es una persona sino **otro servidor**, no hablamos de roles (un proceso no tiene un puesto de trabajo), hablamos directamente de las capacidades exactas que se le conceden a su credencial.

```yaml
serviceClients:
  billing-service:
    description: Consulta precios para facturar.
    scopes: [product:read]          # ← la llave abre exactamente esta puerta
```

Y aquí está el remate, que es la decisión de diseño más importante de toda la capa: **`product:read` es la misma puerta en los dos casos.** Un permiso y un scope no son dos cosas parecidas: son **la misma capacidad**, mirada desde dos tipos de portador de credencial.

La tabla seca:

| | Qué es | A quién se aplica | Cómo llega al servicio | Palabra en el DSL |
|---|---|---|---|---|
| **Permiso** | Una capacidad atómica `recurso:accion` | Persona **o** máquina | Claim `permissions`, o derivado del rol | `permissions:` (catálogo) y `permissions: [...]` (en una regla) |
| **Rol** | Un nombre para un paquete de permisos | Solo personas | Claim `realm_access.roles` | `roles:` |
| **Scope** | Una capacidad concedida a una credencial de máquina | Solo máquinas | Claim `scope` | `scopes: [...]` |

## 4. ¿Por qué hacen falta los tres? ¿No bastaría con uno?

Es la pregunta correcta, y la respuesta es que cada uno resuelve un problema que los otros no.

**Si solo tuvieras roles**, tu código estaría lleno de `if (rol == "catalog-admin" || rol == "supervisor" || rol == "auditor")`. Cada vez que la organización crea un puesto nuevo, hay que **tocar código y desplegar**. El rol es una etiqueta organizativa, y las organizaciones cambian mucho más rápido que el software.

**Si solo tuvieras permisos**, el código quedaría perfecto (`if (puede("product:write"))`) pero la administración sería un infierno: dar de alta a un empleado significaría marcar 40 casillas a mano, y nadie recordaría cuáles. El rol existe precisamente para eso: es la **unidad de gobernanza**, el paquete que un administrador entrega de una vez.

**Los scopes resuelven un problema distinto de los dos anteriores**, y es el que más cuesta ver. Roles y permisos responden a *"¿qué puede este sujeto?"*. El scope responde a *"¿qué puede **esta credencial concreta**?"*, que no es lo mismo.

Piénsalo así: `billing-service` podría necesitar leer y escribir productos en general, pero **la llave que le diste para el proceso nocturno de facturación** solo debería poder leer. Si esa llave se filtra en un log, lo que se filtra es la capacidad de leer precios, no la de destruir el catálogo. El scope es **mínimo privilegio aplicado a la credencial**, no al sujeto.

> **Idea clave:** rol = *gobernanza* (cambias el paquete sin tocar código). Permiso = *lo que el código comprueba*. Scope = *mínimo privilegio de la credencial*. Los tres coexisten porque resuelven tres problemas que no se solapan.

## 5. Personas y máquinas: los dos grants

En OAuth2, la forma de conseguir un token se llama **grant**. Para lo que nos ocupa solo hay dos que importan:

| | **Password grant** (usuarios) | **Client credentials** (máquinas) |
|---|---|---|
| Quién lo usa | Una persona, a través de una app | Un servidor, sin nadie delante |
| Qué presenta | usuario + contraseña | `client_id` + `client_secret` |
| Qué representa el token | A **una persona** | A **un programa** |
| Claim de identidad | `preferred_username`, `sub` | `client_id`, `sub` (la cuenta de servicio) |
| Lleva roles | Sí | **No** — y esto es lo importante |
| Lleva scopes | Puede | Sí, es su forma natural |

Nota práctica: el password grant está en desuso para aplicaciones reales de usuario (lo correcto hoy es *authorization code* + PKCE, con un navegador de por medio). Pero en el contexto de las **pruebas automatizadas** sigue siendo la herramienta correcta y por eso Keel lo usa en los tests de integración: es la única forma de que un test JUnit obtenga un token de usuario sin abrir un navegador.

Que una máquina **no tenga roles** no es una limitación técnica: es semántica. `billing-service` no es "administrador del catálogo"; no tiene puesto de trabajo, tiene una tarea. Keel convierte esa observación en una **regla dura de validación**: declarar `level: service` junto con `roles` es un **error** de `keel validate`, con este mensaje:

```
level 'service' no admite roles (los roles son de usuarios humanos)
```

## 6. La audiencia: por qué un token válido puede no valer aquí

Escenario que hay que entender antes de seguir. Tienes un solo Keycloak sirviendo a cinco microservicios. `billing-service` pide un token para hablar con `inventory-service`. Ese token está **firmado por el mismo IdP** que valida tu `product-service`. ¿Qué impide que `billing-service` lo reutilice contra ti?

La firma no. La firma es válida: la emitió el IdP correcto. La caducidad tampoco. Si tu servicio solo comprueba "firma OK + no caducado", **ese token entra**.

Lo que lo impide es el claim **`aud`** (*audience*): para quién se emitió el token.

```yaml
authentication:
  protocol: oidc
  serviceAuth:
    protocol: client-credentials
    validateAudience: true          # ← exige que aud incluya mi audiencia
    audience: product-service       # ← opcional; por defecto, el nombre del servicio
```

Con `validateAudience: true`, un token emitido para otro servicio se rechaza aunque todo lo demás cuadre.

> **Idea clave:** en un sistema con un IdP compartido, **la firma válida no significa "para mí"**. Sin `validateAudience`, cualquier servicio del ecosistema que pueda obtener un token puede llamarte, y con los scopes que le hayan dado en otro sitio. Es el fallo de seguridad más común y más silencioso de las arquitecturas M2M: no rompe nada, solo deja la puerta abierta.

Y una sutileza que la Parte III retoma: cuando ese token llega, ¿es `401` o `403`? Keel dice **`403`**, y el argumento es limpio: el token es auténtico y sabemos perfectamente quién lo trae. Lo que pasa es que no le autoriza a hablar con nosotros. Eso es autorización, no autenticación.

## 7. Dónde acaba esta parte

Para el detalle del proveedor —tipos de cliente en Keycloak, mappers de protocolo, federación, sesiones, alta disponibilidad— tienes ya [`../keycloak.md`](../keycloak.md), que cubre eso a fondo. A partir de aquí, la nota va sobre **cómo Keel declara todo lo anterior y qué código sale de esa declaración**.

---

# Parte II — Cómo lo declara Keel: la capa `security`

## 8. Dónde encaja

`security` es una capa **opcional** del DSL, un archivo por servicio:

```yaml
# specs/<servicio>/service.keel.yaml
layers:
  domain:    domain.keel.yaml
  use-cases: use-cases.keel.yaml
  api:       api.keel.yaml
  security:  security.keel.yaml      # ← opcional
```

El orden de diseño no es arbitrario:

```
domain ──► use-cases ──► api ──► security
```

Se diseña **la última de las capas de superficie** porque depende de las anteriores: no puedes decidir quién invoca `createProduct` antes de que `createProduct` exista, y no puedes decidir la audiencia de un endpoint antes de que el endpoint exista.

Es opcional, pero hay dos empujones fuertes hacia declararla:

- Si hay capa `api` sin capa `security`, `keel validate` avisa:
  > `Hay capa api pero no capa security: todos los endpoints quedarían sin regla de acceso explícita`
- Si `persistence` declara `audit.authorship` distinto de `none`, la capa `security` pasa a ser **obligatoria** (error duro): sin principal autenticado no hay autor que estampar en `@CreatedBy`/`@LastModifiedBy`.

## 9. Anatomía completa de la capa

Las claves de primer nivel son **exactamente estas siete**, y no hay más (`additionalProperties: false`):

| Clave | Obligatoria | Qué declara |
|---|---|---|
| `authentication` | **Sí** | Protocolo, dónde viaja el token, y la sub-sección M2M |
| `access` | **Sí** | La regla por defecto y las excepciones por operación |
| `roles` | No | Catálogo de roles |
| `permissions` | No | Catálogo de capacidades |
| `roleGrants` | No | Qué permisos otorga cada rol |
| `serviceClients` | No | Catálogo de servicios consumidores y sus scopes |
| `cors` | No | Política de acceso desde navegador |

Cosas que **no existen** y que la gente busca por costumbre: no hay `providers` (el producto se elige al generar, no al diseñar), no hay `policies`, y **no hay catálogo de `scopes`** (§10).

El ejemplo canónico completo:

```yaml
authentication:
  protocol: oidc                 # oidc | jwt | api-key | none
  tokenLocation: header          # header | cookie

roles:
  catalog-admin:  { description: Gestiona el catálogo completo. }
  catalog-reader: { description: Solo consulta el catálogo. }

permissions:
  product:write: { description: Crear y mutar productos. }
  product:read:  { description: Leer productos. }

roleGrants:
  catalog-admin: [product:write, product:read]
  catalog-reader: [product:read]

access:
  default: { level: required, permissions: [product:read] }
  rules:
    createProduct: { level: required, permissions: [product:write] }
    retireProduct: { level: admin, roles: [catalog-admin] }
    listProducts:  { level: public }
```

Las reglas de forma que conviene saber de memoria:

| Regla | Patrón exacto |
|---|---|
| Nombre de rol | kebab-case: `^[a-z][a-z0-9-]*$` |
| Nombre de permiso | `recurso:accion`, ambos kebab-case: `^[a-z][a-z0-9-]*:[a-z][a-z0-9-]*$` |
| Clave de `access.rules` | **nombre de operación**, camelCase: `^[a-z][A-Za-z0-9]*$` |
| Un rol admite | **solo** `description` (mínimo 5 caracteres) |
| Un permiso admite | **solo** `description` |

Y el `protocol`, que es un enum cerrado y **agnóstico de producto**:

| `protocol` | Significado |
|---|---|
| `oidc` | OpenID Connect completo (el habitual con Keycloak, Cognito, Auth0) |
| `jwt` | JWT validado por firma, sin el discovery de OIDC |
| `api-key` | Clave estática en cabecera. Simple, sin identidad rica |
| `none` | Sin autenticación |

> **Idea clave:** el diseño **nunca nombra el producto**. Dice `oidc`, no "Keycloak". El proveedor concreto se elige en el cuestionario de stack al generar, y el mismo `security.keel.yaml` sirve para Keycloak, Cognito o lo que venga. Esa frontera es lo que permite que un diseño publicado en el registry sea adoptable por alguien con otro IdP.

Un ejemplo real de un servicio distinto (el fixture `asset-vault`, un custodio de archivos), para ver la misma estructura con otra silueta:

```yaml
# La capa que `audit.authorship: all` exige: sin principal autenticado no hay autor
# que estampar en @CreatedBy/@LastModifiedBy.
authentication:
  protocol: oidc
  tokenLocation: header
  serviceAuth:
    protocol: client-credentials
    description: El servicio de renderizado consulta fichas para generar sus miniaturas.

roles:
  vault-admin:  { description: Custodia archivos y decide qué se publica. }
  vault-reader: { description: Solo consulta los archivos custodiados. }

permissions:
  asset:write: { description: Custodiar archivos y publicarlos. }
  asset:read:  { description: Consultar los archivos custodiados. }

roleGrants:
  vault-admin:  [asset:write, asset:read]
  vault-reader: [asset:read]

serviceClients:
  rendering-service:
    description: Genera las miniaturas de los archivos publicados.
    scopes: [asset:read]

access:
  default: { level: required }
  rules:
    uploadAsset:  { level: required, permissions: [asset:write] }
    publishAsset: { level: required, permissions: [asset:write] }
    getAsset:     { level: required, permissions: [asset:read], scopes: [asset:read] }
    listAssets:   { level: required, permissions: [asset:read] }
```

Fíjate en `getAsset`: es el único con `permissions` **y** `scopes` a la vez. Ese es el caso "cualquiera de": lo puede invocar un usuario con el permiso, **o** una máquina con el scope. Volvemos a él en §12.

## 10. La decisión central: por qué no hay catálogo de scopes

Esta es la línea del DSL que resuelve la confusión de la que parte esta nota:

> **Los scopes reutilizan el catálogo `permissions`**: `permissions` es el catálogo único de capacidades del servicio; `accessRule.permissions` las exige a usuarios humanos y `scopes` a clientes máquina. **No hay catálogo de scopes aparte.**

En el schema esto es literal: tanto `accessRule.scopes` como `serviceClients.<x>.scopes` referencian el tipo `permissionName`, el mismo que `accessRule.permissions`. Y `keel validate` lo comprueba con un mensaje que lo dice sin ambigüedad:

```
el scope 'product:read' no existe en security: permissions
```

Un scope que no está en `permissions` **no compila**. No es que se acepte y no se use: es un error.

Seguir el mismo permiso por las dos ramas es el mejor ejercicio para fijar la idea:

```
                       permissions:
                         product:read      ← EL CATÁLOGO (una sola entrada)
                              │
              ┌───────────────┴───────────────┐
              │                               │
        RAMA HUMANA                      RAMA MÁQUINA
              │                               │
   roleGrants:                     serviceClients:
     catalog-reader:                 billing-service:
       - product:read                  scopes: [product:read]
              │                               │
   El usuario recibe el rol        El cliente recibe el scope
   'catalog-reader'                en su credencial
              │                               │
   Token:                          Token:
     realm_access.roles:             scope: "product:read"
       ["catalog-reader"]                     │
              │                               │
              ▼                               ▼
   authority: product:read         authority: SCOPE_product:read
              │                               │
              └───────────────┬───────────────┘
                              ▼
            access.rules.getProduct exige la capacidad
              permissions: [product:read]   (humanos)
              scopes:      [product:read]   (máquinas)
```

Dos ramas, un catálogo, dos formas de exigir la misma capacidad. Cuando alguien te pregunte "¿esto lo pongo en permissions o en scopes?", la respuesta es: **en las dos, si quieres que lo puedan hacer los dos**. Y no es duplicación, porque el nombre de la capacidad es uno solo.

## 11. `access`: por operación, nunca por ruta

```yaml
access:
  default: { level: required, permissions: [product:read] }
  rules:
    createProduct: { level: required, permissions: [product:write] }
    retireProduct: { level: admin, roles: [catalog-admin] }
    listProducts:  { level: public }
```

- `default` cubre **toda** operación que no tenga regla explícita. Es obligatorio: no hay forma de dejar operaciones sin cubrir por olvido.
- `rules` sobrescribe **por operación**, y la clave es el **nombre de la operación** de `use-cases`, no la ruta HTTP.

Ese detalle es una decisión con consecuencias. Las rutas cambian: `/api/v1/products/{id}/retire` se convierte en `/api/v2/catalog/products/{id}:retire` en un rediseño de API, y si tus reglas de acceso estuvieran escritas contra rutas, **cada rediseño de API sería un rediseño de seguridad**, con el riesgo obvio de que un matcher quede huérfano y una operación se destape.

El nombre de la operación es estable porque es un concepto de negocio, no de transporte. Y `keel validate` lo ata:

```
security: access.rules.retireProducts: la operación no existe en use-cases
```

Un typo en el nombre no pasa. Es la clase de error que en un `SecurityFilterChain` escrito a mano sería un `.requestMatchers("/api/v1/product")` (singular) silenciosamente inútil.

> **Idea clave:** además, esto significa que la capa `use-cases` **no declara nada de seguridad**. Cero campos. La operación no sabe quién la puede llamar; es `security` quien la referencia. Una operación es una capacidad de negocio, y quién puede ejercerla es una política que cambia por otros motivos y a otro ritmo.

Los cuatro niveles:

| `level` | Significado | Nota |
|---|---|---|
| `public` | Sin token | Cuestiónatelo dos veces en cualquier mutación |
| `required` | Token válido | El caso normal; se afina con `roles`/`permissions` |
| `admin` | Token con privilegio elevado | Sin `roles` explícitos, el generador lo traduce a `hasRole("admin")` |
| `service` | Token de cliente máquina | **No admite `roles`** |

## 12. La matriz `audience` × `level`

La capa `api` declara el **público** de cada endpoint, y es lo único de seguridad que vive allí:

```yaml
# api.keel.yaml
defaultAudience: users              # default global
endpoints:
  getProductPrice:
    audience: services              # users | services | both
```

Cruzarla con el `level` de `security` da la matriz de combinaciones válidas:

| `audience` | Niveles válidos | Notas |
|---|---|---|
| `users` (default) | `public`, `required`, `admin` | `scopes` **prohibido** |
| `services` | `service` (con `scopes`), `public` | `roles` **prohibido** con `service` |
| `both` | `required` (opcionalmente `scopes` + `roles`/`permissions`, semántica "cualquiera de"), `public` | `service` sería **error**: excluiría a los usuarios |

La casilla que más cuesta y que más vale la pena entender es la última. **¿Por qué `both` + `service` es un error?**

Porque `level: service` significa "solo credenciales de máquina". Si el endpoint es `both`, estás diciendo a la vez "lo usan personas y máquinas" y "solo máquinas". La contradicción no es teórica: el `SecurityFilterChain` resultante rechazaría a todos los usuarios humanos, y lo descubrirías en producción. `keel validate` lo corta antes:

```
level 'service' con audience 'both' excluiría a los usuarios — usa level required con scopes y roles/permissions
```

La forma correcta de `both` es la de `getAsset` del §9: `level: required` (basta con estar autenticado, seas quien seas) más `permissions` **y** `scopes`, que se evalúan como **"cualquiera de"** — el usuario entra por el permiso, la máquina por el scope.

Las demás reglas que cruzan las dos capas, todas comprobadas mecánicamente:

| Situación | `keel validate` |
|---|---|
| `level: service` con `audience: users` | **error** — decláralo `audience: services` |
| `audience: services` con `level: required`/`admin` | **error** — audiencia humana en endpoint de máquinas |
| `scopes` en una regla que no es `service` ni su endpoint es `both` | **error** |
| Endpoints `services`/`both` sin `authentication.serviceAuth` | **error** |
| `serviceClients` sin `serviceAuth` | **error** |
| `audience: services` con `level: public` | *aviso* — ¿de verdad no requiere credencial? |
| `level: service` sin `scopes` | *aviso* — cualquier cliente autenticado podrá invocarla |

## 13. Máquina a máquina: entrante

Tres piezas, y las tres tienen que estar:

```yaml
authentication:
  protocol: oidc
  serviceAuth:                    # (1) CÓMO se autentican las máquinas
    protocol: client-credentials  #     client-credentials | api-key
    validateAudience: true
    audience: product-service

serviceClients:                   # (2) QUIÉNES son y QUÉ se les concede
  billing-service:
    description: Consulta precios para facturar.
    scopes: [product:read]

access:
  rules:
    getProductPrice:              # (3) QUÉ se exige en cada operación
      level: service
      scopes: [product:read]
```

Lo que hace que este trío sea robusto es que `keel validate` cierra el círculo por los dos lados:

- Un scope **concedido** a un cliente que **ninguna regla exige** → aviso: `el scope 'x' no lo exige ninguna regla de acceso`. Privilegio sobrante.
- Un scope **exigido** por una regla que **ningún cliente tiene** → aviso: `ningún cliente podría invocar esas operaciones`. Puerta que nadie puede abrir.

> **Idea clave:** ese par de avisos es la implementación mecánica del mínimo privilegio. No es que alguien revise la lista: es que conceder de más y exigir de más son **dos deriva detectables**, y la CLI las detecta.

### No confundir con el M2M **saliente**

Esto es una fuente de confusión real y merece su propio recuadro. La capa `security` habla **solo de quién entra**. Cuando **tú** llamas a otro servidor y necesitas autenticarte contra él, eso vive en `http-clients`, que es otra capa entera:

```yaml
# http-clients.keel.yaml — autenticación SALIENTE
clients:
  pricing:
    auth:
      type: oauth2-client-credentials   # none | api-key | bearer-static | basic | oauth2-client-credentials
      tokenUrl: https://idp.proveedor/oauth/token
      scopes: [prices.read]             # ← strings LIBRES: son los del proveedor ajeno
```

| | `security` (entrante) | `http-clients.auth` (saliente) |
|---|---|---|
| Pregunta | ¿quién puede llamarme? | ¿cómo me presento yo? |
| Los `scopes` son | `permissionName` de **mi** catálogo | Strings libres del **proveedor ajeno** |
| Se valida contra | Mi `permissions` | Nada — no son míos |

Y una advertencia del propio schema que conviene repetir: **las credenciales jamás van en el diseño**. Ni ahí ni en ningún sitio de `specs/`. El diseño declara el mecanismo; el secreto vive en configuración. Detalle completo en [`capas-http-clients-y-messaging.md`](capas-http-clients-y-messaging.md).

## 14. CORS: lo que se declara y lo que no

```yaml
cors:
  description: Consumido desde el navegador por la SPA de back-office.
  allowCredentials: false
  allowedHeaders: [Authorization, Content-Type, Idempotency-Key]
  exposedHeaders: [X-Correlation-Id]
  maxAgeSeconds: 3600
```

La presencia del bloque **es en sí misma la declaración**: "este servicio se consume desde un navegador". Sin él, el servidor generado rechaza toda petición cross-origin (el preflight muere en la cadena de seguridad).

Lo que llama la atención es lo que **no** está:

- **No hay `allowedOrigins`.** Los orígenes son URLs de despliegue: cambian por ambiente y no deben obligar a regenerar código. El generador los expone como configuración — en Spring, la variable `SECURITY_CORS_ALLOWED_ORIGINS`, obligatoria fuera de local.
- **No hay `allowedMethods`.** Se **derivan** de los endpoints que declara la capa `api`, más `OPTIONS` para el preflight. Declararlos a mano sería una segunda fuente de verdad garantizada a divergir.

Dos reglas más, ambas validadas:
- `cors` sin capa `api` es **error**: CORS sin HTTP entrante no significa nada.
- `allowCredentials: true` es **obligatorio** si `tokenLocation: cookie`, porque si no el navegador simplemente no enviaría la cookie cross-origin. Con el token en `Authorization`, déjalo en `false`.

## 15. Qué comprueba la máquina y qué comprueba el criterio

Keel parte la revisión en dos, y la frontera es nítida: **lo mecánico va en código, lo semántico va en una skill**.

`keel validate` (`crossrefs.js`) comprueba **errores duros**:

| Comprobación | Mensaje |
|---|---|
| Rol inexistente | `el rol 'x' no existe en security: roles` |
| Permiso inexistente | `el permiso 'x' no existe en security: permissions` |
| Scope inexistente | `el scope 'x' no existe en security: permissions` |
| Roles con `level: service` | `level 'service' no admite roles (los roles son de usuarios humanos)` |
| Operación inexistente en `rules` | `security: access.rules.x: la operación no existe en use-cases` |
| Rol inexistente en `roleGrants` | `security: roleGrants.x: el rol no existe en security: roles` |
| `protocol: none` + regla que exige identidad | error |
| `cors` sin capa `api` | error |
| `allowCredentials: false` con `tokenLocation: cookie` | error |
| Toda la matriz audiencia × nivel del §12 | error |

Y **avisos** (no bloquean, pero cada uno es una decisión sin tomar):

- `level 'service' sin scopes — cualquier cliente autenticado podrá invocar la operación`
- `serviceClients declarado pero ningún endpoint es audience 'services' ni 'both'`
- Scopes concedidos y no exigidos / exigidos y no concedidos (§13)
- `Hay capa api pero no capa security`

La revisión **semántica** —lo que ninguna regla mecánica puede decidir— la hace la skill `/keel-validate`, y su lista para esta capa es exactamente esta:

- **Mutaciones con `level: public`**: cuestiónalas siempre.
- **Roles con permisos que ninguna regla usa**: exceso de privilegio.
- **Permisos huérfanos**: están en el catálogo y nadie los exige.
- **`serviceClients` con más scopes de los que sus llamadas necesitan**: mínimo privilegio también para máquinas.
- **Endpoints M2M sin `validateAudience`** cuando el servicio convive con otros que comparten servidor de autenticación: un token emitido para otro servicio valdría aquí (§6).

> **Idea clave:** ninguna de esas cinco es decidible por una regla. "¿Debería `listProducts` ser público?" depende del negocio. Por eso no están en `crossrefs.js`: meter juicio en un validador mecánico produce falsos positivos que la gente aprende a ignorar, y entonces el validador entero deja de servir.

---

# Parte III — Qué código sale de ahí (Spring)

Aquí es donde la declaración se vuelve concreta. Recuerda el flujo:

```bash
keel-spring build specs/product-service    # desde el workspace de diseño
cd services/product-service-spring
/keel-generate-spring                      # dentro del proyecto, sin argumentos
```

Toda la seguridad sale de **`build`**, el paso determinista. El agente **no escribe código de seguridad**: la skill del proveedor se lo prohíbe explícitamente. Volvemos a eso en §18.

## 16. El mapa de clases

Todo aterriza en el paquete `infrastructure.configurations.security`, y qué clases se generan depende de qué declara el diseño:

| Clase | Se genera cuando |
|---|---|
| `SecurityConfig` | Siempre que hay capa `security` |
| `SecurityErrorHandlers` | `protocol` distinto de `none` |
| `JwtAuthConverter` | `protocol` `oidc`/`jwt` **y** alguna regla usa roles, permisos, scopes o `level: admin` |
| `CorsConfig` | Hay bloque `cors` |
| `AudienceAuthorizationFilter` | `serviceAuth.validateAudience: true` con JWT |
| `ApiKeyAuthFilter` | `protocol: api-key` |
| `ServiceApiKeyAuthFilter` | `serviceAuth.protocol: api-key` con clientes declarados, sobre un protocolo principal distinto |
| `OpenApiSecurityConfig` | **Sin** capa `security`, pero con un `http-clients` que usa `oauth2-client-credentials` |

La última merece una nota porque es un efecto colateral no obvio, y el propio código lo documenta: el starter `spring-boot-starter-oauth2-client` —que entra por la autenticación **saliente**— arrastra Spring Security al classpath. Y la autoconfiguración de Boot, si no encuentra ninguna `SecurityFilterChain` declarada, **registra la suya**: login obligatorio en toda la API, `/actuator/health` incluido, que pasa a contestar un `302` al formulario. `OpenApiSecurityConfig` existe solo para desactivar eso:

```java
@Configuration
@EnableWebSecurity
public class OpenApiSecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .httpBasic(AbstractHttpConfigurer::disable)
            .formLogin(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll());
        return http.build();
    }
}
```

## 17. La tabla de traducción: de una regla del DSL a Spring

Toda la Parte II se reduce, en el generador, a **una función de 18 líneas**. Merece la pena leerla entera porque es la bisagra de todo:

```js
export function accessAuthority(rule) {
  if (rule.level === 'public') return 'permitAll()';
  const roles = rule.roles ?? [];
  const perms = rule.permissions ?? [];
  // Los scopes del diseño llegan como authorities SCOPE_<scope> (prefijo estándar
  // del resource server de Spring para el claim scope).
  const scopes = (rule.scopes ?? []).map((s) => `SCOPE_${s}`);
  const quote = (v) => JSON.stringify(v);
  const mixed = [...roles.map((r) => `ROLE_${r}`), ...perms, ...scopes];
  if ((roles.length > 0 ? 1 : 0) + (perms.length > 0 ? 1 : 0) + (scopes.length > 0 ? 1 : 0) > 1) {
    return `hasAnyAuthority(${mixed.map(quote).join(', ')})`;
  }
  if (scopes.length > 0) return `hasAnyAuthority(${scopes.map(quote).join(', ')})`;
  if (perms.length > 0) return `hasAnyAuthority(${perms.map(quote).join(', ')})`;
  if (roles.length > 0) return `hasAnyRole(${roles.map(quote).join(', ')})`;
  if (rule.level === 'admin') return 'hasRole("admin")';
  return 'authenticated()';
}
```

La tabla que produce:

| Regla del DSL | Java generado |
|---|---|
| `{ level: public }` | `permitAll()` |
| `{ level: required }` | `authenticated()` |
| `{ level: admin }` (sin roles) | `hasRole("admin")` |
| `{ roles: [catalog-admin] }` | `hasAnyRole("catalog-admin")` |
| `{ permissions: [product:write] }` | `hasAnyAuthority("product:write")` |
| `{ scopes: [product:read] }` | `hasAnyAuthority("SCOPE_product:read")` |
| `{ roles: [...], scopes: [...] }` (mezcla) | `hasAnyAuthority("ROLE_catalog-admin", "SCOPE_product:read")` |

**La trampa de los prefijos**, que es donde todo el mundo se equivoca al escribir esto a mano:

- `hasAnyRole("x")` **antepone `ROLE_` por ti**. Por eso el rol va sin prefijo.
- `hasAnyAuthority("x")` compara **literalmente**. Por eso ahí el rol sí lleva `ROLE_` explícito.
- `SCOPE_` **siempre** explícito: es el prefijo estándar del resource server de Spring para el claim `scope`, y `hasAnyAuthority` no lo añade solo.
- Los permisos van **sin prefijo ninguno**: `product:write` es literalmente la authority.

> **Idea clave:** mezclar `hasRole` y `hasAuthority` con los prefijos cambiados produce un servicio que **compila, arranca y deniega todo** (o peor, que permite todo). No lanza excepción; simplemente no coincide nunca. Es exactamente la clase de error que desaparece cuando la traducción la hace una función determinista en vez de una persona con prisa.

Y el bloque completo que se emite, una línea por operación:

```java
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/actuator/health/**", "/swagger-ui/**", "/swagger-ui.html", "/v3/api-docs/**").permitAll()
                    .requestMatchers(HttpMethod.POST, "/api/v1/products").hasAnyAuthority("product:write")
                    .requestMatchers(HttpMethod.DELETE, "/api/v1/products/{id}").hasAnyRole("catalog-admin")
                    .requestMatchers(HttpMethod.GET, "/api/v1/products").permitAll()
                    .anyRequest().hasAnyAuthority("product:read"))
```

Observa el `.anyRequest()` del final: **es el `access.default` del diseño**. Nada queda sin cubrir.

## 18. Por qué no hay ni un solo `@PreAuthorize`

Es un dato verificable: en todo el generador hay **cero** ocurrencias de `@PreAuthorize`, `@Secured`, `@EnableMethodSecurity` o `@EnableGlobalMethodSecurity`. Toda la autorización vive en los `requestMatchers` de la cadena.

La razón es de **fuente única**. Los matchers y los controllers se construyen del mismo índice `operación → ruta`:

```js
  // Índice operación → ruta (solo las expuestas por REST), fuente única con los
  // controllers para que los matchers no se desincronicen de los endpoints.
  const routeByOp = new Map();
```

Con `@PreAuthorize` habría dos lugares donde vive la ruta (la anotación `@PostMapping` y el matcher) y dos lugares donde vive la política. Con este diseño, si el endpoint cambia de ruta, el matcher cambia **en el mismo build**, del mismo dato. No pueden divergir.

Y hay un tercer punto, más sutil: una regla que apunta a una operación **sin endpoint REST** (una operación `internal`, disparada por schedule o suscripción) no puede materializarse como matcher. En vez de fallar o de ignorarlo en silencio, `build` avisa:

```
Regla de acceso 'reconcilePending' (security) no corresponde a ninguna operación
con endpoint REST; se ignora en el SecurityFilterChain.
```

Corolario práctico: **no reescribas `SecurityConfig`**. La skill del proveedor lo dice explícitamente y con los dos anti-patrones concretos:

> - No reescribas `SecurityConfig`/`JwtAuthConverter` para «arreglar» un 401/403: el arreglo casi siempre está en el realm (roles, mappers) o en el issuer.
> - No desactives la validación de firma ni uses `permitAll` para desbloquear escenarios: los escenarios de seguridad validan exactamente eso.

## 19. `JwtAuthConverter`: de claims a authorities

Este es el traductor de la frontera de entrada. Convierte lo que el IdP **afirma** en lo que Spring **comprueba**. Tiene **cuatro** fuentes de authorities y conviene verlas por separado:

```java
    private java.util.Collection<GrantedAuthority> extractAuthorities(Jwt jwt) {
        java.util.Collection<GrantedAuthority> roles = extractRoles(jwt);
        java.util.List<GrantedAuthority> authorities = new java.util.ArrayList<>(roles);
        authorities.addAll(extractScopes(jwt));
        authorities.addAll(extractPermissions(jwt));
        authorities.addAll(extractGrantedPermissions(roles));
        return authorities;
    }
```

**(1) Roles.** La forma del claim depende del proveedor, y el generador lo sabe. Con Keycloak están **anidados** en `realm_access.roles`; con Cognito son **planos** en `cognito:groups`:

```java
    private java.util.Collection<GrantedAuthority> extractRoles(Jwt jwt) {
        java.util.Map<String, Object> parent = jwt.getClaimAsMap("realm_access");
        if (parent == null) {
            return java.util.Collections.emptyList();
        }
        Object rolesObj = parent.get("roles");
        if (!(rolesObj instanceof java.util.List<?> roles)) {
            return java.util.Collections.emptyList();
        }
        return roles.stream()
                .filter(String.class::isInstance)
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .map(GrantedAuthority.class::cast)
                .toList();
    }
```

**(2) Scopes.** El claim `scope` de OAuth2 es un **string único separado por espacios**, no un array. De ahí el `split`:

```java
        return java.util.List.of(scopeClaim.split(" ")).stream()
                .filter(s -> !s.isBlank())
                .map(scope -> new SimpleGrantedAuthority("SCOPE_" + scope))
```

**(3) Permisos directos.** Si el IdP emite un claim `permissions`, entran como authority literal, sin prefijo.

**(4) `ROLE_GRANTS`, y aquí está lo interesante.** El `roleGrants` del diseño se materializa como un **mapa estático en el código Java**:

```java
    /** Permisos que otorga cada rol (security.roleGrants del diseño). */
    private static final java.util.Map<String, java.util.List<String>> ROLE_GRANTS = java.util.Map.ofEntries(
            java.util.Map.entry("catalog-admin", java.util.List.of("product:write", "product:read")),
            java.util.Map.entry("catalog-reader", java.util.List.of("product:read")));
```
```java
    /**
     * Expande los roles del token a las authorities de permiso que otorgan, según
     * el catálogo roleGrants del diseño.
     */
    private java.util.Collection<GrantedAuthority> extractGrantedPermissions(java.util.Collection<GrantedAuthority> roles) {
        return roles.stream()
                .map(GrantedAuthority::getAuthority)
                .map(authority -> authority.startsWith("ROLE_") ? authority.substring(5) : authority)
                .flatMap(role -> ROLE_GRANTS.getOrDefault(role, java.util.List.of()).stream())
                .distinct()
                .map(SimpleGrantedAuthority::new)
                .map(GrantedAuthority.class::cast)
                .toList();
    }
```

**¿Por qué en el código y no en el token?** Porque **ningún IdP emite `roleGrants` por defecto**. Keycloak manda roles y nada más. Sin esta expansión, un token con `realm_access.roles: ["catalog-reader"]` llegaría a un matcher que exige `hasAnyAuthority("product:read")` y sería denegado — aunque el diseño diga clarísimamente que ese rol otorga ese permiso.

Las alternativas serían: (a) configurar un protocol mapper en el IdP que meta los permisos en el token, lo que ata tu diseño a la configuración de un producto externo y la duplica; o (b) escribir todas las reglas contra roles en vez de contra permisos, perdiendo el nivel de granularidad. Keel elige la tercera: **`roleGrants` es información del diseño, así que viaja en el código generado**.

> **Idea clave:** esta es la respuesta operativa a "¿para qué sirve un rol si al final todo son permisos?". El rol es lo que el IdP administra y lo que viaja en el token; el permiso es lo que el código comprueba; y `ROLE_GRANTS` es el puente, generado, entre los dos. Cambias `roleGrants` en el YAML, regeneras, y el paquete cambia sin tocar ni una regla de acceso.

Y una decisión por omisión que conviene registrar: **no se genera ningún bean `JwtDecoder`**. El decoder lo autoconfigura Boot desde `spring.security.oauth2.resourceserver.jwt.*`, que `build` siembra por perfil. Un decoder propio obligaría a hardcodear una clave en `src/main` y acoplaría esa clase al perfil de pruebas.

## 20. La cadena partida por audiencia, y el 403 que no es un 401

Cuando `validateAudience: true`, el generador **parte la cadena en dos** con `@Order`:

```java
    /**
     * Endpoints de audiencia services (clientes máquina): mismo resource server,
     * más la comprobación de audiencia como paso de AUTORIZACIÓN — un token de
     * usuario válido aquí da 403 (autenticado, sin permiso), no 401.
     */
    @Bean
    @Order(1)
    public SecurityFilterChain serviceFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/v1/products/{id}/price")
            ...
            .addFilterBefore(new AudienceAuthorizationFilter(audience), AuthorizationFilter.class);
        return http.build();
    }
```

Y solo caen ahí las rutas `audience: services`, **nunca las `both`**, por un motivo que no es evidente y que el propio código explica:

```js
// Rutas de audiencia 'services' (clientes máquina). Las 'both' quedan fuera a
// propósito: las sirve también un usuario, cuyo token no lleva la audiencia del
// servicio, así que no pueden caer en la cadena que la valida.
```

El filtro en sí es corto y su decisión es toda la §6 hecha código:

```java
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication instanceof JwtAuthenticationToken token
                && (token.getToken().getAudience() == null || !token.getToken().getAudience().contains(audience))) {
            throw new AccessDeniedException("El token no está emitido para la audiencia " + audience);
        }
```

**Por qué es un filtro y no una validación del decoder.** Validar la audiencia en el `JwtDecoder` sería lo obvio y Spring lo soporta. Pero entonces el token fallaría en la fase de **autenticación** y el resultado sería `401`. Y eso es una mentira: el token es auténtico, la firma es correcta, sabemos quién es. Lo que ocurre es que no le autoriza a hablar con nosotros. Poniéndolo como filtro de autorización, el resultado es `403`, que es la verdad.

La tabla canónica de códigos del método, que es también el contrato que los escenarios verifican:

| Situación | Status |
|---|---|
| Sin cabecera `Authorization`, o token inválido/caducado | `401` |
| Token válido sin el scope exigido | `403` |
| Token válido con la audiencia de otro servicio | `403` |

> El `401` está reservado a la **autenticación**.

Los dos handlers que producen esos códigos también se generan:

```java
@Component
public class SecurityErrorHandlers implements AuthenticationEntryPoint, AccessDeniedHandler {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
            AuthenticationException exception) throws IOException {
        write(response, HttpStatus.UNAUTHORIZED, "Unauthorized", "Credenciales ausentes o no válidas");
    }

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
            AccessDeniedException exception) throws IOException {
        write(response, HttpStatus.FORBIDDEN, "Forbidden", "La credencial no autoriza esta operación");
    }
```

## 21. Configuración por perfil, y el issuer partido

El resource server necesita saber **de dónde bajar las claves públicas**. Eso se siembra por perfil, con un gradiente deliberado de "conveniencia → rigor":

| Perfil | `issuer-uri` generado (Keycloak) | Intención |
|---|---|---|
| `local` | `http://localhost:8180/realms/<servicio>` | Valor literal: arranca sin configurar nada |
| `develop` | `${OAUTH2_ISSUER_URI:http://localhost:8180/realms/<servicio>}` | Redirigible, con default |
| `production` | `${OAUTH2_ISSUER_URI}` | **Obligatorio**: si falta, no arranca |

El perfil `test` es un caso especial con una solución bonita:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Perfil test: no hay proveedor de identidad. Puerto 9 (discard) a
          # propósito: si algo intentara resolverlo, falla en local y rápido.
          jwk-set-uri: http://localhost:9/.well-known/jwks.json
```

El problema: sin `issuer-uri` **ni** `jwk-set-uri`, Boot no autoconfigura ningún `JwtDecoder`, y entonces la `SecurityFilterChain` generada no se puede construir — `@SpringBootTest` muere con `NoSuchBeanDefinitionException`. La solución no puede ser un `issuer-uri`, porque ese **sí** se resuelve al arrancar (hace discovery OIDC contra la red). Pero el JWK set se pide de forma **perezosa**, en la primera validación de token: basta para crear el decoder y no toca la red nunca. Y el puerto 9 es el puerto *discard*: si algo llegara a intentarlo, falla instantáneamente y en local, no con un timeout de 30 segundos contra internet.

### El issuer partido

Este merece su apartado porque es el `401` más desconcertante que te vas a encontrar.

En `deploy/` (el compose de pruebas manuales, donde la app corre **dentro** de la red de contenedores):

- Tú pides el token contra `http://localhost:8180/...`
- La app alcanza a Keycloak como `http://keycloak:8080/...`

Y la validación de OIDC compara el claim `iss` del token **carácter a carácter** con el issuer configurado. `localhost:8180` ≠ `keycloak:8080`. **Todo da 401**, con un token perfectamente válido.

La solución, y su precio explícito:

```js
      // El issuer partido: Keycloak emite tokens con `iss` = localhost:8180 (la URL
      // por la que el diseñador pide el token) pero la app lo alcanza como
      // keycloak:8080. Validar por issuer-uri daría 401 a todo. Con jwk-set-uri
      // —que Boot prioriza sobre issuer-uri— el decoder resuelve las claves por la
      // ruta interna y no hace discovery. Se deja de validar el claim `iss`, y solo
      // aquí: el perfil local y la suite integrationTest siguen usando issuer-uri.
```

Traducido: en `deploy/` **no se comprueba el claim `iss`** — sí la firma, la caducidad, la audiencia y los roles. Es un entorno de prueba manual y el trade-off está declarado por escrito, no escondido. La validación completa sigue viva en el perfil `local` y en `./gradlew integrationTest`, que hablan con Keycloak por la misma URL que la app.

> **Idea clave:** la regla general que hay detrás vale para cualquier despliegue con OIDC: **el issuer es una cadena, no una dirección**. Si el token se pide por una URL y se valida por otra, falla, aunque las dos apunten a la misma máquina. Es la causa raíz de la mayoría de los `401` en Docker, Kubernetes e ingress con reescritura de host.

## 22. La rama `api-key`

Para servicios donde OIDC es desproporcionado, `protocol: api-key` genera un filtro simple:

```java
public class ApiKeyAuthFilter extends OncePerRequestFilter {

    private static final String HEADER = "X-API-Key";
    ...
        if (apiKey != null && !apiKey.isBlank() && apiKey.equals(provided)) {
            UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken("api-key-client", null, List.of());
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }
```

Nota el `List.of()` vacío: una API key global no lleva identidad ni capacidades. Es autenticación sin autorización granular.

Más interesante es el caso mixto —usuarios por OIDC, máquinas por API key (`serviceAuth.protocol: api-key`)—, porque ahí **sí** hay granularidad, y sale directa del diseño. Cada `serviceClient` produce un `@Value` y una entrada con **sus scopes como authorities**:

```java
    @Value("${security.api-keys.billing-service:}")
    private String billingServiceApiKey;

    private List<ServiceApiKeyAuthFilter.ServiceClient> serviceClients() {
        return List.of(
                new ServiceApiKeyAuthFilter.ServiceClient("billing-service", billingServiceApiKey, List.of("SCOPE_product:read")));
    }
```

Es decir: **el mismo modelo conceptual de scopes funciona sin OAuth2**. El scope no es una cosa de OAuth2; es "lo que esta credencial puede". OAuth2 es solo la forma más común de transportarlo.

---

# Parte IV — Cómo se prueba

Un servicio con seguridad que no se ha probado con tokens reales no está probado. Keel resuelve esto generando el entorno de identidad **del mismo diseño**.

## 23. `realmSpec()`: una fuente, tres artefactos

El generador construye una estructura intermedia a partir de la capa `security`, y de ella salen tres cosas distintas:

```
security.keel.yaml
       │
       ▼
   realmSpec()
       │
       ├──► infra/init-keycloak.sh          (script kcadm: crea el realm)
       ├──► infra/test-credentials.env      (los valores, para los tests)
       └──► deploy/keycloak/realm-export.json  (el mismo realm en declarativo)
```

Los dos primeros son un **contrato entre dos agentes** del pipeline, y el código lo dice:

```js
//   infra/init-keycloak.sh     — lo que hay que crear (realm, roles, usuarios,
//                                clientes de diseño y la matriz M2M de prueba).
//   infra/test-credentials.env — los valores con los que se crea, que es lo que
//                                AbstractFlowIT lee. Un solo productor, un solo
//                                consumidor, ningún literal inventado a los dos lados.
```

El tercero existe porque el compose de `deploy/` importa el realm al arrancar en vez de ejecutar un script. Son **dos formatos del mismo realm**, de la misma función — y hay un test de paridad que lo garantiza.

## 24. Del diseño al realm

Las conversiones, una por una:

| En el diseño | En Keycloak |
|---|---|
| `roles: { catalog-admin: ... }` | Un **rol de realm** `catalog-admin` |
| Cada rol | Un **usuario de prueba homónimo** con ese rol y contraseña `password` |
| — | Un usuario extra `no-role`, sin roles |
| `permissions` usados como `scopes` | Un **client scope** con `include.in.token.scope=true` |
| `serviceAuth.audience` | Un client scope `aud-<servicio>` con un **audience mapper** |
| `serviceClients: { billing-service: ... }` | Un **cliente confidencial** `serviceAccountsEnabled=true`, secreto `billing-service-secret` |

Fragmentos reales del script generado:

```bash
echo "== Roles del diseno (security.keel.yaml) =="
run "create roles -r $REALM -s name=catalog-admin"
run "create roles -r $REALM -s name=catalog-reader"

echo "== Usuarios de prueba: uno por rol (username = rol) + uno sin roles =="
for USER in catalog-admin catalog-reader no-role; do
  run "create users -r $REALM -s username=$USER -s enabled=true ..."
  run "set-password -r $REALM --username $USER --new-password $PASSWORD"
done
run "add-roles -r $REALM --uusername catalog-admin --rolename catalog-admin"
```

La convención **`username = rol`** es lo que hace que un test pueda escribir `tokenFor("catalog-admin")` sin inventarse nada.

Y una separación fina que vale la pena señalar: los client scopes de **audiencia** están desacoplados de los de **permiso**:

```bash
echo "== Client scopes de audiencia (desacoplados de los de permisos) =="
run "create client-scopes -r $REALM -s name=aud-$SVC -s protocol=openid-connect"
...
echo "== Client scopes de permiso (sin mapper de audiencia) =="
run "create client-scopes -r $REALM -s name=product:read -s protocol=openid-connect -s 'attributes.\"include.in.token.scope\"=true'"
```

Están separados porque son **dos ejes independientes**, y hace falta poder combinarlos libremente para el §25.

El script es **idempotente**: tolera el `409` de Keycloak (recurso ya existente), así que se puede reejecutar tras cada `compose up`.

## 25. La matriz `test-m2m-*`

Aquí está la razón de separar audiencia y scope. Para poder **aislar la causa** de un `403`, hacen falta cuatro clientes de prueba con las cuatro combinaciones:

| Cliente | Client scopes | Token resultante | Sirve para |
|---|---|---|---|
| `test-m2m-ok` | `aud-<servicio>` + `<recurso>:<accion>` | scope ✓ / aud ✓ | Camino feliz M2M → 2xx |
| `test-m2m-no-scope` | `aud-<servicio>` | scope ✗ / aud ✓ | Aísla el **403 por scope** |
| `test-m2m-bad-aud` | `aud-wrong` + `<recurso>:<accion>` | scope ✓ / aud ✗ | Aísla el **403 por audiencia** |
| `test-m2m-none` | (ninguno) | scope ✗ / aud ✗ | Control: con las dos fallando sigue siendo `403`, **nunca** `401` |

> **Idea clave:** el cuarto cliente es el que más gente omitiría y el más valioso. Sin él, un servicio que devuelve `401` cuando debería devolver `403` pasa desapercibido, y eso es un bug de contrato real: un cliente bien programado reacciona a un `401` **renovando el token y reintentando** — y si la causa era de autorización, entra en un bucle de reintentos que nunca converge.

Los cuatro solo se crean cuando tienen sentido: si no hay `serviceAuth`, o es `api-key`, o el diseño no nombra ningún scope, la matriz está vacía. Y los dos de audiencia solo aparecen con `validateAudience: true`.

## 26. Cómo los tests consiguen tokens

El arnés de integración (`AbstractFlowIT`, generado por `build`) expone dos métodos y nada más:

```java
    protected String tokenFor(String role) {
        return credentials.computeIfAbsent(role, key ->
            requestToken("grant_type=password"
                + "&client_id=" + env("AUTH_TEST_CLIENT", "product-service-spring-test")
                + "&username=" + key
                + "&password=" + env("AUTH_TEST_PASSWORD", "password")));
    }

    protected String serviceCredential(String client) {
        return credentials.computeIfAbsent("client:" + client, key ->
            requestToken("grant_type=client_credentials"
                + "&client_id=" + client
                + "&client_secret=" + clientSecret(client)));
    }
```

Los dos grants de §5, uno cada uno, cacheados. Y la resolución de valores tiene tres niveles, en orden:

1. Variable de entorno (`AUTH_TOKEN_URL`, `AUTH_TEST_CLIENT`…) — para CI.
2. `infra/test-credentials.env` — lo que generó `build`.
3. La convención derivable (`<cliente>-secret`) — para que funcione aunque el archivo no esté.

El humo del arnés lo comprueba antes de correr nada:

```java
    @Test
    @Order(3)
    @DisplayName("SMOKE-3: el proveedor de identidad emite credenciales")
    void issuesCredentials() {
        Assertions.assertFalse(tokenFor("catalog-admin").isBlank(),
            "El proveedor de identidad no devolvió token para el rol 'catalog-admin'.");

        Assertions.assertFalse(serviceCredential("billing-service").isBlank(),
            "No hay credencial de máquina para el cliente 'billing-service': revisa infra/test-credentials.env.");
    }
```

Si eso falla, la suite entera se para: es mucho mejor que 40 escenarios rojos por un realm mal aprovisionado.

## 27. Qué escenarios obliga a escribir la capa `security`

Del catálogo de escenarios de validación del método:

**Para toda operación protegida** (no solo M2M):

> **Autorización por operación**: cada operación protegida cubre la llamada **sin credencial** (`401`) y **con credencial sin el permiso exigido** (`403`).

**Para endpoints M2M** (`audience: services`/`both` con `serviceAuth`), cuatro bloques, y nótese que **no** es solo auth:

- **Contrato funcional**: llamada con credencial de máquina válida y scopes correctos, con la forma real del request y la verificación del **response completo** que el otro servidor consume, no solo el `2xx`.
- **Errores declarados**: cada `error` de la operación, ejercido **con credencial de máquina**, con su `code` y status exactos.
- **Auth**: sin el scope exigido → `403`; y con `validateAudience: true`, token de otra audiencia → `401`.
- Los endpoints `audience: both` cubren **además** el acceso con token de usuario, y fijan si la respuesta es idéntica a la del público humano.

Y una regla de estilo que es más importante de lo que parece:

> Los escenarios hablan de "credencial de máquina del cliente `<serviceClient>`", nunca del proveedor concreto.

Igual que el diseño no nombra Keycloak, el escenario tampoco. El escenario describe **qué se observa**; cómo se consigue el token es del generador.

---

# Cierre

## Diseñar la seguridad de un servicio, paso a paso

1. **Enumera las capacidades**, no los puestos. Recorre las operaciones de `use-cases` y pregúntate qué capacidad ejerce cada una. Salen 4-8 permisos `recurso:accion`, no 40.
2. **Agrupa en roles** los paquetes que un administrador entregaría de una vez. Escríbelos en `roles` y su contenido en `roleGrants`.
3. **Escribe `access.default` primero**, y que sea restrictivo. Lo que quede sin regla explícita cae ahí, y quieres que ese default sea seguro.
4. **Añade `rules` solo donde difiera** del default. Una regla por operación, por su **nombre**.
5. **Si hay consumidores máquina**: declara `serviceAuth` (con `validateAudience: true` si compartes IdP), enumera los `serviceClients` con los scopes mínimos, y marca las reglas con `level: service` + `scopes`. Ajusta la `audience` de esos endpoints en la capa `api`.
6. **Si lo consume un navegador**: añade el bloque `cors`. Los orígenes no van ahí.
7. **`keel validate`** y **cierra todos los avisos**. Cada aviso es una decisión sin tomar, no ruido.
8. **`/keel-validate`** para la revisión semántica: mínimo privilegio, mutaciones públicas, permisos huérfanos.

## Errores frecuentes

| Síntoma | Causa probable |
|---|---|
| `level 'service' no admite roles` | Estás pensando en la máquina como si fuera una persona. Usa `scopes`. |
| `el scope 'x' no existe en security: permissions` | Buscas un catálogo de scopes que no existe. Declara la capacidad en `permissions`. |
| `level 'service' con audience 'both' excluiría a los usuarios` | Querías "cualquiera de". Usa `level: required` + `permissions` + `scopes`. |
| Aviso: *el scope no lo exige ninguna regla* | Privilegio concedido de más a un cliente máquina. |
| Aviso: *ningún cliente podría invocar esas operaciones* | Exiges un scope que no has concedido a nadie. Puerta tapiada. |
| Aviso: *`level 'service'` sin scopes* | Cualquier cliente autenticado del ecosistema entra. Casi nunca es lo que quieres. |
| `403` con un rol que "debería" tener el permiso | Falta la entrada en `roleGrants`. El rol vacío no otorga nada. |
| `401` con un token perfectamente válido en `deploy/` | El issuer partido (§21): el token se pidió por una URL y se valida por otra. |
| Un `401` donde esperabas `403` | Estás validando algo de autorización (audiencia, scope) en la fase de autenticación. |
| Mutación accesible sin token | `access.default` demasiado laxo, o un `level: public` que nadie cuestionó. |

## La tesis, en una frase

La capa `security` de Keel no es "configuración de Spring Security en YAML". Es la afirmación de que **quién puede hacer qué** es una decisión de **diseño** —revisable, versionable, discutible en un pull request y comparable entre servicios— y que el `SecurityFilterChain`, el realm de Keycloak, los usuarios de prueba y los escenarios de autorización son todos **proyecciones deterministas** de esa decisión.

Cuando cambias `roleGrants`, regeneras, y cambian a la vez el mapa `ROLE_GRANTS` del converter, el realm de prueba y la matriz de escenarios. No hay tres sitios que mantener sincronizados. Hay uno.
