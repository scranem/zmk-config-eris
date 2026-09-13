# zmk-config-eris — split BLE (2 mitades)

Conversión del Eris de una sola mitad a un teclado split inalámbrico de 2 mitades
sobre ZMK v0.3, con controladores TENSTAR nRF52840 SuperMini (rojo), target
`nice_nano_v2`.

- **Izquierda = central** (habla con el computador, ejecuta el keymap)
- **Derecha = periférico** (envía sus teclas a la izquierda por BLE)
- Sin TRRS, sin cables entre mitades.

## Estructura

```
zmk-config-eris/
├── build.yaml                    # eris_left + eris_right + settings_reset
├── zephyr/module.yml             # sin cambios
├── boards/shields/eris/
│   ├── Kconfig.shield            # SHIELD_ERIS_LEFT / SHIELD_ERIS_RIGHT
│   ├── Kconfig.defconfig         # izquierda = central; ZMK_SPLIT en ambas
│   ├── eris.dtsi                 # kscan + transform 8x10 + physical layout (compartido)
│   ├── eris_left.overlay
│   ├── eris_right.overlay        # col-offset = 5
│   └── eris.zmk.yml              # metadata
└── config/
    ├── west.yml                  # sin cambios (ZMK v0.3)
    ├── eris.conf                 # sin cambios de fondo (aplica a ambas mitades)
    ├── eris.keymap               # 62 teclas x 5 capas
    └── combos.dtsi               # posiciones re-indexadas + macros
```

## Pines — decisión importante

El mapa `&pro_micro` de ZMK para `nice_nano_v2` define **14 → P1.11** y
**16 → P0.10** (ruteo del nice!nano v2 genuino). El TENSTAR SuperMini expone
sus pines rotulados **D14 en P1.01** y **D16 en P1.02**. Con `&pro_micro 14/16`
las filas 7 y 8 quedarían muertas en esta placa, así que el overlay usa
referencias GPIO crudas (`&gpioX Y`) para TODOS los pines, comentadas con su
rótulo D y su puerto nRF. Mismo cableado en ambas mitades.

| Fila | Rótulo | nRF   | | Columna | Rótulo | nRF   |
|------|--------|-------|-|---------|--------|-------|
| ROW1 | D6     | P1.00 | | COL1    | D9     | P1.06 |
| ROW2 | D5     | P0.24 | | COL2    | D8     | P1.04 |
| ROW3 | D4     | P0.22 | | COL3    | D7     | P0.11 |
| ROW4 | D3     | P0.20 | | COL4    | D20    | P0.29 |
| ROW5 | D2     | P0.17 | | COL5    | D21    | P0.31 |
| ROW6 | D10    | P0.09 | |         |        |       |
| ROW7 | D16    | **P1.02** ⚠ | | | | |
| ROW8 | D14    | **P1.01** ⚠ | | | | |

## Mitad derecha espejada

Como la PCB derecha es espejo de la izquierda con la misma numeración
eléctrica, COL1 es la columna EXTERIOR en ambas mitades. Por eso, en el
transform, las columnas de la derecha van en orden inverso (9,8,7,6,5) al
leerlas de izquierda a derecha. Si al probar la mitad derecha queda invertida
horizontalmente, basta invertir el orden de las columnas derechas en cada fila
del `map` de `eris.dtsi` (nada más).

## Capas y controles (mitad derecha)

| Capa | Uso | Activación |
|------|-----|-----------|
| 0 | Default (izquierda intacta, derecha = placeholders) | — |
| 1 | Toggle 1 (letras placeholder) — también destino del combo BT | `&tog 1` en pos 2 |
| 2 | Toggle 2 (números/símbolos placeholder) | `&tog 2` en pos 3 |
| 3 | Toggle 3 (F1-F12 / atajos CAD placeholder) | `&tog 3` en pos 9 |
| 4 | Modo USB (toda transparente) | combo USB |

`&to 0` (volver a default) está en la **pos 10** en todas las capas. Cada
toggle está en la MISMA tecla física en la capa 0 y en su propia capa, así la
misma tecla enciende y apaga la capa. La izquierda es `&trans` en las capas
1-4, por lo que siempre se comporta como la capa default.

## Combos (mismas teclas físicas de siempre, índices nuevos)

| Combo | Teclas | Posiciones | Acción |
|-------|--------|-----------|--------|
| shift_esc_pair | ESC + LSHIFT | 0 + 24 | `BT_CLR` |
| force_usb_mode | ESC + LCTRL | 0 + 32 | `BT_SEL 1` → `OUT_USB` → `&to 4` |
| force_bt_mode | BSPC + LCTRL | 1 + 32 | `BT_SEL 0` → `OUT_BLE` → `&to 1` |

Nota: un combo de ZMK solo ejecuta UN behavior; el archivo antiguo listaba tres
bindings pero solo corría el primero. Ahora las secuencias van en macros
(`usb_mode`, `bt_mode`) y se ejecutan completas.

## Cambios forzados por la matriz nueva (izquierda)

La PCB nueva agrega R7C5 y R8C5 y elimina R8C3/R8C4. No se perdió ningún
binding:

- `Z` (viejo R8C3) → nuevo R7C5 (pos 52)
- `F12` (viejo R8C4) → nuevo R8C5 (pos 59)

## Compilar, flashear y emparejar

1. Push a GitHub → Actions genera `eris_left-nice_nano_v2-zmk.uf2`,
   `eris_right-...uf2` y `settings_reset-...uf2`.
2. Doble tap al reset de cada SuperMini → aparece unidad USB (bootloader
   nice!nano) → arrastrar el `.uf2` que corresponde a cada mitad.
3. Encender ambas mitades cerca: se emparejan solas entre sí; luego emparejar
   la **izquierda** con el computador (perfil BT 0).
4. Si las mitades no se emparejan entre sí (típico tras reflasheos), flashear
   `settings_reset` en AMBAS, encenderlas una vez, y volver a flashear los
   firmwares normales.

El periférico no ejecuta el keymap (lo ejecuta la central), pero ambos se
compilan desde este mismo repo.
