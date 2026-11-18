# Eventos del DOM — Guía de tecnologías

Cómo están organizados los objetos de evento que el navegador entrega a `addEventListener` y cómo aprovechar esa organización: qué propiedades trae cada tipo de evento, por qué un `PointerEvent` puede leerse como un `MouseEvent` y cómo reaccionar solo al botón del ratón que interesa.

Está pensada para perfiles backend que ya han escrito algo de JavaScript, pero no se han parado a mirar qué es exactamente el objeto `evento` de un manejador. Se apoya en la analogía con la herencia de clases de C# o Java.

---

## Orden de lectura recomendado

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Herencia de eventos del DOM](Herencia-de-eventos-del-DOM.md) | La jerarquía `Event` → `UIEvent` → `MouseEvent` → `PointerEvent`, cómo leer la notación de MDN, `instanceof`, `pointerType` y el filtrado por botón con *early return*. |
