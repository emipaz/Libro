# Capítulo 12 — Moldes y objetos: la POO sin rejas

El capítulo 12.0 preparó el terreno: la historia de la POO, sus cuatro pilares clásicos y el porqué de que Python los tome por convención en lugar de por decreto. Quedó pendiente lo que importa de verdad: la construcción. Y la puerta por donde entra es la promesa del capítulo 11: los **métodos** — esos `.split()`, `.capitalize()` y `.items()` que usás desde el capítulo 3 — son la primera fusión de *dato y comportamiento* en una misma cosa. Eso que usaste sin saberlo tiene nombre, apellido y llegó la hora de construirlo vos, molde en mano: es la **Programación Orientada a Objetos (POO)**, y ocupa toda la Parte VI.

Y un spoiler con identidad propia, que completa el cuadro del prólogo. Cuando en la Parte I aprendiste que *todo en Python es un objeto*, quizá lo anotaste como una frase linda y seguiste. Acá la frase se convierte en el alma del capítulo: Python hace POO **sin rejas**. En Java o C# te *obligan*: `private` para esconder, `final` para congelar, `interface` para prometer, clases abstractas para obligar herencia. Python, el rebelde, hace lo mismo pero **por convención, no por decreto** — te deja la responsabilidad y te dice "somos adultos". En este capítulo vas a abrir la caja de herramientas de esa POO libre, y los próximos cinco la van a destapar hasta el fondo.

---

## 1. La lista de los rebeldes

Los materiales que inspiran esta parte arrancan con un inventario que no tiene desperdicio: **las particularidades de Python**. Es la lista de qué hace distinto a este lenguaje dentro del mundo orientado a objetos. Mirala completa, y después la vamos desmenuzando capítulo a capítulo:

> **Particularidades de Python:**
> - Todo es un objeto, incluyendo los tipos y las clases.
> - Tipos y clases son sinónimos.
> - Permite herencia múltiple.
> - No existen métodos ni atributos privados.
> - Los atributos pueden ser modificados directamente.
> - Las clases abstractas son opcionales, pero pueden ser implementadas.
> - Permite **monkey patching**.
> - Permite **duck typing**.
> - Permite **mixins**.
> - Permite la **sobrecarga de operadores**.
> - Permite la creación de **nuevos tipos de datos**.

Cada ítem de esa lista es una "rebeldía" frente a los lenguajes estrictos, y cada una tiene su capítulo. Como hoja de ruta de la Parte VI, armá este mapa en tu cabeza:

| La rebeldía de Python | Qué implica | Dónde la vemos |
|-----------------------|-------------|----------------|
| Todo es un objeto | incluso los tipos y las clases son objetos | este capítulo (y su culminación en capítulo 17) |
| Tipos y clases son sinónimos | `type(x)` devuelve una clase | este capítulo |
| No hay privados | `_` y `__` son convención, no decreto | capítulo 14 |
| Atributos modificables al toque | sumar, cambiar y eliminar con un punto | este capítulo |
| Monkey patching | parchear en caliente sin tocar el molde | este capítulo |
| Duck typing | importa el comportamiento, no la etiqueta | capítulo 16 |
| Herencia múltiple | mezclar varios moldes | capítulo 15 |
| Sobrecarga de operadores | darle sentido propio a `+`, `*`, `==`… | capítulo 13 |
| Mixins | piezas de comportamiento reutilizables | capítulo 16 |
| Clases abstractas opcionales | el rebelde no te obliga a prometer | capítulo 16 |
| Nuevos tipos de datos | una clase es una fábrica de tipos | este capítulo y el 17 |

La tesis que vas a escuchar cuatro veces en la Parte VI es esta: **Python orientado a objetos está diseñado para confiar en vos.** Las reglas existen, pero son reglas de la casa (convenciones), no leyes con policía. Una vez que lo sientas, vas a leer cualquier código de la industria con otros ojos.

> **Dato clave — el nombre de la parte:** por eso la Parte VI se llama *"Python, el rebelde"*. No es un insulto: es el diagnóstico de un lenguaje que toma de la POO lo que le sirve — clases, herencia, objetos — y descarta los candados de los lenguajes que la *imponen* por decreto.

---

## 2. Moldes y objetos

Vamos a lo concreto: **qué es una clase y qué es un objeto.**

Una **clase** es la descripción de una cosa: qué propiedades tiene y qué comportamientos sabe hacer. Un **objeto** (o *instancia*) es una materialización concreta de esa descripción. La analogía clásica es el molde: la clase es el molde de galletitas; cada objeto es una galletita salida de ese molde — todas con la misma forma, cada una con su realidad propia.

> **Otra forma de mirarlo:** ya conocés decenas de clases de memoria, porque los tipos del capítulo 3 son clases. `int` es un molde; `3`, `42` y `-7` son objetos salidos de ese molde. `str` es otro molde; `"hola"` y `"mundo"` son objetos suyos. El tipo de un objeto ES su clase: `type(3)` te contesta `<class 'int'>`. Tipos y clases son sinónimos.

Para crear tu primer molde usás la palabra **`class`** y, como toda sintaxis con cuerpo, no puede quedarse vacía — el `pass` del capítulo 7 es tu amigo para darle un cuerpo mínimo:

```python
class Clase:
    """Una clase mínima, lista para instanciar."""
    pass

print(type(Clase))   # <class 'type'>         → la clase también es un objeto
```

Por convención (PEP 8), los nombres de **clases usan CamelCase** — cada palabra empieza con mayúscula, sin separadores: `PersonajeViajero`, `CuentaBancaria`, `AutoRobot`. Los nombres de variables, funciones y métodos siguen siendo `snake_case`. Es la única diferencia visual que vas a usar para leer rápido.

Y fijate el primer disparo de la lista de rebeldes: `type(Clase)` devuelve `<class 'type'>`. O sea, **la clase misma es un objeto** — instanciada a partir de la clase `type`. Esa es la puerta a las *metaclases* que vas a abrir en el capítulo 17; por ahora, guarda la idea: clases que fabrican objetos, y detrás, una clase que fabrica clases.

Para **instanciar** (crear un objeto a partir del molde) se escribe el nombre de la clase con paréntesis, como si llamaras una función:

```python
class Clase:
    """Una clase mínima."""
    pass

objeto = Clase()          # se crea el objeto y se guarda en la variable
print(type(objeto))       # <class '__main__.Clase'>
```

Si no le ponés nombre, el objeto se crea y el intérprete lo desecha al instante (igual que un literal que no guardás). Y cada instanciación produce un objeto *distinto*: la función `id()` del capítulo 3 te muestra el código de identidad de cada uno:

```python
class Clase:
    """Una clase mínima."""
    pass

objetos = (Clase(), Clase(), Clase())
for elemento in objetos:
    print(id(elemento), elemento)

# 2267481390192 <__main__.Clase object at 0x0000020FE7E61710>
# 2267481390000 <__main__.Clase object at 0x0000020FE7E616D0>
# 2267481389904 <__main__.Clase object at 0x0000020FE7E61690>
```

Tres galletitas del mismo molde, tres identidades distintas.

### El abuelo de todo: `object`

Viene con la sintaxis un detalle histórico. Una clase se puede declarar de tres maneras que acá quedan **exactamente iguales**:

```python
class Minimalista:            # sin paréntesis: la más legible, y la moderna
    pass

class ConParentesis():        # paréntesis vacíos: mismo resultado
    pass

class Abuela(object):         # el estilo de Python 2, hoy innecesario
    pass

print(Minimalista.__bases__)   # (<class 'object'>,)
print(ConParentesis.__bases__) # (<class 'object'>,)
```

¿De dónde sale ese `object`? De que **todo en Python emana de la clase `object`**: si no decís nada, el intérprete asume que tu clase hereda de `object` — la madrastra de todos los moldes. Los paréntesis de la declaración son el lugar donde se declaran las **superclases** de las que tu clase *hereda* características (caja que abrimos completa en el capítulo 15); el `__bases__` que acabo de mostrarte es el atributo que guarda la lista de esas superclases. Sin superclases declaradas, la única es `object`.

> **Dato clave:** `object` es un objeto y una clase a la vez, y todas las clases (también `int`, `str` y la tuya) son descendientes suyas. Lo vas a sentir cuando veas que hasta tu clase más vacía "sabe" hacer cosas que nunca programaste — esas capacidades vienen heredadas de `object`, y la mayoría se esconden entre atributos con doble guion bajo, los famosos *dunders* del capítulo 13.

---

## 3. `self` y `__init__`: el molde se llena

Un molde sirve cuando al sacar cada galletita la personalizás. Ahí entra el método más famoso de Python: **`__init__`** (de *initialize*, inicializar). Es el **constructor**: el primer método que se ejecuta al instanciar, y todos los argumentos que pasás dentro del paréntesis de la clase (`Persona("…)`) llegan directo a él.

Fijate el nombre: empieza y termina con doble guion bajo. Esa es la marca oficial de los **métodos especiales** — los que Python reserva para darle comportamiento básico al objeto. El capítulo 13 los multiplica; acá vas a conocer al primero.

Pero primero, una aclaración que evita la confusión más común al leer una clase: un **método se define con la misma palabra `def` de las funciones del capítulo 7** — mismos parámetros, mismo ámbito local, mismo `return` — pero vive **adentro** de la clase y, sobre todo, **indentado con ella**. Que veas un `def` con una sangría adentro de un `class` no es una función mal indentada ni un error: la indentación de un nivel es *la marca* de que ese `def` pertenece a la clase y pasa a ser un método suyo. `__init__` no escapa a esa regla: es un método más, con un `def` común y corriente — lo único especial es su nombre. Lo que cambia entre una función y un método no es la definición, es **cómo se invoca** (lo vemos ya mismo, recién después de este ejemplo).

Y para escribirlo necesitás entender **`self`**. Mirá el molde más simple con método:

```python
class Modales:
    """Una clase mínima con un método simple."""

    def saludo(self):
        return "Hola."

portero = Modales()
print(portero.saludo())   # Hola.
```

Todo método tiene que declarar —al menos— un parámetro inicial llamado por convención **`self`**. Ese parámetro ES el objeto sobre el que se llamó el método: cuando escribís `portero.saludo()`, Python toma a `portero` y lo mete dentro de `self`. La rebelión de este capítulo es que `self` **no es magia**: es un parámetro posicional como cualquier otro. Estas dos líneas hacen lo mismo:

```python
class Modales:
    """Una clase con un método que ni usa el self."""

    def saludo(self):
        return "Hola."

portero = Modales()
print(portero.saludo())        # Hola.
print(Modales.saludo(portero)) # Hola.  → sugar: esto es lo que pasa por debajo
```

`objeto.metodo(arg)` es azúcar sintáctica de `Clase.metodo(objeto, arg)`. Si el método no usa `self`, podés incluso pasarle otro "objeto" cualquiera:

```python
class Modales:
    """Método que ignora por completo a self."""
    def saludo(self):
        return "Hola."

print(Modales.saludo("pepe"))   # Hola.  → nadie verifica nada
```

Ese es el rebelde en su estado puro: en otros lenguajes el objeto que te llama te **llega magia**; acá llega en un parámetro y el nombre `self` es solo la convención.

Y la pista para leer métodos sin asustarte, ahora que viste uno en vivo: son funciones pero se invocan distinto — `objeto.metodo()` con un punto, en vez de `metodo(objeto)`. El punto no es decorativo: es la promesa de que Python va a meter a `objeto` en el primer parámetro del `def`, el parámetro que por convención se llama `self`.

### `self` es una convención: se llama como quieras

El parámetro se puede llamar de cualquier manera, porque el intérprete no mira la etiqueta: mira el **primer parámetro inicial** y le mete el objeto. Esta clase funciona idéntico aunque su método use `el_objeto` en vez de `self`:

```python
class Renombrable:
    """Mismo molde, otra etiqueta: mirá quién llega al primer parámetro."""

    def saludar(el_objeto):
        return f"Hola, soy {type(el_objeto).__name__}."

p = Renombrable()
print(p.saludar())            # Hola, soy Renombrable.
print(Renombrable.saludar(p)) # Hola, soy Renombrable.  → el azúcar de siempre
```

Podría llamarse `objeto`, `yo_mismo` o `mi_instancia` — todo funciona. Entonces, ¿por qué `self` y no otra cosa? Por dos ventajas prácticas:

- **Legibilidad universal**: cuando abrís cualquier código de Python escrito por cualquiera, `self` te salta a la vista como "el objeto de acá adentro", igual que `for` te salta como un bucle. Es el idioma común de la comunidad.
- **Las herramientas te premian**: los editores e IDEs reconocen la convención y te resaltan `self`, autocompletan sus atributos y te muestran el objeto con más cariño que un nombre inventado.

Es una regla de la casa, como `snake_case` para variables y `CamelCase` para clases (capítulo 12): podés romperla y el código sigue andando, pero te perdés el idioma que comparten todos los Pythonistas.

### El constructor con validación

Ahora la versión seria. Un molde `Persona` que recibe nombre, apellido y edad — y **no deja** que la edad sea un disparate. Combina los hints del capítulo 9, el `try`/`except` del capítulo 6 y el operador walrus del capítulo 3:

```python
class Persona:
    """Una persona con nombre, apellido y edad validada."""

    def __init__(self, nombre: str, apellido: str, edad: int):
        self.nombre = nombre
        self.apellido = apellido
        try:
            if 0 < (edad := int(edad)) < 120:      # walrus: asigna y compara
                self.edad = edad
            else:
                print(edad, "fuera de rango (0-120)")
                self.edad = None
        except ValueError:
            print(edad, "no es un número de edad válido")
            self.edad = None

emi = Persona("Emiliano", "Passarello", 48)
belen = Persona("Belen", "Cianfagna", 42)
print(emi.nombre, emi.edad)     # Emiliano 48
print(belen.nombre, belen.edad) # Belen 42

Persona("Juan", "Perez", 135)         # 135 fuera de rango (0-120)
Persona("Juan", "Perez", "muy grande")# muy grande no es un número de edad válido
```

Desmenuzá lo nuevo:

- `self.nombre = nombre` **crea los atributos del objeto**: de ahora en más cada instancia de `Persona` lleva consigo su propio `nombre`, `apellido` y `edad`. Ese conjunto de valores a un momento dado es el **estado** del objeto.
- La validación usa el walrus del capítulo 3: `(edad := int(edad))` convierte a `int` y guarda el resultado en la variable local `edad`, en la misma expresión donde se compara. Si alguien manda `"muy grande"`, `int()` explota y el `except ValueError` del capítulo 6 lo atrapa.
- Cuando el dato es inválido, el atributo `edad` queda en `None` (como el patrón `lista=None` del capítulo 7): un objeto existe siempre, pero marca claramente que el dato no se pudo cargar.

> **Buenas prácticas:** el constructor es el primer guardián del estado. Validar ahí adentro —"no crees objetos inválidos"— es la cintura mínima de un buen diseño. El capítulo 14 le va a dar el cinturón completo con las `@property`.

---

## 4. Atributos: el estado del objeto

Ya viste que `self.nombre = nombre` crea atributos. Ahora, a fondo: de dónde salen, quién los comparte y cómo se manejan. Miremos un ejemplo base, una clase `Cuadrado` donde cada cuadrado tiene un `lado` de valor 1:

```python
class Cuadrado:
    """Clase que ejemplifica el uso de atributos."""
    lado = 1

cuadrados = (Cuadrado(), Cuadrado(), Cuadrado(), Cuadrado())
print(cuadrados[1].lado)   # 1
```

Acá `lado = 1` es un **atributo de clase**: vive en el molde, y todas las instancias lo *ven* por herencia. Ahora la rebelión: podés darle a un objeto su propio valor, distinto del molde, con una simple asignación:

```python
cuadrados[0].lado = 3      # crea un atributo propio solo para este objeto
cuadrados[3].lado = 11     # lo mismo para el cuarto

for cuadro in cuadrados:
    print(f"id={id(cuadro)} {cuadro.lado=}")

# id=...  cuadro.lado=3    ← la instancia 0 tiene atributo propio
# id=...  cuadro.lado=1    ← la 1 y la 2 no tienen el suyo: ven el de la clase
# id=...  cuadro.lado=1
# id=...  cuadro.lado=11   ← la 3 tiene atributo propio
```

Y acá la jugada que no se puede hacer en muchos lenguajes: **cambiar el valor en el molde mismo**:

```python
Cuadrado.lado = 20         # todos a la vez… pero respetando los propios

for cuadro in cuadrados:
    print(f"id={id(cuadro)} {cuadro.lado=}")

# id=...  cuadro.lado=3     ← siguió con el suyo (3)
# id=...  cuadro.lado=20    ← cambió: tomó el nuevo valor de la clase
# id=...  cuadro.lado=20
# id=...  cuadro.lado=11    ← siguió con el suyo (11)
```

La regla mental es una sola y es la más importante del capítulo: **al leer, un objeto primero busca el atributo en sí mismo; si no lo tiene, lo busca en la clase.** Esto se ve crudo mirando adentro. Cada objeto guarda sus atributos en un diccionario interno llamado `__dict__`:

```python
nuevo = Cuadrado()
print(nuevo.__dict__)      # {}              → el objeto no guarda nada
print(cuadrados[0].__dict__)  # {'lado': 3}  → acá está guardado el propio
```

`cuadrados[0]` tiene su `lado` en el `__dict__` (por eso ni el molde lo puede pisar); una instancia fresca tiene el `__dict__` vacío y resuelve `lado` en la clase. Bonus de este mundo libre: podés **crear atributos que no existen en el molde**, tanto en la clase como en un objeto puntual:

```python
Cuadrado.unidades = "metros"        # atributo nuevo en la clase → para todos
cuadrados[0].nombre = "Cuadradito"  # atributo nuevo SOLO en ese objeto

print(cuadrados[0].nombre)   # Cuadradito
print(cuadrados[1].nombre)   # AttributeError: 'Cuadrado' object has no attribute 'nombre'
print(Cuadrado.unidades)     # metros
print(cuadrados[1].unidades) # metros  → vendió la clase
```

Y se eliminan con la palabra que ya conocés del capítulo 3, `del`:

```python
del cuadrados[0].nombre      # borra el atributo del objeto 0
del Cuadrado.unidades        # borra el atributo de la clase (para todos)
```

Pero ojo con las reglas del aseo: **de una instancia solo podés borrar lo suyo.** Intentar `del cuadrados[1].unidades` (un atributo de clase, sin `__dict__` propio en la instancia) lanza `AttributeError`; la única forma de borrarlo es en la clase, `del Cuadrado.unidades`.

### Las funciones de la dinastía `attr`

Python trae cuatro funciones que consultan y modifican atributos *por nombre de texto* — joyas cuando el nombre viene de una variable o de un usuario:

```python
class Cuadrado:
    """Clase para probar la dinastía attr."""
    lado = 1

cuadrados = [Cuadrado(), Cuadrado()]

setattr(cuadrados[1], "lado", 15)                 # cuadrados[1].lado = 15
print(cuadrados[1].lado, cuadrados[0].lado)       # 15 1

setattr(Cuadrado, "unidad", "metros")             # agrega atributo a la clase
print(hasattr(cuadrados[0], "unidad"))            # True  → ¿existe?
print(getattr(cuadrados[0], "unidad"))            # metros → su valor

delattr(Cuadrado, "unidad")                       # borra el atributo de clase
print(hasattr(Cuadrado, "unidad"))                # False → ya no existe
print(getattr(Cuadrado, "unidad", "no tiene"))    # no tiene → default por si falta
```

La tabla de la dinastía, para memorizar de una:

| Función | Qué hace | Si el atributo no existe |
|---------|----------|--------------------------|
| `hasattr(obj, "atr")` | ¿Existe `atr`? Devuelve `True`/`False` | devuelve `False` |
| `getattr(obj, "atr")` | Devuelve el valor de `atr` | `AttributeError`, a menos que pases un tercer argumento (default) |
| `setattr(obj, "atr", valor)` | Asigna `atr = valor` (creándolo si falta) | lo crea |
| `delattr(obj, "atr")` | Elimina `atr` | `AttributeError` |

> **Para curiosear:** estas cuatro funciones, más `dir()` y `vars()`, son el material crudo de la *introspección* (sección 7): la capacidad de un programa de examinarse a sí mismo. Los frameworks modernos, de las interfaces gráficas a los validadores como pydantic (que vas a conocer tras la Parte VI), viven de eso.

---

## 5. Métodos: el comportamiento

Los **métodos** son el segundo ingrediente del objeto: no guardan solo datos, también **comportamiento**. Se definen como las funciones normales — con `def`, los parámetros que quieras, ámbito local — con una salvedad: viven *dentro* de la clase y su primer parámetro (convención `self`) recibe la instancia.

La síntesis del capítulo 11, ahora sobre una base firme: los objetos que usás desde el capítulo 3 son esa fusión. Mirá el `Cuadrado` completo, con su atributo y su comportamiento:

```python
class Cuadrado:
    """Cuadrado con lado y superficie calculada."""
    lado = 1

    def superficie(self):
        return self.lado ** 2

cuadro = Cuadrado()
cuadro.lado = 23
print(cuadro.superficie())   # 529
```

El método `superficie` *lee un atributo* (`self.lado`) y **devuelve** un valor calculado. Esa interacción —métodos que leen y escriben el estado del objeto— es el corazón de la POO. Y lo mismo que leen, pueden **crear**: un método puede asignar un atributo nuevo en el objeto, con `self.<nombre> = <valor>`. Esa creación pasa **cuando el método se ejecuta**, no antes:

```python
class Cuadrado:
    """Cuadrado que además guarda la superficie como atributo."""
    lado = 1

    def sup(self):                      # sup: guarda y devuelve la superficie
        self.superficie = self.lado ** 2
        return self.superficie

cuadro = Cuadrado()
# print(cuadro.superficie)  → AttributeError: aún no existe
cuadro.lado = 15
print(cuadro.sup())          # 225 → crea el atributo superficie
print(cuadro.superficie)     # 225 → ahora sí está guardado
```

Antes de llamar a `sup()`, `cuadro.superficie` no existe (`AttributeError`); después de llamarlo, quedó guardado en el `__dict__` de la instancia. Es el mismo comportamiento de "atributos que nacen en caliente" de la sección 4, con la diferencia de que acá es el propio objeto quien se los crea.

### Métodos que llaman métodos y los tres ámbitos

Los métodos pueden llamarse entre sí dentro de la clase (con `self.metodo(...)`), y cada método tiene **tres ámbitos de lectura**: el local (sus variables), el global (lo que importaste a nivel módulo) y el del objeto (lo que ve a través de `self`). El clásico del curso lo muestra con un `Circulo`:

```python
from math import pi

class Circulo:
    """Círculo que calcula superficie y volumen de extrusión."""
    radio = 1

    def sup(self):
        self.superficie = pi * self.radio ** 2    # global (pi) + objeto (radio)
        return self.superficie

    def volumen_extrusion(self, alto):
        return self.sup() * alto                  # llame a otro método del objeto

rueda = Circulo()
rueda.radio = 15
print(rueda.volumen_extrusion(12))   # 8482.300164692441
print(rueda.superficie)              # 706.8583470577034
```

Desglosado: `sup` lee `pi`, que viene del **ámbito global** (el módulo importó `from math import pi`), y `self.radio`, del **ámbito del objeto**. Guarda el resultado en el atributo `self.superficie`. El método `volumen_extrusion` llama a `sup()` con `self.sup()` — como `self.superficie`, pero para invocar. Y fijate: `alto` es un parámetro, no un atributo — `rueda.alto` no quedó guardado en el objeto; para eso habría que asignarlo, `self.alto = alto`, y todavía no lo hicimos.

> **Dato clave — `self` no es opcional:** la diferencia entre "atributo de instancia" y "variable local" es exactamente el `self`. `self.radio` es del objeto (perdura, y cada instancia tiene el suyo); `alto` es del método (desaparece al volver). El que venga del capítulo 8 ya conoce este baile: `self` es el puente hacia el *ámbito del objeto*, uno más de la lista local → global → (nuevo) objeto.

Para cerrar con la imagen que abre el capítulo — recordá que `objeto.metodo(arg)` es azúcar de `Clase.metodo(objeto, arg)`:

```python
print("hola".upper())          # HOLA
print(str.upper("hola"))       # HOLA  → self es solo un parámetro posicional
```

Las van a usar igual; pero saber que no hay magia es el primer paso para entender los decoradores de métodos (capítulo 14) y los *method descriptors* (capítulo 17).

---

## 6. Clases internas: cajas dentro de cajas

Como todo es un objeto y una clase es un objeto más, una clase puede vivir **dentro de otra clase**: se llaman **clases internas** (o anidadas). Sirven, sobre todo, para *organizar*: cuando una clase es tan específica que solo tiene sentido dentro de la otra, la declarás adentro, y su nombre queda colgado del molde padre como un atributo más.

El ejemplo del curso es un `Estudiante` que tiene una biblioteca de libros, y cada libro es un tipo propio definido adentro:

```python
class Estudiante:
    """Un estudiante con su biblioteca de libros."""

    def __init__(self, nombre: str, carrera: str):
        self.nombre = nombre
        self.carrera = carrera
        self.libros = []

    class Libro:
        """Un libro de la biblioteca del estudiante."""

        def __init__(self, nombre: str, autor: str):
            self.nombre = nombre
            self.autor = autor

        def get_autor(self):
            return self.autor

    def nuevo_biblio(self, libro):
        if type(libro) == Estudiante.Libro:
            self.libros.append(libro)
            return "listo"
        return "no es un libro valido"
```

Para referirte al molde interno usás la *notación de puntos* completa: `Estudiante.Libro`. Para instanciar un libro, dos caminos equivalentes —desde la clase o desde cualquier estudiante:

```python
libro_puy = Estudiante.Libro("El principito", "Saint-Exupéry")
print(type(libro_puy))   # <class '__main__.Estudiante.Libro'>

emi = Estudiante("Emiliano", "Datos")
python = emi.Libro("Python para todos", "Chiquete")   # también se puede
print(type(python))      # <class '__main__.Estudiante.Libro'>

emi.nuevo_biblio(libro_puy)
emi.nuevo_biblio(python)
print(emi.libros)        # [<Estudiante.Libro>, <Estudiante.Libro>]

for libro in emi.libros:
    print(libro.autor)   # Saint-Exupéry / Chiquete
```

Tres detalles para no tropezar:

- **El molde interno no ve a la instancia externa.** No hay magia de closures como en las funciones anidadas del capítulo 8: un `Libro` no accede por arte de magia al `Estudiante` que lo crea. Son dos nombres, uno dentro del otro — organiza el nombre, no la memoria.
- **El `type` con el camino completo** valida: `type(libro) == Estudiante.Libro`. Es una verificación decente, aunque su prima más noble ya la conocés: `isinstance` (la viste en el capítulo 3, en el `case Tipo(variable)` del capítulo 5 y en las guardas del 10). La diferencia fina entre ambas la cerramos en la sección 8.
- El atributo `libros` nace en `__init__` como `[]` — nunca lo declares como atributo de clase (sección 4 te enseñó el peligro: sería compartido por todos los estudiantes, como el `def f(x=[])` del capítulo 7).

> **Buenas prácticas:** usá clases internas cuando el tipo de adentro sea subordinado y específico del de afuera (un `Libro` de un `Estudiante`, un `Nodo` de un `Arbol`). Si el tipo interno va a ser usado en otro lado, subilo al módulo: la anidación es organización, no obligación.

---

## 7. Monkey patching e introspección

Llegamos a dos de las rebeldías más puras de la lista. La primera, **monkey patching** ("parche de mono"): Python te deja **agregarle comportamiento a un objeto o a una clase en la marcha**, sin tocar su definición original. Empezá por el caso suave, parchando un objeto puntual con una función suelta:

```python
class Animal:
    """Un animal genérico con nombre."""
    nombre = "Fido"

perros = (Animal(), Animal())
gato = Animal()

def maulla():
    print("miau")

gato.maulla = maulla       # un atributo-función solo para el gato
gato.maulla()              # miau

perros[0].maulla()         # AttributeError: los perros no maúllan
```

Fijate que acá `maulla` es una **función común** (sin `self`) colgada a un objeto: al pedir `gato.maulla()` se ejecuta tal cual. Está "viva" en el `__dict__` del gato y solo del gato. Ahora la versión con trampa — parcheando la **clase** con una función que olvida el `self`:

```python
def duerme():
    print("zzzz")

Animal.duerme = duerme        # pegado a la clase → se convierte en método
perros[1].duerme()            # TypeError: duerme() takes 0 positional arguments but 1 was given
```

¡El error es pura enseñanza! Al pegarlo a la clase, cada llamada vía instancia le mete el `self` de regalo, y la función que no lo declara explota. La cura es escribirla *como método* — con su `self`, usando el nombre del animal:

```python
def duerme(self):
    print(f"{self.nombre} zzzzz")

Animal.duerme = duerme
perros[1].nombre = "pepe"
perros[1].duerme()   # pepe zzzzz
gato.duerme()        # Fido zzzzz
```

Ahí está la otra cara de la moneda de la sección 3: `self` no es magia *porque se puede fallar sin él*. Los decoradores del capítulo 8, las closures y las funciones como valores ya te mostraron que las funciones son objetos; el monkey patching es esa misma verdad aplicada al molde: podés enchufar y desenchufar comportamiento en caliente. ¿Es prudente en un sistema productivo? Raramente — pero es el porqué de que bibliotecas como `pytest` o `unittest.mock` puedan "enmascarar" funciones sin que te enteres.

Ahora la segunda rebeldía, que ya intuías en la sección 4: la **introspección**. `dir(objeto)` te devuelve la lista de atributos y métodos que el objeto sabe manejar (sus "cuernos").

```python
print(dir(gato))
# ['__class__', '__delattr__', '__dict__', ..., 'duerme', 'maulla', 'nombre']

publico = [x for x in dir(gato) if not x.startswith("__")]
print(publico)   # ['duerme', 'maulla', 'nombre']
```

Y `help(objeto)` te muestra, formateada, la documentación de ese objeto (sus docstrings). La combinación con `__doc__` es el mecanismo detrás del `help()` de `str.find`:

```python
print(str.find.__doc__)
# Return the lowest index in the string where substring sub is found,
# such that sub is contained within a slice of the object...
```

Un ejemplo la clase `Emi` que lo demuestra de la forma más simple:

```python
class Emi:
    """Clase de ejemplo: un nombre a la vista."""
    nombre = "emiliano"

print(Emi.nombre)   # emiliano
print(dir(Emi))     # todas las capacidades del molde
```

> **Para curiosear:** `dir()` y `help()` son el *ojo* de Python: con ellos descubriste métodos antes de conocerlos (seguro lo hiciste con una `str` o una `list`). `vars(objeto)` es primo de `dir()` y te devuelve literalmente el `__dict__`. En la sección 4 ya viste la síntesis: `dir` hasta el fondo es introspección, el programa mirándose a sí mismo.

---

## 8. `isinstance`: ¿este objeto es de este molde?

En la sección 1 viste que tipos y clases son sinónimos. Ahora la pregunta de control: **¿cómo pregunto a qué clase pertenece un objeto?** La función es `isinstance(objeto, clase)`, que responde `True` si el objeto es una instancia de esa clase (o de alguna descendiente):

```python
class Clase:
    """Molde de ejemplo."""
    pass

objeto = Clase()
print(isinstance(objeto, Clase))   # True
print(isinstance(3, int))          # True   → 3 es un objeto de la clase int
print(isinstance("3", int))        # False  → el texto no es un número
```

Compañera inseparable: `issubclass(clase_hija, clase_padre)` pregunta si una *clase* desciende de otra:

```python
print(issubclass(Clase, object))   # True  → todo desciende de object
print(isinstance(Clase, object))   # True  → hasta las clases son objetos
```

Y dos verdades del rebelde para el cierre del capítulo. La primera: existen jerarquías que quizá no conocías de nombre, y `isinstance` te las revela:

```python
print(isinstance(True, int))    # True  → bool es una subclase de int
print(isinstance(True, object)) # True  → y claro, todo baja de object
print(isinstance(False, bool))  # True
```

`True` es un `bool`, y `bool` desciende de `int` — por eso `True + True` te dio `2` en el capítulo 3 sin que nadie te lo explicara. La segunda verdad es el gran spoiler de la POO libre: `isinstance` existe y es útil (validar entradas, ramificar por tipo), pero el rebelde Python *prefiere* no preguntar *qué eres* sino *qué sabés hacer*. Esa filosofía se llama **duck typing** — "si camina como pato y suena como pato, es un pato" — y tiene su capítulo entero, el 16.

> **Dato clave:** `isinstance(obj, Clase)` pregunta "¿este objeto es de este molde o de alguno de sus descendientes?"; `type(obj) == Clase` pregunta la versión estricta, sin hijos. En la sección 6 validaste con `type(libro) == Estudiante.Libro`; de ahora en más, cuando quieras aceptar subclases también, preferí `isinstance`.

---

## 9. Resumen y conceptos clave

Este capítulo abrió la Parte VI presentándote a Python como **el rebelde de la POO**: hace orientación a objetos *por convención, no por decreto*, y te deja la responsabilidad de respetar los pilares (encapsulación, herencia, abstracción, polimorfismo). Diste nombre a la lista de rebeldías — todo es objeto, tipos son clases, sin privados, monkey patching, duck typing, herencia múltiple, sobrecarga, abstractas opcionales, mixins y tipos propios — y obtuviste el mapa de dónde se desarrolla cada una en la Parte VI. Sobre esa base conociste los **moldes**: la clase como prototipo (nombrada en CamelCase, que sin superclases declaradas desciende de `object`), el objeto como galletita, `type` como botón de identidad, y `id()` para ver que cada instancia es única. Llenaste el molde con `__init__` (el constructor, primer método especial) y aprendiste que **`self` no es magia** sino un parámetro posicional — `objeto.metodo(x)` es azúcar de `Clase.metodo(objeto, x)`; lo validaste con la `Persona` del curso y su control de rango con walrus y `try`. Después exploraste el **estado**: atributos de clase que se comparten, atributos de instancia que se crean con una asignación, la regla "primero yo, después el molde", el `__dict__` que lo guarda todo, agregar y eliminar en caliente, y la dinastía `hasattr`/`getattr`/`setattr`/`delattr`. Le sumaste **comportamiento**: métodos que leen y crean atributos (`self.<x>`), que se llaman entre sí (`self.metodo()`), y sus tres ámbitos de lectura (local, global y del objeto). Viste cómo una **clase interna** organiza tipos subordinados (`Estudiante.Libro`) sin magia de acceso. Y cerraste con las dos rebeldías puras: el **monkey patching** (parchar objetos y clases en la marcha — y el error de `self` que lo tornó didáctico) y la **introspección** (`dir()`, `help()`, `__doc__`). Finalmente `isinstance` y `issubclass` quedaron como las preguntas formales de parentesco — con la advertencia de que al rebelde le importa más lo que hacés que la etiqueta (capítulo 16).

Repasá el checklist antes de la siguiente galletita:

- [ ] Python es el **rebelde de la POO**: orientación a objetos **por convención, no por decreto** (sin `private`, sin `final`, "somos adultos").
- [ ] **Todo es un objeto**: tipos y clases también son objetos; los tipos *son* clases.
- [ ] **Clase = molde**, **objeto/instancia = galletita**; instanciar es `Clase(...)`; sin nombre, el objeto se desecha.
- [ ] Nombres de clase en **CamelCase** (PEP 8). Sin superclases declaradas, toda clase desciende de **`object`**.
- [ ] **`__init__`** es el constructor: primer método en ejecutarse; los argumentos de `Clase(...)` llegan ahí. Los métodos especiales llevan doble guion bajo.
- [ ] **`self`** es un parámetro posicional: `objeto.metodo(x)` ≡ `Clase.metodo(objeto, x)`. Es el puente al ámbito del objeto.
- [ ] **Atributos de clase** (viven en el molde) vs **atributos de instancia** (en `__dict__` del objeto). Al leer: primero el objeto, después la clase.
- [ ] El **estado** puede crecer en caliente: asignar un atributo nuevo en la clase afecta a todos; en un objeto, solo a él. `del` los elimina.
- [ ] La dinastía `attr`: `hasattr`, `getattr` (con default), `setattr`, `delattr`.
- [ ] **Métodos** = comportamiento; leen y crean atributos con `self.…`, se llaman con `self.metodo()`, y tienen tres ámbitos: local, global y del objeto.
- [ ] **Clases internas**: organización de tipos subordinados (`Estudiante.Libro`); no hay magia de acceso a la instancia externa.
- [ ] **Monkey patching**: agregar funciones a objetos o métodos a clases en la marcha; el `self` es el que separa una función suelta de un método.
- [ ] **Introspección**: `dir(objeto)`, `help(objeto)`, `objeto.__doc__`, `__dict__`.
- [ ] **`isinstance(obj, Clase)`** (acepta descendientes) vs `type(obj) == Clase` (estricto); `issubclass`. `bool` es subclase de `int`.

---

## 10. Ejercicios

1. **Molde mínimo**: creá la clase `Robot` con atributo de clase `nombre = "sin nombre"`. Instanciá tres robots en una lista, cambiá el `nombre` a uno solo, agregale a la *clase* el atributo `energia = 100`, poné a otro robot `energia = 50`, y después eliminá `energia` de la clase. Explicá con tus palabras el estado final de cada robot usando `__dict__`.
2. **Constructor validado**: escribí la clase `Cuenta` con `__init__(self, saldo: float)` que valide el saldo como la `Persona` del capítulo (con `int()` o `float()` y `try`/`except`): acepta números mayores o iguales a cero, imprime un aviso y deja `self.saldo = None` si es negativo o no numérico. Probá con `100`, `-5` y `"mucho"`.
3. **Áreas y volúmenes**: creá la clase `Rectangulo` con `__init__(self, base: float, altura: float)` y el método `superficie(self)` que devuelve `base * altura`. Instanciá un rectángulo y calculá su superficie. Después amplialo: un método `almacenar_superficie(self)` que *guarde* la superficie en el atributo `self.superficie` (como `sup()` del Circulo); demostrá que el atributo no existe antes de llamarlo y sí después.
4. **La biblioteca del estudiante**: copiá la clase `Estudiante.Libro` del capítulo y agregale un método `ficha(self)` que devuelva `"<nombre> — <autor>"`. Cargá dos libros con `nuevo_biblio` y recorré `emi.libros` imprimiendo sus fichas.
5. **Monkey patch a medida**: `class Gato:` con atributo de clase `nombre = "Michi"`. Adjuntale a *un* gato una función `saludar()` sin `self` (imprime `"hola gato"`), y a la clase una función `comer(self)` que imprima `"{self.nombre} come pescado"`. Explicá con palabras por qué una necesita `self` y la otra no.
6. **El plan de investigación**: con `dir()` y `help()`, investigá el objeto `"hola"` (una `str`): listá cinco métodos públicos (sin `__`…), y mirá el `__doc__` de `.find` explicando qué hace.
7. **`isinstance` detective**: para la lista `[3, 3.0, "3", True, [3]]`, decí cuáles son instancias de `int`, cuáles de `float`, y explicá el caso del `True`. Después verificá que todas son instancias de `object`.
8. **El molde con pasajeros**: creá la clase `Vuelo` con `__init__(self, numero: str, ocupantes: int)` que valide `0 <= ocupantes <= 300` (estilo `Persona`). Instanciá un vuelo válido y dos inválidos (uno negativo, otro `"miles"`) y mostrá cómo quedó cada `ocupantes`.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. Molde mínimo
class Robot:
    """Un robot genérico."""
    nombre = "sin nombre"

robots = [Robot(), Robot(), Robot()]
robots[0].nombre = "R2"
Robot.energia = 100            # atributo de clase → para todos
robots[1].energia = 50         # atributo propio solo de robots[1]

print(robots[0].__dict__)      # {'nombre': 'R2'}
print(robots[1].__dict__)      # {'energia': 50}
print(robots[2].__dict__)      # {}

print(robots[0].energia)       # 100  → no tiene el suyo, lo ve en la clase
print(robots[1].energia)       # 50   → tiene el suyo
print(robots[2].nombre)        # sin nombre

del Robot.energia              # lo eliminamos de la clase
# print(robots[0].energia)     → AttributeError: ya no queda dónde mirar

# 2. Constructor validado
class Cuenta:
    """Cuenta bancaria con saldo validado."""

    def __init__(self, saldo: float):
        try:
            if (saldo := float(saldo)) >= 0:
                self.saldo = saldo
            else:
                print(saldo, "saldo negativo no permitido")
                self.saldo = None
        except ValueError:
            print(saldo, "no es un numero valido")
            self.saldo = None

cargas = Cuenta(100)
print(cargas.saldo)     # 100.0
Cuenta(-5)              # -5.0 saldo negativo no permitido
Cuenta("mucho")         # mucho no es un numero valido

# 3. Áreas y volúmenes
class Rectangulo:
    """Rectángulo con base y altura."""

    def __init__(self, base: float, altura: float):
        self.base = base
        self.altura = altura

    def superficie(self):
        return self.base * self.altura

    def almacenar_superficie(self):
        self.superficie = self.base * self.altura
        return self.superficie

habitacion = Rectangulo(4.5, 3.0)
print(habitacion.superficie())      # 13.5
# print(habitacion.superficie)      → AttributeError: aún no existe
print(habitacion.almacenar_superficie())  # 13.5
print(habitacion.superficie)              # 13.5 → ahora sí existe

# 4. La biblioteca del estudiante
class Estudiante:
    """Estudiante con biblioteca."""

    def __init__(self, nombre: str, carrera: str):
        self.nombre = nombre
        self.carrera = carrera
        self.libros = []

    class Libro:
        """Un libro de la biblioteca."""

        def __init__(self, nombre: str, autor: str):
            self.nombre = nombre
            self.autor = autor

        def ficha(self):
            return f"{self.nombre} — {self.autor}"

    def nuevo_biblio(self, libro):
        if type(libro) == Estudiante.Libro:
            self.libros.append(libro)
            return "listo"
        return "no es un libro valido"

emi = Estudiante("Emiliano", "Datos")
emi.nuevo_biblio(Estudiante.Libro("El principito", "Saint-Exupéry"))
emi.nuevo_biblio(Estudiante.Libro("Python para todos", "Chiquete"))
for libro in emi.libros:
    print(libro.ficha())
# El principito — Saint-Exupéry
# Python para todos — Chiquete

# 5. Monkey patch a medida
class Gato:
    """Un gato genérico."""
    nombre = "Michi"

def saludar():                      # sin self: va a un objeto puntual
    print("hola gato")

def comer(self):                    # con self: va a la clase
    print(f"{self.nombre} come pescado")

michi = Gato()
michi.saludar = saludar             # atributo-función solo de michi
michi.saludar()                     # hola gato

Gato.comer = comer                  # método para toda la clase
michi.comer()                       # Michi come pescado
Gato.comer(michi)                   # Michi come pescado (sugar explícito)

# 6. El plan de investigación
print([x for x in dir("hola") if not x.startswith("__")])
# ['capitalize', 'count', 'endswith', 'find', 'format', ...]
print(str.find.__doc__)

# 7. isinstance detective
datos = [3, 3.0, "3", True, [3]]
print([isinstance(x, int) for x in datos])     # [True, False, False, True, False]
print([isinstance(x, float) for x in datos])   # [False, True, False, False, False]
print(isinstance(True, int))                   # True → bool desciende de int
print([isinstance(x, object) for x in datos])  # [True, ...] todas

# 8. El molde con pasajeros
class Vuelo:
    """Vuelo con ocupación validada."""

    def __init__(self, numero: str, ocupantes: int):
        self.numero = numero
        try:
            if 0 <= (ocupantes := int(ocupantes)) <= 300:
                self.ocupantes = ocupantes
            else:
                print(ocupantes, "fuera de rango (0-300)")
                self.ocupantes = None
        except ValueError:
            print(ocupantes, "no es un numero de pasajeros valido")
            self.ocupantes = None

f1 = Vuelo("AR1200", 180)
print(f1.ocupantes)     # 180
Vuelo("AR1300", -4)     # -4 fuera de rango (0-300)
Vuelo("AR1400", "miles")# miles no es un numero de pasajeros valido
```

Con esto hiciste entrar en calor: sabés crear moldes, llenarlos con estado, enseñarles comportamiento, incorporarles piezas en caliente y preguntarles su parentesco. Pero hay una pregunta que quedó flotando desde la sección 2: ¿dónde están todas esas capacidades *heredadas* que tu clase tiene sin que las programes — los `__eq__`, `__str__`, `__len__`? Ellas explican por qué `print(objeto)` muestra un galimatías de direcciones de memoria y cómo podés hacer que tus objetos se comporten como "ciudadanos de primera" del lenguaje.

Ese es el espejo en el que Python se mira: los **métodos especiales** (los dunders). En el próximo capítulo los vas a invocar indirectamente millones de veces y, por primera vez, vas a escribir los tuyos — con una pieza que ya conocés de memoria y que te toca de cerca: vas a fabricarte un `rango` propio. Nos vemos en el capítulo 13.