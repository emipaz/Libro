# Capítulo 23 — Grafos: el dato que conecta

Recibiste el mapa de rutas de una empresa de reparto a domicilio. Es una hoja enorme donde cada barrio es un punto y cada punto tiene una o varias líneas hacia otros puntos; junto a la línea, un número: los kilómetros que separan un barrio de otro. Tu tarea es hacer que esa hoja cobre vida en una computadora: responder cosas como *de qué barrios salen rutas directas*, *cuántos kilómetros hay entre dos barrios* o *cuál es el barrio mejor conectado de todos*. Ahora mirá tu teléfono: la app de navegación que te dice "girá en 300 metros" trabaja sobre una abstracción parecida — calles que son conexiones, esquinas que son puntos — y tu red social también, con personas en el lugar de los barrios y "te siguen" en el lugar de las rutas. Todo eso es la misma idea, y en este capítulo vas a llevarla a código.

La estructura que te toca hoy es el **grafo**, la más general y la más familiar de todas: no es una fila como la lista enlazada del capítulo 21, ni una jerarquía como los árboles del capítulo 22 — es una **red**, donde cualquier elemento puede conectarse con cualquier otro, con o sin orden y con o sin medida. De hecho, todo lo que viste antes cabe en el grafo: una lista enlazada es un grafo donde cada punto tiene una sola conexión hacia adelante, y un árbol es un grafo con la regla de no cerrar ciclos. El grafo es la forma de pensar en grande — y grande, perdón por el suspenso, va a significar decenas de miles de nodos en las aplicaciones reales. En este capítulo vas a construir la estructura con tus manos, a elegir entre las dos maneras clásicas de guardarla en memoria, a ver qué cambia cuando le agregás **pesos** a las conexiones (ahí entra tu mapa de reparto) y a cerrar con la pasada de producción que ya es costumbre en esta parte del libro. Los dos capítulos siguientes van a aprender a *caminar* sobre ella: primero a recorrer la red entera sin perderse en un ciclo y a encontrar el camino más barato entre dos puntos — la promesa de tu app de navegación, cumplida en código —, y después el desafío mayor, el circuito que visita todos los barrios y vuelve al origen.

---

## 1. La red: un mapa, no una fila

Las estructuras de los capítulos 21 y 22 ordenaron el mundo en dos figuras: la fila y la jerarquía. La lista enlazada caminaba de a un dato hacia adelante; el árbol ramificaba de arriba hacia abajo, con cada elemento teniendo un padre y a lo sumo dos hijos. El grafo rompe ambos moldes con una declaración que parece una libertad demasiado cómoda: **cualquier cosa puede estar conectada con cualquier otra**. Mirá de nuevo el mapa de reparto: el barrio 5 puede tocar al 3 y al 9 sin que nadie los ordene, y dos barrios pueden no tener ruta directa entre sí. No hay "arriba" ni "abajo": hay una red de puntos unidos por líneas.

Como en los dos capítulos anteriores, lo primero es bautizar estas piezas con el vocabulario que la computación entera usa:

| Término | Qué es |
|---|---|
| **Vértice / nodo** | cada punto de la red — un barrio, una persona, una ciudad; en código, cada dato |
| **Arista** | la conexión entre dos vértices — la ruta, el "te sigo", el vuelo |
| **Vecino / adyacente** | un vértice conectado al que te interesa por una arista |
| **Grado** | la cantidad de aristas que tocan a un vértice; en tu red, "cuántos contactos tiene" |
| **Camino** | una secuencia de aristas que va de un vértice a otro — el equivalente a dar dos vueltas de metro |
| **Ciclo** | un camino que empieza y termina en el mismo vértice sin repetir aristas |
| **No dirigido** | las aristas no tienen flecha: la conexión vale para los dos sentidos (amistad) |
| **Dirigido** | las aristas tienen dirección: de A sale hacia B pero no al revés (seguir en Instagram) |
| **Ponderado** | las aristas llevan un número que las acompaña: kilómetros, costo, tiempo |
| **Completo** | cada par de vértices tiene su arista — el grado máximo posible |
| **Conectado** | existe un camino entre cualquier par de vértices de la red |

Mirá qué contundente es el grado. En tu red social de amigos, el grado de una persona es literalmente su cantidad de amigos; en una red de calles, el grado de una esquina es cuántas calles le llegan. Y hay una relación que conviene guardar para siempre: si un grafo con `V` vértices está conectado, tiene que tener **al menos** `V - 1` aristas. Un árbol, no por casualidad, tiene exactamente esa cuenta: `V` vértices y `V - 1` aristas, sin ciclos. El árbol del capítulo 22 es, con otras palabras, un **grafo conectado sin ciclos** — y ahora entendés por qué las promesas de eficiencia de entonces dependían tanto de esa forma.

> **Dato clave:** el árbol de la Parte VIII anterior no era una especie de "lista especial" — era un grafo con reglas: conectado y sin ciclos. El grafo es el contenedor general, y las estructuras "más ordenadas" son grafos que aceptaron una o dos apuestas extra.

El grafo, entonces, no ordena los datos como una fila o una jerarquía: les da **relaciones**. Y con esa decisión aparecen problemas nuevos y fascinantes — "¿hay un camino entre A y C?", "¿cuál es el camino más corto?" — que son exactamente los que vas a resolver en los próximos dos capítulos. Hoy construís el escenario.

## 2. Tipos de grafo y las dos maneras de guardarlos

Antes de escribir una sola clase, tenés que tomar dos decisiones, y las dos tienen consecuencias directas sobre el código.

### 2.1 ¿Dirigido o no?

La primera decisión es si las aristas tienen **flecha**. Mirá la diferencia entre dos apps que usás todos los días:

- En una red de **amistad**, si Alma es amiga de Beto, Beto es amigo de Alma. La conexión es simétrica, no hay flecha: es un grafo **no dirigido**.
- En Instagram, si Alma **sigue** a Beto, Beto no necesariamente sigue a Alma. La conexión tiene dirección: es un grafo **dirigido** — a veces llamado **digrafo**.

La decisión no es estética: en un grafo no dirigido, guardar la arista `Alma—Beto` implica guardarla en los dos sentidos (la lista de Alma contiene a Beto **y** la de Beto contiene a Alma). En el dirigido, solo donde sale la flecha. Para el código, esta decisión es un parámetro, y la vas a ver aparecer en todas las clases del capítulo: un booleano `dirigido` que decide si `agregar_arista` escribe una entrada o dos.

### 2.2 ¿Con pesos o sin ellos?

La segunda decisión es si las aristas **valen** algo. El mapa de reparto del arranque no solo dice que hay ruta entre dos barrios: dice cuántos *kilómetros* hay. Tu app de navegación pondría *minutos* o *semaforos*; una calculadora de rutas aéreas, *escalas* y *precio*. Cuando cada arista lleva un número, el grafo es **ponderado**, y ese número se llama **peso** de la arista.

El peso es la diferencia entre "existe un camino de A a B" y "existe un camino *barato* de A a B". Fijate qué importante: sin pesos, cualquier camino entre el barrio 2 y el barrio 9 sirve por igual. Con pesos, un camino que da cuatro vueltas puede terminar siendo más corto en kilómetros que el directo (o no). El capítulo 24 entero va a vivir de esa pregunta, así que hoy solo vas a preparar el terreno guardando los pesos.

### 2.3 Guardarlos: lista de adyacencia o matriz de adyacencia

Con las decisiones tomadas, queda el problema estructural: **¿cómo se guarda una red en la memoria de una computadora?** Hay dos respuestas clásicas, y conviene conocer las dos para elegir bien.

La primera es la **lista de adyacencia**: un diccionario donde cada vértice es una clave, y el valor es la lista de sus vecinos. Un mini-grafo de tres vértices conectados todos con todos se ve así:

```python
lista_adyacencia = {
    "A": ["B", "C"],
    "B": ["A", "C"],
    "C": ["A", "B"],
}
```

Mirá la información: para saber quién es vecino de `B`, simplemente leés `lista_adyacencia["B"]` — tiempo constante de diccionario, como en el capítulo 6. La segunda es la **matriz de adyacencia**: una tabla de `V × V` donde la casilla `[i][j]` dice si el vértice `i` está conectado con el `j`. El mismo grafo, en matriz:

```python
matriz_adyacencia = [
    [0, 1, 1],
    [1, 0, 1],
    [1, 1, 0],
]
```

En la matriz, la pregunta "¿A y C están conectados?" es `matriz[0][2] != 0` — un acceso directo a la casilla, O(1) y sin listas que recorrer. Pero mirá el costo de esa comodidad: la matriz siempre ocupa `V × V` celdas, **aunque el grafo tenga poquísimas aristas**. Un grafo de 1 000 vértices con solo 5 conexiones ocupa una tabla de un millón de celdas. Por eso la regla práctica es:

> **Dato clave:** la lista de adyacencia gana cuando el grafo es **disperso** — pocas aristas respecto del máximo posible — que es el caso de casi todo lo real (una persona no conoce a los 8 000 millones). La matriz gana cuando el grafo es **denso** o completo, o cuando necesitás preguntar "¿existe esta arista?" a una tasa altísima. El resto del libro trabaja con listas de adyacencia, y el capítulo 25 va a usar una matriz — sin que sepas todavía por qué, es la pista de un grafista.

La lista de adyacencia de Python tiene un bonus que la matriz no perdona: las claves del diccionario **son los vértices**, así que los vértices pueden ser cualquier cosa hashable — cadenas (`"Buenos Aires"`), números, tuplas — sin mapear cada uno a un índice. Con esa ventaja sobre la mesa, a programar.

## 3. La clase `Grafo`: un diccionario que apunta a listas

La lista de adyacencia se escribe en Python casi sola: un diccionario donde las claves son vértices y los valores son listas de vecinos. La clase mínima que la administra debería sonarte orgánica después del capítulo 6:

```python
class Grafo:
    def __init__(self):
        self.adyacencias = {}

    def agregar_vertice(self, vertice):
        if vertice not in self.adyacencias:
            self.adyacencias[vertice] = []

    def agregar_arista(self, origen, destino, dirigido=False):
        self.agregar_vertice(origen)
        self.agregar_vertice(destino)
        self.adyacencias[origen].append(destino)
        if not dirigido:
            self.adyacencias[destino].append(origen)

    def obtener_vertices(self):
        return list(self.adyacencias.keys())

    def obtener_aristas(self):
        aristas = []
        for vertice, vecinos in self.adyacencias.items():
            for vecino in vecinos:
                aristas.append((vertice, vecino))
        return aristas

    def __str__(self):
        lineas = []
        for vertice, vecinos in self.adyacencias.items():
            lineas.append(f"{vertice} → {', '.join(vecinos)}")
        return "\n".join(lineas)
```

Comparala con la clase `ArbolBinario` del capítulo 22 y fijate qué sencilla se volvió: no hubo que decidir dónde va cada dato porque **no hay regla de orden**. Acá cualquier vértice apunta a cualquier otro, y `agregar_vertice` es un `if` que evita pisar una clave existente. El método más lindo es `agregar_arista`: conecta los dos vértices y, si el grafo es no dirigido, escribe la conexión en los dos sentidos — el vecino de la derecha en la lista de la izquierda, y al revés.

> **Importante:** en un grafo no dirigido, cada arista se guarda **dos veces** — en la lista de `origen` y en la de `destino`. Por eso `obtener_aristas` te va a devolver cada arista duplicada en sus dos direcciones. No es un error: es la estructura honrando la definición. Cuando quieras contar *aristas reales* de un grafo no dirigido, dividí por dos — o pensalo como el grado acumulado de la red.

Armá tu red social ahora y mirala por dentro:

```python
red = Grafo()
red.agregar_arista("Alma", "Beto")
red.agregar_arista("Alma", "Cami")
red.agregar_arista("Beto", "Cami")
red.agregar_arista("Cami", "Dani")

print(red.obtener_vertices())          # ['Alma', 'Beto', 'Cami', 'Dani']
print(red.adyacencias["Alma"])         # ['Beto', 'Cami']
print(len(red.adyacencias["Cami"]))    # 3
print(red)
```

Fijate el detalle que ahorra líneas: `agregar_arista("Cami", "Dani")` **crea** los vértices que no existían. El `len` sobre una lista de adyacencia es el grado del vértice — la cantidad de amigos, en tu red; y el `print(red)` te muestra el mapa completo, línea por línea, con cada vértice y sus vecinos. Lo que acabás de construir ya responde una de las preguntas del arranque: *cuál es el barrio mejor conectado* es, en términos de esta clase, el vértice con mayor `len` de lista de adyacencia.

## 4. Los pesos: cuando la conexión vale algo

La clase `Grafo` cuenta conexiones, pero el mapa de reparto del arranque exige más: cada ruta viene con sus **kilómetros**. Ahí está el salto de esta sección — cambiar de grafo simple a **grafo ponderado**, y el cambio en el código es tan elegante que primero vas a dudar de que alcance: el valor de cada lista de vectores pasa de ser una lista de vecinos a un **diccionario de vecinos**, donde la clave es el vecino y el valor es el peso.

Pensalo en términos de la pregunta del capítulo 6: un diccionario es "de clave a valor", y acá queremos que *de cada vecino* podamos recuperar *el peso de esa ruta*. La lista de adyacencia ponderada se ve así:

```python
{
    "Buenos Aires": {"Santiago": 1890, "Lima": 3100},
    "Santiago": {"Lima": 2450},
    "Lima": {"Bogotá": 1890},
}
```

Cada vértice apunta a un diccionario; cada entrada del diccionario es `vecino: peso`. Cantás la misma estructura genérica de la sección anterior — un diccionario de diccionarios — pero con una ganancia extra que vas a usar a partir del capítulo 24: el acceso `peso_de_arista("Buenos Aires", "Santiago")` es un `get` directo, y cuando la arista **no existe**, el peso natural que contestamos es el **infinito**. No hay camino = costo infinito, en un tema donde vas a estar *minimizando* costos todo el tiempo. Guardá ese reflejo: en algoritmos de caminos, `float("inf")` es "la arista no existe".

La clase completa de esta sección toma las decisiones de la sección 2 como parámetro:

```python
class GrafoPonderado:
    def __init__(self, dirigido=False):
        self.adyacencias = {}
        self.dirigido = dirigido

    def agregar_vertice(self, vertice):
        if vertice not in self.adyacencias:
            self.adyacencias[vertice] = {}

    def agregar_arista(self, origen, destino, peso):
        self.agregar_vertice(origen)
        self.agregar_vertice(destino)
        if destino not in self.adyacencias[origen]:
            self.adyacencias[origen][destino] = peso
        if not self.dirigido and origen not in self.adyacencias[destino]:
            self.adyacencias[destino][origen] = peso

    def remover_arista(self, origen, destino):
        if origen in self.adyacencias:
            self.adyacencias[origen].pop(destino, None)
        if not self.dirigido and destino in self.adyacencias:
            self.adyacencias[destino].pop(origen, None)

    def remover_vertice(self, vertice):
        if vertice in self.adyacencias:
            for vecino in list(self.adyacencias):
                self.adyacencias[vecino].pop(vertice, None)
            del self.adyacencias[vertice]

    def obtener_adyacentes(self, vertice):
        return list(self.adyacencias.get(vertice, {}).keys())

    def peso_de_arista(self, origen, destino):
        return self.adyacencias.get(origen, {}).get(destino, float("inf"))

    def __str__(self):
        lineas = []
        for vertice, vecinos in self.adyacencias.items():
            conexiones = ", ".join(f"{vecino} ({peso})" for vecino, peso in vecinos.items())
            lineas.append(f"{vertice} → {conexiones}")
        return "\n".join(lineas)
```

La pregunta del capítulo 6 sobre conjuntos encuentra su respuesta acá con claridad: los valores `{}` del vértice **tienen** que ser diccionarios porque necesitamos guardar dos cosas por arista — el vecino *y* el peso — y un conjunto solo guarda una. Mirá los métodos nuevos: `obtener_adyacentes` devuelve las claves del diccionario interno; `peso_de_arista` devuelve el peso o `float("inf")` si no hay arista; y `remover_vertice` da el ejemplo de una limpieza completa — saca el vértice de todas las listas de sus vecinos **antes** de borrarlo de la red (una limpieza que, si te la salteás, deja aristas apuntando a un fantasma).

Ahora tu mapa de reparto, con barrios y kilómetros:

```python
rutas = GrafoPonderado()  # no dirigido por defecto

rutas.agregar_arista("Centro", "Norte", 5)
rutas.agregar_arista("Centro", "Oeste", 3)
rutas.agregar_arista("Centro", "Sur", 7)
rutas.agregar_arista("Norte", "Sur", 4)
rutas.agregar_arista("Oeste", "Sur", 6)

print(rutas.obtener_adyacentes("Centro"))          # ['Norte', 'Oeste', 'Sur']
print(rutas.peso_de_arista("Centro", "Sur"))       # 7
print(rutas.peso_de_arista("Norte", "Oeste"))      # inf
print(rutas)
```

El `print` te muestra, por cada barrio, sus conexiones con sus kilómetros entre paréntesis — la red completa lista para los capítulos siguientes. Y mirá que ya apareció la herramienta de siempre del grafista: `peso_de_arista("Norte", "Oeste")` devuelve `inf` porque no hay ruta directa entre esos dos barrios, aunque *se pueda llegar* pasando por Centro. Detectar que se puede llegar, y por dónde, es el problema del capítulo 24 — pero para eso falta el último refinamiento de hoy.

## 5. En producción: validar antes de conectar

Las dos clases que construiste son correctas, pero tienen la actitud de un prototipo: aceptan cualquier cosa y rara vez avisan. Pasásela a un colega en un proyecto real y van a pasar tres desastres que ya son viejos conocidos del capítulo anterior (¿te acordás de `ArbolProtegido`?). El primero: un vértice que no puede ser clave de diccionario — una lista, por ejemplo — entra en `agregar_vertice` y rompe el grafo desde adentro con un error críptico de "unhashable". El segundo: agregar una arista entre vértices que no existen *crea* los vértices silenciosamente, y un typo en un nombre de barrio termina fabricando un vértice fantasma que nadie pidió. El tercero: un peso negativo entra sin chistar, y va a hacer aguas todos los algoritmos de caminos del próximo capítulo (el de Dijkstra, que conocerás mañana, asume pesos no negativos).

La pasada de producción, entonces, es la misma de la sección 10 del capítulo 22 pero adaptada a la red: **validar antes de mutar, y avisar con tipos de excepción que el llamador pueda capturar**. La clase nueva hereda todo lo bueno de `GrafoPonderado` y endurece las fronteras:

```python
class GrafoProtegido(GrafoPonderado):
    def agregar_vertice(self, vertice):
        try:
            hash(vertice)
        except TypeError as error:
            raise ValueError(
                f"{vertice!r} no puede ser clave de un diccionario."
            ) from error
        super().agregar_vertice(vertice)

    def agregar_arista(self, origen, destino, peso):
        if origen not in self.adyacencias or destino not in self.adyacencias:
            raise KeyError("Ambos vértices deben existir en el grafo.")
        if peso < 0:
            raise ValueError("Los pesos no pueden ser negativos.")
        super().agregar_arista(origen, destino, peso)
```

Mirá cómo se defiende cada línea. `agregar_vertice` prueba el vértice con `hash()` — la operación exacta que un diccionario va a necesitar — y si explota con `TypeError`, lo envoltura en un `ValueError` con un mensaje que le dice al programador qué pasó, en vez de dejar que el error crudo de dict estalle dos llamadas más tarde. Y fijate el detalle fino del `agregar_arista` producction: ahora **exige** que los vértices ya existan. El prototipo los creaba callado; el de producción quiere que crees los vértices a propósito, porque los vértices no aparecen solos en el modelo de un negocio — primero modelás qué barrios existen, después qué rutas los unen.

Mirá la clase en acción, incluyendo el momento en que valida cada regla:

```python
red = GrafoProtegido()

try:
    red.agregar_arista("Alma", "Beto", 10)
except KeyError as error:
    print(error)      # 'Ambos vértices deben existir en el grafo.'

red.agregar_vertice("Alma")
red.agregar_vertice("Beto")
red.agregar_arista("Alma", "Beto", 10)
red.agregar_arista("Alma", "Beto", 10)   # duplicado: la base ya lo ignora

try:
    red.agregar_arista("Alma", "Cami", -5)
except KeyError as error:
    print(error)      # 'Ambos vértices deben existir en el grafo.'
# el error de peso negativo recién se ve cuando ambos vértices existen:
red.agregar_vertice("Cami")
try:
    red.agregar_arista("Alma", "Cami", -5)
except ValueError as error:
    print(error)      # Los pesos no pueden ser negativos.

print(red.obtener_adyacentes("Alma"))    # ['Beto']
```

La dupla de excepciones es la firma de la clase: `KeyError` para "el barrio no existe", `ValueError` para "el dato es inválido por contenido". Aprendé a leerlo así y te vas a ahorrar noches enteras de depuración ajena.

Y una última cosa que en el capítulo 22 dejaste pendiente: ¿qué pasa cuando *dos procesos* quieren modificar la red a la vez? En un sistema de reparto de verdad, la app del repartidor y el panel del operador van a tocar el mismo grafo, y ahí nace un campo entero llamado **concurrencia** — cómo coordinar modificaciones simultáneas sin que los datos se corrompan. Te lo prometo en serio: lo vas a ver en detalle cuando llegues al capítulo 35, con hilos y candados. Por ahora, el `GrafoProtegido` de hoy valida la persona correcta, los barrios reales y los pesos sanos — que es la defensa que el resto del libro va a dar por sentada.

> **Importante:** el `KeyError` del `agregar_arista` protegido no es una molestia: es el contrato. Cambiás la creación silenciosa de vértices por una *validación explícita*, y eso convierte al "oops, fui yo que inventé un barrio" en una excepción imposible de ignorar. Los tiranos del prototipo nunca avisan; los gráfos de producción te cortan la mano antes.

## 6. Resumen y conceptos clave

Recorriste la familia más grande de la Parte VIII. Arrancaste con una hoja de rutas de reparto y una intuición — "cualquier cosa puede conectarse con cualquier otra" — y la convertiste en estructura con vocabulario: **vértices y aristas**, con sus grados, caminos y ciclos, y las dos preguntas de diseño que todas las clases del capítulo responden a su manera: dirigido o no, ponderado o no. Viste las **dos representaciones** de la memoria — la lista de adyacencia (un diccionario que apunta a listas, ideal para grafos dispersos) y la matriz de adyacencia (una tabla `V × V`, rápida para preguntas "¿existe esta arista?" pero costosa en memoria) — y elegiste la lista para el resto del libro. Programaste la clase `Grafo` más simple que existe y la viste resolver preguntas reales ("¿cuál barrio está mejor conectado?") leyendo las listas. Aprendiste a agregarle **pesos** dándole a cada vértice un diccionario de vecinos, con `peso_de_arista` devolviendo `float("inf")` cuando no hay arista — el reflejo que el capítulo 24 va a usar sin parar. Y cerraste endureciendo el prototipo: la clase `GrafoProtegido` que valida hashabilidad, exige vértices existentes y rechaza pesos negativos, con excepciones que el llamador puede leer (`KeyError` contra `ValueError`).

La moraleja de hoy, para sumarla a la de los capítulos 21 y 22: cada estructura elige su forma de ordenar el mundo, y el grafo eligió **no ordenar — conectar**. La lista apostó a la fila, el árbol a la jerarquía, y el grafo a la red, sin "arriba" ni "abajo" entre sus vértices. Esa libertad es la que paga en aplicaciones reales — mapas, redes sociales, rutas — y la que va a pedir, a partir del próximo capítulo, las destrezas que una fila jamás necesitó: recorrer la red entera sin perderse en un ciclo, y encontrar el camino más barato entre dos puntos. Eso es lo que aprendés a continuación, con el mapa de reparto de hoy servido en bandeja.

Repasá el checklist antes de seguir:

- [ ] Un **grafo** es una red de **vértices** unidos por **aristas**; no hay orden "arriba/abajo" como en el árbol.
- [ ] Vocabulario: **grado** (cuántas aristas tocan a un vértice), **camino**, **ciclo**, **completo**, **conectado** — y un árbol es un grafo conectado sin ciclos con `V - 1` aristas.
- [ ] Un **grafo no dirigido** guarda cada arista dos veces (alma—beto en los dos sentidos); un **dirigido** (digrafo) solo en la dirección de la flecha — param `dirigido`.
- [ ] **Ponderado**: cada arista lleva un peso; el valor de cada vértice pasa de lista a diccionario `vecino: peso`.
- [ ] **Lista de adyacencia**: dict de dict/listas — gana en grafos **dispersos**; los vértices pueden ser cualquier hashable.
- [ ] **Matriz de adyacencia**: tabla `V × V` — gana en grafos **densos** o para consultas de arista O(1) constante; cuesta `V²` celdas siempre (el capítulo 25 le hace un guiño).
- [ ] Clase base `Grafo`: `agregar_arista` **crea** vértices ausentes y escribe la conexión en los dos sentidos si no es dirigido.
- [ ] El **grado** de un vértice es `len` de su lista de adyacencia → "el más conectado" es `max` sobre los grados.
- [ ] `peso_de_arista` devuelve `float("inf")` cuando no hay arista — "no hay camino" = costo infinito.
- [ ] `remover_vertice` limpia el vértice de las listas de **todos** sus vecinos antes de borrarlo.
- [ ] Producción: **`hash()`** para validar hashabilidad, **`KeyError`** para vértices inexistentes (ya no se crean solos), **`ValueError`** para pesos negativos.
- [ ] El `agregar_arista` protegido **exige** vértices existentes: primero modelás los barrios, después las rutas.

## 7. Ejercicios

1. **El barrio del metro.** Con la clase `Grafo` de la sección 3, armá un tramo de subte con las estaciones `Retiro`, `Tribunales`, `Corrientes`, `Independencia` y `Perú`, conectadas en ese orden de a par (Retiro—Tribunales, Tribunales—Corrientes…). Mostrá los vértices, el grado de `Tribunales` y los vecinos de `Retiro`, y después el grafo completo con `print`. — Vas a ver que los extremos tienen grado 1 y el resto grado 2, la firma de una línea de subte.

2. **La red social y sus grados.** Con la misma clase, armá la red de amistad de 5 personas (`Alma`, `Beto`, `Cami`, `Dani`, `Ema`) con las aristas Alma—Beto, Alma—Cami, Beto—Cami y Cami—Dani. Calculá el **grado de cada persona**, la persona con más amigos y la que quedó aislada (grado 0). — Esperás que `Cami` sea la más popular con grado 3 y `Ema` la única sin amigos.

3. **El dígrafo de los seguidores.** Con `Grafo` y aristas dirigidas, modelá una red de Instagram: `Ema → Dani`, `Dani → Cami` y `Alma → Cami` (cada flecha es "sigue a"). Mostrá a quién sigue cada usuario y verifiquemos que **Dani no sigue a Ema** aunque Ema siga a Dani. Después corregí la injusticia con `agregar_arista` dirigida en el otro sentido y volvé a mirar.

4. **El mapa de vuelos con pesos.** Con `GrafoPonderado` de la sección 4, armá la red de vuelos con las ciudades `Buenos Aires`, `Santiago`, `Lima`, `Bogotá` y `Ciudad de México` y las rutas directas BA—Santiago (1 890 km), BA—Lima (3 100), Santiago—Lima (2 450), Lima—Bogotá (1 890) y Bogotá—Ciudad de México (3 150). Mostrá los vecinos de `Buenos Aires`, el peso del vuelo directo BA—Lima, el peso de un "vuelo" que no existe en la red (BA—Bogotá, que deberías leer como `inf`) y el `print` completo del grafo.

5. **Lista contra matriz.** Con las 5 ciudades del ejercicio 4, armá la **matriz de adyacencia** (`5 × 5`) con los kilómetros o `float("inf")` donde no hay vuelo, con la diagonal en `0`. Mostrá cuántas celdas ocupa la matriz (`25`) contra cuántas aristas reales hay (`5`). Después mostrá el valor de `matriz[0][4]` — "¿hay vuelo directo BA—Ciudad de México?" — y pensá por qué esa pregunta respondida devolvería `inf`.

6. **La pasada de producción.** Con `GrafoProtegido` de la sección 5, mostrá que: (a) agregar una arista sin los vértices dispara `KeyError`; (b) un vértice de tipo `tuple` como `("Buenos Aires", "Argentina")` se acepta (es hashable); (c) un vértice `list` como `["Alma", "Ciudad"]` se rechaza con `ValueError`; (d) un peso negativo se rechaza con `ValueError`; y (e) agregar dos veces la misma arista no la duplica (la base ya lo evitaba). Terminá mostrando los vecinos de `("Buenos Aires", "Argentina")`.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. El barrio del metro
class Grafo:
    def __init__(self):
        self.adyacencias = {}

    def agregar_vertice(self, vertice):
        if vertice not in self.adyacencias:
            self.adyacencias[vertice] = []

    def agregar_arista(self, origen, destino, dirigido=False):
        self.agregar_vertice(origen)
        self.agregar_vertice(destino)
        self.adyacencias[origen].append(destino)
        if not dirigido:
            self.adyacencias[destino].append(origen)

    def obtener_vertices(self):
        return list(self.adyacencias.keys())

    def obtener_aristas(self):
        aristas = []
        for vertice, vecinos in self.adyacencias.items():
            for vecino in vecinos:
                aristas.append((vertice, vecino))
        return aristas

    def __str__(self):
        lineas = []
        for vertice, vecinos in self.adyacencias.items():
            lineas.append(f"{vertice} → {', '.join(vecinos)}")
        return "\n".join(lineas)

metro = Grafo()
estaciones = ("Retiro", "Tribunales", "Corrientes", "Independencia", "Perú")
for estacion in estaciones:
    metro.agregar_vertice(estacion)
metro.agregar_arista("Retiro", "Tribunales")
metro.agregar_arista("Tribunales", "Corrientes")
metro.agregar_arista("Corrientes", "Independencia")
metro.agregar_arista("Independencia", "Perú")

print(metro.obtener_vertices())               # ['Retiro', 'Tribunales', 'Corrientes', 'Independencia', 'Perú']
print(len(metro.adyacencias["Tribunales"]))   # 2
print(metro.adyacencias["Retiro"])            # ['Tribunales']
print(metro)


# 2. La red social y sus grados
red = Grafo()
for persona in ("Alma", "Beto", "Cami", "Dani", "Ema"):
    red.agregar_vertice(persona)
red.agregar_arista("Alma", "Beto")
red.agregar_arista("Alma", "Cami")
red.agregar_arista("Beto", "Cami")
red.agregar_arista("Cami", "Dani")

grados = {p: len(red.adyacencias[p]) for p in red.obtener_vertices()}
print(grados)                                 # {'Alma': 2, 'Beto': 2, 'Cami': 3, 'Dani': 1, 'Ema': 0}
mas_popular = max(grados, key=grados.get)
print(mas_popular, grados[mas_popular])       # Cami 3
print([p for p, g in grados.items() if g == 0])  # ['Ema']


# 3. El dígrafo de los seguidores
seguidores = Grafo()
for usuario in ("Ema", "Dani", "Cami", "Alma"):
    seguidores.agregar_vertice(usuario)
seguidores.agregar_arista("Ema", "Dani", dirigido=True)
seguidores.agregar_arista("Dani", "Cami", dirigido=True)
seguidores.agregar_arista("Alma", "Cami", dirigido=True)

print("Ema sigue a:", seguidores.adyacencias["Ema"])      # Ema sigue a: ['Dani']
print("Dani sigue a Ema?", "Ema" in seguidores.adyacencias["Dani"])  # Dani sigue a Ema? False
seguidores.agregar_arista("Dani", "Ema", dirigido=True)
print("Dani sigue a:", seguidores.adyacencias["Dani"])    # Dani sigue a: ['Cami', 'Ema']


# 4. El mapa de vuelos con pesos
class GrafoPonderado:
    def __init__(self, dirigido=False):
        self.adyacencias = {}
        self.dirigido = dirigido

    def agregar_vertice(self, vertice):
        if vertice not in self.adyacencias:
            self.adyacencias[vertice] = {}

    def agregar_arista(self, origen, destino, peso):
        self.agregar_vertice(origen)
        self.agregar_vertice(destino)
        if destino not in self.adyacencias[origen]:
            self.adyacencias[origen][destino] = peso
        if not self.dirigido and origen not in self.adyacencias[destino]:
            self.adyacencias[destino][origen] = peso

    def obtener_adyacentes(self, vertice):
        return list(self.adyacencias.get(vertice, {}).keys())

    def peso_de_arista(self, origen, destino):
        return self.adyacencias.get(origen, {}).get(destino, float("inf"))

    def __str__(self):
        lineas = []
        for vertice, vecinos in self.adyacencias.items():
            conexiones = ", ".join(f"{vecino} ({peso})" for vecino, peso in vecinos.items())
            lineas.append(f"{vertice} → {conexiones}")
        return "\n".join(lineas)

vuelos = GrafoPonderado()
for ciudad in ("Buenos Aires", "Santiago", "Lima", "Bogotá", "Ciudad de México"):
    vuelos.agregar_vertice(ciudad)
vuelos.agregar_arista("Buenos Aires", "Santiago", 1890)
vuelos.agregar_arista("Buenos Aires", "Lima", 3100)
vuelos.agregar_arista("Santiago", "Lima", 2450)
vuelos.agregar_arista("Lima", "Bogotá", 1890)
vuelos.agregar_arista("Bogotá", "Ciudad de México", 3150)

print(vuelos.obtener_adyacentes("Buenos Aires"))      # ['Santiago', 'Lima']
print(vuelos.peso_de_arista("Buenos Aires", "Lima"))  # 3100
print(vuelos.peso_de_arista("Buenos Aires", "Bogotá"))  # inf
print(vuelos)


# 5. Lista contra matriz
ciudades = ["Buenos Aires", "Santiago", "Lima", "Bogotá", "Ciudad de México"]
inf = float("inf")
matriz = [
    [0, 1890, 3100, inf, inf],
    [1890, 0, 2450, inf, inf],
    [3100, 2450, 0, 1890, inf],
    [inf, inf, 1890, 0, 3150],
    [inf, inf, inf, 3150, 0],
]
print("celdas:", len(matriz) * len(matriz[0]))                # celdas: 25
aristas = sum(1 for fila in matriz for costo in fila
              if costo and costo != inf) // 2
print("aristas reales:", aristas)                              # aristas reales: 5
print("¿vuelo directo BA → CDMX?", matriz[0][4])               # ¿vuelo directo BA → CDMX? inf


# 6. La pasada de producción
class GrafoProtegido(GrafoPonderado):
    def agregar_vertice(self, vertice):
        try:
            hash(vertice)
        except TypeError as error:
            raise ValueError(
                f"{vertice!r} no puede ser clave de un diccionario."
            ) from error
        super().agregar_vertice(vertice)

    def agregar_arista(self, origen, destino, peso):
        if origen not in self.adyacencias or destino not in self.adyacencias:
            raise KeyError("Ambos vértices deben existir en el grafo.")
        if peso < 0:
            raise ValueError("Los pesos no pueden ser negativos.")
        super().agregar_arista(origen, destino, peso)

red = GrafoProtegido()

try:
    red.agregar_arista("Alma", "Beto", 10)
except KeyError as error:
    print(error)          # 'Ambos vértices deben existir en el grafo.'

red.agregar_vertice(("Buenos Aires", "Argentina"))
red.agregar_vertice(("Lima", "Perú"))
red.agregar_arista(("Buenos Aires", "Argentina"), ("Lima", "Perú"), 3100)

try:
    red.agregar_vertice(["Alma", "Ciudad"])
except ValueError as error:
    print(error)          # ['Alma', 'Ciudad'] no puede ser clave de un diccionario.

try:
    red.agregar_arista(("Buenos Aires", "Argentina"), ("Lima", "Perú"), -5)
except ValueError as error:
    print(error)          # Los pesos no pueden ser negativos.

red.agregar_arista(("Buenos Aires", "Argentina"), ("Lima", "Perú"), 3100)
print(red.obtener_adyacentes(("Buenos Aires", "Argentina")))   # [('Lima', 'Perú')]
```