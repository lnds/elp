---
title: "ELP — Empirical Lane Parallelism"
subtitle: "A method for human-orchestrated parallel multi-agent software engineering"
author: "Eduardo Díaz Cortés"
date: "2026-05-05"
version: "0.1"
---

# ELP — Empirical Lane Parallelism

> *Un método para construir software con múltiples agentes IA trabajando en paralelo, bajo orquestación humana, con captura empírica del proceso.*

---

> *"Welcome back, my friends, to the show that never ends."*
> — Emerson, Lake & Palmer, *Karn Evil 9: First Impression Part 2*, 1973.

---

## Resumen

**ELP** (*Empirical Lane Parallelism*) es un método de ingeniería de software que combina tres ideas:

1. **Lanes paralelas:** cada unidad de trabajo se ejecuta como una sesión aislada (un *git worktree*, una rama propia, una sesión `tmux` propia). Múltiples lanes corren en paralelo.
2. **Brief como contrato:** cada lane recibe un documento estructurado que define su scope, decisiones ya tomadas, qué hacer y qué no, y qué entregar.
3. **Captura empírica del proceso:** auditorías previas al código, retrospectivas obligatorias por lane y mediciones de cada build se conservan como dataset que alimenta a las siguientes lanes.

ELP no reemplaza a los métodos clásicos de desarrollo, los complementa con un patrón explícito para coordinar a múltiples agentes sin que el sistema se degrade en semanas.

---

## 1. Motivación

Trabajar con asistentes IA en una sola conversación funciona bien para tareas pequeñas. A medida que el proyecto crece, el patrón se rompe:

- El **contexto se llena**: el agente pierde detalles relevantes que estaban al inicio.
- Los **cambios se pisan** entre sí cuando varias tareas se intercalan.
- **No queda registro** de lo que se intentó y por qué se descartó.
- **Cada conversación empieza desde cero**, sin acceso al aprendizaje previo.

Los métodos del espectro actual (vibe coding, Pantser, SDD, SPDD, BMAD) no ofrecen una respuesta operativa al problema cuando se combina con escala (decenas de PRs por día) y duración (proyecto de meses). ELP apunta a llenar ese hueco.

---

## 2. Definiciones

**Lane.** Unidad de trabajo paralelizable, vinculada a un único issue del backlog. Cada lane se materializa en un *git worktree* con su rama, su brief y su sesión `tmux`. El agente que ejecuta una lane corre en **modo autónomo** (en Claude: *auto mode on*): ejecuta el brief de principio a fin sin pausas para confirmación, salvo en los puntos de STOP-and-report explícitamente declarados.

**Brief.** Documento estructurado que define el contrato de la lane. Es el input del agente que ejecuta la lane.

**Integrador.** Agente IA con sesión interactiva que mantiene el contexto compartido con el humano, lee el state del repo, escribe los briefs, lanza las lanes y monitorea su progreso. No toca código directamente.

**Arquitecto.** El humano. Decide scope, sequencing, mergea PRs, autoriza acciones irreversibles, corrige cuando el integrador o las lanes se equivocan.

**Audit (Phase 0).** Lane *doc-only* que valida la premisa de un milestone con datos antes de escribir código. Puede descartar la propuesta entera.

**Lane retro.** Documento que cada lane escribe **antes** de abrir el PR, con métricas, errores encontrados, ambigüedades de spec y *friction points*.

**TSV de builds.** Archivo *tab-separated* donde cada invocación a build/test queda registrada con timestamp, comando, outcome y elapsed.

**STOP-and-report.** Comportamiento del agente lane cuando encuentra una situación fuera de su scope: se detiene y reporta, no improvisa.

---

## 3. Roles

ELP define tres roles con responsabilidades claras y no superpuestas.

### 3.1 Arquitecto (humano)

Responsabilidades:

- Decidir scope de cada milestone y sequencing entre lanes.
- Aprobar o rechazar PRs (o habilitar auto-merge bajo condiciones).
- Autorizar explícitamente acciones irreversibles (force-push, cleanup destructivo, push a *main*).
- Corregir cuando el integrador o las lanes se equivocan.
- Tomar la decisión final cuando hay desacuerdo.

Lo que **no** hace típicamente:

- Escribir código línea por línea.
- Re-discutir decisiones ya pinneadas en briefs.
- Aprobar PRs sin que la retro de la lane esté escrita.

### 3.2 Integrador (agente IA, sesión interactiva)

Responsabilidades:

- Mantener el contexto compartido del proyecto, leyendo state del repo (issues, PRs, retros previas, audits).
- Escribir briefs detallados antes de lanzar cada lane.
- Lanzar agentes lane (en sesiones `tmux` separadas con su worktree).
- Monitorear progreso vía `tmux capture-pane` y *wakeup scheduling*.
- Reportar hallazgos críticos al humano.

Lo que **no** hace:

- Tocar código de la lane directamente.
- Tomar decisiones irreversibles sin autorización del humano.
- Cleanup de worktrees o sesiones sin autorización explícita.

### 3.3 Lane (agente IA, sesión aislada, modo autónomo)

El agente lane corre en **modo autónomo de principio a fin**. En el caso de Claude, esto corresponde a operar con *auto mode on*: el agente no pide confirmación interactiva entre pasos del brief, ejecuta toda la cadena (explorar, diagnosticar, implementar, validar, retro, commit, push, PR) y solo se detiene en los puntos de STOP-and-report declarados explícitamente en el brief, o cuando una acción irreversible requiere autorización (ver §8).

La autonomía es lo que hace viable la ejecución paralela: un humano no puede babysittear N lanes a la vez. El brief reemplaza al *babysitting* — por eso su calidad es el techo de la calidad de la lane.

Responsabilidades:

- Recibir un brief al arrancar, leerlo completo antes de actuar.
- Trabajar en su worktree aislado (rama propia).
- Ejecutar el flujo: *explorar → diagnosticar → implementar → validar → retro → commit → push → abrir PR*, sin pausas para confirmación.
- Habilitar auto-merge cuando CI verde es esperable.
- Hacer **STOP-and-report** ante ambigüedades de diseño que excedan su brief.

Lo que **no** hace:

- Improvisar decisiones de scope.
- Tocar archivos fuera del scope declarado en el brief.
- Cleanup de su propio worktree o sesión.

---

## 4. Anatomía de una lane

### 4.1 Estructura de archivos

Cada lane vive en tres ubicaciones:

```
/tmp/wt-claude-prompt-<issue>-<descr>.txt   # el brief
/tmp/launch-<issue>.sh                      # wrapper que invoca al agente
~/work/<proyecto>.<issue>-<descr>/          # el worktree con su rama
```

Una sesión `tmux` con nombre `<issue>-<descr>` ejecuta el agente.

### 4.2 Lanzamiento

```sh
chmod +x /tmp/launch-NNN.sh
tmux new-session -d -s issue-NNN-descr \
    "wt switch --create issue-NNN-descr -x /tmp/launch-NNN.sh"
```

`wt switch --create` crea worktree + rama, hace `cd` y ejecuta el wrapper. El wrapper hace `exec claude "$(cat brief)"` y arranca el agente con el brief como input inicial.

### 4.3 Monitoreo

```sh
tmux list-sessions
tmux capture-pane -t <session-name> -p | tail -30
```

Si se necesita corregir trayectoria, se usa `tmux send-keys` para preservar el contexto que el agente ya tiene. **Re-briefear desde cero pierde el aprendizaje acumulado en la sesión.**

### 4.4 Cleanup post-merge

Solo después de que el PR de la lane mergeó:

```sh
tmux kill-session -t issue-NNN-descr
git worktree remove ~/work/<proyecto>.issue-NNN-descr
git branch -D issue-NNN-descr
git pull --ff-only origin main
```

**Regla pinneada:** el cleanup necesita autorización explícita del arquitecto. Cleanup prematuro ya destruyó contexto útil en la práctica.

---

## 5. El brief como contrato

El brief es la diferencia entre un agente que entrega un PR *review-ready* y uno que pierde tiempo. Estructura canónica:

### 5.1 Goal

Una a tres frases. Qué se entrega y por qué importa ahora. Idealmente cita el número de issue.

### 5.2 Context

- Repo + branch (siempre `issue-NNN-descr` desde el `main` actual).
- Verificación esperada (ej.: `git log --oneline -5 origin/main`).
- Reglas pinneadas del proyecto (`CLAUDE.md`, archivos a no tocar).
- Lanes paralelas en flight, para anticipar conflictos.

### 5.3 Read first

Lista de archivos que el agente debe leer antes de escribir código:

- El issue completo.
- El audit o design doc relevante, cuando exista.
- Retros de lanes anteriores que comparten contexto.
- Archivos específicos del código que la lane va a tocar.

### 5.4 Decisions pinned

Decisiones ya tomadas que el agente **no debe re-discutir**. Esta sección es crítica: la IA tiende a "mejorar" lo que ya estaba decidido. Ejemplos típicos:

- *"No fix issue #219 en este lane."*
- *"No tocar `CHANGELOG.md` ni `VERSION`, los regenera el bump al release."*
- *"Smallest-delta wins: cambia lo mínimo para que pase."*
- *"Diagnose before implementing: primer paso es entender la falla."*

### 5.5 Lane scope (in / out)

Lista explícita de qué SÍ y qué NO hacer.

`DO NOT` es tan importante como `DO`. El modelo está entrenado para ser útil; si no se le dice que pare, sigue. Casos típicos:

- No tocar otros módulos.
- No tocar archivos de versionamiento (`CHANGELOG`, `VERSION`).
- No abrir issues nuevos para gaps adyacentes (separar scope).
- STOP-and-report ante condiciones X.

### 5.6 Required outputs

- Archivos modificados esperados.
- Mensaje de commit en el formato del proyecto (típicamente *Conventional Commits*).
- Body del PR con qué incluir.
- Comandos de cierre: `gh pr create`, `gh pr merge --auto`.

### 5.7 Discipline reminders

- Tier de tests requeridos antes de push.
- Working tree limpio en cada commit.
- Auto-merge habilitado si corresponde.

### 5.8 Instrumentation (obligatorio)

- Logging de builds en TSV.
- Retro doc en `docs/lane-experience-<lane>.md` con secciones fijas (ver §6.2).

---

## 6. Captura empírica

Los métodos formales del espectro actual (SDD, SPDD, BMAD) producen artefactos. ELP **además mide el proceso**. Sin esto, el flujo se degrada en semanas.

### 6.1 Phase 0 audits

Cuando un milestone abarca varias zonas o asume una hipótesis no validada, se ejecuta una lane **doc-only** que valida la premisa con datos antes de escribir código.

Estructura del audit (`docs/<milestone>-phase0-audit.md`):

1. Inventario empírico (greps, conteos, scans del código actual).
2. Cuantificación de superficie (refs por archivo, tipos de cambio).
3. Survey de opciones (A/B/C/D) con costo + riesgo + estado resultante.
4. Estrategia de gating (cómo se mantiene verde la build en cada PR).
5. Inventario de riesgos.
6. Verdict + recomendación de phase order.

**Resultado típico documentado en la práctica:** algunos audits descartan la propuesta entera con datos empíricos, ahorrando semanas de trabajo en un milestone equivocado.

### 6.2 Lane retros

Cada lane escribe `docs/lane-experience-<lane>.md` **antes** del PR. **Sin retro, no hay PR.** Estructura fija:

- **Objective metrics:** timestamps de start/end, conteos de invocaciones a build/test.
- **Migration / change inventory:** archivos modificados, líneas agregadas/borradas, sites migrados.
- **Compiler errors I encountered:** por clase de error, dónde, el fix, intentos fallidos.
- **Friction points:** preguntas específicas que el brief pidió responder.
- **Spec ambiguities or interpretive choices:** dónde el agente tuvo que decidir sin que el brief le dijera.
- **Subjective summary:** confidence, hardest, easiest, en qué ayudó/estorbó la herramienta.
- **Limitations of this report.**

Las retros sirven cuatro propósitos:

1. Capturan información mientras el contexto está fresco.
2. Son input del siguiente lane (lessons learned).
3. Son dataset para evaluar honestamente *LLM authorability* (qué porción del trabajo el agente realmente puede completar).
4. Documentan gaps que se vuelven issues nuevos.

### 6.3 Mediciones (TSV)

Antes de la primera invocación a `make` o equivalente, el brief instruye:

```bash
export LANE="issue-NNN-descr"
date -Iseconds > /tmp/lane-${LANE}-start.txt
echo -e "timestamp\tcmd\toutcome\telapsed_s" \
    > /tmp/lane-${LANE}-builds.tsv
```

Cada `make` se loguea como una línea TSV. Al cerrar el lane, se appendea al retro.

El TSV captura:

- Wall-clock real del lane.
- Cuántos retries de cada tier necesitó.
- Tiempo gastado en debugging vs implementación pura.

Es el dataset para evaluar honestamente cuánto cuesta cada lane y comparar lanes similares.

### 6.4 Honesty targets

Documentos pinneados (`docs/<feature>-honesty-targets.md`) que registran dónde el comportamiento del sistema es **ficticio vs real**. Por ejemplo:

> *"0 fires en self-compile, aquí está por qué — el typer pre-incref'a antes del check."*

Esto no infla la métrica. Reporta lo medido, no lo proyectado. La cultura es verificable contra el TSV y las retros.

### 6.5 Hallazgos como issues

Cuando un agente identifica un gap fuera de scope, el flujo es:

1. Documenta en *Friction points* del retro.
2. **NO lo arregla.** STOP-and-report.
3. El integrador decide si abrir issue ahora.
4. Si se abre, el body del nuevo issue cita la retro como fuente.

Cada gap nuevo es **su propio lane**. Esto evita que un lane se convierta en un yak-shave infinito.

---

## 7. Patrones operativos

### 7.1 Lanes paralelas vs secuenciales

**Paralelas** cuando:

- Tocan zonas distintas del sistema.
- No comparten archivos consumidores.
- Outputs ortogonales.

**Secuenciales** cuando:

- Tocan los mismos archivos consumidores.
- Una establece patrón que la siguiente replica.
- La siguiente necesita la **retro** de la anterior.

Criterio operativo: ¿se pisan en el área de trabajo o en la lección?

### 7.2 STOP-and-report

El brief siempre incluye explícitamente: *"si encuentras X, STOP y reporta"*. Casos típicos:

- Bug adyacente al scope.
- Decisión de diseño que el brief no cubre.
- Necesidad de tocar más zonas que las anticipadas.

El agente **no improvisa decisiones de scope**. Reporta y espera.

### 7.3 Wakeup scheduling

Para no requerir polling humano constante ni dejar lanes huérfanas, se programa al integrador a despertarse en intervalos definidos.

Reglas operativas:

- Delay típico entre 60 y 3600 segundos.
- Saltar el bracket 60-270 si no hay urgencia: el cache de prompt dura 5 minutos, esperas más cortas pierden cache.
- El prompt del wakeup es self-contained: cuando dispara, debe interpretarse sin contexto previo.

### 7.4 Auto-merge en CI verde

Cuando *branch protection* requiere checks específicos verdes, el lane habilita auto-merge en cada PR. Cuando los checks pasan, el merge dispara automáticamente.

**Excepciones aprendidas:**

- Doc-only PRs pueden quedar bloqueados si `paths-ignore` skipea el workflow y el check nunca se genera. Solución: workflow self-skip que reporta SUCCESS sin correr.
- Sub-checks no requeridos pueden estar rojos y el PR igual mergea. Lección: correr todos los gates **localmente** antes de push, no confiar solo en CI.

---

## 8. Reglas absolutas

Reglas no negociables que aplican a todos los actores del flujo:

1. **No tocar archivos de versionamiento** (`CHANGELOG`, `VERSION`); los regenera el bump al release.
2. **No push a `main` directo**; siempre PR + auto-merge.
3. **Sin autorización explícita: no `worktree remove` ni `branch -D`.**
4. **No comprometer trabajo del usuario**; investigar antes de borrar.
5. **Retros obligatorias** por cada lane.
6. **Phase 0 audits** para milestones grandes (>1 PR, hipótesis no validada, varias zonas).
7. **Honesty over optimism:** reportar lo medido, no lo proyectado.
8. **Briefs detallados:** un brief delgado produce un agente perdido.

---

## 9. Anti-patrones

Patrones documentados como problemáticos que ELP evita explícitamente:

- **Doc files prematuros** (TODOs, plan.md, decision.md). Issues + lane retros cubren lo persistente; el resto vive en la conversación.
- **Re-fix de bug fuera de scope:** abre issue, no extiendas el lane.
- **Migración con aliases supervivientes no documentados:** cada alias vivo necesita causa pinneada en el PR body.
- **Auto-merge sin verificar gates locales:** sub-checks pueden mentir.
- **Briefs delgados:** *"implementa X"* sin contexto produce un agente perdido.

---

## 10. Cuándo aplicar / cuándo no

### 10.1 ELP encaja cuando

- El proyecto va a durar meses, no días.
- Hay scope para múltiples PRs en paralelo (zonas ortogonales del sistema).
- Hay valor en conservar el aprendizaje entre lanes.
- El arquitecto puede dedicar tiempo a escribir briefs.

### 10.2 ELP NO encaja cuando

- Es un prototipo desechable. La sobrecarga de briefs y retros no se amortiza.
- Es un hotfix urgente. La cadencia de PRs en paralelo no aplica.
- El equipo no tiene tests ni CI maduros. ELP **amplifica** lo que ya hay; sin pipeline, amplifica el caos.
- El arquitecto no puede dedicar 10-20 minutos por brief. La calidad del brief es el techo de la calidad de la lane.

---

## 11. Comparación con métodos cercanos

### 11.1 ELP vs BMAD-METHOD

BMAD divide por **especialidad funcional** (Analyst → PM → Architect → Dev → QA). ELP divide por **unidad paralelizable** (lanes generalistas con scope acotado).

| Dimensión | BMAD | ELP |
|---|---|---|
| División del trabajo | por especialidad | por unidad paralelizable |
| Topología | secuencial (línea de fábrica) | paralela (pool de generalistas) |
| Artefacto central | PRD compartido | brief + retro por lane |
| Fuente del *qué construir* | PRD generado por PM | issues del backlog |
| Captura del proceso | parcial | persistente (audits + retros + TSV) |
| Aislamiento de cambios | un branch | un worktree por lane |

### 11.2 ELP vs SPDD

SPDD trata al prompt como **artefacto vivo** versionado en sincronía con el código. ELP trata al **brief** como contrato de un solo uso, pero acumula aprendizaje en las **retros**, que sí se versionan. Son ortogonales: se pueden combinar.

### 11.3 ELP vs SDD

SDD entrega la spec antes del código y la descarta. ELP entrega un brief antes del código y descarta el brief, pero conserva la retro. La retro reemplaza a la spec descartable como input para iteraciones futuras.

### 11.4 ELP vs Pantser

El término *Pantser* aplicado al desarrollo de software fue articulado por Eduardo Díaz en *Mapas o Brújulas* (lnds.net, 2025), tomado prestado de la literatura: el *Pantser* es quien escribe **by the seat of his pants**, sin un mapa rígido, ajustando dirección con cada paso. Su contraparte clásica es el *Plotter*, que planifica meticulosamente antes de ejecutar.

Pantser itera empíricamente con review humano disciplinado, pero el aprendizaje **muere con la conversación**. ELP captura el aprendizaje en formatos persistentes que sobreviven al cierre de la sesión. La diferencia central es **persistencia del aprendizaje**, no presencia de empirismo.

---

## 12. Métricas

Métricas que ELP recomienda capturar a nivel de proyecto:

- **PRs mergeados** por día / semana.
- **Lanes paralelas pico** (concurrencia simultánea).
- **% de lanes que terminan en STOP-and-report útil** (indicador de calidad de brief).
- **Wall-clock por lane** vs estimado (desde el TSV).
- **Retries por lane** (debugging real vs implementación primera vez).
- **Phase 0 audits ejecutados** y **% que descartaron el milestone**.
- **Issues abiertos como follow-up** desde retros (señal de salud del flujo).

---

## 13. Caso de estudio: compilador para un lenguaje funcional con efectos algebraicos

ELP se formalizó a partir de la práctica acumulada en un proyecto concreto: la construcción desde cero de un **lenguaje de programación funcional self-hosted con efectos algebraicos** y backend LLVM, ejecutado entre el **20 de abril y el 4 de mayo de 2026** (14 días corridos).

### 13.1 Cifras crudas

| Indicador | Valor |
|---|---|
| Wall-clock total | **14 días corridos** |
| Commits | **868** |
| Pull requests mergeados | **144** |
| Issues cerrados | **69** |
| Releases | v0.1.0 → **v0.31.0** |
| Pico de commits en un día | **135** |
| Phase 0 audits ejecutados | 3 |
| Lane retros producidas | 25+ |
| Lanes paralelas pico | 3 simultáneas |

Las cifras no son percepción: salen del `git log`, del `gh pr list` y del TSV de cada lane. Cada PR cita la retro que lo precede.

### 13.2 Distribución de commits

```
   feat        ████████████████████   38%   features nuevas
   docs        ███████████████        29%   audits + retros + design docs
   refactor    ██████                 12%
   fix         █████                  11%
   test/ci/chore █████                10%
```

**29 % docs es alto** para un compilador. Es el dataset empírico que ELP recomienda: Phase 0 audits, lane retros, design docs por feature, honesty targets. **11 % fix es bajo**: las features llegaron limpias gracias a los gates (selfhost byte-identical desde el día 1, tier 1 obligatorio antes de push, retro obligatoria antes del PR).

> Pagar adelante el 29 % de docs es lo que mantiene el 11 % de fix.

### 13.3 Lo que se construyó

Un lenguaje funcional self-hosted con propiedades poco frecuentes en una sola implementación:

- **Compilador 3-stage**: stage 0 en C (~10K líneas) → stage 1 bridge (~6K) → stage 2 self-hosted (~41K líneas).
- **Selfhost byte-identical** como gate desde el día 1: cada cambio debe poder recompilarse a sí mismo bytes idénticos.
- **Effect system** estilo Effekt: capabilities, handlers, *row inference*, efectos como `Console`, `Spawn`, `Cancel`, `Actor`, `Mutable`, `Ffi`.
- **Perceus RC** estilo Koka: conteo de referencias compilado en tiempo de compilación, sin borrow checker, con `reuse-in-place` y eliminación de increfs/decrefs redundantes.
- **Fibers BEAM-style**: heap privado por fiber, mensajes copiados, scheduler cooperativo con `swapcontext`.
- **Stdlib + tooling completo**: `kai build/run/test/fmt/repl/lsp/doc/bench/check`.
- **TCO mandatory**: tail-call optimization como regla del lenguaje, *self-tail-calls* reescritos a goto-loops en C-emit.

### 13.4 Comparación con la industria

Proyectos comparables públicos, con su tiempo a self-hosting funcional:

| Proyecto | Tiempo | Equipo |
|---|---|---|
| Roc (efectos + funcional) | ~5 años | research |
| Koka (inventó Perceus) | ~10+ años | académico |
| Effekt (capabilities) | ~5 años | PhD |
| Crystal (LLVM + inferencia) | ~3 años | core team |

Equivalente humano para alcanzar el mismo *scope* técnico:

> **3-4 ingenieros senior compiler-experts × 12-18 meses ≈ USD 1.0-1.5 M en payroll.**

ELP llegó al mismo punto en **12-14 días, con un humano (arquitecto) + un agente integrador + lanes paralelas**.

### 13.5 Qué del método contribuyó

Atribución honesta del resultado, no extrapolación:

1. **Selfhost byte-identical desde el día 1.** Cualquier cambio que rompa la auto-compilación se descubre en la build, no en producción. Blindó cada feature contra regresiones invisibles.
2. **Three-stage bootstrap.** Permite que stage 2 use el lenguaje completo sin limitarse al subset que un compilador C-only podría implementar. Stage 1 funciona como bridge.
3. **Phase 0 audits como cultura.** Tres audits ya ahorraron semanas: uno descartó por completo una propuesta de bootstrap split con datos empíricos. La premisa era falsa; sin el audit, el equipo habría implementado sobre una hipótesis equivocada.
4. **Lane retros como dataset, no documentación.** Retros que documentan compiler errors, fix paths fallidos y *friction points* alimentan el siguiente lane y son el dato real para evaluar *LLM authorability*.
5. **Honesty targets pinneados.** Cada feature documenta dónde su comportamiento es ficticio vs real (ej.: *"0 fires en self-compile, aquí está por qué"*). La cultura escala porque la honestidad es verificable contra los TSV.
6. **Issues como roadmap.** El *qué construir* funcional vive en GitHub, separado del *cómo* (que vive en el brief). Esto evita que cada lane se convierta en un yak-shave.

### 13.6 Lo que se cerró como negativo

Decisiones que no calzaron, pinneadas como *negativos*:

- *Drop specialisation*: medido −1.7 % al `-O2`, +5.4 % regresión al `-O0`. Lane cerrada como negativo. La phase 2 unboxing había absorbido el overhead.
- *Bootstrap split*: descartado por audit empírico antes de implementarse. Phase 0 ahorró 2-3 semanas de trabajo equivocado.

Estos negativos no son fallas, son **datos**. Cada uno tiene retro pinneada que evita repetir.

### 13.7 Observación final

El proyecto tiene una propiedad poco frecuente: es **ambicioso técnicamente y honesto operacionalmente** al mismo tiempo. Phase 0 audits + lane retros + honesty targets construyen un dataset que un equipo humano normalmente posterga o pierde. El humano (arquitecto) no escribió código; mantuvo juicio de scope, decisiones de arquitectura y *calls* de "esto no me convence". Los agentes ejecutaron lanes con briefs detallados.

Si la hipótesis de fondo (LLM authorability validable contra el dataset acumulado) se sostiene, este caso es la primera evidencia empírica documentada de un proyecto técnicamente complejo construido bajo ELP.

---

## 14. Limitaciones conocidas

- Si el arquitecto está dormido y un agente hace STOP-and-report, la lane queda **bloqueada** hasta que el humano responda. *Workaround:* briefs con autorización pre-aprobada para casos comunes.
- Si CI tarda y el wakeup dispara antes, se gasta cache de prompt sin avance. *Tradeoff* consciente.
- Briefs detallados toman 10-20 minutos del arquitecto. No siempre vale para lanes triviales (ej.: edición de un archivo).
- El método asume herramientas de versionamiento y CI maduras; en proyectos sin ellas, no aplica directamente.
- Cambio de máquina o reinicio de sesión del integrador pierde contexto vivo. No hay solución estructural hoy.

---

## 15. Origen y referencias

ELP fue formalizado a partir de la práctica acumulada en proyectos de larga duración con múltiples agentes IA trabajando en paralelo. La primera articulación pública del método ocurre en este documento.

Métodos y referencias relacionadas:

- **Díaz, E.,** *"Mapas o Brújulas"*, lnds.net, 2025-10-31. URL: <https://lnds.net/blog/lnds/2025/10/31/mapas-o-brujulas/>. Articulación en español del binomio *Plotter / Pantser* aplicado al desarrollo de software.
- **BMAD-METHOD** (Breakthrough Method for Agile AI-Driven Development) — pipeline multi-agente con roles funcionales.
- **SDD** (Spec-Driven Development) — la especificación es el input del modelo.
- **SPDD** (Structured Prompt-Driven Development) — Zhang & Xia, Thoughtworks, 2026. El prompt como artefacto vivo.
- **Saltzer & Schroeder, *The Protection of Information in Computer Systems***, 1975 — los principios de diseño seguro que aplican igual al agente.
- **Karpathy, A.,** *"Vibe Coding"*, 2024.
- **Krouse, S.,** *"Vibe Code is Legacy Code"*, 2025.
- **Emerson, Lake & Palmer**, *Karn Evil 9: First Impression Part 2*, 1973. Origen del *backronym* ELP.

---

## Apéndice A — Plantilla mínima de brief

```markdown
# Lane brief — issue-NNN-<descr>

## Goal
[1-3 frases. Qué se entrega y por qué.]

## Context
- Repo: <repo>, branch: issue-NNN-<descr>
- Verificación: git log --oneline -5 origin/main
- Reglas pinneadas del proyecto: CLAUDE.md
- Lanes paralelas en flight: [lista]

## Read first
- Issue: gh issue view NNN
- Audit: docs/<milestone>-phase0-audit.md (si aplica)
- Retros relacionadas: docs/lane-experience-<previo>.md
- Archivos a inspeccionar: <paths>

## Decisions pinned
- [decisión 1]
- [decisión 2]

## Lane scope
DO:
- [acción 1]
- [acción 2]

DO NOT:
- [prohibición 1]
- [prohibición 2]

STOP-and-report si:
- [condición 1]

## Required outputs
- Archivos esperados modificados: <paths>
- Mensaje de commit: <type>(<scope>): <subject>
- PR body: [qué incluir]
- Cierre: gh pr create + gh pr merge --auto

## Discipline
- Tier 0 + Tier 1 local antes de push.
- Working tree limpio en cada commit.

## Instrumentation
- TSV de builds en /tmp/lane-${LANE}-builds.tsv.
- Retro en docs/lane-experience-issue-NNN-<descr>.md (secciones obligatorias).
```

---

## Apéndice B — Plantilla mínima de lane retro

```markdown
# Lane retro — issue-NNN-<descr>

## Objective metrics
- Start: <ISO timestamp>
- End: <ISO timestamp>
- Wall-clock: <duración>
- Build invocations: <conteo por tier>

## Migration / change inventory
- Archivos modificados: <lista>
- Líneas agregadas / borradas: +X / -Y
- Sites tocados: <conteo>

## Compiler errors I encountered
- [Clase de error]: dónde, fix, intentos.

## Friction points
- [Pregunta del brief]: respuesta basada en lo observado.

## Spec ambiguities or interpretive choices
- [Decisión sin spec explícita]: justificación.

## Subjective summary
- Confidence: alto / medio / bajo.
- Hardest: ...
- Easiest: ...
- Compiler / tooling help / hinder: ...

## Limitations of this report
- [Qué este reporte no cubre.]
```
