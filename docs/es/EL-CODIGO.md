# El código

## El bucle principal

`0x40BF` cuenta el cuadro, mira la pausa y reparte por escena. `(0xE000)` es la
escena y `(0xE001)` la subescena, y cada escena reparte sus subescenas con una
cadena de `djnz`: el registro `b` llega con la subescena y va bajando hasta
caer en el trozo que toca.

Las nueve escenas están en la tabla de `0x40F6`:

| escena | qué hace |
|---|---|
| 0 | la presentación: el cartel que baja y el rótulo |
| 1 | espera, no toca la VRAM |
| 2 | el título, los nueve andando y el juego |
| 3 | la cuenta atrás |
| 4 | empieza el nivel |
| 5 | el juego |
| 6 | se acabó una vida |
| 7 | fin de partida |
| 8 | ronda pasada |

## El despachador por tablas

Veintiséis veces aparece esto:

    call 04060h
    defw destino_0, destino_1, ...

`0x4060` hace `pop hl` para recuperar su propia dirección de retorno —que es el
principio de la tabla—, `add a,a`, y salta a la entrada número `a`. El `pop`
también quita la vuelta, así que **el destino remata por su cuenta**.

Cuántas entradas tiene cada tabla no se estima: lo cierran, por este orden, el
`ld hl,<nnnn> / push hl` que fabrica la dirección de vuelta, el encaje de que
ninguna entrada apunte dentro de la propia tabla, y en un caso un `call`. Las
veintiséis están en `src/goonies.entries` con su aritmética.

## Los tres intérpretes de dibujo

El cartucho no guarda pantallas: guarda guiones, y tiene tres maneras de
leerlos.

**`0x4682`, guión literal.** `[dirVRAM][bytes...]`, con `0xFE` para cambiar de
dirección y seguir y `0xFF` para cerrar. Escribe byte a byte con WRTVRM.

**`0x4699`, el mismo guión pero borrando.** Es la misma rutina con `c = 0` en
vez de `0xFF`, así que el `and c` de `0x4692` convierte cada byte en un cero.
El mismo guión sirve para pintar y para borrar, sin guardar nada aparte.

**`0x469D`, guión RLE**, por el puerto del VDP:

    0x00        fin
    0x80        cambia de dirección y sigue
    0x01..0x7F  el byte siguiente, repetido n veces
    0x81..0xFF  n & 0x7F bytes literales

Y **se lee al revés de lo que parece**. Quién separa los dos casos es a dónde
vuelve el `djnz`: el de `0x46BA` vuelve al `ld a,(de) / inc de` —o sea
**relee**, y por eso el bit 7 puesto es el literal—, y el de `0x46C4` vuelve
solo al `out (c),a`, así que el bit 7 a cero es la repetición.

Que la medida es buena lo demuestra el encaje: **cada guión acaba exactamente
donde empieza el siguiente**, sin un byte de sobra. Los recorre
`tools/formatos.py` y `tools/cobertura.py` comprueba el encaje.

## Cómo caben 64.000 casillas en 2.983 bytes

Son cien salas de 32x20 casillas: **64.000 casillas**. Y ocupan esto:

| |bytes|
|---|---:|
|los 25 mapas, 80 bytes cada uno|2.000|
|las definiciones de los 108 bloques|717|
|la tabla de punteros a bloques|216|
|la tabla de punteros a mapas|50|
|**total**|**2.983**|

Que son **veintiuna casillas por byte**. Tres capas lo hacen posible:

**Uno.** Un nivel son ochenta bytes: cuatro columnas por veinte filas de
bloques de **8x4 casillas**. Eso da 32 columnas por 80 filas, que es lo mismo
que decir las cuatro salas seguidas. Se montan en RAM en `0xE600`, `0xE880`,
`0xEB00` y `0xED80` —640 bytes cada una— y `0x5F13` vuelca la que toca a la
tabla de nombres a partir de la fila 2, que las dos de arriba son el marcador.

**Dos.** El **bit 7 del byte pide el bloque espejado**, y se usa **615 veces de
las 2.000**. Espejar un bloque es gratis: se lee del revés.

**Tres.** Los 108 bloques **comparten cola**. Cuando dos acaban con las mismas
filas, el puntero del segundo entra a mitad del primero y esos bytes valen para
los dos. Contando cada byte una sola vez ocupan **717** en lugar de **841**. El
bloque 9, por ejemplo, son dos bytes propios (`0x39 0x03`) y detrás se mete
entero el bloque 10.

Y encima, cada bloque va comprimido. Lo descomprime `0x5A05` sobre `0xE1F0` y
para cuando el puntero llega a `0xE210`, o sea a las 32 casillas justas:

    0x0n        n ceros
    0x1n        n veces 0x40
    0x2n <b>    n veces <b>
    0x3n <b>    la fábrica de parejas de 0x5A50
    otro        esa casilla, tal cual

**Los 108 bloques se usan todos.** Ni uno sobra.

## Cómo se enlazan las cuatro salas

Dos tablas gemelas de **19 filas por 4 columnas**, en `0x52DB` y `0x5327`.
`0x52CD` entra con `(0xE06A)*4 + la sala de ahora`: la fila la elige el
*reparto de salas* del nivel y la columna la sala en la que estás.

Las 19 filas no son una estimación. Los 25 bytes de `0x9D67` —uno por nivel,
los que van a `(0xE06A)`— valen como mucho 18, así que hacen falta 19 filas y
ni una más. Y 19×4 = 76 bytes es justo lo que hay de `0x52DB` a `0x5327`, y
otro tanto de `0x5327` a `0x5373`.

Cuál de las dos se mira lo decide por dónde te sales: en `0x5290`, si la
columna baja de `0x15` se va a la primera; en `0x5296`, si llega a `0xC0`, a la
segunda. Una es la salida por la izquierda y la otra por la derecha.

## Los sprites

`0x586A` monta **tres sprites de golpe** a partir de una entrada de nueve
bytes: desplazamiento en y, desplazamiento en x y número de patrón, tres veces.
La entrada la elige `(ix+005h)` multiplicado por nueve.

Y hay una rotación para que no parpadeen siempre los mismos. Los atributos de
sprite se copian a la VRAM empezando **cada cuadro por un sitio distinto** del
búfer de `0xE0AC`. Son once posiciones, y cada entrada trae de dónde empieza y
cuántos bytes van antes de dar la vuelta: `0xE0AC+8n` y `0x54-8n`, con n de 0 a
10. Los `0x54` bytes son 21 sprites.

Pero los once primeros sprites no entran en la rotación: van siempre en el
mismo sitio, porque son **el marcador y el jugador**, que no pueden parpadear.

## El sonido

El motor está en `0xB585..0xB80A` y toca por el PSG con WRTPSG. Se le pide una
pieza con `ld a,<n> / call 0B590h`, y el `and 03fh` de `0xB59F` dice que el
número cabe en seis bits. Antes de tocar mira la prioridad de lo que ya suena
(`cp e / ret c`), así que **un efecto flojo no pisa a la música**.

Cada voz es un puntero, y las voces de una pieza van **seguidas** en la tabla:
`0x5BCB` hace `add a,a` y coge 2 o 3 entradas a partir de `0xB814 + n*2`. Por
eso la tabla no se reparte en piezas: es una bolsa de 47 punteros. Las tres
últimas valen `0xBFEE`, que es donde empieza el relleno: son voces vacías.

Y ojo con dónde empieza esa tabla, que estuvo a punto de colarse. Parece que va
de `0xB814` a `0xB874` con 48 entradas, y entonces la primera vale `0x393C`,
que no es ningún sitio de la música. No es así: **la tabla de periodos tiene
doce bytes, no diez** —es una octava cromática entera, y los doce salen a razón
1,059 unos de otros, que es la raíz doceava de dos—, así que llega hasta
`0xB816` y los punteros son 47. Que la cuenta sea `0xB814 + n*2` solo quiere
decir que **la primera pieza es la número 1**: no hay pieza 0.

## Código compartido con otros cartuchos de Konami

Este desensamblado se ha comparado, instrucción por instrucción, con
todos los demás de Konami de la serie. Cuenta un bloque cuando las
mismas instrucciones, al menos seis desde una etiqueta hasta su `ret`
o su salto sin condición, están en los dos cartuchos; las direcciones
no se miran, porque cambian de un cartucho a otro. De los otros 57,
**52** tienen al menos un bloque en común con este; estos son los
que más comparten:

| Cartucho | Bloques | Instrucciones | Por ejemplo (aquí = allí) |
|---|---:|---:|---|
| [Knightmare](https://antxiko.github.io/Knightmare-disassembly/es/) | 31 | 394 | `lee_mando_y_teclado` (0x4742) |
| [Yie Ar Kung-Fu II](https://antxiko.github.io/YieArKungFu2-disassembly/es/) | 29 | 319 | `guion_rle` (0x469D) |
| [Hyper Sports 3](https://antxiko.github.io/HyperSports3-disassembly/es/) | 24 | 294 | `guion_rle` (0x469D) = `descomprime_a_vram` |
| [Konami’s Soccer](https://antxiko.github.io/KonamisSoccer-disassembly/es/) | 24 | 301 | `lee_mando_y_teclado` (0x4742) = `lee_el_mando_1` |
| [Konami's Ping Pong](https://antxiko.github.io/KonamisPingPong-disassembly/es/) | 22 | 235 | `guion_rle` (0x469D) = `descomprime` |
| [Hyper Rally](https://antxiko.github.io/HyperRally-disassembly/es/) | 21 | 266 | `lee_mando_y_teclado` (0x4742) = `LEE_MANDOS` |
| [Yie Ar Kung-Fu](https://antxiko.github.io/YieArKungFu-disassembly/es/) | 21 | 208 | `pinta_guion` (0x4682) = `dibuja_la_lista` |
| [Road Fighter](https://antxiko.github.io/RoadFighter-disassembly/es/) | 19 | 247 | `lee_mando_y_teclado` (0x4742) = `lee_el_mando` |
| [King's Valley](https://antxiko.github.io/KingsValley-disassembly/es/) | 18 | 253 | `toca_la_nota` (0xB700) = `tono_con_periodo_explicito` |
| [Mopi Ranger](https://antxiko.github.io/MopiRanger-disassembly/es/) | 18 | 178 | `baja_un_paso` (0x49DD) = `avanza_la_cortinilla` |

La matriz entera, los 58 cartuchos entre sí, está en
[Las rutinas](https://antxiko.github.io/MSX-disassembly-database/es/LAS-RUTINAS.html) de la base de datos de la
serie, con el método y el programa que la mide.
