# Capítulo 20 — Arrays: la memoria al descubierto

Cuando en el capítulo 19 te prometí la Parte VIII, te dije que este capítulo arranca con los **arrays** — *el primo honesto de la lista nativa de Python, el bloque contiguo de memoria*. Hoy te toca conocerlo en persona, y en la primera parada hay una sorpresa que le toca la fibra a cualquier persona curiosa: en casi todos los idiomas de la industria, la palabra *array* (arreglo) nombra una pieza precisa de la máquina, con memoria pegada y tamaño fijo. En Python pasa algo distinto. Fijate la escena: un programador de C o Java habla de arrays, te muestra `int datos[] = {1, 2, 3, 4, 5}`, te dice que son cinco enteros pegados en la memoria, uno al lado del otro, y que agarrar el tercero es instantáneo. Vos lo escuchás y pensás: "sí, es como mi `[1, 2, 3, 4, 5]`". Y tenés razón a medias. Tu lista se parece al array por fuera, pero por dentro la historia es otra.

Este capítulo abre la Parte VIII del libro y te lleva a la estructura de datos más vieja del oficio — tan vieja que coincide con la memoria misma de la máquina. Vas a recorrer su historia en unos pocos párrafos, y te va a sorprender lo directamente que cae en el territorio de los otros idiomas; después vas a ver cómo la declaran C, Java, JavaScript — y de paso C# y Rust — para entender qué promete cada uno cuando dice "array". Recién ahí vas a conocer al primo honesto de tu lista: el módulo `array` de la biblioteca estándar, que empaqueta los números de verdad, uno al lado del otro. Y para cuando los números sean muchos y haya que hacerles cuentas, vas a cerrar con **NumPy**, la biblioteca que recorre esos bloques sin que tengas que recorrerlos vos. En el camino vas a descubrir que "mil millones de números" no es una frase: son **4 GB** en una sola pieza de memoria.

---

## 1. La historia: el array es la memoria misma

No hace falta salir de tu computadora para encontrar el primer array: está corriendo debajo de tus dedos. La memoria de la máquina — la RAM, la *memoria de acceso aleatorio* (*random access memory*, "aleatorio" porque cualquier celda se alcanza de inmediato por su dirección) — es literalmente un array gigante de celdas, cada una con un número de dirección, y el hardware sabe saltar a cualquier dirección en un solo movimiento. Toda la informática moderna se apoya en esa idea física: **un bloque de celdas, numeradas, alcanzables por índice**. Cuando un lenguaje te da "arrays", te está pasando la memoria cruda con una sintaxis cómoda encima.

La línea de tiempo es breve y sorprendente:

| Año | Acontecimiento | Qué aportó |
|-----|----------------|-----------|
| 1957 | **FORTRAN** (John Backus, IBM 704) | el primer lenguaje de alto nivel con una palabra para "estos números viven juntos": la sentencia `DIMENSION A(1000), B(1000)` |
| 1960 | **ALGOL 60** (el informe de la ACM) | formaliza el término **array** como concepto del lenguaje: declaración con límites (`array m[0:15]`), dimensiones y tipos |
| 1972 | **C** (Dennis Ritchie, Bell Labs) | el array como ciudadano humilde: un bloque de memoria al que se llega por un puntero, sin límites chequeados — toda la responsabilidad, tuya |
| 1995 | **Java** (James Gosling, Sun) | el array seguro: vive en el *heap* (montón de memoria) como objeto, con `length` y chequeo de límites en caliente |
| 1995 | **JavaScript** (Brendan Eich, Netscape) | el embustero: su "array" común es una lista dinámica de propiedades — no es un array. La verdad binaria recién llegó en 2015 con `Int32Array` y compañía |
| 2005–06 | **NumPy** (Travis Oliphant) en Python | el array que unió a la comunidad científica: en 2006 la versión 1.0 fusionó `Numeric` (de Jim Hugunin, 1995) y `Numarray` (del instituto científico STScI) en un solo motor numérico |

Mirá la tabla con los dos ojos. Con la historia: desde el `DIMENSION` de 1957 a NumPy pasaron menos de cincuenta años, y la idea que se mantuvo intacta es la misma de la RAM — *bloque contiguo + índice*. Con el lenguaje: cada idioma le puso un precio a esa idea. FORTRAN la ofreció con sintaxis matemática, C te la dejó pelada y peligrosa, Java la blindó con errores en caliente, y JavaScript demoró dos décadas en ofrecerla de verdad. Ese abanico de decisiones es exactamente lo que vas a comparar en la sección 3.

> **Dato clave:** el array no se inventó para guardar datos "lindos" — es la materia prima de la memoria. Por eso es la estructura más rápida para *acceder* (O(1), con el índice a la mano) y la más torpe para *reorganizar* (cada cambio de lugar desplaza a los vecinos). Esa dualidad — acceso veloz, mutación cara — es su personalidad completa, y la vas a ver repetida en todos los idiomas de la tabla.

## 2. El contrato del array

Cuando un lenguaje te dice "esto es un array", te está firmando un contrato de cuatro cláusulas. Conviene conocerlas antes de comparar idiomas, porque cada lenguaje cumple el contrato con trampas distintas:

1. **Memoria contigua** — los elementos viven en un único bloque, uno al lado del otro. No hay "espacios en blanco" entre el elemento 3 y el 4. Eso compra el acceso rápido: el hardware sabe que el elemento del índice `i` está en la dirección `dirección_base + i × tamaño_del_elemento`, una simple multiplicación y una suma.
2. **Tamaño fijo** — el bloque se reserva de una vez, con todos sus huecos, cuando nace el array. Si te quedaste corto, tenés que reservar otro más grande y **copiar todo**.
3. **Tipo homogéneo** — todos los elementos son del mismo tipo y ocupan lo mismo (cinco `int` de cuatro bytes cada uno, digamos). Esa uniformidad es la que hace posible la cuenta de la dirección.
4. **Acceso O(1)** — leer o escribir cualquier elemento cuesta lo mismo, sin importar el índice. Esta es la cláusula que ya conocés del capítulo 19, y acá viste su origen físico: no hay que buscar, la posición sale de una cuenta.

Fijate cómo leíste el punto 4 en el capítulo 19: "acceder a una lista por índice (`datos[5]`) es O(1)". Y tenés razón — pero hay un truco debajo. Tu lista de Python es, por dentro, un **array de punteros**: un bloque contiguo de referencias que apuntan a los objetos, que viven tirados por toda la memoria (los enteros, los strings, cualquier cosa). Por eso `datos[5]` es O(1) — salta al puntero 5, que está en un bloque pegadito — pero los *datos en sí* no están pegados. El array clásico no tiene punteros intermedios: el dato mismo ocupa su lugar en el bloque.

> **Importante:** "acceso O(1)" en el capítulo 19 lo viste con listas y era verdad. Acá vas a distinguir dos O(1) que se sienten igual: el de tu lista (salta al puntero, y el objeto puede estar en cualquier lado) y el del array clásico (el número está ahí mismo). Según el dato y el volumen, esa diferencia decide entre programar cómodo o programar rápido.

## 3. El array en otros idiomas

La forma de declararlo cambia el contrato en cada idioma, y mirarlos a todos de una vez explica más que cualquier definición. Arranquemos por los tres que más vas a escuchar en el trabajo.

En **Java**, el array es un ciudadano de primera clase: objeto con `length`, tamaño fijo y límites chequeados en cada acceso:

```java
int[] datos = {1, 2, 3, 4, 5};      // fijo, homogéneo, chequeado en caliente
System.out.println(datos[2]);       // 3
```

En **C**, el array es el hardware sin disfraz: un bloque de memoria al que llegás caminando por un puntero. Sin `length`, sin chequeo de límites, sin nada que te frene si leés un lugar que no existe:

```c
int datos[] = {1, 2, 3, 4, 5};      // bloque contiguo, responsabilidad del programador
printf("%d\n", datos[2]);           // 3
```

En **JavaScript**, la trampa es otra. El "array" común de JS — `let datos = [1, 2, 3]` — no es un array: es un objeto con propiedades numéricas, dinámico y hasta heterogéneo. El array de verdad, el que respeta el contrato, llegó recién en 2015 con los **arrays tipados** (*typed arrays*), de la mano de WebGL y el estándar `TypedArray`:

```javascript
let datos = new Int32Array([1, 2, 3, 4, 5]);   // buffer binario de enteros de 32 bits
console.log(datos[2]);                         // 3
```

`Int32Array` fija tamaño, tipo y memoria contigua — por eso la industria de gráficos y audio lo usa para manejar *buffers* (búferes) binarios, mientras el `Array` común sigue siendo la lista flexible de todos los días.

Y para completar el zoom, **C#** y **Rust** siguen el espíritu de Java y C, respectivamente:

```csharp
int[] datos = new int[] {1, 2, 3, 4, 5};       // objeto en el heap, límites chequeados
```

```rust
let datos: [i32; 5] = [1, 2, 3, 4, 5];         // por valor, seguro, tamaño fijo
```

Toda esa competencia se resume en una tabla. Mirá cómo varían las cinco decisiones del contrato según el idioma:

| Idioma | Se declara | Tamaño | Tipo | Chequeo de límites | ¿Dónde vive el dato? |
|---|---|---|---|---|---|
| C | `int datos[] = {...}` | fijo | homogéneo | no (responsabilidad tuya) | pegado en el bloque |
| Java | `int[] datos = {...}` | fijo | homogéneo | sí, en caliente | pegado en el bloque, en el *heap* |
| C# | `int[] datos = new int[] {...}` | fijo | homogéneo | sí, en caliente | pegado en el bloque, en el *heap* |
| Rust | `let datos: [i32; 5] = [...]` | fijo | homogéneo | sí, en compilación | pegado en el bloque, por valor |
| JS común | `let datos = [...]` | dinámico | heterogéneo | no aplica | no contiguo: objetos sueltos |
| JS tipado | `new Int32Array([...])` | fijo | homogéneo | no, pero acotado por `length` | buffer binario contiguo |
| **Python lista** | `datos = [...]` | dinámico | heterogéneo | no (te tira `IndexError`) | punteros contiguos a objetos sueltos |
| **Python `array`** | `array.array('i', [...])` | crece, pero compacto | homogéneo | no (te tira el error del tipo) | dato pegado en el bloque |
| **NumPy** | `np.array([...])` | fijo | homogéneo | no (te tira el error del rango y del tipo) | dato pegado, con matemática encima |

> **Buenas prácticas:** cuando leas código de otro idioma o te expliquen "un array" en una entrevista, lo primero que preguntás es *cuál* de estas filas es. Casi todos los malentendidos entre lenguajes viven en esas tres casillas: tamaño, tipo y dónde duerme el dato.

La fila de Python de la tabla es la que importa en este libro, y vas a desarmarla en sus tres variantes. La lista, que ya es tu amiga desde el capítulo 5, es la fila "punteros contiguos": flexible, heterogénea, la mejor para el día a día. Pero si tu programa hace una sola cosa — guardar muchos números del mismo tipo y hacerles cuentas — la memoria desparramada de los objetos te pesa. Ahí aparecen las dos filas de abajo: el `array` de la biblioteca estándar y NumPy. Van a ser las protagonistas del resto del capítulo.

## 4. `array.array`: el primo honesto de la biblioteca estándar

Llegó el turno del primo. Python trae, en la biblioteca estándar, un módulo llamado `array` que ofrece la versión compacta del array clásico: todos los elementos **del mismo tipo** y **pegados en memoria**, como en C. La diferencia con C es que acá no tenés que manejar la memoria a mano — Python reserva el bloque, lo estira cuando hace falta y lo libera solo. La idea, eso sí, es la misma: un bloque de números iguales, uno al lado del otro.

La clave está en el primer argumento: un **código de tipo** (un *type code*), una letra que le dice al módulo qué clase de número va a guardar y, con eso, cuántos bytes ocupa cada uno. Los códigos más útiles para empezar:

| Código | Tipo de C | Tamaño | Rango típico |
|---|---|---|---|
| `'b'` | `signed char` | 1 byte | de −128 a 127 |
| `'B'` | `unsigned char` | 1 byte | de 0 a 255 |
| `'h'` | `short` | 2 bytes | de −32.768 a 32.767 |
| `'i'` | `int` | 4 bytes | de −2.147.483.648 a 2.147.483.647 |
| `'l'` | `long` | 4 u 8 bytes | depende de la plataforma |
| `'q'` | `long long` | 8 bytes | enteros de 64 bits |
| `'f'` | `float` | 4 bytes | un real de 32 bits |
| `'d'` | `double` | 8 bytes | un real de 64 bits |

Fijate que ese código es la traducción directa de la tabla de la sección 3: son los tipos primitivos de C, con nombre y tamaño. Si guardás temperaturas con decimales finos, te conviene `'d'`; si es un contador discreto chico, `'i'`; si medís memoria con rigor, cada byte cuenta.

Armá un array, andá a buscar un elemento, cambialo y agregale uno — las operaciones más comunes del capítulo 5 aplican igual:

```python
import array as arr

a = arr.array("i", [1, 2, 3, 4, 5])   # 'i' -> enteros de 4 bytes, pegados
print(a)                                # array('i', [1, 2, 3, 4, 5])
print("primer elemento:", a[0])         # primer elemento: 1

a[0] = 10                               # modificar por índice es O(1)
print("modificado:", a)                 # modificado: array('i', [10, 2, 3, 4, 5])

a.append(6)                             # al final del bloque
print("tras append:", a)                # tras append: array('i', [10, 2, 3, 4, 5, 6])
```

El acceso por índice, la modificación y el reconocimiento de `len` y de los bucles `for` funcionan como con una lista — porque en el fondo seguís teniendo una secuencia ordenada con índices. La diferencia aparece en el tipo: como todo el mundo *tiene* que ser del mismo tipo, si intentás meter un `float` en un array de enteros, Python frena con un `TypeError`:

```python
a.append(3.5)
# TypeError: 'float' object cannot be interpreted as an integer
```

Ese error es la otra cara del contrato: la lista te deja mezclar tipos sin chistar, el array te frena en la puerta. La homogeneidad no es una restricción molesta — es la moneda de cambio de la memoria compacta. Para confirmarlo, pesalos a los dos:

```python
import sys

lista = [1, 2, 3, 4, 5]
array_compacto = arr.array("i", lista)

print(sys.getsizeof(lista))          # 120
print(sys.getsizeof(array_compacto)) # 100
```

Cinco números y la lista ya pesa más que el array. El `sys.getsizeof` mide el objeto en sí, y acá hay un detalle que conviene mirar con lupa porque es el corazón del capítulo:

> **Dato clave:** el `120` de la lista es el tamaño del *contenedor de punteros*, no de los datos. Los enteros de la lista viven en otro lado de la memoria (cada objeto `int` ocupa su propio espacio) y la lista solo guarda las referencias. El array, en cambio, es auto-suficiente: el `100` ya incluye los cinco enteros pegados adentro. La diferencia crece con la cantidad — pasá de cinco números a un millón y mirá:

```python
millon_con_prefijo = [i for i in range(1_000_000)]       # la misma lista del capítulo 10
array_de_un_millon = arr.array("i", millon_con_prefijo)

print(sys.getsizeof(millon_con_prefijo))                 # 8448728  -> ≈ 8,4 MB (el número del capítulo 10)
print(sys.getsizeof(array_de_un_millon))                 # 4000080  -> ≈ 4,0 MB
```

Un millón de enteros: la lista te pide **8,4 MB solo de punteros** (los objetos `int` en sí suman otra montaña aparte), y el array comprime lo mismo en **4,0 MB**, con el dato ya adentro. Ese es el valor del primo honesto: memoria compacta y tipada, al precio de perder la flexibilidad heterogénea.

Ahora, una aclaración que puede confundir. La tabla de la sección 2 dice que el array clásico es de *tamaño fijo*, y acá viste que `append` funciona. No hay contradicción: el `array` de Python arranca con tamaño fijo, pero crece dinámicamente *a lo lista* — el módulo reserva un bloque más grande cuando hace falta y copia. Lo que no cambia es el espíritu: sigue siendo un bloque de datos homogéneos pegados de a uno, no una caja de objetos sueltos. El tamaño fijo fue el precio que otros idiomas pagaron porque el hardware nació así; acá el intérprete absorbe ese costo por vos, como en las listas.

Y como el dato viaja en binario puro dentro del bloque, `array` es el puente natural hacia los archivos en formato binario y hacia las librerías de C. Dos métodos lo hacen trivial:

```python
import array

temp = array.array("i", [1, 2, 3, 4, 5])

with open("datos.bin", "wb") as archivo:
    temp.tofile(archivo)                # escribe el bloque tal cual, sin texto

recuperado = array.array("i")
with open("datos.bin", "rb") as archivo:
    recuperado.fromfile(archivo, 5)     # lee 5 enteros de vuelta
print(recuperado)                       # array('i', [1, 2, 3, 4, 5])
```

Ese diálogo con archivos binarios te va a resultar familiar cuando llegues al capítulo 27. Por ahora, guardalo como el "caso de uso de más bajo nivel": si el dato es un número, viaja apretado.

Entonces, el resumen honesto de esta sección: usá `array` cuando necesités **muchos números del mismo tipo, compactos y pegados** — memoria, archivos binarios, intercambio con C. Pero si el plan es hacerles *cuentas*, el primo se queda corto. Adivinaste: para eso existe la sección 6.

## 5. Cuando el array crece: mil millones de números

Antes de pasar a las cuentas, una advertencia que te va a servir toda la vida profesional: la memoria contigua es un lujo, y cuando la cantidad de datos crece, el lujo empieza a cobrar. Tomá la pregunta que se hace cualquier persona seria en una entrevista de trabajo con una empresa que maneja datos en serio: *¿qué pasaría si tuviera que guardar mil millones de números en un array?*

Armemos las cuentas. Un entero de 4 bytes, multiplicado por mil millones:

```python
uno_mil_millones = 10 ** 9
print(uno_mil_millones * 4)                 # 4000000000  -> 4 GB
```

**4 GB reservados de una sola vez** para un solo array. Esa cifra trae tres consecuencias:

1. **La memoria se pide toda de golpe.** El sistema tiene que encontrar un solo bloque contiguo de 4 GB. Si la RAM está ocupada por otros programas, puede no existir un hueco así de largo — o el sistema empieza a usar el *swap* (el archivo de intercambio en disco), y el rendimiento cae al piso porque el disco es miles de veces más lento que la RAM.
2. **Cada copia es catastrófica.** Recordá la cláusula del tamaño fijo: ¿crecer una vez? No existe un "pedacito extra" — se reserva otro bloque más grande y se **copia entero**. Hacer eso con 4 GB cada vez que querés agregar un puñado de elementos es inviable. El universo ahí te está pidiendo otra estructura de datos.
3. **La reorganización cuesta lo que cuesta.** Insertar un elemento en el medio de un array de gran tamaño implica desplazar la mitad del bloque, una operación O(n) del capítulo 19 que, con mil millones de elementos, es una operación de miles de millones de movimientos. Es literalmente "no corresponde".

Y si a la memoria le sumás la búsqueda, el panorama se completa: acceder por índice es O(1), pero *encontrar* un dato en un array de mil millones de elementos que no están ordenados es O(n) — en el peor caso, recorrer mil millones. El array no tiene noción de orden interno: si querés ir directo a un valor, tenés que ordenarlo vos (y eso cuesta O(n log n)), o buscar de a uno.

En los lenguajes que manejan la memoria a mano — C y C++, por ejemplo — este escenario suma una cuarta consecuencia que conviene tener presente aunque no programes en esos lenguajes: cada bloque que reservás hay que **liberarlo** cuando terminás. Si te olvidás, quedás con una *fuga de memoria* (memory leak): el programa sigue pidiendo recursos y no los devuelve, y con el tiempo se arrastra y hasta colapsa. Python te ahorra esa responsabilidad con su recolector de basura — pero el costo de la memoria contigua a gran escala no te lo ahorra nadie, porque es física.

> **Dato clave:** el array brilla en el acceso y duele en el cambio. Cuando el tamaño crece, sus dos O(fáciles) (acceso O(1), `append` amortizado O(1)) se enfrentan a sus dos O(caros) (insertar/borrar en el medio O(n), copiar todo al crecer O(n)). Esa es la grieta por la que se inventaron las **listas enlazadas**, que vas a construir con tus manos en el próximo capítulo: guardan cada dato repartido donde haya lugar y se pasan el testigo con un puntero al siguiente, cambiando memoria apretada por libertad para crecer sin copiar.

Fijate cómo el problema no es "el array está mal": es la herramienta justa para el volumen equivocado. Para un puñado de números, es la mejor opción. Para mil millones de números del mismo tipo con cuentas encima, directamente no entra — y ahí el mundo Python tiene una respuesta de lujo, que es la protagonista de la próxima sección: un array que además sabe hacer matemática.

## 6. NumPy: el array que piensa en bloques

Llegó la hora de la herramienta estrella. Si tu problema es "muchos números, del mismo tipo, y encima tengo que hacerles cuentas", el `array` de la biblioteca estándar te da el bloque compacto pero te deja haciendo las operaciones vos, número por número. **NumPy** va un paso más allá: te da el mismo bloque compacto, sí, pero además sabe **pensar en bloque** — las operaciones matemáticas se aplican a todo el array de una sola vez, sin que vos recorras nada.

NumPy no es parte de la biblioteca estándar: es un paquete externo, y ya lo conocés de nombre desde el capítulo 2, cuando te lo instalaron en el entorno junto con `pandas` y `matplotlib`. Por si fuera poco, es una pieza central de casi toda la ciencia de datos en Python — el capítulo del cierre del libro, Data Analytics, se apoya en eso. La importación estándar es con su apodo histórico:

```python
import numpy as np
```

La estructura de datos principal se llama `ndarray` (por *N-dimensional array*, "array de N dimensiones"): un bloque contiguo de datos numéricos del mismo tipo, como los que estuviste viendo, pero con matemática para todos ellos encima. Se crea con `np.array` y acepta listas, tuplas o rangos:

```python
arr = np.array([1, 2, 3, 4, 5])
print(arr)                          # [1 2 3 4 5]
```

El acceso y la modificación son los de siempre — por eso en el fondo seguís en el mismo terreno de la sección 2:

```python
print(arr[2])                       # 3
arr[0] = 10
print(arr)                          # [10  2  3  4  5]
```

La diferencia con el `array` del módulo estándar aparece apenas hacés una cuenta. Sumale cinco a todo el array: no hay `for`, no hay comprensión, no hay recorrida. Una sola expresión suma cinco a cada elemento del bloque:

```python
print(arr + 5)                      # [15  7  8  9 10]
```

Eso se llama una operación **vectorizada** (vectorizada, sí, del inglés *vectorized*): la operación se aplica sobre el bloque completo en un solo barrido — en código compilado de C, no en Python. Vos no ves el bucle porque no existe en tu código: existe, pero corre dentro de la librería, millones de veces por segundo, mientras tu parte queda legible y sin un solo `range`.

Lo mismo con las agregaciones: `np.sum` recorre el bloque y devuelve el total:

```python
print(np.sum(arr))                  # 24
```

Fijate el detalle: recién modificaste el array antes de sumar, y por eso el total es **24** (10 + 2 + 3 + 4 + 5), no 15. NumPy siempre trabaja sobre el estado actual del bloque.

Y el segundo rasgo que le cambia la vida a tu código: los arrays son **multidimensionales** — una matriz se crea con una lista de listas, y se accede con dos índices separados por coma:

```python
matriz = np.array([[1, 2, 3],
                   [4, 5, 6]])

print(matriz)                       # [[1 2 3]
                                    #  [4 5 6]]
print(matriz[1, 2])                 # 6   (fila 1, columna 2)
print(matriz.shape)                 # (2, 3)
print(matriz.ndim)                  # 2
```

`shape` te dice el tamaño de cada dimensión (2 filas × 3 columnas) y `ndim` cuántas dimensiones tiene. Con eso, el array deja de ser "una fila de números" y pasa a ser una grilla sobre la que podés pensar matrices reales — transponelas:

```python
transpuesta = np.transpose(matriz)   # o matriz.T, es equivalente
print(transpuesta)                   # [[1 4]
                                     #  [2 5]
                                     #  [3 6]]
```

Lo que era una matriz 2×3 quedó en 3×2, girando filas y columnas de lugar. Esto va directo al corazón de las matemáticas de matrices que vas a pisar fuerte en la parte de datos y de IA del libro.

NumPy además trae un repertorio de fábricas listas para usar. La que vas a ocupar siempre en las pruebas y en el curso del libro: `np.arange` es el `range` de NumPy (devuelve un array), `np.zeros` te da una matriz llena de ceros con las dimensiones que le pidas, y `np.mean` promedia todo el bloque:

```python
print(np.arange(0, 20, 2))          # [ 0  2  4  6  8 10 12 14 16 18]

ceros = np.zeros((3, 3))
print(ceros)                        # [[0. 0. 0.]
                                    #  [0. 0. 0.]
                                    #  [0. 0. 0.]]

print(np.mean(matriz))              # 3.5
```

Y una más que vas a necesitar desde hoy para hacer experimentos: los números aleatorios. Acá conviene usar el generador moderno de NumPy, `default_rng`, que es reproducible — con la misma semilla te da siempre los mismos números, ideal para que tu prueba de hoy sea tu prueba de mañana:

```python
rng = np.random.default_rng(7)
print(rng.random(5))                # [0.62509547 0.8972138  0.77568569 0.22520719 0.30016628]
```

> **Dato clave:** si corrés con una semilla *reproducible* (`default_rng(7)`), tu experimento da los mismos números en cualquier máquina. Eso no es una curiosidad: es la diferencia entre una simulación que podés documentar y una que cambia cada vez que la corrés.

Ahora, la prueba que te convence de por qué esta sección entera vale la pena. Volvé a la pregunta del capítulo 19 — la del costo — y corré el mismo `sum` contra el mismo array, en sus dos versiones. La diferencia no está en el resultado (el mismo), está en el precio:

```python
from time import perf_counter

n = 10_000_000
grande = np.arange(n, dtype=np.float64)        # diez millones de números, pegados

inicio = perf_counter()
total = np.sum(grande)                          # vectorizado: corre en C
print(f"numpy:  {perf_counter() - inicio:.4f} s")   # numpy:  ≈ 0.01 s

inicio = perf_counter()
total = sum(grande)                             # recorrido de a uno en Python
print(f"python: {perf_counter() - inicio:.4f} s")   # python: ≈ 0.66 s
```

En una corrida típica, `np.sum` tarda unas centésimas y el `sum` de Python alrededor de medio segundo o más (los valores exactos cambian; el orden de magnitud, no). Y si la operación es más intensa, la brecha crece: elevar al cuadrado los diez millones de números vectorizados puede andar en las centésimas, mientras el bucle equivalente te lleva alrededor de un segundo y medio. Estás viendo las dos caras del mismo dato: **el mismo bloque, el mismo resultado, y un recorrido que desaparece porque pasó al interior de la librería**.

> **Importante:** fijate qué tan limpio quedó el código vectorizado. No es que escribiste menos líneas por suerte: es que *pensaste en bloques* en lugar de pensar en elementos. Esa es la habilidad que vas a practicar con los ejercicios del final — y la misma mentalidad que te va a pedir `pandas` cuando llegues a las tablas.

Falta un último ladrillo para completar el panorama: el tipo de los datos. NumPy tiene tipos explícitos, los mismos de C que viste en la tabla del módulo `array`, y el tipo dice cuánto pesa cada elemento:

```python
enteros = np.array([1, 2, 3, 4, 5], dtype=np.int32)   # enteros de 4 bytes
print(enteros.itemsize)                               # 4
print(enteros.nbytes)                                 # 20

reales = np.array([1.5, 2.5], dtype=np.float64)       # dobles de 8 bytes
print(reales.itemsize)                                # 8
```

`.itemsize` te dice los bytes de cada elemento, `.nbytes` el total del bloque. Ahí tenés la misma tabla de la sección 4, pero con el apellido de NumPy: `int32`, `float64` y compañía. Son los tipos C, traducidos a Python con un número que indica bits.

> **Dato clave — el int de plataforma:** si creás `np.array([1, 2, 3])` sin decir el tipo, NumPy elige por vos el entero nativo — y ese "nativo" depende de la máquina: en Windows suele ser `int32` (porque el `int` de C es de 32 bits) y en Linux `int64`. La moraleja práctica: si el tamaño importa, **declará el `dtype` a mano**. Ese detalle es un eco eterno de la sección 3: el array, en el fondo, siempre hereda algo del hardware y de los tipos de C.

Y para redondear, la tabla comparativa que resume todo el capítulo — las tres formas de guardar datos en fila en Python:

| Característica | Lista | `array` (módulo) | NumPy (`ndarray`) |
|---|---|---|---|
| Tipos de datos | heterogéneos | homogéneos | homogéneos |
| Memoria | punteros + objetos sueltos | bloque pegadito | bloque pegadito |
| Flexibilidad | dinámica | crece compacto | fijo (reservás el bloque) |
| Eficiencia numérica | baja (bucle por elemento) | media (bloque, sin matemática) | alta (vectorizada, en C) |
| Funciones matemáticas | limitadas | limitadas | amplias: `sum`, `mean`, `transpose`… |
| Multidimensional | listas de listas | incómodo | nativo |

La regla de oro que cierra la comparación: para el día a día te quedás con la lista; si necesitás muchos números del mismo tipo y compactos, el `array`; y si encima hay que **hacerles cuentas**, NumPy se lleva todo el juego. Esa es la misma escalera que te anticipaba el cierre del capítulo 19.

## 7. Resumen y conceptos clave

Este capítulo te llevó de "la lista de Python es un array" a saber exactamente por qué no lo es, y qué opciones reales tenés cuando sí lo necesitás. Arrancaste en el origen físico: la memoria de la computadora es, ella misma, un array de celdas numeradas alcanzables por dirección — y por eso el array es la estructura más vieja y la más rápida para acceder (O(1) garantizado por hardware). Después firmaste el contrato de cuatro cláusulas — memoria contigua, tamaño fijo, tipo homogéneo, acceso O(1) — y lo viste en acción en C, Java, JavaScript y sus vecinos, donde cada idioma lo cumple con trampas distintas; JavaScript, incluso, pasó dos décadas sin ofrecerlo de verdad hasta los `TypedArray`. Con ese mapa en la cabeza, conociste al primo honesto: el módulo `array` de la biblioteca estándar, que guarda los números pegados y del mismo tipo, pesa menos que tu lista (120 contra 100 bytes en cinco elementos y 8,4 MB contra 4,0 MB en un millón) y te abre la puerta de los archivos binarios. Viste sus límites cuando la escala se dispara — esos 4 GB por mil millones de enteros — y entendiste por qué el array brilla en el acceso y duele en el cambio, dejando la puerta abierta a las listas enlazadas del próximo capítulo. Y cerraste con la herramienta que resuelve el caso "muchos números con cuentas encima": **NumPy**, con su `ndarray` vectorizado que suma, multiplica, transpone y promedia el bloque entero sin un bucle visible — más de 50 veces más rápido en las pruebas del capítulo.

Repasá el checklist antes de seguir:

- [ ] La memoria RAM es un array de celdas numeradas; el array es la estructura más vieja de la informática.
- [ ] **FORTRAN 1957** (`DIMENSION`) y **ALGOL 60** (el término "array") la hicieron formal; Java la blindó, C la dejó pelada y JavaScript la tuvo que reimportar con `TypedArray` en 2015.
- [ ] El contrato del array: memoria **contigua**, **tamaño fijo**, **tipo homogéneo**, acceso **O(1)**.
- [ ] La lista de Python es un **array de punteros**: bloque de referencias a objetos sueltos; el array clásico guarda el dato mismo en el bloque.
- [ ] `array.array` requiere un **código de tipo** (`'b'`, `'h'`, `'i'`, `'d'`…) y rechaza el tipo incorrecto con `TypeError`.
- [ ] Memoria medida: 5 enteros → lista **120 B** vs `array` **100 B**; un millón → lista **8,4 MB** (solo punteros) vs `array` **4,0 MB**.
- [ ] A gran escala el array acumula riesgo: **4 GB** por mil millones de enteros, copias al crecer, desplazamientos O(n), búsqueda O(n) sin orden.
- [ ] NumPy se importa como `np`, su tipo central es `ndarray`, y las operaciones como `arr + 5`, `np.sum`, `np.transpose` o `np.mean` son **vectorizadas** (corren en C, sin bucle tuyo).
- [ ] `np.arange`, `np.zeros`, `np.random.default_rng(semilla)` dan fábricas y aleatoriedad **reproducible**.
- [ ] El `dtype` hereda el entero de plataforma (en Windows `int32`, en Linux `int64`); si el tamaño importa, declarálo.
- [ ] Escalera de elección: lista para el día a día → `array` para memoria compacta → NumPy para cuentas sobre bloques.

## 8. Ejercicios

1. **Ida y vuelta entre las tres caras.** Armá una lista con los enteros del 1 al 1000. Convertila a `array('i')` y a un `ndarray` de NumPy con `dtype=np.int32`. Comprobá los tres tipos, verificá que los tres suman 500500 y que el tamaño de cada elemento (`.itemsize`) es el mismo. Después mirá los últimos tres elementos del `ndarray` y comprobá que son `[ 998  999 1000]`.

2. **El micro-dato que no entra.** ¿Qué pasa si intentás meter un número que no cabe en el tipo? Probá con `array.array('b', [200])` y con `array.array('h', [80_000])`, y explicá por qué el error de cada uno menciona un tipo de C distinto. Y volvé a convertirte en el array del capítulo: confirmá que `array.array('i', [1, 2, 3]).append(3.5)` tira `TypeError`, no un resultado raro.

3. **La conversión de temperaturas, vectorizada.** Tené las temperaturas de una semana en grados Celsius como `np.array([10.0, 20.0, 30.0, 40.0])`. Convertilas a Fahrenheit con la fórmula que ya te acompañó en el capítulo 11 — multiplicar por `1.8` y sumar `32` — en **una sola expresión**, y verificá que da `[50.  68.  86. 104.]`. Después, sin bucle, decí: ¿cuántos grados Fahrenheit es 37 °C?

4. **Ventas con NumPy.** Tenés las ventas del mes en `np.array([1200, 3400, 2100, 4500, 1800, 3900])`. Calculá total, promedio, mínimo, máximo y en qué posición del mes se vendió más (`argmax`), todo con métodos de NumPy, sin un solo `for`.

5. **La diferencia que crece.** Medí con `sys.getsizeof` una lista y un `array('i')` para `n` de 1000 y de 100 000 elementos, y compará los cuatro números. Explicá por qué, al pasar de 1 000 a 100 000, la lista crece lineal y el array también — y por qué el del array es mucho más chico. (Pista: mirá cuántas veces entra 4 bytes por elemento.)

6. **Cuadrados, de las dos maneras.** Generá 100 000 números al azar con `np.random.default_rng(7)` (guardalos en un array), y calculá la suma de sus cuadrados de dos formas: vectorizada (`(datos ** 2).sum()`) y con un bucle de comprensión Python (`sum(x * x for x in datos)`). Verificá que dan el mismo resultado y comentá con tus palabras cuál de las dos esperás que escale a un millón de elementos y por qué, a la luz de la sección 6.

7. **La decisión de estructura.** En dos o tres líneas de prosa, para cada caso decidí si usarías lista, `array` o NumPy, y justificá en una palabra: (a) los nombres de los clientes del mes; (b) un millón de temperaturas para promediar y graficar; (c) los bytes de una imagen en blanco y negro para mandar por la red a un programa en C.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. Ida y vuelta
import array as arr
import numpy as np

mil = list(range(1, 1001))
como_array = arr.array("i", mil)
como_ndarray = np.array(mil, dtype=np.int32)

print(type(mil).__name__, type(como_array).__name__, type(como_ndarray).__name__)
# list array ndarray

print(como_array.itemsize, como_ndarray.itemsize)     # 4 4
print(sum(como_array), int(como_ndarray.sum()))       # 500500 500500
print(como_ndarray[:5], como_ndarray[-3:])            # [1 2 3 4 5] [ 998  999 1000]

# 2. El micro-dato que no entra
try:
    arr.array("b", [200])
except OverflowError as error:
    print(error)                                      # signed char is greater than maximum

try:
    arr.array("h", [80_000])
except OverflowError as error:
    print(error)                                      # signed short integer is greater than maximum

try:
    arr.array("i", [1, 2, 3]).append(3.5)
except TypeError as error:
    print(error)                                      # 'float' object cannot be interpreted as an integer

# 3. Temperaturas vectorizadas
grados_c = np.array([10.0, 20.0, 30.0, 40.0])
print(grados_c * 1.8 + 32)                            # [ 50.  68.  86. 104.]
print(np.round(37.0 * 1.8 + 32, 1))                   # 98.6

# 4. Ventas con NumPy
ventas = np.array([1200, 3400, 2100, 4500, 1800, 3900])
print(ventas.sum(), ventas.mean(), ventas.min(), ventas.max())   # 16900 2816.6666666666665 1200 4500
print(ventas.argmax())                                # 3

# 5. La diferencia que crece
import sys

for n in (1_000, 100_000):
    lista = list(range(n))
    compacto = arr.array("i", lista)
    print(n, sys.getsizeof(lista), sys.getsizeof(compacto))
# 1000 8056 4080
# 100000 800056 400080

# 6. Cuadrados, de las dos maneras
rng = np.random.default_rng(7)
datos = rng.random(100_000)
print((datos ** 2).sum())                             # 33376.791131646125
print(sum(x * x for x in datos))                      # 33376.79113164564

# 7. La decisión de estructura (prosa de ejemplo)
# (a) lista: los nombres son texto heterogéneo y no van a hacer cuentas.
# (b) NumPy: un millón de temperaturas para promediar y graficar pide vectorización.
# (c) array: bytes exactos y viaje binario hacia un programa de C.
```