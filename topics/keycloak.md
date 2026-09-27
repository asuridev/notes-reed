# Guía de Keycloak — del concepto al caso de uso real

> Lectura progresiva: cada sección construye sobre la anterior.
> Los ejemplos están tomados de escenarios reales de industria: banca digital, salud, SaaS multi-tenant, e-commerce y logística.

---

## Parte 1 — Conceptos fundamentales

### 1.1 ¿Qué es Keycloak?

Keycloak es un servidor de **autorización e identidad** (IAM — Identity and Access Management) de código abierto que implementa los estándares:

| Estándar | Para qué sirve |
|---|---|
| **OAuth 2.0** | Delegar acceso a recursos mediante tokens |
| **OpenID Connect (OIDC)** | Capa de identidad sobre OAuth 2.0 — provee un `id_token` con datos del usuario |
| **SAML 2.0** | Federación de identidad empresarial (SSO entre organizaciones) |

En el contexto de este proyecto, Keycloak actúa como **Authorization Server** en el flujo OAuth 2.0:

```
App móvil de banca (cliente público)
   │
   │  1. Usuario ingresa usuario/contraseña o Face ID
   ▼
Keycloak (Authorization Server)
   │
   │  2. Emite JWT firmado con claims de roles (TELLER, MANAGER, CUSTOMER)
   ▼
API de Cuentas — Spring Boot (Resource Server)
   │
   │  3. Valida la firma del JWT contra el JWKS de Keycloak
   │  4. Extrae roles del claim `realm_access.roles`
   │  5. Aplica @PreAuthorize("hasAnyRole(...)")
   ▼
Recurso protegido: GET /accounts/{id}/transactions → 200 / 403
```

---

### 1.2 El Realm — unidad raíz de aislamiento

Un **realm** es el espacio de configuración aislado dentro de Keycloak. Cada realm tiene su propio conjunto de:

- Usuarios
- Roles
- Clientes (aplicaciones registradas)
- Políticas de contraseña
- Proveedores de identidad externos (Google, LDAP, etc.)
- Claves de firma JWT (par RSA o EC)

```
Keycloak instance
├── realm: master            ← Solo para administrar Keycloak. NUNCA usar para apps.
├── realm: bankify-retail    ← Clientes del banco (app móvil, web)
├── realm: bankify-internal  ← Empleados (banca en línea, back-office)
└── realm: bankify-partners  ← APIs B2B para fintechs asociadas
```

**Regla operacional:** nunca registres clientes de aplicación en el realm `master`. Crea un realm dedicado por dominio de negocio o por audiencia.

Otro patrón común en **SaaS multi-tenant**: un realm por producto, con el `tenant_id` como atributo del usuario (no como realm separado). Un realm separado por tenant solo tiene sentido cuando los tenants son grandes empresas con requisitos de aislamiento total (cumplimiento regulatorio, LDAP propio).

**En este proyecto** (`arch/system/system.yaml`):
```yaml
authServer: true
authProvider:
  type: keycloak
  realm: dsl-demo
```

La URL base de un realm sigue el patrón:
```
http://localhost:8180/realms/{realm-name}
```

El JWKS (claves públicas para validar JWT) está en:
```
http://localhost:8180/realms/{realm-name}/protocol/openid-connect/certs
```

---

### 1.3 Clientes — las aplicaciones registradas

Un **Client** en Keycloak representa una aplicación que puede solicitar tokens. Cada cliente tiene:

| Campo | Significado |
|---|---|
| `Client ID` | Identificador público de la aplicación (`my-app`, `my-app-backend`) |
| `Client Secret` | Secreto usado en flujos confidenciales (machine-to-machine) |
| `Root URL` | URL base de la aplicación (para validar redirect URIs) |
| `Valid Redirect URIs` | Patrones permitidos de redirección post-login |
| `Access Type` | `public` (SPAs, móvil) o `confidential` (backends) |

#### Tipos de cliente según quién consume el token

**Public client** — el secreto no es seguro (frontend SPA, app móvil):
```
Client ID: bankify-mobile
Client Secret: (ninguno — no se puede proteger en una app móvil)
Direct Access Grants: true  ← permite Resource Owner Password Credentials
Standard Flow: true         ← Authorization Code + PKCE para login con UI
```

**Confidential client** — tiene secreto, típico en backends y microservicios:
```
Client ID: fraud-detection-service
Client Secret: f7d3a1b9...
Service Accounts: true  ← permite Client Credentials flow (machine-to-machine)
```

**Industria financiera — clientes típicos en un banco digital:**

| Cliente | Tipo | Flujo | Para qué |
|---|---|---|---|
| `bankify-web` | public | Authorization Code + PKCE | Portal web del cliente |
| `bankify-mobile-ios` | public | Authorization Code + PKCE | App iOS |
| `bankify-backoffice` | confidential | Authorization Code | Portal de empleados |
| `fraud-detection-svc` | confidential | Client Credentials | Microservicio de fraude |
| `notification-svc` | confidential | Client Credentials | Servicio de notificaciones |

**En este proyecto** la aplicación Spring Boot es un **Resource Server** (no solicita tokens, solo los valida). El cliente que solicita tokens es Postman / el frontend, configurado como `public` con `Direct Access Grants: true` para pruebas.

---

### 1.4 Usuarios y credenciales

Un usuario en Keycloak tiene:

```json
{
  "username": "maria.gonzalez",
  "email": "maria.gonzalez@hospital-central.com",
  "enabled": true,
  "attributes": {
    "employee_id": ["EMP-00421"],
    "department": ["cardiologia"],
    "license_number": ["MED-2019-041"]
  },
  "credentials": [
    { "type": "password", "value": "Temp@2024!", "temporary": true }
  ],
  "realmRoles": ["ATTENDING_PHYSICIAN"]
}
```

**`temporary: true`** — el médico deberá cambiar la contraseña en el primer login (buena práctica para cuentas corporativas creadas por el administrador).

**`attributes`** — campos personalizados. Se pueden proyectar al JWT mediante Protocol Mappers (ver sección 4.2).

**`realmRoles`** — roles a nivel realm. Son los que Keycloak incluirá en `realm_access.roles` dentro del JWT.

---

### 1.5 El JWT — anatomía del token

Cuando Keycloak emite un token con éxito, devuelve un JWT firmado. Decodificado (`jwt.io`):

**Header:**
```json
{
  "alg": "RS256",
  "kid": "abc123",     ← Key ID, usado para buscar la clave pública en el JWKS
  "typ": "JWT"
}
```

**Payload (claims) — ejemplo de un operador logístico:**
```json
{
  "exp": 1746537600,
  "iat": 1746534000,
  "jti": "3f7a1c2e-...",
  "iss": "https://auth.rapidoenvios.com/realms/operations",
  "sub": "8e4b2f1a-0c3d-...",
  "preferred_username": "carlos.mendez",
  "email": "carlos.mendez@rapidoenvios.com",
  "given_name": "Carlos",
  "family_name": "Méndez",
  "realm_access": {
    "roles": ["DISPATCH_OPERATOR", "default-roles-operations"]
  },
  "depot_id": "MX-GDL-04",        ← atributo personalizado (Protocol Mapper)
  "scope": "openid email profile"
}
```

`depot_id` permite al microservicio de ruteo filtrar automáticamente los envíos del depósito del operador, sin necesidad de un parámetro en la URL.

**Firma:** hash RSA-256 del header+payload usando la clave privada del realm. Spring Boot valida esta firma descargando las claves públicas del JWKS endpoint.

---

## Parte 2 — Flujos de autenticación (Grant Types)

### 2.1 Resource Owner Password Credentials (ROPC)

El usuario entrega usuario/contraseña directamente al cliente (Postman, script de prueba). **No recomendado en producción**, pero útil para testing local y scripts de integración internos.

```bash
# Caso de uso: equipo de QA obtiene token para pruebas de regresión del API de pedidos
curl -X POST https://auth.ecommerce-corp.com/realms/marketplace/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=qa-testing-client" \
  -d "username=qa.robot@ecommerce-corp.com" \
  -d "password=QaTest@2024"
```

Respuesta:
```json
{
  "access_token": "eyJhbGci...",
  "expires_in": 900,
  "refresh_token": "eyJhbGci...",
  "token_type": "Bearer"
}
```

Usar el token:
```bash
curl -H "Authorization: Bearer eyJhbGci..." \
  https://api.ecommerce-corp.com/v1/orders/ORD-00129483
```

---

### 2.2 Client Credentials (machine-to-machine)

Cuando no hay usuario humano involucrado — un microservicio llama a otro. El cliente se autentica con su propio secreto.

```bash
# Caso de uso: el servicio de detección de fraude consulta el historial de
# transacciones del servicio de cuentas para evaluar una operación sospechosa.
curl -X POST https://auth.bankify.com/realms/banking/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=fraud-detection-svc" \
  -d "client_secret=f7d3a1b9..."
```

En este flujo no hay `preferred_username` real — Keycloak pone el `client_id` como subject (`sub`). El JWT resultante:

```json
{
  "sub": "fraud-detection-svc",
  "realm_access": { "roles": ["SERVICE_ACCOUNT", "TRANSACTIONS_READER"] },
  "scope": "transactions:read"
}
```

**Otros ejemplos de M2M en industria:**
- Servicio de notificaciones → consulta datos del usuario para personalizar el SMS
- Pipeline de ETL → extrae datos del ERP cada noche
- Scheduler de facturación → genera y envía facturas automáticamente
- Servicio de reportes regulatorios → agrega datos de múltiples microservicios

---

### 2.3 Authorization Code + PKCE (el flujo correcto para producción)

Flujo seguro para aplicaciones web y móviles. El usuario autentica en Keycloak sin exponer sus credenciales al cliente:

```
1. App móvil del paciente → Keycloak:
   GET /auth?client_id=salud-app&redirect_uri=salud://callback&code_challenge=XYZ&scope=openid profile

2. Keycloak muestra pantalla de login (el usuario escribe sus credenciales en Keycloak, no en la app)

3. Keycloak → App: redirect a salud://callback?code=ABC123

4. App → Keycloak: POST /token { code=ABC123, code_verifier=... }

5. Keycloak → App: { access_token, id_token, refresh_token }
```

**`code_challenge` / `code_verifier`** es PKCE (Proof Key for Code Exchange) — obligatorio para apps móviles. Previene que una app maliciosa intercepte el `code` de autorización antes de canjearlo por un token.

**Caso de uso — portal de pacientes de un hospital:**
- La app nunca ve la contraseña del hospital (HIPAA compliance)
- El médico puede usar su cuenta corporativa (Azure AD federeado en Keycloak)
- El paciente usa su email personal
- Ambos obtienen tokens con diferentes roles: `PATIENT` vs `ATTENDING_PHYSICIAN`

---

### 2.4 Refresh Token

Cuando el `access_token` expira (por defecto 5 minutos en Keycloak), se puede renovar sin pedir credenciales:

```bash
curl -X POST https://auth.bankify.com/realms/banking/protocol/openid-connect/token \
  -d "grant_type=refresh_token" \
  -d "client_id=bankify-web" \
  -d "refresh_token=eyJhbGci..."
```

El `refresh_token` tiene una vida mucho mayor (configurable hasta días).

> **Caso de uso — banca en línea:** el cajero trabaja un turno de 8 horas. El `access_token` expira cada 10 minutos (reduciendo el riesgo si el token es robado), pero la sesión se mantiene activa durante todo el turno gracias al `refresh_token`. Si el cajero cierra el navegador, el `refresh_token` se invalida en el logout. Si está inactivo más de 30 minutos, el `SSO Session Idle` expira y debe autenticarse de nuevo.

---

## Parte 3 — Roles y autorización

### 3.1 Roles de Realm vs Roles de Cliente

| Tipo | Scope | Claim en JWT | Uso |
|---|---|---|---|
| **Realm Roles** | Todo el realm | `realm_access.roles` | Roles de dominio de negocio |
| **Client Roles** | Solo ese cliente | `resource_access.{client}.roles` | Permisos específicos de una app |

**Ejemplo — sistema hospitalario con Realm Roles:**

```
Realm: hospital-central
  Roles:
    ATTENDING_PHYSICIAN   ← médico titular, puede prescribir y ordenar estudios
    RESIDENT              ← residente, puede ver expedientes y cargar notas
    NURSE                 ← enfermera, puede ver expedientes y registrar signos vitales
    LAB_TECHNICIAN        ← laboratorista, solo accede a solicitudes de estudios
    PATIENT               ← paciente, solo ve su propio expediente
    BILLING_STAFF         ← facturación, sin acceso a datos clínicos
    ADMIN                 ← administración del sistema, sin acceso a datos clínicos
```

**Ejemplo — Client Roles para separar permisos por módulo:**

```
Client: erp-inventory
  Roles:
    inventory:read     ← puede consultar stock
    inventory:write    ← puede modificar stock
    inventory:approve  ← puede aprobar ajustes de inventario
```

Un agente de almacén tiene el Realm Role `WAREHOUSE_STAFF` (acceso general al ERP) más el Client Role `erp-inventory:write` (puede ajustar stock). Un auditor externo tiene solo `erp-inventory:read` sin rol de realm.

**En este proyecto** se usan Realm Roles porque los roles de dominio aplican transversalmente a todos los bounded contexts del sistema.

---

```java
// JwtAuthConverter.java — generado por el generador
// Lee realm_access.roles del JWT (Keycloak) y construye las GrantedAuthority de Spring
Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
List<String> roles = (List<String>) realmAccess.get("roles");
// "ATTENDING_PHYSICIAN" → SimpleGrantedAuthority("ROLE_ATTENDING_PHYSICIAN")
```

---

### 3.2 Asignación de roles a usuarios

En la UI de Keycloak: `Users → {usuario} → Role Mapping → Realm Roles → Assign Role`

Ejemplo — plataforma de logística de última milla:
```json
{
  "users": [
    {
      "username": "jorge.ramirez",
      "realmRoles": ["DISPATCH_OPERATOR"]
    },
    {
      "username": "lucia.torres",
      "realmRoles": ["FLEET_MANAGER", "DISPATCH_OPERATOR"]
    },
    {
      "username": "api.reporting",
      "realmRoles": ["SERVICE_ACCOUNT", "REPORTS_READER"]
    }
  ]
}
```

`lucia.torres` es gerente de flota y también opera despacho — tiene dos roles simultáneos. `api.reporting` es una cuenta de servicio para el microservicio de reportes.

---

### 3.3 Composite Roles — roles que incluyen otros roles

Un rol compuesto es un rol que automáticamente incluye otros roles. Modela jerarquías de acceso sin duplicar asignaciones.

**Caso de uso — sistema de salud:**

```
ATTENDING_PHYSICIAN (composite)
  ├── RESIDENT            ← el médico titular tiene todo lo que tiene un residente
  │     └── NURSE         ← el residente tiene todo lo que tiene una enfermera
  │           └── PATIENT ← todos pueden ver información básica del paciente
  └── PRESCRIBER          ← exclusivo del médico titular: puede emitir recetas
```

Si se asigna solo `ATTENDING_PHYSICIAN` al usuario, Keycloak incluye automáticamente `RESIDENT`, `NURSE` y `PATIENT` en `realm_access.roles` del JWT. El endpoint de signos vitales que requiere `hasAnyRole('NURSE')` acepta al médico titular sin configuración adicional.

**Caso de uso — e-commerce B2B:**

```
ACCOUNT_MANAGER (composite)
  └── BUYER               ← puede crear órdenes de compra
        └── CATALOG_VIEWER ← puede consultar catálogo y precios
```

Un `ACCOUNT_MANAGER` puede gestionar la cuenta, crear órdenes y ver el catálogo. Un `BUYER` puede crear órdenes y ver catálogo pero no gestionar la cuenta.

---

### 3.4 @PreAuthorize y cómo encaja Spring Security

**Caso de uso — API de transferencias bancarias:**

```java
// Solo el cajero y el supervisor pueden iniciar una transferencia
@PreAuthorize("hasAnyRole('TELLER', 'BRANCH_SUPERVISOR')")
public ResponseEntity<TransferReceipt> initiateTransfer(...) { ... }

// Solo el supervisor puede aprobar transferencias de alto valor
@PreAuthorize("hasRole('BRANCH_SUPERVISOR')")
public ResponseEntity<Void> approveHighValueTransfer(...) { ... }

// Cualquier empleado autenticado puede consultar el estado de una transferencia
@PreAuthorize("hasAnyRole('TELLER', 'BRANCH_SUPERVISOR', 'COMPLIANCE_OFFICER')")
public ResponseEntity<TransferStatus> getTransferStatus(...) { ... }
```

`hasAnyRole('TELLER')` en Spring Security busca la `GrantedAuthority` con nombre `ROLE_TELLER`. El `JwtAuthConverter` del generador antepone `ROLE_` al nombre del rol de Keycloak, haciendo el mapeo automático.

**Flujo completo de autorización — cajero intenta aprobar transferencia de alto valor:**

```
Petición HTTP: POST /transfers/{id}/approve
Authorization: Bearer {jwt con realm_access.roles: ["TELLER"]}
   │
   ▼
JwtDecoder (NimbusJwtDecoder)
   → descarga JWKS de Keycloak (https://auth.bankify.com/realms/banking/...)
   → valida firma RSA-256 con la clave pública del realm
   → valida expiración del token
   │
   ▼
JwtAuthConverter.convert(jwt)
   → realm_access.roles: ["TELLER"]
   → crea SimpleGrantedAuthority("ROLE_TELLER")
   → extrae preferred_username: "juan.garcia" como principal
   │
   ▼
SecurityContext ← Authentication { principal: "juan.garcia", authorities: [ROLE_TELLER] }
   │
   ▼
@PreAuthorize("hasRole('BRANCH_SUPERVISOR')")
   → ROLE_TELLER no es ROLE_BRANCH_SUPERVISOR → AuthorizationDeniedException
   │
   ▼
HandlerExceptions.onAccessDeniedException
   → HTTP 403 Forbidden  {"status": 403, "error": "Forbidden", "message": "Access denied"}
```

---

## Parte 4 — Configuración avanzada

### 4.1 Tiempos de vida de tokens

En la UI: `Realm Settings → Tokens`

| Parámetro | Default | Banca / Salud (PCI-DSS / HIPAA) | SaaS productividad |
|---|---|---|---|
| `Access Token Lifespan` | 5 min | 5–10 min | 15–30 min |
| `Refresh Token Lifespan` | 30 min | 30 min–2 h | 8–24 h |
| `SSO Session Idle` | 30 min | 15 min | 1–4 h |
| `SSO Session Max` | 10 h | 8 h (turno laboral) | 30 días |

**Tradeoffs por industria:**

- **Banca / Salud:** tokens cortos son un requisito de cumplimiento (PCI-DSS, HIPAA). Un access token robado en tránsito tiene ventana de 5 minutos. La sesión SSO expira por inactividad de 15 minutos (estación de trabajo desatendida en un hospital).

- **E-commerce / SaaS:** el usuario espera no tener que re-loguearse durante días. Sesiones largas con refresh tokens de larga duración. El riesgo es menor si el canal es HTTPS y los tokens no tienen permisos de transferencia de dinero.

- **APIs B2B (Client Credentials):** no hay `refresh_token`. El microservicio obtiene un nuevo token cuando el actual expira. Tiempo de vida razonable: 5–15 minutos.

> **Durante desarrollo**, subir `Access Token Lifespan` a 30 min evita la fricción de renovar el token cada 5 minutos en Postman.

---

### 4.2 Protocol Mappers — personalizar el contenido del JWT

Los **Protocol Mappers** controlan qué claims aparecen en el JWT. Acceso: `Clients → {client} → Client Scopes → {scope} → Mappers`.

**Caso de uso 1 — SaaS multi-tenant (plataforma de RRHH):**

El claim `tenant_id` permite que el microservicio de nómina filtre automáticamente los datos de la empresa correcta sin necesidad de un parámetro en cada request.

```
Mapper type: User Attribute
User Attribute: tenant_id
Token Claim Name: tenant_id
Add to access token: ON
```

JWT resultante:
```json
{
  "preferred_username": "sofia.herrera",
  "tenant_id": "empresa-acme-sa",
  "subscription_tier": "enterprise",
  "realm_access": { "roles": ["HR_MANAGER"] }
}
```

En Spring Boot:
```java
String tenantId = jwt.getClaimAsString("tenant_id");
// Usar tenantId para aplicar Row-Level Security en la consulta JPA
```

**Caso de uso 2 — logística, incluir el depósito asignado:**

```
Mapper type: User Attribute
User Attribute: depot_id
Token Claim Name: depot_id
Add to access token: ON
```

El microservicio de ruteo usa `depot_id` para mostrar solo los envíos del depósito del operador, sin pasar el ID por la URL (que podría ser manipulado).

**Caso de uso 3 — salud, incluir el número de licencia médica:**

```
Mapper type: User Attribute
User Attribute: license_number
Token Claim Name: license_number
Add to access token: ON
```

Permite que el sistema de prescripciones electrónicas valide automáticamente que el médico que firma tiene licencia activa, sin una consulta adicional a la base de datos de usuarios.

---

### 4.3 Scopes — control granular de acceso

Un **scope** es una colección de mappers y políticas de acceso que el cliente puede solicitar. En flujos con UI, el usuario da consentimiento explícito a los scopes.

Scopes estándar de OIDC:
- `openid` — obligatorio, activa OIDC, emite `id_token`
- `profile` — incluye `preferred_username`, `given_name`, `family_name`
- `email` — incluye `email`, `email_verified`
- `roles` — incluye `realm_access.roles` (client scope en Keycloak)

**Caso de uso — Open Banking (PSD2):**

Un cliente del banco autoriza a una fintech de terceros a leer solo su saldo y sus últimas transacciones, sin acceso a transferencias ni productos:

```bash
# La fintech solicita solo los scopes necesarios (principio de mínimo privilegio)
GET /auth?client_id=fintech-gastos&scope=openid accounts:read transactions:read
```

El banco (Keycloak) muestra al usuario la pantalla de consentimiento:
```
"Fintech Gastos App" solicita acceso a:
  ✅ Ver su saldo y datos de cuentas
  ✅ Ver sus últimas 90 transacciones
  ❌ No tendrá acceso a: transferencias, créditos, productos
```

El JWT emitido solo contiene los claims del scope aprobado. Si la fintech intenta llamar a `POST /transfers`, obtiene 403 aunque el token sea válido — no tiene el scope `transfers:write`.

> **Importante:** si no se solicita el scope `roles` (o está deshabilitado en el cliente), `realm_access.roles` no aparecerá en el JWT. El `JwtAuthConverter` devolverá una lista vacía y todos los endpoints con `@PreAuthorize` retornarán 403.

---

### 4.4 Importación automática de realm — infraestructura como código

Keycloak soporta importar un realm completo (usuarios, roles, clientes, mappers) al arrancar, leyendo archivos JSON del directorio `/opt/keycloak/data/import/`.

**docker-compose.yaml:**
```yaml
keycloak:
  image: quay.io/keycloak/keycloak:26.3.1
  command: start-dev --import-realm        # flag que activa la importación
  volumes:
    - ./keycloak/realm-export.json:/opt/keycloak/data/import/realm-export.json
  environment:
    KC_BOOTSTRAP_ADMIN_USERNAME: admin
    KC_BOOTSTRAP_ADMIN_PASSWORD: admin
```

**Estructura del archivo de importación — plataforma de e-commerce:**
```json
{
  "realm": "marketplace",
  "enabled": true,
  "ssoSessionIdleTimeout": 3600,
  "accessTokenLifespan": 900,
  "roles": {
    "realm": [
      { "name": "SELLER",         "description": "Vendedor: gestiona sus productos y pedidos" },
      { "name": "BUYER",          "description": "Comprador: navega y realiza compras" },
      { "name": "SUPPORT_AGENT",  "description": "Agente de soporte: ve todos los pedidos y tickets" },
      { "name": "FINANCE",        "description": "Finanzas: accede a reportes y liquidaciones" },
      { "name": "PLATFORM_ADMIN", "description": "Administrador de la plataforma" }
    ]
  },
  "clients": [
    {
      "clientId": "marketplace-web",
      "enabled": true,
      "publicClient": true,
      "directAccessGrantsEnabled": true,
      "redirectUris": ["http://localhost:3000/*"],
      "webOrigins": ["http://localhost:3000"]
    },
    {
      "clientId": "payment-gateway-svc",
      "enabled": true,
      "publicClient": false,
      "secret": "pgw-secret-local",
      "serviceAccountsEnabled": true
    }
  ],
  "users": [
    {
      "username": "seller.demo",
      "enabled": true,
      "credentials": [{ "type": "password", "value": "Seller@2024", "temporary": false }],
      "realmRoles": ["SELLER"]
    },
    {
      "username": "buyer.demo",
      "enabled": true,
      "credentials": [{ "type": "password", "value": "Buyer@2024", "temporary": false }],
      "realmRoles": ["BUYER"]
    },
    {
      "username": "support.demo",
      "enabled": true,
      "credentials": [{ "type": "password", "value": "Support@2024", "temporary": false }],
      "realmRoles": ["SUPPORT_AGENT"]
    },
    {
      "username": "admin.demo",
      "enabled": true,
      "credentials": [{ "type": "password", "value": "Admin@2024", "temporary": false }],
      "realmRoles": ["PLATFORM_ADMIN"]
    }
  ]
}
```

**Comportamiento:** si el realm ya existe al arrancar, Keycloak lo omite (no sobreescribe). Para forzar reimportación: `--import-realm` con la variable de entorno `KC_IMPORT_STRATEGY=OVERWRITE_EXISTING` (Keycloak 26+).

**Cómo exportar el realm desde la UI:**
`Realm Settings → Action (esquina superior derecha) → Export → (activar users) → Export`

---

### 4.5 Eventos y auditoría

Keycloak registra eventos de seguridad. Activar en: `Realm Settings → Events → Event Listeners`.

Eventos relevantes:
- `LOGIN` / `LOGIN_ERROR` — intentos de autenticación (exitosos o fallidos)
- `TOKEN_EXCHANGE` — intercambio de tokens
- `LOGOUT` — cierre de sesión
- `REGISTER` — nuevo usuario registrado

**Caso de uso — cumplimiento regulatorio (PCI-DSS, SOX, HIPAA):**

En una institución financiera o de salud, los reguladores exigen tener un log de auditoría de quién accedió a qué sistema y cuándo. Keycloak puede enviar eventos a un SIEM (Splunk, Elastic) vía el listener `jboss-logging` o un SPI personalizado:

```
LOGIN_ERROR usuario=pedro.montoya ip=185.220.101.43 intentos=5
→ alerta de fuerza bruta → bloqueo temporal de IP en el WAF
```

```
LOGIN usuario=dra.garcia hora=02:14 (horario fuera de turno)
→ alerta de acceso inusual al sistema de expedientes clínicos
→ notificación al oficial de seguridad
```

Para persistir eventos más allá de los 15 días por defecto: configurar el proveedor `jboss-logging` con un appender a ElasticSearch, o implementar un `EventListenerProvider` personalizado que publique a Kafka.

---

### 4.6 Proveedores de identidad externos (Identity Brokering)

Keycloak puede delegar la autenticación a un proveedor externo y luego emitir sus propios tokens JWT internos con los roles de negocio propios.

**Caso de uso 1 — empresa con directorio corporativo (Active Directory / LDAP):**

```
Empleado → portal interno → "Login con cuenta corporativa"
  → Keycloak redirige al Azure AD de la empresa
  → Azure AD autentica con las credenciales de Windows del empleado
  → Keycloak recibe el id_token de Azure AD (email, nombre, grupos LDAP)
  → Keycloak mapea grupos LDAP a roles de Keycloak:
      "CN=RRHH-Managers,OU=Groups" → ROLE_HR_MANAGER
  → Keycloak emite su propio JWT con roles de negocio
```

El empleado nunca crea una segunda contraseña. Los roles en el sistema se gestionan en el directorio corporativo.

**Caso de uso 2 — app de salud para consumidores finales:**

```
Paciente → app → "Continuar con Google"
  → Keycloak redirige a Google OAuth2
  → Google autentica al paciente
  → Keycloak crea usuario local con email de Google (si no existe)
  → Asigna rol PATIENT automáticamente (regla de mapeo en Keycloak)
  → Emite JWT de Keycloak con realm_access.roles: ["PATIENT"]
```

El sistema de salud controla sus propios roles. Si el paciente mañana usa GitHub en lugar de Google, el mismo usuario local de Keycloak recibe el mismo token con el mismo rol `PATIENT`.

Configuración: `Identity Providers → Add provider → Google/Microsoft/OIDC/SAML/LDAP`

---

### 4.7 Fine-Grained Authorization (Políticas avanzadas)

Para modelos de autorización donde el acceso depende de atributos del recurso (no solo del rol), Keycloak ofrece Authorization Services:

| Concepto | Descripción |
|---|---|
| **Resource** | Un recurso protegido (ej: `/api/patients/{id}/records`) |
| **Scope** | Acción sobre el recurso (`read`, `write`, `delete`) |
| **Policy** | Regla que determina acceso (por rol, por usuario, por atributo, por script JS) |
| **Permission** | Asocia Resources + Scopes con Policies |

**Caso de uso 1 — salud: el paciente solo puede ver su propio expediente:**

```
Resource: MedicalRecord
Scope: read
Policy: "Own record" (JavaScript)
  → context.identity.id === resource.owner    ← paciente solo ve el suyo
  → context.identity.hasRole("ATTENDING_PHYSICIAN") ← médico ve todos
Permission: "Read medical record" = MedicalRecord + read + "Own record"
```

**Caso de uso 2 — banca: el cliente solo puede operar sus propias cuentas:**

```
Resource: BankAccount
Scope: transfer
Policy: "Account owner"
  → token.sub === account.customerId
Permission: "Transfer from account" = BankAccount + transfer + "Account owner"
```

Sin esta política, un token válido con rol `CUSTOMER` podría en teoría hacer una transferencia desde la cuenta de otro cliente si conoce el ID. La policy de Keycloak bloquea esto sin necesidad de lógica en el microservicio.

**Caso de uso 3 — logística: el conductor solo puede actualizar sus propios envíos:**

```
Resource: Shipment
Scope: update-status
Policy: "Assigned driver"
  → token.sub === shipment.assignedDriverId
```

> **Cuándo usarlo vs `@PreAuthorize`:** Authorization Services agrega latencia (consulta a Keycloak en cada request) y complejidad operacional. Úsalo cuando la regla de acceso depende de datos del recurso que no están en el token. Para acceso basado puramente en rol, `@PreAuthorize` es suficiente y más eficiente.

---

## Parte 5 — Keycloak en diferentes entornos

### 5.1 Configuración por entorno en este proyecto

El generador produce archivos de configuración separados por entorno:

```
src/main/resources/
├── parameters/
│   ├── local/
│   │   └── auth-server.yaml     ← localhost:8180 hardcodeado (docker-compose)
│   ├── develop/
│   │   └── auth-server.yaml     ← ${AUTH_JWKS_URI} variables de entorno
│   └── production/
│       └── auth-server.yaml     ← ${AUTH_JWKS_URI} variables de entorno
```

**local (`parameters/local/auth-server.yaml`):**
```yaml
auth:
  jwks-uri: http://localhost:8180/realms/dsl-demo/protocol/openid-connect/certs
  issuer-uri: http://localhost:8180/realms/dsl-demo
```

**develop / production (`parameters/develop/auth-server.yaml`):**
```yaml
auth:
  jwks-uri: ${AUTH_JWKS_URI}
  issuer-uri: ${AUTH_ISSUER_URI}
```

Las variables de entorno se inyectan en el contenedor mediante el orquestador (Kubernetes Secrets, AWS ECS task definition, etc.).

---

### 5.2 Diferencias entre start-dev y producción

| Aspecto | `start-dev` | Producción |
|---|---|---|
| Base de datos | H2 en memoria | PostgreSQL / MySQL / Aurora |
| HTTPS | Opcional | Obligatorio (TLS 1.2+) |
| Clustering | No | Infinispan + JGroups / Kubernetes |
| Comando | `start-dev` | `start --optimized` |
| Build JVM | En runtime | Pre-compilado (`kc.sh build`) |
| Performance | ~5 s arranque | ~2 s con build optimizado |

En producción, Keycloak necesita una base de datos externa. Ejemplo con AWS RDS:

```yaml
keycloak:
  image: quay.io/keycloak/keycloak:26.3.1
  command: start --optimized
  environment:
    KC_DB: postgres
    KC_DB_URL: jdbc:postgresql://rds-endpoint.us-east-1.rds.amazonaws.com:5432/keycloak
    KC_DB_USERNAME: keycloak_user
    KC_DB_PASSWORD: ${KC_DB_PASSWORD}          # secreto inyectado por ECS Task Definition
    KC_HTTPS_CERTIFICATE_FILE: /opt/keycloak/conf/tls.crt
    KC_HTTPS_CERTIFICATE_KEY_FILE: /opt/keycloak/conf/tls.key
    KC_HOSTNAME: auth.bankify.com              # dominio público con certificado ACM
    KC_HOSTNAME_STRICT: "true"
    KC_PROXY: edge                             # hay un ALB/Nginx por delante
```

**Alta disponibilidad — Keycloak en Kubernetes:**

```yaml
# 3 réplicas con sticky sessions en el Load Balancer (o sesiones distribuidas con Infinispan)
replicas: 3
env:
  KC_CACHE: ispn                             # Infinispan distribuido
  KC_CACHE_STACK: kubernetes                 # descubrimiento automático de pods
  JAVA_OPTS_APPEND: "-Djgroups.dns.query=keycloak-headless.auth.svc.cluster.local"
```

---

### 5.3 Validación del JWT en Spring Boot — qué pasa internamente

Cuando llega una petición con `Authorization: Bearer {token}`:

1. **`BearerTokenAuthenticationFilter`** extrae el token del header.
2. **`NimbusJwtDecoder`** descarga el JWKS de Keycloak (con caché) y busca la clave pública cuyo `kid` coincide con el header del JWT.
3. Valida la firma RSA-256.
4. Valida `exp` (expiración) y `nbf` (not before).
5. (Opcional) Valida `iss` (issuer) si está configurado `spring.security.oauth2.resourceserver.jwt.issuer-uri`.
6. **`JwtAuthConverter`** (generado por este proyecto) convierte el JWT en un `AbstractAuthenticationToken` con las `GrantedAuthority` extraídas de `realm_access.roles`.
7. El `Authentication` se almacena en `SecurityContextHolder`.
8. Los filtros de autorización evalúan `@PreAuthorize`.

**Si el JWKS endpoint de Keycloak no está disponible** (Keycloak caído), Spring Boot no puede validar tokens y todas las peticiones retornan 401, incluso con tokens válidos.

**Estrategias de resiliencia en producción:**

| Estrategia | Descripción | Industria |
|---|---|---|
| Clúster de Keycloak (3+ nodos) | Alta disponibilidad nativa | Banca, salud |
| Caché de JWKS agresiva en Spring | Ampliar el TTL del cache JWKS (default 5 min) | Cualquiera |
| Caché local de claves públicas | Guardar el JWKS en Redis; usar si Keycloak no responde | E-commerce de alto tráfico |
| Circuit breaker en JwkSetUriJwtDecoder | Fallback a validación offline si el JWKS no está disponible | Servicios críticos |

La librería `spring-security-oauth2-resource-server` cachea el JWKS automáticamente. Para ampliar el TTL:

```java
@Bean
JwtDecoder jwtDecoder() {
    NimbusJwtDecoder decoder = NimbusJwtDecoder
        .withJwkSetUri(jwksUri)
        .jwsAlgorithm(SignatureAlgorithm.RS256)
        .cache(Cache.builder()
            .expireAfterWrite(Duration.ofMinutes(30))  // default: 5 min
            .build())
        .build();
    return decoder;
}
```

---

## Referencia rápida — Comandos útiles

### Obtener token — usuario comprador en marketplace
```bash
curl -s -X POST \
  https://auth.ecommerce-corp.com/realms/marketplace/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password&client_id=marketplace-web&username=buyer.demo&password=Buyer@2024" \
  | jq -r .access_token
```

### Obtener token — microservicio de pagos (Client Credentials)
```bash
curl -s -X POST \
  https://auth.ecommerce-corp.com/realms/marketplace/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=payment-gateway-svc&client_secret=pgw-secret-local" \
  | jq -r .access_token
```

### Inspeccionar token sin herramientas externas
```bash
TOKEN="eyJhbGci..."
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq '{sub, preferred_username, roles: .realm_access.roles, exp}'
```

### Llamar endpoint protegido
```bash
TOKEN=$(curl -s -X POST .../token -d "..." | jq -r .access_token)
curl -H "Authorization: Bearer $TOKEN" \
  https://api.ecommerce-corp.com/v1/orders/ORD-00129483
```

### Verificar que el JWKS está accesible
```bash
curl -s https://auth.ecommerce-corp.com/realms/marketplace/protocol/openid-connect/certs \
  | jq '[.keys[] | {kid, alg, use}]'
```

### Listar usuarios del realm (API admin de Keycloak)
```bash
ADMIN_TOKEN=$(curl -s -X POST http://localhost:8180/realms/master/protocol/openid-connect/token \
  -d "grant_type=password&client_id=admin-cli&username=admin&password=admin" \
  | jq -r .access_token)

curl -H "Authorization: Bearer $ADMIN_TOKEN" \
  "http://localhost:8180/admin/realms/marketplace/users" \
  | jq '[.[] | {username, enabled, roles: .realmRoles}]'
```

### Revocar sesiones de un usuario (logout forzado)
```bash
# Caso de uso: empleado despedido, revocar acceso inmediato
ADMIN_TOKEN=...
USER_ID="uuid-del-usuario"

curl -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  "http://localhost:8180/admin/realms/marketplace/users/$USER_ID/logout"
```
