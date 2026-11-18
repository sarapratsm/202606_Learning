# Llamar a APIs externas desde Odoo

## ¿Qué es?

Es la dirección contraria a las dos guías anteriores: en vez de que otro sistema llame a Odoo, es **Odoo quien llama** a una API ajena (una pasarela de pago, un transportista, un servicio de facturación electrónica) desde código Python de un módulo.

## ¿Por qué existe?

Cuando se confirma un pedido, a menudo hay que avisar a un sistema externo: crear el envío en el transportista, registrar la factura en el servicio fiscal. Hacerlo a mano desde la interfaz del otro sistema no escala, y hacerlo desde Odoo introduce un problema nuevo: **tu código de negocio ahora depende de un servidor que no controlas**. Puede tardar, caer, devolver errores o cambiar su formato.

Por eso una llamada externa dentro de Odoo no es solo `requests.post(...)`. Hay que decidir cuánto esperar, qué hacer si falla y, sobre todo, **en qué momento de la transacción** se hace.

## ¿Cuándo y para qué se usa?

- Enviar un pedido confirmado al **transportista** y guardar el número de seguimiento.
- Cobrar o consultar un pago en una **pasarela**.
- Sincronizar stock o precios con una **tienda online**.
- Consultar un dato de referencia (cambio de divisa, validación de un NIF).

---

## La llamada mínima, bien hecha

Odoo trae `requests` disponible. Un cliente pequeño, con las precauciones básicas:

```python
import logging
import requests
from odoo import models
from odoo.exceptions import UserError

_logger = logging.getLogger(__name__)


class SaleOrder(models.Model):
    _inherit = "sale.order"

    def _enviar_a_transportista(self):
        get = self.env["ir.config_parameter"].sudo().get_param
        url = get("transporte.api_url")
        token = get("transporte.api_token")

        try:
            r = requests.post(
                f"{url}/shipments",
                json={"reference": self.name, "address": self.partner_shipping_id.contact_address},
                headers={"Authorization": f"Bearer {token}"},
                timeout=(3.05, 10),          # (conexión, lectura) en segundos
            )
            r.raise_for_status()
        except requests.Timeout:
            raise UserError("El transportista no ha respondido a tiempo. Inténtalo de nuevo en unos minutos.")
        except requests.HTTPError as e:
            _logger.warning("Transportista rechazó el envío %s: %s %s",
                            self.name, e.response.status_code, e.response.text[:500])
            raise UserError("El transportista rechazó el envío. Revisa la dirección.")
        except requests.RequestException:
            _logger.exception("Fallo de red al contactar con el transportista")
            raise UserError("No se pudo contactar con el transportista.")

        self.carrier_tracking_ref = r.json()["tracking"]
```

Qué hace cada pieza:

- **`timeout` siempre.** `requests` **no** tiene tiempo límite por defecto: sin él, un servidor que no responde deja el proceso de Odoo bloqueado indefinidamente y, con unos pocos casos, agota todos los *workers*. Se pasa una tupla: tiempo para conectar y tiempo para leer.
- **`raise_for_status()`** convierte un 4xx/5xx en una excepción; sin él, un error del otro lado pasaría como una respuesta «normal».
- **Tres tipos de fallo, tres mensajes.** El usuario ve un mensaje comprensible (`UserError`); el detalle técnico va al log.
- **La URL y el token no están en el código**: salen de parámetros del sistema ([Parámetros del sistema](../configuracion-parametros/Parametros-del-Sistema.md)), así que cambian por entorno sin tocar el módulo.

## Reintentos para fallos transitorios

Una caída de red de un segundo no debería detener el proceso. `requests` permite reintentar automáticamente solo lo seguro mediante una sesión con política de reintentos:

```python
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

def _sesion_con_reintentos():
    reintentos = Retry(
        total=3,
        backoff_factor=1,                       # esperas de 1 s, 2 s, 4 s...
        status_forcelist=[429, 502, 503, 504],  # solo errores transitorios
        allowed_methods=["GET", "PUT", "DELETE"],  # métodos idempotentes
    )
    s = requests.Session()
    s.mount("https://", HTTPAdapter(max_retries=reintentos))
    return s
```

Fíjate en `allowed_methods`: un `POST` **no** está, y es deliberado. Reintentar un `POST` que quizá sí llegó al servidor puede crear el envío dos veces. Para reintentar un `POST` con seguridad, necesitas una clave de idempotencia (ver [Manejo de errores en integraciones](Manejo-de-Errores-en-Integraciones.md)).

## El problema de la transacción

Esta es la parte que más sorprende. Odoo ejecuta cada petición dentro de **una transacción de base de datos** que se confirma al final o se deshace si algo falla. Una llamada HTTP externa **no participa** en esa transacción:

```python
def action_confirm(self):
    super().action_confirm()          # (1) cambia el estado del pedido en la base de datos
    self._enviar_a_transportista()    # (2) llama a una API externa
    self._generar_factura()           # (3) falla -> la transacción se deshace
```

Si el paso 3 lanza una excepción, Odoo deshace el 1, pero el envío del paso 2 **ya existe en el transportista**. Resultado: el pedido vuelve a «borrador» y hay un paquete creado que nadie reclama. Y el contrario: si la llamada del paso 2 tarda 30 segundos, mantienes abierta una transacción (y los bloqueos de las filas) durante todo ese tiempo.

Las salidas, de más simple a más robusta:

1. **Hacer la llamada al final**, cuando todo lo demás ha ido bien, y asumir que si ella falla se puede reintentar sin deshacer lo anterior.
2. **No llamar dentro de la petición**: registrar en una tabla que hay un envío pendiente y que un proceso aparte (una acción programada, `ir.cron`) lo envíe. Así la transacción del usuario no depende de un tercero. Es el patrón que se desarrolla en [Manejo de errores en integraciones](Manejo-de-Errores-en-Integraciones.md).
3. **Usar una cola de trabajos** (el módulo `queue_job` de la OCA) para delegar justo eso.

## Buenas prácticas avanzadas

- **Nunca llames a una API externa dentro de un bucle sobre un recordset largo en una sola transacción.** 500 pedidos × 1 s son 8 minutos con bloqueos abiertos y un fallo en el 499 deshace todo. Procesa en lotes pequeños y confirma por lotes.
- **Registra la petición y la respuesta (sin secretos).** Cuando el tercero diga «nunca recibí eso», un registro con fecha, URL, estado HTTP y cuerpo es lo que zanja la discusión. Enmascara tokens y datos personales.
- **Respeta los límites del tercero.** Un `429 Too Many Requests` suele traer una cabecera `Retry-After` con los segundos a esperar; úsala en vez de un valor fijo.
- **Valida lo que recibes.** Una respuesta con un campo ausente (`r.json()["tracking"]` lanza `KeyError`) no debe tumbar el flujo con una traza ilegible: comprueba el formato y falla con un mensaje claro.
- **Aísla el cliente en una clase.** Pon todas las llamadas a un tercero en un único módulo o clase (`transporte_client.py`), no esparcidas por los modelos. Cuando cambie la API, cambias un archivo, y para las pruebas lo sustituyes por un doble.
- **Pruebas sin red.** Los tests de Odoo no deberían llamar a servicios reales: sustituye la función de petición con `unittest.mock.patch` y prueba cada rama de error.

## Documentación oficial

- [Requests: Quickstart y Advanced Usage](https://requests.readthedocs.io/en/latest/user/advanced/) — sesiones, tiempos límite y adaptadores; la sección *Timeouts* y la de *Transport Adapters* cubren lo esencial.
- [urllib3: Retry](https://urllib3.readthedocs.io/en/stable/reference/urllib3.util.html#urllib3.util.Retry) — todos los parámetros de la política de reintentos, incluido el respeto a `Retry-After`.

---

*En resumen: una llamada externa desde Odoo necesita tiempo límite, mensajes de error legibles y una decisión consciente sobre la transacción, porque el tercero no se deshace cuando Odoo hace rollback.*
