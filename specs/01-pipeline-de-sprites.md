# SPEC 01 — Pipeline de sprites

> **Estado:** Aprobado
> **Depende de:** ninguna
> **Fecha:** 2026-10-02
> **Objetivo:** Convertir las cinco hojas de personaje de `assets/` en atlas PNG transparentes con su JSON de frames, verificables en una página de previsualización.

## Por qué existe esta spec

Las hojas de `assets/` son láminas de referencia, no atlas listos para un motor.
Los frames no están en grilla, tienen fondo pintado y llevan etiquetas y líneas divisorias incrustadas.
Medir y limpiar esos frames es el mayor riesgo del MVP.
Por eso queda aislado en una spec propia, verificable sin que exista el juego.

## Alcance

**Entra:**

- Inicializar el repositorio git y el `package.json` del proyecto.
- Servidor estático local sin dependencias (`npm start`).
- Script de Node que recorta frames, quita el fondo y empaqueta un atlas por luchador (`npm run sprites`).
- Definiciones de recorte medidas para los 5 luchadores: Agustín, Casco, Carlos, Marian y Ricar.
- Las 10 animaciones comunes a las 5 hojas: Idle / Guardia, Caminar adelante, Caminar atrás, Agacharse, Salto, Caída / En el suelo, Puño, Patada, Recibir daño, Levantarse.
- El retrato de cada luchador, recortado sin quitar fondo.
- Página de previsualización para revisar recortes y animaciones.
- Tests con `node:test` para la lógica pura de imagen.

**Fuera de alcance (para specs futuras):**

- El juego: combate, input, pantallas. Va en SPEC 02.
- Las tiras "Victoria" (solo Agustín, Casco y Ricar) y "Ataque fuerte" (solo Carlos y Marian).
- Los ataques especiales. En las hojas son ilustraciones únicas, no frames de animación.
- Lectura de las barras de stats.
- Cajas de colisión por frame.

## Modelo de datos

Identificadores de luchador y hoja de origen:

| `id` | Nombre | Hoja |
| --- | --- | --- |
| `agustin` | Agustín | `assets/sprite agus.png` |
| `casco` | Casco | `assets/sprite casco.png` |
| `carlos` | Carlos | `assets/sprite ia.png` |
| `marian` | Marian | `assets/sprite marian.png` |
| `ricar` | Ricar | `assets/sprite ricar.png` |

Claves de animación, iguales en entrada y salida: `idle`, `walkForward`, `walkBack`, `crouch`, `jump`, `knockdown`, `punch`, `kick`, `hurt`, `getUp`.

**Entrada: `tools/sprites/sheets/<id>.json`**, escrita a mano, una por luchador:

```json
{
  "id": "casco",
  "name": "Casco",
  "source": "assets/sprite casco.png",
  "bgTolerance": 48,
  "portrait": { "x": 96, "y": 40, "w": 280, "h": 280 },
  "koFrame": 3,
  "animations": {
    "idle": { "strip": { "x": 400, "y": 50, "w": 370, "h": 120 }, "count": 4 },
    "kick": {
      "bgTolerance": 60,
      "frames": [
        { "x": 1010, "y": 365, "w": 95, "h": 120 },
        { "x": 1110, "y": 365, "w": 100, "h": 120, "ax": 40 }
      ]
    }
  }
}
```

Los números del ejemplo son ilustrativos; los reales se miden durante la implementación.

- Cada animación se define de una de dos formas.
- `strip` + `count`: el script segmenta la tira en frames por columnas vacías. Si el número detectado no es `count`, el build falla.
- `frames`: rectángulos explícitos. Se usa cuando un efecto (agua, fuego, hojas) une dos figuras y la segmentación automática no sirve.
- `bgTolerance` de una animación pisa el del luchador.
- `ax` es opcional: posición horizontal del ancla dentro del frame recortado. Por defecto es el centro.
- `koFrame` es el índice, dentro de `knockdown`, del frame con el luchador tendido en el suelo.

**Salida: `assets/atlas/<id>.png` y `assets/atlas/<id>.json`**, generadas y versionadas en git:

```json
{
  "id": "casco",
  "name": "Casco",
  "image": "casco.png",
  "portrait": { "x": 0, "y": 0, "w": 280, "h": 280 },
  "koFrame": 3,
  "animations": {
    "idle": [
      { "x": 0, "y": 282, "w": 78, "h": 112, "ax": 39, "src": { "x": 412, "y": 56, "w": 78, "h": 112 } }
    ]
  }
}
```

Convenciones:

- Coordenadas en píxeles de la imagen, origen arriba a la izquierda.
- `x`, `y`, `w`, `h` son la posición del frame dentro del atlas, ya recortado a su caja opaca.
- El ancla de un frame es el punto (`ax`, `h`): centro de los pies, sobre el borde inferior. El juego apoya ese punto en el piso.
- `src` es el rectángulo de origen en la hoja. Solo lo usa la previsualización.
- Todos los frames miran a la derecha, igual que en las hojas. El espejado lo hace el juego.
- El retrato conserva su fondo pintado. No se le aplica quitado de fondo.
- El atlas se empaqueta en filas: retrato primero, luego una fila por animación, con 2 px de separación entre frames.

**Quitado de fondo**, aplicado a cada tira o frame antes de recortar:

1. Color de fondo = mediana de los píxeles del borde del rectángulo.
2. Un píxel es "fondo" si su distancia de color al color de fondo es menor que `bgTolerance`.
3. Relleno por inundación de 4 vecinos desde todos los píxeles del borde, avanzando solo por píxeles "fondo". Lo alcanzado queda con alfa 0.
4. Se eliminan las islas opacas de menos de 12 píxeles.

El relleno desde el borde conserva los píxeles oscuros interiores de la figura que no tocan el fondo.

## Plan de implementación

1. Ejecutar `git init`. Crear `.gitignore` con `node_modules/`. Commit inicial con `README.md`, `CLAUDE.md` y `assets/`.
2. Crear `package.json` con `"type": "module"`, `pngjs` como `devDependency` y los scripts `start`, `sprites` y `test` (`node --test`). Crear `tools/serve.js`, servidor estático sin dependencias en el puerto 8080. Prueba manual: `npm start` y abrir `http://localhost:8080/README.md`.
3. Crear `tools/sprites/lib/background.js` con el quitado de fondo como función pura sobre un buffer RGBA. Tests en `test/sprites/background.test.js` con imágenes sintéticas.
4. Crear `tools/sprites/lib/segment.js` con la segmentación por columnas vacías y el recorte a la caja opaca. Tests en `test/sprites/segment.test.js`.
5. Crear `tools/sprites/lib/pack.js` con el empaquetado en filas. Tests en `test/sprites/pack.test.js`.
6. Crear `tools/sprites/build.js`. Procesa cada archivo de `tools/sprites/sheets/` y escribe `assets/atlas/<id>.png` y `.json`. Valida `count`, las 10 claves de animación y la proporción de píxeles opacos de cada frame (entre 15 % y 95 % de su caja). Con la carpeta `sheets/` vacía termina sin error y sin generar nada.
7. Crear `tools/sprites/sheets/casco.json` con todas las medidas. Prueba manual: `npm run sprites` genera el atlas de Casco.
8. Crear `tools/sprites/preview.html`. Tiene selector de luchador y animación, reproducción en bucle a 2x sobre fondo de damero, línea de piso en el ancla y un modo "hoja" que dibuja los rectángulos `src` sobre la hoja original. Lista solo los atlas que existen.
9. Ajustar `casco.json` con la previsualización hasta cumplir los criterios visuales.
10. Crear y ajustar `tools/sprites/sheets/agustin.json`.
11. Crear y ajustar `tools/sprites/sheets/carlos.json`.
12. Crear y ajustar `tools/sprites/sheets/marian.json`.
13. Crear y ajustar `tools/sprites/sheets/ricar.json`.
14. Actualizar `CLAUDE.md` con los comandos (`npm start`, `npm run sprites`, `npm test`, test único con `node --test <archivo>`) y el flujo para ajustar un recorte.

## Criterios de aceptación

- [ ] `git status` funciona en la carpeta y `node_modules/` está ignorado.
- [ ] `package.json` declara `pngjs` como única dependencia, en `devDependencies`.
- [ ] `npm run sprites` termina con código 0 y deja 10 archivos en `assets/atlas/`: un `.png` y un `.json` por cada uno de los 5 `id`.
- [ ] Cada `.json` de atlas contiene las 10 claves de animación, cada una con 2 frames o más, más `portrait` y `koFrame`.
- [ ] `koFrame` es un índice válido de `knockdown` en los 5 atlas.
- [ ] Ejecutar `npm run sprites` dos veces seguidas produce archivos idénticos byte a byte.
- [ ] Cambiar un `count` a un valor incorrecto hace fallar `npm run sprites` con un mensaje que nombra luchador y animación.
- [ ] Las cinco hojas originales de `assets/` no cambian (`git status` no las marca como modificadas).
- [ ] `npm test` pasa.
- [ ] `http://localhost:8080/tools/sprites/preview.html` reproduce las 10 animaciones de los 5 luchadores sin errores en consola.
- [ ] En la previsualización, ningún frame muestra texto de etiqueta, línea divisoria ni restos rectangulares de fondo.
- [ ] En la previsualización, ningún frame de `idle` de los 5 luchadores tiene huecos transparentes dentro de cabeza o torso.
- [ ] En la previsualización, los pies quedan sobre la línea de piso en todos los frames de `idle`, `walkForward` y `walkBack`.
- [ ] En el modo "hoja", ningún rectángulo `src` corta una figura ni incluye parte de la figura vecina.

## Decisiones

- **Sí:** dos specs en lugar de una. La medición de frames es el mayor riesgo y así se verifica sola.
- **Sí:** script de preprocesado. Deja atlas limpios y el juego no carga con la irregularidad de las hojas.
- **No:** recorte en runtime desde la hoja original. Cada frame se vería con su recuadro oscuro.
- **No:** sprites limpios entregados a mano. El pipeline los genera desde las hojas existentes.
- **Sí:** Node + `pngjs`. Mismo lenguaje que el juego, JS puro, sin binarios nativos.
- **No:** `sharp`. Trae binario nativo y no se necesitan sus operaciones.
- **No:** Python + Pillow. Agrega un segundo lenguaje y Pillow no está instalado.
- **Sí:** los 5 luchadores. El pipeline es el mismo y la selección de personaje del juego lo necesita.
- **Sí:** segmentación automática con `count` obligatorio. Reduce la medición manual de unos 40 rectángulos a 10 tiras por hoja, y `count` detecta segmentaciones erróneas.
- **Sí:** rectángulos explícitos como alternativa por animación. Los efectos de ataque unen figuras en varias tiras.
- **Sí:** fondo por distancia de color a la mediana del borde, no por luminancia. Marian y Casco visten ropa oscura que un umbral de brillo borraría.
- **Sí:** atlas generados versionados en git. El juego corre tras clonar sin ejecutar el build.
- **Sí:** solo las 10 animaciones comunes. "Victoria" y "Ataque fuerte" no existen en las 5 hojas.
- **Sí:** servidor estático propio en `tools/serve.js`. Los módulos ES y `fetch` exigen HTTP y así no se agrega otra dependencia.
- **Sí:** tests junto a `test/sprites/`. SPEC 02 usa `test/logic/` y `node --test` recoge ambos.

## Riesgos

| Riesgo | Mitigación |
| --- | --- |
| El quitado de fondo borra ropa o pelo oscuros, o deja halo. | `bgTolerance` ajustable por luchador y por animación. La validación de proporción opaca del build detecta los casos graves. Si un luchador no se puede limpiar, se vuelve al usuario antes de cambiar de técnica. |
| Efectos de ataque unen dos figuras y rompen la segmentación. | `count` hace fallar el build. Esa animación pasa a `frames` explícitos. |
| Los fondos pintados no son uniformes dentro de una tira. | El color de fondo se calcula por tira, no por hoja. Una tira problemática se divide en `frames` explícitos, cada uno con su propia mediana. |
| Las tiras de las 5 hojas tienen distinta cantidad de frames. | El juego de SPEC 02 reparte la duración de cada estado entre los frames disponibles. No se exige un número fijo. |
| La pose "tendido en el suelo" cae en distinto índice según la hoja. | `koFrame` por luchador. |

## Qué **no** entra en esta spec

- El juego jugable (SPEC 02).
- Tiras "Victoria" y "Ataque fuerte".
- Ataques especiales.
- Stats de los luchadores.
- Cajas de colisión por frame.

Cada una, si llega, va en su propia spec.
