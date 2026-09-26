# Capítulo 17 — La fábrica de las clases: excepciones propias, metaclases, descriptores y Enum

La Parte VI te regaló el objeto desde afuera hacia adentro: el molde (capítulo 12), el espejo (capítulo 13), los secretos (capítulo 14), la familia (capítulo 15) y los contratos (capítulo 16). Falta una sola pieza: **la maquinaria que fabrica a las fábricas**. Y con ella, cobramos dos promesas que quedaron anotadas en el camino.

En el capítulo 6 viste `raise MiError` y te quedaste con una buena pregunta: ¿qué tiene adentro una excepción propia, y cómo la hago a medida? El capítulo 15 te mostró por qué `MiError` servía sin siquiera definirse; el capítulo 6 prometió que la Parte VI profundizaría. Bueno: acá estamos.

Y hay más. Aprendiste que las clases son objetos (capítulo 12), que los objetos se construyen con `__new__` y se inicializan con `__init__` (capítulo 13) y que los decoradores son azúcar de una envoltura que pasa en el momento de la definición (capítulo 8). Hoy vas a ver cómo esas tres ideas se multiplican: hay una clase de las clases, un gancho en el nacimiento de toda clase, un vigilante por atributo, y un tipo nuevo que junta valores fijos con nombres. Para el final, el rebelde te muestra la maquinaria por dentro — no para que la uses todos los días, sino para que sepas qué hay adentro cuando la veas pasar.

## 1. La excepción con equipaje: `__init__` a medida

Tu excepción puede llevar datos propios. Esta clase guarda el sueldo que quedó fuera de rango:

```python
class SalarioRango(Exception):
    """Se lanza cuando un sueldo queda fuera del rango permitido."""

    def __init__(self, sueldo, mensaje="Fuera de rango"):
        self.sueldo = sueldo
        self.mensaje = mensaje
        super().__init__(mensaje)
```

El `super().__init__(mensaje)` del final es el detalle que no hay que olvidar: le pasa el mensaje a la cadena `BaseException`, que es la que guarda `args` y la que usa `str()` al imprimir. Si te lo salteás, tu error quedará mudo:

```python
error = SalarioRango(200000)
print(error)          # Fuera de rango
print(error.args)     # ('Fuera de rango',)
print(error.sueldo)   # 200000
print(SalarioRango.__mro__)
# (<class '__main__.SalarioRango'>, <class 'Exception'>,
#  <class 'BaseException'>, <class 'object'>)
```

Y en el `try` de tu programa, la excepción llega con su equipaje intacto:

```python
try:
    sueldo = 200000
    if not 10000 < sueldo < 150000:
        raise SalarioRango(sueldo)
    print("Sueldo:", sueldo)
except SalarioRango as e:
    print("problema:", e, "| sueldo pedido:", e.sueldo)
# problema: Fuera de rango | sueldo pedido: 200000
```

Fijate el mensaje en dos capas: el `e` de dentro del `except` se imprime como el mensaje (`Fuera de rango`), y `e.sueldo` te deja leer el dato del dominio. Eso es lo que convierte a un error en una pieza de información: **el fracaso también sabe explicarse**.

> **Dato clave:** con un `__init__` a medida, el fracaso sabe explicarse: `str(error)` muestra el mensaje, `.args` guarda la tupla, y los atributos propios como `e.sueldo` cargan el dato del dominio.

## 2. Tu excepción, tu idioma: el `pidenumero`

La excepción propia más austera es una clase vacía que funciona como etiqueta, y un juego de adivinanza la muestra en acción con dos tipos de `except` hermanos:

```python
class MiError(Exception):
    """La excepción propia más simple: una etiqueta con nombre."""

    pass


def pidenumero(numero):
    """Adivina el número secreto: el 20."""
    try:
        numero = int(numero)
        if numero == 20:
            print("Adivinaste.")
        else:
            raise MiError
    except MiError:
        print("Ese no era el número.")
    except:
        print("Hubo un error.")
    print("Buen día.")
```

Observá cada módulo del mecanismo. `raise MiError` (sin paréntesis) funciona porque Python instancia la clase por vos: es atajo de `raise MiError()`. Y el `except MiError` que va primero captura exactamente ese error; el `except:` pelado que le sigue es el anti-patrón del capítulo 6, y aparece acá para capturar lo que no alcanzaste a anticipar:

```python
pidenumero(-1j)
# Hubo un error.   <- int(-1j) lanza TypeError y cae en el except pelado
# Buen día.
pidenumero(12)
# Ese no era el número.
# Buen día.
pidenumero(20)
# Adivinaste.
# Buen día.
```

Tres caminos con tres mensajes: el que acertó, el que se desvió (con su `MiError` nombre y apellido), y el que rompió el molde por completo. Ahí está todo: `int(-1j)` lanza `TypeError` (los complejos, como viste, no se convierten a entero real), y como no hay un `except TypeError`, cae en el `except:` genérico que dice "Hubo un error."

Visto con ojos del capítulo 6, el `except:` pelado es un gesto peligroso: se traga *cualquier* cosa, incluso errores ajenos a tu lógica. Sirve como red de seguridad, pero vos, adulto, preferirás nombrar lo que esperás y dejar que lo imprevisto explote. Y sobre el idioma: levantar `raise Exception("No es un catálogo permitido")` para cualquier fracaso significa que nadie puede distinguir un fracaso del otro. Ahora ya sabés la alternativa: dales nombre propio, como `MiError` o `SalarioRango`.

> **Importante:** el `except:` pelado se traga *cualquier* cosa, incluso errores ajenos a tu lógica — es el anti-patrón del capítulo 6: nombrá lo que esperás y dejá que lo imprevisto explote.

## 3. La fábrica de clases: `type`

En el capítulo 12 viste que `[1, 2, 3]` es un objeto de la clase `list`. ¿Y de qué clase es la clase `list`? Mirá:

```python
print(type([1, 2, 3]))   # <class 'list'>
print(type(list))        # <class 'type'>
print(type(int))         # <class 'type'>
print(type(type))        # <class 'type'>
```

Las clases son objetos, y los objetos de las clases son objetos de `type`. `type` es **la clase de las clases** — la fábrica que fabrica a las fábricas. Siguiendo esa idea, la definición de una clase no es magia: es una llamada a `type` con tres argumentos. El azúcar `class Punto:` esconde esto:

```python
Punto = type("Punto", (object,), {"x": 15, "y": 33})
print(Punto)            # <class '__main__.Punto'>
print(type(Punto))      # <class 'type'>
p = Punto()
print(p.x, p.y)         # 15 33
```

`type(nombre, bases, espacio)` recibe: el nombre de la clase, una tupla de clases de las que hereda, y un diccionario con las variables de clase y métodos. Con eso te alcanza para fabricar una clase al vuelo, sin `class`. El rebelde hace que esto sea posible, aunque en la práctica vas a escribir `class` casi siempre — porque es más legible.

## 4. Las metaclases: tu propia fábrica

Si `type` es la fábrica por defecto, una **metaclase** es la fábrica a medida: una clase que fabrica clases, y que participa en ese nacimiento. Ojo al cambio de escenario: el `__new__` que viste en el capítulo 13 fabricaba una instancia; el que estás por ver fabrica **la clase entera**. Se declara como una clase que hereda de `type`, y se engancha en `__new__`:

```python
class MetaSensible(type):
    """Metaclase que anuncia cada clase que ayuda a fabricar."""

    def __new__(mcs, nombre, bases, espacio):
        print(f"Creando la clase {nombre} ...")
        return super().__new__(mcs, nombre, bases, espacio)
```

Y para que una clase use esa fábrica, se lo decís con `metaclass=`:

```python
class Robot(metaclass=MetaSensible):
    """Un robot: ejemplo de metaclase participando al nacer."""

    def saludar(self):
        return "Soy un robot"

# Creando la clase Robot ...

print(type(Robot))         # <class '__main__.MetaSensible'>
print(Robot().saludar())   # Soy un robot
```

El `print` aparece **en el momento en que se define `Robot`**, no cuando se usa. Ahí está el punto: la metaclase interviene en el nacimiento de la clase, igual que el decorador interviene en el nacimiento de una función (capítulo 8), y que la clase-decorador interviene como capítulo 13 te adelantó.

Las metaclases también pueden acumular conocimiento. Como una clase es un objeto, puede tener atributos que se van llenando a medida que la fábrica trabaja:

```python
class MetaRegistro(type):
    """Metaclase que lleva la lista de las clases que fabrica."""

    creadas = []

    def __new__(mcs, nombre, bases, espacio):
        clase = super().__new__(mcs, nombre, bases, espacio)
        mcs.creadas.append(nombre)
        return clase


class Guerrera(metaclass=MetaRegistro):
    """Una guerrera del registro."""

    pass


class Mago(metaclass=MetaRegistro):
    """Un mago del registro."""

    pass


print(MetaRegistro.creadas)   # ['Guerrera', 'Mago']
```

Un registro que se autocompleta. Es exactamente la clase de truco que usan los *frameworks* (marcos de trabajo) para saber qué clases les fueron declaradas, y para registrarlas sin que nadie tenga que anotarlas a mano. Para el rebelde esto es un 10; para el adulto, una herramienta de baja frecuencia que solo se justifica cuando escribir a mano saldría más caro.

> **Dato clave:** como una clase es un objeto, puede acumular conocimiento: el atributo `creadas` se llena solo, mientras la fábrica trabaja. Ese es el truco que usan los frameworks para registrar clases sin que nadie las anote a mano.

## 5. Descriptores: el vigilante de un atributo

Con la clase vacía y el protocolo `__get__`/`__set__`, podés crear un **descriptor**: un objeto que se adueña de un atributo y controla cómo se lee y se escribe. Un termómetro lo deja claro:

```python
class RangoValidado:
    """Descriptor: controla el pase de entrada y salida de un atributo."""

    def __init__(self, minimo=0, maximo=100):
        self.minimo = minimo
        self.maximo = maximo

    def __set_name__(self, duenio, nombre):
        self.nombre = nombre

    def __get__(self, objeto, tipo=None):
        if objeto is None:
            return self
        return objeto.__dict__[self.nombre]

    def __set__(self, objeto, valor):
        if not (self.minimo <= valor <= self.maximo):
            raise ValueError(
                f"{self.nombre} debe estar entre {self.minimo} y {self.maximo}")
        objeto.__dict__[self.nombre] = valor


class Termometro:
    temperatura = RangoValidado(-20, 120)
    volumen = RangoValidado(0, 10)
```

Cada método tiene un papel: `__set_name__` avisa al descriptor cuál es el nombre del atributo que vigila; `__get__` responde cuando se lee (devolviendo el valor guardado, o a sí mismo si se pide desde la clase); `__set__` es el portero, y puede rechazar un valor indebido con `ValueError`. En acción:

```python
t = Termometro()
t.temperatura = 30
print(t.temperatura)                   # 30
print(type(Termometro.temperatura).__name__)   # RangoValidado
try:
    t.volumen = 500
except ValueError as e:
    print(e)                           # volumen debe estar entre 0 y 10
```

Dos detalles finos. El valor no vive en el descriptor: vive en el `__dict__` del objeto dueño (`t.__dict__["temperatura"]`), y el descriptor solo regula el pase. Y si pedís `Termometro.temperatura` (desde la clase, no desde una instancia), el descriptor se muestra a sí mismo — es la señal de que seguís hablando con el portero, no con el valor.

¿Te suena familiar el patrón? Claro: `@property` del capítulo 14 es, por dentro, un descriptor. Ahora ya tenés la vista de rayos X que te muestra el truco debajo del decorador. Los descriptores se escriben con poca frecuencia — pero leerlos, cuando aparecen en librerías y frameworks, te deja ver exactamente quién está vigilando cada atributo.

> **Dato clave:** `@property` del capítulo 14 es, por dentro, un descriptor: trabajás con descriptores todo el tiempo sin saberlo, y ahora ya sabés leer el truco debajo del decorador.

## 6. Descriptores con herencia: una lista que exige tipo

Y ahora, la historia más divertida de la clase: una lista que solo acepta un tipo de dato, con un docstring de ejemplos y tests:

```python
class Descriptor(list):
    """Una lista que solo acepta un tipo de dato.

    Sobrescribe append para validar lo que entra.

    >>> desc = Descriptor(int)
    >>> desc.append(1)
    >>> desc.append(2)
    >>> desc.append(3)
    >>> print(desc)
    [1, 2, 3]
    """

    def __init__(self, data_type):
        self.data_type = data_type
        super().__init__()

    def append(self, item):
        if isinstance(item, self.data_type):
            super().append(item)
        else:
            raise TypeError(
                f"Invalid data type. Descriptor can only store "
                f"{self.data_type.__name__} data type.")
```

Miralo por dentro: `Descriptor(list)` hereda todo el comportamiento de las listas del capítulo 12; su única contribución es controlar la puerta de entrada. `__init__` guarda el tipo exigido y `append` decide: si lo que llega es del tipo pedido (`isinstance`, capítulo 12), pasa directo al `append` original; si no, `TypeError` con un mensaje que nombra el tipo esperado.

```python
desc = Descriptor(int)
desc.append(1)
desc.append(2)
desc.append(3)
print(desc)                      # [1, 2, 3]
print(isinstance(desc, list))    # True
try:
    desc.append(1.5)
except TypeError as e:
    print(e)                     # Invalid data type. Descriptor can only store int data type.
```

Y acá viene la aclaración honesta, la que te vuelve adulto: este `Descriptor` **no es** el descriptor del protocolo de la sección 5. Es una lista tipada que se llama "descriptor" por casualidad, y el nombre, suelto, confunde: hay dos consolas distintas usando la misma palabra. Lo importante no es pelear por la semántica: es saber leer código con precisión y verificar que el resultado haga lo que promete. El docstring trae ejemplos con `>>>`, así que podés dejar que `doctest` revise el ejemplo (lo vas a hacer en los ejercicios). Y el guardián de `isinstance` convierte a esta lista en el molde exacto: almacenar un solo tipo de dato.

## 7. `Enum`: tipos con nombre

Última pieza de la fábrica, y quizás la más cotidiana. Cuando un atributo solo puede tomar un conjunto fijo de valores — los géneros de un libro, los días de la semana —, `Enum` los agrupa en un tipo con miembros nombrados:

```python
from enum import Enum


class TipoLibro(Enum):
    """Los géneros posibles de un libro."""

    FICCION    = "ficcion"
    NO_FICCION = "no ficcion"
    BIOGRAFIA  = "biografia"
    AUTOAYUDA  = "autoayuda"


print(TipoLibro.FICCION)            # TipoLibro.FICCION
print(TipoLibro.FICCION.name)       # FICCION
print(TipoLibro.FICCION.value)      # ficcion
print(type(TipoLibro))              # <class 'enum.EnumMeta'>
print(list(TipoLibro))              # [<TipoLibro.FICCION: 'ficcion'>, ...]
print(TipoLibro("ficcion"))         # TipoLibro.FICCION
```

Otro espejismo del capítulo 13, y dos anotaciones: cada miembro tiene `.name` (el identificador estable, para el código) y `.value` (el dato humano, para mostrar); `TipoLibro("ficcion")` permite viajar desde el valor hacia el miembro; y — sorpresa — `type(TipoLibro)` es `enum.EnumMeta`. Un `Enum` es una clase fabricada por una metaclase de la biblioteca estándar: la sección 4, viva en plena biblioteca. Ni un atributo que se te escape: `match` y las comparaciones comparan por identidad del miembro, así que `TipoLibro.FICCION == TipoLibro.FICCION` y nadie se confunde con el valor.

> **Dato clave:** `type(TipoLibro)` es `enum.EnumMeta`: un `Enum` es una clase fabricada por una metaclase de la biblioteca estándar. La sección 4, viva en plena biblioteca.

## 8. `Enum` + `match`: la vitrina de la rebeldía

Juntemos lo mejor del capítulo 5 con lo mejor de hoy. Un libro con su género tipado:

```python
class Libro:
    """Un libro: título, autor y género."""

    def __init__(self, titulo, autor, tipo):
        self.titulo = titulo
        self.autor = autor
        self.tipo = tipo

    def __str__(self):
        return (f"El libro '{self.titulo}' fue escrito por {self.autor}; "
                f"género: {self.tipo.value}")


libros = [
    Libro("Cien años de soledad", "García Márquez", TipoLibro.FICCION),
    Libro("Sapiens", "Harari", TipoLibro.NO_FICCION),
    Libro("Steve Jobs", "Isaacson", TipoLibro.BIOGRAFIA),
    Libro("El poder del ahora", "Tolle", TipoLibro.AUTOAYUDA),
]

for libro in libros:
    print(libro)
# El libro 'Cien años de soledad' fue escrito por García Márquez; género: ficcion
# El libro 'Sapiens' fue escrito por Harari; género: no ficcion
# El libro 'Steve Jobs' fue escrito por Isaacson; género: biografia
# El libro 'El poder del ahora' fue escrito por Tolle; género: autoayuda
```

Y la versión `match`, con una frase distinta por género:

```python
def imprimir_info_libro(libro):
    """Cuenta sobre un libro según su género, con match del capítulo 5."""
    match libro.tipo:
        case TipoLibro.FICCION:
            print(f"El libro de ficción '{libro.titulo}' fue escrito por {libro.autor}.")
        case TipoLibro.NO_FICCION:
            print(f"El libro de no ficción '{libro.titulo}' fue escrito por {libro.autor}.")
        case TipoLibro.BIOGRAFIA:
            print(f"La biografía '{libro.titulo}' fue escrita por {libro.autor}.")
        case TipoLibro.AUTOAYUDA:
            print(f"El libro de autoayuda '{libro.titulo}' fue escrito por {libro.autor}.")


for libro in libros:
    imprimir_info_libro(libro)
# El libro de ficción 'Cien años de soledad' fue escrito por García Márquez.
# El libro de no ficción 'Sapiens' fue escrito por Harari.
# La biografía 'Steve Jobs' fue escrita por Isaacson.
# El libro de autoayuda 'El poder del ahora' fue escrito por Tolle.
```

Cada `case` compara contra un miembro del `Enum`, y el `match` elige el que corresponde. No hay cadenas sueltas: el género es un tipo con nombre, el `match` se vuelve legible y el compilador calla. La vitrina que junta las dos herramientas más modernas de esta parte del libro.

## 9. La decisión honesta y el cierre de la rebelión

Después de recorrer la fábrica, hacete la misma pregunta de todos los capítulos: ¿qué uso a diario, y qué me guardo para cuando lo necesite?

| Herramienta | Para qué | Frecuencia con la que la vas a usar |
| --- | --- | --- |
| Excepción propia | fracasos del dominio con nombre y datos propios (`SalarioRango`) | seguido |
| `except` específicos | escalonar la caza de errores, sin el anti-patrón pelado | a cada rato |
| `Enum` + `match` | conjuntos fijos de valores con nombre y ramas legibles | seguido |
| `type` con tres argumentos | fabricar una clase al vuelo, entender las clases como objetos | entenderlo, usarlo poco |
| Metaclase | intervenir en el nacimiento de clases (registros, validaciones) | rara — librerías y frameworks |
| Descriptor (protocolo) | vigilar un atributo puntual; es el motor de `@property` | rara de escribir, constante de leer |

El rebelde de este capítulo no te presenta un checklist para hacer todo: te presenta la totalidad del capó, y el adulto elige con criterio. Las excepciones propias y los `Enum` van a tu mochila cotidiana; las metaclases y los descriptores, a la caja de herramientas de emergencia — y cuando una librería los use, vas a reconocerlos sin asustarte.

## 10. Resumen y conceptos clave

Acá termina la Parte VI, la rebelión del objeto. Mirá el arco completo: en el capítulo 12 moldeaste la sustancia; en el 13 le diste espejo; en el 14 viste sus secretos y aprendiste que la encapsulación es cortesía; en el 15 descubriste la familia por herencia; en el 16 aceptaste que el comportamiento manda por encima del parentesco; y hoy conociste la fábrica donde nace todo. Ya no hay pieza de Python que sea "magia": hay fábricas, porteros y etiquetas. Y si algo te queda para la Parte VII, recordá esta frase: ahora sabés de qué habla el lenguaje cuando echa a andar su maquinaria — y podés echarla a andar vos también.

Repasá la lista y fijate de que cada punto te resulte familiar:

- [ ] **Excepción a medida con equipaje**: `__init__` propio para guardar datos del dominio + `super().__init__(mensaje)` para que `str()` y `.args` sigan contando el mensaje.
- [ ] **`raise MiError` y `raise MiError(...)`**: la clase se instancia sola; elegí la versión que exprese mejor.
- [ ] **`except` hermanos**: `except MiError` captura lo específico, `except:` pelado captura todo — el anti-patrón del capítulo 6, a usar con ojos bien abiertos.
- [ ] **`type` es la clase de las clases**: `type(type)` es `type`; todas las clases, incluso `list` e `int`, son objetos.
- [ ] **`type(nombre, bases, espacio)`** fabrica una clase al vuelo; `class Punto:` es azúcar de esa llamada.
- [ ] **Metaclase** (`class Meta(type)`): interviene en el nacimiento de clases desde `__new__`; se elige con `class Algo(metaclass=Meta)`.
- [ ] Las clases son objetos → pueden tener atributos de clase que se llenan al fabricar (el registro `creadas`).
- [ ] **Descriptor**: protocolo `__get__`/`__set__`/`__set_name__` que vigila un atributo; la policía del valor, muchas veces con `ValueError` a mano.
- [ ] Los descriptores no guardan el valor: lo guardan en el `__dict__` del objeto dueño.
- [ ] **`@property` del capítulo 14 es un descriptor** por dentro: ya podés leer el truco debajo del decorador.
- [ ] El `Descriptor` que hereda de `list` es una lista tipada por `isinstance`, no el protocolo del descriptor — dos significados para la misma palabra, y está bien mientras sepas cuál es cuál.
- [ ] **`Enum`**: conjunto fijo de valores con nombre; `.name` (código) y `.value` (humano), `list(TipoLibro)`, viaje inverso con `TipoLibro("ficcion")`.
- [ ] **`Enum` + `match`**: `case TipoLibro.FICCION:` compara por identidad; `type(TipoLibro)` es `EnumMeta`, una metaclase de la biblioteca estándar que ya conocés.

---

## 11. Ejercicios

1. **La excepción con equipaje**: clase `CuentaError(Exception)` con `__init__(monto, saldo)` que guarde ambos y pase el mensaje "Saldo insuficiente" al `super().__init__`. Una función `retirar(saldo, monto)` que lance `CuentaError(monto, saldo)` si `monto > saldo`, y si no devuelva el saldo restante. Capturá con `except CuentaError as e` e imprimí el mensaje, `e.monto` y `e.saldo`.
2. **La familia de tus errores**: base `ErrorPago(Exception)`, y dos hijas `SaldoInsuficiente(ErrorPago)` y `TarjetaRechazada(ErrorPago)`. Una función `pagar(medio, monto)` que lance la hija correcta según el medio ("saldo" o "tarjeta"). Capturá con un solo `except ErrorPago as e` y mostrá que atrapa a las dos hijas.
3. **El error con maestro**: clase `NotaFueraDeRangoError(ValueError)` cuyo mensaje sea "La nota debe estar entre 1 y 10". Una función `registrar(nota)` que lance el error si `nota` escapa del rango 1–10. Llamala con `7` y mostrá que funciona; llamala con `11` dentro de un `try/except` y mostrá el mensaje.
4. **La clase al vuelo**: con `type("PuntoImaginario", (object,), {...})` fabricá una clase con `x=3`, `y=4` y un método `modulo` que devuelva la raíz de `x**2 + y**2`. Instanociala y mostrá `modulo` (5.0) y `type(...)` de la clase.
5. **La metaclase contadora**: metaclase `MetaContadora(type)` que incremente un atributo de clase `contador` en cada `__new__`. Definí dos clases con esa metaclase y mostrá `MetaContadora.contador`.
6. **El descriptor NoNegativo**: descriptor que solo acepte valores mayores o iguales a cero, con `__set_name__`, `__get__` y `__set__` (levantando `ValueError` con "no puede ser negativo"). Usalo en `Heladera.temperatura`, seteá `4` (que funcione) y `-1` (que explote bonito).
7. **El Enum del día**: `class Dia(Enum)` con `LUNES="lunes"`, `DOMINGO="domingo"` y los días que quieras en el medio. Función `que_hace(dia)` con `match`: `LUNES` → "arrancar la semana", `DOMINGO` → "descansar", `case _:` → "trabajar". Probalo con los tres caminos.
8. **El Descriptor con examen**: reproducí la clase `Descriptor(list)` de la sección 6 con su docstring de ejemplos, y al final ejecutá el examen:

```python
if __name__ == "__main__":
    import doctest
    resultados = doctest.testmod()
    print("tests:", resultados.attempted, "| fallos:", resultados.failed)
```

Correla como script (`python cap_17_ejercicios.py`) y dejá que `doctest` te diga cuántos ejemplos pasaron.

```python
# Soluciones (no las mires antes de intentarlo)
# 1. La excepción con equipaje
class CuentaError(Exception):
    """Se lanza cuando no alcanza el saldo."""

    def __init__(self, monto, saldo, mensaje="Saldo insuficiente"):
        self.monto = monto
        self.saldo = saldo
        super().__init__(mensaje)


def retirar(saldo, monto):
    if monto > saldo:
        raise CuentaError(monto, saldo)
    return saldo - monto


print(retirar(100, 40))           # 60
try:
    retirar(100, 250)
except CuentaError as e:
    print(e, "|", e.monto, "|", e.saldo)
# Saldo insuficiente | 250 | 100

# 2. La familia de tus errores
class ErrorPago(Exception):
    """La raíz: cualquier falla a la hora de pagar."""

    pass


class SaldoInsuficiente(ErrorPago):
    """El saldo no alcanza."""

    pass


class TarjetaRechazada(ErrorPago):
    """La tarjeta no pasó."""

    pass


def pagar(medio, monto):
    if medio == "saldo":
        raise SaldoInsuficiente("no alcanza el saldo")
    if medio == "tarjeta":
        raise TarjetaRechazada("la tarjeta fue rechazada")


for medio in ("saldo", "tarjeta"):
    try:
        pagar(medio, 100)
    except ErrorPago as e:
        print(type(e).__name__, "-", e)
# SaldoInsuficiente - no alcanza el saldo
# TarjetaRechazada - la tarjeta fue rechazada

# 3. El error con maestro
class NotaFueraDeRangoError(ValueError):
    """Se lanza cuando la nota escapa del rango 1-10."""

    def __init__(self, nota):
        super().__init__("La nota debe estar entre 1 y 10")


def registrar(nota):
    if not 1 <= nota <= 10:
        raise NotaFueraDeRangoError(nota)
    return f"nota {nota} registrada"


print(registrar(7))               # nota 7 registrada
try:
    registrar(11)
except NotaFueraDeRangoError as e:
    print(e)                      # La nota debe estar entre 1 y 10

# 4. La clase al vuelo
PuntoImaginario = type("PuntoImaginario", (object,), {
    "x": 3,
    "y": 4,
    "modulo": lambda self: (self.x ** 2 + self.y ** 2) ** 0.5,
})
p = PuntoImaginario()
print(p.modulo())                 # 5.0
print(type(PuntoImaginario))      # <class 'type'>

# 5. La metaclase contadora
class MetaContadora(type):
    """Metaclase que cuenta las clases que fabrica."""

    contador = 0

    def __new__(mcs, nombre, bases, espacio):
        clase = super().__new__(mcs, nombre, bases, espacio)
        mcs.contador += 1
        return clase


class Primera(metaclass=MetaContadora):
    pass


class Segunda(metaclass=MetaContadora):
    pass


print(MetaContadora.contador)     # 2

# 6. El descriptor NoNegativo
class NoNegativo:
    """Descriptor: solo acepta valores mayores o iguales a cero."""

    def __set_name__(self, duenio, nombre):
        self.nombre = nombre

    def __get__(self, objeto, tipo=None):
        if objeto is None:
            return self
        return objeto.__dict__[self.nombre]

    def __set__(self, objeto, valor):
        if valor < 0:
            raise ValueError(f"{self.nombre} no puede ser negativo")
        objeto.__dict__[self.nombre] = valor


class Heladera:
    temperatura = NoNegativo()


h = Heladera()
h.temperatura = 4
print(h.temperatura)              # 4
try:
    h.temperatura = -1
except ValueError as e:
    print(e)                      # temperatura no puede ser negativo

# 7. El Enum del día
from enum import Enum


class Dia(Enum):
    """Los días de la semana con su valor humano."""

    LUNES    = "lunes"
    MARTES   = "martes"
    MIERCOLES = "miercoles"
    JUEVES   = "jueves"
    VIERNES  = "viernes"
    SABADO   = "sabado"
    DOMINGO  = "domingo"


def que_hace(dia):
    match dia:
        case Dia.LUNES:
            return "arrancar la semana"
        case Dia.DOMINGO:
            return "descansar"
        case _:
            return "trabajar"


print(que_hace(Dia.LUNES))        # arrancar la semana
print(que_hace(Dia.DOMINGO))      # descansar
print(que_hace(Dia.MIERCOLES))    # trabajar

# 8. El Descriptor con examen
class Descriptor(list):
    """Una lista que solo acepta un tipo de dato.

    >>> desc = Descriptor(int)
    >>> desc.append(1)
    >>> desc.append(2)
    >>> desc.append(3)
    >>> print(desc)
    [1, 2, 3]

    >>> desc.append(1.5)
    Traceback (most recent call last):
        ...
    TypeError: Invalid data type. Descriptor can only store int data type.
    """

    def __init__(self, data_type):
        self.data_type = data_type
        super().__init__()

    def append(self, item):
        if isinstance(item, self.data_type):
            super().append(item)
        else:
            raise TypeError(
                f"Invalid data type. Descriptor can only store "
                f"{self.data_type.__name__} data type.")


if __name__ == "__main__":
    import doctest
    resultados = doctest.testmod()
    print("tests:", resultados.attempted, "| fallos:", resultados.failed)
# tests: 6 | fallos: 0
```