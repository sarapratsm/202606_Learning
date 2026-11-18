# Dependabot — Guía de tecnologías

Cómo mantener al día las dependencias de un repositorio de GitHub sin dedicarle tardes enteras: qué hace Dependabot, cómo se configura para que no ahogue al equipo en pull requests, cómo reacciona ante una vulnerabilidad y cómo automatizar la fusión de lo que es seguro.

Está pensada para perfiles backend que ya trabajan con GitHub y con algún pipeline de CI. No presupone experiencia previa con Dependabot: cada ficha parte de la intro y avanza con ejemplos de configuración y de workflows.

Todas las fichas usan el mismo escenario: una tienda online con un backend en .NET (`/backend`) y un frontend en React (`/frontend`) en un único repositorio.

---

## Orden de lectura recomendado

Cada ficha se apoya en la anterior.

### 1. Entender y poner en marcha

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 1 | [Dependabot](Dependabot.md) | Las tres piezas que se confunden (alertas, security updates y version updates), el primer `dependabot.yml` y cómo se ve una pull request de Dependabot. |
| 2 | [Configuración de dependabot.yml](Configuracion-de-dependabot-yml.md) | Agrupar, ignorar, esperar con `cooldown` y diagnosticar por qué no abre nada. Aquí se pasa del ruido a un flujo llevadero. |

### 2. Seguridad y automatización

| # | Archivo | Por qué leerlo aquí |
|---|---|---|
| 3 | [Alertas y actualizaciones de seguridad](Alertas-y-Actualizaciones-de-Seguridad.md) | Cómo sabe GitHub que una dependencia es vulnerable, qué hacer con una alerta y los casos en que la corrección no es una simple subida de versión. |
| 4 | [Dependabot en CI y fusión automática](Dependabot-en-CI-y-Fusion-Automatica.md) | Por qué el CI se comporta distinto con pull requests de un bot y cómo fusionar automáticamente solo lo de bajo riesgo. |

---

## Índice completo

<details>
<summary>Ver todos los archivos</summary>

- [Dependabot](Dependabot.md)
- [Configuración de dependabot.yml](Configuracion-de-dependabot-yml.md)
- [Alertas y actualizaciones de seguridad](Alertas-y-Actualizaciones-de-Seguridad.md)
- [Dependabot en CI y fusión automática](Dependabot-en-CI-y-Fusion-Automatica.md)

</details>

---

## Qué ficha consultar según la duda

| Duda | Ficha |
|---|---|
| ¿Qué diferencia hay entre alertas, security updates y version updates? | [Dependabot](Dependabot.md) |
| ¿Por qué Dependabot no me abre ninguna pull request? | [Configuración de dependabot.yml](Configuracion-de-dependabot-yml.md) |
| ¿Cómo evito quince pull requests cada lunes? | [Configuración de dependabot.yml](Configuracion-de-dependabot-yml.md) |
| Ha salido una alerta crítica y no hay pull request | [Alertas y actualizaciones de seguridad](Alertas-y-Actualizaciones-de-Seguridad.md) |
| Mi workflow falla solo en pull requests de Dependabot | [Dependabot en CI y fusión automática](Dependabot-en-CI-y-Fusion-Automatica.md) |
| ¿Cómo fusiono automáticamente los parches? | [Dependabot en CI y fusión automática](Dependabot-en-CI-y-Fusion-Automatica.md) |

---

> Piezas relacionadas en otras colecciones: [GitHub Actions](../ci-cd/GitHub-Actions.md) para escribir el workflow de fusión y fijar acciones por SHA, [Seguridad de la organización](../github-organizaciones/Seguridad-de-la-organizacion.md) para activar alertas en toda la organización, y [Rulesets y protección de ramas](../github-organizaciones/Rulesets-y-proteccion-de-ramas.md) para exigir los checks que hacen seguro el auto-merge.
