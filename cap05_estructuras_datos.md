# Capítulo 5 — Estructuras de datos: listas, tuplas, diccionarios y conjuntos

Hasta acá trabajaste con variables que guardan un solo valor: un número, una cadena, un `True` o un `False`. Pero cuando los datos crecen —una lista de precios, los contactos de tu agenda, las notas de un curso— necesitás guardar **muchos valores juntos**. De eso se trata este capítulo: las **estructuras de datos** nativas de Python, es decir, las formas que tiene el lenguaje para agrupar y organizar valores.

No es un capítulo más: es la base que usarás en prácticamente todo el resto del libro. Los `DataFrame` de pandas (Parte IX) son la evolución "de laboratorio" de estas estructuras, y entender bien qué diferencia una lista de un diccionario te va a ahorrar horas de confusión ahí adelante. Además vas a descubrir una herramienta que se vuelve adictiva: las **listas y diccionarios por comprensión**, una forma corta, elegante y muy pythónica de construir colecciones.

---

## 1. El mapa de las estructuras: tres criterios

Antes de ver cada estructura por separado, te conviene ver **el mapa completo**. Python ofrece seis estructuras de datos nativas —`str`, `list`, `tuple`, `dict`, `set` y `frozenset`— y la pregunta no es "cuál es mejor", sino "cuál encaja con lo que necesito". Para responderla solo tenés que cruzar **tres criterios**:

- **¿Mantiene orden e índice?** Es decir, ¿podés acceder con `mi_estructura[3]` y el resultado depende de la *posición*? Las que sí se llaman **secuencias**.
- **¿Es mutable?** Es decir, ¿podés cambiarla después de crearla (agregar, quitar, reemplazar elementos)?
- **¿Admite elementos repetidos?** Es decir, ¿un mismo valor puede aparecer dos veces?

Cruzá esos tres criterios contra cada estructura:

| Estructura | Tipo         | Orden / índice | Mutable | Elementos repetidos | Ejemplo |
|------------|--------------|----------------|---------|---------------------|---------|
| `str`      | secuencia    | sí             | no      | sí                  | `"hola"` |
| `list`     | secuencia    | sí             | sí      | sí                  | `[1, 2, 2, 3]` |
| `tuple`    | secuencia    | sí             | no      | sí                  | `(1, 2, 2, 3)` |
| `dict`     | mapeo        | por clave      | sí      | solo claves únicas  | `{"edades": 10}` |
| `set`      | conjunto     | no             | sí      | no                  | `{1, 2, 3}` |
| `frozenset`| conjunto     | no             | no      | no                  | `frozenset([1, 2])` |
| `range`    | secuencia    | sí             | no      | no                  | `range(5)` |

Este mapa es tu *hoja de ruta*. Cada sección del capítulo explora una casilla de esta tabla; al final, en la sección "¿Cuál usar cuándo?", vas a tener una guía de decisión completa.

> **Dato clave:** los tres criterios son **independientes**. Una tupla es ordenada **y** inmutable; un conjunto es mutable **pero** desordenado; un diccionario es mutable **y** no usa posición sino clave. Cada estructura es una combinación distinta de los mismos tres ejes.

> **Importante:** cuando hablamos de "orden e índice" en `dict`, cuidado con una trampa. Los diccionarios **sí conservan el orden de inserción desde Python 3.7**, pero NO se accede por índice de posición como en una lista: se accede **por clave**. Es un orden de *recorrido*, no de *posicionamiento*.

---

## 2. Secuencias: el índice es la ventaja

Las secuencias —`str`, `list` y `tuple`— son las estructuras que **mantienen un orden y permiten indexar**. Ya conocés una perfectamente: los strings del capítulo 3. Todo lo que aprendiste ahí se traslada a las listas y tuplas:

```python
# strings (capítulo 3)
print("python"[0])        # p
print("python"[-1])       # n
print("python"[1:4])      # yth
print("python"[::2])      # pto

# listas: mismas reglas de indexing y slicing
colores = ["rojo", "verde", "azul"]
print(colores[0])         # rojo
print(colores[-1])        # azul
print(colores[0:2])       # ['rojo', 'verde']

# tuplas: también
planetas = ("mercurio", "venus", "tierra")
print(planetas[1])        # venus
print(planetas[-1])       # tierra
```

Lo que comparten las tres secuencias:

- **Indexing** con `[i]`, incluidos índices negativos (`[-1]` es el último).
- **Slicing** con `[ inicio : fin : paso ]`.
- **`len()`** para la cantidad de elementos.
- **`in`** para preguntar si un elemento está: `"verde" in colores`.
- **Concatenación** con `+` y **repetición** con `*`.
- **Iteración** con `for` (lo viste en el capítulo 4).

```python
a = [1, 2]
b = [3, 4]
print(a + b)      # [1, 2, 3, 4]
print(a * 2)      # [1, 2, 1, 2]

t = (1, 2)
print(t + (3,))   # (1, 2, 3)
print(t * 3)      # (1, 2, 1, 2, 1, 2)
```

La diferencia grande entre las tres es la **mutabilidad**, y por eso las tratamos por separado en las dos secciones que siguen.

---

## 3. Listas: mutables y ordenadas

La **lista** es la secuencia **mutable**: podés crear una vacía, agregarle elementos, quitar otros y reordenarla. Se escribe entre **corchetes** `[...]` y separando los elementos con comas.

```python
frutas = []                    # lista vacía
frutas.append("manzana")       # agrego al final
frutas.append("pera")
print(frutas)                  # ['manzana', 'pera']
```

### Métodos principales de las listas

| Método       | Qué hace                                         |
|--------------|--------------------------------------------------|
| `append(x)`  | agrega `x` al **final**                          |
| `extend(it)` | agrega al final **varios** elementos de un iterable |
| `insert(i, x)` | inserta `x` en la **posición** `i`              |
| `remove(x)`  | elimina la **primera** aparición de `x` (error si no existe) |
| `pop()` / `pop(i)` | elimina y devuelve el último / el de posición `i` |
| `del lista[i]` | elimina la posición `i` sin devolverla          |
| `clear()`    | vacía la lista                                   |
| `index(x)`   | posición de la primera aparición de `x`          |
| `count(x)`   | cuántas veces aparece `x`                        |
| `sort()`     | ordena en el lugar (la lista queda ordenada)     |
| `reverse()`  | invierte el orden en el lugar                    |

```python
numeros = [3, 1, 4, 1, 5]
numeros.reverse()
print(numeros)               # [5, 4, 3, 1, 1]

letras = ["b", "a"]
letras.extend(["c", "e"])    # agrego varios
letras.insert(3, "d")        # inserto en la posición 3
print(letras)                # ['b', 'a', 'c', 'd', 'e']
```

### Buscar posiciones: `index()`

El método `index(x)` devuelve el **índice (posición) de la primera aparición** del valor `x`. Si no existe, lanza un **`ValueError`**:

```python
numeros = [10, 20, 30, 20, 40]
print(numeros.index(20))      # 1  -> posición del primer 20
print(numeros.index(40))      # 4

# frutas = ["manzana", "pera"]
# print(frutas.index("uva"))  # ValueError: 'uva' is not in list
```

Para evitar el error si no estás seguro de que existe, combiná `index()` con un `if` usando el operador `in`:

```python
frutas = ["manzana", "pera"]
if "uva" in frutas:
    posicion = frutas.index("uva")
else:
    posicion = None  # None indica "no encontrado"
print(posicion)   # None
```

**Opciones: buscar desde una posición específica**

`index()` admite parámetros opcionales para buscar dentro de un rango: `lista.index(x, inicio, fin)`:

```python
numeros = [10, 20, 30, 20, 40, 20]
print(numeros.index(20))           # 1  -> primer 20
print(numeros.index(20, 2))        # 3  -> primer 20 a partir de índice 2
print(numeros.index(20, 4))        # 5  -> primer 20 a partir de índice 4

# también se puede limitar el final (no incluye el índice fin)
print(numeros.index(20, 0, 4))     # 1  -> busca en [10, 20, 30, 20], encuentra en 1

# print(numeros.index(20, 5, 6))   # ValueError: 20 no está en la posición 5
```

### Ordenar: `sort()` y sus opciones

El método `sort()` ordena **la lista en el lugar** (no devuelve nada, modifica la lista original). Tiene dos opciones clave:

```python
# Ordenamiento básico
numeros = [3, 1, 4, 1, 5, 9]
numeros.sort()
print(numeros)                # [1, 1, 3, 4, 5, 9]

# reverse=True invierte el orden (descendente)
numeros2 = [3, 1, 4, 1, 5]
numeros2.sort(reverse=True)
print(numeros2)               # [5, 4, 3, 1, 1]

# key= permite definir el criterio de ordenamiento
palabras = ["python", "java", "c", "javascript"]
palabras.sort(key=len)        # ordena por LONGITUD
print(palabras)               # ['c', 'java', 'python', 'javascript']

# key= y reverse= se pueden combinar
palabras2 = ["python", "java", "c", "javascript"]
palabras2.sort(key=len, reverse=True)   # por longitud, de mayor a menor
print(palabras2)              # ['javascript', 'python', 'java', 'c']
```

> **Dato clave:** `sort()` modifica la lista original y **devuelve `None`**. Si querés una **nueva lista ordenada sin tocar la original**, usá la función global `sorted(lista)`:
>
> ```python
> original = [3, 1, 4]
> ordenada = sorted(original)      # nueva lista, original intacta
> print(original)                  # [3, 1, 4]
> print(ordenada)                  # [1, 3, 4]
> ```


### Borrar elementos: `del`, `pop` y `remove`

Hay tres maneras de borrar, y no son intercambiables:

```python
mi_lista = [10, 20, 30, 20]

mi_lista.remove(20)      # borra la PRIMERA aparición del VALOR 20
print(mi_lista)          # [10, 30, 20]

ultimo = mi_lista.pop()  # quita y DEVUELVE el último
print(ultimo, mi_lista)  # 20 [10, 30]

del mi_lista[0]          # borra por ÍNDICE, sin devolver nada
print(mi_lista)          # [30]
```

- `remove(x)` borra por **valor**: sabés qué contenido ya no querés.
- `pop(i)` borra por **índice** y te **devuelve** lo quitado (sin argumento, el último).
- `del lista[i]` borra por **índice** sin devolver nada.

> **Dato clave:** con un `for` que recorre una lista **estás recorriendo una copia a la vista, pero la lista es la misma**. Si modificás la lista mientras la recorrés, los índices se corren y vas a saltear elementos. Si necesitás filtrar, preferí una comprensión (sección 8) o recorré una copia: `for x in lista[:]`.


### Slicing devuelve una copia

Cuando hacés `lista[inicio:fin]`, el resultado es una **nueva lista**, no una ventana sobre la original. Eso significa que modificar la copia no toca a la original:

```python
original = [1, 2, 3, 4, 5]
copia = original[1:4]        # una NUEVA lista
copia[0] = 99
print(copia)                 # [99, 3, 4]
print(original)              # [1, 2, 3, 4, 5]  sin cambios
```

Y un atajo muy usado: **`lista[:]`** copia toda la lista.

### Listas anidadas

Una lista puede contener cualquier cosa, incluida otra lista. Eso te da **tablas**: una lista de filas, donde cada fila es una lista de celdas.

```python
personas = [["emiliano", 45], ["luca", 10], ["belen", 40]]

print(personas[0])          # ['emiliano', 45]
print(personas[0][1])       # 45   -> primer fila, segunda columna
```

Para recorrerla, usás dos `for` anidados o un acceso de dos pasos.


---

## 4. Tuplas: inmutabilidad y unpacking

La **tupla** es la secuencia **inmutable**: se ve y se recorre igual que una lista, pero **no se puede modificar** después de crearla. Se escribe entre **paréntesis** `(...)`.

```python
punto = (3, 5)
print(punto[0])        # 3
print(len(punto))      # 2

# intentar modificarla es un error:
# punto[0] = 99        -> TypeError: 'tuple' object does not support item assignment
```

### La coma, no los paréntesis

Los paréntesis son opcionales en muchos casos; lo que define a una tupla es la **coma**. De hecho, una tupla de un solo elemento necesita la coma final, y una coma suelta sin paréntesis también crea tupla:

```python
t1 = (1,)              # tupla de UN elemento (la coma es obligatoria)
t2 = 1, 2, 3           # también es una tupla
vacia = ()             # tupla vacía

print(t1)              # (1,)
print(t2)              # (1, 2, 3)
```

> **Dato clave:** `(1)` NO es una tupla, es el número `1` entre paréntesis. La tupla de un elemento es `(1,)`, con la coma. Es la pregunta trampa clásica de exámenes.

### Packing y unpacking

Las tuplas se crean agrupando valores (**packing**, "empaquetar") y se "abren" en variables individuales (**unpacking**, "desempaquetar"):

```python
coordenadas = (-34.6, -58.4)      # packing: dos valores en una tupla
lat, lon = coordenadas            # unpacking: la tupla se abre en dos variables
print(lat, lon)                   # -34.6 -58.4
```

El unpacking no es solo para tuplas: funciona con **cualquier secuencia** y es la base de varios trucos de Python:

```python
a, b = 1, 2          # el clásico intercambio sin variable auxiliar
a, b = b, a
print(a, b)          # 2 1

var1, var2 = ["x", "y"]     # también con listas
print(var1, var2)           # x y
```

Y se puede usar con `*` para capturar "el resto":

```python
primero, *resto, ultimo = [1, 2, 3, 4, 5]
print(primero)      # 1
print(resto)        # [2, 3, 4]
print(ultimo)       # 5
```

### Métodos de tuplas: solo lectura

Como las tuplas son **inmutables**, no tienen métodos de modificación como `append`, `sort` o `remove`. Solo heredan dos métodos de secuencia:

- **`index(x)`** — devuelve la posición de la primera aparición de `x` (igual que en listas).
- **`count(x)`** — cuenta cuántas veces aparece `x` en la tupla, mismo comportamiento con listas.

```python
numeros = (1, 2, 3, 2, 4, 2, 5)
print(numeros.index(2))      # 1  -> primera aparición en posición 1
print(numeros.count(2))      # 3  -> aparece 3 veces

nombres = ("ana", "juan", "ana", "pedro", "ana")
print(nombres.count("ana"))  # 3
print(nombres.index("juan")) # 1
```

> **Dato clave:** `count()` es especialmente útil en tuplas porque es la única forma de "investigar" sin poder modificar. En listas también existe, pero muchas veces se prefiere `remove()` o filtrado directo.


### ¿Lista o tupla?

Esta es la pregunta que más confunde y la respuesta depende **solo de la mutabilidad**:

- **Tupla** cuando los datos representan un **valor que no debe cambiar**: las coordenadas de un punto, una fecha, un par ordenado. Además, la tupla es **inmutable y por lo tanto "hashable"** (se puede usar como clave de diccionario o elemento de conjunto); la lista **no**.
- **Lista** cuando la colección va a **crecer o cambiar**: una lista de compras, el historial de temperaturas, los nombres de una tabla.

La regla práctica: "si en tu intención **nunca vas a modificarla**, usá tupla; si algo puede crecer o cambiar, lista". Python te deja elegir, y elegir bien documenta tu **intención** en el código.

```python
color = (255, 0, 0)          # un valor fijo -> tupla
colores = []                 # colección que crece -> lista
colores.append(color)        # la lista puede contener tuplas
```

> **Buenas prácticas:** cuando una función devuelve varios valores, en Python devuelve **una tupla**. Al llamarla con unpacking —`lat, lon = obtener_coordenadas()`— estás "abriendo" esa tupla. Es el patrón más frecuente del lenguaje, y ahora ya sabés por qué existe.

### Operaciones con tuplas: `+`, `*` y slicing crean nuevas tuplas

Aquí viene un punto que **confunde**: si hacés `tupla = tupla + (4,)` o `tupla = tupla * 2`, parece que la estás modificando. **Pero no es así.** La tupla es inmutable; lo que sucede es que estás **creando una tupla nueva** y guardándola en la misma variable.

Para verlo claro, usá dos variables:

```python
original = (1, 2, 3)
copia = original          # la MISMA tupla en dos variables

# ahora, operación + guardada en original
original = original + (4,)

print(original)     # (1, 2, 3, 4) -> nueva tupla
print(copia)        # (1, 2, 3)    -> la tupla original NO cambió
```

Ojo: `copia = original` **no copia nada**, simplemente hace que ambas variables **apunten a la misma tupla en memoria**. Por eso cuando modificás `original`, `copia` sigue viendo la tupla original. La tupla no fue "reasignada" a `original`; fue **reemplazada por una nueva tupla**, y `original` ahora apunta a esa nueva tupla, pero `copia` sigue apuntando a la vieja.

Lo mismo ocurre con `*` y slicing:

```python
t = (1, 2)
t2 = t

t = t * 3        # crea una tupla nueva (1, 2, 1, 2, 1, 2)
print(t)         # (1, 2, 1, 2, 1, 2)
print(t2)        # (1, 2)  -> la tupla original no cambió

t3 = (10, 20, 30, 40)
t4 = t3

t3 = t3[1:3]     # crea una tupla nueva (20, 30)
print(t3)        # (20, 30)
print(t4)        # (10, 20, 30, 40)  -> la original intacta
```

> **Dato clave:** esto es exactamente lo que viste con strings en el capítulo 3. Las strings también son inmutables, y `s = s.upper()` no modifica `s`: crea una nueva string y la guarda en `s`. Con tuplas ocurre lo mismo. La **inmutabilidad** significa que una vez creada, el objeto no cambia; si querés un "resultado nuevo", tenés que **crear un objeto nuevo** y guardarlo (en la misma variable o en otra, eso es una decisión de código, no de Python).

---

## 4.5 `slice()`: objetos reutilizables

Cuando hacés `lista[1:4:2]`, Python crea internamente un objeto `slice`. Pero podés crear esos objetos explícitamente y reutilizarlos con **cualquier secuencia**:

```python
# Crear un slice reutilizable
mi_slice = slice(1, 4)        # desde índice 1 hasta 4 (4 no incluido)

# Usarlo con una lista
lista = [10, 20, 30, 40, 50]
print(lista[mi_slice])        # [20, 30, 40]

# Usarlo con una tupla
tupla = (10, 20, 30, 40, 50)
print(tupla[mi_slice])        # (20, 30, 40)

# Usarlo con un string
texto = "python"
print(texto[mi_slice])        # yth
```

También podés incluir el `paso`:

```python
cada_dos = slice(0, 6, 2)     # desde 0 hasta 6, cada 2

lista = [0, 1, 2, 3, 4, 5]
print(lista[cada_dos])        # [0, 2, 4]

texto = "python"
print(texto[cada_dos])        # pto
```

Esto es especialmente útil cuando necesitás **aplicar el mismo slice a múltiples secuencias** sin escribir el índice cada vez:

```python
# Extraer el mismo rango de varias listas
datos_enero = [10, 20, 30, 40, 50, 60]
datos_febrero = [15, 25, 35, 45, 55, 65]

trimestre = slice(1, 4)       # meses 1 a 3 (índices 1, 2, 3)

print(datos_enero[trimestre])    # [20, 30, 40]
print(datos_febrero[trimestre])  # [25, 35, 45]
```

> **Dato clave:** `slice()` es útil cuando necesitás el mismo rango en varias secuencias. Si solo lo usas una vez, la notación directa `[1:4]` es más legible. Pero cuando se repite, `slice()` evita repetición y documenta tu intención: "este rango tiene significado, lo reutilizo en varios lados".

---

## 5. Diccionarios: acceso por clave

El **diccionario** (`dict`) es un **mapeo**: asocia **claves únicas** con **valores**. No se accede por posición sino **por clave**. Es la estructura perfecta cuando el problema es "dado un nombre, dame su valor" — en vez de buscar en una lista elemento por elemento.

### Las tres formas de crear un diccionario

```python
# 1. Con llaves, pares clave: valor
calorias = {"manzana": 52, "banana": 89, "naranja": 47}

# 2. Con dict(nombre=valor) — las claves son identificadores válidos
persona = dict(nombre="Ana", edad=30, ciudad="Buenos Aires")

# 3. Con dict(iterable de pares) — para claves que NO son identificadores
temperaturas = dict([("lunes", 22), ("martes", 24)])

print(persona["edad"])        # 30
print(temperaturas["lunes"])  # 22

```

### Claves: inmutables, únicas y "hashables"

Las claves de un diccionario tienen que cumplir **dos reglas de hierro**:

1. **Tienen que ser únicas**: no podés tener dos pares con la misma clave (si intentás, la segunda sobrescribe la primera).
2. **Tienen que ser inmutables**: Python necesita que no cambien después de crearlas. Esto es porque Python usa un sistema de "hash" para encontrar rápidamente dónde está cada clave en memoria.

Lo importante: **solo objetos "hashables"** (inmutables) pueden ser claves. ¿Cuáles son hashables?

```python
# ✅ SÍ pueden ser claves (inmutables):
numeros = {1: "uno", 2: "dos"}                    # int
precios = {"manzana": 50, "pera": 60}            # str
coordenadas = {(0, 0): "origen", (1, 1): "diagonal"}  # tuple

# ❌ NO pueden ser claves (mutables):
# mal = {["a", "b"]: "valor"}    # TypeError: unhashable type: 'list'
# mal = {{1, 2}: "valor"}        # TypeError: unhashable type: 'set'
```

Si necesitás una **clave compuesta** (varios datos juntos), la tupla es la solución:

```python
personas = {
    ("juan", 1990): "dato1",
    ("ana", 1985): "dato2",
    ("pedro", 1992): "dato3",
}

print(personas[("ana", 1985)])   # dato2
```

> **Dato clave:** los diccionarios son **mutables** (podés agregar/quitar/modificar pares), pero las claves son **inmutables y únicas**. Esa es la garantía que Python necesita para acceso veloz. Podés usar strings, números, tuplas… pero no listas, sets o diccionarios como claves.

> **Nota hacia adelante:** en **POO** (Parte VI) aprenderás a definir tus propias clases. Más adelante, en **Estructuras de Datos Avanzadas**, descubrirás cómo hacer que tus objetos personalizados sean **hashables** (es decir, que puedan usarse como claves de diccionario o elementos de conjuntos). Por ahora, mantené en mente que esta capacidad existe y que será una herramienta poderosa.

### Acceso seguro: `get()` y `setdefault()`

Acceder con `diccionario[clave]` lanza un **`KeyError`** si la clave no existe. Tienes dos formas de evitarlo:

**Forma clásica: `if` e `in`**

```python
calorias = {"manzana": 52, "banana": 89}

if "kiwi" in calorias:
    valor = calorias["kiwi"]
else:
    valor = 0  # valor por defecto
print(valor)    # 0
```

**Forma optimizada: `get()`**

El método `get()` hace lo mismo en una sola expresión:

```python
calorias = {"manzana": 52, "banana": 89}

print(calorias.get("kiwi"))        # None  (por defecto)
print(calorias.get("kiwi", 0))     # 0     (tu valor por defecto)
print(calorias.get("banana", 0))   # 52    (existe, devuelve el valor real)
```

Y `setdefault(clave, valor)` hace algo diferente: devuelve el valor si la clave **ya existe**; si no existe, **la agrega** con ese valor y te lo devuelve.

```python
dic = {"a": 1}
print(dic.setdefault("b", 2))   # 2 (no existía "b"; la agrega)
print(dic.setdefault("a", 99))  # 1 (ya existía "a"; la deja igual)
print(dic)                      # {'a': 1, 'b': 2}
```

> **Diferencia clave:** `get(clave, default)` **nunca modifica** el diccionario (solo lee). `setdefault(clave, valor)` **agrega la clave** si no existe. Usá `get()` para leer con seguridad; usá `setdefault()` cuando tu intención es "agrega la clave si falta".

### Agregar, modificar y actualizar

```python
calorias = {"manzana": 52}

calorias["pera"] = 57           # agrega la clave
calorias["manzana"] = 50        # MODIFICA el valor existente

print(calorias)                 # {'manzana': 50, 'pera': 57}

# update(): agrega/varios en una sola llamada
calorias.update({"uva": 69, "kiwi": 61})
print(calorias)
```

### Borrar elementos: `pop`, `popitem` y `del`

```python
calorias = {"manzana": 52, "pera": 57, "uva": 69}

del calorias["pera"]            # borra por clave, sin devolver nada
x = calorias.pop("manzana")     # borra y DEVUELVE el valor
print(x)                        # 52

y = calorias.popitem()          # borra y devuelve (clave, valor) del final
print(y)                        # ('uva', 69)  -> en versiones viejas era aleatorio
print(calorias)                 # {}
```

- `pop(clave)` devuelve el valor y **lanza `KeyError`** si la clave no existe (a diferencia de `get`).
- `popitem()` quita el último par insertado y lo devuelve como tupla.

### Vistas dinámicas: `keys()`, `values()` y `items()`

Para recorrer un diccionario usás sus **vistas** (`keys()`, `values()`, `items()`), que son la versión moderna de las tres maneras de mirar un dict:

```python
calorias = {"manzana": 52, "pera": 57}

for fruta in calorias:                    # equivale a calorias.keys()
    print(fruta)

for valor in calorias.values():
    print(valor)

for fruta, valor in calorias.items():     # cada item es (clave, valor)
    print(f"{fruta}: {valor} calorías")
```

Resultado:

```
manzana
pera
52
57
manzana: 52 calorías
pera: 57 calorías
```

> **Dato clave:** `keys()`, `values()` y `items()` devuelven **vistas dinámicas**, no copias. Si modificás el diccionario, la vista "ve" el cambio. Y la **sintaxis `for fruta, valor in calorias.items()`** es el patrón de recorrido más usado en todo Python: combinás el `for` del capítulo 4 con el *unpacking* de tuplas del capítulo actual.

### El caso clásico: contar con un diccionario

El ejemplo que justifica la existencia de `get()`:

```python
texto = "anaconda"
conteo = {}

for letra in texto:
    # sin get: habría que hacer if letra in conteo: ... else: ...
    # con get: una sola expresión
    conteo[letra] = conteo.get(letra, 0) + 1

print(conteo)          # {'a': 3, 'n': 2, 'c': 1, 'o': 1, 'd': 1}
```

Lee así: "sumale 1 al valor de `letra`, y si no existe todavía usá `0` como punto de partida". Con la lista habrías tenido que preguntar "¿está?" con un `if`; acá la pregunta es una sola expresión.

### ¿Diccionario o lista?

Si buscás "el elemento que tiene tal **nombre**", el diccionario gana siempre: la búsqueda por clave es **directa** (Python calcula dónde está), sin recorrer todo. Con la misma intención, la lista te obligaría a buscar uno por uno.

| Necesitás…            | Estructura          |
|-----------------------|---------------------|
| una colección ordenada que cambia y recorrés en orden | `list`  |
| un valor que no cambia y se "abre" en variables        | `tuple` |
| asociar un nombre/clave con un valor y buscarlo rápido | `dict`  |

---

## 6. Conjuntos y `frozenset`

El **conjunto** (`set`) es una colección **desordenada, mutable y sin elementos repetidos**. Solo admite elementos inmutables (números, cadenas, tuplas). Se escribe entre **llaves** `{...}` —pero cuidado con la confusión clásica:

- `{}` crea un **diccionario** vacío (no un conjunto).
- Para un conjunto vacío se usa `set()`.

```python
letras = {"a", "b", "c", "a", "b"}   # los duplicados se descartan solos
print(letras)                        # {'c', 'b', 'a'} (orden no garantizado)
print(len(letras))                   # 3
```

### Para qué sirve un conjunto

Dos usos principales, y ambos vienen solos de la definición:

**1. Eliminar duplicados** de cualquier colección que se pueda recorrer:

```python
numeros = [1, 2, 2, 3, 3, 3, 4]
unicos = set(numeros)
print(unicos)              # {1, 2, 3, 4}
print(list(unicos))        # [1, 2, 3, 4]
```

**2. Probar pertenencia a gran velocidad**: preguntar `x in conjunto` es instantáneo (por hashing, igual que las claves de un dict), sin importar cuán grande sea el conjunto.

### Modificar un conjunto

```python
letras = {"a", "b", "c"}
letras.add("d")            # agrega un elemento
letras.discard("b")        # elimina si existe; NO error si no existe
letras.remove("a")         # elimina; SÍ lanza KeyError si no existe
print(letras)              # {'c', 'd'}
```

- `add(x)` agrega. 
- `discard(x)` elimina sin reclamar. 
- `remove(x)` elimina y falla si no está.

### Operaciones de conjuntos

Los conjuntos de Python implementan la **teoría de conjuntos** de siempre: unión, intersección, diferencia y diferencia simétrica, con operadores o con métodos:

| Operador   | Operación              | Método equivalente     |
|------------|------------------------|------------------------|
| `\|`       | unión                  | `union()`              |
| `&`        | intersección           | `intersection()`       |
| `-`        | diferencia             | `difference()`         |
| `^`        | diferencia simétrica   | `symmetric_difference()` |
| `<=`       | subconjunto            | `issubset()`           |

```python
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

print(a | b)   # unión:             {1, 2, 3, 4, 5, 6}
print(a & b)   # intersección:      {3, 4}
print(a - b)   # diferencia:        {1, 2}
print(a ^ b)   # dif. simétrica:    {1, 2, 5, 6}
print({1, 2} <= a)   # True, {1,2} es subconjunto de a
```

### `frozenset`: el conjunto inmutable

El **`frozenset`** es un conjunto **inmutable**: se crea una vez y no se puede modificar. Como es inmutable, es "hashable" y —igual que la tupla frente a la lista— sirve de **clave de diccionario** y de **elemento de otro conjunto**.

```python
claves_legales = frozenset(["leer", "escribir", "ejecutar"])
print(claves_legales)           # frozenset({'escribir', 'leer', 'ejecutar'})

# frozenset es la versión inmutable de set, igual que tuple lo es de list
```

> **Dato clave:** el par **tupla/lista** y el par **`frozenset`/`set`** siguen exactamente el mismo patrón: la versión **inmutable** (tuple, frozenset) se puede usar donde se necesita algo que "no cambie" — como clave de diccionario — y la **mutable** (list, set) es la de uso general.

| Estructura | Orden | Mutable | Duplicados | ¿Clave de dict? |
|------------|-------|---------|------------|-----------------|
| `list`     | sí    | sí      | sí         | no              |
| `tuple`    | sí    | no      | sí         | sí              |
| `set`      | no    | sí      | no         | no              |
| `frozenset`| no    | no      | no         | sí              |

---

## 7. `match`/`case` con estructuras: la coincidencia de patrones estructurales

Ya tenés las seis estructuras y sabés cómo se comportan. Ahora vas a ver una herramienta que las aprovecha de forma espectacular: el **`match`/`case`** del capítulo 4. Lo que viste ahí —comparar contra literales, `|` y comodín `_`— era apenas la puerta de entrada. Desde Python 3.10, `match`/`case` puede **deconstruir** listas, tuplas y diccionarios: no solo comparar un valor, sino *desarmarlo* y ligar sus partes a variables en un solo paso. Eso se llama **coincidencia de patrones estructurales** (*structural pattern matching*), y con estructuras es donde muestra su verdadero poder.

### 7.1 Patrones de mapeo: deconstruir diccionarios

El patrón de **mapeo** es un diccionario con *variables donde van los valores*: Python intenta "encajar" tu dict contra el patrón y liga cada clave a su variable.

```python
emoji = {"nombre": "emiliano", "edad": 47, "pais": "Argentina"}

match emoji:
    case {"nombre": nombre, "edad": edad}:
        print(nombre, edad)
```

> **Dato clave:** los patrones de **mapeo** son *parciales*: `case {"nombre": nombre}` coincide aunque el diccionario tenga **más claves** de las que figuran en el patrón. Con `==` eso sería `False`, pero `match` no compara por igualdad: pregunta "¿tiene estas claves?" y liga el resto.

Y como los dicts se anidan (viste listas de dicts en la sección 5), los patrones también lo hacen:

```python
persona = {
    "nombre": "belen",
    "edad": 42,
    "direccion": {"calle": "Rivadavia 11111", "codigo": 1142},
}

match persona:
    case {"nombre": nombre, "edad": edad, "direccion": {"calle": direccion}}:
        pass
    case {"nombre": nombre, "edad": edad}:
        direccion = "Sin domicilio"
    case _:
        nombre, edad, direccion = "NN", "sin datos", "sin datos"

print(f"{nombre} con {edad} años vive en {direccion}")
```

Mirá el orden: el **patrón más específico primero** (con dirección), luego el general, y al final el comodín `_`. Es exactamente el principio del capítulo 4: se evalúan en orden y gana el **primero** que coincida.

### 7.2 Patrones de secuencia: deconstruir listas y tuplas

El patrón de **secuencia** encaja listas y tuplas y liga cada posición a una variable:

```python
coordenada = (3, 5)

match coordenada:
    case (x, y):
        print(f"en ({x}, {y})")
```

> **Dato clave:** a diferencia de los mapeos, los patrones de **secuencia exigen la longitud exacta**: `(x, y)` no coincide con una tupla de 3. Para "el resto" se usa la versión estructural de `*resto`: `case [primero, *resto]:`. La variable `resto` recibe una lista con todos los elementos que quedan.

El encadenamiento completo —vacía, un elemento, varios— ordenado de más específico a más general:

```python
cesta = ["manzanas", "peras", "naranjas"]

match cesta:
    case []:
        descripcion = "cesta vacía"
    case [unico]:
        descripcion = f"solo {unico}"
    case [primero, *resto]:
        descripcion = f"{primero} y {len(resto)} más"
    case _:
        descripcion = "esto ni siquiera es una lista"

print(descripcion)  # manzanas y 2 más
```

El código se lee de arriba hacia abajo. Primero se guarda una lista en `cesta`; después, `match` revisa su forma y asigna el texto correspondiente a `descripcion`. Al final se imprime esa variable. Probá reemplazar el valor de `cesta` por `[]`, `["peras"]` o `"peras"` para observar las demás ramas.

### 7.3 Patrones OR, AS y guardas: el clásico "a cenar"

Ahora sí, la joya. Combinando patrón de secuencia + **OR** (`|`) + **AS** (`as`) en un solo `case`, podés expresar reglas que con `if`/`elif` ocupan el doble de código:

```python
lista = ["empanadas", "helado"]

match lista:
    case [("pizza" | "empanadas" as cena), postre]:
        bebida = "Cerveza"
        mensaje = f"Vamos a cenar {cena}, tomar {bebida} y comer {postre} de postre"
    case [("Asado" | "Fideos" as cena), postre]:
        bebida = "vino"
        mensaje = f"Vamos a cenar {cena}, tomar {bebida} y comer {postre} de postre"
    case [cena, postre]:
        bebida = "Agua"
        mensaje = f"Vamos a cenar {cena}, tomar {bebida} y comer {postre} de postre"
    case [cena]:
        bebida = "Agua"
        postre = "Nada"
        mensaje = f"Vamos a cenar {cena}, tomar {bebida} y de postre: {postre}"
    case []:
        mensaje = "No se cena"
    case _:
        mensaje = "Formato de menú no reconocido"

print(mensaje)
```

Acá no se define ninguna función propia: `lista` contiene el menú que se quiere analizar y cada `case` prepara directamente el mensaje. Cambiá su contenido por `["Asado", "Flan"]`, `["verduras", "torta"]`, `["empanadas"]` o `[]` para recorrer los otros casos.

Desarmá el patrón de la primera línea:

- `[cena, postre]` — patrón de secuencia: exactamente dos elementos.
- `("pizza" | "empanadas" as cena)` — **OR**: coincide si el primer elemento es "pizza" *o* "empanadas"; **AS**: liga el valor capturado a la variable `cena`.
- El caso general `[cena, postre]` atrapa cualquier otra pareja (le asignás Agua).

Y para exigir una *condición extra*, se agrega la **guarda** `if` al final del `case` (misma idea que en el capítulo 4):

```python
lista = ["empanadas", "helado"]

match lista:
    case [("pizza" | "empanadas" as cena), postre] if postre in ["helado", "flan"]:
        bebida = "Cerveza"
        mensaje = f"{cena}, {bebida} y {postre}"
    case [("pizza" | "empanadas" as cena), postre]:
        bebida = "Gaseosa"
        mensaje = f"{cena}, {bebida} y {postre}"
    case [("Asado" | "Fideos" as cena), postre]:
        bebida = "vino"
        mensaje = f"{cena}, {bebida} y {postre}"
    case []:
        mensaje = "No se cena"
    case _:
        mensaje = "Formato de menú no reconocido"

print(mensaje)  # empanadas, Cerveza y helado
```

Si cambiás `helado` por `fruta`, la guarda de la primera rama resulta falsa. Entonces Python continúa con el siguiente `case`, que también reconoce las empanadas pero elige `Gaseosa`.

> **Dato clave:** la **guarda** no captura el caso: lo *descarta* si la condición falla, y el `match` sigue probando los patrones siguientes. Por eso el patrón con guarda va arriba y su "plan B" justo abajo.

### 7.4 Patrones de clase: exigir tipo

Con la sintaxis `Tipo(variable)` el `case` exige que el valor **sea de ese tipo** (equivale a un `isinstance`) y liga la variable. Es el control de tipo integrado al patrón:

```python
datos = (10, 4)

match datos:
    case (int(a), int(b)) if b != 0:
        resultado = a / b
        print(resultado)  # 2.5
    case _:
        print(f"Se esperaba (int, int) con el segundo valor distinto de 0, no {datos!r}")
```

> **¿Qué significa `!r`?** Dentro de una f-string, `!r` pide mostrar la **representación técnica** del valor. Esto ayuda a distinguir los tipos de datos y a reconocer su estructura; por ejemplo, conserva las comillas de las cadenas. Así, `{datos!r}` puede mostrar `('a', 2)` y deja visible que `a` es texto. Se usa especialmente en mensajes de diagnóstico. Por ahora alcanza con recordar que `{variable!r}` muestra el valor de una manera más precisa que `{variable}`.

El primer caso solo se ejecuta si `datos` contiene exactamente dos enteros y el segundo no es cero. Probá con `("a", 2)` o `(10, 0)`: ninguno cumple el patrón completo y ambos llegan al comodín `_`.

> **Importante:** `int(x)` en un patrón **no convierte** nada: **exige** que el valor sea `int` y recién ahí liga `x`. No es `int(value)` de conversión. Es `str(x)`, `int(x)`, `float(x)`, `list(x)`… con el *check de tipo* incluido.

Como regla práctica: cuando tus datos son dicts o listas que pueden tomar **formas distintas**, `match`/`case` reemplaza a la cadena de `if`/`elif` que pregunta el tipo o la longitud. Lo vas a usar mucho en la Parte IV, cuando veas errores y validación de datos.

---

## 8. Comprensiones: listas y diccionarios

Llegamos al plato fuerte. Una **comprensión de lista** (*list comprehension*, "lista por comprensión") es una forma de construir una **nueva lista a partir de otra colección en una sola línea**, sin `for` explícito ni `append`. Es una de las señas de identidad del estilo Python: código corto, legible y expresivo.

### La forma básica: transformar

Compará el camino "clásico" con la comprensión:

```python
# camino clásico
cuadrados = []
for n in range(10):
    cuadrados.append(n ** 2)

# con comprension: [expresión for variable in iterable]
cuadrados = [n ** 2 for n in range(10)]
```

Leelo así: "para cada `n` en `range(10)`, **poné** `n ** 2` en la lista". La comprensión siempre va entre **corchetes** y empieza por la **expresión** (el valor que va a cada elemento).

```python
dobles = [n * 2 for n in range(11)]              # [0, 2, 4, ..., 20]
iniciales = [nombre[0] for nombre in ["juan", "ana", "luca"]]   # ['j', 'a', 'l']
mayusculas = [n.upper() for n in ["hola", "mundo"]]            # ['HOLA', 'MUNDO']
```

> **Importante:** la comprensión **crea una lista nueva**; no modifica la original. Si tu intención es solo *recorrer* (por ejemplo, imprimir), seguí usando `for`. La comprensión existe para **construir**.

### Con filtro: `if` al final

Cuando le agregás un `if` al final, la comprensión **selecciona** cuáles entran:

```python
pares = [n for n in range(20) if n % 2 == 0]
print(pares)          # [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

nombres = "Juan Pedro Pablo Lucas Mateo Andres Felipe Bartolome Tomas Matias".split()
largos = [n for n in nombres if len(n) >= 6]
print(largos)         # ['Andres', 'Felipe', 'Bartolome', 'Matias']
```

El orden es fijo: **expresión, `for`, `if`**. "Para cada `n`…, si `n` es par…”

### Con `if`/`else`: transformar con alternativa

Si querés elegir **qué valor poner** según una condición, el condicional ternario va en la **expresión** (antes del `for`):

```python
tipo = ["par" if n % 2 == 0 else "impar" for n in range(6)]
print(tipo)           # ['par', 'impar', 'par', 'impar', 'par', 'impar']

eleccion = ["pan" if n % 2 == 0 else "queso" for n in range(6)]
print(eleccion)       # ['pan', 'queso', 'pan', 'queso', 'pan', 'queso']
```

> **Dato clave:** nunca mezcles los dos `if`. Uno está **antes** del `for` (en la expresión, forma ternaria `x if cond else y`, elige *qué valor*) y el otro **después** del `for` (filtro sin `else`, decide *si entra*). Si los confundís, el código falla o filtra cuando querías transformar y al revés.

### Anidadas: dos `for`

Con dos `for` conseguís combinaciones, como un "producto cartesiano":

```python
pares = [(a, b) for a in range(3) for b in range(3)]
print(pares)
# [(0,0), (0,1), (0,2), (1,0), (1,1), (1,2), (2,0), (2,1), (2,2)]

herramienta = "martillo destornillador espatula pinza".split()
colores = "rojo azul verde".split()

combinaciones = [h + " " + c for h in herramienta for c in colores]
print(combinaciones)
```

Leelo en el mismo orden que un `for` anidado: "para cada herramienta, para cada color, combinalos".

Y el caso súper útil: **aplanar** una lista de listas:

```python
matriz = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
plana = [x for fila in matriz for x in fila]
print(plana)          # [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### Comprensiones de diccionarios: `{clave: valor for …}`

La misma idea funciona con diccionarios, al estilo de la sección 5. La sintaxis cambia: **llaves** y un par clave: valor como "expresión":

```python
cuadrados = {n: n ** 2 for n in range(5)}
print(cuadrados)       # {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}
```

Un uso clásico: **invertir** un diccionario (claves pasan a ser valores y al revés):

```python
calorias = {"manzana": 52, "pera": 57}
invertido = {valor: fruta for fruta, valor in calorias.items()}
print(invertido)       # {52: 'manzana', 57: 'pera'}
```

Y la combinación con listas de diccionarios (datos "en forma de tabla" guardados como una lista de registros):

```python
personas = [
    {"nombre": "emiliano", "edad": 45},
    {"nombre": "luca", "edad": 10},
    {"nombre": "belen", "edad": 40},
]

nombres_mayores = [p["nombre"] for p in personas if p["edad"] > 18]
print(nombres_mayores)      # ['emiliano', 'belen']

edades_por_nombre = {p["nombre"]: p["edad"] for p in personas}
print(edades_por_nombre)    # {'emiliano': 45, 'luca': 10, 'belen': 40}
```

> **Buenas prácticas:** la comprensión de diccionarios de la última línea es un patrón que vas a ver en código real todo el tiempo: "convierte esta lista de registros en un dict para buscarlos por nombre". Ya tenés la segunda herramienta de búsqueda rápida del capítulo.

### Conjuntos por comprensión

El mismo esquema sirve para conjuntos — el resultado queda sin duplicados y desordenado:

```python
pares_set = {x for x in range(10) if x % 2 == 0}
print(pares_set)       # {0, 2, 4, 6, 8}
```

### El anticipo de los generadores

Cuidado con una sintaxis parecida: si en vez de `[...]` o `{...}` usás **paréntesis**, no es una lista: es una **expresión generadora** (*generator expression*):

```python
gen = (n ** 2 for n in range(10))
print(gen)             # <generator object ...> (no es una lista)
```

Un generador no construye la lista: **produce valores a demanda, uno a la vez** (*lazy*, "perezoso"), lo que ahorra memoria con colecciones enormes. Pero para *usarlos* con tranquilidad, en este capítulo alcanza con saber que existen: los vamos a estudiar formalmente en la **Parte V (funciones)**, junto con `yield`.

> **Dato clave:** por ahora, la regla para los paréntesis es no confundirlos: no es "una lista con paréntesis", es **otra cosa** (un generador). Si querés una lista, usá corchetes. El apartado "Comparación de memoria: lista vs generador" te espera en el capítulo de funciones.

---

## 9. `iter()` y `next()`: consumir de a uno

En el capítulo 4 prometimos que acá lo íbamos a desarrollar: llegó la hora. ¿Te acordás de la simulación de la entrada?

```python
simulacion = ["a", "b", "q"]
iter_sim = iter(simulacion)
entrada = next(iter_sim)
```

Ahí había una pareja de funciones que no te habíamos explicado. Tomátelas como una **cinta transportadora de supermercado**: una colección es un montón de productos en el estante, y el **iterador** es la cinta que te los va llevando de a uno hasta el mostrador.

**`iter(coleccion)`** fabrica el iterador: un "recorredor" que se acuerda por dónde va y qué le falta entregar. **`next(iterador)`** le pide el siguiente producto y lo devuelve. Cada llamada a `next` hace avanzar la cinta un paso:

```python
compras = ["pan", "leche", "queso"]
cinta = iter(compras)        # armamos la cinta transportadora

print(next(cinta))   # pan
print(next(cinta))   # leche
print(next(cinta))   # queso
```

Y ojo: el `gen` de la sección anterior ("el anticipo de los generadores") también es un recorredor. Podés pedirle valores de a uno: `next(gen)` te va dando los cuadrados sin construir la lista. Es la misma cinta, otro producto.

¿Qué pasa si pedís más productos de los que hay? Aparece una excepción llamada **`StopIteration`** — la señal que dice "che, la cinta se terminó":

```python
dulce = iter(["alfajor"])
print(next(dulce))   # alfajor
print(next(dulce))   # StopIteration: ya no queda nada para entregar
```

> **Dato clave:** no te asustes por el nombre raro: `StopIteration` es una **excepción**, del mismo clima que las que vas a aprender a manejar con `try`/`except` en el capítulo 6. De hecho, el `for` usa esto por debajo: pide con `next` hasta que llega `StopIteration` y termina el bucle. Por eso un `for` finaliza solo, aunque vos nunca veas esa excepción: la ve el bucle.

El iterador es **descartable**: no se reinicia. Si lo agotás y querés volver a recorrer la colección, tenés que fabricar otro con `iter()`. Fijate:

```python
compras = ["pan", "leche", "queso"]
cinta = iter(compras)

print(next(cinta))          # pan

print(list(cinta))          # ['leche', 'queso']  → sigue donde quedó, no vuelve atrás
print(list(cinta))          # []  → descartable: ya no da más nada
```

> **Dato clave:** un **iterador es de un solo uso**. Si lo pasás a `list()` o a un `for` completo, se consume; para recorrer de nuevo necesitás un `iter()` fresco. La colección original, eso sí, no se toca: `compras` sigue intacta con sus tres productos. Quien avanza es la cinta, no el estante.

Todo lo que sabés recorrer con un `for` sabe fabricar su iterador: las listas, las tuplas, las cadenas (carácter por carácter), los diccionarios (sus **claves**) y los conjuntos. Es la misma maquinaria, siempre.

¿Y esto para qué te sirve, más allá de entender el `for`? Primero, te hace leer mejor el código: vas a encontrarte `next(...)` en librerías por todos lados. Segundo, es la base de las herramientas de la **próxima sección**: el `itertools.count()` infinito que sigue se consume con `next()`. Y tercero, es la puerta de entrada a algo grande: cuando llegues a la **Parte VI (POO)**, vas a poder fabricar **tus propios objetos iterables** definiendo `__iter__()` y `__next__()` — objetos que se puedan recorrer con un `for`.

---

## 10. La biblioteca estándar: `collections` e `itertools`

Python trae estas estructuras listas para usar, pero encima tiene **bibliotecas de la biblioteca estándar** (las que vienen instaladas con el lenguaje, sin instalar nada) que las potencian. Dos que no te pueden faltar: **`collections`**, con variantes y herramientas para contenedores, e **`itertools`**, con herramientas para combinar y recorrer iterables. Solo hay que importarlas:

```python
from collections import Counter, defaultdict, namedtuple
import itertools
```

### `collections`: variantes listas para usar

**`Counter`** cuenta automáticamente elementos de cualquier colección (el ejemplo "contar letras" de la sección 5, resuelto en una línea):

```python
from collections import Counter

conteo = Counter("anaconda")
print(conteo)               # Counter({'a': 3, 'n': 2, 'c': 1, 'o': 1, 'd': 1})
print(conteo["a"])          # 3
print(conteo.most_common(2))  # los 2 más frecuentes: [('a', 3), ('n', 2)]
```

**`defaultdict`** es un diccionario que, ante una clave que falta, la **crea automáticamente** con un valor inicial. Es la evolución del patrón `conteo[letra] = conteo.get(letra, 0) + 1`:

```python
from collections import defaultdict

conteo = defaultdict(int)          # las claves nuevas nacen valiendo 0
for letra in "cocodrilo":
    conteo[letra] += 1
print(dict(conteo))                # {'c': 2, 'o': 3, 'd': 1, 'r': 1, 'i': 1, 'l': 1}

agrupado = defaultdict(list)       # las claves nuevas nacen como lista vacía
agrupado["pares"].append(2)
print(dict(agrupado))              # {'pares': [2]}
```

> **Dato clave:** la única diferencia con un `dict` común es el argumento que recibe al crearse: `defaultdict(int)`, `defaultdict(list)`, `defaultdict(float)`. Ese "valor alemán" (default factory) se usa para inicializar cualquier clave que no exista todavía.

**`namedtuple`** crea tuplas con **campos con nombre**: la inmutabilidad de la tupla, pero accedes por `.nombre` en vez de `[0]`:

```python
from collections import namedtuple

Punto = namedtuple("Punto", ["x", "y"])
p = Punto(3, 5)

print(p.x, p.y)      # 3 5
print(p[0])          # 3   (sigue siendo indexable como tupla)
x, y = p             # 3 5 (sigue "desempaquetando")
```

Y `deque` ("double-ended queue", "cola de doble extremo") es una lista optimizada para agregar/quitar **en ambos extremos**. Pero antes de verlo, una aclaración importante:

**Pilas (stacks) y colas (queues)**

Una **pila** (stack) es LIFO ("último en entrar, primero en salir", como un plato en una pila de platos). Una **cola** (queue) es FIFO ("primero en entrar, primero en salir", como esperar el pedido en la pizeria). Ambas son estructuras fundamentales que veremos en detalle en **Estructuras de Datos Avanzadas**. Por ahora, mostremos cómo implementarlas con `list`:

```python
# PILA con list: append para agregar, pop para sacar
pila = []
pila.append(1)      # agregar al final
pila.append(2)
pila.append(3)
print(pila.pop())   # 3  <- sale el último (LIFO)
print(pila.pop())   # 2
print(pila)         # [1]

# COLA con list: append para agregar, pop(0) para sacar del principio
cola = []
cola.append(1)      # agregar al final
cola.append(2)
cola.append(3)
print(cola.pop(0))  # 1  <- sale el primero (FIFO)
print(cola.pop(0))  # 2
print(cola)         # [3]
```

El problema: operaciones en el principio de una lista son **lentas** con muchos elementos. Tanto `pop(0)` como `insert(0, elemento)` tienen que recorrer y reacomodar todos los elementos. Por eso existe `deque`:

> **Adelanto:** ¿Por qué es lento? Lo veremos en detalle en el capítulo de **Big-O y Complejidad Temporal**: `pop(0)` e `insert(0)` son operaciones **O(n)**, mientras que `append()` y `pop()` al final son **O(1)**. Por eso `deque`, que es O(1) en ambos extremos, es la solución correcta para colas.

```python
from collections import deque

# deque es eficiente en AMBOS extremos
cola = deque([1, 2, 3])
cola.appendleft(0)        # agrega al principio (eficiente)
cola.append(4)            # agrega al final
print(cola)               # deque([0, 1, 2, 3, 4])
print(cola.popleft())     # 0  (saca del principio, rápido)
print(cola.pop())         # 4  (saca del final, rápido)
print(cola)               # deque([1, 2, 3])
```

> **Dato clave:** `deque` está optimizado para agregar/quitar en ambos extremos sin coste. Con `list`, si usas `pop(0)` frecuentemente, es lento. Para pilas, `list` es suficiente (solo usas `append` y `pop`); para colas, `deque` es la opción correcta.

> **Nota:** pilas y colas son tan importantes que en **Estructuras de Datos Avanzadas** tendrás capítulos completos sobre ellas, cómo implementarlas en diferentes contextos y cuándo elegir cada una.

### `itertools`: combinar y generar secuencias

**`chain`** encadena varios iterables como si fueran uno solo (tu "aplanar lista de listas" de la sección 8, sin comprensión):

```python
import itertools

plana = list(itertools.chain([1, 2], [3, 4], [5]))
print(plana)             # [1, 2, 3, 4, 5]
```

**`product`** genera el producto cartesiano (todas las combinaciones), igual que la comprensión anidada:

```python
combos = list(itertools.product([0, 1], ["a", "b"]))
print(combos)            # [(0, 'a'), (0, 'b'), (1, 'a'), (1, 'b')]
```

**`combinations`** y **`permutations`** generan agrupaciones sin y con orden:

```python
import itertools

letras = ["a", "b", "c"]

print(list(itertools.combinations(letras, 2)))   # [(a,b),(a,c),(b,c)] sin repetir orden
print(list(itertools.permutations(letras, 2)))   # orden importa: 6 combinaciones
```

**`count`** genera números **infinitos** a partir de un inicio con un paso:

```python
import itertools

pares = itertools.count(0, 2)     # 0, 2, 4, 6, ... sin fin
# nunca lo conviertas a lista completa: no termina
print(next(pares), next(pares), next(pares))   # 0 2 4
```

**Otras funciones útiles de `itertools`**

Hay muchas más en la librería. Acá están las que más aportan a un principiante:

- **`repeat(x, n)`** — repite un valor `n` veces (o infinitas si omitís `n`):
```python
repetido = list(itertools.repeat("a", 3))
print(repetido)    # ['a', 'a', 'a']
```

- **`cycle(iterable)`** — cicla sobre un iterable **infinitamente**:
```python
colores = itertools.cycle(["rojo", "verde", "azul"])
print(next(colores), next(colores), next(colores), next(colores))  # rojo verde azul rojo
```

- **`islice(iterable, inicio, fin, paso)`** — "corta" un iterable como si fuera un slice `[inicio:fin:paso]`, sin convertir a lista:
```python
numeros = itertools.count(0)  # infinito: 0, 1, 2, 3, ...
primeros_cinco = list(itertools.islice(numeros, 5))
print(primeros_cinco)         # [0, 1, 2, 3, 4]

cada_dos = list(itertools.islice(numeros, 0, 10, 2))
print(cada_dos)               # [0, 2, 4, 6, 8]
```

- **`zip_longest(*iterables, fillvalue=x)`** — como `zip()` pero rellenando con un valor si los iterables tienen largo distinto:
```python
nombres = ["ana", "juan"]
edades = [25, 30, 35]  # uno más

# zip normal deja fuera el último
print(list(zip(nombres, edades)))              # [('ana', 25), ('juan', 30)]

# zip_longest rellena el faltante
print(list(itertools.zip_longest(nombres, edades, fillvalue="?")))
# [('ana', 25), ('juan', 30), ('?', 35)]
```

- **`groupby(iterable, key=func)`** — agrupa elementos consecutivos iguales (o según una función):
```python
datos = [1, 1, 2, 2, 2, 3, 1, 1]
for clave, grupo in itertools.groupby(datos):
    print(clave, list(grupo))
# 1 [1, 1]
# 2 [2, 2, 2]
# 3 [3]
# 1 [1, 1]
```

Y hay más: `compress`, `dropwhile`, `takewhile`, `filterfalse`, `accumulate`, `starmap`, `tee`, `pairwise`, `combinations_with_replacement`. Explóralas en la documentación cuando las necesites.

> **Importante:** ojo con `itertools.count()`, `cycle()` y `repeat()` sin límite: son **infinitos**. No los pases a `list()` ni a un `for` sin freno, o el programa no termina. Se usan con `next()`, `islice()` o con comprensiones y `break`.

---

## 11. ¿Cuál usar cuándo?

Cerró el recorrido por las seis estructuras (y por el mecanismo de los iteradores). Acá está la **guía de decisión** completa, ordenada como pregunta:

1. **¿Buscás por nombre/clave?** → **`dict`** (o `defaultdict` de `collections` si las claves nuevas tienen un valor inicial automático).
2. **¿Necesitás un orden estable y vas a modificar la colección?** → **`list`**.
3. **¿Necesitás orden pero el valor es fijo / de solo lectura?** → **`tuple`** (además: sirve de clave de dict y de *unpacking* elegante).
4. **¿Solo te importa "qué elementos hay" (sin duplicados, sin orden)?** → **`set`** para pertenencia veloz y eliminar duplicados.
5. **¿Necesitás un conjunto que no cambie (para que sea clave de dict o elemento de otro set)?** → **`frozenset`**.
6. **¿Querés construir una colección nueva transformando/filtrando?** → **comprensión** (lista, dict o set), la sección 8.
7. **¿Necesitás distinguir las *formas* que toman tus datos** (un dict con ciertas claves, una lista de uno o dos elementos, un tipo concreto)? → **`match`/`case`** con patrones de mapeo y secuencia (sección 7).
8. **¿Necesitás contar o agrupar con configuraciones puntuales?** → **`collections`**; ¿combinar o recorrer inteligentemente? → **`itertools`**.
9. **¿Querés recorrer un iterable a mano, de a un paso?** → **`iter()`**/`next()` (sección 9).

La receta mental del capítulo entero:

| Situación                      | Estructura                    |
|--------------------------------|-------------------------------|
| Lista de compras que cambia    | `list`                        |
| Coordenadas que nunca cambian  | `tuple`                       |
| "Dado este nombre, dame su dato" | `dict`                      |
| "¿Está este elemento?" sin duplicados | `set` (o `frozenset` inmutable) |
| Construir una lista/dict nuevo en una línea | comprensión |
| Contar frecuencias, agrupar   | `collections.Counter` / `defaultdict` |
| Combinar iterables, productos cartesianos | `itertools.product` / `combinations` |
| Consumir un iterable de a un paso | `iter()` / `next()` |
| Distinguir las *formas* de tus datos (dict con ciertas claves, listas de distinto largo) | `match`/`case` estructural |

---

## 12. Resumen y conceptos clave

Este capítulo te dio el **mapa** completo: tres criterios (orden e índice, mutabilidad, unicidad) y seis estructuras que son combinaciones de esos criterios. Aprendiste que una secuencia se recorre por posición, que mutabilidad separa lista de tupla, que el diccionario es un mapeo clave → valor (mutable, con claves únicas e inmutables), y que el conjunto es un "saco" sin duplicados ni orden, con su versión inmutable `frozenset`. Sobre esa base, descubriste la herramienta más elegante del estilo Python —las comprensiones de listas, diccionarios y conjuntos—, viste cómo `match`/`case` deconstruye estructuras en patrones, descubriste que detrás de todo `for` hay un **iterador** — `iter()`/`next()` te dan control manual sobre esa cinta — y miraste de reojo la biblioteca estándar que potencia todo: `collections` e `itertools`. Los generadores, anunciados con los paréntesis `(x for x in …)`, quedaron formalmente para la Parte V, cuando ya sepas funciones.

Repasa con esta lista y asegúrate de que cada punto te resulta familiar antes de continuar:

- [ ] Las estructuras se clasifican por **tres criterios**: orden e índice, mutabilidad, unicidad.
- [ ] **Secuencias** (`str`, `list`, `tuple`): ordenadas e indexables, con `[i]`, slicing `[inicio:fin:paso]`, `len`, `in`, `+`, `*`, iteración.
- [ ] **`list`** es la secuencia **mutable**: `append`, `extend`, `insert`, `remove`, `pop`, `del`, `sort`, `reverse`, `index`, `count`.
- [ ] **`sort()` y `sorted()`**: `sort()` modifica en lugar con opciones `key=` y `reverse=`; `sorted()` devuelve una nueva lista sin modificar.
- [ ] **`index()`**: busca la posición con parámetros opcionales `index(x, inicio, fin)` para limitar la búsqueda.
- [ ] Slicing devuelve una **copia** (`lista[:]` copia completa); modificarla no toca la original.
- [ ] **`slice(inicio, fin, paso)`**: crear un objeto slice reutilizable que funciona con cualquier secuencia.
- [ ] **`tuple`** es la secuencia **inmutable**; la define la **coma**, no los paréntesis (`(1,)` es tupla, `(1)` es número).
- [ ] **Métodos de tuplas**: solo `index()` y `count()` (por ser inmutables); operaciones `+`, `*` y slicing crean **tuplas nuevas**.
- [ ] **Operaciones con tuplas**: `tupla = tupla + (x,)` crea una tupla nueva y la guarda en la variable (la original no cambió).
- [ ] **Packing/unpacking**: `lat, lon = coordenadas`; el resto se captura con `*resto`.
- [ ] **`dict`** asocia claves únicas e inmutables a valores; mutable y por clave, no por índice.
- [ ] **Claves "hashables"**: deben ser **inmutables** (números, strings, tuplas); NO listas, sets ni dicts. Para claves compuestas, usar tuplas.
- [ ] Tres formas de crear un dict: llaves `{}`, `dict(nombre=…)`, `dict(iterable_de_pares)`.
- [ ] **`get()` vs `setdefault()`**: `get(clave, default)` solo **lee** sin modificar; `setdefault(clave, valor)` **agrega la clave si falta**. Usar `get()` para contar elementos.
- [ ] Acceso seguro: `if clave in dict` es lo clásico; `get()` es la forma moderna y expresiva; `setdefault()` cuando se quiere garantizar que la clave exista; `update` agrega/actualiza.
- [ ] Borrar: `pop(clave)` devuelve y puede lanzar `KeyError`; `popitem()` quita el último par; `del`.
- [ ] **Vistas dinámicas** `keys()`, `values()`, `items()`; patrón de recorrido `for clave, valor in d.items()`.
- [ ] **`set`** es desordenado, mutable y **sin duplicados**; `{}` es dict, el conjunto vacío es `set()`.
- [ ] Operaciones: unión `\|`, intersección `&`, diferencia `-`, diferencia simétrica `^`, subconjunto `<=`.
- [ ] **`frozenset`** es el set **inmutable** (igual que `tuple` es la versión inmutable de `list`): sirve de clave de dict.
- [ ] **Comprensiones**: `[expr for x in iter]`, `[expr for x in iter if cond]`, ternario en la expresión, dos `for` anidados.
- [ ] Comprensión de dict `{clave: valor for …}` y de set `{x for …}`; invertir un dict con `{v: k for k, v in d.items()}`.
- [ ] `(x for x in …)` es una **expresión generadora**, no una lista: los generadores se ven en la Parte V.
- [ ] **`iter(colección)`** fabrica un **iterador** (el "recorredor" que entrega de a uno) y **`next(iterador)`** pide el siguiente valor; agotado, lanza `StopIteration`. El `for` usa este mecanismo por dentro; tus propios iterables con `__iter__`/`__next__` te esperan en POO.
- [ ] `match`/`case` **estructural** deconstruye estructuras: los **patrones de mapeo** son *parciales* (no exigen todas las claves) y los de **secuencia** exigen longitud exacta (salvo `*resto`).
- [ ] Potencia con **OR** `|`, **AS** `as`, **guardas** `if`, y **patrones de clase** `int(v)`/`str(v)`/`float(v)` para exigir tipos.
- [ ] `collections`: `Counter`, `defaultdict(tipo)`, `namedtuple`, `deque` (optimizado para ambos extremos; mejor que `list` para colas).
- [ ] **Pilas y colas**: pila es LIFO (list + `append`/`pop` funciona bien), cola es FIFO (`deque` con `append`/`popleft()`; evitar `pop(0)` en listas porque es lento).
- [ ] `itertools`: `chain`, `product`, `combinations`/`permutations`, `count` (infinito); también `repeat`, `cycle`, `islice`, `zip_longest`, `groupby` y más.
- [ ] **Infinitos en itertools**: `count()`, `cycle()`, `repeat(x)` son **infinitos**. Usar con `next()`, `islice()` o un `break`, nunca convertir a `list()`.

## 13. Ejercicios

1. **Cuadrados con comprensión**: construí `[0, 1, 4, 9, …, 81]` con una sola expresión.
2. **Filtrar con comprensión**: de los números del 1 al 100, guardá solo los divisibles por 3 y menores a 40. ¿Y los divisibles por 7 o por 11?
3. **Diccionario por comprensión**: de la lista `frutas = ["manzana", "pera", "uva"]`, creá un dict `{fruta: len(fruta)}`.
4. **Invertir un dict**: con `invertido = {v: k for k, v in …}`, dale la vuelta a `{"a": 1, "b": 2}`.
5. **Mayores con comprensión**: con la lista de `personas` de la sección 8, obtené los nombres de los mayores de 18 y los menores de 18 en dos comprensiones.
6. **Contar con `Counter`**: usá `Counter` sobre `"cocodrilo"` y mostrá el elemento más frecuente con `most_common(1)`.
7. **Agrupar con `defaultdict`**: agrupá los números del 1 al 10 en "par" e "impar" con `defaultdict(list)`.
8. **Aplanar con `chain`**: aplaná `[[1, 2], [3], [4, 5]]` con `itertools.chain` y también con una comprensión anidada.
9. **Combinaciones**: con `itertools.combinations("abcd", 2)`, mostrá cuántas parejas distintas de letras hay.
10. **Conjuntos**: con `A = {1, 2, 3}` y `B = {2, 3, 4}`, calculá unión, intersección, diferencia y diferencia simétrica. ¿`{1, 2}` es subconjunto de `A`?
11. **`match`/`case` con estructuras**: guardá una lista en la variable `seleccion` y analizala con `match`: con dos elementos que empiecen con `"a"` debe guardar `"sube"` en `resultado`; con dos que empiecen con `"b"`, `"baja"`; con un solo elemento, `"pausa"`; y con cualquier otra cosa, `"sin datos"`. Al final, imprimí `resultado`. Pista: podés combinar guardas `if` y el patrón `[primero, segundo]`.
12. **`iter()`/`next()`**: con `cinta = iter(["pan", "leche", "queso"])`, mostrá los tres elementos con `next()`. Después imprimí `list(cinta)` y explicá por qué da `[]`. Por último, recorré de nuevo la misma lista con un `iter()` nuevo y confirmá que ahí sí vuelve a tener los tres productos.

```python
# Soluciones (no las mires antes de intentarlo)

# 1
cuadrados = [n ** 2 for n in range(10)]

# 2
mult3 = [n for n in range(1, 101) if n % 3 == 0 and n < 40]
mult7o11 = [n for n in range(1, 101) if n % 7 == 0 or n % 11 == 0]

# 3
largos = {fruta: len(fruta) for fruta in ["manzana", "pera", "uva"]}

# 4
original = {"a": 1, "b": 2}
invertido = {v: k for k, v in original.items()}

# 5
personas = [
    {"nombre": "emiliano", "edad": 45},
    {"nombre": "luca", "edad": 10},
    {"nombre": "belen", "edad": 40},
]
mayores = [p["nombre"] for p in personas if p["edad"] > 18]
menores = [p["nombre"] for p in personas if p["edad"] <= 18]

# 6
from collections import Counter
conteo = Counter("cocodrilo")
print(conteo.most_common(1))

# 7
from collections import defaultdict
grupos = defaultdict(list)
for n in range(1, 11):
    grupos["par" if n % 2 == 0 else "impar"].append(n)
print(dict(grupos))

# 8
import itertools
plana1 = list(itertools.chain([1, 2], [3], [4, 5]))
plana2 = [x for sub in [[1, 2], [3], [4, 5]] for x in sub]

# 9
import itertools
print(len(list(itertools.combinations("abcd", 2))))   # 6

# 10
A = {1, 2, 3}
B = {2, 3, 4}
print(A | B, A & B, A - B, A ^ B)
print({1, 2} <= A)

# 11
seleccion = ["auto", "avion"]

match seleccion:
    case [primero, segundo] if primero.startswith("a") and segundo.startswith("a"):
        resultado = "sube"
    case [primero, segundo] if primero.startswith("b") and segundo.startswith("b"):
        resultado = "baja"
    case [unico]:
        resultado = "pausa"
    case _:
        resultado = "sin datos"

print(resultado)  # sube

# 12
cinta = iter(["pan", "leche", "queso"])
print(next(cinta))      # pan
print(next(cinta))      # leche
print(next(cinta))      # queso
print(list(cinta))      # []  → el iterador quedó agotado

cinta2 = iter(["pan", "leche", "queso"])
print(list(cinta2))     # ['pan', 'leche', 'queso']  → iter() fresco volvió a recorrer
```

Con las seis estructuras —y tres herramientas que las potencian: **comprensiones**, **`match`/`case` estructural** e **`iter()`/`next()`**— ya tenés el material para modelar la mayoría de los problemas cotidianos de agrupación de datos. El siguiente paso natural es la **Parte IV**, donde vas a aprender a manejar los **errores** con `try`/`except` y las herramientas modernas del lenguaje (tipos y `match`/`case` al servicio de la validación). Y cuando llegues a la **Parte V**, el `match`/`case` con patrones se va a apoyar en todo lo que viste hoy.
