# Sistema base compartible para OpenClaw + Codex

Este repositorio contiene una base reutilizable para preparar un asistente técnico persistente que pueda:

- trabajar con OpenClaw como runtime
- usar Codex como componente de ejecución
- preparar proyectos nuevos o existentes
- ofrecer workflow estructurado con GSD cuando se confirme
- desplegar y verificar cada fase con evidencia
- mantener continuidad del trabajo en artefactos del proyecto

Este sistema no está ligado a un único proyecto.
Se consume una vez para dejar preparada la base general.
Después, el bot puede reutilizar esa base para cualquier nuevo desarrollo.

## Archivo que debe consumir la máquina

La máquina debe consumir:

- `SYSTEM_base_codex.md`

Ese archivo define:

- verificación de salud del sistema
- despliegue de la base común
- despliegue del wrapper de soporte
- creación de estructura global
- protocolo de inicialización de proyecto
- confirmación de uso de GSD por proyecto
- artefactos vivos del proyecto
- reglas de comunicación con evidencia

## Qué deja preparado este sistema

Después de consumir el sistema base, el bot debe quedar listo para:

- verificar OpenClaw y el canal seguro
- verificar Codex
- desplegar el wrapper de soporte
- preparar estructura global de trabajo
- inicializar proyectos nuevos o existentes
- instalar GSD local solo si se confirma para ese proyecto
- crear y mantener:
  - `AGENTS.md`
  - `STATUS.md`
  - `HANDOFF.md`
  - `BLOCKERS.md`

## Qué hace el humano

El humano se ocupa de:

- instalar OpenClaw
- completar onboarding
- dejar al menos un canal seguro funcionando
- resolver secretos y autenticación sensible
- indicar el proyecto objetivo
- decidir si un proyecto concreto debe usar GSD

## Qué hace la máquina

La máquina se ocupa de:

- verificar salud
- preparar la base común
- desplegar wrapper y estructura global
- inicializar proyectos
- instalar herramientas locales por proyecto
- generar artefactos vivos
- validar cada fase
- reportar siempre con evidencia breve

## Resultado esperado

Después del despliegue base, deberías poder decir cosas como:

- quiero hacer una app X
- tengo esta necesidad
- quiero arrancar un proyecto nuevo en esta ruta
- quiero trabajar sobre este repo

Y el bot debería:

1. detectar el objetivo
2. preguntar lo mínimo que falte
3. confirmar si quieres usar GSD
4. desplegar automáticamente lo necesario
5. validar
6. reportar resultado o bloqueo con evidencia

## Regla operativa de continuidad

Cuando un proyecto use GSD, la máquina debe usar adecuadamente ambos niveles de seguimiento:

- archivos ligeros del bootstrap para continuidad operativa rápida:
  - `AGENTS.md`
  - `STATUS.md`
  - `HANDOFF.md`
  - `BLOCKERS.md`
- artefactos propios de GSD para workflow, planificación, memoria y reentrada profunda

La máquina no debe duplicar de forma pesada la misma información en ambos sistemas.

Uso recomendado:
- `AGENTS.md` para reglas estables del proyecto
- `STATUS.md` para estado ejecutivo corto
- `HANDOFF.md` para reentrada rápida
- `BLOCKERS.md` para bloqueos reales visibles
- artefactos GSD como fuente principal de planificación, fases, estado detallado, resúmenes y threads cuando GSD esté activo
