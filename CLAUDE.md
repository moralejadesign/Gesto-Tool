# Haya (nombre provisional)

Herramienta web para posar maniquíes de dibujo de madera en 3D y usarlos como referencia de pose, perspectiva y luz. Hay dos módulos en la misma página: **Mano** y **Cuerpo** (maniquí completo articulado), con pestañas arriba del panel. `#cuerpo` en la URL abre directo el Cuerpo. La herramienta es para artistas e ilustradores: cada decisión prioriza que la referencia sea útil para dibujar.

## Estado actual

- Un solo archivo: `index.html` (HTML + CSS + JS inline, sin build).
- Three.js r147 en build UMD desde jsDelivr, más `OrbitControls` desde `examples/js` (la última versión con UMD es r147; desde r148 hay que usar módulos ES).
- Para correrlo basta abrir `index.html` o servir la carpeta: `npx serve .`

## Módulos

Cada módulo es un objeto en `MODULES` (`hand`, `body`) con su `root` en la escena, su panel (`#mod-hand`, `#mod-body`), `pose`, `presets`, `apply()`, `exportPose()`, `details` (acordeones por parte) y su encuadre (`target`, `dir`, `dist`, `views`, `shadow`). `M` es el módulo activo y `setModule()` cambia entre ellos guardando la cámara de cada uno. Lo compartido usa `M`: `applyLens()` (con `M.dist` como distancia base), `setView()`, `tweenPose()`, `resetModule()`, `copyPose()`, el clic y el hover (`partAt()`, `openPart()`; cada malla lleva `userData.part`).

## Arquitectura de la mano

La mano es un componente: `buildHand(parent)` devuelve `{wrist, rig}` y `applyHand(h, fingers, wrist)` aplica la pose. El módulo Mano la monta sobre su base y el Cuerpo la monta en cada muñeca.

### Jerarquía (scene graph)
```
root del módulo Mano (scale.x = ±1 para mano izquierda/derecha)
├─ base               (torno cónico + anillo de muñeca, fijo)
└─ wrist              (rotation.order 'YXZ': twist, flex, dev)
   ├─ wristBall, palm (palma = LatheGeometry con scale.z 0.48)
   ├─ <dedo>.root     (rotation.order 'ZXY': spread en z, flexión MCP en x)
   │   └─ joint PIP → joint DIP   (flexión en x)
   └─ thumb.root      (z = abrir, x = hacia palma)
       └─ roll        (y = rotar el pulgar sobre su eje)
           └─ joint MCP → joint IP
```
Cada joint contiene una bisagra (`hinge()`, cilindro horizontal) y su falange (`phalanxGeo()`, lathe con cantos redondeados; la última termina en domo).

### Convenciones de ejes
- +Y va a lo largo de los dedos, +Z es el lado de la palma y el pulgar está en +X.
- Flexionar es rotación positiva en X: el dedo se curva hacia la palma.
- Separar: índice y medio usan signo negativo en Z y anular y meñique positivo, así que un `spr` positivo siempre abre hacia afuera.
- Con palma en +Z y pulgar en +X, el modelo sin espejar es una mano **derecha** (vista de frente se ve la palma y el pulgar queda a la derecha del espectador). "Derecha" deja `scale.x = 1` y "Izquierda" espeja con `scale.x = -1`. La mano arranca como derecha.

### Estado de pose
Todo el estado está en grados, en `M.pose`. `M.apply()` (que para la mano llama a `applyHand()`) es lo único que traduce el estado a rotaciones de Three.js. No rotes los huesos directamente desde la UI.
```js
{
  wrist:  { flex, dev, twist },
  thumb:  { abd, fwd, roll, mcp, ip },
  index:  { mcp, pip, dip, spr },   // igual para middle, ring, pinky
}
```
Los rangos de cada articulación están definidos en los sliders y son los límites anatómicos. Si cambias uno, revisa que ninguna falange se doble al revés.

### Presets
`HAND_PRESETS` guarda poses completas (Abierta, Relajada, Puño, Señalar, Paz, OK, Garra, Pinza). `M.goPreset()` (vía `tweenPose()`) interpola de la pose actual a la nueva en 450 ms (en 0 ms si el usuario tiene `prefers-reduced-motion`). "Copiar pose" exporta el mismo esquema en JSON, así que cualquier pose copiada se puede pegar como preset nuevo.

## Arquitectura del cuerpo

- Unidades: 1 = una cabeza. Canon de 8 cabezas (176 cm, `CM = 22` cm por cabeza). El chip 7.5 escala solo la cabeza (`8/canon`).
- Soporte de taller: disco + varilla que entra en la pelvis. La figura cuelga de la pelvis, así que "Altura" (cm) sube o baja el cuerpo sobre la varilla y los pies no se apoyan solos.
- El maniquí mira a +Z y su izquierda está en +X. Brazos y piernas se construyen del lado derecho (-X); el izquierdo es el mismo grupo espejado (`scale.x = -1`), así que los mismos valores dan movimientos simétricos.
```
figure
└─ pelvis              ('YXZ'; altura, doblar, inclinar, girar)
   ├─ waist → chest    (el torso reparte la flexión 50/50)
   │   ├─ neck → head  (40/60)
   │   └─ sides.R/L → shoulder ('ZXY') → elbow → mount → buildHand()
   └─ legSides.R/L → hip ('ZXY') → knee → ankle → pie
```
- El mount de la mano usa un giro de 180° sobre (1,0,1): dedos hacia abajo, palma hacia el muslo y pulgar adelante. La mano va a escala 0.22.
- Pose del cuerpo: `hips, torso, head, armL/R, wristL/R, handL/R (dedos), legL/R`. Los presets (`BODY_PRESETS`) se escriben con `bodyPose({...})`, que solo declara lo que cambia respecto a `BASE`; una mano se puede nombrar por su preset (`handR:'Puño'`).
- `tweenPose()` interpola objetos anidados, por eso las manos del cuerpo se animan junto con el resto.

### Madera
La textura es procedural (`makeWoodCanvas`): ruido 3D de valor con fbm, muestreado en coordenadas cilíndricas para que no haya costura alrededor del torno. Hay 4 variantes de veta y cada pieza clona una con offset aleatorio, para que parezcan talladas por separado. Las bisagras llevan un tinte un poco más oscuro (`pinTint`). Material: `MeshStandardMaterial` con `bumpMap` sobre la misma textura y roughness 0.62.

### Cámara y luz
- El lente va en mm sobre sensor full frame: `fov = 2·atan(12/mm)`.
- `applyLens()` mueve la cámara a `M.dist · mm/50` (15 para la mano, 22 para el cuerpo) para mantener el encuadre. Así, cambiar el lente cambia la perspectiva (escorzo) sin cambiar el tamaño. Esta función es central en la herramienta y no debe romperse.
- Vistas predefinidas en `M.views` (dirección normalizada desde el objetivo). El cuerpo usa vistas más bajas, a la altura de los ojos.
- Luz principal direccional con sombras PCF suaves, controlada por azimut y altura. Además hay una hemisférica y un relleno frío.

### UI
- El panel lateral se arma con `slider(container, id, label, min, max, get, set, unit)`. Cada slider lee y escribe el estado mediante closures, y `syncAll()` refresca todos después de un cambio global o un preset.
- Los colores son tokens CSS en `:root` y tienen variante oscura (`prefers-color-scheme` y `[data-theme]`). El fondo de la escena 3D lee `--stage`.
- Tipografía: Bricolage Grotesque para la interfaz e IBM Plex Mono para los valores numéricos.

## Convenciones

- La interfaz y el copy van en español. Los nombres en el código van en inglés.
- No usar guiones largos (em dash) en ningún texto.
- Copy corto y concreto. Los controles dicen lo que hacen.
- Las unidades reales son parte del contenido: grados en las articulaciones, mm en el lente.
- Cada cambio visual se verifica con un screenshot (Playwright + Chromium) desde Frente, Tres cuartos y Perfil, en al menos una pose abierta y una cerrada.

## Problemas conocidos

- **Pulgar:** los valores de Puño, Señalar y Paz se ajustaron sin una revisión visual final. Hay que validarlos y probablemente ajustar la conversión de `roll` y `fwd` en `applyHand()` para que el pulgar cruce sobre los dedos.
- La palma es un torno aplanado y le falta la concavidad de la palma real.
- Las falanges pueden atravesarse entre sí en flexiones extremas. Todavía no hay detección de colisiones.
- **Cuerpo:** tampoco hay colisiones (en Cuclillas y Correr los antebrazos pueden rozar el torso o los muslos). Los presets del cuerpo se revisaron visualmente pero son un primer ajuste.

## Roadmap

1. Validar y afinar el pulgar.
2. Exportar PNG (local sí funciona, porque `preserveDrawingBuffer` ya está activo) y guardar o cargar poses JSON.
3. ~~**Módulo Cuerpo**~~ (hecho: canon 8/7.5, límites por articulación, mano reutilizada). Siguiente: hombros con clavícula, columna con más segmentos y pies que se apoyen en el piso.
4. Cinemática inversa (IK): arrastrar la mano o el pie y que la cadena se acomode sola.
5. Decidir el stack para crecer: o sigue como HTML único o migra a Vite + React Three Fiber + Leva. Si se migra, separar `rig/`, `materials/`, `poses/` y `ui/`.
6. Definir el nombre final y la identidad. Opciones evaluadas: Haya, Bisagra, Escorzo, Gesto, Tilo.
