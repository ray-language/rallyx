# rallyx

Rally-X de terminal a 30 fps, escrito en [raylang](https://github.com/ray-language/raylang): el coche que no se detiene, las 10 banderas que valen más cuanto más llevas, los coches rojos que te persiguen, las rocas, el combustible que se agota — y la cortina de humo que atonta al que la pisa. Como en el arcade, la ciudad es **más grande que la pantalla**: la cámara sigue al coche y el **radar** del panel muestra dónde quedan las banderas.

```text
$ rallyx             # ↑↓←→ conducir · espacio humo · p pausa · r reiniciar · q salir
$ rallyx --seed      # ciudad determinista (banderas/rocas con semilla fija)
```

## Las reglas

- El coche **siempre avanza** en su rumbo. Las flechas **encolan** el giro:
  el coche lo toma en la primera bocacalle abierta (ninguna pulsación se
  pierde contra una pared — el volante de Pac-Man/Rally-X).
- Ciudad de **48×32** calles con manzanas y tres plazas; viewport de 26×18
  con cámara que sigue al coche, y radar 16×8 de toda la ciudad: tú (cian),
  los rojos, las banderas (la especial en magenta).
- **10 banderas** por ronda: la n-ésima vale `100×n`. Una es la **especial
  (S)**: desde que la coges, todo se **duplica**. Completar la ronda suma el
  combustible restante como bonus y trae un perseguidor más (hasta 4), un
  poco más rápido cada ronda.
- **Espacio** suelta humo en la celda que acabas de dejar (cuesta 30 de
  combustible). La nube se desvanece en tres fases (▓▒░); el perseguidor
  que la pisa queda atontado (××) 10 turnos y la consume.
- Sin combustible no mueres: el coche va a **media velocidad** (el limp-home
  del arcade) — y los rojos no.
- 3 vidas; te cazan o chocas con una roca (◢◣) → todos a sus esquinas, con
  turnos de gracia. High score persistente en `~/.rallyx_hiscore`.

## Cómo está hecho

La disciplina de raygame (Tetris), con un reloj más y una cámara:

- **Reglas puras y sin reloj** (`src/rally.ray`): ciudad generada por
  construcción (calles en retícula + plazas talladas — conexa por diseño),
  volante encolado, persecución greedy Manhattan sin marcha atrás,
  humo/stun, puntuación secuencial con duplicador, rondas. Determinista con
  `random.seed` — los 10 tests la ejercitan sin terminal ni tiempo.
- **Tres relojes en un bucle**: frame (33 ms, repinta solo si algo cambió),
  tick del jugador (150 ms; 300 ms sin combustible) y tick de los
  perseguidores (170 ms − 6/ronda, mínimo 120). La espera única es
  `io.read_timeout(64, hasta_el_frame)`.
- Frames de líneas fijas + diff absoluto (`ESC[n;1H` + `ESC[2K`); coches
  como flechas según rumbo (▲▼◀▶), celdas de doble ancho, colores dentro de
  la línea. Con la cámara clavada en un borde el diff sigue siendo mínimo
  (≤4 líneas por movimiento — hay test); cuando la cámara desplaza, repinta
  el viewport y el bench de raygame ya demostró que eso son microsegundos.
- Verificado bajo pty real (`script -q /dev/null`): entra/sale limpio del
  alt-screen, conduce, fuma y guarda el hi-score.

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| Ciudad 48×32 con cámara + radar de banderas/perseguidores | ✅ |
| Volante encolado: el giro se toma en la primera bocacalle (hay test) | ✅ |
| 10 banderas + especial que duplica, bonus de combustible por ronda | ✅ |
| Perseguidores con chase greedy, humo que se desvanece y atonta, rocas | ✅ |
| Rondas progresivas (más rojos, más rápidos), 3 vidas, high score | ✅ |
| 30 fps con input sin bloqueo + diff mínimo con cámara clavada | ✅ |
| Binario nativo (jugado bajo pty) | ✅ |
| Tests (reglas + volante + cámara + frame) | ✅ 10 |
| Bache que frena, persecución con lookahead (BFS), túneles laterales | 📋 v2 |

## Hallazgos de dogfood

1. **`ray build --native -o X` sobre un `X` existente → SIGKILL en macOS**
   (anotado en `raylang/IDEAS.md` §77): sobrescribir el binario in-place
   invalida la firma ad-hoc y el kernel mata el proceso al exec (exit 137,
   incluso `--help`). Workaround: `rm -f` antes de recompilar. Propuesta:
   que `ray build` haga unlink/rename del output.
2. Gotcha de parser (documentado): una tupla `(cx, cy)` como cola de función
   justo tras un bloque `if` se parsea como llamada al valor del bloque — el
   diagnóstico del checker lo explica y sugiere `return`/`let`. Buen error.
3. Lo demás salió a la primera sobre la superficie M115–M127 (v1 del juego:
   tests 8/8 y pty al primer intento).

## Desarrollo

```sh
ray test
rm -f rallyx && ray build --native src/main.ray -o rallyx --release   # ver hallazgo 1
```

Estructura: `src/main.ray` · `rally.ray` (reglas puras) · `screen.ray`
(cámara + radar + frame/diff) · `app.ray` (bucle de tres relojes).
