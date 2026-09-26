# Capítulo 15 — La herencia: el molde que nace de otro molde

El capítulo 14 terminó con una pregunta sobre la mesa: las clases `Fotovoltaica` e `Hidroelectrica` compartían la firma de `energia()`, el mismo contrato de `Dispositivo`, pero la escribieron **dos veces**. Copiar y pegar funciona, pero huele mal: si el contrato cambia, hay que acordarse de cambiarlo en todos lados. La pregunta de fondo era: *¿qué pasa si un molde pudiera nacer de otro molde, heredando sus atributos y sus métodos?*

Esa pregunta tiene nombre y apellido en la POO: **la herencia**. Es el segundo pilar de la Parte VI — y el más rebelde de los tres. Porque Python es de los pocos lenguajes "de manual" que además de herencia simple permite **herencia múltiple**: una clase que nace de dos moldes a la vez, como el ornitorrinco que es reptil *y* mamífero. En Java o C# eso no se puede; en Python, el rebelde, sí.

En este capítulo vas a hacer dos movimientos. Primero ves el poder del "es un": `class Perro(Animal)` le da a `Perro` todo lo de `Animal` — y de paso le ponés nombre al fruto más jugoso de la sobreescritura: el **polimorfismo**, el mismo mensaje con respuestas distintas. Después ves el contrapeso: **la composición**, el "tiene un". Porque en la ingeniería real la herencia no es la única manera (ni siempre la mejor) de reutilizar código — y saber cuál elegir es la diferencia entre un diseño que respira y una explosión de clases.

---

## 1. El molde que nace de otro molde

Empezá con lo más simple. Una clase puede heredar de otra escribiendo el nombre de la clase base entre paréntesis:

```python
class Animal:
    """Clase base de todos los animales."""

    def __init__(self, nombre):
        self.nombre = nombre

    def respirar(self):
        return f"{self.nombre} inhala y exhala."


class Perro(Animal):
    """Un perro ES un animal; además, ladra."""

    def ladrar(self):
        return "¡Guau!"


p = Perro("Firuláis")
print(p.respirar())    # Firuláis inhala y exhala.   → viene de Animal
print(p.ladrar())      # ¡Guau!                      → viene de Perro
```

Mirá la magia fina de la primera línea: `Perro("Firuláis")` está llamando al **`__init__` que ni siquiera escribimos**. `Perro` no definió constructor; lo *heredó* de `Animal`. Lo mismo con `respirar()`. Recién `ladrar()` es cosa del perro.

El vocabulario de la familia importa:

- **Superclase** o **clase base**: el molde del que se hereda (`Animal`).
- **Subclase** o **clase derivada**: el molde que hereda (`Perro`).
- Se dice que `Perro` *extiende* o *deriva* de `Animal`, y que `Animal` es *base* de `Perro`.

Y lo más importante, la lectura del mundo real: **una relación de herencia es un "es un"**. Un perro *es* un animal. Por eso tiene sentido que herede lo de animal. Si la relación no es "es un", es probable que estés usando herencia para forzar algo — lo vas a destripar en la sección 11.

> **Dato clave:** la sintaxis es una sola línea: `class SubClase(Superclase):`. Adentro podés agregar atributos y métodos nuevos, y también **sobrescribir** (reescribir) los heredados. No se hereda el *código* de la clase: se hereda un **comportamiento** y una **interfaz**, y la subclase decide qué mantiene, qué agrega y qué cambia.

---

## 2. `issubclass()` y `isinstance()`: las preguntas de la familia

¿Cómo se pregunta si una clase es subclase de otra? Con la función `issubclass()`. ¿Cómo se pregunta si un objeto pertenece a una clase (o a su familia)? Con `isinstance()` — que ya usaste en los capítulos 12 y 13, y que ahora cobra todo su sentido:

```python
print(issubclass(Perro, Animal))   # True  → Perro es subclase de Animal
print(issubclass(Animal, Perro))   # False → Animal NO es subclase de Perro
print(isinstance(p, Perro))        # True  → p es un Perro
print(isinstance(p, Animal))       # True  → y además ES un Animal
```

La última línea es la joya: `p` fue creada con `Perro(...)`, pero `isinstance(p, Animal)` da `True`. No es un disfraz ni una copia: *es* un animal. La herencia no copia código a la fuerza; la subclase **pertenece a la familia** de la superclase. Eso es el **polimorfismo** en su forma más simple: "lo que sirve para un `Animal` sirve para un `Perro`". Lo vas a ver en plena forma en la sección 6.

Y como toda clase, `Perro` también es subclase de `object` — la superclase universal que ya rozamos en el capítulo 12:

```python
print(issubclass(Perro, object))   # True
print(Perro.__bases__)             # (<class '__main__.Animal'>,)
print(p.__class__.__mro__)
# (<class '__main__.Perro'>, <class '__main__.Animal'>, <class 'object'>)
```

`__bases__` te dice la superclase inmediata; `__mro__` (Method Resolution Order) te da **toda la escalera**: Perro, sube a Animal, sube a object. Esta escalera va a ser la estrella de las secciones 7 y 8.

---

## 3. La superclase universal: `object`

Fijate un detalle del código de recién: nadie escribió `class Animal(object)`. Sin embargo, `issubclass(Perro, object)` es `True`. ¿De dónde sale?

Toda clase que creás en Python deriva de `object`, **aunque no lo digas**. Es el molde de todos los moldes, el bisabuelo de todo lo que existe. Así de explícito se ve con un molde al que no le declarás nada:

```python
class MoldeVacio:
    """No declara nada... y sin embargo tiene métodos."""

    pass


vacio = MoldeVacio()
print(MoldeVacio.__bases__)       # (<class 'object'>,)
print(issubclass(MoldeVacio, object))   # True
print(dir(vacio))
```

Ese `dir(vacio)` no está vacío: aparecen `__str__`, `__repr__`, `__eq__`, `__hash__` y todos los métodos especiales que viste en el capítulo 13. **No los escribiste vos: los heredaste de `object`.** Ahí está el origen de la frase "todo en Python es un objeto": hasta la clase más tonta tiene un padre.

Existe una sola excepción a este "todos descienden de `object`": **las excepciones**. Son la excepción a la regla, y se ve con un experimento lindo. Si creás una clase que quiere ser un error *sin* heredar de la familia correcta, Python se enoja:

```python
class MiError:
    """Quiere ser una excepción... pero no hereda de la familia correcta."""

    pass


try:
    raise MiError()
except TypeError as e:
    print(type(e).__name__, ":", e)
    # TypeError : exceptions must derive from BaseException
```

Para que `raise` la acepte como error, tiene que nacer de la familia de las excepciones: `Exception` (o `BaseException`). Ahí la escalera se ve clara con `__mro__`:

```python
class MiError(Exception):
    """Ahora sí: un error de verdad."""

    pass


print(MiError.__mro__)
# (<class '__main__.MiError'>, <class 'Exception'>, <class 'BaseException'>, <class 'object'>)
```

> **Dato clave:** todas las clases son subclases de `object`; las excepciones son la excepción que confirma la regla: deben descender de `BaseException`. Esta es la base del `class MiError(Exception)` que el capítulo 6 te prometió para la Parte VI — y que vas a cerrar en el capítulo 17.

---

## 4. Subclase que suma: heredar todo y agregar

La forma más común de usar herencia no es cambiar lo heredado, sino **sumar**. La clase `Estudiante`, ya pariente de la `Persona` del capítulo 14, hereda todo de `Persona` y agrega la inscripción a materias:

```python
class Persona:
    """Clase base: datos personales (la vimos en el capítulo 14)."""

    def __init__(self, nombre, apellido):
        self.nombre = nombre
        self.apellido = apellido
        self.__clave = (nombre + apellido).casefold()

    @property
    def clave(self):
        """La 'clave escondida' del capítulo 14, por name mangling."""
        return self.__clave

    def saludar(self):
        return f"Hola, soy {self.nombre} {self.apellido}"


class Estudiante(Persona):
    """Hereda todo de Persona y además se inscribe en materias."""

    tira_de_materias = []

    def inscripcion(self, materia):
        self.tira_de_materias.append(materia)
        return self.tira_de_materias


bc2186 = Estudiante("Brenda", "Cynthia")
print(bc2186.saludar())                  # Hola, soy Brenda Cynthia
print(bc2186.clave)                      # brendacynthia → el __init__ de Persona
print('inscripcion' in dir(Estudiante))  # True → el método nuevo
print(Estudiante.__bases__)              # (<class '__main__.Persona'>,)
```

Tres lecturas finas:

- `bc2186` no es un `Persona` "con extras": es por completo un `Estudiante`, y por incluir a `Persona` **por dentro hereda `clave`, `saludar` y hasta el name mangling** del capítulo 14 (si mirás un `dir(bc2186)` aparece `_Persona__clave`: el atributo vive en la clase `Persona`, no en su subclase).
- El `@property clave` se hereda tal cual: la subclase no tiene que hacer nada para exponerlo.
- Ojo con `tira_de_materias = []`: es un **atributo de clase** (capítulo 12: "primero ella, después el molde"). Lo compartimos a propósito entre todos los estudiantes para que veas el efecto en vivo. En código de verdad, una lista por instancia se arma en el `__init__`, como vas a ver en la sección siguiente.

> **Dato clave:** la herencia es *composición por delegación*: el objeto de la subclase "contiene" todo lo de la superclase y le delega lo que no está redefinido. Agregar métodos nuevos es la forma más inocente y más usada de heredar.

---

## 5. Sobrescritura y `super()`: reutilizar la receta del molde

Hasta acá la subclase solo sumaba. Pero a veces el `__init__` del molde no alcanza: el `Estudiante` tiene un dato extra (el género) y necesita su propio constructor. Lo escribimos a mano... y eso **rompe la herencia del `__init__`**: si redefinís `__init__`, el de `Persona` ya no corre solo. Entra en escena la función estrella:

```python
class Estudiante(Persona):
    """Subclase con __init__ propio que reutiliza el de Persona."""

    def __init__(self, nombre, apellido, genero):
        if genero.casefold() in ("masculino", "femenino", "otro"):
            self.genero = genero
        else:
            raise ValueError("género no registrado")
        super().__init__(nombre, apellido)

    def saludar(self):
        return Persona.saludar(self) + f". Género: {self.genero}"


bc471221 = Estudiante("Brenda", "Cynthia", "femenino")
print(bc471221.saludar())    # Hola, soy Brenda Cynthia. Género: femenino
print(bc471221.clave)        # brendacynthia → lo armó el __init__ de Persona
```

`super().__init__(nombre, apellido)` es el puente: **"anda y ejecutá el `__init__` de mi superclase"**. Así el `Estudiante` configura su parte (el género) *y* le paga a `Persona` su parte (nombre, apellido, clave). Sin `super()`, tendrías que reescribir a mano lo que ya escribe el padre — copiar y pegar de vuelta, el olor del capítulo 14.

Y notá el `saludar()` sobrescrito: en vez de tirar a la basura el saludo heredado, lo reutilizó con `Persona.saludar(self)` y le agregó el género. Ese es el patrón completo: **sobrescribir sin perder la receta**.

Esto se ve a escala más grande con una familia de polígonos: cuatro pisos de escalera, y cada subclase agrega o sobrescribe algo:

```python
class Poligono:
    """Base: un polígono con N lados."""

    def __init__(self, lados):
        self.n = lados
        self.lados = [0] * lados

    def ver_lados(self):
        print("Lados:", self.lados)

    @property
    def perimetro(self):
        return sum(self.lados)


class Triangulo(Poligono):
    """Un triángulo ES un polígono de 3 lados."""

    def __init__(self):
        super().__init__(3)

    @property
    def area(self):
        a, b, c = self.lados
        s = (a + b + c) / 2
        return round((s * (s - a) * (s - b) * (s - c)) ** 0.5, 2)


class Rectangulo(Poligono):
    """Un rectángulo ES un polígono de 4 lados."""

    def __init__(self, base, altura):
        super().__init__(4)
        self.base = base
        self.altura = altura
        self.lados = [base, altura, base, altura]

    def ver_lados(self):
        print("Medidas:", self.lados, "(base, altura, base, altura)")

    @property
    def area(self):
        return self.base * self.altura

    @property
    def diagonal(self):
        return round((self.base ** 2 + self.altura ** 2) ** 0.5, 2)


class Cuadrado(Rectangulo):
    """Un cuadrado ES un rectángulo con todos los lados iguales."""

    def __init__(self, lado):
        super().__init__(lado, lado)


tri = Triangulo()
tri.lados = [3, 4, 5]
print(tri.perimetro, tri.area)                  # 12 6.0
cua = Cuadrado(5)
cua.ver_lados()   # Medidas: [5, 5, 5, 5] (base, altura, base, altura)
print(cua.perimetro, cua.area, cua.diagonal)    # 20 25 7.07
print(issubclass(Cuadrado, Rectangulo), issubclass(Cuadrado, Poligono))
# True True
print(Cuadrado.__mro__)
# (<class '__main__.Cuadrado'>, <class '__main__.Rectangulo'>, <class '__main__.Poligono'>, <class 'object'>)
```

Cada nivel de la escalera encadena `super()` al de abajo. `Cuadrado` ni se define completo: le alcanza su `__init__` de un parámetro y **heredó** `perimetro` de `Poligono`, `area` y `diagonal` de `Rectangulo`. Esa es la herencia en su mejor momento: la cadena de "es un" (un cuadrado es un rectángulo es un polígono) que propaga comportamiento sin repetir una sola línea.

> **Dato clave:** `super()` no es magia: es "la clase que sigue en mi MRO". Se usa típicamente en `__init__` y en métodos sobrescritos para reutilizar el trabajo del padre y agregar el propio. Si lo dejás afuera, la subclase le pega a la herencia: el constructor base ya no corre.

---

## 6. Polimorfismo: el mismo mensaje, respuestas distintas

En la sección 5 llamaste a `area` y a `perimetro` sobre `Triangulo`, `Rectangulo` y `Cuadrado`. Los mismos nombres, cálculos distintos — y ninguna línea que le preguntara a cada figura quién era. Le hablaste a *un polígono* y cada molde respondió con lo suyo. Ese poder tiene nombre y apellido en la POO, y es la recompensa de todo lo que armaste desde el `class Perro(Animal)` de la sección 1.

La palabra viene de dos raíces griegas: *poli* (muchos) y *morfo* (forma). El **polimorfismo** es la capacidad de que objetos de clases distintas respondan al mismo mensaje con comportamientos propios. Ya te cruzaste con él en la sección 2, cuando `isinstance(p, Animal)` dio `True`: "lo que sirve para un `Animal` sirve para un `Perro`". Ahora vas a verlo en toda su gloria.

Imaginá un **lienzo de dibujo**: un programa que guarda figuras de colores, sabe calcular su área y sabe dibujarlas. La base declara el comportamiento genérico; cada subclase lo sobrescribe con su versión de la receta:

```python
from math import pi


class Figura:
    """Un lienzo: una figura con color de fondo y de borde."""

    def __init__(self, color_fondo, color_borde):
        self.color_fondo = color_fondo
        self.color_borde = color_borde

    def area(self):
        return "no definida"

    def dibujar(self):
        print("Dibujando una figura genérica.")


class Rectangulo(Figura):
    def __init__(self, color_fondo, color_borde, ancho, alto):
        super().__init__(color_fondo, color_borde)
        self.ancho = ancho
        self.alto = alto

    def area(self):
        return self.ancho * self.alto

    def dibujar(self):
        print("Dibujando un rectángulo.")


class Circulo(Figura):
    def __init__(self, color_fondo, color_borde, radio):
        super().__init__(color_fondo, color_borde)
        self.radio = radio

    def area(self):
        return round(pi * self.radio ** 2, 2)

    def dibujar(self):
        print("Dibujando un círculo.")


class Triangulo(Figura):
    def __init__(self, color_fondo, color_borde, base, altura):
        super().__init__(color_fondo, color_borde)
        self.base = base
        self.altura = altura

    def area(self):
        return self.base * self.altura / 2

    def dibujar(self):
        print("Dibujando un triángulo.")
```

Ahora la parte linda. Meté las tres figuras en una sola lista y pedile lo mismo a cada una — un solo `for`, tres comportamientos:

```python
lienzo = [Rectangulo("rojo", "negro", 5, 10),
          Circulo("verde", "azul", 5),
          Triangulo("azul", "amarillo", 7, 13)]

for f in lienzo:
    print(f"{type(f).__name__} → área: {f.area()}")
    f.dibujar()
# Rectangulo → área: 50
# Dibujando un rectángulo.
# Circulo → área: 78.54
# Dibujando un círculo.
# Triangulo → área: 45.5
# Dibujando un triángulo.
```

Fijate lo que *no* hace el `for`: no hay `if` por tipo, no hay `isinstance`, no hay una cadena de `elif`. El mensaje `area()` viaja parejo para todas las figuras y cada una responde con la suya. Eso es el polimorfismo el día después del examen.

Y acá está el truco que lo vuelve valioso: **abierto para crecer, cerrado para cambiar**. Si mañana llega una `Romboide` nueva al catálogo, la definís con su `area()` y su `dibujar()`, la metés a la lista — y el `for` anda sin tocarle una coma:

```python
class Romboide(Figura):
    def __init__(self, color_fondo, color_borde, base, altura):
        super().__init__(color_fondo, color_borde)
        self.base = base
        self.altura = altura

    def area(self):
        return self.base * self.altura

    def dibujar(self):
        print("Dibujando un romboide.")


lienzo.append(Romboide("negro", "blanco", 5, 7))
print(f"{type(lienzo[-1]).__name__} → área: {lienzo[-1].area()}")
lienzo[-1].dibujar()
# Romboide → área: 35
# Dibujando un romboide.
```

Eso es polimorfismo **por herencia**: el "es un" garantiza que el mensaje existe, y la sobreescritura decide quién responde. En Python hay una segunda versión, más rebelde todavía, donde ni siquiera hace falta heredar: alcanza con que el objeto tenga el método. Se llama **duck typing** — el tipado del pato — y es la protagonista del capítulo 16.

> **Dato clave:** el polimorfismo es lo que hace que valga la pena el "es un". Escribís el código una vez contra la clase base y atiende a todas las subclases, pasadas y futuras — cada una responde el mismo mensaje a su manera, y sumar una figura nueva no toca ni una línea de lo que ya escribiste.

Y ya que el polimorfismo reparte el mensaje según el **MRO** — el orden de búsqueda que viste con `__mro__` en la sección 2 —, la pregunta siguiente es inevitable: ¿y si un molde pudiera nacer de **dos** a la vez? Esa es la rebeldía que hace único a Python.

---

## 7. Herencia múltiple: `class Ornitorrinco(Reptil, Mamifero)`

Acá está lo que hace único a Python entre los lenguajes clásicos. La sintaxis admite **más de una superclase**: se listan entre paréntesis separadas por comas. Y para contarlo no hay mejor animal que la personificación de la herencia múltiple: el ornitorrinco, reptil *y* mamífero a la vez.

```python
class Animal:
    """Clase base de todos los animales."""

    def __init__(self, nombre):
        self.nombre = nombre
        print(f"Hola. Mi nombre es {self.nombre}.")

    def reproduccion(self):
        """Solo define una interfaz: la implementación es de cada subclase."""

    def __del__(self):
        print(f"El animal {self.nombre} acaba de fallecer.")


class Mamifero(Animal):
    """Actividades de los mamíferos."""

    def reproduccion(self):
        print("Toma un cachorro.")

    def amamanta(self):
        print("Toma un vaso de leche.")


class Reptil(Animal):
    """Actividades de los reptiles."""

    venenoso = True

    def reproduccion(self):
        print("Toma un huevo.")

    def veneno(self):
        print("Estás envenenado." if self.venenoso else "No soy venenoso.")


class Ornitorrinco(Reptil, Mamifero):
    """Los ornitorrincos son animales muy raros."""

    def __init__(self, nombre):
        super().__init__(nombre)
        print("¿Pero qué es esto?")


perry = Ornitorrinco("Agente P")
# Hola. Mi nombre es Agente P.
# ¿Pero qué es esto?
perry.reproduccion()     # Toma un huevo.      → el método de Reptil
perry.veneno()           # Estás envenenado.   → el método de Reptil
perry.amamanta()         # Toma un vaso de leche. → el método de Mamifero
del perry                # El animal Agente P acaba de fallecer.
```

El ornitorrinco es `Reptil` *y* `Mamifero` (y los dos, `Animal`). Hereda las dos vidas: la de reptil (veneno, huevos) y la de mamífero (amamantar). Y fijate el conflicto que era inevitable: `Animal` definió `reproduccion()` y **las dos** ramas la sobrescribieron. ¿Cuál gana para `perry`?

**La primera que se escriba en la lista.** `class Ornitorrinco(Reptil, Mamifero)` puso a `Reptil` primero, y ganó su `reproduccion()`: "Toma un huevo". Si hubiera escrito `class Ornitorrinco(Mamifero, Reptil)`, ganaría el cachorro. Esa regla de "el primero gana" es la puerta al concepto más fino de la herencia múltiple: **el MRO**.

---

## 8. El MRO: el orden de la búsqueda

`__mro__` (Method Resolution Order) es la escalera completa que Python arma para **saber en qué orden buscar** cuando un método o atributo puede existir en varios pisos. Para el ornitorrinco quedó así:

```python
print(Ornitorrinco.__mro__)
# (<class '__main__.Ornitorrinco'>, <class '__main__.Reptil'>, <class '__main__.Mamifero'>, <class '__main__.Animal'>, <class 'object'>)
```

Leelo como "primero el piso más bajo (el ornitorrinco, que es lo más específico), y desde ahí para arriba hasta `object`". Todo lo que hace `perry.reproduccion()` se resuelve buscando **en este orden**: Ornitorrinco no lo define → saca de Reptil (primer superclase) → listo, encontró. Ese es el "primero gana" mezclado con "sube primero".

¿Y el cerebro de todas las consultas familiares?

```python
print(issubclass(Ornitorrinco, Reptil))     # True
print(issubclass(Ornitorrinco, Mamifero))   # True
print(issubclass(Ornitorrinco, Animal))     # True
```

El detalle fino es que `object` aparece **una sola vez** en el MRO, aunque las dos ramas desciendan de él. Python usa un algoritmo llamado **C3 linearization** (el algoritmo de linealización C3) que garantiza dos cosas sobre la escalera: (1) cada clase aparece una vez, y (2) si una clase tiene una relación de herencia con otra, respeta ese orden. Lo ves con un caso más enredado: un humano que también es superrobot.

```python
class Terricola:
    """Todos los seres de la Tierra."""

    def __init__(self):
        pass


class Humano(Terricola):
    """Un terricola que piensa y siente."""

    def __init__(self, nombre, apellido, genero):
        self.nombre = nombre
        self.apellido = apellido
        self.genero = genero
        self.habilitado = True


class Robot:
    """Una máquina."""

    def __init__(self):
        pass


class SuperRobot(Robot):
    """Un robot mejorado."""

    def __init__(self):
        pass


class Programador(Humano, SuperRobot):
    """Mitad humano, mitad superrobot... y escribe código."""

    def __init__(self, nombre, apellido, genero, rol):
        Humano.__init__(self, nombre, apellido, genero)
        self.rol = rol
        self.lenguajesconocidos = ["PHP", "JavaScript", "Python"]


print(Programador.__mro__)
# (<class '__main__.Programador'>, <class '__main__.Humano'>, <class '__main__.Terricola'>, <class '__main__.SuperRobot'>, <class '__main__.Robot'>, <class 'object'>)
```

Fijate lo que hizo el algoritmo: `Programador`, después `Humano` y su rama (`Terricola`), después `SuperRobot` y su rama (`Robot`), y al final `object` **una vez**. La escalera respetó el "es un" de cada rama sin duplicar a nadie.

> **Dato clave:** el MRO es el orden de búsqueda de métodos y atributos. La regla fácil: "primero el piso más específico, después la superclase 1 con toda su rama, después la superclase 2 con la suya, y `object` al final, una sola vez". `Clase.__mro__` te lo imprime para consultarlo cuando lo dudes.

---

## 9. El `super()` cooperativo: la cadena que no se rompe

Preparate, porque acá se sube la apuesta. En el ejemplo del ornitorrinco, `super().__init__(nombre)` de la subclase llamó al `__init__` de... ¿quién? Según el MRO, el siguiente después de `Ornitorrinco` es `Reptil`. Pero `Reptil` **no definió `__init__`**: entonces Python sigue subiendo por la escalera (Reptil → Mamifero → Animal) hasta que encuentra `Animal.__init__`, lo ejecuta, y recién ahí vuelve. Ese es el **`super()` cooperativo**: la llamada no va al "padre" sino **al siguiente en el MRO**, y el MRO es cooperación pura.

El experimento definitivo es una familia de clases donde cada una suma a su propio contador:

```python
class Base:
    a = 0

    def met(self):
        print("metodo Base", self.a)
        self.a += 1


class Padre(Base):
    b = 0

    def met(self):
        super().met()
        print("metodo Padre", self.b)
        self.b += 1


class Madre(Base):
    c = 0

    def met(self):
        super().met()
        print("metodo Madre", self.c)
        self.c += 1


class Hijo(Padre, Madre):
    d = 0

    def met(self):
        super().met()
        print("metodo Hijo", self.d)
        self.d += 1


h = Hijo()
h.met()
```

El resultado, si todo cooperara, sería un desfile en orden de MRO: `Hijo`, `Padre`, `Madre`, `Base`. Y como cada `super()` salta al siguiente de la escalera, el desfile se arma **de abajo hacia arriba en la llamada, pero la ejecución baja**:

```
metodo Base 0
metodo Madre 0
metodo Padre 0
metodo Hijo 0
```

¿Notaste? `metodo Madre` corre **antes** que `metodo Padre`, aunque el MRO sea Hijo → Padre → Madre. Porque el MRO manda: `Hijo.super()` → `Padre.super()` → `Madre.super()` → `Base`, y recién al llegar al fondo empiezan a imprimirse los "metodo X" en el camino de regreso. Ese es `super()` cooperativo de verdad: **cadena completa, no salto al padre directo**.

Y acá está la versión *no* cooperativa, donde cada piso llama al padre por su nombre a mano (`Base.met(self)`):

```python
class BaseM:
    a = 0

    def met(self):
        print("metodo base", self.a)
        self.a += 1


class PadreM(BaseM):
    b = 0

    def met(self):
        BaseM.met(self)          # llama 'a mano' a Base
        print("metodo padre", self.b)
        self.b += 1


class MadreM(BaseM):
    c = 0

    def met(self):
        BaseM.met(self)          # llama 'a mano' a Base
        print("metodo madre", self.c)
        self.c += 1


class HijoM(PadreM, MadreM):
    d = 0

    def met(self):
        PadreM.met(self)         # llama 'a mano' a Padre
        MadreM.met(self)         # llama 'a mano' a Madre
        print("metodo hijo", self.d)
        self.d += 1


h = HijoM()
h.met()
```

La salida delata el problema:

```
metodo base 0
metodo padre 0
metodo base 1
metodo madre 0
metodo hijo 0
```

`metodo base` corrió **dos veces** (una por cada rama), y el orden real no respeta la escalera. La llamada a mano duplica trabajo y rompe la lógica; `super()` se encarga de repartir una sola vez. Por eso, en herencia múltiple, **siempre `super()`, nunca `Clase.metodo(self)` a mano** — salvo que sepas exactamente qué estás haciendo, como el `Humano.__init__(self, ...)` del `Programador` (que ahí sí fue intencional: solo quiere la rama humana).

Para cerrar el tema, la cadena cooperativa armando un texto, donde se puede ver el orden de los `__init__` de una forma casi física:

```python
class Uno:
    def __init__(self):
        super().__init__()
        self.t += "1"


class Dos:
    def __init__(self):
        super().__init__()
        self.t = "2"


class Tres(Uno, Dos):
    def __init__(self):
        super().__init__()
        self.t += "3"
        print(self.t)


Tres()    # 213
```

Fijate la coreografía del super cooperativo: el MRO es Tres → Uno → Dos. La cadena baja `Tres.super()` → `Uno.super()` → `Dos`, que ejecuta primero (`self.t = "2"`), luego vuelve a `Uno` (le agrega el `"1"`: queda `"21"`), y recién después `Tres` le suma el `"3"`: **`213`**. La ejecución de los `__init__` va **en contra** del orden de declaración de las superclases.

> **Dato clave:** `super()` no llama al padre: llama al *siguiente en el MRO*. Con herencia múltiple, la cadena de `super()` recorre la escalera completa exactamente una vez, y los constructores se ejecutan del más lejano al más cercano. Por eso siempre cooperativo: las llamadas "a mano" (`Base.met(self)`) pueden repetir trabajo.

---

## 10. RadioReloj: un caso real de herencia multifunción

Para que no quede en la teoría, un ejemplo de la vida real: el **radio-reloj**, un aparato que es radio *y* reloj a la vez. Dos clases independientes (`Radio` y `Reloj`) y una tercera que las fusiona:

```python
import datetime


class Reloj:
    """Marca la hora actual."""

    def __init__(self, hora=None):
        self.hora = hora or datetime.datetime.now().strftime("%H:%M:%S")

    def mostrar(self):
        return f"Son las {self.hora}"


class Radio:
    """Sintoniza estaciones AM/FM."""

    def __init__(self, freq="AM", sinton=590):
        self.freq = freq.upper()
        self.sintonia = sinton

    def mostrar(self):
        return f"{self.sintonia} kHz en {self.freq}"

    def sintonizar(self, freq, valor):
        self.freq = freq.upper()
        self.sintonia = valor


class RadioReloj(Radio, Reloj):
    """Un radio-reloj: hereda la radio y el reloj a la vez."""

    def __init__(self, freq="FM", sinton=88.5, hora=None):
        Radio.__init__(self, freq, sinton)
        Reloj.__init__(self, hora)

    def mostrar(self):
        return f"{Radio.mostrar(self)} y {Reloj.mostrar(self)}"


emi = RadioReloj("FM", 106.3, "07:45:30")
print(emi.mostrar())          # 106.3 kHz en FM y Son las 07:45:30
emi.sintonizar("AM", 1030)
print(emi.mostrar())          # 1030 kHz en AM y Son las 07:45:30
print(RadioReloj.__mro__)
# (<class '__main__.RadioReloj'>, <class '__main__.Radio'>, <class '__main__.Reloj'>, <class 'object'>)
```

Lo valioso del ejemplo:

- `RadioReloj` es **a la vez** una radio (usa `freq`, la frecuencia, y `sintonia`) y un reloj (usa `hora`). Los dos `__init__` corren, porque los llamó explícitamente a los dos (aquí la llamada a mano es intencional: querés *ambas* ramas, no el MRO).
- `Radio.mostrar()` y `Reloj.mostrar()` definen **el mismo método con nombres iguales**. Para combinarlos, `RadioReloj` define el suyo y llama a los otros **con el objeto como primer argumento**: `Radio.mostrar(self)`. Es la misma técnica de "método = función + objeto" que viste en el capítulo 12.
- Si `RadioReloj` no definiera `mostrar()`, ganaría el de `Radio` por MRO (primera superclase). Al definirlo, la subclase *siempre* tiene prioridad.

> **Dato clave:** la herencia múltiple es para combinar **comportamientos independientes** (una radio Y un reloj). Cuando las dos bases declaran el mismo método, la subclase decide cómo mezclarlos — o deja que el MRO elija por ella.

---

## 11. La composición: "tiene un" en vez de "es un"

Y ahora el contrapeso, la otra mitad del título de este capítulo. La herencia modela el "es un": un perro *es* un animal. Pero hay relaciones que **no** son de "es un" sino de **"tiene un"**: un auto *tiene* ruedas. Un motor no es un auto; el auto *tiene* un motor. Forzar esa relación con herencia sería un desastre — un `Auto` no debe heredar de `Rueda`:

```python
class Rueda:
    """Un componente: algo que gira."""

    def girar(self):
        return "girando"


class Auto:
    """Un todo: compuesto por Ruedas."""

    def __init__(self):
        self.ruedas = [Rueda() for _ in range(4)]

    def avanzar(self):
        return " ".join(r.girar() for r in self.ruedas)


familiar = Auto()
print(familiar.avanzar())   # girando girando girando girando
```

`Auto` no hereda nada: **guarda** cuatro `Rueda` dentro y les delega el trabajo. Eso es la **composición**: crear tipos complejos combinando objetos de otros tipos. Reutilizás código *agregando* objetos a otros objetos, en vez de *heredar* la interfaz y la implementación. Y tiene una ventaja enorme: podés cambiar las piezas en el momento, sin tener que crear jerarquías infinitas.

La definición cortita queda así: la herencia modela una relación *es un* (`Hija` es versión especializada de `Base`); la composición modela una relación *tiene un* (`Composite` tiene un/a `Component`).

El ejemplo de la nómina lo hace tangible. Los empleados heredan de una base común (el "es un empleado"), pero **la nómina** no hereda de nadie: contiene una lista de empleados y les pregunta el cálculo. Es composición pura:

```python
class Empleado:
    """Base: todos los empleados comparten id y nombre."""

    def __init__(self, id, nombre):
        self.id = id
        self.nombre = nombre

    def calculo(self):
        return "No implementado"


class EmpleadoMensual(Empleado):
    """Empleado con sueldo fijo."""

    def __init__(self, id, nombre, salario):
        super().__init__(id, nombre)
        self.salario = salario

    def calculo(self):
        return self.salario


class EmpleadoPorHoras(Empleado):
    """Empleado que cobra por hora trabajada."""

    def __init__(self, id, nombre, horas, valor_hora):
        super().__init__(id, nombre)
        self.horas = horas
        self.valor_hora = valor_hora

    def calculo(self):
        return self.horas * self.valor_hora


class Nomina:
    """Composición: contiene empleados y les pregunta su cálculo."""

    def mostrar(self, empleados):
        for emp in empleados:
            print(f"{emp.id} - {emp.nombre}: {emp.calculo()}")


nomina = Nomina()
nomina.mostrar([EmpleadoMensual(1, "Ana", 45000),
                EmpleadoPorHoras(2, "Luis", 40, 500),
                Empleado(3, "Ernesto")])
# 1 - Ana: 45000         → heredó calculo() de EmpleadoMensual
# 2 - Luis: 20000        → heredó calculo() de EmpleadoPorHoras (40 × 500)
# 3 - Ernesto: No implementado   → el base sin implementar
```

Fijate el tercer empleado: `Empleado(3, "Ernesto")` no tiene `calculo()` implementado, y `Nomina` no se entera. La intención es que un `calculo()` sin implementar reviente con una excepción, pero no: silenciosamente imprime el texto comodín. Esa flojera del ejemplo es justamente la puerta del capítulo 16: cuando el contrato es tan importante, se usa una **clase base abstracta** que *obliga* a implementar `calculo()`. Por ahora te quedás con la idea de la explosión de clases: cada rol nuevo (gerente, secretaria, vendedor, trabajador de fábrica) hacía falta una clase nueva que combinara "cómo cobra" con "qué hace en el trabajo". **Ese es el virus de la herencia mal usada.** La solución del diseño moderno es mezclar roles por composición — y un adelanto de eso es que `Nomina` no le pregunta a nadie *quién sos*, solo *cuánto calculás*:

```python
class Externo:
    """Un externo NO hereda de Empleado... pero tiene calculo() igual."""

    def __init__(self, id, nombre, horas, valor_hora):
        self.id = id
        self.nombre = nombre
        self.horas = horas
        self.valor_hora = valor_hora

    def calculo(self):
        return self.horas * self.valor_hora


externo = Externo(1001, "Marcos", 10, 600)
nomina.mostrar([externo])    # 1001 - Marcos: 6000
```

¿Ves? `Externo` no es `Empleado` y sin embargo `Nomina` lo procesó sin protestar. Esto es **duck typing** (el tipado del pato, el adelanto que quedó pendiente en el capítulo 12): si camina como un empleado y calcula como un empleado... para `Nomina` es un empleado. Ese es el personaje principal del capítulo 16.

> **Dato clave:** herencia = "es un" (heredás la interfaz y la implementación). Composición = "tiene un" (guardás objetos dentro y les delegás). La composición gana casi siempre, porque es flexible, cambiable en caliente y no explota cuando los requisitos crecen.

---

## 12. Herencia o composición: la decisión

Entonces, ¿cuándo se usa cada una? La guía cortita, la que se repite en infinidad de libros:

| Pregunta | Herencia | Composición |
|---|---|---|
| ¿Es la relación un **"es un"** real? | Sí: Perro es Animal, Cuadrado es Rectángulo | No |
| ¿Es un **"tiene un"**? | Forzado: Auto "es un" Rueda es un crimen | Sí: Auto tiene un Motor, Nomina tiene Empleados |
| ¿Necesito reutilizar la *interfaz*? | Sí: heredar el contrato (métodos/promesa) | Delegás manualmente método por método |
| ¿Necesito reutilizar la *implementación*? | Sí: `super()` reutiliza el código del molde | Guardás el objeto y llamás sus métodos |
| ¿El comportamiento va a cambiar en caliente? | Difícil: cambiar de clase = recrear objeto | Fácil: `auto.ruedas[0] = RuedaNueva()` |
| ¿Crecen los roles/requisitos? | Explosión de clases (gerente, secretaria...) | Agregás un componente nuevo y listo |
| Jerarquía profunda y estable | Natural | Obligás de más |

Las reglas del pragmatismo:

1. **Preferí la composición** como primera opción. Casi siempre resuelve el problema con menos clases y más flexibilidad. Es el consejo clásico: "favorecé la composición sobre la herencia".
2. **Usá herencia cuando sea un "es un" genuino** y quieras reutilizar la interfaz: `Perro` es `Animal`, `Triangulo` es `Poligono`. Ahí la herencia es la herramienta natural y el polimorfismo te hace el trabajo gratis.
3. **Usá herencia (múltiple o simple) para mezclar comportamientos independientes**: `RadioReloj` es radio y reloj. Pero con `super()` cooperativo y ojo al MRO.
4. **El "es un" falso es el síntoma del mal uso.** Si dudás entre "es un" y "tiene un", la respuesta casi siempre es "tiene un".
5. **`isinstance()` es el detector de la decisión**: si algún día necesitás `isinstance(obj, ...)` para decidir qué hacer, tu diseño está oliendo a herencia para rato — y por eso existe el duck typing del capítulo 16 para zafar de eso.

---

## 13. Resumen y conceptos clave

Este capítulo fue la segunda pata de la POO: el **"es un"**. Aprendiste que una clase puede nacer de otra (`class Perro(Animal)`) heredando sus métodos, sus atributos y su constructor, y que la subclase puede **agregar** cosas nuevas, **sobrescribir** las heredadas y **reutilizar** la receta del molde con `super()`. A la sobreescritura le pusiste nombre grande: el **polimorfismo**, el mismo mensaje con respuestas distintas — viste el lienzo de figuras donde un solo `for` calcula el área y dibuja a cada una a su manera, y cómo agregar una figura nueva no toca al que recorre la lista. Viste que toda clase desciende de la superclase universal `object` (por eso `issubclass(Perro, object)` da `True`, y por eso `dir()` de una clase vacía está lleno de dunders del capítulo 13) — con las excepciones como excepción: tienen que derivar de `BaseException`. Descubriste que Python permite la herencia múltiple (`class Ornitorrinco(Reptil, Mamifero)`), rara entre los lenguajes, y que el **MRO** (`__mro__`) decide el orden de la búsqueda con la regla "la primera superclase gana, `object` al final una sola vez". Entendiste que `super()` es el siguiente de la escalera, no el padre, y que su uso **cooperativo** recorre el MRO una sola vez (el experimento de los contadores y el divino `213`). Y cerraste con el contrapeso: la **composición**, el "tiene un", que agrega objetos en vez de heredar — `Auto` tiene cuatro `Rueda`, `Nomina` guarda empleados — y que casi siempre es la mejor opción de diseño.

Repasá el checklist antes de seguir:

- [ ] **Herencia**: `class Sub(Super):` — la subclase hereda métodos, atributos y `__init__` de la superclase (el "es un").
- [ ] **Superclase / subclase / clases base y derivadas**: el vocabulario de la familia; `__bases__` te da la superclase inmediata.
- [ ] **`issubclass(A, B)`** pregunta si `A` es subclase de `B`; **`isinstance(x, B)`** pregunta si `x` es de la familia de `B` (aunque se haya creado con otra clase).
- [ ] **`object`** es la superclase universal: toda clase desciende de él aunque no lo diga; de ahí vienen los dunders del capítulo 13.
- [ ] Las **excepciones** son la excepción: para usar `raise MiError()`, `MiError` debe heredar de `Exception`/`BaseException` (promesa del capítulo 17).
- [ ] **Sobrescritura**: redefinís un método heredado en la subclase; ganás vos (el piso más bajo del MRO manda).
- [ ] **Polimorfismo**: objetos de clases distintas responden al mismo mensaje con comportamientos propios — un `for` contra la clase base atiende a toda la familia, y cada una responde a su manera.
- [ ] **`super()`** = "el siguiente en mi MRO". En `__init__` y en métodos sobrescritos reutiliza la receta del molde (`super().__init__(...)`).
- [ ] **Herencia múltiple**: `class Hijo(Padre, Madre)` — el **primero de la lista gana** en conflictos (perry elige "huevo" a "cachorro").
- [ ] **MRO** (`__mro__`): orden de búsqueda, C3 linearization; cada clase una vez; `object` al final. Consultalo con `print(Clase.__mro__)`.
- [ ] **`super()` cooperativo**: la cadena salta al siguiente del MRO, no al padre directo; los `__init__` corren del más lejano al más cercano (ej. `213`); evita duplicar trabajo.
- [ ] En herencia múltiple, **preferí `super()`** a las llamadas a mano (`Base.met(self)`) — salvo que quieras deliberadamente una sola rama.
- [ ] **Composición**: "tiene un" — guardás objetos dentro y delegás (`Auto` tiene `Rueda`; `Nomina` tiene empleados). 
- [ ] **Favorecé la composición sobre la herencia**: casi siempre es más flexible y no explota en clases cuando crecen los requisitos.
- [ ] Regla de oro: si no hay un "es un" real detrás, casi seguro querés composición.

---

## 14. Ejercicios

1. **El primer molde**: clase `Animal` con `__init__(nombre)`, `comer()` (" está comiendo."), y `Perro(Animal)` con `ladrar()` ("¡Guau!"). Instanciá, llamá a ambos métodos y mostrá con `isinstance` que el perro también es un `Animal`.
2. **La familia completa**: tres clases `Persona` → `Docente` y `Alumno` (las dos subclases). Mostrá los `issubclass` en las tres direcciones (verdadero y falso), el `issubclass(Alumno, object)`, y `isinstance` de una instancia de `Alumno` respecto a `Persona`.
3. **`super()` al nacer**: `Empleado` con `__init__(nombre)` y `Programador(Empleado)` con `__init__(nombre, lenguaje)` que haga `super().__init__(nombre)` y guarde `lenguaje`. Sumá `presentarse()` (" escribe en ") y demostrá.
4. **Sobrescribir sin perder la receta**: clase `Base` con `metodo()` que devuelve `"base"`, y `Hija(Base)` que sobrescribe `metodo()` devolviendo `super().metodo() + " y hija"`. Mostrá el resultado.
5. **Herencia múltiple y el orden**: clases `Animal`, `Perro(Animal)`, `Robot`, y `PerroRobot(Perro, Robot)`. Imprimí su `__mro__` y un `issubclass` de las tres ramas.
6. **Mini radio-reloj**: creá un `RadioReloj(Radio, Hora)` con los `__init__` de las dos bases llamados a mano y un `mostrar()` que combine ambos; probalo con `("FM", 100.1)` y hora `"07:00"`.
7. **Composición pura**: clase `Motor` con `encender()` ("run run") y `Auto` que *tiene* un `Motor`; método `arrancar()` que delega. Sin ninguna herencia.
8. **Duck typing en acción**: `Peaton.moverse()` → "caminando" y `Ciclista.moverse()` → "pedaleando"; escribí una función `viajar(quien)` que llame `.moverse()` y pasale un `Peaton` y un `Ciclista` (adelanto del capítulo 16).
9. **El mismo mensaje, muchas respuestas**: clase `Instrumento` con `tocar()` que devuelve "Sonando..."; `Guitarra` y `Piano` que lo sobrescriban ("Rasgueo de guitarra" y "Notas de piano"). Guardá uno de cada en una lista y ejecutá `tocar()` sobre todos en un solo `for`. Sin `if`, sin `isinstance`: puro polimorfismo.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. El primer molde
class Animal:
    """Clase base: un animal con nombre."""

    def __init__(self, nombre):
        self.nombre = nombre

    def comer(self):
        return f"{self.nombre} está comiendo."


class Perro(Animal):
    """Un perro ES un animal; además ladra."""

    def ladrar(self):
        return f"{self.nombre} dice: ¡Guau!"


roco = Perro("Roco")
print(roco.comer())               # Roco está comiendo.
print(roco.ladrar())              # Roco dice: ¡Guau!
print(isinstance(roco, Animal))   # True

# 2. La familia completa
class Persona:
    """Base de las personas."""

    pass


class Docente(Persona):
    """Un docente ES una persona."""

    pass


class Alumno(Persona):
    """Un alumno ES una persona."""

    pass


print(issubclass(Docente, Persona))    # True
print(issubclass(Persona, Alumno))     # False
print(issubclass(Alumno, object))      # True
a = Alumno()
print(isinstance(a, Persona))          # True

# 3. super() al nacer
class Empleado:
    """Base: todo empleado tiene nombre."""

    def __init__(self, nombre):
        self.nombre = nombre


class Programador(Empleado):
    """Un programador ES un empleado, con lenguaje favorito."""

    def __init__(self, nombre, lenguaje):
        super().__init__(nombre)
        self.lenguaje = lenguaje

    def presentarse(self):
        return f"{self.nombre} escribe en {self.lenguaje}"


p = Programador("Dami", "Python")
print(p.presentarse())    # Dami escribe en Python

# 4. Sobrescribir sin perder la receta
class Base:
    def metodo(self):
        return "base"


class Hija(Base):
    def metodo(self):
        return super().metodo() + " y hija"


h = Hija()
print(h.metodo())    # base y hija

# 5. Herencia múltiple y el orden
class Animal:
    pass


class Perro(Animal):
    pass


class Robot:
    pass


class PerroRobot(Perro, Robot):
    """Un perro que además es un robot."""

    pass


print(PerroRobot.__mro__)
# (<class '__main__.PerroRobot'>, <class '__main__.Perro'>, <class '__main__.Animal'>, <class '__main__.Robot'>, <class 'object'>)
print(issubclass(PerroRobot, Animal))    # True
print(issubclass(PerroRobot, Robot))     # True
print(issubclass(PerroRobot, object))    # True

# 6. Mini radio-reloj
class Hora:
    """Marca una hora dada."""

    def __init__(self, hora):
        self.hora = hora

    def mostrar(self):
        return f"Son las {self.hora}"


class Radio:
    """Sintoniza AM/FM."""

    def __init__(self, freq="AM", sinton=590):
        self.freq = freq.upper()
        self.sintonia = sinton

    def mostrar(self):
        return f"{self.sintonia} kHz en {self.freq}"


class RadioReloj(Radio, Hora):
    """Radio y hora a la vez."""

    def __init__(self, hora, freq="AM", sinton=590):
        Radio.__init__(self, freq, sinton)
        Hora.__init__(self, hora)

    def mostrar(self):
        return f"{Radio.mostrar(self)} y {Hora.mostrar(self)}"


a = RadioReloj("07:00", "FM", 100.1)
print(a.mostrar())    # 100.1 kHz en FM y Son las 07:00

# 7. Composición pura
class Motor:
    """Un componente: hace fuerza."""

    def encender(self):
        return "run run"


class Auto:
    """Un todo: TIENE un motor (composición)."""

    def __init__(self, motor):
        self.motor = motor

    def arrancar(self):
        return self.motor.encender()


m = Motor()
auto = Auto(m)
print(auto.arrancar())    # run run

# 8. Duck typing en acción
class Peaton:
    def moverse(self):
        return "caminando"


class Ciclista:
    def moverse(self):
        return "pedaleando"


def viajar(quien):
    return quien.moverse()


print(viajar(Peaton()))     # caminando
print(viajar(Ciclista()))   # pedaleando

# 9. El mismo mensaje, muchas respuestas
class Instrumento:
    def tocar(self):
        return "Sonando..."


class Guitarra(Instrumento):
    def tocar(self):
        return "Rasgueo de guitarra"


class Piano(Instrumento):
    def tocar(self):
        return "Notas de piano"


orquesta = [Instrumento(), Guitarra(), Piano()]
for i in orquesta:
    print(f"{type(i).__name__} → {i.tocar()}")
# Instrumento → Sonando...
# Guitarra → Rasgueo de guitarra
# Piano → Notas de piano
```

Este capítulo te dejó el "es un" y su contrapeso, el "tiene un". Con la herencia reutilizás interfaces y reutilizás implementación; con la composición armás sistemas flexibles que no explotan cuando crecen. Pero hay una pregunta flotando desde el `Empleado(3, "Ernesto")` de la nómina: si todas las subclases *deben* implementar `calculo()`, ¿cómo hacés que el molde **obligue** en vez de devolver un texto comodín? Y otra, más juguetona: si `Nomina` aceptó al `Externo` que no hereda nada ("si camina como empleado y calcula como empleado..."), ¿para qué sirve la herencia entonces? El próximo capítulo de la Parte VI responde con **duck typing, clases base abstractas y mixins** (piezas que se ensamblan): el "es un pato" del capítulo 16. Nos vemos ahí.