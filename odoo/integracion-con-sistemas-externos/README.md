# Integración de Odoo con sistemas externos — Guía de tecnología

Cómo conectar Odoo con otros sistemas (una tienda online, un transportista, una pasarela de pago) en las dos direcciones: **otros llaman a Odoo** (con su API externa o con endpoints REST propios) y **Odoo llama a otros** (con `requests` desde un módulo). Y, sobre todo, cómo hacerlo de forma **robusta**: con credenciales acotadas, reintentos, idempotencia y una cola de pendientes para que ningún fallo pase desapercibido.

Pensada para perfiles backend que ya saben consumir una API REST, pero no han integrado Odoo con nada. Todas las fichas comparten el mismo escenario —una tienda online que envía pedidos a Odoo y recibe de vuelta el estado del envío— para que las piezas encajen entre sí.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [API externa de Odoo](API-Externa-de-Odoo.md) | El punto de partida: la puerta que Odoo ya trae (JSON-2 y la clásica XML-RPC), sin escribir código en el servidor. |
| 2 | [Claves de API y usuarios de integración](Claves-de-API-y-Usuarios-de-Integracion.md) | Cómo autenticar una integración sin usar la contraseña de nadie y con los permisos mínimos. |
| 3 | [Endpoints REST con controladores](Endpoints-REST-con-Controladores.md) | Cuando necesitas tu propio contrato o recibir webhooks: controladores, autenticación y códigos HTTP. |
| 4 | [Llamar a APIs externas desde Odoo](Llamar-a-APIs-Externas-desde-Odoo.md) | La dirección contraria: tiempos límite, reintentos y el problema de la transacción. |
| 5 | [Manejo de errores en integraciones](Manejo-de-Errores-en-Integraciones.md) | La síntesis: clasificar errores, idempotencia, cola de pendientes y qué hacer con lo fallido. |

---

<details>
<summary>Ver todos los archivos</summary>

- [API externa de Odoo](API-Externa-de-Odoo.md)
- [Claves de API y usuarios de integración](Claves-de-API-y-Usuarios-de-Integracion.md)
- [Endpoints REST con controladores](Endpoints-REST-con-Controladores.md)
- [Llamar a APIs externas desde Odoo](Llamar-a-APIs-Externas-desde-Odoo.md)
- [Manejo de errores en integraciones](Manejo-de-Errores-en-Integraciones.md)

</details>

---

> Piezas relacionadas: [Parámetros del sistema](../configuracion-parametros/Parametros-del-Sistema.md) para guardar URLs y tokens, [Recordsets](../fundamentos/Recordsets.md) para entender lo que devuelve el ORM que expone la API, y [Entornos: desarrollo, staging y producción](../pruebas-seguras/Entornos-dev-staging-produccion.md) para probar una integración sin tocar datos reales.
