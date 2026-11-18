# Claves de API y usuarios de integración en Odoo

## ¿Qué es?

Una **clave de API** es una credencial que sustituye a la contraseña de una persona cuando un programa (no un humano) se conecta a Odoo. Un **usuario de integración** es un usuario de Odoo creado solo para ese programa, con los permisos mínimos que necesita.

## ¿Por qué existe?

Si una integración se conecta con la contraseña de una persona, aparecen tres problemas a la vez: si esa persona cambia la contraseña, la integración se rompe; si se va de la empresa y se desactiva su usuario, también; y si alguien filtra el script, regala el acceso completo a su cuenta, incluida la interfaz web.

Las claves resuelven los tres: se pueden crear, caducar y revocar sin tocar la contraseña, y no sirven para iniciar sesión en la interfaz. Combinadas con un usuario propio y acotado, el daño de una filtración queda limitado a lo que ese usuario podía hacer.

> Si conoces los *tokens* de acceso personal de GitHub, es la misma idea: una credencial con nombre, caducidad y revocable, que no es tu contraseña.

## ¿Cuándo y para qué se usa?

En cualquier integración desatendida: la tienda online que crea pedidos, el script nocturno de facturación, un panel de informes. Para uso interactivo (explorar la API desde tu propia máquina) puedes usar tu cuenta personal; para todo lo que corre solo, usa un usuario de integración.

---

## Crear una clave

En la interfaz: **Preferencias del usuario → Seguridad de la cuenta → Nueva clave de API**. Hay que indicar una descripción y una duración. Odoo muestra la clave **una sola vez**, al crearla: si la pierdes, no se puede recuperar y hay que generar otra. En Odoo 19 la duración máxima de una clave es de unos tres meses, por lo que una integración permanente necesita **rotación** periódica.

> Algunas de estas particularidades (duración máxima, límite de claves por usuario) han cambiado entre versiones. Compruébalas en la documentación de la tuya.

La clave se usa así:

```python
# JSON-2 (Odoo 19+): cabecera Authorization
headers = {"Authorization": "bearer CLAVE_GENERADA"}

# XML-RPC: la clave va donde iría la contraseña
uid = common.authenticate(DB, "api@ejemplo.com", "CLAVE_GENERADA", {})
```

Además, un método de modelo permite generarlas por código (útil para automatizar la rotación), pero por defecto solo lo pueden hacer personas con acceso de administración de ajustes. Cualquier clave tiene, como límite, los mismos permisos que su usuario.

## Un usuario por integración

Crea un usuario específico (`api-tienda@ejemplo.com`, no el de una persona) y asígnale solo los grupos que la integración necesita. El principio es **privilegio mínimo**: si la integración solo crea pedidos y consulta productos, el usuario no debería ver contabilidad ni poder borrar clientes.

Lo que Odoo aplica a ese usuario, igual que a cualquiera:

- **Permisos de acceso (ACL)** por modelo: leer, escribir, crear, borrar.
- **Reglas de registro** (*record rules*): qué filas puede ver, por ejemplo, solo las de su compañía.
- **Permisos por campo**: campos restringidos a ciertos grupos.

Un ejemplo de diagnóstico habitual: la integración falla con `AccessError` al crear un pedido. En vez de dar permisos de administrador «para que funcione», lee el mensaje: dice qué modelo y qué operación faltan, y eso es justo lo único que hay que conceder.

```python
# Respuesta típica cuando faltan permisos (JSON-2)
# HTTP 403
# {"name": "odoo.exceptions.AccessError",
#  "message": "You are not allowed to create 'Sales Order' (sale.order) records. ..."}
```

## Dónde guardar la clave

La clave es un secreto. Reglas que se cumplen siempre:

- **Fuera del código y del repositorio.** Variable de entorno, gestor de secretos o fichero de configuración que no se versiona.
- **Una clave por integración.** Si una se filtra, revocas solo esa.
- **Nunca en un registro, URL o mensaje de error.** Ni siquiera parcialmente.

```python
import os

API_KEY = os.environ["ODOO_API_KEY"]   # falla en alto y claro si falta
```

Cuando el secreto debe vivir dentro de Odoo (por ejemplo, la clave de una API externa que Odoo consume), se guarda en un parámetro del sistema (`ir.config_parameter`) accesible solo para administración. Cómo funciona ese almacén está en [Parámetros del sistema](../configuracion-parametros/Parametros-del-Sistema.md).

## Buenas prácticas avanzadas

- **Planifica la rotación antes de necesitarla.** Una clave que caduca un domingo a las 3:00 rompe la integración sin que nadie mire. Anota la fecha, crea la clave nueva con solapamiento y revoca la antigua cuando la nueva ya funcione.
- **Revoca de inmediato ante cualquier duda.** Revocar una clave es barato y reversible (creas otra); una filtración no revocada no.
- **Separa claves por entorno.** Producción, *staging* y desarrollo tienen claves y usuarios distintos. Una prueba mal configurada no debe poder escribir en producción. Más sobre entornos en [Entornos: desarrollo, staging y producción](../pruebas-seguras/Entornos-dev-staging-produccion.md).
- **Audita lo que hace el usuario de integración.** Al ser un usuario propio, el chatter y los campos `create_uid`/`write_uid` de cada registro indican qué creó o modificó la integración, lo que separa un fallo suyo de un cambio humano.
- **Cuidado con `sudo()` en tus propios endpoints.** Un controlador personalizado que usa `sudo()` ignora los permisos del usuario de integración y anula todo lo anterior. Úsalo solo cuando sea imprescindible y valida la entrada.

## Documentación oficial

- [External API: API keys (documentación de Odoo)](https://www.odoo.com/documentation/19.0/developer/reference/external_api.html) — cómo se crean, el alcance (*scope*) y el comportamiento de revocación; la sección de claves es la que importa.
- [Security in Odoo (documentación de Odoo)](https://www.odoo.com/documentation/19.0/developer/reference/backend/security.html) — grupos, ACL y reglas de registro: la base para decidir qué permisos concede cada usuario de integración.

---

*En resumen: una integración desatendida usa su propio usuario con permisos mínimos y una clave de API revocable, guardada como secreto y rotada antes de que caduque.*
