# Capítulo 11 — El paradigma funcional: describir qué, no cómo

El capítulo 10 te dejó con una promesa de despedida: la "familia **funcional**" — `map`, `filter`, `reduce` y la biblioteca `functools` — primos directos de las listas por comprensión y de los generadores. Llegó el último capítulo de la Parte V, y es el más filosófico.

Hasta acá, cuando programabas, le **decías a la máquina cómo** hacer las cosas paso a paso: un `for` que recorre, un `if` que decide, una variable que acumula. Ese estilo se llama **imperativo**, y es el que trajiste del capítulo 4. El **paradigma funcional** propone un giro de cámara: en vez de detallar el recorrido, le **describís qué** querés lograr con cada elemento, dejando los detalles del recorrido a las herramientas.

Mirá este acertijo mínimo, la "suma del doble de los números pares" de una lista. Primero, como venías haciendo (imperativo):

```python
datos = [3, 8, 11, 16, 21, 24]

suma = 0
for numero in datos:
    if numero % 2 == 0:
        suma += numero * 2
print(suma)   # 96
```

Después, con la comprensión del capítulo 5 (nada que explicar, ya es tuya):

```python
suma = sum([numero * 2 for numero in datos if numero % 2 == 0])
print(suma)   # 96
```

Y ahora la versión funcional — no la intentes entender todavía, es la foto de portada del capítulo:

```python
from functools import reduce

suma = reduce(
    lambda acumulado, numero: acumulado + numero,
    map(lambda numero: numero * 2,
        filter(lambda numero: numero % 2 == 0, datos)),
)
print(suma)   # 96
```

Las tres responden 96. Las tres *expresan* lo mismo — pero fijate dónde vive la diferencia: 

- El código imperativo usa un bucle for explícito y un acumulador manual (suma += ...).
- La comprensión de lista usa for, pero incrustado dentro de una expresión única que devuelve la lista directamente, sin acumulador manual.
- La versión funcional no usa for en absoluto: compone funciones (filter, map, reduce) donde cada una hace un trabajo específico.

La clave es que el bucle no es el centro del código. Ni en la comprensión ni en la versión funcional le decís a la máquina "recorre esto", sino "aplica estas transformaciones".

Y ahí está el truco del paradigma: **no le decís a la máquina que recorra, le decís qué hacer con lo que recorre.** Este capítulo te enseña a leer y a escribir esas tres herramientas: `map` (mapear), `filter` (filtrar) y `reduce` (acumular).

---

## 1. ¿Qué es un paradigma? (y por qué Python no te obliga a elegir)

"Paradigma" viene del griego y significa algo así como "modelo de ejemplo": en ciencia, un paradigma es la lente desde la cual una comunidad mira el mundo. En programación, un **paradigma de programación** es *la manera de pensar los problemas al escribir código*: qué considerás un dato, qué considerás una función, de qué está hecha una "unidad de trabajo".

Los lenguajes suelen nacer con un paradigma favorito. Python es distinto: es **multiparadigma**. Por un lado, *todo es un objeto* — incluso las funciones (capítulo 8) son objetos que se pasan, se guardan y se devuelven. Por otro, el lenguaje no te impone un estilo: el mismo problema se resuelve como te quede más cómodo. Los paradigmas principales, son:

- **Imperativo** (tus capítulos 3-7): escribís *órdenes* — "hacé esto, después aquello, y si pasa esto, lo otro". El estado vive en las variables, y `for`, `while` e `if` son la materia prima. Se llama **procedimental** cuando agrupás las órdenes en funciones.
- **Funcional** (este capítulo): en vez de órdenes, *composición de funciones*. Funciones **puras** —misma entrada, misma salida, sin efectos de lado— que se transforman y encadenan. Los tres verbos `map`, `filter`, `reduce`.
- **Orientado a objetos**: juntás *datos y comportamiento* en un mismo objeto. Ya lo usás sin saberlo: las `str`, `list` y `dict` del capítulo 3 son objetos, y sus *métodos* (`.split()`, `.capitalize()`, `.items()`) son esa fusión. La Parte VI lo formaliza con clases.
- **Declarativo**: describís *qué* resultado querés y el sistema decide *cómo* lograrlo (el estilo de las consultas SQL). Se asoma en las comprensiones y expresiones generadoras, que describen un resultado sin detallar el recorrido.

La tesis del capítulo — *"describir qué, no cómo"* — es la diferencia entre el imperativo y el resto: no le contás a la máquina **cómo** recorrer, le decís **qué** hacer con lo que encuentre. Y un spoiler que conecta todo lo visto: ya programaste funcional sin saberlo — el `sorted(palabras, key=lambda p: len(p))` del capítulo 8 le *pasa una función* a otra función, igual que `map` o `filter`.

> **Dato clave — función pura:** la pieza "legal" del paradigma. Una **función pura** depende solo de sus argumentos y no produce efectos externos: mismas entradas, misma salida, sin tocar globales, sin imprimir, sin pedir datos. `celsius_a_fahrenheit` es pura; un `for` que acumula en una variable global no. Las puras son predecibles, fáciles de probar y *memorizables* — esa pureza es lo que en la sección 10 le permite a `lru_cache` guardar resultados sin riesgo.

---

## 2. Tres verbos: mapear, filtrar, acumular

Todo el paradigma funcional se apoya en tres operaciones que ya hacés con las manos, solo que con nombres propios:

| Verbo | Función | Qué hace |
|-------|---------|----------|
| **mapear** | `map(funcion, secuencia)` | Aplica una función a **cada elemento** |
| **filtrar** | `filter(funcion, secuencia)` | Se **queda con los que** la función aprueba |
| **reducir** | `reduce(funcion, secuencia)` | **Pliega** toda la secuencia a un solo valor |

Volvé a la foto de portada con esta clave. Se lee de **adentro hacia afuera**, en tres pasos: primero *qué se queda* (los pares), después *qué se hace con cada uno* (duplicar) y por último *cómo se junta todo* (sumar):

```python
pares      = filter(lambda numero: numero % 2 == 0, datos)      # 1º: qué se queda
duplicados = map(lambda numero: numero * 2, pares)              # 2º: qué se hace
resultado  = reduce(lambda acumulado, numero: acumulado + numero,
                    duplicados)                                 # 3º: cómo se junta
print(resultado)   # 96
```

Tres funciones, tres preguntas: *"¿qué me llevo?, ¿con qué transformo?, ¿cómo lo junto?"*. Y un detalle que ya te debería sonar de antes: `filter` y `map` **no devuelven listas** — devuelven *iteradores*, la misma cinta perezosa del capítulo 10 (y de `iter()`/`next()` del capítulo 5). Por eso la versión funcional del final la envolvés en `reduce`, que se encarga de jalarholala. A lo largo del capítulo vamos a destapar cada verbo, compararlo con la comprensión que ya dominás, y terminar con las reglas de oro para elegir.

> **Dato clave:** `map` y `filter` son *perezosos* y *descartables* (capítulo 10): emiten cada valor a demanda y solo una vez. No "cometen" la transformación hasta que alguien consume el iterador — un `for`, un `next()`, o un `list()`.

---

## 3. `map`: aplicar una función a cada elemento

**`map(funcion, secuencia)`** devuelve un iterador que, al consumirlo, entrega el resultado de aplicar `funcion` a cada elemento de `secuencia`. Empezá por el caso elemental:

```python
datos = [3, 4, 5, 6, 7, 8, 9]

dobles = map(lambda x: x * 2, datos)
print(type(dobles))   # <class 'map'>  → es un iterador, no una lista

print(next(dobles))   # 6
```

Y como todo iterador, lo podés volcar a donde quieras sin que sufra: `list()`, `tuple()`, `set()` o `frozenset()` (**recorriendo cada uno, una vez, sobre el mismo `map()`**):

```python
cuadrados = [x * x for x in range(5)]          # 0, 1, 4, 9, 16  (comprensión, tuya)
por_dos = list(map(lambda x: x * 2, range(5))) # 0, 2, 4, 6, 8
print(por_dos)
```

La magia es que `funcion` puede ser **cualquier función** — la tuya de `def`, no solo una lambda. En el capítulo 8 las funciones se volvieron ciudadanos de primera (las pasás como argumentos, sin `()`): acá es el momento de cobrarlas. Un ejemplo del material del curso: convertir temperaturas. Fijate que definís la transformación una vez y **mapeás** la lista entera con ella:

```python
def celsius_a_fahrenheit(c: float) -> float:
    """Convierte grados Celsius a Fahrenheit."""
    return (c * 1.8) + 32

temperaturas = (-32, 0, 12, 25, 32, 100, 150)
farenheit = tuple(map(celsius_a_fahrenheit, temperaturas))
print(farenheit)
# (-25.6, 32.0, 53.6, 77.0, 89.6, 212.0, 302.0)
```

También podés mapear con **métodos** ya existentes. `str.capitalize` es la función que pasa un texto a mayúscula inicial; pasándola a `map` junto con tu lista de nombres, transformás todos de una:

```python
nombres = "emiliano adrian belen luca federico micaela".split()
mayusculas = set(map(str.capitalize, nombres))
print(mayusculas)
# {'Luca', 'Belen', 'Adrian', 'Emiliano', 'Micaela', 'Federico'}
```

Y un plus que pocos conocen: `map` acepta **varias secuencias**, y aplica la función tomando un elemento de cada una, por posición — como una cremallera:

```python
def sumar(a, b):
    return a + b

uno = (1, 2, 3, 4, 5, 6, 7, 8, 9)
dos = [9, 8, 7, 6, 5, 4, 3, 2, 1]
resultado = list(map(sumar, uno, dos))
print(resultado)   # [10, 10, 10, 10, 10, 10, 10, 10, 10]
```

> **Dato clave:** `map(funcion, a, b)` es ideal cuando ya *tenés* una función lista (de `def` o de la biblioteca). Si la transformación es una expresión corta que vas a escribir ahí mismo, en la sección 5 vas a ver que la comprensión suele leer mejor.

---

## 4. `filter`: qué se queda

**`filter(funcion, secuencia)`** recorre la secuencia, le pregunta a `funcion` por cada elemento, y se queda **solo con los que responden `True`**. Es un pase de selección. El ejemplo de los negativos:

```python
def es_negativo(numero):
    return numero < 0

valores = [-3, -2, 0, 1, 9, -5, -45, 25, 35, -102, -0.5]

negativos = filter(es_negativo, valores)
print(next(negativos))   # -3         → pide de a uno, como siempre
print(list(negativos))   # [-2, -5, -45, -102, -0.5]   → el resto (se agotó la cinta)
```

Con una lambda, lo mismo:

```python
valores = [-3, -2, 0, 1, 9, -5, -45, 25, 35, -102, -0.5]
print(tuple(filter(lambda x: x < 0, valores)))   # (-3, -2, -5, -45, -102, -0.5)
```

### El truco de la verdad: `filter(None, …)`

El predicado de `filter` no tiene que devolver `True`/`False` literal: `filter` se queda con **todo lo que sea *verdadero*** — y recordá la regla de los capítulos 3-4: `0`, `0.0`, `""`, `None`, `[]`, `{}` son *falsy* (falsos). Por eso existe un atajo sorprendente: **`filter(None, secuencia)` descarta todos los falsos**:

```python
datos_mixtos = (0, 1, 2, "", False, None, [], [1, 2, 3], {})

print(list(filter(None, datos_mixtos)))   # [1, 2, [1, 2, 3]]  → se fueron los falsy
print(list(filter(bool, datos_mixtos)))   # lo mismo: bool(x) de la mano
```

`filter(None)` y `filter(bool)` hacen lo mismo. Y no es lo mismo que `filter(lambda x: x is not None)`, que solo descarta los `None` (deja los `0`, `""`, `False`). Detalle de precisión quirúrgica, pero de ahí salen algunos de los bugs más lindos de la profesión — ahora los ves venir.

### Filtrar un diccionario: se filtran sus claves

Recorrer un dict con un `for` recorre sus **claves** (capítulo 5). Lo mismo pasa con `filter`: filtra claves, y adentro del predicado accedés a los valores. Con un "menú" de metales del material del curso:

```python
metales = {
    "hierro": 10, "cobre": 11, "zinc": 25, "plata": 100,
    "oro": 500, "aluminio": 50, "plomo": 25,
}

preciosos = list(filter(lambda metal: metales[metal] >= 100, metales))
print(preciosos)   # ['plata', 'oro']

print([m for m in metales if metales[m] >= 100])   # la comprensión equivalente
```

Y un entrenamiento de lectura: dos condiciones juntas. Se puede encadenar filters, combinarlas en un solo predicado con `and`, o traducirlo a comprensión — todo lo mismo:

```python
print(list(filter(lambda m: "o" in m,
                  filter(lambda m: metales[m] >= 51, metales))))   # ['oro']
print(list(filter(lambda m: metales[m] >= 51 and "o" in m, metales)))  # ['oro']
print([m for m in metales if metales[m] >= 51 and "o" in m])            # ['oro']
```

> **Dato clave:** `filter` también es un iterador — perezoso y de un solo uso. Si necesitás recorrer el resultado varias veces, guardalo en una lista o tupla con `list()`/`tuple()`.

---

## 5. La alternativa que ya conocés: comprensiones

El capítulo 5 te dio las **listas por comprensión**, y probablemente ya lo adivinaste: son el primo directo de `map` y `filter` combinados. Las dos líneas siguientes producen **exactamente el mismo resultado**:

```python
datos = [3, 8, 11, 16, 21, 24]

comprension = [x * 2 for x in datos if x % 2 == 0]
funcional = list(map(lambda x: x * 2, filter(lambda x: x % 2 == 0, datos)))

print(comprension == funcional)   # True
print(comprension)                # [16, 32, 48]
```

La equivalencia general es tan limpia que se escribe sola:

- `list(map(f, secuencia))` ↔ **`[f(x) for x in secuencia]`** — el *map* es la *expresión* de la comprensión.
- `list(filter(f, secuencia))` ↔ **`[x for x in secuencia if f(x)]`** — el *filter* es el *`if`* de la comprensión.

Entonces, ¿`map`/`filter` o comprensión? La guía de oro del paradigma en Python:

- **Comprensión**: cuando la transformación y el filtro son expresiones cortas y querés la secuencia completa (una lista). Es la versión más legible — no hay que anidar ni mirar a contrarreloj.
- **`map`/`filter`**: cuando ya tenés *la función* definida (`map(celsius_a_fahrenheit, ...)`, `filter(es_negativo, ...)`), cuando querés **encadenar** varias etapas cuanto las hayas nombrado, o cuando querés el iterador perezoso sin construir nada (sección 10 del capítulo 10).

Ambos, además, tienen versión perezosa: la **expresión generadora** `(x * 2 for x in datos if x % 2 == 0)` es a la comprensión lo que `map`/`filter` al `list()` — misión económica para cuando recorrés una sola vez.

> **Buenas prácticas:** si podés escribir la comprensión en una línea que se lea sola, usala. Si ya tenés la función a mano o la cadena es larga, `map`/`filter`. La peor elección es forzar `lambda` cuando una comprensión diría lo mismo en menos palabras.

---

## 6. Filtrar y mapear: un pipeline

El patrón que cierra el paradigma no es `map` ni `filter` por separado: es **encadenarlos** — primero me quedo con lo que importa, después transformo. A esa cadena de etapas el mundo de los datos la llama **pipeline** (del inglés *pipeline*, "conducto" — nada de plomería, aunque la palabra literal diga eso). El acertijo del arranque, ahora sin `reduce`, para ver el pipeline de cerca:

```python
datos = [3, 8, 11, 16, 21, 24]

pipeline = list(map(lambda x: x * 2, filter(lambda x: x % 2 == 0, datos)))
print(pipeline)   # [16, 32, 48]

# el mismo pipeline con comprensión (se lee igual pero sin girar la cabeza):
print([x * 2 for x in datos if x % 2 == 0])   # [16, 32, 48]

# y la versión perezosa, si después solo vamos a recorrer una vez:
perezosa = (x * 2 for x in datos if x % 2 == 0)
print(sum(perezosa))   # 96
```

Fijate cómo cambia la lectura entre los dos estilos: en el pipeline funcional leés de **derecha a izquierda** ("primero filtra, después mapea"); en la comprensión, la lógica va de **adentro hacia afuera**, más cerca del orden natural del pensamiento. Ambas están bien — eliges por legibilidad.

> **Buenas prácticas:** un pipeline `map`/`filter` brilla cuando cada etapa es una **función con nombre** (`map(celsius_a_fahrenheit, filter(es_negativo, ...))`): el código se vuelve un relato. Si las etapas son expresiones cortas que inventarías en el momento, la comprensión es la más pythonica — una sola línea que no obliga a leer de atrás hacia adelante. (Y con la expresión generadora tenés la versión económica del capítulo 10.)

---

## 7. `reduce`: plegar todo en un solo valor

`map` transforma elementos; `filter` los selecciona; pero hay un tercer verbo, quizá el menos conocido (por eso vive en una biblioteca aparte): **`reduce(funcion, secuencia)`** va **combinando de a dos** hasta quedarse con un único valor. Vive en `functools`:

```python
from functools import reduce

def multiplicar(a, b):
    return a * b

print(reduce(multiplicar, [1, 2, 3, 4]))   # 24
```

¿De dónde sale el 24? `reduce` toma los dos primeros (`1 * 2 = 2`), mezcla el resultado con el siguiente (`2 * 3 = 6`), y así hasta el final (`6 * 4 = 24`). Es literalmente "doblar" la secuencia sobre sí misma hasta aplastarla a un valor:

```
reduce(multiplicar, [1, 2, 3, 4])
  = multiplicar(multiplicar(multiplicar(1, 2), 3), 4)
  = multiplicar(multiplicar(2, 3), 4)
  = multiplicar(6, 4)
  = 24
```

`reduce` acepta un **tercer argumento**: el *valor inicial*. La acumulación arranca de ahí en vez del primer elemento:

```python
from functools import reduce

print(reduce(lambda a, b: a * b, [1, 2, 3, 4], 10))   # 240 → arranca en 10
print(reduce(lambda a, b: a + " - " + b, ["pan", "leche", "queso"]))
# 'pan - leche - queso'   → incluso sirve para "pegar" textos
```

Acá hay una revelación de escalera al sótano: `sum([1, 2, 3])` es un `reduce(lambda a, b: a + b, [1, 2, 3])` con nombre lindo — y lo mismo `max` y `min`. Los "built-ins" que usás desde el capítulo 3 son reduces especializados. El caso general te lo dejan a vos.

> **Dato clave — la advertencia honesta:** `reduce` es el menos legible de los tres, porque obliga a mantener un estado mental ("¿qué viene acumulado?"). Usalo cuando *tenés* una operación binaria acumulativa clara (multiplicar, sumar, "pegar"). Para sumar números siempre vas a usar `sum`, no `reduce`. La virtud de Python es justamente dar una manera obvia de hacer las cosas — y acá la manera obvia suele ser la comprensión o el bucle.

### La palabra de Guido: la polémica de `reduce`

Hay una historia real que explica por qué `reduce` vive en `functools` y no entre los built-ins. En Python 2, `reduce` era de fábrica, como `sum` o `max`. Cuando Guido van Rossum (el creador de Python) diseñó Python 3 —en ese entonces apodado "Python 3000"— propuso **eliminarla del lenguaje**. Su argumento, resumido de su ensayo *"The fate of reduce() in Python 3000"* (2005), se hizo famoso: *"salvo un par de ejemplos con `+` o `*`, cada vez que veo un `reduce()` con una función no trivial tengo que agarrar lápiz y papel para entender qué quiso hacer"*. La discusión fue pública y caliente dentro de la comunidad; el desenlace fue un acuerdo intermedio: `reduce` **sobrevivió, pero se mudó a `functools`** y dejó de ser built-in en Python 3. Es el eco más literal de la frase que ya conocés: *"there should be one obvious way to do it"*.

Guido opinó en la misma línea sobre `map` y `filter`: con las **listas por comprensión** (esa equivalencia `[f(x) for x in s]` de la sección 5), la mayoría de los usos simples quedan más claros. No están prohibidos — los usamos todo el capítulo — pero la recomendación de la casa es la de las reglas de oro: **comprensión por defecto; `map`/`filter` cuando ya tenés la función o un pipeline largo; `reduce` en `functools` para acumuladores que lo merezcan**. (Y un dato del barrio funcional: los lenguajes puramente funcionales optimizan la recursión de cola — *tail-call optimization* —; Python no la tiene, por eso la recursión profunda del capítulo 10 choca con el límite de ~1000. Python toma del funcional lo que le sirve, con su propia personalidad.)

---

## 8. El mismo problema, tres miradas

Volvé a mirar cómo resolviste "suma del doble de los pares" en tres estilos distintos — luego de este capítulo, los tres son *tuyos*:

```python
datos = [3, 8, 11, 16, 21, 24]

# Imperativo: le digo cómo (miro, decido, acumulo)
suma = 0
for numero in datos:
    if numero % 2 == 0:
        suma += numero * 2

# Comprensión: transformo y filtro en una línea
suma = sum([numero * 2 for numero in datos if numero % 2 == 0])

# Funcional: describo qué (filtrar → mapear → reducir)
suma = reduce(lambda a, n: a + n,
              map(lambda n: n * 2,
                  filter(lambda n: n % 2 == 0, datos)))
print(suma)   # 96, siempre 96
```

La tabla de balance, para decidir de memoria:

| Estilo | Fuerza | Cuidado |
|--------|--------|---------|
| **Imperativo** (`for` + `if`) | Control total; familiar | Muchas líneas de "mecánica" que ocultan el *qué* |
| **Comprensión** | Legible, pythonica, una línea | Se enreda si las etapas son muchas |
| **Funcional** (`map`/`filter`/`reduce`) | Reutiliza funciones, pipelines, perezoso | Menos legible con lambdas largas; `reduce` pide cuidado |

Y las reglas de oro, para cerrar la Parte V: **empezá por la comprensión** (es la más legible); **pasá a `map`/`filter` cuando ya tengas la función o la cadena sea varias etapas**; **reservá `reduce` para acumulaciones binarias claras**. Y si el código se te estaba enredando, recordá que el bucle sigue siendo un ciudadano respetable — el paradigma funcional *amplía* tu caja de herramientas, no prohíbe tu martillo.

> **Para curiosear:** este estilo viene de las familias Lisp y Haskell, donde no hay "bucles" como los conocés. Python toma la idea a su manera: sin renunciar a la legibilidad. Las funciones `map`/`filter`/`reduce` existen en Python moderno, sí, pero la palabra de orden del lenguaje es *"there should be one obvious way"* — por eso los built-ins y las comprensiones conviven en paz con ellas (y la historia de por qué `reduce` no es built-in te espera en la sección 7).

---

## 9. `zip`: unir de a pares

Ahora incluimos una cuarta herramienta, prima de las otras: **`zip(secuencia1, secuencia2, ...)`**, que **prende con un broche** los elementos del mismo índice de varias secuencias, de a dos (o más). Es una cremallera entre listas:

```python
numeros = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
letras = "a b c d e f g h i j".split()

pares = zip(numeros, letras)
print(list(pares))
# [(1, 'a'), (2, 'b'), (3, 'c'), (4, 'd'), (5, 'e'),
#  (6, 'f'), (7, 'g'), (8, 'h'), (9, 'i'), (10, 'j')]
```

Como `map`/`filter`, `zip` es **perezoso**: devuelve un iterador que vas consumiendo. Y tiene un detalle honesto: **yendo hasta donde alcanza** — se termina cuando la secuencia más corta se acaba:

```python
print(list(zip([1, 2, 3], ["a", "b"])))   # [(1, 'a'), (2, 'b')]  → el 3 queda de lado
```

El clásico del capítulo (diccionarios, capítulo 5): **`dict(zip(...))` construye un dict de dos listas en una sola línea** — claves en una, valores en la otra:

```python
print(dict(zip(["nombre", "edad"], ["luca", 10])))
# {'nombre': 'luca', 'edad': 10}
```

> **Dato clave:** `zip` no es estrictamente funcional — es una *combinadora* de iterables, prima de `itertools.product` del capítulo 5 (producto cartesiano: todas las combinaciones) pero uniendo por **posición**, no multiplicando.

---

## 10. `functools`: la caja de herramientas del paradigma

`reduce` vive en `functools`, y no es el único habitante del barrio. **`partial(funcion, valor_fijo)`** te fabrica una función nueva *con algunos argumentos ya puestos* — sin escribir ninguna closure a mano (esa fábrica de multiplicadores del capítulo 8, resuelta por la biblioteca):

```python
from functools import partial

def multiplicar(a, b):
    return a * b

duplicar = partial(multiplicar, 2)      # fija el primer argumento
print(duplicar(10), duplicar(7))        # 20 14

print(partial(pow, 2)(10))              # 1024  → pow(2, 10) con el 2 ya puesto
```

Y ahora una deuda pagada con el capítulo 10. Ahí quedó prometida una técnica para el problema de `fibonacci_rec` (2,6 millones de llamadas porque *recalculaba* lo mismo una y otra vez): la **memoización**, guardar resultados ya calculados. `functools` te la regala con un decorador — **`lru_cache`** ("least recently used", guarda los resultados recientes). Decorar es la magia `@` del capítulo 8, y el resultado es ese mismo fibonacci recursivo, ahora *que no vuelve a calcular nada dos veces*:

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print(fibonacci(35))            # 9227465
print(fibonacci.cache_info())   # CacheInfo(hits=33, misses=36, maxsize=None, currsize=36)
```

Mirá el `CacheInfo`: **36 "misses"** (cálculos que realmente se hicieron) contra los millones del capítulo 10. La primera llamada a cada `fibonacci(n)` se calcula una vez; las demás, `hits`, se contestan de memoria. La recursión que antes daba miedo vuelve a ser defendible — con la receta del capítulo 10 (estructura recursiva) y la memoria de `functools`.

> **Buenas prácticas:** `@lru_cache` es para funciones **puras** (sección 1): mismas entradas, mismas salidas, sin efectos externos. Esa es precisamente la filosofía del capítulo: funciones que describen qué devuelven, y que se pueden *recordar* sin riesgo.

Y una joya más de `functools`, esta para el que apreció los decoradores del capítulo 8: **`wraps`**. Un decorador envuelve la función original en un *envoltorio*; sin ayuda, el envoltorio le "roba" el nombre (`__name__`) y el docstring. `functools.wraps` copia esa identidad de la original al envoltorio, para que `help()` y tu editor sigan viendo a la función real:

```python
from functools import wraps

def decorar(funcion):
    @wraps(funcion)                       # le copia nombre y docstring a envoltorio
    def envoltorio(*args, **kwargs):
        return funcion(*args, **kwargs)
    return envoltorio

@decorar
def saludar():
    """Saluda educadamente."""
    return "hola"

print(saludar.__name__)   # 'saludar'  → sin wraps sería 'envoltorio'
print(saludar.__doc__)    # 'Saluda educadamente.'
```

> **Dato clave — el barrio de `functools`:** `reduce` (acumular), `partial` (fijar argumentos), `lru_cache` (memorizar) y `wraps` (preservar la identidad en decoradores). Cuatro herramientas, cuatro ángulos del mismo ideal: **reutilizar funciones** en vez de reescribirlas.

---

## 11. `any`, `all` y `reversed`: el veredicto y la vuelta

Al paradigma funcional le quedó un veredicto pendiente. `filter` te dice *qué se queda* de una secuencia — pero ¿y si solo necesitás saber **si alguien aprueba** o **si todos aprueban**, sin quedarte con nadie? Para eso existen dos funciones que son como `filter`, pero del veredicto: **`any(secuencia)`** devuelve `True` si **al menos un** elemento es verdadero; **`all(secuencia)`** devuelve `True` si **todos** lo son:

```python
print(any([False, False, True]))   # True  → hay por lo menos un verdadero
print(all([True, True, True]))     # True  → todos verdaderos
print(all([True, False, True]))    # False → uno falso, todo falso
```

Son a los booleanos lo que `sum` y `max` eran a los números (sección 7): **reduces especializados**. `any` pliega la secuencia con un "o" lógico y `all`, con un "y". Y heredan la regla de *verdad* del `filter(None)` (sección 4): no preguntan "¿es exactamente `True`?", sino "¿es *verdadero*?" — por eso `0`, `""`, `[]` y `None` cuentan como falsos.

El uso idiomático se luce cuando les pasás una **expresión generadora** (capítulo 10): no hace falta construir ninguna lista, y además **cortan apenas tienen el veredicto** — `any` deja de mirar en cuanto encuentra un verdadero:

```python
temperaturas = [-5.4, 3.2, 8.9, 12.0]

print(all(t > 0 for t in temperaturas))      # False → el -5.4 hiela el veredicto
print(any(t < 0 for t in temperaturas))      # True  → apareció el -5.4
print(all(t > -100 for t in temperaturas))   # True  → todas aguantan
```

Y un detalle honesto que parece contradicción y es lógica pura: sobre una secuencia **vacía**, `any` da `False` (no hay ningún verdadero) y `all` da `True` (no hay ningún contraejemplo) — la famosa "verdad vacía":

```python
print(any([]))   # False  → ¿alguno cumple? no hay a quién preguntarle
print(all([]))   # True   → ¿todos cumplen? por defecto, sí
```

> **Dato clave:** `any`/`all` son la versión booleana de `filter`: `filter` te entrega a los que pasan; `any`/`all` te contestan solo "¿alguno?" / "¿todos?", sin construir resultados. Validar entradas (¿todos los campos no están vacíos? ¿algún dato está repetido?) se vuelve una línea legible.

La otra función que faltaba en la caja es la **vuelta**: **`reversed(secuencia)`** devuelve un iterador que recorre la secuencia **de atrás hacia adelante** — perezoso y descartable, como todos los del capítulo:

```python
nums = [40, 10, 20, 30]
print(list(reversed(nums)))   # [30, 20, 10, 40]
print(nums)                   # [40, 10, 20, 30]  → la original intacta
```

Tiene dos primas conocidas: el `[::-1]` del capítulo 3 recorre igual pero **construye una copia completa**; el `reverse()` del capítulo 5 invierte **en el lugar** (y solo sirve para listas). `reversed()` es la opción perezosa: no crea nada — va emitiendo de a uno hacia atrás, perfecta para recorrer una sola vez o para alimentar el `zip`/`map` que ya sabés combinar. Por ser perezosa, `list(reversed(...))` y `reversed(...)` pelado no son lo mismo: sin consumo, no hace nada.

---

## 12. Resumen y conceptos clave

Este capítulo cerró la Parte V con el **paradigma funcional**: programar *describiendo qué* en vez de *detallando cómo*. Arrancaste parando la pelota y preguntándote **qué es un paradigma** — y que Python, por ser *multiparadigma* ("todo es un objeto"), te deja elegir entre el imperativo (capítulos 3-7), el funcional (este capítulo), el orientado a objetos (Parte VI) y el declarativo. Sobre esa base conociste los tres verbos — **`map`** (aplicar una función a cada elemento), **`filter`** (quedarse con lo que aprueba un predicado) y **`reduce`** (plegar toda la secuencia a un valor) —, su primo **`zip`** (unir por posición) y la **función pura** como pieza legal del estilo. Viste que las **comprensiones** del capítulo 5 son la alternativa pythonica de `map`+`filter` (mismas equivalencias), que `map`/`filter`/`zip` son **iteradores perezosos y descartables** (capítulos 5 y 10), que `reduce` es el abuelo de `sum`/`max`/`min` — y que la **discusión histórica de Guido** sobre `reduce` terminó exiliándolo a `functools`. La caja de `functools` quedó completa con `partial`, `lru_cache` (la memoización prometida en el capítulo 10) y `wraps`. Y la cerraron el **veredicto** — `any`/`all`, reduces booleanos que cortan apenas saben la respuesta, con su "verdad vacía" sobre secuencias vacías — y la **vuelta** — `reversed`, el iterador perezoso hacia atrás, prima del `[::-1]` y del `reverse()`. Los seis juntos (mapear, filtrar, plegar, unir, veredictar, dar la vuelta) son tu caja para leer y escribir el estilo funcional de Python.

Repasá el checklist antes de dar por terminada la Parte V:

- [ ] **Paradigma** = la manera de pensar los problemas al programar. Python es **multiparadigma**: "todo es un objeto" y no impone un solo estilo.
- [ ] Cuatro paradigmas: **imperativo** (órdenes + estado), **funcional** (composición de funciones puras), **orientado a objetos** (dato + comportamiento; Parte VI), **declarativo** (describís el resultado, el sistema decide el cómo).
- [ ] **Función pura**: mismas entradas → mismas salidas, sin efectos externos; por eso `lru_cache` puede memorizarla.
- [ ] **Paradigma funcional** = describir *qué* hacer con cada elemento, delegando el *cómo* a las herramientas.
- [ ] **`map(funcion, secuencia)`** transforma cada elemento; acepta varias secuencias (una por parámetro de la función).
- [ ] **`filter(funcion, secuencia)`** se queda con los elementos que devuelven verdadero; `filter(None)` descarta todos los *falsy*.
- [ ] `map`, `filter` y `zip` son **iteradores**: perezosos, descartables, se consumen con `next`, `for`, `list()`, `tuple()`, `set()`, `frozenset()`.
- [ ] Equivalencias: `list(map(f, s))` ↔ `[f(x) for x in s]`; `list(filter(f, s))` ↔ `[x for x in s if f(x)]`.
- [ ] **Regla de oro:** comprensión por defecto; `map`/`filter` cuando ya tenés la función o una cadena de etapas; la **expresión generadora** como versión perezosa.
- [ ] **`reduce(funcion, secuencia[, inicial])`** pliega a un valor; de `functools`. `sum`/`max`/`min` son reduces especializados.
- [ ] **La polémica de Guido:** `reduce` dejó de ser built-in en Python 3 y vive en `functools`; su consejo — comprensión por defecto, `map`/`filter` para funciones o pipelines, `reduce` para acumuladores claros.
- [ ] `reduce` pide cuidado: usalo con operaciones binarias acumulativas claras.
- [ ] **`zip(a, b)`** une por posición hasta la secuencia más corta; `dict(zip(claves, valores))` arma un dict.
- [ ] **`functools.partial(funcion, valor)`** fija argumentos (sin escribir closures manuales).
- [ ] **`functools.lru_cache`** memoiza: guarda resultados de funciones puras → el `fibonacci` recursivo deja de recalcular (36 cálculos para `n=35`).
- [ ] **`functools.wraps`** le copia nombre y docstring al envoltorio de un decorador (capítulo 8).
- [ ] **`any(secuencia)`** / **`all(secuencia)`**: el veredicto booleano — "¿alguno es verdadero?" / "¿todos lo son?"; reduces con "o"/"y" que cortan apenas deciden; en secuencias vacías: `False` / `True` (verdad vacía).
- [ ] **`reversed(secuencia)`**: iterador perezoso de atrás hacia adelante — a diferencia de `[::-1]` (copia) y `reverse()` (en el lugar, solo listas).
- [ ] Orden de lectura: el pipeline `filter` → `map` → `reduce` se lee de adentro hacia afuera / de derecha a izquierda.

---

## 13. Ejercicios

1. **Doble de pares, a tu manera**: en `datos = [3, 8, 11, 16, 21, 24]`, escribí las tres versiones (imperativa, por comprensión y funcional con `reduce`) y verificá que las tres dan 96.
2. **Cuadrados con `map`**: con `range(1, 11)`, construí la lista `[1, 4, 9, …, 100]` con `list(map(...))` y también por comprensión. ¿Cuál te gusta más para leer?
3. **Feria de temperaturas**: reutilizá `celsius_a_fahrenheit` y convertí `(0, 25, 100)` a tupla con `map`.
4. **Filtro de nombres**: con `nombres = "emiliano adrian belen luca".split()`, quedate solo con los que empiezan con "b" usando `filter` y la comprensión equivalente.
5. **Metales doble condición**: en `metales` de la sección 4, filtrá los que valen `>= 100` **y** contienen "o" de tres maneras (filters encadenados, un solo `and`, y comprensión). Pista: ¿cuántos quedan?
6. **Pipeline completo**: con `datos = [1, 2, 3, 4, 5, 6, 7, 8]`, sacale el cuadrado a los pares con el pipeline `map`/`filter` y con comprensión; después sumá el resultado con `reduce`.
7. **`reduce` con texto**: uní las palabras `["pan", "leche", "queso"]` en `"pan - leche - queso"` con `reduce`. Y un segundo: product la lista `[1, 2, 3, 4]`.
8. **`zip` + `dict`**: con `claves = ["nombre", "edad", "ciudad"]` y `valores = ["luca", 10, "rosario"]`, armá el dict de tres pares con `dict(zip(...))`. ¿Qué pasa si una lista tiene más elementos que la otra?
9. **Fibonacci sin recalcular**: copiá el `fibonacci` con `@lru_cache` y calculá `fibonacci(35)`. Compará el resultado con los "2,6 millones de llamadas" del capítulo 10.
10. **`partial` al poder**: con `partial`, fabricá una función `potencia_de_2` que calcule potencias de 2 (`potencia_de_2(10)` → `1024`). Pista: `pow`.
11. **El veredicto**: con `notas = [7.5, 4.0, 9.0, 5.5, 10.0]`, decidí con `all` si todas aprueban (6 o más) y con `any` si hay algún 10. Después verificá qué devuelven `all([])` y `any([])`, y explicá esa "verdad vacía" con una frase.
12. **La vuelta**: con `nums = [1, 2, 3, 4, 5]`, construí la lista invertida de tres maneras — `list(reversed(...))`, `nums[::-1]` y `.reverse()` — y decí cuál copia, cuál modifica en el lugar y cuál no toca nada.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. Doble de pares, tres versiones
datos = [3, 8, 11, 16, 21, 24]

suma = 0
for numero in datos:
    if numero % 2 == 0:
        suma += numero * 2
print(suma)                       # 96

print(sum([numero * 2 for numero in datos if numero % 2 == 0]))   # 96

from functools import reduce
print(reduce(lambda a, n: a + n,
             map(lambda n: n * 2,
                 filter(lambda n: n % 2 == 0, datos))))            # 96

# 2. Cuadrados con map y comprensión
print(list(map(lambda x: x ** 2, range(1, 11))))    # [1, 4, 9, ..., 100]
print([x ** 2 for x in range(1, 11)])               # igual

# 3. Feria de temperaturas
def celsius_a_fahrenheit(c: float) -> float:
    return (c * 1.8) + 32

print(tuple(map(celsius_a_fahrenheit, (0, 25, 100))))  # (32.0, 77.0, 212.0)

# 4. Filtro de nombres
nombres = "emiliano adrian belen luca".split()
print(list(filter(lambda n: n.startswith("b"), nombres)))   # ['belen']
print([n for n in nombres if n.startswith("b")])            # ['belen']

# 5. Metales doble condición
metales = {"hierro": 10, "cobre": 11, "zinc": 25, "plata": 100,
           "oro": 500, "aluminio": 50, "plomo": 25}
print(list(filter(lambda m: "o" in m,
                  filter(lambda m: metales[m] >= 100, metales))))   # ['oro']
print(list(filter(lambda m: metales[m] >= 100 and "o" in m, metales)))  # ['oro']
print([m for m in metales if metales[m] >= 100 and "o" in m])           # ['oro']

# 6. Tubería completa (cuadrado de los pares, sumado con reduce)
datos = [1, 2, 3, 4, 5, 6, 7, 8]
print(list(map(lambda x: x ** 2, filter(lambda x: x % 2 == 0, datos))))
# [4, 16, 36, 64]
print([x ** 2 for x in datos if x % 2 == 0])            # [4, 16, 36, 64]
print(reduce(lambda a, b: a + b,
             map(lambda x: x ** 2,
                 filter(lambda x: x % 2 == 0, datos))))  # 120

# 7. reduce con texto y con producto
print(reduce(lambda a, b: a + " - " + b, ["pan", "leche", "queso"]))
# 'pan - leche - queso'
print(reduce(lambda a, b: a * b, [1, 2, 3, 4]))        # 24

# 8. zip + dict
claves = ["nombre", "edad", "ciudad"]
valores = ["luca", 10, "rosario"]
print(dict(zip(claves, valores)))
# {'nombre': 'luca', 'edad': 10, 'ciudad': 'rosario'}
print(list(zip([1, 2, 3], ["a", "b"])))   # [(1, 'a'), (2, 'b')]  → el 3 sobra

# 9. Fibonacci con lru_cache
from functools import lru_cache

@lru_cache(maxsize=None)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print(fibonacci(35))            # 9227465
print(fibonacci.cache_info())   # hits=33, misses=36 (contra 2,6 millones del cap10)

# 10. partial al poder
from functools import partial
potencia_de_2 = partial(pow, 2)
print(potencia_de_2(10))        # 1024

# 11. El veredicto
notas = [7.5, 4.0, 9.0, 5.5, 10.0]
print(all(n >= 6 for n in notas))    # False → el 4.0 y el 5.5 no aprueban
print(any(n == 10 for n in notas))   # True  → hay un 10
print(all([]), any([]))              # True False → sin elementos, no hay contraejemplo

# 12. La vuelta
nums = [1, 2, 3, 4, 5]
print(list(reversed(nums)))   # [5, 4, 3, 2, 1]  → perezoso, no toca la original
print(nums[::-1])             # [5, 4, 3, 2, 1]  → construye una copia
nums.reverse()                # invierte en el lugar (solo listas)
print(nums)                   # [5, 4, 3, 2, 1]
```

Con esto **la Parte V queda completa**: capítulo 7 armaste funciones, capítulo 8 aprendiste a que se recuerden y compongan (closures, lambdas, decoradores), capítulo 9 les pusiste el plano (type hints), capítulo 10 descubriste que se llaman a sí mismas (recursión) y que se pausan (generadores), y capítulo 11 que describen qué, no cómo (funcional). Es un camino que va de la sintaxis a la filosofía — y la filosofía es la que te va a sostener cuando los problemas se vuelvan grandes.

El siguiente capítulo deja las funciones atrás y presenta el estilo de programación más poderoso de Python: la **Programación Orientada a Objetos (POO)**. Y vas a entrar con ventaja: la idea de juntar *datos y comportamiento* en una misma cosa ya la conocés de cerca — porque las strings, listas y diccionarios que llevás usando desde el capítulo 3 son todos objetos, y esos `métodos` (`.split()`, `.capitalize()`, `.items()`) son la primera muestra de esa fusión. Nos vemos en la Parte VI.