# Capítulo 4 — Control de flujo: decisiones, bucles y coincidencia de patrones

En el capítulo anterior armaste el vocabulario: variables, tipos, operadores y cadenas. Pero un programa que solo ejecuta líneas de arriba hacia abajo, sin decidir ni repetir, sirve de poco. Este capítulo te da las dos piezas que faltan: **tomar decisiones** (condicionales) y **repetir trabajo** (bucles). Además cerramos con la **coincidencia de patrones** (`match`/`case`), la forma moderna que Python 3.10+ ofrece para distinguir entre muchas opciones. Con esto, ya podrás escribir programas de verdad.

---

## 1. Los bloques, la indentación y los comentarios

Antes de hablar de condicionales hay que entender cómo Python delimita los "trozos" de código. En lenguajes como C se usaban llaves `{ }`; en Python, la **indentación forma parte de la sintaxis**.

```python
if condicion:
    print("este bloque está indentado")
print("este print ya no pertenece al if")
```

- El bloque de código "tributario" de una estructura (`if`, `while`, `for`, `def`…) es todo lo que esté **indentado** debajo de ella.
- Por convención (PEP 8) se usan **cuatro espacios** por nivel, nunca tabuladores mezclados.
- Si una estructura necesita un bloque y no lo tiene, tendrás un `IndentationError`.

Los **comentarios** también forman parte del código, pero Python los ignora por completo:

```python
print("Hola")   # esto es un comentario
```

Todo lo que siga a `#` hasta el final de la línea no se ejecuta: sirve para documentar.

> **Importante:** en Python la indentación *es* el código. Un `if` con su bloque mal indentado no es "código feo": directamente falla o cambia el significado. Respeta siempre los cuatro espacios y la consistencia.

---

## 2. La sentencia `if`: decidir en tu programa

La palabra reservada **`if`** evalúa una expresión lógica (del capítulo anterior: los operadores de relación y lógicos) y, si da `True`, ejecuta el bloque indentado. Si da `False`, lo ignora y continúa el flujo normal.

```python
<flujo principal>
if <expresión lógica>:
    <bloque inscrito al if>
<flujo principal>
```

![Diagrama de flujo: if simple](if-simple.png)

### El `if` simple

```python
numero = 4

if numero % 2 == 0:
    print("el número", numero, "ES PAR")

cuadrado = numero ** 2
print("El cuadrado de {} es: {}".format(numero, cuadrado))
```

El flujo sigue después del `if` haya o no entrado en el bloque.

> **Dato clave:** en un programa real, `numero` vendría de `input()`, y `input()` **siempre devuelve una cadena**. Para comparar con números conviene convertir apenas se lee: `numero = int(input("ingrese un número: "))`. Si el texto no es numérico, `int()` lanza un `ValueError`.

### `if`...`else`: el plan B

Si la condición da `False`, puedes ejecutar otro bloque con `else`:

```text
if <expresión lógica>:
    <bloque inscrito al if>
else:
    <bloque inscrito al else>
```

```python
usuario = "admin"
contraseña = "1234"

if usuario == "admin" and contraseña == "1234":
    print("Bienvenido al sistema")
else:
    print("usuario o contraseña incorrecta")

print("siga aprendiendo")
```

> **Dato clave:** El `else` evalua cuando la condicion del `if` da `False` pero no es de uso obligatorio su uso. 


### `if`...`elif`...`else`: más de una condición

Con **`elif`** (else-if) puedes encadenar condiciones. Python las evalúa **en orden** y ejecuta el bloque de la **primera** que sea `True`; si ninguna lo es, ejecuta `else`.

```python
a = 5
b = 3
c = 7

if a > b and a > c:
    print(a, "es el mayor")
elif b > a and b > c:
    print(b, "es el mayor")
elif c > b and c > a:
    print(c, "es el mayor")
else:
    print("son iguales")
```

> **Dato clave:** el orden importa. Python ejecuta el **primer** `True` que encuentre y **no sigue evaluando** los demás `elif`. Para los rangos de edad, por ejemplo, conviene ir de mayor a menor para no pisarse:

```python
edad = 25

if 110 >= edad >= 65:
    print("sos adulto mayor")
elif 64 >= edad >= 18:
    print("sos adulto")
elif 18 > edad > 0:
    print("sos menor de edad")
else:
    print("edad incorrecta")
```

### La instrucción `pass`

`pass` **no hace nada**, pero Python exige un bloque en muchos lugares. Se usa como "marcador de posición" mientras estás pensando la lógica:

```python
if condicion:
    pass          # implementación pendiente
```

Sin `pass` (ni nada indentado), tendrías un `IndentationError`.

### `bool` es numérico: `True` vale `1`

Recordá del capítulo anterior que `True == 1` y `False == 0`. Eso permite **sumar condiciones** para contar cuántas se cumplen. Un ejemplo clásico: calificar un examen de 10 preguntas cuenta los aciertos `True`.

```python
p1 = True; p2 = True; p3 = True; p4 = True; p5 = False
p6 = True; p7 = True; p8 = True; p9 = True; p10 = False

aciertos = p1 + p2 + p3 + p4 + p5 + p6 + p7 + p8 + p9 + p10   # 8

if aciertos == 10:
    print("Excelente!!!")
elif aciertos == 9:
    print("Casi Excelente")
elif aciertos == 8:
    print("Muy Bien")
# ...
```

No es la forma más elegante de contarlo (en la Parte III verás listas y funciones que simplifican esto), pero útil para recordar que **los booleanos suman**.

---

## 3. El bucle `while`: repetir mientras…

Con **`while`** repites un bloque **mientras** una condición sea `True`. Cuando la expresión lógica da `False`, el flujo continúa después del bloque.

```python
<flujo principal>
while <expresión lógica>:
    <bloque inscrito a while>
<flujo principal>
```

![Diagrama de flujo: while](while.png)

```python
i = 1
while i <= 5:
    print(i)
    i += 1          # ¡sin esta línea, el bucle nunca termina!
```

```python
i = 100
while i >= 20:
    print(i)
    i -= 10
```

> **Dato clave:** quien "se olvida" de actualizar la variable de control crea un **bucle infinito**. Todo `while` necesita que la condición llegue a `False` en algún momento: incrementando, decrementando o cambiando una *bandera* (una variable de corte).

### El patrón de la bandera

Un uso muy común es repetir hasta que el usuario ingrese algo especial ("q" de *quit*). En el notebook interactivo leeríamos con `input()`; para probar sin escribir a mano, lo simulamos con una lista:

Para eso usamos dos funciones que se llevan bien: `iter(lista)` envuelve la lista en un **recorredor** (un *iterador*), un objeto que va entregando los elementos de a uno, como quien va sacando latas de una caja. Y `next(...)` le pide el siguiente elemento. En este capítulo es solo un *truco de prueba* para simular lo que escribiría una persona con `input()`; todavía no hace falta que profundices: lo vamos a desarrollar completo en el capítulo 5.

```python
entrada = ""
suma = 0
simulacion = ["a", "b", "q"]          # simula 3 ingresos: dos válidos y "q" de corte
iter_sim = iter(simulacion)

while suma < 3 and entrada != "q":
    entrada = next(iter_sim)
    print("Clave:", entrada)
    suma += 1
    print("Intento %d." % suma)

print("Utilizaste %d intentos." % suma)
```

La condición combina dos cosas: un tope de intentos (`suma < 3`) y la bandera (`entrada != "q"`). Basta que una falle para salir del bucle.

Los bucles también tienen un `else`... pero es tan contraintuitivo y tan poco explicado que le dedicamos su propia sección más adelante.

---

## 4. Interrupciones: `continue`, `break` y `exit()`

A veces necesitas cortar el flujo desde *adentro* del bucle. Python ofrece tres herramientas.

### `continue`: saltar a la siguiente vuelta

**`continue`** termina de forma prematura la iteración actual: vuelve a evaluar la condición del bucle, ignorando todo lo que venía después dentro del bloque.

![Diagrama de flujo: continue](while-continue.png)

```python
entrada = ""
suma = 0
fallido = 0
simulacion = ["q", "1", "15"]        # con "q" se saltea el conteo de fallidos
iter_sim = iter(simulacion)

while suma < 3:
    suma += 1
    print("Intento N°:{:-^15}".format(suma))
    entrada = next(iter_sim)
    print("Clave:", entrada)
    if entrada == "q":
        continue                     # no cuenta como fallido
    fallido += 1

print("Tuviste {} intentos fallidos.".format(fallido))
```

### `break`: cortar el bucle por completo

**`break`** termina de inmediato el bucle *más interno* en el que se encuentra y el flujo continúa después de él.

![Diagrama de flujo: break](while-break.png)

```python
suma = 0
simulacion = ["1", "23", "q"]
iter_sim = iter(simulacion)

while suma < 3:
    entrada = next(iter_sim)
    if entrada == "q":
        break                        # salimos del bucle acá
    suma = suma + 1
    print(f"Intento {suma}.")

print("Tuviste {} intentos." .format(suma))   # el while ya terminó (por break)
```

> **Dato clave:** la combinación `while True:` + `break` es un patrón idiomático de Python: un bucle "infinito" que se corta explícitamente cuando corresponde (una opción del menú, un error de entrada, etc.).

### `exit()`: terminar el programa

**`exit()`** termina la ejecución del programa por completo y cierra el intérprete. Es la salida "dura": nada de lo que sigue se ejecuta.

```python
entrada = ""
suma = 0

while suma < 3:
    entrada = input("Clave: ")
    if entrada == "q":
        break
    elif entrada == "s":
        print("en un script normal se cerraría el intérprete")
        exit()                        # se termina todo
    suma = suma + 1
    print("Intento %d." % suma)

print("Tuviste %d intentos." % suma)
```

![Diagrama de flujo: exit](while-exit.png)

> **Importante:** dentro de una **notebook de Jupyter**, `exit()` detiene el kernel y hay que esperar a que levante uno nuevo. Por eso en las notebooks del curso se reemplaza por `break`. En tus scripts `.py` normales, `exit()` funciona como esperás.

---

## 5. El bucle `for`...`in`: recorrer objetos iterables

El `while` repite *mientras se cumpla una condición*. El **`for`** repite *una vez por cada elemento* de un objeto iterable, sin preocuparte por índices ni variables de control.

```python
for <contador> in <objeto iterable>:
    <bloque>
```

**Objetos iterables** son los que pueden entregar sus elementos uno a uno: `str`, `list`, `tuple`, `dict`, `set`, `frozenset`, `bytes`… (la mayoría ya los viste como tipos; en la Parte III los estudiarás como estructuras de datos).

```python
for letra in "Hola":
    print(letra)
# H
# o
# l
# a
```

### La función `range()`: iterar números

Para contar de a números, `range()` genera la secuencia:

- `range(n, m, s)`: desde `n` hasta **menos de** `m`, en pasos de `s`.
- `range(n, m)`: desde `n` hasta *menos de* `m`, de a 1.
- `range(m)`: desde `0` hasta *menos de* `m`, de a 1.

Ojo con el detalle: **`range` excluye el límite superior** (`< m`).

```python
for contador in range(5, 9):
    print(contador)        # 5 6 7 8

for contador in range(3, 11, 2):
    print(contador)        # 3 5 7 9

for contador in range(26, 10, -4):
    print(contador)        # 26 22 18 14  (pasos negativos sirven para decrecer)
```

```python
suma = 0
for x in range(1, 101):
    suma += x
print("Suma:", suma)                    # 5050 (la famosa suma de Gauss)
```

### `enumerate()`: índice y elemento a la vez

Cuando necesitás **el índice** además del elemento, `enumerate(iterable)` entrega pares `(índice, elemento)` en cada vuelta. Es la forma idiomática de recorrer una lista "con posición".

```python
comidas = ["Entrada", "Cena", "Postre"]

for index, item in enumerate(comidas):
    print(index, item)
# 0 Entrada
# 1 Cena
# 2 Postre
```

Podés indicar desde qué número arranca: `enumerate(comidas, start=1)`.

---

## 6. El `else` en los flujos de control: el tema que casi nadie enseña

Aquí paramos la marcha para un tema que casi ningún tutorial muestra, y que genera alguno de los errores más difíciles de encontrar. Spoiler: **todos** los flujos de control de Python tienen `else` — menos `match`/`case`.

Ya conocés el `else` del `if`. Pero ojo: en los **bucles** (y en `try`, como verás en la Parte IV) el `else` **no significa "si no pasa la condición"**. En un bucle significa *si el bucle terminó sin interrupción*, es decir, **sin `break`**.

### `for`/`else`: la búsqueda que "avisa si no encontró"

El caso de uso clásico es buscar algo en una secuencia y avisar si **no** lo encontraste:

```python
numeros = [1, 3, 5, 7]

for n in numeros:
    if n % 2 == 0:
        print("Encontré un par:", n)
        break
else:
    print("No hay ningún par en la lista")
```

- Si el `for` encuentra un par, entra al `if`, `break` corta el bucle y el `else` **no se ejecuta**.
- Si el `for` recorre toda la lista sin encontrar pares, se agota con normalidad y entonces el `else` **sí se ejecuta**: "No hay ningún par en la lista".

Hacé la misma búsqueda sin `else` y vas a necesitar una *bandera* extra. El `for`/`else` te evita esa variable.

### `while`/`else`: avisar que se agotaron los intentos

El ejemplo de las claves: si el usuario nunca ingresa "q" (o sea, no hubo `break`), el bucle se agota y el `else` dispara el aviso:

```python
suma = 0
simulacion = ["1", "23", "2"]        # sin "q": el bucle se agota
iter_sim = iter(simulacion)

while suma < 3:
    entrada = next(iter_sim)
    if entrada == "q":
        break
    suma = suma + 1
    print(f"Intento {suma}.")
else:
    print(f"Tuviste {suma} intentos fallidos.")   # corre porque no hubo break

print("continúa el flujo del programa")
```

> **Dato clave:** la regla de oro: el `else` de un bucle se ejecuta **si y solo si** el bucle terminó por su cuenta (la condición dio `False`, o el iterable se agotó). Si saliste por `break`, el `else` **no corre** — sin importar cuántas vueltas se dieron.

### El error silencioso: un `else` mal indentado NO da error de sintaxis

Acá está el peligro que casi nadie advierte. Mirá este código. ¿Qué creés que hace?

```python
numeros = [1, 3, 5, 7]

for n in numeros:
    if n % 2 == 0:
        print("Encontré un par:", n)
        break
    else:                     # ← pegado al `if`, no al `for`
        print(n, "es impar")
```

Parece que alguien quiso escribir el `else` del `for` ("avisar si no hay pares"). Pero por un desliz de indentación, el `else` quedó **adentro del bucle, alineado con el `if`**. Resultado: Python no se queja (no hay `SyntaxError`), y si ejecutás, vas a ver `"1 es impar"`, `"3 es impar"`, `"5 es impar"`, `"7 es impar"`… Cada vuelta, el `else` del `if` decide si imprimir "impar". La lógica que el programador quiso (avisar una sola vez que no había pares) **nunca** ocurre.

La versión correcta tiene el `else` **a la misma altura que el `for`**:

```python
numeros = [1, 3, 5, 7]

for n in numeros:
    if n % 2 == 0:
        print("Encontré un par:", n)
        break
else:                         # ← alineado con el `for`: recién ahora es el else del bucle
    print("No hay ningún par")
```

> **Importante:** un `else` mal indentado **no es un error de sintaxis** — Python lo interpreta como el `else` de un `if` interno (válido) y el programa corre igual, pero con otra lógica. Es un **error de lógica silencioso**: el programa "funciona", no tira errores, y el bug puede pasar meses desapercibido. La regla es simple: el `else` del bucle va **a la misma altura que su `while`/`for`**, jamás pegado a un `if` interno.

### ¿En qué flujos de control hay `else`? En todos… menos en `match`/`case`

| Flujo de control | ¿Tiene `else`? | Qué significa su `else` |
|:-----------------|:--------------:|:------------------------|
| `if` / `elif`    | ✅ Sí | Se ejecuta si ninguna condición dio `True`. |
| `while`          | ✅ Sí | Se ejecuta si el bucle terminó **sin** `break`. |
| `for`            | ✅ Sí | Se ejecuta si se agotó el iterable **sin** `break`. |
| `try` / `except` | ✅ Sí (Parte IV) | Se ejecuta si **no** hubo excepción. |
| `match` / `case` | ❌ **No** | No existe: se usa el comodín `case _`. |

> **Dato clave:** esta es una de las rarezas del lenguaje que más confunde a quien viene de C, Java o JavaScript. El `else` no es exclusivo del `if`: acompaña a **todos** los flujos de control de Python excepto `match`/`case`. Pensalo como "lo que pasa si lo anterior no ocurrió": sin condición verdadera (if), sin `break` (while/for), sin excepción (try), o sin coincidencia (`case _`). Una vez que lo ves así, el `else` de los bucles deja de ser un misterio y se vuelve una herramienta.

---

## 7. Coincidencia de patrones: `match`/`case` (Python 3.10+)

Ya que dominás `if`/`elif`/`else`, estás listo para una herramienta **moderna** del lenguaje: la **coincidencia de patrones** (`match`/`case`). Compara un valor con varios **patrones en orden** y ejecuta el bloque del **primero** que coincida. Es más declarativa y legible que una larga cadena de `if`/`elif` cuando distinguís entre muchos casos.

### Sintaxis general

```python
match <expresión>:
    case <patrón 1>:
        ...
    case <patrón 2>:
        ...
    case _:            # comodín: cualquier otro caso
        ...
```

### `if`/`elif`/`else` vs `match`/`case`

El mismo flujo, expresado de las dos maneras. Primero con condicionales clásicos:

```python
variable = "python3.10"

if variable == "python2":
    print("estamos en python 2")
elif variable == "python3":
    print("estamos en python 3")
else:
    print(variable)
```

Y la versión con `match`/`case`, que expresa la intención de forma más directa:

```python
variable = "python3.10"

match variable:
    case "python3.10":
        print("usando Python 3.10")
    case "python3":
        print("usando Python 3")
    case "python2":
        print("usando Python 2")
    case _:
        print("opción no válida")
```

> **Dato clave:** el **comodín `_`** es el equivalente del `else` (de hecho, `match`/`case` es el **único** flujo de control de Python que no tiene `else`: se usa `case _`). Atrapa cualquier valor que no coincidió con los patrones anteriores. Si ningún `case` coincide y no existe el comodín, simplemente no pasa nada.

### Patrones básicos (sin estructuras de datos)

En esta etapa alcanza con cuatro patrones (en la Parte III, al conocer listas y diccionarios, aparecerán patrones de secuencia y mapeo):

- **Literales**: `case "python3":` compara por igualdad.
- **Alternativas** con `|`: `case "si" | "sí" | "SI":` coincide con cualquiera de ellos.
- **Comodín** `_`: captura cualquier otro valor.
- **Variables y guardas** con `if`: `case x if x > 10:` exige, además, una condición sobre el valor capturado.

#### Alternativas con `|`

```python
respuesta = "sí"

match respuesta:
    case "si" | "sí" | "SI":
        print("respuesta afirmativa")
    case "no" | "NO":
        print("respuesta negativa")
    case _:
        print("respuesta inválida")
```

#### Variables y guardas

Un patrón variable captura el valor, y la **guarda** `if` agrega una condición extra para decidir si el caso aplica:

```python
n = -5

match n:
    case x if x > 0:
        print("positivo")
    case x if x < 0:
        print("negativo")
    case 0:
        print("cero")
    case _:
        print("otro valor")
```

#### Un menú de opciones

El ejemplo típico de "flujo del programa" con `match`: un menú donde cada opción dispara una acción distinta. En un programa real la opción vendría de `input()`; acá la fijamos para concentrarnos en el flujo:

```python
opcion = "b"

match opcion:
    case "a":
        print("Ejecutando opción A ...")
    case "b":
        print("Ejecutando opción B ...")
    case "c" | "salir" | "s":
        print("Saliendo del programa ...")
    case _:
        print("Opción no reconocida")
```

> **Dato clave:** los patrones se evalúan **en orden** y se ejecuta el bloque del **primero** que coincida. Cuanto más específico sea el patrón, más arriba debería estar.

---

## 8. Resumen y conceptos clave

Con este capítulo tu repertorio pasa de "saber escribir Python" a "python que decide y repite". Tomaste el control absoluto del flujo de ejecución.

Repasa con esta lista y asegúrate de que cada punto te resulta familiar antes de continuar:

- [ ] La **indentación** (4 espacios) delimita los bloques; sin ella: `IndentationError`.
- [ ] Los **comentarios** con `#` documentan y no se ejecutan.
- [ ] `if`/`elif`/`else` evalúa condiciones **en orden** y ejecuta el bloque de la **primera** que sea `True`; `else` atrapa el resto.
- [ ] `pass` es un bloque vacío: marcador de posición.
- [ ] `True` vale `1` y `False` vale `0`: se pueden sumar condiciones.
- [ ] `while` repite **mientras** la condición sea `True`; actualizar la variable de control (o bandera) evita bucles infinitos.
- [ ] `continue` salta a la siguiente vuelta; `break` corta el bucle; `exit()` termina el programa (en notebooks de Jupyter detiene el kernel).
- [ ] El `else` de un bucle se ejecuta **solo si el bucle terminó sin `break`** (condición dio `False` / iterable agotado), sin importar cuántas vueltas dio.
- [ ] Un `else` mal indentado (pegado a un `if` interno) **no da error de sintaxis**: es un error de lógica silencioso y difícil de detectar.
- [ ] El `else` existe en todos los flujos de control: `if`, `while`, `for`, `try` … **excepto `match`/`case`**, que usa `case _`.
- [ ] `for` recorre **cualquier objeto iterable** (`str`, `list`, `tuple`, `dict`, `set`, `bytes`…).
- [ ] `iter()` y `next()`: aparecen en las simulaciones de entrada como *truco de prueba*; su explicación completa te espera en el capítulo 5.
- [ ] `range(n, m, s)` produce `n <= i < m` en pasos de `s` (pasos negativos para decrecer).
- [ ] `enumerate(iterable)` da el par `(índice, elemento)` en cada vuelta del `for`.
- [ ] `match`/`case` (Python 3.10+) compara contra **patrones en orden**; `case _` es el comodín (el único flujo sin `else`).
- [ ] Patrones básicos: literales, alternativas con `|`, variables con guardas `if`.

En el próximo capítulo empieza una de las partes más ricas del libro: las **estructuras de datos**. Vas a aprender a agrupar valores en `listas`, `tuplas`, `diccionarios` y `conjuntos` — y con lo que viste de slicing y colecciones en el capítulo 3, el camino será mucho más corto. De paso, ahí vas a entender por fin el truco del `iter()`/`next()` que venimos usando para simular la entrada.