# Dependabot

## ¿Qué es?

Dependabot es el servicio de GitHub que vigila las dependencias de un repositorio y abre pull requests automáticos para actualizarlas, ya sea porque hay una versión nueva o porque la que usas tiene una vulnerabilidad conocida.

## ¿Por qué existe?

Un proyecto real depende de cientos de paquetes (NuGet, npm, imágenes Docker, acciones de GitHub...) y cada uno publica versiones nuevas cuando quiere. Nadie revisa esa lista a mano, así que ocurre siempre lo mismo: el proyecto se queda congelado en las versiones del día en que se creó, y la actualización que nadie hizo en dos años se convierte en una migración de dos semanas con veinte cambios incompatibles a la vez.

Peor aún, algunas de esas versiones antiguas tienen fallos de seguridad publicados. Cuando se anuncia una vulnerabilidad grave, no hay tiempo para descubrir que estás cuatro versiones mayores por detrás.

Dependabot convierte esa tarea invisible en trabajo pequeño y visible: una pull request por actualización, con las notas de la versión, que pasa por tu CI como cualquier otro cambio.

> Si ya conoces `npm outdated` o `dotnet list package --outdated`, piensa en Dependabot como esos comandos ejecutándose solos todas las semanas y, además, escribiendo el commit con la actualización.

## ¿Cuándo y para qué se usa?

En cualquier repositorio alojado en GitHub que tenga dependencias, que es casi cualquiera. Los escenarios habituales:

- Una tienda online con un backend en .NET y un frontend en React: cada semana llegan las pull requests con las versiones nuevas de los paquetes NuGet y npm, y se revisan en cinco minutos en lugar de dejarlas acumular.
- Un `Dockerfile` con `FROM node:22`: Dependabot propone el cambio de imagen base cuando sale una versión nueva.
- Los workflows de GitHub Actions: también tienen dependencias (`uses: actions/checkout@v4`) y también envejecen.
- Un aviso de seguridad sobre una librería que usas: Dependabot abre la pull request con la versión corregida sin que nadie tenga que pedirlo.

---

## Las tres piezas que se confunden

«Dependabot» no es una sola cosa. Son tres funciones independientes que se activan por separado, y confundirlas es la causa de casi todas las dudas del tipo «¿por qué no me ha abierto nada?».

| Pieza | Qué hace | Cómo se activa |
|---|---|---|
| **Dependabot alerts** | Avisa de que una dependencia tiene una vulnerabilidad conocida. No toca el código. | Ajustes del repositorio (o de la organización). |
| **Dependabot security updates** | Abre una pull request que sube la dependencia a la versión que corrige la vulnerabilidad. | Ajustes del repositorio. Requiere las alertas. |
| **Dependabot version updates** | Abre pull requests para mantener **todas** las dependencias al día, haya o no vulnerabilidad. | Un fichero `.github/dependabot.yml` en el repositorio. |

La diferencia práctica: las dos primeras reaccionan a vulnerabilidades y funcionan con solo marcar casillas; la tercera es la que mantiene el proyecto actualizado y exige que escribas el fichero de configuración. Las alertas y las actualizaciones de seguridad se desarrollan en [Alertas y actualizaciones de seguridad](Alertas-y-Actualizaciones-de-Seguridad.md); el resto de esta ficha y la siguiente tratan sobre las *version updates*.

## Tu primer `dependabot.yml`

El fichero vive siempre en `.github/dependabot.yml`, en la rama por defecto. Este es un ejemplo mínimo y funcional para una tienda con backend en .NET y frontend en React:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/backend"
    schedule:
      interval: "weekly"

  - package-ecosystem: "npm"
    directory: "/frontend"
    schedule:
      interval: "weekly"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

Cada elemento de `updates` es una pareja «ecosistema + carpeta» que Dependabot revisa de forma independiente:

- `package-ecosystem` dice qué gestor de paquetes mirar (`nuget`, `npm`, `docker`, `github-actions`, `pip`, `maven`...).
- `directory` es la carpeta donde está el fichero de dependencias (`package.json`, el `.csproj`, el `Dockerfile`). Es la ruta desde la raíz del repositorio, y no se busca recursivamente: si pones la carpeta equivocada, no pasa nada visible, simplemente no se abre ninguna pull request.
- `schedule.interval` indica cada cuánto revisar.

Para `github-actions` el directorio es siempre `/`: Dependabot sabe que debe buscar en `.github/workflows/`.

Al hacer commit del fichero, Dependabot ejecuta una primera revisión casi de inmediato; no hace falta esperar a la fecha programada. Si el proyecto lleva tiempo sin actualizarse, la primera tanda puede traer varias pull requests a la vez (hay un límite por defecto de cinco abiertas por bloque, que se ve en la siguiente ficha).

## Qué aparece en el repositorio

Cada actualización llega como una pull request normal, abierta por el usuario `dependabot[bot]`, con una rama propia y un título predecible:

```
Bump Serilog.AspNetCore from 8.0.3 to 9.0.0 in /backend
```

La rama sigue el patrón `dependabot/<ecosistema>/<carpeta>/<paquete>-<versión>`, y el cuerpo de la pull request incluye lo que haría falta para decidir sin salir de GitHub:

- las **notas de la versión** y el *changelog* del paquete,
- la lista de commits entre las dos versiones,
- una **puntuación de compatibilidad**: el porcentaje de repositorios públicos cuyo CI siguió en verde tras esa misma actualización. Es un indicador, no una garantía: un 98 % no dice nada sobre *tu* código.

Esa pull request dispara tu CI igual que cualquier otra. Si los tests pasan y el cambio es pequeño, se fusiona; si fallan, el fallo te dice exactamente qué rompe esa versión, en un cambio aislado y fácil de revertir.

## Hablar con Dependabot: comandos en comentarios

No hace falta tocar la configuración para controlar una pull request concreta. Dependabot responde a comentarios que empiezan por `@dependabot`:

| Comentario | Efecto |
|---|---|
| `@dependabot rebase` | Rehace la rama sobre la rama destino actual. Útil cuando hay conflictos o la rama quedó vieja. |
| `@dependabot recreate` | Descarta la rama y la crea de nuevo desde cero (también borra tus commits manuales). |
| `@dependabot merge` | Fusiona la pull request cuando los checks requeridos pasan. |
| `@dependabot ignore this major version` | Cierra la pull request y no vuelve a proponer actualizaciones de esa versión mayor. |
| `@dependabot ignore this minor version` | Igual, pero para esa versión menor. |
| `@dependabot ignore this dependency` | Deja de actualizar ese paquete por completo. |

Cada comando se escribe como un comentario en la pull request. Las variantes de `ignore` merecen un uso consciente: son la forma de decir «ahora no» sin dejar la pull request abierta indefinidamente, pero silencian la dependencia hasta que alguien lo revierta.

## Lo que Dependabot no hace

Conviene tenerlo claro para no esperar magia:

- **No fusiona nada por su cuenta.** Abre la pull request; decidir y fusionar es cosa tuya (o de una automatización que configures, como se ve en [Dependabot en CI y fusión automática](Dependabot-en-CI-y-Fusion-Automatica.md)).
- **No arregla cambios incompatibles.** Si la versión nueva cambia una API, la pull request llegará con el CI en rojo y el código por adaptar.
- **Solo ve lo que está declarado.** Una librería copiada a mano en una carpeta `vendor/`, o descargada con un `curl` en un script, es invisible para él.

## Buenas prácticas avanzadas

- **Fusiona poco y a menudo, nunca en lote.** Diez pull requests de patch revisadas en una semana cuestan menos que una migración mayor pospuesta un año. La cadencia semanal es un buen punto de partida porque equilibra ruido y frescura; con la mensual cada tanda llega más grande y más difícil de revisar.
- **Lee las notas de la versión en las mayores, aunque el CI esté verde.** Los tests solo detectan lo que cubren. Un cambio de comportamiento sin romper la compilación (un valor por defecto distinto, un orden de ejecución) pasa el CI y rompe producción.
- **Usa `ignore` con motivo, no el cierre silencioso.** Cuando descartes una actualización, hazlo con el comando o con una regla en `dependabot.yml` (que queda en el control de versiones) y deja escrito el porqué en un comentario. En seis meses nadie recordará que se ignoró React 19 «hasta terminar la migración del router».
- **Incluye siempre `github-actions` en la configuración.** Las acciones son la dependencia que más se olvida y la que se ejecuta con acceso a tus secrets. Si además las fijas por commit SHA (ver [GitHub Actions](../ci-cd/GitHub-Actions.md)), Dependabot es lo que impide que esos SHA queden obsoletos.
- **Trata un `dependabot.yml` ausente como una deuda.** Un repositorio con alertas activadas pero sin *version updates* acumula avisos que solo se resuelven con grandes saltos de versión; el fichero cuesta diez líneas.

## Documentación oficial

- [Documentación de Dependabot](https://docs.github.com/es/code-security/dependabot) — el punto de entrada; la sección de *version updates* explica cada comportamiento, y la de *security updates* cubre el lado de vulnerabilidades.
- [Comandos de Dependabot en comentarios](https://docs.github.com/es/code-security/dependabot/working-with-dependabot/managing-pull-requests-for-dependency-updates) — la lista completa de lo que se puede pedir en una pull request.

## Recursos didácticos

- [GitHub Advisory Database](https://github.com/advisories) — la base de vulnerabilidades que alimenta a Dependabot, navegable por ecosistema: sirve para entender qué se considera una vulnerabilidad y cuánto tarda en aparecer un aviso.

---

*En resumen: Dependabot convierte «actualizar dependencias» de una tarea pendiente y enorme en una pull request pequeña cada semana, pero solo funciona si alguien las revisa y las fusiona.*
