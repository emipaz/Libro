# Capítulo 26 — La tabla hash: el corazón del diccionario

Venis usando el diccionario desde el capítulo 5 como quien respira: `patron["Alma"]`, `"Alma" in patron`, `{clave: valor}`. Rápido, limpio, sin arrugas. Pero en el capítulo 23 apareció una pista incómoda: el `GrafoProtegido` valida los vértices llamando a `hash()`, "la operación exacta que un diccionario va a necesitar". ¿Qué operación es esa? ¿Por qué un diccionario *necesita* algo que nunca le pediste? Hoy llega el momento que el capítulo 5 te anticipó ("objetos personalizados **hashables**… herramientas poderosas") y que el capítulo 19 te prometió de palabra ("la **tabla hash**, el corazón invisible del `dict` y del `set`… de adentro hacia afuera"). Es la última estructura de la Parte VIII: la que explica el porqué de todas las demás.

La promesa de hoy tiene tres escalones. Primero vas a abrir el diccionario por dentro: vas a construir tu propia **tabla hash** a mano — un arreglo donde cada clave vive en una posición calculada, no buscada — y la vas a ver devolver valores al instante mientras una lista de los mismos datos se arrastra. Segundo, vas a entender por qué **`hash()`** es la puerta de entrada: qué le pasa a tu objeto cuando lo ponen de clave, y qué regla sagrada tiene que respetar (si lo rompés, rompés tu diccionario). Y tercero, la promesa original: vas a hacer que **tus propias clases sean hashables** — primero la `Persona` simple que se porta bien, después la **a prueba de balas** que ni siquiera se deja mutar, y al final la versión de producción con `@dataclass(frozen=True)` del capítulo 13. Al terminar, vas a mirar un diccionario y a ver lo que es: un arreglo con una función matemática encima.

---

## 1. El `in` que no recorre nada

Había una vez una lista de 30.000 socios. `"Ana" in lista` recorre los 30.000 hasta encontrarla, y el capítulo 19 te enseñó a medir eso: en el peor caso es `O(n)`. El diccionario hace la misma pregunta con la misma data y… no camina. Parece magia, y por eso el capítulo 19 la dejó como la última de las estructuras: porque antes de destriparla tenías que entender qué es `O(1)` buscado. Cronometremos la magia para ver su precio:

```python
import time, random

random.seed(7)
cantidad = 30000
nombres = [f"socio_{i:05d}" for i in range(cantidad)]
agenda = {nombre: i for i, nombre in enumerate(nombres)}
busquedas = [nombres[random.randrange(cantidad)] for _ in range(5000)]

inicio = time.perf_counter()
en_lista = sum(1 for nombre in busquedas if nombre in nombres)
t_lista = time.perf_counter() - inicio

inicio = time.perf_counter()
en_dict = sum(1 for nombre in busquedas if nombre in agenda)
t_dict = time.perf_counter() - inicio

print(en_lista, en_dict)                     # 5000 5000
print(en_lista == en_dict)                   # True
print(f"lista: {t_lista:.4f}s   dict: {t_dict:.4f}s   ratio: {t_lista / t_dict:.0f}x")
```

Las dos contestaron la verdad: las 5.000 búsquedas existen en ambos. Pero los tiempos no se parecen en nada. En la máquina del libro la lista tarda **más de un segundo**, y el diccionario **alrededor de una milésima** — mil veces arriba, mil veces abajo según la corrida, pero siempre con varios órdenes de magnitud de ventaja. Y fijate: no es "el dict es un poco más rápido". Es una **familia de complejidad distinta**. La lista mira cada elemento: `O(n)`. El diccionario *saltó directo a un lugar*: `O(1)`. ¿A qué lugar? ¿Cómo sabía el diccionario dónde estaba `"socio_12345"` sin mirarlo?

La respuesta tiene dos palabras, y la primera es lo que hoy vamos a destripar:

```python
print("socio_12345" in agenda)               # True
print("socio_99999" in agenda)               # False
```

> **Dato clave:** un diccionario no *busca* su clave: la **transforma en un número** y va directo a la casilla de ese número. Lo que transforma es la **función hash**. En los próximos minutos vas a escribir la tuya propia y a ver el "salto" con tus propios ojos.

## 2. La función hash: de cualquier cosa, a un número

`hash()` existe en Python desde antes que existieras: es una función incorporada que toma un objeto y devuelve un **entero**. Miralá en acción con los tipos que el capítulo 5 te presentó:

```python
print(hash(1))                               # 1
print(hash(1.5))                             # 1152921504606846977
print(hash((1, 2)))                          # -3550055125485641917
print(hash("ana") == hash("ana"))            # True
```

Tres propiedades que son la constitución de toda función hash. La primera: **el mismo objeto siempre da el mismo número** — `hash(1)` es `1` hoy, mañana y en cualquier máquina; `hash("ana") == hash("ana")` es `True` en la misma corrida. La segunda, atenta, porque es sutil: los números de las tuplas de números son **estables entre corridas**, pero el de las cadenas **no**. Si imprimís `hash("ana")` en dos procesos distintos, en general van a ser números distintos. ¿Por qué? Porque Python le mezcla a los strings una **semilla aleatoria por proceso** (la variable de entorno `PYTHONHASHSEED`) para que un atacante no pueda predecir en qué casilla cae cada clave y reventarte la tabla con colisiones hechas a propósito. La igualdad se mantiene — lo que cambia es el número exacto.

La tercera propiedad es la que más se olvida: `hash()` es **solo la mitad de la historia**. Mirá este choque de trenes conceptual:

```python
print(hash(True) == hash(1))                 # True
print({True: "sí"}[1])                       # sí
```

`hash(True) == hash(1)` da `True`… ¿y `{True: "sí"}[1]` sigue devolviendo `"sí"`? ¿No debería ser una colisión? Acá está la clave de todo el capítulo: **la tabla hash resuelve las colisiones con la igualdad**. Dos claves distintas pueden caer en la misma casilla, y cuando eso pasa el diccionario no se enoja: compara con `==` y desempata. Que `True` y `1` compartan número no es un problema porque `True == 1` es `True` — para el diccionario *son la misma clave*. Ahora, ¿y si la clave ni siquiera tiene número? Mirá:

```python
try:
    {[1, 2]: "clave ilegal"}
except TypeError as error:
    print(type(error).__name__)              # TypeError
```

Esa es la jaula que el capítulo 5 ya te había mostrado: **las listas no se pueden usar de clave**. Ahora sabés el porqué profundo: la lista es **mutable**, su contenido cambia, y un hash que cambia es un número mentiroso — mañana el diccionario buscaría con otro número y perdería la clave. `hash()` exige que el objeto sea **inmutable** (o que se comporte como inmutable), porque el contrato completo es este: *dos objetos iguales deben tener el mismo hash, y el hash no debe cambiar mientras el objeto viva en la tabla*. Violar eso, como vas a ver en la sección 6, no da error: da una clave fantasma que existe pero ya no se encuentra.

> **Importante:** `hash()` + `==` son un equipo. La función te dice **dónde mirar**; la igualdad te dice **si encontraste lo correcto**. Si dos objetos son `==`, **tienen que** compartir `hash()`. Al revés no hace falta: dos objetos distintos *pueden* compartir número — eso es una colisión, y la tabla está diseñada para vivir con ellas.

## 3. La tabla hash hecha a mano: el arreglo con matemática adentro

Hora de construir. La idea es tan simple que parece una estafa: tenés un **arreglo de N casillas** (en Python, una lista de listas), y en vez de buscar una clave recorriéndolas, le pedís a una función que convierta la clave en un **índice** de casilla. La clave entra, la función calcula "casilla número 4", y es un salto directo. Lo único que falta es una función hash decente, y como no queremos depender de la semilla aleatoria de Python para nuestro ejemplo, escribimos la nuestra — didáctica, determinista, y la misma técnica (multiplicar y sumar los valores de los caracteres) que usan los diccionarios reales:

```python
def hash_polinomico(clave, base=31):
    h = 0
    for caracter in clave:
        h = h * base + ord(caracter)
    return h
```

Cada letra aporta su código `ord()` y el acumulador `h` va creciendo: `"Ana"` se convierte en un número grande, estable y reproducible en cualquier máquina. Miralo:

```python
print(hash_polinomico("Ana"))                # 65972
print(hash_polinomico("Ana") % 8)            # 4
```

El segundo paso es la magia del diccionario: **`hash % tamaño`**. El hash de `"Ana"` es `65972`, y `65972 % 8` (dividir por el tamaño del arreglo y quedarnos con el resto) da `4`. Ese 4 es la **casilla** donde va a vivir `"Ana"`. Ahora tenemos el plano completo de una tabla hash:

> **El corazón del diccionario en tres líneas:** `índice = hash(clave) % tamaño`. Todo lo demás es detalle — detalles importantes, pero detalle.

Construyamos la clase. Cada casilla es una **lista chica** (la cadena de una lista enlazada, de adentro hacia adentro), y cuando una casilla recibe dos claves distintas que comparten índice, ambas se apilan ahí sin drama. Eso es el **encadenamiento** (chaining), la técnica que usan los diccionarios reales de Python, solo que ellos con listas más sofisticadas. La clase completa, como cimiento del capítulo, igual que hicimos con `Grafo` en el 23:

```python
class TablaHash:
    def __init__(self, tamano=8, funcion=hash_polinomico):
        self.tamano = tamano
        self.funcion = funcion
        self.casillas = [[] for _ in range(tamano)]
        self.cantidad = 0

    def indice(self, clave):
        return self.funcion(clave) % self.tamano

    def agregar(self, clave, valor):
        i = self.indice(clave)
        for pos, (k, v) in enumerate(self.casillas[i]):
            if k == clave:
                self.casillas[i][pos] = (clave, valor)
                return
        self.casillas[i].append((clave, valor))
        self.cantidad += 1

    def obtener(self, clave):
        i = self.indice(clave)
        for k, v in self.casillas[i]:
            if k == clave:
                return v
        raise KeyError(clave)

    def __contains__(self, clave):
        i = self.indice(clave)
        return any(k == clave for k, v in self.casillas[i])

    def redimensionar(self, nuevo_tamano):
        viejo = self.casillas
        self.tamano = nuevo_tamano
        self.casillas = [[] for _ in range(nuevo_tamano)]
        self.cantidad = 0
        for casilla in viejo:
            for clave, valor in casilla:
                self.agregar(clave, valor)

    def factor_carga(self):
        return self.cantidad / self.tamano

    def __str__(self):
        lineas = []
        for i, casilla in enumerate(self.casillas):
            if casilla:
                lineas.append(f"[{i}] " + ", ".join(f"{k}: {v}" for k, v in casilla))
        return "\n".join(lineas) if lineas else "(tabla vacía)"
```

Repasá las piezas a la luz de lo que ya sabés. `indice()` es el salto: `funcion(clave) % tamano`, exactamente el `65972 % 8 = 4` de recién. `agregar()` hace **dos preguntas distintas**: primero calcula la casilla con el `hash`, y *después* recorre esa casillita comparando con `==` — si ya existe la clave, actualiza; si no, la apila. Eso es el contrato de la sección 2 viviendo en 5 líneas: `hash` para saltar, `==` para desempatar. Y `redimensionar()` va a ser el protagonista de la sección 5, así que por ahora mirala de reojo.

Carguemos el directorio del local con los apodos de siempre — Alma, Beto, Cami, Dani, Ema — y mirémosla trabajar:

```python
directorio = TablaHash()
datos = [("Alma", "555-0134"), ("Beto", "555-0241"), ("Cami", "555-0112"),
         ("Dani", "555-0800"), ("Ema", "555-0909")]
for nombre, telefono in datos:
    directorio.agregar(nombre, telefono)

print([directorio.indice(nombre) for nombre, _ in datos])   # [7, 6, 2, 0, 1]
print(round(directorio.factor_carga(), 2))                  # 0.62
print(directorio.obtener("Alma"))                           # 555-0134
print("Alma" in directorio, "Zara" in directorio)           # True False
print(directorio)
```

```text
[0] Dani: 555-0800
[1] Ema: 555-0909
[2] Cami: 555-0112
[6] Beto: 555-0241
[7] Alma: 555-0134
```

Fijate qué pasó: con una tabla de 8 casillas y 5 claves, cada una cayó en **su propia casilla** — `[7, 6, 2, 0, 1]`, ninguna se repite. Cinco nombres, cinco casillas, cero colisiones. Sin recorrer nada, `obtener("Alma")` salta al 7 y devuelve el teléfono. Y `"Alma" in directorio` hace `True` sin mirar los otros cuatro. Esto, ahora, con el arreglo pelado, es el mismo milagro del `in` que cronometraste en la sección 1: **el salto directo de `hash % tamaño`**. `Zara`? Su hash da otro número, y esa casilla está vacía: `False` al instante.

> **Dato clave:** el costo de la tabla hash no está en *leer* la casilla (eso es `O(1)`), está en **cuánto se llena una misma casilla**. Por eso el `factor_carga` (`cantidad / tamano`) es la métrica de salud de la tabla: acá, `5 / 8 = 0.62`, cinco elementos repartidos en ocho casillas. Cuando ese número sube, las casillas empiezan a apilarse… y eso es exactamente lo próximo.

## 4. Colisiones: cuando dos claves tocan la misma puerta

Nuestra función polinómica repartió bien porque mezcla el **orden** de las letras. Pero no toda función lo hace. Mirá esta — la ingenua, la que todo principiante escribe la primera vez:

```python
def hash_suma(clave):
    return sum(ord(caracter) for caracter in clave)
```

Suma los códigos sin importar el orden. Ahora mirá qué hacen estas tres "claves":

```python
print(hash_suma("Roma") % 7, hash_polinomico("Roma") % 7)   # 0 4
print(hash_suma("Mora") % 7, hash_polinomico("Mora") % 7)   # 0 3
print(hash_suma("Amor") % 7, hash_polinomico("Amor") % 7)   # 0 5
```

`"Roma"`, `"Mora"` y `"Amor"` son **anagramas**: las mismas letras en distinto orden. Para `hash_suma` son indistinguibles (todas suman 399, y `399 % 7 = 0`), así que las tres tocan la **misma casilla**. La polinómica, que respeta el orden, las esparció (`4`, `3`, `5`). Pero ojo — el punto del capítulo no es "elegí bien tu función". Es que **ninguna función perfecta existe**: como los hashes son números y las claves infinitas (o simplemente más numerosas que las casillas), por el principio del palomar **forzosamente hay colisiones**. La pregunta profesional no es "¿cómo evito las colisiones?" sino "¿cómo sobrevivo cuando llegan?". La respuesta ya la escribiste en `TablaHash.agregar`: **encadenamiento**.

Carguemos los anagramas en una tabla chica de 5 casillas con la función ingenua y mirá la casilla 4:

```python
tabla_anagramas = TablaHash(tamano=5, funcion=hash_suma)
for nombre in ("Roma", "Mora", "Amor"):
    tabla_anagramas.agregar(nombre, nombre[0])

print(tabla_anagramas)                       # [4] Roma: R, Mora: M, Amor: A
print(len(tabla_anagramas.casillas[4]))      # 3
print(tabla_anagramas.obtener("Mora"))       # M
```

Las tres cayeron en la casilla `4`, apiladas una sobre otra — la "lista dentro de la lista" que `agregar` hace `append`. ¿Y cómo sobrevivimos? Mirá `obtener("Mora")`: calcula su índice (`4`), entra a la casilla, y **recorre la cadenita comparando con `==`** hasta dar con `"Mora"`. La colisión no rompió nada: solo convirtió una búsqueda de `O(1)` en una búsqueda dentro de una casillita de 3 elementos. Esto es lo que hace el diccionario real cuando dos de tus claves comparten número: mismo hash, dirección distinta, la igualdad desempata.

> **Imprimir la tabla con anagramas fue el momento incómodo:** con una función pobre, la tabla entera se cayó a la casilla 4. Buenas noticias: acá el encadenamiento te salvó. Malas noticias: si la tabla crece y el factor de carga sube, las cadenitas dejan de ser "casillitas de 3" y se vuelven "listas de 300" — y ahí hasta una buena tabla pierde su `O(1)`. De eso trata la última pieza.

## 5. Crecimiento: el factor de carga que mantiene todo en orden

La tabla empieza chica (8 casillas) y la vida es fácil. Pero el directorio del local crece — el negocio vende más, llegan más clientes — y las casillas se van llenando. En algún punto las cadenitas se vuelven listas largas y el "salto directo" degenera en "recorrer una lista". ¿Qué hace un diccionario profesional cuando eso pasa? **Crece.** El truco es medir primero. Acá jugamos a que la tabla se llena demasiado — empecemos por solo 4 casillas:

```python
tabla = TablaHash(tamano=4)
for i in range(10):
    tabla.agregar(f"clave_{i}", i)

print(tabla.cantidad, "en", tabla.tamano, "casillas")       # 10 en 4 casillas
print(round(tabla.factor_carga(), 2))                       # 2.5
print("índice de clave_1:", tabla.indice("clave_1"))        # índice de clave_1: 1
```

Diez claves hacinadas en cuatro casillas: **factor de carga 2.5**, o sea en promedio hay 2.5 claves por casilla. Ya está degenerando. La solución es `redimensionar`: **duplicar el tamaño y reubicar todo**. Mirá qué pasa con `clave_1` — y esto es lo importante — el índice cambia:

```python
tabla.redimensionar(16)

print(tabla.cantidad, "en", tabla.tamano, "casillas")       # 10 en 16 casillas
print(round(tabla.factor_carga(), 2))                       # 0.62
print("índice de clave_1:", tabla.indice("clave_1"), "| valor:", tabla.obtener("clave_1"))
# índice de clave_1: 9 | valor: 1
```

`clave_1` saltó de la casilla `1` a la `9`. ¿Por qué? Porque el índice es `hash % tamano`: al cambiar el tamaño, cambia el resto, y **cada clave se reubica** según su nuevo índice. `redimensionar` no mueve los datos "más o menos": crea un arreglo nuevo, reaplica `hash % nuevo_tamano` a cada clave, y la tabla vuelve a tener espacio de sobra (factor 0.62, las mismas cuentas de la sección 3). La clave de la reubicación la pagás **una sola vez** por duplicación: aunque redimensionar cuesta `O(n)`, hacerlo el doble de veces menos hace que en promedio cada inserción siga costando `O(1)` — ese promedio con asterisco se llama **costo amortizado**, y es la misma matemática del crecimiento de las listas de Python del capítulo 11.

> **Dato clave:** el diccionario de Python no espera a que se llene: **redimensiona cuando el factor de carga pasa de 2/3** (y nunca encoge cuando borrás, para no perder la inversión). El `grow` real es más fino que el nuestro — arreglos con orificios en vez de cadenas — pero el principio es idéntico: *si las casillas se llenan, duplicá y rehash*. Tu `TablaHash` y el `dict` de la sección 1 son primos carnales.

## 6. Datos propios hashables: la `Persona` que se porta bien

Llegó el momento que el capítulo 5 prometió y el 19 confirmó. Tenés tu propia clase — `Persona`, la misma amiga de los capítulos 12 y 13 — y querés usarla como clave de diccionario. La regla de oro: **`Persona` necesita un `__hash__` consistente con su `__eq__`**. Por defecto, `object` les da a los objetos el hash de la identidad (cada instancia es única) y la igualdad de la identidad (cada instancia se compara solo consigo misma). Para una persona por *valor*, eso no sirve: dos objetos distintos con el mismo nombre y edad tendrían que ser la misma clave. Definamos los dos métodos **a la par**, con el mismo material y sin separarlos jamás:

```python
class Persona:
    def __init__(self, nombre, edad):
        self.nombre = nombre
        self.edad = edad

    def __eq__(self, otro):
        if isinstance(otro, Persona):
            return self.nombre == otro.nombre and self.edad == otro.edad
        return False

    def __hash__(self):
        return hash((self.nombre, self.edad))
```

El truco que ya conocés del capítulo 5: la `Persona` pide prestado el hash de su **tupla de campos** — `hash((self.nombre, self.edad))`. Como la tupla es inmutable y hashable, la `Persona` se vuelve hashable por delegación, y el `__eq__` usa exactamente esos mismos campos: la regla "iguales ⇒ mismo hash" se cumple por construcción. Ahora la vemos en la selva — sets, diccionarios, pertenencia:

```python
alice = Persona("Alice", 30)
clon = Persona("Alice", 30)
bob = Persona("Bob", 25)

print(alice == clon)                        # True
print(alice == bob)                         # False
print(len({alice, clon, bob}))              # 2
padron = {alice: "Doctora", bob: "Ingeniero"}
print(padron[clon])                         # Doctora
print(clon in {alice, bob})                 # True
```

`alice` y `clon` son dos objetos distintos (identidades distintas), pero para el mundo hashable son **la misma persona**: `==` da `True`, el set los fusiona (`len` da 2, no 3), y `padron[clon]` devuelve `"Doctora"` aunque la clave insertada fue `alice`. Ese es el poder que el capítulo 5 te prometió: claves que *son tus objetos*, no strings que los representan. Tu `Persona` entró a la misma liga que los números y las tuplas.

Ahora la parte aterradora. Recordá el contrato de la sección 2: *el hash no debe cambiar mientras el objeto vive en la tabla*. ¿Qué pasa si lo violamos?

```python
cobros = {alice: "Doctora", bob: "Ingeniero"}
alice.nombre = "Alicia"        # mutamos la clave… ¡ya está en el dict!

print(alice == clon)                        # False
try:
    print(cobros[alice])
except KeyError:
    print("KeyError")                       # KeyError
print(alice in cobros)                      # False
```

Diagnóstico: al mutar `alice.nombre`, su `__hash__` cambió (ahora es el de `"Alicia"`), y el diccionario **guardó la clave bajo el hash viejo**. Cuando pedís `cobros[alice]`, Python calcula el hash *nuevo*, salta a otra casilla, no encuentra nada y te lanza `KeyError` — la clave existe en la tabla, pero nadie puede encontrarla. Y fijate el agravante cósmico: **ni siquiera da error de diseño**. El `try/except` te atrapó la excepción, pero si no lo hubieras puesto, el programa habría explotado en silencio con un `KeyError` de una clave que *tu mismo programa puso ahí*. Esto, en el mundo real, son bugs que tardan días en aparecer porque mutás una persona en un módulo y la buscás en otro.

> **Importante:** con objetos mutables como clave, el problema no es "si" la mutás — es que *cualquier* mutación posterior es una bomba de tiempo. Por eso el capítulo 5 decía "claves inmutables". Y por eso la producción no se conforma con que tu clase *tenga* un `__hash__`: exige que no puedas romperlo. Ese es el próximo escalón.

## 7. La `Persona` a prueba de balas: congelada antes de entrar a la tabla

La `Persona` de la sección 6 se porta bien *si no la tocás*. Pero el capítulo 14 te enseñó el reflejo del profesional: no dejés la puerta abierta esperando que nadie entre — **cerrala**. La versión a prueba de balas le prohíbe al objeto mutar los campos que forman su hash. Y la arma viene del capítulo 13: `__slots__` (que ya adelgazó objetos a un tercio de memoria y que acá da un bonus extra, como vas a ver) más un `__setattr__` que rechaza cualquier escritura, más un `set_dni` que permite *un solo* cambio controlado antes de congelar. Material premium, la clase entera:

```python
from datetime import date

class PersonaCongelada:
    __slots__ = ("_nombre", "_fecha_nacimiento", "_dni", "_bloqueado")

    def __init__(self, nombre, fecha_nacimiento, dni=""):
        object.__setattr__(self, "_nombre", nombre)
        object.__setattr__(self, "_fecha_nacimiento", fecha_nacimiento)
        object.__setattr__(self, "_dni", dni)
        object.__setattr__(self, "_bloqueado", bool(dni))

    def __setattr__(self, nombre, valor):
        raise AttributeError(f"No puedes modificar '{nombre}'")

    def __eq__(self, otro):
        if isinstance(otro, PersonaCongelada):
            return (self._nombre, self._fecha_nacimiento, self._dni) == (
                otro._nombre, otro._fecha_nacimiento, otro._dni)
        return False

    def __hash__(self):
        return hash((self._nombre, self._fecha_nacimiento, self._dni))

    @property
    def nombre(self):
        return self._nombre

    @property
    def fecha_nacimiento(self):
        return self._fecha_nacimiento

    @property
    def dni(self):
        return self._dni if self._dni else "DNI no registrado"

    def set_dni(self, valor):
        if not self._bloqueado:
            object.__setattr__(self, "_dni", valor)
            object.__setattr__(self, "_bloqueado", True)
        else:
            raise AttributeError("No puedes modificar el dni, ya está registrado.")
```

Leamos la defensa línea por línea, con los lentes del capítulo 14. `__slots__` no es solo memoria: al declarar los atributos a mano y **no incluir `__dict__`**, el objeto no tiene diccionario interno — no hay manera de inventar atributos nuevos por afuera (`persona.nueva_propiedad = 1` explota). El `__setattr__` sobreescrito convierte *toda* asignación en un `AttributeError`: `per_1.nombre = "Carlos"` ni siquiera arranca. Y como los datos se escriben solo una vez, en el `__init__` (con `object.__setattr__`, el escape legal del capítulo 13), el objeto nace congelado. El único "poro" es `set_dni`: permite escribir el DNI una sola vez (mientras `_bloqueado` sea falso) y luego **se sella solo** — porque el DNI forma parte del hash (junto a nombre y fecha), y el hash no puede cambiar una vez que la persona vive en una tabla. Contratás inmutable, recibís descanso.

Carguemos tres personas y juguemos con la tabla:

```python
per_1 = PersonaCongelada("Juan", date(1990, 5, 15))
per_2 = PersonaCongelada("Juan", date(1990, 5, 15))
per_3 = PersonaCongelada("Ana", date(1985, 8, 23))

print(per_1 == per_2, per_1 == per_3)       # True False
print(hash(per_1) == hash(per_2))           # True
per_1.set_dni("12345678A")
per_2.set_dni("12345678A")

socios = {per_1: "Amigo de la infancia", per_3: "Compañera de trabajo"}
print(socios[per_2], socios[per_3])         # Amigo de la infancia Compañera de trabajo

try:
    per_1._nombre = "Carlos"
except AttributeError as error:
    print(error)                            # No puedes modificar '_nombre'

try:
    per_1.set_dni("99999999")
except AttributeError as error:
    print(error)                            # No puedes modificar el dni, ya está registrado.

print(socios[per_1], socios[per_2])         # Amigo de la infancia Amigo de la infancia
print(len({per_1, per_2, per_3}))           # 2
```

`per_1` y `per_2` son objetos distintos con el mismo contenido: mismo `hash`, misma igualdad — la misma llave con dos copias, y el diccionario las trata como una (`len` de set = 2 con tres objetos). Después del `set_dni`, la persona queda sellada: `_nombre` es intocable y el segundo `set_dni` lanza el `AttributeError` que acabamos de escribir. Y la prueba del pudín: `socios[per_1]` y `socios[per_2]` siguen devolviendo el valor, porque su hash **nunca cambió** — la congelación no es una medida de seguridad paranoica, es lo que mantiene vivo el contrato de la sección 2.

> **Dato clave:** mirá el contraste con la sección 6. La `Persona` simple se tuvo que portar bien *por convención*; la `PersonaCongelada` **no puede portarse mal, ni siquiera por accidente**. Esa es la diferencia entre "funciona si…" y "funciona porque la estructura no lo permite" — el mismo salto de calidad que diste cuando el capítulo 23 convirtió el `Grafo` en `GrafoProtegido`.

## 8. En producción: la promesa que cierra la Parte VIII

Todo lo de la sección 7 — `__slots__`, `__setattr__` con candado, el `set_dni` de una sola vez — fue bajar a mano lo que el capítulo 13 ya te había regalado. Acordate de la línea 807 de ese capítulo: `@dataclass(frozen=True)`, "la respuesta de la POO por convención". La `dataclass` congelada es exactamente una `PersonaCongelada` de fábrica: genera el `__init__`, el `__repr__`, el **`__eq__` por los campos** y, como es `frozen=True`, el **`__hash__` automático** consistente con esa igualdad — y además prohíbe la mutación igual que nuestro `__setattr__`. Contratás 10 líneas como servicio. Mirá el mismo padrón, versión producción:

```python
from dataclasses import dataclass, FrozenInstanceError

@dataclass(frozen=True)
class Socio:
    codigo: str
    nombre: str

socio_1 = Socio("B-0001", "Alma")
socio_2 = Socio("B-0001", "Alma")

print(socio_1 == socio_2)                    # True
padron_socios = {socio_1: "Activo"}
print(padron_socios[socio_2])                # Activo
try:
    socio_1.nombre = "Ana"
except FrozenInstanceError as error:
    print(type(error).__name__)              # FrozenInstanceError
```

`Socio` nasce con `__eq__` y `__hash__` sincronizados de fábrica, se puede usar de clave, y el intento de mutación explota con `FrozenInstanceError`. Dos objetos distintos pero iguales van a la misma "casilla" (mismo hash), y la tabla los trata como un solo socio. Esta es la pieza que el capítulo 23 ya estaba usando callado: el `GrafoProtegido` validaba los vértices con `hash()` porque sabía que sus diccionarios de aristas — `red[("Cami", "Dani")] = 15` — dependían de que las claves fueran hashables. Las **tuplas de vértices** como clave de arista, los **conjuntos** de visitados en BFS, el `dp[(mascara, ultimo)]` del viajante del capítulo 25… todos eran tablas hash usando el contrato de hoy. Ahora ya viste el motor:

```python
caminos = {}
caminos[("Cami", "Dani")] = 15
caminos[("Alma", "Beto")] = 7

print(caminos[("Cami", "Dani")])             # 15
print(len(caminos))                          # 2
```

Cada par `(origen, destino)` es una clave de dos partes: Python le calcula el hash a la tupla entera, le saca el `%` del tamaño y salta a la casilla. Lo que el capítulo 23 llamaba "la lista de adyacencia" no es otra cosa que **una tabla hash en la que cada clave es un par ordenado**. Y tu `TablaHash`, la `Persona`, la `PersonaCongelada` y el `Socio` congelado son todos eslabones de la misma cadena: objetos que saben transformarse en su propio índice de casilla.

Cerra el círculo con el capítulo 19. Ahí te prometieron la estructura "para terminar de entender el `dict` de adentro hacia afuera". Hoy la construiste: un arreglo, una función `hash` determinista, un `%` que convierte el hash en dirección, encadenamiento para las colisiones, redimensionamiento para mantener el factor de carga bajo, y el `__hash__`/`__eq__` como contrato de los tipos hashables. Lo que en el capítulo 5 era magia ("¿por qué es tan rápido?") y en el 19 teoría ("es una tabla hash") hoy es código tuyo corriendo: **tu tabla hash salta directo donde un diccionario real salta, por el mismo motivo**. Como cierre de la Parte VIII, este es el lugar donde la estructura de datos dejó de ser una caja negra y pasó a ser una herramienta que podés diseñar, medir y defender.

> **Dato clave:** el diccionario real de Python no usa exactamente nuestro arreglo de listas — usa un esquema más fino de "arreglo con agujeros" (open addressing) que ahorra la memoria de las listas internas — pero resuelve el mismo problema con el mismo contrato: `hash` para ubicar, `==` para desempatar, crecimiento al `2/3` de carga, y la exigencia de claves inmutables o congeladas. Tu `TablaHash` es el mismo animal, con el esqueleto al aire.

## 9. Resumen y conceptos clave

Este capítulo abrió el diccionario — la estructura que usaste desde el capítulo 5 como quien respira — y te mostró el motor: una **tabla hash**. Un arreglo de casillas donde cada clave no se *busca* sino que se **transforma en un número** con `hash()`, se reduce al tamaño del arreglo con `%`, y se **salta directo** a su casilla: la diferencia entre `O(n)` (recorrer una lista) y `O(1)` (saltar a un lugar) que cronometraste como 5000 búsquedas en más de un segundo contra una milésima. La función hash respeta tres leyes: determinismo (mismo objeto, mismo número), eficiencia, y el **contrato** con la igualdad — *dos objetos `==` comparten hash, y el hash no cambia mientras la clave vive en la tabla*. El `hash()` de Python les mete a los strings una semilla aleatoria por proceso (por seguridad), pero los números y las tuplas de números son estables entre corridas, y por eso las tuplas son las claves de cable del capítulo 23: `red[("Cami", "Dani")] = 15`.

Construiste tu **`TablaHash`** a mano: `índice = funcion(clave) % tamano`, **encadenamiento** para las colisiones (cada casilla es una lista chica que apila; la colisión no rompe nada, solo a alarga esa casillita), y **crecimiento** cuando el factor de carga se dispara: `redimensionar` duplica el arreglo, reubica cada clave con su nuevo `%`, y devuelve el `O(1)` amortizado — el mismo reflejo del `dict` real, que crece al `2/3` de carga. Y con la tabla en la mano, cumpliste la promesa de los capítulos 5 y 19: **tus objetos son hashables**. La `Persona` simple con `__hash__` delegado a su tupla de campos y `__eq__` con los mismos campos; la `PersonaCongelada` a prueba de balas con `__slots__`, `__setattr__` que rechaza toda escritura y `set_dni` que sella después de un único uso; y la `Socio` de producción con `@dataclass(frozen=True)`, que te regala el par sync de fábrica. Viste también el crimen perfecto de las claves mutables: mutar un atributo que forma parte del hash convierte tu clave en un fantasma — existe en la tabla pero nadie la encuentra (`KeyError` de una llave que el mismo programa insertó). Hoy sabés por qué el capítulo 5 insistía en "claves inmutables", por qué el 13 te regaló `frozen=True`, por qué el 19 te prometió esta estructura por última, y por qué el 23 validaba sus vértices con `hash()`: todos eran la misma respuesta esperando a que la construyeras.

Repasá el checklist antes de seguir:

- [ ] La **tabla hash** es un arreglo donde la clave se convierte en índice con `hash(clave) % tamano`: el salto `O(1)` del diccionario.
- [ ] **`hash()`** devuelve un entero por objeto; determinista en el mismo proceso; las tuplas de números son estables entre corridas, los strings no (semilla aleatoria).
- [ ] El **contrato**: si `a == b` entonces `hash(a) == hash(b)`; y el hash **no puede cambiar** mientras la clave vive en la tabla.
- [ ] **Colisiones**: dos claves distintas en la misma casilla. Se resuelven con **encadenamiento** (listas por casilla) + comparación con `==`.
- [ ] **Factor de carga** = cantidad / tamaño. Si sube, `redimensionar` duplica el arreglo y rehash: `O(n)` de vez en cuando, `O(1)` amortizado.
- [ ] Objetos de tu clase como clave: definir **`__hash__` y `__eq__` juntos**, con los mismos campos (delegar a `hash((...))`).
- [ ] Claves mutables = **claves fantasma**: mutar un campo del hash hace que el dict ya no encuentre la clave (`KeyError`).
- [ ] Inmutabilidad real con `__slots__` + `__setattr__` bloqueado, o el atajo `@dataclass(frozen=True)` para `__eq__`/`__hash__` en sync de fábrica.

## 10. Ejercicios

1. **El salto en números.** Con 30.000 socios y el dict `agenda` de la sección 1, mostrá que las búsquedas `"socio_00007"`, `"socio_15000"` y `"socio_29999"` están todas en `agenda`, y que `"socio_30000"` (uno más que la lista) no está. Después verificá con `==` que la función `hash_polinomico` de `"Alma"` es la misma en tu máquina que la del libro: `hash_polinomico("Alma") % 8` tiene que dar el índice del `directorio` de la sección 3.

2. **El padrón con tu class.** Definí la `Persona` hashable de la sección 6 (nombre y edad) y construí un set con tres personas: dos idénticas (`"Alice", 30`) y una distinta (`"Bob", 25`). Mostrá cuántas entradas quedan, verificá con `==` que la copia es la original, y usala como clave de un diccionario: `{alice: "Doctora", bob: "Ingeniero"}` accedido con la copia.

3. **El fantasma indeseado.** Con la `Persona` de la sección 6: insertá `alice` en un dict, después **mutá** `alice.nombre` a `"Alicia"`, y mostrá qué pasa cuando (a) comparás `alice == clon`, (b) preguntás `alice in dict` y (c) pedís `dict[alice]` con `try/except`. Vas a ver las tres caras del mismo crimen: la igualdad rota, la pertenencia miente, y el acceso lanza `KeyError`.

4. **La tabla que se llena.** Con la `TablaHash` de la sección 3, creá una tabla de tamaño 4, cargá las 10 `clave_0..9`, y mostrá el factor de carga antes de redimensionar y el índice que tenía `clave_1` en la tabla vieja. Después `redimensionar(16)` y mostrá el nuevo factor, el nuevo índice de `clave_1`, y que `obtener("clave_1")` sigue devolviendo el valor original — la reubicación no pierde nada.

5. **La persona sellada.** Con `PersonaCongelada`, creá `"Juan"` (1990-05-15) con dos instancias, fixed el DNI a `"12345678A"` en las dos, y usalas como claves. Mostrá que el hash es el mismo, que el dict accede con la copia, que `len` del set con una tercera persona (`"Ana"`) es 2, y que **ambos** intentos de romperla (cambiar `_nombre` y volver a hacer `set_dni`) lanzan `AttributeError`.

6. **La promesa de producción.** Con `@dataclass(frozen=True)`, definí la clase `Socio` con `codigo` y `nombre`, creá dos iguales (`"B-0001", "Alma"`), mostrá que son `==`, que el dict accede con la copia, y que mutar `nombre` lanza `FrozenInstanceError`. Después, con el diccionario `caminos` de aristas del capítulo 23, agregá un par `("Dani", "Ema")` con peso `12` y mostrá que se accede con la tupla — la clave de dos partes hashable.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. El salto en números
import time, random
random.seed(7)
nombres = [f"socio_{i:05d}" for i in range(30000)]
agenda = {n: i for i, n in enumerate(nombres)}
print("socio_00007" in agenda, "socio_15000" in agenda, "socio_29999" in agenda, "socio_30000" in agenda)
# True True True False
print(hash_polinomico("Alma") % 8)                           # 7


# 2. El padrón con tu class
p1 = Persona("Alice", 30)
p2 = Persona("Alice", 30)
p3 = Persona("Bob", 25)
print(len({p1, p2, p3}))                                     # 2
print(p1 == p2)                                              # True
padron_2 = {p1: "Doctora", p3: "Ingeniero"}
print(padron_2[p2])                                          # Doctora


# 3. El fantasma indeseado
alice_3 = Persona("Alice", 30)
clon_3 = Persona("Alice", 30)
cobros_3 = {alice_3: "Doctora"}
alice_3.nombre = "Alicia"
print(alice_3 == clon_3)                                     # False
print(alice_3 in cobros_3)                                   # False
try:
    print(cobros_3[alice_3])
except KeyError:
    print("KeyError")                                        # KeyError


# 4. La tabla que se llena
tabla_4 = TablaHash(tamano=4)
for i in range(10):
    tabla_4.agregar(f"clave_{i}", i)
print(round(tabla_4.factor_carga(), 2))                      # 2.5
print("clave_1 en la vieja:", tabla_4.indice("clave_1"))     # clave_1 en la vieja: 1
tabla_4.redimensionar(16)
print(round(tabla_4.factor_carga(), 2))                      # 0.62
print("clave_1 en la nueva:", tabla_4.indice("clave_1"), tabla_4.obtener("clave_1"))
# clave_1 en la nueva: 9 1


# 5. La persona sellada
from datetime import date
juan_5 = PersonaCongelada("Juan", date(1990, 5, 15))
copia_5 = PersonaCongelada("Juan", date(1990, 5, 15))
ana_5 = PersonaCongelada("Ana", date(1985, 8, 23))
juan_5.set_dni("12345678A")
copia_5.set_dni("12345678A")
socios_5 = {juan_5: "Amigo de la infancia", ana_5: "Compañera de trabajo"}
print(hash(juan_5) == hash(copia_5))                         # True
print(socios_5[copia_5])                                     # Amigo de la infancia
print(len({juan_5, copia_5, ana_5}))                         # 2
try:
    juan_5._nombre = "Carlos"
except AttributeError as error:
    print(error)                                             # No puedes modificar '_nombre'
try:
    juan_5.set_dni("99999999")
except AttributeError as error:
    print(error)                                             # No puedes modificar el dni, ya está registrado.


# 6. La promesa de producción
from dataclasses import dataclass, FrozenInstanceError

@dataclass(frozen=True)
class SocioProd:
    codigo: str
    nombre: str

s1 = SocioProd("B-0001", "Alma")
s2 = SocioProd("B-0001", "Alma")
print(s1 == s2)                                              # True
padron_6 = {s1: "Activo"}
print(padron_6[s2])                                          # Activo
try:
    s1.nombre = "Ana"
except FrozenInstanceError as error:
    print(type(error).__name__)                              # FrozenInstanceError
caminos_6 = {("Cami", "Dani"): 15, ("Alma", "Beto"): 7}
caminos_6[("Dani", "Ema")] = 12
print(caminos_6[("Dani", "Ema")])                            # 12
print(caminos_6[("Cami", "Dani")])                           # 15
```