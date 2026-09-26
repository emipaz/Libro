# Capítulo 19 — Complejidad algorítmica y la notación O grande

En el capítulo 5 te di un adelanto que quedó colgado: `pop(0)` e `insert(0, elemento)` son **lentos**, y te prometí que este capítulo te explicaría por qué. En el capítulo 10 pasó algo parecido, y más escandaloso: el fibonacci recursivo hizo **2.692.537 llamadas** para `n = 30`, y el texto prometió que esa intuición de costos "se formaliza más adelante". Ese "más adelante" es ahora.

Este capítulo te da una **lente** para mirar cualquier programa y responder una pregunta que antecede a todo el resto: *¿qué le pasa a este código cuando los datos crecen?* Ese programa que "se cuelga" con el archivo grande, esa búsqueda que demora cuando hay diez mil registros, esa función que anda perfecto en tus pruebas y explota en producción — en casi todos los casos el diagnóstico es el mismo: el *costo* de la solución crece mal. Vas a aprender la herramienta estándar para hablar de ese costo con precisión: la **notación O grande** (la *Big-O* del mundo anglo, donde la "O" viene de *order*, "orden"), y con ella vas a poder anticipar si una solución se va a portar bien antes de escribir una sola línea más.

---

## 1. El costo que no se ve a simple vista

Imaginá la escena más frustrante del oficio: dos programas hacen **exactamente la misma tarea**, y uno demora una eternidad mientras el otro vuelve al toque. No es una cuestión de suerte ni de computadora: la diferencia es matemática, y se puede medir *antes* de ejecutar nada.

La tentación natural es cronometrar. Corrés los dos programas, medís los segundos con un cronómetro y comparás. Ese reflejo es comprensible, pero tiene un problema de fondo que ya se te adelantó en el capítulo 2: el tiempo real depende de la máquina, del sistema operativo, de qué más esté haciendo tu computadora en ese momento. Median "0,3 segundos" en tu equipo hoy, y mañana sobre el mismo código la CPU está ocupada y son 2 segundos. El cronómetro mide el *momento*; vos necesitás medir el **algoritmo**.

> **Dato clave:** lo que importa no es cuántos *segundos* tarda un algoritmo, sino **cómo cambia ese costo cuando el tamaño de la entrada cambia**. Si al doblar los datos el tiempo se duplica, es una cosa; si se cuadruplica, es otra muy distinta. Esa relación es matemática y no depende de ninguna máquina.

## 2. Medir en pasos: la suma de Gauss

Para soltar el cronómetro, vas a contar **pasos**: unidades de trabajo del algoritmo que no dependen de la máquina. En el capítulo 4 sumaste los enteros del 1 al 100 con un bucle y te quedó la famosa suma de Gauss: 5050. Ahora vas a comparar *dos algoritmos* que resuelven ese mismo problema — y a llevar la cuenta de cuántos pasos gasta cada uno.

La historia detrás de ese nombre es tan buena que conviene contarla — con el aviso de que, como toda anécdota, tiene variantes y verificación dudosa, pero la sustancia es esta. Carl Friedrich Gauss tenía unos siete años en el aula de su escuela cuando el maestro pidió a la clase sumar todos los enteros del 1 al 100. El resto de los chicos se puso a sumar en orden, uno por vez. Gauss observó un momento y se dio cuenta de que los números se podían **aparear**: 1 + 100 = 101, 2 + 99 = 101, 3 + 98 = 101... hasta el medio hay **cincuenta parejas**, todas con el mismo total. Entonces 50 × 101 = 5050. Escribió la respuesta casi de inmediato, y fue el único de la clase con el número correcto.

Fijate qué tipo de genio es ese: no **sumó más rápido**, sumó *distinto*. El resto de la clase hizo el trabajo del bucle — recorrer un número por vez, con costo proporcional a cuántos números hay. Gauss encontró la cuenta cerrada: un puñado de operaciones, las mismas aunque el maestro hubiera pedido llegar al millón. Es exactamente el enfrentamiento que vas a medir a continuación: el bucle contra la fórmula, los pasos contra la cuenta.

```python
pasos = 0

def sumar_bucle(m, n):
    """Suma los enteros entre m y n recorriendo cada número."""
    global pasos
    resultado = 0
    for num in range(m, n + 1):
        resultado += num
        pasos += 1              # un paso por cada número visitado
    return resultado

def sumar_gauss(m, n):
    """Suma los enteros entre m y n con la fórmula cerrada de Gauss."""
    return (m + n) * (n - m + 1) // 2
```

El `global pasos` no es una técnica para copiar en código serio (lo viste bien señalado en el capítulo 8 y lo volviste a usar como contador de laboratorio en el capítulo 10): es la regla de medir de esta sección. Corré las dos versiones:

```python
print(sumar_bucle(0, 100))      # 5050
print(f"pasos = {pasos}")       # pasos = 101
print(sumar_gauss(0, 100))      # 5050

pasos = 0
sumar_bucle(0, 1_000)
print(f"pasos = {pasos}")       # pasos = 1001
```

El bucle gasta **101 pasos** para cien números y **1001** para mil: un paso por cada número, y listo. Gauss, en cambio, devuelve exactamente el mismo `5050` y *no toca el contador*: su fórmula `(m + n) * (n - m + 1) // 2` hace siempre la misma cantidad de trabajo, sin importar si sumás diez números o un millón. Fijate el detalle: el bucle es *correcto*, devuelve lo mismo que Gauss — el bug no está en el resultado, está en el **costo**. Ahí es donde la suma de Gauss pasa de ser una curiosidad histórica a una decisión de ingeniería.

Y por si el contador te pareciera demasiado abstracto, mirá el mismo enfrentamiento con un cronómetro en serio. Este decorador es la promesa que te quedó del capítulo 8 — ahí se anunció que con los ladrillos de las closures y el `@` ibas a "montar contadores de llamadas y medición de tiempos", y es exactamente lo que hace `cronometrar`:

```python
from time import perf_counter

def cronometrar(funcion):
    """Decorador: envuelve la función y mide su tiempo real."""
    def envoltorio(*args, **kwargs):
        inicio = perf_counter()
        resultado = funcion(*args, **kwargs)
        print(f"{funcion.__name__}: {(perf_counter() - inicio):.4f} s")
        return resultado
    return envoltorio

@cronometrar
def medir_bucle():
    return sumar_bucle(0, 1_000_000)

@cronometrar
def medir_gauss():
    return sumar_gauss(0, 1_000_000)

print(medir_bucle())        # medir_bucle: ≈ 0.05 a 0.10 s
                            # 500000500000
print(medir_gauss())        # medir_gauss: ≈ 0.00 s
                            # 500000500000
```

Un millón de números: el bucle tarda unas centésimas y la fórmula ni se ve en el cronómetro. Pero ojo con la lección real: esos `≈` van a cambiar en tu máquina, en mi máquina y mañana en la misma máquina. Lo único que *no* cambia es lo que contaste a mano: el bucle hace `n` pasos, Gauss hace siempre los mismos.

Y si querés ahorrarte el decorador, la biblioteca estándar ya trae `timeit`: repite el fragmento muchísimas veces y te devuelve el promedio, para que el ruido de la máquina se diluya. Es la versión estándar de tu `cronometrar`, con repeticiones automáticas — pero fijate qué es lo que no te da: el reporte sigue en segundos, y los segundos vuelven a depender de la máquina. Para la O grande, seguís necesitando los pasos.

## 3. Notación O grande: cuando n tiende al infinito

Tomá `n` como el **tamaño de la entrada**: cuántos números hay que sumar, cuántos elementos tiene la lista, cuántas filas tiene la tabla. La pregunta central de este capítulo se escribe así: *¿cómo varía la cantidad de pasos cuando `n` crece?*

La **notación asintótica** (del griego "que no se toca": estudia el comportamiento en el límite) responde esa pregunta con una frase compacta. Para la suma con bucle decimos que la complejidad es **O(n)** ("O de n"), y lo leemos así: en el peor de los casos, el número de operaciones crece en proporción directa al tamaño de la entrada. Para Gauss decimos **O(1)** ("O de uno"): la cantidad de operaciones es constante, no crece con `n`.

> **Importante:** la O grande se define sobre el **peor caso** — la garantía de que el algoritmo jamás tarda más que ese techo. Existen dos hermanas menos usadas: la **Ω** (omega grande) que describe el mejor caso, y la **Θ** (theta grande) que describe el caso promedio con precisión. En la práctica de la industria, cuando alguien dice "la complejidad de esto es tal", casi siempre habla de la O grande, porque es el compromiso que te conviene conocer: el que no te va a dejar tirado.

La belleza de esta notación es que **ignora todo lo accesorio**: las constantes (`n / 2` es O(n), `3n` es O(n), `2n + 5` es O(n)) y los términos menores (`n² + n` es O(n²)). Cuando `n` es un millón, sumar o restar un "cinco" no le importa a nadie; lo único que escala es el término que manda.

## 4. Las familias de complejidad

Cada orden de complejidad tiene su personalidad y su ejemplo arquetípico. Las familias van en orden de crecimiento, del más económico al más caro.

### O(1): el acceso directo

La complejidad constante no depende del tamaño. Acceder a una lista por índice (`datos[5]`) es O(1): la lista es un bloque contiguo y Python sabe dónde está el lugar 5 sin mirar nada más. La fórmula de Gauss es O(1), y más adelante en este mismo capítulo vas a ver un caso que duele: contar cuántos coches y motos hay con una cuenta, no con un recorrido.

### O(n): la búsqueda lineal

La complejidad lineal recorre una vez cada elemento. Es la búsqueda que escribís a mano y que el operador `in` repite por debajo cuando la lista es una lista:

```python
def buscar_lineal(datos, dato):
    """Recorre de a uno hasta encontrar el dato."""
    for elemento in datos:
        if elemento == dato:
            return True
    return False

numeros = [14, 26, 39, 25, 11, 18, 42, 31]
print(buscar_lineal(numeros, 31))      # True
print(buscar_lineal(numeros, 99))      # False
print(31 in numeros)                   # True
```

Si el dato está al final (o no está), hay que mirar los `n` elementos: **O(n)**. Ese es el caso del capítulo 5: `pop(0)` e `insert(0, x)` también tienen que desplazar todos los elementos que siguen — por más que "solo saques el primero", Python mueve a los otros `n` de lugar.

Pero el mismo `in` sobre un `set` es otra historia. Los conjuntos se guardan con una **tabla hash**, una estructura que salta directo a la casilla donde debería estar el elemento:

```python
unicos = set(numeros)
print(31 in unicos)      # True
```

Preguntarle a un `set` es **O(1) por consulta**. La trampa honesta: *construir* el conjunto con `set(numeros)` recorrió la lista completa una vez (O(n)). El negocio cierra cuando hacés muchas consultas: pagás O(n) una sola vez y después cada pregunta cuesta constante. Esto es una decisión de estructura de datos, y es el corazón de la sección 6.

### O(log n): partir por la mitad

Imaginá que la lista está **ordenada**. Ya no hace falta mirar de a uno: mirás el medio; si el dato es menor, descartás la mitad derecha entera; si es mayor, descartás la izquierda; y repetís. Esa es la **búsqueda binaria**:

```python
def busqueda_binaria(lista, elemento):
    """Busca en una lista ordenada descartando la mitad que no sirve."""
    inicio, final = 0, len(lista) - 1
    while inicio <= final:
        medio = (inicio + final) // 2
        if lista[medio] == elemento:
            return medio
        if lista[medio] < elemento:
            inicio = medio + 1
        else:
            final = medio - 1
    return -1

multiplicos = list(range(0, 700_001, 7))
print(len(multiplicos))                        # 100001
print(busqueda_binaria(multiplicos, 70))       # 10
print(busqueda_binaria(multiplicos, 700_000))  # 100000
print(busqueda_binaria(multiplicos, 5))        # -1
```

En cada vuelta del bucle descartás la mitad de lo que quedaba. Una lista de 100.001 elementos se reduce a la mitad unas **17 veces** hasta quedar en un solo elemento (porque 2¹⁷ ≈ 131.072, apenas por encima de 100.001). Esa es la **O(log n)**: el logaritmo. ¿Y de qué base? No importa — y no es un detalle vago, es matemática: cambiar de base solo multiplica por una constante, y las constantes se ignoran. Lo que sí importa es la forma: la logarítmica crece *despacísimo* (cada vez que multiplicás la entrada por 10, sumás apenas unas pocas operaciones), y es la recompensa de los datos **ordenados**.

### O(n log n): ordenar

Si buscar bien requiere datos ordenados, ¿cuánto cuesta ordenarlos? Los buenos algoritmos de ordenamiento — `sorted` y `list.sort()` de Python, y los *mergesort* y *heapsort* clásicos — son **O(n log n)**: más caro que recorrer (O(n)), pero infinitamente más barato que los pares "todos contra todos" (O(n²)). Miralo crecer:

```python
import random
from time import perf_counter

for n in (10_000, 100_000, 1_000_000):
    datos = [random.randint(0, 10 ** 6) for _ in range(n)]
    inicio = perf_counter()
    datos.sort()
    print(n, f"{perf_counter() - inicio:.3f} s")
```

En una corrida típica da algo así (los valores exactos cambian, la forma no):

- `10000` → `0.001 s`
- `100000` → `0.018 s`
- `1000000` → `0.285 s`

Diez veces más datos, y el tiempo no se multiplica por diez: se multiplica apenas por menos de dos. Ese aplanamiento es la firma de la curva `n log n`.

### O(n²): los pares de a dos

Ahora la curva empieza a doler. La complejidad cuadrática aparece cuando hay que comparar **cada elemento con cada elemento**. El caso arquetípico son los bucles anidados: por cada fila de una matriz `n × n` recorrés las `n` columnas, y eso da `n × n = n²` accesos.

```python
def suma_matriz(matriz):
    """Suma todos los valores de una matriz cuadrada."""
    total = 0
    for fila in matriz:
        for valor in fila:
            total += valor
    return total

grilla = [[1, 2, 3],
          [4, 5, 6],
          [7, 8, 9]]
print(suma_matriz(grilla))      # 45
```

Con una grilla de 3×3 son nueve accesos y todo es instantáneo. Duplicá el lado a 6×6 y son 36: cuadruplicaste el trabajo. Con 1.000×1.000 ya son un *millón* de pasos. La cuadrática es el orden típico de la solución "ingenua" que se escribe primero, y el que la sección de ejercicios te va a enseñar a desarmar.

El ejemplo del problema de los coches y las motos en una autopista lo muestra en carne viva: sabés el total de vehículos y el total de ruedas, y querés saber cuántos son autos (4 ruedas) y cuántas motos (2). La versión inocente prueba todas las combinaciones posibles de autos y motos:

```text
for coches in range(v + 1):
    for motos in range(v + 1):
        if coches + motos == v and coches * 4 + motos * 2 == r:
            return coches, motos
```

Si pasan 8.000 vehículos, son 8.001 × 8.001 ≈ **64 millones** de combinaciones probadas. Pero hay una simplificación: una vez que fijás la cantidad de coches, las motos quedan determinadas (total − coches). Un solo bucle alcanza:

```python
def contar_vehiculos(vehiculos, ruedas):
    """Recorre las posibles cantidades de coches; las motos se deducen."""
    for coches in range(vehiculos + 1):
        motos = vehiculos - coches
        if coches * 4 + motos * 2 == ruedas:
            return coches, motos

print(contar_vehiculos(8000, 22000))      # (3000, 5000)
```

Mismo resultado, sin las 64 millones de pruebas: de O(n²) descendés a **O(n)**. Y todavía queda un escalón más abajo, que va a ser tu ejercicio final: con dos ecuaciones y dos incógnitas, no hace falta *ningún* bucle.

### O(2ⁿ): el abanico que se duplica

Llegó el turno de lo peor. La complejidad exponencial duplica el trabajo en cada elemento adicional — y la viste nacer en el capítulo 10. El fibonacci recursivo pide `fib(n)` que llama *dos veces* a `fib(n - 1)` y `fib(n - 2)`, y cada una de esas llamadas vuelve a llamar dos veces, y así por abajo: un abanico que se duplica en cada nivel.

```python
llamadas = 0

def fib(n):
    """Fibonacci recursivo, contando cada llamada que se hace."""
    global llamadas
    llamadas += 1
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)

print(fib(30))           # 832040
print(llamadas)          # 2692537
```

Las cuentas ya las hiciste en el capítulo 10 y no cambiaron: para llegar a `fib(30)` (832.040), el abanico generó **2.692.537 llamadas**. Eso es O(2ⁿ) en su estado puro, y es la razón por la que el capítulo 10 lo marcó como "caso donde la recursión duele". A `n = 40`, `2ⁿ` ya son más de un *billón* de operaciones: impracticable, punto.

Ahora viene la parte interesante, la que el capítulo 10 te dejó prometida y el capítulo 11 te mostró con `@lru_cache` (recordá: 36 *misses* contra los millones de antes). ¿Y si a mano guardás cada resultado ya calculado? Esa idea de **guardar resultados intermedios para no repetir trabajo** tiene nombre — es la **memoización**, la base de la *programación dinámica* — y acá podés verla hacer su magia con las cuentas en la mano:

```python
def fib_memo(n, memo=None):
    """Fibonacci recursivo que recuerda lo que ya calculó."""
    global llamadas
    llamadas += 1
    if n < 2:
        return n
    if memo is None:
        memo = {}
    if n in memo:
        return memo[n]
    memo[n] = fib_memo(n - 1, memo) + fib_memo(n - 2, memo)
    return memo[n]

llamadas = 0
print(fib_memo(30))      # 832040
print(llamadas)          # 59
```

El mismo resultado, `832040`, pero las llamadas pasan de 2.692.537 a **59** — apenas unas dos por cada nivel de `n`. La clave es el diccionario `memo`: cada `fib(k)` se calcula una sola vez y las demás consultas contestan de memoria. Es exactamente el `@lru_cache` del capítulo 11, pero ahora sabés *qué es* lo que ese decorador guarda para vos. Esto no es un truco del capítulo 10 que "no hay que usar": es una técnica real, y el contador de llamadas te muestra dónde está su potencia (evitar recalcular) y su límite (no baja la imagen *enumerada* de un problema: si algo imprime 2ⁿ resultados, nadie lo salva).

La memoización no convierte mágicamente a la recursión en la mejor opción: **cuando existe una versión con bucle, el bucle siempre corre más rápido**. Lo viste nacer en el capítulo 10 — `fibonacci_bucle` con `a, b = 0, 1` y el `a, b = b, a + b` en un `for` — que resuelve `n = 30` en **29 vueltas** (el contador que mediste ahí), sin diccionario, sin pila y sin una sola llamada extra: O(n) en tiempo y O(1) en espacio. La recursión memoizada llega a la misma O(n), pero cada llamada paga su costo de pila y cada consulta paga su búsqueda en el `memo`. La regla práctica: la recursión es para *leer* problemas que se parten en subproblemas idénticos (árboles, carpetas, la estructura de datos del capítulo 10); la memoización rescata a la recursión cuando es la forma natural de escribirlo, pero no la convierte en el algoritmo más rápido. Si el problema admite un bucle directo, el bucle manda.

### El profiler: cuando no sabés dónde revolver

El contador de llamadas es ideal para aprender y para probar hipótesis: dos líneas y sabés exactamente cuánto trabajo gastó cada función. Pero en un programa real, con decenas de funciones y bibliotecas, no vas a ir instrumentando todo a mano. Ahí entra la herramienta estándar: el **analizador de perfiles** (el *profiler* de la jerga), un programa que ejecuta el tuyo por debajo mientras registra, función por función, cuántas veces fue llamada y cuánto tiempo acumuló. El que viene con Python se llama `cProfile`.

```python
import cProfile
import pstats

with cProfile.Profile() as perfil:
    fib(25)

pstats.Stats(perfil).strip_dirs().sort_stats("cumulative").print_stats(4)
# 242787 function calls (3 primitive calls) in ≈ 0.05 a 0.10 s
#
# Ordered by: cumulative time
#
# ncalls   tottime  percall  cumtime  percall  filename:lineno(function)
# 242785/1 ≈0.05–0.10 0.000  ≈0.05–0.10  0.000  ... (fib)
# 1        0.000    0.000    0.000    0.000    cProfile.py:117(__exit__)
# 1        0.000    0.000    0.000    0.000    {method 'disable' ...}
```

Las cuentas de esta corrida: en la columna `ncalls` leés `242785/1`, y el `/1` recuerda que solo la primera fue la llamada original — las demás fueron recursión, exactamente como en tu contador del capítulo 10. Las dos filas chicas son el propio profiler midiéndose; la fila que manda es la de `fib`. Fijate qué pregunta responde el profiler y cuál no: te dice *dónde* se fue el tiempo (todo, en una sola función), no *por qué* (eso lo responde la O grande: cada `n` duplica el abanico). Y depende de la corrida concreta: con `n = 25` son 242.785 llamadas; rehacé el mismo perfil sobre `fib_memo` y el número baja a **59**. El profiler y la notación son aliados: la O grande te dice qué debería estar mal, `cProfile` te confirma en la ejecución dónde está.

## 5. La tabla del crecimiento y las reglas de oro

Poné las familias juntas en una tabla. Tomá una cifra redonda: asumí que cada operación cuesta un nanosegundo, y mirá cuántas operaciones necesita cada complejidad según el tamaño de la entrada:

| Complejidad | `n = 10` | `n = 100` | `n = 1.000` | `n = 1.000.000` |
|---|---|---|---|---|
| **O(1)** | 1 | 1 | 1 | 1 |
| **O(log n)** | ~3 | ~7 | ~10 | ~20 |
| **O(n)** | 10 | 100 | 1.000 | 1.000.000 |
| **O(n log n)** | ~33 | ~664 | ~10.000 | ~20 millones |
| **O(n²)** | 100 | 10.000 | 1.000.000 | 1 billón |
| **O(2ⁿ)** | 1.024 | ya es astronómico | impracticable | impracticable |

La primera columna es esperanza, la última es la que decide. Con un millón de datos, la O(n²) pide un billón de pasos; la exponencial directamente no entra en ninguna tabla. Por eso el orden de costos es una escala que conviene memorizar de menor a mayor:

**O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ)**

Y para calcular la notación de cualquier algoritmo, tres reglas de oro:

1. **Ignorá las constantes multiplicativas**: `2n` es O(n), `10` es O(1).
2. **Quedate con el término que más crece**: `1 + 2n` es O(n); `n² + n` es O(n²).
3. **En bucles anidados, multiplicá**: un bucle de tamaño `n` dentro de otro bucle de tamaño `n` da `n × n = O(n²)`. Dos bucles en serie, en cambio, suman: `O(n) + O(n)` sigue siendo O(n).

> **Buenas prácticas:** la notación describe la *forma* de la curva, no el lugar exacto. Decir "2n" u "O(n)" es la misma afirmación; decir "O(n²) contra O(n log n)" es una diferencia que, con datos grandes, decide entre segundos y horas.

## 6. La estructura de datos como decisión

La gran moraleja práctica del capítulo es que la misma operación cuesta distinto según dónde vivan tus datos. A eso se refería el capítulo 5 cuando te adelantó lo lento de `pop(0)` e `insert(0)` y te presentó a `deque`. Ahora podés explicarlo: para sacar el primer elemento, la lista desplaza los `n` restantes — O(n). La cola `deque` (*double-ended queue*, "cola de doble extremo") mantiene punteros en ambos extremos y trabaja en O(1):

```python
from collections import deque

cola = deque([1, 2, 3])
cola.append(4)          # al final
cola.appendleft(0)      # al principio
print(list(cola))       # [0, 1, 2, 3, 4]
print(cola.popleft())   # 0
print(cola.pop())       # 4
```

Ahí se cerró el adelanto del capítulo 5: no es que "sacar de adelante sea un poco molesto", es que pasa de O(n) a O(1). Y el mismo ojo que mira listas contra colas mira diccionarios y conjuntos: el `set` te da O(1) en pertenencia porque es una tabla hash, el `dict` es la misma esencia con un valor pegada a cada clave — por eso las claves de diccionario tienen que ser *inmutables y con hash*, como el capítulo 5 te explicó.

La tabla de referencia canónica de estas operaciones está publicada en el wiki de Python, bajo el nombre *TimeComplexity* (podés buscarla así en la web): ahí figuran, fila por fila, las complejidades de `list`, `dict` y `set`. Un extracto típico, con las que más vas a usar:

| Operación | `list` | `set` / `dict` |
|---|---|---|
| Acceder por índice / por clave | O(1) | O(1) |
| `in` (pertenencia) | O(n) | O(1) |
| Agregar al final (`append`) | O(1) amortizado | O(1) |
| Sacar el primero | O(n) | — |
| Ordenar | O(n log n) | — |

> **Dato clave:** "**amortizado**" es un término que conviene no dejar pasar: *append* es O(1) la mayoría de las veces, pero ocasionalmente la lista queda corta y tiene que mudarse a un bloque de memoria más grande (esa copia puntual cuesta O(n)). El costo *promedio a lo largo de muchas operaciones* sigue siendo O(1) — por eso tu lista de un millón de `append` no te tira un segundo por cada uno.

Y una última familia, la que cierra el tema de los recursos: además del **tiempo**, un algoritmo consume **espacio** (memoria), y también se mide con O grande. Ya tenés el caso memorable del capítulo 10: la lista con un millón de números pesa 8,4 MB, el generador que los produce, 200 bytes — eso es "O(n) en espacio contra O(1)". El capítulo se concentró en el tiempo por ser el recurso que primero grita, pero la pregunta "¿cuánta memoria me va a pedir esto?" es la misma lente, aplicada a otro recurso.

> **Importante:** el lente que ejercitaste en este capítulo no es solo para tareas de laboratorio. Es el primero de los **cuatro lentes** con los que se revisa código antes de dar por bueno un cambio — *errores, eficiencia, escalabilidad y seguridad* — y vas a volver a él en el anexo del final del libro. Todo lo que sigue ya lo vas a leer con esta pregunta encima: ¿esto crece bien?

Este es además el puente hacia la Parte VIII: las **estructuras clásicas** que vas a construir con tus propias manos, en una escalera natural desde lo que ya usás hoy sin pensar. Arranca con los **arrays** — el primo honesto de la lista nativa de Python, el bloque contiguo de memoria que el capítulo 5 te dejó entrever y que el capítulo 10 pesó en 8,4 MB — y cuando tus arrays sean números y haya muchos, vas a conocer a **NumPy**, la biblioteca que opera sobre esos bloques sin recorrerlos de a uno. Después vienen las **listas enlazadas**, que guardan cada dato repartido en la memoria y se pasan el testigo con un puntero al siguiente, y sus primas **doblemente enlazadas**, la base del `deque` que ya usás. Le siguen los **árboles**, donde el nodo se ramifica y ofrece de verdad las búsquedas O(log n) que hoy viste en la búsqueda binaria; los **grafos**, la estructura de los mapas, las redes y las conexiones, con sus O(n²) a cuestas; y por último la **tabla hash**, el corazón invisible del `dict` y del `set`, que vas a terminar de entender de adentro hacia afuera — hasta preparar que *tus propias clases* sean **hashables** y puedan vivir como claves de diccionario.

Y vas a llegar a cada estructura con la pregunta del capítulo ya incorporada: no "¿funciona?" sino "¿cuánto cuesta?". Cada una entra al guión con su precio del día — una lista enlazada te mete constantes y operaciones O(n) en la mitad, un árbol binario ordenado promete las búsquedas O(log n) de la búsqueda binaria, un grafo arrastra sus O(n²) a la mesa, y un dato bien hashable te abre el O(1) que hoy solo tienen los nativos. Lo que aprendiste en este capítulo es el lente con el que vas a juzgarlas a todas.

---

## 7. Resumen y conceptos clave

Este capítulo te dio una forma nueva de mirar el código: en vez de preguntar si un algoritmo *resuelve* el problema, preguntá **cómo se comporta cuando los datos crecen**. Arrancaste sacando el cronómetro de la mesa — porque el tiempo real depende de la máquina y del momento — y lo reemplazaste por el conteo de **pasos**, unidades de trabajo que no mienten. Ese conteo te dio la primera gran victoria con la suma de Gauss: el bucle es O(n), la fórmula es O(1), y los dos devuelven el mismo 5050. Sobre esa base definiste la **notación O grande**, la garantía del peor caso que ignora constantes y términos menores, y recorriste sus familias: desde el acceso directo O(1) y la búsqueda binaria O(log n), pasando por el recorrido O(n) y el ordenamiento O(n log n), hasta los bucles anidados O(n²) y el abanico exponencial O(2ⁿ) que el fibonacci del capítulo 10 formalizó de una vez por todas — con la memoización mostrándote, en números, cómo bajar de 2,6 millones de llamadas a 59. Cerraron las tres reglas de oro para calcular complejidad y la aplicación más rentable de todas: la operación que elegís (o la estructura donde la hacés) cambia el costo por completo — el `deque` de O(n) a O(1), el `set` de O(n) a O(1) por consulta, y hasta un problema de 64 millones de combinaciones que se resuelve con una cuenta.

Repasá el checklist antes de seguir:

- [ ] El **tiempo real** depende de la máquina; se mide en **pasos**, unidades de trabajo que no dependen del hardware.
- [ ] **O(1)**: operaciones constantes (acceso por índice, fórmula de Gauss, `in` en un `set`).
- [ ] **O(n)**: recorrer una vez (búsqueda lineal, `in` en una lista, `pop(0)`/`insert(0)`).
- [ ] **O(log n)**: partir por la mitad (búsqueda binaria); la base del logaritmo no importa.
- [ ] **O(n log n)**: ordenar con `sorted`/`sort`; crece más lento que el tamaño por un poco.
- [ ] **O(n²)**: bucles anidados y pares de elementos; con un millón de datos ya es un billón de pasos.
- [ ] **O(2ⁿ)**: el fibonacci recursivo (2.692.537 llamadas para `n = 30`); la **memoización** lo baja a 59 (O(n)) guardando resultados intermedios.
- [ ] La O grande describe el **peor caso**; Ω describe el mejor caso y Θ el promedio.
- [ ] **Tres reglas**: ignorar constantes, quedarte con el término mayor, multiplicar en bucles anidados.
- [ ] La **estructura de datos es parte del algoritmo**: `deque` O(1) contra lista O(n), `set`/`dict` O(1) contra lista O(n); `append` es O(1) amortizado.
- [ ] El algoritmo también cuesta **espacio** (memoria): mide ambas O grande.

---

## 8. Ejercicios

1. **Identificá las complejidades.** Sin correr, decí la notación O grande de cada bloque (donde `n` es una variable ya definida):

```text
# a)
for i in range(n):
    print(i)

# b)
for i in range(n):
    for j in range(n):
        print(i, j)

# c)
i = 0
while i < n:
    print(i)
    i += 2

# d)
i = 1
while i < n:
    print(i)
    i *= 2
```

2. **`in`: lista contra set.** Armá una lista con los enteros del `0` al `999.999`, preguntá si contiene al `999.999`, y repetí la misma pregunta contra el `set` equivalente. Cronometrá las dos consultas y explicá la diferencia a la luz de la sección 4.

3. **Búsqueda binaria en grande.** Escribí tu propia `busqueda_binaria` y buscá el `999.999` en una lista ordenada de un millón de elementos. Verificá que el resultado sea coherente comparándolo con `in`, y comprobá qué devuelve tu función cuando el elemento *no* está.

4. **Coches y motos, sin ningún bucle.** En la sección 4 viste la versión O(n). Con dos ecuaciones y dos incógnitas (`coches + motos = total` y `coches · 4 + motos · 2 = ruedas`) escribí una versión **O(1)** que resuelva `(8000, 22000)` sin recorrer nada. Cuidado con el caso imposible (por ejemplo `(10, 2)` o ruedas impares): tu función debería avisar en vez de devolver un disparatado.

5. **El costo de enumerar.** Escribí la función `subconjuntos(lista)` que devuelva todas las combinaciones posibles de una lista (la del estilo que rumió el capítulo 10): cada elemento entra o no entra en cada subconjunto — dos caminos por elemento. Comprobá que `len(subconjuntos([1, 2, 3]))` da `8` (2³) y que con diez elementos da `1024` (2¹⁰). Ahora entendés por qué esa familia es O(2ⁿ): enumerar 2ⁿ resultados cuesta, como mínimo, 2ⁿ.

6. **El contador de frecuencias con `dict`.** Dado un texto, contá cuántas veces aparece cada palabra con un diccionario (`frecuencias[palabra] = frecuencias.get(palabra, 0) + 1`). ¿Cuál es la complejidad *total* de contar así, sabiendo que cada acceso a un `dict` es O(1)? Probálo con la frase `"el pan el vino y el pan de ayer"` y verificá el resultado.

7. **De O(n²) a O(n).** Un colega detecta duplicados comparando "todos contra todos" con dos bucles anidados. Reemplazá esa idea por un `set` que vaya acumulando los vistos: si el elemento ya está en el conjunto, hay duplicado. Probá con `[3, 1, 4, 1, 5]` (tiene duplicado) y `[3, 1, 4, 5, 9]` (no). Explicá por qué la versión del colega era O(n²) y la tuya O(n).

```python
# Soluciones (no las mires antes de intentarlo)

# 1. Complejidades
# a) Un solo recorrido de tamaño n -> O(n).
# b) Dos bucles anidados de tamaño n -> O(n²).
# c) Itera n/2 veces; la constante se ignora -> O(n).
# d) i se duplica en cada vuelta: recorre log2(n) veces -> O(log n).

# 2. `in` en lista vs set
import time

numeros = list(range(1_000_000))
numeros_set = set(numeros)

inicio = time.perf_counter()
print(999_999 in numeros)                 # True
print(f"lista: {time.perf_counter() - inicio:.5f} s")   # ≈ 0.011 s

inicio = time.perf_counter()
print(999_999 in numeros_set)             # True
print(f"set: {time.perf_counter() - inicio:.5f} s")      # ≈ 0.000 s

# 3. Búsqueda binaria en una lista de un millón
def binaria(lista, elemento):
    inicio, final = 0, len(lista) - 1
    while inicio <= final:
        medio = (inicio + final) // 2
        if lista[medio] == elemento:
            return medio
        if lista[medio] < elemento:
            inicio = medio + 1
        else:
            final = medio - 1
    return -1

ordenados = list(range(1_000_000))
print(binaria(ordenados, 999_999))        # 999999
print(binaria(ordenados, -7))             # -1
print(999_999 in ordenados)               # True

# 4. Coches y motos en forma cerrada (O(1))
def contar_directo(vehiculos, ruedas):
    coches = (ruedas - 2 * vehiculos) // 2
    motos = vehiculos - coches
    if ruedas % 2 != 0 or coches < 0 or motos < 0:
        return None
    return coches, motos

print(contar_directo(8000, 22000))        # (3000, 5000)
print(contar_directo(8000, 22001))        # None
print(contar_directo(10, 2))              # None

# 5. Subconjuntos: enumerar cuesta O(2ⁿ)
def subconjuntos(lista):
    if not lista:
        return [[]]
    resto = subconjuntos(lista[1:])
    return resto + [[lista[0]] + r for r in resto]

print(len(subconjuntos([1, 2, 3])))       # 8
print(len(subconjuntos(list(range(10))))) # 1024

# 6. Contador de frecuencias con dict (O(n) total)
def contar_frecuencias(texto):
    frecuencias = {}
    for palabra in texto.split():
        frecuencias[palabra] = frecuencias.get(palabra, 0) + 1
    return frecuencias

frase = "el pan el vino y el pan de ayer"
print(contar_frecuencias(frase))
# {'el': 3, 'pan': 2, 'vino': 1, 'y': 1, 'de': 1, 'ayer': 1}

# 7. Duplicados con un set (O(n))
def hay_duplicados(lista):
    vistos = set()
    for valor in lista:
        if valor in vistos:
            return True
        vistos.add(valor)
    return False

print(hay_duplicados([3, 1, 4, 1, 5]))    # True
print(hay_duplicados([3, 1, 4, 5, 9]))    # False
```