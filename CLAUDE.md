# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Language

Always respond to the user in Spanish in this project.

## Project state

Street Fighter Lo Pibe: a Street Fighter-style arcade fighting game with custom characters. The project language (README, in-game text, move names) is Spanish.

There is no code yet — only `README.md` and five character sheets in `assets/`. No engine, framework, build tooling, tests, or git repository have been chosen or set up. When a stack is introduced, add its build/run/test commands here.

## Character sheets (`assets/`)

Each `assets/sprite <name>.png` is a 1536x1024 composite reference sheet for one fighter, not an engine-ready sprite atlas:

| File | Fighter |
| --- | --- |
| `sprite agus.png` | Agustín |
| `sprite casco.png` | Casco |
| `sprite ia.png` | Carlos |
| `sprite marian.png` | Marian |
| `sprite ricar.png` | Ricar |

Note that `sprite ia.png` is Carlos; the file name does not match the fighter name. File names contain a space, so quote paths.

Sheet layout (same general arrangement on every sheet):

- Left column: portrait, name, tagline, quote, and five stat bars (Fuerza, Velocidad, Defensa, Especial, plus one fighter-specific stat).
- Upper right: labelled animation strips — Idle / Guardia, Caminar adelante, Caminar atrás, Agacharse, Salto, Caída / En el suelo, Ataques básicos (puño), Ataques básicos (patada), Recibir daño, Levantarse.
- Bottom band: four "Ataques especiales" panels, each with a name, a motion input (direction arrows + P and/or K), an illustration, and a one-line effect description.

Things to know before slicing these into game sprites:

- Frames are not on a uniform grid. Frame counts, frame widths, and strip positions differ per animation and per sheet, so each frame needs its own measured rectangle (per-fighter frame data), not a single cell size.
- Frames have no transparency: they sit on a dark painted background with label text and divider lines baked in, and attack effects (water, fire, leaves) overlap the figures. Background removal is required.
- The set of strips varies by fighter. Agustín, Casco, and Ricar have a Victoria pose; Carlos and Marian have an "Ataque fuerte" strip instead and no Victoria pose.
- Special attacks exist only as single illustrated panels, not as animation frames.

The sheets are the source of truth for each fighter's stats, move names, motion inputs, and effect descriptions — read the relevant sheet rather than inventing them.
