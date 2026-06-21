# juegos-sobrinos

Aplicación educativa de juegos para niños de 4, 6 y 10 años. Corre en el navegador del TV Android como un solo archivo HTML sin dependencias ni build step.

## Contexto del proyecto

- **Target principal:** niños de 4 y 6 años. El de 10 es usuario ocasional.
- **Dispositivo:** TV Android 75". Navegación exclusiva con control remoto (flechas + OK).
- **Idioma:** español. TTS en español via Web Speech API.
- **Restricción crítica:** un solo archivo `.html` autocontenido. Sin npm, sin bundler, sin frameworks.

## Archivo

```
juegos-sobrinos.html   ← todo el proyecto vive aquí
```

## Arquitectura

### Flujo de pantallas

```
screen-avatar → screen-diff → screen-menu → screen-{juego}
                    ↑                              ↓
                 goHome()                       goMenu()
```

### State global

```js
const state = {
  player: null,       // 'dino' | 'cohete' | 'leon' | 'estrella' | 'robot'
  diff: null,         // 'facil' | 'normal' | 'dificil'
  score: 0,
  round: 0,
  totalRounds: 5,
  focusIdx: 0,        // índice del elemento enfocado en la pantalla actual
  screen: 'screen-avatar',
  answering: false,   // bloquea input mientras se procesa una respuesta
};
```

### Navegación con control remoto

Toda la navegación pasa por `navItems(containerId, selector, e, mode, onSelect)`.

- **mode `'h'`** — navegación horizontal (flechas izq/der). Usado en la mayoría de pantallas.
- **mode `'g2'`** — grid 2 columnas (4 direcciones). Usado solo en juego Intruso.

El foco se marca con clase `.focused` en el elemento activo. `state.focusIdx` siempre refleja el índice del elemento con `.focused`.

**Regla crítica:** nunca modificar el DOM de opciones mientras `state.answering === true`.

### TTS

```js
speak(text, rate=0.85)  // cancela utterance anterior antes de hablar
speakLetter(letter)     // repite la letra dos veces, más lento (rate=0.75)
```

La voz preferida es la local en español (`v.localService && v.lang.startsWith('es')`). En TV Android puede no haber voz local — el navegador usa la del sistema.

### CSS tokens

```css
--bg: #1a1a2e        /* fondo dark */
--panel: #16213e     /* cards y panels */
--accent1: #f5a623   /* naranja — foco principal */
--accent2: #7ed321   /* verde — correcto */
--accent3: #e74c8b   /* rosa — incorrecto */
--accent4: #4ecdc4   /* teal — foco Intruso/Planetas */
--radius: 24px
```

Tamaños con `clamp()` en todo — escalan de móvil a TV sin media queries.

## Juegos

### Cómo agregar un juego nuevo

1. Agregar botón en `#game-menu`:
```html
<div class="game-btn" data-game="nombre" tabindex="0">
  <span class="g-emoji">🎯</span><span class="g-name">Título</span>
</div>
```

2. Agregar screen HTML (copiar estructura de cualquier screen existente):
```html
<div class="screen" id="screen-nombre">
  <div class="player-badge" id="nombre-badge"></div>
  <div class="score-badge" id="nombre-score">⭐ 0</div>
  ...
  <button class="nav-btn home-btn" id="nombre-home">🏠 Inicio</button>
  <button class="nav-btn back-btn" id="nombre-back">↩ Menú</button>
</div>
```

3. Registrar nav buttons:
```js
// En el forEach existente:
['contar','letras','intruso','planetas','nombre'].forEach(g => { ... })
```

4. Agregar case en el keyboard handler y en `handleGamePick`.

5. Implementar `startNombre()`, `renderNombre()`, `handleNombreAnswer()` siguiendo el patrón de los juegos existentes.

### Juegos actuales

#### 🍎 Contar (`screen-contar`)
- Muestra N emojis repetidos, el niño elige el número correcto entre 3 opciones.
- Dificultad: fácil=hasta 5, normal=hasta 10, difícil=hasta 20.
- Data generada en runtime, sin repetir números en la misma sesión.

#### 🔤 Letras (`screen-letras`)
- Alterna rondas pares/impares:
  - **Par (audio):** suena la letra, el niño elige cuál es. Muestra 🔊.
  - **Impar (imagen):** muestra emoji grande + `_palabra` (primera letra oculta), el niño elige con qué letra empieza.
- Pool de letras por dificultad: fácil=vocales, normal=vocales+consonantes básicas, difícil=alfabeto completo.
- `EMOJI_PALABRAS` tiene 30 entradas por nivel con emoji, palabra y letra inicial.
- **Bug conocido resuelto:** nunca mostrar la letra en el display cuando se pregunta qué letra es.

#### 🔍 Intruso (`screen-intruso`)
- 4 emojis en grid 2x2, 3 del mismo grupo y 1 que no pertenece.
- Navegación en modo `'g2'` (4 direcciones).
- `INTRUSO_DATA`: array de `{ items: [...], intruso: indexDelIntrusoEnItems }`.

#### 🪐 Planetas (`screen-planetas`)
- Imagen real de NASA + pista hablada, el niño elige el nombre entre 3 opciones.
- Imágenes verificadas en `images-assets.nasa.gov` — requieren internet.
- Cada planeta tiene 3 pistas rotativas, se elige una aleatoriamente por ronda.
- `PLANETAS_DATA`: 8 planetas con `{ nombre, img, pistas: [] }`.
- Feedback al fallar: repite la pista hablada.

## Patrones de código

### Estructura de un render de ronda

```js
function renderJuego() {
  if (state.round >= state.totalRounds) { endGame(); return; }

  // 1. UI de progreso
  renderDots('juego-dots', state.totalRounds, state.round);
  document.getElementById('juego-score').textContent = `⭐ ${state.score}`;

  // 2. Contenido de la pregunta
  // ...

  // 3. Generar opciones (siempre shuffle)
  const opts = shuffle([correct, ...wrong]);
  const cont = document.getElementById('juego-options');
  cont.innerHTML = '';
  opts.forEach((opt, i) => {
    const b = document.createElement('div');
    b.className = 'option-btn' + (i === 0 ? ' focused' : '');
    // ...
    cont.appendChild(b);
  });

  // 4. Reset foco y estado
  state.focusIdx = 0;
  state.answering = false;

  // 5. TTS después de render
  setTimeout(() => speak('...'), 300);
}
```

### Estructura de un handler de respuesta

```js
function handleAnswer(btn, correct) {
  if (state.answering) return;   // guard obligatorio
  state.answering = true;

  const ok = /* comparación */;
  btn.classList.add(ok ? 'correct' : 'wrong');

  if (ok) {
    state.score++;
    showFB(true);
    speak(getPositive());
  } else {
    // marcar la opción correcta
    showFB(false);
    speak('feedback de error');
  }

  setTimeout(() => { hideFB(); state.round++; renderJuego(); }, 1800);
}
```

### Helpers disponibles

```js
shuffle(arr)                    // Fisher-Yates, no muta el original
genOpts(correct, count, min, max) // genera N opciones numéricas con la correcta incluida
renderDots(id, total, current)  // pinta puntos de progreso
getPositive()                   // frase aleatoria de éxito en español
diffMax()                       // retorna 5 | 10 | 20 según state.diff
showFB(ok, emoji?, text?)       // muestra overlay de feedback
hideFB()                        // oculta overlay
speak(text, rate?)              // TTS español
speakLetter(letter)             // TTS letra x2, lento
goHome()                        // → screen-avatar
goMenu()                        // → screen-menu (mantiene player y diff)
```

## Ideas pendientes / backlog

- [ ] Pantalla de resultados final más festiva (confetti, música)
- [ ] Sonidos adicionales (no solo voz) — beep correcto, buzz incorrecto
- [ ] Modo "Repite la letra" — botón OK sobre 🔊 repite el audio (hint bar lo indica pero no está implementado)
- [ ] Más datos en `INTRUSO_DATA` (actualmente 12 sets)
- [ ] Pistas adicionales por planeta en `PLANETAS_DATA`
- [ ] Guardar highscore en localStorage por jugador
- [ ] Juego nuevo: "Ordena los números" — poner secuencia en orden
- [ ] Juego nuevo: "¿Qué falta?" — secuencia de colores/figuras con un hueco

## Notas TV Android

- El navegador del TV es Chromium-based — Web Speech API funciona.
- Si la voz TTS no carga al inicio, es porque `onvoiceschanged` no disparó. Workaround: el usuario hace una interacción y luego funciona.
- Las imágenes de NASA en Planetas requieren internet. Si hay conexión lenta, agregar un spinner antes del `img.onload`.
- El control remoto envía `ArrowLeft/Right/Up/Down` y `Enter`. `Backspace` puede no funcionar en todos los TV — considerar botón visual alternativo si hay problemas.
- Probar siempre en el TV real antes de las vacaciones — el renderizado de `clamp()` en pantallas 4K puede variar.