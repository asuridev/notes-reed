# ¿Es válido que un endpoint de tipo `command` retorne algo distinto de `void`?

**Respuesta corta:** Sí. Es una práctica aceptada y ampliamente usada en la industria. La postura "un command nunca devuelve nada" es un ideal teórico que en la práctica casi nadie sigue de forma estricta.

---

## De dónde viene la regla estricta

- **CQS (Command-Query Separation)**, de Bertrand Meyer: un método o *muta* o *devuelve*, nunca ambos. Es una regla a nivel de método/objeto en el estilo OO.
- **CQRS** (Greg Young) escala esa idea a nivel de arquitectura: separas el modelo de escritura del de lectura. Pero CQRS trata de *separar modelos*, no de prohibir que un handler de comando retorne datos.

El propio Greg Young y Udi Dahan han aclarado repetidamente que CQRS **no** obliga a que los commands devuelvan `void`. Esa es la confusión más común del patrón.

---

## Qué hace la industria realmente

- **REST / HTTP**: la guía canónica (y RFC 7231) es que un `POST` que crea un recurso devuelva `201 Created` **con el recurso en el body** y una cabecera `Location`. Un `PUT`/`PATCH` que actualiza suele devolver `200` con el estado resultante. Devolver el recurso mutado es la norma, no la excepción.
- **gRPC / APIs RPC**: prácticamente toda RPC de mutación devuelve un mensaje de respuesta (el objeto creado, un id, un status). El Google API Design Guide lo asume por defecto.
- **GraphQL**: las *mutations* devuelven un payload por diseño; el patrón recomendado (Relay) es que devuelvan el objeto modificado para refrescar la caché del cliente.
- **MediatR (.NET), Axon (Java)**: `IRequest<TResponse>` — los commands son genéricos sobre un tipo de retorno precisamente porque devolver algo es lo habitual.

---

## Dónde SÍ se mantiene el `void`

Donde el estilo estricto sigue vivo es en arquitecturas **event-sourced / asíncronas puras**: el command se despacha *fire-and-forget*, y el resultado llega por otro canal (un evento, una proyección que consultas después). Ahí el `void` (o solo un ack de "aceptado", `202 Accepted`) es la buena práctica, porque acoplar la respuesta al resultado rompería la asincronía.

---

## Regla práctica de la industria

La convención madura no es "command = void", sino:

- ✅ Un command **puede** devolver el **resultado directo de su propia mutación** (el recurso creado/actualizado, su id, su nuevo estado) → buena práctica.
- ❌ Un command **no** debe usarse para *consultar* datos ajenos a esa mutación ni exponerse como mecanismo de lectura → eso sí viola CQS y es un *code smell*. Para eso está la query.

---

## Cómo lo trata Keel

Keel adopta exactamente esta línea. La distinción `command`/`query` es sobre **efecto** (`command` muta estado, `query` solo lee), **no** sobre la forma de la respuesta. El campo `output` de cualquier operación admite las tres formas por igual:

```
"void"  |  { fields: {...} }  |  { entity: X }   (con list, paginated, exclude)
```

Ejemplos canónicos del DSL (`docs/dsl/use-cases.md`):

```yaml
createProduct:
  kind: command
  output: { entity: Product }   # command que retorna el agregado creado — válido y esperado

reconcilePrices:
  kind: command
  input: "void"
  output: "void"                # command fire-and-forget disparado por schedule
```

**En resumen:** retornar algo distinto de `void` en un command es aceptable y esperado; lo que define al command es que *muta estado*, no la forma de su salida. El `void` se reserva para lo genuinamente fire-and-forget (schedules, subscriptions), no como dogma.
