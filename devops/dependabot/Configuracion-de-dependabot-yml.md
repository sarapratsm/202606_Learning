# Configuración de dependabot.yml

## ¿Qué es?

`dependabot.yml` es el fichero que le dice a Dependabot **qué** dependencias vigilar, **dónde** están, **cada cuánto** revisarlas y **cómo** quieres recibir las actualizaciones.

## ¿Por qué existe?

Con la configuración mínima de la [ficha anterior](Dependabot.md), Dependabot funciona, pero se comporta de forma ingenua: abre una pull request por cada paquete, sin distinguir un parche de una versión mayor, y se lanza sobre cualquier versión recién publicada. En un proyecto con 80 dependencias eso es una avalancha de pull requests pequeñas que acaba ignorándose, que es peor que no tener Dependabot.

El fichero existe para que la herramienta se ajuste al proyecto y no al revés: agrupar lo que se revisa junto, silenciar lo que no toca todavía y limitar el ritmo.

## ¿Cuándo y para qué se usa?

Se retoca en tres momentos típicos: cuando aparece el primer lunes con quince pull requests, cuando una actualización mayor se sabe que no se puede aceptar aún (por ejemplo, un framework que exige una migración planificada) y cuando el repositorio crece y pasa a tener varias carpetas con sus propias dependencias (un monorepo).

---

## Un bloque por ecosistema y carpeta

Cada entrada de `updates` es independiente: tiene su calendario, sus agrupaciones y sus límites. Para un proyecto con varios ecosistemas, la regla es repetir el bloque, no intentar meterlo todo en uno:

```yaml
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

  - package-ecosystem: "docker"
    directory: "/backend"          # donde está el Dockerfile
    schedule:
      interval: "monthly"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"
```

Aquí las dependencias de código se revisan cada semana, mientras que la imagen base y las acciones, que cambian menos, cada mes. Ajustar la frecuencia por tipo de dependencia es la primera palanca contra el ruido.

Cuando hay varias carpetas con el mismo ecosistema (un monorepo con un servicio por carpeta), `directories` admite una lista y comodines, y evita repetir el bloque:

```yaml
  - package-ecosystem: "npm"
    directories:
      - "/services/*"
      - "/frontend"
    schedule:
      interval: "weekly"
```

## Cuándo se lanza: `schedule`

`interval` admite `daily`, `weekly` y `monthly` (existen además intervalos más espaciados; consulta la referencia oficial). Con `weekly` puedes elegir el día y la hora, algo útil para que las pull requests lleguen al principio de la jornada y no un domingo:

```yaml
    schedule:
      interval: "weekly"
      day: "monday"
      time: "08:00"
      timezone: "Europe/Madrid"
```

Sin `timezone`, la hora se interpreta en UTC, y de ahí vienen la mayoría de «¿por qué ha llegado a las 10 en lugar de a las 8?».

## Agrupar: de quince pull requests a una

`groups` junta varias actualizaciones en **una sola pull request**. Es la herramienta más eficaz contra el ruido:

```yaml
  - package-ecosystem: "npm"
    directory: "/frontend"
    schedule:
      interval: "weekly"
    groups:
      react-ecosystem:
        patterns:
          - "react*"
          - "@types/react*"
      minor-and-patch:
        update-types:
          - "minor"
          - "patch"
```

El primer grupo junta los paquetes cuyo nombre empieza por `react`, que casi siempre tienen que actualizarse a la vez. El segundo agrupa **todas** las actualizaciones menores y de parche que no hayan caído ya en el primero; las versiones mayores quedan fuera y siguen llegando por separado, cada una con su pull request, que es lo que quieres: las mayores merecen atención individual.

Un grupo cuenta como **una** pull request abierta a efectos del límite, y cada grupo se rehace como una unidad: si una de las actualizaciones del grupo rompe el CI, hay que ver cuál es entre todas. Por eso conviene agrupar lo que es de bajo riesgo (menores y parches) y no mezclar versiones mayores.

## Silenciar lo que no toca: `ignore`

Hay actualizaciones que se sabe que no se pueden aceptar todavía. En lugar de cerrar la pull request cada semana, se declara en el fichero:

```yaml
    ignore:
      - dependency-name: "react"
        update-types: ["version-update:semver-major"]
      - dependency-name: "Microsoft.EntityFrameworkCore*"
        versions: ["9.x"]
```

La primera regla ignora las versiones **mayores** de `react` pero sigue proponiendo menores y parches. La segunda bloquea una línea de versiones concreta de un conjunto de paquetes (el `*` funciona como comodín). Al estar en el repositorio, la decisión queda versionada, se revisa en pull request y alguien puede encontrarla seis meses después y preguntarse si ya es el momento de quitarla.

## Esperar antes de adoptar: `cooldown`

Una versión recién publicada es la que más riesgo tiene: puede traer un fallo que se corrige a las pocas horas, o incluso haber sido publicada por una cuenta comprometida (un ataque a la cadena de suministro) y retirada al día siguiente. `cooldown` hace que Dependabot espere unos días tras la publicación antes de proponerla:

```yaml
    cooldown:
      default-days: 5
      semver-major-days: 14
```

Aquí las versiones menores y de parche se proponen cinco días después de publicarse, y las mayores dos semanas. El *cooldown* no retrasa las actualizaciones de seguridad: esas siguen su propio camino (ver [Alertas y actualizaciones de seguridad](Alertas-y-Actualizaciones-de-Seguridad.md)).

## Controlar el ritmo y el aspecto

Algunas opciones sueltas, de menor peso pero que se tocan a menudo:

| Opción | Para qué sirve | Ejemplo |
|---|---|---|
| `open-pull-requests-limit` | Máximo de pull requests de *version updates* abiertas a la vez para ese bloque. Por defecto, 5. | `10` |
| `labels` | Etiquetas que se añaden a las pull requests, útiles para filtrar o para disparar reglas. | `["dependencias"]` |
| `commit-message.prefix` | Prefijo del mensaje de commit, para respetar convenciones como Conventional Commits. | `"chore(deps)"` |
| `target-branch` | Rama sobre la que se abren las pull requests, si no es la rama por defecto. | `"develop"` |

```yaml
    open-pull-requests-limit: 10
    labels: ["dependencias"]
    commit-message:
      prefix: "chore(deps)"
```

Para asignar quién revisa, el camino recomendado es un fichero `CODEOWNERS`: las pull requests de Dependabot piden revisión al responsable de esos ficheros igual que cualquier otra.

## Registros privados de paquetes

Si el proyecto usa paquetes de un feed privado (un NuGet interno, un registro npm de la empresa), Dependabot necesita credenciales para consultarlo. Se declara el registro y se referencia desde el bloque:

```yaml
registries:
  feed-interno:
    type: nuget-feed
    url: https://nuget.ejemplo.com/v3/index.json
    token: ${{ secrets.DEPENDABOT_NUGET_TOKEN }}

updates:
  - package-ecosystem: "nuget"
    directory: "/backend"
    registries:
      - feed-interno
    schedule:
      interval: "weekly"
```

El secret **no** se lee de los secrets de Actions: Dependabot tiene su propio almacén (*Settings → Secrets and variables → Dependabot*). Poner el token solo en Actions es el motivo habitual de que el registro privado «no funcione».

## Cuando no abre nada: cómo diagnosticar

Dependabot no avisa por correo de que algo va mal; hay que mirar. En el repositorio, *Insights → Dependency graph → Dependabot* lista cada bloque del fichero, cuándo se revisó por última vez y el registro del último trabajo, con el error si lo hubo. Es el primer sitio al que ir.

Las causas más habituales, de más a menos frecuente:

| Síntoma | Causa probable |
|---|---|
| No aparece ninguna pull request nueva | Todo está al día, o ya se alcanzó `open-pull-requests-limit`. Mira el registro del trabajo. |
| La pestaña muestra un error de configuración | YAML mal indentado o clave desconocida. Corrige el fichero; se vuelve a evaluar al hacer commit. |
| «No dependency files found» o similar | `directory` apunta a una carpeta sin fichero de dependencias (ruta mal escrita o con barra final). |
| Un paquete nunca se actualiza | Hay una regla `ignore` que lo cubre, o una restricción de versión en el propio manifiesto lo impide. |
| Error al acceder a un feed | Falta el secret en el almacén de Dependabot (no en el de Actions) o ha caducado el token. |

Tras corregir el fichero, desde esa misma pestaña se puede forzar una revisión inmediata con *Check for updates*, sin esperar al calendario.

## Buenas prácticas avanzadas

- **Agrupa por riesgo, no por comodidad.** Menores y parches juntos en un grupo; versiones mayores siempre aparte. Un grupo que mezcla una mayor con diez parches vuelve ilegible la causa cuando el CI falla.
- **Combina `cooldown` con agrupación.** Es la pareja que más ruido y más riesgo recorta: el *cooldown* evita adoptar versiones recién publicadas y los grupos reducen el número de pull requests; la mayoría de los incidentes de cadena de suministro en paquetes se detectan y retiran en cuestión de días.
- **Tus reglas `ignore` caducan.** Cada una nace con una razón temporal («hasta migrar el router»). Añade el motivo en un comentario YAML (`# TODO: quitar al terminar la migración a React Router 7`) y revisa la lista cada pocos meses; un `ignore` olvidado es una dependencia congelada en secreto.
- **Un solo `dependabot.yml`, con nombres coherentes en `labels` y `prefix`.** Si luego automatizas con workflows (ver [Dependabot en CI y fusión automática](Dependabot-en-CI-y-Fusion-Automatica.md)), esas etiquetas y prefijos son lo que permite filtrar y decidir sin parsear títulos.
- **Fija las acciones por SHA y deja que Dependabot las mueva.** Con `package-ecosystem: "github-actions"`, las pull requests actualizan el SHA y el comentario de versión a la vez; es lo que permite fijar sin quedarse atrás.

## Documentación oficial

- [Referencia de opciones de dependabot.yml](https://docs.github.com/es/code-security/dependabot/working-with-dependabot/dependabot-options-reference) — la lista completa y vigente de claves; es la que hay que consultar para `cooldown`, intervalos adicionales y particularidades de cada ecosistema.
- [Ecosistemas de paquetes admitidos](https://docs.github.com/es/code-security/dependabot/ecosystems-supported-by-dependabot/supported-ecosystems-and-repositories) — para saber si tu gestor de paquetes está soportado y con qué requisitos (por ejemplo, ficheros de bloqueo).

---

*En resumen: el `dependabot.yml` por defecto avisa de todo; el bueno agrupa lo seguro, silencia lo que no toca y espera unos días antes de fiarse de una versión recién salida.*
