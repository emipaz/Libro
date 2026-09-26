# Capítulo 27 — Archivos: el mundo exterior, byte a byte

Este libro entero, hasta acá, estuvo hablando de cosas que viven **dentro** de la memoria del programa. Listas, diccionarios, clases, grafos: estructuras que se crean cuando el programa arranca y se borran cuando termina. Hay un solo momento en que eso no alcanza, y lo has estado esquivando sin darte cuenta. Cuando cerrás Python, se acabó: tus datos se fueron con él. El capítulo 3 te lo dijo al pasar por `str` y `bytes` — "las computadoras, y sobre todo **los archivos** y la red, trabajan con bytes" — y clavó una regla que hoy vas a cobrar entera: *"cuando leas o escribas archivos, siempre debes **indicar la codificación**"*. El capítulo 6 te avisó que el `finally` sirve para cerrar recursos "**pase lo que pase**" y que lo íbas a explotar "con archivos". El capítulo 10, cuando mostró que una carpeta contiene carpetas, dejó una curiosidad sin explicar: existe `os.walk`, "que hace este recorrido ya listo". El capítulo 20 te enseñó a volcar números a un archivo binario con `tofile` y te dijo que ese diálogo te iba a resultar familiar "cuando llegues al capítulo 27". Y el capítulo 18, el más reciente, te dio la puerta de salida de `pydantic` —el `model_dump_json()`— y `utf-8` a mano.

Son cinco promesas distintas sobre el mismo tema, y este capítulo las paga todas. La promesa de hoy tiene tres escalones. Primero vas a abrir un archivo y a mirar **qué te devuelve `open()`**: no una cadena, sino un objeto con posición, con permisos y con memoria propia. Segundo vas a entender las dos decisiones que están detrás de cada lectura y de cada escritura —**texto o binario**, y **qué codificación**— que son la causa número uno de los errores más desconcertantes del oficio, y vas a ver, byte a byte, cómo una `ñ` se vuelve dos números y por qué una máquina Windows y una Linux pueden discutir sobre el mismo archivo. Tercero vas a salir a la calle: vas a recorrer carpetas, mover archivos, leer un CSV con comas adentro de los campos y armar un programa que ordene una carpeta de documentos de verdad. Y al final vas a poder cerrar, sin dudar, la pregunta que quedó flotando desde el capítulo 6: cómo se escribe un archivo que no existe todavía sin pisar el que sí existe.

> **Dato clave:** hasta acá tus datos vivían en la RAM, que es **volátil**: se apaga cuando apagás la computadora. Un archivo es la única forma que tiene un programa de sobrevivir a su propia ejecución. Todo lo que hiciste en memoria —los grafos del capítulo 24, la tabla hash del 26— se puede volcar a disco. Y al revés: todo lo que hubo en disco alguna vez se puede meter en memoria y trabajar con las estructuras de este libro.

## 1. El objeto que devuelve `open()`

Todo empieza con una línea que escribiste mil veces sin mirar. Abrimos un archivo:

```python
from pathlib import Path

BASE = Path("datos")
BASE.mkdir(exist_ok=True)
ruta = BASE / "notas.txt"

with open(ruta, "w", encoding="utf-8") as f:
    f.write("primera linea del cuaderno\nsegunda linea del cuaderno\ntercera linea del cuaderno\n")
```

La primera sorpresa es que `open()` **no devuelve el contenido**. Devuelve un **objeto**: un archivo abierto, con posición, con permisos y con memoria propia. Mirá de qué tipo es:

```python
with open(ruta, encoding="utf-8") as f:
    print(type(f))                                          # <class '_io.TextIOWrapper'>
    print(f.readable(), f.writable(), f.seekable())          # True False True
    print(f.tell())                                          # 0
    print(repr(f.read(7)))                                   # 'primera'
    print(f.tell())                                          # 7
    print(f.name)                                            # datos\notas.txt
    print(f.mode)                                            # r
    print(f.closed)                                          # False
print(f.closed)                                              # True
```

Seis cosas en un bloque, y las seis importan.

La primera es que el objeto **contesta preguntas sobre sí mismo**. `readable()` te dice `True`, `writable()` te dice `False` —este archivo lo abrimos solo para leer— y `seekable()` te dice `True`. No es adivinanza: el objeto sabe qué le permitiste hacer.

La segunda es `tell()`, que devuelve **en qué posición estás**. Arrancás en `0`. Leés 7 caracteres y la posición pasa a `7`. Esa es la idea entera detrás de `seek()`: el archivo es una cinta, y vos tenés un dedo puesto encima.

La tercera es la trampa más útil del capítulo. Volvé a leer los mismos 7 caracteres sin moverte:

```python
with open(ruta, encoding="utf-8") as f:
    print(repr(f.read(7)))                                   # 'primera'
    f.seek(0)
    print(repr(f.read(7)))                                   # 'primera'
```

Sin el `seek(0)`, el segundo `read(7)` no devuelve `'primera'` otra vez. Devuelve `'segund'`. **El cursor no vuelve solo**: avanza con cada lectura, y para releer hay que volver a propósito. Cualquier programa que lea un archivo dos veces —buscar un dato, después procesar— necesita ese `seek(0)` explícito, y olvidarlo produce un `KeyError` o un resultado vacío que parece un bug de lógica cuando en realidad es un bug de cursor.

La cuarta es `f.name` y `f.mode`: el objeto se acuerda de dónde lo abriste y con qué permiso. La quinta es `f.closed`, que dentro del `with` da `False` y afuera da `True`. Volvés a esa diferencia en dos secciones, porque es de las cosas que más caro sale en la vida real.

> **Dato clave:** `with open(...) as f:` no es azúcar sintáctico. Es un **gestor de contexto** —el mismo mecanismo del capítulo 13— que le promete al objeto del archivo "cuando termine el bloque, te cierro". Cuando el bloque termina —o cuando una excepción salta a mitad de camino— Python se encarga solo. Es el `finally` del capítulo 6, pero escrito una sola vez y sin que tengas que acordarte.

## 2. Los modos: el archivo no es un diccionario, es un martillo

`open()` necesita un permiso. Se lo pedís con el **modo**, y el modo decide tres cosas: si podés leer, si podés escribir, y **qué le pasa a lo que ya estaba adentro**. Esa última es la peligrosa.

```python
from pathlib import Path

prueba = Path("datos/borrador.txt")
with open(prueba, "w", encoding="utf-8") as f:
    f.write("contenido viejo\n")
print(prueba.read_text(encoding="utf-8"))                     # contenido viejo

with open(prueba, "w", encoding="utf-8") as f:
    f.write("contenido nuevo\n")
print(prueba.read_text(encoding="utf-8"))                     # contenido nuevo
```

Fijate lo que pasó entre las dos escrituras: **"contenido viejo" desapareció**. No se movió, no se copió, se **evaporó**. El modo `w` no significa "escribir": significa "**vaciar y escribir**". Trunca el archivo a cero en el instante de abrirlo, antes de que escribas un solo byte. Si el `write` de la segunda línea hubiera fallado — disco lleno, excepción a mitad de camino— ya habías perdido la primera versión igual.

Ese es el modo `w` del que hay que acordarse, y por eso existe el cuarto:

```python
try:
    with open(prueba, "x", encoding="utf-8") as f:
        f.write("nunca llego\n")
except FileExistsError as error:
    print(type(error).__name__, error.errno)                  # FileExistsError 17
```

El modo `x` es `w` **con un candado**: crea el archivo, pero si ya existe, se niega y lanza `FileExistsError`. Es exactamente la validación del capítulo 6 —`raise` cuando el estado no es el que esperabas— aplicada al disco. La diferencia con el `w` clásico es que `x` es **seguro por defecto**: nunca borra nada sin que te des cuenta.

Y el quinto, el más usado del día a día, no trunca nada:

```python
with open(prueba, "a", encoding="utf-8") as f:
    f.write("linea agregada\n")
print(prueba.read_text(encoding="utf-8"))                     # contenido nuevo
                                                             # linea agregada
```

El modo `a` de *append* se para **al final** y escribe desde ahí. Es el modo de las bitácoras: un log se abre en `a` y se le agregan líneas, jamás se reescribe.

El resumen de los cuatro, y el quinto que es la combinación habitual:

| Modos | Lee | Escribe | Si el archivo no existe | Trunca lo que había |
|---|---|---|---|---|
| `r` | sí | no | `FileNotFoundError` | no |
| `w` | no | sí | lo crea | **sí** |
| `a` | no | sí (al final) | lo crea | no |
| `x` | no | sí | lo crea | no (falla si existe) |
| `r+` | sí | sí | `FileNotFoundError` | no |
| `w+` | sí | sí | lo crea | **sí** |
| `a+` | sí | sí (al final) | lo crea | no |
| `rb`, `wb`, `ab` | igual, pero en **bytes** | | | |

Y el quinto modo que no es un modo sino una excepción:

```python
try:
    with open("datos/no_existe.txt", encoding="utf-8") as f:
        f.read()
except FileNotFoundError as error:
    print(type(error).__name__, error.errno)                  # FileNotFoundError 2
```

Un `FileNotFoundError` con `errno` 17 es un archivo que **ya estaba**; con `errno` 2 es un archivo que **no está**. Los dos son `OSError`, y por eso los dos los agarra un mismo `except OSError`. No son dos principios distintos: es la misma familia, y el código del error te dice cuál de los dos pasó.

> **Importante:** el modo por defecto es `r`. Si abrís un archivo para escribir y te olvidás del modo, `open()` te va a dar `UnsupportedOperation` al primer `write`, no un archivo nuevo. El modo explícito siempre: `open(ruta, "w", encoding="utf-8")`. Cuesta tres caracteres y evita una clase entera de sorpresas.

## 3. Texto o binario: dos lenguajes, no dos tonos

Acá está la primera decisión de fondo, y no tiene nada que ver con los modos. `open()` puede devolverte dos tipos de objeto distintos según le pidas **texto** o **binario**, y la diferencia no es de estilo: es de tipo de dato.

```python
texto = Path("datos/notas.txt")
with open(texto, encoding="utf-8") as f:
    print(type(f).__name__)                                  # TextIOWrapper
    print(type(f.read()))                                    # <class 'str'>

numeros = Path("datos/numeros.bin")
numeros.write_bytes(bytes([2, 3, 5, 7, 11]))
with open(numeros, "rb") as f:
    print(type(f).__name__)                                  # BufferedReader
    print(type(f.read()))                                    # <class 'bytes'>
```

Dos clases distintas: `TextIOWrapper` y `BufferedReader`. Y dos tipos distintos de vuelta: `str` y `bytes`. La razón es la que aprendiste en el capítulo 3: un archivo de texto en disco **no tiene letras**, tiene números de 0 a 255. Alguien tiene que decidir qué número es qué letra, y esa decisión es la **codificación**. El `TextIOWrapper` la aplica por vos (por eso necesita el `encoding`); el `BufferedReader` no: te pasa los números crudos, sin interpretar.

Probemos que el binario no interpreta nada. Tomemos los primeros 8 bytes de un archivo de imagen:

```python
imagen = Path("datos/disfraz.png")
imagen.write_bytes(b"\x89PNG\r\n\x1a\n" + b"\x00" * 20)
with open(imagen, "rb") as f:
    cabecera = f.read(8)
    print(cabecera)                                          # b'\x89PNG\r\n\x1a\n'
    print(cabecera.startswith(b"\x89PNG"))                   # True
```

Si esos mismos 8 bytes los leyeras en modo texto, tendrías un problema: `b"\x89"` no es un carácter válido en casi ninguna codificación, y el `read()` explotaría. Ese es el criterio de decisión, y es simple: **¿el archivo contiene letras que alguien tiene que leer, o bytes que solo un programa sabe interpretar?** Lo primero, modo texto. Lo segundo, modo binario.

Y prestá atención a `cabecera`: `b'\x89PNG\r\n\x1a\n'`. Ahí hay un `\r\n` en medio de los bytes. En modo binario no existe la traducción de saltos de línea: lo que hay en disco es **exactamente** lo que recibís. En modo texto, en cambio, hay traducción —y esa es la sección siguiente.

> **Dato clave:** un archivo binario **no tiene codificación**. La codificación es un concepto del mundo del texto. Si abrís un `.png`, un `.pdf` o un `.mp3` en modo texto, no estás "leyendo mal": estás pidiéndole a Python que convierta bytes arbitrarios en letras, y no hay conversión posible. Por eso el capítulo 20 pudo escribir con `tofile` y `fromfile` directo: esos métodos escriben **bytes**, no texto, y por eso no necesitan `encoding`.

## 4. La codificación: la deuda más antigua del libro

Volvamos a la promesa del capítulo 3, que ahora sí podemos pagar inteira. Tomemos una frase con todo lo que le complica la vida a una computadora:

```python
acentos = Path("datos/acentos.txt")
frase = "La Camila pidió jalapeños: Ñandú, pingüino, ñoño"
acentos.write_text(frase + "\n", encoding="utf-8")

bruto = acentos.read_bytes()
print(len(bruto), len(frase))                                # 57 48
print(bruto[bruto.index(b"\xc3\xb1"):bruto.index(b"\xc3\xb1") + 2])  # b'\xc3\xb1'
```

Ochenta y siete caracteres de la frase, pero **48** en el archivo. Y la `ñ` no ocupa un byte: ocupa **dos**, `b'\xc3\xb1'`. Porque `str` guarda caracteres —puntos de código, un concepto humano— y el archivo guarda bytes. La `ñ` es un carácter; en UTF-8 es una secuencia de dos bytes. Cada vez que escribís texto, hay una traducción happening, y esa traducción es la codificación.

Ahora el desastre. Leamos el mismo archivo con la codificación **equivocada**:

```python
print(acentos.read_text(encoding="utf-8").strip())
# La Camila pidió jalapeños: Ñandú, pingüino, ñoño

print(acentos.read_text(encoding="latin-1").strip())
# La Camila pidiÃ³ jalapeÃ±os: ÃandÃº, pingÃ¼ino, Ã±oÃ±o
```

Los mismos bytes, leídos con otra tabla, dan otra frase. `pidió` se vuelve `pidiÃ³`: la `ó` que en UTF-8 son los bytes `0xC3 0xB3`, en latin-1 son dos caracteres: `Ã` y `³`. Nada se rompió: **estás mirando los mismos números con un diccionario distinto**. Esto tiene nombre propio y vos lo vas a reconocer toda tu vida: **mojibake**. Es la mitad de los "caracteres raros" que se quejan los usuarios de un sistema, y su causa es casi siempre esta: alguien leyó un archivo con la codificación equivocada.

Y si la codificación que elegís no puede representar lo que hay, no hay explicación: hay excepción.

```python
try:
    acentos.read_text(encoding="ascii")
except UnicodeDecodeError as error:
    print(type(error).__name__, error.reason, error.start)
# UnicodeDecodeError ordinal not in range(128) 14
```

`ascii` solo sabe de 128 caracteres y la frase tiene acentos. Python te dice **en qué posición** del archivo se rompió (`14`, apenas empieza la `ó` de "pidió"), y por qué. Esa información es oro: un `UnicodeDecodeError` con `start` en una posición concreta te señala la línea culpable sin que tengas que leer el archivo entero.

> **Importante:** ¿y si **no** pasás `encoding`? Ahí Python adivina, y la adivinanza depende de tu sistema operativo. En la máquina de este libro:
>
> ```python
> import locale, sys
>
> print(locale.getpreferredencoding(False))                  # cp1252
> print(sys.getdefaultencoding(), sys.getfilesystemencoding())
> # utf-8 utf-8
> ```
>
> `cp1252` es la codificación de Windows con acentos. El mismo código, en la notebook de otra persona, correría sobre `utf-8` y leería bien. **Tu programa funciona en tu máquina y falla en la del otro** — y no es culpa de la otra máquina. Por eso la regla del capítulo 3 no es una recomendación de estilo: es `encoding="utf-8"` en **cada** `open()` de texto, siempre. Escribirlo cuesta 20 caracteres y te saca de la categoría de los bugs que se reproducen "solo en la compu de mi primo".

Y un detalle más, que aparece solo cuando mirás los bytes en crudo. ¿Qué pasó con el `\n` que escribimos?

```python
acentos.write_text(frase + "\n", encoding="utf-8")
print(acentos.read_bytes()[-3:])                             # b'o\r\n'
```

En disco no hay `\n`. Hay **`\r\n`**. En modo texto, Python no copia tu `\n` tal cual: lo **traduce** al separador de líneas de tu sistema, que en Windows es `\r\n` (retorno de carro + salto de línea) y en Linux es `\n`. Es un detalle de compatibilidad de los años 80, todavía activo.

Y se puede desactivar, que a veces es justo lo que querés:

```python
acentos.write_text(frase + "\n", encoding="utf-8", newline="\n")
print(acentos.read_bytes()[-3:])                             # b'\xb1o\n'
```

Con `newline="\n"` el archivo tiene exactamente lo que le pediste, en cualquier sistema. Guardá este parámetro: la sección 10 lo necesita urgente.

> **Buenas prácticas:** `newline=""` (con comillas) en **lectura** y **escritura** significa "no traduzcas nada, dame los bytes/línea tal cual están". `newline="\n"` (con el valor) significa "escribí siempre con `\n`". No son lo mismo, y confundirlas es la causa del bug más desconcertante de la sección 10.

## 5. El cursor: `tell()` y `seek()`, y por qué importan

Volvamos un momento a la cinta de la sección 1, porque merece su propio espacio. El archivo es una secuencia de bytes con **una posición de lectura actual**. `tell()` te la da; `seek()` la mueve. Y hay dos flavors de `seek` que conviene tener claros.

```python
ruta = Path("datos/notas.txt")
with open(ruta, encoding="utf-8") as f:
    print(f.tell())                                          # 0
    f.read(7)                       # lee 'primera', el cursor queda en 7
    print(f.tell())                                          # 7
    f.seek(0)                       # vuelve al principio
    print(f.tell())                                          # 0
    f.read(7)
    print(f.tell())                                          # 7
    f.seek(0, 2)                    # 0 = desde el principio, 2 = desde el final
    print(f.tell())                                          # 84
```

El segundo argumento de `seek()` es la **referencia**: `0` desde el principio, `1` desde la posición actual, `2` desde el final. Sirve para el caso clásico de "necesito leer los últimos bytes del archivo", que con `seek(0, 2)` + `seek(-100, 0)` es un par de líneas y no un problema.

Y prestá atención al número que salió: **84**. El archivo tiene **81** caracteres. ¿Por qué 84? Porque `tell()` no cuenta caracteres: en un archivo de **texto** cuenta **bytes**, y el archivo tiene `\r\n` en vez de `\n` (sección 4). Son tres saltos de línea, tres `\r` de más, tres bytes de más: 81 + 3 = 84. En una máquina que no traduce saltos de línea, el mismo archivo da 81.

> **Dato clave:** el número que devuelve `tell()` sobre un archivo de texto es un **número opaco**: la documentación de Python te promete que podés guardarlo y devolvérselo a `seek()`, y nada más. No lo trates como "la cantidad de caracteres leídos", porque no lo es. Si lo que querés es contar caracteres, contalos vos con `len()`; si lo que querés es **recordar dónde estabas**, `tell()` es exactamente eso. Y curiosamente, el mismo `tell()` sobre un archivo **binario** sí es un índice de bytes limpio, porque ahí no hay traducción que hacer.

Y el segundo argumento de `read()` es el tamaño:

```python
with open(ruta, encoding="utf-8") as f:
    print(f.read(5))                                          # prime
    print(f.read())                                           # ra linea del cuaderno
                                                          # segunda linea del cuaderno
                                                          # tercera linea del cuaderno
```

`read()` sin argumento se come **todo lo que queda** y deja el cursor al final. `read(5)` se come 5. `readline()` se come **una línea**. Los tres son el mismo cursor en tres 감염ias distintas decerpt.

> **Dato clave:** el cursor es lo que hace que un archivo sea un **archivo**, y no una lista. En una lista, `lista[0]` y `lista[0]` te devuelven lo mismo las veces que quieras. En un archivo, leer **consume**: la segunda lectura arranca donde terminó la primera. Esa asimetría —"leí otra vez y no te da lo mismo"— es la diferencia entre un dato en memoria y un dato en disco, y es la razón por la que todo programa real que toca archivos necesita `seek()` en algún momento.

## 6. Tres formas de leer, y una promesa del capítulo 13 que se paga acá

Acá se junta todo lo anterior. Vamos a leer un archivo de verdad, de 100.000 líneas, de las tres formas que existen, y a medir qué pasa:

```python
grande = Path("datos/grande.txt")
with open(grande, "w", encoding="utf-8") as f:
    for i in range(100000):
        f.write(f"linea {i:06d} con un poco de texto para que pese algo\n")

print(grande.stat().st_size)                                 # 5400000
```

5.400.000 bytes en disco. Ahora las tres lecturas, midiendo **memoria**:

```python
import sys

def peso(objeto):
    total = sys.getsizeof(objeto)
    if isinstance(objeto, list):
        total += sum(sys.getsizeof(item) for item in objeto)
    return total

with open(grande, encoding="utf-8") as f:
    todo = f.read()
print(len(todo), peso(todo))                                 # 5300000 5300049

with open(grande, encoding="utf-8") as f:
    lineas = f.readlines()
print(len(lineas), peso(lineas))                             # 100000 11000984
```

Y el resultado es el que ninguna intuición te prepara:

| Estrategia | Qué trae a la memoria | Bytes de memoria |
|---|---|---|
| `f.read()` | una `str` gigante | **5.300.049** |
| `f.readlines()` | una lista de 100.000 `str` | **11.000.984** |
| `for linea in f:` | nada, una línea a la vez | **~0** |

`readlines()` usa **el doble** de memoria que `read()`, para traer exactamente lo mismo. ¿Por qué? Porque una lista de 100.000 cadenas tiene 100.000 objetos, y cada objeto `str` tiene su propio encabezado:

```python
print(sys.getsizeof(lineas[0]))                              # 102
```

Una cadena de 53 caracteres ocupa **102 bytes**. Los 53 son los letras; los otros 49 son la etiqueta que Python le pega a cada objeto para saber qué es, cuánto mide y cuántas referencias tiene. Pagás esos 49 bytes **100.000 veces**.

Y ahora, la sorpresa honesta: el tiempo es prácticamente el mismo.

```python
import timeit

def recorre():
    total = 0
    with open(grande, encoding="utf-8") as f:
        for linea in f:
            total += len(linea)
    return total

print(f"{min(timeit.repeat(lambda: grande.read_text(encoding='utf-8'), number=1, repeat=3)):.4f}")
# 0.0101
print(f"{min(timeit.repeat(lambda: grande.read_text(encoding='utf-8').splitlines(True), number=1, repeat=3)):.4f}")
# 0.0198
print(f"{min(timeit.repeat(recorre, number=1, repeat=3)):.4f}")
# 0.0196
```

Diez milisegundos contra veinte. **No hay diferencia de velocidad que te haga elegir**: `for linea in f` es tan rápido como `readlines()`. Lo que cambia es la **memoria**, y ahí la diferencia es de 11 millones de bytes contra cero. Ese es el verdadero intercambio, y no es "streaming es más lento": es "streaming usa la memoria justa y cuesta lo mismo".

(Los tres números son de la máquina del libro y van a cambiar en la tuya, siempre en el mismo orden de magnitud. Lo que no cambia es la conclusión: los tres tiempos están en la misma escala, y la memoria no.)

> **Dato clave:** la respuesta a "¿cuál de las tres uso?" no es una preferencia, es una pregunta sobre el archivo. Si el archivo entra cómodo en memoria —un CSV de 2 MB, un log de 50 MB—, usá `readlines()` o `read()`: son más simples y más rápidos. Si el archivo **no entra** —un log de 8 GB, un volcado de millones de filas—, `readlines()` no es "menos elegante": es un `MemoryError` esperando. Ahí el `for` no es una opción estética, es la única que funciona. Y el capítulo 19 te dio el nombre de esto: el costo de memoria de tu programa es `O(tamaño del archivo)` si lo cargás entero, y `O(1)` si lo recorrés.

Y mientras estamos, una observación que costó medio capítulo para llegar: el objeto archivo **es su propio iterador**.

```python
with open(ruta, encoding="utf-8") as f:
    print(f is iter(f))                                      # True
    print(hasattr(f, "__next__"))                            # True
```

¿Eso no es exactamente lo que prometía el capítulo 13? Que una lista y un archivo se comportan igual porque los dos implementan el mismo protocolo: `__iter__` y `__next__`. Escribilo de nuevo, porque la promesa era grande: **cualquier iterable de Python funciona con cualquier consumidor**. Y es literalmente cierto:

```python
with open(ruta, encoding="utf-8") as f:
    lineas = f.readlines()

print(len(list(lineas)))                                     # 3
print(len(tuple(lineas)))                                    # 3
print(any("segunda" in l for l in lineas))                   # True
print(all(l.strip() for l in lineas))                        # True
print(sum(len(l) for l in lineas))                           # 81
print(sorted(l.strip()[:6] for l in lineas))
# ['primer', 'segund', 'tercer']
print([i for i, _ in enumerate(lineas)])                    # [0, 1, 2]
print([a.strip()[:5] for a, b in zip(lineas, lineas)])       # ['prime', 'segun', 'terce']
print([l.strip()[:6] for l in reversed(lineas)])             # ['tercer', 'segund', 'primer']
```

La tabla entera de "consumidores de iterables" del capítulo 13, ahora con un archivo real. Y no es una curiosidad: es la razón por la que podés escribir `with open(...) as f: for fila in f:` y sentir que estás haciendo lo mismo que con una lista. Porque lo estás haciendo.

## 7. `Path`: la anatomía de una ruta

Hasta acá escribimos `open("datos/notas.txt")` a pelo, con la barra pegada. Funciona, pero es frágil: no podés preguntar "¿esto es una carpeta?", ni cambiarle la extensión, ni saber cuántas partes tiene. Para todo eso, y para todo lo que viene, existe `Path`, que ya conociste de pasada en el capítulo 2 —el único lugar del libro donde aparecía, en un ejemplo de `.gitignore`, sin explicación—. Hoy se explica. `Path` es un objeto que **representa una ruta**, y una ruta no es un string: es una estructura con partes que podés interrogar.

```python
p = Path("datos/informes/2024/reporte.pdf")
print(p.as_posix())                                          # datos/informes/2024/reporte.pdf
print(p.name)                                                # reporte.pdf
print(p.stem)                                                # reporte
print(p.suffix)                                              # .pdf
print(p.parts)                                               # ('datos', 'informes', '2024', 'reporte.pdf')
print(p.parent.as_posix())                                   # datos/informes/2024
print(p.parent.parent.name)                                  # informes
```

Siete preguntas, siete respuestas. `name` es el último pedazo, `stem` es el nombre **sin** la extensión, `suffix` es la extensión con el punto, `parts` es la ruta desarmada en tupla, y `parent` sube un nivel. Con `parent.parent.name` ya estás pidiendo "la carpeta que contiene a la carpeta que contiene al archivo", que es una operación que en `os.path` eran tres llamadas y acá es una línea.

Y `with_suffix()` es el que más se usa, porque cambiar de extensión es la operación más común del scripting:

```python
print(p.with_suffix(".bak").as_posix())                      # datos/informes/2024/reporte.bak
print(p.with_suffix(p.suffix.upper()).as_posix())            # datos/informes/2024/reporte.PDF
```

`with_suffix()` no toca el archivo: devuelve un **Path nuevo** con otra extensión. Todos los métodos de `Path` devuelven rutas nuevas y no modifican la original, así que podés encadenarlos sin miedo.

Y ahora las preguntas de existencia y de parentesco, que son las que el capítulo 6 te enseñó a hacer con `try/except`. Ojo con una cosa antes: la ruta `datos/informes/2024/reporte.pdf` **todavía no existe** en tu disco, y sin embargo todo lo de arriba funcionó. `name`, `stem`, `suffix`, `parts` y `with_suffix()` son **aritmética sobre el texto** de la ruta: no tocan el disco, y por eso funcionan con rutas inventadas. Las que sí necesitan el archivo real son las de existencia:

```python
print(p.is_file(), p.is_dir())                               # False False
print(p.resolve().exists())                                  # False
print(p.relative_to("datos").as_posix())                     # informes/2024/reporte.pdf
print(p.relative_to(Path("datos/informes")).as_posix())      # 2024/reporte.pdf
```

`is_file()` y `exists()` necesitan que la cosa exista. Y `relative_to()` funciona igual, porque es puro texto: te devuelve la ruta **relativa** aunque la ruta completa ni siquiera esté en tu máquina.

`relative_to()` es la inversa de construir: si tenés una ruta larga y querés **mostrársela al usuario** en forma corta, le sacás el prefijo. Y si le pedís algo imposible, explota:

```python
try:
    p.relative_to(Path("datos/facturas"))
except ValueError:
    print("ValueError")                                      # ValueError
```

Un `ValueError` honesto: esa ruta no está dentro de la otra. Es la misma forma del capítulo 6 —el estado no es el que esperabas, así que lo digo— pero con el detalle extra de que `relative_to` acepta tanto un `Path` como un `str`, y falla igual si mezclás relativo con absoluto.

Un detalle de borde que muerde: `Path("reporte.pdf")` no tiene carpeta, y su `parent` no es `None`. Es `.`:

```python
print(repr(Path("reporte.pdf").parent.as_posix()))          # '.'
print(bool(Path("")))                                        # True
```

El punto significa "la carpeta actual", que es lo que hace que `Path("")` no sea falso. Si alguna vez escribís `if ruta:` para chequear que te pasaron una ruta, estabas chequeando cualquier cosa. Usá `if ruta.name:` o `ruta.is_file()`.

> **Buenas prácticas:** `Path("datos") / "informes"` usa `/` como operador, no como parte del string. No es decoración: es que `/` construye la ruta **correcta para el sistema operativo** donde estés. En Windows une con `\`, en Linux con `/`, y tu código funciona en los dos sin que te entere. Es la diferencia entre `os.path.join("datos", "informes")` y `os.path.join("datos", "informes/")` —que en Windows te deja una barra pegada que después rompe `os.listdir`— y este operador, que no tiene forma de hacerlo mal. Y `as_posix()` es tu atajo para **imprimir** rutas: te da la forma con `/` siempre, así el mismo programa muestra las mismas rutas en cualquier máquina.

## 8. Recorrer carpetas: la promesa del capítulo 10, cobrada

El capítulo 10 usó el sistema de archivos como ejemplo de estructura recursiva y te dejó dos temas: el código con `os.listdir` + `os.path.isdir` + recursión, y la curiosidad de que existe `os.walk` "que hace este recorrido ya listo". Cobremos las dos. Primero armemos el laberinto:

```python
for carpeta in ("datos/texto", "datos/facturas/vacias", "datos/informes/2024", "datos/informes/2025"):
    Path(carpeta).mkdir(parents=True, exist_ok=True)

for nombre in ("factura_0001.pdf", "factura_0002.pdf", "nota_credito_0003.pdf"):
    Path("datos/facturas", nombre).write_text("%%PDF-1.4\n", encoding="utf-8")
for anio in ("2024", "2025"):
    for nombre in ("reporte.pdf", "resumen.pdf"):
        Path("datos/informes", anio, nombre).write_text("%%PDF-1.4\n", encoding="utf-8")

Path("datos/facturas/inventario.csv").write_text(
    "cliente,email,monto\n", encoding="utf-8", newline="")
```

Y ahora, el recorrido. `iterdir()` es el `listdir` con clase: te da los hijos de **una** carpeta.

```python
print(sorted(p.name for p in Path("datos/facturas").iterdir()))
# ['factura_0001.pdf', 'factura_0002.pdf', 'inventario.csv', 'nota_credito_0003.pdf', 'vacias']
```

Cinco entradas: cuatro archivos y una carpeta. `iterdir()` no distingue ni te importa: te da todo, y cada elemento es un `Path` con sus propios métodos. Esa es la diferencia con `os.listdir`, que te daba strings sueltos y sin herramientas.

`glob()` es lo mismo **con filtro**:

```python
print(sorted(p.name for p in Path("datos/facturas").glob("*.pdf")))
# ['factura_0001.pdf', 'factura_0002.pdf', 'nota_credito_0003.pdf']
```

Y `rglob()` es el `glob()` que **baja solo**:

```python
print(sorted(p.as_posix() for p in Path("datos").rglob("*.pdf")))
# datos/facturas/factura_0001.pdf
# datos/facturas/factura_0002.pdf
# datos/facturas/nota_credito_0003.pdf
# datos/informes/2024/reporte.pdf
# datos/informes/2024/resumen.pdf
# datos/informes/2025/reporte.pdf
# datos/informes/2025/resumen.pdf
```

`rglob` es exactamente la recursión del capítulo 10, resuelta por la biblioteca estándar. Y `glob` acepta patrones con `*` en cualquier parte, no solo al final:

```python
print(sorted(p.as_posix() for p in Path("datos").glob("informes/2*/*.pdf")))
# datos/informes/2024/reporte.pdf
# datos/informes/2024/resumen.pdf
# datos/informes/2025/reporte.pdf
# datos/informes/2025/resumen.pdf
```

`informes/2*/*.pdf` quiere decir "dentro de `informes`, cualquier carpeta que empiece con `2`, y adentro los PDF". Es el poder de los **glob**: no te da una lista de nombres, te da una **forma**. Y esa forma es la misma que usan los sistemas operativos y las consolas de todo el mundo, así que no es un invento de Python: es un idioma.

Y la curiosidad del capítulo 10, la que existía sin explicación:

```python
import os

for raiz, carpetas, archivos in os.walk("datos"):
    print(raiz, archivos)
```

`os.walk` es un **generador** que te va tirando tuplas de tres: dónde estás, qué carpetas hay ahí, qué archivos hay ahí. Y como es un generador —el capítulo 10 te enseñó qué son—, no carga nada en memoria: te va entregando carpeta por carpeta mientras vos las procesás. La diferencia real entre `rglob` y `os.walk` no es la memoria: los dos son perezosos. Es la **forma** de la respuesta. `rglob` te da **archivos**, uno por uno, y el path completo de cada uno; `os.walk` te da **la estructura**, carpeta por carpeta, y te dice qué carpetas hay adentro antes de que vos entres.

> **Dato clave:** el sistema de archivos **es un árbol**, y hace rato que tenés la herramienta para recorrerlo. El capítulo 22 lo dijo al definir el árbol general: "una carpeta tiene muchas subcarpetas". Lo que acabás de escribir es un `preorden` —el mismo recorrido del capítulo 22— aplicado a un árbol que no construiste vos, que ya estaba ahí, hecho de carpetas y de archivos. La recursión del capítulo 10, el `preorden` del capítulo 22 y el `rglob` de hoy son la misma idea con tres ropa distintas.

## 9. `os` y `shutil`: lo que `Path` no hace

`Path` es hermoso para leer, preguntar y recorrer. Pero hay operaciones que no le pertenecen: **mover**, **copiar** y **borrar carpetas enteras** no son cosas de una ruta, son cosas del sistema de archivos. Para eso está `shutil` (de *shell utility*), y para lo demás, `os`.

```python
import shutil

origen = Path("datos/informes/2024/reporte.pdf")
destino = Path("datos/informes/2024/reporte.copia.pdf")
shutil.copy(origen, destino)
print(origen.read_bytes() == destino.read_bytes())          # True
print(destino.name)                                          # reporte.copia.pdf
```

`shutil.copy()` copia **contenido**. Si el archivo eran 30 KB, el destino pesa 30 KB. Y el nombre lo ponés vos: no hay regla de "copia de", la decís.

Copiar una **carpeta entera** recursivamente es `copytree`:

```python
copia = Path("datos/respaldo")
shutil.copytree("datos/informes", copia)
print(sorted(p.relative_to(copia).as_posix() for p in copia.rglob("*.pdf")))
# 2024/reporte.copia.pdf
# 2024/reporte.pdf
# 2024/resumen.pdf
# 2025/reporte.pdf
# 2025/resumen.pdf
```

Y borrar una carpeta con todo adentro es `rmtree`, que merece unWarning honesto: **no pregunta, no tira a la papelera, no hay `undo`**. Borra.

```python
shutil.rmtree(copia)
print(copia.exists())                                        # False
```

> **Importante:** `rmtree` es la función más peligrosa de este capítulo. No le importa si la carpeta tiene 3 archivos o 3 millones, no avisa, no pide confirmación. Todo programa que borre cosas tiene que tener **una** línea de `print` antes del borrado ("voy a borrar X archivos de Y"), porque la versión sin esa línea es la que borra la carpeta equivocada un viernes a las 18. La seguridad no es un permiso del sistema operativo: es un `print` y una regla.

`sí hay un `os` que sí conviene conocer, y es `scandir`, que es `iterdir` pero **con los datos del sistema ya cargados**:

```python
for entrada in sorted(os.scandir("datos/facturas"), key=lambda e: e.name):
    print(f"{entrada.name:<25} dir={entrada.is_dir()!s:<5} {entrada.stat().st_size:>5} bytes")
```

Y la salida:

```
factura_0001.pdf         dir=False    11 bytes
factura_0002.pdf         dir=False    11 bytes
inventario.csv           dir=False    20 bytes
nota_credito_0003.pdf    dir=False    11 bytes
vacias                   dir=True      0 bytes
```

Fijate la diferencia con la sección 8: `iterdir()` te da un `Path` y **preguntás** `is_dir()` cuando lo necesitás. `scandir` te da la respuesta **ya resuelta**, sin una llamada al sistema operativo por entrada. En una carpeta con 100.000 archivos eso es la diferencia entre un programa que se puede usar y uno que tarda diez minutos. Es la misma lógica del capítulo 19: `scandir` no es "más moderno", es **menos syscalls**.

Y para lo demás, `os` tiene lo básico que `Path` no cubre:

```python
print(os.path.getsize(origen))                               # 11
print(os.path.dirname(str(origen)))                          # datos\informes\2024
print(os.rename("datos/facturas/inventario.csv", "datos/facturas/inv.csv"))
```

Y `os.remove()` y `os.mkdir()` y `os.rmdir()`, que son los primos hermanos de `Path.unlink()`, `Path.mkdir()` y `Path.rmdir()`. Todo existe en los dos. `Path` ganó la pelea de la legibilidad, pero `os` no se va, y vas a ver los dos.

## 10. CSV: el módulo que no hay que reinventar

Acá hay un archivo que no es texto libre: es **estructurado**. Filas y columnas, separadas por comas. Y parece facilísimo de leer:

```python
inventario = Path("datos/facturas/inventario.csv")
inventario.write_text(
    "cliente,email,monto\n"
    "Acero SRL,compras@acero.com,15200.50\n"
    '"Talleres del Sur, S.A.",admin@talleres.com,8900\n'
    "Norte Textil,ventas@norte.com,33120.75\n",
    encoding="utf-8", newline="")
```

Mirá la segunda fila: el nombre del cliente tiene **comas adentro**, y está entre comillas dobles. Ahí está la trampa, y es la razón de ser del módulo `csv`. Veamos las dos lecturas:

```python
import csv

with open(inventario, encoding="utf-8", newline="") as f:
    for fila in csv.reader(f):
        print(fila)
```

```
['cliente', 'email', 'monto']
['Acero SRL', 'compras@acero.com', '15200.50']
['Talleres del Sur, S.A.', 'admin@talleres.com', '8900']
['Norte Textil', 'ventas@norte.com', '33120.75']
```

Y ahora lo que hacés si sos curioso y no leés la documentación:

```python
with open(inventario, encoding="utf-8") as f:
    for i, linea in enumerate(f):
        if i == 2:
            print(linea.rstrip().split(","))
            # ['"Talleres del Sur', ' S.A."', 'admin@talleres.com', '8900']
```

**Cuatro columnas donde hay tres.** El nombre del cliente se partió en dos, y el monto se corrió de lugar. Y lo peor: no es un error visible. No lanza excepción, no warns, no rompe. Simplemente **miente**: tu programa cree que el cliente se llama `"Talleres del Sur` y que hay una columna extra que no sabe qué es. Ese tipo de bug es el peor de todos, porque no se rompe: **funciona mal**.

Por eso existe `csv.reader`, que entiende las comillas, los separadores, los saltos de línea adentro de un campo y el escapado. Y por eso existe `csv.DictWriter`/`DictReader`, que es la versión que realmente vas a usar:

```python
with open(inventario, encoding="utf-8", newline="") as f:
    lector = csv.DictReader(f)
    print(lector.fieldnames)                                 # ['cliente', 'email', 'monto']
    for fila in lector:
        print(fila["cliente"], "->", fila["email"], fila["monto"])
```

```
Acero SRL -> compras@acero.com 15200.50
Talleres del Sur, S.A. -> admin@talleres.com 8900
Norte Textil -> ventas@norte.com 33120.75
```

Con `DictReader`, en vez de acceder por posición (`fila[0]`, `fila[1]`, que se rompen si alguien agrega una columna), accedés por nombre: `fila["cliente"]`. Esa es la diferencia entre un programa que sobrevive a que le agreguen una columna y uno que hay que arreglar.

Y ahora, la parte que parece un detalle de sincronía pero es **el** bug del módulo:

```python
salida = Path("datos/salida.csv")
with open(salida, "w", encoding="utf-8", newline="") as f:
    w = csv.DictWriter(f, fieldnames=["cliente", "email", "monto"])
    w.writeheader()
    w.writerows([
        {"cliente": "Acero SRL", "email": "compras@acero.com", "monto": "15200.50"},
        {"cliente": "Norte Textil", "email": "ventas@norte.com", "monto": "33120.75"},
    ])

print(salida.read_bytes())
# b'cliente,email,monto\r\nAcero SRL,compras@acero.com,15200.50\r\nNorte Textil,ventas@norte.com,33120.75\r\n'
```

El `newline=""` del `open` no es opcional acá. Quitáselo y mirá:

```python
mal = Path("datos/mal.csv")
with open(mal, "w", encoding="utf-8") as f:          # sin newline=""
    w = csv.DictWriter(f, fieldnames=["cliente", "email", "monto"])
    w.writeheader()
    w.writerows([{"cliente": "Acero SRL", "email": "compras@acero.com", "monto": "15200.50"}])

print(mal.read_bytes())
# b'cliente,email,monto\r\r\nAcero SRL,compras@acero.com,15200.50\r\r\n'
```

**`\r\r\n`.** Cada línea del archivo tiene un retorno de carro de más. ¿Qué pasó? El módulo `csv` ya escribe su propio terminador `\r\n` (es lo que dice el estándar CSV, y por eso funciona igual en todos los sistemas). Pero como el `open` está en modo texto **por defecto**, hace su traducción de la sección 4: cada `\n` que el `csv` le pasa se convierte en `\r\n`. Entonces `\r` + (`\n` → `\r\n`) = `\r\r\n`. Y un `\r` solo **también es un fin de línea**: el archivo queda con una línea en blanco entre cada registro, y cualquier lector que respet el estándar —Excel incluido— lo abre con el doble de filas.

> **Dato clave:** `newline=""` en un `open` de CSV no es un detalle de estilo: es **desactivar la traducción** para que el módulo `csv` sea el único dueño de los saltos de línea. Sin él, `csv` y `open` se pelean por el mismo `\n` y los dos ganan, y el archivo queda corrupto. Es el mismo `newline=""` de la sección 4 aplicado por el otro lado: en el `write_text` de los acentos lo querías *fijado* a `\n`; en el CSV lo querés *desactivado*. Por eso el valor `""` (desactivar) y el valor `"\n"` (fijar) no son lo mismo.

Y para escribir, el espejo de `DictReader` es `DictWriter`, que ya usaste. Un detalle más: si le pasás una fila con **más** campos que los del `fieldnames`, `DictWriter` no te avisa: los ignora y escribe la fila incompleta. Con `extrasaction="raise"` sí se queja:

```python
try:
    with open(salida, "w", encoding="utf-8", newline="") as f:
        w = csv.DictWriter(f, fieldnames=["cliente"], extrasaction="raise")
        w.writerow({"cliente": "Acero SRL", "email": "sobra"})
except ValueError as e:
    print(e)                                                 # dict contains fields not in fieldnames: 'email'
```

El problema espejo es el otro: si te **falta** un campo, `DictWriter` escribe una **cadena vacía** en esa columna, sin avisar. Un CSV de ventas con la columna `monto` vacía en tres filas es un CSV que parece andando y no lo está.

## 11. Errores propios: cuando `OSError` no alcanza

Ya tenés tu héroe: `except OSError`. Y ya sabés, desde el capítulo 6, que cuando un mismo concepto falla en varios lugares lo correcto es **una excepción propia**. El archivo es el candidato perfecto, porque `OSError` es una sola clase para **diez** problemas distintos. Mirá el archivo de texto de la sección 1:

```python
try:
    with open("datos/no_existe.txt", encoding="utf-8") as f:
        f.read()
except FileNotFoundError:
    print("no existe")
```

Eso anda, pero `FileNotFoundError` es **la** respuesta estándar, y a veces querés otra cosa: querés un mensaje que diga qué ibas a hacer con ese archivo, no solo que no está. Es la diferencia entre `raise ValueError("edad inválida")` y `raise ValueError(f"edad {edad} fuera de rango")` del capítulo 6: el detalle que hace que el error sea **actionable**. Con un archivo, ese detalle suele ser la ruta.

```python
class RutaInvalida(OSError):
    """Una ruta que no se puede usar como archivo de texto."""

def leer_texto(ruta):
    ruta = Path(ruta)
    if not ruta.exists():
        raise RutaInvalida(f"no puedo usar {ruta.as_posix()}: no existe")
    if ruta.is_dir():
        raise RutaInvalida(f"no puedo usar {ruta.as_posix()}: es una carpeta, no un archivo")
    try:
        return ruta.read_text(encoding="utf-8")
    except UnicodeDecodeError as e:
        raise RutaInvalida(f"no puedo usar {ruta.as_posix()}: no es texto utf-8 (byte {e.start})") from e
```

Tres validaciones, tres mensajes, un solo tipo de error. Y probemos:

```python
for candi in ("datos/no_existe.txt", "datos/informes", "datos/disfraz.png"):
    try:
        leer_texto(candi)
    except RutaInvalida as error:
        print(error)
```

```
no puedo usar datos/no_existe.txt: no existe
no puedo usar datos/informes: es una carpeta, no un archivo
no puedo usar datos/disfraz.png: no es texto utf-8 (byte 0)
```

Tres errores que un `OSError` te habría dado como `FileNotFoundError`, `IsADirectoryError` y `UnicodeDecodeError`: tres clases distintas, tres números de `errno` distintos, tres textos que no dicen nada del contexto. Y ahora tenés **una** clase, un `except`, y mensajes que explican la decisión.

Fijate también el `from e` del final. No es decoración: es la **cadena de causas** del capítulo 6. Dice "este error lo provocó este otro", y sin perder el `UnicodeDecodeError` original, que es la información que de verdad te sirve para debuggear. `str(error)` te muestra el mensaje nuevo; `error.__cause__` te deja bajar al original.

Y un detalle de diseño que vale la pena: `RutaInvalida` hereda de `OSError`, no de `Exception`.

```python
print(issubclass(RutaInvalida, OSError))                    # True
```

Eso significa que el `except OSError` que ya tenías en el código **sigue funcionando**. Tu error nuevo es un `OSError` con más información, no un sustituto de la jerarquía. Es la diferencia entre **extender** un contrato y romperlo, y es exactamente el criterio del capítulo 15: cambiar la base es cambiar el comportamiento de todos los que dependían de la base.

> **Buenas prácticas:** `from e` en cuanto relances. Si re-lanzás sin `from`, perdés el rastro original y te queda un `RutaInvalida` sin explicación de por qué. Y nunca uses `raise RutaInvalida(str(e))` para "arreglar" el problema: estás cambiando el mensaje y tirando la causa a la basura, que es peor que no capturarla.

## 12. En producción: un organizador de carpetas

Juntemos todo en algo que se parezca a un programa real. El problema: una carpeta de documentos que llega mezclada, y hay que ordenarla sin perder nada y sin que un archivo raro corte todo.

```python
from collections import Counter
from datetime import date

RAIZ = Path("datos")
ANIO = date.today().year            # el año en el que estamos archivando
DESTINO = RAIZ / "informes" / str(ANIO)
DESTINO.mkdir(parents=True, exist_ok=True)

def organizar(origen: Path, destino: Path, anio: str) -> tuple[int, list[str]]:
    movidos = 0
    errores = []
    for archivo in sorted(origen.glob("*")):
        if not archivo.is_file() or archivo.suffix.lower() != ".pdf":
            continue
        nuevo = destino / f"{anio}_{archivo.name}"
        if nuevo.exists():
            errores.append(f"{archivo.name}: ya existe {nuevo.name}")
            continue
        try:
            shutil.move(str(archivo), str(nuevo))
            movidos += 1
        except OSError as e:
            errores.append(f"{archivo.name}: {e.strerror}")
    return movidos, errores
```

Y el programa que lo usa:

```python
movidos, errores = organizar(RAIZ / "facturas", DESTINO, "2024")
print(movidos)                                               # 3
print(errores)                                               # []
print(sorted(p.name for p in (RAIZ / "facturas").glob("*.pdf")))  # []
print(sorted(p.name for p in DESTINO.glob("*.pdf")))
# ['2024_factura_0001.pdf', '2024_factura_0002.pdf', '2024_nota_credito_0003.pdf']
```

Tres movidos, cero errores, la carpeta de facturas vacía, y los tres archivos en su lugar con el año adelante. Repasemos las cinco decisiones que hacen que esto sea un programa y no un script:

La primera, `shutil.move` en vez de `copy` + `unlink`. Mover en el mismo disco es un `rename`: el archivo pasa de un nombre a otro sin ocupar espacio doble ni dejar una copia a medio camino. Ojo con el "sin dejar basura": `move` **no** garantiza atomicidad en todos los casos, y entre discos distintos `shutil` copia y después borra, así que un corte de luz te puede dejar el archivo en los dos lados. Para 3 archivos da igual. Para 3 millones, no.

La segunda, `if not archivo.is_file()` antes de tocar nada. `glob("*")` devuelve carpetas también, y `shutil.move` sobre una carpeta no falla: mueve **la carpeta entera**. Un `is_file()` de tres palabras te ahorra mover sin querer un árbol de directorios.

La tercera, **`sorted()`**. Escribir archivos no es atómico, y el orden en que aparecen las entradas de una carpeta no está garantizado. Si el resultado de tu programa depende del orden, ordená. En la sección 8 viste que `iterdir` y `glob` devuelven las cosas en el orden del sistema de archivos, que es arbitrario; un programa que funciona en tu máquina y falla en la del otro **por el orden** es el bug más difícil de reproducir que existe.

La cuarta, la lista de `errores` en vez de `raise`. Un archivo que no se puede mover **no debería tirar abajo los otros dos**. Acá el `except OSError` se convierte en un string, se guarda, y el programa sigue. Esa es la diferencia entre un programa que procesa 9.999 de 10.000 archivos y uno que procesa 0 de 10.000. Y la razón por la que `errores` se devuelve en vez de imprimirse: **la función informa, el programa decide**. Si la función imprimiera, no podrías usarla en otro contexto.

La quinta, el `if nuevo.exists()` antes de mover. Te acordás del modo `x` de la sección 2, el que no pisa nada: acá se agradece. Un `move` sobre un archivo que ya existe **lo pisa**. Con el chequeo previo, te dice "ya está" y lo deja hacer a la persona. Y de paso: los nombres quedan con año adelante, y si vuelve a correr el programa, el segundo intento no rompe nada.

Contar por extensión es el resumen más útil que hay de "qué hay en esta carpeta", y sale de la función `Counter` del capítulo 5 aplicada a un `rglob`. Pero primero, un archivo de texto más: el **log**, que no es otra cosa que líneas de `fecha nivel mensaje`.

```python
bitacora = Path("datos/texto/bitacora.log")
with open(bitacora, "w", encoding="utf-8") as f:
    f.write("2024-05-01 INFO arranque\n"
            "2024-05-01 ERROR no se pudo leer la factura 1001\n"
            "2024-05-02 INFO lectura completa\n"
            "2024-05-02 WARN archivo vacio\n"
            "2024-05-03 ERROR no se pudo leer la factura 1002\n"
            "2024-05-03 INFO cierre\n")

niveles = Counter(linea.split()[1] for linea in bitacora.read_text(encoding="utf-8").splitlines())
print(dict(niveles))                                         # {'INFO': 3, 'ERROR': 2, 'WARN': 1}
```

Un `Counter` que te dice cuántos INFO, cuántos ERROR y cuántos WARN hubo. Es exactamente lo que hace `logging` —que lo cubre en detalle más adelante—, hecho a mano en cuatro líneas. Y la diferencia con el `w` de acá arriba es una sola palabra: un log real se abre en `a`, nunca en `w`.

Y ahora el conteo de la carpeta entera, con el log ya adentro:

```python
conteo = Counter(p.suffix for p in RAIZ.rglob("*") if p.is_file())
for extension, cantidad in sorted(conteo.items()):
    print(f"{extension:<6} {cantidad:>3} archivos")
```

```
.bin     1 archivos
.csv     4 archivos
.log     1 archivos
.pdf     8 archivos
.png     1 archivos
.txt     4 archivos
```

Los `.pdf` son los que movió el programa, más los informes y la copia. Y notá el `sorted()`: sin él, el orden de un `Counter` es el orden en que el sistema de archivos devolvió las entradas, que es **arbitrario**. Un `Counter` que se imprime distinto en cada máquina es un `Counter` que no podés comparar con el de nadie.

> **Importante:** el código de la sección 12 tiene una debilidad a propósito, y la vas a notar: la carpeta de destino tiene que **existir** (la creamos con `mkdir(parents=True, exist_ok=True)`), y los archivos se mueven **por nombre**, no por fecha de creación. Un programa de verdad necesita las dos cosas: decidir a qué año va cada archivo leyendo algo del archivo (o del sistema), y garantizar la carpeta. Son las dos metidas del `mkdir(parents=True, exist_ok=True)`: `parents` sube creando todas las carpetas intermedias, `exist_ok` no se queja si ya estaba. Las dos son necesarias. Con solo una de las dos, el programa funciona una vez.

## 13. `os.path` contra `pathlib`

Cerramos con lo que todo el mundo te va a preguntar: **¿cuál uso?** Porque vas a ver código con `os.path` por todos lados, y va a haber gente que jura que `os.path` es viejo y `pathlib` es lo nuevo. Las dos cosas son verdad y las dos están bien.

```python
import os.path

print(os.path.join("datos", "texto", "notas.txt"))           # datos\texto\notas.txt
print(Path("datos", "texto", "notas.txt").as_posix())        # datos/texto/notas.txt
print(os.path.exists("datos/notas.txt"), Path("datos/notas.txt").exists())  # True True
print(os.path.splitext("datos/informes/2024/reporte.pdf"))
# ('datos/informes/2024/reporte', '.pdf')
print(os.path.basename("datos/informes/2024/reporte.pdf"))  # reporte.pdf
print(Path("datos/informes/2024/reporte.pdf").name)         # reporte.pdf
```

La respuesta corta: **`Path` para lo nuevo, `os.path` para lo que ya existe**. `os.path` es una api de strings: te devuelve strings, necesita strings, y hace `os.path.join` con una barra pegada si se la pasás mal. `Path` es un objeto: encadena con `/`, devuelve `Path`, y se niega a hacer las cosas raras. Cuando leés código viejo no lo toques; cuando escribís código nuevo, `Path`.

Pero hay una diferencia de fondo que vale la pena, y no es de estilo. `os.path` es una **función que recibe un string**, así que el string puede ser cualquier cosa: `"datos/notas.txt"`, `""`, `"datos/"`, un path con `..` en el medio. `Path` es un **objeto que se puede interrogar antes de usarlo**:

```python
p = Path("datos/notas.txt")
print(p.suffix, p.stem, p.is_absolute(), p.exists())
```

Esa es la misma diferencia que viste entre `int` y `str` en el capítulo 9, y entre una lista y un `Path`: los tipos con preguntas se validan solos. Un `os.path.exists(os.path.join(algo, "x"))` no te dice si `algo` era una carpeta o un archivo; un `Path(algo).is_dir()` sí. Esa es la razón de ser de `pathlib`, y la razón por la que un `Path` se puede pasar a cualquier función y confiar.

Y `os.path` tiene dos cosas que `Path` **no** tiene: acepta **bytes** además de strings, y trae `abspath()`, que resuelve una ruta contra el directorio de trabajo y te devuelve el camino **completo**:

```python
print(hasattr(os.path, "abspath"))                          # True
print(os.path.abspath("datos/notas.txt"))
# C:\Users\...\datos\notas.txt
print(os.path.join(b"datos", b"notas.txt"))
# b'datos\\notas.txt'
```

Y una confusión que conviene desarmar ya, porque después se vuelve un bug: las **variables de entorno** no viven en `os.path`, sino en el módulo `os` que lo contiene —`os.environ` y `os.getenv`—, y las rutas del sistema están en `sys.path`, que es del módulo `sys`. De las dos vas a necesitar el capítulo 29.

Y la respuesta a la pregunta que ibas a hacer: **no, `os.path` no va a desaparecer**. Está ahí desde hace 30 años, está en toda la biblioteca estándar, y hay librerías de terceros que devuelven strings de ruta. Saber leer las dos es parte de saber Python.

> **Buenas prácticas:** la regla es una sola y es simple: **no mezcles `str` y `Path` en la misma expresión**. `os.path.join("datos", Path("notas.txt"))` anda, y `Path("datos") / "notas.txt"` anda, pero `"datos/" + Path("notas.txt")` revienta. Elegí una de las dos familias y mantenela. La tentación de `os.path` es usar `os.path` "porque es más rápido" —y lo es, un poco— pero la diferencia es de nanosegundos y el costo de legibilidad es de horas.

## 14. Resumen y conceptos clave

Este capítulo abrió el mundo exterior: la primera vez que tus datos dejaron de vivir en la memoria del programa para sobrevivirlo. Y arrancó con una sola línea que venías usando sin mirar, `open(ruta, modo, encoding)`, para descubrir que **no devuelve el contenido sino un objeto**: un `TextIOWrapper` con posición, con permisos y con memoria propia. Viste que `tell()` te dice dónde estás, que `seek()` te deja volver, y que la diferencia entre un archivo y una lista es que **leer consume**: la segunda lectura arranca donde terminó la primera, y por eso todo programa que toca archivos necesita `seek(0)` explícito.

Los **modos** deciden el destino de lo que ya estaba: `w` **trunca** (y borra el archivo antes de que escribas un solo byte, así que perdés la versión anterior si el `write` falla), `a` se para al final y agrega, `x` crea pero **se niega si ya existe** (`FileExistsError`), `r` falla si no está. Y el detalle fino: `FileNotFoundError` con `errno` 2 es un archivo que no está, con `errno` 17 es uno que ya estaba — los dos son `OSError`, y el `errno` te dice cuál.

**Texto o binario** no es un modo: es un tipo de dato distinto. En texto, `open` devuelve `str` y necesita `encoding`; en binario, devuelve `bytes` y no traduce nada — por eso el capítulo 20 pudo escribir con `tofile` sin mencionar una codificación. Y los bytes de una imagen (`b'\x89PNG\r\n\x1a\n'`) no son texto ni pueden serlo.

La **codificación** es la lección más cara del capítulo. `str` guarda caracteres, el archivo guarda bytes: la frase "La Camila pidió jalapeños" son 48 caracteres y 57 bytes, y la `ñ` son dos números (`b'\xc3\xb1'`). Leerla con otra tabla no rompe nada: te da otra frase — el **mojibake**, esa mitad de los caracteres raros que se quejan los usuarios. Si la tabla no alcanza, `UnicodeDecodeError` te dice **en qué byte** se rompió. Y sin `encoding`, Python adivina según el sistema (`cp1252` en Windows): por eso `encoding="utf-8"` no es estilo, es la regla del capítulo 3 hecha código. En el medio apareció la traducción de saltos de línea: escribir `\n` en modo texto pone `\r\n` en disco en Windows, y se desactiva con `newline=""` o se fija con `newline="\n"`.

Las **tres formas de leer** medidas sobre 100.000 líneas y 5.400.000 bytes: `read()` carga 5.300.049 bytes, `readlines()` carga **11.000.984** — el doble, porque cada `str` de 53 caracteres ocupa 102 bytes y paga 49 de encabezado — y el `for` carga **~0**. Y el tiempo es del mismo orden de magnitud (10 contra 20 milisegundos). El compromiso no es "streaming es más lento": es "streaming usa la memoria justa y no paga una factura por ella". Y el `for linea in f` es la promesa del capítulo 13 cobrándose: `f is iter(f)`, y por eso `list`, `any`, `sum`, `zip`, `reversed` y compañía funcionan **igual** con un archivo que con una lista.

`Path` es el objeto que representa una ruta, y tiene respuestas para todo: `name`, `stem`, `suffix`, `parts`, `parent`, `with_suffix()`, `is_file()`, `is_dir()`, `exists()`, `relative_to()`. Encadenar con `/` es lo que hace que la ruta se construya bien en cualquier sistema. `iterdir`, `glob` y `rglob` son el `listdir` con clase, y `rglob` es literalmente la recursión del capítulo 10 resuelta por la biblioteca — un `preorden` del capítulo 22 aplicado a un árbol de carpetas. `os.walk` es lo mismo pero como generador, y te da la estructura (carpeta, subcarpetas, archivos) en vez de solo los archivos. `shutil` mueve, copia y borra — y `rmtree` es la única función del capítulo que no avisa a quién borra.

**CSV** es donde se cae la mayoría de la gente: `split(",")` parte mal un nombre de empresa con comas y **no rompe**, miente, y por eso existe `csv.reader`. `DictReader` accede por nombre en vez de por posición, y sobrevive a que le agreguen una columna. Y el bug estrella: sin `newline=""`, el `csv` que escribe `\r\n` y el `open` que traduce a `\r\n` producen **`\r\r\n`**, una línea en blanco entre cada registro.

Y el cierre con `RutaInvalida(OSError)`: una sola clase propia que valida existencia, tipo y codificación, devuelve mensajes que dicen **qué ibas a hacer** con esa ruta, encadena la causa con `from e`, y **hereda de `OSError`** para no romper el `except` que ya tenías. Extender el contrato, no romperlo. Y el programa final, que mueve, no pisa, ordena, no se corta por un archivo raro y devuelve errores en vez de imprimirlos. Porque un programa real **informa** y deja que quien lo llama decida.

Repasá el checklist antes de seguir:

- [ ] `open()` devuelve un **objeto** (`TextIOWrapper` o `BufferedReader`), no el contenido. Tiene `tell()`, `seek()`, `read()`, `readline()`, `name`, `mode`, `closed`.
- [ ] Leer **consume**: para releer, `seek(0)`. El archivo es su propio iterador (`f is iter(f)`).
- [ ] `with open(...)` es un gestor de contexto: cierra solo, pase lo que pase. Por eso `finally` del capítulo 6.
- [ ] Modos: `r` lee, `w` **trunca**, `a` agrega al final, `x` falla si existe, `+` agrega lectura/escritura. `b` = bytes.
- [ ] `w` **borra antes de escribir**: si el `write` falla, perdiste la versión anterior. `x` es el seguro.
- [ ] `encoding="utf-8"` en **cada** `open` de texto. Sin eso, Python adivina (`cp1252` en Windows) y el bug se reproduce solo en la máquina del otro.
- [ ] En texto, `\n` se traduce a `\r\n` en Windows. `newline=""` desactiva la traducción; `newline="\n"` la fija.
- [ ] `read()` = todo en un `str`; `readlines()` = lista de `str` (el doble de memoria, mismos datos); `for` = streaming, `O(1)` de memoria. Mismo orden de magnitud de tiempo, memoria muy distinta.
- [ ] `Path` te da `name`/`stem`/`suffix`/`parts`/`parent`/`with_suffix()`/`is_file()`/`exists()`/`relative_to()`, y encadena con `/` para no romper en Windows.
- [ ] `iterdir` = un nivel; `glob` = un nivel con patrón; `rglob` = recursivo; `os.walk` = recursivo y generador, con la estructura completa.
- [ ] `shutil.copy` / `copytree` / `move` / `rmtree`. `rmtree` no avisa: poné un `print` antes.
- [ ] **Nunca** parsees un CSV con `split(",")`. `csv.reader` + `newline=""` siempre. Sin `newline=""` sale `\r\r\n`.
- [ ] `DictReader` accede por nombre; `DictWriter` con `extrasaction="raise"` si querés que las columnas de más se quejen.
- [ ] Excepción propia que hereda de `OSError`, valida existencia/tipo/codificación, y relanza con `from e` para no perder la causa.

## 15. Ejercicios

1. **El inventario del disco.** Escribí una función `inventario(raiz)` que recorra una carpeta con `rglob` y devuelva un `dict` con la cantidad de archivos por extensión, **sin contar** las carpetas. Verificá con el laberinto de la sección 8 que devuelve un `dict` con las extensiones que esperás. Después agregale que devuelva también el **tamaño total** en bytes de los archivos con una extensión que le pases como parámetro.

2. **La codificación traviesa.** Guardá un archivo con `write_text` en `utf-8`, después leelo con `latin-1` y con `ascii` en un `try/except`, y mostrá el mojibake del primero y el `UnicodeDecodeError` del segundo con su `start`. Después repetí la lectura pasando `errors="replace"` y mostrá cómo cambia la salida. ¿Qué le pasó a los bytes que no se pudieron decodificar?

3. **El cursor.** Escribí una función que abra un archivo y devuelva la **estadística** de cuántas veces aparece una palabra, usando solo `readline()` en un `while` (sin `for`, sin `readlines`). Después reescribila con `for linea in f` y compará: ¿qué versión es más corta? ¿Cuál es más clara? Mirá `f.tell()` en dos momentos distintos de cada versión y explicá por qué el `for` no te deja elegir el tamaño del salto.

4. **El modo `x` contra el modo `w`.** Escribí dos funciones, `crear_seguro(ruta, texto)` y `crear_peligoso(ruta, texto)`: la primera usa modo `x` y devuelve `False` si el archivo ya existía, la segunda usa `w` y devuelve `False` si también existía. Correlas dos veces con un archivo que ya tiene contenido viejo y mostrá que la primera **conserva** el contenido y la segunda lo **destruye**. Justificá cuál usarías en un programa que genera un archivo de configuración.

5. **El contador de palabras de un archivo grande.** Escribí un programa que cuente palabras de un archivo de texto de **una línea a la vez** con un `dict`, sin `read()` ni `readlines()`. Después **demostrá** que la memoria del contador depende de la cantidad de palabras **distintas** y no del tamaño del archivo: escribí dos archivos, uno veinte veces más grande que el otro, con las mismas palabras repetidas, y compará los tamaños del diccionario. Guardá el resultado en un archivo con `csv.writer` y `newline=""`, y **verificá los bytes** con `read_bytes()`: tenés que ver `\r\n` y no `\r\r\n`.

6. **El organizador a prueba de todo.** Agregale al programa de la sección 12 tres cosas. Primera: que la carpeta del destino la decida el **número de 4 dígitos** que aparece en el nombre del archivo, buscado con `re`, en vez de una constante. Segunda: que los archivos que **no** tienen ese número (o no sean PDF ni CSV) se muevan a una carpeta `otros/` en vez de ignorarse. Tercera: que la función **no** sobreescriba nunca, y que los conflictos se informen en la lista de `errores`. Corrélo dos veces seguidas y verificá que la segunda pasada no mueve nada y no rompe nada.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. El inventario del disco
import csv
import re
import sys
import shutil
from collections import Counter
from pathlib import Path

def inventario(raiz, extension=None):
    """Cuenta archivos por extension y, si se pide, suma el peso de una."""
    conteo = Counter()
    total = 0
    for p in Path(raiz).rglob("*"):
        if not p.is_file():
            continue
        conteo[p.suffix] += 1
        if extension is not None and p.suffix == extension:
            total += p.stat().st_size
    return dict(conteo), total

print(inventario("datos")[0])
# {'.txt': 2, '.png': 1, '.pdf': 7, '.csv': 1}
print(inventario("datos", ".csv")[1])                         # 145


# 2. La codificacion traviesa
acentos = Path("datos/acentos.txt")
print(acentos.read_text(encoding="latin-1").strip())
# La Camila pidiÃ³ jalapeÃ±os: ÃandÃº, pingÃ¼ino, Ã±oÃ±o
try:
    acentos.read_text(encoding="ascii")
except UnicodeDecodeError as e:
    print(type(e).__name__, "|", e.reason, "|", e.start)
# UnicodeDecodeError | ordinal not in range(128) | 14
arreglado = acentos.read_text(encoding="ascii", errors="replace")
print(arreglado.strip())
# La Camila pidi�� jalape��os: ��and��, ping��ino, ��o��o
print("caracteres de reemplazo:", arreglado.count("\ufffd"))   # 14


# 3. El cursor
def cuenta_readline(ruta):
    """Sin for y sin readlines: solo readline adentro de un while."""
    conteo = {}
    with open(ruta, encoding="utf-8") as f:
        while (linea := f.readline()):      # el walrus del capitulo 5
            for palabra in linea.split():
                conteo[palabra] = conteo.get(palabra, 0) + 1
    return conteo

def cuenta_for(ruta):
    """La misma cuenta, con el for de la seccion 6."""
    conteo = {}
    with open(ruta, encoding="utf-8") as f:
        for linea in f:
            for palabra in linea.split():
                conteo[palabra] = conteo.get(palabra, 0) + 1
    return conteo

print(cuenta_readline("datos/texto/grande.txt")["linea"])     # 5000
print(cuenta_for("datos/texto/grande.txt")["linea"])          # 5000


# 4. El modo x contra el modo w
def crear_seguro(ruta, texto):
    """Devuelve False si ya existia, y en ese caso NO lo toco."""
    try:
        with open(ruta, "x", encoding="utf-8") as f:
            f.write(texto)
        return True
    except FileExistsError:
        return False

def crear_peligroso(ruta, texto):
    """Devuelve False si ya existia, pero igual lo HABIA BORRADO."""
    existed = Path(ruta).exists()
    with open(ruta, "w", encoding="utf-8") as f:
        f.write(texto)
    return not existed

Path("datos/config.txt").write_text("version viejo\n", encoding="utf-8")
print(crear_seguro("datos/config.txt", "version nueva\n"))    # False
print(Path("datos/config.txt").read_text(encoding="utf-8").strip())
# version viejo
print(crear_peligroso("datos/config.txt", "version nueva\n"))  # False
print(Path("datos/config.txt").read_text(encoding="utf-8").strip())
# version nueva


# 5. El contador de palabras, linea a linea
def cuenta_palabras(ruta):
    conteo = {}
    with open(ruta, encoding="utf-8") as f:
        for linea in f:
            for palabra in linea.split():
                conteo[palabra] = conteo.get(palabra, 0) + 1
    return conteo

# la prueba: 20 veces mas de archivo, las MISMAS palabras repetidas
chico = Path("datos/texto/chico.txt")
enorme = Path("datos/texto/enorme.txt")
with open(chico, "w", encoding="utf-8") as f:
    for _ in range(5000):
        f.write("linea con datos\n")
with open(enorme, "w", encoding="utf-8") as f:
    for _ in range(100000):
        f.write("linea con datos\n")

c_chico = cuenta_palabras(chico)
c_enorme = cuenta_palabras(enorme)
print(f"chico:  {chico.stat().st_size:>8} bytes | {len(c_chico)} distintas | dict de {sys.getsizeof(c_chico)} bytes")
# chico:     85000 bytes | 3 distintas | dict de 232 bytes
print(f"enorme: {enorme.stat().st_size:>8} bytes | {len(c_enorme)} distintas | dict de {sys.getsizeof(c_enorme)} bytes")
# enorme:  1700000 bytes | 3 distintas | dict de 232 bytes

salida = Path("datos/conteo.csv")
with open(salida, "w", encoding="utf-8", newline="") as f:
    escritor = csv.writer(f)
    escritor.writerow(["palabra", "cantidad"])
    escritor.writerows(sorted(c_enorme.items(), key=lambda par: -par[1]))
print(salida.read_bytes())
# b'palabra,cantidad\r\nlinea,100000\r\ncon,100000\r\ndatos,100000\r\n'


# 6. El organizador a prueba de todo
def organizar(origen, raiz_destino):
    """Cada archivo va a la carpeta de su grupo de 4 digitos, o a 'otros'."""
    movidos = 0
    errores = []
    for archivo in sorted(Path(origen).glob("*")):
        if not archivo.is_file():
            continue
        encontrado = re.search(r"(\d{4})", archivo.stem)
        destino = Path(raiz_destino) / (encontrado.group(1) if encontrado else "otros")
        destino.mkdir(parents=True, exist_ok=True)
        nuevo = destino / archivo.name
        if nuevo.exists():
            errores.append(f"{archivo.name}: ya existe {nuevo.name}")
            continue
        try:
            shutil.move(str(archivo), str(nuevo))
            movidos += 1
        except OSError as e:
            errores.append(f"{archivo.name}: {e.strerror}")
    return movidos, errores

print(organizar("datos/facturas", "datos/archivado"))        # (4, [])
print(organizar("datos/facturas", "datos/archivado"))        # (0, [])
for p in sorted(Path("datos/archivado").rglob("*")):
    if p.is_file():
        print("   ", p.relative_to("datos/archivado").as_posix())
#     0001/factura_0001.pdf
#     0002/factura_0002.pdf
#     0003/nota_credito_0003.pdf
#     otros/inventario.csv
```

Dos apuntes sobre la solución 6. El primero: `re.search(r"(\d{4})", archivo.stem)` es la herramienta que te faltaba para pensar en nombres, y devuelve el **primer** grupo de cuatro dígitos del nombre. Con estos archivos de ejemplo eso da `0001`, `0002`, `0003`; con nombres de la vida real como `factura_2024_0001.pdf` el primer grupo es el año, que es justo lo que querés. Si querés atar el patrón a años de verdad, escribí `(20\d{2})` y el filtro se pone preciso. El segundo: la segunda pasada devuelve `(0, [])` porque la carpeta de facturas ya quedó vacía — no porque el programa se pueda repetir sin efectos por casualidad, sino porque no hay nada que organizar. Corrélo con archivos que **sí** estén en el destino y vas a ver la lista de `errores` llenarse de "ya existe": ese es el comportamiento que querías, y no lo vas a ver en una segunda pasada sobre una carpeta vacía.
