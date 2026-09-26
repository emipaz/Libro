# Capítulo 25 — Grafos III: el viajante y la explosión combinatoria

Terminá de escribir el último método del capítulo anterior y la app de reparto pasa la prueba: sabe decir cuánto cuesta el camino más barato entre dos barrios. Pero esa noche el jefe te llama con una pregunta nueva, y no es sobre dos barrios: *"Necesito que un repartidor visite todos los barrios, pase exactamente una vez por cada uno, y vuelva al depósito. ¿Cuál es el recorrido más barato?"*

Si lo pensás un segundo, ves que no es una pregunta más difícil del capítulo 24 — es **otro tipo** de pregunta. Dijkstra busca un camino entre dos puntos y no le importa nada más. Acá no hay dos puntos: hay que **armar un circuito entero** que toque cada vértice una sola vez y cierre volviendo al inicio. Es el famoso **Problema del Viajante** (TSP, *Traveling Salesperson Problem*), y tiene una propiedad que lo vuelve el enemigo más hermoso de esta parte del libro: cuando agregás un barrio, el trabajo **no crece un poco — explota**.

Este capítulo te da las tres respuestas que la industria usa, en orden de peligrosidad. Primero la inocente: **probar todas las rutas** (fuerza bruta), que para 4 barrios resuelve todo en un chasquido y para 12 ya es un programa que no termina nunca — porque las permutaciones del capítulo 3 ("De bits") crecen como `n!`. Después la elegante: **la programación dinámica con máscaras de bits**, que baja la cuenta de `n!` a `n² · 2ⁿ` — el mismo "no repetir trabajo" de la memoización del capítulo 19, aplicado a conjuntos. Y cuando hasta eso se queda corto, la respuesta de producción: **la heurística del vecino más cercano**, que no promete el óptimo pero corre en un grafo de mil vértices en milisegundos. Al final vas a saber exactamente cuándo usar quién — y a leer una tabla de permutaciones sin asustarte.

---

## 1. El problema del viajante: el circuito y la explosión

Un día de reparto es esto: un depósito, un montón de barrios, y la regla de oro de visitar **cada barrio exactamente una vez** y **volver al depósito**. El resultado, si lo dibujás, es un circuito cerrado que toca todos los vértices y cierra sobre sí mismo — los matemáticos lo llaman **circuito hamiltoniano**, pero vos vas a llamarlo simplemente **recorrido**. La diferencia con todo lo anterior:

| Término | Qué significa en el TSP |
|---|---|
| **Grafo completo** | el del capítulo 23: todo par de vértices tiene su arista; el viajante puede ir de donde sea a donde sea |
| **Recorrido** | el circuito que visita cada vértice una sola vez y vuelve al inicio |
| **Permutación** | un orden de los vértices intermedios; cada permutación distinta es una ruta distinta |
| **Costo de un recorrido** | la suma de los pesos de sus aristas, una por paso |
| **Óptimo** | el recorrido de menor costo; eso y nada menos es lo que pedimos |
| **Heurística** | un algoritmo que encuentra un buen recorrido pero no garantiza el óptimo — la herramienta de producción |

La pregunta de Dijkstra era "¿cuál es el camino de *menos costo* entre A y B?". La del viajante es: "¿cuál es el **circuito cerrado** de menos costo que pase por todos?". Son hermanos gemelos que de grandes se fueron a mundos distintos: Dijkstra se resuelve rápido incluso con miles de nodos; el TSP, en su versión exacta, es un problema que la computación llama **NP-difícil**, y te va a tocar ver con tus propios ojos por qué. El motivo de fondo es la **explosión combinatoria**: la cantidad de rutas posibles crece como el factorial de la cantidad de barrios. Mirá la tabla que ya conoés del capítulo 3 pero ahora con un significado brutal:

| Barrios | Rutas a probar |
|---|---|
| 4 | 6 |
| 6 | 120 |
| 8 | 5 040 |
| 10 | 362 880 |
| 12 | 39 916 800 |
| 14 | 6 227 020 800 |

Entre 10 y 12 barrios, la cuenta se multiplica por 110. Entre 10 y 14, por más de **17 000**. No es que "tarda un poco más": es otra categoría de tiempo, y ese salto es todo lo que este capítulo intenta domesticar.

## 2. Fuerza bruta: la respuesta inocente

El camino más directo de atacar el TSP es también el más honesto: **si puedo listar todas las permutaciones, mido el costo de cada una y me quedo con la mejor, tengo el óptimo garantizado.** Para 4 barrios es tan simple que se puede resolver a mano. La clase ponderada es la del capítulo 24, compacta para que este capítulo siga siendo autocontenido — y la función generadora de grafos completos también:

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

    def obtener_adyacentes(self, vertice):
        return list(self.adyacencias.get(vertice, {}).keys())

    def peso_de_arista(self, origen, destino):
        return self.adyacencias.get(origen, {}).get(destino, float("inf"))

    def obtener_vertices(self):
        return list(self.adyacencias.keys())


import random

def generar_grafo_completo(cantidad, peso_minimo=1, peso_maximo=100, semilla=None):
    random.seed(semilla)
    grafo = GrafoPonderado()
    for indice in range(cantidad):
        grafo.agregar_vertice(indice)
    for indice in range(cantidad):
        for otro in range(indice + 1, cantidad):
            grafo.agregar_arista(indice, otro, random.randint(peso_minimo, peso_maximo))
    return grafo
```

Con `semilla` fija, el mismo grafo sale siempre igual, en tu máquina y en la mía — el mismo truco del capítulo 24, y la garantía de que los números que leas acá van a ser los mismos que veas vos. Ahora la fuerza bruta de 4 barrios, con el depósito en el vértice `0` y los tres barrios `1`, `2`, `3` por recorrer. Las permutaciones de `[1, 2, 3]` son seis, y el `itertools` del capítulo 9 las genera todas:

```python
import itertools

nodo_4 = generar_grafo_completo(4, peso_maximo=100, semilla=1)

recorridos = []
for permutacion in itertools.permutations([1, 2, 3]):
    ruta = (0,) + permutacion + (0,)
    costo = sum(nodo_4.peso_de_arista(ruta[i], ruta[i + 1]) for i in range(len(ruta) - 1))
    recorridos.append((costo, ruta))

for costo, ruta in recorridos:
    print(costo, ruta)
```

```text
141 (0, 1, 2, 3, 0)
140 (0, 1, 3, 2, 0)
213 (0, 2, 1, 3, 0)
140 (0, 2, 3, 1, 0)
213 (0, 3, 1, 2, 0)
141 (0, 3, 2, 1, 0)
```

Seis rutas, dos de ellas atan en 140 — el recorrido óptimo cuesta **140** y hay dos maneras de lograrlo (una es el espejo de la otra). Fijate el patrón que va a sentar la tónica del capítulo: el costo de "1 → 2 → 3" es 141, pero "1 → 3 → 2" es 140. El orden de visita cambia el costo de todo, y ese orden está en las permutaciones. Ahora la pregunta de verdad: ¿qué pasa cuando no hay 4 barrios, sino 8? La versión general tiene el esqueleto exacto de arriba — probar todas las permutaciones de los vértices intermedios y quedarse con la mejor:

```python
def recorrido_viajante_opc1(grafo, inicio):
    vertices = [v for v in grafo.obtener_vertices() if v != inicio]
    mejor_costo = float("inf")
    mejor_ruta = None

    for permutacion in itertools.permutations(vertices):
        ruta = (inicio,) + tuple(permutacion) + (inicio,)
        costo = sum(grafo.peso_de_arista(ruta[i], ruta[i + 1]) for i in range(len(ruta) - 1))
        if costo < mejor_costo:
            mejor_costo = costo
            mejor_ruta = ruta

    return mejor_costo, mejor_ruta


nodo_8 = generar_grafo_completo(8, peso_maximo=100, semilla=1)
print(recorrido_viajante_opc1(nodo_8, 0))
```

```text
(194, (0, 4, 6, 1, 7, 3, 2, 5, 0))
```

El óptimo de 8 barrios: **194**, con la ruta `0 → 4 → 6 → 1 → 7 → 3 → 2 → 5 → 0`. Cinco mil cuatrocientas permutaciones y el programita no se inmuta — tardó unos milisegundos. Pero acá está el detalle que define todo: a 8 barrios le tocó `7! = 5040` permutaciones. A 10 le tocan `9! = 362 880`. A 14, más de seis mil millones. La fuerza bruta **funciona perfecto y nunca llega** — no hay un solo error de lógica en el código, y sin embargo es inútil para el mundo real. Esa es la lección más cara de la programación: un algoritmo correcto y muy lento es un algoritmo muerto.

> **Dato clave:** probar todas las rutas es **O(n!)**: por cada permutación sumás `n` pesos. Para `n = 10` son ~360 mil rutas; para `n = 14`, más de 6 **mil millones**. La brute force entrega el óptimo garantizado — el problema es que "garantizado" y "a tiempo" no viajan juntos.

## 3. La pasada elegante: programación dinámica con máscaras de bits

Si probar todas las permutaciones es tirar el trabajo a la basura, la solución es el reflejo que ya entrenaste en el capítulo 19: **no repetir trabajo**. Y acá hay un montón de trabajo repetido. Pensá en el grafo de 8 barrios: la ruta `0 → 4 → 6 → 1 → 7 → ...` y la ruta `0 → 6 → 4 → 1 → 7 → ...` comparten exactamente el mismo asunto de fondo: "¿cuál es la mejor manera de visitar el conjunto `{4, 6, 1, 7, 3, 2, 5}` terminando en tal vértice?". La fuerza bruta recalcula ese subproblema un montón de veces; la programación dinámica lo calcula **una sola vez** y lo reutiliza. La estructura de la lista enlazada y el árbol te enseñaron a memorizar; acá memorizás costos de subconjuntos.

Pero hay un problema de representación: un subconjunto de vértices no tiene un "índice" natural para meterlo en un diccionario. La solución viene de una sección del capítulo 3 que te debe haber parecido juguetería en su momento — **"De bits"**. Resulta que un número entero *es* un conjunto si querés: el bit `i` prendido significa "el vértice `i` está en el conjunto". Con `n = 8` barrios intermedios, el número `0b10110111` dice exactamente qué combinación de barrios visitamos. Las operaciones de bits del capítulo 3 se vuelven operaciones de conjunto:

| Operación | Qué hace |
|---|---|
| `1 << i` | prende solo el bit `i` — el conjunto `{i}` |
| `mascara & (1 << i)` | ¿está `i` en el conjunto? (distinto de 0 si sí) |
| `mascara | (1 << i)` | agregar `i` al conjunto |
| `mascara ^ (1 << i)` | sacar `i` del conjunto |

Ahora planteamos el problema de la forma que la memoización quiere. Definimos `dp[(mascara, ultimo)]` = el costo mínimo de un camino que arranca en el depósito, visita **exactamente** el conjunto `mascara` y termina en el vértice `ultimo`. El caso base es trivial: visitar un solo barrio `i` cuesta `peso(inicio, i)`. Y la regla de construcción es la de cualquier camino: el mejor camino a `mascara` terminando en `ultimo` es el mejor camino al conjunto de *antes* (sin `ultimo`) terminando en `previo`, más `peso(previo, ultimo)`. Eso es la **subestructura óptima**: la solución grande se arma sobre soluciones chicas ya resueltas, sin recalcular. Al final, cerramos el circuito sumándole a cada `(todo, ultimo)` el viaje de vuelta al inicio:

```python
def recorrido_viajante_dp(grafo, inicio):
    vertices = [v for v in grafo.obtener_vertices() if v != inicio]
    n = len(vertices)
    en_orden = {i: vertice for i, vertice in enumerate(vertices)}

    dp = {}
    padre = {}

    for i, vertice in enumerate(vertices):
        dp[(1 << i, i)] = grafo.peso_de_arista(inicio, vertice)

    for mascara in range(1, 1 << n):
        for ultimo in range(n):
            if not (mascara & (1 << ultimo)):
                continue
            clave = (mascara, ultimo)
            if clave in dp:
                continue
            sin_ultimo = mascara ^ (1 << ultimo)
            if sin_ultimo == 0:
                continue
            mejor = float("inf")
            mejor_previo = None
            for previo in range(n):
                if sin_ultimo & (1 << previo):
                    candidato = dp[(sin_ultimo, previo)] + grafo.peso_de_arista(en_orden[previo], en_orden[ultimo])
                    if candidato < mejor:
                        mejor = candidato
                        mejor_previo = previo
            dp[clave] = mejor
            padre[clave] = mejor_previo

    total = (1 << n) - 1
    costo_final = float("inf")
    ultimo_final = None
    for ultimo in range(n):
        candidato = dp[(total, ultimo)] + grafo.peso_de_arista(en_orden[ultimo], inicio)
        if candidato < costo_final:
            costo_final = candidato
            ultimo_final = ultimo

    camino = []
    mascara = total
    ultimo = ultimo_final
    while mascara:
        camino.append(en_orden[ultimo])
        previo = padre.get((mascara, ultimo))
        mascara = mascara ^ (1 << ultimo)
        ultimo = previo

    return costo_final, tuple([inicio] + camino[::-1] + [inicio])


print(recorrido_viajante_dp(nodo_8, 0))
```

```text
(194, (0, 5, 2, 3, 7, 1, 6, 4, 0))
```

**194** — la misma cifra que la fuerza bruta, porque el óptimo es uno solo. La ruta es distinta (`0 → 5 → 2 → 3 → 7 → 1 → 6 → 4 → 0`) pero cuesta lo mismo: hay más de un recorrido óptimo, y cualquiera de los dos está bien. Eso es la señal de que los dos algoritmos están **encontrando el mismo número por caminos distintos**. Mirá el precio que pagamos por esa elegancia: en vez de pasar por todas las permutaciones (`n!`), el bucle recorre todas las máscaras (hay `2ⁿ`) y por cada una prueba `n` últimos y `n` previos. Total: **O(n² · 2ⁿ)**. Para `n = 8`: en vez de 5040 operaciones con estructura, unas pocas miles a lo sumo; para `n = 10`, la diferencia es abismal.

> **Importante:** programación dinámica no es un truco mágico — es **la misma memoización del capítulo 19 con una representación más barata**: en vez de la clave `(conjunto_visitado, ultimo)` hecha con tuplas y listas, una clave `(int, int)` hecha con la máscara de bits del capítulo 3. La máscara no solo ahorra memoria: hace que "¿este vértice ya fue visitado?" sea un `&` en vez de una búsqueda en lista, y que "agregalo" sea un `|`. Las herramientas del capítulo 3 no eran un pasatiempo; eran las piezas que faltaban para este momento.

## 4. En producción: cuando el exacto no alcanza

La programación dinámica es hermosa, y tiene un techo duro: `O(n² · 2ⁿ)` con `n = 40` ya es un número del orden de los trillones. Tu mapa de reparto tiene **mil barrios**. No hay Ó(exacto) que sobreviva a eso — y no es un límite de tu código: el TSP exacto es NP-difícil, no se conoce un algoritmo que lo resuelva en tiempo polinómico, y los investigadores creen que no existe. Asique la industria no espera el óptimo: espera **un buen recorrido, rápido**. Ahí entra la heurística.

La más famosa de todas es la del **vecino más cercano**, y ya la conocés sin saberlo: es la versión greedy del viajante. Arrancás en el depósito, y en cada paso te movés al barrio **no visitado más cercano** — el mismo "agarra el más barato a mano" del capítulo 24, pero decidido paso a paso, sin mirar el mapa completo:

```python
def recorrido_viajante_aprox(grafo, inicio):
    sin_visitar = set(grafo.obtener_vertices())
    sin_visitar.remove(inicio)
    actual = inicio
    ruta = [inicio]
    costo = 0

    while sin_visitar:
        vecino = min(sin_visitar, key=lambda v: grafo.peso_de_arista(actual, v))
        costo += grafo.peso_de_arista(actual, vecino)
        actual = vecino
        sin_visitar.remove(actual)
        ruta.append(actual)

    costo += grafo.peso_de_arista(actual, inicio)
    ruta.append(inicio)
    return costo, tuple(ruta)


print(recorrido_viajante_aprox(nodo_8, 0))
```

```text
(211, (0, 4, 7, 3, 2, 5, 1, 6, 0))
```

Ese costo, **211**, es más caro que el óptimo de 194 — la heurística no encontró el mejor recorrido. Y ese es exactamente el punto: te da una **buena** solución (no desastre) en un tiempo que el exacto nunca podría soñar. Por cada paso barre los vértices sin visitar: **O(n²)**, con cero permutaciones. Mirá qué pasa cuando la dejamos suelta en un grafo de trescientos barrios:

```python
nodo_300 = generar_grafo_completo(300, peso_maximo=100, semilla=1)
distancia_300, ruta_300 = recorrido_viajante_aprox(nodo_300, 0)
print(distancia_300)     # 852
print(len(ruta_300))     # 301
```

El recorrido toca los 300 barrios una vez cada uno (el camino cerrado tiene 301 pasos: los 300 barrios más el regreso al depósito) y cuesta 852 unidades de peso — imposible de calcular en forma exacta en segundos, trivial en milisegundos con la heurística. ¿Es 852 el óptimo? No lo sabemos, y con 300 barrios ni siquiera podemos preguntárselo a la computadora en un tiempo razonable. Esa es la negociación profesional: **exactitud contra viabilidad**, y el lente de producción del capítulo 22 brilla acá como en ningún lado.

> **Dato clave:** el viajante nos deja una de las lecciones más grandes de la Parte VIII: hay problemas donde **lo perfecto es enemigo de lo posible**. La fuerza bruta da el óptimo y muere en las permutaciones; la programación dinámica baja la cuenta a `n²·2ⁿ` y tambien muere, más tarde; la heurística del vecino más cercano da "un buen recorrido" en `O(n²)` y vive para siempre. Elegir el algoritmo es elegir **la pregunta que tu negocio de verdad necesita responder**: el óptimo exacto de 8 barrios o un recorrido decente de 1000 barrios entregado a tiempo. Los dos son respuestas correctas — solo que a preguntas distintas.

## 5. Resumen y conceptos clave

Este capítulo te llevó desde "probar todo" hasta "ceder lo justo para sobrevivir", y los tres escalones te dejaron tres herramientas que usás según el tamaño. La **fuerza bruta** (`recorrido_viajante_opc1`) lista todas las permutaciones, mide cada ruta y garantiza el óptimo — hasta que las permutaciones explotan (`O(n!)`: de 8 a 14 barrios la cuenta se multiplica por más de un millón). La **programación dinámica** (`recorrido_viajante_dp`) dejó de repetir trabajo: representó los subconjuntos visitados como **máscaras de bits** (las herramientas del capítulo 3) y armó `dp[(mascara, ultimo)]`, el costo mínimo de visitar ese conjunto terminando en `ultimo`, bajando la cuenta a `O(n²·2ⁿ)` — el mismo reflejo de memoización del capítulo 19, ahora con enteros por clave. Y la **heurística del vecino más cercano** (`recorrido_viajante_aprox`) tiró por la borda la garantía del óptimo para quedarse con la viabilidad: `O(n²)`, un buen recorrido en milésimas, la herramienta para el mapa real de mil barrios. Las tres dieron el óptimo de 4 barrios (`140`), la fuerza bruta y la DP coincidieron mágicamente en 8 barrios (`194`), y la heurística entregó un digno `211` — más caro, pero a tiempo.

La moraleja que se suma a las de los capítulos 23 y 24: **no existe un algoritmo "mejor" en abstracto, existe el algoritmo indicado para el tamaño y la garantía que tu problema exige.** Dijkstra es polinómico y lo usás sin pensarlo; el TSP exacto es combinatorio y te obliga a elegir. Y en esa elección estuvo todo el capítulo: saber contar permutaciones, entender qué hace el `&` y el `|` con los bits de un conjunto, y tener el coraje de cambiar "el óptimo" por "a tiempo". Esa negociación no se aprende en el capítulo del algoritmo; se aprende con experiencia — y hoy diste el primer paso. Repasá el checklist antes de seguir:

- [ ] El **TSP** busca el circuito que visita cada vértice una sola vez y vuelve al inicio con el menor costo — otra pregunta que la de Dijkstra.
- [ ] La cantidad de rutas posibles crece como **`n!`**: 4 barrios → 6, 10 barrios → 362 880, 14 → mil millones. Esa es la **explosión combinatoria**.
- [ ] **Fuerza bruta**: probá todas las permutaciones (`itertools.permutations`), quedate con la de menor costo — óptimo garantizado, `O(n!)`.
- [ ] **Máscaras de bits** (capítulo 3): el bit `i` de un entero dice si el vértice `i` está en el conjunto visitado; `&` pregunta, `|` agrega, `^` saca.
- [ ] **Programación dinámica**: `dp[(mascara, ultimo)]` = costo mínimo de visitar ese conjunto terminando en `ultimo`; memoización de subconjuntos, `O(n² · 2ⁿ)`.
- [ ] La reconstrucción del recorrido se hace con `padre`: caminar hacia atrás siguiendo quién fue el previo, igual que en BFS/Dijkstra.
- [ ] **Heurística del vecino más cercano**: paso a paso, al barrio no visitado más cercano; `O(n²)`, sin garantía de óptimo (211 vs 194 en el ejemplo).
- [ ] El TSP exacto es **NP-difícil**: no se conoce algoritmo exacto eficiente; en grafos grandes se negocia exactitud por viabilidad.

## 6. Ejercicios

1. **Contar la explosión.** Con `math.factorial`, imprimí la cantidad de permutaciones que evalúa la fuerza bruta para 5, 8, 10 y 12 barrios (recordá que el depósito está fijo: son `(n-1)!`). Después calculá cuántas veces más rutas son 12 que 10. Es el sentimiento de "explosión" en un número.

2. **El óptimo de 8 barrios.** Con `nodo_8` (peso máximo 100, semilla 1) y `recorrido_viajante_opc1`, mostrá el costo y el recorrido óptimos. Verificá que la ruta empieza y termina en `0` y que no repite ningún vértice intermedio. ¿Cuántas medidas distintas de costo hubo que probar? (pista: `math.factorial` de nuevo, con `7` barrios intermedios).

3. **La DP confirma.** Con `recorrido_viajante_dp` sobre el mismo `nodo_8`, mostrá el costo y el recorrido. Compará el costo con el del ejercicio 2 y confirmá con `==` que son el mismo número. Después mostrá que el recorrido de la DP tiene la misma cantidad de pasos que el de la fuerza bruta (los dos cierran el circuito). *Pista:* los recorridos pueden diferir (hay empates), pero las longitudes de las tuplas y el costo tienen que coincidir.

4. **El salto de tamaño.** Con `nodo_10 = generar_grafo_completo(10, peso_maximo=100, semilla=1)`, cronometrá `recorrido_viajante_opc1` y `recorrido_viajante_dp` con `time.perf_counter` (como en el capítulo 24), mostrá que los costos coinciden, e imprimí la longitud de cada recorrido (las dos tienen que dar 11: el depósito, 9 intermedios y el regreso). ¿Qué relación de tiempos ves entre una y otra? *(Spoiler: en la máquina del libro, la DP le gana a la fuerza bruta por más de dos dígitos.)*

5. **El precio de la heurística.** Con `nodo_8` y `recorrido_viajante_aprox`, mostrá cuánto cuesta el recorrido aproximado y compáralo contra el óptimo de 194 de la sección 2. Calculá el porcentaje extra que pagás por la heurística: `(costo_aprox / costo_optimo - 1) * 100`. ¿Es un precio razonable por lo que ahorrás en tiempo?

6. **El mapa real.** Con `generar_grafo_completo(300, peso_maximo=100, semilla=1)` y `recorrido_viajante_aprox`, mostrá el costo del recorrido de 300 barrios y verificá que toca los 300 vértices exactamente una vez: compará `len(set(ruta))` contra `len(ruta) - 1` (el `-1` es porque el depósito aparece dos veces, al principio y al cierre). ¿Cuántos pasos tiene el circuito completo?

```python
# Soluciones (no las mires antes de intentarlo)

# 1. Contar la explosión
import math

print([(barrios, math.factorial(barrios - 1)) for barrios in (5, 8, 10, 12)])
# [(5, 24), (8, 5040), (10, 362880), (12, 39916800)]

print(math.factorial(11) / math.factorial(9))       # 110.0
```

*12 barrios piden 39 916 800 rutas — 110 veces más que 10. Multiplicás el problema por 110 y ni siquiera estás duplicando los barrios: los sumaste de a dos.*

```python
# 2. El óptimo de 8 barrios
nodo_8 = generar_grafo_completo(8, peso_maximo=100, semilla=1)
costo, ruta = recorrido_viajante_opc1(nodo_8, 0)
print(costo)                 # 194
print(ruta)                  # (0, 4, 6, 1, 7, 3, 2, 5, 0)
print(ruta[0] == 0 and ruta[-1] == 0)   # True
print(len(ruta) == 9)        # True
```

*La ruta arranca y cierra en el depósito `0` y tiene 9 posiciones: el depósito, 7 barrios intermedios y el regreso. Y para encontrarla hubo que probar `7! = 5040` permutaciones — el `math.factorial(7)` del ejercicio 1.*

```python
# 3. La DP confirma
costo_dp, ruta_dp = recorrido_viajante_dp(nodo_8, 0)
print(costo_dp)              # 194
print(costo == costo_dp)     # True
print(len(ruta), len(ruta_dp))      # 9 9
```

*El costo coincide con el de la fuerza bruta y el circuito también tiene 9 pasos. Los recorridos pueden ser distintos (hay empates, como en la sección 3), pero el precio del viaje es innegociable: 194.*

```python
# 4. El salto de tamaño
import time

nodo_10 = generar_grafo_completo(10, peso_maximo=100, semilla=1)

inicio = time.perf_counter()
costo_10, ruta_1 = recorrido_viajante_opc1(nodo_10, 0)
t_opc1 = time.perf_counter() - inicio

inicio = time.perf_counter()
costo_10_dp, ruta_2 = recorrido_viajante_dp(nodo_10, 0)
t_dp = time.perf_counter() - inicio

print(costo_10 == costo_10_dp)          # True
print(costo_10)                         # 141
print(len(ruta_1), len(ruta_2))         # 11 11
print(f"opc1: {t_opc1:.4f}s   dp: {t_dp:.4f}s   ratio: {t_opc1 / t_dp:.0f}x")
```

*En la máquina donde corre el libro, la fuerza bruta tardó unos 1.8 segundos y la DP unos 10 milisegundos: el ratio real lo vas a ver en tu consola, el orden de magnitud no va a mentir. Para 10 barrios la DP es más de cien veces más rápida — y esa brecha crece con cada barrio que agregás.*

```python
# 5. El precio de la heurística
costo_aprox, ruta_aprox = recorrido_viajante_aprox(nodo_8, 0)
print(costo_aprox)                            # 211
print(round((costo_aprox / costo - 1) * 100, 1))     # 8.8
```

*La heurística te cobra 8.8% más que el óptimo — 17 unidades de peso de más. Otros grafos pagan más caro (o más barato), pero así de simple es la negociación: un 9% extra por pasar de "planteo inutilizable" a "resolverlo en milisegundos".*

```python
# 6. El mapa real
nodo_300 = generar_grafo_completo(300, peso_maximo=100, semilla=1)
distancia_300, ruta_300 = recorrido_viajante_aprox(nodo_300, 0)
print(distancia_300)                       # 852
print(len(ruta_300))                       # 301
print(len(set(ruta_300)))                  # 300
print(len(ruta_300) - 1 == len(set(ruta_300)))    # True
```

*El circuito tiene 301 pasos (300 barrios + el regreso) y visita cada barrio exactamente una vez: el conjunto de la ruta tiene 300 vértices distintos, uno menos que la tupla porque el depósito `0` aparece dos veces, en el arranque y en el cierre. Ese `True` es tu garantía de que la heurística no repitió ni se salteó ningún barrio — en 300, sin permutaciones.*
```