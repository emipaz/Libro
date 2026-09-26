# Capítulo 13 — El espejo del objeto: métodos especiales

El capítulo 12 te dejó prometiendo un espejo: tus clases "saben hacer" cosas que nunca programaste, y los nombres con doble guion bajo alrededor esconden esos secretos heredados de `object`. Llegó el momento de mirarlos de frente.

**Los métodos especiales** — también llamados *dunders* (de *double underscore*, doble guion bajo) o *magic methods* — son los que dan a los objetos su comportamiento de "ciudadanos de primera" del lenguaje: que `+` funcione, que `print()` muestre algo lindo, que `len()` y `for` te entiendan. No los llamás por su nombre: los llamás con sintaxis normal (`+`, `print`, `len`), y el intérprete traduce solo. Aprender dunders es aprender **cómo se ve Python desde la piel de un objeto** — y es la única forma de fabricar tipos que se sientan nativos, como las `str`, las `list` o los `int` que usás desde el capítulo 3.

La pieza central del capítulo: un `rango` hecho a medida, con paso decimal y una precisión configurable — y lo vamos a leer con lupa, pulirlo y recién después recorrer el espejo entero.

---

## 1. La vida secreta de los operadores

Cuando el intérprete encuentra un símbolo como `+`, **no realiza aritmética**: llama a un método especial del objeto de la izquierda. `3 + 4` es azúcar de `int.__add__(3, 4)`; `"hola" + " " + "mundo"` es azúcar de `str.__add__(...)`; `[1] + [2]` lo mismo con la clase `list`. Repasá el concepto del capítulo 12: `objeto.metodo(arg)` ≡ `Clase.metodo(objeto, arg)`. El `+` es exactamente eso, con otro disfraz:

```python
print("a" + "b")     # ab      → str.__add__("a", "b")
print([1] + [2])     # [1, 2]  → list.__add__([1], [2])
print(3 + 4)         # 7       → int.__add__(3, 4)
```

Eso explica por qué el capítulo 3 te contaba que cada tipo "sabe" sumarse a su manera: no hay un `+` universal — cada clase trae su `__add__`. Y abre la puerta a una de las rebeldías de la lista del capítulo 12: **la sobrecarga de operadores**. Como "operador" es solo un método, **vos** decidís qué significa para tus objetos. Un clásico para verlo en acción: sumar superficies de cuadrados:

```python
class SuperficieCuadrada:
    """Suma superficies de cuadrados."""

    lado = 1

    def superficie(self):
        """Calcula la superficie de un cuadrado."""
        return self.lado ** 2

    def __add__(self, elemento):
        """Suma: mi superficie más la de otro cuadrado (o un número)."""
        if type(elemento) is SuperficieCuadrada:
            return self.superficie() + elemento.superficie()
        elif type(elemento) in (int, float):
            return self.superficie() + elemento
        raise NotImplementedError("no sé sumar eso a una superficie")

cuadros = (SuperficieCuadrada(), SuperficieCuadrada())
cuadros[0].lado = 25
cuadros[1].lado = 12

print(cuadros[0] + cuadros[1])   # 769  → 625 + 144
print(cuadros[0] + 12)           # 637  → 625 + 12
```

Fijate la forma: `__add__` recibe `self` y al *otro* (`elemento`), verifica de qué tipo es el otro y decide. Dos detalles finos:

- **`type(x) is Clase`** es la verificación *estricta* (capítulo 12 sección 8): solo acepta exactamente `SuperficieCuadrada`, no subclases aún (la herencia llega en el capítulo 15).
- El último `raise NotImplementedError` es bueno por partida doble: demuestra que el `raise` manual del capítulo 6 se usa adentro de los objetos, y deja explícito el límite de la operación.

Los operadores binarios forman una familia completa con nombre propio. Fijate un detalle fácil de confundir: el `>>` es `__rshift__` (la "r" es de *right*), el `<<` es `__lshift__` (la "l" es de *left*).

<center>

| Operador    | Método                      |
| ----------- | --------------------------- |
| `+`         | `__add__`                   |
| `-`         | `__sub__`                   |
| `*`         | `__mul__`                   |
| `/`         | `__truediv__`               |
| `//`        | `__floordiv__`              |
| `%`         | `__mod__`                   |
| `**`        | `__pow__`                   |
| `<<` / `>>` | `__lshift__` / `__rshift__` |
| `&`         | `__and__`                   |
| `\|`        | `__or__`                    |
| `^`         | `__xor__`                   |

</center>
---

## 2. Los recíprocos, la tienda y las comparaciones

Hay un problema elegante escondido en la suma: **el orden importa**. `cuadros[0] + 12` funciona porque `__add__` (el método de la izquierda) sabe sumar enteros. Pero `12 + cuadros[0]` llama primero a `int.__add__(12, cuadros[0])` — y la clase `int` no tiene idea de qué es un `SuperficieCuadrada`. Veámoslo:

```python
try:
    print(12 + cuadros[0])
except TypeError as e:
    print("TypeError:", e)   # unsupported operand type(s) for +: 'int' and 'SuperficieCuadrada'
```

Python, el rebelde, no se rinde ahí: cuando el método de la izquierda no lo logra, **le da otra chance al objeto de la derecha** con un método *recíproco* — el `__radd__` (la "r" de *reverse*):*

```python
class SuperficieCuadrada:
    """Suma superficies, ahora con método recíproco."""

    lado = 1

    def superficie(self):
        return self.lado ** 2

    def __add__(self, elemento):
        if type(elemento) is SuperficieCuadrada:
            return self.superficie() + elemento.superficie()
        elif type(elemento) in (int, float):
            return self.superficie() + elemento
        raise NotImplementedError("no sé sumar eso a una superficie")

    def __radd__(self, elemento):
        """Cuando el otro objeto no supo sumar: yo hago la suma."""
        return self.__add__(elemento)

cuadrados = [SuperficieCuadrada(), SuperficieCuadrada(), SuperficieCuadrada()]
cuadrados[0].lado = 10        # superficie 100
cuadrados[1].lado = 20        # superficie 400
cuadrados[2].lado = 3         # superficie 9

print(12.5 + cuadrados[1])    # 412.5  → recíproco: 400 + 12.5
print(cuadrados[2] + 15.4)    # 24.4   → el lado "normal"
print(sum(cuadrados))         # 509    → 0 + 100 + 400 + 9, todo por el recíproco
```

El `sum()` es la joya del recíproco: `sum` empeza en el `0` (un `int`), y ante cada `0 + cuadrado` el `int` se raja y el cuadrado toma la posta con `__radd__`. El resto de la familia recíproca: `__rsub__`, `__rmul__`, `__rmod__`, `__rdiv__`.

### La tienda: sobrecarga con sentido

La clase `Producto` usa los operadores como lenguaje de negocio: `pro1 + pro2` es el costo total, `pro1 % 15` es un descuento del 15%, `+=` agrega gastos. Mirala completa:

```python
class Producto:
    """Un producto de la tienda con su costo en pesos."""

    def __init__(self, nombre: str, costo: float):
        self.nombre = nombre
        self.costo = costo

    def __str__(self):
        return f"Producto {self.nombre} cuesta ${self.costo}"

    def __add__(self, otro):
        """Suma de costos: da el total en un mensaje."""
        return f"La suma de {self.nombre} más {otro.nombre} es de ${self.costo + otro.costo}"

    def __mod__(self, porcentaje):
        """Descuento: vuelve a calcular el precio con el % indicado."""
        self.costo = round(self.costo + self.costo * porcentaje / 100, 2)
        return f"nuevo precio : {self.costo}"

    def __iadd__(self, valor):
        """+= : agrega un gasto al costo del producto."""
        self.costo += float(valor)
        return self

    def __gt__(self, otro):
        """> : compara costos."""
        return self.costo > otro.costo

    def __lt__(self, otro):
        """< : compara costos."""
        return self.costo < otro.costo

    def __float__(self):
        return self.costo

    def __int__(self):
        return int(round(self.costo))

pro1 = Producto("ecotermo", 13000)
pro2 = Producto("calefon", 15000)

print(pro1)                # Producto ecotermo cuesta $13000
print(pro1 + pro2)         # La suma de ecotermo más calefon es de $28000
print(pro1 % 15)           # nuevo precio : 14950.0
print(pro1)                # Producto ecotermo cuesta $14950.0
pro1 += 100                # __iadd__ modifica al objeto y lo devuelve
print(pro1)                # Producto ecotermo cuesta $15050.0
print(pro2 > pro1, pro1 < pro2)   # False False
print(float(pro1), int(pro1))     # 15050.0 15050
```

Tres lecturas nuevas:

- **`__iadd__`** es la **asignación extendida** (`+=`): debe modificar el objeto y **devolverlo** (`return self`) para que la asignación se complete. La familia: `__iadd__`, `__isub__`, `__imul__`, ...
- **Las comparaciones** viven en `__gt__`/`__lt__` (y sus hermanas `__le__`, `__eq__`, `__ne__`, `__ge__`), y `int()`/`float()` en `__int__`/`__float__` — de la sección 3.
- Nota el arriesgado pero didáctico `%` como "porcentaje de descuento": el operador módulo que en los números es el resto, acá significa "aplicá este porcentaje". Funciona porque sos vos quien define el lenguaje.

> **Buenas prácticas — el lado oscuro de la sobrecarga:** un `+` o un `%` con significado *inusual* son el superpoder y el kriptonita del rebelde. Si `producto % 15` deja dudas, el capítulo 14 te va a enseñar el movimiento correcto: un método con nombre (`descuento(15)`). Convención clara > capricho: que el símbolo diga exactamente lo que dicen las matemáticas o las colecciones (`+` suma, `in` contiene, `==` iguala).

Tabla descriptiva de operadores de comparacion

<center>

| Operador | Método dunder | Se lee como       |
| -------- | ------------- | ----------------- |
| `<`      | `__lt__`      | menor que         |
| `<=`     | `__le__`      | menor o igual que |
| `>`      | `__gt__`      | mayor que         |
| `>=`     | `__ge__`      | mayor o igual que |
| `==`     | `__eq__`      | igual que         |
| `!=`     | `__ne__`      | distinto de       |

</center>

---

## 3. El espejo del objeto: `__repr__` y `__str__`

Recordá el fin del capítulo 12: imprimías un objeto y salía el críptico `<__main__.Clase object at 0x...>`. Eso es el `__repr__` heredado de `object`, la representación *técnica*. Cuando **vos** definís los métodos de espejo, decidís cómo se presenta tu objeto en dos audiencias: los programadores (`__repr__`) y los humanos (`__str__`). Un ejemplo: la clase `Ente`, que se presenta:

```python
class Ente:
    """Un ente que sabe mostrarse y decidir su verdad."""

    def __init__(self, nombre: str):
        self.nombre = nombre

    def __repr__(self):
        """Representación para el intérprete (técnica)."""
        return f"Ente(id={id(self)})"

    def __str__(self):
        """La versión humana, la que usa print y str."""
        return self.nombre.upper()

    def __bool__(self):
        """¿Es verdadero? Depende de cómo empieza cada palabra del nombre."""
        return self.nombre.istitle()

ente = Ente("Luis Ramón")
print(repr(ente))      # Ente(id=...)     → __repr__
print(str(ente))       # LUIS RAMÓN       → __str__
print(ente)            # LUIS RAMÓN       → print usa __str__

print(bool(ente))      # True   → "Luis Ramón".istitle() es True
emi = Ente("emi")
print(bool(emi))       # False  → "emi".istitle() es False
```

Bloque de reglas que vas a agradecer en serio:

| Función/contexto                        | Qué usa                              | Notas                                      |
| --------------------------------------- | ------------------------------------ | ------------------------------------------ |
| `print(obj)`                            | `__str__`                            | si no existe, Python cae en `__repr__`     |
| `str(obj)`                              | `__str__`                            | lo mismo                                   |
| `repr(obj)` / el intérprete             | `__repr__`                           | la "receta" para reconstruir               |
| f-strings                               | `__format__`                         | por defecto delega en `__str__`            |
| `int(obj)` / `float(obj)` / `bool(obj)` | `__int__` / `__float__` / `__bool__` | conversiones hechas a medida               |
| `if obj:`                               | `__bool__`                           | si no existe, cae en `__len__` (0 = falso) |

El `__bool__` esconde una regla que ya manejás desde el capítulo 3: cuando Python evalúa "verdad de un objeto", pregunta `bool(obj)`, y vos podés decidir el criterio — acá, "¿el nombre tiene mayúscula inicial?".

> **Buenas prácticas:** definí siempre `__repr__`. Es lo primero que ve quien depura y quien abre un REPL. Si querés una presentación de humanos aparte, sumá `__str__`; si no, dejá que `print` use la técnica. El capítulo 17 y los `dataclasses` (sección 11) te van a dar reglas y atajos para no escribirlos a mano.

---

## 4. Quién crea al objeto: `__new__`

El espejo del objeto supone que ya hay un objeto que mirar. Pero, ¿dónde nace? En el capítulo 12 conociste a `__init__` como "el constructor"; un espejo honesto te obliga a completar la frase: `__init__` *inicializa*, pero quien *crea* al objeto es otro dunder, `__new__`. Cuando escribís `Clase(...)`, Python hace dos pasadas: primero llama a `__new__` para fabricar el objeto en crudo y recién después corre `__init__` para llenarlo. `__new__` es el único método especial que recibe la **clase** como primer parámetro en vez de una instancia: el parámetro se llama `cls` — el `self` del capítulo 12 es un objeto concreto, `cls` es el molde. Recién en la segunda pasada, con `__init__`, aparece el `self`; cuando corre `__new__` la instancia todavía no existe, y su trabajo es traerla al mundo y devolverla. Y ese `cls` es la puerta a la clase misma: `cls.nacidos += 1` suma en el atributo compartido de `Fantasma`, no en un objeto suelto. El mismo `cls` vuelve en el capítulo 14 como primer parámetro de los **métodos de clase** (`@classmethod`) — ahí lo vas a ver en serio, cuando el molde sea el protagonista.

Verlo es más fácil que contarlo. Un fantasma que se censa solo:

```python
class Fantasma:
    """Un espectro que se registra apenas nace."""

    nacidos = 0

    def __new__(cls, *args, **kwargs):
        cls.nacidos += 1
        return super().__new__(cls)

    def __init__(self, nombre):
        self.nombre = nombre

casilda = Fantasma("Casilda")
melchor = Fantasma("Melchor")
print(Fantasma.nacidos)    # 2
print(casilda.nombre)      # Casilda
```

La regla de oro: `__new__` tiene que **devolver** una instancia — acá, la que fabrica `super().__new__(cls)`, la herencia del `object` de siempre. Si te olvidás de devolverla, `__init__` jamás se ejecuta. Y fijate un detalle: `__new__(cls, *args, **kwargs)` acepta los argumentos de `Fantasma("Casilda")` aunque no los use — `Clase(...)` le pasa a `__new__` los mismos argumentos que le pasa a `__init__`. Y como `__new__` corre antes que todo lo demás, podés decidir *qué* objeto devolver — incluso uno que ya nació antes:

```python
class Unica:
    """Una sola instancia por clase: la misma entrada, siempre."""

    _unica = None

    def __new__(cls):
        if cls._unica is None:
            cls._unica = super().__new__(cls)
        return cls._unica

a = Unica()
b = Unica()
print(a is b)   # True
```

Ese es el patrón *singleton* (instancia única) en cinco líneas: el truco de las fábricas y de las conexiones de base de datos para no duplicar recursos. Guardátelo, porque vas a retomarlo en serio más adelante, en el capítulo de **Patrones de Diseño**. Y en el capítulo 17 vas a ver el mismo `__new__` en otro escenario: ahí no fabrica una instancia, fabrica **la clase entera** — son las metaclases.

> **Dato clave:** `Clase(...)` son dos pasadas: `__new__` crea el objeto (recibe `cls`, la clase — no un objeto —, y devolver la instancia es obligatorio) y `__init__` lo inicializa. Ese gancho, que corre antes que todo, te permite censar nacimientos o devolver siempre el mismo objeto.

---

## 5. Medir y cortar: `__len__` y `__getitem__`

Hay dos dunders que vuelven a tus objetos *colecciones de verdad*: `__len__` (para `len()`) y `__getitem__` (para `obj[i]`, el indexado). Un ejemplo propio, una bolsita de palabras:

```python
class Palabras:
    """Una bolsita de palabras indexable y medible."""

    def __init__(self, *palabras: str):
        self.items = list(palabras)

    def __len__(self):
        return len(self.items)

    def __getitem__(self, i):
        return self.items[i]

caja = Palabras("hola", "mundo", "python")

print(len(caja))      # 3           → __len__
print(caja[0])        # hola        → __getitem__ con índice
print(caja[-1])       # python      → índices negativos, gratis
print(caja[1:])       # ['mundo', 'python']  → slices, gratis
```

Al delegar en la `list` interna, te regalan los casos difíciles (índices negativos, rebanadas) sin escribir una línea. Y hay una sorpresa de escalera: **si una clase define `__getitem__`, el `for` la puede recorrer aunque no exista `__iter__`**. Python prueba `caja[0]`, `caja[1]`, `caja[2]`… hasta que el indexado lanza `IndexError`, y entonces frena:

```python
for palabra in caja:
    print(palabra)   # hola / mundo / python
```

Esa es la iteración "heredada". Está buenísima para colecciones — pero para *generar* valores a medida (o infinitos, como el `itertools.count` del capítulo 5) se queda corta. Ahí entran los dunders de la sección que viene.

---

## 6. La pieza central: tu `rango`

En el capítulo 5, sección 9, hay una promesa en letra azul: cuando llegues a POO vas a poder fabricar tus propios objetos iterables definiendo `__iter__` y `__next__`. Escribiéndolo, además, dirías la versión *verdadera* de "atrás de todo `for` hay un iterador" (capítulo 5) y de la cinta perezosa de los generadores (capítulo 10).

Y la materialización: un `rango` con **paso decimal** y **precisión configurable** — algo que el `range` built-in ni sueña (el `range` de verdad no acepta `0.2` de paso; el nuestro sí). Este es el código canónico, para leerlo con lupa:

```python
def rango(*args, pres=1):
    """Rango personalizado con cierre de precisión (paso 0.2 detectado)."""

    class Rango:
        def __init__(self, nicio, final, paso):
            self.__inicio = nicio
            self.__final = final
            self.__paso = paso

        def __repr__(self):
            return f"rango de {inicio} a {final} con paso {paso}"

        def __iter__(self):
            return self

        def __next__(self):
            if self.__inicio < self.__final:
                self.__inicio = round(self.__inicio + self.__paso, pres)
                return self.__inicio
            else:
                raise StopIteration

    if len(args) == 1:
        inicio, final, paso = 0, args[0], 1
    elif len(args) == 2:
        inicio, final, paso = args[0], args[1], 1
    elif len(args) == 3 and args[2] < abs(args[1] - args[0]):
        inicio, final, paso = args
    elif len(args) == 3 and args[2] > abs(args[1] - args[0]):
        inicio, final, paso = args
    else:
        raise AttributeError("rango espera start end and step")

    return Rango(inicio, final, paso)
```

Antes de probarlo, la lectura en tres capas, porque esto es mucho código bueno:

- **El dispatch por `len(args)`** es el ya conocido `*args` del capítulo 7: con 1 argumento es `rango(stop)` con inicio 0 y paso 1; con 2, `rango(start, stop)`; con 3, `rango(start, stop, paso)`. Las dos ramas `elif` de tres argumentos filtran pasos demasiado grandes en una u otra dirección; si nada encaja, un `AttributeError` con el mensaje claro de lo que el `rango` esperaba.
- **La clase `Rango` vive adentro de la función.** El capítulo 12 te mostró clases internas; acá además van a *cerrar* sobre los locales de la función: `pres`, `inicio`, `final` y `paso` se leen dentro de los métodos sin ser parámetros (a diferencia de la clase anidada del capítulo 12, que no veía a su instancia externa, una clase definida dentro de una función sí captura los locales de esa función — son las closures del capítulo 8, aplicadas a clases enteras).
- **`__iter__` devuelve `self`** (el objeto ES su propio iterador, un boleto descartable, como la cinta del capítulo 5) y **`__next__`** avanza el contador interno y **lanza `StopIteration`** cuando se termina. Esa excepción es el fin de cinta del capítulo 5: el `for` la ve y corta. Los atributos `__inicio`, `__final`, `__paso` llevan doble guion: es el *name mangling* del capítulo 14, un maquillaje "privado" que después vas a entender completo.

Corré los seis demos:

```python
for i in rango(5, 15, 2):
    print(i)

print(list(rango(0, 10, 2)))                  # [2, 4, 6, 8, 10]
print(list(rango(20, 10, -0.5)))              # []   ← inesperado
print(list(rango(2, 1, -0.2)))                # []   ← inesperado
print(list(rango(12, 12.1, 0.01, pres=2)))    # [12.01, ..., 12.1]
print(list(rango(1, 10, 11)))                 # [12]  ← inesperado
```

| demo (5,15,2): 7 9 11 13 15 — funciona, pero **saltea el 5** inicial.

Fijate que la máquina responde: el `rango` funciona para pasos positivos, el redondeo con `pres` es un lujo (**[12.01…12.1]** con 0.01 de paso), y suelta **StopIteration** limpiamente. Pero en los tres casos marcados como `inesperado` hay *bugs lindos* — errores que enseñan. Vamos a la cirugía:

1. **El `nicio` del constructor** es un typo de tipeo: el parámetro se llama `nicio` y se guarda igual en `self.__inicio`. Como el typo es consistente, todo funciona — pero lo limpiamos a `inicio`.
2. **`__next__` nunca entrega el primer valor** y **camina una sola dirección**. Mirá la mecánica: `__next__` *primero* hace `self.__inicio + paso` y *recién después* devuelve, así que el valor inicial (el `5` de `rango(5, 15, 2)`) jamás se entrega. Y el crítico viene por el costado de la dirección: `__next__` solo sabe ir *para adelante* (`inicio < final`), por eso los pasos negativos dan `[]` — ¡y hasta el demo `rango(20, 10, -0.5)` te quedó vacío! Un `range` de verdad recorre en ambas direcciones.
3. **El `__repr__` lee la closure, no el estado**: muestra `inicio/final/paso` de la función (que existen), pero no los del objeto. Coincidirá casi siempre, pero no es el objeto el que se muestra — es la receta.
4. **El filtro de pasos** (`abs(...)`) es frágil: `rango(1, 10, 11)` pasa por la segunda rama y te devuelve el disparatado `[12]`.

El `rango` pulido, con las cuatro lecciones aplicadas — pasa de "arrancar en el primer paso" a **devolver el inicio** de verdad, entiende **pasos negativos**, muestra un `repr` honesto y simplifica el dispatch:

```python
def rango(*args, pres=1):
    """Rango a medida: paso decimal, precisión y pasos negativos."""

    class Rango:
        def __init__(self, inicio, final, paso):
            self.__actual = inicio
            self.__final = final
            self.__paso = paso

        def __repr__(self):
            return f"Rango({self.__actual!r}, {self.__final!r}, paso={self.__paso!r})"

        def __iter__(self):
            return self

        def __next__(self):
            if (self.__paso > 0 and self.__actual >= self.__final) or \
               (self.__paso < 0 and self.__actual <= self.__final):
                raise StopIteration
            actual = self.__actual
            self.__actual = round(self.__actual + self.__paso, pres)
            return actual

    if len(args) == 1:
        inicio, final, paso = 0, args[0], 1
    elif len(args) == 2:
        inicio, final, paso = args[0], args[1], 1
    elif len(args) == 3:
        inicio, final, paso = args
    else:
        raise AttributeError("rango espera start end and step")

    return Rango(inicio, final, paso)

print(list(rango(0, 10, 2)))               # [0, 2, 4, 6, 8]
print(list(rango(2, 1, -0.2)))             # [2, 1.8, 1.6, 1.4, 1.2]
print(list(rango(12, 12.1, 0.01, pres=2))) # [12, 12.01, ..., 12.09]
print(list(rango(1, 10, 11)))              # [1]      → el 1 inicial; el salto (12) ya queda fuera del final
```

El `__next__` ahora pregunta por la **dirección del paso**: si es positivo, se detiene cuando pasa el final; si es negativo, cuando lo pasa para abajo. Devuelve `actual` (que empezó en `inicio`) *y después* avanza — por eso el primer valor es el de verdad. Y el guard abs del dispatch desapareció: `rango(1, 10, 11)` ahora es `[1]` — devuelve el inicio, y el salto al `12` (que ya pasó el `final`) queda afuera, como lo haría el `range` de verdad — honesto.

> **Dato clave:** los dunders de iteración son el idioma universal: `__iter__` (el boleto descartable), `__next__` (un paso adelante), `StopIteration` (el fin de cinta). Con esos tres, tu objeto participa en `for`, en `list()`, en cualquier herramienta que consuma iterables. Eso es lo que venís haciendo con los generadores desde el capítulo 10, pero ahora con una clase bajo control total de su estado.

### La versión profesional

Las dos versiones anteriores existen para enseñar: una se equivoca, la otra se corrige. Esta tercera es la que dejarías en un proyecto en serio — misma lógica que la pulida, pero con el **plano** (los type hints del capítulo 9), la **documentación** (docstring estilo Google, del mismo capítulo, con un `Example` que `doctest` puede ejecutar) y comentarios que explican el *porqué*. Mirá la firma: `-> Iterator[float]` dice "esto devuelve algo recorrible", sin importar que por dentro sea una clase fabricada al vuelo. `Iterator` sale de `collections.abc`, el hogar moderno de estos protocolos (Python 3.9+); `typing.Iterator` sigue funcionando, pero es el alias viejo.

```python
from collections.abc import Iterator   # el hogar moderno de los protocolos (Python 3.9+)

def rango(*args: float, pres: int = 1) -> Iterator[float]:
    """Rango a medida con paso decimal, precisión y sentido configurables.

    Genera un iterador numérico que recorre desde el inicio hasta el final
    avanzando según el paso; admite pasos positivos y negativos.

    Args:
        *args: fin, o inicio y fin, o inicio, fin y paso.
        pres: decimales a los que se redondea cada valor.

    Returns:
        Un iterador de los valores del rango.

    Raises:
        AttributeError: si no se pasan entre 1 y 3 argumentos.

    Example:
        >>> list(rango(3))
        [0, 1, 2]
        >>> list(rango(5, 15, 2))
        [5, 7, 9, 11, 13]
    """

    class Rango:
        """Iterador de un solo uso: es su propio boleto (__iter__ devuelve self)."""

        def __init__(self, inicio: float, final: float, paso: float) -> None:
            # El estado vive en el objeto: punto actual, límite y salto.
            self.__actual = inicio
            self.__final = final
            self.__paso = paso

        def __repr__(self) -> str:
            return f"Rango({self.__actual!r}, {self.__final!r}, paso={self.__paso!r})"

        def __iter__(self) -> Iterator[float]:
            return self

        def __next__(self) -> float:
            # Se agotó: el paso ya cruzó el final (hacia arriba o hacia abajo).
            if (self.__paso > 0 and self.__actual >= self.__final) or \
               (self.__paso < 0 and self.__actual <= self.__final):
                raise StopIteration
            actual = self.__actual                                    # devolvemos el actual...
            self.__actual = round(self.__actual + self.__paso, pres)  # ...y recién ahí avanzamos
            return actual

    # Dispatch por cantidad de argumentos (1, 2 o 3), igual que el range built-in.
    if len(args) == 1:
        inicio, final, paso = 0, args[0], 1
    elif len(args) == 2:
        inicio, final, paso = args[0], args[1], 1
    elif len(args) == 3:
        inicio, final, paso = args
    else:
        raise AttributeError("rango espera inicio, fin y paso (1, 2 o 3 argumentos)")

    return Rango(inicio, final, paso)
```

Y ya que estamos en modo ingeniero, una pregunta tramposa: **¿cuántos elementos genera este rango sin recorrerlo?** `rango(0, 2.1, 0.3)` imprime 7 valores (`0`, `0.3`, … `1.8`). La tentación "de manual" es `math.ceil((final - inicio) / paso)`, y mirá lo que hace:

```python
from math import ceil
ceil((2.1 - 0) / 0.3)        # 8   ¡miente!
```

Ocho cuando en verdad son 7: `2.1 / 0.3` da `7.000000000000001` — el fantasma de los floats — y `ceil` redondea hacia arriba esa mentira. El truco, con el mismo espíritu que `pres`: si todo vive en múltiplos de `10**-pres`, **multipliquemos por `10**pres` y pensemos en enteros**. La división de enteros no miente:

```python
k = 10                                        # 10**pres, con pres=1
round(2.1 * k), round(0.3 * k)                # (21, 3) exactos
-((-21) // 3)                                 # 7 → el ceil entre enteros
```

Poder saberlo en cerrado no es solo un juego: habilita un dunder de iterador que todavía no viste, **`__length_hint__`** — "una pista de cuánto queda", el protocolo que `list()` y `tuple()` consultan para preasignar memoria. Es el *primo honesto* de `__len__`: lo tiene `iter(range)` y no lo tiene `range`, porque un iterador cuenta cuánto *le falta*, no su tamaño. Sumémoslo a la `Rango`:

```python
def __length_hint__(self) -> int:
    """Cuántos valores deja por generar, sin consumir ninguno."""
    k = 10 ** pres
    num = round(self.__final * k) - round(self.__actual * k)
    p = round(self.__paso * k)
    if p == 0 or num * p <= 0:
        return 0
    return -((-num) // p) if p > 0 else -((num) // (-p))
```

```python
import operator
operator.length_hint(rango(0, 10, 2))        # 5 → list() ya reserva de una
len(rango(0, 10, 2))                          # TypeError: iterador, no secuencia
```

> **Dato clave:** `__length_hint__` es la señal de "cuánto me queda" que el runtime acepta como *pista*: si no existe, el default es 0; `operator.length_hint()` la lee y `list()`/`tuple()` reservan memoria con ella, pero la verdad la fija `StopIteration`. `len()` sigue siendo territorio de las secuencias — el `range` built-in lo tiene por ser secuencia; su iterador, como el nuestro, no. Y un detalle de película de terror: si el paso fuera menor que la precisión (`rango(0, 1, 0.04, pres=1)`), el paso redondearía a `0.0` y el rango **nunca se agotaría**. Ahí el hint devuelve `0`, y no queda otra: es imposible contar algo que no termina.

---

## 7. Consumidores de iterables

Y acá está la recompensa de aprender el idioma: tu `rango` ahora es **un ciudadano más**, y todas las herramientas del libro que consumen secuencias lo aceptan sin chistar. Es momento de enamorarte del coro completo — cada uno con su capítulo de origen:

```python
print(list(rango(0, 10, 2)))                       # [0, 2, 4, 6, 8]      (cap03)
print(tuple(rango(0, 10, 2)))                      # (0, 2, 4, 6, 8)     (cap03)
print(sum(rango(0, 10, 2)))                        # 20                  (cap11)
print(sorted(rango(0, 10, 2), reverse=True))       # [8, 6, 4, 2, 0]     (cap11)
print(list(map(lambda n: n * 10, rango(0, 10, 2))))   # [0, 20, 40, 60, 80] (cap11)
print(list(zip(rango(0, 10, 2), "abcde")))         # [(0,'a'), (2,'b'), ...] (cap11)
print(list(enumerate(rango(0, 10, 2))))            # [(0,0), (1,2), (2,4), ...] (cap04)
print(any(n > 6 for n in rango(0, 10, 2)))         # True   (cap11)
print(all(n < 20 for n in rango(0, 10, 2)))        # True   (cap11)
print(list(reversed(list(rango(0, 10, 2)))))       # [8, 6, 4, 2, 0]     (cap11)
```

Dos aclaraciones honestas del coro:

- `any`/`all`, `map`, `filter`, `sum` **consumen** el iterador descartable: se usan una sola vez y listo. Si necesitás el mismo `rango` dos veces, volvé a llamar a la función (cada llamada fabrica un boleto nuevo).
- `reversed` es especial: **pide una secuencia** (algo con `__len__` y `__getitem__`), y un iterador descartable no lo es. Por eso hay que pasarlo a `list()` primero: `reversed(list(rango(...)))`. Si querés que tu clase lo soporte directo, definí `__reversed__` — el manual de instrucciones de cada dunder dice exactamente qué protocolo exige.

Tabla del coro, para guardarla de memoria:

| Consumidor                     | Qué hace                       | Dónde lo viste |
| ------------------------------ | ------------------------------ | -------------- |
| `for` / `next()`               | recorre de a uno               | capítulos 4-5  |
| `list()` / `tuple()` / `set()` | vuelca el iterable completo    | capítulo 3     |
| `sum()` / `sorted()`           | reduce y ordena                | capítulo 11    |
| `map()` / `filter()` / `zip()` | transforma, selecciona, une    | capítulo 11    |
| `enumerate()`                  | índice + elemento              | capítulo 4     |
| `any()` / `all()`              | el veredicto booleano          | capítulo 11    |
| `reversed()`                   | la vuelta, pero pide secuencia | capítulo 11    |

---

## 8. `__slots__`: adelgazar al objeto

Volvemos al espejo por el lado de la patria de la memoria. Viste en el capítulo 12 que los atributos de instancia viven en el `__dict__` del objeto — **un diccionario entero por objeto**. Además de la libertad de agregar atributos en caliente, ese dict tiene un costo. La rebeldía de los `__slots__` es decirle a la clase: *"los atributos van a ser exactamente estos; no hace falta dict por objeto"*:

```python
class Punto:
    """Un punto con huecos fijos: sin __dict__ por objeto."""

    __slots__ = ("x", "y")

    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

p = Punto(1, 2)
print(p.x, p.y)        # 1 2
print(p.__dict__)      # AttributeError: 'Punto' object has no attribute '__dict__'
p.z = 3                # AttributeError: no hay slot 'z'
```

Punto: **los slots fijan la lista de atributos**. Se acabó agregar atributos en caliente (el superpoder del capítulo 12), y a cambio ganás el `__dict__` entero. ¿Cuánto vale? Medirlo en el mundo real:

```python
import sys

class PuntoComun:
    """Un punto normal: cada objeto carga su diccionario."""

    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

lista_comun = [PuntoComun(i, i) for i in range(10_000)]
lista_slots = [Punto(i, i) for i in range(10_000)]

def huella(objetos):
    """Suma del objeto más su __dict__, si tiene."""
    total = sum(sys.getsizeof(o) for o in objetos)
    for o in objetos:
        if hasattr(o, "__dict__"):
            total += sys.getsizeof(o.__dict__)
    return total

print(huella(lista_comun))    # 1360000  → ~1,36 MB
print(huella(lista_slots))    # 480000   → ~0,48 MB
```

Con 10.000 puntos, los comunes ocupan casi **tres veces** más (los números exactos dependen de la versión de Python y del sistema; lo que no depende es el orden de magnitud). Por eso las clases que fabrican cientos de miles de objetos en ciencia de datos, videojuegos o procesamiento de texto declaran `__slots__` y se llevan a enteros la memoria.

> **Buenas prácticas:** usá `__slots__` cuando tenés **muchísimas instancias** y sabés de antemano todos sus atributos, o cuando querés *congelar* el esquema. Para el 99% de las clases, dejá el `__dict__` — la libertad de crecer vale más que los bytes. Y ojo: con `__slots__` perdés `__dict__`, así que decenas de funciones (de `json` a `copy`) que dependen de él exigen adaptarse.

> **Dato clave:** `__slots__` **no es encapsulación — es optimización**. Que no se puedan *crear* atributos nuevos es un efecto lateral, no un candado: los atributos listados siguen siendo públicos y cambiables (`p.x = 99` sigue funcionando). Si buscás **proteger valores**, eso es trabajo del capítulo 14 (`_x` + `@property`); `__slots__` solo adelgaza el objeto.

---

## 9. Context managers: `__enter__` y `__exit__`

El capítulo 6 te mostró el `with` de pasada: *abre y libera solo*. Ahora la deuda se paga: detrás de `with` hay dos dunders, `__enter__` (qué pasa al entrar) y `__exit__` (qué pasa al salir, pase lo que pase). La joya del ejemplo, una tostadora que se maneja sola:

```python
class Tostadora:
    """Una tostadora que se enciende al entrar al contexto y se apaga al salir."""

    def __init__(self, potencia: int = 3):
        self.potencia = potencia
        self.ocupada = False

    def __enter__(self):
        self.ocupada = True
        print(f"Tostadora encendida a potencia {self.potencia}.")
        return self           # lo que recibe la cláusula 'as'

    def tostar(self, pan: str):
        if not self.ocupada:
            raise RuntimeError("la tostadora está apagada")
        print(f"Tostando {pan}...")

    def __exit__(self, tipo, valor, traceback):
        self.ocupada = False
        print("Tostadora apagada.")
        return False          # False → no tragamos excepciones: siguen su viaje

with Tostadora(5) as tostadora:
    tostadora.tostar("pan de campo")

print(tostadora.ocupada)      # False → al salir del with, la tostadora se apagó
```

El contrato completo:

- `__enter__` corre al llegar al bloque y **lo que devuelve** se asigna a la variable del `as`.
- `__exit__(tipo, valor, traceback)` corre **siempre** al salir: normal o con excepción (los tres parámetros describen la excepción, o son `None` si no hubo). Si devolvés `True`, la excepción se *traga*; si devolvés `False` (o `None`), sigue propagándose. Es el `finally` de los capítulos 6, con clase.

Y si la excepción ocurre en el medio, el protocolo se cumple igual:

```python
with Tostadora() as t:
    t.tostar("pan")
    raise ValueError("se cortó la luz")
# Tostadora encendida a potencia 3.
# Tostando pan...
# Tostadora apagada.
# Traceback: ValueError: se cortó la luz   ← la excepción siguió su camino
```

El mundo real lo usa por todos lados: abrir un archivo y cerrarlo (`with open(...)`, capítulo 6), una transacción SQL, un bloqueo de red. Hasta lo de arriba de todo: los **test frameworks** registran y desregistran fixtures así.

### El atajo: `@contextmanager`

Definir una clase entera para cerrar un recurso puede ser mucho ceremonia. `contextlib` te regala el atajo: decorá una **función generadora** (capítulo 10) y transformala en context manager. Todo lo que va antes del `yield` es la entrada; todo lo que va después, la salida; y el `yield` entrega lo del `as`:

```python
from contextlib import contextmanager

@contextmanager
def apuntando(nombre):
    print(f"-> entrando a {nombre}")
    yield nombre
    print(f"<- saliendo de {nombre}")

with apuntando("la cueva") as cueva:
    print("dentro de", cueva)
```

Salida:

```
-> entrando a la cueva
dentro de la cueva
<- saliendo de la cueva
```

Generadores otra vez (sanción del capítulo 10): el cuerpo se *pausa* en el `yield` mientras el bloque del `with` corre, y *se retoma* al salir. El `@contextmanager` es literalmente "ponele la cara de `with` a un generador".

> **Dato clave:** `with` no captura errores: **compone** con el `try`/`except` del capítulo 6. El context manager garantiza el después (liberar, apagar, cerrar); el `try/except` decide qué hacer si algo falla. Dos protocolos, dos responsabilidades.

---

## 10. `__call__`: objetos que se llaman

Último dunder estrella de este capítulo: **`__call__`** hace que un objeto sea *invocable* — que puedas ponerle paréntesis y llamarlo como a una función. Recordá el capítulo 8: los decoradores son funciones que reciben y devuelven funciones. Y las **clases también son decoradores**: el objeto decora con su propio estado. Un aplausómetro de funciones:

```python
class Aplausometro:
    """Un decorador que cuenta cuántas veces se llama a una función."""

    def __init__(self, funcion):
        self.funcion = funcion
        self.aplausos = 0

    def __call__(self, *args, **kwargs):
        self.aplausos += 1
        print(f"aplauso {self.aplausos} para {self.funcion.__name__}")
        return self.funcion(*args, **kwargs)

@Aplausometro
def chiste(palabra):
    return f"por que {palabra} y no {palabra[::-1]}?"

print(chiste("programar"))
print(chiste("programar"))
print(chiste.aplausos)   # 2 → el contador vive en el objeto
```

Salida:

```
aplauso 1 para chiste
por que programar y no ramargorp?
aplauso 2 para chiste
por que programar y no ramargorp?
2
```

Desglose: `@Aplausometro` es `chiste = Aplausometro(chiste)` (capítulo 8): el nombre `chiste` ahora es **un objeto** de la clase. Cuando le ponés paréntesis, corre `__call__`, que cuenta, grita y llama a la función original. La ventaja frente al decorador-función: el contador es **estado del objeto** — se sostiene en `self.aplausos` sin closures ni variables globales. Esas son las clases como decoradores; vas a ver más decoradores con `@` en los capítulos 14 y 17.

> **Para curiosear:** si una clase define `__call__`, también es un *callable type* en el sentido del capítulo 9 (`Callable[..., int]`). Por eso `chiste` se puede anotar como `Callable[[str], str]` aunque sea un objeto — la "llamabilidad" es del comportamiento, no del nombre.

---

## 11. dataclasses: los dunders por mí

Cerramos el espejo con la mayor rebeldía de todas: **que Python te escriba los dunders**. Una `dataclass` (módulo estándar `dataclasses`, Python 3.7+, y ya es doctrina en la industria) mira tus *anotaciones de tipo* — los hints del capítulo 9 — y te genera el `__init__`, el `__repr__` y el `__eq__` automáticamente. Compará el `Persona` a mano del capítulo 12 con esta:

```python
from dataclasses import dataclass

@dataclass
class Jugador:
    """El molde de un jugador de la liga: los dunders por mí."""

    nombre: str
    posicion: str
    goles: int = 0

luca = Jugador("Luca", "delantero")
marco = Jugador("Luca", "delantero")

print(luca)                # Jugador(nombre='Luca', posicion='delantero', goles=0)
print(luca == marco)       # True  → __eq__ generado: comparan TODOS los campos
print(luca == Jugador("Luca", "delantero", 3))   # False → cambió un campo
```

Lo que te regala el decorador `@dataclass`, sin escribir una línea:

- **`__init__`** con los parámetros en orden (hints = tipo; los que tienen default van al final, como los defaults del capítulo 7).
- **`__repr__`** que es una joya de depuración: te muestra todos los campos y sus valores.
- **`__eq__`** por valor, no por identidad: dos jugadores con los mismos campos son iguales.

Y si querés más, vienen con llave inglesa incluida:

```python
@dataclass(frozen=True)
class Coordenada:
    """Una coordenada inmodificable."""
    x: float
    y: float

c = Coordenada(3, 4)
print(c.x, c.y)                  # 3.0 4.0
# c.x = 99                       → FrozenInstanceError: la dataclass congelada es de solo lectura
```

`frozen=True` te da un objeto inmutable (los `__setattr__` se prohíben), ideal para configuraciones y claves de diccionarios. Es la respuesta de la POO "por convención" al `final` de Java.

> **Dato clave — la promesa a futuro:** las dataclasses son el puente hacia **pydantic**, el validador que va a cerrar el temita de "los hints no validan" que quedó pendiente en el capítulo 9: pydantic toma el mismo modelo de clase con anotaciones y, cuando le pasás datos, **los valida y los convierte** en el acto. Será uno de los primeros invitados después de la Parte VI.

---

## 12. Resumen y conceptos clave

Este capítulo fue el mayor del espejo: los **métodos especiales** (dunders) que convierten a tus objetos en ciudadanos de primera. Viste que los **operadores** son azúcar de métodos — el `+` llama a `__add__` de la *izquierda*, y cuando no puede, el de la *derecha* con los **recíprocos** (`__radd__`), lo que le permitió a `sum()` sumar tus cuadrados —, y que la **sobrecarga de operadores** (una de las rebeldías del capítulo 12) te deja darle significado propio a `+`, `%`, `+=`, `<` y compañía (con la responsabilidad de que el significado se entienda solo, método con nombre mediante). Distinguiste **`__repr__`** (técnica, para depurar) de **`__str__`** (humana, para `print`), y dudaste de `__bool__`, `__int__`, `__float__` como conversiones a medida. Mediste con **`__len__`** e indexaste con **`__getitem__`**, y descubriste que el `for` puede recorrer tu clase sin `__iter__`. La pieza central fue **tu `rango`**: lo leímos con lupa y le hicimos la cirugía de los bugs lindos — el `nicio`, el `__next__` que salteaba el inicio y no entendía pasos negativos, el `__repr__` que leía la closure y el dispatch frágil — hasta convertirlo en un iterador de primera con `__iter__`/`__next__`/`StopIteration`, que abrió el **coro de consumidores** (`for`, `list`, `sum`, `sorted`, `map`, `filter`, `zip`, `enumerate`, `any`, `all`, `reversed` — todos los del libro, desde el capítulo 3 al 11, aceptándolo sin chistar). Adelgazaste objetos con **`__slots__`** (un tercio de la memoria con miles de instancias), aprendiste el protocolo **`with`** (deuda del capítulo 6): `__enter__`/`__exit__` para encender y apagar recursos siempre, más el atajo `@contextmanager` que convierte generadores en context managers. Hiciste objetos invocables con **`__call__`** y descubriste las **clases como decoradores** con estado propio. Miraste además el nacimiento del objeto: **`__new__`** lo *crea* y `__init__` lo *inicializa*, con el censo de nacimientos y el *singleton* (instancia única) como platos del día. Y cerraste con las **dataclasses**, que generan por vos `__init__`, `__repr__` y `__eq__` a partir de los hints (y `frozen=True` para inmutabilidad).

Repasá el checklist antes de seguir:

- [ ] Los **operadores** son azúcar de métodos: `a + b` es `type(a).__add__(a, b)`; en cada clase, el significado lo define su `__add__`.
- [ ] **Sobrecarga**: definís qué significa `+`, `-`, `*`, `/`, `//`, `%`, `**`, `<<`, `&`, `|`, `^` para tus objetos. La izquierda manda: si no sabe, prueba el **recíproco** de la derecha (`__radd__`, `__rsub__`, ...).
- [ ] **Asignaciones extendidas** (`+=`, `*=`, ...): `__iadd__` y su familia **modifican y devuelven el objeto** (`return self`).
- [ ] **Comparaciones**: `__lt__`, `__le__`, `__eq__`, `__ne__`, `__gt__`, `__ge__`.
- [ ] **`__repr__`** (técnica, para `repr()` y el intérprete) vs **`__str__`** (humana, para `print`/`str`); sin `__str__`, Python cae en `__repr__`.
- [ ] **Conversiones a medida**: `__bool__`, `__int__`, `__float__`, `__format__` (default → `__str__`).
- [ ] **`__len__`** y **`__getitem__`** hacen a tu clase una colección (`len()`, `obj[i]`, rebanadas, índices negativos), y `__getitem__` solo habilita la iteración por índice.
- [ ] **Iteración completa** = `__iter__` (devuelve el boleto descartable, a menudo `self`) + `__next__` (avanza y **`raise StopIteration`** al final). *Tu `rango` es el modelo.*
- [ ] **Consumidores** del mismo idioma: `for`, `next`, `list`, `tuple`, `set`, `sum`, `sorted`, `map`, `filter`, `zip`, `enumerate`, `any`, `all`; `reversed` pide secuencia (envolvelo en `list`).
- [ ] **`__slots__`** fija los atributos y elimina el `__dict__` por objeto: ~3× menos memoria en masa, adiós atributos en caliente, y nadie puede jugar con `__dict__`.
- [ ] **Context managers**: `__enter__` (devuelve lo del `as`) y `__exit__(tipo, valor, traceback)` (corre siempre; `True` traga la excepción, `False`/`None` la deja pasar). El `@contextmanager` lo arma a partir de un generador.
- [ ] **`__call__`** hace invocables a los objetos: clases como decoradores con estado propio (el contador vive en `self`).
- [ ] **`__new__`** *crea* al objeto y `__init__` lo *inicializa*: `Clase(...)` son dos pasadas; `__new__` recibe `cls` y debe devolver sí o sí la instancia (censo de nacimientos, *singleton*). Es el mismo gancho que el capítulo 17 usa para las metaclases.
- [ ] **dataclasses**: @dataclass genera `__init__`/`__repr__`/`__eq__` desde los hints; `frozen=True` para inmutables. Puente a pydantic tras la Parte VI.

---

## 13. Ejercicios

1. **Dinero con sentido**: clase `Dinero` con `monto: float`; definí `__str__` ("$100.0"), `__add__` (suma montos y devuelve otro `Dinero`), `__gt__` (compara montos) y `__eq__`. Probá `dinero1 + dinero2`, `dinero1 > dinero2` y `dinero1 == dinero2`.
2. **El recíproco del dinero**: agregale a `Dinero` el `__radd__` para que `100 + dinero` funcione y que `sum([dinero1, dinero2])` dé la suma total. Explicá por qué sin `__radd__` esas dos fallarían.
3. **El espejo de una canción**: clase `Cancion` con `titulo` y `artista`; `__repr__` técnica (`Cancion('...', '...')`), `__str__` humana (`"ARTISTA — Titulo"` con artista en mayúsculas), y `__bool__` que sea falsa si el `titulo` está vacío (`""`).
4. **Caja medible**: clase `Caja` con `__len__` y `__getitem__` que deleguen en una lista interna; probá `len()`, `caja[0]`, `caja[-1]`, `caja[1:]` y un `for`. (Bonus: sin `__iter__`, ¿por qué el `for` anda igual?)
5. **Tu `rango`, versión cirugía**: copiá la versión pulida del capítulo y probá que `rango(0, 10, 2)` incluye el `0`, que `rango(2, 1, -0.2)` anda, y que un `repr()` honesto se ve así con un `rango` cualquiera.
6. **Slot de memoria**: agregale `__slots__ = ("x", "y")` a una clase `Punto`, demostrá que no tiene `__dict__` y que no se pueden agregar atributos nuevos; después, con 10.000 puntos, compará la huella con una clase sin slots.
7. **Context manager de cafetera**: clase `Cafetera` con `__enter__` (llena tazas) y `__exit__` (apaga y devuelve `False`); mostrá que una excepción dentro del `with` apaga a la cafetera igual. Después una versión `@contextmanager` `prendiendo()`.
8. **Conteo con `__call__`**: un decorador-clase `Contador` que cuente cuántas veces se invoca a `saludar()`, con el contador accesible como atributo del objeto. Decorá una función y contá tres llamadas.
9. **El censo de nacimientos**: clase `Alma` con atributo de clase `nacidas = 0` y un `__new__` que lo incremente y devuelva `super().__new__(cls)`. Creá tres almas y mostrá `Alma.nacidas`. Bonus: convertila en *singleton* para que las tres sean la misma (`a is b` da `True`).

```python
# Soluciones (no las mires antes de intentarlo)

# 1 y 2. Dinero con sentido (+ su recíproco)
class Dinero:
    """Dinero con suma, comparación e igualdad a medida."""

    def __init__(self, monto: float):
        self.monto = monto

    def __str__(self):
        return f"${self.monto}"

    def __add__(self, otro):
        return Dinero(self.monto + otro.monto)

    def __radd__(self, otro):
        return self.__add__(Dinero(otro))

    def __gt__(self, otro):
        return self.monto > otro.monto

    def __eq__(self, otro):
        return self.monto == otro.monto

d1 = Dinero(100.5)
d2 = Dinero(50)
print(d1 + d2)              # $150.5
print(100 + d1)             # $200.5   → sin __radd__: TypeError
print(sum([d1, d2]))        # $150.5   → sin __radd__: TypeError (arranca en 0 int)
print(d1 > d2, d1 == d2)    # True False

# 3. El espejo de una canción
class Cancion:
    """Una canción con su espejo técnico y humano."""

    def __init__(self, titulo: str, artista: str):
        self.titulo = titulo
        self.artista = artista

    def __repr__(self):
        return f"Cancion({self.titulo!r}, {self.artista!r})"

    def __str__(self):
        return f"{self.artista.upper()} — {self.titulo}"

    def __bool__(self):
        return bool(self.titulo)

cancion = Cancion("Bohemian Rhapsody", "Queen")
print(repr(cancion))        # Cancion('Bohemian Rhapsody', 'Queen')
print(str(cancion))         # QUEEN — Bohemian Rhapsody
print(bool(cancion))        # True
print(bool(Cancion("", "Queen")))   # False

# 4. Caja medible
class Caja:
    """Caja medible e indexable a partir de una lista."""

    def __init__(self, *items):
        self.items = list(items)

    def __len__(self):
        return len(self.items)

    def __getitem__(self, i):
        return self.items[i]

caja = Caja("uva", "pera", "kiwi")
print(len(caja))            # 3
print(caja[0], caja[-1])    # uva kiwi
print(caja[1:])             # ['pera', 'kiwi']
for fruta in caja:
    print(fruta)            # uva pera kiwi (el for usa __getitem__ hasta IndexError)

# 5. Tu rango, versión cirugía
def rango(*args, pres=1):
    """Rango a medida: paso decimal, precisión y pasos negativos."""
    class Rango:
        def __init__(self, inicio, final, paso):
            self.__actual = inicio
            self.__final = final
            self.__paso = paso

        def __repr__(self):
            return f"Rango({self.__actual!r}, {self.__final!r}, paso={self.__paso!r})"

        def __iter__(self):
            return self

        def __next__(self):
            if (self.__paso > 0 and self.__actual >= self.__final) or \
               (self.__paso < 0 and self.__actual <= self.__final):
                raise StopIteration
            actual = self.__actual
            self.__actual = round(self.__actual + self.__paso, pres)
            return actual

    if len(args) == 1:
        inicio, final, paso = 0, args[0], 1
    elif len(args) == 2:
        inicio, final, paso = args[0], args[1], 1
    elif len(args) == 3:
        inicio, final, paso = args
    else:
        raise AttributeError("rango espera start end and step")
    return Rango(inicio, final, paso)

print(list(rango(0, 10, 2)))        # [0, 2, 4, 6, 8]
print(list(rango(2, 1, -0.2)))      # [2, 1.8, 1.6, 1.4, 1.2]
print(repr(rango(0, 10, 2)))        # Rango(0, 10, paso=2)

# 6. Slot de memoria
import sys

class Punto:
    """Punto con huecos fijos."""
    __slots__ = ("x", "y")

    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

class PuntoComun:
    """Punto con dict por objeto."""
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

p = Punto(1, 2)
# print(p.__dict__)  → AttributeError
# p.z = 3            → AttributeError

def huella(objetos):
    total = sum(sys.getsizeof(o) for o in objetos)
    for o in objetos:
        if hasattr(o, "__dict__"):
            total += sys.getsizeof(o.__dict__)
    return total

print(huella([PuntoComun(i, i) for i in range(10_000)]))  # 1360000 (aprox)
print(huella([Punto(i, i) for i in range(10_000)]))        # 480000  (aprox)

# 7. Context manager de cafetera
class Cafetera:
    """Cafetera que se apaga sola al salir del contexto."""

    def __init__(self, tazas: int = 2):
        self.tazas = tazas
        self.encendida = False

    def __enter__(self):
        self.encendida = True
        print(f"Llenando {self.tazas} tazas...")
        return self

    def servir(self):
        if not self.encendida:
            raise RuntimeError("la cafetera esta apagada")
        print("taza servida")

    def __exit__(self, tipo, valor, traceback):
        self.encendida = False
        print("Cafetera apagada.")
        return False

try:
    with Cafetera(1) as cafetera:
        cafetera.servir()
        raise ValueError("se acabo el cafe")
except ValueError:
    print("atrapado: la excepcion salio despues de apagar")

from contextlib import contextmanager

@contextmanager
def prendiendo(nombre):
    print(f"-> {nombre} encendida")
    yield nombre
    print(f"-> {nombre} apagada")

with prendiendo("la pileta") as cosa:
    print("nadando en", cosa)

# 8. Conteo con __call__
class Contador:
    """Decorador que cuenta las llamadas a una función."""

    def __init__(self, funcion):
        self.funcion = funcion
        self.llamadas = 0

    def __call__(self, *args, **kwargs):
        self.llamadas += 1
        return self.funcion(*args, **kwargs)

@Contador
def saludar(nombre):
    return f"hola {nombre}"

print(saludar("Luca"))     # hola Luca
print(saludar("Micaela"))  # hola Micaela
print(saludar.llamadas)    # 2

# 9. El censo de nacimientos
class Alma:
    """Cuenta sus propios nacimientos desde __new__."""

    nacidas = 0

    def __new__(cls, *args, **kwargs):
        cls.nacidas += 1
        return super().__new__(cls)

    def __init__(self, nombre):
        self.nombre = nombre

clara = Alma("Clara")
fede = Alma("Fede")
roco = Alma("Roco")
print(Alma.nacidas)    # 3
```

Después de este capítulo, tus objetos hablan el idioma del lenguaje: suman, se imprimen lindos, se miden, se cortan, se recorren, abren y cierran recursos y hasta se llaman a sí mismos. Pero queda una pregunta incómoda flotando, y la viste asomar dos veces: ese `self.__inicio` del `rango` — ¿por qué el doble guion? ¿Y por qué el capítulo 12 decía que "no hay privados" cuando todo el mundo esconde atributos a dos dedos?

Esa tensión es el corazón del próximo capítulo de la Parte VI: **la encapsulación sin decreto**. El rebelde no tiene `private`, pero tiene un lenguaje de sombras — `_`, `__`, `property`, `@property` — y la filosofía de "somos adultos". Nos vemos en el capítulo 14.