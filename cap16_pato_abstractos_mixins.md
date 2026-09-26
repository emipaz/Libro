# Capítulo 16 — El "es un pato": duck typing, clases abstractas y mixins

El capítulo 15 terminó con dos preguntas incómodas. Primera: en la nómina, `Empleado(3, "Ernesto")` imprimía `No implementado` — un texto comodín que nadie cobra. Si el contrato de la base es tan importante, ¿cómo hacés para que el molde **obligue** en vez de dejar pasar? Segunda: `Nomina` aceptó al `Externo` que no hereda nada y solo tiene una función con la misma firma. Si cualquiera entra sin declarar parentesco, ¿para qué declara la familia en primer lugar?

Las dos preguntas son la misma crisis creativa de la POO, y en este capítulo vas a ver al rebelde Python hacerse cargo con tres herramientas:

- **Duck typing**: el comportamiento manda, la clase no importa ("si camina como pato y suena como pato, para mí es un pato").
- **Clases base abstractas** (`ABC` y `@abstractmethod`): el molde que *obliga* a implementar, con TypeError si no cumplís en el momento de crear.
- **Mixins**: comportamientos sueltos que se ensamblan por herencia múltiple, como las piezas de un robot.

Y de regalo, un cuarto juguete que cierra la promesa del capítulo 9: **`Protocol`**, el tipo del pato para el *type checker* (el verificador de tipos) — "esto cumple con este contrato sin heredar de nadie", justo lo que quedó pendiente desde los type hints.

---

## 1. La ley del pato: el comportamiento manda

Empezá por el ejemplo que ya asomó en el ejercicio 8 del capítulo 15. Seis clases que no se conocen entre sí — no heredan de ninguna base común — pero todas responden a un mismo llamado: `moverse()`:

```python
class Peaton:
    """Camina a pie."""

    def moverse(self):
        return "caminando"


class Runner:
    """Corre, no camina."""

    def moverse(self):
        return "corriendo"


class Patinador:
    """Anda en patineta."""

    def moverse(self):
        return "patinando"


class Ciclista:
    """Pedalea."""

    def moverse(self):
        return "pedaleando"


class Automovilista:
    """Va en auto."""

    def moverse(self):
        return "en auto"


class Campesino:
    """Va montado a caballo."""

    def moverse(self):
        return "montando a caballo"
```

Nada de `class Peaton(EsCapazDeMoverse)`. Ahora una `Persona` que recibe una de estas formas de moverse — y un método `describir()` que las aprovecha:

```python
class Persona:
    """Una persona con un nombre y una forma de moverse."""

    def __init__(self, nombre, accion):
        self.nombre = nombre
        self.accion = accion

    def describir(self):
        return f"{self.nombre} está {self.accion.moverse()}"


personas = [Persona("luca", Runner()),
            Persona("luca", Peaton()),
            Persona("luca", Campesino()),
            Persona("luca", Automovilista()),
            Persona("luca", Patinador()),
            Persona("luca", Ciclista())]

for i in personas:
    print(i.describir())
# luca está corriendo
# luca está caminando
# luca está montando a caballo
# luca está en auto
# luca está patinando
# luca está pedaleando
```

Mirá lo que NO hace `describir()`: nunca le pregunta a `accion` quién sos. No hay `type(accion)` ni `isinstance(accion, Peaton)` ni una cadena de `elif`. Solo llama `.moverse()`. Si el objeto tiene el método, funciona; si no lo tiene, explota con un `AttributeError` claro. El concepto, en una frase: *cualquier clase que tenga una interfaz compatible puede interactuar con cualquier otra clase*. Eso es el **duck typing**.

La frase célebre (atribuida a un proverbio inglés, y que da nombre a la técnica): **"si camina como pato y suena como pato, entonces es un pato."** No importa el DNI de la clase; importa lo que el objeto *hace*.

> **Dato clave:** el duck typing es el reverso exacto de la herencia. La herencia dice "declarás tu familia y por eso tenés estos métodos". El duck typing dice "tenés estos métodos y por eso entrás al club, familia o no". Es la misma filosofía del capítulo 14: en Python no hay candados, hay comportamientos — y "somos adultos" confía en que el programa hable por interfaces.

---

## 2. El método fantasma: la interfaz por convención

Duck typing suena a fiesta total, pero tiene un lado oscuro. Volvé al sistema de energía del capítulo 14, pero con una variante que lo delata: ahora las **bases** sí existen, y son las que crean el problema:

```python
class EnergiaSolar:
    """Base de las que producen energía con luz solar."""

    fuente = "luz solar"

    def energia(self):
        pass


class EnergiaDinamica:
    """Base de las que producen energía con movimiento mecánico."""

    fuente = "movimiento mecánico"

    def energia(self):
        pass


class Fotovoltaica(EnergiaSolar):
    """Paneles: luz solar convertida en watts."""

    rendimiento = 500

    def __init__(self, lumenes):
        self.lumenes = lumenes

    def energia(self):
        return (self.lumenes * self.rendimiento, "watts/hr")


class Hidroelectrica(EnergiaDinamica):
    """Represa: movimiento convertido en watts."""

    rendimiento = 2000

    def __init__(self, litros):
        self.litros = litros

    def energia(self):
        return (self.litros * self.rendimiento, "watts/hr")
```

Las bases `EnergiaSolar` y `EnergiaDinamica` declaran `energia()` con el cuerpo `pass`. Fijate qué es: **una interfaz que no hace nada** — el método fantasma. "Prometo que todo lo solar tiene `energia()`", pero la promesa se cumple con silencio. Si preguntás `EnergiaSolar().energia()`, te devuelve `None` sin ninguna explosión; hay que ver el resultado esperando el `(valor, "watts/hr")` del contrato y... nada. Es el mismo comodín `No implementado` del `Empleado` del capítulo 15, ahora devuelto en `None`.

Ahora sí, el cliente. La tostadora es el `Dispositivo` del capítulo 14 — la misma validación estricta con `"watts/hr"`:

```python
class Dispositivo:
    """El 'Dispositivo' del capítulo 14: regala horas de uso."""

    def __init__(self, consumo=50):
        self.consumo = float(consumo)

    def duracion(self, energia):
        """Horas que dura con una energía dada en (valor, 'watts/hr')."""
        if type(energia) in (tuple, list) and len(energia) == 2 \
                and type(energia[0]) in (int, float) \
                and energia[1].casefold() == "watts/hr":
            return energia[0] / self.consumo
        raise ValueError("Interfaz incorrecta.")


tostadora = Dispositivo(20)
print(tostadora.duracion(Fotovoltaica(500).energia()))   # 12500.0  → 500 × 500 / 20
print(tostadora.duracion(Hidroelectrica(500).energia())) # 50000.0  → 500 × 2000 / 20
```

La tostadora usa duck typing puro: recibe lo que le traigan y solo valida la *forma* de los datos (`duracion` no le pregunta a la fuente de energía qué clase es). Y el sistema anda. El problema no está en el cliente, está en el contrato: la base **declara** `energia()`, pero si una subclase la olvida, nadie se entera hasta el `None` en la cara. La promesa del capítulo 15 se vuelve concreta:

> Cuando el contrato es tan importante que incumplirlo no puede ser silencio ni `None`, la interfaz por convención no alcanza. Ahí entra la **clase base abstracta**.

---

## 3. El molde que obliga: `ABC` y `@abstractmethod`

Python no trae "clase abstracta" de fábrica — el rebelde no decreta. Pero el módulo `abc` (de *abstract base classes*) del propio lenguaje te da las piezas para construir el molde que obliga: la clase hereda de `ABC` y marca cada método "de promesa" con `@abstractmethod`.

Resucitá la nómina del capítulo 15, pero ahora la base no puede mentir:

```python
from abc import ABC, abstractmethod


class Empleado(ABC):
    """El molde que OBLIGA: todo empleado debe saber calcular su paga."""

    def __init__(self, id, nombre):
        self.id = id
        self.nombre = nombre

    @abstractmethod
    def calculo(self):
        """Cuánto cobra este empleado."""


class EmpleadoMensual(Empleado):
    """Sueldo fijo."""

    def __init__(self, id, nombre, salario):
        super().__init__(id, nombre)
        self.salario = salario

    def calculo(self):
        return self.salario


class EmpleadoPorHoras(Empleado):
    """Paga por horas trabajadas."""

    def __init__(self, id, nombre, horas, valor_hora):
        super().__init__(id, nombre)
        self.horas = horas
        self.valor_hora = valor_hora

    def calculo(self):
        return self.horas * self.valor_hora
```

Tres lecturas nuevas:

- **`class Empleado(ABC)`**: ya no es una clase común, es un molde incompleto por diseño. No se puede instanciar.
- **`@abstractmethod`** encima de `calculo()`: marca el método declarado pero sin implementar. La subclase *debe* llenarlo o seguirá siendo abstracta para siempre.
- Ahora el comodín `No implementado` del capítulo 15 **no puede ocurrir**: el golpe de queda pasa en el momento de *construir*.

Mirá cómo se comporta la clase base en el momento de la creación:

```python
try:
    fantasma = Empleado(9, "Fantasma")
except TypeError as error:
    print(type(error).__name__, "-", error)
# TypeError - Can't instantiate abstract class Empleado with abstract method calculo
```

Y una subclase que "se olvida" de implementar tampoco se puede crear:

```python
class Becario(Empleado):
    """Pasa, pero todavía no sabe calcular su paga."""

    pass


try:
    b = Becario(4, "Brenda")
except TypeError as error:
    print(type(error).__name__, "-", error)
# TypeError - Can't instantiate abstract class Becario with abstract method calculo
```

El `TypeError` dice exactamente *qué* falta. Es el guardia de la interfaz que el capítulo 14 dejó como promesa ("interfaz rota = excepción clara"), ahora movido a la propia definición del molde. La nómina quedó blindada:

```python
class Nomina:
    """Composición: contiene empleados y les pregunta su cálculo."""

    def mostrar(self, empleados):
        for emp in empleados:
            print(f"{emp.id} - {emp.nombre}: {emp.calculo()}")


nomina = Nomina()
nomina.mostrar([EmpleadoMensual(1, "Ana", 45000),
                EmpleadoPorHoras(2, "Luis", 40, 500)])
# 1 - Ana: 45000
# 2 - Luis: 20000
```

Si intentás meter al Ernesto otra vez... no alcanza ni a llegar a la nómina: `Empleado(3, "Ernesto")` explota en la misma línea del constructor, antes de la lista.

> **Dato clave:** la clase abstracta es la "interfaz que obliga". Declarás qué hay que hacer (métodos abstractos, sin cuerpo) y qué ya está hecho (métodos concretos, con cuerpo). En Java o C# la palabra clave es `abstract`/`interface`; en Python, el rebelde, se arma con `ABC` + `@abstractmethod` — igual de potente, pero por convención y decoradores.

---

## 4. El molde abstracto con cuerpo: abstracto y concreto a la vez

La clase abstracta no es solo una lista de promesas: puede mezclar métodos abstractos (los que obliga) con métodos concretos (los que comparte). Y pueden existir jerarquías enteras donde cada piso cumple de a poco. La familia `Animal` del capítulo 15 sirve para verlo en grande, ahora como molde que obliga:

```python
class Animal(ABC):
    """Clase base de todos los animales."""

    def __init__(self, nombre):
        self.nombre = nombre
        print(f"Hola. Mi nombre es {self.nombre}.")

    @abstractmethod
    def reproduccion(self):
        """Cómo cría este animal."""

    @abstractmethod
    def alimentacion(self):
        """Qué come este animal."""

    def presentarse(self):
        """Método concreto: sirve para toda la familia, sin repetirlo."""

        return f"Soy {self.nombre}."

    def __del__(self):
        print(f"El animal {self.nombre} acaba de fallecer.")
```

`Animal` tiene un `__init__` que asigna el nombre e imprime el saludo (herencia del capítulo 15), un `presentarse()` concreto que toda la familia hereda sin repetir, y dos métodos abstractos que son las dos promesas: cada animal *debe* saber reproducirse y alimentarse.

Ahora el piso intermedio. `Mamifero` implementa ÚNICAMENTE la reproducción:

```python
class Mamifero(Animal):
    """Implementa la reproducción... pero se olvida de la alimentación."""

    def reproduccion(self):
        print("Toma un cachorro.")

    def amamanta(self):
        print("Toma un vaso de leche.")


try:
    animalito = Mamifero("Cosa")
except TypeError as error:
    print(type(error).__name__, "-", error)
# TypeError - Can't instantiate abstract class Mamifero with abstract methods alimentacion
```

El mensaje es quirúrgico: falta `alimentacion`. `Mamifero` sigue siendo abstracto — no se puede crear, y el error te dice el método exacto que olvidaste. Falta el último eslabón:

```python
class Perro(Mamifero):
    """Completa lo que faltaba: la alimentación."""

    def alimentacion(self):
        print("Deme dos tacos de pastor sin tortillas ni verdura.")
```

Y recién ahora, con las dos promesas cumplidas a lo largo de la cadena `Perro → Mamifero → Animal`, el molde deja nacer:

```python
beagle = Perro("Snoopy")            # Hola. Mi nombre es Snoopy.
beagle.alimentacion()               # Deme dos tacos de pastor sin tortillas ni verdura.
print(beagle.presentarse())         # Soy Snoopy.   ← método concreto heredado
del beagle                          # El animal Snoopy acaba de fallecer.
```

Dos ideas para llevarte:

- **La abstracción se completa de a poco.** Cada subclase cumple las promesas que puede; recién cuando la última promesa se cumple, la clase se vuelve instanciable. Es un camino sin vuelta atrás: nunca "casi abstracto".
- **Lo concreto se hereda sin repetir.** `presentarse()` y el `__init__` son la parte del molde que *sí* se reutiliza — la interfaz obliga (abstracto) y la implementación comparte (concreto). Eso es un patrón de diseño conocido como "plantilla" (template method): el esqueleto vive en la base, los detalles en cada subclase.

---

## 5. `collections.abc`: los protocolos de la biblioteca estándar

Ya viste `Iterable`, `Callable` y `Mapping` como *tipos* en el capítulo 9. Ahora descubrí que son **clases abstractas** de verdad, y que la biblioteca estándar (módulo `collections.abc`) las usa para preguntas estructurales del tipo "¿este objeto es recorrible?":

```python
from collections.abc import Iterable, Sized, Container, Callable, Reversible, Sequence

print(isinstance([1, 2, 3], Iterable))    # True  → list es Iterable
print(isinstance("hola", Sized))          # True  → str tiene __len__
print(isinstance({"a": 1}, Container))    # True  → dict sabe responder al "in"
print(isinstance(len, Callable))          # True  → len() es llamable
print(isinstance("hola", Sequence))       # True  → str es una secuencia
```

Y la parte linda: varias de estas ABCs responden **estructuralmente**. No necesitás heredar de `Iterable` para que `isinstance` diga que sos recorrible; alcanza con que la clase tenga `__iter__`, que ya conocés del capítulo 5 y del capítulo 13. La ABC tiene un enganche interno (`__subclasshook__`) que le pregunta a la clase "¿tenés el método?" y responde `True` si existe:

```python
class Caja:
    """Una caja de cosas: su único superpoder es `__iter__`."""

    def __init__(self, *cosas):
        self.cosas = cosas

    def __iter__(self):
        return iter(self.cosas)


caja = Caja("moneda", "goma", "dado")
print(isinstance(caja, Iterable))     # True  → no heredó nada, no registró nada
print(issubclass(Caja, Iterable))     # True  → el "es un" estructural
print(list(caja))                     # ['moneda', 'goma', 'dado']
```

`isinstance(caja, Iterable)` da `True` sin que `Caja` declare un solo vínculo de herencia. Es el duck typing hecho `isinstance`.

Con un poco más de arsenal, `Iterator` y `Reversible` también tienen su pregunta estructural:

```python
class CuentaRegresiva:
    """Cuenta `n, ..., 0`, y además sabe recorrerse al revés."""

    def __init__(self, tope):
        self.tope = tope

    def __iter__(self):
        return iter(range(self.tope + 1))

    def __reversed__(self):
        return iter(range(self.tope, -1, -1))


c = CuentaRegresiva(3)
print(isinstance(c, Iterable))      # True   → tiene __iter__
print(isinstance(c, Reversible))    # True   → tiene __iter__ y __reversed__
print(isinstance(c, Sequence))      # False  → Sequence NO hace chequeo estructural en 3.10
```

Y acá hay un paseo fino: **no todas las ABCs de `collections.abc` chequean estructura.** En Python 3.10.4, `Iterable`, `Iterator`, `Reversible`, `Sized`, `Container` y `Callable` definen su propia pregunta estructural; `Sequence` y `Mapping`, en cambio, no lo hacen (sus subclases sí, por herencia, pero el chequeo "tenés `__getitem__` y `__len__`, entrás" no corre). Por eso el comentario `# False` de recién no es un capricho: es un agujero que la siguiente herramienta — `Protocol` — viene a tapar con rigor de *type checker*.

> **Dato clave:** `collections.abc` es la biblioteca de protocolos del *runtime* (tiempo de ejecución). `for` recorre tu objeto porque, en algún momento, Python pregunta "¿tenés `__iter__`?" — y las ABCs son la forma de hacer esa pregunta a mano con `isinstance`/`issubclass`, sin tocar tu código.

---

## 6. Mixins: piezas que se ensamblan

Con las clases abstractas ya tenés el molde que obliga. Pero hay comportamientos que no pegan con una jerarquía estricta — son *piezas*: se ensamblan, no se heredan de a una. Eso es el **mixin**, y no hay historia que lo cuente mejor que la del robot Voltron: cinco leones, cada uno con una parte, que juntos forman el robot completo.

> **Término:** del inglés *mix in* ("mezclar, incorporar"). Un mixin es una clase pensada para ser *combinada*: aporta métodos de un comportamiento, no suele instanciarse sola, y aprovecha la herencia múltiple del capítulo 15 para conformar clases modulares.

```python
class AutoRobot:
    """El robot que se arma con las piezas felinas."""

    def conversion(self):
        return "Soy un felino."


class LeonNegro(AutoRobot):
    """Pieza: la cabeza."""

    def opera_cabeza(self):
        return "Jeje, soy la más importante."


class LeonRojo(AutoRobot):
    """Pieza: el brazo derecho."""

    def opera_brazo_derecho(self):
        return "Operando brazo derecho."


class LeonVerde(AutoRobot):
    """Pieza: el brazo izquierdo."""

    def opera_brazo_izquierdo(self):
        return "Operando brazo izquierdo."


class LeonAzul(AutoRobot):
    """Pieza: la pierna derecha."""

    def opera_pierna_derecha(self):
        return "Operando pierna derecha."


class LeonAmarillo(AutoRobot):
    """Pieza: la pierna izquierda."""

    def opera_pierna_izquierda(self):
        return "Quería ser la cabeza."
```

Cada león es un comportamiento independiente con nombre propio. Ahora el ensamblado — la clase modular que mezcla todas las piezas en un solo orden:

```python
class Voltron(LeonNegro, LeonRojo, LeonAzul, LeonVerde, LeonAmarillo):
    """El robot completo: todas las piezas de una vez."""

    pass
```

```python
ensamblar = Voltron()
print(ensamblar.conversion())
print(ensamblar.opera_cabeza())
print(ensamblar.opera_pierna_izquierda())
print(Voltron.mro())
# Soy un felino.
# Jeje, soy la más importante.
# Quería ser la cabeza.
# [<class '__main__.Voltron'>, <class '__main__.LeonNegro'>, <class '__main__.LeonRojo'>,
#  <class '__main__.LeonAzul'>, <class '__main__.LeonVerde'>, <class '__main__.LeonAmarillo'>,
#  <class '__main__.AutoRobot'>, <class 'object'>]
```

La jugada es elegante: cada pieza aporta su parte sin pisar a las demás, y el MRO (el orden de búsqueda del capítulo 15) queda como la **lista de ensamblado** — `Voltron, LeonNegro, LeonRojo, LeonAzul, LeonVerde, LeonAmarillo, AutoRobot, object`. Si dos piezas declararan el mismo método, ganaría la que aparece más temprano en esa lista: el orden de `class Voltron(...)` decide.

Tres precisiones de ingeniería honesta:

- Los cinco leones acá son subclases de `AutoRobot`. En la vida real, un mixin puro suele *no* depender de una base: aporta comportamiento, no promesas.
- No va a haber `LeonRojo()` suelto en tu programa: la pieza existe para ser ensamblada.
- El código más limpio casi nunca es un mixin: cuando un comportamiento se repite en lugares *sin parentesco*, el mixin es la técnica de la rebeldía — pero primero agotá la composición del capítulo 15.

---

## 7. `Protocol`: el tipo del pato para el checker

El capítulo 9 dejó dos promesas con nombre y apellido: la clase `Persona` completa y la palabra `Protocol`. Las dos se cobran ahora, y van juntas porque son el mismo movimiento — **escribir el contrato sin exigir herencia**, con la diferencia de que esta vez no es solo opinión: el *type checker* la controla.

`Protocol` (de `typing`) define *qué* métodos/atributos debe tener un objeto, sin que ninguna clase tenga que heredar de él. Es duck typing convertido en tipo:

```python
from typing import Protocol


class Volador(Protocol):
    """La promesa: todo objeto que sepa `volar()`."""

    def volar(self) -> str: ...
```

El cuerpo `...` es el famoso "esto es una firma, no una implementación" (del capítulo 13). Ahora dos clases que no miran a `Volador` ni por asomo:

```python
class Pajaro:
    def volar(self) -> str:
        return "El pájaro vuela."


class Avion:
    def volar(self) -> str:
        return "El avión vuela."


def anunciar_vuelo(objeto: Volador) -> None:
    print(objeto.volar())


anunciar_vuelo(Pajaro())     # El pájaro vuela.
anunciar_vuelo(Avion())      # El avión vuela.
```

`anunciar_vuelo` declara que espera un `Volador`, y el type checker (mypy, pyright, los que viste en el capítulo 9) acepta `Pajaro` y `Avion` **sin** que hereden de nada: cumplen la estructura. Ese es el "esto cumple este contrato sin heredar de nadie" que quedó pendiente.

Ojo con un detalle que confunde a todo el mundo: en *runtime*, un `Protocol` no se cumple solo. Fijate qué pasa con `isinstance` sobre el `Protocol` pelado:

```python
try:
    print(isinstance(Pajaro(), Volador))
except TypeError as error:
    print(type(error).__name__, "-", error)
# TypeError - Instance and class checks can only be used with @runtime_checkable protocols
```

Para que el pato pase el *control de identidad* de runtime también, hay que decorar el protocolo con `@runtime_checkable` — y entonces sí, el `isinstance` chequea los métodos:

```python
from typing import runtime_checkable


@runtime_checkable
class Movible(Protocol):
    """Todo lo que se sirva para moverse."""

    def moverse(self) -> str: ...


class Tortuga:
    def moverse(self) -> str:
        return "muy despacio"


print(isinstance(Tortuga(), Movible))     # True  → el pato pasa el control
```

Y la *segunda* promesa del capítulo 9: la clase `Persona` tipada, con la función de orden superior `filtrar_mayores` que usa `Callable` (capítulos 8 y 9), cerrando así la vuelta de los type hints a las clases:

```python
from typing import Callable


class Persona:
    """Una persona, tipada de verdad."""

    def __init__(self, nombre: str, edad: int) -> None:
        self.nombre: str = nombre
        self.edad: int = edad

    def es_mayor(self) -> bool:
        return self.edad >= 18


def filtrar_mayores(personas: list[Persona], criterio: Callable[[Persona], bool]) -> list[Persona]:
    """Devuelve las personas que cumplen un criterio."""
    return [p for p in personas if criterio(p)]


personas = [Persona("Ana", 25), Persona("Luis", 16), Persona("Eva", 34)]
for p in filtrar_mayores(personas, lambda p: p.es_mayor()):
    print(f"{p.nombre} ({p.edad}) es mayor de edad")
# Ana (25) es mayor de edad
# Eva (34) es mayor de edad
```

---

## 8. Duck typing, ABC, mixin o Protocol: la decisión

Cuatro herramientas y un solo objetivo: definir interfaces a la manera del rebelde. La diferencia es *a quién le hablan* y *cuánto obligan*.

| Herramienta | ¿Qué es? | ¿Obliga? | ¿Para qué la uso? |
| --- | --- | --- | --- |
| **Duck typing** | Confiar en el comportamiento, no decir nada | No — si el método no está, `AttributeError` | El 90% de las funciones: "me importa que tengas `.moverse()`". |
| **Interfaz por convención** (`pass`) | Un método declarado que no hace nada | No — silencio y `None` | Para documentar (y ojalá no te olviden implementar). |
| **`ABC` + `@abstractmethod`** | Molde que no puede nacer incompleto | Sí — `TypeError` al instanciar | Cuando el contrato es tan central que romperlo en silencio sería un bug (la nómina, el animal). |
| **Mixins** | Piezas de comportamiento ensamblables | Aportan, no exigen | Comportamientos que se repiten sin parentesco; módulos de "poderes". |
| **`collections.abc`** | Protocolos del runtime | Chequean estructura por `isinstance` | Preguntar "¿es recorrible?" sin tocar tu código. |
| **`Protocol`** | El contrato para el type checker | Chequea el checker (y el runtime con `@runtime_checkable`) | "Esto cumple con este contrato sin heredar de nadie" — la promesa del capítulo 9. |

Las reglas de la casa, para que la mesa no se desborde:

1. **El duck typing es el default.** Si te alcanza con llamar un método, llamalo y listo. No declares jerarquías para un solo método.
2. **La clase abstracta es para contratos críticos.** Cuando "olvidarse" no puede ser silencio, `@abstractmethod` convierte el olvido en `TypeError` en el momento exacto de crear.
3. **El mixin es para comportamientos que se repiten sin parentesco.** Si se repiten y hay parentesco, es herencia (capítulo 15) o composición (también capítulo 15).
4. **`Protocol` es para los tipos que los humanos y el checker ven.** El runtime no lo exige; la disciplina, sí.
5. **Nada de esto reemplaza el juicio.** El rebelde te suelta las cuatro herramientas y confía: la elección correcta es la que expresa mejor *tu* dominio.

---

## 9. Resumen y conceptos clave

Este capítulo cerró el trío de la POO rebelde. Empezaste con la **ley del pato**: el comportamiento manda sobre la clase, y `Persona` le habló a `Runner`, `Ciclista`, `Campesino` y compañía sin preguntarles el DNI — duck typing. Después viste la cara oscura de la interfaz por convención: el **método fantasma** (`pass` en la base) que deja `None` en silencio, como el `No implementado` del capítulo 15. Y entonces llegó el molde que obliga: **`ABC` + `@abstractmethod`**, que convierte el olvido en `TypeError` en el momento de instanciar — y que admite jerarquías que se completan de a poco (Animal → Mamifero → Perro) mezclando métodos abstractos con concretos. Descubriste que `collections.abc` es la librería de protocolos del runtime, con chequeos estructurales verdaderos para `Iterable`, `Iterator`, `Reversible`, `Sized`, `Container` y `Callable` — y con el agujero honesto de `Sequence`/`Mapping` en 3.10. Viste los **mixins** como piezas que se ensamblan (Voltron y sus cinco leones), y cerraste cobrando la deuda del capítulo 9: **`Protocol`**, el tipo del pato para el type checker, con `Persona` tipada y `filtrar_mayores`.

Repasá el checklist antes de seguir:

- [ ] **Duck typing**: "si camina como pato y suena como pato, para mí es un pato" — no importa la clase, importa que el método exista.
- [ ] **Interfaz por convención** (`pass` en la base): documenta pero no obliga; el método fantasma devuelve `None` en silencio.
- [ ] **`class X(ABC)`**: la clase no se puede instanciar mientras tenga métodos abstractos sin implementar.
- [ ] **`@abstractmethod`**: marca el método-promesa; `TypeError` al instanciar si está pendiente, con el nombre del método en el mensaje.
- [ ] Las subclases implementan de a poco: cada piso cumple lo que puede, y recién con todas las promesas la clase se vuelve instanciable.
- [ ] Un ABC puede mezclar **abstracto y concreto**: lo abstracto obliga, lo concreto se hereda sin repetir (patrón plantilla).
- [ ] **`collections.abc`**: protocolos del runtime; `Iterable`/`Iterator`/`Reversible`/`Sized`/`Container`/`Callable` chequean estructura por `isinstance` sin herencia.
- [ ] En 3.10, `Sequence`/`Mapping` **no** chequean estructura por `isinstance` — por eso ahí gana `Protocol`.
- [ ] **Mixins**: clases pensadas para ensamblar por herencia múltiple; cada una aporta un comportamiento; el orden de `class ...(...)` define el MRO.
- [ ] **`Protocol`** (de `typing`): el contrato estructural para el type checker; `@runtime_checkable` habilita `isinstance` en runtime.
- [ ] La elección: duck typing de default, ABC para contratos críticos, mixin para comportamientos sin parentesco, Protocol para la disciplina del tipo — y el juicio de "somos adultos" por encima de todo.

---

## 10. Ejercicios

1. **El coro del pato**: clases `Pato` con `vocear()` (devuelve "cuac") y `Gallina` con `vocear()` (devuelve "cloc cloc"). Escribí una función `coro(animales)` que imprima el `vocear()` de cada uno, y llamala con `[Pato(), Gallina()]`. Sin herencia, sin `isinstance`: puro duck typing.
2. **El molde que obliga**: clase `Figura(ABC)` con `@abstractmethod area()`. `Cuadrado` implementa `area()`; `Circulo` se la olvida. Mostrá que `Circulo()` lanza `TypeError` y que `Cuadrado(4)` funciona.
3. **El ABC con piernas**: `Figura(ABC)` con `__init__(lados)`, `@abstractmethod area()`, y un método concreto `describir()` que devuelve "Soy una figura con N lados." `Rectangulo(Figura)` implementa `area()` y hereda `describir()` sin reescribirlo.
4. **El pato de la biblioteca**: clase `Mazo` con `__len__` y `__iter__`. Mostrá `isinstance(Mazo(), Sized)` (True), `isinstance(Mazo(), Iterable)` (True) y `isinstance(Mazo(), Sequence)` (False en 3.10 — y explicá por qué con lo de los hooks).
5. **Mixins y el orden importa**: mixins `Caminante` (`moverse()` → "caminando") y `Nadador` (`moverse()` → "nadando"), y `Anfibio(Caminante, Nadador)` con `pass`. Imprimí `Anfibio().moverse()` y el MRO. Después definí `Anfibio2(Nadador, Caminante)` y mostrá que ahora gana el nadador.
6. **Protocol**: `Imprimible(Protocol)` con `a_texto() -> str`; `Reporte` y `Foto` que lo cumplen sin heredar. Función `imprimir(objeto: Imprimible)` que haga `print(objeto.a_texto())`. Probalo con `@runtime_checkable` y un `isinstance(Reporte(), Imprimible)`.
7. **Persona tipada, versión completa**: `Persona(nombre: str, edad: int)` con `es_mayor() -> bool`, y `filtrar(personas: list[Persona], criterio: Callable[[Persona], bool]) -> list[Persona]` del capítulo 9. Filtrá mayores con una lambda y mostrá el resultado.
8. **El detector de patos**: `@runtime_checkable` `Protocol Ruidoso` con `hacer_ruido() -> str`. `Vaca` (devuelve "mu") y `Motor` (no tiene el método). Mostrá `isinstance(Vaca(), Ruidoso)` y `isinstance(Motor(), Ruidoso)`, y el `AttributeError` honesto al intentar `Motor().hacer_ruido()`.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. El coro del pato
class Pato:
    def vocear(self):
        return "cuac"


class Gallina:
    def vocear(self):
        return "cloc cloc"


def coro(animales):
    for a in animales:
        print(a.vocear())


coro([Pato(), Gallina()])
# cuac
# cloc cloc

# 2. El molde que obliga
from abc import ABC, abstractmethod


class Figura(ABC):
    @abstractmethod
    def area(self):
        """Superficie de la figura."""


class Cuadrado(Figura):
    def __init__(self, lado):
        self.lado = lado

    def area(self):
        return self.lado * self.lado


class Circulo(Figura):
    def __init__(self, radio):
        self.radio = radio


print(Cuadrado(4).area())   # 16
try:
    c = Circulo(3)
except TypeError as error:
    print(type(error).__name__, "-", error)
# TypeError - Can't instantiate abstract class Circulo with abstract method area

# 3. El ABC con piernas
class Figura(ABC):
    """Molde con lo abstracto que obliga y lo concreto que comparte."""

    def __init__(self, lados):
        self.lados = lados

    @abstractmethod
    def area(self):
        """Superficie de la figura."""

    def describir(self):
        return f"Soy una figura con {self.lados} lados."


class Rectangulo(Figura):
    def __init__(self, base, altura):
        super().__init__(4)
        self.base = base
        self.altura = altura

    def area(self):
        return self.base * self.altura


r = Rectangulo(3, 4)
print(r.describir())   # Soy una figura con 4 lados.
print(r.area())        # 12

# 4. El pato de la biblioteca
from collections.abc import Sized, Iterable, Sequence


class Mazo:
    """Un mazo de cartas."""

    def __init__(self):
        self.cartas = list(range(52))

    def __len__(self):
        return len(self.cartas)

    def __iter__(self):
        return iter(self.cartas)


m = Mazo()
print(isinstance(m, Sized))           # True
print(isinstance(m, Iterable))        # True
print(isinstance(m, Sequence))        # False (3.10: Sequence no define hook estructural)

# 5. Mixins y el orden importa
class Caminante:
    def moverse(self):
        return "caminando"


class Nadador:
    def moverse(self):
        return "nadando"


class Anfibio(Caminante, Nadador):
    pass


class Anfibio2(Nadador, Caminante):
    pass


print(Anfibio().moverse())     # caminando → el primero de la lista gana
print(Anfibio2().moverse())    # nadando    → el primero de la lista gana
print(Anfibio.__mro__)
# (<class '__main__.Anfibio'>, <class '__main__.Caminante'>, <class '__main__.Nadador'>, <class 'object'>)

# 6. Protocol
from typing import Protocol, runtime_checkable


@runtime_checkable
class Imprimible(Protocol):
    def a_texto(self) -> str: ...


class Reporte:
    def a_texto(self) -> str:
        return "Reporte de ventas"


class Foto:
    def a_texto(self) -> str:
        return "Una foto de la playa"


def imprimir(objeto: Imprimible) -> None:
    print(objeto.a_texto())


imprimir(Reporte())     # Reporte de ventas
imprimir(Foto())        # Una foto de la playa
print(isinstance(Reporte(), Imprimible))    # True → el pato pasa el control

# 7. Persona tipada, versión completa
from typing import Callable


class Persona:
    def __init__(self, nombre: str, edad: int) -> None:
        self.nombre: str = nombre
        self.edad: int = edad

    def es_mayor(self) -> bool:
        return self.edad >= 18


def filtrar(personas: list[Persona], criterio: Callable[[Persona], bool]) -> list[Persona]:
    return [p for p in personas if criterio(p)]


personas = [Persona("Ana", 25), Persona("Luis", 16), Persona("Eva", 34)]
for p in filtrar(personas, lambda p: p.es_mayor()):
    print(f"{p.nombre} ({p.edad}) es mayor de edad")
# Ana (25) es mayor de edad
# Eva (34) es mayor de edad

# 8. El detector de patos
@runtime_checkable
class Ruidoso(Protocol):
    def hacer_ruido(self) -> str: ...


class Vaca:
    def hacer_ruido(self) -> str:
        return "mu"


class Motor:
    pass


print(isinstance(Vaca(), Ruidoso))     # True
print(isinstance(Motor(), Ruidoso))    # False
try:
    Motor().hacer_ruido()
except AttributeError as error:
    print(type(error).__name__, "-", error)
# AttributeError - 'Motor' object has no attribute 'hacer_ruido'
```