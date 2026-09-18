# Capítulo 8 — Funciones II: ámbitos, funciones anidadas, closures y lambdas

En el capítulo anterior construiste tu primer ladrillo: la función. Pero hay un secreto que no te conté del todo. Las funciones no viven solas en el vacío: viven **dentro de un mapa de nombres**, pueden nacer *dentro de otras funciones*, pueden recordar su entorno después de morir, y hay una versión de ellas que ni siquiera necesita nombre. Este capítulo es la segunda parte de la *Parte V*: el territorio completo donde viven los nombres (`global` y `nonlocal`), las funciones anidadas, las **closures** y las **lambdas**. Cuando termines, vas a entender por qué una función puede devolver *otra función* — y eso te va a dejar en la puerta de los **decoradores**, uno de los trucos más elegantes de Python.

---

## 1. Los ámbitos en serio: leer no es escribir

Arrancamos donde quedó el capítulo 7: cada función crea su propio ámbito local, y la búsqueda de un nombre va de lo local a lo global. Hoy te doy la pieza que faltaba, la que explica el 90% de los dolores de cabeza de la gente: **leer una variable global es distinto de escribirla**.

Mirá este par de funciones:

```python
# Leer: sí se puede
def que_dia_es():
    return dia            # usa "dia" sin definirla adentro

dia = "lunes"
que_dia_es()              # lunes → encontró el global

# Escribir: crea una local (REPASO del capítulo 7)
def cambiar_dia():
    dia = "martes"        # ¡esto NO toca el global!

cambiar_dia()
print(dia)                # lunes → el global quedó intacto
```

La lectura es "hacia afuera": la función mira, y si no encuentra el nombre adentro, lo busca en el ámbito global. La escritura es "hacia adentro": si una función *asigna* un nombre (`dia = ...`), ese nombre pasa a ser **local a la función**, pase lo que pase afuera. Ese comportamiento tiene nombre: **sombreado** (*shadowing*); la variable local le hace sombra a la global mientras la función vive.

Hay una trampa sutil que muestra esto con claridad brutal. Mirá:

```python
def sintomas():
    print("síntoma:", sintoma)     # ¿funcionará?
    sintoma = "fiebre"
```

Si invocás `sintomas()`, Python te tira un error que sorprende a medio mundo:

```python
# UnboundLocalError: local variable 'sintoma' referenced before assignment
```

¿Por qué, si la línea `print` viene primero? Porque Python decide el "mapa" **antes de ejecutar**: al ver que *en algún lugar* de la función hay `sintoma = ...`, declara a `sintoma` como variable local de arriba a abajo. La línea del `print` ya está leyendo la local... que todavía no fue asignada. Moraleja incómoda: si asignás un nombre adentro de una función, no esperes usar *el mismo nombre* para leer el global — para Python ya son dos cosas distintas.

Para que el mapa se haga visible, el intérprete te da dos espejos que ya saludamos en el capítulo 7: **`locals()`** muestra el ámbito local actual como diccionario, y **`globals()`** muestra el global. La diferencia entre los dos se siente *adentro* de una función:

```python
precio = 100                       # global

def recargo():
    extra = precio * 0.10          # leemos el global
    return extra

recargo()                          # 10.0

def mostrar_locales():
    local = "solo existo acá"
    return locals()

mostrar_locales()                  # {'local': 'solo existo acá'} → el ámbito local en acción
globals()["precio"]                # 100 → el global también se lee como un dict
```

Afuera de toda función, `locals()` y `globals()` son *el mismo* mapa; la separación real aparece cuando entrás a una función, y `locals()` te muestra el cajón privado que se te abre en ese momento.

> **Dato clave:** el sombreado es *leer afuera, escribir adentro*. Tu función puede mirar las variables del programa, pero si asigna un nombre, ese nombre se vuelve privado. Y si asignás y después querés "mirar" ese mismo nombre como global, `UnboundLocalError`. La salida limpia para "devolver un dato cambiado" siempre es la misma: **`return`** (capítulo 7), no tocar variables ajenas.

---

## 2. `global`: escribir en el ámbito global, a propósito

¿Y si *de verdad* querés que una función modifique una variable del programa? Existe la palabra clave `global`. Declarala **dentro de la función, antes de asignar**, y el nombre apunta al ámbito global ya no a una local:

```python
total = 0

def sumar_al_total(x):
    global total
    total += x

sumar_al_total(10)
sumar_al_total(5)
print(total)            # 15 → la función SÍ tocó el global
```

Funciona, pero prestá atención a lo que acaba de pasar: la función ahora tiene un **efecto lateral** invisible — no devuelve nada, cambia algo que está afuera. Eso es exactamente lo que hace al código difícil de seguir: apretás el botón, y se mueve una pesa del otro lado del cuarto sin que la llamada lo anuncie.

Por eso la regla de oro del profesional es: **`global` es una rareza, no una costumbre**. El caso legítimo y clásico es el **contador** — una variable que va sumando llamadas, como para medir cuántas veces se abrió una cosa. Para "devolver un resultado", siempre preferís `return`. La existencia de `global` es el espejo que te muestra *por qué* `return` es tan valioso: te evita escribir en el mapa de otro.

> **Dato clave:** `global` te deja *escribir* el ámbito global desde adentro de una función — pero cada vez que lo uses, preguntate si podrías resolverlo con un `return`. Son contadas las veces en un programa real que la respuesta es sí (un contador global, una config). Y hay una regla del idioma: declará `global` **antes** de usar el nombre, en la primera línea del cuerpo.

---

## 3. Funciones anidadas: funciones dentro de funciones

Si las funciones son bloques, los bloques pueden contener bloques. Python permite **definir una función dentro de otra función**, y la interna es tan "local" como una variable: nace con la llamada de la externa, muere con ella, y **nadie afuera la conoce**. Ese es justamente su propósito: helpers íntimos que solo tienen sentido dentro de la receta.

El ejemplo clásico de la materia es una función que genera los números primos hasta un límite. La versión que te muestro define adentro un ayudante que decide si un número es primo:

```python
def lista_primos(limite=100):
    """Genera la lista de primos entre 2 y limite."""
    primos = [2]

    def esprimo(numero):
        for primo in primos:
            if numero % primo == 0:
                return False
        return True

    for numero in range(3, limite + 1):
        if esprimo(numero):
            primos.append(numero)

    return primos

lista_primos(30)
# [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

Tres observaciones importantes:

- **`esprimo` no existe afuera.** Ejecutá `esprimo(5)` en el intérprete y te va a dar `NameError`: es del club privado de `lista_primos`.
- **La interna lee variables de la externa.** `esprimo` usa `primos`, que pertenece a `lista_primos`. Está *leyendo* el ámbito de su madre — sin conflictos, con las reglas del capítulo anterior.
- **La externa se beneficia.** El bucle principal queda limpio: no tiene que saber *cómo* se decide la primacía, solo preguntar. Organización pura, y un adelanto de algo enorme: una función que usa a otra como si fuera una herramienta.

Las funciones anidadas se usan todo el tiempo, y no solo con números: helpers de validación "(este dato es válido para mi lógica y para nadie más)", pasos internos de una receta larga... Y ahora viene lo que las vuelve fascinantes: cuando una función anidada **sale al mundo**.

> **Dato clave:** definir una función adentro de otra es como guardar una herramienta en el cajón de la cocina de la receta: accesible para la receta, invisible para el resto de la casa. Y la interna puede **leer** tranquila el ámbito de la externa.

---

## 4. Closures: funciones que devuelven funciones que recuerdan

Acá se te va a doblar un poco la cabeza, y está bien: es el momento más jugoso de la parte de funciones.

Dijimos que si una función define a otra, la interna es local. ¿Y si la externa **devuelve** la interna? El llamado termina, el ámbito de la externa "debería" morir... pero la función devuelta sigue agarrada al entorno donde nació. A ese objeto —**función + entorno recordado**— se lo llama **closure** (*"clausura"* o *"cerradura"*): una función que devuelve otra función que *recuerda* su entorno.

Mirá este ejemplo, el más didáctico que existe:

```python
def crear_saludo(apellido):
    def saludar(nombre):
        return f"{nombre} {apellido}"   # usa "apellido", el dato de la madre
    return saludar

saludo_de_lopez = crear_saludo("López")
saludo_de_garcia = crear_saludo("García")

print(saludo_de_lopez("Ana"))     # Ana López
print(saludo_de_garcia("Leo"))    # Leo García
```

Fijate la magia: `crear_saludo` terminó hace rato, su parámetro `apellido` "debería" evaporarse... pero `saludo_de_lopez` y `saludo_de_garcia` son **dos funciones distintas** que recuerdan cada una *su* `apellido`. Cada llamada a `crear_saludo` fabricó una closure con su propio recuerdo. Es una fábrica de funciones personalizadas.

El caso más emblemático: la **fábrica de contadores**, donde además nos toca `nonlocal`.

### `nonlocal`: la hermana de `global` para funciones anidadas

¿Y si la función interna quiere **escribir** en el ámbito de la externa, no solo leerlo? Con `global` ya viste que el intérprete separa estrictamente los ámbitos; escribir en el de "la madre" desde la hija se declara con **`nonlocal`**. 

El ejemplo clásico es este:

```python
 = 0                      # global

def uno():
    x = 15                 # local de uno
    print("en la funcion uno x vale:", x)

    def dos():
        nonlocal x         # la x de uno, no la global
        x = 10
        return x

    print("en la funcion interna dos x vale:", dos())
    print("en la funcion uno después de llamar a la funcion dos x vale:", x)

uno()
print("en el ámbito global vale:", x)
```

Salida :

```text
en la funcion uno x vale: 15
en la funcion interna dos x vale: 10
en la funcion uno después de llamar a la funcion dos x vale: 10
en el ámbito global vale: 0

``` 


Leé la salida con paciencia: hay **tres** `x` distintas en juego. `dos` con `nonlocal x` escribe en la `x` de `uno` (la madre), por eso `uno` pasa de `15` a `10`. Y la `x` global ni se entera: sigue `0`.

`nonlocal` se vuelve imprescindible en el contador con closure, porque el contador necesita *actualizar* una variable de su entorno capturado:

```python
def hacer_contador():
    cuenta = 0
    def incrementar():
        nonlocal cuenta        # sin esto, UnboundLocalError
        cuenta += 1
        return cuenta
    return incrementar

contador = hacer_contador()
contador()            # 1
contador()            # 2
contador()            # 3
```

Si sacaras el `nonlocal`, `cuenta += 1` escribiría una local de `incrementar` y explotaría con `UnboundLocalError` — la misma trampa de la sección 1, ahora adentro de la closure. El `nonlocal` le dice al intérprete: "no, esa `cuenta` es de mi madre, conectámela a esta". Ese contador que "pide turno" y se acuerda solo dónde quedó es la cara más útil de las closures.

> **Dato clave:** una **closure** es una función que se lleva su entorno de recuerdo. `nonlocal` es el `global` de las funciones anidadas: sirve para *escribir* en el ámbito de la función madre. Sin él, la interna solo puede leer; escribiendo, se cae en `UnboundLocalError`. Si te toca enfrentarte a una closure y no sabés si "recuerda" un valor, sí, lo recuerda: ese recuerdo es la definición misma.

---

## 5. Funciones de orden superior: pasar funciones como argumentos

Ya viste una función devolver a otra. Ahora la otra mitad: **pasar una función como argumento**. A las funciones que reciben funciones, devuelven funciones, o ambas, se las llama **funciones de orden superior** (de *"higher-order functions"*). Son la base del capítulo del paradigma funcional.

Tu primer contacto, sencillo y honesto:

```python
def aplicar(funcion, valor):
    """Aplica funcion a valor y devuelve el resultado."""
    return funcion(valor)

def duplicar(n):
    return n * 2

aplicar(duplicar, 21)      # 42 → le pasaste una función como dato
```

Nada místico: `funcion` es un parámetro como cualquier otro; simplemente resulta que el valor que llega es *invocable*. Fijate que pasás `duplicar` **sin paréntesis** — con paréntesis estarías pasando su resultado, no la función. Ese detalle es el error #1 cuando la gente arranca con orden superior.

¿Para qué sirve? Para escribir funciones que *personalizan* comportamiento. El ejemplo de la materia (de `13_funciones.ipynb`) envuelve texto en etiquetas HTML:

```python
def envolver(funcion):
    """Devuelve una función que envuelve el resultado en una caja `<div>`."""
    etiquetas = "<div>{}</div>"

    def empaqueta(texto):
        return etiquetas.format(funcion(texto))

    return empaqueta

def parrafo(texto):
    return f"<p>{texto}</p>"

print(parrafo("Hola"))                       # <p>Hola</p>
print(envolver(parrafo)("Hola"))               # <div><p>Hola</p></div>
```

`envolver` fabrica una función que envuelve al resultado de `parrafo` en un `<div>` por fuera — para que quede claro, `<div>` es la caja contenedora real de HTML (la que agrupa bloques y le da estructura a una página), así que acá no inventamos ninguna etiqueta rara: estamos viendo cómo se anida la maquinaria de las funciones, con HTML real de juguete. Acá se juntó todo lo del capítulo: funciones anidadas + closure (empaqueta se lleva a `funcion` y `etiquetas`) + funciones como argumento + función que devuelve función.

¿Y ese es el gancho que te falta para los decoradores: **un decorador es exactamente una orden superior** — recibe una función y la devuelve lista para usar — con un poquito de azúcar sintáctica. Cuando veas en el futuro

```python
@envolver
def parrafo(texto):
    return f"<p>{texto}</p>"
```

lo que pasa por debajo es `parrafo = envolver(parrafo)`: el `@` le dice a Python "aplica `envolver` a la función que viene". Pero hay un detalle que en ese ejemplo pasa desapercibido: el `@` **no tiene dónde pasarle los argumentos extra**. ¿Y si la personalización, además, necesita *configuración*?

Ese es justamente este patrón: una orden superior que recibe la función destino **y un parámetro de configuración** (el borde `v`), y devuelve la función lista para imprimir el texto enmarcado — como los títulos decorados de un archivo de código:

```python
def decorador_encabezado(func, v="*"):
    """Devuelve una función que imprime un texto enmarcado con un borde de "v"."""
    def banner(texto):
        borde = len(texto) * v            # tantas "v" como letras tenga el texto
        marco = f"{borde}\n{texto}\n{borde}"
        func(f"{marco}\n")                # la función destino hace la impresión
    return banner

def display_text(texto):
    print(texto)

encabezar = decorador_encabezado          # lo usamos "a mano", eligiendo el borde
encabezar(display_text, "#")("hola jan")
encabezar(display_text, ".")("genial no?")
```

Salida:

```text
########
hola jan
########

..........
genial no?
..........
```

Fijate qué siguiente escalón perfecto es del ejemplo anterior: `banner` es una **closure** que se lleva *dos* recuerdos — a `v` (el borde configurado) y a `func` (el destino que imprime) — y `decorador_encabezado` es una orden superior que devuelve otra función. La configuración viaja como el dato más común del mundo: un parámetro y listo.

Ahora, la pregunta que te está quemando: ¿podemos usar esa configuración con la sintaxis `@`? No directamente: `@decorador_encabezado` **andaría**, pero te dejaría el borde por defecto (`"*"` a cara de perro), porque `@` aplica la función tal cual, sin argumentos de más. Para **configurar con `@`**, la receta se reordena en tres pisos — una **fábrica** que primero recibe la configuración y recién después se convierte en decorador:

```python
def decorador_encabezado(v="*"):            # 1er piso: la fábrica (configuración)
    def recibe(func):                       # 2do piso: el decorador
        def banner(texto):                  # 3er piso: el envoltorio
            borde = (len(texto) + 4 ) * v
            marco = f"{borde}\n{v} {texto} {v}\n{borde}"
            func(f"{marco}\n")
        return banner
    return recibe

@decorador_encabezado("#")
def display_text(texto):
    print(texto)

display_text("Que lindo queda !!!")
```

```text
#######################
# Que lindo queda !!! #
#######################
```

¿Qué pasó recién? `@decorador_encabezado("#")` ejecuta la fábrica, que devuelve `recibe` con el `"#"` guardado en el bolsillo; con eso Python hace `display_text = recibe(display_text)`, y `display_text` queda siendo el envoltorio de tres pisos. Es *exactamente* la misma figura que `@envolver` del ejemplo anterior — ahora con configuración. Y acá está la madurez de la parte de funciones: ya sabés *todos* los ladrillos (funciones anidadas, closures, orden superior, fábricas); el capítulo de decoradores se encarga de montarlos en serie (contadores de llamadas, medición de tiempos, permisos...).

> **Dato clave:** "pasar una función" es pasarla **sin paréntesis** (`aplicar(duplicar, 21)`), porque con paréntesis pasás su resultado. Una función que recibe y/o devuelve funciones es de **orden superior** — y es la mamá de los decoradores (`@` aplica una orden superior con azúcar sintáctico: `@dec` ≡ `f = dec(f)`). Y ojo: el `@` no tiene dónde pasar argumentos; si el decorador necesita configuración (como el `v` del encabezado), se usa una **fábrica**: `@decorador_encabezado("#")` ejecuta la fábrica y aplica el decorador que devuelve.

---

## 6. Lambdas: funciones sin nombre

De vez en cuando necesitás una función tan pequeña y momentánea que definirla con `def` y ponerle nombre parece desperdiciar dos líneas y un nombre del mapa. Para esos casos existe la **lambda** (de *"lambda"*, el cálculo lambda de la matemática): una **función anónima** de una sola expresión.

Sintaxis: `lambda <argumentos>: <expresión>`. La expresión se evalúa y se devuelve — sí, un lambda tiene `return` implícito, y solo ese, porque una expresión no puede tener lógica complicada.

```python
es_par = lambda n: n % 2 == 0

es_par(4)     # True
es_par(7)     # False
```

Fijate como se traduce: "una función que recibe `n` y devuelve `n % 2 == 0`". Pueden llevar varios parámetros, y valores por defecto — como aca:

```python
saludo = lambda x, y="hola": y + " " + x

saludo("emi")            # hola emi
saludo("emi", "buenas")  # buenas emi
```

Y como cualquier función, se puede invocar **en el acto**, sin asignarla a nada:

```python
(lambda x: x * 2)(5)      # 10 → una función anónima usada ni bien nace
```

Pero ahora la pregunta honesta: si `es_par = lambda n: ...` escribe lo mismo que un `def`, ¿por qué no usar siempre `def`? Por dos motivos muy concretos:

1. **La lambda brilla como argumento.** Es en los orden superiores donde las anónimas tienen sentido: cuando necesitás una función *solo para pasarla a otra*. El ejemplo más visto en código real es ordenar con una clave:

```python
nombres = ["Ana", "roberto", "luis", "EVELYN"]
print(sorted(nombres))                    # ordena por valor Unicode (las mayúsculas primero)
print(sorted(nombres, key=lambda nombre: nombre.lower()))
# ['Ana', 'EVELYN', 'luis', 'roberto']   → ordena ignorando mayúsculas
```

`key=` recibe una función que le dice a `sorted` "qué mirar" de cada elemento. Esa función rara vez merece un `def` + nombre; la lambda es perfecta.

2. **`def` es mejor para funciones con nombre.** Si vas a reutilizar la función en varios lugares, darle nombre y docstring gana siempre. El estilo oficial (PEP 8) hasta recomienda *no* asignar una lambda a una variable si podés usar `def`: `es_par = lambda n: ...` es legal, pero la escritura idiomática es `def es_par(n): return n % 2 == 0`. La lambda se reserva para el momento volátil: "la necesito acá, ahora, y no la vuelvo a ver".

> **Dato clave:** lambda = función anónima de una sola expresión, con `return` implícito. Brilla como *argumento* de orden superior (`key=lambda x: ...`); es una práctica pobre asignarla a un nombre que podría tener un `def`. Regla mnemotécnica: **lambda al vuelo, def para la casa**.

---

## 7. Resumen y ejercicios

**Resumen — el checklist de las funciones II:**

- [ ] Leer un global desde una función: se puede. Escribir un nombre adentro: crea una *local* (sombreado).
- [ ] Si asignás y después leés el mismo nombre en la misma función → `UnboundLocalError`.
- [ ] `globals()` y `locals()` muestran los ámbitos como diccionarios.
- [ ] `global` escribe en el ámbito global de forma deliberada — úsalo rarísima veces (contador); casi siempre `return` gana.
- [ ] Las funciones anidadas son privadas de la externa y pueden **leer** el ámbito de su madre.
- [ ] Una **closure** es una función devuelta que recuerda su entorno (`crear_saludo`, `hacer_contador`).
- [ ] `nonlocal` permite a la interna **escribir** en el ámbito de la madre (contador con closure).
- [ ] Las **funciones de orden superior** reciben y/o devuelven funciones; pasar una función es pasarla sin paréntesis.
- [ ] Decorador = orden superior con azúcar `@` → `@dec` ≡ `f = dec(f)`; para pasarle configuración, una **fábrica** devuelve el decorador (`@dec("#")`).
- [ ] Las **lambdas** son funciones anónimas de una sola expresión, ideales como argumento (`sorted(key=...)`); no las asignes a nombres innecesariamente.

Con esto ya tenés la maquinaria completa de las funciones: definirlas, darles parámetros, y ahora también *armarlas escalonadas* (anidadas, con recuerdos, anónimas, en fábricas). El próximo capítulo no es magia nueva todavía — es ponerle el **plano** a todo esto: los type hints, las etiquetas que documentan cada firma. Y recién después llega la manera más sorprendente de usar lo que construiste: **funciones que se llaman a sí mismas**.

**Ejercicios:**

1. **Contador global**: definí `seguimiento()` que use `global` para incrementar una variable `pasos` y la devuelva. Llamala 4 veces y mostrá la salida.
2. **Fábrica de multiplicadores**: definí `crear_multiplicador(n)` que devuelva una función que multiplica su argumento por `n`. Fabricá `por_2` y `por_3` y probalas.
3. **Contador con closure**: reescribí `hacer_contador()` del capítulo para que acepte un inicio (`hacer_contador(10)` arranca en 10). ¿Dónde tiene que vivir `nonlocal`?
4. **Helper anidado**: definí `es_cercano_a(limite, tolerancia)` que use una función anidada `dentro(valor)` (devuelve `True` si `abs(valor - limite) <= tolerancia`) y devuelva esa función. Probala con valores.
5. **Lambda con `sorted`**: dada `palabras = ["hola", "mundo", "py", "a"]`, ordenala por **longitud** usando `key=lambda`.
6. **Lambda al vuelo**: usá la invocación inmediata `(lambda ...)(...)` para calcular `(3 + 5) * 2` sin definir ninguna función con `def`.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. Contador global
pasos = 0

def seguimiento():
    global pasos
    pasos += 1
    return pasos

for _ in range(4):
    print(seguimiento(), end=" ")        # 1 2 3 4
print()

# 2. Fábrica de multiplicadores
def crear_multiplicador(n):
    def multiplicar(x):
        return x * n
    return multiplicar

por_2 = crear_multiplicador(2)
por_3 = crear_multiplicador(3)
print(por_2(10), por_3(10))              # 20 30

# 3. Contador con closure y valor inicial
def hacer_contador(inicio=0):
    cuenta = inicio
    def incrementar():
        nonlocal cuenta
        cuenta += 1
        return cuenta
    return incrementar

c10 = hacer_contador(10)
print(c10(), c10())                      # 11 12

# 4. Helper anidado que se devuelve
def es_cercano_a(limite, tolerancia):
    def dentro(valor):
        return abs(valor - limite) <= tolerancia
    return dentro

cerca_de_50 = es_cercano_a(50, 5)
print(cerca_de_50(52), cerca_de_50(60))  # True False

# 5. Lambda con sorted por longitud
palabras = ["hola", "mundo", "py", "a"]
print(sorted(palabras, key=lambda p: len(p)))   # ['a', 'py', 'hola', 'mundo']

# 6. Lambda al vuelo
print((lambda a, b: (a + b) * 2)(3, 5))   # 16
```

Cerramos la segunda mitad de las funciones. Ya sabés *armar* funciones y *fabricar* funciones que fabrican funciones. El próximo capítulo es un lindo descanso de la magia: les vamos a poner el plano a tus funciones, los **type hints** (las etiquetas que documentan cada firma). 

Y en el que viene después sí llega una de las ideas más hermosas de la programación — que usa todo lo que construiste: una función que se llama a sí misma, hasta que el problema se vuelve tan chico que se resuelve solo. Bienvenido en dos capítulos a la **recursión**.