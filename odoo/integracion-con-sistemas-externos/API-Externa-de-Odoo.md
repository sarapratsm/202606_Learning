# API externa de Odoo (JSON-2, XML-RPC y JSON-RPC)

## ¿Qué es?

La API externa de Odoo es la puerta que el propio Odoo ofrece a programas ajenos para leer y escribir datos: cualquier modelo (`res.partner`, `sale.order`...) y cualquier método público del ORM quedan accesibles por HTTP sin escribir una línea de código en el servidor.

## ¿Por qué existe?

Un ERP casi nunca vive solo: la tienda online necesita crear pedidos, el CRM necesita leer clientes, un script nocturno necesita cerrar facturas. Sin una API, cada integración tendría que hablar directamente con PostgreSQL, saltándose la lógica de negocio (cálculo de impuestos, secuencias, permisos, automatismos) y rompiendo la base de datos en cuanto cambie una tabla.

La API externa expone el **ORM**, no las tablas: cada llamada ejecuta el mismo código que ejecutaría un usuario desde la interfaz, con sus mismos permisos.

> Si conoces las APIs REST, la idea es parecida, pero con una diferencia importante: no hay un recurso por URL con verbos HTTP. Hay **una sola forma de llamada**, «modelo + método + parámetros», y la URL solo indica a qué modelo y método.

## ¿Cuándo y para qué se usa?

- Sincronizar con una **tienda online** (pedidos que entran, stock que sale).
- **Importaciones masivas** desde hojas de cálculo o sistemas heredados.
- **Cuadros de mando** o informes externos que leen datos de Odoo.
- Scripts de mantenimiento y automatizaciones que no merece la pena empaquetar como módulo.

Si lo que necesitas es una API con **tu propio contrato** (URLs, formatos y nombres que tú decides, o recibir *webhooks*), esta no es la herramienta: mira [Endpoints REST con controladores](Endpoints-REST-con-Controladores.md).

---

## Dos generaciones de API

Conviene saber que conviven dos y cuál usar en cada caso:

| | JSON-2 | XML-RPC / JSON-RPC (clásica) |
|---|---|---|
| Endpoint | `POST /json/2/<modelo>/<método>` | `/xmlrpc/2/common`, `/xmlrpc/2/object`, `/jsonrpc` |
| Autenticación | Cabecera `Authorization: bearer <clave>` | Base de datos + login + clave, en cada llamada |
| Disponible desde | Odoo 19 | Todas las versiones |
| Estado | La recomendada | **Obsoleta (*deprecated*)** desde Odoo 19 |

Según la documentación de Odoo 19, los endpoints clásicos están programados para desaparecer en una versión futura (la fecha figura en la documentación oficial de tu versión; compruébala ahí, porque ha variado entre fuentes). Para una integración nueva sobre Odoo 19 o posterior, usa JSON-2. Aprende también la clásica: la mayoría de integraciones existentes, y todas las instalaciones de Odoo 17 y anteriores, la usan.

> **Disponibilidad.** En Odoo Online la API externa solo está disponible en ciertos planes de pago (según la documentación, los planes *Custom*). En Odoo.sh o instalaciones propias no hay esa restricción.

## JSON-2: la forma moderna

Una llamada es un `POST` a `/json/2/<modelo>/<método>`. El cuerpo es un JSON con los argumentos del método con nombre, y la autenticación viaja en una cabecera.

```python
import requests

BASE_URL = "https://odoo.ejemplo.com/json/2"
HEADERS = {
    "Authorization": "bearer TU_CLAVE_DE_API",
    "X-Odoo-Database": "tienda",   # solo si el servidor aloja varias bases
    "Content-Type": "application/json",
}

respuesta = requests.post(
    f"{BASE_URL}/res.partner/search_read",
    headers=HEADERS,
    json={
        "domain": [["is_company", "=", True]],
        "fields": ["name", "email"],
        "limit": 5,
    },
    timeout=10,
)
respuesta.raise_for_status()
print(respuesta.json())
```

Qué hace: busca las cinco primeras empresas y devuelve su nombre y correo. Qué devuelve (HTTP 200, el valor de retorno del método serializado como JSON):

```json
[{"id": 7, "name": "Tienda Ejemplo S.L.", "email": "info@ejemplo.com"}]
```

Detalles del contrato:

- **`ids`** — para los métodos que actúan sobre registros concretos (`read`, `write`, `unlink`, acciones `action_*`), la lista de ids va en el cuerpo. Para los métodos de modelo (`search`, `create`, `search_read`), se omite.
- **`context`** — objeto opcional, por ejemplo `{"lang": "es_ES"}` para recibir textos traducidos.
- **`X-Odoo-Database`** — hace falta cuando el servidor tiene varias bases y la configuración (`dbfilter`) no puede deducirla del dominio de la petición.

Ejemplo de escritura: crear un cliente y después modificarlo.

```python
# Crear: devuelve la lista de ids creados
r = requests.post(f"{BASE_URL}/res.partner/create", headers=HEADERS,
                  json={"vals_list": [{"name": "Ana Pérez", "email": "ana@ejemplo.com"}]},
                  timeout=10)
r.raise_for_status()
nuevo_id = r.json()[0]

# Modificar: write recibe los ids y los valores
requests.post(f"{BASE_URL}/res.partner/write", headers=HEADERS,
              json={"ids": [nuevo_id], "vals": {"phone": "+34 600 000 000"}},
              timeout=10).raise_for_status()
```

Los nombres de los argumentos (`vals_list`, `vals`, `domain`, `fields`) son los del método de Python en el modelo. Si dudas de cómo se llama un parámetro, mira la firma del método en el código de Odoo.

### Errores y transacciones en JSON-2

Un error devuelve un estado HTTP 4xx o 5xx con un JSON que explica qué pasó:

```json
{
  "name": "werkzeug.exceptions.Unauthorized",
  "message": "Invalid apikey",
  "arguments": ["Invalid apikey", 401],
  "context": {},
  "debug": "Traceback (most recent call last): ..."
}
```

Dos cosas que conviene saber desde el principio:

- **Cada llamada es una transacción independiente.** Si termina bien, se confirma; si falla, se descarta entera. No puedes encadenar dos llamadas en una misma transacción: si necesitas que dos operaciones sean atómicas, tiene que haber **un solo método** que las haga (una acción `action_...` del modelo o un método propio de un módulo).
- **El campo `debug` contiene la traza completa del servidor.** Úsalo para diagnosticar, pero no lo reenvíes a usuarios finales ni lo guardes sin cuidado: puede revelar rutas y detalles internos.

## La API clásica: XML-RPC

Funciona en dos pasos: autenticarse en el servicio `common` para obtener el identificador del usuario (`uid`) y llamar al servicio `object` con `execute_kw` para todo lo demás.

```python
import xmlrpc.client

URL, DB, USUARIO, CLAVE = "https://odoo.ejemplo.com", "tienda", "api@ejemplo.com", "TU_CLAVE_DE_API"

common = xmlrpc.client.ServerProxy(f"{URL}/xmlrpc/2/common")
uid = common.authenticate(DB, USUARIO, CLAVE, {})          # (1)

modelos = xmlrpc.client.ServerProxy(f"{URL}/xmlrpc/2/object")
empresas = modelos.execute_kw(
    DB, uid, CLAVE,                                          # (2) credenciales en cada llamada
    "res.partner", "search_read",                            # modelo y método
    [[["is_company", "=", True]]],                           # (3) argumentos posicionales
    {"fields": ["name", "email"], "limit": 5},               # (4) argumentos con nombre
)
```

Lo que más confunde: `execute_kw` recibe los argumentos posicionales como **una lista** (de ahí el corchete doble en el dominio: la lista de argumentos que contiene un dominio, que a su vez es una lista de condiciones) y los argumentos con nombre como un **diccionario** aparte. En JSON-2 todo eso desaparece: todo son argumentos con nombre.

Las operaciones de escritura son las mismas con otra sintaxis:

```python
id_nuevo = modelos.execute_kw(DB, uid, CLAVE, "res.partner", "create",
                              [{"name": "Ana Pérez", "email": "ana@ejemplo.com"}])
modelos.execute_kw(DB, uid, CLAVE, "res.partner", "write",
                   [[id_nuevo], {"phone": "+34 600 000 000"}])
```

Cuando algo falla, `xmlrpc.client` lanza `xmlrpc.client.Fault`, con `faultCode` y `faultString` (la traza del servidor):

```python
try:
    modelos.execute_kw(DB, uid, CLAVE, "res.partner", "unlink", [[999999]])
except xmlrpc.client.Fault as error:
    print(error.faultCode, error.faultString.splitlines()[-1])
```

> Las dos variantes (`/xmlrpc/2` y `/jsonrpc`) usan el mismo esquema de credenciales y los mismos métodos. Lo que has aprendido con una sirve para la otra; solo cambia el transporte.

## Buenas prácticas avanzadas

- **Pide solo los campos que necesitas.** `search_read` sin `fields` trae todos los campos del modelo, incluidos los calculados y los relacionales, y en modelos grandes (`product.product`, `sale.order`) multiplica el tiempo de respuesta. Indica siempre `fields` y un `limit`.
- **Pagina con `offset` y `limit`, y ordena de forma estable.** Una paginación sin `order` explícito puede repetir u omitir registros entre páginas si los datos cambian a mitad. Ordena por un campo único, como `id`.
- **Agrupa en lotes, no en bucle.** Crear 500 pedidos con 500 llamadas `create` es mucho más lento que una sola llamada con una lista de 500 `vals`. Pero un lote falla entero: elige un tamaño (50-200) que equilibre velocidad y coste de reintento.
- **Trata `id` como interno a cada base.** Los ids numéricos no son iguales entre una copia de pruebas y producción. Para referenciar de forma estable usa un identificador propio o el XML ID (`ir.model.data`), nunca un id numérico escrito a mano en tu integración.
- **Establece el `context` de forma explícita.** El idioma, la zona horaria y la compañía activa cambian el resultado de muchos métodos. Una integración que funciona con el contexto por defecto del usuario puede devolver textos o importes distintos cuando cambie su configuración.
- **No uses un usuario administrador.** Usa un usuario dedicado con los permisos mínimos; cubierto en [Claves de API y usuarios de integración](Claves-de-API-y-Usuarios-de-Integracion.md).

## Documentación oficial

- [External API (documentación de Odoo)](https://www.odoo.com/documentation/19.0/developer/reference/external_api.html) — la referencia de JSON-2: formato de petición, cabeceras, códigos de error y calendario de retirada de la API clásica. Empieza por la sección de ejemplos y vuelve a la de errores cuando algo falle.
- [ORM API (documentación de Odoo)](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html) — los métodos que puedes llamar (`search_read`, `create`, `write`...) y sus parámetros; cuando no sepas cómo se llama un argumento, está aquí.

---

*En resumen: la API externa expone el ORM de Odoo por HTTP, con JSON-2 como camino moderno y XML-RPC como herencia que seguirás encontrando, y cada llamada es una transacción independiente con los permisos del usuario que la hace.*
