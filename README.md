# rallyx

Rally-X de terminal a 30 fps, escrito en [raylang](https://github.com/ray-language/raylang): el coche que no se detiene, las 10 banderas que valen más cuanto más llevas, los coches rojos que te persiguen, las rocas, el combustible que se agota — y la cortina de humo que atonta al que la pisa.

```text
$ rallyx             # ↑↓←→ conducir · espacio humo · p pausa · r reiniciar · q salir
$ rallyx --seed      # laberinto determinista (banderas/rocas con semilla fija)
```

## Las reglas

- El coche **siempre avanza** en su rumbo; las flechas cambian el rumbo. Una
  pared lo detiene (y sigue quemando combustible); una roca es choque.
- **10 banderas** por ronda: la n-ésima vale `100×n`. Una es la **especial
  (S)**: desde que la coges, todo se **duplica**. Completar la ronda suma el
  combustible restante como bonus y trae un perseguidor más (hasta 4), un
  poco más rápido cada ronda.
- **Espacio** suelta humo en la celda que acabas de dejar (cuesta 25 de
  combustible). El perseguidor que lo pisa queda atontado 8 turnos y consume
  la nube.
- Sin combustible no mueres: el coche va a **media velocidad** (el limp-home
  del arcade) — y los rojos no.
- 3 vidas; te cazan o chocas → todos a sus esquinas, con turnos de gracia.
  High score persistente en `~/.rallyx_hiscore`.

## Cómo está hecho

La disciplina de raygame (Tetris), con un reloj más:

- **Reglas puras y sin reloj** (`src/rally.ray`): laberinto 24×16 declarado
  como strings, persecución greedy Manhattan sin marcha atrás, humo/stun,
  puntuación secuencial con duplicador, rondas. Determinista con
  `random.seed` — los 8 tests la ejercitan sin terminal ni tiempo.
- **Tres relojes en un bucle**: frame (33 ms, repinta solo si algo cambió),
  tick del jugador (90 ms; 180 ms sin combustible) y tick de los
  perseguidores (115 ms − 5/ronda, mínimo 85). La espera única es
  `io.read_timeout(64, hasta_el_frame)`.
- Frames de líneas fijas + diff absoluto (`ESC[n;1H` + `ESC[2K`); celdas de
  doble ancho, colores dentro de la línea. Verificado bajo pty real
  (`script -q /dev/null`): entra/sale limpio del alt-screen, conduce, fuma
  y guarda el hi-score.

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| Laberinto, coche continuo, 10 banderas + especial que duplica | ✅ |
| Perseguidores con chase greedy, humo que atonta, rocas | ✅ |
| Combustible: coste del humo, bonus de ronda, limp-home a media velocidad | ✅ |
| Rondas progresivas (más rojos, más rápidos), 3 vidas, high score | ✅ |
| 30 fps con input sin bloqueo + diff mínimo (≤3 líneas por movimiento) | ✅ |
| Binario nativo (jugado bajo pty) | ✅ |
| Tests (reglas puras + shape del frame + diff) | ✅ 8 |
| Radar de banderas, bache que frena, persecución con lookahead (BFS) | 📋 v2 |

## Hallazgos de dogfood

Ninguno nuevo: la app entera salió a la primera (tests 8/8, nativo incluido)
sobre la superficie ya endurecida por el arco M115–M127 — `\u{1b}` literal,
`s.chars()`, structs por referencia (incl. `g.player.x = nx` anidado y alias
de elementos de array), `ray fmt --write`. Es el resultado esperado tras
raygame/raytop: el patrón TUI de tres relojes ya no encuentra fricción.

## Desarrollo

```sh
ray test
ray build --native src/main.ray -o rallyx --release
```

Estructura: `src/main.ray` · `rally.ray` (reglas puras) · `screen.ray`
(frame + diff) · `app.ray` (bucle de tres relojes).
