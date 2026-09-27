# Servicio de pago para varias pasarelas

**Respuesta corta:** sí se puede, con una condición y tres concesiones. La condición es que lo general sea el **dominio**, no la pasarela: el núcleo común es grande y se deja expresar entero. Las concesiones son el precio de cubrir a la vez las dos formas de interacción que existen —tokenizada servidor-a-servidor y de redirección—, y hay que aceptarlas explícitamente porque cada una empeora algo.

Lo que **no** funciona es la intuición más natural: «una capa de pagos agnóstica, y la pasarela se elige al desplegar, como se elige S3 o MinIO». Las pasarelas no son intercambiables de esa manera, y la sección 5 explica por qué.

El alcance de esta nota es **cobro y devolución**, con pasarelas tokenizadas (Stripe, Adyen) y de redirección (Redsys y las bancarias equivalentes).

---

## 1. Por qué estos servicios degeneran

Un servicio de pago compartido suele morir por una de dos causas, y conviene nombrarlas antes de nada porque todo lo que sigue está ordenado para evitarlas.

**La primera es la de siempre en un servicio compartido:** acaba conociendo el dominio de sus llamantes. Empieza cobrando pedidos, luego cobra suscripciones, luego cobra matrículas, y para entonces tiene un `if` por tipo de cosa cobrada. La cura es la misma que en cualquier servicio compartido: su contrato pertenece al dominio «pago» —un importe, una divisa, una referencia opaca— y nunca al dominio de quien lo llama.

**La segunda es propia de los pagos, y es más cara:** el servicio se escribe contra la pasarela que había el primer día, y la forma de esa pasarela se filtra al dominio. Cuando llega la segunda —y llega, porque cambian las condiciones comerciales, o se entra en otro país— no hay un adaptador que sustituir: hay un servicio que reescribir. La señal de que ha ocurrido es concreta: el estado del pago en nuestra base de datos tiene los nombres que usa la pasarela.

> **Idea clave:** la frontera correcta no está entre «nuestro código» y «la API de la pasarela». Está entre **el hecho de negocio** (se autorizó un cobro de 42,50 €) y **el relato de la pasarela sobre ese hecho** (un `Ds_Response` de `0000`, o un `charge.succeeded`). Todo lo que cruce esa frontera hacia dentro es deuda.

---

## 2. Lo que sí es común entre todas las pasarelas

Más de lo que parece, y es la razón de que el ejercicio valga la pena. El núcleo invariante:

| Pieza | Qué resuelve |
|---|---|
| Agregado `Payment` con su máquina de estados | La verdad de nuestro lado sobre cada cobro |
| Idempotencia del llamante | Que el comprador que da dos veces al botón no pague dos veces |
| Idempotencia saliente | Que **nuestro** reintento contra la pasarela no cobre dos veces |
| Deduplicación de la notificación entrante | Que la reentrega del webhook no aplique el desenlace dos veces |
| Devolución como operación de vuelta | Deshacer trabajo ya encargado a un tercero |
| Barrido de reconciliación | El desenlace que **nunca llega** |

Los cuatro primeros parecen el mismo mecanismo y son cuatro distintos, con almacenes distintos. Confundirlos declara garantías que nada implementa: la clave que evita el doble clic del comprador no tiene nada que ver con la que evita que nuestro reintento duplique el cargo, y ninguna de las dos frena la reentrega del webhook.

### El barrido no es opcional

De todas las piezas, la que más se omite y más cara sale es la última. En un cobro, «la pasarela autorizó y el webhook nunca llegó» **no produce ningún hecho**: ni excepción, ni evento, ni línea de log. El pago se queda en `authorizing` para siempre. El sistema no está roto, está callado — y lo que hay al otro lado de ese silencio es dinero cobrado al cliente que nosotros creemos no cobrado.

Lo único que puede ver algo que no ocurrió es algo que corre solo. Hace falta un proceso periódico que busque pagos atascados en el estado de espera y les pregunte a la pasarela por su estado real. Y necesita tres cosas, no una:

1. Un **estado que signifique «esperando»** (`authorizing`).
2. Una **marca de cuándo empezó a esperar**, en un campo propio. No vale `createdAt` (el pago pudo crearse mucho antes de intentarse) ni `updatedAt` (rejuvenece con cualquier otra escritura, y el barrido no lo alcanza nunca — es el fallo que pasa todas las pruebas y falla en producción).
3. Un **umbral** configurable, no una constante en el código.

---

## 3. Las dos formas de interacción

**Tokenizada (Stripe, Adyen).** Creamos un intent en el servidor con importe, divisa y clave de idempotencia. O se resuelve en la misma llamada, o la autenticación reforzada (3DS/SCA) devuelve una acción pendiente y el desenlace llega después.

**De redirección (Redsys y la mayoría de bancarias).** Firmamos unos parámetros, el navegador del pagador **sale de nuestro sitio** hacia el banco, y el desenlace real llega por una notificación servidor-a-servidor.

Y aquí la regla que más se incumple en integraciones reales: **la vuelta del navegador a la URL de OK no es autoritativa**. Es una pantalla, no un hecho. El usuario puede cerrar la pestaña con el cobro ya hecho, o volver por la URL de KO con el cobro hecho igualmente. Da la falsa sensación de funcionar durante todas las pruebas manuales, porque en una prueba manual nadie cierra la pestaña.

### Dónde convergen de verdad

En que **todo pago pasa por un estado de autorización en curso**, y lo único que cambia es cuánto dura: en el caso tokenizado, microsegundos; en el de redirección, minutos. Visto así, **el caso de redirección es el general y el tokenizado es su forma degenerada** — no al revés, que es como suele modelarse y por eso no encaja después.

```
pending ──▶ authorizing ──▶ authorized ──▶ captured ──▶ refunded | partiallyRefunded
                  │
                  └──────▶ failed | expired
```

La operación de inicio devuelve una **acción siguiente**:

| Acción | Cuándo | Qué lleva |
|---|---|---|
| `none` | Ya resuelto en la propia llamada | Nada |
| `redirect` | El pagador tiene que ir a una URL | La URL |
| `formPost` | Hay que enviar un formulario firmado | URL + campos |

Las tres hacen falta: una redirección simple no expresa un `POST` con parámetros firmados, que es exactamente lo que necesita Redsys. Un único contrato de salida cubre así las dos familias. Es la forma a la que Stripe convergió después de soportar los dos mundos, así que no es una invención de esta nota.

---

## 4. Las tres concesiones

Cubrir las dos formas con un solo diseño no sale gratis. Cada concesión empeora algo, y lo importante es aceptarlas por escrito en vez de descubrirlas.

**1. El caso síncrono paga el precio del asíncrono.** Ningún integrador puede asumir que el pago está hecho cuando la llamada vuelve: todos tienen que tratar la acción siguiente, incluso los que usan una pasarela que resuelve en el acto. Es el coste real de «general».

**2. El webhook es la única fuente autoritativa.** La vuelta del navegador solo sirve para pintar una pantalla. Tiene que estar escrito en el diseño, porque el camino de menor resistencia de cualquiera que lo implemente es confiar en la vuelta — que es la que sí se ve al probar a mano.

**3. El vocabulario de desenlaces es deliberadamente grueso.** Un conjunto cerrado y normalizado —`insufficientFunds`, `rejectedByIssuer`, `authenticationFailed`, `expired`, `technical`— es lo único que lee la lógica de negocio. El código crudo de la pasarela se guarda en un campo opaco, para soporte, y **nunca** se usa para decidir. Los motivos de rechazo del emisor son decenas y no mapean uno a uno entre pasarelas; cualquier lógica que dependa del código crudo se rompe al cambiar de proveedor, que es justo lo que este ejercicio intenta evitar.

---

## 5. Lo que ninguna modelización arregla

Tres asimetrías reales entre pasarelas. No son problemas de diseño: son diferencias de producto, y conviene conocerlas antes de prometer que el servicio es general.

**Modelo de captura.** Con captura inmediata, autorizar y capturar ocurren en el mismo instante: `authorized` es un estado que esa pasarela nunca ocupa de forma observable. Con captura diferida es un estado real, con caducidad (la autorización expira en N días). El lifecycle general incluye los dos; hay que declarar por pasarela cuál aplica, no dejarlo implícito.

**Anular no es devolver.** Antes de la liquidación se anula: es gratis e inmediato. Después se devuelve: tiene coste y tarda días. Unificarlos en una sola operación «devolver» esconde una distinción que el negocio sí nota en la factura. Si el alcance es «cobro y devolución», decidir esto **es** parte del alcance, y es una decisión de negocio, no técnica.

**Autorización incremental** —subir el importe de una autorización viva— existe en las tokenizadas y no en las de redirección, donde la firma cubre el importe y cambiarlo la invalida. Si el negocio la necesita, el modelo general no la expresa y hay que salirse de él.

---

## 6. Por qué la pasarela no puede ser una elección de despliegue

La intuición natural es tratar la pasarela como se trata el almacenamiento de archivos: el diseño declara «guardo archivos» en abstracto y al desplegar se elige S3, MinIO o Azure Blob. Lo mismo con el correo: el diseño declara «mando correo» y el relay concreto se elige después.

Esa abstracción funciona ahí por una razón que **no se cumple en pagos**: todos los proveedores de almacenamiento hacen lo mismo. Poner un objeto, recuperarlo, firmar una URL. Un relay SMTP es un relay SMTP. La elección es de producto, no de comportamiento.

Las pasarelas no comparten semántica: una redirige y otra no, una tiene captura diferida y otra no, cada una firma su notificación a su manera. Una capa «agnóstica de pasarela» tendría que exponer como parámetros la acción siguiente, el modelo de captura y el esquema de firma — y en ese punto ya no está abstrayendo la pasarela, la está volviendo a describir con otros nombres.

**Lo que sí funciona es lo contrario:** un diseño base con el dominio completo, del que se deriva una variante por pasarela. Lo que cambia entre variantes es el contrato del tercero y la forma de su notificación; el dominio, los casos de uso, la persistencia y los eventos son prácticamente idénticos. Aislar el contrato del tercero tras una capa de anticorrupción es exactamente el trabajo para el que esa capa existe: si el tercero cambia su respuesta, solo cambia el adaptador.

### ¿Y elegir la pasarela en caliente?

Enrutar por país, por inquilino o por importe dentro de un mismo despliegue es una necesidad legítima —sobre todo con presencia en varios países—, pero es más una decisión de **topología** que de diseño del servicio. Lo más honesto de partida es un despliegue por pasarela, o que el enrutado viva por encima de este servicio. Convertirlo en «un puerto con N adaptadores intercambiables» solo se sostiene si las N pasarelas comparten de verdad el modelo de captura y la forma de interacción; si no, el punto de conmutación acaba teniendo un `if` por pasarela, que es lo que se quería evitar.

---

## 7. El webhook: el hueco que casi nadie declara

Es lo más serio de toda la nota, y no es incidental — es el canal por el que entra la verdad del dinero.

Los modelos de autorización habituales de un servicio se apoyan todos en un token: público, autenticado, administrador, cliente máquina. **Un webhook de pasarela no trae token.** Su autenticidad es un HMAC sobre el **cuerpo crudo** con un secreto compartido, más una ventana temporal contra reenvíos.

Declararlo «público» no es una simplificación: es una **afirmación falsa del contrato**. Dice que cualquiera puede llamarlo, y en ese endpoint concreto «cualquiera» significa que cualquiera puede fabricar un cobro confirmado. Tampoco vale tratarlo como cliente máquina con clave de API: ahí el llamante presenta *nuestra* clave, y aquí es la pasarela la que firma con la *suya*.

Y hay un detalle de implementación que se pierde en cuanto no se declara: la firma es sobre el **cuerpo crudo**, byte a byte. Cualquier capa que deserialice y vuelva a serializar antes de verificar rompe la comprobación — o peor, la hace pasar sobre algo que no es lo que firmó la pasarela.

> **Idea clave:** un webhook es el gemelo HTTP de una suscripción a un broker, y necesita exactamente su mismo vocabulario: un identificador de mensaje para deduplicar la reentrega, una identidad de quién lo publica con la asunción que la sostiene, y una política de fallo con reintento y descarte. No lo tiene porque entra por la puerta de la API, que se diseñó para llamantes a los que **autenticamos**, no para publicadores a los que **verificamos**.

Esto no afecta solo a pagos: cualquier servicio que reciba notificaciones de un tercero —mensajería, firma electrónica, logística— tiene el mismo agujero.

---

## 8. El pagador no es una dependencia como las demás

Un servicio depende de otros de dos maneras: le **lee un dato** o le **encarga trabajo**. Las dos son servidor-a-servidor, y con las dos el otro lado es un sistema que responde.

Un pago con redirección no es ninguna de las dos. Le entregamos trabajo al **navegador de un tercero** —el pagador, que no es un sistema, no responde y puede simplemente irse— y el resultado vuelve por un canal completamente distinto, en otro momento y sin correlación con la petición original más allá de una referencia que nosotros mismos generamos.

Es lo que hace que un pago no se parezca a ninguna otra integración, y es la razón de fondo de casi todo lo anterior: del estado `authorizing`, del barrido de reconciliación, de que la vuelta del navegador no sea autoritativa. Merece decirse explícitamente en el diseño, porque quien no lo tenga presente modelará el pago como una llamada que devuelve un resultado, y esa forma no admite arreglo incremental después.

---

## 9. Probar sin cobrar: simuladores y stubs

Sí existen imágenes de Docker que dicen simular pasarelas de pago. Ordenadas de mayor a menor decepción, y conviene conocer el porqué antes de montar nada.

**`stripe/stripe-mock`** es la única imagen oficial de una pasarela grande, y su propio README la descarta para este uso:

> *"does not attempt to reproduce the **behavior** of the real Stripe API at all"* · *"stripe-mock is stateless. Data you send on a `POST` request will be validated, but it will be completely ignored beyond that"* · *"**Testing for specific responses and errors is currently not supported. It will return a success response instead of the desired error response.**"*

La última frase la descalifica sola. En un servicio de pago los caminos que importan son los rechazos: fondos insuficientes, rechazo del emisor, desafío 3DS, autorización caducada. stripe-mock devuelve éxito en todos ellos. Está pensada para que los mantenedores de los SDK comprueben que se llama a la URL correcta con los parámetros correctos, y para nada más.

**El CLI de Stripe** (`stripe listen`, `stripe trigger`) sí dispara webhooks reales y bien firmados, pero contra el entorno de pruebas real: necesita red y credenciales. Sirve para explorar a mano, no para una suite en integración continua.

**Adyen, PayPal y Redsys no publican imagen.** Lo que ofrecen es un entorno de pruebas remoto con credenciales de comercio. Redsys es el peor caso: ni simulador, ni contenedor, ni forma de reproducir su esquema de firma sin implementarlo uno mismo.

### Por qué no existe un simulador fiel, y no es un descuido

El comportamiento de la pasarela **es** el producto. Lo que un simulador tendría que reproducir con fidelidad es justo lo que nadie fuera del proveedor puede: la taxonomía de rechazos del emisor, cuándo se dispara el desafío 3DS, el retardo y el orden de los webhooks, la ventana de anulación frente a devolución, y el esquema de firma.

> **Idea clave:** un simulador fiel de una pasarela sería una segunda implementación de la pasarela — y si existiera, habría que probarlo a él. Por eso la vía que funciona no es simular la pasarela, sino **programar sus respuestas escenario a escenario**.

### Lo que sí funciona: stubs programables

Un stub HTTP programable —WireMock es el habitual— levantado como un proceso aparte al que apuntan las URL base del cliente en el perfil local. No es un doble dentro del proceso: habla HTTP por el mismo socket que hablaría el proveedor, así que se ejercitan el adaptador real, la serialización, el timeout y el cortacircuitos. El servidor no sabe que no es real.

Lo que hay que poder programar desde cada escenario:

| Capacidad | Qué camino de pago ejercita |
|---|---|
| Responder un cuerpo y un status concretos | El desenlace: autorizado, fondos insuficientes, rechazo del emisor |
| Responder 5xx | Fallo del proveedor — y ojo, si la llamada declara reintento solo ante timeout y corte, un 5xx **no** se reintenta |
| Cortar la conexión / retrasar más allá del timeout | El cobro **en duda**: la pasarela pudo haber cobrado y nosotros no lo sabemos |
| Contar llamadas y leer el cuerpo y las cabeceras enviadas | Que nuestro reintento llevaba la clave de idempotencia. Sin asertarlo, esa garantía queda sin probar |

Y dos reglas que se pagan caras si se olvidan: **los stubs se programan en el test, no en un archivo de configuración compartido** —un archivo esconde en otro sitio la mitad del escenario— y **se limpian entre flujos**, porque un stub que sobrevive a su escenario convierte el orden de ejecución en parte del resultado.

### Las dos piezas que un stub no cubre

**Provocar el webhook.** Es la mitad que un stub no puede cubrir por definición: ahí la pasarela llama hacia nosotros. El test tiene que hacer ese `POST` él mismo contra nuestro endpoint, y **firmado correctamente**, lo que obliga a aprovisionar el secreto de firma como credencial de prueba.

Y tiene una consecuencia que conviene ver con claridad: para poder firmar, el arnés reimplementa el algoritmo de la pasarela. Eso significa que la prueba **no valida que tu verificación sea correcta** — valida que tu verificación y tu firma de prueba coinciden. Un error simétrico en ambas pasa en verde. Es el límite real de esta técnica, y como el webhook es el canal por el que entra la verdad del dinero, es un límite que hay que asumir a sabiendas y no descubrir.

**Respuestas que cambian entre llamadas.** Consultar el estado de un pago devuelve `pending` y después `succeeded`. No hace falta un stub con máquina de estados: basta con reprogramar la misma ruta a mitad del test y dejar que sea el propio test quien conduzca la línea temporal.

### Las dos suites

Dos suites con propósitos distintos, y solo una de ellas bloqueando:

- **Hermética, con stubs.** Todos los flujos, incluidos los rechazos, los timeouts, la reentrega del webhook y el barrido de reconciliación. Es la que puntúa y la que bloquea una entrega.
- **De contrato, contra el sandbox real de cada pasarela.** Fuera de integración continua, a mano o en su propia cadencia. No prueba la lógica —eso ya lo hizo la anterior— sino que **el contrato del cable no ha cambiado**: que los campos siguen llamándose igual y que la firma sigue verificando. Es lo único capaz de detectar la deriva del proveedor, que es exactamente lo que la capa de anticorrupción existe para absorber.

### Microcks: dónde encaja y dónde no

Microcks convierte los ejemplos de un OpenAPI o un AsyncAPI en mocks vivos y elige qué respuesta devolver con reglas de despacho, una de ellas por expresión regular sobre el cuerpo JSON. Y ahí hay un encaje casi demasiado bonito: **la tabla de tarjetas de prueba que publica cada pasarela es exactamente eso**. «Esta tarjeta → rechazada por el emisor, esta otra → fondos insuficientes» es una tabla de despacho por cuerpo, literal.

Aun así, para la suite hermética no compensa, por cinco razones:

1. **Es una plataforma, no un stub.** Arrastra base de datos y aplicación web, frente a un contenedor con una API de administración. La infraestructura de prueba quiere ser mínima, y el aislamiento entre flujos —resetear el stub entre clases— es lo que sostiene la suite entera.
2. **Invierte dónde vive el `Given`.** Las respuestas viven en un artefacto importado y el test elige entre ellas fabricando una petición que case con una regla. El escenario deja de leerse entero en su propia clase, que es justo lo que la regla anterior evita.
3. **No resuelve la mitad difícil.** Es un servidor que responde; su mocking asíncrono publica en brokers, no hace llamadas HTTP hacia nosotros. Provocar el webhook firmado queda igual.
4. **Cero beneficio para la familia de redirección.** No hay OpenAPI que importar. Todo su valor está en el lado tokenizado, que es el que ya estaba bien cubierto.
5. **Dos tecnologías de stub en un mismo proyecto** cuesta más de lo que ahorra.

Donde sí encaja es en la **segunda suite**: la otra mitad de Microcks es ejecutar conformidad de contrato contra una implementación real, que es precisamente el trabajo que hoy no tiene herramienta. No como sustituto del stub, sino como herramienta distinta para un trabajo distinto.

Y si lo único atractivo era la tabla de tarjetas, eso se replica en un stub corriente con un criterio de coincidencia por número de tarjeta: menos elegante, y una dependencia menos.

---

## 10. Cómo empezar

El orden natural, si se decide construirlo:

1. **El diseño**, con el lifecycle unificado y las tres concesiones escritas como reglas explícitas.
2. **Los escenarios de validación**, con atención especial a los dos caminos que ningún flujo feliz ejercita: el desenlace que llega por webhook y el barrido de reconciliación. Son los que se quedan sin cubrir por defecto, y son los que guardan el dinero.
3. **Publicarlo como diseño base**, del que derivar cada pasarela concreta.
4. **La primera pasarela**, completa de punta a punta antes de empezar la segunda. La segunda es la que valida que la frontera estaba en el sitio correcto; la primera solo valida que funciona.

Antes de todo eso hacen falta dos decisiones de negocio que el diseño no puede tomar solo:

- **¿Anular y devolver son la misma operación de cara al integrador, o dos?**
- **¿Quién es la fuente de verdad del importe** si el pedido vive en otro servicio? El pago guarda una copia del importe en el momento de cobrar, y esa copia y el pedido pueden divergir.
