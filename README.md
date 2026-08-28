# rallyx

Rally-X de terminal a 30 fps, escrito en [raylang](https://github.com/ray-language/raylang): el coche que no se detiene, las 10 banderas que valen más cuanto más llevas, los coches rojos que te persiguen, las rocas, el combustible que se agota — y la cortina de humo que atonta al que la pisa. Como en el arcade, la ciudad es **más grande que la pantalla**: la cámara sigue al coche y el **radar** del panel muestra dónde quedan las banderas.

```text
$ rallyx             # ↑↓←→ conducir · espacio humo · p pausa · r reiniciar · q salir
$ rallyx --seed      # ciudad determinista (banderas/rocas con semilla fija)
$ rallyx --img f.png # dibuja cualquier PNG en el terminal (half-blocks truecolor) y sale
```

Con `std/inflate` en el lenguaje, rallyx trae ahora un **codec PNG puro en
raylang** (`src/png.ray`: IDAT vía `zlib_inflate`, filtros 0–4, RGB/RGBA/
paleta) y un **renderer de sprites** por half-blocks truecolor
(`src/sprite.ray`: `▀` con fg = píxel superior y bg = inferior — 2 px por
celda, la técnica de chafa/viu). El splash de arranque dibuja el coche desde
`assets/car.png` — un PNG **generado por el propio encoder raylang**
(`ray run tools/gen_assets.ray`: bloques DEFLATE stored + CRC-32 + Adler-32;
`file` y `sips` lo aceptan).

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
| Codec PNG puro (decode 2/3/6 + filtros 0–4; encode stored+CRC) | ✅ |
| Sprites half-block truecolor: splash con el coche + visor `--img` | ✅ |
| Tests (reglas + volante + cámara + frame + codec PNG + sprites) | ✅ 16 |
| Bache que frena, persecución con lookahead (BFS), túneles laterales | 📋 v2 |
| Sprites en celda de juego, música WSG vía `stdin_pipe` + ffplay | 📋 v2 |

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
4. **`std/inflate` sostiene un decoder PNG completo** sin fricción: con
   `zlib_inflate` + `bytes` indexables + `bytes_of` + bits/hex, el codec
   entero (decode con los 5 filtros + encode stored con CRC-32/Adler-32
   correctos) son ~300 líneas puras y testeables; el PNG generado lo aceptan
   `file` y `sips`. Único tropiezo: dos veces el gotcha de la cola `(a | b)`
   / `(a, b)` tras un bloque (el diagnóstico del checker lo resuelve solo).
5. Correcciones recibidas al mapa de audio del README anterior:
   `process.cmd(...).stdin_pipe().stream()` + `Proc.write` (M100 v3) da
   stdin vivo CON contrapresión (música reactiva vía `ffplay -f s16le -i -`
   funciona hoy), y el FFI existe desde M41 — lo no bindeable es solo el
   audio *pull* de CoreAudio (sin callbacks C→raylang); ALSA (push) sí.
   La asimetría macOS/Linux es el argumento real para un `std/audio`.

## Desarrollo

```sh
ray test
rm -f rallyx && ray build --native src/main.ray -o rallyx --release   # ver hallazgo 1
```

Estructura: `src/main.ray` · `rally.ray` (reglas puras) · `screen.ray`
(cámara + radar + frame/diff) · `app.ray` (bucle de tres relojes).
