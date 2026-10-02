# SPEC 02 — Combate MVP para 2 jugadores

> **Estado:** Borrador
> **Depende de:** SPEC 01
> **Fecha:** 2026-10-02
> **Objetivo:** Un juego de pelea en navegador donde dos jugadores en el mismo teclado eligen entre los 5 luchadores y combaten al mejor de 3 rondas con movimientos básicos.

## Alcance

**Entra:**

- Juego en HTML5 Canvas 2D con JavaScript vanilla y módulos ES, sin build ni dependencias de runtime.
- Canvas lógico de 960x540, escalado a la ventana manteniendo proporción y sin suavizado.
- Cuatro pantallas: título, selección de luchador, combate y resultado.
- Los 5 luchadores jugables, con los atlas de SPEC 01.
- Dos jugadores locales en un teclado.
- Movimientos: caminar adelante y atrás, agacharse, saltar, puño, patada, recibir daño, caída y levantarse.
- Combate al mejor de 3 rondas con reloj de 99 segundos.
- HUD: barras de vida, nombres, marcadores de rondas ganadas y reloj.
- Pausa durante el combate.
- Un único escenario dibujado por código.
- Tests con `node:test` para la lógica pura.

**Fuera de alcance (para specs futuras):**

- Rival controlado por CPU.
- Bloqueo.
- Ataques especiales e inputs de dirección.
- Ataques agachado y ataques aéreos. Las hojas no tienen frames para ellos.
- Stats distintos por luchador. Los 5 tienen la misma vida, daño y velocidad.
- Sonido y música.
- Poses de victoria y "Ataque fuerte".
- Escenarios con imagen, más de un escenario.
- Gamepad, controles táctiles, reasignación de teclas.
- Persistencia de cualquier tipo.
- Juego en red.

## Modelo de datos

Constantes en `src/config.js`:

```js
export const WIDTH = 960;
export const HEIGHT = 540;
export const FLOOR_Y = 470;        // y del piso en el canvas
export const STAGE_LEFT = 60;
export const STAGE_RIGHT = 900;
export const SPRITE_SCALE = 2;
export const STEP = 1 / 60;        // paso fijo de simulación, en segundos

export const MAX_HEALTH = 100;
export const WALK_FORWARD_SPEED = 240;
export const WALK_BACK_SPEED = 180;
export const JUMP_VELOCITY = 900;
export const GRAVITY = 2400;
export const PUSHBOX_WIDTH = 70;

export const ATTACKS = {
  punch: { startup: 0.10, active: 0.10, recovery: 0.15, damage: 8,
           box: { x: 35, y: 150, w: 90, h: 40 } },
  kick:  { startup: 0.15, active: 0.12, recovery: 0.23, damage: 12,
           box: { x: 35, y: 90, w: 110, h: 60 } },
};
export const HURTBOXES = {
  stand:  { w: 70, h: 220 },
  crouch: { w: 80, h: 140 },
  air:    { w: 70, h: 180 },
};
export const HURT_TIME = 0.30;
export const HURT_PUSHBACK = 40;
export const KNOCKDOWN_TIME = 0.80;
export const GETUP_TIME = 0.50;

export const ROUND_TIME = 99;
export const ROUNDS_TO_WIN = 2;
export const INTRO_TIME = 1.5;
export const KO_TIME = 2.0;
```

Estado de un luchador (`src/logic/fighter.js`):

```js
const fighter = {
  id: 'casco',     // id de atlas de SPEC 01
  x: 300,          // posición horizontal del ancla (pies)
  y: 0,            // altura sobre el piso, 0 = apoyado
  vx: 0,
  vy: 0,
  facing: 1,       // 1 mira a la derecha, -1 a la izquierda
  state: 'idle',
  stateTime: 0,    // segundos desde que entró al estado
  health: 100,
  hasHit: false,   // el ataque actual ya conectó
};
```

Estados: `idle`, `walkForward`, `walkBack`, `crouch`, `jump`, `punch`, `kick`, `hurt`, `knockdown`, `getUp`, `ko`.

Estado del combate (`src/logic/match.js`):

```js
const match = {
  round: 1,
  wins: [0, 0],
  timeLeft: 99,        // segundos
  phase: 'intro',      // 'intro' | 'fight' | 'ko' | 'over'
  phaseTime: 0,
  roundWinner: null,   // 0, 1 o null si la ronda fue empate
  winner: null,        // 0 o 1 cuando phase es 'over'
};
```

Entrada por jugador y por paso, producida por `src/input.js`:

```js
const input = { left: false, right: false, up: false, down: false, punch: false, kick: false };
```

Convenciones:

- Distancias en píxeles lógicos, tiempos en segundos, velocidades en píxeles por segundo.
- Las cajas se miden desde el ancla del luchador: `x` hacia adelante según `facing`, `y` hacia arriba desde el piso.
- Las cajas de golpe y de daño son fijas por estado e iguales para los 5 luchadores.
- La simulación avanza en pasos fijos de `STEP` con acumulador. El render es independiente.
- Cada estado dura un tiempo fijo y reparte esa duración entre los frames que tenga la animación en el atlas.
- `punch` y `kick` del input valen solo en el paso en que se presiona la tecla.

Reglas de combate:

- Los luchadores en `idle`, `walkForward` y `walkBack` giran solos para mirar al rival.
- Caminar hacia el rival usa `walkForward`; alejarse usa `walkBack`.
- El salto conserva la dirección horizontal pulsada al despegar.
- Puño y patada solo salen desde `idle`, `walkForward` y `walkBack`.
- Un ataque conecta como máximo una vez, durante su ventana `active`.
- Un golpe a un rival en el suelo lo deja en `hurt` y lo empuja `HURT_PUSHBACK`.
- Un golpe a un rival en el aire lo deja en `knockdown` y luego `getUp`.
- En `knockdown`, `getUp` y `ko` el luchador no recibe golpes.
- Agachado, la caja de daño no alcanza la caja del puño y sí la de la patada.
- Los luchadores no se atraviesan ni salen de `STAGE_LEFT`..`STAGE_RIGHT`.
- Vida en 0: el luchador pasa a `ko`, que reproduce `knockdown` y se detiene en `koFrame`.
- Reloj en 0: gana la ronda quien tenga más vida.
- Empate por reloj o doble KO: nadie suma y la ronda se repite con el mismo número.
- Gana el combate quien llega a `ROUNDS_TO_WIN`.

Controles:

| Acción | Jugador 1 | Jugador 2 |
| --- | --- | --- |
| Caminar | `A` / `D` | `←` / `→` |
| Saltar | `W` | `↑` |
| Agacharse | `S` | `↓` |
| Puño | `F` | `K` |
| Patada | `G` | `L` |

- `Enter` avanza desde título y pausa o reanuda el combate.
- En selección, cada jugador mueve su cursor con sus teclas de caminar y confirma con su tecla de puño. Su tecla de patada deshace la confirmación.
- Los dos jugadores pueden elegir el mismo luchador. Un marcador "J1" / "J2" sobre cada luchador los distingue.
- En resultado, `Enter` juega la revancha con los mismos luchadores y `Escape` vuelve a selección.
- Las teclas se leen por `event.code`.

Archivos:

- `index.html`: canvas y carga de `src/main.js`.
- `src/main.js`: bucle, escalado del canvas y gestor de escenas.
- `src/config.js`, `src/input.js`, `src/assets.js`, `src/render.js`.
- `src/logic/fighter.js`, `src/logic/combat.js`, `src/logic/match.js`: lógica pura, sin DOM.
- `src/scenes/title.js`, `src/scenes/select.js`, `src/scenes/fight.js`, `src/scenes/result.js`.
- `test/logic/fighter.test.js`, `test/logic/combat.test.js`, `test/logic/match.test.js`.

Esta spec no modifica nada de SPEC 01 salvo `CLAUDE.md` y `README.md`.

## Plan de implementación

1. Crear `index.html`, `src/config.js` y `src/main.js` con canvas de 960x540 escalado con proporción fija, bucle de paso fijo y `src/render.js` dibujando el escenario (cielo en degradado y piso). Prueba manual: `npm start`, abrir `http://localhost:8080/` y ver el escenario al redimensionar la ventana.
2. Crear `src/assets.js`, que carga los 5 atlas de `assets/atlas/`. Si falla una carga, se muestra un mensaje de error en el canvas. Dibujar el primer frame de `idle` de dos luchadores fijos en sus posiciones iniciales (x = 300 y x = 660), el segundo espejado.
3. Crear `src/input.js` con los dos mapas de teclas y `preventDefault` en las teclas del juego.
4. Crear `src/logic/fighter.js` con los estados `idle`, `walkForward`, `walkBack`, `crouch` y `jump`, el giro automático y los límites del escenario. Tests en `test/logic/fighter.test.js`.
5. Crear `src/scenes/fight.js` y conectar input, lógica y render con animación por estado. Prueba manual: los dos luchadores se mueven, saltan y se agachan.
6. Agregar a `src/logic/fighter.js` la separación entre luchadores (pushbox). Tests.
7. Agregar los estados `punch` y `kick` con sus tiempos. Tests de duración y de que no se encadenan antes de terminar.
8. Crear `src/logic/combat.js` con las cajas, la detección de golpe, el daño y el paso a `hurt`. Tests en `test/logic/combat.test.js`, incluido el caso agachado contra puño y contra patada.
9. Agregar `knockdown`, `getUp` y la invulnerabilidad de esos estados. Tests.
10. Dibujar el HUD en `src/render.js`: barras de vida, nombres y marcadores "J1" / "J2".
11. Crear `src/logic/match.js` con fases, reloj, KO, empate y conteo de rondas. Tests en `test/logic/match.test.js`.
12. Integrar `match` en `src/scenes/fight.js`: textos "RONDA N" y "¡PELEA!", input bloqueado fuera de la fase `fight`, estado `ko`, reinicio de posiciones y vida por ronda, reloj y marcadores de ronda en el HUD.
13. Agregar el gestor de escenas a `src/main.js` y crear `src/scenes/title.js`.
14. Crear `src/scenes/select.js` con los 5 retratos, un cursor por jugador y confirmación.
15. Crear `src/scenes/result.js` con el nombre y retrato del ganador, revancha y vuelta a selección.
16. Agregar la pausa con `Enter` a `src/scenes/fight.js`.
17. Actualizar `CLAUDE.md` y `README.md` con cómo ejecutar el juego, los controles y la arquitectura (lógica pura en `src/logic/`, escenas, render).

## Criterios de aceptación

- [ ] `npm start` y abrir `http://localhost:8080/` muestra la pantalla de título sin errores en consola.
- [ ] `package.json` no tiene `dependencies` de runtime.
- [ ] `npm test` pasa, incluidos los tests de `test/logic/`.
- [ ] El canvas conserva la proporción 16:9 al redimensionar la ventana.
- [ ] `Enter` en título lleva a selección.
- [ ] En selección se ven los 5 retratos y cada jugador puede elegir cualquiera de los 5, incluso el mismo.
- [ ] El combate empieza solo cuando los dos jugadores confirmaron.
- [ ] Cada jugador camina, salta y se agacha con sus teclas, sin mover al otro.
- [ ] Los luchadores no se atraviesan ni salen de la pantalla.
- [ ] Los luchadores se miran entre sí después de cruzarse con un salto.
- [ ] Un puño que conecta resta exactamente 8 de vida y una patada 12.
- [ ] Mantener un ataque en contacto resta vida una sola vez por ataque.
- [ ] Un puño no conecta contra un rival agachado; una patada sí.
- [ ] Un golpe a un rival en el aire lo tira al suelo y luego se levanta.
- [ ] Un rival en el suelo o levantándose no recibe daño.
- [ ] Durante "RONDA N" y "¡PELEA!" los luchadores no responden al teclado.
- [ ] Vida en 0 termina la ronda, el perdedor queda tendido y el ganador suma un marcador.
- [ ] Con el reloj en 0 gana la ronda quien tiene más vida.
- [ ] Con el reloj en 0 y vidas iguales, la ronda se repite y nadie suma.
- [ ] Quien gana 2 rondas lleva a la pantalla de resultado con su nombre.
- [ ] `Enter` en resultado inicia la revancha con los mismos luchadores y `Escape` vuelve a selección.
- [ ] `Enter` en combate pausa: reloj y luchadores se detienen. Otro `Enter` reanuda.
- [ ] Las flechas no desplazan la página durante el juego.
- [ ] Los 5 luchadores se ven con sus pies sobre el piso en `idle` y al caminar.

## Decisiones

- **Sí:** Canvas 2D + JS vanilla sin build. Sin dependencias y con control total del dibujo de frames irregulares.
- **No:** Phaser 3. Agrega dependencia y formato de atlas propio para un juego de una sola escena de combate.
- **No:** Godot 4. Flujo basado en editor, poco apto para generar todo desde código.
- **No:** Python + pygame. No corre en navegador.
- **Sí:** 2 jugadores locales. Camino más corto a un combate jugable.
- **No:** rival CPU. Si llega, va en spec propia.
- **Sí:** solo movimientos básicos. Los especiales existen como ilustración, sin frames.
- **No:** bloqueo. Las hojas no tienen frame propio; queda para una spec con su arte.
- **Sí:** mejor de 3 con reloj de 99 segundos. Formato clásico del género.
- **Sí:** empate repite la ronda. Evita definir un criterio de desempate en el MVP.
- **Sí:** todos los luchadores con los mismos valores. Sin balance que ajustar; solo cambia el arte.
- **No:** stats leídos de las barras de las hojas. Spec posterior.
- **Sí:** escenario dibujado por código. No requiere assets nuevos.
- **No:** sonido. Spec propia.
- **Sí:** 960x540 con sprites a 2x. Los frames miden unos 110–130 px de alto y así ocupan cerca de media pantalla.
- **Sí:** cajas de colisión fijas por estado. Evita medir cajas por frame en 5 hojas.
- **Sí:** `knockdown` y `getUp` se disparan al golpear a un rival en el aire. Da uso a esas animaciones sin agregar una mecánica nueva de derribo.
- **Sí:** paso fijo de 1/60 s y lógica pura en `src/logic/`. Hace la simulación determinista y testeable con `node:test`.
- **Sí:** teclas `WASD` + `F`/`G` y flechas + `K`/`L`. Funciona en notebooks sin teclado numérico.
- **Sí:** tests solo para lógica pura. Render e input se verifican a mano.

## Riesgos

| Riesgo | Mitigación |
| --- | --- |
| Ghosting de teclado: algunos teclados no registran ciertas combinaciones de 3 o más teclas. | Se documenta en `README.md`. La reasignación de teclas queda para otra spec. |
| Las figuras de los 5 luchadores no tienen el mismo tamaño y las cajas fijas no coinciden con el arte. | Las cajas viven en `src/config.js` y se ajustan jugando. Cajas por luchador quedan fuera de alcance. |
| Frames de ataque con efectos anchos desplazan el ancla y el luchador "salta" de lado. | Se corrige con `ax` en la definición de SPEC 01 y se regenera el atlas. |
| Una ronda empatada se repite indefinidamente. | Aceptado en el MVP. Solo ocurre con vidas idénticas al agotarse el reloj o con doble KO. |

## Qué **no** entra en esta spec

- Rival CPU.
- Bloqueo.
- Ataques especiales, ataques agachado y ataques aéreos.
- Stats por luchador.
- Sonido.
- Más escenarios.
- Gamepad, táctil y reasignación de teclas.
- Persistencia y juego en red.

Cada una, si llega, va en su propia spec.
