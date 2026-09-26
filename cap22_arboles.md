# Capítulo 22 — Árboles: el dato que se ramifica

En el capítulo 21 construiste una cadena: cada dato en su caja, una sola puerta de salida, y el testigo saltando de caja en caja hasta el `None`. En el capítulo 19 te había quedado una promesa colgando: que *los árboles* iban a ofrecer de verdad las búsquedas O(log n) que la búsqueda binaria te mostró en teoría. Hoy la cobrás. Un árbol es la apuesta en la que el nodo **no se pasa un único testigo: se ramifica**. En vez de una fila donde solo podés mirar para adelante, cada caja abre dos puertas — izquierda y derecha — y con esa decisión simple, la cadena se convierte en una **jerarquía**: algo que ya manejás todos los días sin llamarlo árbol.

Pensá en un organigrama de trabajo: arriba, un director; abajo, jefes; más abajo, equipos; al final, cada persona. Pensá en las carpetas de tu disco, del capítulo 10: una carpeta contiene carpetas, que contienen carpetas, hasta llegar a los archivos. Pensá en las líneas de un árbol genealógico o en la estructura de un sitio web (una sección que abre subsecciones). Todos son el mismo patrón: **un elemento tiene dependientes, y cada dependiente puede tener los suyos**. Contra lo que intuís, ese patrón no es decorativo: es la forma en que una máquina decide y busca rápido, y acá vas a descubrir por qué.

En este capítulo vas a construir el árbol binario con tus manos, a ponerle número a cada parte — raíz, hijos, hojas, niveles — y a recorrerlo de las tres maneras clásicas. Es el escenario que el capítulo 10 te señaló cuando prometió que la recursión brilla en las estructuras *recursivas por naturaleza*: esa cuenta prometida se cobra acá, y de qué manera. Vas a ver cómo una simple regla de ordenamiento convierte a un árbol en un **buscador veloz** que divide el problema por la mitad en cada paso (la promesa del capítulo 19, cumplida) y a borrar nodos sin romper la jerarquía. Después vas a encontrarte con el enemigo silencioso de todo esto — el árbol que se *tuerce* hasta parecerse a la lista enlazada del capítulo 21 — y a conocer la respuesta moderna a ese problema: los árboles **balanceados**, con su versión más famosa, el **AVL**, que se endereza solo con rotaciones. De paso vas a aplicar todo a un caso real (tu agenda de cumpleaños), y a cerrar con la pasada de producción que ya es costumbre en esta parte del libro. Al final, las búsquedas que eran O(n) van a pasar a O(log n) delante de tus ojos.

---

## 1. La jerarquía: un nodo, dos puertas

Volvé con la mente a la lista enlazada del capítulo 21. Su limitación es orgánica: cada caja abre **una sola puerta** — el `siguiente` — y por eso la cadena solo sabe caminar en fila. Para llegar al dato cinco hay que atravesar a los cuatro anteriores; no hay atajos porque la información "quién viene después" es una sola y lineal. El árbol nace de un cambio mínimo y potente: **cada caja puede tener más de un sucesor**. Y en su versión más importante, exactamente dos.

Antes de ver el código, conviene bautizar cada parte del árbol, porque todo el capítulo habla con este vocabulario. Un árbol es una estructura **jerárquica** (*jerarquía*, del griego "gobierno sagrado": niveles donde cada uno está por encima o por debajo de otro) compuesta de:

| Término | Qué es |
|---|---|
| **Nodo** | cada caja del árbol, con su dato y sus referencias a los hijos |
| **Raíz** | el nodo superior, el único sin padre; es la entrada al árbol |
| **Hijo / padre** | cada nodo (salvo la raíz) tiene un padre y puede tener hijos |
| **Hermano** | los hijos de un mismo nodo |
| **Hoja** | un nodo sin hijos — el final de una rama |
| **Nivel / profundidad** | qué tan lejos está un nodo de la raíz; la raíz es nivel 0, sus hijos nivel 1, y así |
| **Altura** | la profundidad máxima del árbol: el camino más largo de raíz a hoja |
| **Subárbol** | cualquier nodo, con todos sus descendientes, es la raíz de su propio árbol chico |

> **Dato clave:** un árbol es una lista enlazada *con memoria de que el mundo no es lineal*. La lista solo conoce a su siguiente; el árbol conoce a sus descendientes, y a cada nivel la cantidad de posibles destinos se duplica. Esa multiplicación por dos es exactamente la que va a convertir caminatas O(n) en saltos O(log n).

La palabra "árbol" en informática tiene una particularidad que confunde siempre: **se dibuja con la raíz arriba y las hojas abajo**, al revés del árbol biológico. Cuando en el capítulo 21 veías `None` hacia abajo en la cadena, acá la raíz está en la cima y las ramas cuelgan. Si querés, pensalo como un árbol genealógico invertido: el tronco arriba, las ramas bajando. En ese sentido, el árbol **general** — un nodo con cualquier cantidad de hijos, sin techo — es la forma más fiel del concepto (el sistema de archivos del capítulo 10 es un árbol general: una carpeta tiene muchas subcarpetas). Pero el capítulo entero, y la informática entera, casi siempre usa una versión acotada que lo hace todo más simple y más poderoso a la vez: el **árbol binario**.

¿Por qué binario, entonces, si el general es más fiel? Tres razones que se juntan. La primera, la **simplicidad**: con dos hijos fijos — izquierdo y derecho — la lógica de insertar, eliminar y buscar es directa; con "muchos hijos" cada operación arrastra un bucle extra y muchas decisiones. La segunda, el **ordenamiento**: un árbol binario puede mantener una regla — lo que va a la izquierda es menor, lo que va a la derecha es mayor — y esa regla es la que hace que la búsqueda se parta por la mitad en cada paso (el secreto de la promesa O(log n)). Con muchos hijos, mantener esa regla razonable se vuelve un problema de diseño en sí. Y la tercera, la **historia**: los árboles binarios están en los cimientos de la disciplina — montículos, árboles AVL, árboles rojo-negro — y conocerlos es conocer el idioma en el que se hablan todas las bases de datos. Un árbol de *muchos* hijos se usa en casos puntuales (los **árboles B** de las bases de datos, que vas a nombrar en la sección 8), pero el binario es la puerta de entrada a todos.

Fijate entonces el contrato que firmamos para el resto del capítulo: **un nodo, un dato, y a lo sumo dos puertas** — `izquierdo` y `derecho`. Esa es toda la infraestructura. Todo lo que va a venir — insertar, ordenar, buscar, borrar, balancear — son destrezas sobre esa caja de dos puertas.

## 2. El nodo que se ramifica

La caja mínima del capítulo 21 tenía dos atributos: `valor` y `siguiente`. La caja del árbol tiene tres: el `valor` y dos referencias, `izquierdo` y `derecho`, que nacen apuntando a `None` — la misma convención de "hasta acá llegamos" de la lista enlazada, pero en duplicado:

```python
class Nodo:
    def __init__(self, valor, izquierdo=None, derecho=None):
        self.valor = valor
        self.izquierdo = izquierdo
        self.derecho = derecho
```

Dos puertas, y un dato: la caja es una réplica de la del capítulo 21, con un testigo extra. Y acá arranca lo interesante: un árbol no es solo una estructura — es una estructura **con reglas**. El árbol binario que ordena se llama **árbol binario de búsqueda** — en inglés, *binary search tree*, **BST** — y su regla es justamente la que le da nombre: para cada nodo, **todo** lo que cuelga de su `izquierdo` es menor que su `valor`, y **todo** lo que cuelga de su `derecho` es mayor. No basta con que los hijos directos cumplan: la regla vale para el subárbol entero.

> **Importante (no confundir con el capítulo 19):** acá "búsqueda" no se refiere a la *búsqueda binaria* — el algoritmo que viste en el capítulo 19, que divide una lista ordenada por la mitad. El **BST** es la *estructura* que hace posible esa búsqueda; vas a ver la conexión exacta en la sección 5. Por ahora, guardá el nombre completo: **árbol binario de búsqueda**.

> **Importante:** la propiedad del BST no es "izquierdo menor, derecho mayor" aplicada a los hijos de a uno por vez. Es global: el subárbol izquierdo *completo* es menor que el nodo, y el derecho *completo* es mayor. Si la violás en un solo lugar, el ordenamiento entero se rompe, y los recorridos de la próxima sección dejan de salir ordenados.

Ahora construí la clase que administra el árbol. Como la lista enlazada, sabe dónde está su punto de entrada — la `raiz` — y delega el resto en la recursión del capítulo 10:

```python
class ArbolBinario:
    def __init__(self):
        self.raiz = None

    def insertar(self, valor):
        if self.raiz is None:
            self.raiz = Nodo(valor)
        else:
            self._insertar(self.raiz, valor)

    def _insertar(self, nodo, valor):
        if valor < nodo.valor:
            if nodo.izquierdo is None:
                nodo.izquierdo = Nodo(valor)
            else:
                self._insertar(nodo.izquierdo, valor)
        else:
            if nodo.derecho is None:
                nodo.derecho = Nodo(valor)
            else:
                self._insertar(nodo.derecho, valor)
```

Mirá el `_insertar` y fijate que la estructura de la función **calca** la estructura de los datos: es la lección del capítulo 10 en estado puro. El árbol es recursivo — un subárbol es un árbol — y el código recursivo acompaña esa forma sin zancos: si el valor es menor, bajás por la izquierda; si es mayor, por la derecha; cuando llegás a un hueco (`None`), plantás el nodo. No escribiste un bucle porque no hay camino predefinido: el camino lo decide cada dato comparando. Este es el momento que el capítulo 10 te anunció: los datos **recursivos por naturaleza** — un árbol que contiene Árboles — son el escenario donde la recursión brilla de verdad. El `_insertar` ni siquiera pregunta si está profundo o superficial: se llama a sí mismo con un problema más chico y confía.

> **Dato clave — el código se parece a los datos:** la señal que ya conocés del capítulo 10 — "la estructura de los datos es recursiva" — vuelve a aparecer acá con todo su poder. Cuando la recursión es *por naturaleza* — listas anidadas, carpetas, árboles — el código que la recorre se lee como la propia estructura, sin un solo bucle; la promesa que te dejó el capítulo 10 se cumple método a método. El `_insertar` no dice "recorré la fila de arriba a abajo"; dice "si es menor, andá a la izquierda; si es mayor, andá a la derecha", y la recursión se encarga del resto. Todo el capítulo va a usar esta misma confianza.

Armá un árbol de verdad y miralo por dentro, enganchando atributos a mano como en capítulo 21:

```python
arbol = ArbolBinario()
for numero in (8, 3, 10, 1, 6, 4, 7, 14, 13):
    arbol.insertar(numero)

print(arbol.raiz.valor)                          # 8
print(arbol.raiz.izquierdo.valor)                # 3
print(arbol.raiz.izquierdo.izquierdo.valor)      # 1
print(arbol.raiz.izquierdo.derecho.derecho.valor)  # 7
print(arbol.raiz.derecho.derecho.izquierdo.valor)  # 13
```

La noche de las cinco líneas deja clara la geografía: el `8` es la raíz, el `3` cuelga a su izquierda, el `10` a su derecha; el `1` es la rama izquierda del `3`, el `6` la derecha, y así hasta las hojas. Cada `insertar` fue una decisión binaria que terminó plantando el nodo en su lugar único.

Y ahora la pregunta que te tiene que saltar sola: ¿qué pasaría si, con este mismo `ArbolBinario`, insertás los números **en orden** — `1, 2, 3, 4, 5`? Cada nuevo dato es más grande que todos los anteriores, así que cada uno cae siempre a la *derecha* de todos los visitados. El árbol no se ramifica: se convierte en una lista enlazada vertical, colgada de la raíz. Esa degeneración es el fantasma de la sección 7, pero antes de asustarnos, aprendamos a caminar el árbol como se debe.

## 3. `inorden`: el recorrido que sale ordenado

Una cadena se recorre de una sola manera: del principio al final. Un árbol es más rico: cada rama es una decisión, y la decisión define el orden en que vemos los datos. Hay tres recorridos clásicos para árboles binarios, y la diferencia es *cuándo* visitás el nodo respecto de sus hijos:

- **`inorden`** — primero el hijo izquierdo, después el nodo, después el derecho.
- **`preorden`** — primero el nodo, después los hijos.
- **`postorden`** — primero los hijos, después el nodo.

Empezá por el más útil, el `inorden`. Su receta, si arrancás desde la raíz, es: *bajá por la izquierda hasta el fondo, visitá al llegar, y subí de rama en rama*. En código recursivo, la receta es tres líneas que se leen a sí mismas:

```python
class Nodo:
    def __init__(self, valor, izquierdo=None, derecho=None):
        self.valor = valor
        self.izquierdo = izquierdo
        self.derecho = derecho


class ArbolBinario:
    def __init__(self):
        self.raiz = None

    def insertar(self, valor):
        if self.raiz is None:
            self.raiz = Nodo(valor)
        else:
            self._insertar(self.raiz, valor)

    def _insertar(self, nodo, valor):
        if valor < nodo.valor:
            if nodo.izquierdo is None:
                nodo.izquierdo = Nodo(valor)
            else:
                self._insertar(nodo.izquierdo, valor)
        else:
            if nodo.derecho is None:
                nodo.derecho = Nodo(valor)
            else:
                self._insertar(nodo.derecho, valor)

    def inorden(self):
        visitados = []
        self._inorden(self.raiz, visitados)
        return visitados

    def _inorden(self, nodo, visitados):
        if nodo is not None:
            self._inorden(nodo.izquierdo, visitados)
            visitados.append(nodo.valor)
            self._inorden(nodo.derecho, visitados)
```

El `_inorden` se lee como el anuncio de la receta: *izquierda, nodo, derecha*. Y ahora la magia que motiva todo el capítulo. Fijate que los datos entraron **desordenados** — `8, 3, 10, 1, 6, 4, 7, 14, 13` — y sin embargo:

```python
arbol = ArbolBinario()
for numero in (8, 3, 10, 1, 6, 4, 7, 14, 13):
    arbol.insertar(numero)

print(arbol.inorden())   # [1, 3, 4, 6, 7, 8, 10, 13, 14]
```

**Sale ordenado.** No es suerte: es la propiedad del BST — el *árbol binario de búsqueda* de la sección 2 — cayendo en su premio merecido. Como el subárbol izquierdo de cada nodo es menor que él, visitándolo primero el recorrido siempre sube; y como el derecho es mayor, al ir después el recorrido nunca baja. La regla *izquierda → nodo → derecha* recorre el árbol en orden ascendente sin hacer una sola comparación de más. Insertar cobra, ordenar sale gratis: cargaste nueve datos, y la lista ordenada es un subproducto del árbol.

> **Dato clave:** `inorden` es el "imprimí todo ordenado" del BST. En una lista nativa ordenar te costaba O(n log n) (lo viste en el capítulo 19); en un árbol binario, cargar `n` elementos ordenados cuesta O(n log n) de paso, y cada vez que querés la lista completa en orden solo caminás los `n` en O(n). El árbol *retiene* el orden: lo paga de a poco, al insertar, y lo cobra al instante.

Miralo desde el espejo del capítulo 13: si el árbol sabe recorrer sus datos en orden, puede prestarle ese derecho a Python para que `for`, `list()`, `max()` y los amigos trabajen solos. El `__iter__` con `yield` del generador (capítulo 10) convierte el `inorden` en un iterador:

```python
    def __iter__(self):
        yield from self.inorden()
```

Con esa línea, un árbol se comporta como cualquier colección del capítulo 5. Compartido con Python en un solo gesto:

```python
for numero in arbol:
    print(numero, end=" ")
# 1 3 4 6 7 8 10 13 14

print(list(arbol))   # [1, 3, 4, 6, 7, 8, 10, 13, 14]
print(max(arbol))    # 14
print(min(arbol))    # 1
```

Y los otros dos recorridos, que vas a necesitar en la sección 8 (y en la Parte VIII, cuando los árboles se llamen *grafos*), cambian un solo renglón:

```python
    def preorden(self):
        visitados = []
        self._preorden(self.raiz, visitados)
        return visitados

    def _preorden(self, nodo, visitados):
        if nodo is not None:
            visitados.append(nodo.valor)
            self._preorden(nodo.izquierdo, visitados)
            self._preorden(nodo.derecho, visitados)

    def postorden(self):
        visitados = []
        self._postorden(self.raiz, visitados)
        return visitados

    def _postorden(self, nodo, visitados):
        if nodo is not None:
            self._postorden(nodo.izquierdo, visitados)
            self._postorden(nodo.derecho, visitados)
            visitados.append(nodo.valor)
```

Compará los tres sobre el mismo árbol y fijate qué lugar ocupa el nodo en cada lista:

```python
print(arbol.inorden())    # [1, 3, 4, 6, 7, 8, 10, 13, 14]
print(arbol.preorden())   # [8, 3, 1, 6, 4, 7, 10, 14, 13]
print(arbol.postorden())  # [1, 4, 7, 6, 3, 13, 14, 10, 8]
```

Esto vale la pena mirarlo con lupa. El `preorden` empieza por la raíz — `8` — y describe el árbol *de arriba para abajo*: es el orden en que un montón de sistemas guardan y copian árboles (primero el padre, después los hijos), y el que vas a usar para *serializar* estructuras en la Parte IX. El `postorden` termina en la raíz — con el `8` de cierre — y se usa cuando los hijos tienen que completarse antes que el padre (calcular un resultado que depende de los subárboles, como un evaluador de expresiones). Y el `inorden` es el único que ordena. Tres cambiaformas sobre el mismo árbol, una sola diferencia de posición de dos líneas. Y pensá qué frase los describe a los tres sin dibujarlos: *"visitá la izquierda, visitá el nodo, visitá la derecha"*, con el nodo movido de lugar en cada uno — pura recursión del capítulo 10, sin un bucle, sin una pila propia. El capítulo 10 te lo prometió, y este es el escenario que tenía en mente.

## 4. El árbol que se ve

Los capítulos 20 y 21 te enseñaron que una estructura de datos honesta sabe **mostrarse**: la lista enlazada armaba su flecha `->`; el árbol va a armar algo más parecido a un mapa. El truco clásico para dibujar un árbol en texto plano es rotarlo 90 grados: la raíz a la izquierda, las ramas subiendo hacia la derecha (la rama derecha arriba) y bajando a la izquierda (la rama izquierda abajo). Cada nivel se identa con un `|  ` por cada nivel de profundidad que tenga debajo.

La receta es recursiva, otra vez, y otra vez *calca* la estructura: dibujá el subárbol derecho, dibujá el nodo, dibujá el subárbol izquierdo:

```python
class Nodo:
    def __init__(self, valor, izquierdo=None, derecho=None):
        self.valor = valor
        self.izquierdo = izquierdo
        self.derecho = derecho


class ArbolBinario:
    def __init__(self):
        self.raiz = None

    def insertar(self, valor):
        if self.raiz is None:
            self.raiz = Nodo(valor)
        else:
            self._insertar(self.raiz, valor)

    def _insertar(self, nodo, valor):
        if valor < nodo.valor:
            if nodo.izquierdo is None:
                nodo.izquierdo = Nodo(valor)
            else:
                self._insertar(nodo.izquierdo, valor)
        else:
            if nodo.derecho is None:
                nodo.derecho = Nodo(valor)
            else:
                self._insertar(nodo.derecho, valor)

    def inorden(self):
        visitados = []
        self._inorden(self.raiz, visitados)
        return visitados

    def _inorden(self, nodo, visitados):
        if nodo is not None:
            self._inorden(nodo.izquierdo, visitados)
            visitados.append(nodo.valor)
            self._inorden(nodo.derecho, visitados)

    def __str__(self):
        return self._detalle(self.raiz, 0)

    def _detalle(self, nodo, nivel):
        resultado = ""
        if nodo is not None:
            resultado += self._detalle(nodo.derecho, nivel + 1)
            resultado += "|  " * nivel + str(nodo.valor) + "\n"
            resultado += self._detalle(nodo.izquierdo, nivel + 1)
        return resultado
```

`_detalle` pide tres cosas y el resto lo hace la recursión: dibujar la derecha *más adentro* (porque es más arriba del mapa), escribir el nodo con su sangría según el nivel, y dibujar la izquierda. Probá el mapa con los nueve datos de siempre:

```python
arbol = ArbolBinario()
for numero in (8, 3, 10, 1, 6, 4, 7, 14, 13):
    arbol.insertar(numero)

print(arbol)
```

El dibujo, mirándolo de costado, es el árbol de la sección 2:

```
|  |  14
|  |  |  13
|  10
8
|  |  |  7
|  |  6
|  |  |  4
|  3
|  |  1
```

Leelo así: el `8` es la raíz (sangría cero, a la izquierda del todo). Arriba de él, con una sangría, vive su rama derecha (`10`), y arriba de esa, la del `10` (`14` con `13` de hijo izquierdo). Abajo del `8`, con una sangría, su rama izquierda (`3`), y debajo de `3` el `6` con sus hojas `7` y `4`, y el `1` colgando de `3`. Cuanto más a la derecha del texto, más profundo en el árbol. Cuando en las secciones que vienen mires un árbol "de costado" en la consola, esta es la convención: **derecha arriba, izquierda abajo, sangría = profundidad**.

Mirá qué pensó el `_detalle` para dibujar esto: *"primero la derecha, más adentro; después mi valor; después la izquierda, más adentro"* — tres líneas que se llaman a sí mismas hasta tocar `None`. Si acá se hubiera intentado con bucles, habrías tenido que mantener a mano una pila de nodos pendientes y un vector de sangrías. La recursión no solo *lee* el árbol: también lo *dibuja* como quien lo abraza por rama. Otra promesa del capítulo 10, cumplida sin vuelta.

> **Dato clave:** `print(arbol)` no muestra "la lista de los datos": muestra la *forma* del árbol. Y la forma importa — la sección 7 entera va a depender de mirar si un árbol es achaparrado (pocos niveles, mucha rama) o es un palo torcido (un nivel por dato). El `__str__` no es cosmética: es tu ojo clínico sobre la estructura.

## 5. Buscar: partir por la mitad

Acá se paga la promesa más vieja de la Parte VIII. En una lista nativa, preguntar `if dato in lista` recorre de a uno — O(n), te dijo capítulo 19. En un `set` o un `dict`, es O(1) por magia de tabla hash (estructura que vas a desarmar en secciones por venir). En un **árbol binario**, la respuesta es otra pieza clásica del teatro: la **búsqueda binaria** del capítulo 19, pero ahora viva en una estructura que la sostiene en el tiempo.

El truco es la propiedad del BST: en cada nodo, una sola comparación te dice *a qué mitad ir*. ¿Buscás el 13? Empezá por la raíz. 13 > 8, así que no puede estar en todo el subárbol izquierdo — tirás la mitad de los datos sin mirarlos. Seguís por la derecha: 13 > 10, derecha de nuevo; 13 < 14, ahora la izquierda; `13` — lo encontraste en **cuatro comparaciones** sobre nueve datos. Cada paso descarta la mitad, y el número de pasos es la altura del árbol.

Escribilo, otra vez con la estructura calcando los datos:

```python
class Nodo:
    def __init__(self, valor, izquierdo=None, derecho=None):
        self.valor = valor
        self.izquierdo = izquierdo
        self.derecho = derecho


class ArbolBinario:
    def __init__(self):
        self.raiz = None

    def insertar(self, valor):
        if self.raiz is None:
            self.raiz = Nodo(valor)
        else:
            self._insertar(self.raiz, valor)

    def _insertar(self, nodo, valor):
        if valor < nodo.valor:
            if nodo.izquierdo is None:
                nodo.izquierdo = Nodo(valor)
            else:
                self._insertar(nodo.izquierdo, valor)
        else:
            if nodo.derecho is None:
                nodo.derecho = Nodo(valor)
            else:
                self._insertar(nodo.derecho, valor)

    def inorden(self):
        visitados = []
        self._inorden(self.raiz, visitados)
        return visitados

    def _inorden(self, nodo, visitados):
        if nodo is not None:
            self._inorden(nodo.izquierdo, visitados)
            visitados.append(nodo.valor)
            self._inorden(nodo.derecho, visitados)

    def __str__(self):
        return self._detalle(self.raiz, 0)

    def _detalle(self, nodo, nivel):
        resultado = ""
        if nodo is not None:
            resultado += self._detalle(nodo.derecho, nivel + 1)
            resultado += "|  " * nivel + str(nodo.valor) + "\n"
            resultado += self._detalle(nodo.izquierdo, nivel + 1)
        return resultado

    def buscar(self, valor):
        return self._buscar(self.raiz, valor)

    def _buscar(self, nodo, valor):
        if nodo is None:
            return False
        if valor == nodo.valor:
            return True
        if valor < nodo.valor:
            return self._buscar(nodo.izquierdo, valor)
        return self._buscar(nodo.derecho, valor)

    def __contains__(self, valor):
        return self.buscar(valor)
```

El `__contains__` es el puente del capítulo 13 que hace que el operador `in` del capítulo 5 trabaje solo: cuando escribas `if 13 in arbol`, Python va a llamar a `__contains__` y vos vas a tener tu O(log n), no el O(n) de la lista. Probá la búsqueda en el árbol de los nueve:

```python
arbol = ArbolBinario()
for numero in (8, 3, 10, 1, 6, 4, 7, 14, 13):
    arbol.insertar(numero)

print(arbol.buscar(13))    # True
print(arbol.buscar(5))     # False
print(13 in arbol)         # True
print(5 in arbol)          # False
```

Ahora, la parte del show que el capítulo 19 te dejó predisponiendo: **medir cuánto cuesta**. Si cada paso descarta la mitad, la cantidad de pasos es la altura del árbol, y la altura depende de la forma. Escribí un contador que camine con un `while` (la versión iterativa del `buscar`, que te muestra los pasos en carne propia):

```python
def pasos_de_busqueda(arbol, valor):
    pasos = 0
    nodo = arbol.raiz
    while nodo is not None:
        pasos += 1
        if valor == nodo.valor:
            return pasos
        if valor < nodo.valor:
            nodo = nodo.izquierdo
        else:
            nodo = nodo.derecho
    return pasos
```

Compará el costo sobre un árbol achaparrado y sobre un "palo":

```python
achaparrado = ArbolBinario()
for numero in (8, 3, 10, 1, 6, 4, 7, 14, 13):
    achaparrado.insertar(numero)

torcido = ArbolBinario()
for numero in range(1, 10):
    torcido.insertar(numero)

print(pasos_de_busqueda(achaparrado, 13))  # 4
print(pasos_de_busqueda(torcido, 9))       # 9
```

Nueve datos, mismo `buscar`, y la distancia entre los dos es el argumento de todo el capítulo:

> **Dato clave:** el `achaparrado` encontró el `13` en **4 pasos** — u O(log n), como la búsqueda binaria del capítulo 19. El `torcido` tardó **9 pasos** — O(n), como el `in` de una lista, porque su altura degeneró en fila. La diferencia no está en el código del `buscar` (es idéntico): está en la **forma** del árbol. La búsqueda binaria no era un algoritmo suelto: era la promesa de un árbol bien formado, y acá la viste cumplirse.

La tabla de costos de esta sección, para dejar el balance en números:

| Operación | Lista enlazada (cap21) | BST balanceado | BST torcido |
|---|---|---|---|
| Buscar un valor | O(n) — camina toda la cadena | **O(log n)** — una rama | O(n) — todo el palo |
| Insertar | O(1) si conocés la punta | O(log n) | O(n) |
| Listar ordenado | O(n log n) ordenando | **O(n)** — `inorden` | O(n) — `inorden` (pero el palo lo paga al buscar) |

En el renglón "buscar" está el contrato completo: la lista enlazada camina, el árbol bien formado salta. Y la palabra "bien formado" es la que va a gobernar las secciones 7 y 8.

## 6. Borrar y extraer: los tres casos

La lista enlazada del capítulo 21 te enseñó a borrar cosiendo la cadena. El árbol borra *recosiendo la jerarquía*, y la dificultad depende de un detalle: **cuántos hijos tenía el condenado**. Hay tres casos, y es la parte más quirúrgica del capítulo:

1. **Hoja (cero hijos).** Se corta y listo: el padre pasa a apuntar a `None`. Nada más que coser.
2. **Un solo hijo.** El hijo toma el puesto del padre: el abuelo pasa a apuntar directo al nieto. Es el mismo "puente" del `delete` del capítulo 21.
3. **Dos hijos.** El complicado. Si borrás el nodo y ponés cualquier cosa en su lugar, la propiedad del BST se rompe. El truco clásico: buscar al **sucesor inorden** — el más chico del subárbol derecho —, copiarle el valor al nodo que se borra, y eliminar al sucesor (que, por ser el más chico de su subárbol, tiene a lo sumo un hijo). La jerarquía queda intacta.

El `sucesor inorden` es la respuesta a una pregunta linda: "¿quién me sigue cuando me ordena el `inorden`?" Si me borran a mí, el que venía enseguida toma mi lugar, porque es el *menor de todos los mayores que yo*.

Implementalo todo de una, con el buscador de la sección 5 ya instalado, y un `_minimo`/`_maximo` para extraer puntas:

```python
class Nodo:
    def __init__(self, valor, izquierdo=None, derecho=None):
        self.valor = valor
        self.izquierdo = izquierdo
        self.derecho = derecho


class ArbolBinario:
    def __init__(self):
        self.raiz = None

    def insertar(self, valor):
        if self.raiz is None:
            self.raiz = Nodo(valor)
        else:
            self._insertar(self.raiz, valor)

    def _insertar(self, nodo, valor):
        if valor < nodo.valor:
            if nodo.izquierdo is None:
                nodo.izquierdo = Nodo(valor)
            else:
                self._insertar(nodo.izquierdo, valor)
        else:
            if nodo.derecho is None:
                nodo.derecho = Nodo(valor)
            else:
                self._insertar(nodo.derecho, valor)

    def inorden(self):
        visitados = []
        self._inorden(self.raiz, visitados)
        return visitados

    def _inorden(self, nodo, visitados):
        if nodo is not None:
            self._inorden(nodo.izquierdo, visitados)
            visitados.append(nodo.valor)
            self._inorden(nodo.derecho, visitados)

    def __str__(self):
        return self._detalle(self.raiz, 0)

    def _detalle(self, nodo, nivel):
        resultado = ""
        if nodo is not None:
            resultado += self._detalle(nodo.derecho, nivel + 1)
            resultado += "|  " * nivel + str(nodo.valor) + "\n"
            resultado += self._detalle(nodo.izquierdo, nivel + 1)
        return resultado

    def _minimo(self, nodo):
        while nodo.izquierdo is not None:
            nodo = nodo.izquierdo
        return nodo

    def _maximo(self, nodo):
        while nodo.derecho is not None:
            nodo = nodo.derecho
        return nodo

    def eliminar(self, valor):
        self.raiz = self._eliminar(self.raiz, valor)

    def _eliminar(self, nodo, valor):
        if nodo is None:
            raise ValueError(f"{valor} no está en el árbol")
        if valor < nodo.valor:
            nodo.izquierdo = self._eliminar(nodo.izquierdo, valor)
        elif valor > nodo.valor:
            nodo.derecho = self._eliminar(nodo.derecho, valor)
        else:
            if nodo.izquierdo is None:
                return nodo.derecho
            if nodo.derecho is None:
                return nodo.izquierdo
            sucesor = self._minimo(nodo.derecho)
            nodo.valor = sucesor.valor
            nodo.derecho = self._eliminar(nodo.derecho, sucesor.valor)
        return nodo

    def pop(self, menor=True):
        if self.raiz is None:
            raise ValueError("el árbol está vacío")
        objetivo = self._minimo(self.raiz) if menor else self._maximo(self.raiz)
        valor = objetivo.valor
        self.eliminar(valor)
        return valor
```

Mirá el `_eliminar` con los tres casos a la vista. El caso hoja y el caso un-hijo se resuelven en la misma línea devolviendo el hijo que sobrevive (si no hay ninguno, devuelve `None`, y el padre pierde la referencia: caso hoja cubierto). El caso dos-hijos es la transacción de la sección: copiar el valor del sucesor y borrarlo de su rincón original. El `pop`, por su parte, es un mini-`heap` (montículo): te devuelve el mínimo (o el máximo, con `menor=False`) y lo saca, en dos movimientos que ya sabés. `_minimo` y `_maximo` son simpáticos: en un BST, el mínimo está *todo a la izquierda* y el máximo *todo a la derecha* — caminos de una sola dirección, sin comparaciones.

Probalo con el demo clásico, y fijate qué pasa con la raíz cuando la borrás:

```python
arbol = ArbolBinario()
for numero in (5, 3, 7, 2, 4, 6, 8):
    arbol.insertar(numero)

print(arbol.inorden())              # [2, 3, 4, 5, 6, 7, 8]

arbol.eliminar(5)                   # la raíz, con dos hijos
print(arbol.inorden())              # [2, 3, 4, 6, 7, 8]
print(arbol.raiz.valor)             # 6  (el sucesor inorden tomó el trono)

print(arbol.pop())                  # 2  (el mínimo)
print(arbol.inorden())              # [3, 4, 6, 7, 8]

print(arbol.pop(menor=False))       # 8  (el máximo)
print(arbol.inorden())              # [3, 4, 6, 7]

try:
    arbol.eliminar(500)
except ValueError as error:
    print(error)                    # 500 no está en el árbol
```

La línea más jugosa es `arbol.raiz.valor == 6`: borraste al `5` y la raíz *cambió de dueño* sin romper la jerarquía. El `inorden` sigue saliendo ordenado, que es la prueba de fuego de que la propiedad del BST sobrevivió la cirugía.

> **Importante:** el `pop` no es "borrar de arriba de una pila": es *extraer el extremo*. `pop()` saca el más chico, `pop(menor=False)` el más grande, y el árbol se vuelve a acomodar solo. Esta es la base exacta de un **montículo** (*heap*), que vas a construir formalmente en la Parte VIII — pero ya tenés el gesto aprendido.

## 7. El árbol que se tuerce

Volvé a la tabla de la sección 5 y a esa palabra que la gobernaba: *bien formado*. Ahora hacé la pregunta incómoda: ¿qué garantiza que el árbol *nazca* bien formado? Nada. La forma del árbol está decidida, dato por dato, por el **orden en que llegan** — que en un programa real puede ser cualquier cosa, incluido el peor de los casos. Mirá qué pasa cuando los datos llegan ya ordenados (o casi ordenados, que en la vida real es lo que pasa con las fechas y los IDs):

```python
torcido = ArbolBinario()
for numero in range(1, 6):
    torcido.insertar(numero)

print(torcido)
```

```
|  |  |  |  5
|  |  |  4
|  |  3
|  2
1
```

Ese palo, colgando del `1`, **es una lista enlazada con otro nombre**. Inserta datos ordenados en un BST y vas a tener la lista del capítulo 21 parada de costado: cada nodo con un solo hijo, altura máxima, y las bondades de la sección 5 evaporadas. El `buscar` que tardaba 4 pasos en el achaparrado vuelve a tardar `n` en el palo. La promesa O(log n) del BST es una promesa *condicional*: vale solo mientras el árbol reparta sus datos entre las dos ramas.

La distancia entre la promesa y la realidad de los datos llega al extremo en la **memoria**. La recursión de todo el capítulo — `insertar`, `buscar`, `inorden` — hunde su pila de llamadas hasta la profundidad del árbol (capítulo 10): por cada nivel de profundidad, un marco de pila en la memoria. En un árbol achaparrado, un millón de datos viven en 20 niveles: la pila ni se entera. En el palo torcido, cada dato es un nivel: un millón de datos = un millón de marcos de pila = el `RecursionError` del capítulo 10, a los pocos mil. Probalo con un palo de verdad, sin cuidado:

```python
bomba = ArbolBinario()
try:
    for numero in range(1100):
        bomba.insertar(numero)
except RecursionError as error:
    print(type(error).__name__)    # RecursionError
```

El intérprete no aguanta ni siquiera **insertar** mil y pico de datos ordenados: cada `_insertar` se hunde un nivel más que el anterior, y la pila (que capítulo 10 te contó que rondea los 1000 marcos por defecto) revienta antes de terminar. No es un límite de "tamaño": es un límite de *profundidad*, y la profundidad la decide la forma.

> **Dato clave:** la altura del árbol es la moneda de cambio de todo el capítulo. Balanceado, la altura es O(log n) — un millón de datos, 20 niveles. Torcido, la altura es O(n) — un millón de datos, un millón de niveles, pila reventada y búsquedas a pata. Cuando alguien te mida "¿qué tan difícil es tu estructura?", lo que va a mirar es la altura.

Entonces, el problema queda plantado con todas sus piezas: la *forma* decide el costo, la forma la decide el *orden de llegada*, y el orden de llegada no lo controlás. ¿La solución? Que el árbol mismo se **enderece** cuando detecta que se está torciendo. Ese es el trabajo de los **árboles balanceados**, la familia aristocrática del tema. Todos comparten la idea: después de cada cambio, se *miden* y, si algún nodo quedó desnivelado, se **rotan** para recuperar la forma. Los más famosos de la casa:

- **AVL** (por sus inventores, Adelson-Velsky y Landis, 1962): el estricto. Exige que en cada nodo las alturas de sus dos ramas difieran en a lo sumo 1, y rotá para cumplirlo siempre. Es el que vas a construir con tus manos en la sección siguiente.
- **Rojo-negro** (*red-black*): el flexible. Colorea los nodos de rojo y negro con reglas que garantizan un balanceo *aproximado* — más rápido de mantener en cada inserción, combinación de colores que le da el nombre.
- **B**: el de las bases de datos; un nodo con **muchas** claves y muchos hijos (el "más de dos" de la sección 1, reivindicado) que hace que las bases lean y escriban en pocos bloques de disco.
- **Splay**: el memorioso, que cada vez que accedés a un nodo lo *escalpa* hasta la raíz, dejando la estructura caliente para el siguiente acceso.

La moraleja anticipada: vivimos la mayor parte del tiempo con BST torcidos aceptables... hasta que la escala o los datos ordenados los condenan. La sección siguiente es la cura completa.

## 8. AVL: el árbol que se endereza solo

Cuando la altura importa tanto, la solución es no dejar que se descontrole. El **árbol AVL** le agrega al nodo un dato nuevo — su **altura** — y una regla de hierro: después de cada inserción y cada borrado, **si en algún nodo la diferencia de altura entre sus dos ramas pasa de 1, se rota**. Las rotaciones son las manos del árbol: unos cuantos enganches de punteros que reordenan el vecindario sin tocar la propiedad del BST. Y la promesa se cumple sola: con la regla de hierro, la altura nunca puede crecer más allá de O(log n), sin importar en qué orden lleguen los datos.

El nodo AVL es el `Nodo` de siempre con un atributo más:

```python
class NodoAVL:
    def __init__(self, valor):
        self.valor = valor
        self.izquierdo = None
        self.derecho = None
        self.altura = 1
```

Una hoja nace con altura 1. La altura de un nodo interno se calcula al vuelo: `1 + max(altura(izquierdo), altura(derecho))`. Y con dos alturas a mano, la **medida de desnivel** — el *factor de balance* — es la resta:

```python
class ArbolAVL:
    def __init__(self):
        self.raiz = None

    def _altura(self, nodo):
        if nodo is None:
            return 0
        return nodo.altura

    def _factor_balance(self, nodo):
        if nodo is None:
            return 0
        return self._altura(nodo.izquierdo) - self._altura(nodo.derecho)
```

La resta es la brújula del árbol: si el `izquierdo` pesa más que el `derecho` en más de 1, el árbol se cayó para la izquierda; si el resultado es menor a −1, para la derecha. Dentro del rango (−1, 0, 1), todo respira normal.

### Insertar y la cuenta pendiente

El `_insertar` del AVL es el del BST con **dos líneas de más**: actualizar la altura del nodo que bajó, y luego pedir el enderezamiento:

```python
    def insertar(self, valor):
        self.raiz = self._insertar(self.raiz, valor)

    def _insertar(self, nodo, valor):
        if nodo is None:
            return NodoAVL(valor)
        if valor < nodo.valor:
            nodo.izquierdo = self._insertar(nodo.izquierdo, valor)
        elif valor > nodo.valor:
            nodo.derecho = self._insertar(nodo.derecho, valor)
        else:
            return nodo   # los duplicados no entran

        nodo.altura = 1 + max(self._altura(nodo.izquierdo),
                              self._altura(nodo.derecho))
        return self._equilibrar(nodo)
```

Fijate que el `_insertar` *devuelve* el nodo al que lo llama: eso es lo que le permite al árbol, al volver de cada llamada, pisar el subárbol con su versión ya enderezada. La recursión baja plantando la hoja, y **sube ajustando cuentas**: cada ancestro actualiza su altura y se chequea el desnivel. Nacés abajo, te enderezás arriba.

### Las cuatro rotaciones

El desnivel se corrige con una **rotación**: un puñado de enganches que hace que el hijo gire y ocupe el lugar del padre. Hay dos rotaciones básicas — derecha e izquierda — y dos combinaciones de las dos para los casos dobles. Mirá la primera, la rotación *a la derecha*, que arregla la caída hacia la izquierda:

```python
    def _rotar_derecha(self, nodo):
        nuevo_tope = nodo.izquierdo
        nodo.izquierdo = nuevo_tope.derecho
        nuevo_tope.derecho = nodo

        nodo.altura = 1 + max(self._altura(nodo.izquierdo),
                              self._altura(nodo.derecho))
        nuevo_tope.altura = 1 + max(self._altura(nuevo_tope.izquierdo),
                                    self._altura(nuevo_tope.derecho))
        return nuevo_tope
```

Imaginala así: el hijo izquierdo (`nuevo_tope`) toma la escalera del padre; el padre pasa a ser su hijo derecho, y el nieto que en el medio sobraba se muda a la izquierda del padre. Tres movimientos de punteros y el árbol volvió a la vertical. La rotación simétrica, que arregla la caída hacia la derecha, es el espejo exacto:

```python
    def _rotar_izquierda(self, nodo):
        nuevo_tope = nodo.derecho
        nodo.derecho = nuevo_tope.izquierdo
        nuevo_tope.izquierdo = nodo

        nodo.altura = 1 + max(self._altura(nodo.izquierdo),
                              self._altura(nodo.derecho))
        nuevo_tope.altura = 1 + max(self._altura(nuevo_tope.izquierdo),
                                    self._altura(nuevo_tope.derecho))
        return nuevo_tope
```

Y el enderezamiento completo, que decide *cuál* de las cuatro cirugías corresponde leyendo el factor de balance, tanto del nodo como de su hijo: el famoso **LL, RR, LR, RL** (según la rama doblemente pesada):

- **Izquierda-Izquierda (LL):** el nodo pesa para la izquierda y su hijo izquierdo también → una rotación a la derecha y listo.
- **Derecha-Derecha (RR):** el nodo pesa para la derecha y su hijo derecho también → rotación a la izquierda.
- **Izquierda-Derecha (LR):** el nodo pesa para la izquierda, pero la manija está en el hijo *derecho* del hijo izquierdo → primero rotá al hijo izquierdo a la izquierda, después al nodo a la derecha.
- **Derecha-Izquierda (RL):** el espejo del anterior, dos vueltas en el otro sentido.

El `_equilibrar` es la regla escrita con la brújula, sin necesidad de comparar contra el valor nuevo (funciona igual para inserciones y borrados):

```python
    def _equilibrar(self, nodo):
        balance = self._factor_balance(nodo)

        if balance > 1:
            if self._factor_balance(nodo.izquierdo) < 0:
                nodo.izquierdo = self._rotar_izquierda(nodo.izquierdo)
            return self._rotar_derecha(nodo)

        if balance < -1:
            if self._factor_balance(nodo.derecho) > 0:
                nodo.derecho = self._rotar_derecha(nodo.derecho)
            return self._rotar_izquierda(nodo)

        return nodo
```

Leelo en las cuatro ramas: si `balance > 1`, el problema es la izquierda (LL o LR); si además el hijo izquierdo pesa *menos* de 0, es LR — dale primero la vuelta chica. Si `balance < -1`, el problema es la derecha (RR o RL); el caso RL pide la vuelta chica previa. Cuando nada se desnivela, el árbol se devuelve intacto.

### Verlo enderezarse

El `__str__` del AVL le agrega al mapa de la sección 4 un **encabezado de niveles**, para que la altura del árbol se lea de un vistazo:

```python
    def __str__(self):
        if self.raiz is None:
            return "árbol vacío"
        altura = self._altura(self.raiz)
        encabezado = "   ".join(str(nivel + 1) for nivel in range(altura))
        return (encabezado + "\n" + "---" * altura + "\n"
                + self._detalle(self.raiz, 0))

    def _detalle(self, nodo, nivel):
        resultado = ""
        if nodo is not None:
            resultado += self._detalle(nodo.derecho, nivel + 1)
            resultado += "|  " * nivel + str(nodo.valor) + "\n"
            resultado += self._detalle(nodo.izquierdo, nivel + 1)
        return resultado
```

La primera fila (`1   2   3…`) cuenta los niveles; la fila de `---` los dibuja. Ahora la prueba que convence: mirá al AVL **enderezarse en vivo** mientras inserta. Cargá de a uno los primeros tres números del peor caso y observá qué pasa cuando llega el `30`:

```python
progreso = ArbolAVL()
for valor in (10, 20, 30):
    progreso.insertar(valor)
    print(progreso)
    print()
```

La salida es la historia de una rotación contada en dos actos:

```
1
---
10


1   2
------
|  20
10


1   2
------
|  30
20
|  10
```

Con `10` solo, un nivel. Llega el `20`, cuelga a la derecha, dos niveles (`|` de más). Llega el `30`... y el `10` está con dos alturas a la derecha contra cero a la izquierda — desnivel de −2, caída RR. El `_equilibrar` rota a la izquierda, y el mapa termina con el **`20` de nuevo tope**: `30` arriba, `10` abajo, dos niveles y aire de árbol. El `20` se *cambió de asiento* solo, sin que nadie lo pida.

### La suite completa

Para que sea un árbol de verdad, el AVL necesita lo que aprendiste en las secciones 5 y 6 — buscar y borrar — más el detalle AVL aplicado también al borrado. Mirá el final de la clase, de una:

```python
    def buscar(self, valor):
        return self._buscar(self.raiz, valor)

    def _buscar(self, nodo, valor):
        if nodo is None:
            return False
        if valor == nodo.valor:
            return True
        if valor < nodo.valor:
            return self._buscar(nodo.izquierdo, valor)
        return self._buscar(nodo.derecho, valor)

    def __contains__(self, valor):
        return self.buscar(valor)

    def inorden(self, reverse=False):
        resultado = []
        self._inorden(self.raiz, resultado, reverse)
        return resultado

    def _inorden(self, nodo, resultado, reverse):
        if nodo is None:
            return
        if reverse:
            self._inorden(nodo.derecho, resultado, reverse)
            resultado.append(nodo.valor)
            self._inorden(nodo.izquierdo, resultado, reverse)
        else:
            self._inorden(nodo.izquierdo, resultado, reverse)
            resultado.append(nodo.valor)
            self._inorden(nodo.derecho, resultado, reverse)

    def _minimo_nodo(self, nodo):
        while nodo.izquierdo is not None:
            nodo = nodo.izquierdo
        return nodo

    def _maximo_nodo(self, nodo):
        while nodo.derecho is not None:
            nodo = nodo.derecho
        return nodo

    def eliminar(self, valor):
        self.raiz = self._eliminar(self.raiz, valor)

    def _eliminar(self, nodo, valor):
        if nodo is None:
            raise ValueError(f"{valor} no está en el árbol")
        if valor < nodo.valor:
            nodo.izquierdo = self._eliminar(nodo.izquierdo, valor)
        elif valor > nodo.valor:
            nodo.derecho = self._eliminar(nodo.derecho, valor)
        else:
            if nodo.izquierdo is None:
                return nodo.derecho
            if nodo.derecho is None:
                return nodo.izquierdo
            sucesor = self._minimo_nodo(nodo.derecho)
            nodo.valor = sucesor.valor
            nodo.derecho = self._eliminar(nodo.derecho, sucesor.valor)

        nodo.altura = 1 + max(self._altura(nodo.izquierdo),
                              self._altura(nodo.derecho))
        return self._equilibrar(nodo)

    def pop(self, menor=True):
        if self.raiz is None:
            raise ValueError("el árbol está vacío")
        objetivo = self._minimo_nodo(self.raiz) if menor else self._maximo_nodo(self.raiz)
        valor = objetivo.valor
        self.eliminar(valor)
        return valor
```

Fijate que el borrado es el `_eliminar` de la sección 6 (los tres casos, el sucesor inorden) más la misma cuenta que la inserción: altura arriba, `_equilibrar` abajo. La regla de hierro no distingue cómo cambió el árbol — solo que cambió — y se aplica con la misma fe en ambos sentidos. El `inorden` ganó una vuelta: `reverse=True` te da la lista de mayor a menor, el espejo del recorrido.

Ahora la prueba que le da sentido a todo el capítulo: ese terror de la sección 7 — *datos ya ordenados* — inyectado directo en el AVL:

```python
avl = ArbolAVL()
for valor in (10, 20, 30, 40, 50, 25):
    avl.insertar(valor)

print(avl.inorden())              # [10, 20, 25, 30, 40, 50]
print(avl)
```

El mapa, con su encabezado, muestra la altura dominada:

```
1   2   3
---------
|  |  50
|  40
30
|  |  25
|  20
|  |  10
```

Vas a verlo en tu consola con la altura en **3** para seis datos (los mismos seis, torcidos en un BST, habrían llegado a 6 niveles). Y la búsqueda y el borrado que ya conocés, funcionando a la altura correcta:

```python
print(avl.buscar(25))             # True
print(avl.buscar(90))             # False
print(25 in avl)                  # True
print(90 in avl)                  # False

avl.eliminar(20)
print(avl.inorden())              # [10, 25, 30, 40, 50]

print(avl.pop())                  # 10
print(avl.pop(menor=False))       # 50
print(avl.inorden())              # [25, 30, 40]
```

Y el golpe final de la comparación: el palo torcido de la sección 7, reconstruido como AVL. Los diez números *en orden* sobre un achaparrado que no se toma a mal ninguno:

```python
avl_torcido = ArbolAVL()
for numero in range(1, 11):
    avl_torcido.insertar(numero)

print(avl_torcido.inorden())      # [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
print(avl_torcido)
```

En vez del palo de diez niveles, un árbol de cuatro. La altura del AVL con `n` nodos nunca supera un peinado finito: aproximadamente 1,44 · log₂(n + 2), y con diez datos eso es un árbol achaparrado, no una escalera.

> **Para curiosear:** *"¿rotaciones, en serio, a mano?"* El AVL que acabás de construir es, en miniatura, lo que corre por dentro de filas ordenadas de bases de datos y de estructuras del sistema — pero con dos diferencias: están escritas en C (no en Python) y usan primos más sofisticados. Las bases de datos usan **árboles B**, que guardan muchas claves por nodo para que el disco lea poco; los lenguajes de alto nivel usan **árboles rojo-negro** (el `TreeMap` de Java, el `std::map` de C++). El AVL es el de la estrictez: lo elegís cuando las búsquedas mandan y el borrado escasea. Lo que viste acá es la idea madre de todos.

Una aclaración honesta antes de seguir, para el capítulo de concurrencia: si este árbol viviera compartido entre **hilos** (Parte XI, capítulo 35), insertar y buscar a la vez podrían pisarse y corromper la estructura. La solución de la industria es un **bloqueo** (*lock*) que hace que un solo hilo toque el árbol por vez — lo vas a ver en su capítulo. Nosotros mantenemos la atención en el árbol en sí.

## 9. Un caso real: la agenda de cumpleaños

Todo esto estaría bonito pero vacío si el árbol no sirviera para datos que te importan. Y acá reaparece la lección secreta de la sección 2: la propiedad del BST solo pide que las claves **sepan compararse** con `<`. No importa si son números, cadenas o tuplas: por dentro, el árbol compara y ordena. Te va a resultar familiar de los diccionarios del capítulo 5, donde lo importante es *la clave*, no el dato.

Tu agenda de cumpleaños, entonces, sin escribir **una sola** clase nueva. Guardá cada amigo como una tupla `(fecha, nombre)` — la fecha primero, porque es lo que ordena — y dejala caer en el `ArbolBinario` de la sección 6:

```python
agenda = ArbolBinario()
for nombre, cumple in (
    ("Alicia", "05-17"),
    ("Bruno",  "12-15"),
    ("Carla",  "01-25"),
    ("Diego",  "07-30"),
    ("Elena",  "04-10"),
    ("Franco", "08-22"),
):
    agenda.insertar((cumple, nombre))

for cumple, nombre in agenda.inorden():
    print(nombre, cumple)
```

Y la magia de las tuplas del capítulo 5 — que se comparan elemento por elemento — hace el resto del trabajo. Los cumpleaños llegan en el desorden de tu vida, y salen **ordenados por fecha**, sin una función de ordenamiento en ninguna parte:

```
Carla 01-25
Elena 04-10
Alicia 05-17
Diego 07-30
Franco 08-22
Bruno 12-15
```

El `"05-17" < "12-15"` funciona porque las fechas están en el formato `MM-DD` con ceros a la izquierda: con todos los meses de dos dígitos, la comparación de texto coincide con la cronológica. Ese detalle de la sección 3 del capítulo 3 — el formato — es el que le da a la cadena el poder de ordenar. ¿Un cumpleaños repetido? Los duplicados no cambian el árbol: caen a la derecha del igual (la rama `else` del `_insertar`) y el `inorden` los lista juntos.

Miralo actuar con las herramientas que ya te sabés. El `in` te dice si un amigo está (clave completa), el `pop` extrae al que cumple **primero** y al que cumple **último** sin romper el grupo, y el `__str__` dibuja al calendario de costado:

```python
print(("01-25", "Carla") in agenda)    # True
print(("09-09", "Nadie") in agenda)    # False

print(agenda.pop())                     # ('01-25', 'Carla')  -> la primera del año
print(agenda.pop(menor=False))          # ('12-15', 'Bruno')  -> la última

print(agenda.inorden())
# [('04-10', 'Elena'), ('05-17', 'Alicia'), ('07-30', 'Diego'), ('08-22', 'Franco')]
```

> **Buenas prácticas:** el árbol no sabe (ni le importa) que tus claves son fechas: solo pide que sepan compararse. Esa es la frontera entre *usar* una estructura y *dominarla*: cuando el mecanismo queda atrás y el problema del día — ¿quién cumple antes? — se responde con una línea, la estructura pasó a segundo plano, que es donde tiene que estar. La misma idea te va a perseguir en la tabla hash, en el capítulo de la Parte VIII que viene.

Ese es el punto donde el árbol deja de ser una curiosidad de laboratorio y se vuelve una herramienta tuya: cambiaste los números por fechas, y nada del código de las secciones 2 a 6 se enteró.

## 10. En producción: validar antes de dejar crecer

La lista enlazada del capítulo 21 te dejó un reflejo: antes de dejar que una estructura respire sola en un servidor, hay que ponerle límites. El árbol trae un peligro extra que la lista no tenía — recordá la sección 7: si alguien te manda los datos **ya ordenados**, tu BST se pega un tiro en el pie y tu O(log n) se cae a O(n). Un par de datos torcidos, y el servicio que prometía buscar rápido empieza a tardar como si estuviera en una lista — el clásico *denial of service* (denegación de servicio) silencioso, y eso sin que nadie "ataque": solo con datos mal formateados.

La pasada de producción de esta sección aplica los cuatro lentes que ya son la casa editorial de la Parte VIII — **errores, eficiencia, escalabilidad, seguridad** — y deja al árbol blindado con tres guardias: validar el tipo y el rango de cada valor (el guardián del capítulo 6), rechazar duplicados (para que nadie engorde el árbol con copias de un mismo dato) y cortar por lo sano si la cantidad de nodos pasa el tope.

```python
class ArbolProtegido:
    MAX_NODOS = 10_000

    def __init__(self):
        self.raiz = None
        self.cantidad = 0

    def _validar(self, valor):
        if not isinstance(valor, int):
            raise ValueError("los valores deben ser enteros")
        if valor < 1 or valor > 100:
            raise ValueError("el valor debe estar entre 1 y 100")
        if self.cantidad >= self.MAX_NODOS:
            raise ValueError("el árbol está lleno")

    def insertar(self, valor):
        self._validar(valor)
        if self.buscar(valor):
            raise ValueError(f"el valor {valor} ya está en el árbol")
        self.raiz = self._insertar(self.raiz, valor)
        self.cantidad += 1

    def _insertar(self, nodo, valor):
        if nodo is None:
            return Nodo(valor)
        if valor < nodo.valor:
            nodo.izquierdo = self._insertar(nodo.izquierdo, valor)
        else:
            nodo.derecho = self._insertar(nodo.derecho, valor)
        return nodo

    def inorden(self):
        visitados = []
        self._inorden(self.raiz, visitados)
        return visitados

    def _inorden(self, nodo, visitados):
        if nodo is not None:
            self._inorden(nodo.izquierdo, visitados)
            visitados.append(nodo.valor)
            self._inorden(nodo.derecho, visitados)

    def buscar(self, valor):
        return self._buscar(self.raiz, valor)

    def _buscar(self, nodo, valor):
        if nodo is None:
            return False
        if valor == nodo.valor:
            return True
        if valor < nodo.valor:
            return self._buscar(nodo.izquierdo, valor)
        return self._buscar(nodo.derecho, valor)
```

Mirá la decisión de duplicados de cerca: el `_validar` corre primero, y el `buscar` de después es un O(log n) barato que decide si el dato ya vive en el árbol. La clase lleva un contador `cantidad`, y ese contador es la llave del tercer guardián: cuando pase el `MAX_NODOS`, el árbol deja de aceptar, pase lo que pase. Probalo con un tope de juguete y datos sucios:

```python
guardado = ArbolProtegido()
for valor in (50, 30, 70):
    guardado.insertar(valor)

print(guardado.inorden())        # [30, 50, 70]

try:
    guardado.insertar("diez")
except ValueError as error:
    print(error)                 # los valores deben ser enteros

try:
    guardado.insertar(0)
except ValueError as error:
    print(error)                 # el valor debe estar entre 1 y 100

try:
    guardado.insertar(50)
except ValueError as error:
    print(error)                 # el valor 50 ya está en el árbol
```

Y el cortacircuitos de la escalabilidad, con un árbol de juguete sediento:

```python
chico = ArbolProtegido()
chico.MAX_NODOS = 3

chico.insertar(10)
chico.insertar(20)
chico.insertar(5)

try:
    chico.insertar(7)
except ValueError as error:
    print(error)                 # el árbol está lleno
```

> **Buenas prácticas:** el reflejo de las secciones que vienen debería ser: toda estructura que va a vivir en un programa real se construye con su *traje de calle* — validación de entrada, política de duplicados y tope de crecimiento. No es relleno: es la diferencia entre "el usuario mandó basura" y "el servicio se cayó". El árbol de la sección 2 te sirvió para aprender; el de esta sección es el que dejás en producción.

Y una nota del cuarto lente, la **seguridad** con datos torcidos: un AVL (sección 8) de paso neutraliza el ataque más barato contra un BST — mandar los datos ordenados para degenerar la altura. Si tu árbol vive en un entorno hostil, la respuesta no es solo validar: es balancear. Las dos capas — validación *y* balance — son las que dejan al árbol listo para el mundo real.

## 11. Resumen y conceptos clave

Este capítulo te llevó de la fila a la jerarquía. Arrancaste con la intuición que ya tenías — un organigrama, tus carpetas del capítulo 10 — y la convertiste en estructura: la lista enlazada que se **ramifica**, con un nodo de dos puertas (`izquierdo` y `derecho`) y un vocabulario propio (raíz, hoja, nivel, altura, subárbol). Le agregaste una **regla** — todo el subárbol izquierdo es menor, todo el derecho mayor, el **BST** — y esa regla sola pagó todo el capítulo: el `inorden` (izquierda → nodo → derecha) devuelve la lista **ordenada** sin ordenar nada, el `__str__` dibuja el árbol de costado por niveles, y el `buscar` se convirtió en la **búsqueda binaria** del capítulo 19 hecha estructura: cada paso descarta la mitad y O(log n) deja de ser teoría. Borraste con los tres casos — hoja, un hijo, dos hijos con sucesor inorden — y extrajiste mínimo y máximo con `pop`. Pero la hora de la verdad llegó cuando viste al árbol **torecerse**: datos ordenados, un solo hijo por nodo, y la promesa O(log n) derrumbada a O(n) con `RecursionError` incluido por pila reventada. La respuesta fue el **AVL**: cada nodo con su altura, el factor de balance que mide el desnivel, y las **rotaciones** (LL, RR, LR, RL) que enderezan el árbol en cada inserción y borrado — con la promesa de altura O(log n) sin importar el orden de llegada. Y cerraste demostrando que todo el mecanismo es agnóstico a tus datos: tu agenda de cumpleaños ordenada con tuplas, y una pasada de producción — validación, duplicados y tope — que deja al árbol en condiciones de trabajar.

La moraleja que se lleva la Parte VIII: cada estructura es una apuesta con moneda propia. La lista enlazada apostó a la libertad de crecer sin copiar y paga acceso a pata. El árbol apostó a la forma: ordena desde la raíz, y cuando mantiene la forma, compra la búsqueda por la mitad. Que se te tuerca — y que esté en tus manos enderezarlo — es el precio de esa forma, y el AVL te enseñó cómo pagarlo solo.

Repasá el checklist antes de seguir:

- [ ] Un árbol es una lista enlazada que se **ramifica**: el nodo guarda un dato y dos referencias (`izquierdo`, `derecho`).
- [ ] Vocabulario: **raíz** (sin padre), **hoja** (sin hijos), **nivel** (distancia a la raíz), **altura** (máximo camino), **subárbol**.
- [ ] El árbol se dibuja **con la raíz arriba y las hojas abajo**, al revés del árbol biológico.
- [ ] La propiedad **BST**: el subárbol izquierdo *entero* es menor y el derecho *entero* mayor; es global, no de a un hijo.
- [ ] `insertar` recursivo **calca los datos**: si es menor, izquierda; si es mayor, derecha — la lección del capítulo 10.
- [ ] **Promesa del capítulo 10 cobrada:** los árboles son los datos *recursivos por naturaleza* donde la recursión brilla — `_insertar`, `_inorden`, `_detalle` y `_eliminar` se llaman a sí mismas con un problema más chico y se leen como la propia estructura.
- [ ] `inorden` (izq → nodo → der) devuelve la lista **ordenada**; `preorden` visita el nodo primero; `postorden` al final.
- [ ] `__iter__` con `yield from` presta el recorrido a `for`, `list()`, `max()`, `min()` (capítulo 13).
- [ ] El `__str__` por niveles dibuja el árbol **de costado**: derecha arriba, izquierda abajo, sangría = profundidad.
- [ ] `buscar` y `in` (con `__contains__`) **descartan la mitad por paso**: O(log n) en árbol balanceado, O(n) en el torcido.
- [ ] `eliminar` tiene 3 casos: hoja, un hijo (el hijo sube) y **dos hijos (el sucesor inorden toma el trono)**.
- [ ] `pop()` extrae el mínimo, `pop(menor=False)` el máximo — la base del *heap*.
- [ ] Insertar datos **ordenados** tuerce el BST en una lista enlazada: altura O(n), pila reventada con `RecursionError` (~1000 marcos, capítulo 10).
- [ ] Los **árboles balanceados** (AVL, rojo-negro, B, splay) mantienen la altura O(log n) con rotaciones.
- [ ] **AVL**: el nodo guarda su `altura`; el factor de balance = altura(izq) − altura(der); si se desnivela de ±1, rota (**LL/RR/LR/RL**).
- [ ] Las claves del árbol solo necesitan **saber compararse**: tuplas de fechas ordenan sin tocar el código (la agenda).
- [ ] En producción: validar tipo/rango, rechazar duplicados y cortar con `MAX_NODOS` — y balancear contra datos hostiles.

## 12. Ejercicios

1. **La caja que se ramifica.** Con la clase `Nodo` de la sección 2, armá a mano el árbol del ejemplo: raíz `8`, hijo izquierdo `3`, hijo derecho `10`. Después armá una cadena *vertical* con `Nodo`s enlazados para simular el árbol torcido de la sección 7 (raíz `1`, derecho `2`, derecho `3`), y recorrelos con un `while` mostrando cada valor.

2. **Contar y medir.** Escribí dos funciones recursivas al estilo capítulo 10: `cantidad(nodo)` que cuente los nodos de un árbol y `altura(nodo)` que devuelva su altura (contando niveles, con `None` = 0). Probalas sobre el árbol `[8, 3, 10, 1, 6, 4, 7, 14, 13]`: esperás `9` nodos y altura `4`.

3. **El atajo de la mitad.** Usando `pasos_de_busqueda` de la sección 5, medí cuántos pasos tarda en encontrar `10` el árbol `[8, 3, 10, 1, 6, 4, 7, 14, 13]`, y cuántos tarda un `torcido` con `1..8` en encontrar el `8`. Explicá con tus palabras por qué la diferencia es la altura.

4. **El trono que no se rompe.** Con el árbol `[5, 3, 7, 2, 4, 6, 8]` de la sección 6, borrá el `7` (una hoja), después el `5` (la raíz, con dos hijos) y mostrá el `inorden` después de cada paso y el valor de la nueva raíz. Comprobá que el resultado final es `[2, 3, 4, 6, 8]`.

5. **Los tres recorridos.** Sobre el árbol `[8, 3, 10, 1, 6, 4, 7, 14, 13]`, calculá a mano el `preorden` y el `postorden`, y después comprobalos con el código de la sección 3. Mirá qué posición ocupa la raíz en cada lista.

6. **El árbol que aguanta la fila.** Cargá `1` a `15` en un `ArbolAVL` y mostrá su `inorden` (debe salir ordenado), su altura (`_altura(raiz)`) y su `__str__`. Compará con lo que pasaba al cargar lo mismo en un `ArbolBinario` torcido: ¿cuántos niveles tenía y qué riesgo corría al buscar?

7. **La agenda en orden inverso.** Con la agenda de la sección 9, armá una función `proximo_cumple(agenda, fecha)` que, usando el recorrido `inorden`, devuelva el **primer** cumpleaños *después* de una fecha dada (si la fecha cae a mitad de año, el próximo debe ser posterior). Probalo con `"03-01"` (esperás a Elena `04-10`) y con `"11-30"` (esperás a Bruno `12-15`).

8. **La pasada de producción.** Extendé de a un lente: agregale a `ArbolProtegido` la validación de un tipo extra (`float`) y mostrá cómo se comporta con un `insertar(3.5)`; y probá que el `MAX_NODOS` corta el crecimiento con un tope de `2` y tres inserciones.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. La caja que se ramifica
class Nodo:
    def __init__(self, valor, izquierdo=None, derecho=None):
        self.valor = valor
        self.izquierdo = izquierdo
        self.derecho = derecho

raiz = Nodo(8, Nodo(3), Nodo(10))
print(raiz.valor)                        # 8
print(raiz.izquierdo.valor)              # 3
print(raiz.derecho.valor)                # 10

cadena = Nodo(1)
actual = cadena
for numero in (2, 3):
    actual.derecho = Nodo(numero)
    actual = actual.derecho

actual = cadena
while actual is not None:
    print(actual.valor, end=" ")
    actual = actual.derecho
# 1 2 3

print()

# 2. Contar y medir
def cantidad(nodo):
    if nodo is None:
        return 0
    return 1 + cantidad(nodo.izquierdo) + cantidad(nodo.derecho)

def altura(nodo):
    if nodo is None:
        return 0
    return 1 + max(altura(nodo.izquierdo), altura(nodo.derecho))

arbol = ArbolBinario()
for numero in (8, 3, 10, 1, 6, 4, 7, 14, 13):
    arbol.insertar(numero)

print(cantidad(arbol.raiz))              # 9
print(altura(arbol.raiz))                # 4


# 3. El atajo de la mitad
def pasos_de_busqueda(arbol, valor):
    pasos = 0
    nodo = arbol.raiz
    while nodo is not None:
        pasos += 1
        if valor == nodo.valor:
            return pasos
        if valor < nodo.valor:
            nodo = nodo.izquierdo
        else:
            nodo = nodo.derecho
    return pasos

achaparrado = ArbolBinario()
for numero in (8, 3, 10, 1, 6, 4, 7, 14, 13):
    achaparrado.insertar(numero)

torcido = ArbolBinario()
for numero in range(1, 9):
    torcido.insertar(numero)

print(pasos_de_busqueda(achaparrado, 10))   # 2
print(pasos_de_busqueda(torcido, 8))        # 8


# 4. El trono que no se rompe
arbol = ArbolBinario()
for numero in (5, 3, 7, 2, 4, 6, 8):
    arbol.insertar(numero)

arbol.eliminar(7)                        # una hoja
print(arbol.inorden())                   # [2, 3, 4, 5, 6, 8]

arbol.eliminar(5)                        # la raíz, dos hijos
print(arbol.inorden())                   # [2, 3, 4, 6, 8]
print(arbol.raiz.valor)                  # 6


# 5. Los tres recorridos
arbol = ArbolBinario()
for numero in (8, 3, 10, 1, 6, 4, 7, 14, 13):
    arbol.insertar(numero)

print(arbol.preorden())                  # [8, 3, 1, 6, 4, 7, 10, 14, 13]
print(arbol.postorden())                 # [1, 4, 7, 6, 3, 13, 14, 10, 8]

# a mano: preorden = nodo, izquierda, derecha -> 8, 3, 1, 6, 4, 7, 10, 14, 13
# a mano: postorden = izquierda, derecha, nodo -> 1, 4, 7, 6, 3, 13, 14, 10, 8


# 6. El árbol que aguanta la fila
avl = ArbolAVL()
for numero in range(1, 16):
    avl.insertar(numero)

print(avl.inorden())                     # [1, ..., 15]
print(avl._altura(avl.raiz))             # 4
print(avl)


# 7. La agenda en orden inverso
def proximo_cumple(agenda, fecha):
    for cumple, nombre in agenda.inorden():
        if cumple > fecha:
            return nombre, cumple
    return None

agenda = ArbolBinario()
for nombre, cumple in (
    ("Alicia", "05-17"),
    ("Bruno",  "12-15"),
    ("Carla",  "01-25"),
    ("Diego",  "07-30"),
    ("Elena",  "04-10"),
    ("Franco", "08-22"),
):
    agenda.insertar((cumple, nombre))

print(proximo_cumple(agenda, "03-01"))   # ('Elena', '04-10')
print(proximo_cumple(agenda, "11-30"))   # ('Bruno', '12-15')


# 8. La pasada de producción
arbol_protege = ArbolProtegido()
try:
    arbol_protege.insertar(3.5)
except ValueError as error:
    print(error)                         # los valores deben ser enteros

chico_protege = ArbolProtegido()
chico_protege.MAX_NODOS = 2
chico_protege.insertar(1)
chico_protege.insertar(2)
try:
    chico_protege.insertar(3)
except ValueError as error:
    print(error)                         # el árbol está lleno
```