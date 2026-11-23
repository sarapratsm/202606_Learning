# La API REST de Ollama

## ¿Qué es?

La API REST de Ollama es el conjunto de rutas HTTP que expone el servicio local (por defecto en `http://localhost:11434`) para chatear con un modelo, listar los que tienes instalados o descargar otros. Es lo que usa tu código; la línea de comandos es solo un cliente más.

## ¿Por qué existe?

Una aplicación necesita enviar texto a un modelo y recibir texto de vuelta de forma programática, no escribiendo en una terminal. La API da esa puerta con el formato más universal posible: JSON sobre HTTP. No hay SDK obligatorio, ni dependencias, ni clave: cualquier lenguaje con un cliente HTTP funciona.

## ¿Cuándo y para qué se usa?

Siempre que una aplicación quiera apoyarse en un modelo local: un clasificador de tickets de soporte, un extractor de datos de facturas, un asistente interno, un script que resume documentos. Esta guía cubre las tres llamadas que cubren casi todos los casos: comprobar qué hay instalado, conversar con un modelo y pedirle una respuesta con forma de datos.

---

## Comprobar qué modelos hay

```bash
curl http://localhost:11434/api/tags
```

Devuelve los modelos descargados:

```json
{"models":[{"name":"qwen2.5:7b","size":4683087332,"details":{"parameter_size":"7.6B","quantization_level":"Q4_K_M"}}]}
```

Es la forma más barata de saber si el servicio está vivo y si el modelo que necesitas está presente, sin gastar tiempo de inferencia.

## Conversar con un modelo: `/api/chat`

La ruta de chat recibe una lista de mensajes, cada uno con un `role` (`system`, `user` o `assistant`) y un `content`. El mensaje `system` fija las reglas del juego; el `user` es la petición.

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "qwen2.5:7b",
  "stream": false,
  "messages": [
    {"role": "system", "content": "Responde en una sola frase, en español."},
    {"role": "user", "content": "¿Qué es una clave foránea?"}
  ]
}'
```

Con `"stream": false` la respuesta llega entera en un único JSON; el texto está en `message.content`:

```json
{
  "model": "qwen2.5:7b",
  "message": {"role": "assistant", "content": "Una clave foránea es una columna que referencia la clave primaria de otra tabla."},
  "done": true,
  "total_duration": 2412345678
}
```

Sin `stream: false`, Ollama devuelve **un JSON por línea** a medida que el modelo genera, cada uno con un trozo del texto, y un último con `"done": true`. Eso es lo que quieres para una interfaz de chat que va pintando palabras; para un proceso por lotes, `stream: false` es más sencillo.

`/api/generate` existe también y recibe un único `prompt` en lugar de mensajes. Para empezar, quédate con `/api/chat`: es la que usan casi todos los ejemplos y permite separar instrucciones de datos.

## Ajustar la generación: `options`

El objeto `options` controla cómo genera el modelo. Tres que de verdad se tocan:

```json
{
  "model": "qwen2.5:7b",
  "stream": false,
  "messages": [{"role": "user", "content": "Clasifica: 'No me llega el pedido'"}],
  "options": {"temperature": 0, "num_ctx": 8192, "num_predict": 200}
}
```

| Opción | Qué hace | Cuándo tocarla |
|---|---|---|
| `temperature` | Cuánta aleatoriedad hay al elegir cada palabra. `0` es lo más determinista. | A `0` para extraer o clasificar; más alta para escritura creativa. |
| `num_ctx` | Tamaño de la **ventana de contexto** en tokens: cuánto texto (entrada + salida) cabe. | Cuando envías textos largos. Ver la advertencia más abajo. |
| `num_predict` | Máximo de tokens que generará. | Para acotar respuestas y evitar que un modelo se enrolle. |

> **Ojo con `num_ctx`.** Si el texto que envías es más largo que la ventana de contexto, Ollama no da error: **lo recorta en silencio** y el modelo responde con solo una parte del texto. Es la causa más común de "el modelo ignora la mitad de mi documento". Si envías textos de varios miles de palabras, fija `num_ctx` explícitamente y comprueba que el modelo lo soporta (`ollama show <modelo>` indica su máximo).
>
> Se comprueba fácil. Con un texto de unos 31 000 caracteres que empieza por "La palabra clave secreta es TULIPAN" y termina preguntando cuál es, y `qwen2.5:7b` con la ventana por defecto de esa instalación (4096 tokens), la respuesta fue `contexto`: el campo `prompt_eval_count` de la respuesta marcó solo ~2 050 tokens procesados, es decir, el principio del texto se había descartado. Con `"num_ctx": 16384`, `prompt_eval_count` subió a ~9 750 y la respuesta fue `TULIPAN`. Compara siempre `prompt_eval_count` con el tamaño de lo que enviaste: es la forma de detectar el recorte.

## Respuestas con forma de datos: `format`

Pedirle a un modelo "responde en JSON" funciona a medias: a veces añade explicaciones, a veces inventa campos. Ollama permite **forzar** la forma pasando un esquema JSON en `format`: el motor solo deja generar texto que cumpla el esquema.

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "qwen2.5:7b",
  "stream": false,
  "messages": [
    {"role": "user", "content": "Pedido #4821 de Laura Gómez, 3 camisetas, envío urgente."}
  ],
  "format": {
    "type": "object",
    "properties": {
      "orderId": {"type": "string"},
      "units": {"type": "integer"},
      "urgent": {"type": "boolean"}
    },
    "required": ["orderId", "units", "urgent"]
  }
}'
```

`message.content` viene como **texto** que contiene el JSON (hay que parsearlo, no es un objeto ya hecho):

```json
{"orderId": "4821", "units": 3, "urgent": true}
```

Forzar el esquema garantiza la **forma**, no la **verdad**: el modelo puede devolver un JSON válido con un dato equivocado. Por eso se valida igualmente el contenido en código (ver buenas prácticas).

## Errores frecuentes

| Respuesta | Causa | Qué hacer |
|---|---|---|
| `connection refused` | Servicio apagado. | Arrancar Ollama. |
| `404` con `{"error":"model 'x' not found"}` | El modelo no está descargado. | `ollama pull x`. |
| La petición tarda minutos o expira | Modelo cargándose por primera vez, o demasiado grande para tu memoria. | Calentar el modelo y revisar `ollama ps`. |
| Respuesta que ignora parte del texto | Texto más largo que `num_ctx`. | Subir `num_ctx` o dividir el texto. |
| `content` no es JSON válido pese a `format` | Respuesta cortada por `num_predict` demasiado bajo. | Subirlo o no limitarlo. |

## Buenas prácticas avanzadas

- **Separa instrucciones de datos** — pon las reglas en el mensaje `system` y los datos externos (un correo, el texto de una web) en `user`, delimitados con marcas claras. Si el texto procede de terceros, puede contener frases tipo "ignora lo anterior y…": trátalo siempre como datos, nunca como órdenes.
- **No te fíes de `format` para validar** — el esquema asegura que el JSON tiene la forma pedida, no que los valores sean ciertos. Comprueba rangos, longitudes y, si extraes citas de un texto, que la cita aparezca literalmente en él.
- **Fija `temperature: 0` en tareas de extracción** — reduce la variación entre ejecuciones y hace que los fallos sean reproducibles al depurar.
- **Pon un *timeout* propio y generoso** — la primera llamada incluye cargar el modelo. Un límite de 120 s es razonable para modelos de 7B en CPU; fija el tuyo midiendo, no adivinando. Y limita también el tamaño de respuesta que aceptas leer.
- **Reintenta una vez, no en bucle** — si la respuesta no es interpretable, un segundo intento suele arreglarlo; más intentos solo multiplican el tiempo cuando el problema es el modelo, no la suerte.
- **Cuida qué dirección aceptas** — si tu aplicación permite configurar la URL de Ollama, restríngela a `localhost`: aceptar cualquier URL convierte tu aplicación en una puerta para hacer peticiones a otras máquinas.

## Documentación oficial

- [Referencia de la API](https://github.com/ollama/ollama/blob/main/docs/api.md) — todas las rutas y parámetros; consúltala para opciones que esta guía no cubre (`keep_alive`, `images`, `tools`).
- [Structured outputs](https://ollama.com/blog/structured-outputs) — el funcionamiento del parámetro `format` con esquemas, con ejemplos.

---

*En resumen: con `POST /api/chat`, `stream: false` y un esquema en `format` tienes un modelo local convertido en una función que devuelve datos; recuerda que garantiza la forma, no la verdad, y que un texto más largo que `num_ctx` se recorta sin avisar.*
