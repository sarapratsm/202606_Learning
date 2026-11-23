# Ollama — Guía de tecnologías

Cómo ejecutar modelos de lenguaje en tu propio equipo con Ollama y usarlos desde código: instalarlo, descargar un modelo, llamar a su API HTTP, forzar respuestas con forma de datos y elegir un tamaño de modelo que de verdad quepa en tu memoria.

Está pensada para perfiles backend que ya han llamado a alguna API de LLM en la nube (o ni siquiera) y quieren una alternativa local, gratuita y privada. No presupone experiencia en inteligencia artificial: cada término se define antes de usarlo, y los ejemplos usan escenarios genéricos (clasificar pedidos, resumir textos) para que sean fáciles de trasladar.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Ollama](Ollama.md) | Qué es, cómo instalarlo y los comandos del día a día. Punto de partida para tener un modelo funcionando. |
| 2 | [La API REST de Ollama](API-REST-de-Ollama.md) | Cómo llamar al modelo desde tu código: chat, opciones, respuestas con esquema JSON y errores típicos. |
| 3 | [Modelos y rendimiento](Modelos-y-Rendimiento.md) | Cómo elegir el modelo y qué hacer cuando va lento: parámetros, cuantización, GPU, memoria, `keep_alive` y dónde se guardan los modelos en disco. |

---

> Para profundizar en cómo construir aplicaciones con modelos de lenguaje (salidas estructuradas, *tool use*, evaluación, seguridad), ver [Ingeniería con LLMs](../ingenieria-con-llms/README.md).
