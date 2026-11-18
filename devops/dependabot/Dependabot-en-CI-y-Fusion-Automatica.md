# Dependabot en CI y fusión automática

## ¿Qué es?

Es la forma de integrar las pull requests de Dependabot con tu pipeline: entender cómo se comporta el CI cuando las abre un bot, y automatizar la fusión de las actualizaciones seguras para que solo las arriesgadas requieran atención humana.

## ¿Por qué existe?

Una vez Dependabot funciona, aparece un problema nuevo: la cantidad. Un proyecto con una cadencia semanal recibe un goteo constante de pull requests, y revisar y pulsar «Merge» en cada actualización de parche de una librería de formato de fechas es un trabajo que consume tiempo sin aportar criterio.

La solución de fondo es dejar que la máquina fusione lo que tus pruebas ya garantizan, y reservar a las personas lo que requiere juicio. Pero hay dos trampas: los workflows lanzados por Dependabot **no tienen los mismos permisos** que los normales, y automatizar la fusión sin un CI sólido es una forma elegante de romper producción sin enterarte.

## ¿Cuándo y para qué se usa?

Cuando el repositorio tiene un CI en el que se confía (tests que fallan de verdad cuando algo se rompe) y el volumen de pull requests de Dependabot empieza a ser una carga. Si el CI es débil, el primer paso es mejorarlo, no automatizar la fusión.

---

## Lo que cambia cuando abre la pull request un bot

Los workflows disparados por `pull_request` desde una rama de Dependabot se ejecutan con dos restricciones respecto a los de una persona:

- El `GITHUB_TOKEN` es de **solo lectura** por defecto.
- **No tienen acceso a los secrets de Actions.** Solo ven los secrets del almacén de Dependabot.

El motivo es de seguridad: un paquete malicioso que llegase en una actualización no debería poder leer tus secrets durante la ejecución del CI. La consecuencia práctica es un fallo muy reconocible: un workflow que funciona con todas las demás pull requests falla solo con las de Dependabot.

```
Error: Input required and not supplied: password
```

Este es el mensaje típico de una acción de login (`docker/login-action`, por ejemplo) a la que le llega vacío un secret que en realidad existe. La solución es dar a Dependabot su propia copia de ese secret en *Settings → Secrets and variables → Dependabot* (ver [Secrets, variables y entornos](../github-organizaciones/Secrets-variables-y-entornos.md)). Y la pregunta que conviene hacerse antes de copiarlo es si ese paso necesita de verdad correr en una pull request de actualización: un `docker login` para *publicar* una imagen solo debería ejecutarse al fusionar, no al validar.

## Fusión automática, paso a paso

La idea: un workflow identifica la pull request de Dependabot, mira qué tipo de actualización es, y si es de bajo riesgo activa el *auto-merge*. GitHub fusionará entonces la pull request **cuando todos los checks requeridos pasen**; no antes.

Para que funcione, hacen falta dos cosas previas:

1. Activar *Allow auto-merge* en *Settings → General* del repositorio.
2. Tener **checks requeridos** en la protección de la rama (ver [Rulesets y protección de ramas](../github-organizaciones/Rulesets-y-proteccion-de-ramas.md)). Sin ellos, el auto-merge fusiona de inmediato, sin esperar a ningún test.

Con eso, el workflow:

```yaml
# .github/workflows/dependabot-auto-merge.yml
name: Dependabot auto-merge

on: pull_request

permissions:
  contents: write
  pull-requests: write

jobs:
  auto-merge:
    runs-on: ubuntu-latest
    if: github.event.pull_request.user.login == 'dependabot[bot]'
    steps:
      - name: Leer metadatos de la actualización
        id: metadata
        uses: dependabot/fetch-metadata@v2
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Activar auto-merge si no es una versión mayor
        if: steps.metadata.outputs.update-type != 'version-update:semver-major'
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Qué hace cada pieza:

- `permissions` da al token permiso de escritura **solo en este workflow**, que es lo que necesita `gh pr merge`.
- La condición `if` comprueba que quien abrió la pull request es Dependabot. Se usa `pull_request.user.login` y no `github.actor` porque este último cambia si otra persona relanza el workflow, y la comprobación dejaría de ser fiable.
- `dependabot/fetch-metadata` es la acción oficial que lee la pull request y expone datos como `update-type` (`version-update:semver-patch`, `semver-minor` o `semver-major`), `dependency-type` (`direct:production`, `direct:development`, `indirect`) y `dependency-names`.
- `gh pr merge --auto` no fusiona en ese momento: deja la pull request «armada» para fusionarse cuando pasen los checks requeridos.

> Para abreviar, el ejemplo usa la etiqueta `@v2`. En un repositorio real conviene fijar la acción por SHA de commit, como se explica en [GitHub Actions](../ci-cd/GitHub-Actions.md), y dejar que el ecosistema `github-actions` de Dependabot lo mantenga al día.

## Qué se fusiona solo y qué no

La condición del ejemplo (todo salvo las mayores) es el punto de partida más habitual, pero conviene afinarla según el tipo de dependencia:

| Tipo de actualización | Decisión recomendada | Por qué |
|---|---|---|
| Parche de una dependencia de desarrollo (linters, herramientas de test) | Automática | No llega a producción y los tests lo validan. |
| Parche o menor de una dependencia de producción | Automática si el CI la cubre bien | Compatibilidad semántica, pero el cambio de comportamiento sutil sigue siendo posible. |
| Versión mayor | Siempre manual | Por definición, puede romper la API. |
| Acciones de GitHub | Manual, aunque sea parche | Se ejecutan con acceso a secrets y permisos del repositorio. |
| Imagen base de Docker | Manual | Un cambio de sistema operativo o de runtime no lo detectan los tests unitarios. |

En el workflow, esto se traduce en condiciones sobre las salidas de `fetch-metadata`:

```yaml
      - name: Auto-merge solo de herramientas de desarrollo y parches
        if: >-
          steps.metadata.outputs.dependency-type == 'direct:development' ||
          steps.metadata.outputs.update-type == 'version-update:semver-patch'
        run: gh pr merge --auto --squash "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## El peligro de `pull_request_target`

Al toparse con el problema de los secrets, mucha gente encuentra una «solución» en internet: cambiar el disparador a `pull_request_target`, que sí tiene acceso a secrets y permisos de escritura. Funciona, y es una trampa: ese evento se ejecuta en el contexto de la rama base, con todos los permisos, y si el workflow además descarga y ejecuta el código de la pull request (con `actions/checkout` sobre `github.event.pull_request.head.sha`, por ejemplo), cualquier código malicioso dentro de la pull request se ejecuta con acceso a tus secrets.

Con Dependabot es especialmente grave porque el contenido de la pull request viene de paquetes de terceros. Si necesitas que un workflow de Dependabot acceda a un secret, usa el almacén propio de Dependabot; no cambies a `pull_request_target`.

## Un flujo semanal razonable

La automatización no sustituye al criterio, lo concentra. Un ritmo que funciona en equipos pequeños:

1. La automatización fusiona sola los parches de bajo riesgo durante la semana.
2. Una vez por semana, quince minutos para revisar lo que quedó abierto: menores de producción, acciones, imágenes y mayores.
3. Para cada una, tres salidas posibles: fusionar, `@dependabot ignore ...` con motivo, o abrir una tarea si exige una migración.
4. Cualquier pull request abierta con más de un mes de antigüedad es una señal: o el CI está roto, o la actualización se ha convertido en deuda y hay que decidirlo.

## Errores frecuentes

| Síntoma | Causa y arreglo |
|---|---|
| El auto-merge fusiona de inmediato, antes de que acaben los tests | No hay checks requeridos en la protección de la rama. Añádelos. |
| `gh pr merge --auto` falla con «auto-merge is not allowed» | Falta activar *Allow auto-merge* en los ajustes del repositorio. |
| Un workflow funciona en todas las pull requests menos en las de Dependabot | El secret existe en Actions pero no en el almacén de Dependabot, o el token es de solo lectura. |
| La pull request se queda esperando «Expected — Waiting for status» para siempre | Un check requerido nunca se ejecuta porque los filtros de ruta del workflow excluyen los ficheros que toca Dependabot (ver [Monorepo y path-filters](../ci-cd/Monorepo-y-Path-Filters.md)). |
| Las pull requests acumulan conflictos | Comenta `@dependabot rebase`, o fusiona más a menudo para que no se acumulen. |

## Buenas prácticas avanzadas

- **Empieza solo con el auto-merge de lo más seguro y amplía con datos.** Parches de herramientas de desarrollo primero; a las pocas semanas, mira cuántas veces el CI salvó una pull request mala, y solo entonces extiende el alcance. Subir el listón con evidencia es más defendible que adivinarlo.
- **Una regla de fusión automática vale lo que valga tu cobertura.** Si los tests no ejercitan la ruta que usa esa librería, el verde no significa nada. Usa las mayores y las bibliotecas críticas (autenticación, pagos, serialización) como lista de exclusión permanente, aunque el CI pase.
- **Da permisos mínimos al workflow de fusión.** Declara `permissions` explícitos en el propio fichero (`contents: write` y `pull-requests: write`) y nada más. Un workflow que maneja pull requests de terceros no debería tener el token por defecto de la organización.
- **Etiqueta y filtra.** Con `labels` en el `dependabot.yml`, puedes buscar «todas las pull requests de dependencias abiertas hace más de 30 días» con un solo filtro, y es la métrica más barata para medir si el sistema funciona o se está pudriendo.
- **Revisa el *auto-merge* tras cambiar el CI.** Si renombras un job o cambias los checks requeridos, el auto-merge puede quedarse esperando uno que ya no existe, o empezar a fusionar sin esperar el que sí.

## Documentación oficial

- [Automatizar Dependabot con GitHub Actions](https://docs.github.com/es/code-security/dependabot/working-with-dependabot/automating-dependabot-with-github-actions) — los ejemplos oficiales de etiquetado y auto-merge, y las consideraciones de seguridad sobre `pull_request_target`.
- [dependabot/fetch-metadata](https://github.com/dependabot/fetch-metadata) — todas las salidas disponibles de la acción y casos de uso adicionales.
- [Solucionar problemas de Dependabot en Actions](https://docs.github.com/es/code-security/dependabot/troubleshooting-dependabot/troubleshooting-dependabot-on-github-actions) — los fallos de permisos y secrets que aparecen solo con pull requests de Dependabot.

---

*En resumen: deja que la máquina fusione lo que tus tests ya garantizan y reserva tu atención para lo que no, sin saltarte nunca los checks requeridos.*
