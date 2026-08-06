# Afinidad (相性) en Uma Musume — nota técnica

Referencia del mecanismo real del cálculo de **相性 (afinidad)** para la herencia
de factores en *Uma Musume Pretty Derby* y de su implementación en esta app.

## Fuentes

La fórmula se obtuvo de **wikiru.jp** (wiki de referencia):

- Fórmula y tabla de rango: <https://umamusume.wikiru.jp/?%E8%82%B2%E6%88%90%E3%82%A6%E3%83%9E%E5%A8%98/%E7%9B%B8%E6%80%A7%E4%B8%80%E8%A6%A7>

Fuentes complementarias:

- アルテマ: <https://altema.jp/umamusume/aisyouhyou>
- 神ゲー攻略: <https://kamigame.jp/umamusume/page/162531637953870891.html>
- GameWith: <https://gamewith.jp/uma-musume/article/show/259457>
- Game8: <https://game8.jp/umamusume/447661>
- Gamerch: <https://gamerch.com/umamusume/233448>
- U-tools (ranking de afinidad): <https://xn--gck1f423k.xn--1bvt37a.tools/characters/rankings/succession/total>
- Dormitorios y curso por personaje: <https://kotoyasyou.work/archives/2353>

> La matriz completa (~150×150) no se vuelca en este repo: pertenece a las wikis.
> Aquí se guardan la fórmula, las reglas y los datos de personajes del juego. El
> cálculo parte de esos datos y aplica los bonus oficiales.

---

## Mecanismo (resumen)

Al heredar factores, los personajes del árbol genético establecen un valor
numérico de afinidad. El total determina el **rango** (◎ / ○ / △) y condiciona
cuántos puntos se heredan.

### Rangos

| Rango | Puntos  |
|-------|---------|
| ◎ (doble círculo) | ≥ 151 |
| ○ (círculo) | 51 – 150 |
| △ (triángulo) | 0 – 50 |

> La app usa estos umbrales reales (antes usaba 150 / 50). El 0 se muestra como `-`.

### Composición de la puntuación

Se valoran **7 afinidades** del árbol más el bonus de la trayectoria propia:

- 親子相性 (padre-hijo): objetivo × padre y objetivo × madre → **2** vías
- 三代相性 (tercera gen): objetivo × padre × abuelo (4 abuelos) → **4** vías
- 両親 (padre-madre): padre × madre → **1** vía

La app lo replica en `calcularAfinidad`:

```ts
puntuación = afinidadPar(objetivo, padre)
           + afinidadPar(objetivo, madre)
           + afinidadPar(padre, madre)
           + afinidadTriple(objetivo, padre, abueloPaterno)
           + afinidadTriple(objetivo, padre, abuelaPaterna)
           + afinidadTriple(objetivo, madre, abueloMaterno)
           + afinidadTriple(objetivo, madre, abuelaMaterna)
```

---

## Bonus comunes (dentro de cada afinidad)

| Condición                | Puntos      | Campo en la app      |
|--------------------------|-------------|----------------------|
| Aptitud de herencia      | +7/aptitud  | `adaptabilidad` (≥2) |
| Mismo dormitorio (寮)    | +2          | `afinidad.dorm`      |
| Mismo curso / 学年       | +2          | `afinidad.grado`     |
| Misma clase (クラス)     | +2          | — (sin modelar)      |
| Grupo (仲間)             | +1/grupo    | `afinidad.grupos`    |
| Nacimiento (牝牡/fecha)  | +1          | — (sin modelar)      |
| Generación (世代)        | +1          | — (sin modelar)      |
| Línea de sangre (血統)   | +1          | — (sin modelar)      |

## Bonus de pareja (solo 親子 y 両親, no en la triple)

| Condición         | Puntos | Campo en la app    |
|-------------------|--------|--------------------|
| Misma habitación (同室) | +2 | `afinidad.room` |
| ニクス (parentesco)     | +1 | — (sin modelar) |
| 絆                    | +1 | — (sin modelar) |
| ウララブースト          | +8 | — (sin modelar) |

## Bonus por G1 (solo en la trayectoria propia)

- **+3** por cada G1 compartido y **ganado** durante la trayectoria.
- **+15** si los seis personajes de la línea heredan el **mismo G1**.
- Un mismo G1 en años distintos cuenta como **1 tipo** (+3, no +6).
- **No cuentan** las G2 ni menores.
- **Excluidas** las finales de URA.

---

## Mecánica clave

### Personaje repetido = 0 puntos

Si el objetivo (A) y un **abuelo** de la misma rama son el mismo personaje
(cualquier versión), esa afinidad de tercera generación vale **0 puntos
incondicional**. `afinidadTriple` devuelve 0 con identificadores repetidos.

### Dormitorios oficiales

| Dormitorio     | Valor `dorm` |
|----------------|--------------|
| 栗東寮 (Ritto) | `ritto`      |
| 美浦寮 (Miho)  | `miho`       |
| 一人暮らし     | `solo`       |

`solo` no otorga el bonus de dormitorio.

---

## Estado en la app

### Implementado
- [x] Umbrales reales (◎≥151 / ○≥51 / △≥1; 0 → `-`).
- [x] Bonus de aptitudes (+7), dormitorio (+2), curso (+2), grupo (+1) y habitación (+2, de pareja).
- [x] Regla de personaje repetido → 0.
- [x] `dorm` y `grado` para **52 de 62** personajes (según la tabla de kotoyasu).

### Pendiente
Para replicar la fórmula total hacen falta por personaje:

1. `クラス` / `仲間` (grupos) → `afinidad.grupos` (+1 por grupo).
2. Fecha de nacimiento (año/mes/día) (+1 en conjunto).
3. `世代` (+1) y `血統` (+1; de ahí la pareja `ニクス` en la 親子).
4. `同室` / room-mate → `afinidad.room` (+2 de pareja).
5. Bonus G1 de cada personaje (para `calcularAfinidad` completo).
6. Completar los 10 personajes sin dorm/curso:
   `brian`, `copano-rickey`, `bamboo-memory`, `admire-vega`, `mejiro-ardan`,
   `sakura-chiyono-o`, `haru-urara`, `tosen-jordan`, `kawakami-princess`, `hishi-akebono`.

Fuente fiable para completarlos: las wikis citadas (wikiru.jp y U-tools),
rellenando el campo opcional `afinidad` en `src/data/characters.json`.

---

## Implementación en el código

- `src/utils/affinityCalculator.ts` — `afinidadPar`, `afinidadTriple`, `getRango`, `calcularAfinidad`.
- `src/types/index.ts` — `Dormitorio`, `AfinidadMetadata`, campo `afinidad?` en `Personaje`.
- `src/data/characters.json` — datos de dormitorio y curso.
- `src/utils/domain.test.ts` — verificación de la fórmula.
