# Capítulo 14 — La encapsulación sin decreto: `_`, `__` y `@property`

El capítulo 13 terminó con una pregunta incómoda: en el `rango` guardaste el contador en `self.__inicio`, y el capítulo 12 aseguró que "no hay privados". Si no hay candados, ¿qué es ese doble guion bajo? La respuesta es la primera gran técnica de *ingeniería* de la Parte VI: **la encapsulación** — esconder el esqueleto, exponer solo la piel.

En Java o C# el contrato se impone con palabras mágicas: `private`, `public`, `protected`. Python, el rebelde, **no decreta**: la palabra `private` ni siquiera existe, y todo se puede tocar. A cambio te da un kit completo de sombras — `_`, `__`, `property`, `@property` — donde el célebre par *getter y setter* no se escribe como métodos `getNombre()`/`setNombre()` al estilo Java, sino que nace disfrazado de atributo: el getter es la propiedad que se lee sin paréntesis, y su `.setter` es la guardia que valida cada `objeto.atributo = x`. Todo para declarar intenciones y, sobre todo, para **proteger el estado** del objeto: que un atributo no pueda quedar en un valor absurdo, aunque lo intenten desde cualquier parte del programa. Es la filosofía de "somos adultos": el lenguaje te da todas las herramientas, el profesional decide cuándo y por qué restringir.

---

## 1. La interfaz: el contrato

En POO hay dos conceptos que suelen confundirse, y la distinción es oro puro. Empecemos por la **interfaz**: las reglas a partir de las cuales un método puede comunicarse con los métodos de otros objetos. Es el *qué*: qué parámetros acepta, qué devuelve. En Python, una interfaz se declara simplemente escribiendo el método que indica los parámetros que usa.

Una clase `Dispositivo` define dos interfaces con distinta dureza: la del `__init__` es blanda (acepta lo que sea convertible a `float`), y la de `duracion` es estricta (exige una lista o tupla de exactamente dos elementos, con un número en el primer lugar y el texto `"watts/hr"` en el segundo — y si no, `ValueError`):

```python
class Dispositivo:
    """Un dispositivo eléctrico: define interfaces para consumo y duración."""

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
print(tostadora.duracion((12.234, "wAtTs/HR")))   # 0.6117  → 12.234 / 20
```

La interfaz de `duracion` es un contrato flexible y amigable: acepta tupla *o* lista, el valor numérico *o* decimal, y el texto en cualquier capitalización (el `casefold()` del capítulo 3). Pero si no se cumple el contrato, no hay silencio: se levanta `ValueError`:

```python
try:
    tostadora.duracion(1500)
except ValueError as e:
    print("ValueError:", e)   # Interfaz incorrecta.
```

Tres lecturas nuevas:

- **El `__init__` también define una interfaz**: `Dispositivo(20)` promete "dame algo convertible a `float`", y el `float(consumo)` interno obliga a cumplirla. Con `Dispositivo("150")` entrás un texto y adentro te lo convierte — la interfaz se cumple igual.
- La validación *dentro* del método es el patrón que ya viste en el capítulo 12 y que vas a ejercitar todo el capítulo: **el método es la frontera**. Todo lo que entra por un objeto de verdad pasa por ahí.
- Notá el `raise ValueError` manual: el `raise` del capítulo 6 ahora es el guardia de las interfaces. Interfaz rota = excepción clara.

> **Dato clave:** una interfaz *no es* una clase ni un tipo: es una promesa de comunicación. "Este método recibe tal cosa y te devuelve tal otra." En Python la promesa se lee en la firma (y desde el capítulo 9, en los type hints); los lenguajes con `private`/`public` hacen lo mismo, pero escribiendo un adverbio en cada línea.

---

## 2. Muchas implementaciones, una interfaz

La **implementación** es la *otra mitad*: la manera concreta en que un método realiza las operaciones para devolver lo que su interfaz prometió. Pueden existir **muchas implementaciones para una misma interfaz**, e incluso cambiarse con el tiempo — lo que no cambia es la entrega garantizada por la interfaz.

Un sistema de energía con dos clases que comparten la interfaz `energia(cantidad) -> (valor, "watts/hr")`, pero cada una lo implementa a su manera:

```python
class Fotovoltaica:
    rendimiento = 500

    def energia(self, lumenes):
        return (lumenes * self.rendimiento, "watts/hr")


class Hidroelectrica:
    rendimiento = 2000

    def energia(self, litros):
        return (litros * self.rendimiento, "watts/hr")


def horas(fuente, cantidad):
    """Cuántas horas aguanta un Dispositivo con una fuente de energía dada."""
    tostadora = Dispositivo(10)
    origen = fuente()
    energia = origen.energia(cantidad)
    return tostadora.duracion(energia)

print(horas(Hidroelectrica, 500))   # 100000.0  → 500 × 2000 = 1.000.000 W / 10 W
print(horas(Fotovoltaica, 500))     # 25000.0   → 500 × 500  = 250.000  W / 10 W
```

Mirá la arquitectura: la función `horas` **no sabe** si le pasaste una hidroeléctrica o una fotovoltaica. Solo usa la interfaz — llamado `fuente()`, luego `origen.energia(cantidad)` — y ambas respetan el contrato de `Dispositivo.duracion`. Si mañana aparece una clase `Nuclear` con su `energia()`, `horas` funciona sin tocarla. Eso es exactamente código **desacoplado**: la pieza que produce energía y la pieza que la consume no se conocen entre sí; hablan por la interfaz.

> **Dato clave:** interfaz e implementación son las dos caras de la misma moneda. La interfaz es el *qué* (no cambia); la implementación es el *cómo* (puede cambiar, y es la que encapsulás). Por eso en el idioma de la POO se dice "programar contra interfaces, no contra implementaciones".

---

## 3. El encapsulamiento: esconder el esqueleto

De interfaz e implementación nace el concepto estrella del capítulo: la **encapsulación**. Es la técnica de exponer los atributos de un objeto **exclusivamente a través de sus interfaces** y restringir el acceso al estado y a las implementaciones internas. Pensalo como un teléfono: el *qué* es el teclado y la pantalla (la interfaz); el *cómo* son los circuitos de adentro, que nadie toca desde afuera (la implementación). Vos no abrís el teléfono para cambiar la batería del chip: llamás al número que corresponde.

En Java, C# y casi todos los lenguajes "de manual", esa contención se impone declarando atributos y métodos como `private`, `protected` o `public` — el compilador *te impide* tocar lo privado. Python no tiene ninguno de esos modificadores. El rebelde hace la contención con una regla social: **convención y ofuscación, no decreto**. La regla social se ve con una caja fuerte:

```python
class CajaDeSeguridad:
    """Caja fuerte con una contraseña 'escondida'."""

    __contraclave = "123qwe"

    def seguro(self, clave):
        if self.__contraclave == clave:
            print("Acceso concedido.")
        else:
            print("Acceso denegado.")

caja = CajaDeSeguridad()
caja.seguro("Hola")       # Acceso denegado.
caja.seguro("123qwe")     # Acceso concedido.
```

La clase guarda su contraseña "en la sombra" y la usa adentro, en `seguro`. Desde afuera, la interfaz es un solo método: `caja.seguro(clave)`. No hay manera (buena) de leer la contraseña. En un lenguaje con `private`, `__contraclave` sería intocable por una regla del compilador. Acá, en cambio, pasó otra cosa — y esa otra cosa es la estrella de la próxima sección.

---

## 4. Name mangling: `__` y `_`, las dos sombras

¿Qué hace exactamente el doble guion bajo de `__contraclave`? No es un candado: es **name mangling** (mangling = "destrozar, deformar el nombre"). El intérprete, al compilar la clase, **renombra** los atributos que empiezan con `__` anteponiendo `_NombreDeLaClase`, casi como un apellido:

```python
print(caja._CajaDeSeguridad__contraclave)   # 123qwe → está "renombrada", no "borrada"
print("_CajaDeSeguridad__contraclave" in dir(caja))   # True → dir() lo delata
```

Reglas del juego:

- Dentro de la clase, `self.__contraclave` se escribe normal y el intérprete lo traduce. **Fuera** de la clase, escribir `caja.__contraclave` levanta `AttributeError` — no porque esté prohibido, sino porque *ese nombre ya no existe*: el real es `_CajaDeSeguridad__contraclave`.
- **Es ofuscación, no seguridad.** `dir(caja)` lo muestra (el capítulo 12 te enseñó a usarlo) y cualquiera puede escribir el nombre manglado:

```python
caja._CajaDeSeguridad__contraclave = "emi"
caja.seguro("emi")        # Acceso concedido. → el maquillaje se puede maquillar
```

- Un matiz fino: el mangling aplica a nombres que **no terminan** en doble guion bajo. Por eso `__init__`, `__add__` y todos los dunders del capítulo 13 **no** se renombran: son los pactados con el intérprete. La regla es *mangling para el que empieza y no termina con `__`*.

El doble guion bajo se usa en serio para esconder los ingredientes internos que ningún consumidor debería tocar. Mirá dos ejemplos reales. Primero un cantante que construye su nombre completo con apellido privado:

```python
class Cantante:
    def __init__(self):
        self.name = "Freddie"
        self.__lastname = "Mercury"

    def PrintName(self):
        return self.name + " " + self.__lastname

favorito = Cantante()
print(favorito.name)          # Freddie
print(favorito.PrintName())   # Freddie Mercury
# print(favorito.__lastname)   # AttributeError: '_Cantante__lastname' renombrado
```

Y después la trampa clásica: por ser *ofuscación* y no candado, un "pirata" puede intentar `bender.__build_year = 5000`, pero eso **no toca** el atributo privado: crea un atributo nuevo, con un nombre distinto, que vive aparte:

```python
class Robot:
    def __init__(self, name=None, year=1900):
        self.__name = name
        self.__build_year = year

    def set_name(self, name):
        self.__name = name

    def get_name(self):
        return self.__name

    def set_build_year(self, by):
        self.__build_year = by

    def get_build_year(self):
        return self.__build_year


bender = Robot("Bender", 2978)
bender.__build_year = 5000        # intento de piratería: crea OTRO atributo
bender.set_build_year(2900)
print(bender.get_name())          # Bender
print(bender.get_build_year())    # 2900   → el interno sigue su curso
print(bender.__build_year)        # 5000   → el atributo pirata, viviendo aparte
print(bender.__dict__)            # nos muestra toda la familia
```

`bender.__dict__` (del capítulo 12) lo confiesa todo: `{'_Robot__name': 'Bender', '_Robot__build_year': 2900, '__build_year': 5000}`. El pirata pintó su garabato en la pared de *afuera*; las máquinas siguieron girando adentro.

### El guion simple: `_` — la línea de la convención

El doble guion bajo es el mangling. El **guion simple** (`_algo`) es otra cosa, más sutil y *más usada en la práctica*: un cartelito que dice "esto es interno, no lo uses". No hay mangling, no hay ningún renombrado — solo una promesa entre adultos. La sombra del guion simple alcanza hasta los módulos: importar una clase que empieza con `_` funciona, pero es como entrar por la puerta del fondo:

```python
class _Secreto:
    """Una clase 'privada por convención'."""

    def saludo(self):
        print("hola desde un método de una clase con guion simple")

objeto = _Secreto()
objeto.saludo()    # funciona: el _ es un cartel, no un candado
```

Eso vale para clases, métodos y hasta módulos y funciones: `_helper`, `_Mensaje`, `_modulo_privado` — todos funcionan si los llamás, pero dicen "esto es implementación, no contrato". De acá en adelante, en el libro:

- **`nombre`** → público: el contrato.
- **`_nombre`** → protegido por convención: interno, no tocar desde afuera.
- **`__nombre`** → manglado: ofuscado con `_Clase__nombre` cuando querés que ni siquiera *aparezca* cómodamente.

Y la frase que resume todo, del capítulo 12: **somos adultos**. El lenguaje no impide nada; la comunidad entiende el significado de cada sombra.

> **Dato clave:** el mangling existe para **blindar el atributo contra la herencia**, no para escondértelo a vos. Si un día `Cantante` tuviera una subclase y esta definiera su propio `__lastname`, el intérprete lo llamaría `_Subclase__lastname` — no pisa al `_Cantante__lastname` de la base. Por eso la guía del mundo real es simple: en el código común usá `_algo` por convención, y reservá `__algo` para los internos que querés que ni siquiera las subclases toquen.

---

## 5. La medicina equivocada: getters y setters a la Java

Viste arriba el `Robot` con `set_name()` y `get_name()`. Eso es **la medicina equivocada**: una traducción literal de los getters y setters de Java, y en Python es verbosa, ruidosa y sin beneficios reales. Si encapsulás para esconder, pero después hay que escribirlo así:

```python
bender = Robot("Bender", 2978)
bender.set_build_year(2900)
print(bender.get_build_year())
```

…el código lector pierde la frescura de los atributos (`bender.build_year`) y no ganó nada que un atributo común no diera. El rebelde tiene una medicina mucho mejor: **`@property`**, que transforma un método en un atributo *con guardia*. Y de paso sana el problema que la clase `Producto` del capítulo 13 prometió: en vez de secuestrar operadores con significados raros, el movimiento correcto es un método con nombre (`descuento(15)`) y una propiedad que proteja el estado. Prepárate, porque la propiedad es la estrella de este capítulo.

---

## 6. `@property`: el atributo con guardia

Una **propiedad** es un método que se comporta como atributo: se lee sin paréntesis, se puede asignar con `=` y, si la dejás sin setter, es **de solo lectura**. Una `Persona` guarda una clave interna generada automáticamente y un nombre que debe ser una lista de 2 a 3 elementos:

```python
class Persona:
    """Persona con una clave interna 'escondida' y un nombre validado."""

    def __init__(self):
        from time import time
        self.__clave = str(int(time() / 0.017))[1:]

    @property
    def clave(self):
        """Solo lectura: no tiene setter."""
        return self.__clave

    @property
    def nombre(self):
        """Construye la cadena completa desde la lista interna."""
        return " ".join(self.lista_nombre)

    @nombre.setter
    def nombre(self, nombre):
        """Debe ingresarse una lista o tupla con entre 2 y 3 elementos."""
        if type(nombre) not in (list, tuple) or not (2 <= len(nombre) <= 3):
            raise ValueError("Formato incorrecto.")
        self.lista_nombre = nombre

sujeto = Persona()
sujeto.nombre = ["Jorge", "Sánchez", "Pérez"]
print(sujeto.nombre)             # Jorge Sánchez Pérez
print(sujeto.lista_nombre)       # ['Jorge', 'Sánchez', 'Pérez']
```

Fijate la sintaxis, que es pura azúcar del capítulo 8: `@property` sobre el getter, `@nombre.setter` sobre el setter (con el *mismo nombre* de método). El setter es el guardia: si alguien intenta `sujeto.nombre = "Jorge"` (un texto, no una lista), el `ValueError` lo frena:

```python
try:
    sujeto.nombre = "Pícaro"
except ValueError as e:
    print("ValueError:", e)    # Formato incorrecto.
```

Y el detalle que explica el "sin candado": la `clave` es solo lectura, pero el mangling del capítulo anterior hace que el escondite siga siendo *visible* para quien sepa el truco — la demostración es directa:

```python
# print(sujeto.clave = 12)   # AttributeError: property sin setter → solo lectura
print(sujeto._Persona__clave)          # el maquillaje sigue ahí
sujeto._Persona__clave = "te juanquié"  # se puede tocar con el nombre manglado
print(sujeto.clave)                     # te juanquié
```

El caso `Donacion` muestra el poder real de la propiedad: **el setter se aplica incluso dentro del `__init__`**. Cuando le das un monto inválido, la clase lo clampéa (lo limita) en vez de romper:

```python
class Donacion:
    """Una donación que se auto-limita entre 0 y 1.000.000."""

    def __init__(self, monto):
        self.monto1 = monto     # ← pasa por el setter, que aplica el tope

    @property
    def monto1(self):
        return self.__monto

    @monto1.setter
    def monto1(self, monto):
        if monto < 0:
            self.__monto = 0
        elif monto > 1_000_000:
            self.__monto = 1_000_000
        else:
            self.__monto = monto

oferta = Donacion(-5)
print(oferta.monto1)            # 0          → el negativo quedó en el piso
oferta.monto1 = 100
print(oferta.monto1)            # 100
oferta.monto1 = 5_000_000
print(oferta.monto1)            # 1000000    → el tope
```

> **Dato clave:** "getter" y "setter" existen en Python, pero **no se escriben como métodos**: se escriben como una propiedad y se usan como atributos (`objeto.atributo` y `objeto.atributo = x`). Si ves en un código `getAlgo()`/`setAlgo()` como los del `Robot` de la sección 5, estás mirando tinta de otro idioma.

---

## 7. La Persona del capítulo 12, ahora con guardia

Recordá la `Persona` validada del capítulo 12: en su `__init__` convertía la edad con `int()`, dentro de un `try`, y la dejaba en `None` si se iba de rango. Funcionaba, pero tenía un agujero: **la validación corría una sola vez, al crear el objeto**. Después, `emi.edad = 200` entraba sin que nadie chistara.

La propiedad cierra ese agujero: la guardia pasa a vivir **en el atributo mismo**, y vale para siempre — en el constructor y en cada asignación posterior. Esa es la *reescritura* que el capítulo 12 te quedó debiendo:

```python
class Persona:
    """La Persona del capítulo 12, ahora con guardia permanente."""

    def __init__(self, nombre: str, apellido: str, edad: int):
        self.nombre = nombre
        self.apellido = apellido
        self.edad = edad            # → pasa por el setter, que valida

    @property
    def edad(self):
        return self._edad

    @edad.setter
    def edad(self, valor):
        try:
            valor = int(valor)
        except ValueError:
            print(valor, "no es un parámetro válido para la edad")
            self._edad = None
            return
        if 0 < valor < 120:
            self._edad = valor
        else:
            print(valor, "está fuera de rango: permitido entre 0 y 120")
            self._edad = None

emi = Persona("Emiliano", "Passarello", 48)
belen = Persona("Belen", "Cianfagna", 42)
peron = Persona("Juan", "Perez", 135)      # fuera de rango → edad None
print(emi.nombre, emi.edad)                # Emiliano Passarello 48
print(peron.nombre, peron.edad)            # Juan Perez None
```

Comparalo con el capítulo 12 y vas a ver que la lógica de validación es la misma (`try` + `int()` + rango `0-120`). Lo que cambió es el *dónde*: en vez de estar encerrada en `__init__`, ahora vive en el setter, y **cada asignación desde cualquier parte del programa pasa por la guardia**:

```python
emi.edad = "cuarenta"      # no es numérico → mensaje + None
print(emi.edad)            # None
emi.edad = 200             # fuera de rango → mensaje + None
emi.edad = 33
print(emi.edad)            # 33
```

Notá de paso el `_edad` (un solo guion, la convención de la sección 4): la propiedad guarda el valor real en un "escondite por convención", y el nombre público `edad` queda reservado para la puerta con guardia. Ese es el patrón completo: **atributo `_x` (privado por convención) + propiedad `x` (interfaz con validación)**.

> **Buenas prácticas:** validá en el **borde**, no en el centro. Un objeto con `edad = -3` que nadie detectó hasta la página 500 es una bomba; un objeto que rechaza el valor *en el momento de la asignación* es un sistema que no puede romperse solo. La propiedad es la frontera donde el borde se aplica una sola vez y para siempre.

---

## 8. `del` con ceremonia: el deleter

El capítulo 12 te mostró `del objeto.atributo` pelado. La propiedad suma el tercer miembro de la familia: el **deleter** — lo que pasa *cuando borrás* la propiedad. Un `Alumno` tiene una propiedad `mostrar` que es un resumen armado en el vuelo:

```python
class Alumno:
    def __init__(self, nombre, apellido, edad):
        self.nombre = nombre
        self.apellido = apellido
        self.edad = edad

    @property
    def mostrar(self):
        """El resumen completo, armado cuando se lee (sin paréntesis)."""
        if self.nombre is None:
            return "usuario inexistente"
        return f"{self.nombre} {self.apellido} tiene la edad de {self.edad}"

    @mostrar.setter
    def mostrar(self, datos):
        self.nombre, self.apellido, self.edad = datos

    @mostrar.deleter
    def mostrar(self):
        print(f"vamos a borrar {self.nombre} {self.apellido}")
        self.nombre = None
        self.apellido = None
        self.edad = None

emi = Alumno("emi", "pas", 45)
print(emi.mostrar)                  # emi pas tiene la edad de 45
emi.mostrar = ("belen", "cianfagna", 45)
print(emi.mostrar)                  # belen cianfagna tiene la edad de 45
del emi.mostrar                     # → el deleter corre su ceremonia
print(emi.mostrar)                  # usuario inexistente
```

Tres detalles:

- La propiedad `mostrar` se lee **sin paréntesis** (`emi.mostrar`, no `emi.mostrar()`): ya no es un método, es un atributo calculado en el vuelo. Esa es la diferencia entre `@property` y un método común: `print(emi.mostrar)` luce como acceso a dato, pero corre código.
- El setter desempaqueta una tupla (el desempaquetado del capítulo 7): `emi.mostrar = ("belen", "cianfagna", 45)` actualiza los tres campos de un saque.
- El `del` con ceremonia: en vez de borrar el atributo y dejar el objeto cojo, el deleter lo limpia con elegancia. `del` deja de ser un garrotazo y pasa a ser un protocolo.

---

## 9. `@classmethod`: la clase como protagonista

Todas las clases que viste operan sobre **instancias**: el primer parámetro es `self` y toca el estado de ese objeto en particular. Pero hay datos que no viven en la instancia, sino **en la clase entera**: el atributo de clase del capítulo 12. Para operar sobre *ese* estado, el rebelde tiene el método de clase: primer parámetro `cls` (no `self`) — el mismo `cls` que `__new__` te presentó en el capítulo 13 —, decorado con `@classmethod`.

Un censo de población lo deja a la vista: cada instancia suma al total común de su clase:

```python
class PoblacionCensada:
    """Registra la población total de todas sus instancias (vivitas y coleando)."""

    poblacion = 0

    @classmethod
    def opera_poblacion(cls, operador, cantidad):
        """Suma o resta al total de la clase."""
        if operador == "+":
            cls.poblacion += cantidad
        elif operador == "-":
            cls.poblacion -= cantidad
        return cls.poblacion

    @classmethod
    def despliega_total(cls):
        """El total de la clase, consultable en cualquier momento."""
        return cls.poblacion

    def __init__(self, nombre, numero=0):
        print(f"Se creó la población {nombre} con {numero} habitantes.")
        self.nombre = nombre
        self.poblacion = numero         # atributo de la instancia
        self.opera_poblacion("+", self.poblacion)

    def __del__(self):
        self.opera_poblacion("-", self.poblacion)
```

La tentación clásica acá es `eval()` (que ejecuta texto como código — peligroso); esta versión la resuelve con un `if` honesto. Los métodos de clase tienen dos particularidades que brillan acá:

- **Tocan el estado de la clase, no el de la instancia.** `cls.poblacion` es un solo número compartido; cada instancia suma al crearse (`__init__`, de nuevo el constructor del capítulo 12) y resta al morir (`__del__`). Mirá la magia del censo:

```python
edomex = [PoblacionCensada("Tlalnepantla", 600000),
          PoblacionCensada("Toluca", 1000000),
          PoblacionCensada("Valle de Chalco", 750000),
          PoblacionCensada("Valle de Bravo", 100000)]

print(PoblacionCensada.poblacion)        # 2450000 → 600K + 1M + 750K + 100K
print(edomex[0].despliega_total())       # 2450000 → método de clase invocado desde un objeto
print(edomex[0].poblacion)               # 600000  → el de la instancia no se tocó
```

- **Pueden invocarse desde un objeto** (`edomex[0].despliega_total()`) y aun así operan sobre la clase. Cuando `del` elimina una instancia, el total se acomoda solo:

```python
del edomex[1]                            # Toluca se va → resta 1.000.000
print(PoblacionCensada.poblacion)        # 1450000
for entidad in edomex:
    print(entidad.nombre, entidad.poblacion)   # Tlalnepantla 600000 / Valle de Chalco 750000 / Valle de Bravo 100000
```

La ventaja frente a tocar `PoblacionCensada.poblacion = ...` a mano: la lógica (sumar con `+`, restar con `-`) queda **encapsulada en la clase**, y cualquier futuro cambio de reglas se hace en un solo lugar.

> **Dato clave:** un `@classmethod` es el "señor de la clase". No importa desde dónde lo llames (clase u objeto): siempre recibe `cls` y opera sobre el molde. Es la herramienta natural para contadores globales, configuraciones compartidas y **fábricas** — métodos que construyen instancias de otra manera (lo vas a ver en el capítulo 15).

---

## 10. `@staticmethod`: la utilidad sin estado

El **método estático** es el tercer vértice del triángulo: no recibe `self` ni `cls`, y por lo tanto **no tiene acceso a nada** del objeto ni de la clase — es, literalmente, una *función común disfrazada de método*, útil cuando pertenece conceptualmente al molde pero no necesita su estado. Un `Servidor` con un `ping` que responde una tupla (la IP y la palabra):

```python
class Servidor:
    """Un servidor muy básico, con un ping que no necesita estado."""

    usuarios_activos = set()     # clase compartida: la sesión de todos los servidores

    def __init__(self, dominio, lista):
        self.lista_usuarios = lista
        self.dominio = dominio

    def conexion(self, usuario):
        """Conecta a un usuario válido, agrega a la sesión compartida."""
        if usuario in self.lista_usuarios:
            self.usuarios_activos.add(usuario)
        else:
            return False

    @staticmethod
    def ping(ip):
        """Responde la IP y un 'ping': no necesita ni self ni cls."""
        return (ip, "ping")

server = Servidor("demo.miweb.com", ["josech", "juan", "mglez", "jklx"])
print(server.ping("182.168.100.1"))    # ('182.168.100.1', 'ping')
print(Servidor.ping("127.0.0.1"))      # también se llama por la clase
```

El `ping` no mira `self.lista_usuarios` ni `cls.usuarios_activos`: solo recibe la IP y devuelve la respuesta. Por eso puede llamarse desde un objeto **o** desde la propia clase — cualquiera de las dos vías está bien. El método normal `conexion`, en cambio, *sí* toca estado: valida contra la lista de ese servidor y agrega a la sesión de la clase:

```python
server.conexion("juan")
print(server.usuarios_activos)     # {'juan'}
print(server.conexion("lucía"))    # False → no es usuario válido
```

Mirá que `usuarios_activos` es atributo de clase: si crearas un segundo servidor, compartiría la misma sesión — el estado vive en el molde, no en cada copia. Los tres vértices, de una vez:

| Tipo | Decorador | Primer parámetro | Ve la instancia | Ve la clase | Cómo se llama |
|------|-----------|------------------|-----------------|-------------|---------------|
| método de instancia | (ninguno) | `self` | sí | sí | `objeto.metodo()` |
| método de clase | `@classmethod` | `cls` | no | sí | `Clase.metodo()` o `objeto.metodo()` |
| método estático | `@staticmethod` | (ninguno) | no | no (salvo nombrarla) | `Clase.metodo()` o `objeto.metodo()` |

> **Buenas prácticas:** la regla de oro al decidir — *¿necesita el objeto?* entonces instancia; *¿necesita el molde?* entonces `@classmethod`; *¿no necesita nada pero conceptualmente le pertenece al molde?* entonces `@staticmethod`. Cuando no estés seguro, empezá por el método de instancia: es la opción natural y la que menos decisiones raras esconde.

---

## 11. `@dataclass` no protege: la guardia la ponés vos

El capítulo 13 terminó con las `dataclasses`, que generan `__init__`, `__repr__` y `__eq__` por vos. Pero ojo con una tentación: **la dataclass abre la puerta de par en par**. Sus campos son atributos públicos, sin validación. Podés cargar un `descuento` de 250 cuando el negocio dice que el máximo es 50, y la dataclass no se entera:

```python
from dataclasses import dataclass

@dataclass
class Cliente:
    """Un cliente con su descuento. Los campos son públicos y sin guardia."""

    nombre: str
    descuento: int = 0

cliente = Cliente("Luca")
cliente.descuento = 250        # aceptado sin chistar...
print(cliente)                 # Cliente(nombre='Luca', descuento=250) ← el negocio llora
```

¿Por qué? Porque `@dataclass` genera los dunders del capítulo 13, pero **no genera un `__setattr__` con validación** (y si definieras `__setattr__` a mano, romperías la comodidad de la dataclass). El rebelde no se queda llorando: toma lo mejor de cada mundo — la dataclass para la ceremonia (`__init__`/`__repr__`/`__eq__`) y la propiedad para el guardia. Los dos conviven: dejá como campo dataclass lo que es libre, y convertí en propiedad lo que necesita frontera.

Una clase `Fraccion` con TODOS los operadores de `__add__`/`__mul__`/comparaciones (capítulo 13) y una función `mcd` con el algoritmo de Euclides para reducir. Nada protege su denominador: cualquiera hace `f.den = 0` y la fracción se rompe. Con la propiedad, la guardia llega al lugar exacto — y de paso se reduce sola vía `mcd`:

```python
def mcd(a, b):
    """Máximo común divisor por el algoritmo de Euclides."""
    while b:
        a, b = b, a % b
    return a or 1


class Fraccion:
    """Una fracción que se reduce sola y jamás acepta denominador 0."""

    def __init__(self, num, den=1):
        self.den = den          # → pasa por el setter (den ≠ 0)
        self.num = num

    @property
    def num(self):
        return self._num

    @num.setter
    def num(self, valor):
        self._num = int(valor)

    @property
    def den(self):
        return self._den

    @den.setter
    def den(self, valor):
        valor = int(valor)
        if valor == 0:
            raise ZeroDivisionError("el denominador no puede ser 0")
        self._den = valor

    def __str__(self):
        divisor = mcd(self.num, self.den)
        num = self.num // divisor
        den = self.den // divisor
        if den < 0:
            num, den = -num, -den
        return f"{num}/{den}"

    def numero(self):
        return self.num / self.den


f = Fraccion(8, 12)
print(f)                   # 2/3 → se reduce sola
print(f.numero())          # 0.6666666666666666
try:
    Fraccion(1, 0)
except ZeroDivisionError as e:
    print("ZeroDivisionError:", e)   # el denominador no puede ser 0
```

Dos notas finas, de la vida real:

- **Guardás solo lo que necesita guardia.** Acá el `den` es sagrado (nunca 0) y el `num` usa propiedad por simetría con el ejemplo didáctico; en tu código vas a decidir campo por campo cuánta valla necesita cada uno. La sobreingeniería es el arte de esconder hasta lo que no hace falta proteger.
- El `__str__` reduce la fracción con `mcd` cada vez que se muestra — el estado se exhibe ya simplificado, sin tocar los atributos internos (otra pequeña dosis de encapsulación: el *cómo* se guarda (8/12) y el *qué* se muestra (2/3) están separados).

---

## 12. Resumen y conceptos clave

Este capítulo fue el de la **encapsulación sin decreto**: el rebelde no tiene `private`, pero tiene un idioma completo de sombras y de guardias. Empezaste separando **interfaz** (*qué* promete un método) de **implementación** (*cómo* lo cumple), y viste el desacople con el sistema de energía: `horas()` conversa con cualquiera que respete la interfaz `energia()`, por eso muchas implementaciones (fotovoltaica, hidroeléctrica, la que venga) conviven bajo el mismo contrato. Definiste **encapsulamiento**: exponer el estado solo a través de las interfaces y esconder la implementación. Y ahí entró la caja de herramientas: el **name mangling** de `__nombre` (renombra a `_Clase__nombre`: ofuscación, no candado — `dir()` lo delata), el guion simple `_nombre` como cartel de "interno, no toques" (protegido *por convención*), y la trampa del atributo pirata (`bender.__build_year = 5000` crea otro atributo, no toca al manglado). Descartaste los getters/setters al estilo Java como medicina equivocada y abrazaste la estrella del capítulo: **`@property`**, que convierte un método en atributo con guardia (lectura sin paréntesis, setter que valida, y sin setter = solo lectura). Reescribiste la `Persona` del capítulo 12 para que la validación de la edad viva **en el atributo** y valga para siempre — bordes, no centros. Sumaste el **deleter** (`del` con ceremonia), el **`@classmethod`** (el señor de la clase: opera sobre `cls`, contadores y censos compartidos sin `eval`), el **`@staticmethod`** (la función del molde sin estado), la tabla para elegir entre los tres, y cerraste con la advertencia de que **`@dataclass` no protege**: los campos públicos quedan abiertos, y la guardia (como en `Fraccion` con su `mcd` de Euclides) la ponés vos donde la necesitás.

Repasá el checklist antes de seguir:

- [ ] **Interfaz** = el *qué*: reglas de comunicación del método (parámetros y devolución). Se declara escribiendo el método, se robustece validando adentro y `raise` ante violación.
- [ ] **Implementación** = el *cómo*: varias pueden cumplir una misma interfaz; la interfaz es estable, la implementación es reemplazable (desacople).
- [ ] **Encapsulamiento** = exponer el estado solo vía interfaces y esconder la implementación. En Python: convención y ofuscación, no decreto.
- [ ] **`__nombre`** (doble guion al inicio): name mangling → `_Clase__nombre`. Útil para ofuscar internos; **no es seguridad** (`dir()` y el nombre manglado lo muestran). No aplica a dunders como `__init__`.
- [ ] **`_nombre`** (guion simple): protegido *por convención* ("somos adultos"); no se renombra, solo advierte.
- [ ] **`@property`**: método que se usa como atributo (sin paréntesis). Setter = validación en cada asignación; sin setter = **solo lectura**.
- [ ] El **setter corre hasta en el `__init__`** (`self.x = valor` pasa por la guardia) — validá en el borde, no en el centro.
- [ ] **`del`** puede tener ceremonia: `@x.deleter` (borrado limpio, no garrotazo).
- [ ] **`@classmethod`** recibe `cls`, opera sobre la clase (total, censo, fábrica), y puede invocarse desde un objeto.
- [ ] **`@staticmethod`** no recibe nada del objeto ni de la clase: utilidad con nombre, llamada por clase u objeto.
- [ ] Elegir el método: instancia si necesita el objeto; `@classmethod` si toca el molde; `@staticmethod` si no toca ninguno.
- [ ] **`@dataclass` no protege**: campos públicos sin validación; la guardia se agrega con `@property` campo por campo.

---

## 13. Ejercicios

1. **La interfaz del cajero**: clase `Cajero` con `saldo` inicial; `extraer(monto)` que valide la interfaz (número, mayor que 0, y que no supere el saldo) y `depositar(monto)` con su validación; ambos con `ValueError` claro ante interfaz rota. Probá un extra válido, un depósito feliz y un intento de extraer con un texto.
2. **La bóveda con maquillaje**: clase `Boveda` con `__codigo`, un método `abrir(clave)` ("Abierta."/"Denegado.") y la demostración completa del mangling: `dir()`, el acceso `_Boveda__codigo` y el cambio "pirata" del código.
3. **El nombre con guardia**: clase `Nombre` con `@property` `nombre` que solo acepte un `str` no vacío (sin espacios a los costados), con `ValueError` si no. Probá asignar "Luca", asignar `""` y leer.
4. **La `Persona` del capítulo 12, versión guardia**: reescribila con `@property` en `edad` (validación `0 < int(edad) < 120` con `try`/`except`, `None` en caso contrario) y demostrá que la guardia aplica *también después* de crear el objeto (`emi.edad = 200`).
5. **La tinta con ceremonia**: clase `Tinta` con `@property` `color` (setter que solo acepte `str`, deleter que lo deje en `None` imprimiendo qué borra) y probá `del`.
6. **El contador de ventas**: clase `Ventas` con `total` de clase, `@classmethod sumar(monto)` que acumule, y por cada venta (nombre y monto) sumar al total; demostrá `del` restando.
7. **El conversor sin estado**: clase `Conversor` con `cotizacion` de clase y un `@staticmethod euros_a_pesos(euros)`; llamalo por la clase y por un objeto.
8. **La fracción blindada**: dale a una `Fraccion` un `@property den` que rechace `denominador == 0` (con `ZeroDivisionError`) y que reduzca sola con una función `mcd` (Euclides). Probá `Fraccion(6, 4)`, `Fraccion(1, 0)` y el `str` ya reducido.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. La interfaz del cajero
class Cajero:
    """Cajero que cobra y extrae con interfaz validada."""

    def __init__(self, saldo=0):
        self.saldo = saldo

    def extraer(self, monto):
        if type(monto) not in (int, float):
            raise ValueError("Interfaz incorrecta: monto numérico.")
        if monto <= 0:
            raise ValueError("Interfaz incorrecta: monto positivo.")
        if monto > self.saldo:
            raise ValueError("Saldo insuficiente.")
        self.saldo -= monto
        return self.saldo

    def depositar(self, monto):
        if type(monto) not in (int, float) or monto <= 0:
            raise ValueError("Interfaz incorrecta: monto numérico positivo.")
        self.saldo += monto
        return self.saldo

cajero = Cajero(1000)
print(cajero.depositar(500))    # 1500
print(cajero.extraer(200))      # 1300
try:
    cajero.extraer("mucho")
except ValueError as e:
    print("ValueError:", e)     # Interfaz incorrecta: monto numérico.

# 2. La bóveda con maquillaje
class Boveda:
    """Bóveda con código 'escondido' por name mangling."""

    def __init__(self, codigo):
        self.__codigo = codigo

    def abrir(self, clave):
        if clave == self.__codigo:
            print("Abierta.")
        else:
            print("Denegado.")

b = Boveda("1234")
b.abrir("0000")                 # Denegado.
print("__codigo" in dir(b))     # False → el nombre literal ya no existe
print(b._Boveda__codigo)        # 1234 → el manglado, viva la vista
b._Boveda__codigo = "9999"      # lo cambiamos 'de afuera'...
b.abrir("9999")                 # Abierta. → es maquillaje, no candado

# 3. El nombre con guardia
class Nombre:
    """Propiedad con guardia: un nombre que no puede quedar vacío."""

    def __init__(self, valor):
        self.nombre = valor

    @property
    def nombre(self):
        return self._nombre

    @nombre.setter
    def nombre(self, valor):
        if type(valor) is not str or not valor.strip():
            raise ValueError("el nombre debe ser un texto no vacío")
        self._nombre = valor

n = Nombre("Luca")
print(n.nombre)                 # Luca
try:
    n.nombre = ""
except ValueError as e:
    print("ValueError:", e)     # el nombre debe ser un texto no vacío

# 4. La Persona del capítulo 12, versión guardia
class Persona:
    def __init__(self, nombre: str, apellido: str, edad: int):
        self.nombre = nombre
        self.apellido = apellido
        self.edad = edad

    @property
    def edad(self):
        return self._edad

    @edad.setter
    def edad(self, valor):
        try:
            valor = int(valor)
        except ValueError:
            print(valor, "no es un parámetro válido para la edad")
            self._edad = None
            return
        if 0 < valor < 120:
            self._edad = valor
        else:
            print(valor, "está fuera de rango: permitido entre 0 y 120")
            self._edad = None


emi = Persona("Emiliano", "Passarello", 48)
emi.edad = 200                   # imprime el mensaje de rango
print(emi.edad)                  # None
emi.edad = 35
print(emi.edad)                  # 35

# 5. La tinta con ceremonia
class Tinta:
    """Color de tinta con del ceremonioso."""

    def __init__(self, color):
        self.color = color

    @property
    def color(self):
        return self._color

    @color.setter
    def color(self, valor):
        if type(valor) is not str:
            raise ValueError("el color debe ser un texto")
        self._color = valor

    @color.deleter
    def color(self):
        print(f"borrando el color {self._color}")
        self._color = None

t = Tinta("azul")
print(t.color)                   # azul
del t.color                      # borrando el color azul
print(t.color)                   # None

# 6. El contador de ventas
class Ventas:
    """Registra ventas: el total vive en la clase."""

    total = 0

    @classmethod
    def sumar(cls, monto):
        cls.total += monto
        return cls.total

    def __init__(self, articulo, monto):
        self.articulo = articulo
        self.monto = monto
        self.sumar(monto)

    def __del__(self):
        self.sumar(-self.monto)

venta1 = Ventas("remera", 2000)
venta2 = Ventas("jean", 4500)
print(Ventas.total)              # 6500
del venta1
print(Ventas.total)              # 4500

# 7. El conversor sin estado
class Conversor:
    """Utilidades de conversión sin estado."""

    cotizacion = 1400

    @staticmethod
    def euros_a_pesos(euros):
        return euros * Conversor.cotizacion

    def __init__(self, moneda):
        self.moneda = moneda

c = Conversor("PESO ARGENTINO")
print(Conversor.euros_a_pesos(10))   # 14000 → por la clase
print(c.euros_a_pesos(10))           # 14000 → por un objeto

# 8. La fracción blindada
def mcd(a, b):
    """Máximo común divisor por el algoritmo de Euclides."""
    while b:
        a, b = b, a % b
    return a or 1


class Fraccion:
    def __init__(self, num, den=1):
        self.num = num
        self.den = den

    @property
    def num(self):
        return self._num

    @num.setter
    def num(self, valor):
        self._num = int(valor)

    @property
    def den(self):
        return self._den

    @den.setter
    def den(self, valor):
        valor = int(valor)
        if valor == 0:
            raise ZeroDivisionError("el denominador no puede ser 0")
        self._den = valor

    def __str__(self):
        divisor = mcd(self.num, self.den)
        num = self.num // divisor
        den = self.den // divisor
        if den < 0:
            num, den = -num, -den
        return f"{num}/{den}"


f = Fraccion(6, 4)
print(f)                         # 3/2 → reduce sola
try:
    Fraccion(1, 0)
except ZeroDivisionError as e:
    print("ZeroDivisionError:", e)   # el denominador no puede ser 0
```

Este capítulo te dejó la primera gran técnica de ingeniería: sabés **separar el qué del cómo**, esconder el esqueleto con `_`/`__` y proteger el estado con `@property`. Pero hay una pregunta que quedó flotando desde la sección 1: ¿y si pudiéramos **reutilizar** las interfaces y las implementaciones? Las clases `Fotovoltaica` e `Hidroelectrica` comparten la firma de `energia()`, pero la escribieron dos veces. El próximo capítulo de la Parte VI responde con el segundo gran pilar de la POO: **la herencia** — un molde que nace de otro molde, el "es un" del mundo de los objetos, con `super()`, MRO y la composición como contrapeso. Nos vemos en el capítulo 15.