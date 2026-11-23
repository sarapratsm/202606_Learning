# Ollama

## ¿Qué es?

Ollama es un programa que descarga y ejecuta modelos de lenguaje (LLMs) en tu propio ordenador, y los expone por una API HTTP local. Piensa en él como un "Docker para modelos": un comando para bajar un modelo, otro para arrancarlo, y un servicio siempre disponible en `localhost`.

## ¿Por qué existe?

Usar un LLM por API en la nube es cómodo, pero tiene tres costes que a veces no compensan: se paga por uso, los textos viajan a un tercero y dependes de que el servicio esté disponible. Ejecutar un modelo abierto en local elimina los tres, pero a mano es un camino tedioso: hay que localizar los pesos, elegir un formato, compilar un motor de inferencia y gestionar la memoria.

Ollama empaqueta todo eso. Descarga el modelo ya en un formato listo para usar, detecta si tienes tarjeta gráfica compatible, carga y descarga el modelo de memoria según se usa, y ofrece siempre la misma API sea cual sea el modelo.

> Si ya conoces Docker, la analogía es casi literal: `ollama pull` es `docker pull`, `ollama run` es `docker run`, `ollama list` es `docker images`, y la biblioteca de modelos hace de Docker Hub.

## ¿Cuándo y para qué se usa?

- **Datos que no pueden salir de tu máquina:** extraer información de documentos internos, correos o textos con datos personales sin enviarlos a un proveedor.
- **Desarrollo y pruebas:** una app que llama a un LLM puede probarse en local sin gastar créditos ni depender de la red.
- **Tareas sencillas y repetitivas:** clasificar, resumir o extraer campos de miles de textos, donde un modelo pequeño basta y el coste por llamada en la nube se acumularía.
- **Aprender:** probar modelos distintos y comparar sus respuestas sin cuenta ni tarjeta.

No es la mejor opción cuando necesitas la máxima calidad de razonamiento (los modelos abiertos que caben en un portátil son bastante más modestos que los grandes de la nube) ni cuando tu equipo no tiene memoria suficiente (ver [Modelos y rendimiento](Modelos-y-Rendimiento.md)).

---

## Instalación y primer arranque

Ollama se instala con el instalador oficial de [ollama.com](https://ollama.com/download) para Windows, macOS y Linux. Tras instalarlo queda un servicio en segundo plano escuchando en `http://localhost:11434`. Para comprobarlo:

```bash
curl http://localhost:11434/api/tags
```

Esta ruta lista los modelos instalados. En una instalación recién hecha devuelve una lista vacía:

```json
{"models":[]}
```

Que responda significa que el servicio funciona; que la lista esté vacía, que aún no has descargado ningún modelo.

## Descargar y probar un modelo

Un modelo se identifica por `nombre:etiqueta`, donde la etiqueta suele indicar el tamaño (`qwen2.5:7b`, `llama3.2:3b`). Sin etiqueta se usa `latest`.

```bash
ollama pull qwen2.5:7b
```

Descarga unos 5 GB la primera vez (las capas se reutilizan si luego bajas otra variante). Para chatear con él desde la terminal:

```bash
ollama run qwen2.5:7b
```

Se abre un prompt interactivo (`>>>`): escribes, el modelo responde, y `/bye` sale. Si pasas el texto como argumento, responde una vez y termina, lo que sirve para scripts:

```bash
ollama run qwen2.5:7b "Resume en una frase qué es un índice en una base de datos"
```

La primera respuesta tarda más: Ollama tiene que cargar el modelo en memoria. Las siguientes son inmediatas mientras siga cargado.

## Los comandos del día a día

| Comando | Para qué |
|---|---|
| `ollama list` | Modelos descargados y su tamaño en disco. |
| `ollama ps` | Modelos cargados ahora en memoria y si usan GPU o CPU. |
| `ollama show <modelo>` | Detalles: arquitectura, parámetros, contexto máximo, plantilla. |
| `ollama rm <modelo>` | Borra un modelo y libera disco. |
| `ollama stop <modelo>` | Lo descarga de memoria sin esperar al tiempo de inactividad. |
| `ollama serve` | Arranca el servicio a mano (el instalador normalmente ya lo deja en marcha). |

`ollama ps` es el que más se olvida y el que más ayuda: la columna `PROCESSOR` te dice si el modelo cabe entero en la GPU (`100% GPU`), en parte (`40%/60% CPU/GPU`) o solo corre en CPU (`100% CPU`). Esa columna explica casi siempre por qué algo va lento.

## Cómo encaja con tu código

Tu aplicación no habla con el programa de línea de comandos, sino con el servicio por HTTP. Eso significa que cualquier lenguaje que pueda hacer una petición POST puede usar Ollama, sin SDK obligatorio. El detalle de las llamadas está en [La API REST de Ollama](API-REST-de-Ollama.md).

Ollama ofrece además un endpoint compatible con la API de OpenAI (`/v1/chat/completions`). Si ya tienes código escrito contra esa API, normalmente basta con cambiar la dirección base a `http://localhost:11434/v1` y poner cualquier texto como clave.

## Errores frecuentes

- **`connection refused` en el puerto 11434:** el servicio no está en marcha. Arráncalo con `ollama serve` o reinicia la aplicación de Ollama.
- **`model 'xxx' not found`:** el modelo no está descargado, o la etiqueta no coincide con la que usas (`qwen2.5` no es `qwen2.5:7b` si descargaste otra variante). Compara con `ollama list`.
- **Va muy lento, o el equipo se queda sin memoria:** el modelo es demasiado grande para tu equipo. Consulta `ollama ps` y la guía de [Modelos y rendimiento](Modelos-y-Rendimiento.md).

## Buenas prácticas avanzadas

- **Fija siempre la etiqueta del modelo en tu código** — `qwen2.5:7b`, no `qwen2.5`. `latest` puede apuntar mañana a otra variante y cambiar el comportamiento de tu aplicación sin que toques nada.
- **Comprueba el modelo en el arranque, no en la primera petición** — una llamada a `/api/tags` al iniciar tu aplicación permite avisar con un mensaje claro ("falta `ollama pull qwen2.5:7b`") en lugar de un error genérico a mitad de un proceso largo.
- **Trata el servicio como un recurso local sin autenticación** — Ollama no pide clave. Escucha solo en `localhost` por defecto; si lo expones a la red (`OLLAMA_HOST=0.0.0.0`), cualquiera que llegue a ese puerto puede usar tu GPU y tus modelos. Déjalo detrás de un firewall o una VPN.
- **Valida siempre lo que devuelve el modelo** — que el modelo corra en tu máquina no lo hace más fiable. Una respuesta sigue siendo texto generado: compruébala en código antes de fiarte de ella.
- **Cuenta el tiempo de la primera llamada aparte** — cargar el modelo puede tardar decenas de segundos. Si mides latencia o pones un *timeout*, haz una llamada de calentamiento antes o el primer resultado te dará una falsa alarma.

## Documentación oficial

- [Repositorio de Ollama](https://github.com/ollama/ollama) — el README resume los comandos; la carpeta `docs` tiene la referencia de la API, la configuración y el formato del Modelfile.
- [Biblioteca de modelos](https://ollama.com/library) — qué modelos hay, qué tamaños ofrece cada uno y cuánto ocupan; consúltala antes de descargar.

## Recursos didácticos

- [Structured outputs en Ollama](https://ollama.com/blog/structured-outputs) — la entrada del blog que explica cómo forzar una respuesta con esquema JSON, con ejemplos en Python y JavaScript.

---

*En resumen: Ollama convierte "ejecutar un modelo de lenguaje en mi máquina" en `ollama pull`, `ollama run` y una API HTTP en `localhost`; lo que decide si te sirve es cuánta memoria tienes y qué tamaño de modelo eliges.*
