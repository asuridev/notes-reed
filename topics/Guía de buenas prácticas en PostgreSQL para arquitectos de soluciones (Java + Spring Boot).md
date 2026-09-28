# Guía de buenas prácticas en PostgreSQL para arquitectos de soluciones (Java + Spring Boot)

Sep 27, 2026 · @Antonio

## 1. Introducción: optimizar antes de escalar

La mayoría de los problemas de rendimiento en PostgreSQL se resuelven con buen modelado, índices correctos y consultas bien escritas, no con réplicas ni particionado. Esta guía recorre, en ese orden, todo lo que un arquitecto de soluciones debe verificar antes de añadir complejidad de infraestructura, con ejemplos para Java y Spring Boot (Spring Data JPA, Hibernate 6, HikariCP y Flyway).

Réplicas y particionado resuelven problemas reales, pero traen costos permanentes: consistencia eventual, enrutamiento de conexiones, claves de partición que condicionan todas las consultas y más superficie operativa. Si se aplican sobre una base mal indexada, solo multiplican el problema.

&#91;embedded content: orden de optimización · 7 pasos, 1 decisión\]

Cada vuelta del ciclo empieza midiendo; solo cuando los pasos 1 a 4 están agotados y el SLO sigue sin cumplirse se escala la infraestructura.

**Principios que guían todo el documento:**

- **Medir antes de cambiar.** Ninguna optimización sin un plan de ejecución o una métrica que la justifique.
- **La base de datos es un recurso compartido.** Una consulta lenta no solo afecta a su endpoint: consume conexiones, CPU y memoria de todos.
- **Traer solo lo que se usa.** Columnas, filas y relaciones: cada byte innecesario cuesta en disco, red y memoria de la JVM.
- **La base protege la integridad.** Las reglas críticas viven en constraints, no solo en el código Java.
- **El ORM es una herramienta, no una abstracción total.** Hay que saber qué SQL genera Hibernate y cuándo conviene escribirlo a mano.

## 2. Modelado de datos

Un buen modelo es la optimización más barata: los errores de tipos, claves y constraints se pagan en cada consulta durante toda la vida del sistema. Normaliza hasta tercera forma normal por defecto y desnormaliza solo cuando una métrica lo justifique.

### Normalización y desnormalización

- **Normalizar** evita duplicidad e inconsistencias de actualización. Es el punto de partida en sistemas transaccionales (OLTP).
- **Desnormalizar** es válido cuando un cálculo caro se lee mucho más de lo que se escribe: un contador de pedidos por cliente, un total de factura o un nombre copiado para reportes.
- Alternativas antes de desnormalizar a mano: columnas generadas (`GENERATED ALWAYS AS (...) STORED`), vistas materializadas (sección 6) o `jsonb` para atributos variables.

### Tipos de datos recomendados

| Dato | Tipo PostgreSQL | Tipo Java | Evitar |
| --- | --- | --- | --- |
| Identificador interno | `bigint` con identity o secuencia | `Long` | `int` (se agota), `serial` (legado) |
| Identificador público | `uuid` (idealmente v7) | `UUID` | `uuid` guardado como `text` o `varchar` |
| Dinero | `numeric(19,4)` | `BigDecimal` | `float`, `double`, `money` |
| Fecha y hora | `timestamptz` | `Instant` u `OffsetDateTime` | `timestamp` sin zona |
| Solo fecha | `date` | `LocalDate` | fechas en `text` |
| Texto | `text` + `CHECK` de longitud | `String` | `char(n)` |
| Atributos flexibles | `jsonb` | clase o `Map` con `@JdbcTypeCode(SqlTypes.JSON)` | `json`, modelos EAV |
| Estado o categoría | `text` + `CHECK`, o tabla catálogo | `enum` con `@Enumerated(EnumType.STRING)` | `EnumType.ORDINAL` |

En PostgreSQL, `text` y `varchar(n)` rinden igual; `varchar(n)` solo añade una validación de longitud. `timestamptz` guarda el instante en UTC y evita errores de zona horaria entre servicios.

### Claves primarias

- **`bigint` identity o secuencia**: compacta (8 bytes), ordenada y amigable para índices B-tree. Es la opción por defecto.
- **UUID v4 (aleatorio)**: útil para generar IDs fuera de la base o exponerlos públicamente, pero sus inserciones aleatorias fragmentan el índice y degradan la caché en tablas grandes.
- **UUID v7**: ordenado por tiempo, combina lo mejor de ambos. PostgreSQL 18 incluye la función `uuidv7()`; en versiones anteriores se genera en Java.
- Patrón común: `bigint` como PK interna y un `uuid` único como identificador público en la API.

### Constraints: la última línea de defensa

Bean Validation (`@NotNull`, `@Size`) protege la API, pero no protege contra otro servicio, un script o una migración. Declara en la base:

- `NOT NULL` en todo lo obligatorio (además ayuda al planificador).
- `UNIQUE` para reglas de negocio (email, número de documento).
- `CHECK` para rangos y estados válidos.
- `FOREIGN KEY` para integridad referencial. PostgreSQL **no** crea índices en la columna que referencia; hay que crearlos (sección 3).

```sql
CREATE TABLE orders (
    id           bigint PRIMARY KEY,  -- lo asigna Hibernate desde orders_seq (ver abajo)
    public_id    uuid        NOT NULL UNIQUE,
    customer_id  bigint      NOT NULL REFERENCES customers(id),
    status       text        NOT NULL CHECK (status IN ('NEW','PAID','SHIPPED','CANCELLED')),
    total        numeric(19,4) NOT NULL CHECK (total >= 0),
    created_at   timestamptz NOT NULL DEFAULT now(),
    updated_at   timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
```

### Mapeo en JPA

Con `GenerationType.IDENTITY`, Hibernate necesita el ID tras cada `INSERT` y **desactiva el batching** de inserciones. Para cargas masivas usa una secuencia cuyo `INCREMENT BY` coincida con `allocationSize`:

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "orders_seq")
    @SequenceGenerator(name = "orders_seq", sequenceName = "orders_seq", allocationSize = 50)
    private Long id;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status;

    @Column(nullable = false, precision = 19, scale = 4)
    private BigDecimal total;

    private Instant createdAt;
}
```

```sql
CREATE SEQUENCE orders_seq INCREMENT BY 50;
```

Convenciones útiles: nombres en `snake_case`, columnas de auditoría (`created_at`, `updated_at`) y el esquema definido por migraciones (sección 13), nunca por `ddl-auto=update`.

## 3. Índices

Un índice correcto convierte una lectura de millones de filas en unas pocas páginas; uno innecesario ralentiza cada escritura. Se diseñan a partir de las consultas reales (sección 12), no de las columnas de la tabla.

### Cómo funcionan

Un índice es una estructura aparte que apunta a las filas de la tabla. PostgreSQL decide si lo usa según sus estadísticas: si la consulta devuelve una fracción grande de la tabla, leerla completa (Seq Scan) es más barato que saltar por el índice. Por eso un índice sobre una columna poco selectiva (un booleano, un estado con 3 valores) casi nunca se usa por sí solo.

### Tipos de índice

| Tipo | Cuándo usarlo | Ejemplo |
| --- | --- | --- |
| B-tree (por defecto) | Igualdad, rangos, `ORDER BY`, `LIKE 'abc%'` | `CREATE INDEX ON orders (created_at);` |
| Compuesto | Filtros sobre varias columnas a la vez | `CREATE INDEX ON orders (customer_id, created_at);` |
| Parcial | Solo un subconjunto de filas se consulta | `CREATE INDEX ON orders (created_at) WHERE status = 'NEW';` |
| De expresión | Se filtra por una función de la columna | `CREATE INDEX ON users (lower(email));` |
| Cubriente (`INCLUDE`) | Permitir Index Only Scan sin ir a la tabla | `CREATE INDEX ON orders (customer_id) INCLUDE (total, status);` |
| GIN | `jsonb`, arrays, búsqueda de texto, `pg_trgm` | `CREATE INDEX ON products USING gin (attributes jsonb_path_ops);` |
| GiST | Rangos, geometría (PostGIS), exclusión de solapamientos | `CREATE INDEX ON bookings USING gist (period);` |
| BRIN | Tablas enormes con datos físicamente ordenados (logs, eventos por fecha) | `CREATE INDEX ON events USING brin (created_at);` |
| Hash | Solo igualdad; rara vez supera a B-tree | — |

### Reglas para índices compuestos

El orden de las columnas importa: el índice sirve para filtros que usan su **prefijo izquierdo**.

1. Columnas con **igualdad** primero (`customer_id = ?`).
2. Luego la columna de **rango** u **orden** (`created_at > ?`, `ORDER BY created_at DESC`).
3. Un índice `(a, b)` sirve para `WHERE a = ?` y `WHERE a = ? AND b = ?`, pero no para `WHERE b = ?` solo.
4. Si existe `(a, b)`, un índice solo sobre `(a)` suele ser redundante.

```sql
-- Consulta: últimos pedidos de un cliente
SELECT id, total, status FROM orders
WHERE customer_id = $1
ORDER BY created_at DESC
LIMIT 20;

-- Índice que la resuelve sin ordenar en memoria y sin tocar la tabla
CREATE INDEX CONCURRENTLY idx_orders_customer_created
    ON orders (customer_id, created_at DESC) INCLUDE (total, status);
```

### Errores que anulan un índice

- Aplicar una función a la columna: `WHERE date(created_at) = '2026-09-01'` no usa el índice en `created_at`. Reescribe como rango: `created_at >= '2026-09-01' AND created_at < '2026-09-02'`.
- Comodines al inicio: `LIKE '%perez'` no usa B-tree. Usa la extensión `pg_trgm` con un índice GIN.
- Tipos distintos: comparar un `bigint` con un parámetro `numeric` o `text` puede forzar conversiones.
- `OR` entre columnas distintas: a veces conviene reescribir con `UNION ALL`.
- Búsquedas sin distinguir mayúsculas: `lower(email) = lower(?)` requiere un índice de expresión sobre `lower(email)`.

### El costo de indexar

- Cada índice se actualiza en cada `INSERT`, en cada `DELETE` y en los `UPDATE` que tocan sus columnas.
- Un índice sobre una columna que se actualiza mucho impide las actualizaciones HOT (Heap-Only Tuple), que son más baratas.
- Los índices ocupan disco y memoria compartida. Revisa periódicamente los que no se usan (consulta en la sección 12) y los duplicados.

### Crear índices en producción

`CREATE INDEX` normal bloquea las escrituras en la tabla mientras se construye. En producción usa siempre `CREATE INDEX CONCURRENTLY`, que tarda más pero no bloquea. No puede ejecutarse dentro de una transacción y, si falla, deja un índice `INVALID` que hay que borrar y volver a crear. La sección 13 explica cómo hacerlo con Flyway.

### Índices y JPA

`@Table(indexes = @Index(columnList = "customer_id, created_at"))` solo se usa si Hibernate genera el esquema. En producción los índices se definen en migraciones SQL, donde puedes usar `CONCURRENTLY`, `INCLUDE`, índices parciales y de expresión. Recuerda indexar las columnas de las `@ManyToOne`/`@JoinColumn`: se usan en joins y en los borrados en cascada.

## 4. Leer planes de ejecución con EXPLAIN

`EXPLAIN (ANALYZE, BUFFERS)` es la herramienta central de diagnóstico: muestra cómo PostgreSQL ejecutó realmente la consulta, cuánto tardó cada paso y cuántas páginas leyó. Sin él, cualquier índice nuevo es una suposición.

### Obtener el SQL real desde Spring Boot

Primero hay que ver qué SQL genera Hibernate y con qué parámetros (solo en entornos de desarrollo o pruebas):

```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE   # valores de los parámetros (Hibernate 6)
```

Luego se ejecuta en `psql` o DBeaver con valores representativos:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total, status FROM orders
WHERE customer_id = 4521
ORDER BY created_at DESC
LIMIT 20;
```

`EXPLAIN ANALYZE` **ejecuta** la consulta. Para un `UPDATE` o `DELETE`, envuélvelo en `BEGIN; ... ROLLBACK;`.

### Cómo leer el plan

- El plan es un árbol: se lee de adentro hacia afuera; los nodos más indentados se ejecutan primero.
- `cost=inicio..total` es una estimación en unidades internas; `actual time` es el tiempo real en milisegundos.
- `rows` estimadas frente a `actual rows`: si difieren en un orden de magnitud o más, el planificador está decidiendo con estadísticas malas.
- `loops`: el tiempo y las filas se multiplican por este número.
- `Buffers: shared hit` son páginas en memoria; `read` son páginas leídas del disco.

### Nodos más comunes

| Nodo | Qué significa | ¿Preocupa? |
| --- | --- | --- |
| Seq Scan | Lee toda la tabla | Normal en tablas pequeñas; alerta si la tabla es grande y el filtro devuelve pocas filas |
| Index Scan | Recorre el índice y va a la tabla por cada fila | Bien para pocas filas |
| Index Only Scan | Responde solo con el índice | Ideal; requiere `INCLUDE` o columnas en el índice y tabla bien vaciada |
| Bitmap Heap Scan | Junta punteros del índice y luego lee la tabla en orden | Bien para volúmenes medios |
| Nested Loop | Por cada fila externa, busca en la interna | Bien si el lado externo es pequeño y el interno está indexado |
| Hash Join | Construye una tabla hash con un lado | Bien para volúmenes grandes sin índice útil |
| Merge Join | Une dos entradas ya ordenadas | Bien con índices ordenados en ambos lados |
| Sort | Ordena en memoria o en disco | Alerta si dice `external merge Disk` (falta `work_mem` o un índice) |

### Señales de alerta

- Seq Scan sobre una tabla grande con `Rows Removed by Filter` muy alto: falta un índice o no se está usando.
- Estimación de filas muy distinta a la real: ejecuta `ANALYZE tabla;` o aumenta las estadísticas.
- Nested Loop con muchos `loops` sobre un Seq Scan interno: falta un índice en la columna del join.
- Sort o Hash que usan disco: revisa `work_mem` (sección 11) o crea un índice que entregue el orden.

### Mejorar las estadísticas

```sql
-- Más detalle para una columna con distribución irregular
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;

-- Columnas correlacionadas (ciudad y país, por ejemplo)
CREATE STATISTICS st_addr_city_country (dependencies) ON city, country FROM addresses;
ANALYZE addresses;
```

Herramientas como [explain.depesz.com](https://explain.depesz.com) o [explain.dalibo.com](https://explain.dalibo.com) muestran el plan de forma visual y marcan los nodos más caros.

## 5. Consultas y proyecciones

Traer solo las columnas y filas que el caso de uso necesita es la optimización con mejor relación esfuerzo-beneficio en una aplicación JPA. Cargar entidades completas para mostrar tres campos gasta red, memoria de la JVM y tiempo de Hibernate en el dirty checking.

### Evitar SELECT \*

- Menos columnas = menos bytes leídos y transferidos.
- Permite Index Only Scan cuando las columnas están en un índice cubriente.
- Evita traer columnas pesadas (`text` largo, `jsonb`, `bytea`) que no se usan.
- Las entidades JPA son para **modificar** datos; las proyecciones, para **leerlos**.

### Proyecciones en Spring Data JPA

**1. Proyección por interfaz (cerrada).** Spring genera un `SELECT` solo con esas columnas:

```java
public interface OrderSummary {
    Long getId();
    BigDecimal getTotal();
    OrderStatus getStatus();
}

public interface OrderRepository extends JpaRepository<Order, Long> {
    List<OrderSummary> findByCustomerIdOrderByCreatedAtDesc(Long customerId, Limit limit);
}
```

Evita las proyecciones "abiertas" con `@Value("#{target...}")`: cargan la entidad completa.

**2. Proyección por clase o `record` (DTO).** Recomendada: inmutable y sin proxies:

```java
public record OrderSummaryDto(Long id, BigDecimal total, OrderStatus status) {}

@Query("""
    select new com.acme.orders.OrderSummaryDto(o.id, o.total, o.status)
    from Order o
    where o.customer.id = :customerId
    order by o.createdAt desc
    """)
List<OrderSummaryDto> findSummaries(Long customerId, Pageable pageable);
```

**3. Proyección dinámica.** Un solo método, distintas vistas del dato:

```java
<T> List<T> findByStatus(OrderStatus status, Class<T> type);
```

**4. SQL nativo con proyección.** Para CTE, funciones de ventana o SQL específico de PostgreSQL:

```java
@Query(value = """
    select c.id as customerId, c.name as name, sum(o.total) as totalSpent
    from customers c join orders o on o.customer_id = c.id
    where o.created_at >= :since
    group by c.id, c.name
    order by totalSpent desc
    limit 10
    """, nativeQuery = true)
List<TopCustomer> findTopCustomers(Instant since);
```

### Paginación: OFFSET frente a keyset

`Page<T>` usa `LIMIT ... OFFSET ...` y además ejecuta un `count(*)` en cada página. Con `OFFSET 100000`, PostgreSQL lee y descarta 100.000 filas: cada página es más lenta que la anterior.

| Opción | Cuándo | Costo |
| --- | --- | --- |
| `Page<T>` | Tablas pequeñas, necesitas el total de páginas | `count(*)` extra + OFFSET creciente |
| `Slice<T>` | Scroll "cargar más" sin total | Sin `count`, OFFSET creciente |
| Keyset (`Window<T>` / `ScrollPosition`) | Tablas grandes, APIs, feeds, exportaciones | Constante en cualquier página |

Keyset filtra por el último valor visto en lugar de saltar filas:

```sql
SELECT id, total FROM orders
WHERE customer_id = $1 AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

En Spring Data 3.1+ se usa con `ScrollPosition.keyset()` y el tipo de retorno `Window<T>`. Necesita un orden determinista (incluye el `id` como desempate) y un índice que coincida con ese orden.

### El problema N+1

Ocurre cuando se carga una lista y luego se accede a una relación perezosa de cada elemento: 1 consulta para la lista + N consultas para las relaciones. Es la causa más frecuente de lentitud en aplicaciones JPA.

Soluciones:

- **`JOIN FETCH`** en JPQL: `select o from Order o join fetch o.customer where ...`.
- **`@EntityGraph`** en el repositorio: `@EntityGraph(attributePaths = {"customer", "items"})`.
- **Batch fetching**: `spring.jpa.properties.hibernate.default_batch_fetch_size=50` agrupa las cargas perezosas en consultas `IN (...)`.
- **Proyecciones DTO**, que traen exactamente lo necesario en una consulta.

Cuidado: `JOIN FETCH` sobre una colección combinado con paginación hace que Hibernate pagine **en memoria** (advertencia `HHH90003004`). Pagina primero los IDs y luego carga las relaciones de esos IDs.

### Otras buenas prácticas de consulta

- Usa `existsBy...` en lugar de `countBy... > 0`: `EXISTS` se detiene en la primera fila.
- Operaciones masivas con `@Modifying @Query("update ...")` en lugar de cargar entidades una por una.
- Evita `IN` con miles de valores; usa `= ANY(?)` con un array o una tabla temporal.
- Usa siempre parámetros enlazados: nunca concatenes valores en el SQL (rendimiento del plan en caché y prevención de inyección SQL).
- Aprovecha SQL de PostgreSQL cuando simplifica: `INSERT ... ON CONFLICT` (upsert), `RETURNING`, CTE y funciones de ventana.

## 6. Vistas y vistas materializadas

Una vista normal no mejora el rendimiento: es una consulta guardada que se expande cada vez que se usa. Una vista materializada sí lo mejora, porque guarda el resultado en disco, a cambio de datos que pueden estar desactualizados.

| Aspecto | Vista (`VIEW`) | Vista materializada (`MATERIALIZED VIEW`) |
| --- | --- | --- |
| Almacena datos | No | Sí, como una tabla |
| Frescura | Siempre actual | Hasta el último `REFRESH` |
| Rendimiento | Igual a la consulta base | Lectura rápida, refresco costoso |
| Admite índices | No (usa los de las tablas base) | Sí |
| Uso típico | Encapsular joins, contratos de lectura, seguridad | Dashboards, reportes, agregados caros |

### Vistas normales: cuándo sí

- **Encapsular complejidad**: joins repetidos se definen una vez y la aplicación consulta algo simple.
- **Contrato estable**: el esquema interno cambia sin romper a los consumidores de la vista.
- **Seguridad**: exponer solo ciertas columnas o filas a un rol. Desde PostgreSQL 15, `WITH (security_invoker = true)` hace que la vista respete los permisos y el Row Level Security de quien consulta.

Riesgos: vistas construidas sobre otras vistas generan consultas enormes y difíciles de optimizar; un `ORDER BY` dentro de la vista añade un ordenamiento inútil; y usar una vista con muchos joins para leer una sola columna obliga a ejecutar joins que no se necesitan. PostgreSQL solo elimina un `LEFT JOIN` innecesario si puede demostrar, por una restricción `UNIQUE` o PK, que no cambia el resultado.

### Vistas materializadas

```sql
CREATE MATERIALIZED VIEW mv_sales_daily AS
SELECT date_trunc('day', created_at) AS day,
       count(*)   AS orders,
       sum(total) AS revenue
FROM orders
WHERE status <> 'CANCELLED'
GROUP BY 1;

-- Obligatorio para refrescar sin bloquear lecturas
CREATE UNIQUE INDEX ON mv_sales_daily (day);

-- Refresco que no bloquea a quienes leen
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_sales_daily;
```

- `REFRESH` sin `CONCURRENTLY` bloquea las lecturas durante el refresco.
- `CONCURRENTLY` requiere un índice único y es más lento, pero no interrumpe a los usuarios.
- El refresco recalcula todo: si la consulta base tarda 10 minutos, cada refresco tarda al menos eso. Para agregados que crecen, considera una tabla resumen actualizada incrementalmente.
- Programa el refresco con `@Scheduled` en Spring (con un bloqueo distribuido como ShedLock si hay varias instancias) o con la extensión `pg_cron`.
- Define con negocio cuánta desactualización es aceptable (por ejemplo, 15 minutos para un dashboard).

### Mapeo en Spring Boot

Una vista se mapea como una entidad de solo lectura:

```java
@Entity
@Immutable                       // Hibernate nunca genera UPDATE
@Table(name = "mv_sales_daily")
public class SalesDaily {
    @Id
    private Instant day;
    private long orders;
    private BigDecimal revenue;
}

public interface SalesDailyRepository extends Repository<SalesDaily, Instant> {
    List<SalesDaily> findByDayBetweenOrderByDay(Instant from, Instant to);
}
```

Extiende `Repository` en lugar de `JpaRepository` para no exponer `save` ni `delete`. Hibernate también ofrece `@Subselect` para mapear una consulta sin crear la vista en la base, aunque es más difícil de mantener y optimizar.

```java
@Service
@RequiredArgsConstructor
class SalesDailyRefresher {
    private final JdbcTemplate jdbc;

    @Scheduled(cron = "0 */15 * * * *")
    @SchedulerLock(name = "refreshSalesDaily")   // ShedLock, evita refrescos duplicados
    void refresh() {
        jdbc.execute("REFRESH MATERIALIZED VIEW CONCURRENTLY mv_sales_daily");
    }
}
```

## 7. Buenas prácticas con JPA e Hibernate

La configuración por defecto de Spring Boot prioriza la comodidad sobre el rendimiento. Estos ajustes deberían estar en todo proyecto desde el primer día.

### Configuración base recomendada

```yaml
spring:
  datasource:
    url: jdbc:postgresql://db:5432/app?reWriteBatchedInserts=true
  jpa:
    open-in-view: false            # no mantener la sesión abierta en la vista
    hibernate:
      ddl-auto: validate           # el esquema lo gestiona Flyway
    properties:
      hibernate:
        jdbc:
          batch_size: 50
          time_zone: UTC
        order_inserts: true
        order_updates: true
        default_batch_fetch_size: 50
        query:
          in_clause_parameter_padding: true   # reutiliza planes con IN de tamaño variable
          fail_on_pagination_over_collection_fetch: true
```

### Relaciones y carga perezosa

- Declara **todas** las relaciones como `LAZY`. `@ManyToOne` y `@OneToOne` son `EAGER` por defecto y provocan joins o consultas que nadie pidió.
- Decide qué cargar en cada caso de uso con `JOIN FETCH`, `@EntityGraph` o proyecciones (sección 5).
- Evita colecciones `@OneToMany` que pueden crecer sin límite (los pedidos de un cliente, los eventos de un dispositivo). Consúltalas con un repositorio paginado.
- Con `open-in-view: false` aparecerán `LazyInitializationException`: es una señal útil de que un caso de uso no declaró qué datos necesita.

### Transacciones de solo lectura

```java
@Service
@Transactional(readOnly = true)          // por defecto en la clase
public class OrderQueryService { ... }

@Transactional                            // escritura explícita donde aplica
public void placeOrder(...) { ... }
```

`readOnly = true` hace que Hibernate omita el dirty checking y el flush, y permite enrutar la consulta a una réplica en el futuro (sección 16).

### Escrituras masivas

- El batching necesita secuencias (no `IDENTITY`), `batch_size` y, en PostgreSQL, `reWriteBatchedInserts=true` en la URL JDBC.
- En lotes grandes, llama a `entityManager.flush()` y `entityManager.clear()` cada N entidades para no llenar el contexto de persistencia.
- Para cargas de cientos de miles de filas, usa `JdbcTemplate.batchUpdate` o `COPY` (con `CopyManager` del driver): son órdenes de magnitud más rápidos que JPA.

### Cuándo salir de JPA

JPA es excelente para el modelo de dominio y operaciones CRUD. Para lo demás, usa `JdbcTemplate`, `JdbcClient` (Spring 6.1+) o jOOQ:

- Reportes con agregaciones, CTE o funciones de ventana.
- Upserts con `INSERT ... ON CONFLICT`.
- Operaciones masivas y cargas de datos.
- Consultas donde necesitas control total del SQL y del plan.

### Otros detalles que importan

- Implementa `equals` y `hashCode` basados en el identificador de forma estable, o en una clave natural; nunca en todas las columnas.
- La caché de segundo nivel de Hibernate solo tiene sentido para entidades de lectura frecuente y cambio raro (catálogos). Mal usada, causa datos obsoletos entre instancias.
- El driver `pgjdbc` convierte en sentencia preparada del servidor una consulta repetida 5 veces (`prepareThreshold`). Si usas PgBouncer en modo transacción, revisa la compatibilidad (sección 9).
- Prueba con PostgreSQL real usando Testcontainers, no con H2: los planes, tipos y funciones difieren.

## 8. Transacciones, concurrencia y bloqueos

Las transacciones deben ser cortas, hacer solo trabajo de base de datos y tener un tiempo límite. Una transacción larga retiene bloqueos, impide que VACUUM limpie filas muertas y agota el pool de conexiones.

### MVCC en una página

PostgreSQL usa control de concurrencia multiversión (MVCC): cada `UPDATE` crea una versión nueva de la fila y deja la anterior como "muerta" hasta que VACUUM la limpia. Consecuencias prácticas:

- Las lecturas no bloquean a las escrituras y viceversa.
- Dos escrituras sobre la **misma fila** sí se bloquean entre sí.
- Tablas con muchos `UPDATE` generan filas muertas (bloat) y dependen de un autovacuum sano (sección 10).

### Niveles de aislamiento

| Nivel | Qué garantiza | Cuándo usarlo |
| --- | --- | --- |
| Read Committed (por defecto) | Cada sentencia ve los datos confirmados al empezar esa sentencia | La mayoría de casos, junto con bloqueo optimista |
| Repeatable Read | Toda la transacción ve la misma foto de los datos | Reportes consistentes, lecturas de varias tablas relacionadas |
| Serializable | Resultado equivalente a ejecutar las transacciones una tras otra | Reglas de negocio complejas entre filas; exige reintentos |

Con Repeatable Read y Serializable, PostgreSQL aborta una transacción en conflicto con el código `40001`: la aplicación debe reintentarla.

### Bloqueo optimista con @Version

Es la opción por defecto para ediciones concurrentes. No bloquea nada; detecta el conflicto al guardar:

```java
@Entity
public class Account {
    @Id private Long id;
    private BigDecimal balance;
    @Version private long version;   // UPDATE ... WHERE id = ? AND version = ?
}
```

Si otra transacción modificó la fila, Hibernate lanza `OptimisticLockException`: se reintenta o se informa al usuario.

### Bloqueo pesimista y colas de trabajo

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)            // SELECT ... FOR UPDATE
@Query("select a from Account a where a.id = :id")
Optional<Account> findForUpdate(Long id);
```

Para procesar tareas en paralelo desde varias instancias sin que dos tomen la misma, usa `FOR UPDATE SKIP LOCKED`:

```sql
SELECT id, payload FROM jobs
WHERE status = 'PENDING'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

### Deadlocks

Ocurren cuando dos transacciones bloquean recursos en orden inverso. PostgreSQL detecta el ciclo y aborta una con el código `40P01`. Prevención: bloquear siempre en el mismo orden (por ejemplo, por `id` ascendente) y mantener las transacciones cortas.

### Timeouts: la red de seguridad

Configúralos por rol en la base, para que apliquen aunque la aplicación falle:

```sql
ALTER ROLE app_user SET statement_timeout = '30s';
ALTER ROLE app_user SET lock_timeout = '5s';
ALTER ROLE app_user SET idle_in_transaction_session_timeout = '60s';
```

- `statement_timeout`: corta consultas desbocadas.
- `lock_timeout`: evita esperar indefinidamente por un bloqueo.
- `idle_in_transaction_session_timeout`: cierra sesiones que abrieron una transacción y la dejaron colgada.

En Spring también puedes usar `@Transactional(timeout = 10)` para límites por caso de uso.

### Errores comunes en Spring

- **Llamadas externas dentro de `@Transactional`**: una llamada HTTP de 5 segundos mantiene la conexión y los bloqueos ocupados esos 5 segundos. Llama primero al servicio externo o usa el patrón outbox.
- **Autoinvocación**: un método que llama a otro `@Transactional` de la misma clase no pasa por el proxy y la anotación se ignora.
- **Reintentos**: los errores `40001` y `40P01` son transitorios; reintenta la transacción completa con Spring Retry y un número limitado de intentos.

```java
@Retryable(retryFor = {CannotAcquireLockException.class, OptimisticLockingFailureException.class},
           maxAttempts = 3, backoff = @Backoff(delay = 50, multiplier = 2))
@Transactional
public void transfer(Long from, Long to, BigDecimal amount) { ... }
```

## 9. Pool de conexiones

Un pool pequeño y bien dimensionado rinde más que uno grande. En PostgreSQL cada conexión es un proceso del sistema operativo con su propia memoria, y cientos de conexiones activas compiten por CPU y bloqueos en lugar de avanzar más rápido.

### Dimensionar HikariCP

- Punto de partida sugerido por la documentación de HikariCP: `conexiones = (núcleos del servidor de BD × 2) + discos efectivos`. Para un servidor de 8 núcleos con SSD, unas 17 a 20 conexiones activas en **total**.
- Ese total se reparte entre todas las instancias: `maximumPoolSize × número de instancias ≤ max_connections − conexiones reservadas` (administración, migraciones, monitoreo).
- Si hay hilos esperando conexión, casi nunca la solución es subir el pool: primero acorta las transacciones y optimiza las consultas lentas.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 10              # pool fijo: sin picos de creación de conexiones
      connection-timeout: 3000      # ms; falla rápido si no hay conexiones
      max-lifetime: 1680000         # ms; menor que cualquier timeout de red o del proxy
      idle-timeout: 600000
      leak-detection-threshold: 20000   # avisa si una conexión no se devuelve en 20 s
      pool-name: orders-pool
```

### Cuando hay muchas instancias: PgBouncer

Con autoescalado (Kubernetes, por ejemplo), 30 pods × 10 conexiones = 300 conexiones, aunque la mayoría estén ociosas. Un pooler externo como PgBouncer multiplexa muchas conexiones de clientes sobre pocas conexiones reales:

| Modo | Cómo funciona | Consideraciones |
| --- | --- | --- |
| Session | Una conexión real por sesión del cliente | Compatible con todo, ahorra poco |
| Transaction | La conexión real se asigna solo durante cada transacción | El más usado; no admite `SET` de sesión, `LISTEN` ni bloqueos advisory de sesión |
| Statement | Por cada sentencia | Rara vez aplicable a aplicaciones Spring |

En modo transacción, las sentencias preparadas del servidor requieren PgBouncer 1.21 o superior con `max_prepared_statements` configurado; con versiones anteriores, añade `prepareThreshold=0` a la URL JDBC. Los servicios gestionados suelen ofrecer su propio pooler (RDS Proxy, el pooler de Azure Flexible Server o el de Cloud SQL).

### Métricas a vigilar

Con Spring Boot Actuator y Micrometer, HikariCP publica métricas automáticamente:

- `hikaricp.connections.pending`: hilos esperando conexión. Debería ser casi siempre 0.
- `hikaricp.connections.usage`: cuánto tiempo se retiene cada conexión. Revela transacciones largas.
- `hikaricp.connections.acquire`: tiempo para obtener una conexión.
- `hikaricp.connections.timeout`: número de timeouts; cualquier valor mayor que 0 merece investigación.

## 10. Mantenimiento: VACUUM, ANALYZE y bloat

Autovacuum no se desactiva nunca; se ajusta. Es el proceso que mantiene sanas las tablas bajo MVCC, y cuando se queda atrás, las consultas se degradan poco a poco sin que ningún código haya cambiado.

### Qué hace cada operación

| Operación | Qué hace | Bloquea |
| --- | --- | --- |
| `VACUUM` | Marca el espacio de filas muertas como reutilizable y actualiza el visibility map (necesario para Index Only Scan) | No bloquea lecturas ni escrituras |
| `ANALYZE` | Actualiza las estadísticas que usa el planificador | No |
| `VACUUM FULL` | Reescribe la tabla completa y devuelve espacio al sistema operativo | **Sí, bloqueo exclusivo**: evitar en producción |
| `REINDEX CONCURRENTLY` | Reconstruye un índice inflado sin bloquear (PostgreSQL 12+) | No |
| `pg_repack` (extensión) | Alternativa a `VACUUM FULL` sin bloqueo prolongado | Mínimo |

VACUUM también previene el **wraparound** de identificadores de transacción. Si se descuida durante mucho tiempo, PostgreSQL termina forzando un vacuum de emergencia e incluso deja de aceptar escrituras para protegerse.

### Ajustar autovacuum en tablas grandes

Por defecto, autovacuum actúa cuando cambia alrededor del 20 % de una tabla (`autovacuum_vacuum_scale_factor = 0.2`). En una tabla de 100 millones de filas eso significa esperar 20 millones de filas muertas. Ajústalo por tabla:

```sql
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor  = 0.02,
    autovacuum_analyze_scale_factor = 0.01
);
```

A nivel de servidor suele convenir subir `autovacuum_max_workers` y `autovacuum_vacuum_cost_limit` para que termine más rápido en máquinas con discos SSD.

### Actualizaciones HOT y fillfactor

En tablas con muchos `UPDATE` de columnas no indexadas (estados, contadores), un `fillfactor` menor deja espacio libre en cada página para que la nueva versión quepa en la misma página (actualización HOT), sin tocar los índices:

```sql
ALTER TABLE jobs SET (fillfactor = 85);
```

### Monitorear la salud de las tablas

```sql
-- Tablas con más filas muertas y último vacuum
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 1) AS pct_dead,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

- Un porcentaje de filas muertas alto y sostenido indica que autovacuum no alcanza o que algo lo bloquea.
- El bloqueo más común: una transacción abierta durante horas (revisa `pg_stat_activity`), un slot de replicación abandonado o una transacción preparada olvidada.
- Después de una carga masiva o un borrado grande, ejecuta `ANALYZE tabla;` manualmente.

## 11. Configuración del servidor

La configuración por defecto de PostgreSQL está pensada para arrancar en máquinas muy pequeñas, no para producción. Los valores siguientes son puntos de partida aproximados para un servidor dedicado con SSD; valídalos siempre con métricas y con la documentación de tu proveedor.

| Parámetro | Qué controla | Punto de partida |
| --- | --- | --- |
| `shared_buffers` | Caché propia de PostgreSQL | \~25 % de la RAM |
| `effective_cache_size` | Estimación de memoria disponible para caché (no reserva nada) | 50–75 % de la RAM |
| `work_mem` | Memoria por operación de ordenamiento o hash, **por nodo y por conexión** | 16–64 MB; subir por sesión para reportes |
| `maintenance_work_mem` | Memoria para VACUUM, `CREATE INDEX` | 512 MB–1 GB |
| `random_page_cost` | Costo relativo de lectura aleatoria | 1.1 con SSD (el valor por defecto, 4, asume discos mecánicos) |
| `effective_io_concurrency` | Lecturas concurrentes que el disco admite | 200 con SSD |
| `max_wal_size` | Cuánto WAL se acumula entre checkpoints | 4–16 GB según volumen de escritura |
| `checkpoint_completion_target` | Reparte la escritura del checkpoint en el tiempo | 0.9 |
| `max_connections` | Conexiones máximas | 100–300, con pooler delante |

### Cómo razonar sobre work\_mem

`work_mem` no es un límite por conexión: una consulta con tres ordenamientos y dos hash joins puede usar hasta 5 × `work_mem`, multiplicado por las conexiones activas. Un valor global alto con muchas conexiones puede agotar la RAM. La estrategia segura es un valor global moderado y subirlo solo donde se necesita:

```sql
-- Solo para la transacción de un reporte pesado
SET LOCAL work_mem = '256MB';
```

Activa `log_temp_files = 0` para ver qué consultas escriben archivos temporales en disco por falta de `work_mem`.

### Servicios gestionados

En Amazon RDS o Aurora, Azure Database for PostgreSQL o Google Cloud SQL, estos parámetros se cambian mediante grupos de parámetros o flags, y muchos ya vienen ajustados al tamaño de la instancia. Revisa especialmente `random_page_cost`, `work_mem` y la configuración de autovacuum, que suelen quedar en valores genéricos. Herramientas como [PGTune](https://pgtune.leopard.in.ua) generan una primera propuesta a partir de la RAM, los núcleos y el tipo de carga.

## 12. Observabilidad

`pg_stat_statements` es la extensión más importante que puedes activar: agrupa todas las consultas normalizadas y muestra cuáles consumen más tiempo total. Optimizar las 10 primeras de esa lista suele dar más resultado que cualquier cambio de infraestructura.

### Activar la instrumentación en PostgreSQL

```ini
# postgresql.conf (o grupo de parámetros del servicio gestionado)
shared_preload_libraries = 'pg_stat_statements,auto_explain'
pg_stat_statements.track = all
log_min_duration_statement = 500ms     # registra consultas lentas
log_lock_waits = on                    # registra esperas de bloqueo > deadlock_timeout
log_temp_files = 0                     # consultas que usan disco temporal
log_autovacuum_min_duration = 1s
auto_explain.log_min_duration = 2s     # guarda el plan de las consultas muy lentas
auto_explain.log_analyze = off         # on da más detalle pero añade sobrecarga
```

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### Consultas de diagnóstico esenciales

**Top de consultas por tiempo total:**

```sql
SELECT round(total_exec_time::numeric, 0) AS total_ms,
       calls,
       round(mean_exec_time::numeric, 2)  AS mean_ms,
       rows,
       left(query, 120) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Una consulta de 5 ms llamada 2 millones de veces pesa más que una de 3 segundos llamada 10 veces. Ordenar por tiempo total, no por tiempo medio, muestra dónde está el costo real (y suele delatar un N+1).

**Sesiones activas, transacciones largas y bloqueos:**

```sql
SELECT pid, state, now() - xact_start AS xact_age, wait_event_type, wait_event,
       pg_blocking_pids(pid) AS blocked_by, left(query, 100) AS query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY xact_start NULLS LAST;
```

**Índices que nunca se usan:**

```sql
SELECT s.relname AS table, s.indexrelname AS index,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size, s.idx_scan
FROM pg_stat_user_indexes s
JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE s.idx_scan = 0 AND NOT i.indisunique
ORDER BY pg_relation_size(s.indexrelid) DESC;
```

Antes de borrar un índice, confirma que las estadísticas cubren un periodo representativo (cierres de mes, procesos trimestrales) y revisa también las réplicas, que tienen sus propias estadísticas.

### Métricas de base de datos para el tablero

- Tasa de aciertos de caché (`blks_hit / (blks_hit + blks_read)` en `pg_stat_database`); en OLTP suele estar por encima del 99 %.
- Transacciones por segundo, conexiones activas frente a `max_connections`.
- Duración de la transacción abierta más antigua.
- Filas muertas y actividad de autovacuum por tabla.
- Uso de CPU, IOPS, latencia de disco y crecimiento de almacenamiento.
- Deadlocks y archivos temporales.

### Del lado de Spring Boot

- **Actuator + Micrometer**: métricas de HikariCP y latencia por endpoint.
- **OpenTelemetry**: trazas que muestran cada consulta dentro de la petición que la originó.
- **Detectar N+1 en pruebas**: `datasource-proxy` o `hibernate.generate_statistics=true` (solo fuera de producción) para contar consultas por operación y fallar el test si aumentan.
- **Testcontainers** con PostgreSQL real para pruebas de integración y para revisar planes con datos de volumen realista.

## 13. Migraciones de esquema sin downtime

El esquema se versiona con Flyway o Liquibase junto al código, y cada cambio se diseña para que la versión anterior y la nueva de la aplicación funcionen al mismo tiempo durante el despliegue.

### Reglas básicas con Flyway

- Scripts versionados en `src/main/resources/db/migration` (`V12__add_order_channel.sql`).
- Nunca edites una migración ya aplicada; crea una nueva.
- `spring.jpa.hibernate.ddl-auto=validate`: Hibernate verifica que las entidades coinciden con el esquema, pero no lo modifica.
- Considera ejecutar las migraciones como un paso separado del pipeline (un Job de Kubernetes, por ejemplo) con un usuario propietario del esquema, distinto al usuario de la aplicación.
- Empieza cada migración con `SET lock_timeout = '5s';`: si la tabla está ocupada, la migración falla rápido en lugar de hacer cola y bloquear todo el tráfico detrás de ella.

### Patrón expand/contract

Para cambios incompatibles, como renombrar una columna:

1. **Expand**: añadir la columna nueva (anulable).
2. Desplegar código que escribe en ambas columnas.
3. Rellenar datos históricos por lotes (por ejemplo, 10.000 filas por transacción).
4. Desplegar código que lee de la columna nueva.
5. **Contract**: en un despliegue posterior, eliminar la columna antigua.

### Operaciones peligrosas y su alternativa segura

| Operación | Riesgo | Alternativa segura |
| --- | --- | --- |
| `CREATE INDEX` | Bloquea escrituras | `CREATE INDEX CONCURRENTLY` en una migración sin transacción |
| `ADD COLUMN ... DEFAULT` volátil (`now()`, `random()`) | Reescribe la tabla | Default constante (instantáneo desde PostgreSQL 11) o rellenar por lotes |
| `ALTER COLUMN ... SET NOT NULL` | Recorre toda la tabla con bloqueo | `ADD CONSTRAINT ... CHECK (col IS NOT NULL) NOT VALID`, luego `VALIDATE CONSTRAINT`, luego `SET NOT NULL` |
| `ADD FOREIGN KEY` | Valida todas las filas con bloqueo | `ADD CONSTRAINT ... NOT VALID` y luego `VALIDATE CONSTRAINT` |
| `ALTER COLUMN TYPE` | Suele reescribir la tabla e índices | Columna nueva + expand/contract |
| `RENAME COLUMN` | Rompe la versión anterior de la aplicación | Expand/contract |
| `UPDATE` masivo en una sola transacción | Bloqueos largos, bloat, WAL enorme | Lotes pequeños con commits intermedios |

### Índices concurrentes con Flyway

`CREATE INDEX CONCURRENTLY` no puede ejecutarse dentro de una transacción. Ponlo en su propia migración, sin otras sentencias. En versiones recientes de Flyway puedes declararlo con un archivo de configuración junto al script (`V13__idx_orders_customer.sql.conf` con `executeInTransaction=false`); revisa la documentación de tu versión.

```sql
-- V13__idx_orders_customer.sql
SET lock_timeout = '5s';
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_customer_created
    ON orders (customer_id, created_at DESC);
```

Si la creación falla, el índice queda `INVALID`: una migración posterior debe hacer `DROP INDEX CONCURRENTLY` y volver a crearlo.

## 14. Caché y reducción de carga

La consulta más rápida es la que no se ejecuta. Antes de añadir réplicas para absorber lecturas, elimina las lecturas repetidas y los datos que ya nadie consulta.

### Niveles de caché

| Nivel | Herramienta | Ideal para | Cuidado con |
| --- | --- | --- | --- |
| Local en la JVM | Caffeine con Spring Cache | Catálogos, configuración, datos que cambian poco | Cada instancia tiene su copia; usa TTL cortos |
| Distribuida | Redis o Valkey con Spring Cache | Datos compartidos entre instancias, sesiones, resultados costosos | Invalidación y consistencia; un salto de red extra |
| Segundo nivel de Hibernate | JCache con Ehcache o Infinispan | Entidades de solo lectura o lectura muy frecuente | Datos obsoletos si otro sistema escribe en la base |
| HTTP | `Cache-Control`, ETag, CDN | Respuestas públicas o por usuario que cambian poco | Datos personalizados o sensibles |

```java
@Cacheable(cacheNames = "countries")
public List<CountryDto> findAllCountries() { ... }

@CacheEvict(cacheNames = "product", key = "#product.id")
public void update(Product product) { ... }
```

Reglas prácticas: define siempre un TTL, decide explícitamente la estrategia de invalidación, no caches objetos enormes por usuario y mide la tasa de aciertos. Una caché con pocos aciertos solo añade complejidad.

### Archivado y retención de datos

Muchas tablas crecen porque nadie definió cuánto tiempo se guardan los datos. Acordar una política de retención con negocio y cumplimiento suele evitar el particionado:

- Mueve datos históricos (pedidos cerrados de hace más de N años, logs, eventos) a una tabla de archivo, a almacenamiento de objetos o a un data lake.
- Borra por lotes pequeños en horarios de baja carga para no generar bloqueos ni bloat masivo.
- Si el borrado por fecha es frecuente y voluminoso, es una señal fuerte para el particionado por rango de fechas (sección 16).

### Separar cargas que compiten

- **Reportes y analítica**: tablas resumen, vistas materializadas o una réplica dedicada a BI; a largo plazo, un almacén analítico alimentado por CDC (Debezium, por ejemplo).
- **Picos de escritura**: una cola (Kafka, RabbitMQ, SQS) que absorbe el pico y escribe a ritmo constante.
- **Integraciones**: el patrón outbox publica eventos a partir de una tabla dentro de la misma transacción, en lugar de que otros sistemas consulten la base directamente.

## 15. Seguridad y resiliencia básica

La aplicación debe conectarse con el mínimo privilegio necesario, por un canal cifrado y con respaldos probados. Estas bases deben existir antes de hablar de réplicas: una réplica no es un backup, porque replica también los borrados accidentales.

### Roles y privilegios

```sql
-- Propietario del esquema: solo lo usan las migraciones
CREATE ROLE app_owner LOGIN;
CREATE SCHEMA orders AUTHORIZATION app_owner;

-- Usuario de la aplicación: solo DML
CREATE ROLE app_user LOGIN;
GRANT USAGE ON SCHEMA orders TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA orders TO app_user;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA orders TO app_user;
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA orders
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;

-- Usuario de solo lectura para reportes o soporte
CREATE ROLE app_readonly LOGIN;
GRANT USAGE ON SCHEMA orders TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA orders TO app_readonly;
```

- Nunca conectes la aplicación con un superusuario ni con el propietario del esquema.
- Un rol por servicio facilita auditar quién hace qué y aplicar timeouts por rol (sección 8).

### Conexión y secretos

- TLS obligatorio: `sslmode=verify-full` en la URL JDBC valida el certificado del servidor.
- Credenciales en un gestor de secretos (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault) integrado con Spring Cloud, con rotación periódica. Nunca en el repositorio.
- Parámetros enlazados en todo SQL: nunca concatenes entradas del usuario en `@Query(nativeQuery = true)`, `JdbcTemplate` ni en nombres de columna para ordenar (usa una lista blanca).

### Multi-tenant y auditoría

- **Row Level Security (RLS)** permite que la base filtre las filas por inquilino, como defensa adicional al filtro en el código. Requiere fijar el inquilino por transacción (`SET LOCAL app.tenant_id = ...`) y tener cuidado con el pooling.
- La extensión `pgaudit` registra accesos y cambios para requisitos de cumplimiento.

### Respaldos y recuperación

- Respaldos físicos con recuperación a un punto en el tiempo (PITR): pgBackRest, Barman o la función nativa del servicio gestionado.
- Define el RPO (cuántos datos puedes perder) y el RTO (cuánto tiempo puedes estar sin servicio) con negocio.
- **Prueba la restauración** periódicamente. Un respaldo que nunca se restauró no es un respaldo confiable.

## 16. Checklist final y cuándo escalar

Solo cuando todos los puntos de esta lista están cubiertos y las métricas siguen mostrando saturación, tiene sentido hablar de réplicas o particionado.

### Checklist antes de escalar

**Modelo y consultas**

- [ ] Tipos de datos correctos, PK `bigint` o UUID v7, constraints y FKs declarados.
- [ ] Todas las FKs tienen índice.
- [ ] Las 10 consultas con mayor tiempo total en `pg_stat_statements` fueron revisadas con `EXPLAIN (ANALYZE, BUFFERS)`.
- [ ] No hay Seq Scan inesperados sobre tablas grandes en consultas frecuentes.
- [ ] Lecturas con proyecciones DTO; sin `SELECT *` innecesarios.
- [ ] Sin N+1 en los flujos principales (verificado en pruebas).
- [ ] Paginación keyset en listados grandes y APIs.
- [ ] Índices no usados y duplicados eliminados.

**Aplicación**

- [ ] `open-in-view: false`, relaciones `LAZY`, `ddl-auto: validate`.
- [ ] `@Transactional(readOnly = true)` en lecturas; sin llamadas externas dentro de transacciones.
- [ ] Batching configurado para escrituras masivas.
- [ ] HikariCP dimensionado; `connections.pending` cercano a 0.
- [ ] Caché para datos de lectura frecuente y cambio raro.

**Servidor y operación**

- [ ] Parámetros de memoria y costos ajustados al hardware.
- [ ] Autovacuum ajustado en tablas grandes; sin transacciones de horas.
- [ ] Timeouts por rol configurados.
- [ ] `pg_stat_statements`, logs de consultas lentas y tablero de métricas activos.
- [ ] Migraciones sin downtime; `CREATE INDEX CONCURRENTLY` en producción.
- [ ] Política de retención y archivado aplicada.
- [ ] Respaldos con PITR y restauración probada.

### Señales para el siguiente paso

| Señal medida | Siguiente paso |
| --- | --- |
| CPU o memoria sostenidamente altas después de optimizar, con carga mixta | Escalar verticalmente (más CPU, RAM o IOPS): es el paso más simple y a menudo suficiente |
| Las lecturas dominan la carga (por ejemplo, más del 80 %) y el primario sigue saturado | Réplicas de lectura |
| Reportes o analítica que degradan las transacciones | Réplica dedicada a reportes o almacén analítico |
| Alta disponibilidad con conmutación automática exigida por el RTO | Réplica en espera (standby) en otra zona |
| Tablas de cientos de millones de filas donde VACUUM, índices y borrados ya no son manejables | Particionado |
| Borrado periódico de datos antiguos por fecha | Particionado por rango de fechas |
| Escrituras que superan la capacidad de un solo servidor ya escalado | Sharding (Citus, por ejemplo): la opción de mayor complejidad |

### Lo que traen las réplicas de lectura

- **Retraso de replicación**: un usuario puede no ver lo que acaba de escribir si la siguiente lectura va a la réplica. Los flujos que leen justo después de escribir deben ir al primario.
- **Enrutamiento en Spring**: `AbstractRoutingDataSource` con `LazyConnectionDataSourceProxy` para elegir primario o réplica según `@Transactional(readOnly = true)`. Aquí se aprovecha el haber marcado bien las transacciones de lectura desde el inicio.
- Consultas largas en la réplica pueden cancelarse por conflictos con la replicación; revisa `hot_standby_feedback` y `max_standby_streaming_delay`.

### Lo que trae el particionado

- Beneficia a las consultas que filtran por la clave de partición (partition pruning); las que no la incluyen recorren todas las particiones y pueden ser más lentas.
- La PK y las restricciones `UNIQUE` deben incluir la clave de partición, lo que afecta al mapeo JPA.
- Borrar datos antiguos pasa a ser `DETACH` o `DROP` de una partición: instantáneo y sin bloat.
- Requiere crear particiones futuras con antelación (manualmente o con la extensión `pg_partman`).

### Lecturas recomendadas

- [Documentación oficial de PostgreSQL: índices](https://www.postgresql.org/docs/current/indexes.html)
- [Documentación oficial: uso de EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [Documentación oficial: VACUUM rutinario](https://www.postgresql.org/docs/current/routine-vacuuming.html)
- [Documentación oficial: particionado de tablas](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- [HikariCP: About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [Use The Index, Luke](https://use-the-index-luke.com) (índices y SQL para desarrolladores)

Los valores numéricos de esta guía son puntos de partida aproximados basados en prácticas comunes; confirma los detalles específicos de cada versión de PostgreSQL, Hibernate y Spring Boot en su documentación oficial.
