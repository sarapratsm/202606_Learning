# Herencia de eventos del DOM

## ¿Qué es?

Los objetos de evento que recibe un `addEventListener` forman una **jerarquía de clases**: `Event` → `UIEvent` → `MouseEvent` → `PointerEvent`. Cada nivel hereda todas las propiedades del anterior y añade las suyas, igual que en cualquier lenguaje con clases.

## ¿Por qué existe?

El navegador necesita describir cosas muy distintas: un clic, una tecla, un cambio en un formulario, un toque con el dedo. Todas comparten una base (qué ocurrió, sobre qué elemento, si se puede cancelar) y cada familia añade lo suyo: un evento de ratón sabe en qué coordenadas ocurrió; uno de teclado, qué tecla era.

En lugar de repetir lo común en cada tipo de evento, el DOM lo reparte en capas. Si vienes de C# o Java, es exactamente esto:

```csharp
class Event { /* type, target... */ }
class UIEvent : Event { /* view, detail */ }
class MouseEvent : UIEvent { /* clientX, clientY, button... */ }
class PointerEvent : MouseEvent { /* pointerType, pressure... */ }
```

Una consecuencia práctica: un `PointerEvent` **es** un `MouseEvent`. Por eso el mismo código que lee `clientX` en un evento de ratón sigue funcionando cuando pasas a eventos de puntero (que cubren ratón, táctil y lápiz).

## ¿Cuándo y para qué se usa?

Aparece cada vez que un manejador necesita algo más que "ha pasado algo". Escenarios típicos con una app de dibujo o un tablero de tarjetas:

- Saber **dónde** se ha hecho clic (`clientX`, `clientY`) para colocar un elemento.
- Reaccionar solo a **un botón concreto** del ratón (`button`): el principal selecciona, el central cierra.
- Distinguir **ratón, dedo o lápiz** (`pointerType`) para ajustar el comportamiento.

Y también al leer documentación: entender la jerarquía es lo que permite saber en qué página buscar una propiedad.

---

## La jerarquía de un vistazo

Estas son las propiedades más usadas de cada nivel. La columna "Nivel" es la clase donde la propiedad está **definida**:

| Nivel | Qué aporta | Propiedades y métodos habituales |
|---|---|---|
| `Event` | Lo común a todo evento | `type`, `target`, `timeStamp`, `preventDefault()`, `stopPropagation()` |
| `UIEvent` | Evento originado en la interfaz de usuario | `view`, `detail` |
| `MouseEvent` | Posición y botones del ratón | `clientX`, `clientY`, `button`, `buttons`, `ctrlKey`, `shiftKey` |
| `PointerEvent` | Cualquier dispositivo apuntador | `pointerId`, `pointerType`, `isPrimary`, `pressure` |

Otras ramas del árbol (`KeyboardEvent`, `InputEvent`, `TouchEvent`...) cuelgan de `UIEvent` o de `Event` y no tienen `clientX`: por eso un evento de teclado no sabe nada de coordenadas.

## Heredar en la práctica: `evento.button` sin salir de `PointerEvent`

Un listener de `pointerdown` recibe un `PointerEvent`. Aun así, puede leer propiedades de los cuatro niveles directamente sobre el objeto:

```js
document.addEventListener("pointerdown", (evento) => {
    console.log(evento.type);        // "pointerdown"  → definida en Event
    console.log(evento.clientX);     // 342            → definida en MouseEvent
    console.log(evento.button);      // 0              → definida en MouseEvent
    console.log(evento.pointerType); // "mouse"        → definida en PointerEvent
});
```

No hay que escribir `MouseEvent` en ninguna parte: la herencia hace que `button` esté disponible en la instancia.

### Cómo leer la notación de MDN

La documentación escribe `MouseEvent.button`. Esa notación es una **dirección**, no una instrucción: indica en qué clase se define la propiedad y, por tanto, en qué página de MDN encontrarla. Para usarla, siempre se accede desde la instancia (`evento.button`), nunca desde la clase.

Aplicado al caso anterior: `button` no aparece en la página de `PointerEvent` entre las propiedades propias, sino en la de `MouseEvent`. La página de `PointerEvent` lo recuerda en su cabecera, donde indica de quién hereda.

## Comprobar la herencia con `instanceof`

El operador `instanceof` pregunta si un objeto pertenece a una clase **o a cualquiera de sus antepasados**. Es el equivalente de `is` en C#:

```js
document.addEventListener("pointerdown", (evento) => {
    console.log(evento instanceof PointerEvent); // true
    console.log(evento instanceof MouseEvent);   // true
    console.log(evento instanceof UIEvent);      // true
    console.log(evento instanceof Event);        // true
    console.log(evento instanceof KeyboardEvent);// false: es de otra rama
});
```

La relación es de **un solo sentido**: todo `PointerEvent` es un `MouseEvent`, pero no al revés. Un evento `mousedown` clásico es un `MouseEvent` a secas y no tiene `pointerType`.

Para ver la cadena completa, se puede recorrer la cadena de prototipos, que es el mecanismo con el que JavaScript implementa la herencia:

```js
let nivel = Object.getPrototypeOf(evento);
while (nivel) {
    console.log(nivel.constructor.name);
    nivel = Object.getPrototypeOf(nivel);
}
// PointerEvent
// MouseEvent
// UIEvent
// Event
// Object
```

El último escalón, `Object`, es la raíz de todos los objetos de JavaScript. Para inspeccionarlo sin código, `console.dir(evento)` en las DevTools muestra el objeto y, desplegando `[[Prototype]]` repetidas veces, la misma cadena.

## Lo propio de `PointerEvent`: `pointerType`

`PointerEvent` existe para unificar ratón, pantalla táctil y lápiz bajo un único modelo. Lo que añade sobre `MouseEvent` es justo la información sobre el dispositivo:

```js
document.addEventListener("pointerdown", (evento) => {
    switch (evento.pointerType) {
        case "mouse": console.log("Ratón");  break;
        case "touch": console.log("Dedo");   break;
        case "pen":   console.log("Lápiz");  break;
    }
});
```

Con esto una misma comprobación cubre los tres dispositivos, y por eso `pointerdown` sustituye con ventaja a `mousedown` + `touchstart` escritos por separado.

## Filtrar por botón con *early return*

Un `pointerdown` se dispara con **cualquier** botón. Para actuar solo con el principal, se mira `button` al principio del manejador y se sale si no coincide. Esta técnica se llama *early return* (salida temprana) y evita anidar todo el cuerpo dentro de un `if`.

Los valores de `button`:

| Valor | Botón |
|---|---|
| `0` | Principal (normalmente el izquierdo) |
| `1` | Central (rueda) |
| `2` | Secundario (normalmente el derecho) |
| `3` / `4` | Atrás / adelante (ratones con botones laterales) |

Si el sistema tiene los botones invertidos (configuración para zurdos), el `0` corresponde al que el usuario haya definido como principal.

```js
// ❌ MAL — el cuerpo queda dentro del if
tablero.addEventListener("pointerdown", (evento) => {
    if (evento.button === 0) {
        seleccionarTarjeta(evento.target);
        actualizarPanel();
    }
});

// ✅ BIEN — se descarta primero lo que no interesa
tablero.addEventListener("pointerdown", (evento) => {
    if (evento.button !== 0) return;

    seleccionarTarjeta(evento.target);
    actualizarPanel();
});
```

Las dos versiones hacen lo mismo. La segunda gana cuando el manejador crece: cada condición de descarte ocupa una línea al principio y el camino principal queda sin sangría.

¿Y en táctil o con lápiz? En un `pointerdown` de un toque con el dedo, `button` vale `0`, igual que el clic principal del ratón. Un único `if (evento.button !== 0) return;` ya cubre los tres dispositivos, sin consultar `pointerType`.

---

## Buenas prácticas avanzadas

- **Programa contra el nivel más general que necesites.** Si el manejador solo usa `clientX`, `clientY` y `button`, no depende de `PointerEvent`, sino de `MouseEvent`. Mientras te mantengas ahí, el mismo código sirve para `mousedown` y `pointerdown`. Solo baja a `PointerEvent` (con `pointerType`, `pressure`...) cuando de verdad necesites distinguir el dispositivo.
- **No confundas `button` con `buttons`.** `button` dice **qué botón provocó** este evento; `buttons` es una máscara de bits con **qué botones están pulsados ahora**. En `pointermove`, `button` vale `-1` (nadie ha cambiado de estado) y la pregunta correcta es `(evento.buttons & 1) !== 0` para saber si el principal sigue pulsado, por ejemplo al arrastrar.
- **Filtrar el botón no elimina el menú contextual.** Con el botón secundario, `pointerdown` se dispara igualmente y después llega el evento `contextmenu`, que abre el menú del navegador. Tu `return` ignora el `pointerdown`, pero el menú aparece. Si quieres sustituirlo por uno propio, escucha `contextmenu` y llama a `evento.preventDefault()`, sin romper el acceso desde teclado.
- **Elige `click` para acciones y `pointerdown` para respuesta inmediata.** `click` se activa también con teclado (`Enter` o espacio sobre un botón) y es lo que espera la tecnología de asistencia. `pointerdown` responde antes, pero solo existe para punteros: reserva su uso para dibujo, arrastre o juegos, no para botones normales.
- **En tests, crea el tipo correcto de evento.** `new MouseEvent("pointerdown")` crea un `MouseEvent`, no un `PointerEvent`: no tendrá `pointerType`. Usa `new PointerEvent("pointerdown", { button: 0, pointerType: "mouse" })` y comprueba que tu entorno de pruebas define `PointerEvent`, porque algunos entornos simulados del DOM no lo incluyen según su versión.

## Documentación oficial

- [PointerEvent (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/PointerEvent) — la lista de propiedades propias (`pointerType`, `isPrimary`, `pressure`) y, arriba, de quién hereda. Empieza por la tabla de propiedades de instancia.
- [MouseEvent.button (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent/button) — los valores de `button` con ejemplos y los matices de los botones configurados a la inversa.
- [Pointer Events (W3C)](https://www.w3.org/TR/pointerevents/) — la especificación: cuando MDN no concreta cómo se comporta `button` o `buttons` con táctil o lápiz, la respuesta normativa está aquí.

## Recursos didácticos

- [Guía de Pointer Events (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Pointer_events) — recorrido por el modelo de eventos de puntero, con los tipos de evento y casos de uso.
- [Herencia y cadena de prototipos (MDN)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Inheritance_and_the_prototype_chain) — el mecanismo de JavaScript que hay debajo de la jerarquía de eventos y de `instanceof`.

---

*En resumen: un `PointerEvent` es un `MouseEvent` con información extra del dispositivo, así que `evento.clientX` y `evento.button` se leen directamente de él, y `button !== 0` con un `return` temprano basta para atender solo al botón principal.*
