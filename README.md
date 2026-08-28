# rallyx

Rally-X de terminal a 30 fps, escrito en [raylang](https://github.com/ray-language/raylang): el coche que no se detiene, las 10 banderas que valen más cuanto más llevas, los coches rojos que te persiguen, las rocas, el combustible que se agota — y la cortina de humo que atonta al que la pisa. Como en el arcade, la ciudad es **más grande que la pantalla**: la cámara sigue al coche y el **radar** del panel muestra dónde quedan las banderas.

```text
$ rallyx             # ↑↓←→ conducir · espacio humo · p pausa · r reiniciar · q salir
$ rallyx --seed      # ciudad determinista (banderas/rocas con semilla fija)
$ rallyx --img f.png # dibuja cualquier PNG en el terminal (half-blocks truecolor) y sale
$ rallyx --no-music  # sin música (sin dispositivo de audio, calla solo)
$ ray run tools/play_demo.ray  # tour audible de la partitura reactiva (~6 s; RAY_AUDIO_SINK=null para CI)
```

**Música reactiva estilo Namco WSG**, sintetizada en vivo y escrita
**directo al dispositivo con `std/audio`** (M145 — la tercera generación:
nació en afplay-batch mental, vivió en ffplay+stdin_pipe, y terminó en PCM
nativo): 3 voces de wavetable de 32 entradas × 4 bits (`src/wsg.ray`, puro
y determinista — la partitura se testea byte a byte), mezcladas a s16le
22050 Hz por una fibra (`src/music.ray`) que empuja un paso de 110 ms por
escritura. El juego le manda eventos por
canal: **sirena** cuando un rojo vivo está a ≤6 celdas, **jingle** al coger
bandera, **pshh de humo** (ruido LFSR de 15 bits que *roba la voz del
arpegio* 3 pasos, como el WSG real robaba voces para los SFX), **barrido**
al chocar, **despedida** en el game over y reset con `r`. Debajo de todo,
el **drone del motor** en una 4ª voz, ligado al coche real: zumba con
wobble cuando rueda, baja a ralentí grave si está parado contra una pared,
y **petardea** (un cilindro falla cada dos pasos) con el tanque vacío;
solo calla cuando el coche no está (choque, game over). La melodía es un **homenaje**
al galope del arcade (la partitura original es de Namco y no se transcribe):
fanfarria mayor con galope de semicorcheas en el bajo y el giro descendente
de cierre.
La clave del pacing, medida con `tools/audio_buf.ray`: la cola del
dispositivo absorbe **~1.7 s** antes de que `audio.write` aparque la fibra,
así que la contrapresión pura marca el *tempo* pero retrasaría los eventos
la cola entera — la fibra mantiene un **adelanto de reloj de pared de
~100 ms**. Lo que `std/audio` arregla de raíz es el modo de fallo que tuvo
la era ffplay: el dispositivo reproduce lo recién escrito **inmediatamente**
tras cualquier hueco (no inserta silencio en una línea de tiempo), así que
un arranque lento o un underrun se autocura en vez de volverse desfase
permanente — el reloj de pared vuelve a ser correcto, sin scraping del
reloj de ffplay.

Imágenes con la superficie M143/M144: el decode es **`std/image`**
(`decode_png` estricto — CRC por chunk, tipos 0/2/3/4/6, tRNS) y el dibujo
elige la **mejor capacidad del terminal** (`term.capabilities()`): gráficos
**kitty** de píxel real (el PNG entero por APC, `cell_px()` para encajar el
texto al layout) o half-blocks truecolor (`▀` fg/bg — 2 px por celda,
chafa/viu) como fallback universal. `src/png.ray` queda como **encoder**
puro (stored + CRC-32/Adler-32): genera `assets/car.png`
(`ray run tools/gen_assets.ray`, auto-verificado contra `std/image`) y
alimenta los **tests diferenciales** — los vectores de filtros calculados a
mano que validaron el decoder propio ahora fijan el de `std/image`.

## Las reglas

- El coche **siempre avanza** en su rumbo. Las flechas **encolan** el giro:
  el coche lo toma en la primera bocacalle abierta (ninguna pulsación se
  pierde contra una pared — el volante de Pac-Man/Rally-X).
- Ciudad de **48×32** calles con manzanas, tres plazas y **dos túneles
  laterales** (filas 13 y 19): salir por un borde es aparecer por el otro,
  y el humo también cruza. Viewport de 26×18 con cámara que sigue al coche,
  y radar 16×8 de toda la ciudad: tú (cian), los rojos, las banderas (la
  especial en magenta).
- **Baches** (▁▁ amarillos): pisarlos arrastra el coche unos turnos (150 →
  260 ms por celda). Los rojos conocen sus calles y no se inmutan.
- Los rojos cercanos (≤14 celdas) persiguen por **camino real (BFS)** —
  rodean manzanas, cruzan túneles y no se dejan engañar por la distancia en
  línea recta; de lejos van por olfato (greedy). Las rocas les cortan el
  paso; el humo sigue siendo tu única arma.
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
| PNG: decode vía `std/image` + encoder propio (stored+CRC) con tests diferenciales | ✅ |
| Sprites: kitty graphics si el terminal puede (`capabilities`/`cell_px`), half-blocks si no | ✅ |
| Música WSG reactiva en vivo (sirena/jingle/choque/game over) vía `std/audio` | ✅ |
| SFX: pshh de humo (ruido LFSR, roba la voz del arpegio) + drone de motor | ✅ |
| v2: túneles laterales con wrap (coche, humo y BFS los cruzan) | ✅ |
| v2: baches que arrastran el coche; persecución BFS con radio + fallback | ✅ |
| v2: motor ligado al coche real (rueda / ralentí / petardeo sin gasolina) | ✅ |
| Tests (reglas + volante + cámara + frame + PNG diferencial + sprites + sinte) | ✅ 31 |
| Sprites en celda de juego (necesita ≥8×8 px/celda: no cabe en un term 80×24) | 📋 v3 |

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
3b. **`term.capabilities()` no es reentrante dentro de `term.raw`**
   (`raylang/IDEAS.md` §80): llamada dentro de nuestra sesión raw, su
   restauración interna deja el terminal cocinado → todas las teclas
   muertas. Workaround aplicado: detectarla ANTES de `term.raw` y pasar el
   struct. Moraleja de arnés: `script -q` de macOS ni responde DA1 ni
   termina antes del EOF de stdin — para bugs de input hace falta un pty
   que conteste como un terminal real (hay arnés Python en el hallazgo).
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
6. **El patrón `stdin_pipe` validado E2E** (VM y nativo, bajo pty, sin
   zombies de ffplay al salir) — y la lección de dogfood más fina del
   proyecto, la **anatomía de la latencia de audio por pipe**, medida en
   `tools/audio_probe.ray`: (a) ffplay Y ffmpeg→AudioToolbox tragan PCM sin
   pacing hacia colas ilimitadas — la contrapresión de `Proc.write` nunca
   llega a actuar; (b) el hijo reproduce a 1× y nunca recupera, así que su
   arranque lento y cada underrun se acumulan como desfase PERMANENTE — con
   reloj de pared el juego sonaba ~1 s tarde hiciera lo que hiciera el
   lead; (c) la solución es cerrar el lazo con el **reloj de reproducción
   real**: ffplay lo publica en su línea de stats por stderr
   (`   1.25 M-A: …`), `Proc.err` lo entrega, y el pump escribe solo
   mientras `written − playhead < 120 ms`. Convergencia medida: adelanto
   estable en 130–220 ms, autocorregido tras cualquier hipo. Es el
   argumento definitivo para un `std/audio` con PCM directo al dispositivo:
   con pipe + scraping de stderr, ~150 ms es el suelo; con callback serían
   ~15 ms. Dos detalles de
   lenguaje: `Channel.bounded` necesita anotación de tipo en el `let` (el
   build nativo lo exige; los tests VM nunca compilaron ese módulo), y la
   doc de `spawn` aún dice "Requires the VM engine" — otro caso de la nota
   estale "VM only" (funciona nativo, validado aquí).
7. **`std/audio` adoptado el día que salió (M145)** — la música pasó de
   ffplay+scraping a PCM directo, y el pump quedó a la mitad de líneas.
   Medido (`tools/audio_buf.ray`): la cola del dispositivo absorbe **~1.7 s**
   antes de que `write` aparque la fibra — la promesa "la contrapresión ES
   el pacing" vale para el tempo, pero para audio *reactivo* la cola entera
   sería la latencia: sigue haciendo falta el adelanto de reloj de pared
   (~100 ms). La gran mejora es el modo de fallo: tras un hueco el
   dispositivo reproduce lo nuevo al instante (autocura, sin el desfase
   permanente de ffplay). Anotado en `raylang/IDEAS.md` §81: o un tope de
   cola configurable en `open` (latency hint) o un `audio.played_ms(h)`
   para cerrar el lazo sin reloj propio.

## Desarrollo

```sh
ray test
rm -f rallyx && ray build --native src/main.ray -o rallyx --release   # ver hallazgo 1
```

Estructura: `src/main.ray` · `rally.ray` (reglas puras) · `screen.ray`
(cámara + radar + frame/diff) · `app.ray` (bucle de tres relojes).
