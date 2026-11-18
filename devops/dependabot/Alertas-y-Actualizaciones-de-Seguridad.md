# Alertas y actualizaciones de seguridad

## ¿Qué es?

Son las dos funciones de Dependabot que reaccionan a vulnerabilidades: las **alertas** te avisan de que una dependencia que usas tiene un fallo de seguridad publicado, y las **security updates** abren la pull request que la sube a la versión corregida.

## ¿Por qué existe?

Una vulnerabilidad en una librería popular se publica con un identificador (un CVE o un GHSA de GitHub), una gravedad y la versión en la que se corrige. A partir de ese momento cualquier persona atacante sabe exactamente qué versiones son explotables, y la ventana para actualizar se mide en días, no en meses.

Nadie puede vigilar a mano los avisos de las cientos de dependencias de un proyecto, ni las de las dependencias de sus dependencias. Esta parte de Dependabot hace ese cruce de forma automática: compara lo que tu repositorio usa con una base de datos de vulnerabilidades y avisa solo cuando hay coincidencia.

> A diferencia de las *version updates* de la [ficha anterior](Configuracion-de-dependabot-yml.md), esto no se configura con un fichero ni sigue un calendario: se activa con casillas en los ajustes y reacciona en cuanto aparece un aviso.

## ¿Cuándo y para qué se usa?

En todo repositorio, incluidos los que «ya no se tocan». Es el caso donde más valor tiene: un proyecto parado con dependencias de hace tres años es el que más probabilidades tiene de arrastrar una vulnerabilidad conocida, y el que menos gente mira.

---

## Cómo sabe GitHub que eres vulnerable

El mecanismo tiene dos mitades:

1. **El grafo de dependencias** (*dependency graph*): GitHub lee los ficheros de dependencias del repositorio (`package.json` y `package-lock.json`, los `.csproj`, `requirements.txt`...) y construye la lista de paquetes y versiones que usas, incluidas las **transitivas** en los ecosistemas donde hay fichero de bloqueo.
2. **La GitHub Advisory Database**: un catálogo de avisos de seguridad (los de GitHub más los CVE de las bases públicas) que indica, para cada uno, qué paquete afecta, qué rango de versiones es vulnerable y cuál lo corrige.

Cuando un paquete de tu grafo cae dentro de un rango vulnerable, nace una **alerta**. Por eso un repositorio sin ficheros de bloqueo suele dar peores resultados: sin `package-lock.json`, las dependencias transitivas no se pueden resolver con exactitud.

## Activarlo

En el repositorio: *Settings → Advanced Security* (en algunas cuentas figura como *Code security*). Tres interruptores, que se encadenan:

| Interruptor | Qué habilita | Depende de |
|---|---|---|
| Dependency graph | La lista de dependencias de la que parte todo lo demás. | Nada (en repositorios públicos ya viene activado). |
| Dependabot alerts | Las alertas de vulnerabilidad. | Dependency graph. |
| Dependabot security updates | La pull request automática con la corrección. | Dependabot alerts. |

En una organización se puede activar para todos los repositorios a la vez y para los que se creen en el futuro, que es lo recomendable (ver [Seguridad de la organización](../github-organizaciones/Seguridad-de-la-organizacion.md)).

## Leer una alerta

Las alertas viven en la pestaña *Security → Dependabot* del repositorio. Cada una lleva la información justa para decidir:

- la **gravedad** (baja, media, alta, crítica), derivada de la puntuación CVSS del aviso,
- el **paquete afectado** y el rango de versiones vulnerable,
- la **versión corregida**, si existe,
- si la dependencia es **directa** o **transitiva**,
- y, si la hay, un enlace a la pull request que lo arregla.

Una alerta típica en una tienda online: *«`axios` < 1.7.4 — gravedad alta — corregido en 1.7.4»*. Si has activado las security updates, junto a ella aparece una pull request titulada algo como `Bump axios from 1.6.2 to 1.7.4`, con la corrección ya preparada.

## Security updates frente a version updates

Las dos abren pull requests con el mismo aspecto, pero no son lo mismo y no se configuran igual:

| | Security updates | Version updates |
|---|---|---|
| **Qué las dispara** | Una alerta de vulnerabilidad. | El calendario del `dependabot.yml`. |
| **Cuándo** | En cuanto hay alerta, sin esperar al calendario. | Cuando toca según `schedule`. |
| **Qué actualiza** | Solo el paquete vulnerable, a la versión mínima que lo corrige. | Cualquier dependencia con versión nueva. |
| **Necesita `dependabot.yml`** | No. | Sí. |
| **Límite de abiertas** | Propio (10 por defecto), independiente del de las *version updates*. | `open-pull-requests-limit` (5 por defecto). |
| **Afectan `cooldown` o `ignore`** | `cooldown` no se aplica. Con `ignore`, hay matices: consulta la documentación antes de fiarte de él. | Sí. |

Que sean independientes tiene una consecuencia útil: aunque tengas el límite de *version updates* agotado, una vulnerabilidad grave seguirá recibiendo su pull request.

Se puede personalizar su aspecto con la clave `applies-to` en los grupos, para que las security updates se agrupen de forma distinta a las demás:

```yaml
    groups:
      parches-de-seguridad:
        applies-to: security-updates
        patterns: ["*"]
```

## Cuando no se puede actualizar

El caso fácil (hay versión corregida y la subida no rompe nada) lo resuelve la pull request. Los casos difíciles son los que justifican entender esta ficha:

**Dependencia transitiva.** La vulnerabilidad no está en un paquete que tú declaras, sino en uno que arrastra otro. Dependabot intenta subir el paquete padre; si el padre aún no actualiza su dependencia, no hay pull request posible. Mientras tanto, se puede forzar la versión segura desde tu propio proyecto:

```json
{
  "overrides": {
    "follow-redirects": "1.15.6"
  }
}
```

Este `overrides` de `package.json` (npm) obliga a usar esa versión en todo el árbol, y en NuGet existe un equivalente con la gestión central de paquetes (`CentralPackageTransitivePinningEnabled`). Es una solución temporal: hay que quitarla cuando el paquete padre se actualice, o te quedarás con una versión forzada olvidada.

**No hay versión corregida.** Hay aviso, pero el autor no ha publicado parche. Las opciones reales son sustituir la librería, aislar el uso vulnerable o aceptar el riesgo de forma explícita, y documentar la decisión.

**La corrección es una versión mayor incompatible.** La pull request existe, el CI falla. No se descarta: es una tarea con fecha, y la alerta es el recordatorio de que hay que hacer esa migración.

## Descartar una alerta con criterio

A veces una alerta es correcta pero no te afecta: la función vulnerable no se llama nunca, o el paquete solo se usa en pruebas. GitHub permite descartarla, y pide un motivo:

| Motivo | Cuándo usarlo |
|---|---|
| *A fix has already been started* | Hay una pull request o tarea en marcha. |
| *No bandwidth to fix this* | Se reconoce el riesgo pero no hay capacidad ahora. |
| *Risk is tolerable to this project* | Se ha evaluado y se asume. |
| *This alert is inaccurate or incorrect* | Falso positivo. |
| *Vulnerable code is not actually used* | El código vulnerable no se alcanza desde tu aplicación. |

Descartar sin motivo real es esconder el problema. Elige el motivo exacto y añade un comentario: queda registrado, y dentro de un año explicará por qué esa alerta no se actuó.

## Contrastar en local y en CI

Las alertas viven en GitHub, pero puedes consultar lo mismo desde tu máquina o desde un paso del pipeline:

```bash
# .NET: lista paquetes con vulnerabilidades conocidas, incluidos los transitivos
dotnet list package --vulnerable --include-transitive

# npm: audita el árbol contra la base de avisos de npm
npm audit --omit=dev
```

La primera salida lista el paquete, la versión instalada, la gravedad y el enlace al aviso. Es útil para comprobar que, tras fusionar la pull request, el aviso desapareció de verdad. Tenlo en cuenta: `dotnet list package --vulnerable` termina con código 0 aunque encuentre vulnerabilidades, así que, para romper el pipeline, hay que analizar su salida.

## Buenas prácticas avanzadas

- **Fija un plazo por gravedad, no un orden de llegada.** Por ejemplo: crítica en días, alta en dos semanas, media en el siguiente ciclo. Sin un criterio escrito, las alertas se atienden por quien grita más fuerte, y las que nadie mira son las que acaban explotándose.
- **El verde del CI no basta en una security update.** Comprueba que el comportamiento afectado sigue funcionando, sobre todo si la corrección salta de versión menor: los parches de seguridad a veces cambian justo el comportamiento por defecto que explotaba el fallo.
- **Atiende las alertas de dependencias de desarrollo con menos urgencia, pero no las ignores.** Una vulnerabilidad en una herramienta de *build* no está en producción, pero se ejecuta en tu CI con acceso a secrets, y por ahí también entran los ataques.
- **Revisa las alertas descartadas cada trimestre.** El motivo «código no usado» deja de ser cierto en cuanto alguien empieza a usar ese código; es una decisión con fecha de caducidad.
- **Activa las notificaciones a nivel de equipo, no de persona.** Una alerta que solo le llega al correo de quien creó el repositorio hace tres años no la ve nadie. Dirígela a un canal o a un grupo responsable.

## Documentación oficial

- [Acerca de las alertas de Dependabot](https://docs.github.com/es/code-security/dependabot/dependabot-alerts/about-dependabot-alerts) — cómo se detectan, qué datos tiene cada alerta y cómo se descartan.
- [Acerca de las actualizaciones de seguridad de Dependabot](https://docs.github.com/es/code-security/dependabot/dependabot-security-updates/about-dependabot-security-updates) — cuándo se abre la pull request y qué casos quedan sin cubrir.
- [GitHub Advisory Database](https://github.com/advisories) — la fuente de los avisos, navegable por ecosistema y gravedad.

## Recursos didácticos

- [CVSS Calculator de FIRST](https://www.first.org/cvss/calculator/3.1) — permite jugar con las métricas (vector de ataque, complejidad, impacto) y ver cómo se obtiene la puntuación de gravedad que muestran las alertas.

---

*En resumen: las alertas te dicen qué dependencia es vulnerable y las security updates te preparan la corrección; lo difícil es decidir qué hacer cuando la corrección no es una simple subida de versión.*
