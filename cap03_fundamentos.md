# Capítulo 3 — Fundamentos del lenguaje y Python moderno

En los capítulos 1 y 2 construiste la base: entendiste qué es Python, por qué importa su presente y su futuro, y dejaste tu entorno de desarrollo listo con `uv`. Ahora llega la parte que estabas esperando: **ensuciarte las manos con tu primer código real**.

Este es el capítulo de los fundamentos. Vas a aprender el vocabulario mínimo que usará todo el resto del libro: qué es una expresión y qué es una declaración, cómo funcionan las variables y los tipos de dato, cómo operan los operadores, y cómo dar formato a las cadenas de texto. Además te dejarás listas dos herramientas que usarás a diario: las **convenciones para nombrar variables** y las **funciones introductorias** que trae Python. Y no nos quedaremos en la teoría clásica: cerraremos el capítulo con lo mejor del **Python moderno** — las f-strings (3.6+) y el operador *walrus* (3.8) — además de un repaso de las novedades que están llegando al lenguaje en este mismo momento. Para cuando termines, tendrás el repertorio básico y, además, sabrás escribir Python "a la moderna".

---

## 1. Expresiones y declaraciones

Python distingue dos conceptos fundamentales que conviene tener claros desde el principio.

Una **expresión** es una combinación de valores, operadores, funciones y métodos que produce un valor en una sola línea. Es, literalmente, un trozo de código que *algo devuelve*:

```python
1 + 1
45 >= 11
"carro".upper()
2 ** 10
```

En el **entorno interactivo** de la notebook, cada expresión muestra su resultado automáticamente al final de la celda. Por eso las tres líneas anteriores, si las escribes y ejecutas en una celda sepadas, te mostrarán `2`, `True` , `CARRO` y `20`, pero si la ejecutas en un celda todas juntas solo te mostrara la ultima `20`. Si quires mostrar todas deberas envolver cada expresion en un `print`.

Una **declaración** (*statement*), en cambio, es una orden completa que Python puede ejecutar: una asignación, una estructura de control, una llamada que no aprovecha el valor de retorno. Por ejemplo:

```python
y = 0
for x in range(25):
    y += x
print(y)
```

> **Dato clave:** en un *script* ejecutado como programa, los resultados de las expresiones **no se despliegan por sí solos**. Solo los ves si los pides explícitamente con `print()`. La "magia" de que la celda muestre algo al final es exclusiva del entorno interactivo (la notebook).

### Declaraciones múltiples con `;`

Python permite separar varias declaraciones en una misma línea con el punto y coma `;`:

```python
a = 3; b = a * 4.5; b; a + 5
```

Solo se mostrará el resultado de la última expresión. **Advertencia:** no se recomienda usar este recurso, porque ofusca el código innecesariamente. Úsalo por curiosidad, pero no como hábito.

---

## 2. Variables y tipos de datos

Una **variable** es un nombre que enlaza a un valor en la memoria. Para darle nombre a un valor se usa el operador de asignación `=`:

```python
ciudad = "Buenos Aires"
anio_nacimiento = 1975
pi = 3.1416
```

### Particularidades del tipado en Python

- **Tipado dinámico:** no declaras el tipo de una variable. El intérprete "infiere" el tipo a partir del valor asignado, y puede cambiar en cualquier momento.
- **Fuertemente tipado:** no todas las operaciones están permitidas entre tipos incompatibles (no puedes sumar un número con una cadena de forma implícita).
- **Los tipos son clases:** en Python todo es un objeto, instancia de su tipo.

### Convenciones para nombrar variables

Elegir buenos nombres no es un lujo: es lo que hace que tu código se lea solo. Python tiene convenciones oficiales (la **PEP 8**) que todo el ecosistema sigue:

- **`snake_case`**: minúsculas y guiones bajos para separar palabras (`mi_variable`, `total_ventas`, `anio_nacimiento`). Es la convención estándar para variables y funciones.
- **Nombres descriptivos**: `total_ventas` dice más que `tv` o `x`. Prefiere que el nombre explique *qué* guarda.
- **Constantes en MAYÚSCULAS**: valores que no cambian se escriben con `MAYUSCULAS_Y_GUIONES` (`IVA = 1.21`, `MAX_INTENTOS = 3`).
- **Las clases usan CapWords —también llamado PascalCase—**: lo verás en la Parte VI.
- **Evitar nombres de una letra** (salvo convenciones muy puntuales como `i` para índices en un bucle).

**Nombres válidos**: letras (incluidas `ñ`, acentos y la `_`), números (pero **no pueden empezar** por número) y guiones bajos. Los nombres son sensibles a mayúsculas/minúsculas: `nombre`, `Nombre` y `NOMBRE` son **tres variables distintas**.

```python
precio_neto = 100
IVA = 1.21
total = precio_neto * IVA   # 121.0

mi_variable = "hola"
MiVariable = "OTRA"
print(mi_variable, MiVariable)   # hola OTRA  (son distintas)
```

> **Dato clave** : **PEP 8** es la guía de estilo oficial para el código Python. Sus recomendaciones no forman parte de la sintaxis del lenguaje, por lo que Python no obliga a seguirlas. Sin embargo, están ampliamente adoptadas y ayudan a escribir código consistente y fácil de leer. En un equipo, curso o proyecto, estas convenciones pueden establecerse como normas obligatorias.

Hay palabras que Python **reserva** para su propia sintaxis: **no pueden usarse como nombres de variables**. Escribirlas produce un `SyntaxError`. Las más importantes:

`if`, `else`, `elif`, `for`, `while`, `def`, `return`, `class`, `import`, `from`, `try`, `except`, `with`, `lambda`, `True`, `False`, `None`, `and`, `or`, `not`, `in`, `is`, `global`, `pass`, `break`, `continue`, `raise`, `yield`, `assert`, `del`, `nonlocal`.

> **Importante:** si Python te muestra `SyntaxError: invalid syntax` al definir algo con una palabra como `class` o `if`, casi siempre es porque intentaste usar una **palabra reservada** como nombre de variable. Cambia el nombre y listo.

> ⚠️ Advertencia — No ocultes nombres incorporados: Python permite usar como variables nombres de funciones y tipos incorporados, como print, input, list, str, int, sum, max o type. Sin embargo, al hacerlo, el nuevo objeto oculta la referencia original dentro de ese ámbito, y ya no podrás utilizarla normalmente por su nombre. Este problema se conoce como name shadowing o sombreado de nombres.

```python
sum = 25

total = sum([10, 20, 30])
# TypeError: 'int' object is not callable
```
En este ejemplo, sum dejó de hacer referencia temporalmente a la función incorporada y pasó a referirse al número 25

También conviene no poner a tus archivos el mismo nombre que un módulo que quieras importar:

```text
random.py
json.py
math.py
```

Por ejemplo, un archivo propio llamado random.py podría ser importado en lugar del módulo random de la biblioteca estándar y provocar errores difíciles de identificar.

    Recomendación: usa nombres descriptivos que no coincidan con funciones, tipos o módulos conocidos de Python, como suma, lista_numeros o tipo_dato.


> **Dato clave:** nombrar bien las variables es la forma más barata de "documentar" código. Un buen nombre explica por sí solo qué hace el programa, y cuando vuelvas a tu código semanas después te lo agradecerás. En el Capítulo 1 viste el Zen: *Readability counts* ("la legibilidad cuenta").


### Los tipos básicos

| Tipo          | Qué representa                          | Ejemplos                          |
|---------------|-----------------------------------------|-----------------------------------|
| `int`         | números enteros                         | `24`, `-3`, `0x18` (hexadecimal)  |
| `float`       | números reales (punto flotante)         | `3.1415`, `12.`, `-45.3556`       |
| `complex`     | números complejos                       | `6.32 + 45j`, `1j`                |
| `bool`        | valores lógicos                         | `True`, `False`                   |
| `str`         | cadenas de caracteres                   | `"Hola"`, `'Mundo'`               |
| `NoneType`    | el valor especial "vacío"               | `None`                            |

Los enteros pueden escribirse en varias bases: decimal (`24`), binario (`0b010011`), hexadecimal (`0x18`) u octal (`0o30`). Python 2 tenía también el *entero largo* (`123L`), que ya no existe en Python 3.

#### Trabajar con bases: literatura, creación y conversión

Para **escribir** un entero en otra base usas un prefijo delante del literal:

| Base      | Prefijo | Ejemplo      | Valor decimal |
|-----------|:-------:|:-------------|:-------------:|
| Decimal   | —       | `255`        | 255           |
| Binaria   | `0b`    | `0b11111111` | 255           |
| Octal     | `0o`    | `0o377`      | 255           |
| Hexa      | `0x`    | `0xFF`       | 255           |

Para **leer** un número desde una cadena en una base concreta, `int()` acepta un segundo argumento con la base (de 2 a 36):

```python
int("11111111", 2)    # 255
int("377", 8)         # 255
int("FF", 16)         # 255
int("ff", 16)         # 255  (no distingue mayúsculas)
```

Y para **obtener la representación** de un entero en otra base, existen las funciones `bin()`, `oct()` y `hex()`, que devuelven una cadena con su prefijo:

```python
bin(255)    # '0b11111111'
oct(255)    # '0o377'
hex(255)    # '0xff'
format(255, "b")   # '11111111'   (sin prefijo)
format(255, "X")   # 'FF'         (hexa en mayúsculas, sin prefijo)
```

Los números en cualquier base circulan e interactúan entre sí sin problema, porque al final todos son `int`:

```python
0xFF + 0b1111 + 0o17    # 255 + 15 + 15 = 285
```

> **Dato clave:** los enteros de Python tienen **precisión arbitraria** — no hay un tamaño máximo. Puedes sumar, elevar a potencias enormes o multiplicar números gigantes sin preocuparte por el desbordamiento que sí sufren otros lenguajes.

### Precisión de los flotantes

Los números de punto flotante dependen de la capacidad del equipo y **no siempre dan el resultado exacto**, sino una aproximación:

```python
2.0 / 3.0      # 0.6666666666666666
0.1 + 0.1 + 0.1  # 0.30000000000000004  (¡no es 0.3!)
```

Esto se debe a la representación binaria de los números reales. Para comparar flotantes con seguridad se usa `round()` o un margen de tolerancia:

```python
round(0.1 + 0.1 + 0.1, 2) == round(0.3, 2)   # True
```

#### Valores especiales: `nan` e `inf`

Los flotantes tienen tres valores especiales que conviene conocer desde ya, porque aparecerán en tus datos reales (sobre todo en análisis de datos con *pandas*):

```python
infinito_positivo = float("inf")
infinito_negativo = float("-inf")
no_es_un_numero = float("nan")
```


| Valor especial | Significado                                        | Cómo se obtiene                  |
|:---------------|:---------------------------------------------------|:---------------------------------|
| `nan`          | *Not a Number* — resultado indefinido o ausente    | `1.0 / float("nan")`, `float("nan")`      |
| `inf`          | *Infinito* positivo                                | `1e308 * 1e308`, `float("inf")`      |
| `-inf`         | *Infinito* negativo                                | `1e308 * (-2)1 `, `float("-inf")`    |

```python
a = float("nan")

1.0 / a         # nan
```

> Nota aclaratoria: En la aritmética de los números reales, la división por cero no está definida. Aunque algunas expresiones pueden tender a infinito al estudiar sus límites, esto no significa que dividir directamente por cero dé como resultado infinito. Por ejemplo, 1/x tiende a +∞ cuando x se aproxima a cero por la derecha y a -∞ cuando se aproxima por la izquierda. 
> Por este motivo, Python considera que una división directa por cero es una operación inválida y genera la excepción `ZeroDivisionError` , incluso cuando se utilizan valores de tipo float.

```python
>>> 1.0 / 0.0
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
    1.0 / 0.0
    ~~~~^~~~~
ZeroDivisionError: float division by zero
```

**La trampa clásica del `nan`:** `nan` **no es igual a sí mismo**. Nunca lo compares con `==` (siempre dará `False`). Para detectarlo usa `math.isnan()`; para lo infinito, `math.isinf()`:

```python
import math

a = float("nan")
a == a            # False   (¡nan != nan!)
math.isnan(a)      # True

math.isinf(float("inf"))   # True
math.isfinite(3.14)        # True   (es un número finito)
```

> **Dato clave:** si alguna vez ves `nan` o `inf` en tus resultados, no es un error del código: es la señal de que hubo una operación indefinida o un dato que falta. En *pandas* los datos ausentes se representan justamente como `nan`.

#### `Decimal`: precisión exacta para cálculos de dinero

Cuando los errores de los `float` **no son aceptables** (típicamente al manejar dinero, impuestos, interés o cualquier cálculo que deba cuadrar al céntimo), se usa el módulo `decimal`, que representa los números de forma exacta con una precisión configurable.

```python
from decimal import Decimal

0.1 + 0.2                      # 0.30000000000000004  (float: impreciso)
Decimal("0.1") + Decimal("0.2")  # Decimal('0.3')      (exacto)
```

Nota importante: se construyen con **cadenas** (`Decimal("0.1")`), no desde `float`, porque de lo contrario el error ya viene "contaminado". `Decimal` respeta las reglas de redondeo y precisión que configures:

```python
from decimal import Decimal, getcontext, ROUND_HALF_UP

getcontext().prec = 28            # precisión por defecto (puedes cambiarla)

precio = Decimal("19.99")
iva = precio * Decimal("1.21")
iva.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)   # Decimal('24.19')
```

```python
getcontext().prec = 2
Decimal("1") / Decimal("14")      # Decimal('0.071')  (respeta la precisión)
```

> **¿Cuándo usar cada uno?** Usa `float` para ciencia, gráficas y cálculos donde el error de representación es despreciable. Usa `Decimal` (o el entero en centavos) para **dinero y contabilidad**, donde la exactitud y el redondeo controlado son obligatorios.

### Números complejos (`complex`) básico

Vamos a llegar a los complejos por un camino que deja a los alumnos boquiabiertos. Si intentas sacar la **raíz cuadrada de un número negativo** con el operador de potencia `** (1/2)` (que es una raíz cuadrada), ocurre algo sorprendente: **Python te devuelve un número complejo**:

```python
(-2) ** 0.5        # (8.659560562354934e-17+1.4142135623730951j)
(-4) ** (1/2)      # (1.2246467991473532e-16+2j)
```

No necesitas importar nada: para raíces cuadradas, la potencia `** (1/2)` es tu amiga. Ese resultado es, de hecho, **la definición misma de los números complejos**: la raíz de un número negativo introduce la unidad imaginaria `i`. En matemáticas `√-4 = 2i`; en Python la parte real es `0` (aparece como `1.22e-16`, un residuo de precisión de punto flotante) y la parte imaginaria es `2j`. Python lo escribe como `(0+2j)`.

> **Dato clave:** `x ** (1/2)` es una raíz cuadrada y **funciona con números negativos**, devolviendo un `complex`. En cambio, `math.sqrt()` **rechaza** los números negativos con un `ValueError`. Por eso, para raíces cuadradas, muchas veces no hace falta tocar `math` — aunque es bueno saber que existe:
>
> ```python
> import math
> (-2) ** 0.5         # (8.65e-17+1.4142135623730951j)   OK, devuelve complex
> # math.sqrt(-2)     # ValueError: math domain error     ¡no se puede!
> ```

Python incluye los **números complejos** como tipo nativo. Un complejo tiene una parte real y una parte imaginaria. En Python la unidad imaginaria se escribe con `j` (no con `i` como en matemáticas):

> **¿Por qué `j` y no `i`?** En matemáticas la unidad imaginaria es `i`, pero en **electrónica** la letra `i` (o `I`) ya está reservada para la **corriente eléctrica** (*current*). Para no confundir la corriente con la unidad imaginaria — que además casi siempre aparece trabajando con corrientes alternas en fasores — los ingenieros eléctricos adoptaron `j`. Guido van Rossum siguió esa **convención de la ingeniería eléctrica** al diseñar Python, así que heredamos la `j`. (Como bonus: en notación vectorial `i`, `j`, `k` son los vectores unitarios de los ejes `x`, `y`, `z`, de modo que `j` además "apunta" naturalmente hacia el eje `y`.)

```python
z = 3 + 4j
z.real        # 3.0    (parte real)
z.imag        # 4.0    (parte imaginaria)
```

Se operan con las mismas reglas que los reales — suma, resta, producto, división — y Python se encarga de las cuentas:

```python
(3 + 4j) + (1 - 2j)     # (4+2j)
(3 + 4j) * (1 - 2j)     # (11-2j)
(3 + 4j) ** 2           # (-7+24j)
```

El módulo `cmath` (complejos matemáticos) trae las funciones como módulo, fase, raíces o exponencial para complejos, al estilo de `math` para reales:

```python
import cmath

z = 3 + 4j
abs(z)          # 5.0   (módulo / magnitud)
cmath.phase(z)  # fase (ángulo en radianes)

z.conjugate()   # (3-4j)  (conjugado)
```

**¿Lo necesito?** Los complejos son esenciales en ingeniería, física, telecomunicaciones, procesamiento de señales y matemática avanzada. Para la mayor parte del desarrollo cotidiano (web, datos, scripts) no los tocarás, pero conviene saber que existen y cómo se escriben.

### `bool` y el concepto de "verdadero/falso"

`False` equivale numéricamente a `0`. Cualquier otro número, o cualquier valor "no vacío", es considerado *verdadero* (por defecto `1`). Lo verás reflejado en la función `bool()`:

```python
bool(0)      # False
bool(-3)     # True
bool("")     # False
bool("0")    # True   (es una cadena no vacía)
bool(None)   # False
```

### `None`

`None` es un valor especial que representa "no hay nada" (un vacío). Es muy usado como valor inicial de variables:

```python
var = None
type(var)      # NoneType
var is None    # True
```

### Tipos inmutables

Son aquellos cuya estructura no puede modificarse después de crearse: `int`, `float`, `bool`, `complex` y `str`. Intentar cambiar una parte de ellos produce un error. 

En Python, una variable no contiene directamente el objeto: es un nombre que hace referencia a él. Aunque el objeto sea inmutable, la variable puede reasignarse para que haga referencia a otro objeto

Por ejemplo:

```python
s = "Hola"
s = "Adiós"  # Es válido: ahora s hace referencia a otra cadena

s = "Hola"
# s[0] = "h"   →  TypeError: 'str' object does not support item assignment
```

Las operaciones que parecen modificar un objeto inmutable realmente crean uno nuevo y reasignan la variable:

```python
s = "Hola"
s = "h" + s[1:]  # Se crea una cadena nueva
print(s)          # hola
```


---

## 3. Conversión entre tipos

Python provee **funciones de conversión** que transforman un valor de un tipo a otro.

### `type()`

Devuelve el tipo de un objeto:

```python
type("Hola")   # str
type(12)       # int
type(23j)      # complex
```

### `str()`

Convierte un objeto compatible en cadena:

```python
str(True)         # 'True'
str(12 + 3.5j)    # '(12+3.5j)'
```

### `int()`

Convierte a entero. Trunca los flotantes hacia cero, convierte `True`→`1` y `False`→`0`, y acepta cadenas que representen un entero. **No** acepta complejos ni cadenas inválidas:

```python
int(True)      # 1
int("-12")     # -12
int(5.3)       # 5
int(-5.3)      # -5
# int("Hola")  →  ValueError (no es un número válido)
# int(45.2j)   →  TypeError  (no se puede convertir un complex)
```

### `float()`

Convierte a número real. Acepta enteros, cadenas que representen un real y `bool` (`True`→`1.0`, `False`→`0.0`). No acepta complejos:

```python
float(False)       # 0.0
float("-12.6")     # -12.6
float(-5)          # -5.0
```

### `complex()`

Crea un número complejo. Con un solo argumento, el componente real es ese número y el imaginario es `0j`:

```python
complex(3.5, 2)        # (3.5+2j)
complex(8)             # (8+0j)
complex("23+5j")       # (23+5j)
```

### `bool()`

Convierte a booleano según el criterio de "verdadero/falso":

```python
bool(-3)     # True
bool(0.0)    # False
bool("Hola") # True
bool("")     # False
```

### `eval()` — con cuidado

`eval()` toma una cadena y la evalúa como si fuera una expresión:

```python
eval("12 * 300")        # 3600
eval("12 > 5")          # True
eval("type('Hola')")    # str
```

**Advertencia:** `eval()` puede ejecutar código arbitrario. No lo uses con datos que no controlas (por ejemplo, entrada de usuario), porque es un riesgo de seguridad.

---

## 4. Operadores

Los **operadores** son signos, símbolos o palabras que el intérprete reconoce para realizar una operación. Se clasifican en varias familias.

### Aritméticos

| Operador | Descripción      |
|:--------:|:-----------------|
| `+`      | Suma / concatenación |
| `-`      | Resta / negativo  |
| `*`      | Multiplicación    |
| `**`     | Exponente         |
| `/`      | División          |
| `//`     | División entera   |
| `%`      | Residuo (módulo)  |

En Python 3, la división `/` entre enteros devuelve `float` cuando el resultado no es entero (a diferencia de Python 2, que devolvía la parte entera):

```python
3 / 4      # 0.75
10 / 5     # 2.0
3 // 4     # 0   (división entera)
7 % 3      # 1   (residuo)
```

**Reglas de precedencia:** se evalúa de izquierda a derecha siguiendo: paréntesis → exponente → multiplicación/división → suma/resta:

```python
12 * 5 + 2 / 3 ** 2        # 60.222...
(12 * 5) + (2 / (3 ** 2))  # 60.222... (equivalente)
(12 * 5) + (2 / 3) ** 2    # 60.444... (¡cambia!)
```

**Nota (PEP 8):** usar espacios alrededor de los operadores mejora la legibilidad.

### Cadenas (`str`)

| Operador | Descripción   |
|:--------:|:--------------|
| `+`      | Concatenación |
| `*`      | Repetición    |

```python
"hola" + "mundo"   # 'holamundo'
'hola' * 3         # 'holaholahola'
```

### De relación

Evalúan si dos valores cumplen una condición y devuelven un `bool`:

| Operador | Significado |
|:--------:|:------------|
| `==`     | igual a     |
| `!=`     | distinto de |
| `>` `<`  | mayor / menor |
| `>=` `<=`| mayor o igual / menor o igual |

```python
"hola" == 'hola'    # True
"hola" != 'Hola'    # True (mayúscula importa)
5 > 3               # True
5 <= 3              # False
```

### Lógicos

| Operador | Significado              |
|:--------:|:-------------------------|
| `and`    | se cumplen a y b         |
| `or`     | se cumplen a o b         |
| `not`    | contrario a             |

Además de funcionar con booleanos, Python permite operaciones lógicas con otros tipos usando su "verdadero/falso":

```python
True and 0       # 0   (retorna el segundo operando "falso")
'Hola' and 123   # 123
not True         # False
not (False or True)  # False
```

### De pertenencia

`in` y `not in` evalúan si un objeto se encuentra *dentro de* otro:

```python
'a' in 'Hola'          # True
'z' in 'Hola'          # False
'la' not in 'Hola'     # False
```

### De identidad

`is` e `is not` evalúan si dos nombres se refieren **al mismo objeto** (comparan identidad, no valor):

```python
a = 45
b = 45
a is b             # True  (ambos apuntan al mismo objeto)
type("Hola") is str  # True
```

```python
uno = 1
UNO = 1.0
uno == UNO     # True   (valores iguales)
uno is UNO     # False  (objetos distintos)
```

> **Importante:** `==` compara **valores**; `is` compara **identidad**. Para comparar contra `None`, lo idiomático es `x is None` y no `x == None`.

### De asignación

Enlazan un objeto a un nombre. Los compuestos combinan operación y asignación:

| Operador | Equivale a       |
|:--------:|:-----------------|
| `=`      | `x = y`          |
| `+=`     | `x = x + y`      |
| `-=`     | `x = x - y`      |
| `*=`     | `x = x * y`      |
| `**=`    | `x = x ** y`     |
| `/=`     | `x = x / y`      |
| `//=`    | `x = x // y`     |
| `%=`     | `x = x % y`      |

```python
hora = 19
hora += 1      # hora = 20
y = 2
y **= 3        # y = 8
```

### De bits

Los operadores de bits, también llamados operadores bit a bit, trabajan sobre la representación binaria de los números enteros. Comparan o desplazan individualmente cada uno de sus bits.
Aunque no aparecen con frecuencia en programas cotidianos, resultan importantes en áreas como:

- Sistemas embebidos y programación de bajo nivel.
- Protocolos de red y formatos binarios.
- Criptografía, compresión y procesamiento de imágenes.
- Manejo eficiente de indicadores o flags.
- Algoritmos que trabajan directamente con datos binarios.
- Computación científica, especialmente al procesar máscaras y grandes conjuntos de datos.

> **Importante**: las operaciones de bits pueden ser útiles para optimizar determinados algoritmos, pero no deben utilizarse únicamente porque parezcan más rápidas. En Python, una solución clara suele ser preferible, salvo que exista una necesidad concreta y se haya medido su rendimiento.

Operan sobre cada bit de la representación binaria: `&` (AND), `|` (OR), `^` (XOR), `~` (NOT), `<<` y `>>` (desplazamientos).

```python
a = 0b01101      # 13
b = 0b11010      # 26
```

Podemos comprobar su representación con bin():

```python
bin(13)  # '0b1101'
bin(26)  # '0b11010'
```

Los ceros ubicados a la izquierda no modifican el valor:

```python 
0b01101 == 0b1101  # True
```

- AND: `&`

El resultado contiene un 1 solamente cuando ambos bits son 1:

```text
  01101   # 13
& 11010   # 26
-------
  01000   # 8
```

```python
a & b            # 8   (0b01000)
```

- OR: `|`

El resultado contiene un 1 cuando al menos uno de los bits es 1:

```text
  01101   # 13
| 11010   # 26
-------
  11111   # 31
```

```python
a | b            # 31  (0b11111)
```


- XOR: `^`

El resultado contiene un 1 cuando los bits son diferentes:

```python
  01101   # 13
^ 11010   # 26
-------
  10111   # 23
```

```python
a ^ b            # 23  (0b10111)
```

- Desplazamiento a la izquierda: `<<`

Desplaza los bits hacia la izquierda y agrega ceros a la derecha:

```text
00001101        # 13
01101000        # 104, después de desplazar 3 posiciones
```

```python
a << 3           # 104
```

Para enteros, desplazar n posiciones hacia la izquierda equivale a multiplicar por 2 ** n:

```python
a << 3 == a * (2 ** 3)  # True
```

- Desplazamiento a la derecha: `>>`

Desplaza los bits hacia la derecha:

```text
11010  # 26
00110  # 6, después de desplazar 2 posiciones
```

```python
b >> 2           # 6
```

Con enteros no negativos, desplazar n posiciones hacia la derecha equivale a realizar una división entera por 2 ** n:

```python
b >> 2 == b // (2 ** 2)  # True
```

#### Ejemplo práctico: indicadores o flags

Cada bit puede representar una opción distinta:

```python
LEER = 0b001 # 1
ESCRIBIR = 0b010 # 2
EJECUTAR = 0b100 # 4

permisos = LEER | ESCRIBIR # 3
```

El operador `|` permite activar varios permisos. Después podemos comprobar uno mediante `&`:

```python

bool(permisos & LEER)      # True
bool(permisos & ESCRIBIR)  # True
bool(permisos & EJECUTAR)  # False

```
Este ejemplo muestra uno de sus usos más habituales: almacenar varias opciones binarias dentro de un único número entero.

### Caso Especial del NOT `~`

El operador `~` invierte los bits de un número: 

los bits 1 pasan a ser 0 y los bits 0 pasan a ser 1.

```python
a = 0b01101  # 13

~a  # -14
```

A primera vista, podría parecer que el resultado debería ser 0b10010, es decir, 18:

```text
  01101   # 13
~ -----
  10010   # 18 si se consideran solamente 5 bits
```

Sin embargo, los enteros de Python no tienen una cantidad fija de bits. Para representar los números negativos, las operaciones bit a bit siguen un comportamiento equivalente al sistema de complemento a dos con una cantidad ilimitada de bits.

Por este motivo, se cumple la siguiente relación:

```python
~x == -(x + 1)
```

Por ejemplo:

```python
~13 == -(13 + 1)  # True
~13                # -14

~0                 # -1
~1                 # -2
~-1                # 0
```

Si se desea invertir solamente una cantidad determinada de bits, debe aplicarse una máscara:

```python
a = 0b01101
mascara = 0b11111  # Cinco bits

resultado = ~a & mascara

bin(resultado)  # '0b10010'
resultado       # 18
```

> **Nota**: `~` no es el operador de negación lógica. 
> Para negar una condición se utiliza `not` ; `~` invierte los bits de un número.

```python
not True  # False
~True     # -2
```

> Nota sobre los booleanos: `bool` es una subclase de `int`. 
> Por razones históricas, `False` se comporta numéricamente como 0 y `True` como 1.

```python
issubclass(bool, int)  # True

int(False)  # 0
int(True)   # 1

True + 1    # 2
False + 1   # 1
```

Esto explica el comportamiento de ~:

```
~True   # -2, porque equivale a ~1
~False  # -1, porque equivale a ~0
```

```
not True   # False
not False  # True
```


>⚠️ Advertencia: `~` realiza una inversión de bits, no una negación lógica. Además, aplicar `~` a un booleano está obsoleto desde **Python 3.12** y está previsto que produzca un error en **Python 3.16**. Para negar un booleano debe utilizarse not. Documentación oficial de bool  https://docs.python.org/3/library/stdtypes.html#boolean-type-bool

```powershell
(Python_Book) PS C:python3.14
Python 3.14.0 (tags/v3.14.0:ebf955d, Oct  7 2025, 10:15:03) [MSC v.1944 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
Ctrl click to launch VS Code Native REPL
>>> ~ True
<stdin-0>:1: DeprecationWarning: Bitwise inversion '~' on bool is deprecated and will be removed in Python 3.16. This returns the bitwise inversion of the underlying int object and is usually not what you expect from negating a bool. Use the 'not' operator for boolean negation or ~int(x) if you really want the bitwise inversion of the underlying int.
-2
>>>
```

---

## 5. Variables, `print()` y el primer programa

Ya tienes casi todo el vocabulario. Unamos piezas en un pequeño programa que usa variables, operaciones y `print()` — la función con la que mandas resultados a la terminal desde un script:

```python
N = [1, 9, 3, 4, 6, 12]
total = sum(N)
prom = total / len(N)
print("La suma es:", total, "y hay", len(N), "elementos")
print("El promedio es:", prom)
```

Cuando ejecutes un script con `python programa.py`, solo aparecerá en pantalla lo que `print()` escriba. Las expresiones por sí solas no muestran nada en un script.

### Las funciones introductorias

Python trae listas (incorporadas) unas cuantas **funciones** que vas a usar todos los días. Como sus argumentos pueden ser de varios tipos, son la mejor muestra de "escribir Python": en muchos casos la función no sabe qué es lo que recibe, y aun así hace lo correcto.

- **`print(*valores)`**: muestra valores en la terminal, separados por espacios.
- **`input(mensaje)`**: pide que el usuario escriba algo; **siempre devuelve una cadena** (deberás convertirla con `int()`/`float()` si necesitas un número).
- **`type(objeto)`**: devuelve el tipo (ya la usamos).
- **`len(secuencia)`**: cantidad de elementos de una cadena, lista, tupla…
- **`dir(objeto)`**: lista los nombres disponibles (atributos y métodos) del objeto.
- **`help(objeto)`**: muestra la documentación de un objeto o función.
- **`isinstance(objeto, tipo)`: ¿el objeto es del tipo (o subtipo) dado?** Devuelve `True`/`False`.
- **`range(inicio : fin : paso)`**: genera una secuencia de números (la verás a fondo con los bucles).
- **`id(objeto)`**: la dirección interna del objeto
- **`round(x, n)`**: redondeo de numerro flotante, `n` es la cantidad de decimales.

```python
nombre = input("¿Cómo te llamas? ")        # devuelve una cadena, p. ej. 'Ana'
print("Hola", nombre)

n = input("Dame un número: ")              # '7'  (cadena, aunque parezca número)
print(n * 2)                               # '77'  (¡por eso conviene convertir!)

edad = int(input("¿Edad? "))               # 7  (entero de verdad)
print(edad * 2)                            # 14

length = len("Python")
isinstance(length, int)                    # True
round(3.14159, 2)                          # 3.14
```

> **Dato clave:** `input()` siempre devuelve **una cadena**. El clásico `TypeError` o resultado raro al sumar o multiplicar después de un `input()` es porque falta convertir. Conviértelo apenas lo leas: `int(input(...))` o `float(input(...))`.

---

## 6. Formateo de cadenas: del `%` a las f-strings modernas

Python permite construir cadenas a partir de otras plantillas. A lo largo de su historia se han usado **cuatro técnicas**. Es importante leer las antiguas porque siguen vivas en mucho código, pero **la recomendación moderna es usar f-strings**.

### Delimitadores de una cadena

- **Comillas simples** `'...'` — útiles cuando el texto contiene comillas dobles.
- **Comillas dobles** `"..."` — útiles cuando el texto contiene comillas simples.
- **Comillas triples** `'''...'''` o `"""..."""` — permiten cadenas de varias líneas que preservan el texto tal como se escribe.

### El operador `%` (estilo `printf` de C)

La forma más antigua. La plantilla usa marcadores `%<tipo>` y luego el operador `%` seguido de los valores (entre paréntesis si son varios):

```python
"El nombre es %s y su edad es %d años" % ("Juan", 29)
"El precio es %.2f" % (29.50)
```

| Marcador | Convierte a            |
|:--------:|:-----------------------|
| `%s`     | cadena                 |
| `%d`, `%i` | entero decimal      |
| `%f`, `%.2f` | flotante (2 decimales) |
| `%e`, `%E` | notación científica |
| `%o`, `%x`, `%X` | octal / hexadecimal |

### El método `str.format()` (Python 3)

Más flexible y legible. La plantilla usa llaves `{}`:

```python
"Tu nombre es {0} y tienes {1} años".format("Emiliano", 45)
"{nombre} tiene {edad} años".format(nombre="Ana", edad=30)
"{:*^50}".format("PYTHON")   # centra y rellena con *
"{:.2f}".format(10 / 3)      # 3.33
```

### Las f-strings (Python 3.6+): la forma recomendada

Anteponiendo `f` (o `F`) al delimitador, puedes interpolar **cualquier expresión** dentro de `{}`:

```python
nombre = "Emiliano"
edad = 45
print(f"Mi nombre es {nombre} y tengo {edad} años")

n = [1, 9, 3, 4, 6, 12]
print(f"La suma es {sum(n)} y el promedio es {sum(n) / len(n):.2f}")
```

**Forma de depuración rápida:** con `{variable=}` se imprime el nombre de la variable **y** su valor. Es el arma secreta para depurar: te ahorra el "espera, ¿qué tenía `edad`?":

```python
print(f"{nombre=} {edad=}")   # nombre='Emiliano' edad=45
```

> **Tip de debug — la `=` dentro de las llaves:** la sintaxis `{expr=}` imprime la expresión textual, un `=` y su resultado. No es solo para variables simples: funciona con **expresiones enteras** (`{a + b = }` → `a + b = 7`), admite **formato** a la derecha (`{pi=:.2f}` → `pi=3.14`) y, adelanto de la Parte VI, también con **atributos de objetos** (`{objeto.atributo=}`). Poné un espacio antes del `=` para que el `=` quede separado a la izquierda también (más limpio). Es la versión moderna del `print(x, x*2, x/2)` de toda la vida, ideal cuando estás en el "¿por qué da esto?" investigando:

```python
a, b = 3, 4
print(f"{a + b = }")     # a + b = 7
pi = 3.14159
print(f"{pi=:.2f}")      # pi=3.14
```

### Caracteres de escape

Empiezan con barra invertida `\` e insertan caracteres especiales: `\n` (nueva línea), `\t` (tabulador), `\"` y `\'` (comillas), `\\` (barra invertida), `\r` (retorno de carro), `\b` (retroceso).

```python
print("Primera línea\nSegunda línea")
print("Tabulador:\tHola")
print("La comilla es \"doble\"")
```

### Inmutabilidad y slicing (rebanado)

Las cadenas son **inmutables**: no puedes cambiar un carácter "en el lugar". Cada vez que "modificas" una cadena, en realidad estás **creando una nueva** y reasignando el nombre:

```python
texto = "Hola"
texto = texto.upper()      # 'HOLA'  (nueva cadena, el nombre ahora apunta a ella)
texto[0]                   # 'H'
# texto[0] = "h"           →  TypeError: 'str' object does not support item assignment
```

Que sean inmutables es una garantía: un `str` nunca cambia debajo tuyo. Compartir cadenas entre funciones no puede "romperlas".

Y como `str` es una **secuencia**, puedes "rebanarla" con el *slicing*. La sintaxis es `texto[inicio:fin:paso]` (los tres son opcionales). Los índices empiezan en `0` (`texto[0]` es el primer carácter) y el `fin` es **exclusivo** (no se incluye):

```python
fruta = "manzanas"

fruta[0]        # 'm'   (primer carácter)
fruta[1]        # 'a'   (segundo)
fruta[-1]       # 's'   (último carácter: índices negativos desde el final)
fruta[-2]       # 'a'   (penúltimo)
fruta[0:5]      # 'manza'  (del 0 al 4 según el índice; 'a'[5] no entra)
fruta[2:6]      # 'nzan'
fruta[:4]       # 'manz'   (desde el inicio)
fruta[4:]       # 'anas'   (hasta el final)
fruta[:]        # 'manzanas'  (copia completa)
fruta[::2]      # 'mnaa'   (de a dos: índices 0, 2, 4, 6)
fruta[::-1]     # 'sanaznam'  (invertida: paso negativo)
```

> **Dato clave:** el *slicing* de `str` es la puerta de entrada a las **colecciones**. En la Parte III retomaremos exactamente esta sintaxis con **listas** y **tuplas**; al estar acostumbrado a las rebanadas de cadenas, las colecciones te parecerán naturales. El concepto de *colección* (agrupar varios valores en un objeto) es uno de los pilares que sostiene todo el resto del libro.

### Cadenas `str` vs. bytes: `str` y codificaciones

Hasta ahora trabajamos con `str`, que es la cadena de texto "de verdad": guarda **caracteres** y conoce el alfabeto del que vienen (letras, acentos, emojis, alfabetos asiáticos…). Pero las computadoras, y sobre todo los archivos y la red, trabajan con **bytes** (números de 0 a 255). Para pasar de uno a otro existe la **codificación**.

#### `bytes` y `bytearray`

El tipo `bytes` es una **secuencia inmutable de enteros** (cada uno de 0 a 255). Se escribe anteponiendo `b` a un literal:

```python
b = b"Hola"
type(b)        # bytes
b[0]           # 72   (el código del carácter 'H')
```

`bytearray` es la versión **mutable** de `bytes`:

```python
ba = bytearray(b"Hola")
ba[0] = 104
ba             # bytearray(b'hola')
```

> **Dato clave:** `"Hola"` (str) y `b"Hola"` (bytes) **no son lo mismo**, aunque se vean iguales. La primera guarda caracteres; la segunda guarda enteros. Por eso `"Hola" == b"Hola"` es `False`.

#### Codificaciones: ASCII, UTF-8 y Latin-1

Para pasar de `str` a `bytes` se **codifica** (`.encode()`); para volver, se **decodifica** (`.decode()`). El código depende de la **codificación** elegida. Las tres que vas a encontrarte casi siempre son:

| Codificación | Años | Caracteres | Características |
|:-------------|:----:|:----------:|:----------------|
| `ASCII`      | 1963 | 128        | Solo inglés básico; base de todas las demás. |
| `latin-1` (ISO-8859-1) | 1987 | 256 | Añade caracteres del oeste de Europa (acentos: `á`, `ñ`, `ü`). |
| `utf-8`      | 1993 | ~1.1M      | **Estándar de facto hoy**: puede representar cualquier carácter (acentos, emojis, chino, árabe…). |

```python
texto = "Hola, ¿cómo estás? ñ 😀"

texto.encode("ascii")                   # ERROR: 'ascii' no puede con ñ/¿/😀
texto.encode("latin-1")                 # ERROR: latin-1 no puede con 😀
texto.encode("utf-8")                   # b'Hola, \xc2\xbfc\xc3\xb3mo est\xc3\xa1s? \xf0\x9f\x98\x80'
texto.encode("utf-8").decode("utf-8")   # 'Hola, ¿cómo estás? ñ 😀'   (ida y vuelta correcta)
```

**¿Por qué es importante conocer UTF-8?** Porque hoy es la codificación estándar de la web, de los archivos y de Python 3 (que guarda el código fuente en UTF-8 por defecto). Mientras todos usen UTF-8 no hay problema; el conflicto aparece cuando un archivo fue guardado en `latin-1` y lo lees como `utf-8` (o viceversa), produciendo caracteres extraños (`Ã¡` en lugar de `á`).

```python
# El clásico "mojibake": bytes latin-1 leídos como utf-8
"café".encode("latin-1").decode("utf-8", errors="replace")   # 'cafÃ©'
"café".encode("latin-1").decode("latin-1")                   # 'café'  (ok)
```

Puedes pedir los **bytes crudos** de una cadena (su representación binaria) e inspeccionarlos:

```python
for c in "Hola":
    print(c, ord(c), bin(ord(c)))
# H 72 0b1001000
# o 111 0b1101111
# l 108 0b1101100
# a 97 0b1100001
```

`ord()` te da el código numérico (punto de código) de un carácter, y `chr()` hace lo inverso (del código al carácter):

```python
ord("A")     # 65
chr(65)      # 'A'
ord("ñ")     # 241
```

> **Dato clave:** cuando leas o escribas archivos (lo verás en la Parte VIII), siempre debes **indicar la codificación** (`encoding="utf-8"`). Es la causa de muchísimos errores de caracteres raros que a primera vista parecen "magia negra", y que en realidad son un problema de codificación: bytes interpretados con el código equivocado.

---

## 7. Python moderno — el operador *walrus* `:=` (Python 3.8)

El **operador walrus** (morsa, por su forma `:=`) se introdujo en la **PEP 572** — la misma cuyo debate encendido provocó la renuncia de Guido van Rossum como BDFL (lo viste en el Capítulo 1). Permite **asignar un valor y usarlo a la vez** dentro de una expresión.

### Ejemplo básico

```python
if (a := 1) < 10:
    print("hola, Walrus :=")

print(a)   # 1  (queda asignada)
```

Sin el walrus, habría que escribir dos líneas (asignar primero, comparar después). Con `:=`, la asignación ocurre dentro de la propia condición.

### Uso útil: evitar calcular dos veces

Un caso muy real: en lugar de llamar `len()` dos veces o crear variables intermedias en un diccionario, puedes asignar mientras construyes:

```python
numbers = [2, 8, 0, 1, 1, 9, 7, 7]
description = {
    "length": (num_length := len(numbers)),
    "sum":    (num_sum    := sum(numbers)),
    "mean":    num_sum / num_length,
}
description   # {'length': 8, 'sum': 35, 'mean': 4.375}
```

### Subexpresiones con nombre

En una fórmula larga (como el cálculo de distancia haversine del material del curso), puedes "nombrar" un término intermedio para inspeccionarlo:

```python
# dis = 2 * radio * asin(sqrt((hav := sin(...) ** 2) + ...))
```

> **Buenas prácticas:** el walrus es poderoso pero fácil de usar mal. Uséalo cuando **claramente** evita duplicar un cálculo o hace el código más legible; no lo fuerces en cualquier asignación, o el código se vuelve difícil de leer.

---

## 8. Python moderno — lo último que está llegando: novedades

El libro se ejecuta en Python 3.13, así que vale la pena cerrar con las mejoras recientes que ya notarás en tu día a día.

### Mensajes de error más claros (3.10+)

Python 3.10 rediseñó muchos mensajes de error para que sean más fáciles de entender. Por ejemplo, una cadena sin cerrar que antes daba el críptico `SyntaxError: EOL while scanning string literal`, ahora te dice directamente que te falta terminar la cadena. Además:

- Los **errores de atributo y nombre** ahora sugieren posibles correcciones si escribes mal algo (`math.py` en lugar de `math.pi` te sugiere alternativas).
- Un `=` donde debería haber `==` genera un mensaje específico.
- Las llaves sin cerrar en un diccionario se detectan correctamente.

Esto te ahorrará muchísimo tiempo de depuración cuando recién estás aprendiendo.

### F-strings más potentes (3.12)

Python 3.12 mejoró las f-strings de dos maneras importantes:

1. **Expresiones más complejas dentro de `{}`**: ahora puedes usar comillas del mismo tipo dentro de la expresión (antes había que cambiar la comilla exterior).
2. **F-strings reutilizables**: las definiciones de f-string se pueden reutilizar varias veces sin la limitación anterior, y puedes anidarlas.

```python
frutas = "manzanas"
print(f"Me gustan las {frutas}")     # Me gustan las manzanas
```

### Hacia adelante

Como viste en el capítulo 1, el lenguaje sigue evolucionando: eliminación del GIL (3.13+), errores más claros, f-strings más potentes y el ecosistema escrito en Rust (uv, Ruff, Polars). Todo lo que aprendas en estos fundamentos es estable y seguirá sirviéndote: el Python moderno **amplía** lo que ya sabes, no lo rompe.

---

## 9. Resumen y conceptos clave

En este capítulo diste el salto de "saber de qué se trata Python" a "escribir Python". Dominaste las expresiones y las declaraciones, las variables y los tipos de dato, la conversión entre ellos, el abanico de operadores y el formateo de cadenas. Y no te quedaste atrás: cerraste con lo mejor del **Python moderno** — las f-strings y el operador *walrus* — más un vistazo a lo último que está llegando al lenguaje.

Repasa con esta lista y asegúrate de que cada punto te resulta familiar antes de continuar:

- [ ] Una **expresión** devuelve un valor; una **declaración** ejecuta una orden completa.
- [ ] En la notebook cada expresión muestra su resultado; en un *script* hay que usar `print()`.
- [ ] Python es de **tipado dinámico** y **fuertemente tipado**; los tipos son clases (`int`, `float`, `complex`, `bool`, `str`, `None`).
- [ ] Los tipos **inmutables** (`int`, `float`, `str`, `complex`, `bool`) no se pueden modificar tras crearse.
- [ ] Conversión con `int()`, `float()`, `complex()`, `str()`, `bool()`; no toda conversión es válida.
- [ ] `bool(0)`, `bool("")` y `bool(None)` son `False`; casi cualquier otro valor es `True`.
- [ ] `==` compara **valores**; `is` compara **identidad**; con `None` se usa `is None`.
- [ ] Operadores: aritméticos, de relación, lógicos, de pertenencia (`in`), de identidad (`is`), de asignación y de bits.
- [ ] Por la representación binaria, `0.1 + 0.1 + 0.1 != 0.3`; se compara con `round()` o tolerancia.
- [ ] Enteros: se escriben en decimal, binario (`0b`), octal (`0o`) y hexa (`0x`); se leen con `int(x, base)` y se muestran con `bin()`, `oct()`, `hex()`.
- [ ] Los `int` tienen **precisión arbitraria**; no se desbordan como en otros lenguajes.
- [ ] `float` tiene valores especiales `nan`, `inf` y `-inf`; `nan != nan`, se detectan con `math.isnan()` / `math.isinf()`.
- [ ] `Decimal` (módulo `decimal`) da **precisión exacta** para dinero/contabilidad; se construye desde cadenas.
- [ ] Complejos (`3 + 4j`): partes `.real`/`.imag`, operaciones normales, módulo `cmath`.
- [ ] `(-2) ** (1/2)` saca raíz cuadrada y devuelve un `complex`; `math.sqrt()` no acepta negativos (`ValueError`).
- [ ] Convención de **nombres de variables**: `snake_case`, descriptivos, constantes en `MAYÚSCULAS`; los nombres respetan mayúsculas/minúsculas.
- [ ] No usar las **palabras reservadas** de Python (`if`, `for`, `def`, `class`, `True`…) como nombres de variable (`SyntaxError`).
- [ ] `input(mensaje)` pide texto y **siempre devuelve una `str`**; convertir con `int()`/`float()` cuando se necesita un número.
- [ ] Funciones introductorias: `print`, `input`, `len`, `dir`, `help`, `isinstance`, `round`, `id`, `range`.
- [ ] `str` guarda **caracteres**; `bytes` guarda **enteros (0–255)**; se convierten con `.encode()` / `.decode()`.
- [ ] Codificaciones: `utf-8` (estándar), `latin-1` y `ascii`; en archivos y red se debe indicar la codificación.
- [ ] Las **f-strings** (`f"..."`) son la forma recomendada de formatear cadenas (Python 3.6+).
- [ ] Las `str` son **inmutables** (`texto[0] = "x"` da `TypeError`); modificar "crea" una cadena nueva.
- [ ] **Slicing** de secuencias: `texto[inicio:fin:paso]` (`fin` exclusivo; índices negativos del final; `[::-1]` invierte).
- [ ] El operador **walrus** `:=` asigna un valor y lo usa en la misma expresión (Python 3.8).
- [ ] Python 3.10+ mejora los mensajes de error; Python 3.12 potencia las f-strings.

Con este repertorio ya puedes leer y escribir programas con la sintaxis moderna del lenguaje. En el próximo capítulo vas a controlar el **flujo de tu programa**: las decisiones con `if`/`elif`/`else`, los bucles que repiten trabajo, y la **coincidencia de patrones** (`match`/`case`). Ahí terminarás de armar las piezas para resolver problemas de verdad.
