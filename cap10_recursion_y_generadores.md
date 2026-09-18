# Capítulo 10 — Recursión y generadores: funciones que se llaman a si mismas y funciones que se pausan.

Terminaste el capítulo 9 con una promesa: funciones que se expresan en sus propios términos, hasta reducir el problema a nada. Buen momento, porque este capítulo tiene dos actos.

El primer acto es la **recursión**: una función que, para resolver un problema, se llama a sí misma con una versión más chica del mismo problema. La vas a ver explicada por partes — el caso base que la detiene, la pila de llamadas que la sostiene por debajo y, porque este libro no te muestra magia, también los casos en los que conviene **no** usarla.

El segundo acto son los **generadores**: funciones que no se quedan con un solo `return`, sino que pueden *pausarse* y entregar valores de a uno. Son los que viste asomar en el capítulo 5 entre paréntesis, `(x for x in …)`, prometidos junto con `yield`. Acá los cobramos.

¿Y por qué van juntos? Porque comparten el mismo truco de fondo: una función cuya descripción vale mucho más que su propio cuerpo. Y porque hay momentos en que se mezclan tan bien que vas a escribir un generador recursivo que aplana listas anidadas en tres líneas. Cuidado con ese momento: es de los que enganchan.

---

## 1. Una función que se llama a sí misma

Imaginá dos espejos enfrentados: cada uno refleja al otro y su reflejo. La recursión es la versión programable de ese juego — con una regla de seguridad para que no se vuelva infinito.

**Recursión** es cuando una función se llama a sí misma para resolver el problema. El ejemplo clásico, el factorial — `n!` es el producto de todos los enteros desde 1 hasta `n`. Mirá la definición elegante de Python, con su docstring en el estilo Google que adoptamos en el capítulo 9:

```python
def factorial_rec(n: int) -> int:
    """Calcula n! (el factorial de n), para n >= 0.

    Args:
        n: un entero no negativo.

    Returns:
        n! = 1 * 2 * ... * n.

    Raises:
        ValueError: si n es negativo.
    """
    if n < 0:
        raise ValueError(f"factorial no está definido para {n}")

    if n == 0:
        return 1

    return n * factorial_rec(n - 1)

print(factorial_rec(5))   # 120
```

¿Lo viste? Dentro de `factorial_rec` hay una llamada a... `factorial_rec`. Pero con un argumento más chico: `n - 1`. Para saber cuánto vale `5!`, la función pregunta cuánto vale `4!`; para eso, cuánto vale `3!`; y así hasta llegar a un caso que conoce sin preguntar: `0! = 1`.

Compará con la versión que ya sabés escribir, el bucle:

```python
def factorial_bucle(n: int) -> int:
    resultado = 1
    for factor in range(2, n + 1):
        resultado *= factor
    return resultado

print(factorial_bucle(5))   # 120
```

Las dos respuestas coinciden. La recursiva no es "mejor" ni "más rápida" — es *otra manera de pensar*, y como tal tiene sus puntos fuertes y sus trampas. La trampa más importante viene primero: para que la recursión funcione, **tiene que haber alguien que la detenga**.

> **Dato clave:** la recursión no es un bucle disfrazado — es una función que *usa su propia definición* para resolver casos más chicos. Sin una pregunta que se responda sin volver a llamarse, el espejo se refleja para siempre.

---

## 2. La receta: caso base y caso recursivo

Toda función recursiva tiene exactamente dos partes, y reconocerlas te deja leerla de un vistazo:

1. **El caso base**: la situación trivial que se responde directamente, sin recursión. En `factorial_rec`, `n == 0` devuelve `1` y no llama a nadie.
2. **El caso recursivo**: la situación general que se reduce a una versión más chica del mismo problema. `n * factorial_rec(n - 1)`.

La función "baja" de a un escalón hasta el caso base, y recién ahí empieza a armar la respuesta de vuelta. Desarrollalo a mano una sola vez y te queda grabado:

```
factorial_rec(5) = 5 · factorial_rec(4)
                 = 5 · 4 · factorial_rec(3)
                 = 5 · 4 · 3 · factorial_rec(2)
                 = 5 · 4 · 3 · 2 · factorial_rec(1)
                 = 5 · 4 · 3 · 2 · 1 · factorial_rec(0)
                 = 5 · 4 · 3 · 2 · 1 · 1
                 = 120
```

Cada escalón convierte el problema en "un número multiplicado por un problema más chico". La cadena siempre termina porque `n - 1` se acerca al `0` del caso base; si el caso base no existiera (o nunca se alcanzara), la función seguiría llamándose hasta que el intérprete se canse. Ese "cansancio" tiene nombre y lo vemos ahora.

---

## 3. La pila de llamadas

**Pila de llamadas** (*call stack*): Es el mecanismo por el cual Python recuerda desde dónde se llamó cada función, para saber a dónde volver cuando termina. La recursión es la cámara lenta perfecta para ver esa pila en acción.

Agreguemos *prints* para mirar por dentro. A la bajada, cada llamada dice "buscando"; a la vuelta, "calculando":

```python
def factorial_debug(n):
    print(f"  buscando factorial({n})")
    if n < 2:
        print(f"  caso base: factorial({n}) = 1")
        return 1

    resultado = n * factorial_debug(n - 1)
    print(f"  calculando: {n} * factorial({n - 1}) = {resultado}")
    return resultado

factorial_debug(4)
```

```
  buscando factorial(4)
  buscando factorial(3)
  buscando factorial(2)
  buscando factorial(1)
  caso base: factorial(1) = 1
  calculando: 2 * factorial(1) = 2
  calculando: 3 * factorial(2) = 6
  calculando: 4 * factorial(3) = 24
```

Fijate el orden: primero todo "buscando" (la bajada), después todo "calculando" (la vuelta). Es literalmente como bajar a un pozo y subir: cada llamada queda *apilada* esperando que la de abajo le devuelva un número, y se van desapilando en orden inverso al que se apilaron.

Una metáfora que ayuda mucho es la de un viaje de ida y vuelta:

```python
def tripular(n):
    print(f"viajando {n}")
    if n == 0:
        print("llegué a la base")
        return 0

    total = n + tripular(n - 1)
    print(f"volviendo: sumo {n} -> {total}")
    return total

tripular(3)
```

```
viajando 3
viajando 2
viajando 1
viajando 0
llegué a la base
volviendo: sumo 1 -> 1
volviendo: sumo 2 -> 3
volviendo: sumo 3 -> 6
```

El resultado final, `6`, no se conoce hasta el último instante, cuando todas las llamadas ya volvieron.

Pero toda pila tiene un tope. Python cuenta con cuántas llamadas anidadas está dispuesto a convivir y pone un límite para no quedarse sin memoria:

```python
import sys

print(sys.getrecursionlimit())   # 1000
```

Si una recursión no tiene caso base (o nunca lo alcanza), se pasa de largo y explota con una excepción bien clara:

```python
def caer(n):
    return caer(n + 1)

caer(1)   # RecursionError: maximum recursion depth exceeded
```

> **Dato clave:** el límite por defecto del `RecursionError` es **1000 niveles** (`sys.getrecursionlimit()`). Existe `sys.setrecursionlimit(...)`, y a veces hace falta subirlo un poco, pero no lo subas de más por costumbre: cada nivel ocupa memoria en la pila, y un límite enorme puede terminar tirando el proceso entero. La recursión profunda se resuelve con otra herramienta (generadores, o escribir un bucle), no con `setrecursionlimit`.

---

## 4. `match`/`case` al servicio de la recursión

Acá se juntan dos capítulos: la **deconstrucción** de `match`/`case` del capítulo 5 y la recursión. El clásico problema "sumar todos los elementos de una lista" se puede escribir preguntándose, al estilo recursivo: *¿la lista está vacía? Entonces 0. ¿Tiene elementos? Entonces el primero más la suma del resto.*

Con `match` esta descripción se traduce línea por línea:

```python
def sum_list(datos: list) -> int:
    match datos:
        case []:                                   # lista vacía: caso base
            return 0
        case [primero, *resto]:                    # al menos un elemento
            return primero + sum_list(resto)       # recursión con el resto
        case _:
            raise ValueError(f"sum_list espera una lista, no un {type(datos).__name__}")

print(sum_list([1, 2, 3, 4]))   # 10
print(sum_list([]))             # 0
print(sum_list(42))             # ValueError: sum_list espera una lista, no un int
```

`type(datos).__name__` es la forma de preguntar "¿cómo se llama el tipo de este dato?" (`int`, `str`, `tuple`, ...). El patrón `[primero, *resto]` es el que viste en el capítulo 5: sirve las partes de una secuencia como platos — cabeza en `primero`, cola en `resto` — y la cola es, siempre, una lista más chica. Perfecto para la recursión.

> **Dato clave (y sorpresa útil):** el patrón de secuencia no distingue "lista" de "tupla" ni de "texto": acepta *toda secuencia ordenada*. Por eso `sum_list("hola")` no explota: desarma el texto y lo suma carácter a carácter (da `"hola"`). Y por eso el `case _` del final solo se alcanza con datos que no sean secuencias (como `42`). Si querés aceptar *solo listas*, sumale una guarda `isinstance`:

```python
def suma_solo_listas(datos):
    match datos:
        case []:
            return 0
        case [primero, *resto] if isinstance(datos, list):
            return primero + suma_solo_listas(resto)
        case _:
            raise ValueError("esperaba una lista")

print(suma_solo_listas([1, 2, 3]))   # 6
```

El mismo esquema de "caso base + caso recursivo" arma una lista nueva. Invertir, por ejemplo: *si la lista está vacía, el resultado es vacío; si no, invertí el resto y pegale la cabeza al final.*

```python
def invertir(lista: list) -> list:
    if not lista:
        return []
    return invertir(lista[1:]) + [lista[0]]

print(invertir([1, 2, 3]))   # [3, 2, 1]
```

Desarrollado, se lee solo:

```
invertir([1, 2, 3]) = invertir([2, 3]) + [1]
                    = (invertir([3]) + [2]) + [1]
                    = (([] + [3]) + [2]) + [1]
                    = [3, 2, 1]
```

---

## 5. Ventajas y desventajas (y dónde brilla de verdad)

Antes de enamorarte de la recursión, mirala con los ojos abiertos. Esta tabla resume el balance:

| Ventajas | Desventajas |
|--------------------------|-----------------------------|
| El código **calca la definición** del problema (o la estructura de los datos) | Más lento que un bucle: cada llamada paga costo de pila y de tiempo |
| Muy legible: el problema grande se lee igual que el chico | Límite de profundidad (~1000) → `RecursionError` |
| Formidable para estructuras recursivas (anidadas, árboles, carpetas) | Puede **recalcular** lo mismo millones de veces (lo vemos con fibonacci) |
| Razonamiento en dos pasos: caso base + caso recursivo | Cada llamada viva ocupa memoria en la pila hasta que vuelve |

Por eso la regla editorial honesta para este capítulo: **aprender cómo funciona el mecanismo no significa que haya que usarlo siempre**. El factorial, que usamos para ver la maquinaria, es el ejemplo perfecto del caso "aprendé, y después usá lo práctico" — en la vida real se resuelve con un bucle o, mejor, con la función lista de la biblioteca estándar:

```python
from math import factorial

print(factorial(10))   # 3628800  → directo, sin que vos escribas recursión
```

La recursión **brilla** cuando el problema (o los datos) son *recursivos por naturaleza*: algo que se compone de versiones más chicas de sí mismo. Tres ejemplos del libro:

- **Listas anidadas** (capítulo 5): aplanar una lista de listas. Con una comprensión aplanás *un* nivel; con recursión, profundidad arbitraria (ya viene, sección 9).
- **Árboles**: carpetas de carpetas, "organigrama", jerarquías. En la **Parte VI (POO)** y en las estructuras clásicas vas a modelar árboles donde la recursión es la herramienta natural.
- **El sistema de archivos**: una carpeta *contiene* carpetas — el mismo problema, más chico. Un mini-ejemplo real, con herramientas que la Parte VIII (archivos) va a estudiar en serio:

```python
import os

def listar(ruta: str) -> None:
    """Imprime todas las rutas dentro de una carpeta, a cualquier profundidad."""
    for nombre in os.listdir(ruta):          # nombres que hay dentro
        camino = os.path.join(ruta, nombre)  # ruta completa con el nombre
        if os.path.isdir(camino):            # ¿es una carpeta?
            listar(camino)                   # recursión: adentro hay el mismo problema
        else:
            print(camino)

listar("C:/Users/alguien/Proyectos")
```

Tres funciones nuevas, y las glosamos para que no quede nada en la sombra: `os.listdir(ruta)` devuelve la **lista de nombres** que hay dentro de una carpeta; `os.path.join(ruta, nombre)` **arma la ruta completa** combinando la carpeta y el nombre; `os.path.isdir(camino)` responde **"¿esto es una carpeta?"** con `True` o `False`. (Las barras `/` funcionan igual en Windows.) Probalo sobre una carpeta con pocas cosas. Y una curiosidad de la biblioteca estándar: existe `os.walk`, que hace este recorrido ya listo — lo vas a conocer en la Parte VIII.

> **Dato clave:** la señal de que la recursión es la herramienta correcta es que **la estructura de datos tome la misma forma a cualquier profundidad**: una carpeta contiene carpetas, un árbol tiene hijos que son árboles. Si tu problema no tiene esa forma, la recursión va a pelear en contra — y el bucle gana.

---

## 6. Fibonacci: el caso del "cuándo no"

Hay un ejemplo famoso (y doloroso) del "cuándo no": la **secuencia de Fibonacci**, donde cada número es la suma de los dos anteriores — `0, 1, 1, 2, 3, 5, 8, 13, ...`. Su definición matemática es *recursiva de manual*, así que escribirla recursiva tienta:

```python
llamadas = 0

def fibonacci_rec(n):
    global llamadas
    llamadas += 1
    if n < 2:
        return n
    return fibonacci_rec(n - 1) + fibonacci_rec(n - 2)   # se llama DOS veces por llamada

print(fibonacci_rec(30))        # 832040
print(llamadas)                 # 2692537  → más de 2,6 millones de llamadas
```

El resultado es correcto, pero el precio es escandaloso: para `n = 30` hizo **2.692.537 llamadas**. ¿Por qué? Fijate que `fibonacci_rec(n)` llama *dos veces*: a `n - 1` y a `n - 2`. Y cada una de esas llamadas vuelve a llamar dos veces, y así hacia abajo — un abanico que se duplica en cada nivel. Lo peor: `fibonacci_rec(28)` se termina calculando varias veces, tirando el trabajo anterior a la basura.

La versión iterativa es la misma historia sin el abanico:

```python
def fibonacci_bucle(n):
    a, b = 0, 1
    pasos = 0
    for _ in range(n-1):
        a, b = b, a + b
        pasos += 1
    return b, pasos

print(fibonacci_bucle(30))    # (832040, 29)
```

Veintinueve pasos contra dos millones y medio. La moraleja no es "la recursión es mala": es que la recursión *ingenuo* que vuelve a calcular el mismo subproblema una y otra vez es un costo evitable. En proximos capitulos vamos a medir formalmente esta diferencia — la llamada *notación O grande*; por ahora alcanza con la intuición: una versión cuyo trabajo se duplica con cada nivel crece demasiado rápido al agrandar `n`.

> **Buenas prácticas:** si tu función recursiva vuelve a contarse a sí misma con los mismos argumentos varias veces, hay un patrón llamado **memoización** que guarda resultados ya calculados (lo vas a ver en el `paradigme funcional`) o directamente conviene reescribirla con un bucle. El `global llamadas` del ejemplo es solo un contador de laboratorio, no una técnica para copiar en código serio (capítulo 8).

---

## 7. Lambdas recursivas: la recursión sin nombre

Cerramos el primer acto con un truco que el capítulo 8 dejó listo para disparar: **¿puede una lambda —una función anónima, sin `def`— ser recursiva?** Sí, y vale la pena entender *por qué* porque demuestra qué son realmente las closures.

```python
fact = lambda n: 1 if n <= 1 else n * fact(n - 1)

print(fact(5))   # 120
```

¿Cómo se llama a sí misma si no tiene nombre? Recorrelo sin magia. Del capítulo 8 sabés que una función anidada **recuerda el entorno donde nació** (eso es una *closure*). La lambda nace en el ámbito global, y en ese ámbito vive el nombre `fact`. Cuando la lambda se ejecuta y llega a `fact(n - 1)`, no se llama "por su nombre propio": **busca el nombre `fact` en su entorno, y ese nombre apunta a la misma función lambda.** El lazo se cierra desde afuera: la variable `fact` es *el puente* que le permite a la función alcanzarse a sí misma.

Es un truco elegante, sí — pero también es un espejo del capítulo 8: asignar lambdas a nombres es justo lo que desaconsejan las buenas prácticas. Guardalo como ejercicio mental sobre closures, y en código real escribí la versión con `def`: hace exactamente lo mismo y se lee mejor. (En los ejercicios te espera uno para probar la versión lambda de `sum_list`.)

---

Cortamos acá el primer acto. Tenés la recursión por dentro: caso base, pila, límites, y el ojo para saber cuándo usarla. Ahora viene el segundo: **funciones que se pausan**. En el capítulo 5 quedaron dos promesas pendientes: el `(x for x in …)` de los paréntesis (sección 8) y la "comparación de memoria: lista vs generador" (sección 9, cuando descubrimos `iter()`/`next()`). Es hora de cobrarlas.

---

## 8. Los generadores: funciones que se pausan

Hasta ahora toda función termina con `return`: entrega su valor y muere. Los **generadores** son funciones que pueden hacer algo distinto: entregar un valor, **pausarse en el punto exacto**, y retomar la ejecución donde quedaron cuando se les pide otro. Todo gracias a la palabra **`yield`** (`"ceder"`).

```python
def semaforo():
    yield "rojo"
    yield "amarillo"
    yield "verde"

luces = semaforo()
print(next(luces))   # rojo
print(next(luces))   # amarillo
print(next(luces))   # verde
print(next(luces))   # StopIteration: la cinta se terminó
```

Cada `yield` entrega un valor y **congela la función ahí mismo**. El siguiente `next(luces)` no empieza de nuevo: *retoma desde la línea siguiente al `yield`*. Y cuando no queda ningún `yield`, la función devuelve la excepción `StopIteration` — la misma "che, la cinta se terminó" del capítulo 5. La diferencia con `return` es la clave del capítulo:

> **Dato clave:** a diferencia de `return`, la función generadora **no se termina cuando entrega un valor** — pausa su ejecución y la retoma donde quedó cuando le piden el siguiente. Cada `yield` es una estación donde la función se baja del tren y espera a que la vuelvan a subir.

Hablaba de "la cinta" a propósito: un generador **es un iterador**, exactamente de la familia que viste en el capítulo 5. Le podés pedir con `next()`, consumirlo con un `for` (que hará `next` hasta `StopIteration`, por debajo), pasarlo a `list()` — y, como todo iterador, es **descartable**: una vez agotado, no se reinicia.

Fijate también *cuándo* se ejecuta el cuerpo. Acá la función ni siquiera corre al llamarse:

```python
def contar_hasta(n):
    contador = 0
    while contador < n:
        yield contador
        contador += 1

numeros = contar_hasta(3)        # no imprime nada: recién acá se "arma" el generador
print(next(numeros))             # 0  → ejecuta hasta el primer yield
print(next(numeros))             # 1  → retoma justo después del yield (contador += 1)
print(next(numeros))             # 2
```

Y como respira el `for`, lo podés usar directo:

```python
for numero in contar_hasta(3):
    print(numero)                # 0 1 2  (cada vuelta pide el siguiente)
```

### El generador infinito

Si una función no tiene `return`, ¿qué pasa si su bucle nunca termina? Con `yield`, *no importa*: la función hace un ciclo de pausar-entregar-retomar infinitas veces, y es **consciente**: nunca construye la lista completa. Este es el momento de los números primos que tanto trabajamos en el capítulo 8:

```python
def es_primo(numero: int) -> bool:
    if numero < 2:
        return False
    for divisor in range(2, numero):
        if numero % divisor == 0:
            return False
    return True

def generador_primos():
    numero = 2
    while True:
        if es_primo(numero):
            yield numero        # entregá, pausá, y cuando vuelvan seguí
        numero += 1

primos = generador_primos()
print(next(primos), next(primos), next(primos),
      next(primos), next(primos))   # 2 3 5 7 11
```

Primos *infinitos*, consumidos de a uno. ¿Le pasás `list(primos)`? No lo hagas: sería pedirle al generador que entregue hasta el fin de los tiempos. Los infinitos se consumen con `next()` o con un `for` que tenga un `break`. (Detalle de biblioteca: `itertools.count` del capítulo 5 también genera infinitos, con la misma regla de oro.)

---

## 9. `yield from`: pasarle la cinta a otra cinta

Cuando un generador quiere entregar, uno por uno, todos los valores de *otra* colección, hay una forma corta de decirlo: **`yield from`** ("cedé desde"). Es como un atajo para el `for` + `yield` que ya imaginás:

```python
def intercalar(*fuentes):
    for fuente in fuentes:
        yield from fuente

print(list(intercalar([1, 2], [3, 4], [5])))   # [1, 2, 3, 4, 5]
```

`yield from fuente` significa: "tomá la cinta entera de `fuente` y pasámela toda, una a una". El generador *delega*; es código equivalente a `for x in fuente: yield x`, pero más declarativo.

Y ahora sí, el momento que el capítulo prometió desde el arranque: **un generador recursivo**. Recordás el aplanar de listas del capítulo 5: la comprensión `[x for fila in matriz for x in fila]` y `itertools.chain` aplanaban **un nivel**. ¿Y si las listas están anidadas a cualquier profundidad? Recordá la sección 5: la estructura es recursiva, así que la buena herramienta es la recursión. Combinada con `yield`, queda cortísimo:

```python
def aplanar(estructura):
    for elemento in estructura:
        if isinstance(elemento, list):
            yield from aplanar(elemento)    # recursión + delegación
        else:
            yield elemento

anidado = [1, [2, [3, 4], 5], [], 6]
print(list(aplanar(anidado)))   # [1, 2, 3, 4, 5, 6]
```

Leelo: "para cada elemento, si es una lista, *entoná* la misma melodía con ella (delegá); si no, entregalo". El resultado es un generador que **no construye la lista aplanada**: la produce a demanda. Aplanar algo enorme sin ocupar memoria — recursión y generadores, las dos ideas del capítulo, en una sola función de cuatro líneas.

> **Buenas prácticas:** `yield from` no es solo azúcar: también le avisa a Python que *delege la iteración* (es más eficiente que los `for`+`yield` manuales). Y si es **muy** profunda, la recursión de `aplanar` choca con el límite de ~1000 de la sección 3 — se nota más en la práctica con estructuras de cientos de miles de niveles que con las listas cotidianas.

---

## 10. Expresiones generadoras: `(x for x in …)`, ahora en serio

En la sección 8 del capítulo 5 conociste los paréntesis que no eran "una lista con paréntesis": **`(x for x in …)` crea un generador directamente, en una línea, sin `def` ni `yield`.** Es la misma sintaxis de comprensión, pero con paréntesis → *perezosa*:

```python
cuadrados = (x * x for x in range(5))   # un generador, no una lista

print(next(cuadrados))    # 0
print(list(cuadrados))    # [1, 4, 9, 16]  → siguió donde quedó, 0 ya se entregó
```

La diferencia con la lista `[x * x for x in range(5)]` es cuándo se fabrica cada valor: la **comprensión** los calcula todos **ya** (ansiosa, *eager*); la **expresión generadora** los produce **cuando se los piden** (perezosa, *lazy*). Para recorrer una vez, la expresión generadora es idéntica — y ni siquiera necesitás guardarla, se usa al vuelo:

```python
suma = sum(x for x in range(101))
print(suma)   # sin construir la lista de los 100 números
```

Y acá se paga la deuda pendiente del capítulo 5 — la **comparación de memoria**:

```python
import sys

lista = [x * x for x in range(1_000_000)]
perezoso = (x * x for x in range(1_000_000))

print(sys.getsizeof(lista))      # 8448728  → unos 8,4 millones de bytes
print(sys.getsizeof(perezoso))   # 200      → el generador ocupa casi nada
```

`sys.getsizeof` mide cuántos bytes ocupa un objeto en memoria. La lista guardó un millón de valores **ya**: 8,4 megabytes. El generador guardó... la *receta* para producirlos: 200 bytes. Eso es lo que venías usando sin saberlo: la lista es el resultado, el generador es la fábrica.

> **Dato clave — cuál usar cuándo:** la comprensión lista si necesitás **volver sobre los datos** (re-recorrer, indexar con `[i]`, medir con `len()`, ordenar, pasar más de una vez). El generador cuando recorrés **una sola vez** y/o la secuencia es gigante o infinita: mismo resultado, fracción de memoria. El generador se agota (capítulo 5) y no tiene `len`; la lista es eterna y medible.

---

## 11. Resumen y conceptos clave

Este capítulo tuvo dos actos y un puente. Del primer acto, la **recursión**: una función que se llama a sí misma con un problema más chico, sostenida por un **caso base** que la detiene y una **pila de llamadas** que la memoria el camino de vuelta. Aprendiste a leer la receta en dos partes, a usar `match`/`case` para deconstruir de a un elemento (`[primero, *resto]`), y ganaste criterio: brilla en estructuras **recursivas** (listas anidadas, árboles, carpetas) y duele cuando **recalcula** (fibonacci, 2,6 millones de llamadas) o pide profundidad infinita (`RecursionError`). Del segundo acto, los **generadores**: funciones que **pausan** con `yield` y retoman donde quedaron, son iteradores de un solo uso (capítulo 5), soportan infinitos sin despeinarse, delegan con `yield from` — y hasta pueden ser recursivos (aplanar). Cerró con las **expresiones generadoras** `(x for x in …)`: la misma idea de comprensión, pero perezosa y económica (200 bytes contra 8,4 MB).

Repasá esta lista antes de seguir:

- [ ] **Recursión** = una función que se llama a sí misma con un problema más chico.
- [ ] Toda función recursiva tiene **caso base** (se responde directo) y **caso recursivo** (reduce el problema).
- [ ] El caso base evita el bucle infinito; sin él → `RecursionError: maximum recursion depth exceeded` (~1000, `sys.getrecursionlimit()`).
- [ ] La **pila de llamadas**: cada llamada queda apilada hasta que las de abajo vuelven; se imprime "buscando" a la bajada y "calculando" a la vuelta.
- [ ] `factorial`: sirve para **aprender el mecanismo**; en la práctica, bucle o `math.factorial`.
- [ ] `sum_list` con `match`/`case`: `case []` (base), `case [primero, *resto]` (recursión), `case _` (`ValueError` con `type(datos).__name__`).
- [ ] El patrón de secuencia acepta listas, tuplas y textos; con `if isinstance(datos, list)` lo restringís.
- [ ] Recursión brilla con **estructuras recursivas**: listas anidadas, árboles (Parte VI), carpetas (`os.listdir`/`os.path.isdir`/`os.path.join`; `os.walk` en Parte VIII).
- [ ] **Fibonacci ingenuo recalcula** (2.692.537 llamadas para `n=30`); el bucle hace 30 pasos. Su intuición de costos se formaliza más adelante (notación O grande, post-POO).
- [ ] **Lambda recursiva**: se llama a sí misma a través del nombre del entorno (closure); preferí siempre `def`.
- [ ] Un **generador** usa `yield`: entrega un valor, **pausa** y **retoma** desde el `yield` siguiente.
- [ ] Generador = iterador: `next()`, `for`, `list()`, y **descartable** (capítulo 5); agotado, `StopIteration`.
- [ ] El cuerpo de un generador **no corre hasta el primer `next()`**.
- [ ] Generadores **infinitos** (`while True` + `yield`, primos): con `next()` o `for` + `break`; jamás `list()`.
- [ ] **`yield from`** delega la cinta de otro iterable (equivalente a `for x in fuente: yield x`).
- [ ] **Generador recursivo**: `aplanar` aplana listas anidadas a cualquier profundidad, a demanda y sin construir nada.
- [ ] **Expresión generadora** `(x for x in …)`: perezosa; la comprensión `[...]` es ansiosa.
- [ ] Memoria: `[x*x for x in range(1_000_000)]` ≈ 8,4 MB vs `(x*x for x in …)` ≈ 200 bytes (`sys.getsizeof`).
- [ ] Cuándo: lista si re-recorrés / indexás / medís; generador si un solo recorrido y la memoria importa.

---

## 12. Ejercicios

1. **Cuenta regresiva**: escribí `cuenta_regresiva(n)` recursiva que imprima `n`, `n-1`, …, `1` y termine con `"¡despegue!"`. ¿Dónde está el caso base?
2. **Potencia recursiva**: `potencia(base, exp)` calcula `base**exp` para `exp >= 0` sin usar `**`. Caso base: `exp == 0` → `1`; caso recursivo: `base * potencia(base, exp - 1)`.
3. **Contar cifras**: `cantidad_de_digitos(n)` devuelve cuántas cifras tiene un entero positivo (pista: dividí por 10 hasta llegar a 0; `n // 10` te quita la última cifra).
4. **`sum_list` a prueba**: probá `sum_list` de la sección 4 con `[1, 2, 3, 4]`, con `[]` y con `(1, 2, 3)`. ¿Por qué la tupla *no* dispara el `ValueError`? ¿Y qué pasa si le sumás la guarda `isinstance`?
5. **Aplanar, versión generadora**: inverti `anidado = [1, [2, [3, 4], 5], [], 6]` con `list(aplanar(anidado))` y comparalo con las formas de *un* solo nivel del capítulo 5 (comprensión y `itertools.chain`). ¿En qué se nota la diferencia?
6. **Fibonacci generador**: escribí `generador_fibonacci()` (generador *infinito* que entregue `0, 1, 1, 2, 3, 5, …`). Consumí los primeros 10 con un `for` y `break` o con `next()`. Cuidado: no lo pases a `list()`.
7. **Pares a la demanda**: escribí `pares_desde(0)` que genere los números pares desde `inicio` hasta el infinito, y mostrá los primeros 5.
8. **Sin `yield from`**: reescribí `intercalar` sin usar `yield from` (anidá un `for` manual). Verificá que da lo mismo.
9. **Suma de cuadrados perezosa**: con una expresión generadora, calculá la suma de los cuadrados del 1 al 100 **sin crear ninguna lista** (pista: `sum(...)`).
10. **Lambda recursiva**: escribí la versión lambda de `sum_list` (pista: `suma = lambda lista: ... if lista else 0`) y verificala con `[1, 2, 3]`.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. Cuenta regresiva
def cuenta_regresiva(n):
    print(n)
    if n == 1:
        print("¡despegue!")
        return
    cuenta_regresiva(n - 1)

cuenta_regresiva(3)

# 2. Potencia recursiva
def potencia(base, exp):
    if exp == 0:
        return 1
    return base * potencia(base, exp - 1)

print(potencia(2, 10))   # 1024

# 3. Contar cifras
def cantidad_de_digitos(n):
    if n < 10:
        return 1
    return 1 + cantidad_de_digitos(n // 10)

print(cantidad_de_digitos(12345))   # 5

# 4. sum_list a prueba (la función es la de la sección 4; copiala acá para probar)
print(sum_list([1, 2, 3, 4]))     # 10
print(sum_list([]))               # 0
print(sum_list((1, 2, 3)))        # 6  → la tupla es secuencia, entra a los patrones
# con la guarda isinstance de suma_solo_listas, la tupla cae en `case _` y lanza ValueError

# 5. Aplanar, versión generadora
anidado = [1, [2, [3, 4], 5], [], 6]
print(list(aplanar(anidado)))              # [1, 2, 3, 4, 5, 6]  → profundidad arbitraria
# la comprensión [x for sub in anidado for x in sub] y chain solo aplanan UN nivel
# (les faltaría la recursividad): probá y vas a ver los subelementos anidados intactos.

# 6. Fibonacci generador
def generador_fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

n = 0
for valor in generador_fibonacci():
    print(valor, end=" ")    # 0 1 1 2 3 5 8 13 21 34
    n += 1
    if n == 10:
        break
print()

# 7. Pares a la demanda
def pares_desde(inicio):
    numero = inicio
    while True:
        if numero % 2 == 0:
            yield numero
        numero += 1

pares = pares_desde(0)
print(next(pares), next(pares), next(pares), next(pares), next(pares))   # 0 2 4 6 8

# 8. Sin yield from
def intercalar_viejo(*fuentes):
    for fuente in fuentes:
        for item in fuente:
            yield item

print(list(intercalar_viejo([1, 2], [3, 4], [5])))   # [1, 2, 3, 4, 5]

# 9. Suma de cuadrados perezosa
print(sum(x * x for x in range(1, 101)))            # 338350

# 10. Lambda recursiva
suma = lambda lista: lista[0] + suma(lista[1:]) if lista else 0
print(suma([1, 2, 3]))                              # 6
```

Con esto cerramos la recursión y los generadores — dos herramientas con las que una función *describe mucho más de lo que escribe*. La Parte V completa su última vuelta en el capítulo que viene, donde se junta toda la familia **funcional**: `map`, `filter`, `reduce` y la biblioteca `functools`, primos directos de las listas por comprensión y de los generadores que recién aprendiste. Si los generadores te dijeron "describí *qué* producir", ahí vas a ver "describí qué *transformar*". Nos vemos en la Parte V, última estación.