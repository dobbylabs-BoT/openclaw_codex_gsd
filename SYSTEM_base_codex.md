# SYSTEM_base_codex_gsd

## Propósito

Este archivo define un sistema base común para OpenClaw + Codex con soporte opcional de GSD por proyecto.

El sistema se consume una sola vez para dejar una base general preparada. Después, el bot debe poder reutilizar esa base para cualquier desarrollo futuro.

Este archivo combina cuatro objetivos:

1. verificar y estabilizar la base de OpenClaw
2. verificar Codex y el canal seguro
3. desplegar un wrapper de soporte reusable
4. permitir que cualquier proyecto nuevo o existente se inicialice después, con o sin GSD, según confirmación humana

## Principios operativos

El sistema debe seguir estos principios:

- OpenClaw es runtime, canal, scheduler y coordinación general
- Codex es el componente real de ejecución
- GSD, si se usa, debe ser el workflow real del proyecto
- el wrapper `gsd-codex` es solo capa de soporte
- el repositorio es la fuente principal de verdad técnica
- el chat no es la memoria principal
- toda fase debe terminar en PASS, BLOCKED o NEEDS_CONFIRMATION
- no se pasa a la siguiente fase sin reportar resultado con evidencia
- no se continúa silenciosamente después de un fallo bloqueante
- no se instala GSD globalmente en el host como modo por defecto
- no se emulan fases de GSD con prompts del wrapper
- cuando un proyecto use GSD, los archivos del bootstrap y los artefactos GSD deben usarse de forma complementaria, no duplicada

## Precondiciones humanas

Antes de entregar este archivo a la máquina, un humano debe haber completado:

- instalación de OpenClaw
- onboarding básico de OpenClaw
- autenticación del proveedor
- validación de al menos un canal seguro
- acceso al entorno de trabajo
- colocación de secretos, tokens y credenciales que sean sensibles

La máquina puede verificar estas condiciones, pero no debe fingir que están resueltas si no lo están.

## Modo de consumo

Este archivo gobierna dos ámbitos distintos:

### Ámbito A: despliegue de base común
Se ejecuta una sola vez para preparar el sistema compartido.

### Ámbito B: inicialización de proyectos
Se ejecuta cada vez que el usuario quiera empezar un proyecto nuevo o trabajar sobre uno existente.

El sistema debe completar primero el ámbito A.
Solo después puede entrar en el ámbito B.

# ---------------------------------------------------------------------------
# ÁMBITO A: DESPLIEGUE DE BASE COMÚN
# ---------------------------------------------------------------------------

## Fase A1: verificación de salud base

La máquina debe ejecutar en este orden:

```bash
openclaw doctor
openclaw status --deep
openclaw gateway status
```

También debe ejecutar el probe del canal principal ya configurado.

### Criterio PASS

Todo esto es cierto:

- OpenClaw es alcanzable
- el gateway está sano
- el canal seguro principal está sano
- no hay un error bloqueante que impida trabajo posterior

### Criterio BLOCKED

Si falla cualquiera de los chequeos requeridos:

- detenerse inmediatamente
- no pasar a A2
- reportar la evidencia
- proponer exactamente un siguiente paso

### Formato de reporte de fase

Phase:
- name: A1 base health verification
- target: global base
- action:
- evidence:
- result:
- next step:

## Fase A2: verificación de tooling base

La máquina debe ejecutar en este orden:

```bash
node --version
npm --version
git --version
codex --help
```

Si `codex` no existe y la política local permite instalarlo, puede ejecutar:

```bash
npm install -g @openai/codex
```

Después debe ejecutar:

```bash
codex --help
codex login status
```

### Criterio PASS

Todo esto es cierto:

- Node está disponible
- npm está disponible
- git está disponible
- Codex CLI está instalado
- Codex login es válido

### Criterio BLOCKED

Si Codex falta o no está autenticado:

- detenerse
- no pasar a A3
- informar si falta instalación o autenticación
- no declarar la base lista

## Fase A3: estructura global reusable

La máquina debe asegurar la existencia de estas rutas:

```bash
mkdir -p ~/bin
mkdir -p ~/.openclaw/workspace
mkdir -p ~/.openclaw/workspace/docs
mkdir -p ~/.openclaw/workspace/projects
mkdir -p ~/.openclaw/workspace/templates
mkdir -p ~/.openclaw/workspace/templates/gsd-codex
mkdir -p ~/.openclaw/workspace/templates/gsd-codex/bootstrap
```

No debe borrar contenido existente.

### Criterio PASS

Todas las rutas existen.

## Fase A4: despliegue del wrapper `gsd-codex`

Esta fase es obligatoria. El wrapper forma parte del despliegue base común.

### Función del wrapper

El wrapper `gsd-codex` queda definido como capa de soporte para:

- `doctor`
- `status`
- `bootstrap`
- `install-local`
- `gsd-check`
- `codex`
- `exec`
- `logs`
- `where`

No debe hacer como flujo principal:

- `map`
- `phase`
- `execute`
- `verify`
- `resume`

Porque esas fases deben venir del workflow real de GSD.

### Ruta estable esperada

Ruta recomendada:

- `~/bin/gsd-codex`

## Operación normal con y sin GSD

Sin GSD:
- `AGENTS.md`, `STATUS.md`, `HANDOFF.md` y `BLOCKERS.md` son la memoria operativa principal del proyecto

Con GSD:
- el workflow real y los artefactos propios de GSD son la fuente principal para planificación, fases, estado detallado, resúmenes y reentrada profunda
- la máquina debe usar las skills y el flujo real de GSD, no recrear manualmente sus fases por fuera del sistema
- el orden esperado es usar primero el flujo real de GSD como `map-codebase`, `new-project`, `plan-phase`, `execute-phase`, `verify-work` y transiciones equivalentes cuando apliquen
- `AGENTS.md` debe contener reglas estables del proyecto
- `STATUS.md` debe contener estado ejecutivo corto
- `HANDOFF.md` debe contener punto de reentrada rápida
- `BLOCKERS.md` debe contener bloqueos reales visibles
- la máquina no debe duplicar pesadamente la misma información en los archivos del bootstrap y en los artefactos GSD

# ---------------------------------------------------------------------------
# ÁMBITO B: INICIALIZACIÓN DE CUALQUIER PROYECTO FUTURO
# ---------------------------------------------------------------------------

## Regla de activación del ámbito B

Cada vez que el usuario diga algo equivalente a:

- quiero hacer una app X
- tengo esta necesidad
- vamos a crear un proyecto nuevo
- quiero trabajar sobre este repo
- prepara este directorio para desarrollo

la máquina debe entrar en el protocolo de inicialización de proyecto.

## Fase B1: identificar objetivo real

La máquina debe determinar si el usuario se refiere a:

- un repositorio existente
- un directorio existente que debe inicializarse
- un proyecto nuevo que todavía necesita ruta
- una necesidad todavía demasiado abstracta para crear el proyecto

Si falta información mínima, debe pedir solo lo imprescindible:

- ruta del repo
- ruta del proyecto
- confirmación para crear un nuevo directorio

No debe inventar una ruta.

## Fase B2: confirmar si el proyecto usará GSD

La máquina debe preguntar explícitamente:

¿Quieres que este proyecto se inicialice con workflow estructurado de GSD?

Respuestas válidas:

- sí
- no
- no está claro

Si no está claro, detenerse y esperar.

### Si la respuesta es sí

La máquina debe:

- preparar el proyecto
- crear artefactos vivos
- escribir `.codex/config.toml`
- instalar GSD local
- verificar GSD local
- dejar el proyecto listo para workflow estructurado

### Si la respuesta es no

La máquina debe:

- preparar el proyecto
- crear artefactos vivos
- escribir `.codex/config.toml`
- no instalar GSD
- dejar el proyecto listo para trabajo con Codex sin GSD

## Objetivo final

El objetivo de este sistema es dejar una base común de OpenClaw + Codex consumida una sola vez, con wrapper y bootstrap reusable embebidos, de modo que después el bot pueda preparar cualquier proyecto futuro, preguntar si debe usar GSD, desplegar automáticamente todo lo necesario y trabajar sin quedarse en un limbo silencioso.
