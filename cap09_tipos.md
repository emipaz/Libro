# Capítulo 9 — Type hints: el plano que documenta cada función

¿Viste cuando una receta dice *"2 tazas de harina"*? No hace falta que la receta lo diga para que el cocinero eche harina — pero a nadie le gusta adivinar. Los **type hints** (*"pistas de tipo"* o *anotaciones de tipo*) son eso: etiquetas que le dicen a vos, al programador de al lado y a tu editor **qué tipo de ingrediente espera cada hueco de una función**. La parte más rara del truco: **no cambian ni una línea de lo que el programa hace**. Y aun así, van a cambiar todo cómo lo escribís.

Este capítulo es la pieza que convierte a tus funciones en *documentación viva*: qué entra, qué sale, en la propia firma. La buena noticia es que ya tenés todo lo que se necesita para entenderlas — las armaste, las hiciste anidadas, las agrupaste en lambdas. Ahora solo les ponemos el plano encima.

---

## 1. Qué son (y qué no son)

Python es de **tipado dinámico**: un parámetro puede recibir lo que sea, sin que nadie le diga nada. Los type hints no cambian eso — son **anotaciones opcionales** que le llevan la cuenta a *otras personas* (vos, el de al lado, tu editor, un programa paciente llamado *type checker*) de los tipos que *intencionaste*. Python los guarda en un diccionario, pero jamás los impone:

```python
def saludar(nombre: str, veces: int) -> str:
    return (nombre + "! ") * veces

saludar("Ana", 3)       # "Ana! Ana! Ana! "
saludar.__annotations__
# {'nombre': <class 'str'>, 'veces': <class 'int'>, 'return': <class 'str'>}
```

Fijate: `nombre: str` dice *"espera un texto"*, `veces: int` *"espera un entero"*, y `-> str` *"esto devuelve un texto"*. Pero nadie va a cobrar esa promesa en la ejecución: si mañana le pasás algo que no coincide, Python **no** te va a frenar en la llamada — el tipado dinámico sigue intacto (si explota, explota *adentro* del cuerpo, por la lógica que escribiste, no por la anotación). Lo que sí cambia es que tu editor te va a marcar la línea, y el *type checker* te va a avisar antes de que el error llegue a la calle.

> **Dato clave:** los hints son **etiquetas, no esposas**. No alteran la ejecución (solo quedan guardados en `__annotations__`); documentan y permiten que las herramientas te avisen. Son opcionales — y son **la norma en la industria**.

---

## 2. La sintaxis en tres lugares

El patrón es siempre el mismo: nombre, dos puntos, tipo. Y aparece en tres lugares de la vida:

```python
edad: int = 30               # variable: nombre : tipo = valor
nombre: str = "Luis"
altura_m: float = 1.75
es_estudiante: bool = True

def es_mayor(edad_persona: int) -> bool:    # parámetro:...  y retorno: -> tipo
    return edad_persona >= 18

es_mayor(20)                  # True
es_mayor.__annotations__
# {'edad_persona': <class 'int'>, 'return': <class 'bool'>}
```

- **En un parámetro**: `edad_persona: int`.
- **En el retorno**: la **flecha** `-> bool` va entre el cierre del paréntesis y los dos puntos. La función `es_mayor` promete "recibir un entero y devolver un booleano".
- **En una variable**: `edad: int = 30` le anota el tipo a un nombre cualquiera del mapa.

Cero magia: son expresiones de Python que el intérprete evalúa y archiva. Que no te asuste el que haya *tres* tipos y medio mirándote: son `str` (texto), `int` (entero), `float` (decimal), `bool` (verdadero/falso) — los cuatro amigos del capítulo 3.

> **Dato clave:** anotar no te obliga a nada: la línea `nombre: str = 42` es legal y corre. Los hints son una promesa para el ojo y el checker, no una ley para el intérprete.

---

## 3. Colecciones: `list[int]`, `dict[str, int]`, `tuple[float, float]`

Esto es lo que convierte a los hints en oro: **las colecciones se anotan entre corchetes con el tipo de lo que contienen**. `list[int]` se lee "una lista de enteros"; `dict[str, int]` "un diccionario de claves texto a valores enteros"; `tuple[float, float]` "una tupla de dos decimales".

```python
numeros = [3, 1, 4]
def sumar_lista(datos: list[int]) -> int:
    return sum(datos)

def contar_matriz(matriz: list[list[int]]) -> dict[str, list[int]]:
    return {"filas": [len(fila) for fila in matriz]}

sumar_lista(numeros)                 # 8
contar_matriz([[1, 2], [3, 4, 5]])   # {'filas': [2, 3]}
```

Y anidan como las cajas: `list[list[int]]` es *una lista de listas de enteros* — exactamente tu matriz. `dict[str, list[int]]` es *un diccionario que mapea textos a listas de enteros*. Leés de adentro hacia afuera y se desarma la aterradora llama (también conocida como firma).

Una nota de arqueología, para cuando veas código viejo: antes de Python 3.9 no había tipos genéricos nativos y se usaba el módulo `typing` — `typing.List[int]`, `typing.Dict[str, int]`, `typing.Tuple[float, float]`. Hoy se usa `list[int]` directamente, y el `typing` queda reservado para los casos que verás en las secciones 5 a 7.

> **Dato clave:** la colección anota **lo que contiene**: `list[int]` = "lista de enteros", `list[list[int]]` = "lista de listas de enteros", `dict[str, int]` = "claves texto, valores enteros". Si no sabés qué tipo es (o te da igual), `Any` dice "cualquier cosa existe y no me comprometo".

---

## 4. Uniones con `|` y el poder del `None`

Desde Python 3.10, el operador `|` ("pipa") expresa **"uno u otro"**: `int | str` se lee "un entero o un texto". Y el patrón más usado de toda la industria es justamente con el que ya te cruzaste: **`X | None`**, la forma moderna de decir *"un X, o nada"*.

```python
def formatear(valor: int | str | None) -> str:
    if valor is None:
        return "sin valor"
    return str(valor)

formatear(42)       # '42'
formatear("hola")   # 'hola'
formatear(None)     # 'sin valor'
```

Lo lindo es que la anotación **documenta lo que tu código ya hace**. ¿Te acordás de la trampa del default mutable del capítulo 7? La solución era `lista=None` para no compartir la lista. Ahora el contrato queda escrito en la propia firma:

```python
def agregar_ingrediente(ingrediente: str, lista: list[str] | None = None) -> list[str]:
    if lista is None:
        lista = []
    lista.append(ingrediente)
    return lista
```

Leé la firma como se escribe: *"recibe un texto y, opcionalmente, una lista de textos — o nada — y devuelve una lista de textos"*. Ese "o nada" es exactamente el `None` que manejaba tu `if`. La anotación no lo hace mágico: lo vuelve **visible**.

> **Dato clave:** `X | None` = "un X o nada" — el patrón del parámetro opcional (necesita Python 3.10+). En código más viejo vas a verlo como `Optional[X]`, y `int | str` como `Union[int, str]`: misma idea, nueva tinta.

---

## 5. `Callable`: el tipo de las funciones

Las funciones también tienen tipo. No es "entero" ni "texto", es **su propia firma**: qué recibe y qué devuelve. Se escribe `Callable[[int], int]` — corchetes para la lista de parámetros, y afuera, después de la flecha interna, el retorno. Es la sección 5 del capítulo 8, tipada:

```python
from typing import Callable

def aplicar(func: Callable[[int], int], valor: int) -> int:
    return func(valor)

def duplicar(n: int) -> int:
    return n * 2

aplicar(duplicar, 21)      # 42 → a "func" le pedimos "una función que recibe int y devuelve int"
```

`aplicar` ahora dice: *"recibo una función que va de entero a entero, y un entero; devuelvo un entero"*. Y lo más gracioso: **tu lambda ya tenía tipo sin saberlo**. `lambda n: n * 2` es un `Callable[[int], int]`; `key=lambda nombre: nombre.lower()` de la sección 6 de Funciones II es un `Callable[[str], str]`. El tipo de una función es su plano: no importa cómo se llama, importa qué entra y qué sale.

Dos vecinos útiles del mismo barrio: `Iterable[int]` ("algo por donde se puede iterar y produce enteros": lista, tupla, set, lo que sea) y `Mapping[str, int]` ("algo que mapea texto a entero", sin atarlo a un dict concreto):

```python
from typing import Iterable, Mapping

def sumar_iterable(valores: Iterable[int]) -> int:
    return sum(valores)

def primera_clave(m: Mapping[str, int]) -> str:
    return next(iter(m.keys()))

sumar_iterable([1, 2, 3])       # 6 → sirve lista, tupla, set...
primera_clave({"a": 1, "b": 2}) # 'a'
```

> **Dato clave:** `Callable[[param1, param2], retorno]` — la lista interna son los parámetros, el tipo externo es el retorno. Una función de orden superior se anota diciendo qué tipo de función espera: `aplicar(func: Callable[[int], int], ...)`. Eso es "tipos como contratos": `Iterable` y `Mapping` piden *comportamiento*, no una clase en particular.

---

## 6. `TypeVar`: funciones genéricas que no repiten tipo

Imaginate una función tan simple que el tipo de entrada define el de salida. La *identidad* del capítulo 8 recibía `T` y devolvía `T`; tiparla con un tipo fijo sería mentir, y con `Any` perder información. Para eso existe el **`TypeVar`** ("variable de tipo"): declarás una letra que representa "un tipo, cualquiera, pero *el mismo* en toda la firma":

```python
from typing import TypeVar

T = TypeVar("T")

def identidad(elemento: T) -> T:
    return elemento

identidad(10)        # T = int
identidad("hola")    # T = str
identidad([1, 2])    # T = list[int]
```

`identidad` promete *"lo que entre es lo que sale"*. ¿Y para qué sirve en la vida real? Para las funciones que no repiten lógica ni tipo. Pensá el `agregar_ingrediente` de hace un momento, pero ahora genérico: si la lista pudiera ser de cualquier cosa, `def primero(coleccion: list[T]) -> T` devuelve el primer elemento sin saber (ni querer saber) de qué es. Es el DRY del capítulo 7 aplicado a los *tipos*: no vuelvas a escribir el tipo de la salida cuando el de la entrada ya lo dijo.

> **Dato clave:** `TypeVar` engancha tipos: "lo que entra, sale". `def identidad(elemento: T) -> T` no fija `T`, pero obliga a que el retorno le haga eco a la entrada. Es el tipo de la función que respeta el principio *no te repitas* — ahora en clave de tipos.

---

## 7. `Literal` y el refinamiento con `isinstance`

Dos herramientas finas que vas a encontrar esparcidas en las bibliotecas más nuevas.

**`Literal`** restringe las *opciones* de un valor, como un semáforo que solo acepta sus tres colores:

```python
from typing import Literal

def semaforo(estado: Literal["rojo", "amarillo", "verde"]) -> str:
    return f"El semáforo está en {estado}"

semaforo("verde")      # El semáforo está en verde
```

Acordate: no es un control — es un contrato. Ninguno de los dos es *runtime*: `semaforo("azul")` corre, pero el checker te va a saltar a la vista. Fijate que es el espíritu del `match/case` del capítulo 5 trasladado a los tipos: "solo estos valores entran por esta puerta".

**El refinamiento (*narrowing*)** es la otra cara de la moneda: el checker te deja **afinar** el tipo después de un `isinstance`, porque ya sabe qué rama estás pisando:

```python
def procesar(valor: int | str) -> str:
    if isinstance(valor, int):
        return f"Entero: {valor * 2}"      # acá "valor" es int, el checker lo sabe
    else:
        return f"Texto: {valor.upper()}"   # acá "valor" es str

procesar(21)        # Entero: 42
procesar("hola")    # Texto: HOLA
```

El tipo declarado era "entero o texto"; después del `if isinstance(valor, int)` el checker refinó: en la primera rama es entero, en la segunda, texto. Ese mismo mecanismo es el que hace que tu editor autocompleté `.upper()` solo en la rama del texto.

> **Dato clave:** `Literal["a", "b"]` dice "solo estos valores" (es un contrato para el checker, no un filtro); el **narrowing** con `isinstance` le dice al checker *"acá el tipo es este"* dentro de cada rama. Ver madurar `int | str` en dos ramas con tipos concretos es de las sensaciones más poderosas del tipado estático.

---

## 8. Herramientas: quién lee estos planos

Los hints valen lo que valen sus lectores. Hay dos tipos de lectores, y conviene conocer a ambos:

- **El editor (IDE)** — VS Code, PyCharm y amigos los usan para el **autocompletado** y para marcar los llamados equivocados *mientras escribís*: si invocás `aplicar("hola", 5)` ahí mismo te subraya que `"hola"` no es una función. Es el primer beneficio que vas a sentir, desde el día uno.
- **Los *type checkers*** — programas que auditan tu código *sin ejecutarlo* (análisis estático): **mypy**, **pyright** (el motor de VS Code) y **pylance**. Son el "otro par de ojos" que corre en la terminal o en el fondo del editor y detecta el tipo equivocado en la línea exacta. Se corren tipo `mypy mis_archivos.py`.

Y en el *runtime*, las anotaciones no quedan solo guardadas: hay una forma prolija de leerlas, `get_type_hints`, que resuelve las uniones complejas (mejor que el `__annotations__` crudo):

```python
from typing import get_type_hints

def mezclar(a: int, b: str) -> str:
    return str(a) + b

mezclar.__annotations__
# {'a': <class 'int'>, 'b': <class 'str'>, 'return': <class 'str'>}
get_type_hints(mezclar) == mezclar.__annotations__   # True
```

No los confundas con **validación**: los hints documentan, no controlan. Cuando un programa quiera *verificar* sus datos de verdad (¿este dict vino bien formado?), eso ya no es tarea de los hints — es el terreno de herramientas como **pydantic**, que van a aparecer en el libro *después* de la programación orientada a objetos, porque ahí finalmente vas a tener todas las piezas. Gancho al futuro: los hints de este capítulo son el combustible de ese entonces.

> **Dato clave:** los hints no tienen sentido solos: existen para el **IDE** (autocompletado, errores en vivo) y para los **type checkers** (`mypy`, `pyright`). No son validación de runtime — eso es otro capítulo (y otra herramienta, pydantic).

---

## 9. Docstrings: la otra documentación (y sus estilos)

Los hints son la documentación para las herramientas. Pero a la persona que va a leer tu código al año siguiente todavía le debés una línea: el **docstring**, ese `"""..."""` que conocés desde el capítulo 7. Y ahí también hay convenciones para elegir. No hay un style obligatorio oficial, pero se consagraron tres — más el plano que ya usás. Mirá las cuatro variantes con la misma función tipada de la sección 5:

**Estilo Google** — el que más se lee de corrido; cada parámetro describe *qué es*, sin repetir el tipo (la firma ya lo dice):

```python
def sumar_iterable(valores: Iterable[int]) -> int:
    """Suma todos los valores de un iterable.

    Args:
        valores: los números a sumar, en cualquier iterable.

    Returns:
        El total de la suma.
    """
```

**Estilo NumPy / SciPy** — el de los proyectos científicos, más rígido y formateado, con el tipo anotado bajo cada parámetro:

```python
def sumar_iterable(valores: Iterable[int]) -> int:
    """Suma todos los valores de un iterable.

    Parameters
    ----------
    valores : iterable
        los números a sumar.

    Returns
    -------
    int
        El total de la suma.
    """
```

**Estilo Sphinx / reStructuredText** — el clásico de siempre (la propia documentación de Python lo usa), con las etiquetas pegadas al margen; es el abono perfecto de la herramienta **Sphinx**, que genera sitios de documentación:

```python
def sumar_iterable(valores: Iterable[int]) -> int:
    """Suma todos los valores de un iterable.

    :param valores: los números a sumar.
    :type valores: Iterable[int]
    :returns: El total de la suma.
    :rtype: int
    """
```

**El estilo plano** — el que ya venís usando desde el capítulo 7: una línea o un párrafo al natural. Perfectamente válido para scripts propios, y muchas veces alcanza:

```python
def sumar_iterable(valores: Iterable[int]) -> int:
    """Suma los valores de un iterable, sea lista, tupla o set."""
```

¿Cuál es el mejor? El que use la gente de tu equipo — y sobre todo: **uno solo**. La regla de oro de la documentación es la *consistencia*: no importa tanto qué estilo elijas como que en todo el proyecto los docstrings se lean igual. Mirá los estilos, elegí el que te caiga más cómodo y quedáte con él.

Y acá está el brindis entre los dos temas del capítulo: en el estilo Google de arriba el tipo **no se repite** — `Args:` solo describe, y la información de tipos vive en la firma (`valores: Iterable[int] -> int`). El estilo NumPy lo vuelve a escribir (`valores : iterable`) y el Sphinx también (`:type valores: Iterable[int]`), duplicando lo que la firma ya dijo. Por eso, en un proyecto con type hints, el estilo Google queda tan liviano: los hints te sacan de encima la repetición y el docstring se queda con puro significado.

Y cuando tu función va a vivir en un **módulo** que otra persona va a importar, al docstring le gusta sumar dos bloques más (el estilo Google los acepta de lo lindo): **`Raises:`** para las excepciones que la función puede lanzar — las del capítulo 6 — y **`Example:`** con un mini uso real, escrito tal como se teclearía en el intérprete:

```python
def convertir_a_entero(texto: str, base: int = 10) -> int:
    """Convierte un texto en un entero.

    Args:
        texto: el texto a interpretar como número.
        base: la base numérica (por defecto, 10).

    Returns:
        El entero resultante.

    Raises:
        ValueError: si el texto no puede interpretarse en esa base.

    Example:
        >>> convertir_a_entero("255")
        255
        >>> convertir_a_entero("ff", base=16)
        255
    """
    return int(texto, base)

convertir_a_entero("ff", base=16)   # 255
convertir_a_entero("hola")          # ValueError: invalid literal for int() with base 10: 'hola'
```

Fijate que es el **contrato completo en una sola pieza**: qué entra, qué sale, qué te puede explotar en la cara (`Raises:`) y un uso de ejemplo. Probá `help(convertir_a_entero)` en el intérprete: te lo pinta entero. Y hay un regalo escondido que a los módulos les encanta: el `Example:` usa el prefijo `>>>` del intérprete, un formato que el módulo **`doctest`** de la biblioteca estándar sabe *ejecutar*, para comprobar que el docstring no esté mintiendo. Docstring + type hints + doctests: casi un manual de uso que se prueba solo — el estándar de oro de un módulo bien hecho.

Y estos estilos no son solo estética. Les hablan a herramientas: los *docstring linters* (como `pydocstyle`) revisan que tus docstrings sigan las reglas del estilo que elegiste, y los generadores de documentación (**Sphinx**, **pdoc**) convierten *docstring + type hints* en una página web de referencia. El `help()` del capítulo 7 y el atributo `__doc__` muestran el docstring crudo, siempre al pie de la letra que escribiste.

> **Dato clave:** los hints le hablan a las herramientas; el **docstring** le habla a la persona que va a leer el código. Hay estilos consagrados (Google, NumPy, Sphinx/reST) y ningún estilo es "el obligatorio": la regla de oro es la **consistencia**. En este libro, de acá en adelante, los docstrings de ejemplo van en **estilo Google** — para que tengas un espejo de cómo se ve en la práctica — y los de módulo suman **`Raises:`** y **`Example:`**: el contrato completo para quien importa tu código.

---

## 10. Resumen y ejercicios

**Resumen — el checklist del tipado:**

- [ ] Los type hints son **anotaciones opcionales**: documentan, no restringen; el tipado dinámico sigue intacto.
- [ ] Se escriben en tres lugares: parámetro (`nombre: str`), retorno (`-> str`) y variable (`edad: int = 30`).
- [ ] Python las guarda en `__annotations__`; `get_type_hints()` las lee resueltas.
- [ ] Las colecciones anotan su contenido: `list[int]`, `dict[str, list[int]]`, `tuple[float, float]`.
- [ ] La pipa une tipos: `int | str` = "uno u otro"; `X | None` = "un X o nada" (el patrón del opcional, recién aprendido en cap07).
- [ ] `Callable[[int], int]` es el tipo de una función: parámetros entre corchetes, retorno afuera.
- [ ] `Iterable` y `Mapping` describen *comportamiento* sin atarse a una clase.
- [ ] `TypeVar` engancha tipos: lo que entra, sale (`def identidad(elemento: T) -> T`).
- [ ] `Literal[...]` restringe valores; el `isinstance` **refina** el tipo rama por rama (narrowing).
- [ ] Los hints brillan con lectores: el IDE autocompleta y avisa en vivo; `mypy`/`pyright` auditan estático.
- [ ] Los docstrings tienen estilos consagrados (Google, NumPy, Sphinx/reST) + el plano; ninguno es obligatorio: la regla de oro es la **consistencia**. Los de módulo suman `Raises:` y `Example:` (`>>>`), que hasta `doctest` puede ejecutar.


**Ejercicios:**

1. **La vitrina de `__annotations__`**: anotá `saludar(nombre: str, veces: int) -> str` y mostrá su `__annotations__` en un `for clave, valor in ...`. Después probá que anotar no ata: definí `presentar(nombre: str) -> str` que haga `return f"Hola, {nombre}"`, llamala con `presentar(42)` y mirá que corre igual — una promesa que nadie cobró.
2. **Lista con longitud mínima**: definí `claves_largas(palabras: list[str], longitud_minima: int) -> list[str]` que devuelva las palabras más largas que `longitud_minima`, tipada. Probala con `["py", "python", "a"]` y `longitud_minima=3`.
3. **Promedio tipado**: reescribí el `promedio(*notas)` del capítulo 7 con hints — ojo, el parámetro variádico se anota raro: `def promedio(*muestras: float) -> float`. Verificá que `__annotations__` diga lo que esperás.
4. **Lo que entra, sale**: definí con `TypeVar` un `duplicar(elemento: T) -> list[T]` que devuelva `[elemento, elemento]`, y probala con un `int`, un `str` y una lista. Mirá cómo el checker "infiere" el tipo sin que lo anotes.
5. **Fábrica tipada**: retípá con hints tu `crear_multiplicador(n)` de Funciones II (la fábrica de closures) — la firma que vas a escribir dice *"recibe un `float`, devuelve una función que recibe `float` y devuelve `float`"*: `def crear_multiplicador(n: float) -> Callable[[float], float]`. Probala con `por_2(10)`.
6. **Narrowing**: definí `medir(valor: int | str) -> float` que devuelva `valor * 2.5` si es entero y `len(valor)` si es texto, usando `isinstance`. Mostrá que las dos ramas corren.
7. **El docstring que te representa**: documentá la función `procesar` del capítulo (la del narrowing con `isinstance`) con un docstring en **estilo Google**, y verificá que se lea con `print(procesar.__doc__)`.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. La vitrina de __annotations__
def saludar(nombre: str, veces: int) -> str:
    return (nombre + "! ") * veces

for clave, valor in saludar.__annotations__.items():
    print(f"{clave}: {valor.__name__}")
# nombre: str
# veces: int
# return: str

def presentar(nombre: str) -> str:
    return f"Hola, {nombre}"

print(presentar(42))     # Hola, 42   ← anotaste str, pasaste int, corrió igual

# 2. Lista con longitud mínima
def claves_largas(palabras: list[str], longitud_minima: int) -> list[str]:
    return [p for p in palabras if len(p) > longitud_minima]

claves_largas(["py", "python", "a"], 3)   # ['python']

# 3. Promedio tipado
def promedio(*muestras: float) -> float:
    return sum(muestras) / len(muestras)

promedio(7, 8)                        # 7.5
promedio.__annotations__
# {'muestras': <class 'float'>, 'return': <class 'float'>}

# 4. Lo que entra, sale
from typing import TypeVar
T = TypeVar("T")

def duplicar(elemento: T) -> list[T]:
    return [elemento, elemento]

duplicar(10)        # [10, 10]    → T = int
duplicar("hola")    # ['hola', 'hola']   → T = str

# 5. Fábrica tipada
from typing import Callable

def crear_multiplicador(n: float) -> Callable[[float], float]:
    def multiplicar(x: float) -> float:
        return x * n
    return multiplicar

por_2 = crear_multiplicador(2)
por_2(10)          # 20.0

# 6. Narrowing
def medir(valor: int | str) -> float:
    if isinstance(valor, int):
        return valor * 2.5     # rama entero
    return len(valor)          # rama texto

medir(4)       # 10.0
medir("arbol") # 5.0

# 7. Docstring en estilo Google
def procesar(valor: int | str) -> str:
    """Interpreta un valor según su tipo.

    Args:
        valor: un entero para duplicar, o un texto para pasar a mayúsculas.

    Returns:
        El resultado listo para mostrarse.
    """
    if isinstance(valor, int):
        return f"Entero: {valor * 2}"
    return f"Texto: {valor.upper()}"

print(procesar.__doc__)
# Interpreta un valor según su tipo.
#
#     Args:
#         valor: un entero para duplicar, o un texto para pasar a mayúsculas.
#
#     Returns:
#         El resultado listo para mostrarse.
```

Y con esto, cada función del libro tiene su plano. Un detalle para despedirnos: los hints son *silenciosos y amables* — nunca rompen una ejecución, solo sostienen una lámpara. Guardalos así: si un día tu propia función *depende* de que la entrada sea de un tipo, eso se controla con la lógica que ya sabés (capítulos de errores y `match`), no con una anotación.

Ahora sí, sobre un programa que se escribe solo. En los próximos capítulos vas a ver la idea más hermosa de toda la Parte V — y usa exactamente las herramientas que construiste: funciones que se expresan en sus propios términos, hasta reducir el problema a nada. Bienvenido a la **recursión**.