# Manejo de errores en integraciones con Odoo

## ¿Qué es?

Es el conjunto de decisiones que hacen que una integración **siga siendo correcta cuando algo falla**: distinguir qué errores merece la pena reintentar, no duplicar datos al reintentar, no perder trabajo pendiente y enterarte de lo que no se pudo resolver.

## ¿Por qué existe?

Una integración que funciona el día de la demostración es fácil. La difícil es la que sobrevive al martes en que el transportista tiene una caída de diez minutos, la tienda reenvía el mismo pedido tres veces porque no recibió tu respuesta, o la red se corta justo entre «creé el pedido» y «confirmé al otro lado». Sin un diseño para el fallo, el resultado es el peor posible: datos duplicados, pedidos perdidos y nadie sabe que ocurrió.

La robustez aquí no consiste en que nada falle, sino en que **cada fallo tenga un destino previsto**.

> Si conoces las colas de mensajes (RabbitMQ, SQS), es el mismo conjunto de ideas: reintentos con espera, mensajes idempotentes y una cola de «cartas muertas» para lo que no se puede procesar.

## ¿Cuándo y para qué se usa?

En cualquier integración que modifique datos: pedidos, pagos, stock, facturas. Una consulta de solo lectura que falla se reintenta sin más; una escritura que falla a medias es lo que obliga a pensar en todo lo que sigue.

---

## 1. Clasificar el error: ¿reintentar o no?

La primera decisión es siempre la misma: **¿volver a intentarlo cambiaría el resultado?**

| Tipo de error | Ejemplos | Qué hacer |
|---|---|---|
| **Transitorio** | Timeout, caída de red, `502`/`503`/`504`, `429` | Reintentar con espera |
| **Permanente** | `400`, `401`, `403`, `404`, `422`, datos inválidos | **No** reintentar: corregir la causa |
| **Ambiguo** | Timeout *después* de enviar un `POST` | Reintentar solo si es seguro (ver idempotencia) |

Reintentar un error permanente solo genera ruido: el mismo `422` por datos inválidos fallará cien veces. Y reintentar uno transitorio sin espera puede empeorar el problema que causó el fallo.

```python
ESTADOS_REINTENTABLES = {408, 429, 500, 502, 503, 504}

def es_transitorio(error):
    if isinstance(error, (requests.Timeout, requests.ConnectionError)):
        return True
    if isinstance(error, requests.HTTPError):
        return error.response.status_code in ESTADOS_REINTENTABLES
    return False
```

Las esperas entre reintentos crecen de forma **exponencial con un poco de azar** (*backoff* con *jitter*): 1 s, 2 s, 4 s, 8 s, cada una con una variación aleatoria. Sin azar, cien reintentos que fallaron a la vez vuelven a llegar a la vez y tumban de nuevo al servidor recién recuperado.

```python
import random

def espera(intento):                       # intento = 1, 2, 3...
    base = min(2 ** intento, 300)          # tope de 5 minutos
    return base * random.uniform(0.5, 1.5)
```

## 2. Idempotencia: poder repetir sin duplicar

Reintentar solo es seguro si **repetir una operación produce el mismo resultado que hacerla una vez**. El caso que lo ilustra: envías un pedido, el servidor lo crea, pero la respuesta se pierde por el camino. Tu código ve un timeout y reintenta. Sin idempotencia, hay dos pedidos.

La solución es darle a cada operación una **referencia única que decide el emisor** (no el receptor) y comprobarla antes de crear:

```python
# En el modelo: una referencia externa única por pedido
class SaleOrder(models.Model):
    _inherit = "sale.order"

    external_ref = fields.Char(index=True, copy=False)

# Al recibir o sincronizar: buscar antes de crear («upsert»)
def _crear_o_actualizar(self, datos):
    pedido = self.search([("external_ref", "=", datos["external_ref"])], limit=1)
    if pedido:
        pedido.write(self._vals_desde(datos))
        return pedido
    return self.create({**self._vals_desde(datos), "external_ref": datos["external_ref"]})
```

La búsqueda-antes-de-crear tiene una carrera: dos peticiones simultáneas con la misma referencia pueden pasar la búsqueda a la vez y crear dos registros. La defensa definitiva es una **restricción de unicidad en la base de datos** sobre `external_ref`: la segunda inserción falla y la capturas como «ya existe». Tu API, para sus clientes, hace lo mismo con una cabecera de idempotencia (`Idempotency-Key`), que los grandes proveedores de pagos usan para que reintentar un cobro nunca cobre dos veces.

## 3. Traducir los errores de Odoo a respuestas claras

Dentro de Odoo, los errores tienen tipos con significado. Mapearlos a códigos HTTP hace que el cliente sepa qué hacer sin leer mensajes:

| Excepción de Odoo | Significa | Código HTTP | ¿Reintentar? |
|---|---|---|---|
| `ValidationError` | Los datos incumplen una regla de negocio | `422` | No |
| `UserError` | Operación no permitida en este estado | `409` o `422` | No |
| `AccessError` | El usuario no tiene permisos | `403` | No |
| `MissingError` | El registro no existe (o fue borrado) | `404` | No |
| Cualquier otra (`Exception`) | Fallo inesperado | `500` | Sí, con prudencia |

```python
from odoo.exceptions import AccessError, MissingError, UserError, ValidationError
import logging

_logger = logging.getLogger(__name__)

def _respuesta_de_error(self, error):
    if isinstance(error, ValidationError):
        return self._error(422, "validation_error", str(error))
    if isinstance(error, AccessError):
        return self._error(403, "forbidden", "Sin permisos para esta operación")
    if isinstance(error, MissingError):
        return self._error(404, "not_found", "El registro no existe")
    if isinstance(error, UserError):
        return self._error(409, "conflict", str(error))
    _logger.exception("Error inesperado en la integración")   # traza completa al log
    return self._error(500, "internal_error", "Error interno")  # nada de detalles al cliente
```

Fíjate en el último caso: el detalle técnico va al **log** y al cliente solo llega un mensaje genérico. Una traza en la respuesta puede exponer nombres de tablas, rutas del servidor o datos de otros clientes.

Una consecuencia de cómo funciona Odoo que conviene recordar: cuando una excepción sube hasta el final de la petición, **toda la transacción se deshace**. Si dentro de un lote quieres que un registro defectuoso no arrastre a los demás, protégelo con un *savepoint*:

```python
for datos in lote:
    try:
        with self.env.cr.savepoint():          # lo que falle aquí se deshace solo
            self._crear_o_actualizar(datos)
    except (ValidationError, UserError) as e:
        errores.append({"ref": datos["external_ref"], "error": str(e)})
```

Sin `savepoint()`, un error en el registro 37 deshace los 36 anteriores. Con él, los 36 se conservan y el 37 queda anotado en `errores`.

## 4. La cola de pendientes: que ningún trabajo se pierda

Cuando una llamada externa falla, lo peor es que el intento quede solo en una excepción que nadie vuelve a ver. El patrón fiable es una **tabla de trabajos pendientes** (a veces llamada *outbox*): en vez de llamar al tercero dentro de la transacción del usuario, registras «hay que enviar esto» y un proceso aparte se encarga, con reintentos.

```python
class SyncJob(models.Model):
    _name = "tienda.sync.job"
    _description = "Trabajo de sincronización pendiente"

    order_id = fields.Many2one("sale.order", required=True, ondelete="cascade")
    state = fields.Selection(
        [("pending", "Pendiente"), ("done", "Hecho"), ("failed", "Fallido")],
        default="pending", index=True)
    attempts = fields.Integer(default=0)
    next_try = fields.Datetime(default=fields.Datetime.now, index=True)
    last_error = fields.Text()
```

La acción programada (`ir.cron`) recoge los pendientes cuyo `next_try` ya pasó y los procesa uno a uno:

```python
from datetime import timedelta

MAX_INTENTOS = 6

def _procesar_pendientes(self):
    jobs = self.search([("state", "=", "pending"),
                        ("next_try", "<=", fields.Datetime.now())], limit=50)
    for job in jobs:
        try:
            with self.env.cr.savepoint():
                job.order_id._enviar_a_transportista()
            job.state = "done"
        except Exception as e:
            job.attempts += 1
            job.last_error = str(e)
            if not es_transitorio(e) or job.attempts >= MAX_INTENTOS:
                job.state = "failed"                  # «carta muerta»: requiere a una persona
            else:
                job.next_try = fields.Datetime.now() + timedelta(seconds=espera(job.attempts))
```

Tres estados bastan: **pendiente** (se reintentará), **hecho** y **fallido** (se agotaron los intentos o el error es permanente). La clave es lo último: lo fallido **no desaparece**, queda visible en una vista de lista filtrable con su `last_error`, y alguien puede corregir la causa y devolverlo a pendiente con un botón.

Si no quieres escribir tu propia cola, el módulo `queue_job` de la OCA ofrece exactamente esto (reintentos con espera, estados, reintento manual, concurrencia por canal) como una pieza ya probada.

## 5. Enterarte de lo que falla

Una integración que falla en silencio es peor que una que se cae con ruido. Mínimos de observabilidad:

- **Registro estructurado** (`_logger`) con la referencia del registro y el motivo, no solo «error en sincronización».
- **Una vista de trabajos fallidos** que alguien revise, y un contador visible (por ejemplo, un filtro «Fallidos» en el menú).
- **Un aviso** cuando haya fallidos acumulados: un correo a una persona responsable o una actividad programada (ver [Actividades](../notificaciones/Actividades.md)).

## Buenas prácticas avanzadas

- **Dos tipos de timeout, dos decisiones distintas.** Un timeout de *conexión* (ni llegaste) siempre es seguro de reintentar; uno de *lectura* en un `POST` es ambiguo: el servidor pudo procesarlo. Resuélvelo con idempotencia, no con optimismo.
- **Comprueba antes de reintentar un `POST` ambiguo.** Si el tercero ofrece una consulta (`GET /shipments?reference=...`), pregunta si el envío ya existe antes de volver a crearlo. Es más fiable que adivinar.
- **Pon un tope a los reintentos y a la edad del trabajo.** Reintentar durante semanas un pedido cancelado es ruido. Pasados N intentos o X horas, marca como fallido y escala a una persona.
- **No tragues excepciones.** Un `except Exception: pass` convierte un error en un dato silenciosamente erróneo. Captura el tipo concreto que sabes manejar, y deja subir el resto.
- **Prueba los caminos de fallo, no solo el feliz.** Simula con `mock.patch` un timeout, un `503` y un `422`, y comprueba que el trabajo queda en el estado correcto y que reintentar dos veces no duplica nada.
- **Confirma por lotes en las tareas largas, con cuidado.** En una acción programada que procesa miles de registros, confirmar la transacción cada pocos registros evita perder todo por un fallo tardío, pero hazlo siguiendo el mecanismo que ofrezca tu versión de Odoo y no con `commit()` sueltos por todo el código: un `commit` rompe la atomicidad de lo que venía antes.

## Documentación oficial

- [Exceptions (documentación de Odoo)](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html#exceptions) — la jerarquía de excepciones (`UserError`, `ValidationError`, `AccessError`, `MissingError`) y cuándo lanzar cada una.
- [OCA/queue: módulo `queue_job`](https://github.com/OCA/queue/tree/19.0/queue_job) — la cola de trabajos de la comunidad; el README explica canales, reintentos y la API para encolar un método.
- [Stripe: Idempotent requests](https://docs.stripe.com/api/idempotent_requests) — una explicación breve y clara del patrón de clave de idempotencia, aplicable a cualquier API.

---

*En resumen: una integración robusta clasifica cada error (reintentable o no), hace sus operaciones idempotentes, guarda el trabajo pendiente en una cola con reintentos con espera, y deja visible todo lo que no pudo resolver.*
