# Capítulo 7 — Funciones: una introducción al DRY, «no te repitas»

Hasta acá escribiste código que se ejecuta en orden: primero esto, luego aquello, después el otro. Los `while` y los `for` te dejaron repetir, los `if` decidir, y con las estructuras del capítulo 5 agrupaste datos. Pero hay algo que todavía te falta y que es **el salto que divide a la programación de verdad de la tarea**: reutilizar. En este capítulo vas a aprender a definir tus propias funciones — los primeros "bloques" que vas a poder encastrar para armar programas cada vez más grandes. Y antes de escribir una sola línea de `def`, quiero contarte una historia, porque entender *por qué* existen las funciones te va a ahorrar años de malas decisiones.

---

## 1. La historia que explica por qué existen las funciones

En los orígenes de la programación, cada programa era casi obra de artesanía: se conectaban cables, se perforaban tarjetas, y modificar una sola cosa podía significar rehacerlo todo. Cuando los lenguajes se volvieron escritos, apareció una herramienta poderosa y, como todo lo poderoso sin control, **peligrosa**: el `goto`. Era una instrucción que decía, literalmente, *"andá a la línea 40"*, *"saltá a la 12"*.

Imaginá un programa donde las líneas saltan de los pasos 1 a 20 a 5 a 33 y otra vez a 12. El flujo se cruzaba una y otra vez, y seguir la lógica era como desenredar un plato de fideos: por eso todavía hablamos de **código espagueti**. Funcionaba, claro — el programa hacía lo que decía el programador. El problema era el que venía después: *cualquier otro ser humano* (o el propio autor tres meses más tarde) tenía que adivinar qué pasaba ahí dentro.

En 1968, el científico Edsger Dijkstra publicó una carta célebre titulada **"Go To Statement Considered Harmful"** — *"la declaración go to considerada dañina"*. Su propuesta se conoció como **programación estructurada**: construir los programas con tres estructuras claras — **secuencia** (línea tras línea), **decisión** (`if`) y **repetición** (los bucles) — y además con **subrutinas**: pedazos de código con nombre a los que podés llamar cuando los necesitás. Las subrutinas de esa época son las bisabuelas de las funciones modernas: **código empaquetado una vez, con un nombre, invocable desde cualquier lado**.

Las funciones no son un capricho del lenguaje: son la respuesta histórica al código espagueti. Cuando definís una función, le estás diciendo al mundo "este pedazo tiene nombre, hace una sola cosa, y no necesitás entrar a leerlo cada vez".

### DRY: "Don't Repeat Yourself"

Unos años después, en *The Pragmatic Programmer* (1999), Andy Hunt y Dave Thomas pusieron nombre a un principio que ya flotaba en el aire: **DRY** — *Don't Repeat Yourself*, "no te repitas". La idea es simple de decir y difícil de cumplir: **cada pieza de conocimiento debe vivir en un solo lugar del programa**.

¿Por qué es tan importante? Pensalo con la trampa más natural del mundo: copiar y pegar. Tenés un código que valida una edad y lo necesitás en cinco partes del programa. Lo pegás cinco veces y… aparece un bug. Corregís una copia. Después te enterás de que las otras cuatro siguen rotas. Peor: el día que la regla cambia ("ahora la edad válida arranca en 21"), tenés que acordarte de **los cinco lugares**. DRY dice: no. Ese código vive **una vez** en una función, y las cinco partes simplemente la llaman. Un solo lugar para corregir, un solo lugar para cambiar.

> **Dato clave:** copiar y pegar es la forma en la que el código se repite *hoy* y se pudre *mañana*. La regla mental para detectarlo: si tenés que hacer el mismo cambio en más de un lugar, lo que está repitiéndose debería ser una función.

### No reinventar la rueda: programar es armar castillos con bloques

Acá aparece la metáfora definitiva. Nadie construye un rascacielos moldeando cada ladrillo en el momento: se usan **bloques** ya hechos, se encastran y se avanza. Programar es exactamente eso. Vos no escribís el código que hace la raíz cuadrada: **lo llama**. Ya lo venís haciendo todo el capítulo y ni cuenta te diste: `print`, `len`, `sum`, `int`, `range` son funciones que escribió otra persona, y las reutilizás sin saber cómo están hechas por dentro.

Esa idea escala en tres niveles, y este capítulo arranca por el primero:

- **Funciones** (esta Parte V): bloques que definís vos, la pieza más básica.
- **Clases** (Parte VI): bloques que además *recuerdan* su estado entre llamadas.
- **Módulos**: cajas donde guardás tus bloques para reutilizarlos en otros programas (los módulos propios que importás con tu propio `import`), la **biblioteca estándar** de Python (`math`, `random`, `statistics`…), y los **de terceros** que ya tocaras en el libro: `pandas` y `matplotlib` son, literalmente, castillos gigantes que otros armaron bloque a bloque para que vos los uses como uno solo.

Cada bloque resuelve un problema chico y conocido. Programar bien no es inventar todo: es **elegir y encastrar** piezas, y definir piezas nuevas cuando ninguna encaja. A eso se le llama, con una frase hermosa, *pararse en hombros de gigantes*: no repetís el esfuerzo de los que ya lo hicieron.

Vamos entonces a la pieza más básica: **definir una función**.

---

## 2. Qué es una función

Una función es una **receta con nombre**. Agrupás un pedazo de código, le ponés un nombre, y a partir de ahí podés ejecutarlo las veces que quieras con solo invocarla. Tres partes la componen, y te adelanto el vocabulario porque lo vas a usar todo el resto del libro:

- **Argumentos**: lo que le entregás a la función (los ingredientes).
- **Cuerpo**: lo que la función hace con eso (la receta en sí).
- **Retorno** (`return`): lo que la función te devuelve (el plato servido).

La mejor analogía para grabártela es una **licuadora**: le metés frutas (argumentos), presionás el botón (invocación), trabaja por dentro (cuerpo), y del otro lado sale el licuado (retorno). Vos no sabés — ni te importa — cómo gira la cuchilla. Sabés *qué hace* y qué esperar de ella. A eso se le llama tratar a la función como una caja **encapsulada** (*encapsulated*): importa la interfaz, no el interior.

Ya usaste decenas de funciones sin pensarlo. A los objetos que se pueden invocar con paréntesis se los llama **callables** (de *"callable"*, "invocable"):

```python
len([1, 2, 3])      # 3
sum([4, 5, 6])      # 15
int("42")           # 42
print("hola")       # hola
```

La novedad de este capítulo es que aprendés a **fabricar tus propios callables** — y ahí es donde el DRY se vuelve un superpoder.

---

## 3. Definir: `def`, el cuerpo y `pass`

La palabra clave es `def` (de *"define"*). La sintaxis general es:

```python
def nombre(parametros):
    cuerpo
```

El nombre sigue las reglas de nombres que ya conocés del capítulo 3, y el cuerpo va **indentado**, igual que el cuerpo de un `if` o de un `for`. Empecemos con la función más simple posible, una que no recibe nada y no devuelve nada:

```python
def saludo():
    """Imprime un mensaje de bienvenida."""
    print("¡Bienvenido a Python!")

saludo()            # ¡Bienvenido a Python!
saludo()            # ¡Bienvenido a Python!
```

Dos detalles importantes:

- **Definir no ejecuta.** El código de la función corre recién cuando la invocás con `nombre()`. La definición, por sí sola, no imprime nada.
- **Una vez definida, se usa igual que cualquier otra.** `saludo()` no se distingue de `print()` para quien la llama. Ese es el punto: vos construiste tu primer bloque.

¿Qué pasa si querés dejar una función "en blanco" para completarla después pero no querés errores de indentación? Python te da la sentencia `pass` — "no hagas nada". Es un marcador de posición:

```python
def construir_modelo():
    pass          # todavía no sé qué va acá

construir_modelo()  # no pasa nada, y no rompe
```

Cuando reemplacemos el `pass` por código real (más adelante) va a quedar claro hasta qué punto es útil para reservar el lugar mientras diseñás la estructura.

En Python las funciones son **objetos**: tienen tipo, se pueden guardar en variables y hasta inspeccionar. Mirá:

```python
def saludo():
    print("hola")

type(saludo)       # <class 'function'>
```

Son objetos de tipo `function`. Esa idea (que una función sea un dato como cualquier otro) no parece gran cosa todavía, pero va a ser la base del capítulo de lambdas y de las funciones de orden superior.

> **Dato clave:** `def` crea la receta; los paréntesis la cocinan. Si definís y no invocás, nada pasa. Si invocás sin definir, `NameError`.

---

## 4. Documentar con docstrings

El código se lee, pero la intención se explica. La forma estándar de documentar una función en Python es el **docstring** (de *"doc"* + *"string"*): el primer valor de texto que aparece justo después de la línea `def`. Por eso en el ejemplo anterior escribí `"""Imprime un mensaje de bienvenida."""`.

```python
def promedio(muestras):
    """Calcula el promedio de una lista de números."""
    return sum(muestras) / len(muestras)
```

Ese comentario especial no es para la gente que mira el código por encima: es para Python mismo. Lo podés consultar en cualquier momento y se vuelve la ficha de ayuda de tu función:

```python
help(promedio)
# Help on function promedio in module __main__:
#
# promedio(muestras)
#     Calcula el promedio de una lista de números.
```

Los docstrings pueden ser tan cortos como una línea o tan largos como un manual de instrucciones (nombres de parámetros, ejemplos de uso, advertencias). La regla práctica que usa todo el mundo: si la función hace una sola cosa, una línea alcanza para decir cuál.

> **Dato clave:** quien escribe el código también soy *yo de mañana* y *mis compañeros*. Un docstring de una línea le gana a cero docstrings siempre. Cuando veamos el estilo profesional, el docstring va a ser la primera línea del curriculum vitae de tu función.

---

## 5. `return`: lo que la función te devuelve

Una función que solo *imprime* está bien para saludar, pero la mayoría de las veces querés que te **entregue** un valor para seguir usándolo. Eso se hace con `return` (de *"return"*, "devolver"):

```python
def triplicar(valor):
    return valor * 3

resultado = triplicar(5)
print(resultado)          # 15
print(resultado + 10)     # 25  → ya es un número normal, sigue su vida
```

El valor que vuelve entra al flujo normal del programa: podés guardarlo en una variable, usarlo en una operación, pasarlo a otra función. Es exactamente la diferencia entre una licuadora que te muestra el licuado en una pantallita y una que te lo entrega en el vaso.

### `print` NO es `return`

Esta es una de las confusiones más típicas de quien recién arranca. `print` **muestra** algo en pantalla y se olvida; `return` **entrega** algo y el programa puede agarrarlo. Compará:

```python
def suma_de_tres(a, b, c):
    print(a + b + c)      # solo muestra

def suma_de_tres_bien(a, b, c):
    return a + b + c      # entrega
```

A primera vista parecen iguales (ambas "hacen" lo mismo cuando las mirás en consola), pero mirá el después:

```python
x = suma_de_tres(1, 2, 3)        # muestra 6 en pantalla
print(x)                         # None   ← no entregó nada

y = suma_de_tres_bien(1, 2, 3)   # no muestra nada
print(y * 2)                     # 12    ← el valor sigue vivo
```

`suma_de_tres` termina entregando `None` (el "vacío" que ya conocés). `suma_de_tres_bien` entrega `6`, y con eso se puede operar. Si una función "calcula algo", sus resultados deberían salir por `return`; el `print` queda para los programas (el lugar donde las cosas se *muestran*), no para las piezas.

### `return` corta la ejecución

Como el `break` en los bucles, `return` tiene un efecto inmediato: **termina la función en el acto**. Todo lo que venga después, dentro de la función, no se ejecuta:

```python
def evaluar(numero):
    if numero < 0:
        return "negativo"
    return "positivo o cero"

print(evaluar(-5))    # negativo
print(evaluar(7))     # positivo o cero
```

Fijate que no hay `else`: si la condición se cumple, el primer `return` *sale* de la función, así que el segundo solo se alcanza cuando no se cumplió. A esto se lo llama **return temprano** (o *early return*) y es una de las señales de código limpio: menos anidamiento, la lectura avanza en línea recta.

También se puede escribir `return` sin valor: solo corta la función. Por defecto, toda función termina con un `return None` invisible — por eso la función que solo imprimía devolvía `None`.

Ese `None` invisible es el origen de un clásico error de principiante : invocar la función **dentro de un `print`** y ver aparecer un `None` donde nadie lo esperaba. Mirá esta escena en cámara lenta:

```python
def pausa():
    print("haciendo una pausa...")

print(pausa())
```

¿Qué imprime? *Dos* líneas:

```python
haciendo una pausa...
None
```

¿Por qué? Porque `print(pausa())` se evalúa **de adentro hacia afuera**: primero corre `pausa()`, que ya imprime su mensaje; después el `print` recibe lo que `pausa()` le entregó — y sin `return`, eso es `None` — y lo imprime como si nada. El resultado: una línea de la función y un `None` "de más". La regla para evitarlo: si la función ya imprime por su cuenta, **no la envuelvas en otro `print`**, invocala suelta: `pausa()`.

> **Dato clave:** `return` tiene dos trabajos: **entregar** un valor y **apagar** la función. Si una función no necesita entregar nada (solo efectos, como imprimir), igual termina devolviendo `None`. Esa es la respuesta a "¿qué devuelve una función que no tiene `return`?": `None`. Y la segunda parte del truco: si la invocás dentro de un `print`, ese `None` **se imprime**. Por eso `print(funcion_sin_return())` muestra la salida de la función *y además* `None`.

---

## 6. Parámetros y argumentos

Llegó el momento de la parte más rica: pasarle datos a las funciones. Repasemos el vocabulario, que acá la gente se enreda. En la **definición** aparecen los **parámetros** (los "huecos" que la función espera). En la **llamada** aparecen los **argumentos** (los valores que entran a esos huecos):

```python
def suma(a, b):       # ← a y b son parámetros
    return a + b

suma(12, 5)           # ← 12 y 5 son argumentos
suma("Hola, ", "mundo!")   # y también va a funcionar: "Hola, mundo!"
```

El número de argumentos debe coincidir con los parámetros. Si no, Python te avisa con el `TypeError` que viste en el capítulo 6:

```python
suma(12)          # TypeError: suma() missing 1 required positional argument: 'b'
suma(1, 2, 3)     # TypeError: suma() takes 2 positional arguments but 3 were given
```

Se pueden pasar los argumentos **por posición** (en orden) o **por nombre**, indicando a qué parámetro va cada uno:

```python
suma(b=5, a=12)    # 17  → el orden no importa si vas por nombre
```

Mezcla rara, pero válida: primero los posicionales (respetando el orden del `def`) y después los nombrados:

```python
suma(12, b=5)      # 17
```

Por qué importa saber esto: a medida que las funciones crecen, nombrar los argumentos hace que la llamada se lea sola. Vas a verlo más adelante cuando las funciones tengan muchos parámetros: `persona(nombre="Ana", edad=30, ciudad="Misiones")` se entiende mucho mejor que `persona("Ana", 30, "Misiones")`.

Fijate que Python no pregunta *qué tipo* de dato llega: `suma(12, 5)` suma enteros, `suma("Hola, ", "mundo!")` concatena texto. La misma receta, dos comportamientos — un gusto del tipado dinámico que la función no conoce de antemano.

---

## 7. Valores por defecto (y la trampa de `[]`)

¿Y si querés que la función funcione aunque falten argumentos? Definís valores por defecto para los parámetros, con `=` en la lista de parámetros:

```python
def saludar(nombre="mundo"):
    return f"Hola, {nombre}"

saludar()               # Hola, mundo
saludar("Ana")          # Hola, Ana
```

Reglas importantes (y el `SyntaxError` de cada una):

- Los parámetros **con** valor por defecto van siempre **al final**. Mezclarlos por delante de los obligatorios es ilegal:

```python
def suma(a=1, b):   # SyntaxError: non-default argument follows default argument
    return a + b
```

- Los argumentos se sustituyen de izquierda a derecha: si la función tiene `(a=1, b=3)`
  - la llamada `suma(2)` 
    - cambia `a` -> 2 y b se mantiebe en 3 y el resultado es 5.
  - la llamada `suma(2, 5)` 
    - cambia `a` -> 2 y `b` -> 5 y ahora el resultado es 7. 
  
### La trampa del valor mutable por defecto

Acá va uno de los errores más famosos de Python, y aparece naturalmente con esta sintaxis. Mirá esta función, que tiene pinta de inocente:

```python
def pedido(ingredientes=[]):
    ingredientes.append("choclo")
    return ingredientes
```

Parece que si no le pasás nada, arranca con una lista vacía, le agrega "choclo" y listo. Pero mirá qué pasa al invocarla varias veces *sin argumentos*:

```python
print(pedido())      # ['choclo']
print(pedido())      # ['choclo', 'choclo']   ← ?
print(pedido())      # ['choclo', 'choclo', 'choclo']  ← ??
```

¡Se acumula! La causa: la lista `[]` del `def` se crea **una sola vez**, cuando se define la función, y después todos los llamados comparten **el mismo** objeto. No es que cada llamada abra una lista nueva: el valor por defecto se evalúa **una sola vez** y se queda.

La solución clásica —la que vas a ver en código profesional y en las respuestas de Stack Overflow— es **no usar nunca un mutable como valor por defecto**. Se usa `None` y se crea la lista adentro, en cada llamada:

```python
def pedido_bien(ingredientes=None):
    if ingredientes is None:
        ingredientes = []
    ingredientes.append("choclo")
    return ingredientes

print(pedido_bien())    # ['choclo']
print(pedido_bien())    # ['choclo']   ← ahora sí, lista nueva cada vez
```

```python
pedido_con_lista = pedido_bien(["queso", "tomate"])
print(pedido_con_lista)   # ['queso', 'tomate', 'choclo']
```

> **Dato clave:** cuando definas una función, antes de guardarte este párrafo: **los valores por defecto mutables son una trampa**. Regla de oro: `def f(lista=None)` y dentro, `if lista is None: lista = []`. Siempre.

---

## 8. `*args` y `**kwargs`: cuando no sabés cuántos datos vienen

Ya sabés cuántos argumentos *esperar*, pero hay funciones a las que les puede llegar **cualquier cantidad**. ¿Cómo hacés? Enterándote de dos operadores que ya conocés con otros nombres: el `*` y el `**`.

### `*args`: cualquier cantidad de argumentos posicionales

Si anteponés un `*` a un parámetro, ese parámetro **atrapa todos los argumentos que sobren** en una tupla:

```python
def promedio(*muestras):
    return len(muestras), sum(muestras) / len(muestras)

promedio(1, 3, 5, 8, 11, 24, 90, 29)   # (8, 21.375)
promedio(10, 20, 30)                    # (3, 20.0)
```

El nombre `args` es una convención (la que usa todo el mundo), pero el `*` es lo importante. Y puede convivir con parámetros normales — siempre y cuando los normales vayan **adelante**:

```python
def promedio_con_titulo(titulo, *muestras):
    promedio_valor = sum(muestras) / len(muestras)
    return f"{titulo}: {promedio_valor:.2f}"

promedio_con_titulo("Conteo de abejas en campo", 34, 45, 61, 23, 47, 41, 52)
# 'Conteo de abejas en campo: 43.29'
```

El `*` también funciona al revés: en la llamada, le dice a Python *"desempaqueta esta colección en argumentos"*. Es la misma estrella que viste con el `match/case` del capítulo 5, pero acá reparte:

```python
notas = [7, 9, 8, 6]
promedio(*notas)       # (4, 7.5) → lo mismo que escribir promedio(7, 9, 8, 6)
```

Esto cierra el círculo: la receta construye tuplas con `*muestras`, y las llamadas pueden picar listas con `*lista`.

### `**kwargs`: cualquier cantidad de argumentos nombrados

Con doble asterisco, el parámetro atrapa los argumentos **por nombre** en un diccionario — clave *string*, valor el que venga:

```python
def superficie(**dato):
    match dato:
        case {"tipo": "rectángulo" as figura, "base": base, "altura": altura}:
            area = float(base) * float(altura)
        case {"tipo": "triángulo" as figura, "base": base, "altura": altura}:
            area = float(base) * float(altura) / 2
        case {"tipo": "círculo" as figura, "radio": radio}:
            area = 3.14159 * float(radio) ** 2
        case _:
            return "Figura desconocida"
    return f"Superficie del {figura}: {area:.2f}"

superficie(tipo="rectángulo", base=22, altura=30)   # Superficie del rectángulo: 660.00
superficie(tipo="círculo", radio=35)                # Superficie del círculo: 3848.45
superficie(base=22, altura=30)                      # Figura desconocida
```

Fijate la magia: cada figura pide claves distintas, y recién en la llamada se sabe cuáles vienen. `**dato` las atrapa todas en un diccionario, y el `match` entra con **patrones de diccionario** — los de la sección 7 del capítulo 5. Cada `case` exige la clave `tipo` con su valor exacto (`"rectángulo"`, `"triángulo"`, `"círculo"`) y las claves que la figura necesita, y las **desempaqueta en variables**: `base`, `altura`, `radio` quedan como nombres listos para usar, sin `dato["..."]` por todos lados.

Y acá aparece un patrón **AS** disfrazado: `"rectángulo" as figura` no es solo un literal — `as` le **rebautiza** el valor que coincidió. Sin eso, tendríamos que escribir `figura = "rectángulo"` adentro de cada caso para poder nombrarla en el mensaje final; con el `as`, Python lo guarda solo. Es exactamente el `as` que viste en `cenamos` del capítulo 5, ahora en su hábitat natural: un literal que además se vuelve variable.

Y hay un regalo disfrazado más: si a la función le falta la clave `tipo`, ninguna `case` coincide — no hay `KeyError` de por medio — y cae en `case _` devolviendo "Figura desconocida". La versión con `if` que escribiste mentalmente recién hubiera explotado.

Como con `args`, la estrella doble se invierte en la llamada para "desparramar" un diccionario:

```python
figura = {"tipo": "triángulo", "base": 4, "altura": 7}
superficie(**figura)    # Superficie del triángulo: 14.00
```

`kwargs` también tiene su caso de uso clásico en la vida real: "recibir configuraciones y pasarle a otra función solo lo que necesite". Pero no lo necesitás desde el día uno: alcanza con saber que existe la herramienta para cuando la encuentres.

> **Dato clave:** resumido en una línea: `*args` empaqueta los argumentos **de a muchos** en una tupla; `**kwargs` empaqueta los **nombrados** en un diccionario. Y en la llamada, `*lista` y `**dict` hacen el camino inverso. Esta dualidad (definir / invocar) es la parte que más se olvida — fijate que es la *misma* estrella, girando en los dos sentidos.

### `*` y `/`: las barreras que controlan cuándo se pasa por nombre

Recién usamos los asteriscos para *cantidad*. Ahora te muestro el uso que más te vas a encontrar en el código real y para el que nadie te prepara: **los separadores `*` y `/` dentro de la firma**. Son dos "barreras" que deciden, solo con mirar la firma, cómo puede cada parámetro recibir su argumento. El mapa completo de una firma puede verse así:

```python
def carta(a, b, /, c, *, d):
    """Las dos barreras en una sola firma."""
    return (a, b, c, d)

carta(1, 2, 3, d=4)          # (1, 2, 3, 4)  → posición + un nombre
carta(1, 2, c=3, d=4)        # (1, 2, 3, 4)  → "c" admite los dos estilos
carta(a=1, b=2, c=3, d=4)    # TypeError: got some positional-only arguments passed as keyword arguments: 'a, b'
carta(1, 2, 3, 4)            # TypeError: takes 3 positional arguments but 4 were given
```

Interpretando con la regla que te va a salvar la vida:

- lo que está **a la izquierda de `/`** acepta solo **posición**: `a` y `b` no se pueden nombrar (`carta(a=1, ...)` explota);
- lo que está **en el medio** acepta los dos estilos: `c` se puede pasar de las dos maneras;
- lo que está **a la derecha de `*`** acepta solo **nombre**: `d` es obligatorio nombrarlo (`carta(1, 2, 3, 4)` explota).

Con esas dos barreras podés leer cualquier firma del mundo. Y la parte que más te va a sorprender: las conocés sin saberlo, porque ya las usaste *mil* veces. La barrera `*` es la que hace que esto falle en silencio:

```python
print(1, 2, 3, "-")          # 1 2 3 -      ← pensaste pasar el separador...
print(1, 2, 3, sep="-")      # 1-2-3        ← ...y esto es lo que querías
```

`print` se define (más o menos) como `print(*objetos, sep=" ", end="\n", ...)`: la estrella de `*objetos` ya estaba haciendo de barrera, y todo lo que viene después — `sep`, `end` — es **obligatorio pasarlo por nombre**. Por eso el `"-"` suelto no da error: fue a parar a la lista de objetos a imprimir. Silencioso y tramposo. Lo mismo con `sorted`:

```python
sorted([3, 1, 2], reverse=True)   # [3, 2, 1]  → "reverse" se pide por su nombre
sorted([3, 1, 2], True)           # TypeError: sorted expected 1 argument, got 2
```

La firma real es `sorted(iterable, *, key=None, reverse=False)`; el famoso `key=...` que ordena listas de maneras raras solo existe así porque vive detrás de una `*` (te vas a enamorar de él en el próximo capítulo).

Y la barrera `/` es la de las funciones que no quieren que **nombres** sus argumentos, porque el nombre no aporta nada. Pensá en `range` y `divmod`:

```python
list(range(0, 10, 2))        # [0, 2, 4, 6, 8]
range(stop=5)                # TypeError: range() takes no keyword arguments

divmod(10, 3)                # (3, 1)  → cociente y resto de un golpe
divmod(a=10, b=3)            # TypeError: divmod() takes no keyword arguments
```

A `range` le da igual cómo se llame "el límite": el nombre existe solo para que la documentación hable de algo. Al bloquear los nombres, la función además queda libre de que mañana le *renombren* el parámetro sin romperte el programa — si vos no usabas el nombre, nada cambia.

¿Y para qué usarías las barreras en tus propias funciones? `*` fuerza a quien te llama a **decir qué opción está tocando** (`reverse=True` se entiende solo; `True` a secas, no), y te deja agregar opciones nuevas sin miedo a que un argumento posicional caiga en el lugar equivocado. `/` es para cuando los nombres no tienen significado (las funciones matemáticas), o para blindar tu propia API como hace `range`. No los vas a usar todos los días al arrancar, pero que no te asusten cuando los veas en la firma de una biblioteca: son el contrato diciendo *por dónde entra cada cosa*.

> **Dato clave:** las dos barreras se leen mirando sus flancos: la barra `/` le corta el *nombre* a lo que tiene a su izquierda (en `def f(a, /)`, llamarla `f(a=1)` explota); la estrella `*` le corta la *posición* a lo que tiene a su derecha (en `def f(*, b)`, llamarla `f(1)` explota). Y `*args` ya es una barrera nata: en `def f(a, *args, b)`, el parámetro `b` queda obligado a nombre de yapa — por eso `print` funciona como funciona.

---

## 9. Primer vistazo a los ámbitos

Ya tuviste un adelanto en el capítulo 3 cuando guardabas cosas en nombres. Ahora entra en juego el concepto que estructura todo lo que sigue: **los nombres tienen un territorio** — el *ámbito* (*scope*). El ámbito determina desde qué partes del programa puede utilizarse directamente un nombre. Está estrechamente relacionado con los **espacios de nombres (namespaces)**, que son los lugares donde Python mantiene las asociaciones entre nombres y objetos.
Esta idea aparece en la última línea del Zen de Python:
“Namespaces are one honking great idea — let’s do more of those!”

Es decir: «Los espacios de nombres son una idea fantástica; ¡hagamos más de ellos!».

Python maneja, a grandes rasgos, dos pedazos de mapa:

- **Ámbito global**: el nivel del programa, el espacio donde definís `objeto = "Hola"` fuera de toda función. Es el espacio de nombres del intérprete.
- **Ámbito local**: cada función crea **el suyo propio**, que nace cuando la invocás y muere cuando termina.

Lo que pasa adentro de una función es de la función. Este es el ejemplo canónico:

```python
objeto = "Hola"

def funcion():
    objeto = 2
    print(objeto)      # 2   → dentro, su propio "objeto"

funcion()              # 2
print(objeto)          # Hola → afuera, el global sigue intacto
```

La función no tocó tu variable global: creó una local en su propio ámbito. Las funciones, en general, **no saben ni tocan** lo que hay fuera.

¿Qué pasa cuando *leés* un nombre que no definiste localmente? Python busca en cadena: primero el ámbito local, después el global, y si nada coincide, `NameError`. Pensalo con este ejemplo:

```python
def trino():
    print(ave * 3)     # usa "ave" sin definirlo adentro

ave = "pio"
trino()                # piopiopio  → encontró el global y lo usó
```

Pero mirá bien el orden: `ave` no existía *cuando se definió* la función — existe *cuando se invoca*. El nombre se resuelve en el momento de la llamada. Si invocás `trino()` antes de definir `ave`, Python te tira `NameError`: `name 'ave' is not defined`, la excepción del capítulo 6 trabajando.

Para terminar el panorama, dos ayudas del intérprete que valen oro cuando te pierdas: `globals()` y `locals()` devuelven, como diccionario, el contenido del ámbito global y del local actual, respectivamente:

```python
def ambitos():
    lista = [1, 2, 3]
    print(locals())    # {'lista': [1, 2, 3]}  → lo que es mío
    print(len(globals()))   # cuántos nombres hay afuera

ambitos()
```

Quedate con la intuición de hoy: **la función ve lo suyo primero, y si no lo encuentra, mira afuera**. Los detalles finos (ahora sí, qué pasa con `global`, `nonlocal`, funciones dentro de funciones y las "cierraduras" — *closures*) los desarrollamos en el próximo capítulo. Este es el momento perfecto para parar: ya tenés la pieza más importante de toda la caja.

---

## 10. Resumen y ejercicios

**Resumen — el checklist de las funciones:**

- [ ] Una función es una receta con nombre; `def` la crea, `nombre()` la invoca.
- [ ] El cuerpo va indentado; `pass` es un marcador de posición.
- [ ] El docstring `"""..."""` documenta y alimenta a `help()`.
- [ ] `return` entrega un valor, corta la función, y sin él la función devuelve `None`.
- [ ] `print` muestra; `return` entrega. No son lo mismo.
- [ ] Los parámetros esperan valores; los argumentos los dan; deben coincidir en cantidad (`TypeError`).
- [ ] Los argumentos se pasan por posición o por nombre.
- [ ] Los valores por defecto van al final — y jamás uses un mutable como default (`def f(x=None)` a la rescate).
- [ ] `*args` y `**kwargs` atrapan cantidades variables de argumentos (tupla y dict).
- [ ] `*lista` y `**dict` desempaquetan en la llamada.
- [ ] `/` y `*` son barreras de la firma: a la izquierda de `/`, solo posición; a la derecha de `*`, solo nombre (`*args` ya hace de barrera).
- [ ] Cada función tiene su ámbito local; busca local → global, y falla con `NameError`.

Ese último punto es la puerta del próximo capítulo, junto con las funciones anidadas y las *closures* — pero antes, practicá.

**Ejercicios:**

1. **Saludo**: definí `saludar(nombre)` que devuelva `"Hola, {nombre}!"` (con `return`, no `print`). Probala con tres nombres.
2. **`es_par`**: definí `es_par(numero)` que devuelva `True` si el número es par y `False` si no, usando `return temprano`.
3. **Máximo de tres**: definí `maximo_de_tres(a, b, c)` que devuelva el mayor, sin usar la función `max`. Usá `return` temprano o comparaciones encadenadas.
4. **Promedio flexible**: definí `promedio(*notas)` que devuelva el promedio como `float`. Probala con 2, 3 y 6 notas.
5. **Ficha por nombre**: definí `mostrar_ficha(**ficha)` que imprima cada `clave: valor` en una línea. Probala con una ficha de persona y otra de producto.
6. **La trampa del default**: definí `agregar_ingrediente(ingrediente, lista=None)` que agregue el ingrediente a una lista nueva, o a la lista que le pasen, y la devuelva. Llamala tres veces sin pasar lista y verificá que cada resultado tenga un solo ingrediente.
7. **Firma con barreras**: definí `configurar(nombre, /, *, host="localhost", puerto=8080)` que devuelva un diccionario con la configuración. Probá las llamadas válidas y comprobá que `configurar(nombre="servidor")` y `configurar("servidor", "x")` explotan con `TypeError`.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. Saludo con return
def saludar(nombre):
    return f"Hola, {nombre}!"

for nombre in ["Ana", "Luis", "Eva"]:
    print(saludar(nombre))
# Hola, Ana!
# Hola, Luis!
# Hola, Eva!

# 2. es_par con return temprano
def es_par(numero):
    if numero % 2 == 0:
        return True
    return False

print(es_par(10))   # True
print(es_par(7))    # False

# 3. Máximo de tres
def maximo_de_tres(a, b, c):
    if a >= b and a >= c:
        return a
    if b >= c:
        return b
    return c

print(maximo_de_tres(3, 9, 5))   # 9
print(maximo_de_tres(7, 2, 7))   # 7

# 4. Promedio flexible
def promedio(*notas):
    return sum(notas) / len(notas)

print(promedio(7, 8))            # 7.5
print(promedio(6, 7, 8, 9, 10, 5))  # 7.5
print(promedio(10, 9, 8))        # 9.0

# 5. Ficha por nombre
def mostrar_ficha(**ficha):
    for clave, valor in ficha.items():
        print(f"{clave}: {valor}")

mostrar_ficha(nombre="Ana", edad=30, ciudad="Misiones")
# nombre: Ana
# edad: 30
# ciudad: Misiones

mostrar_ficha(producto="té", precio=250, stock=18)
# producto: té
# precio: 250
# stock: 18

# 6. La trampa del default, bien resuelta
def agregar_ingrediente(ingrediente, lista=None):
    if lista is None:
        lista = []
    lista.append(ingrediente)
    return lista

print(agregar_ingrediente("choclo"))       # ['choclo']
print(agregar_ingrediente("queso"))        # ['queso']     ← no se acumula
print(agregar_ingrediente("tomate"))       # ['tomate']

mezcla = ["salsa"]
print(agregar_ingrediente("albahaca", mezcla))   # ['salsa', 'albahaca']

# 7. Firma con barreras
def configurar(nombre, /, *, host="localhost", puerto=8080):
    return {"nombre": nombre, "host": host, "puerto": puerto}

print(configurar("servidor"))
# {'nombre': 'servidor', 'host': 'localhost', 'puerto': 8080}
print(configurar("servidor", host="192.168.0.1", puerto=3000))
# {'nombre': 'servidor', 'host': '192.168.0.1', 'puerto': 3000}

# configurar(nombre="servidor")  → TypeError: got some positional-only arguments passed as keyword arguments: 'nombre'
# configurar("servidor", "x")    → TypeError: takes 1 positional argument but 2 were given
```

Ya sabés definir funciones, pasarles datos de a uno o de a muchos, y devolver resultados. Con esa pieza, la máxima del capítulo cobra sentido: **no estás reescribiendo la rueda, estás fabricando tu propio ladrillo**. El próximo paso del libro te va a mostrar el territorio completo donde viven los nombres (los ámbitos en serio), las funciones que viven dentro de otras funciones, y las lambdas — las funciones que no necesitan ni nombre. Ahí sí vamos a empezar a armar castillos.