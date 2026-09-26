# Capítulo 21 — Listas enlazadas: el dato que se pasa el testigo

En el capítulo 20 viste al array desnudo: un bloque de memoria con los datos pegados, uno al lado del otro, veloz para acceder y torpe para cambiar. A cada `insert(0)` o `pop(0)` la fila entera se desplaza un lugar; cuando el bloque se queda corto, hay que reservar otro más grande y **copiar todo**. Es el precio de vivir todos juntos y en orden. Este capítulo te muestra la otra apuesta de la vieja escuela: que cada dato viva donde haya lugar y se pase el testigo con un puntero al siguiente. Sin fila, sin bloque, sin copias. Esa es la **lista enlazada** (*linked list*).

Vas a construirla con tus manos, nodo por nodo, y a medir cada operación con la notación O grande del capítulo 19. Vas a ver dónde brilla —crecer sin copiar, agregar al principio sin desplazar nada—, dónde duele —encontrar un elemento exige caminar— y por qué su prima **doblemente enlazada** es, por dentro, la misma `deque` que usás desde el capítulo 5. En el camino vas a descubrir por qué una lista llena de nodos sueltos vive más lento de lo que su teoría promete, y qué es eso de la *localidad de caché* que tanto le gusta a los entrevistadores. Al final, una de las lecciones más difíciles de esta parte del libro: saber cuándo *no* usar una estructura es tan valioso como saber usarla.

---

## 1. La cadena que se suelta de la memoria

Volvé con la memoria al capítulo 20, a la escena del bloque contiguo. El array reserva una fila de lugares, todos juntos, y promete acceso instantáneo a cualquiera con una simple cuenta. Ese poder tiene un costo que ya conocés: para insertar un dato en el medio, cada vecino se corre un paso; para agregar al principio, la fila entera se mueve; y si el bloque se queda corto, lo tirás y copiás todo a uno más grande. Reorganizar una fila de cien personas con un solo lugar libre requiere que se corran noventa y nueve.

Imaginá ahora la apuesta contraria. ¿Y si cada persona, en vez de esperar un lugar en la fila, se parara donde quisiera, y el único dato que necesitara dejar registrado fuera *quién es el siguiente*? La primera, parada donde le tocó, lleva una nota que dice "el siguiente está en la esquina de Corrientes". El de Corrientes anuncia que el siguiente está en el bar del centro. Y el del bar, un cartel que dice "se acabó". Nadie se movió, nadie esperó lugar: cada elemento **se pasa el testigo** al siguiente, y lo único que tenés que saber es dónde está la primera. Si mañana alguien quiere entrar al principio, no hay que correr a nadie: le pego la nota a la cabeza y listo.

Esa es la **lista enlazada**. Cada dato vive en su propia caja — el **nodo** — y cada nodo guarda dos cosas: el dato y un **puntero** al nodo siguiente (un *puntero*, *pointer*, es una referencia: un "apunta para allá", no el dato mismo). No hay bloque contiguo, no hay fila: hay una cadena de cajas que se pasan el testigo. Agregar al principio ahora es O(1): ni una sola caja se corre. Crecer nunca copia nada: pido una caja nueva, la engancho al final y listo. El precio aparece en otro lado: para leer el quinto dato hay que *caminar* desde el primero, saltando de a una caja por vez, porque ninguna caja sabe quién sigue a menos que le preguntes a la anterior. Lo que el array pagaba en memoria apretada, la cadena lo cambia por **libertad para crecer sin copiar**.

Las listas enlazadas tienen parientes, y a lo largo del capítulo vas a conocer a los dos principales: la **simple** — cada nodo solo apunta al siguiente —, que vas a construir con tus propias manos, y la **doble** — cada nodo apunta al anterior y al siguiente —, la base del `deque` que ya te acompaña. La **circular**, donde el último vuelve a apuntar al primero, existe, y vas a verla nomás de pasada cuando valga la pena. Todo el capítulo se apoya en una idea que ya te resultó familiar en el capítulo 19: cada operación tiene un costo, y ese costo depende de dónde vivan tus datos.

> **Dato clave:** el array te dio acceso veloz a costa del cambio; la lista enlazada invierte el trato — cambio veloz en algunas puntas, acceso siempre a pata. No son rivales: son las dos mitades de un mismo problema, y cada una paga una moneda distinta.

## 2. El nodo: dato + testigo

Todo empieza con la caja más chica del mundo. El **nodo** tiene exactamente dos atributos: el dato y el puntero al siguiente. En Python, ese puntero es una simple referencia — un objeto guarda la dirección del próximo objeto —. No tiene nada de mágico: se escribe como cualquier atributo.

```python
class Nodo:
    def __init__(self, valor, siguiente=None):
        self.valor = valor
        self.siguiente = siguiente
```

El `siguiente=None` es clave: un nodo nuevo nace sin cadena, y cuando una caja no apunta a nadie, su puntero vale `None`, que es como la cadena dice "hasta acá llegamos". En `None` no hay dato que leer — es el fin del viaje.

Armá la cadena a mano, sin atajos, para verla con tus ojos. Tres cajas, tres datos:

```python
n1 = Nodo("datos")
n2 = Nodo("del")
n3 = Nodo("día")

n1.siguiente = n2
n2.siguiente = n3

print(n1.valor)                          # datos
print(n1.siguiente.valor)                # del
print(n1.siguiente.siguiente.valor)      # día
print(n1.siguiente.siguiente.siguiente)  # None
```

Tres cajas, dos enganches, y una forma de leer que ya se parece a la cadena que querés formar: `n1.siguiente.siguiente.valor`. Ese es el testigo pasando de caja en caja. Y como cada caja solo sabe quién viene después, la única manera de conocer la cadena completa es arrancar por la cabeza y saltar:

```python
actual = n1
while actual is not None:
    print(actual.valor)
    actual = actual.siguiente
# datos
# del
# día
```

Ese `while` de tres líneas es el corazón de todo el capítulo: **recorrer una lista enlazada es caminarla**. No hay `[3]`, no hay salto directo — hay que preguntarle a la caja uno quién sigue, a la dos quién sigue, y así hasta encontrar el `None`. El bloque contiguo del capítulo 20 leía `datos[3]` con una cuenta; acá no hay cuenta posible, porque cada caja vive donde se le cantó.

> **Importante:** el `siguiente` no es el dato: es una **referencia** al otro objeto. Cuando escribís `n1.siguiente` estás parado en la caja uno y tu mano apunta a la dos. Se siente parecido al puntero que tu lista nativa salta en el capítulo 20 — solo que acá no está escondido en un bloque: lo escribís vos, atributo por atributo.

## 3. La lista que recorre

Un nodo solo es una caja; la **lista enlazada** es la clase que sabe administrar la cadena. Para administrar una cadena hacen falta solo dos datos: dónde empieza — la **cabeza** — y cuántas cajas tiene. Todo lo demás se consigue caminando:

```python
class ListaEnlazada:
    def __init__(self):
        self.cabeza = None
        self.longitud = 0

    def append(self, valor):
        nuevo = Nodo(valor)
        if self.cabeza is None:
            self.cabeza = nuevo
        else:
            actual = self.cabeza
            while actual.siguiente is not None:
                actual = actual.siguiente
            actual.siguiente = nuevo
        self.longitud += 1

    def __len__(self):
        return self.longitud

    def __str__(self):
        pasos = "["
        actual = self.cabeza
        while actual is not None:
            pasos += str(actual.valor)
            if actual.siguiente is not None:
                pasos += " -> "
            actual = actual.siguiente
        return pasos + " -> " + str(actual) + "]"

    def __repr__(self):
        return f"ListaEnlazada({self})"
```

Mirá la clase en dos pasadas. La primera: el `append`. Si la cadena está vacía, el nuevo nodo se vuelve la cabeza. Si no, hay que **caminar hasta el final** — el `while` de la sección 2 — y recién ahí enganchar el nuevo nodo. Agregar al final de una lista nativa era un juego de niños (`append` O(1) amortizado, te dijo el capítulo 19); acá, cada vez que agregás al final, la lista completa se camina de punta a punta. Ya vas a ver qué precio tiene eso acumulado. La segunda pasada: los dunders. `__len__` devuelve la cuenta que la clase ya lleva, sin caminar nada — recorrer la cadena solo para contarte los dedos sería un desperdicio. `__str__` y `__repr__`, en cambio, sí caminan, para armar la historia con la flecha `->`. Como viste en el capítulo 13, el espejo del objeto hace que `print` y `repr` trabajen solos.

Fijate cómo el fin de la cadena, ese `None` que nadie toca, aparece en la foto con su lugar ganado. Armá una lista de verdad y paseala:

```python
notas = ListaEnlazada()
notas.append("pagar la luz")
notas.append("comprar pan")
notas.append("responder el mail")

notas = ListaEnlazada()
notas.append("pagar la luz")
notas.append("comprar pan")
notas.append(None)
notas.append("responder el mail")

print(len(notas))     # 4
print(notas)          # [pagar la luz -> comprar pan -> None -> responder el mail -> None]
print(repr(notas))    # ListaEnlazada([pagar la luz -> comprar pan -> None -> responder el mail -> None])
```

La flecha `->` ya está contando la historia: del pasado al presente, y el `None` cerrando el mapa. Ahora la pregunta incómoda: ¿cuánto cuesta, *en total*, llenar esta lista de punta a punta con `append`?

Para el primer `append` no caminás nada (la lista está vacía). Para agregar a una lista de dos cajas, caminás todo; a una de tres, un poco más; y así. En general, encontrar el final de una lista de `k` elementos implica preguntarle "¿tenés siguiente?" a cada uno de los `k` — al último, la respuesta es "no". Si llenás una lista vacía con `n` elementos, las preguntas se suman como los primeros números: 0 + 1 + 2 + … + (n − 1). Esa suma, la del pequeño Gauss que ya te acompañó en el capítulo 19:

```python
for n in (4, 100):
    pasos = n * (n - 1) // 2
    print(f"con {n} elementos: {pasos} pasos")
# con 4 elementos: 6 pasos
# con 100 elementos: 4950 pasos
```

Cien elementos, casi cinco mil pasos de caminata. **Llenar la lista con `append` es O(n²)**: cada dato nuevo paga el viaje hasta el final, y el final se alarga con cada dato. Ese es un ejemplo de manual de por qué el capítulo 19 nos enseñó a contar el costo de las operaciones *acumuladas*, no solo de una. Pero fijate el arreglo obvio: si la lista supiera dónde está su **última** caja, `append` no tendría que caminar nada. Ese es el segundo dato que le vamos a enseñar, y con él llegan las operaciones que dan sentido a todo este capítulo.

## 4. Agregar y borrar: el precio según el lugar

Guardá el puntero al final y el `append` deja de caminar. En vez de recorrer la cadena, le pegás el nuevo nodo directo a la última caja:

```python
class ListaEnlazada:
    def __init__(self):
        self.cabeza = None
        self.final = None
        self.longitud = 0

    def append(self, valor):
        nuevo = Nodo(valor)
        if self.final is None:
            self.cabeza = nuevo
        else:
            self.final.siguiente = nuevo
        self.final = nuevo
        self.longitud += 1

    def prepend(self, valor):
        nuevo = Nodo(valor, self.cabeza)
        self.cabeza = nuevo
        if self.final is None:
            self.final = nuevo
        self.longitud += 1

    def delete(self, valor):
        actual, previo = self.cabeza, None
        while actual is not None:
            if actual.valor == valor:
                if previo is None:
                    self.cabeza = actual.siguiente
                else:
                    previo.siguiente = actual.siguiente
                if actual.siguiente is None:
                    self.final = previo
                self.longitud -= 1
                return
            previo, actual = actual, actual.siguiente
        raise ValueError(f"{valor} no está en la lista")

    def __len__(self):
        return self.longitud

    def __iter__(self):
        actual = self.cabeza
        while actual is not None:
            yield actual.valor
            actual = actual.siguiente

    def __str__(self):
        pasos = "["
        actual = self.cabeza
        while actual is not None:
            pasos += str(actual.valor)
            if actual.siguiente is not None:
                pasos += " -> "
            actual = actual.siguiente
        return pasos + " -> " + str(actual) + "]"
```

Con `cabeza` y `final` en la mano, tanto agregar al principio — `prepend` — como agregar al final — `append` — son operaciones O(1): ni una caja se mueve, solo se reengancha un puntero o dos. Acá está el tesoro del capítulo: a tu lista nativa, `insert(0, dato)` le cuesta O(n) porque desplaza toda la fila (lo viste en el capítulo 5 y lo mediste en el 19); a la enlazada, agregar al principio cuesta lo mismo que agregar al final. **Elegís la punta gratis.**

Borrar un valor es otra historia, y la clase la cuenta en `delete`. Hay que llegar hasta la caja — caminar, otra vez — y recién ahí coser la cadena. Mirá el truco de cerca: mientras `actual` no sea quien buscás, `previo` va un paso atrás. Cuando encontrás el dato, si no había `previo` es que el condenado era la cabeza, y la cabeza pasa a ser la que le seguía; si había, la caja anterior apunta directo a la que seguía al borrado. Y si el borrado era el final, el puntero `final` se actualiza a mano. Falló todo el camino: `ValueError`, con los mensajes que ya dominaste en el capítulo 6.

Para que la lista se pueda usar en un `for`, la clase presta el truco del generador (capítulo 10) en `__iter__`: caminar y `yield`, un dato por vuelta, el mismo patrón que te permitió recorrer sin construir listas intermedias. Mirá la colección entera cambiar de forma con la misma cara:

```python
numeros = ListaEnlazada()
numeros.append(3)
numeros.append(5)
numeros.append(9)
numeros.prepend(1)
print(numeros)          # 1 -> 3 -> 5 -> 9 -> None

numeros.delete(5)
print(numeros)          # 1 -> 3 -> 9 -> None
print(len(numeros))     # 3

try:
    numeros.delete(500)
except ValueError as error:
    print(error)        # 500 no está en la lista

for numero in numeros:
    print(numero)       # 1
                        # 3
                        # 9
```

Con la lista viva, ya podés armar la tabla completa de costos. Comparala con la lista nativa de Python, y pagá con la notación O grande del capítulo 19:

| Operación | Lista nativa de Python | Lista enlazada simple |
|---|---|---|
| acceder a `datos[3]` | O(1) — salto directo | O(n) — caminar hasta el nodo 3 |
| agregar al final | O(1) amortizado | O(1) si guardás el `final` |
| agregar al principio | O(n) — desplaza toda la fila | O(1) — `prepend` |
| insertar en el medio | O(n) — desplaza vecinos | O(n) — caminar + coser, sin copiar |
| borrar al principio | O(n) — `pop(0)` | O(1) |
| borrar al final | O(1) — `pop()` | O(n) — hay que encontrar al penúltimo |
| borrar un valor | O(n) | O(n) — con el puntero `previo` |
| memoria por elemento | 1 puntero en el bloque (8 bytes) | nodo entero (≈ 152 bytes) |

Leéla con los dos ojos. Ganás en agregar y borrar en las puntas, y perdés feo en el acceso por índice y en la memoria. Y el renglón de "borrar al final" es una grieta que no vas a dejar pasar: hace falta conocer al penúltimo, y en una lista simple solo hay caminar. Esa grieta, ya lo vas a ver, tiene nombre: lista doble.

### En producción: frenar antes que explotar

Una lista así, sin límites, es un peligro silencioso. Si mañana vive en un servidor y alguien le manda datos arbitrarios, puede crecer hasta reventar la memoria: es el clásico ataque que el material viejo llama *denial of service* (denegación de servicio) — inundar un servicio hasta que se caiga. Y un `None` de regalo en el borde puede hacerte pisar un `AttributeError` feo. La defensa es el hábito del capítulo 6 aplicado a las estructuras: **validar antes de tocar**. Mirá así de chiquita y así de rendidora:

```python
class ListaEnlazada:
    MAX_LONGITUD = 10_000

    def _validar(self, valor):
        if valor is None:
            raise ValueError("no se pueden guardar valores nulos")
        if self.longitud >= self.MAX_LONGITUD:
            raise ValueError("la lista está llena")

    def __init__(self):
        self.cabeza = None
        self.final = None
        self.longitud = 0

    def append(self, valor):
        self._validar(valor)
        nuevo = Nodo(valor)
        if self.final is None:
            self.cabeza = nuevo
        else:
            self.final.siguiente = nuevo
        self.final = nuevo
        self.longitud += 1

    def __str__(self):
        pasos = "["
        actual = self.cabeza
        while actual is not None:
            pasos += str(actual.valor)
            if actual.siguiente is not None:
                pasos += " -> "
            actual = actual.siguiente
        return pasos + " -> " + str(actual) + "]"
```

Probala con un límite chiquito, para que el `MAX_LONGITUD` se despierte en tus manos:

```python
guardada = ListaEnlazada()
guardada.MAX_LONGITUD = 3            # un tope de juguete para probar

guardada.append(1)
guardada.append(2)
guardada.append(3)

try:
    guardada.append(4)
except ValueError as error:
    print(error)       # la lista está llena

try:
    guardada.append(None)
except ValueError as error:
    print(error)       # no se pueden guardar valores nulos

print(guardada)        # 1 -> 2 -> 3 -> None
```

> **Buenas prácticas:** definir un tope (`MAX_LONGITUD`) y validar las entradas no es relleno de estructura de datos: es el traje de calle. Un día esta lista termina viva en un servidor, y el tope es la diferencia entre "el usuario mandó basura" y "el servicio se cayó".

## 5. La cuenta honesta del costo

La tabla de la sección 4 tiene una letra bien grande: el acceso por índice. En la lista nativa, `datos[3]` era un salto; acá no existe tal salto. Si querés el dato de la posición tres, lo único que hay es caminar, y eso conviene verlo con las manos, no solo en la tabla:

```python
def obtener(lista, indice):
    actual = lista.cabeza
    for _ in range(indice):
        actual = actual.siguiente
    return actual.valor

semana = ListaEnlazada()
for dia in ("lunes", "martes", "miércoles", "jueves", "viernes"):
    semana.append(dia)

print(semana)              # lunes -> martes -> miércoles -> jueves -> viernes -> None
print(obtener(semana, 3))  # jueves
```

Tres saltos de caja para la posición tres. Si la lista tuviera un millón de cajas, el elemento del medio te costaría medio millón de pasos. La lista nativa lo hace con una cuenta. Ese es el precio de vivir disperso, y va a volver vestido de otra cosa un poco más adelante, cuando hablemos de la *localidad*.

La segunda letra grande es la memoria. Recordá la cuenta del capítulo 20: un millón de punteros en tu lista ocupaban 8,4 MB, porque por elemento se pagaba un puntero de 8 bytes en el bloque contiguo. Acá se paga, por elemento, un **nodo entero**, y un nodo es un objeto. Medilo en carne propia:

```python
import sys

nodo = Nodo(1)                        # un nodo con un solo dato
print(sys.getsizeof(nodo))            # 48  (el objeto en sí)
print(sys.getsizeof(nodo.__dict__))   # 104 (el diccionario que guarda sus atributos)
print(sys.getsizeof(1))               # 28  (el entero paga su propio espacio aparte)
```

Un nodo pesa **48 bytes de objeto más 104 de su diccionario de atributos**: unos 152 bytes de infraestructura, por cada dato. El entero, 28 bytes, se paga aparte y por las dos vías. Un millón de nodos te pide más de **152 MB** de esqueleto — cuando la lista nativa del capítulo 20 se arreglaba con 8,4 MB de punteros. La flexibilidad tiene costo de alquiler.

Y la tercera letra, la más física de todas, es la **localidad de caché** (*cache locality*). La memoria de la máquina no lee un solo byte cuando necesita uno: lee *manzanas* de datos de golpe y las guarda en una memoria rapidísima al lado del procesador, la caché, por si las vas a usar pronto. Si tus datos viven en un bloque contiguo (el array del capítulo 20) y los recorrés en orden, cada manzana que llega a la caché te alcanza para varios elementos: es la *localidad espacial*. Los nodos de la lista enlazada, en cambio, están tirados donde a cada uno se le dio la gana. El procesador pide el nodo 1, trae para la caché los vecinos *de memoria* del nodo 1 — que no tienen nada que ver con tu lista —, sigue al 2 (un fallo de caché, otra ida a la memoria lenta), y así en cada eslabón. La teoría decía "agregar en O(1)", y es verdad en el papel; pero caminar una lista enlazada real, con los nodos dispersos, le cuesta al hardware mucho más de lo que sugiere esa O(1). La fuente más confiable sobre esto es tu propia máquina: cuanto más grande la lista, más visible la diferencia, y eso no se arregla con mejor notación — se arregla con datos pegados.

> **Dato clave:** la lista enlazada resuelve el problema del *cambio* del array, pero le cambia el problema a otro lado: acceso lento y memoria cara. La balanza verdadera se juega en la linealidad: recorrer la cadena entera de punta a punta (por ejemplo, en un `for`) es lo que la lista enlazada sabe hacer; saltar a cualquier nodo es lo que no sabe.

Entonces, la conclusión incómoda, la que más cuesta aceptar: **en Python, la lista enlazada simple casi nunca gana en la cancha**. La lista nativa recorre más rápido (datos seguidos, caché llena), usa menos memoria y de paso ya tiene mil métodos. ¿Para qué existe entonces el concepto? Porque es la base que forma las piezas del rompecabezas: prepara el terreno de las pilas y colas que vas a armar, y sobre todo de la prima que viene ahora — que sí gana en su cancha, porque es la base de una clase de la biblioteca estándar que usás todos los días.

## 6. La prima doblemente enlazada y el deque

¿Recordás el renglón incómodo de la tabla: borrar el final de una lista simple es O(n), porque hay que encontrar al penúltimo? El arreglo es elegante y de una sola palabra: que cada nodo sepa **quién viene antes**. Si la caja guarda `anterior` además de `siguiente`, caminar para atrás ya no cuesta: es otro salto, con el mismo truco pero al revés. Esa es la **lista doble** (*doubly linked list*), que es una lista enlazada con dos testigos por nodo: uno para adelante, otro para atrás, y por eso los dos extremos se vuelven O(1).

La clase nueva es simétrica, y la simetría se nota en cada método: `push_inicio` y `push_final` son espejos uno del otro.

```python
class NodoDoble:
    def __init__(self, valor, anterior=None, siguiente=None):
        self.valor = valor
        self.anterior = anterior
        self.siguiente = siguiente


class ListaDoble:
    def __init__(self):
        self.inicio = None
        self.final = None
        self.longitud = 0

    def push_inicio(self, valor):
        nuevo = NodoDoble(valor, siguiente=self.inicio)
        if self.inicio is not None:
            self.inicio.anterior = nuevo
        else:
            self.final = nuevo
        self.inicio = nuevo
        self.longitud += 1

    def push_final(self, valor):
        nuevo = NodoDoble(valor, anterior=self.final)
        if self.final is not None:
            self.final.siguiente = nuevo
        else:
            self.inicio = nuevo
        self.final = nuevo
        self.longitud += 1

    def pop_inicio(self):
        if self.inicio is None:
            raise ValueError("la lista está vacía")
        valor = self.inicio.valor
        self.inicio = self.inicio.siguiente
        if self.inicio is not None:
            self.inicio.anterior = None
        else:
            self.final = None
        self.longitud -= 1
        return valor

    def pop_final(self):
        if self.final is None:
            raise ValueError("la lista está vacía")
        valor = self.final.valor
        self.final = self.final.anterior
        if self.final is not None:
            self.final.siguiente = None
        else:
            self.inicio = None
        self.longitud -= 1
        return valor

    def __len__(self):
        return self.longitud

    def __iter__(self):
        actual = self.inicio
        while actual is not None:
            yield actual.valor
            actual = actual.siguiente
```

Los cuatro movimientos — `push` y `pop` en cada punta — son O(1): cada uno solo toca el puntero que corresponde y avisa al vecino. Mirala en acción:

```python
doble = ListaDoble()
doble.push_inicio(2)
doble.push_inicio(1)
doble.push_final(3)

print(list(doble))            # [1, 2, 3]

print(doble.pop_inicio())     # 1
print(doble.pop_final())      # 3

print(list(doble))            # [2]
print(len(doble))             # 1
```

La lista doble arregló el renglón incómodo de la sección 4: borrar del `final` es O(1), porque el último nodo sabe quién lo antecedía. Y de paso ganó la otra punta también. El precio es el diccionario de atributos con un puntero más — la factura de memoria crece, sin drama, hasta que la lista sea enorme.

### Dos usos que te van a perseguir

Dos estructuras que ya conocés del capítulo 5 salen gratis de acá mismo, porque la lista doble puede funcionar como cualquiera de las dos. Avisá la regla por herencia (capítulo 15) y listo:

Las pilas (*stacks*) y las colas (*queues*) son dos contratos sobre la misma máquina: *LIFO* —last in, first out, "el último que entra, el primero que sale"— y *FIFO* —first in, first out, "el primero que entra, el primero que sale"—. Las dos se implementan con una lista doble y dos líneas de traducción, porque la lista doble ya trabaja en O(1) en ambos extremos:

```python
class Pila(ListaDoble):
    def push(self, valor):
        self.push_final(valor)

    def pop(self):
        return self.pop_final()


class Cola(ListaDoble):
    def enqueue(self, valor):
        self.push_final(valor)

    def dequeue(self):
        return self.pop_inicio()
```

Usalas y fijate cómo sale el orden:

```python
pila = Pila()
pila.push(1)
pila.push(2)
pila.push(3)
print(pila.pop(), pila.pop(), pila.pop())   # 3 2 1   (LIFO: sale el último)

fila = Cola()
fila.enqueue(1)
fila.enqueue(2)
fila.enqueue(3)
print(fila.dequeue(), fila.dequeue(), fila.dequeue())   # 1 2 3   (FIFO: sale el primero)
```

La pila sale al revés de como entró; la cola, al derecho. Ese contraste es el idioma de los algoritmos: acá lo hiciste vos a mano, con nodos y punteros, y por eso sabés que no es magia — es la lista doble haciendo su única cosa, en dos direcciones.

Y acá cae la ficha que estaba prometida desde el arranque del capítulo: **el `deque` del capítulo 5 es, por dentro, una lista doble**. La *double-ended queue* (cola de doble extremo) de `collections` es exactamente lo que acabás de construir, pero escrito en C por los autores de la biblioteca estándar: `append` y `pop` en ambos extremos, O(1), con la memoria ya ajustada. La diferencia es que el `deque` está terminado, probado y compilado. Recordalo y probalo con la misma mano:

```python
from collections import deque

cola = deque([1, 2, 3])
cola.append(4)          # al final
cola.appendleft(0)      # al principio

print(list(cola))       # [0, 1, 2, 3, 4]
```

Cuando en el capítulo 19 te explicaban el `deque` con punteros "en ambos extremos", esto era: una lista doble. Ahora lo viste nacer. Hablando de parientes, la variante **circular** — donde el `siguiente` del último vuelve al primero — se usa para las "ruletas": rotar turnos entre procesos, ser la cola de un CPU o dar vuelta un búfer de audio en loop. Es la misma cadena, con el `None` reemplazado por el regreso a la cabeza, y es justamente la prima que la sección que sigue construye de punta a punta.

## 7. La prima circular y el espejo completo

La sección 6 cerró con una promesa: la variante **circular** quedó "para cuando la necesites". Ese día llegó. Y de paso vas a ver cómo la clase de nodos soporta el espejo completo del capítulo 13 — leer con `[i]`, escribir con `[i] = valor`, recorrer con `for`, buscar y borrar por valor —, todo el contrato de la lista nativa, ahora sobre lazos de punteros.

### El borde que se cose

En la lista doble el último nodo apuntaba a `None`: la marca de "hasta acá llegamos". La circular reemplaza ese `None` por una costura: el `siguiente` del `final` vuelve al `inicio`, y el `anterior` del `inicio` apunta al `final`. Ya no hay borde; hay bucle. La cadena se da la mano con su propia cola.

> **Dato clave**: en un anillo no existe el `None`. El `siguiente` del `final` es el `inicio`, y el `anterior` del `inicio` es el `final`. La lista tiene cero extremos, solo costuras.

Agarrá la `ListaDoble` de la sección 6 y hacele dos retoques. Primero: el **primer** nodo se cose consigo mismo, `siguiente` y `anterior` apuntando a sí — así una lista de un solo elemento ya es un anillo de uno, y los `pop` nunca tropiezan con el aire. Segundo: cada movimiento reconstruye la costura del lado que toca:

```python
class NodoDoble:
    def __init__(self, valor, anterior=None, siguiente=None):
        self.valor = valor
        self.anterior = anterior
        self.siguiente = siguiente


class ListaCircularDoble:
    def __init__(self):
        self.inicio = None
        self.final = None
        self.longitud = 0

    def push_inicio(self, valor):
        nuevo = NodoDoble(valor)
        nuevo.siguiente = self.inicio if self.inicio else nuevo
        nuevo.anterior = self.final if self.final else nuevo
        if self.inicio is None:
            self.final = nuevo
        else:
            self.inicio.anterior = nuevo
            self.final.siguiente = nuevo
        self.inicio = nuevo
        self.longitud += 1

    def push_final(self, valor):
        nuevo = NodoDoble(valor)
        nuevo.siguiente = self.inicio if self.inicio else nuevo
        nuevo.anterior = self.final if self.final else nuevo
        if self.final is None:
            self.inicio = nuevo
        else:
            self.final.siguiente = nuevo
            self.inicio.anterior = nuevo
        self.final = nuevo
        self.longitud += 1

    def pop_inicio(self):
        if self.inicio is None:
            raise ValueError("la lista está vacía")
        valor = self.inicio.valor
        self.longitud -= 1
        if self.longitud == 0:
            self.inicio = None
            self.final = None
        else:
            self.inicio = self.inicio.siguiente
            self.inicio.anterior = self.final
            self.final.siguiente = self.inicio
        return valor

    def pop_final(self):
        if self.final is None:
            raise ValueError("la lista está vacía")
        valor = self.final.valor
        self.longitud -= 1
        if self.longitud == 0:
            self.inicio = None
            self.final = None
        else:
            self.final = self.final.anterior
            self.final.siguiente = self.inicio
            self.inicio.anterior = self.final
        return valor

    def __len__(self):
        return self.longitud

    def __str__(self):
        if self.inicio is None:
            return "[]"
        piezas = []
        actual = self.inicio
        for _ in range(self.longitud):
            piezas.append(str(actual.valor))
            actual = actual.siguiente
        return " -> ".join(piezas) + f" -> (vuelve a {self.inicio.valor})"
```

El `__str__` deja la costura a la vista: la línea termina con `(vuelve a …)` en vez de `None`. Mirala en acción, y probá el anillo con las propias puntas:

```python
anillo = ListaCircularDoble()
anillo.push_final(0)
anillo.push_final(1)
anillo.push_final(2)
anillo.push_final(3)

print(anillo)                         # 0 -> 1 -> 2 -> 3 -> (vuelve a 0)

print(anillo.final.siguiente.valor)   # 0
print(anillo.inicio.anterior.valor)   # 3

print(anillo.pop_final())             # 3
print(anillo.pop_inicio())            # 0
print(anillo)                         # 1 -> 2 -> (vuelve a 1)

solo = ListaCircularDoble()
solo.push_final(7)
print(solo)                           # 7 -> (vuelve a 7)
print(solo.pop_final())               # 7
print(solo)                           # []
```

Los dos `print` de las puntas son la prueba del anillo: el `final` sabe su `siguiente` y el `inicio` sabe su `anterior`. Y cuando el `pop` vacía el anillo, no queda nada que coser: la clase vuelve a `[]`, y con un solo `push` el anillo renace de un único nodo.

### El espejo completo

Recordá el capítulo 13: cuando un objeto se mira en el espejo, `print` y `repr` trabajan solos, y también `len`, `[i]`, `+=`, `==`. Hasta acá la `ListaCircularDoble` sabe contarse y mostrarse. Le falta el resto del trato de lista: leer `anillo[i]`, escribir `anillo[i] = valor` y recorrer con `for`. Son tres dunders y una regla que no estaba en la sección 3: la lista nativa corta el recorrido sola, pero la cadena tiene que **contar**.

> **Importante**: caminar un anillo sin cuenta no termina nunca: el `for` seguiría la costura para siempre. Por eso el `__iter__` circular da exactamente `longitud` pasos y frena.

Con esa regla, el espejo se completa por herencia (capítulo 15): la subclase le suma a la clase base los dunders que faltan, sin tocar lo ya escrito.

```python
class ListaCircularEspejo(ListaCircularDoble):
    def __getitem__(self, indice):
        if indice < 0:
            indice += self.longitud
        if indice < 0 or indice >= self.longitud:
            raise IndexError(f"índice {indice} fuera del anillo")
        actual = self.inicio
        for _ in range(indice):
            actual = actual.siguiente
        return actual.valor

    def __setitem__(self, indice, valor):
        if indice < 0:
            indice += self.longitud
        if indice < 0 or indice >= self.longitud:
            raise IndexError(f"índice {indice} fuera del anillo")
        actual = self.inicio
        for _ in range(indice):
            actual = actual.siguiente
        actual.valor = valor

    def __iter__(self):
        actual = self.inicio
        for _ in range(self.longitud):
            yield actual.valor
            actual = actual.siguiente
```

`[i]` camina los mismos pasos que la sección 4, con un extra generoso: los índices negativos, como en las listas nativas. Probá el contrato completo:

```python
anillo = ListaCircularEspejo()
for valor in (1, 2, 3):
    anillo.push_final(valor)

print(anillo[0])          # 1
print(anillo[-1])         # 3
print(len(anillo))        # 3

anillo[-2] = "nuevo"
print(anillo)             # 1 -> nuevo -> 3 -> (vuelve a 1)

print(list(anillo))       # [1, 'nuevo', 3]

try:
    print(anillo[9])
except IndexError as error:
    print(error)          # índice 9 fuera del anillo
```

### Buscar y borrar en el anillo

Faltan dos movimientos para completar el contrato: preguntar por la posición de un valor (`index`, como en las listas del capítulo 5) y borrarlo. En la primera versión de estas piezas, `index` se quedaba a mitad de camino — nunca miraba el último nodo — y el borrado se olvidaba de restar de `longitud` y reventaba con el anillo de uno. Acá la caminata recorre `longitud` pasos, incluido el último, y el corte vuelve a coser la costura:

```python
class ListaCircularCompleta(ListaCircularEspejo):
    def index(self, valor):
        actual = self.inicio
        for posicion in range(self.longitud):
            if actual.valor == valor:
                return posicion
            actual = actual.siguiente
        raise ValueError(f"el valor {valor!r} no está en el anillo")

    def eliminar(self, valor):
        if self.inicio is None:
            raise ValueError("la lista está vacía")
        actual = self.inicio
        for _ in range(self.longitud):
            if actual.valor == valor:
                self.longitud -= 1
                if self.longitud == 0:
                    self.inicio = None
                    self.final = None
                else:
                    actual.anterior.siguiente = actual.siguiente
                    actual.siguiente.anterior = actual.anterior
                    if actual is self.inicio:
                        self.inicio = actual.siguiente
                    if actual is self.final:
                        self.final = actual.anterior
                return
            actual = actual.siguiente
        raise ValueError(f"el valor {valor!r} no está en el anillo")
```

El `eliminar` cosería igual en la lista doble de la sección 6; la única diferencia es que, además, avisa a las puntas cuando el corte las toca. Miralo quitar de todos lados — medio, cabeza y final — sin que el anillo se abra:

```python
cola = ListaCircularCompleta()
for valor in (1, 2, 3, 2, 4):
    cola.push_final(valor)

print(cola.index(3))        # 2
print(cola.index(4))        # 4  (el último nodo, que antes quedaba fuera)
print(len(cola))            # 5

cola.eliminar(2)            # borra la primera aparición
print(cola)                 # 1 -> 3 -> 2 -> 4 -> (vuelve a 1)
print(len(cola))            # 4

cola.eliminar(4)            # borrar el final no rompe el anillo
print(cola)                 # 1 -> 3 -> 2 -> (vuelve a 1)

try:
    cola.eliminar(99)
except ValueError as error:
    print(error)            # el valor 99 no está en el anillo
```

### La cuenta, con una vuelta

La circular no cambia la contabilidad de la sección 5: `push` y `pop` siguen en O(1) en ambos extremos, y `[i]`, `index` y `eliminar` siguen costando O(n) porque caminan. Lo único que cambió es la definición de "el siguiente del final": en vez de `None`, el `inicio`. Ahora la promesa de las ruletas se cumple con una línea — porque Python ya tiene la circular de fábrica, y del capítulo 5 te sabés su nombre:

```python
from collections import deque

turnos = deque(maxlen=3)
for turno in ("A", "B", "C", "D"):
    turnos.append(turno)
    print(list(turnos))
# ['A']
# ['A', 'B']
# ['A', 'B', 'C']
# ['B', 'C', 'D']
```

Cuando el `deque` con `maxlen` se llena, el dato más viejo sale solo: eso es un **búfer circular** listo para usar, la ruleta del turnero que la sección 6 anunciaba. Así cierra el arco del capítulo: la cadena simple que ya te sabés de memoria (secciones 1 a 4), la prima doble con su `deque` (sección 6) y el anillo que cose bordes (esta sección) — y la circular, Python te la presta en una línea.

## 8. Resumen y conceptos clave

Este capítulo arrancó donde cerró el anterior: el array paga caro el cambio, porque vivir todos juntos en un bloque implica desplazamiento y copia. La lista enlazada fue la apuesta contra esa idea: cada dato vive en su caja — el **nodo** —, y el nodo guarda un **puntero** al siguiente. Construiste la cadena con tus manos, nodo a nodo, y le enseñaste a la lista a contarse, mostrarse y recorrerse; tomaste nota de que llenarla con el `append` naive era O(n²), y lo arreglaste guardando el puntero al `final`. Mediste cada operación contra la lista nativa: ganás en las puntas — `prepend`, borrar la cabeza — y perdés en acceso por índice y en memoria (un nodo ronda los 152 bytes contra los 8 del puntero nativo, y la caché no perdona los nodos tirados). Después conociste a la prima **doble**: dos punteros por nodo, O(1) en los dos extremos, y con ella a la **pila** (LIFO) y la **cola** (FIFO) como contratos sobre la misma máquina. Y cuando la ficha cayó, era la que el libro te debía: el `deque` del capítulo 5 es eso, una lista doble en C. Y la prima **circular** le cosió los bordes — `final.siguiente` que vuelve al `inicio` —, con el espejo completo del capítulo 13 (`[i]`, `[i] = valor`, `for`, `index`, `eliminar`) y el `deque` con `maxlen` como la ruleta de fábrica para dar vueltas sin recomenzar. La lección final, la más buscada: **la lista enlazada simple rara vez gana en Python** — se estudia por fundamento, se asoma en el `deque`, y su valor está en el lugar donde los nodos mueven el dato rápido: los extremos, y nada más.

Repasá el checklist antes de seguir:

- [ ] El array (capítulo 20) cambia caro: insertar/borrar desplaza vecinos y crecer copia todo.
- [ ] La lista enlazada guarda cada dato en un **nodo** con un **puntero** al siguiente — el dato vive disperso y se pasa el testigo.
- [ ] El último nodo apunta a `None`: es la marca de "hasta acá llegamos".
- [ ] Recorrer una lista enlazada es **caminarla**: la lista nativa salta con `[i]`, la cadena camina nodo por nodo.
- [ ] `append` naive es O(n) y llenar la lista, **O(n²)** (la suma de Gauss, capítulo 19); guardando el puntero al `final`, `append` pasa a O(1).
- [ ] `prepend` es O(1): la ventaja histórica frente a `insert(0)` O(n) de la lista nativa.
- [ ] Borrar un valor exige el puntero `previo` (para coser la cadena) y dispara `ValueError` si no existe.
- [ ] Blindar para producción: tope `MAX_LONGITUD` y validación de entrada antes de tocar la estructura.
- [ ] Memoria medida: nodo ≈ **152 bytes** (48 + 104 de su diccionario) contra **8 bytes** de puntero de la lista nativa; un millón de nodos ≈ 152 MB.
- [ ] **Localidad de caché**: los bloques contiguos se traen por tandas a la caché; los nodos dispersos pagan fallos en cada eslabón.
- [ ] La **lista doble** tiene `anterior` y `siguiente`: push/pop O(1) en los dos extremos.
- [ ] Pila (LIFO) y cola (FIFO) son contratos de esa lista doble; el `deque` de `collections` es una lista doble en C.
- [ ] La **lista circular** cose los bordes: `final.siguiente` vuelve al `inicio` e `inicio.anterior` al `final`; el `None` desaparece y la lista queda con cero extremos.
- [ ] El **espejo completo** en el anillo: `[i]` y `[i] = valor` con índices negativos, `for` que cuenta `longitud` (si no, da vueltas eternas), y `index`/`eliminar` que re-cosen el anillo y no descuadran `longitud`.
- [ ] En Python, la lista enlazada simple rara vez gana en la cancha: se estudia por fundamento y se usa en sus casos puntuales.

## 9. Ejercicios

1. **La cadena a mano.** Con la clase `Nodo` de la sección 2, armá la cadena `"rojo" -> "verde" -> "azul"` enganchando los nodos a mano, y con un `while` imprimí cada valor de punta a punta.

2. **Medir la caminata.** Escribí una función `pasos_append(n)` que cuente, al estilo de la sección 3, cuántas preguntas de "¿tenés siguiente?" se hacen al llenar una lista vacía con `n` elementos usando el `append` naive. Comprobalo para `n = 4`, `10` y `500` y fijate que coincida con la suma de la sección 3: 0 + 1 + … + (n − 1).

3. **El buscador que camina.** Usando la `ListaEnlazada` de la sección 4 (con su `__iter__`), escribí una función `buscar(lista, valor)` que devuelva la posición donde aparece un valor, o `-1` si no está. Probala con una lista de `1, 3, 9` y buscá el `9` y un valor que no exista.

4. **El iterador al rescate.** Con la misma lista, cargá los valores `5, 3, 12` y mostrá la lista como Python (`list(lista)`), el máximo (`max`) y cuántos elementos tiene (`len`). Todo funciona sin un solo método más, porque `__iter__` y `__len__` ya trabajan.

5. **Leer de atrás para adelante.** Con la `ListaDoble` de la sección 6, cargá `1, 2, 3, 4` por el final y recorré desde `final` usando `anterior` para imprimir cada valor de atrás hacia adelante.

6. **La cola del bar.** Con la clase `Cola` de la sección 6, simulá una fila de pedidos: entran `"café"`, `"tostada"` y `"jugo"`. Sacá el primero que entró (mostralo) y después mostrá los que quedan en la cola con `list(cola)`.

7. **El índice con criterio (para valientes).** Implementá `__getitem__` en la `ListaEnlazada` de la sección 4 para que `lista[i]` funcione caminando, con un `IndexError` con mensaje claro cuando el índice no exista. Con los valores `10, 20, 30`, mostrá `lista[0]`, `lista[1]`, y qué pasa con `lista[5]`.

8. **El anillo que no se corta.** Usando la `ListaCircularDoble` de la sección 7, escribí una función `eliminar_todos(anillo, valor)` que borre **todas** las apariciones de un valor sin cortar la costura ni descuadrar `longitud`, y levante `ValueError` si el valor no está. Probala con `1, 2, 1, 3, 1` eliminando el `1`, y mostrá el anillo y su `len`.

9. **Leer el anillo al revés (para valientes).** Escribí una función `indice_ultimo(anillo, valor)` que devuelva la posición de la **última** aparición caminando desde `final` con `anterior` — el índice contado desde `inicio`, como en las listas — y levante `ValueError` si el valor no existe. Probala con `1, 2, 1, 3, 1`: la última posición del `1` y qué pasa con un valor ausente.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. La cadena a mano
class Nodo:
    def __init__(self, valor, siguiente=None):
        self.valor = valor
        self.siguiente = siguiente

n1 = Nodo("rojo")
n2 = Nodo("verde")
n3 = Nodo("azul")
n1.siguiente = n2
n2.siguiente = n3

actual = n1
while actual is not None:
    print(actual.valor)
    actual = actual.siguiente
# rojo
# verde
# azul


# 2. Medir la caminata
class Nodo:
    def __init__(self, valor, siguiente=None):
        self.valor = valor
        self.siguiente = siguiente


def pasos_append(n):
    cabeza = None
    pasos = 0
    for _ in range(n):
        if cabeza is None:
            cabeza = Nodo("x")
        else:
            actual = cabeza
            while actual.siguiente is not None:
                pasos += 1
                actual = actual.siguiente
            pasos += 1          # la pregunta que descubre el final
            actual.siguiente = Nodo("x")
    return pasos

for n in (4, 10, 500):
    print(n, pasos_append(n))
# 4 6
# 10 45
# 500 124750


# 3. El buscador que camina
class ListaEnlazada:
    def __init__(self):
        self.cabeza = None
        self.final = None
        self.longitud = 0

    def append(self, valor):
        nuevo = Nodo(valor)
        if self.final is None:
            self.cabeza = nuevo
        else:
            self.final.siguiente = nuevo
        self.final = nuevo
        self.longitud += 1

    def __iter__(self):
        actual = self.cabeza
        while actual is not None:
            yield actual.valor
            actual = actual.siguiente


def buscar(lista, valor):
    posicion = 0
    for dato in lista:
        if dato == valor:
            return posicion
        posicion += 1
    return -1


numeros = ListaEnlazada()
for valor in (1, 3, 9):
    numeros.append(valor)

print(buscar(numeros, 9))     # 2
print(buscar(numeros, 100))   # -1


# 4. El iterador al rescate
class ListaEnlazada:
    def __init__(self):
        self.cabeza = None
        self.final = None
        self.longitud = 0

    def append(self, valor):
        nuevo = Nodo(valor)
        if self.final is None:
            self.cabeza = nuevo
        else:
            self.final.siguiente = nuevo
        self.final = nuevo
        self.longitud += 1

    def __len__(self):
        return self.longitud

    def __iter__(self):
        actual = self.cabeza
        while actual is not None:
            yield actual.valor
            actual = actual.siguiente


notas = ListaEnlazada()
for valor in (5, 3, 12):
    notas.append(valor)

print(list(notas))    # [5, 3, 12]
print(max(notas))     # 12
print(len(notas))     # 3


# 5. Leer de atrás para adelante
class NodoDoble:
    def __init__(self, valor, anterior=None, siguiente=None):
        self.valor = valor
        self.anterior = anterior
        self.siguiente = siguiente


class ListaDoble:
    def __init__(self):
        self.inicio = None
        self.final = None
        self.longitud = 0

    def push_inicio(self, valor):
        nuevo = NodoDoble(valor, siguiente=self.inicio)
        if self.inicio is not None:
            self.inicio.anterior = nuevo
        else:
            self.final = nuevo
        self.inicio = nuevo
        self.longitud += 1

    def push_final(self, valor):
        nuevo = NodoDoble(valor, anterior=self.final)
        if self.final is not None:
            self.final.siguiente = nuevo
        else:
            self.inicio = nuevo
        self.final = nuevo
        self.longitud += 1

    def pop_inicio(self):
        if self.inicio is None:
            raise ValueError("la lista está vacía")
        valor = self.inicio.valor
        self.inicio = self.inicio.siguiente
        if self.inicio is not None:
            self.inicio.anterior = None
        else:
            self.final = None
        self.longitud -= 1
        return valor

    def pop_final(self):
        if self.final is None:
            raise ValueError("la lista está vacía")
        valor = self.final.valor
        self.final = self.final.anterior
        if self.final is not None:
            self.final.siguiente = None
        else:
            self.inicio = None
        self.longitud -= 1
        return valor

    def __len__(self):
        return self.longitud

    def __iter__(self):
        actual = self.inicio
        while actual is not None:
            yield actual.valor
            actual = actual.siguiente


doble = ListaDoble()
for valor in (1, 2, 3, 4):
    doble.push_final(valor)

actual = doble.final
while actual is not None:
    print(actual.valor)
    actual = actual.anterior
# 4
# 3
# 2
# 1


# 6. La cola del bar
class Pila(ListaDoble):
    def push(self, valor):
        self.push_final(valor)

    def pop(self):
        return self.pop_final()


class Cola(ListaDoble):
    def enqueue(self, valor):
        self.push_final(valor)

    def dequeue(self):
        return self.pop_inicio()


pedidos = Cola()
for ronda in ("café", "tostada", "jugo"):
    pedidos.enqueue(ronda)

print(pedidos.dequeue())   # café
print(list(pedidos))       # ['tostada', 'jugo']


# 7. El índice con criterio
class ListaEnlazada:
    def __init__(self):
        self.cabeza = None
        self.final = None
        self.longitud = 0

    def append(self, valor):
        nuevo = Nodo(valor)
        if self.final is None:
            self.cabeza = nuevo
        else:
            self.final.siguiente = nuevo
        self.final = nuevo
        self.longitud += 1

    def __getitem__(self, indice):
        actual = self.cabeza
        posicion = 0
        while actual is not None:
            if posicion == indice:
                return actual.valor
            actual = actual.siguiente
            posicion += 1
        raise IndexError(f"índice {indice} fuera de la lista")


decenas = ListaEnlazada()
for valor in (10, 20, 30):
    decenas.append(valor)

print(decenas[0])         # 10
print(decenas[1])         # 20

try:
    print(decenas[5])
except IndexError as error:
    print(error)          # índice 5 fuera de la lista


# 8. El anillo que no se corta
class NodoDoble:
    def __init__(self, valor, anterior=None, siguiente=None):
        self.valor = valor
        self.anterior = anterior
        self.siguiente = siguiente


class ListaCircular:
    def __init__(self):
        self.inicio = None
        self.final = None
        self.longitud = 0

    def push_final(self, valor):
        nuevo = NodoDoble(valor)
        nuevo.siguiente = self.inicio if self.inicio else nuevo
        nuevo.anterior = self.final if self.final else nuevo
        if self.final is None:
            self.inicio = nuevo
        else:
            self.final.siguiente = nuevo
            self.inicio.anterior = nuevo
        self.final = nuevo
        self.longitud += 1

    def __len__(self):
        return self.longitud

    def __iter__(self):
        actual = self.inicio
        for _ in range(self.longitud):
            yield actual.valor
            actual = actual.siguiente

    def __str__(self):
        if self.inicio is None:
            return "[]"
        piezas = []
        actual = self.inicio
        for _ in range(self.longitud):
            piezas.append(str(actual.valor))
            actual = actual.siguiente
        return " -> ".join(piezas) + f" -> (vuelve a {self.inicio.valor})"


def eliminar_todos(anillo, valor):
    encontro = False
    actual = anillo.inicio
    for _ in range(anillo.longitud):
        siguiente = actual.siguiente
        if actual.valor == valor:
            anillo.longitud -= 1
            if anillo.longitud == 0:
                anillo.inicio = None
                anillo.final = None
            else:
                actual.anterior.siguiente = actual.siguiente
                actual.siguiente.anterior = actual.anterior
                if actual is anillo.inicio:
                    anillo.inicio = actual.siguiente
                if actual is anillo.final:
                    anillo.final = actual.anterior
            encontro = True
        actual = siguiente
    if not encontro:
        raise ValueError(f"el valor {valor!r} no está en el anillo")


anillo = ListaCircular()
for valor in (1, 2, 1, 3, 1):
    anillo.push_final(valor)

eliminar_todos(anillo, 1)
print(anillo)       # 2 -> 3 -> (vuelve a 2)
print(len(anillo))  # 2

try:
    eliminar_todos(anillo, 99)
except ValueError as error:
    print(error)    # el valor 99 no está en el anillo


# 9. Leer el anillo al revés
def indice_ultimo(anillo, valor):
    actual = anillo.final
    for atras in range(anillo.longitud):
        posicion = anillo.longitud - 1 - atras
        if actual.valor == valor:
            return posicion
        actual = actual.anterior
    raise ValueError(f"el valor {valor!r} no está en el anillo")


otro_anillo = ListaCircular()
for valor in (1, 2, 1, 3, 1):
    otro_anillo.push_final(valor)

print(indice_ultimo(otro_anillo, 1))   # 4
print(indice_ultimo(otro_anillo, 3))   # 3

try:
    print(indice_ultimo(otro_anillo, 99))
except ValueError as error:
    print(error)                       # el valor 99 no está en el anillo
```