# Endpoints REST con controladores de Odoo

## ¿Qué es?

Un **controlador** es una clase de Python dentro de un módulo de Odoo que declara URLs propias y el código que responde a cada una. Es la forma de ofrecer una API **con tu propio contrato**: las rutas, los métodos HTTP y los formatos los decides tú.

## ¿Por qué existe?

La [API externa](API-Externa-de-Odoo.md) de Odoo es genérica: expone el ORM tal cual, con una forma de llamada fija. Eso es cómodo para scripts, pero no sirve en dos situaciones habituales:

- **Otro sistema te llama a ti** (un *webhook*): la pasarela de pago avisa de que un pago se ha completado, haciendo un `POST` a una URL con un formato que ella decide, no tú.
- **Quieres un contrato estable y reducido**: no exponer modelos enteros, sino una operación concreta (`POST /api/v1/orders`) que valida la entrada, aplica tus reglas y devuelve algo limpio.

Odoo no trae una API REST «de serie»; los controladores son la pieza con la que la construyes.

> Si conoces ASP.NET Core o Express, un controlador de Odoo es lo mismo: un método decorado con la ruta y el método HTTP que atiende. La diferencia es que el decorador es `@http.route(...)` y el contexto de la petición (usuario, entorno, base de datos) llega a través de `request`.

## ¿Cuándo y para qué se usa?

- Recibir **webhooks** de pasarelas de pago, transportistas o la tienda online.
- Ofrecer un **endpoint de ingesta** (`POST /api/v1/orders`) con validación propia.
- Responder a una app móvil o a un front externo con un JSON a medida.

---

## El primer controlador

Un módulo con un controlador mínimo que recibe un pedido de la tienda:

```python
# controllers/orders.py
from odoo import http
from odoo.http import request


class OrderApi(http.Controller):

    @http.route("/api/v1/orders", type="http", auth="public",
                methods=["POST"], csrf=False)
    def create_order(self, **kwargs):
        datos = request.get_json_data()          # cuerpo JSON ya interpretado
        # ... validar y crear el pedido ...
        return request.make_json_response({"status": "created"}, status=201)
```

Y se carga desde el `__init__.py` del módulo (`from . import controllers`, y dentro de `controllers/__init__.py`, `from . import orders`). Si el módulo no importa el controlador, la ruta no existe.

Los parámetros del decorador que importan:

| Parámetro | Qué controla |
|---|---|
| `type` | `"http"` (tú controlas el cuerpo y el formato de respuesta) o `"jsonrpc"` (sobre JSON-RPC; en Odoo 18 y anteriores se llamaba `"json"`). Para una API REST normal, usa `"http"`. |
| `auth` | Quién puede llamar: `"user"` (sesión iniciada), `"public"` (cualquiera, con el usuario público), `"none"` (sin base de datos asociada). |
| `methods` | Métodos HTTP aceptados. Sin esto, la ruta responde a cualquiera. Decláralo siempre. |
| `csrf` | Protección contra falsificación de peticiones. Para APIs llamadas por otros servidores (sin navegador), se desactiva con `False`. |

Qué devuelve: con `make_json_response` obtienes una respuesta con `Content-Type: application/json` y el estado HTTP que indiques. Un `return` de diccionario sin ese envoltorio no se serializa a JSON automáticamente en `type="http"`.

> **Cuidado con el nombre del tipo.** En Odoo 19 `type="json"` pasó a llamarse `type="jsonrpc"` (el nombre antiguo funciona como alias obsoleto). Un controlador `jsonrpc` espera el formato del protocolo JSON-RPC (`{"jsonrpc": "2.0", "params": ...}`), que no es REST. Si un cliente externo te va a enviar un JSON plano, usa `type="http"`.

## Autenticar las peticiones

`auth="public"` significa que Odoo **no autentica a nadie**: eres tú quien debe comprobar que la petición es legítima. Dos patrones habituales.

**1. Secreto compartido en una cabecera**, para llamadas entre tus propios sistemas:

```python
import hmac

def _autorizado(self):
    esperado = request.env["ir.config_parameter"].sudo().get_param("tienda.api_token", "")
    recibido = request.httprequest.headers.get("Authorization", "").removeprefix("Bearer ")
    return bool(esperado) and hmac.compare_digest(esperado, recibido)
```

`hmac.compare_digest` compara en tiempo constante: comparar con `==` permite deducir la clave probando letra a letra midiendo tiempos.

**2. Firma del cuerpo (HMAC)**, para webhooks de terceros. El emisor firma el cuerpo con un secreto compartido y envía la firma en una cabecera; tú recalculas y comparas:

```python
import hashlib, hmac

def _firma_valida(self):
    secreto = request.env["ir.config_parameter"].sudo().get_param("pasarela.webhook_secret", "").encode()
    cuerpo = request.httprequest.get_data()                       # bytes sin interpretar
    esperada = hmac.new(secreto, cuerpo, hashlib.sha256).hexdigest()
    recibida = request.httprequest.headers.get("X-Signature", "")
    return hmac.compare_digest(esperada, recibida)
```

El cuerpo se firma **tal como llega**: valida la firma antes de interpretar el JSON, porque reserializar cambia los bytes y la firma deja de coincidir.

Con `auth="public"` el entorno (`request.env`) pertenece al usuario público, que casi no tiene permisos. Para crear registros hay que actuar como un usuario de integración concreto, no con `sudo()` a ciegas:

```python
usuario_api = request.env.ref("tienda_api.usuario_integracion")
pedidos = request.env["sale.order"].with_user(usuario_api)
```

Así se aplican los permisos del usuario de integración ([Claves de API y usuarios de integración](Claves-de-API-y-Usuarios-de-Integracion.md)) en lugar de saltárselos.

## Un endpoint completo: validar, crear, responder

Juntando las piezas: un `POST /api/v1/orders` que valida, evita duplicados y devuelve códigos HTTP con sentido.

```python
import json
from odoo import http
from odoo.exceptions import ValidationError, AccessError
from odoo.http import request


class OrderApi(http.Controller):

    @http.route("/api/v1/orders", type="http", auth="public",
                methods=["POST"], csrf=False)
    def create_order(self, **kwargs):
        if not self._autorizado():
            return self._error(401, "unauthorized", "Token inválido")

        try:
            datos = request.get_json_data()
        except json.JSONDecodeError:
            return self._error(400, "invalid_json", "El cuerpo no es JSON válido")

        referencia = datos.get("external_ref")
        if not referencia or not datos.get("lines"):
            return self._error(422, "missing_fields", "Faltan external_ref o lines")

        pedidos = request.env["sale.order"].with_user(self._usuario_api())

        existente = pedidos.search([("client_order_ref", "=", referencia)], limit=1)
        if existente:                                   # idempotencia: no duplicar
            return request.make_json_response({"id": existente.id}, status=200)

        try:
            pedido = pedidos.create({
                "partner_id": datos["partner_id"],
                "client_order_ref": referencia,
                "order_line": [(0, 0, {"product_id": l["product_id"],
                                       "product_uom_qty": l["qty"]})
                               for l in datos["lines"]],
            })
        except ValidationError as e:
            return self._error(422, "validation_error", str(e))
        except AccessError as e:
            return self._error(403, "forbidden", str(e))

        return request.make_json_response({"id": pedido.id}, status=201)

    def _error(self, estado, codigo, mensaje):
        return request.make_json_response(
            {"error": {"code": codigo, "message": mensaje}}, status=estado)
```

`_autorizado` es el método de la sección anterior y `_usuario_api` devuelve el usuario de integración (`request.env.ref("tienda_api.usuario_integracion")`).

Lo que merece atención:

- **Cada error tiene su código HTTP** (400, 401, 403, 422): el cliente puede decidir si reintentar o no sin leer el texto. Más en [Manejo de errores en integraciones](Manejo-de-Errores-en-Integraciones.md).
- **La referencia externa evita duplicados.** Si la tienda reintenta el mismo pedido (porque no recibió tu respuesta), el segundo `POST` encuentra el existente y devuelve su id, en vez de crear otro.
- **Un formato de error único** (`{"error": {"code", "message"}}`) en todo el controlador. El cliente escribe un solo manejador.

## Comprobar que funciona

```bash
curl -i -X POST https://odoo.ejemplo.com/api/v1/orders \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"external_ref": "WEB-1001", "partner_id": 7, "lines": [{"product_id": 12, "qty": 2}]}'
```

Primera llamada: `201 Created` con `{"id": 54}`. Repetirla idéntica: `200 OK` con el mismo `{"id": 54}`. Una ruta que devuelve `404` suele ser un módulo sin instalar o sin actualizar (`-u mi_modulo`) o un controlador no importado en el `__init__.py`.

## Buenas prácticas avanzadas

- **Versiona la ruta desde el primer día (`/api/v1/...`).** Cuando cambies el contrato, `v2` convive con `v1` y los clientes migran a su ritmo.
- **Declara siempre `methods`.** Sin esa lista, un `GET` accidental a un endpoint de escritura se ejecuta igual. Un `GET` nunca debe modificar datos.
- **Valida antes de tocar la base de datos, y responde rápido en webhooks.** Muchos emisores consideran fallido un webhook que tarda más de unos segundos y lo reenvían. Si el trabajo pesado dura, guarda la petición en una tabla de pendientes, responde `200` y procésala después (ver la cola en [Manejo de errores en integraciones](Manejo-de-Errores-en-Integraciones.md)).
- **No devuelvas excepciones sin filtrar.** `str(e)` de un error interno puede incluir nombres de tablas o rutas. Para errores de validación es útil; para errores inesperados registra el detalle en el log y devuelve un mensaje genérico con `500`.
- **`sudo()` es la excepción, no la norma.** Cada `sudo()` en un endpoint público es un punto donde cualquiera con el token escribe sin límites. Prefiere `with_user(usuario_integracion)`.
- **Limita el tamaño y la frecuencia en el proxy.** Odoo no aplica límite de tasa a tus rutas; ponlo delante (nginx, Cloudflare), sobre todo en endpoints públicos.

## Documentación oficial

- [Web Controllers (documentación de Odoo)](https://www.odoo.com/documentation/19.0/developer/reference/backend/http.html) — el decorador `route`, los tipos de petición, la autenticación y el objeto `request`. Empieza por la sección de rutas y consulta la lista de parámetros de `route` cuando uses una opción nueva.

---

*En resumen: un controlador te permite diseñar tu propia API REST dentro de Odoo, pero la seguridad es cosa tuya: autentica explícitamente, valida antes de escribir, responde con códigos HTTP significativos y no uses `sudo()` como atajo.*
