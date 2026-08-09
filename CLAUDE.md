# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Proyecto

Tetris clásico en canvas HTML5 vanilla: tres archivos ([index.html](index.html),
[style.css](style.css), [game.js](game.js)), sin dependencias, sin bundler, sin
`package.json` y sin tests.

`game.js` se carga como script clásico (**no** módulo ES), así que abrir
`index.html` con doble clic funciona. Un servidor estático (`npx serve .`,
`python3 -m http.server 8000`) sólo hace falta para recargas cómodas.

No hay linter ni suite de tests; la verificación es `node --check game.js` para
sintaxis y jugar en el navegador para el comportamiento.

Los textos de UI y la documentación están en español (`PAUSA`, `Puntuación`,
`Reiniciar`); los identificadores y comentarios del código, en inglés. Mantener
esa división al editar.

## Acoplamiento entre los tres archivos

`game.js` resuelve todos sus nodos del DOM en `const` de nivel de módulo, al
cargarse: `board`, `next-canvas`, `score`, `lines`, `level`, `overlay`,
`overlay-title`, `overlay-score`, `restart-btn`. Renombrar un `id` en el HTML
rompe el juego en silencio (`getContext` sobre `null`), y el `<script>` debe
seguir al final de `<body>`.

El tamaño del canvas está declarado dos veces: `width`/`height` de
`<canvas id="board">` en el HTML deben ser exactamente `COLS * BLOCK` × `ROWS * BLOCK`
(hoy 300 × 600). Cambiar `COLS`, `ROWS` o `BLOCK` sin tocar el HTML deja el
tablero recortado o con espacio muerto. Lo mismo para `#next-canvas` (120 × 120)
respecto del `NB = 30` inline de `drawNext()`.

## Arquitectura de `game.js`

**Tablas alineadas por índice de pieza.** `PIECES[t]` y `COLORS[t]` comparten el
índice `t` (1–7), con `null` en la posición 0 como relleno. Las matrices de forma
no guardan `1`s sino su propio índice de tipo (la S es todo `4`s), y ese valor es
lo que se copia al tablero en `merge()`. Por eso una celda del tablero es a la
vez "ocupada" y "de qué color": `0` es vacío. Agregar una pieza implica extender
ambas tablas **y** el `Math.floor(Math.random() * 7) + 1` de `randomPiece()`.

**Estado mutable de módulo** (`board`, `current`, `next`, `score`, `lines`,
`level`, `paused`, `gameOver`, `lastTime`, `dropAccum`, `dropInterval`, `animId`):
se declara sin valor y se inicializa entero en `init()`, que es además el handler
de "Reiniciar". Estado nuevo va inicializado en `init()`, no en la declaración, o
sobrevive al reinicio.

**El bucle no tiene máquina de estados.** `loop()` no consulta `paused` ni
`gameOver`: pausar y morir simplemente hacen `cancelAnimationFrame(animId)`, y
`togglePause()` reanuda llamando a `loop()` de nuevo. Por eso resetea `lastTime`
antes de reanudar — sin eso, el primer `dt` incluiría toda la pausa. Consecuencia
práctica: lógica nueva por frame va dentro de `loop()` y sólo corre en juego
activo; el `keydown` se protege aparte con el guard `if (paused || gameOver) return`.

**Tiempo en milisegundos**, a diferencia de otros proyectos del curso: `dropAccum`
acumula el `dt` crudo de `requestAnimationFrame` sin tope. Al superar
`dropInterval` se pone en `0` (no se le resta el intervalo), así que tras una
pestaña en segundo plano se baja una sola fila, no las acumuladas.

**HUD por push, no por frame.** `draw()` sólo pinta los canvas; los marcadores se
actualizan en `updateHUD()`, invocado desde `init()`, `clearLines()`, `softDrop()`
y el final del handler de `keydown`. `hardDrop()` suma puntos sin llamarlo y
depende de ese `updateHUD()` de cola. Cualquier fuente nueva de puntaje tiene que
llamar a `updateHUD()` explícitamente.

**`collide()` permite `ny < 0` a propósito**: sólo son pared las columnas fuera de
`[0, COLS)` y `ny >= ROWS`. Eso deja que una pieza exista parcialmente por encima
del tablero al aparecer. `spawn()` declara game over cuando la pieza nueva ya
colisiona en su posición inicial.

**Rotación sin SRS.** `tryRotate()` rota con `rotateCW()` (transpuesta + inversión)
y prueba desplazamientos horizontales `[0, -1, 1, -2, 2]`; si ninguno entra, el
giro se descarta. No hay estado de rotación ni tablas de kick por pieza.

**`clearLines()` recorre de abajo hacia arriba** y por cada fila completa hace
`splice` + `unshift` y re-incrementa `r` para revisar de nuevo el mismo índice
(ahora ocupado por la fila que bajó). Deriva ahí mismo `level = floor(lines/10)+1`
y `dropInterval = max(100, 1000 - (level-1)*90)`; el puntaje es
`LINE_SCORES[cleared] * level`.

**Dibujo**: `draw()` limpia todo el canvas cada frame y pinta en orden grid →
tablero → fantasma → pieza actual. `drawBlock()` es compartido con la vista previa
— recibe el `context` y el `size`, y el `alpha` opcional es lo que hace translúcida
la pieza fantasma. `drawNext()` no corre por frame: sólo se llama desde `spawn()`.

**Overlay único** para pausa y game over: el mismo `#overlay` cambia de texto y se
muestra/oculta con la clase `.hidden`.
