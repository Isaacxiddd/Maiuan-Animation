# Maiuan Cinematic Reel — Documentación

Archivo principal: `maiuan_reel_v10.html`

---

## 1. Estructura general

El reel es una página HTML de una sola pantalla (`100vw × 100vh`) que reproduce una animación cinematográfica de ~32 segundos dividida en 8 fases.

### Capas visuales (de atrás hacia adelante)

| Capa | Elemento | Líneas aprox. | Descripción |
|------|----------|---------------|-------------|
| Fondo | `<canvas id="mainCanvas">` | Línea ~22 | Renderiza partículas, conexiones y el logo en canvas |
| Overlays DOM | `<div class="overlay">` | Líneas ~24-44 | Textos del manifiesto y logo final |
| Decoración | `.vignette`, `.progress-bar` | Líneas ~46-50 | Viñeta oscura y barra de progreso |

---

## 2. Fases de la animación (Timeline)

Las fases se definen en el array `PHASES` (línea ~260 del JS):

```javascript
// Línea ~260
const PHASES = [
  { name: 'chaos',      duration: 3000 },  // 0-3s   — Caos de partículas
  { name: 'converge',   duration: 3000 },  // 3-6s   — Convergencia a red
  { name: 'compress',   duration: 3000 },  // 6-9s   — Compresión al centro
  { name: 'build',      duration: 4000 },  // 9-13s  — Construcción del logo wireframe
  { name: 'reveal',     duration: 3000 },  // 13-16s — Solidificación del logo
  { name: 'morph',      duration: 3000 },  // 16-19s — Morphing del logo
  { name: 'manifesto',  duration: 6000 },  // 19-25s — Palabras del manifiesto
  { name: 'final',      duration: 7000 },  // 25-32s — Logo DOM final
];
```

**Para cambiar la duración total:** modifica los valores `duration` (en milisegundos).

---

## 3. Sistema de partículas

### 3.1 Cantidad de partículas

```javascript
// Línea ~290
const PARTICLE_COUNT = 120;
```

- **120** = fluidez optimizada para la mayoría de dispositivos
- **Aumentar** → más denso pero puede laggear en móviles
- **Disminuir** → más ligero pero menos impacto visual

### 3.2 Comportamiento de las partículas

La clase `Particle` (línea ~295) tiene 6 modos de movimiento controlados por el parámetro `mode`:

```javascript
// Línea ~310-350 (dentro de update())
if (mode === 0) {      // CHAOS — movimiento libre en esfera 3D
  this.x += this.vx; this.y += this.vy; this.z += this.vz;
}
else if (mode === 1) { // CONVERGE — atracción a posiciones de red
  this.vx += (this.targetX - this.x) * attract;
}
else if (mode === 2) { // COMPRESS — colapso al centro (0,0,0)
  this.vx += (0 - this.x) * collapse;
}
else if (mode === 3) { // BUILD — snap a posiciones del logo
  this.vx += (this.targetX - this.x) * snap;
}
else if (mode === 5) { // MORPH — movimiento orgánico suave
  this.x += this.vx * 0.2;
}
else if (mode === 6) { // FADE OUT — desaparición gradual
  this.alpha -= 0.02;
}
```

### 3.3 Velocidad de atracción (convergencia)

```javascript
// Línea ~322
const attract = 0.025 * t;
```

- **Aumentar** (ej: `0.05`) → partículas convergen más rápido
- **Disminuir** (ej: `0.01`) → convergencia más lenta y dramática

### 3.4 Velocidad de colapso (compresión)

```javascript
// Línea ~332
const collapse = 0.05 * t;
```

- **Aumentar** → colapso más violento
- **Disminuir** → colapso más suave y elegante

---

## 4. El logo — Extracción del SVG

### 4.1 SVG fuente

El logo se extrae de un SVG oculto en el DOM (líneas ~52-60):

```html
<!-- Línea ~52 -->
<svg id="hiddenSvg" style="position:absolute;width:0;height:0;" viewBox="0 0 345 345">
  <g id="faviconGroup" transform="translate(0,207) scale(0.1,-0.1)">
    <path id="favPath1" d="M1520 1400 c113 -142..."/>  <!-- Montaña principal -->
    <path id="favPath2" d="M1553 577 l138 -162..."/>     <!-- Triángulo izquierdo -->
    <path id="favPath3" d="M2203 577 c70 -87..."/>      <!-- Triángulo derecho -->
  </g>
</svg>
```

**Para cambiar el logo:** reemplaza los `d="..."` de los 3 `<path>` con los paths de tu nuevo SVG.

### 4.2 Extracción de puntos

```javascript
// Línea ~155 — función extractLogoPoints()
function extractLogoPoints() {
  const svg = document.getElementById('hiddenSvg');
  const paths = svg.querySelectorAll('path');

  // Transformación del grupo original: translate(0,207) scale(0.1,-0.1)
  const sx = 0.1, sy = -0.1;
  const tx = 0, ty = 207;

  paths.forEach(path => {
    const len = path.getTotalLength();
    const samples = Math.max(80, Math.floor(len / 4));  // Densidad de sampleo
    // ...
  });
}
```

**Para ajustar la densidad del contorno:** modifica `len / 4`:
- `len / 2` → más puntos, contorno más suave (más pesado)
- `len / 8` → menos puntos, contorno más angular (más ligero)

### 4.3 Orientación (volteo 180°)

```javascript
// Línea ~195 — INVERTIR Y para corregir orientación
logoPaths = logoPaths.map(pathPoints => 
  pathPoints.map(p => ({
    x: (p.x - cx) / scale,
    y: -(p.y - cy) / scale,  // <-- El signo - voltea 180° (W → M)
    z: 0
  }))
);
```

**Si tu logo aparece al revés:** cambia `-(p.y - cy)` a `(p.y - cy)` (o viceversa).

---

## 5. Renderizado del logo en canvas

### 5.1 Dibujo progresivo (fase BUILD)

```javascript
// Línea ~430 — función drawLogoProgressive()
function drawLogoProgressive(ctx, progress, scale) {
  // progress: 0→1, controla cuánto del logo está dibujado
  // scale: tamaño del logo en píxeles (ej: 0.35 × min(width,height))

  const pathProgress = 1 / logoPaths.length;  // 3 paths = 33% cada uno

  logoPaths.forEach((path, pathIdx) => {
    const pathStart = pathIdx * pathProgress;
    const pathEnd = pathStart + pathProgress;
    const localProgress = Math.max(0, Math.min(1, 
      (progress - pathStart) / (pathEnd - pathStart)
    ));
    // ... dibuja segmentos progresivamente
  });
}
```

**Para cambiar el orden de construcción:** modifica el orden de `logoPaths.forEach()`.

### 5.2 Dibujo sólido (fase REVEAL)

```javascript
// Línea ~390 — función drawLogoExact()
function drawLogoExact(ctx, scale, alpha, fillAlpha, lineWidth) {
  // alpha: opacidad del borde (0-1)
  // fillAlpha: opacidad del relleno (0-1)
  // lineWidth: grosor del borde en píxeles

  ctx.fillStyle = `rgba(232, 255, 0, ${fillAlpha})`;  // Relleno amarillo
  ctx.strokeStyle = `rgba(232, 255, 0, ${alpha})`;    // Borde amarillo
}
```

**Para cambiar el color:** reemplaza `232, 255, 0` con tu RGB.

**Para hacer el relleno más opaco:** aumenta `fillAlpha` (ej: `0.25` en vez de `0.12`).

### 5.3 Tamaño del logo

```javascript
// Línea ~555 — dentro del render loop
const logoScale = Math.min(W, H) * 0.35;
```

- **0.35** = 35% del tamaño menor de la pantalla
- **0.5** = mitad de la pantalla (más grande)
- **0.2** = 20% de la pantalla (más pequeño)

---

## 6. Manifiesto (fase 6)

### 6.1 Palabras

```javascript
// Línea ~475
const manifestoTexts = ['UTILIDAD','PRECISIÓN','EVOLUCIÓN','IMPACTO','SISTEMAS'];
```

**Para cambiar las palabras:** edita este array.

### 6.2 Timing de las palabras

```javascript
// Línea ~650 — dentro de la fase manifesto
const wordInterval = phase.duration / 5;  // 6000ms / 5 = 1200ms por palabra
```

**Para que cada palabra dure más:** aumenta `phase.duration` en el array `PHASES`.

### 6.3 Estilo de las palabras

```css
/* Línea ~35 */
.word-manifesto {
  font-family: 'Big Shoulders Display', sans-serif;
  font-weight: 900;
  font-size: clamp(48px, 10vw, 120px);  /* Tamaño responsive */
  color: #e8ff00;                        /* Amarillo Maiuan */
}
```

**Para cambiar el color:** modifica `color: #e8ff00`.
**Para cambiar la tipografía:** modifica `font-family`.

---

## 7. Logo final (fase 8)

### 7.1 SVG del mark

```html
<!-- Línea ~72 -->
<div class="final-mark">
  <svg width="148" height="89" viewBox="20 55 325 120">
    <g transform="translate(0,207) scale(0.1,-0.1)" fill="#e8ff00">
      <path d="M1520 1400..."/>  <!-- Mismo path que el favicon -->
      <path d="M1553 577..."/>   <!-- Triángulo izq -->
      <path d="M2203 577..."/>   <!-- Triángulo der -->
    </g>
  </svg>
</div>
```

**Para cambiar el mark:** reemplaza los 3 `<path d="...">`.

### 7.2 Texto "MAIUAN" con animación U↔I

```html
<!-- Línea ~82 -->
<div class="final-word-row" id="finalWordRow">
  <span class="final-L final-L-m">M</span>
  <span class="final-L final-L-a1">A</span>
  <span class="final-L final-L-i">I</span>  <!-- Se anima desde la derecha -->
  <span class="final-L final-L-u">U</span>  <!-- Se anima desde la izquierda -->
  <span class="final-L final-L-a2">A</span>
  <span class="final-L final-L-n">N</span>
</div>
```

### 7.3 Cálculo del swap U↔I

```javascript
// Línea ~700 — función setSwapOffsets()
function setSwapOffsets() {
  const elU = document.querySelector('.final-L-u');
  const elI = document.querySelector('.final-L-i');
  const row = document.getElementById('finalWordRow');
  const gap = parseFloat(getComputedStyle(row).gap) || 0;
  const wU = elU.getBoundingClientRect().width;
  const wI = elI.getBoundingClientRect().width;

  // La U se mueve a la izquierda exactamente el ancho de la I + gap
  document.documentElement.style.setProperty('--u-to-i', `-${wI + gap}px`);
  document.documentElement.style.setProperty('--i-to-u', `${wU + gap}px`);
}
```

**Si el swap se ve desalineado:** asegúrate de que la fuente esté cargada antes de llamar `setSwapOffsets()`. La función se llama en línea ~710 y también en el evento `resize`.

---

## 8. Tagline y subtítulo final

```html
<!-- Línea ~90 -->
<div class="final-tagline">Construimos software útil, simple y bien ejecutado.</div>
<div class="final-sub">SOFTWARE QUE RESUELVE</div>
```

**Para cambiar el texto:** edita directamente el contenido HTML.

---

## 9. Proyección 3D

```javascript
// Línea ~270
const FOCAL = 600;

function project(x, y, z) {
  const scale = FOCAL / (FOCAL + z);
  return { x: CX + x * scale, y: CY + y * scale, scale: scale };
}
```

- **FOCAL = 600** → perspectiva suave, poco efecto 3D
- **FOCAL = 200** → perspectiva fuerte, mucho efecto 3D
- **FOCAL = 1000** → casi sin perspectiva, casi ortográfico

---

## 10. Colores

### 10.1 Paleta principal

| Elemento | Color | Dónde se usa |
|----------|-------|-------------|
| Amarillo Maiuan | `#e8ff00` / `rgb(232,255,0)` | Logo, partículas, texto manifiesto, acentos |
| Negro fondo | `#000` | Todo el fondo |
| Blanco texto | `#f0f0f0` | Textos secundarios |
| Gris tenue | `rgba(255,255,255,0.45)` | Tagline |

### 10.2 Cambiar el color de las partículas

Busca todas las ocurrencias de `rgba(232, 255, 0` en el código y reemplaza con tu color RGB.

Ejemplo para cambiar a rojo:
```javascript
// Antes
ctx.fillStyle = `rgba(232, 255, 0, ${alpha})`;
// Después
ctx.fillStyle = `rgba(255, 50, 50, ${alpha})`;
```

---

## 11. Grabar el reel

1. Abre el archivo en Chrome/Firefox/Safari
2. Presiona `F11` para pantalla completa
3. Usa OBS, QuickTime (Mac) o Xbox Game Bar (Win+G) para grabar
4. La animación dura exactamente la suma de `PHASES.duration` (32 segundos por defecto)
5. Presiona `Space` para reiniciar la animación en cualquier momento

---

## 12. Controles de teclado

| Tecla | Acción | Línea |
|-------|--------|-------|
| `Space` | Reiniciar animación | ~720 |
| `F11` | Pantalla completa | Navegador |

---

## 13. Optimización / Performance

Si se traba en tu dispositivo:

```javascript
// Línea ~290 — reducir partículas
const PARTICLE_COUNT = 120;  // Bajar a 80 o 60

// Línea ~162 — reducir densidad de sampleo del logo
const samples = Math.max(80, Math.floor(len / 4));  // Cambiar a len / 8

// Línea ~360 — reducir conexiones (skip cada 3ra partícula)
for (let i = 0; i < particles.length; i += 3) {  // Era i += 2
```

---

## 14. Glosario de funciones clave

| Función | Línea | Qué hace |
|---------|-------|----------|
| `extractLogoPoints()` | ~155 | Extrae y normaliza los puntos del SVG oculto |
| `drawLogoExact()` | ~390 | Dibuja el logo sólido (fill + stroke) |
| `drawLogoProgressive()` | ~430 | Dibuja el logo wireframe progresivamente |
| `drawConnections()` | ~360 | Dibuja líneas entre partículas cercanas |
| `project()` | ~270 | Proyección 3D a 2D con perspectiva |
| `render()` | ~540 | Loop principal de animación (requestAnimationFrame) |
| `setSwapOffsets()` | ~700 | Calcula el desplazamiento U↔I del logo final |

---

## 15. Dependencias

- **Google Fonts**: `Big Shoulders Display` (títulos), `Inter` (cuerpo)
- **Sin frameworks**: Vanilla HTML/CSS/JS, sin React, Vue, ni jQuery
- **Sin librerías externas**: Todo el canvas es código nativo

---

*Última versión: v10 — 18 Jun 2026*
