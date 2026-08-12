---
description: Analiza un issue de GitHub, le aplica labels de la taxonomía del repo y publica un diagnóstico en Markdown
---

Recibes como argumentos `REPO: <owner/repo>` y `ISSUE_NUMBER: <número>`. Extrae ambos valores del texto del prompt.

## Objetivo

Analizar el issue indicado, clasificarlo con labels de la taxonomía fija del repo, y publicar (o actualizar) un único comentario de diagnóstico en español, en Markdown, en el propio issue.

## Contexto del proyecto

Este es un Tetris clásico en JavaScript vanilla (Canvas 2D), sin build ni dependencias. Toda la lógica vive en `game.js` (~300 líneas, scope global, sin clases). Lee `CLAUDE.md` para la arquitectura completa antes de analizar: modelo de tablero, `PIECES`/rotación, `collide`/`tryRotate` (wall kicks), el loop (`requestAnimationFrame`, `dropAccum`, `dropInterval`), lock/clear de líneas, scoring (`LINE_SCORES`), niveles, ghost piece, `draw()`/`drawNext()`, el listener de `keydown`, y game over/restart.

No hay tests ni pipeline de build que ejecutar — el diagnóstico es siempre un análisis estático del código.

## Pasos

1. **Leer el issue:**
   ```
   gh issue view $ISSUE_NUMBER --repo $REPO --json title,body,labels,author,createdAt
   ```

2. **Leer el código relevante.** Como mínimo `CLAUDE.md`, `game.js`, `index.html`, `style.css`. Usa `Read`/`Grep`/`Glob` — no necesitas `Bash` para esto.

3. **Leer la taxonomía de labels** desde `.github/labels.json` (ya existe en el repo, no la inventes). Elige:
   - Exactamente **un** label de tipo: `bug`, `enhancement`, `question` o `documentation`.
   - **Una o más** áreas: `area:gameplay`, `area:rendering`, `area:input`, `area:scoring`, `area:ui` — las que apliquen según el código involucrado.
   - Exactamente **un** label de prioridad: `priority:high`, `priority:medium`, `priority:low`.
   - `needs-info` si al issue le falta información clave para diagnosticar (pasos de reproducción, versión de navegador, etc.).
   - `good first issue` solo si el fix es genuinamente pequeño y acotado a una función.

   Aplica los labels elegidos:
   ```
   gh issue edit $ISSUE_NUMBER --repo $REPO --add-label "bug,area:rendering,priority:medium"
   ```
   No apliques ningún label que no esté en `.github/labels.json`.

4. **Redactar el diagnóstico** en **español**, en Markdown, con esta estructura exacta:

   ```markdown
   <!-- claude-issue-analysis -->
   ## 🔍 Diagnóstico automático

   ### Resumen
   (1-2 frases: qué pide o reporta el issue)

   ### Clasificación
   | Campo | Valor | Motivo |
   |---|---|---|
   | Tipo | ... | ... |
   | Área(s) | ... | ... |
   | Prioridad | ... | ... |

   ### Análisis técnico
   (Funciones y archivos concretos involucrados, referenciados como `game.js:NNN`. Explica la causa probable o el alcance del cambio pedido.)

   ### Pasos sugeridos
   - [ ] ...
   - [ ] ...

   ### Información faltante
   (Omitir esta sección por completo si no falta nada.)

   ---
   _Generado automáticamente por Claude · [ver ejecución](URL_DEL_RUN)_
   ```

   Para `URL_DEL_RUN`, construye la URL real usando las variables de entorno que ya provee el runner de GitHub Actions: `$GITHUB_SERVER_URL/$GITHUB_REPOSITORY/actions/runs/$GITHUB_RUN_ID`.

5. **Publicar como comentario "sticky"** (uno solo por issue, se actualiza en vez de duplicarse):

   Guarda el Markdown del paso 4 en un archivo temporal y luego:
   ```
   gh api repos/$REPO/issues/$ISSUE_NUMBER/comments --paginate --jq '.[] | select(.body | startswith("<!-- claude-issue-analysis -->")) | .id'
   ```
   - Si devuelve un id → actualiza ese comentario:
     ```
     gh api -X PATCH repos/$REPO/issues/comments/<id> -f body=@archivo.md
     ```
   - Si no devuelve nada → crea uno nuevo:
     ```
     gh issue comment $ISSUE_NUMBER --repo $REPO --body-file archivo.md
     ```

## Prohibido

- No edites el título ni el cuerpo del issue.
- No cierres el issue.
- No hagas commits ni abras pull requests.
- No apliques ni elimines labels fuera de la taxonomía de `.github/labels.json`.
- No publiques más de un comentario de diagnóstico por issue (usa siempre el flujo sticky del paso 5).
