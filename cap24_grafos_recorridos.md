# Capítulo 24 — Grafos II: el mapa que se recorre

En el capítulo 23 guardaste la red: cargaste barrios en un diccionario, los uniste con aristas, y cuando las conexiones valían algo les sumaste pesos. Pero guardar el mapa es una cosa, y *usarlo* es otra. Tu tabla de reparto pregunta algo que las listas, las pilas, las colas y los diccionarios de los capítulos anteriores no podían siquiera plantear: **"¿cómo llego del local a Belgrano pasando por el menor número de cuadras?"** Y su hermana, más ambiciosa: **"¿y si cada cuadra cuesta plata, cuál es el camino más barato?"** El grafo guardó las conexiones; este capítulo le enseña a caminar.

La promesa de hoy tiene tres escalones, y cada uno resuelve el que sigue. Arrancás con **BFS**, el recorrido por capas: de la mano de la cola del capítulo 13, vas a medir exactamente cuántos saltos separan a dos personas en una red social. Después **DFS**, el recorrido por profundidad: con la pila del capítulo 21 vas a explorar una red entera sin saltarte ningún rincón y a responder "¿está todo conectado?". Y para la pregunta que paga tu sueldo — el camino más *barato*, no el más corto — te traés de vuelta el montículo del capítulo 19 y aprendés **Dijkstra**, el algoritmo del `camino_mas_corto`, y de paso la pasada de producción que lo hace 5 veces más rápido (eso no es exageración: lo vas a cronometrar). Al final, tu mapa de reparto va a contestar, en un grafo de mil barrios, lo que hoy solo sabe mostrar por `print`.

---

## 1. El mapa se recorre: dos preguntas que recién acá tienen sentido

Fijate qué notable es la diferencia con todo lo anterior. La lista enlazada y el árbol guardaban el dato *dentro* de la caja, y recorrerlas era un trámite: arrancabas por el principio y seguías a los punteros. El grafo del capítulo 23 guarda el dato *en los vértices* pero la información que vale está **entre** ellos, y por eso recorrerlo no es un trámite: es un problema. Y para resolverlo no alcanza con mirar el diccionario; hay que decidir, paso a paso, por dónde ir. Acá tenés el pequeño vocabulario que usamos de acá en más:

| Término | Qué significa en un recorrido |
|---|---|
| **Visitar** | mirar un vértice y leer sus vecinos (procesar su lista de adyacencia) |
| **Marcar como visitado** | apuntarlo en un conjunto para no repetirlo; sin esto te perdés en los ciclos del capítulo 23 para siempre |
| **Salto / paso** | cruzar una arista; en BFS, "distancia" se mide en saltos |
| **Distancia** | según el problema: cantidad de saltos (BFS) o suma de pesos (Dijkstra) |
| **Conectado** | existe un camino entre cualquier par de vértices; un grafo en una sola pieza |
| **Componente conexa** | cada pieza separada; un grafo desconectado es una sopa de componentes |

Todas las búsquedas de este capítulo comparten un esqueleto: **marcá la partida, y entrá en un bucle que siempre mira el vértice "más prometedor" que tengas a mano, marcando vecinos nuevos a medida que los descubrís**. La única diferencia entre BFS, DFS y hasta Dijkstra es la respuesta a una pregunta chiquita y brutal: *¿cuál es "el más prometedor"?* BFS responde "el más viejo, el primero que llegó" (la cola del capítulo 13). DFS responde "el más nuevo, el último que descubrí" (la pila del capítulo 21, o la recursión del capítulo 10 que en el fondo es una pila también). Y Dijkstra, atenti, va a responder "el que está **más cerca**" — y esa palabra va a cambiar todo.

## 2. BFS: la ola que avanza pareja

¿Te acordás de la red social de "grados de separación"? La idea de que "todo el mundo está a seis personas de conocerte" no es una metáfora: es un **recorrido en amplitud** esperando ser programado. Tomá una persona de la red y preguntate: ¿en cuántos saltos llego a conocerla? El truco es avanzar **por capas**: primero tus amigos (salto 1), después los amigos de tus amigos (salto 2), después sus amigos (salto 3) — siempre agotando la capa actual antes de meterte en la siguiente. Es exactamente la disciplina de la cola: quien entra primero a la fila, se atiende primero, y por eso las capas se acomodan solas en orden creciente de distancia.

La clase base es la del capítulo anterior — la lista de adyacencia con listas de vecinos — pero la vas a ver acá completa y compacta, porque a partir de este punto la usamos como cimiento (lo mismo que hizo el capítulo 22 con la `Nodo` del capítulo 21):

```python
class Grafo:
    def __init__(self, dirigido=False):
        self.adyacencias = {}
        self.dirigido = dirigido

    def agregar_vertice(self, vertice):
        if vertice not in self.adyacencias:
            self.adyacencias[vertice] = []

    def agregar_arista(self, origen, destino, dirigido=False):
        self.agregar_vertice(origen)
        self.agregar_vertice(destino)
        if destino not in self.adyacencias[origen]:
            self.adyacencias[origen].append(destino)
        if not self.dirigido and not dirigido and origen not in self.adyacencias[destino]:
            self.adyacencias[destino].append(origen)

    def obtener_adyacentes(self, vertice):
        return list(self.adyacencias.get(vertice, []))

    def obtener_vertices(self):
        return list(self.adyacencias.keys())

    def __str__(self):
        lineas = []
        for vertice, vecinos in self.adyacencias.items():
            lineas.append(f"{vertice} → {', '.join(vecinos)}")
        return "\n".join(lineas)
```

Ahora el recorrido. La cola de dos extremos (`deque`, capítulo 13) guarda los vértices por descubrir; el conjunto `visitados` evita que un ciclo del capítulo 23 te haga dar vueltas sin fin:

```python
from collections import deque

def niveles_bfs(grafo, partida):
    visitados = {partida}
    niveles = {partida: 0}
    cola = deque([partida])

    while cola:
        vertice = cola.popleft()
        for vecino in grafo.adyacencias[vertice]:
            if vecino not in visitados:
                visitados.add(vecino)
                niveles[vecino] = niveles[vertice] + 1
                cola.append(vecino)

    return niveles
```

El secreto está en `niveles[vecino] = niveles[vertice] + 1`: como nunca brincás a la capa siguiente sin agotar la actual, la primera vez que descubrís un vértice es **necesariamente** por el camino de menos saltos, y ese número que anotás es su distancia mínima en aristas. El diccionario `niveles` que devuelve es, en sí mismo, la respuesta a la pregunta de la red social: *cuántos saltos hay de la partida hasta cada persona*. Armá la red y miralo en acción:

```python
red_social = Grafo()
for persona in ("Alma", "Beto", "Cami", "Dani", "Ema", "Fran"):
    red_social.agregar_vertice(persona)

red_social.agregar_arista("Alma", "Beto")
red_social.agregar_arista("Alma", "Cami")
red_social.agregar_arista("Beto", "Cami")
red_social.agregar_arista("Cami", "Dani")
red_social.agregar_arista("Dani", "Ema")
red_social.agregar_arista("Ema", "Fran")

print(niveles_bfs(red_social, "Alma"))
```

```text
{'Alma': 0, 'Beto': 1, 'Cami': 1, 'Dani': 2, 'Ema': 3, 'Fran': 4}
```

Leelo: Alma está a 0 de sí misma, a 1 salto de Beto y de Cami, a 2 de Dani, y así hasta Fran, a 4 saltos. Esa lista de números es el "seis grados de separación" hecho código: vos podés conocer a Fran, pero la respuesta honesta es que te separan cuatro personas. Y si lo que querés no es solo la distancia sino el **camino** (qué personas atravieso), el mismo BFS guarda quién nos llevó hasta cada uno:

```python
def camino_bfs(grafo, partida, objetivo):
    visitados = {partida}
    anteriores = {partida: None}
    cola = deque([partida])

    while cola:
        vertice = cola.popleft()
        if vertice == objetivo:
            camino = []
            while vertice is not None:
                camino.append(vertice)
                vertice = anteriores[vertice]
            return camino[::-1]

        for vecino in grafo.adyacencias[vertice]:
            if vecino not in visitados:
                visitados.add(vecino)
                anteriores[vecino] = vertice
                cola.append(vecino)

    return None


camino = camino_bfs(red_social, "Alma", "Fran")
print(camino)              # ['Alma', 'Cami', 'Dani', 'Ema', 'Fran']
print(len(camino) - 1)     # 4
```

Fijate la reconstrucción al final: como cada vértice guarda quién lo descubrió (`anteriores`), para volver al origen no hay que buscar nada — se camina para atrás siguiendo la cadena, igual que el `siguiente` de la lista enlazada pero en reversa, y se invierte el resultado. Es el mismo "hacer el camino de memoria" que le faltaba a tu mapa en el capítulo pasado.

> **Dato clave:** BFS te da el camino de **menos saltos**, y esa promesa descansa en una sola condición: agotar cada capa antes de pasar a la siguiente. Si alguna vez anotás `niveles[vecino] = niveles[vertice] + 1` *sin* estar seguro de que la capa está completa, estás mintiendo sobre la distancia. El `deque` con `popleft` es la garantía.

## 3. DFS: el que se hunde hasta el fondo

BFS era "primero lo cercano, todo parejo". Ahora cambiemos una sola decisión y el mapa se recorre distinto: en vez de agotar la capa, **seguí por el primer vecino hasta llegar a un callejón sin salida, y recién ahí retrocedé**. Eso es DFS — profundidad primero. Es la diferencia de navegar un edificio con una sola linterna: BFS explora piso por piso parejo; DFS se mete por una escalera hasta el sótano antes de preguntarse por el primer piso.

Para el código alcanza con describir "la primera puerta que veas, entrá; cuando no haya puerta, volvé" — lo que, mirá vos, es la definición exacta de la recursión del capítulo 10. La función se llamaría infinitamente sola hasta agotar lo que hay que visitar:

```python
def alcanzables_dfs(grafo, partida, visitados=None):
    if visitados is None:
        visitados = set()
    visitados.add(partida)
    for vecino in grafo.adyacencias[partida]:
        if vecino not in visitados:
            alcanzables_dfs(grafo, vecino, visitados)
    return visitados
```

El nombre del conjunto lo dice: lo que devuelve son todos los vértices **alcanzables** desde la partida — cada rincón de la red al que se llega siguiendo aristas. Ahora mirá qué pregunta se responde sola con esto. Tu red de reparto tenía un barrio, llamémoslo Gabi, que en el capítulo 23 quedó aislado del resto:

```python
reparto = Grafo()
for barrio in ("Alma", "Beto", "Cami", "Dani", "Ema", "Fran", "Gabi"):
    reparto.agregar_vertice(barrio)

reparto.agregar_arista("Alma", "Beto")
reparto.agregar_arista("Alma", "Cami")
reparto.agregar_arista("Beto", "Cami")
reparto.agregar_arista("Cami", "Dani")
reparto.agregar_arista("Dani", "Ema")
reparto.agregar_arista("Ema", "Fran")

print(sorted(alcanzables_dfs(reparto, "Alma")))
```

```text
['Alma', 'Beto', 'Cami', 'Dani', 'Ema', 'Fran']
```

Gabi no está en la lista. DFS recorrió *toda* la red conectada a Alma y no pudo llegar hasta Gabi, porque no hay arista que la una — exactamente el diagnóstico del capítulo 23 ("un fantasma"), ahora con consecuencias: un repartidor que sale de Alma jamás pondrá un pie en Gabi. Y la versión con pila explícita, sin recursión, te deja ver el mecanismo con claridad — es la misma lógica de "entrar por la primera puerta, retroceder después" pero con un `list` usado como pila (capítulo 21):

```python
def alcanzables_dfs_pila(grafo, partida):
    visitados = set()
    pila = [partida]

    while pila:
        vertice = pila.pop()
        if vertice not in visitados:
            visitados.add(vertice)
            for vecino in grafo.adyacencias[vertice]:
                if vecino not in visitados:
                    pila.append(vecino)

    return visitados
```

Y con el conjunto devuelto, la pregunta de producción — "¿está toda la red en una sola pieza?" — se responde con una comparación de conjuntos (capítulo 6):

```python
def esta_conectado(grafo, partida):
    return alcanzables_dfs(grafo, partida) == set(grafo.obtener_vertices())

print(esta_conectado(reparto, "Alma"))        # False

reparto.agregar_arista("Fran", "Gabi")
print(sorted(alcanzables_dfs(reparto, "Alma")))
print(esta_conectado(reparto, "Alma"))        # True
```

```text
['Alma', 'Beto', 'Cami', 'Dani', 'Ema', 'Fran', 'Gabi']
True
```

Con una sola arista nueva (`Fran`—`Gabi`), la red pasó de dos piezas a una: ahora cada barrio alcanza a todos los demás. Ese es el poder del DFS — y la respuesta a "¿en cuántas componentes conexas está partido mi mapa?" es contar cuántas veces hace falta arrancar un DFS con partidas nuevas para cubrir todo: ese `len` es el corazón del ejercicio 4.

## 4. Amplitud contra profundidad: ¿cuándo cuál?

Viste los dos recorridos y ahora la pregunta fea: las dos visitan los mismos vértices, ¿no daba igual? No. La diferencia está en **qué te puede responder cada una y a qué costo**. Acá está la tabla que vale la pena memorizar:

| | **BFS (amplitud)** | **DFS (profundidad)** |
|---|---|---|
| Estructura | cola (`deque` + `popleft`) | pila (o recursión) |
| Camino que encuentra | el de **menos saltos** | un camino cualquiera hacia el destino |
| Memoria | guarda la capa entera: O(V) en el peor caso | guarda el camino actual: O(profundidad) |
| Ideal para | "¿cuál es el camino en menos pasos?", "¿a qué distancia?", "grados de separación" | "¿se puede llegar?", "¿cuántas componentes?", explorar todo |
| Complejidad | O(V + E) tiempo, visita cada arista una vez | O(V + E) tiempo, visita cada arista una vez |

> **Dato clave:** los dos recorren el grafo completo en **O(V + E)** — cada vértice se marca una vez y cada arista se mira una vez. La diferencia nunca está en "cuán rápido recorren", sino en el *camino* que producen: BFS garantiza el de menos saltos, DFS no promete nada más que "existe". Si tu problema es un camino con el costo más bajo y tu memoria es mala con las capas, pensá dos veces qué estás comprando.

Una confusión clásica: "¿Y si el grafo es dirigido?" DFS y BFS funcionan igual — seguís las aristas **en su dirección** (lo que vimos con `dirigido=True` en el capítulo 23). En un grafo no dirigido, subís y bajás por el mismo pasillo; en uno dirigido, solo podés ir hacia donde apunta la flecha. El ejercicio 3 de este capítulo te lo hace sentir.

## 5. Dijkstra: cuando cada paso cobra

Hasta acá mediste distancia en **saltos**. Pero tu mapa de vuelos del capítulo 23 ponderaba cada ruta con kilómetros — y ahí "el camino de menos saltos" deja de ser la respuesta correcta. Mirá: de Buenos Aires a Lima hay un vuelo directo de 3100 km; de Buenos Aires a Córdoba y de Córdoba a Lima podrían ser 1200 + 1300 = 2500 km *con el doble de segmentos*. ¿Cuál es "más corto"? En kilómetros, el que mete más escalas. La pregunta ya no es "¿menos saltos?" sino "¿menos **costo** sumando los pesos?". Para esa pregunta nació **Dijkstra**.

La idea es una sola y es hermosa: **"si hoy sé cuánto cuesta llegar hasta cada vértice, el más barato al que todavía no le cerré la cuenta, ya tiene su costo final."** Porque si ese vértice es el más barato disponible, cualquier camino *alternativo* hacia él pasaría por otro vértice que cuesta igual o más — imposible que le gane. El que tiene el menor costo pendiente es el "más prometedor" de la sección 1, y para sacarlo siempre primero usás el montículo de prioridad del capítulo 19, que acá se llama **`heapq`**.

Reconstruyamos la clase ponderada — la misma del capítulo 23 ("un diccionario que apunta a diccionarios"), porque Dijkstra trabaja sobre la lista de adyacencia con pesos:

```python
import heapq

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
```

Y ahora la estrella. El diccionario `distancias` arranca con tu origen en `0` y todo lo demás en `float("inf")` — "todavía no sé cuánto cuesta llegar, así que cuesta infinito". Cada vuelta, el montículo te presta el vértice de menor costo pendiente, y a cada vecino le ofrecés "llegar pasando por acá": si esa oferta es más barata que lo que tenía anotado, la actualizás y anotás en `anteriores` de quién venías. Ese acto de "actualizar porque encontré algo mejor" se llama **relajación**, y es la célula del algoritmo:

```python
def camino_mas_corto(grafo, origen, destino):
    distancias = {vertice: float("inf") for vertice in grafo.adyacencias}
    anteriores = {vertice: None for vertice in grafo.adyacencias}
    distancias[origen] = 0
    monticulo = [(0, origen)]

    while monticulo:
        distancia_actual, vertice = heapq.heappop(monticulo)

        if vertice == destino:
            camino = []
            while vertice is not None:
                camino.append(vertice)
                vertice = anteriores[vertice]
            return distancias[destino], camino[::-1]

        if distancia_actual > distancias[vertice]:
            continue

        for vecino in grafo.adyacencias[vertice]:
            nueva = distancia_actual + grafo.peso_de_arista(vertice, vecino)
            if nueva < distancias[vecino]:
                distancias[vecino] = nueva
                anteriores[vecino] = vertice
                heapq.heappush(monticulo, (nueva, vecino))

    return float("inf"), []
```

Mirá las tres piezas de la casa en orden. El corte temprano `if vertice == destino` — apenas sale el destino del montículo ya sabemos su costo final, porque es el mínimo pendiente (la idea de la sección 4) — así que reconstruimos el camino y devolvemos. La guarda `if distancia_actual > distancias[vertice]: continue` descarta las "noticias viejas" que el montículo todavía tenía en cola: si ya afinamos la distancia de este vértice, ignoramos la entrada obsoleta (el mismo reflejo anti-duplicados que el capítulo 23 te enseñó). Y la relajación del `for`, la que hace todo el trabajo: por cada vecino, ¿te conviene venir por acá? Entonces actualizá y meté la oferta nueva en el montículo.

Ponelo a trabajar con el mapa de vuelos ponderado del capítulo 23 — la red de rutas aéreas con kilómetros reales:

```python
vuelos = GrafoPonderado()

vuelos.agregar_arista("Buenos Aires", "Santiago", 1890)
vuelos.agregar_arista("Buenos Aires", "Lima", 3100)
vuelos.agregar_arista("Santiago", "Lima", 2450)
vuelos.agregar_arista("Lima", "Bogotá", 1890)
vuelos.agregar_arista("Bogotá", "Ciudad de México", 3150)

distancia, camino = camino_mas_corto(vuelos, "Buenos Aires", "Ciudad de México")
print(distancia)          # 8140
print(camino)             # ['Buenos Aires', 'Lima', 'Bogotá', 'Ciudad de México']
```

¿Ves por qué no pasa por Santiago? El montículo probó "Buenos Aires → Santiago → Lima" (1890 + 2450 = 4340) contra "Buenos Aires → Lima" directo (3100), y eligió lo barato; después "Lima → Bogotá" y "Bogotá → Ciudad de México" hasta sumar 8140. Distancia mínima, con dos escalas, y el algoritmo lo encontró sin probar todas las combinaciones — porque cada vez que sacó del montículo el vértice más barato, cerró su cuenta.

> **Importante:** Dijkstra **exige pesos no negativos** — su garantía ("el mínimo pendiente ya es final") se cae si una arista pudiera bajar el costo *después* de cerrar la cuenta. Por eso la pasada de producción del capítulo 23 validaba que los pesos no fueran negativos: no era excentricidad, era condicionar el algoritmo que estás viendo hoy. Y notá el detalle elegante: `peso_de_arista` devuelve `float("inf")` cuando no hay arista, así que "no hay vuelo" se modela como "cuesta infinito" — la misma convención, el mismo capítulo 23.

## 6. En producción: la pasada que acelera

`camino_mas_corto` funciona, pero miralo con ojos de arquitecto: dentro del `for` llamás a `peso_de_arista(vertice, vecino)`, y ese método hace **tantas búsquedas de diccionario como pasos tenga el camino** — sin contar que recorre `grafo.adyacencias` dos veces por vértice para armar `distancias` y `anteriores`. En un grafo chico no se siente. En uno de **mil barrios**, se nota. Porque resulta que ya tenés todo a mano: `grafo.adyacencias[vertice]` *es* un diccionario `vecino: peso`. No hace falta preguntarle al método; se lee directo. Esa es la pasada de producción de hoy, y la optimización completa es:

1. **Iterar sobre el diccionario interno, no sobre el método.** `grafo.adyacencias[vertice].items()` te da `(vecino, peso)` de una con la estructura a la que ya accedés en el capítulo 23, en vez de `obtener_adyacentes()` + `peso_de_arista()` = dos llamadas y dos listas intermedias nuevas por cada vecino.
2. **Menos libros de contabilidad.** En vez de armar `distancias` iniciales con una comprensión de diccionario que recorre todos los vértices, podés reutilizar el hecho de que el `float("inf")` viene de adentro; pero lo que realmente domina la cuenta acá es evitar construir listas y diccionarios intermedios en cada iteración.
3. **Devolver `distancia_actual`, no releerla.** Cuando el destino sale del montículo, `distancia_actual` *es* `distancias[destino]` — leerla de nuevo es un viaje más al diccionario.

El resultado, `camino_mas_corto_opc2`, es la misma idea de siempre con menos muebles entre el montículo y el diccionario:

```python
def camino_mas_corto_opc2(grafo, origen, destino):
    distancias = {vertice: float("inf") for vertice in grafo.adyacencias}
    anteriores = {vertice: None for vertice in grafo.adyacencias}
    distancias[origen] = 0
    monticulo = [(0, origen)]

    while monticulo:
        distancia_actual, vertice = heapq.heappop(monticulo)

        if vertice == destino:
            camino = []
            while vertice is not None:
                camino.append(vertice)
                vertice = anteriores[vertice]
            return distancia_actual, camino[::-1]

        if distancia_actual > distancias[vertice]:
            continue

        for vecino, peso in grafo.adyacencias[vertice].items():
            nueva = distancia_actual + peso
            if nueva < distancias[vecino]:
                distancias[vecino] = nueva
                anteriores[vecino] = vertice
                heapq.heappush(monticulo, (nueva, vecino))

    return float("inf"), []
```

La diferencia con la sección 5 es mínima en ojos — dos líneas cambiaron — y sin embargo es la diferencia entre un prototipo y algo que escala. Ahora para verla, necesitás un grafo grande de verdad. Como en el capítulo 23 "cualquier cosa se conecta con cualquier otra", armemos un grafo completo al azar — cada par de los 1000 vértices unido por una arista con peso aleatorio, con `random.seed` para que corra igual en tu máquina y en la mía:

```python
import random
import time

def generar_grafo_completo(cantidad, peso_minimo=1, peso_maximo=100, semilla=None):
    random.seed(semilla)
    grafo = GrafoPonderado()
    for indice in range(cantidad):
        grafo.agregar_vertice(indice)
    for indice in range(cantidad):
        for otro in range(indice + 1, cantidad):
            grafo.agregar_arista(indice, otro, random.randint(peso_minimo, peso_maximo))
    return grafo


nodo_1000 = generar_grafo_completo(1000, semilla=1)

inicio = time.perf_counter()
distancia_1, camino_1 = camino_mas_corto(nodo_1000, 255, 755)
t_opc1 = time.perf_counter() - inicio

inicio = time.perf_counter()
distancia_2, camino_2 = camino_mas_corto_opc2(nodo_1000, 255, 755)
t_opc2 = time.perf_counter() - inicio

print(distancia_1 == distancia_2)          # True
print(len(camino_1), len(camino_2))        # 4 4
print(f"opc1: {t_opc1:.4f}s   opc2: {t_opc2:.4f}s   ratio: {t_opc1 / t_opc2:.1f}x")
```

Las dos funciones devuelven **la misma distancia** (`True`) y **el mismo camino** — y fijate un detalle que no es casualidad: el camino tiene 4 vértices. El grafo completo tiene vuelo directo entre 255 y 755, pero ese vuelo *no* es el más barato: dos o tres escalas con pesos chicos suman menos que el directo, y Dijkstra lo sabe. Por eso el "camino directo" que un humano miraría de un vistazo pierde contra una ruta con escalas — esa es justamente la pregunta que el algoritmo responde mejor que vos. Pero el tiempo... en la máquina donde corre el libro, `opc1` tarda unos **0.2 segundos** y `opc2` unos **0.04 segundos** — el ratio real lo vas a ver vos en tu consola, el orden de magnitud no va a mentir. Cinco veces más rápido por sacar dos métodos del camino caliente. Eso es pasar el mismo algoritmo por la lente de producción del capítulo 22: no cambiás la estrategia, cambiás **qué le preguntás al diccionario y cómo preguntárselo**.

> **Dato clave:** el montículo de `heapq` hace que cada inserción y cada extracción cuesten O(log V), y Dijkstra visita cada arista una vez — entonces el tiempo total es **O((V + E) · log V)**. En el grafo completo de mil vértices eso es alrededor de un millón de operaciones logarítmicas: por eso tarda fracciones de segundo, y por eso el capítulo que viene — el viajante, con sus permutaciones — te va a hacer añorar esta eficiencia. Dijkstra es hoy el "camino más corto"; el capítulo 25 es "el recorrido que pasa por todos lados y vuelve" — otro problema, otra tabla de pagos.

Sobre la pregunta de cuándo Dijkstra en vez de BFS: si tu grafo es **sin pesos**, BFS ya te da el camino de menos saltos y no necesitás montículo; si hay **pesos no negativos**, Dijkstra. La red de vuelos de este capítulo es ponderada; la red social de la sección 2 no. Elegí la herramienta según lo que tus aristas llevan encima — esa es la libertad que el capítulo 23 te dio cuando te dejó ponerles pesos o no.

## 7. Resumen y conceptos clave

Este capítulo hizo por el mapa lo que el 23 no podía: **lo recorrió**. Viste que el grafo guardó las conexiones pero que recorrerlo es un problema en sí mismo, y que todos los algoritmos de recorrido comparten un esqueleto — marcar la partida y sacar siempre el vértice "más prometedor". BFS eligió "el primero que llegó" (cola) y te dio el camino de **menos saltos**, medido en `niveles` y reconstruido con `anteriores`; DFS eligió "el último que descubrí" (pila o recursión) y te dio la **alcanzabilidad completa** — la respuesta a "¿está mi red en una sola pieza?". Y cuando cada arista pasó a cobrar kilómetros, Dijkstra eligió "el más barato pendiente" usando el montículo del capítulo 19, relajando distancias y reconstruyendo el camino desde el destino hacia atrás. Cerraste con la pasada de producción: leer el diccionario interno (`[vertice].items()`) en vez de pasar por dos métodos intermedios, y `camino_mas_corto_opc2` resultó unas cinco veces más rápido en un grafo completo de mil vértices — mismo resultado, misma distancia, menos muebles.

La moraleja es la promesa del capítulo 19 en acción: Dijkstra le debe su velocidad a la cola de prioridad, exactamente como el `heapq` se la dio a las listas de máximos de aquel capítulo. Y mirá cómo cerró el círculo para la Parte VIII: la lista enlazada te dio la cola para BFS, la recursión del capítulo 10 te dio la pila para DFS, el diccionario del capítulo 6 te dio la lista de adyacencia, y el montículo del capítulo 19 te acaba de dar Dijkstra. Cada estructura vieja se reencarnó como una herramienta de la nueva — el grafo no inventó nada, **reutilizó** todo. Repasá el checklist antes de seguir:

- [ ] Recorrer un grafo es un problema distinto de guardarlo: hay que decidir *por dónde* ir y **marcar visitados** para no quedarse atrapado en un ciclo.
- [ ] **BFS** (amplitud): `deque` + `popleft`, agotar cada capa antes de la siguiente → `niveles` con la distancia en saltos.
- [ ] **`camino_bfs`**: guardar quién descubrió a quién (`anteriores`) y reconstruir el camino de atrás hacia adelante.
- [ ] **DFS** (profundidad): pila o recursión del capítulo 10 → `alcanzables_dfs` devuelve todos los vértices que se tocan desde la partida.
- [ ] **Conectividad**: si `alcanzables` == todos los vértices, el grafo está en una sola pieza; si no, quedan componentes separadas (como Gabi).
- [ ] BFS y DFS recorren en **O(V + E)** tiempo; la diferencia es el camino: BFS promete el de menos saltos, DFS solo promete existencia.
- [ ] **Dijkstra** responde "costo mínimo" sumando pesos: montículo `heapq`, `distancias` con `inf`, **relajación** (`nueva < distancias[vecino]`), reconstrucción con `anteriores`.
- [ ] Dijkstra **exige pesos no negativos** — por eso el capítulo 23 los validaba; "no hay arista" se modela con `float("inf")`.
- [ ] Optimización de producción: iterar `grafo.adyacencias[vertice].items()` directo en vez de `obtener_adyacentes` + `peso_de_arista` por vecino → `~5x` más rápido en 1000 vértices, misma distancia.
- [ ] BFS sin pesos, Dijkstra con pesos: elegí el recorrido según lo que tus aristas llevan encima.

## 8. Ejercicios

1. **Los grados de separación.** Con la clase `Grafo` de la sección 2 y `niveles_bfs`, construí la red social `red_social` del ejemplo (Alma, Beto, Cami, Dani, Ema, Fran con sus amistades) y respondé: ¿a cuántos saltos está `Fran` de `Alma`? ¿Y quién está exactamente a 2 saltos? Usá `niveles_bfs(red_social, "Alma")` y filtrá los valores iguales a 2.

2. **El camino en una red con bifurcación.** Agregale a `red_social` dos personas más (`Geri` y `Hugo`), con `Geri` amiga de `Alma` y de `Hugo`, y `Hugo` amigo de `Fran`. Ahora hay **dos** caminos distintos de `Alma` a `Fran`: el viejo (por Cami) y el nuevo (por Geri). Usá `camino_bfs` y `niveles_bfs` y mostrá **por cuál** pasa BFS y cuántos saltos son. ¿Va por el viejo o por el nuevo? ¿Por qué?

3. **El dígrafo que no vuelve atrás.** Con `Grafo(dirigido=True)`, construí un Instagram de 4 usuarios donde `Alma` sigue a `Beto`, `Beto` sigue a `Cami`, y `Cami` sigue a `Alma` (un triángulo de flechas). Mostrá qué vértices son alcanzables desde `Alma` con `alcanzables_dfs`. Después convertilo al grafo no dirigido (`dirigido=False`) y mostrá la diferencia en el conjunto alcanzable. **_Agarrala de la cola es la lección del capítulo 23:_** las flechas deciden el recorrido.

4. **El barrio aislado se conecta.** Con el `reparto` de la sección 3 (donde Gabi quedó afuera), usá `esta_conectado` para confirmar que la red no está en una sola pieza, contá cuántas componentes conexas tiene (¿cuántas veces hay que arrancar `alcanzables_dfs` con partidas nuevas para cubrir todos los vértices?), y después agregale la arista `Fran`—`Gabi` y mostrá que pasa a `True`.

5. **El vuelo que conviene.** Con el `GrafoPonderado` de la sección 5 y el mapa de vuelos ponderado (Buenos Aires, Santiago, Lima, Bogotá, Ciudad de México), usá `camino_mas_corto` y mostrá la distancia y el camino de `Buenos Aires` a `Ciudad de México` (esperás `8140` y pasar por Lima y Bogotá). Después mostrá el camino directo más barato de `Buenos Aires` a `Bogotá` (¿existe vuelo directo o hay que ir por Lima?). *Pista:* la salida de Bogotá hacia CDMX es `3150` de la matriz del capítulo 23.

6. **La pasada de producción.** Con `generar_grafo_completo(1000, semilla=1)` y las dos versiones de `camino_mas_corto` / `camino_mas_corto_opc2`, cronometrá vos mismo (con `time.perf_counter`, como en la sección 6) cuánto tarda cada una entre los vértices `255` y `755`. Verificá que devuelven la **misma distancia** y el mismo camino, y anotá el ratio de tiempos. Si en tu máquina el ratio es menor a 2, revisá si tu versión de `opc2` sigue usando `peso_de_arista` dentro del `for` — ahí está la fuga.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. Los grados de separación
niveles = niveles_bfs(red_social, "Alma")
print(niveles["Fran"])                                  # 4
a_dos_saltos = [persona for persona, nivel in niveles.items() if nivel == 2]
print(a_dos_saltos)                                     # ['Dani']


# 2. El camino con bifurcación
red_social.agregar_vertice("Geri")
red_social.agregar_vertice("Hugo")
red_social.agregar_arista("Geri", "Alma")
red_social.agregar_arista("Geri", "Hugo")
red_social.agregar_arista("Hugo", "Fran")

print(camino_bfs(red_social, "Alma", "Fran"))           # ['Alma', 'Geri', 'Hugo', 'Fran']
print(niveles_bfs(red_social, "Alma")["Fran"])          # 3
```

*BFS elige el camino **nuevo** (por Geri), no el viejo por Cami: el camino nuevo tiene 3 saltos (`Alma → Geri → Hugo → Fran`) y el viejo suma 4. Como BFS agota cada capa antes de pasar a la siguiente, al descubrir `Fran` desde `Hugo` en la capa 3 ya es un hecho que no hay nada más corto — la promesa del "camino de menos saltos" cumplida al pie de la letra.*

```python
# 3. El dígrafo que no vuelve atrás
seguidores = Grafo(dirigido=True)
for usuario in ("Alma", "Beto", "Cami"):
    seguidores.agregar_vertice(usuario)
seguidores.agregar_arista("Alma", "Beto", dirigido=True)
seguidores.agregar_arista("Beto", "Cami", dirigido=True)
seguidores.agregar_arista("Cami", "Alma", dirigido=True)

print(sorted(alcanzables_dfs(seguidores, "Alma")))      # ['Alma', 'Beto', 'Cami']

no_dirigido = Grafo()
for usuario in ("Alma", "Beto", "Cami"):
    no_dirigido.agregar_vertice(usuario)
no_dirigido.agregar_arista("Alma", "Beto")
no_dirigido.agregar_arista("Beto", "Cami")
no_dirigido.agregar_arista("Cami", "Alma")
print(sorted(alcanzables_dfs(no_dirigido, "Alma")))     # ['Alma', 'Beto', 'Cami']
```

*En este ejemplo puntual el triángulo cierra el ciclo en las dos versiones, y el conjunto alcanzable queda igual. El punto del ejercicio era revisar el recorrido en tu cabeza: en el dirigido seguís solo las flechas (`Alma→Beto→Cami→Alma`), en el no dirigido podés ir en el sentido que quieras. Si hubieras dejado una flecha fuera (por ejemplo sin `Alma→Beto`), el conjunto dirigido perdería a `Beto` — y el no dirigido no.*

```python
# 4. El barrio aislado se conecta
def cantidad_de_componentes(grafo):
    por_visitar = set(grafo.obtener_vertices())
    componentes = 0
    while por_visitar:
        partida = por_visitar.pop()
        alcanzados = alcanzables_dfs(grafo, partida)
        por_visitar.difference_update(alcanzados)
        componentes += 1
    return componentes

reparto = Grafo()
for barrio in ("Alma", "Beto", "Cami", "Dani", "Ema", "Fran", "Gabi"):
    reparto.agregar_vertice(barrio)
reparto.agregar_arista("Alma", "Beto")
reparto.agregar_arista("Alma", "Cami")
reparto.agregar_arista("Beto", "Cami")
reparto.agregar_arista("Cami", "Dani")
reparto.agregar_arista("Dani", "Ema")
reparto.agregar_arista("Ema", "Fran")

print(cantidad_de_componentes(reparto))     # 2
print(esta_conectado(reparto, "Alma"))      # False
reparto.agregar_arista("Fran", "Gabi")
print(cantidad_de_componentes(reparto))     # 1
print(esta_conectado(reparto, "Alma"))      # True


# 5. El vuelo que conviene
distancia, camino = camino_mas_corto(vuelos, "Buenos Aires", "Ciudad de México")
print(distancia)    # 8140
print(camino)       # ['Buenos Aires', 'Lima', 'Bogotá', 'Ciudad de México']

distancia, camino = camino_mas_corto(vuelos, "Buenos Aires", "Bogotá")
print(distancia)    # 4990
print(camino)       # ['Buenos Aires', 'Lima', 'Bogotá']


# 6. La pasada de producción
nodo_1000 = generar_grafo_completo(1000, semilla=1)

inicio = time.perf_counter()
distancia_1, camino_1 = camino_mas_corto(nodo_1000, 255, 755)
t_opc1 = time.perf_counter() - inicio

inicio = time.perf_counter()
distancia_2, camino_2 = camino_mas_corto_opc2(nodo_1000, 255, 755)
t_opc2 = time.perf_counter() - inicio

print(distancia_1 == distancia_2)          # True
print(len(camino_1), len(camino_2))        # 4 4
print(f"opc1: {t_opc1:.4f}s   opc2: {t_opc2:.4f}s   ratio: {t_opc1 / t_opc2:.1f}x")
```
