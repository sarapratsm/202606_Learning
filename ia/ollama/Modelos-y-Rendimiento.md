# Modelos y rendimiento en Ollama

## ¿Qué es?

Es la guía para elegir qué modelo descargar y entender por qué un modelo va rápido o lento en tu equipo: tamaño en parámetros, cuantización, memoria necesaria y uso de la tarjeta gráfica.

## ¿Por qué existe?

En la nube, el proveedor decide el hardware y tú solo eliges un nombre de modelo. En local, la decisión es tuya y tiene consecuencias inmediatas: un modelo demasiado grande no arranca o tarda minutos por respuesta; uno demasiado pequeño responde rápido pero mal. Elegir bien exige entender tres ideas (parámetros, cuantización y memoria) que se resumen en una pregunta práctica: **¿cabe en mi memoria?**

## ¿Cuándo y para qué se usa?

Antes de descargar un modelo nuevo, cuando algo va lento y quieres saber por qué, o cuando decides si un equipo concreto (un portátil, un servidor sin GPU) sirve para una tarea.

---

## Lo que significa el nombre: parámetros y etiqueta

En `qwen2.5:7b`, el `7b` son **7 mil millones de parámetros**, los números aprendidos que forman el modelo. Más parámetros suele significar mejor calidad, mayor tamaño en memoria y mayor lentitud:

| Tamaño | Orden de magnitud en disco | Qué esperar |
|---|---|---|
| 1-3B | 1-2 GB | Muy rápido; vale para clasificación sencilla y respuestas cortas, falla con instrucciones complejas. |
| 7-8B | 4-5 GB | El punto de partida habitual: sigue instrucciones y formatos razonablemente bien. |
| 13-14B | 8-9 GB | Mejor razonamiento y menos errores de formato; exige bastante memoria. |
| 30B o más | 18 GB en adelante | Calidad cercana a modelos de nube, pero necesita equipo específico. |

Las cifras de disco son aproximadas y asumen la cuantización habitual (siguiente apartado). Cada ficha del modelo en la [biblioteca](https://ollama.com/library) muestra los tamaños reales.

## Cuantización: cabe en menos memoria a cambio de algo de calidad

Un modelo guarda cada parámetro con un número. Con 16 bits de precisión, un modelo de 7B pesaría unos 14 GB. La **cuantización** reduce esa precisión (a 8 o a 4 bits) y encoge el fichero proporcionalmente, con una pérdida de calidad pequeña en la mayoría de usos.

Ollama descarga por defecto variantes de 4 bits (etiquetadas como `Q4_K_M` y similares) porque son el mejor compromiso. Puedes ver cuál tienes con `ollama show <modelo>`. Si ves etiquetas como `q8_0` o `fp16` en la biblioteca, son versiones más precisas, más pesadas y más lentas.

## ¿Cabe en mi memoria?

La regla práctica: necesitas **el tamaño del modelo en disco, más un margen** para el contexto y el sistema. Un modelo de 4,7 GB pide unos 6-7 GB libres. Dónde está esa memoria cambia la velocidad radicalmente:

- **En la GPU (VRAM):** la inferencia es rápida, de decenas de tokens por segundo.
- **En la RAM, con CPU:** funciona, pero entre 5 y 20 veces más lento. Un modelo de 7B puede tardar varios minutos en responder a un texto largo.
- **Repartido entre ambas:** si el modelo no cabe entero en la GPU, Ollama coloca parte de las capas en cada sitio; la velocidad cae hacia la de la CPU.

Para ver dónde está cargado ahora mismo:

```bash
ollama ps
```

```text
NAME          ID              SIZE      PROCESSOR    CONTEXT    RUNNER      UNTIL
qwen2.5:7b    845dbda0ea48    4.8 GB    100% GPU     4096       llamacpp    4 minutes from now
```

`PROCESSOR` es la columna clave. `100% GPU` es lo ideal; `100% CPU` significa que tu equipo no tiene GPU compatible (o no hay VRAM libre) y todo irá más lento. `CONTEXT` es la ventana de contexto con la que está cargado el modelo (aquí 4096 tokens, aunque el modelo admita 32 768): si envías más texto que eso, se recorta (ver [La API REST](API-REST-de-Ollama.md)). `UNTIL` indica cuándo se descargará de memoria por inactividad. Las columnas pueden variar entre versiones de Ollama.

## Tiempo en memoria: `keep_alive`

Por defecto, Ollama mantiene un modelo cargado unos 5 minutos tras la última petición y luego lo descarga. Si tu aplicación llama de forma esporádica, cada llamada pagará de nuevo la carga. Se puede ampliar por petición con `"keep_alive": "30m"` o globalmente con la variable de entorno `OLLAMA_KEEP_ALIVE`. Poner `-1` lo mantiene cargado indefinidamente a costa de ocupar la memoria.

## Personalizar un modelo: el Modelfile

Un `Modelfile` define una variante de un modelo con un mensaje de sistema y parámetros propios, y se instala con un nombre nuevo:

```text
FROM qwen2.5:7b
PARAMETER temperature 0
PARAMETER num_ctx 8192
SYSTEM Eres un asistente que extrae datos de pedidos y responde solo con JSON.
```

```bash
ollama create extractor-pedidos -f Modelfile
ollama run extractor-pedidos
```

Es útil para repartir una configuración probada dentro de un equipo. Para una aplicación, suele ser más claro pasar `options` y el mensaje `system` en cada petición: la configuración queda visible en tu código en vez de escondida en un modelo local.

## Dónde se guardan los modelos en disco

Cada modelo que descargas ocupa varios gigas, así que conviene saber dónde van a parar. Ollama los guarda en una carpeta `models` dentro de su directorio de datos:

| Sistema | Ruta por defecto |
|---|---|
| Windows | `C:\Users\<usuario>\.ollama\models` |
| macOS | `~/.ollama/models` |
| Linux (ejecutado como tu usuario) | `~/.ollama/models` |
| Linux (instalado como servicio systemd) | `/usr/share/ollama/.ollama/models` |

Dentro verás estas subcarpetas:

| Carpeta | Contenido |
|---|---|
| `blobs/` | Los ficheros pesados: los pesos del modelo y otras piezas (plantilla, parámetros). Se nombran por su huella, `sha256-...`, y es donde se va casi todo el espacio. |
| `manifests/` | Formato clásico de manifiestos: un pequeño fichero por modelo y etiqueta que lista qué blobs lo componen. Cuelga de `registry.ollama.ai`. |
| `manifests-v2/` | Formato más reciente de los mismos manifiestos, que cuelga de `ollama.com`. |
| `metadata/` | Un `.json` por blob con información adicional sobre él. |

Que `manifests-v2/` y `metadata/` aparezcan o no depende de la versión de Ollama; si faltan, no pasa nada. La idea es la misma que en un registro de contenedores: los **blobs** se identifican por su contenido y se comparten (dos modelos con la misma plantilla guardan una sola copia), y el **manifiesto** es la lista que dice qué blobs forman cada modelo. Por eso no debes borrar ficheros de `blobs/` a mano: usa `ollama rm <modelo>`, que retira el manifiesto y los blobs que ya nadie usa.

### Cambiar la ubicación

Si el disco del sistema va justo, mueve los modelos con la variable de entorno `OLLAMA_MODELS`:

1. Cierra Ollama.
2. Copia **las cuatro subcarpetas** de la ruta antigua a la nueva (los manifiestos sin sus blobs, o al revés, dejan los modelos inservibles).
3. Define `OLLAMA_MODELS` con la ruta nueva: en Windows, en las variables de entorno del usuario; en Linux con systemd, con `systemctl edit ollama` y una línea `Environment="OLLAMA_MODELS=/ruta/nueva"`.
4. Arranca Ollama de nuevo y comprueba con `ollama list` que los modelos siguen ahí.

En Linux con servicio, el usuario `ollama` debe poder leer y escribir en la carpeta nueva.

## Errores frecuentes

- **Descargar el modelo más grande "porque seguro que es mejor":** si no cabe, irá peor que uno mediano que sí cabe. Empieza por 7-8B y sube solo si la calidad no basta.
- **Medir velocidad con la primera llamada:** incluye la carga del modelo. Mide desde la segunda.
- **Subir `num_ctx` sin mirar la memoria:** un contexto mayor consume más memoria; con un modelo ya justo, puede expulsarlo de la GPU y hundir la velocidad.
- **Ejecutar dos modelos a la vez en un equipo justo:** se disputan la memoria y ambos se ralentizan; `ollama ps` lo muestra.
- **Quedarse sin disco sin saber por qué:** los modelos viven en `models/blobs`. Revisa `ollama list` y elimina los que no uses con `ollama rm`.

## Buenas prácticas avanzadas

- **Elige el modelo por la tarea, no por el ranking** — para extraer campos o clasificar, un 7B con esquema forzado suele bastar; reserva modelos grandes para razonamiento. Pruébalo con diez ejemplos reales tuyos antes de decidir.
- **Mide en tu hardware, con tus textos** — los tiempos publicados dependen de GPU y longitud. Cronometra una llamada caliente con un texto representativo y fija tus *timeouts* a partir de eso.
- **Controla el contexto, que es lo que más memoria añade** — el coste de memoria crece con `num_ctx`. Usa el menor que cubra tus textos más largos en lugar del máximo del modelo.
- **Fija la versión exacta del modelo en producción** — `ollama show` muestra un identificador (digest). Si un modelo se actualiza en la biblioteca, tu comportamiento puede cambiar al volver a descargarlo; registra el digest con el que validaste.
- **Comprueba que el cuantizado no rompe tu caso** — pasar de 8 a 4 bits suele ser inocuo, pero en tareas de formato estricto o idiomas minoritarios puede notarse. Compara ambas variantes con tu conjunto de pruebas.

## Documentación oficial

- [Biblioteca de modelos de Ollama](https://ollama.com/library) — tamaños, cuantizaciones disponibles y contexto máximo de cada modelo.
- [Referencia del Modelfile](https://github.com/ollama/ollama/blob/main/docs/modelfile.md) — todas las instrucciones (`FROM`, `PARAMETER`, `SYSTEM`, `TEMPLATE`) y los parámetros admitidos.
- [Preguntas frecuentes](https://github.com/ollama/ollama/blob/main/docs/faq.md) — variables de entorno (`OLLAMA_HOST`, `OLLAMA_KEEP_ALIVE`, `OLLAMA_MODELS`), soporte de GPU y dónde se guardan los modelos.

---

*En resumen: elige el modelo más pequeño que haga bien tu tarea y que quepa entero en la memoria de tu GPU; `ollama ps` te dice si lo consigue.*
