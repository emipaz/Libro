# Capítulo 29 — Configuración: el programa que no se trae todo adentro

Los dos capítulos anteriores te dieron lo que hace falta para que un programa viva fuera de tu cabeza: el 27 te dio archivos, el 28 te dio formatos. Te faltó la pieza que hace que todo eso sirva de algo, y es la que decide si un programa es flexible o es una piedra.

Un programa real rara vez tiene los mismos valores en la máquina de quien lo escribió y en la de quien lo usa. El puerto cambia, la clave de la base de datos cambia, la ruta de los archivos cambia, el nivel de detalle de los mensajes cambia. Un programa que tiene esos valores escritos adentro no es un programa: es un retrato. Funciona en una sola casa y se rompe en la primera mudanza.

La salida es la configuración: un conjunto de valores que el programa **no tiene** y que recibe de afuera al arrancar. Y esa idea, que parece una formalidad, trae consigo una pregunta que va a ser la que vertebra este capítulo entero: **si el valor puede venir de cuatro lugares distintos, cuál gana?** No es una pregunta retórica. Es la pregunta que responde por qué dos personas que ejecutan el mismo archivo pueden obtener resultados distintos, y por qué el mismo programa se comporta bien en desarrollo y se rompe en producción.

Vamos a construir esa máquina de capas de a una: primero el formato más viejo y más simple que vas a ver, después las variables de entorno que llegan desde el sistema operativo, después el formato que todo el mundo usa hoy, y por último un contrato que junte todo eso y te diga qué valor terminó siendo cada campo. Y vamos a verificar cada afirmación con el intérprete real, porque en este terreno la documentación y la verdad se separan seguido.

Lo que vas a poder hacer al terminar: leer un archivo `.ini` sin que te sorprenda nada; entender por qué `list.insert(0, x)` te costó caro y esta vez aplicarlo al puerto y no a la lista; distinguir un JSON de un TOML de un YAML mirando una sola línea; y armar un cargador donde el entorno le pise al archivo y el archivo le pise al valor por defecto del código, sin escribir ni un `if` para decidir quién gana.

---

## 1. El problema: por qué la configuración no es un `if` gigante

Antes de hablar de formatos, conviene ver el problema de la otra manera: qué pasa cuando **no** hay configuración. El camino que recorren casi todos los proyectos al principio, y que no es un descuido de quien recién empieza sino una consecuencia natural de no tener el problema todavía, es este:

```python
import os

# Un programa con todo adentro
DEBUG = False
PUERTO = 8080
BASE_DATOS = "postgres://usuario:clave@localhost:5432/app"
RUTA_ARCHIVOS = "./datos"
TIMEOUT = 30

def main():
    if DEBUG:
        print("iniciando en modo prueba")
    conexion = conectar(BASE_DATOS, puerto=PUERTO, timeout=TIMEOUT)
    procesar(conexion, RUTA_ARCHIVOS)
```

Funciona. Se ejecuta, hace lo que tiene que hacer, y si lo probaste en tu máquina, funciona perfecto. El problema es que cada uno de esos cinco valores es una decisión que tomaste **una vez y para siempre**, y hay varias personas que necesitan cambiarla cada una por su cuenta: la que despliega en el servidor, la que corre las pruebas en otro puerto, la que prueba en Windows y en Linux, y la que quiere ver los mensajes de detalle mientras depura.

Y no es solo que no puedan cambiarlos. Es que **cambiar cualquiera de los cinco obliga a tocar el código**. Para pasar de puerto 8080 a 9090 hay que abrir el archivo, editar una línea, y desplegar una versión nueva. Eso no es un problema de comodidad: es que el valor correcto del puerto no es una verdad del programa, es una verdad **del entorno donde corre**. Y si la verdad no está en el programa, no puede cambiar sin que el programa cambie.

> **Dato clave:** la configuración no es "una lista de valores que se pueden cambiar". Es la separación explícita entre lo que el programa **es** — su lógica, sus reglas, su estructura — y lo que el programa **hace en este lugar y en este momento**: el puerto de esta máquina, la ruta de esta carpeta, el usuario de esta base. Lo primero no cambia entre dos corridas del mismo programa. Lo segundo cambia siempre. Un programa que no hace esa separación tiene su identidad atada a un lugar.

Hay tres salidas posibles a este problema, y las tres existen. Podés escribir los valores en el código y editarlos cuando haga falta (lo que acabamos de ver: no escala). Podés leerlos de un archivo (funciona, es lo que hace casi todo el mundo). O podés leerlos del **entorno**, del sistema operativo, de una capa que está por debajo del programa y que el programa no controla. Las dos últimas son configuración de verdad, y casi siempre aparecen juntas.

La razón por la que esta pregunta importa más de lo que parece es que el orden en que se consultan las capas es, literalmente, la diferencia entre un programa que se puede desplegar y uno que no. Un programa donde el entorno le pisa al archivo, y un programa donde el archivo le pisa al entorno, ejecutan el mismo archivo con los mismos valores escritos adentro. Lo único que cambia es el orden de las preguntas.

Y hay una trampa más, la más sutil de todas: **los valores de configuración no tienen tipos**. Un puerto es un entero. Un puerto *escrito* en un archivo de texto es una cadena de cinco caracteres. Esa diferencia — entre el valor que querés y el valor que llegó — es la causa número uno de errores de configuración, y es la que vamos a pasar el capítulo entero persiguiendo. Porque recién vas a ver que la herramienta que te salva no es un formato mejor: es un **contrato** que dice qué tipo tiene cada campo y se niega a aceptar nada que no lo cumpla.

---

## 2. Las cuatro fuentes y su orden de prioridad

Vamos a poner las cartas sobre la mesa antes de ver las herramientas. Existen cuatro lugares de donde puede venir un valor, y no es que uno sea "el bueno": los cuatro existen porque cada uno resuelve un problema distinto.

1. **Un valor por defecto, escrito en el código.** El puerto es 8080 porque no me dieron otro. Es el piso: nunca falta, y es el único que viaja con el programa. Pero es el menos específico: sirve para desarrollo y para nada más.

2. **Un archivo de configuración.** Un `app.ini` en el disco. Almacena lo que el operador de esa máquina quiere. Se versiona junto al proyecto (a veces), se edita a mano, es cómodo para desarrollo. Su problema: si está en el repositorio, alguien lo edita sin que nadie lo vea; y en el servidor, el archivo puede ser uno distinto al que vos tenés en tu máquina.

3. **Una variable de entorno.** La inyecta el sistema operativo o el contenedor donde corre el proceso. No está en ningún archivo, no se versiona, no se puede "ver" mirando el repositorio, y por eso es el mecanismo natural para **el secreto** y para lo que cambia entre réplicas. Su problema: no tiene estructura, no tiene comentarios, y escribir una línea de texto en un panel de una consola remota no es cómodo.

4. **Un argumento de la línea de comandos.** `--puerto 9090`. El más explícito y el más difícil de automatizar en masa si tu despliegue tiene cien servicios.

Los cuatro existen. La pregunta del capítulo es qué pasa cuando **dos de ellos dicen cosas distintas** al mismo tiempo. Y la respuesta —que es una convención casi universal, no un accidente— es una pirámide. La fuente más específica gana sobre la menos específica, y el orden de especificidad es este:

| Fuente | Específica | ¿La ve un humano? | Ejemplo |
|---|---|---|---|
| Argumento | Máxima | Sí, si lo escribe | `--puerto 9090` en el comando |
| Entorno | Alta | A veces (depende del panel) | `PUERTO=9090` exportada por el sistema |
| Archivo | Media | Sí, siempre | `puerto = 9090` en `app.ini` |
| Valor por defecto del código | Ninguna | Leyendo el código | `puerto: int = 8080` |

La lógica detrás de la pirámide es una sola frase: **lo que el operador puso más cerca del momento de la corrida, gana**. Si alguien exportó una variable de entorno para esta ejecución en particular, esa intención es más específica que un archivo que alguien editó la semana pasada. Y el valor por defecto del código es el último recurso, no el primero: existe para que el programa arranque, no para decidir.

> **Importante:** el orden de la pirámide es una convención, no una ley de la naturaleza. `pydantic-settings`, la herramienta que vamos a ver al final, la respeta; `configparser` no, porque no sabe nada de entornos. Por eso, cuando construyas tu propia pirámide a mano — y la vas a construir en la sección 15 — el orden es una decisión tuya, y si lo escribís en el código y en un comentario al lado, nadie va a tener que adivinarla.

Fijate que las dos primeras filas de la tabla son las que explicarían el bug más caro de un despliegue: alguien pone la variable de entorno, todo parece configurado, y el programa ignora la variable. Pasa cuando el orden real es el inverso del que se supone. Y por eso el orden hay que **escribirlo y probarlo**, no recordarlo.

Con las cuatro fuentes sobre la mesa, pasemos a la primera en detalle: la más vieja, la que viene del mundo Unix, y la que va a ser tu primera capa de verdad.

---

## 3. `configparser`: el formato `.ini` que ya conocés

Cuando decís "archivo de configuración", casi siempre estás pensando en algo así. Guardalo como `app.ini`:

```ini
[servidor]
host = localhost
puerto = 8080
debug = true

[base]
host = db.local
puerto = 5432
usuario = app
pool_maximo = 20
```

Eso es un archivo **INI**: el formato más viejo de los que vas a ver en este capítulo, uno que se usó mucho en Windows y que sigue en la biblioteca estándar de Python por una razón que ya se te va a ocurrir —es casi imposible de escribir mal. Una línea, una clave, un valor, nada más.

Python lo lee con `configparser`, que es un módulo de la biblioteca estándar. No hay que instalar nada:

```python
import configparser

parser = configparser.ConfigParser()
parser.read("app.ini", encoding="utf-8")

print(parser.sections())
print(parser.get("servidor", "host"))
print(parser.getint("servidor", "puerto"))
print(parser.getboolean("servidor", "debug"))
```

Y la salida:

```
['servidor', 'base']
localhost
8080
True
```

`sections()` devuelve las dos secciones **en el orden del archivo**, y lo mismo pasa con las opciones dentro de cada una: `options("servidor")` devuelve `['host', 'puerto', 'debug']`, en el orden en que las escribiste. Puede parecer obvio y no lo es: algunos formatos de configuración reordenan lo que leen, y si te acostumbrás a que el orden se respete, ese "puede parecer" se te vuelve a aparecer como un problema en otro lado. Y anotá otra cosa de ese `read()`: no imprime nada. Devuelve la lista de archivos que leyó, que es la información que te va a servir para detectar un archivo que no existe, en la sección 7.

Lo que `configparser` te da de gratis es algo que la mayoría de los formatos nuevos no: **puede leer y escribir el mismo archivo**. Y sabe distinguir mayúsculas de minúsculas en un lugar y no en el otro, que es una combinación medio traicionera:

```python
print(parser.getint("servidor", "PUERTO"))
try:
    print(parser.get("SERVIDOR", "puerto"))
except Exception as e:
    print(type(e).__name__, ":", e)
```

```
8080
NoSectionError : No section: 'SERVIDOR'
```

Esa asimetría es real y verificada: `ConfigParser` normaliza a minúsculas el nombre de las **opciones** — por eso `get("servidor", "PUERTO")` encuentra la que escribiste como `puerto` y devuelve `8080` — pero deja intacto el nombre de las **secciones**, y por eso `get("SERVIDOR", ...)` lanza `NoSectionError`. Es una decisión que parece arbitraria y responde a la historia del formato, pero el efecto práctico es que tenés que acordarte de cómo escribiste cada parte.

> **Buenas prácticas:** usá minúsculas en los nombres de las opciones, y dejá las secciones en minúsculas también. El formato funciona con cualquier convención, pero como solo las opciones se normalizan, una convención única para las dos mitades te evita la mitad de los `NoSectionError` que vas a ver en tu carrera. Nadie te va a decir que escribir `[Base]` en vez de `[base]` está mal, pero vos te vas a acordar dentro de tres meses de haberlo hecho.

La lectura con `get` es cómoda, pero tiene un defecto que hay que ver de frente, porque es la base de casi todo lo que viene después. `get` **siempre devuelve una cadena**. Acá está el problema completo:

```python
puerto = parser.get("servidor", "puerto")
print(type(puerto).__name__, puerto)
try:
    puerto + 1
except TypeError as e:
    print(type(e).__name__, ":", e)
```

La salida es la trampa completa:

```
str 8080
TypeError : can only concatenate str (not "int") to str
```

Un `TypeError` en una línea de configuración. El archivo dice `puerto = 8080`, que *parece* un número, y Python te lo entrega como las cinco letras `8`, `0`, `8`, `0`, `0`. La diferencia entre "el archivo contiene el número 8080" y "el archivo contiene el texto 8080" es invisible a simple vista y estalla en el momento de la suma.

`configparser` lo sabe, y por eso tiene una familia de funciones que sí convierten: `getint`, `getfloat` y `getboolean`. Es la respuesta directa a ese defecto, y es la razón por la que vas a ver `getint` en todo el código de configuración que se precie. Lo que sigue es lo que estas tres funciones aceptan:

```python
parser.read_string("""
[app]
activo = yes
doble = on
numero = 42
decimal = 2.5
""")

print(parser.getboolean("app", "activo"))
print(parser.getboolean("app", "doble"))
print(parser.getint("app", "numero"))
print(parser.getfloat("app", "decimal"))
```

```
True
True
42
2.5
```

Cuatro verdades, cuatro conversiones, todas justas. Pero antes de cerrar el `getboolean` y pasar a lo que viene, hay un detalle de la familia que merece mirar de cerca, porque es de los que te van a morder en producción y es barato de entender ahora: **qué palabras cuentan como verdad y cuáles como mentira**. Y hay una trampa escondida que no vas a ver hasta que la toques, porque parece que `configparser` es listo. Sigue con la lista de la sección 5.

---

## 4. `[DEFAULT]`: la sección que se hereda sola

Hay una cosa de los archivos INI que no aparece en casi ningún tutorial y que sin embargo es la que más te va a servir el día que tengas veinte opciones repetidas en cinco secciones: una sección llamada `DEFAULT` que **se mezcla sola** con todas las demás.

Mirá cómo funciona. En un archivo así:

```ini
[DEFAULT]
charset = utf-8
timeout = 30

[a]
host = a.local

[b]
host = b.local
timeout = 5
```

La sección `[a]` no menciona ni `charset` ni `timeout`. Y sin embargo los tiene, porque `[DEFAULT]` se aplica a todas. Y la sección `[b]` redefine `timeout` con su propio valor. El resultado, leído con `configparser`, es:

```python
import configparser

parser = configparser.ConfigParser()
parser.read("ejemplo.ini", encoding="utf-8")

print(parser.sections())
print(parser.options("a"))
print(parser.get("a", "charset"))
print(parser.get("a", "timeout"), parser.get("b", "timeout"))
```

Y la salida:

```
['a', 'b']
['host', 'charset', 'timeout']
utf-8
30 5
```

`options("a")` devuelve tres claves, no una: la que el archivo tiene en esa sección más las dos heredadas. Y `get("a", "timeout")` da `30` mientras `get("b", "timeout")` da `5`, sin que nadie haya escrito un `30` dentro de `[a]`.

El orden de esas tres claves merece un segundo de atención, porque no es el del archivo. `options("a")` pone primero las que la sección tiene de verdad y después las heredadas. Preguntale a `[b]`, que pisa una de las dos:

```python
print(parser.options("b"))
```

```
['host', 'timeout', 'charset']
```

Las dos propias de `[b]` van primero, en el orden en que están escritas, y la heredada que no pisa nada va al final. El orden no es "`[DEFAULT]` primero", que es lo que uno esperaría si piensa en la mezcla como si fuera una unión de dos listas: es "lo tuyo primero, y después lo que te aportan los demás".

Y hay un detalle que hace que esto no sea una función menor: `DEFAULT` **no aparece en `sections()`**. La primera línea del ejemplo devolvió `['a', 'b']`, no `['DEFAULT', 'a', 'b']`. Para leerla tenés que pedirla por su nombre, o usar el método que existe para eso:

```python
print(dict(parser["DEFAULT"]))
print(dict(parser.defaults()))
```

Las dos líneas devuelven lo mismo:

```
{'charset': 'utf-8', 'timeout': '30'}
{'charset': 'utf-8', 'timeout': '30'}
```

> **Dato clave:** `[DEFAULT]` no es una sección como las otras, es la **plantilla** contra la que se resuelven las demás. Sirve para los valores que son iguales en toda la aplicación —el `charset`, el `timeout`, el nivel de compresión— y dejás que cada sección escriba solo lo que la cambia. El premio es que el archivo queda mucho más corto y que un valor compartido se cambia en un lugar. El costo es una asimetría que muerde: si ponés en `[DEFAULT]` una opción que solo tiene sentido en una sección, esa opción va a aparecer en todas y `options()` te va a mentir sobre qué hay realmente en cada una.

La resolución va de lo más cercano a lo más lejano, y el orden completo son tres pasos: primero el valor de la sección, si no está el de `[DEFAULT]`, y si no está el `fallback` que le pases a mano en la llamada. No hay herencia entre secciones: `[a]` no ve lo que hay en `[b]`, y `[b]` no ve lo que hay en `[a]`. Si necesitás que compartan algo, ese algo va en `[DEFAULT]`, que es exactamente para lo que existe.

---

## 5. Los tipos: `getint`, `getfloat`, `getboolean` y la trampa del `get`

Volvamos al problema que dejamos abierto en la sección 3, porque es el que va a definir cómo escribís tu código de configuración durante años. `configparser` te da `get`, y `get` te da siempre una cadena. Hay una familia de funciones que convierte, y hay que saber exactamente qué aceptan.

Empecemos por la más importante, `getboolean`, porque su lista de verdades es corta y porque hay un detalle que confunde a todo el mundo la primera vez:

```python
import configparser

parser = configparser.ConfigParser()
parser.read_string("""
[app]
a = 1
b = yes
c = true
d = on
e = 0
f = no
g = false
h = off
""")

for clave in "abcdefgh":
    print(clave, "->", parser.getboolean("app", clave))
```

Salida:

```
a -> True
b -> True
c -> True
d -> True
e -> False
f -> False
g -> False
h -> False
```

Cuatro palabras para el sí —`1`, `yes`, `true`, `on`— y cuatro para el no. Es más de lo que esperás, y esa generosidad tiene un precio. Lo que `getboolean` **no** acepta son las palabras que en español sobresalen. Fijate:

```python
parser.read_string("[app]\nflag = SI\n")
try:
    parser.getboolean("app", "flag")
except ValueError as e:
    print(type(e).__name__, ":", e)
```

```
ValueError : Not a boolean: SI
```

Un `ValueError` limpio, que dice exactamente qué está mal. Pero prestá atención a lo que significa en la práctica: si tu archivo de configuración lo escribe una persona —y lo escribe una persona, porque para eso el INI es cómodo—, esa persona va a escribir `SI` o `NO` en español, y tu programa va a fallar con un mensaje que no dice nada sobre el idioma.

Antes de pasar a los números, una aclaración sobre los espacios que va al revés de lo que uno supone. Los espacios alrededor del valor **se recortan al leer el archivo**, no cuando los usás:

```python
parser.read_string("[app]\nflag = 1 \npad =    2   \n")
print(repr(parser.get("app", "flag")), repr(parser.get("app", "pad")))
print(parser.getint("app", "pad"))
```

```
'1' '2'
2
```

`get` devuelve `'1'`, sin el espacio final, y `getint` devuelve `2` a partir de `'    2   '`. O sea que dentro de la familia no hay ninguna diferencia de cortesía: los dos resolvieron los espacios al leer.

La diferencia real entre `get` y los conversores aparece cuando el valor es un sí o un no, y es una trampa que te puede costar una tarde entera si no la conocés. Una cadena no vacía es verdadera en Python, y punto:

```python
parser = configparser.ConfigParser()
parser.read_string("[app]\nflag = 0\notro = false\n")
print(bool(parser.get("app", "flag")))
print(parser.getboolean("app", "flag"))
print(bool(parser.get("app", "otro")))
print(parser.getboolean("app", "otro"))
```

```
True
False
True
False
```

Las dos primeras líneas son el problema completo: en el archivo el valor es `0`, que significa apagado, y `bool(...)` devuelve `True` igual. La tercera y la cuarta son lo mismo con la palabra `false`. Cualquier código que escriba `if parser.get("app", "flag"):` está leyendo una cadena, y una cadena no vacía siempre es verdadera. La única forma de que ese `if` signifique algo es con `getboolean`, y por eso una opción que sea sí o no se lee **siempre** con `getboolean`, aunque el valor parezca obvio.

En cuanto a `getint` y `getfloat` no hay sorpresas, y por eso son aburridos: hacen `int()` y `float()` y listo. La diferencia con usar `int(parser.get(...))` a mano es que el mensaje de error viene con el nombre de la opción:

```python
parser.read_string("[db]\npuerto = ocho mil\n")
try:
    parser.getint("db", "puerto")
except ValueError as e:
    print(type(e).__name__, ":", e)
```

```
ValueError : invalid literal for int() with base 10: 'ocho mil'
```

Mismo mensaje que te daría `int("ocho mil")` a secas, y acá alcanza: te dice que el problema es el texto `'ocho mil'`, que es lo único que necesitabas saber. La lección es que `getint` no es más mágico que `int()`: es exactamente lo mismo, con un nombre que se entiende en el lugar donde lo llamás.

> **Buenas prácticas:** usá `getint`/`getfloat`/`getboolean` y no `get` con una conversión manual. La diferencia real no está en el resultado —son idénticos— sino en la legibilidad del código. Y prestá atención a `fallback`, que se usa mal casi siempre: cubre el caso de la opción **ausente**, y no el del valor **inválido**.

```python
parser = configparser.ConfigParser()
parser.read_string("[db]\npuerto = ocho mil\n")

print(parser.get("db", "usuario", fallback="app"))
try:
    print(parser.getint("db", "puerto", fallback=5432))
except ValueError as e:
    print("puerto:", type(e).__name__)
```

```
app
puerto: ValueError
```

La primera línea pide `usuario`, que no existe, y el `fallback` responde sin preguntar nada. La segunda pide `puerto`, que existe pero dice "ocho mil": ahí el `fallback` no se usa y el `ValueError` sigue en pie. `fallback` es una red para el hueco, no para la basura: si el valor está pero está mal escrito, el programa tiene que enterarse, porque ese `5432` de emergencia es un número que nadie eligió.

---

## 6. La interpolación: el `%` que parece un signo y es una máquina

Acá hay una capacidad de `configparser` que casi nadie usa y que sin embargo explica una cantidad de errores raros. Cuando leés un valor, `configparser` no te devuelve el texto del archivo: te devuelve el texto **después de resolver unas referencias** a otras opciones del mismo archivo. La sintaxis clásica es `%(nombre)s`:

```ini
[db]
host = localhost
puerto = 5432
url = postgresql://%(host)s:%(puerto)s/app
```

El archivo **no** tiene esa URL escrita: tiene una plantilla. Cuando hacés `parser.get("db", "url")` no te devuelve la plantilla, te devuelve `postgresql://localhost:5432/app`. La referencia se resolvió al leer, no al escribir.

Mirá la segunda forma de la interpolación, que existe en `ConfigParser` pero no hace lo que parece:

```python
import configparser

parser = configparser.ConfigParser()
parser.read_string("""
[db]
host = localhost
puerto = 5432
url = postgresql://%(host)s:%(puerto)s/app
otro = ${host}:${puerto}
""")

print(parser.get("db", "url"))
print(parser.get("db", "otro"))
```

```
postgresql://localhost:5432/app
${host}:${puerto}
```

La primera línea se resolvió y la segunda no. `${...}` **no se interpola en `ConfigParser`**: sale literal, carácter por carácter. La forma que funciona es `%(...)s`. Si venís de un lenguaje donde las variables se escriben `${...}`, este es el momento exacto en que esa costumbre te va a hacer perder veinte minutos mirando un archivo que está perfecto.

Y hay una tercera forma de escribir la referencia, la más antigua, que casi no se usa pero que sigue funcionando: `%()nombre`. Es idéntica a la que usamos antes y el resultado es el mismo. Lo que cambia es la sintaxis de las variables **alrededor** del valor: con `%(nombre)s` la referencia tiene que ocupar el valor entero o casi entero, y para los formatos de fecha existen los especificadores `%Y`, `%m` y `%d`, que `configparser` copia sin tocar.

Ahora la parte que duele, y que es la razón por la que mucha gente termina poniendo `interpolation=None` sin entender qué perdió. La interpolación es automática, y eso significa que **cualquier `%` en un valor es un intento de interpolación**. Escribí esto:

```python
parser = configparser.ConfigParser()
parser.read_string("[db]\nformato = 100%\n")
print("leído sin problema")
```

```
leído sin problema
```

Leído sin problema. El parser se construyó, el archivo se procesó, y no pasó nada. El error aparece **después**, cuando le pedís el valor:

```python
try:
    print(parser.get("db", "formato"))
except Exception as e:
    print(type(e).__name__, ":", e)
```

```
InterpolationSyntaxError : '%' must be followed by '%' or '(', found: '%'
```

Tres cosas están pasando acá y las tres importan. La primera: un porcentaje en un archivo de configuración —y un porcentaje es un valor perfectamente legítimo, un IVA del 21%, un margen del 35%, una tasa de ocupación del 80%— es un **error de sintaxis** en `configparser`. La segunda: el error no aparece al leer el archivo sino al usar el valor, o sea que un archivo con un `%` mal puesto **se lee perfecto y revienta más tarde**, en una línea que no tiene nada que ver. La tercera: el mensaje dice `found: '%'`, que no te dice ni la opción ni la línea donde está el problema.

Y hay un caso peor, porque el valor que falla parece algo que no es un porcentaje. Las URLs codificadas usan `%20` para un espacio:

```python
parser = configparser.ConfigParser()
parser.read_string("[u]\ncon_codigo = 50%20off\n")
try:
    print(parser.get("u", "con_codigo"))
except Exception as e:
    print(type(e).__name__, ":", e)
```

```
InterpolationSyntaxError : '%' must be followed by '%' or '(', found: '%20off'
```

Nadie mira `50%20off` y piensa "esto es una interpolación pendiente". Es un porcentaje de descuento escrito en una URL, y `configparser` lo trata como si quisieras decir "acá va una referencia a otra opción".

La salida existe, y hay dos. La primera es escapar el signo, duplicándolo:

```python
parser = configparser.ConfigParser()
parser.read_string("[u]\ncon_doble = 100%%\n")
print(parser.get("u", "con_doble"))
```

```
100%
```

`%%` es la forma de decir "esto es un signo de porcentaje de verdad, no una referencia". Funciona, y es la solución cuando el archivo lo escribís vos y podés exigir la convención. La segunda es desactivar la interpolación entera al construir el parser:

```python
parser = configparser.ConfigParser(interpolation=None)
parser.read_string("[db]\nformato = 100%\nhost = h\nurl = %(host)s\n")
print(parser.get("db", "formato"))
print(parser.get("db", "url"))
```

```
100%
%(host)s
```

Fijate lo que pasó con la segunda línea. La plantilla `%(host)s` se quedó **literal**: no se expandió. O sea que no es un interruptor de "usar `%` o no usar `%`", es un interruptor de "interpolás todo o no interpolás nada". No hay término medio, y esa es la decisión real que tenés que tomar.

> **Importante:** la interpolación de `configparser` y la de los formatos de datos del capítulo 13 son dos mecanismos distintos que comparten el signo, y confundirlos cuesta tiempo. En `ConfigParser`, `%(nombre)s` es una referencia a **otra opción del mismo archivo** y el resultado es un texto: no hay tipos, no hay fechas, no hay listas. En TOML y en JSON un `%` no significa absolutamente nada. Cuando lleguemos a los tres formatos juntos vas a ver por qué el signo `%` es una de esas cosas que hay que aprender a reconocer por contexto.

Y queda el último caso, que es el más común de los tres y el más confuso: **referenciar una opción que no existe**. El error tiene su propia clase, y el nombre es largo pero exacto:

```python
parser = configparser.ConfigParser()
parser.read_string("[db]\nhost = h\nurl = %(servidor)s://x\n")
try:
    parser.get("db", "url")
except Exception as e:
    print(type(e).__name__, ":", e)
```

```
InterpolationMissingOptionError : Bad value substitution: option 'url' in section 'db' contains an interpolation key 'servidor' which is not a valid option name. Raw value: '%(servidor)s://x'
```

El mensaje es largo, y vale la pena leerlo entero porque es el mejor ejemplo de un error que **te dice exactamente qué está mal**: qué opción se estaba expandiendo (`url`), en qué sección (`db`), qué nombre no encontró (`servidor`) y el valor crudo que no pudo resolver. Es la excepción más útil de todas las de `configparser`, y existe por una razón concreta: el error de interpolación es el único en que el problema **no está en la línea que falló** sino en otra línea del archivo. Si el mensaje no te dijera cuál, estarías buscando en el lugar equivocado.

---

## 7. Los errores: cinco excepciones y un silencio que muerde

`configparser` es generoso con los errores de estructura, y esa generosidad se paga con un precio escondido. Vamos a ver cinco excepciones, que son las que te van a aparecer, y después el caso que no es una excepción: el que no lanza nada.

La sección que no existe y la opción que no existe, que son los dos errores que vas a escribir un día sin darte cuenta:

```python
import configparser

parser = configparser.ConfigParser()
parser.read_string("[db]\nhost = h\n")

try:
    parser.get("nada", "host")
except Exception as e:
    print(type(e).__name__, ":", e)

try:
    parser.get("db", "nada")
except Exception as e:
    print(type(e).__name__, ":", e)
```

```
NoSectionError : No section: 'nada'
NoOptionError : No option 'nada' in section: 'db'
```

Los mensajes son cortos y exactos, y hay algo que vale la pena sacar de ellos: la diferencia entre `parser.get("nada", "host")` y `parser.get("db", "nada")` no cambia nada en el resultado —los dos fallan—, pero en el código real solo uno de los dos es un error de verdad. Pedir una sección que no existe casi siempre es un nombre mal escrito; pedir una opción que no existe puede ser lo mismo, o puede ser que la opción esté en otra sección y vos estés mirando la de al lado. Por eso el mensaje dice en qué sección buscó, y por eso conviene no envolver estas dos llamadas en el mismo `try` mudo.

Después, el error de escritura más común que existe: un archivo sin cabecera de sección, o sea una línea de configuración escrita antes del primer `[nombre]`.

```python
try:
    configparser.ConfigParser().read_string("host = h\npuerto = 1\n")
except Exception as e:
    print(type(e).__name__, ":", e)
```

```
MissingSectionHeaderError : File contains no section headers.
file: '<string>', line: 1
'host = h\n'
```

Este mensaje trae tres cosas: qué pasó, en qué archivo y en qué línea. Y prestá atención a la segunda: dice `'<string>'` porque el contenido no vino de un archivo, vino de un `read_string`. Si el error aparece con `'<string>'` es que estás probando con texto en memoria; si aparece con una ruta, es un archivo de verdad. En los dos casos el número de línea está, y esa es la diferencia entre una hora de programación y dos minutos.

Y los dos de duplicados, que aparecen cuando alguien copia y pega y no actualiza la clave:

```python
try:
    configparser.ConfigParser().read_string("[db]\nhost = a\nhost = b\n")
except Exception as e:
    print(type(e).__name__, ":", e)

try:
    configparser.ConfigParser().read_string("[db]\nhost = a\n\n[db]\npuerto = 1\n")
except Exception as e:
    print(type(e).__name__, ":", e)
```

```
DuplicateOptionError : While reading from '<string>' [line  3]: option 'host' in section 'db' already exists
DuplicateSectionError : While reading from '<string>' [line  4]: section 'db' already exists
```

Los dos mensajes traen el **número de línea**, y el formato es el mismo en los dos casos: `[line  N]`, con el número bien alineado a la derecha. Esos dos mensajes son de los más útiles de la biblioteca, porque dicen dónde está el problema y no solo qué problema es.

Los duplicados son un error por omisión y se pueden apagar: `ConfigParser(strict=False)` acepta el archivo y gana la última definición.

```python
parser = configparser.ConfigParser(strict=False)
parser.read_string("[db]\nhost = a\nhost = b\n")
print(parser.get("db", "host"))
```

```
b
```

No lo hagas. Un archivo de configuración con una clave repetida no es un archivo con dos valores: es un archivo con **un valor y una ambigüedad**, y la versión que gana depende del orden de lectura. Si aceptás el archivo y después leés el valor, perdiste la información de que había un problema. El `strict=True` por omisión existe por esta razón, y desactivarlo es desactivar el único detector de errores de escritura que tenías.

> **Buenas prácticas:** agarrá los errores de `configparser` en un solo lugar, en la función que carga la configuración, y convertilos en un mensaje que nombre el archivo. La forma que se usa siempre es la del capítulo 17: una excepción propia que hereda de `Exception`, un `try` alrededor de la lectura, y un `raise ... from e` para no perder la causa. Un `NoOptionError` que sube hasta la cima de la pila de llamadas no dice en qué archivo estaba mirando el programa, y ese dato es la mitad de lo que necesitás para arreglarlo.

Y ahora el que no lanza nada, que es el más peligroso de los seis porque no hay excepción donde no hay excepción. `read()` **devuelve la lista de archivos que leyó, y no se queja de los que no encontró**:

```python
parser = configparser.ConfigParser()
print(parser.read("no-existe-jamas.ini"))
print(parser.sections())
```

```
[]
[]
```

Un archivo que no existe no es un error: es una lista vacía. Es un comportamiento razonable —muchas veces querés leer un archivo opcional, y no hay forma de que `read()` sepa si vos lo querías opcional o no— pero combinado con lo de arriba es un problema. Si tu programa tiene un archivo de configuración y alguien lo renombra, o el directorio de trabajo cambia, o el despliegue lo monta en otro lado, el programa **arranca igual**, con la configuración vacía, y falla más tarde, en otra parte, por un motivo que no parece relacionado. Y si encima le pedís un valor con `getint`, el error que ves es un `NoOptionError` que no menciona el archivo.

Por eso `read()` devuelve la lista: para que la compares. Es el contrato que te salva:

```python
import os

RUTA = "app.ini"
parser = configparser.ConfigParser()
leidos = parser.read(RUTA, encoding="utf-8")

if not leidos:
    raise FileNotFoundError(f"falta el archivo de configuración: {os.path.abspath(RUTA)}")
```

El `os.path.abspath` está ahí por una razón concreta: `FileNotFoundError` va a decirte que no encontró `app.ini`, que es un nombre relativo, y si el error se muestra tres directorios más arriba, ese nombre no significa nada para quien lo lee. Decir el camino absoluto convierte un error inaccionable en uno que se puede arreglar con copiar y pegar. Es el mismo consejo del capítulo 27 sobre dar rutas absolutas al usuario, aplicado a un caso donde no lo habías visto.

Y hay un comportamiento intermedio que también conviene tener en la cabeza, porque es lo que hace un despliegue con varios archivos. `read()` acepta una **lista**, y cuando hay varios, el último que define una clave gana:

```python
parser = configparser.ConfigParser()
parser.read(["base.ini", "produccion.ini"])
print(parser.get("db", "host"), parser.get("db", "usuario"))
```

Con `base.ini` diciendo `host = del_base` y `usuario = comun`, y `produccion.ini` diciendo solo `host = del_produccion`, la salida es:

```
del_produccion comun
```

No hay mezcla ni herencia: el archivo de producción pisa lo que dice, y lo que no dice queda del otro. Es un mecanismo útil —un archivo base más una diferencia por entorno— y también una forma fácil de olvidar un valor que creías que estaba y ya no está. Si vas a usar varios archivos, escribí los nombres en el orden en que se leen, porque el orden es parte de la configuración.

---

## 8. `os.environ`: la capa que no es de Python

Hasta ahora todo fue un archivo: algo que alguien escribió en un disco, con secciones y con sintaxis. La segunda fuente de la pirámide es de otra naturaleza, y conviene entender por qué antes de ver cómo se usa.

Una **variable de entorno** es un par de nombre y valor que vive **dentro del sistema operativo**, no dentro del programa. El programa no la contiene: la pide. Cuando tu programa arranca, el sistema operativo le entrega una copia de su tabla de variables, y a partir de ahí el programa puede leerla y modificarla. Nadie la guardó en tu proyecto. Nadie la versionó. Nadie la puede ver mirando el repositorio.

Y esa es exactamente la razón por la que existe. Porque hay una clase de información que **no puede estar en un archivo del proyecto**, por más que quieras: la contraseña de la base de datos de producción. Si está en el archivo, está en el control de versiones, está en el historial para siempre, y está en la máquina de todos los que clonaron. Ese es el problema que la variable de entorno vino a resolver, y por eso en cualquier despliegue real es la capa que gana: lo que hay en el entorno es lo que esa máquina necesita.

La API son cuatro llamadas, y dos de ellas se parecen tanto que conviene ver la diferencia:

```python
import os

os.environ["PUERTO"] = "8080"
print(os.environ["PUERTO"])
print(os.getenv("PUERTO"))
print(os.getenv("NO_EXISTE"))
print(os.environ.get("NO_EXISTE", "5555"))
```

```
8080
8080
None
5555
```

La diferencia entre la tercera y la cuarta línea es la que importa: `os.environ["NO_EXISTE"]` **lanza `KeyError`**, como cualquier diccionario, y `os.getenv("NO_EXISTE")` devuelve `None`. Son dos puertas distintas al mismo lugar, y si elegís mal la puerta el programa se cae donde no debía.

```python
try:
    os.environ["NO_EXISTE"]
except KeyError as e:
    print(type(e).__name__, ":", e)
```

```
KeyError : 'NO_EXISTE'
```

Y la razón de fondo de esa diferencia es que `os.environ` **es un diccionario**, con todas las operaciones que eso implica. No es una función rara: es un `dict` con dos capacidades extra, que sus valores son cadenas y que escribir en él cambia el entorno del proceso:

```python
print(type(os.environ).__name__)
os.environ.update({"A": "1", "B": "2"})
print(os.environ.get("A"), os.environ.get("B"))
print(os.environ.pop("B"), os.environ.get("B"))
```

```
_Environ
1 2
2 None
```

El `1` y el `2` son las dos cadenas, no los dos números, y esa es la regla de oro de esta capa: **todo lo que viene del entorno es texto, siempre**. No hay enteros ni booleanos en el entorno, no los hubo nunca, y `os.environ.update({"puerto": 8080})` te da un `TypeError` en la línea siguiente.

Lo de `pop`, `setdefault` y compañía funciona como en cualquier diccionario, pero conviene que sepas una cosa antes de usarlo: **cambiar `os.environ` dentro de tu programa cambia el entorno de tu programa**, no el del sistema operativo ni el de los otros programas que están corriendo. Es una diferencia que importa cuando el código de configuración se usa en las pruebas: poner una variable para un caso y borrarla después es la forma correcta de aislar, y por eso esa API existe.

Ahora el comportamiento que más sorprende de esta capa, que es específico de Windows pero que vas a topar de todas formas si corrés el mismo código en dos sistemas operativos. En Windows, el **nombre de la variable no distingue mayúsculas de minúsculas**:

```python
os.environ["ruta_prueba"] = "x"
print(os.environ.get("RUTA_PRUEBA"))

os.environ["RUTA_PRUEBA2"] = "y"
print(os.environ.get("ruta_prueba2"))
print([k for k in os.environ if k.lower() == "ruta_prueba"])
```

```
x
y
['RUTA_PRUEBA']
```

Escribiste `ruta_prueba` en minúsculas y leíste `RUTA_PRUEBA` en mayúsculas, y funcionó. Al revés también. Windows normaliza los nombres a mayúsculas, y la tercera línea lo confirma: la clave guardada se llama `RUTA_PRUEBA`, aunque vos escribiste otra cosa. En Linux no pasa nada de esto, y ahí el caso importa. Es la clase de diferencia que no aparece en tu máquina y aparece en la de tu compañero, o en producción.

> **Dato clave:** el entorno es un diccionario con dos reglas que lo separan de todos los demás: **los valores son cadenas, siempre**, y **el nombre no es confiable como identificador entre sistemas operativos**. Por eso ningún programa serio lee `os.environ` en medio de la lógica. Lee las variables en un solo lugar, al arranque, conviértelas a los tipos que corresponden y guardalas en un objeto con atributos. Es exactamente lo que hace `pydantic-settings` en la sección 13, y es la razón por la que esa herramienta existe: no es una comodidad, es **la frontera tipada** entre un sistema de textos y tu programa.

Y hay un detalle más que hace que esto no sea solo una API sino una **frontera con el sistema operativo**, y que conviene que entiendas de dónde viene aunque no lo uses nunca. Las variables de entorno son cadenas porque así las implementó el sistema operativo que las inventó, y hay un caso en que no las decodifica. En Python existe `os.environb`, que es lo mismo pero en `bytes` en vez de `str`, y existe **solo donde el sistema operativo las guarda en bytes**:

```python
print(hasattr(os, "environb"), os.supports_bytes_environ)
```

```
False False
```

En Windows da `False False`, y `os.environb` no existe. En Linux y macOS da `True True`, y ahí es la puerta trasera para los procesos que se comunican con un programa compilado que espera bytes crudos. Es una de esas funciones que no vas a necesitar nunca y que sin embargo conviene conocer, porque el día que la encuentres en el código de otra persona no vas a perder una hora buscándola.

---

## 9. `.env`: las variables de entorno en un archivo

La sección anterior dejó a `os.environ` con una desventaja que no tiene solución dentro de Python: leer el valor es fácil, pero **cambiarlo es incómodo**. En tu máquina, cambiar una variable es abrir una terminal, escribir un `set` o un `export`, y acordarte de hacerlo en cada terminal nueva. En un servidor, es abrir un panel web, buscar la casilla correcta y escribir a mano el mismo texto que ya tenés en otro lado. Y el valor que acabás de escribir no está en ningún archivo, así que no se puede revisar, no se puede copiar, no se puede versionar, y la próxima persona que llegue al proyecto no tiene forma de saber que existe.

La solución que usa casi todo el mundo es un archivo de texto con una variable por línea. Guardalo como `demo.env`:

```dotenv
# comentario
PUERTO=8080
DEBUG=true
SALUDO="hola mundo"
export EXPORTADO=si  # comentario al final
VACIO=
CON_ESPACIOS=  con espacios
RUTA_RELATIVA=./datos
CORTE=antes # despues
COMILLA="con # hash"
```

Se lee con `python-dotenv`, que **no** viene con Python: hay que instalarlo. Y conviene decir de entrada que este es el primer paquete externo del capítulo, porque hasta ahora todo fue biblioteca estándar. La diferencia no es cosmética: los archivos `.ini` y `.env` los podés leer con lo que ya tenés, y este no.

```python
from dotenv import dotenv_values

valores = dotenv_values("demo.env")
for clave, valor in valores.items():
    print(clave, "->", repr(valor))
```

```
PUERTO -> '8080'
DEBUG -> 'true'
SALUDO -> 'hola mundo'
EXPORTADO -> 'si'
VACIO -> ''
CON_ESPACIOS -> 'con espacios'
RUTA_RELATIVA -> './datos'
CORTE -> 'antes'
COMILLA -> 'con # hash'
```

`dotenv_values` **no toca `os.environ`**: te devuelve un diccionario con lo que leyó del archivo. Es la mitad prudente de la biblioteca, y es la que conviene usar mientras estás leyendo un archivo para entender qué hay adentro.

De esos nueve valores hay cinco cosas para mirar. La primera: **todos son cadenas**, y por eso `DEBUG` es el texto `'true'` y no el booleano. Segunda: el `export` del principio se acepta y se descarta, porque muchos de los archivos que vas a encontrar exportan todo. Tercera: las comillas no son parte del valor, se arrancan. Cuarta: `VACIO=` es la cadena vacía, no `None` —una diferencia que importa cuando tu programa usa `None` para decir "no está". Y quinta: los espacios alrededor del valor se recortan, pero los de adentro se conservan.

La otra mitad de la biblioteca es la que escribe en el entorno, y es la que se usa en el arranque de cualquier programa:

```python
import os
from dotenv import load_dotenv

os.environ["PUERTO"] = "9999"
print("antes:", os.getenv("PUERTO"))

print("load_dotenv devuelve:", load_dotenv("demo.env"))
print("despues:", os.getenv("PUERTO"))

load_dotenv("demo.env", override=True)
print("con override:", os.getenv("PUERTO"))
```

```
antes: 9999
load_dotenv devuelve: True
despues: 9999
con override: 8080
```

Las tres líneas del medio son el comportamiento más importante de toda la biblioteca, y es una decisión de diseño, no un descuido: **`load_dotenv` no pisa lo que ya está en el entorno**. Lo que exportaste en tu terminal gana sobre lo que dice el archivo. Y hay un segundo detalle en ese `True`: `load_dotenv` devuelve si encontró el archivo o no, así que un archivo mal escrito en la ruta se detecta con un `if` en vez de con una excepción.

Tres cosas más de la sintaxis, porque las tres muerden. La primera son los comentarios: un `#` corta el valor **aunque el valor no esté entre comillas**.

```python
print(repr(os.environ.get("EXPORTADO")), repr(os.environ.get("COMILLA")))
```

```
'si' 'con # hash'
```

Con `CORTE=antes # despues` en el archivo, el valor es `'antes'`. El `#` empieza un comentario en cualquier parte de la línea, y todo lo que sigue se descarta. Con comillas, en cambio, el `#` se conserva: `COMILLA="con # hash"` vale `'con # hash'`. Si tu valor necesita un `#`, tiene que ir entre comillas.

La segunda es la interpolación, y tiene una sintaxis y una casi-sintaxis. Guardá este como `cadena.env`:

```dotenv
BASE=http://localhost
RAIZ=${BASE}/api
OTRA=$BASE/v2
FALTA=${NO_EXISTE}/x
```

```python
from dotenv import dotenv_values

for clave, valor in dotenv_values("cadena.env").items():
    print(clave, "->", repr(valor))
```

La salida es esta:

```
BASE -> 'http://localhost'
RAIZ -> 'http://localhost/api'
OTRA -> '$BASE/v2'
FALTA -> '/x'
```

`${BASE}` se resuelve contra las variables que ya estaban definidas —en el archivo, de arriba hacia abajo, o en el entorno—, y `$BASE` **no se resuelve**: sale literal, con el signo incluido. Y una referencia a una variable que no existe no es un error: se reemplaza por la cadena vacía, que es por qué `FALTA` vale `'/x'` y no `'${NO_EXISTE}/x'`. Un `${...}` mal escrito es, silenciosamente, una variable vacía.

> **Importante:** si el archivo `.env` tiene una contraseña adentro, ese archivo no va al repositorio. Va en `.gitignore` desde el primer día, y al repositorio va un `.env.example` con los mismos nombres y valores inventados. El error clásico es commitear el archivo una vez, borrarlo del código y creer que quedó limpio: el secreto sigue en el historial, para siempre, en todas las copias que alguien haya hecho. Si llega a pasar eso, la contraseña se cambia; no hay nada que limpia el historial.

Y hay un comportamiento de `load_dotenv` que parece un detalle y es una fuente de bugs de producción: por omisión, la función **no recibe la ruta**. Si la llamás sin argumentos, busca el archivo `.env` subiendo por los directorios **desde el archivo que la está llamando**, no desde el directorio donde estás parado.

Las dos llamadas se ven iguales y no son lo mismo. `find_dotenv()` —sin argumentos— sube por los directorios **desde el archivo que la está llamando**; `find_dotenv(usecwd=True)` sube desde el **directorio actual**. La diferencia aparece justo cuando el programa corre desde otro lado: un programa guardado en `proyecto/app/main.py` encuentra el `.env` de `proyecto/` aunque lo ejecutes desde `C:\Windows`, porque sube desde `app/`, mientras que una prueba que corre desde otro directorio puede no encontrarlo. Por eso, en un programa real la ruta se pasa explícita —`load_dotenv(RAIZ / ".env")`, con `RAIZ` calculado a partir de `__file__` como en el capítulo 27— y la búsqueda por omisión se desactiva.

Y si querés que la búsqueda falle en voz alta en vez de en silencio, existe un parámetro para eso:

```python
from dotenv import find_dotenv

try:
    find_dotenv("no-existe.env", raise_error_if_not_found=True, usecwd=True)
except Exception as e:
    print(type(e).__name__, ":", e)
```

```
OSError : File not found
```

Sin ese parámetro, `find_dotenv` devuelve la cadena vacía `''` cuando no encuentra nada, que es un valor que parece un resultado y es una ausencia.

---

## 10. TOML: un formato con tipos de verdad

El formato INI que viene usando tiene un problema que ya conocés: no tiene tipos. El formato que se puso de moda para resolver exactamente eso se llama TOML, y su idea es simple —**parece un INI, pero los valores son de verdad**—. Un archivo tuyo, `app.toml`:

```toml
titulo = "Mi app"
puerto = 8080
debug = true
proporcion = 0.75
inicio = 2024-03-15T10:30:00
dia = 2024-03-15
clientes = [
  "uno",
  "dos",
]

[base]
host = "localhost"
puerto = 5432

[base.pool]
minimo = 1
maximo = 10
```

La estructura es la de un INI —llave igual valor, y secciones entre corchetes— y `[base.pool]` se lee como un diccionario dentro de `[base]`, o sea que el anidamiento es explícito y no una convención de nombres. Lo que cambia está en los valores. Se lee con `tomli`, que es el lector de referencia:

```python
import tomli

with open("app.toml", "rb") as f:
    datos = tomli.load(f)

for clave, valor in datos.items():
    print(clave, "->", repr(valor), type(valor).__name__)
```

Y cada línea sale con su tipo, que es lo que estabas buscando desde la sección 3:

```
titulo -> 'Mi app' str
puerto -> 8080 int
debug -> True bool
proporcion -> 0.75 float
inicio -> datetime.datetime(2024, 3, 15, 10, 30) datetime
dia -> datetime.date(2024, 3, 15) date
clientes -> ['uno', 'dos'] list
base -> {'host': 'localhost', 'puerto': 5432, 'pool': {'minimo': 1, 'maximo': 10}} dict
```

Fijate las dos fechas. `inicio` no es un texto que dice "2024-03-15T10:30:00": es un `datetime`, y `dia` es un `date`. TOML convierte las fechas de verdad, y por eso tenés que importar el módulo `datetime` para compararlas o formatearlas. Ningún otro formato de los tres que vamos a ver hace eso, y es la diferencia entre una configuración donde las fechas se pueden usar y una donde hay que convertirlas a mano cada vez.

> **Dato clave:** dos reglas de TOML que te van a costar un día si no las tenés presentes: **todo texto va entre comillas** —en un INI no hacía falta— y **no existe `null`**. Una clave sin valor se **omite**, o se escribe `activo = false`, que además de explícito se puede comprobar.

Y hay un detalle de sintaxis que te va a dar un error el primer día, porque la familiaridad con el INI te va a hacer escribir lo que no es: **en TOML los textos van entre comillas, siempre**. No hay modo "palabra suelta" como en un INI o en un archivo de texto plano.

```python
for linea in ["host = localhost", 'host = "localhost"']:
    try:
        print(linea, "->", repr(tomli.loads(linea)["host"]))
    except Exception as e:
        print(linea, "->", type(e).__name__)
```

```
host = localhost -> TOMLDecodeError
host = "localhost" -> 'localhost'
```

Dos cosas de esa salida. La primera, el error: es un `TOMLDecodeError`, o sea que el archivo entero es inválido, no solo la línea. La segunda, algo más importante: **la sintaxis se parece a la del INI y no se parece en el punto donde más te va a confundir**. En un INI escribís `host = localhost` y funciona; en TOML eso es un error, y el mensaje ni siquiera menciona la clave, sino la columna donde estaba el valor.

Dos cosas más antes de comparar los formatos. TOML no tiene `null`, y no es una omisión: no existe el concepto.

```python
try:
    tomli.loads("a = null\n")
except Exception as e:
    print(type(e).__name__, ":", e)
```

```
TOMLDecodeError : Invalid value (at line 1, column 5)
```

La columna 5 es donde empieza la palabra `null`, que es donde TOML esperaba un valor de verdad y recibió algo que no existe en su vocabulario. Para dejar una variable sin valor no se escribe `null`: se **omite la clave**, o se escribe `activo = false`, que es un valor explícito y es preferible.

Y las claves se pueden poner entre comillas cuando tienen espacios, y las listas de tablas usan `[[nombre]]` en vez de `[nombre]`:

```python
datos = tomli.loads('"clave con guion" = 1\n[[servidor]]\nnombre = "uno"\n[[servidor]]\nnombre = "dos"\n')
print(datos)
print("servidor es una", type(datos["servidor"]).__name__, "de", len(datos["servidor"]), "elementos")
```

```
{'clave con guion': 1, 'servidor': [{'nombre': 'uno'}, {'nombre': 'dos'}]}
servidor es una list de 2 elementos
```

`[[servidor]]` es la forma de tener **varios bloques con el mismo nombre**, que es lo que un INI no puede expresar de ninguna manera. Es la manera de escribir una lista de servidores, de archivos o de cualquier cosa que se repita, y el resultado es una lista de diccionarios, no un diccionario.

---

## 11. JSON, YAML y TOML: tres formatos, tres_operandos

Ya conocés los tres formatos del capítulo 28, así que acá no vamos a repetirlos: vamos a ponerlos **uno al lado del otro** con la misma línea de texto y a ver qué le hace cada uno. Porque hay un detalle de los tres que cambia el significado de un archivo sin cambiar una sola letra, y ese detalle es la razón por la que "es un archivo de configuración" no alcanza como descripción de nada.

La línea que vamos a pasar por los tres parsers es siempre la misma: `a = <valor>` o su equivalente.

```python
import json

import tomli
import yaml

def leer(formato, texto):
    try:
        if formato == "json":
            return repr(json.loads(texto)["a"])
        if formato == "yaml":
            return repr(yaml.safe_load(texto)["a"])
        return repr(tomli.loads(texto)["a"])
    except Exception as e:
        return f"!{type(e).__name__}"

casos = [
    ("0123", '{"a": 0123}', "a: 0123\n", "a = 0123\n"),
    ('"0123"', '{"a": "0123"}', 'a: "0123"\n', 'a = "0123"\n'),
    ("1.10", '{"a": 1.10}', "a: 1.10\n", "a = 1.10\n"),
    ("yes", '{"a": yes}', "a: yes\n", "a = yes\n"),
    ("on", '{"a": on}', "a: on\n", "a = on\n"),
    ("~", '{"a": null}', "a: ~\n", "a = \n"),
    ("2024-03-15", '{"a": "2024-03-15"}', "a: 2024-03-15\n", "a = 2024-03-15\n"),
    ("0x1F", '{"a": "0x1F"}', "a: 0x1F\n", "a = 0x1F\n"),
    ("1_000", '{"a": "1_000"}', "a: 1_000\n", "a = 1_000\n"),
]

print(f"{'lo que escribis':<14} {'JSON':<22} {'YAML':<22} TOML")
for literal, j, y, t in casos:
    print(f"{literal:<14} {leer('json', j):<22} {leer('yaml', y):<22} {leer('toml', t)}")
```

La salida es la tabla completa del asunto:

```
lo que escribis JSON                   YAML                   TOML
0123           !JSONDecodeError       83                     !TOMLDecodeError
"0123"         '0123'                 '0123'                 '0123'
1.10           1.1                    1.1                    1.1
yes            !JSONDecodeError       True                   !TOMLDecodeError
on             !JSONDecodeError       True                   !TOMLDecodeError
~              None                   None                   !TOMLDecodeError
2024-03-15     '2024-03-15'           datetime.date(2024, 3, 15) datetime.date(2024, 3, 15)
0x1F           '0x1F'                 31                     31
1_000          '1_000'                1000                   1000
```

Leela de a una, porque cada fila enseña algo distinto.

La primera fila es la más peligrosa de toda la tabla, y no por el error: por el **83**. `0123` en YAML no es un error ni un texto: es un número, y vale 83, porque el cero adelante le dice "esto es octal". Si en tu YAML alguien escribe `puerto: 08080` esperando el puerto 8080, tu programa recibe el número 3232 y no se entera de nada. En JSON y en TOML esa misma línea es un error de sintaxis, o sea que el problema es visible de inmediato; en YAML el programa arranca, se conecta al puerto equivocado y falla en el otro lado de la red. La segunda fila es la que arregla eso: entre comillas, los tres te devuelven el texto `'0123'`.

Las filas `yes` y `on` son el otro clásico. En YAML esas dos palabras son **el booleo verdadero**, y no hay comillas que las protejan. Escribís `debug: no` creyendo que apagaste algo, y tu variable de entorno termina valiendo `True`.

La fila `~` muestra el `null` de JSON y el `null` de YAML —el `~` es la abreviatura— frente a un TOML que no tiene el concepto. Y las dos últimas filas muestran el otro extremo: YAML y TOML aceptan `0x1F` y `1_000` como notaciones de número, y los dos te devuelven `31` y `1000`. Los tres formatos tienen palabras reservadas para "esto es un número", y no son las mismas.

Y hay una fila que no parece interesante y es la más incómoda de la tabla: `1.10`. Los tres te devuelven `1.1`. Nadie lo va a notar en un puerto, pero en un valor donde el decimal importa —un porcentaje, un factor, una proporción— ese uno se perdió antes de que tu programa lo viera, y no hay forma de recuperarlo.

> **Dato clave:** la misma línea de texto produce tres cosas distintas según el formato, y en dos de los tres casos el valor cambia de tipo sin que nadie lo haya pedido. Por eso un archivo de configuración no se identifica por su contenido sino por **su extensión y su analizador**: el mismo `a = 0123` es un error en TOML y un 83 en YAML. Y por eso el formato no se elige por cómo se ve el archivo, sino por qué comportamiento querés cuando alguien escribe mal un valor: si querés que el programa se detenga, elegí el que da error.

Y con eso podemos contestar la promesa del principio del capítulo: sí, se distinguen mirando una sola línea. El signo igual es TOML, los dos puntos son YAML, y las comillas en la clave son JSON.

---

## 12. Leer y escribir TOML: dos bibliotecas, no una

Con la sintaxis entendida, quedan dos temas puramente prácticos que te van a hacer perder tiempo si no los tenés claros: **cómo se lee** un archivo TOML en Python y **con qué se escribe**. Y la respuesta a la segunda es la que sorprende: no es la misma biblioteca.

Empecemos por la lectura, que tiene una regla que no tiene excepción. `tomli.load()` quiere el archivo abierto en **modo binario**:

```python
import tomli

with open("app.toml", "rb") as f:
    datos = tomli.load(f)

print(datos["base"]["pool"]["maximo"])
```

```
10
```

Si el archivo lo abrís en modo texto, el error es inmediato y no tiene nada que ver con TOML:

```python
try:
    with open("app.toml", "r", encoding="utf-8") as f:
        tomli.load(f)
except Exception as e:
    print(type(e).__name__, ":", e)
```

```
TypeError : File must be opened in binary mode, e.g. use `open('foo.toml', 'rb')`
```

Y si lo que tenés en la mano es un texto —porque lo copiaste, porque viene de una respuesta HTTP, porque lo armaste vos— usás `tomli.loads()`, que acepta la cadena:

```python
print(tomli.loads('a = 1\n[b]\nc = "x"\n'))
```

```
{'a': 1, 'b': {'c': 'x'}}
```

La diferencia entre `load` y `loads` es la misma que en `json`, y el motivo también: leer un archivo como texto implica adivinar la codificación, y la biblioteca que define el formato decidió no hacerlo.

Ahora la escritura, y acá está la sorpresa. `tomli` **no escribe**. No es una función que falte: la biblioteca entera está diseñada para leer.

```python
print("tomli tiene dumps:", hasattr(tomli, "dumps"))
print("tomli tiene dump:", hasattr(tomli, "dump"))
```

```
tomli tiene dumps: False
tomli tiene dump: False
```

Para escribir TOML hay que instalar `toml`, que es otro paquete, de otro autor, y que funciona muy distinto:

```python
import toml

with open("salida.toml", "w", encoding="utf-8") as f:
    toml.dump({"servidor": {"host": "localhost", "puerto": 8080}, "etiquetas": ["a", "b"]}, f)

print(repr(open("salida.toml", encoding="utf-8").read()))
```

```
'etiquetas = [ "a", "b",]\n\n[servidor]\nhost = "localhost"\npuerto = 8080\n'
```

Fijate en el archivo que salió. Las claves escalares arriba, las tablas abajo, y la lista con una coma pegada al corchete de cierre. Nada de eso está mal —TOML lo acepta y `tomli` lo relee sin quejarse—, pero un archivo escrito por una máquina con esa coma tiene una apariencia distinta de uno escrito a mano, y en un repositorio esa diferencia se nota en cada cambio. Si el archivo lo escribe una persona, se escribe a mano.

Y acá está la trampa que más tiempo cuesta, porque la firma de la función promete algo que no cumple. `toml.load()` acepta **una ruta o un archivo**, no un texto:

```python
try:
    toml.load("a = 1\n")
except Exception as e:
    print(type(e).__name__, ":", e)

print("con loads:", toml.loads("a = 1\n"))
```

```
OSError : [Errno 22] Invalid argument: 'a = 1\n'
con loads: {'a': 1}
```

El mensaje es desconcertante si no sabés qué pasó: le pasaste el contenido de un archivo y la biblioteca lo interpretó como **el nombre de un archivo** que no existe. `Invalid argument` no dice "no encontré el archivo", dice "esto que me diste no es un argumento válido". Es el mismo error que te daría un archivo con un nombre inválido, y por eso el mensaje no ayuda ni un poco.

Lo que sí acepta `toml.load`, y que `tomli` no, es una lista de archivos que se leen en orden, igual que el `read()` de `configparser` de la sección 7:

```python
with open("base.toml", "w", encoding="utf-8") as f:
    toml.dump({"titulo": "Mi app", "puerto": 8080, "debug": True, "base": {"host": "db.local"}}, f)

with open("produccion.toml", "w", encoding="utf-8") as f:
    toml.dump({"puerto": 9090, "base": {"host": "prod.local"}}, f)

mezcla = toml.load(["base.toml", "produccion.toml"])
print(list(mezcla.keys()))
print(mezcla["puerto"], mezcla["base"]["host"])
```

```
['titulo', 'puerto', 'debug', 'base']
9090 prod.local
```

Las claves de `base.toml` están todas, y los valores que repite `produccion.toml` gainearon: el puerto ahora es 9090 y el host es `prod.local`. Es el mismo mecanismo de capas que ya conocés, con otro nombre.

> **Importante:** en Python 3.11 la biblioteca estándar trae un lector de TOML llamado `tomllib`, y ahí la historia se simplifica: leés con `tomllib` y no instalás nada. Este capítulo usa 3.10, donde `tomllib` no existe y por eso aparece `tomli`, que es el mismo código con otro nombre. Si tu código tiene que correr en las dos versiones, el patrón es un `try` con `import tomllib` y un `except ImportError` con `import tomli as tomllib`, que ya se vio en el capítulo 27. Para **escribir**, en ningún caso hay nada en la biblioteca estándar: `toml` es un paquete externo en todas las versiones.

---

## 13. El contrato: `pydantic-settings` lee, convierte y valida

Hasta ahora resolviendo el problema de los tipos con herramientas sueltas: `getint` para un entero, `getboolean` para un sí o un no, `tomli` que devuelve los tipos ya convertidos. Cada una resuelve su parte. Lo que ninguna hace es **las tres cosas juntas, en un solo lugar, y con un mensaje de error que nombre el campo**. Eso es lo que hace `pydantic-settings`, y es la respuesta a la promesa del principio del capítulo: la herramienta que te salva no es un formato mejor, es un contrato.

Se instala como cualquier paquete externo:

```bash
pip install pydantic-settings
```

Y se usa declarando una clase. Esto es lo **único** que hay que escribir para tener configuración con tipos, leída del entorno:

```python
import os

from pydantic_settings import BaseSettings

os.environ["NOMBRE"] = "desde_el_entorno"
os.environ["PUERTO"] = "9000"
os.environ["DEBUG"] = "false"

class ConfigSimple(BaseSettings):
    nombre: str
    puerto: int = 8080
    debug: bool = False
    etiquetas: list[str] = []

config = ConfigSimple()
print("nombre:", config.nombre)
print("puerto:", config.puerto, "| debug:", config.debug)
print("etiquetas:", config.etiquetas, type(config.etiquetas).__name__)
```

Y la salida:

```
nombre: desde_el_entorno
puerto: 9000 | debug: False
etiquetas: [] list
```

Cuatro cosas pasaron en esas tres líneas. La variable `PUERTO=9000` llegó como el texto `'9000'` y salió como el número `9000`, porque el campo está declarado como `int`. El campo `debug` no aparece en el entorno y sale con su valor por omisión, `False`. Y `etiquetas`, que nadie definió, es una lista vacía y no `None`: ese es el detalle que hace que `for etiqueta in config.etiquetas` funcione sin que tengas que comprobar nada.

Esa conversión no es una curiosidad del `int`. Es el mismo mecanismo que acepta las cuatro formas de decir "sí" que ya viste en `getboolean`:

```python
for valor in ["1", "0", "true", "off"]:
    os.environ["DEBUG"] = valor
    print(valor, "->", ConfigSimple().debug)

del os.environ["DEBUG"]
```

```
1 -> True
0 -> False
true -> True
off -> False
```

Y hay un caso donde la conversión se pone dura y es el que más sorprende en la práctica. Un campo `list[str]` **no** acepta valores separados por comas:

```python
os.environ["ETIQUETAS"] = "a,b,c"
try:
    ConfigSimple()
except Exception as e:
    print(type(e).__name__)
    print(e)
```

```
SettingsError
error parsing value for field "etiquetas" from source "EnvSettingsSource"
```

La biblioteca **rechaza** una lista de punto y coma o de comas porque no hay forma de saber si la coma era un separador o parte del texto. La sintaxis que acepta es JSON:

```python
os.environ["ETIQUETAS"] = '["a", "b", "c"]'
print(ConfigSimple().etiquetas)

del os.environ["ETIQUETAS"]
```

```
['a', 'b', 'c']
```

Y el resultado del error es la parte mejor del asunto, porque es la primera vez que un problema de configuración te dice de dónde salió el valor. `from source "EnvSettingsSource"` no es un detalle: es la diferencia entre "tu programa está mal" y "el archivo que editaste tiene una coma de más". Cuando el valor viene del entorno, decí eso mismo en voz alta y vas a encontrar el error en la mitad del tiempo.

Ahora, el otro lado del contrato, que es donde se gana la confianza en la herramienta. Si un valor no se puede convertir, el programa **no arranca con un valor raro**: se detiene, y el mensaje dice qué campo, qué esperaba y qué recibió.

```python
os.environ["PUERTO"] = "ocho mil"
try:
    ConfigSimple()
except Exception as e:
    print(e)
```

```
1 validation error for ConfigSimple
puerto
  Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='ocho mil', input_type=str]
    For further information visit https://errors.pydantic.dev/2.9/v/int_parsing
```

Leé ese mensaje despacio, porque es un modelo de cómo debería ser un error. Dice cuántas validaciones fallaron —una, no tres—, y en qué campo. Dice qué esperaba: un entero. Dice qué recibió: el texto `'ocho mil'`. Y dice el **tipo** de la regla que se rompió, `int_parsing`, que es un identificador estable: podés buscarlo, podés documentarlo, y una versión nueva de la biblioteca no te lo va a cambiar. Compará este mensaje con el `ValueError` de `getint` de la sección 5, que te decía exactamente lo mismo pero sin decir de dónde vino el valor ni cómo se llama la regla.

Y el mismo formato de error aparece cuando falta un campo obligatorio. `nombre` no tiene valor por omisión, así que si no lo definís en ninguna capa, esto es lo que pasa:

```python
for clave in ("NOMBRE", "PUERTO"):
    os.environ.pop(clave, None)

try:
    ConfigSimple()
except Exception as e:
    print(e)
```

```
1 validation error for ConfigSimple
nombre
  Field required [type=missing, input_value={}, input_type=dict]
    For further information visit https://errors.pydantic.dev/2.9/v/missing
```

`type=missing` y `input_value={}`: no llegó nada, y la biblioteca te dice que llegó un diccionario vacío. Un campo obligatorio sin valor por omisión es la forma más barata de hacer obligatoria una variable: no tenés que escribir la validación, la escribiste declarando `nombre: str` sin `=`.

> **Dato clave:** un contrato de configuración no es un molde que se le impone a los datos: es la **frontera tipada** de tu programa. Del lado de la frontera hay ocho cadenas que nadie eligió y que llegaron de tres fuentes distintas; del lado de la frontera hay un objeto con atributos, con tipos, donde `config.puerto + 1` funciona y `config.etiquetas` es una lista. Todo el programa vive del lado derecho, y por eso nunca tiene que preguntar si lo que tiene es un número. Esa es la razón por la que la sección 8 decía que nadie lee `os.environ` en medio de la lógica, y la razón por la que esta clase existe.

Y hay dos detalles de comportamiento que conviene tener en la cabeza porque aparecen solos, sin pedir nada. El primero es que **los nombres no distinguen mayúsculas** por omisión: definís `PUERTO` en el entorno y el campo se llama `puerto` en minúsculas, y funciona. El segundo es qué pasa con una variable que existe pero está vacía:

```python
os.environ["NOMBRE"] = ""
print(repr(ConfigSimple().nombre))
```

```
''
```

El campo recibe la cadena vacía, tal cual. Muchas veces eso es lo que querés —una variable presente pero vacía es una decisión— y muchas veces no: si el valor vacío debería contar como "no puesto", existe `env_ignore_empty=True`, que ignora las variables vacías y deja que el valor por omisión o el archivo `.env` ocupen su lugar.

---

## 14. La pirámide, medida

La sección 2 dibujó la pirámide como una convención. Ahora que tenés la herramienta, la convención se puede **medir**, y medirla cambia la conversación: no es "yo creo que el entorno gana", es "lo probé y gana". Vamos a leer las cuatro capas en orden y después los detalles que hacen que esto funcione en un equipo.

Lo primero es el prefijo, que es lo que hace que la clase se pueda usar en un programa real. Sin prefijo, `PUERTO` se come cualquier variable de entorno de todo el sistema y de todos los demás programas:

```python
import os

from pydantic_settings import BaseSettings, SettingsConfigDict

class Servidor(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="SERVIDOR_")

    puerto: int = 8080

os.environ["SERVIDOR_PUERTO"] = "3000"
os.environ["PUERTO"] = "9999"
print(Servidor().puerto)

del os.environ["PUERTO"]
del os.environ["SERVIDOR_PUERTO"]
```

```
3000
```

Con el prefijo, el campo `puerto` lee `SERVIDOR_PUERTO` y **no lee `PUERTO`**: la variable que estaba suelta ni se mira. En un programa con veinte variables, ese `SERVER_` adelante no es un detalle de estilo, es lo que evita que un `PUERTO=9990` de otro proceso te cambie el puerto sin avisar.

Ahora el orden completo, medido capa por capa. La clase lee, en este orden exacto: los argumentos que le pasás al construir, las variables de entorno, el archivo `.env` y por último los valores por omisión del modelo.

```python
import os

from pydantic_settings import BaseSettings, SettingsConfigDict

class App(BaseSettings):
    model_config = SettingsConfigDict(env_file="app.env", env_prefix="APP_")

    puerto: int = 8080
    nombre: str = "sin nombre"

print("1) argumento:", App(puerto=2222).puerto)
os.environ["APP_PUERTO"] = "1111"
print("2) entorno:", App().puerto)
del os.environ["APP_PUERTO"]
print("3) solo el archivo:", App().puerto)
print("4) nada en ninguna capa:", App().nombre)
```

Y con este `app.env`:

```dotenv
APP_PUERTO=7000
```

La salida:

```
1) argumento: 2222
2) entorno: 1111
3) solo el archivo: 7000
4) nada en ninguna capa: sin nombre
```

Es la pirámide de la sección 2, escrita y medida. Y el caso cuatro es el que completa el cuadro: un campo que no aparece en ninguna capa no da error si tiene valor por omisión, y da error si no lo tiene. La decisión de qué campos son obligatorios y cuáles tienen valor por omisión **es** la política de configuración del programa, y está escrita en cinco líneas de clase en vez de repartida por el código.

> **Importante:** el orden se puede cambiar y hay una razón de peso para no hacerlo. Con `extra="forbid"`, que es el valor por omisión, `BaseSettings` **rechaza** los campos que no están declarados en la clase. Y hay una asimetría que conviene conocer: en la versión que usamos acá, el rechazo muerde con las claves extra del archivo `.env`, no con las variables de entorno del sistema.

```python
import os

from pydantic_settings import BaseSettings, SettingsConfigDict

class ConBasura(BaseSettings):
    model_config = SettingsConfigDict(env_file="basura.env", env_prefix="APP_")

    puerto: int = 8080

class ConExtra(BaseSettings):
    model_config = SettingsConfigDict(env_file="app.env", env_prefix="APP_")

    puerto: int = 8080

os.environ["APP_TIRRE"] = "1"
try:
    ConBasura()
except Exception as e:
    print("archivo:", type(e).__name__, ":", str(e).splitlines()[1].strip())

os.environ["TIRRE"] = "1"
try:
    print("entorno:", ConExtra().puerto)
except Exception as e:
    print("entorno:", type(e).__name__, ":", str(e).splitlines()[1].strip())

del os.environ["APP_TIRRE"]
del os.environ["TIRRE"]
```

Este es `basura.env`, que tiene una línea de más:

```dotenv
APP_PUERTO=7000
APP_TIRRE=1
```

Y la salida dice de qué capa vino el problema:

```
archivo: ValidationError : app_tirre
entorno: 7000
```

La primera línea es la que te sirve: un `extra` en el archivo es casi siempre un nombre mal escrito —alguien quiso escribir `APP_TITRE` y escribió `APP_TIRRE`—, y enterarte al arrancar es exactamente lo que querés. La segunda es la que hay que tener en la cabeza: las variables de entorno que sobran **no producen ningún error**, nunca, y se leen solas. Un programa que corre en un contenedor donde sobran variables no se entera, y no hay forma de enterarse desde tu código. Es un comportamiento deliberado, no un error.

El anidamiento es la parte que más cuesta de configurar, porque el nombre de la variable hay que armarlo a mano y hay un orden que no es intuitivo. Con `env_nested_delimiter="__"` y un prefijo, una estructura se aplana así:

```python
import os

from pydantic_settings import BaseSettings, SettingsConfigDict

class Base(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="DB_", env_nested_delimiter="__")

    host: str = "localhost"
    puerto: int = 5432
    pool: dict[str, int] = {"minimo": 1, "maximo": 10}

print("sin nada:", Base().model_dump())

os.environ["DB__POOL__MAXIMO"] = "20"
print("con DB__POOL__MAXIMO:", Base().model_dump()["pool"])
del os.environ["DB__POOL__MAXIMO"]

os.environ["DB_POOL__MAXIMO"] = "30"
print("con DB_POOL__MAXIMO: ", Base().model_dump()["pool"])
del os.environ["DB_POOL__MAXIMO"]
```

```
sin nada: {'host': 'localhost', 'puerto': 5432, 'pool': {'minimo': 1, 'maximo': 10}}
con DB__POOL__MAXIMO: {'minimo': 1, 'maximo': 10}
con DB_POOL__MAXIMO:  {'maximo': 30}
```

La segunda línea es la que sorprende: `DB__POOL__MAXIMO` **no se lee**. El prefijo se pega al principio una sola vez, y a partir de ahí cada `__` es un nivel de anidamiento. El nombre correcto es `DB_POOL__MAXIMO`: un guion bajo del prefijo, y después el separador. Es un nombre que no se deduce, se mira.

La tercera línea es la que duele, y no tiene nada que ver con los nombres. `maximo` llegó bien, sí, pero `minimo` **desapareció**: el valor por omisión de `pool` era un diccionario con dos claves, y lo que puso el entorno es un diccionario con una. Un campo declarado como `dict` no se mezcla con su valor por omisión: **se reemplaza entero**. Tu programa sigue arrancando, y la primera vez que te vas a enterar es cuando `config.pool["minimo"]` lanza un `KeyError` en producción, con un campo que en tu máquina valía `1`.

La solución es declarar el campo anidado como un modelo en vez de como un `dict`, y el resultado es mejor en las dos direcciones:

```python
class Pool(BaseSettings):
    minimo: int = 1
    maximo: int = 10

class Base2(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="DB_", env_nested_delimiter="__")

    pool: Pool = Pool()

os.environ["DB_POOL__MAXIMO"] = "30"
print("con el modelo:", Base2().model_dump()["pool"])

os.environ["DB_POOL__MAXIMO"] = "muchos"
try:
    Base2()
except Exception as e:
    print(str(e).splitlines()[2].strip())

del os.environ["DB_POOL__MAXIMO"]
```

```
con el modelo: {'minimo': 1, 'maximo': 30}
Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='muchos', input_type=str]
```

`minimo` sigue en `1`, porque ahora cada valor interno tiene su propio valor por omisión. Y `muchos` no produce un error de tipo: produce un error de **validación**, que además dice el camino completo —`pool.maximo`— y puede decir de qué capa salió el valor. Un modelo anidado no cuesta más líneas y arregla las dos cosas.

Y un último detalle, chico pero el que más preguntas genera: los nombres de variables con guion, que existen en algunos sistemas. `pydantic-settings` acepta declararlos con `Field`:

```python
import os

from pydantic import Field
from pydantic_settings import BaseSettings

class ConGuion(BaseSettings):
    mi_clave_secreta: str = Field(alias="MI-CLAVE-SECRETA")

os.environ["MI-CLAVE-SECRETA"] = "valor"
print(ConGuion().mi_clave_secreta)

del os.environ["MI-CLAVE-SECRETA"]
```

```
valor
```

El alias es el nombre de la variable de entorno, y el atributo del objeto sigue siendo `mi_clave_secreta`, con guion bajo. Es el mismo mecanismo del capítulo 28 para cambiarle el nombre a una clave de un archivo, aplicado al entorno.

---

## 15. Integración: un archivo INI y un entorno, con las reglas de la pirámide

Estamos en la última parte del capítulo, y es la que junta todo. El enunciado es concreto: **tenés un `app.ini` que no podés cambiar —porque lo escribió otra persona, o porque está en un repositorio que no es tuyo— y un entorno con variables que sí cambian por despliegue.** El orden que prometiste en la sección 2 es: gana el entorno sobre el archivo.

Lo tentador es pasarle el archivo entero a `BaseSettings` y dejar que la biblioteca haga el resto. Y funciona, en el sentido de que el programa arranca. Lo que **no** hace es dejar que el entorno gane.

```python
import configparser

from pydantic_settings import BaseSettings

parser = configparser.ConfigParser()
parser.read("app.ini", encoding="utf-8")

class Config(BaseSettings):
    puerto: int = 8080

print("del archivo:", Config(puerto=parser.getint("servidor", "puerto")).puerto)
```

```
del archivo: 8080
```

`BaseSettings` considera que un valor que le pasás al constructor es un **argumento**, y el argumento está en el nivel más alto de la pirámide: más alto que el entorno. O sea que la variable de entorno que exportaste para cambiar el puerto en ese servidor no se va a leer nunca, porque el valor del archivo ya ocupa ese lugar. El bug más caro de un despliegue, y acá lo produjo la biblioteca en cuatro líneas.

La salida es más clara si lo mirás con las dos capas puestas:

```python
import os

os.environ["PUERTO"] = "9090"
print("con el archivo presente:", Config(puerto=parser.getint("servidor", "puerto")).puerto)
print("sin archivo, solo entorno:", Config().puerto)
```

```
con el archivo presente: 8080
sin archivo, solo entorno: 9090
```

La primera línea es el problema entero. El entorno dice 9090, el archivo dice 8080, y gana el archivo. Y el programa no se queja: no hay error, no hay aviso, hay un puerto que nadie eligió.

La solución no es un truco de la biblioteca, es una decisión de diseño: **mezclar las capas vos, campo por campo, y recién ahí pasarle el resultado.** El archivo primero —porque es el piso—, el entorno encima —porque es más específico—, y recién después el contrato valida.

```python
import configparser
import os

from pydantic_settings import BaseSettings, SettingsConfigDict


def variables_con_prefijo(prefijo: str) -> dict[str, str]:
    salida = {}
    largo = len(prefijo) + 1
    for clave, valor in os.environ.items():
        if clave.startswith(prefijo + "_"):
            campo = clave[largo:].lower().lstrip("_")
            salida[campo.replace("__", "_")] = valor
    return salida


def capas_de(seccion: str, parser: configparser.ConfigParser) -> dict[str, str]:
    desde_archivo = dict(parser[seccion]) if parser.has_section(seccion) else {}
    desde_entorno = variables_con_prefijo(seccion.upper())
    return {**desde_archivo, **desde_entorno}
```

Esa función es todo el mecanismo de la pirámide, en cuatro líneas: `**desde_archivo` primero, `**desde_entorno` después, y el segundo pisa al primero. Es el mismo `{**a, **b}` del capítulo 2, aplicado a configuración. Y el prefijo se aplica **por sección**, no por programa entero, que es lo que hace que esto escale a un archivo de INI: `[servidor]` y `[base]` se resuelven por separado.

El resto es el contrato de la sección 13, aplicado a dos secciones:

```python
from pydantic import BaseModel


class Servidor(BaseModel):
    host: str = "localhost"
    puerto: int = 8080
    debug: bool = False


class Base(BaseModel):
    host: str = "localhost"
    puerto: int = 5432
    usuario: str = "app"
    pool_maximo: int = 5


class Config(BaseSettings):
    model_config = SettingsConfigDict(extra="ignore")

    servidor: Servidor
    base: Base


def cargar(ruta: str) -> Config:
    parser = configparser.ConfigParser()
    leidos = parser.read(ruta, encoding="utf-8")
    if not leidos:
        raise FileNotFoundError(f"no se encontro la configuracion en {ruta}")

    return Config(**{s: capas_de(s, parser) for s in ("servidor", "base")})
```

Y ahora sí, la pirámide funcionando:

```python
config = cargar("app.ini")
print("puerto:", config.servidor.puerto, "| pool_maximo:", config.base.pool_maximo)

os.environ["SERVIDOR__PUERTO"] = "9090"
os.environ["BASE__POOL_MAXIMO"] = "50"
config = cargar("app.ini")
print("puerto:", config.servidor.puerto, "| pool_maximo:", config.base.pool_maximo)
```

```
puerto: 8080 | pool_maximo: 20
puerto: 9090 | pool_maximo: 50
```

El archivo decía 8080 y 20. Con las dos variables de entorno presentes, dice 9090 y 50. La pirámide de la sección 2, medida y funcionando, con un archivo INI que no se modificó.

Y prestá atención a lo que **no** pasó: `SERVIDOR__PUERTO` no tocó `config.base.puerto`, y `BASE__POOL_MAXIMO` no tocó `config.servidor`. La precedencia es por campo, no por sección.

```python
os.environ["BASE__HOST"] = "override.local"
config = cargar("app.ini")
print("host de base:", config.base.host, "| puerto de base:", config.base.puerto)

del os.environ["BASE__HOST"]
del os.environ["BASE__POOL_MAXIMO"]
del os.environ["SERVIDOR__PUERTO"]
```

```
host de base: override.local | puerto de base: 5432
```

Un campo del entorno, un campo del archivo, en la misma llamada. Ese detalle —que la mezcla sea por campo y no por archivo entero— es lo que hace que el mismo programa funcione con el archivo de desarrollo y con el de producción sin que nadie tenga que escribir dos veces la misma configuración.

Tres cosas más que este cargador resuelve sin que las escribas, y que si las hacés a mano vas a escribir mal:

Los errores de conversión llegan con el camino completo. Una variable con un valor que no es un número no produce un `ValueError` suelto: produce el `ValidationError` de la sección 13, con el campo y el tipo de error.

```python
os.environ["BASE__PUERTO"] = "mil"
try:
    cargar("app.ini")
except Exception as e:
    print(type(e).__name__)
    print(str(e).splitlines()[2].strip())

del os.environ["BASE__PUERTO"]
```

```
ValidationError
Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='mil', input_type=str]
```

Un archivo que falta lanza `FileNotFoundError` con la ruta en el mensaje, que es exactamente el `if not leidos` de la sección 7 aplicado donde corresponde. Y un campo que no aparece en ninguna capa no es un problema si el modelo tiene valor por omisión: se llenan los defaults. La diferencia entre un error y un default la decide una línea de la clase, y esa línea es tu política de configuración escrita en Python y no en un comentario.

> **Buenas prácticas:** el orden de las capas se escribe **una vez**, en la función que carga, y se deja anotado en la documentación de la función —el *docstring*— con la pirámide dibujada en cuatro líneas de texto. Nadie debería poder agregar una fuente de configuración sin pasar por esa función, porque esa función es el único lugar del programa donde se decide de dónde sale un valor. Y agregá una función más, de cuatro líneas, que imprima la configuración effective al arrancar: es lo primero que se mira cuando algo está mal, y como los tipos ya están convertidos, la impresión no puede fallar. La regla de los secretos es la de siempre: no se imprime una contraseña, y mejor todavía, el campo de la contraseña se declara con `SecretStr` para que `print` la muestre como `**********` aunque alguien la imprima sin querer.

Con esto ya tenés la máquina completa: cuatro fuentes, una pirámide con un orden escrito y medido, tres formatos que no se confunden entre sí, y un contrato que convierte, valida y dice de qué capa salió cada valor. Lo que sigue es la decisión práctica —qué usar en cada caso— y los ejercicios.

---

## 16. Resumen y conceptos clave

Este capítulo tomó un programa con sus valores escritos adentro y le dio un lugar del que leerlos: **de afuera, en capas, con un orden explícito**. Y de esa idea sale casi todo lo demás.

**Las cuatro fuentes.** Un INI con `configparser` cuando el archivo lo escribís vos y no lo va a tocar nadie: no instala nada, avisa los errores y devuelve texto que hay que convertir. Un archivo `.env` con `python-dotenv` cuando querés esa misma simplicidad pero los valores llegan como variables de entorno. Una variable de entorno suelta cuando el valor es un secreto o lo cambia el despliegue. Y TOML cuando el archivo lo escriben varias personas: tiene tipos, fechas y anidamiento, y un error de sintaxis te da la línea.

**El orden.** Es lo que convierte cuatro fuentes en una sola configuración: argumento, entorno, archivo, valor por omisión. El más específico gana, y el orden se escribe **una vez**, en la función que carga. Ninguna biblioteca te lo da hecho si el archivo es INI, porque `BaseSettings` ve un argumento y lo pone arriba de todo —y un valor de archivo que entra por ahí ya no puede ser pisado por el entorno.

**La conversión.** Todo lo que llega de un archivo o de una variable de entorno es texto, y el programa que suma enteros con cadenas falla en el lugar equivocado. `getint`, `getboolean` y `getfloat` de un lado; un modelo de `pydantic-settings` del otro, que además convierte, valida y avisa qué campo falta antes de que el programa haga nada. Ese es el argumento a favor del contrato: no es que el programa ande mejor, es que **falla mejor**.

**El orden del archivo, que también es semántica.** `ConfigParser` devuelve lo que leyó en el orden en que está escrito, y `getboolean` acepta `1`, `yes`, `true` y `on` como verdadero: son cuatro maneras de decir la misma cosa. Lo que se escribe primero, se lee primero. Por eso una configuración es un documento que se lee de arriba abajo, no una tabla de variables.

| Fuente | Cuándo gana | Quién la escribe | Se versiona | Trae tipos |
|---|---|---|---|---|
| Argumento | siempre | quien arranca el programa | no | sí, si el constructor los declara |
| Entorno | si está definida | el despliegue | no | no, siempre texto |
| Archivo INI | si no está en el entorno | quien configura la máquina | a veces | no, hay que convertir |
| Archivo `.env` | si no está en el entorno | quien configura la máquina | no | no, hay que convertir |
| Archivo TOML | si no está en el entorno | quien configura la máquina | sí | sí, declarados |
| Valor por omisión | si no está en ninguna | vos, en el código | sí | sí, declarados |

> **Dato clave:** la parte de la pirámide que se olvida siempre es la de abajo del todo: **imprimir la configuración efectiva al arrancar**. Una línea con los campos ya convertidos, más el secreto enmascarado. Cuando algo de la configuración no funciona, el primer movimiento no es abrir el código del programa: es mirar esa línea. La mitad de los errores de configuración que parecen errores de lógica son, en realidad, un valor que llegó de una capa que no sabías que estaba.

Repasá el checklist antes de seguir:

- [ ] Una fuente de configuración, un lugar en el código que la lee. Si hay un segundo camino para obtener un valor, ese segundo camino es un bug esperando.
- [ ] El orden de las capas **escrito** en la documentación de la función que carga, no solamente en tu cabeza.
- [ ] Todos los valores llegan convertidos. Si queda un `int(...)` en la lógica, algo se está haciendo mal.
- [ ] El programa **falla al arrancar** si falta un campo obligatorio, y el error dice el nombre del campo.
- [ ] `ConfigParser.read()` no avisa si el archivo no existe: controlá el valor que devuelve.
- [ ] `fallback` cubre la clave ausente, no la conversión inválida.
- [ ] Cualquier `%` en un valor INI es un error de sintaxis salvo que lo dupliques, o usá `interpolation=None`.
- [ ] `load_dotenv()` **no** pisa el entorno; `override=True` sí. Es la diferencia entre un despliegue que se rompe y uno que se arregla.
- [ ] `find_dotenv()` sube por los directorios desde el archivo que llama, no desde el directorio actual.
- [ ] Un `dict` declarado como campo de Pydantic **se reemplaza** entero; un modelo anidado conserva sus defaults.
- [ ] El secreto no está en el repositorio, no se imprime al arrancar, y si se imprime por error sale enmascarado.
- [ ] Hay una prueba automatizada que pone una variable de entorno y comprueba que pisa al archivo. Tres líneas, y es la que te ahorra el incidente.

Y la pregunta con la que cerramos, que es la que te llevás al capítulo 30: **¿de dónde sale este valor, y quién tiene derecho a cambiarlo?** Es la misma pregunta que se van a hacer los patrones de diseño, pero aplicada a la configuración: si la respuesta es "cualquier parte del programa", el problema no es el formato del archivo, es que el valor no tenía un dueño. El capítulo 30 es el de las soluciones que ya tienen dueño: una clase, un lugar, un nombre.

---

## 17. Ejercicios

Cada ejercicio tiene una respuesta en la sección siguiente. Los tres primeros se resuelven con la biblioteca estándar; los tres últimos necesitan `python-dotenv` o `pydantic-settings`.

**Ejercicio 1.** Escribí un archivo INI con una sección `[app]` que tenga `nombre`, `puerto`, `debug` y `ratio`, y leelo con `configparser` mostrando cada valor **ya convertido** a su tipo. Después cambiá `ratio` a un valor con un signo de porcentaje y leé el archivo otra vez: el programa tiene que fallar, y el mensaje tiene que decir que el problema es de sintaxis de interpolación. Después cambiá `ConfigParser(interpolation=None)` y comprobá que el mismo archivo ahora se lee bien.

**Ejercicio 2.** En un `ConfigParser`, creá la sección `[DEFAULT]` con `timeout = 30` y dos secciones que una lo pisa y otra no. Imprimí `options()` de las dos, `sections()`, y `dict(parser.defaults())`, y explicá con tus palabras por qué `DEFAULT` no aparece en `sections()`.

**Ejercicio 3.** Escribí una función que tome una lista de rutas de archivos INI y los lea **en orden**, y que devuelva un `dict` con la mezcla de todos: el último que define una clave gana. Después probala con dos archivos donde el segundo pisa una clave y deja otra sin tocar, y verificá que la clave que no tocaste vino del primero.

**Ejercicio 4.** Creá un archivo `.env` con cinco variables, una de ellas con un valor entre comillas que contiene un `#`, otra con un comentario al final de la línea, y una con `${}` que referencie otra del mismo archivo. Leelo con `dotenv_values` y después con `load_dotenv`. Comprobá que `load_dotenv` no pisa una variable que ya estaba en el entorno, y que sí la pisa si le pasás `override=True`.

**Ejercicio 5.** Escribí un archivo TOML con una tabla `[servidor]`, una tabla anidada `[servidor.pool]` y una lista de tablas `[[servidor.lista]]` de tres elementos. Leelo con `tomli` e imprimí el tipo de cada valor. Después cambiá el archivo para que un valor sea `null` y comprobá que el archivo es inválido.

**Ejercicio 6.** Declará una clase `Config` con `BaseSettings` que tenga `nombre: str`, `puerto: int = 8080` y `etiquetas: list[str] = []`, con `env_prefix="APP_"`. Medí y escribí el orden de prioridad de las cuatro capas, y comprobá con un `try` que un `APP_PUERTO="ocho mil"` en el entorno detiene el programa con un `ValidationError` que menciona el campo.

---

## 18. Soluciones

**Ejercicio 1.**

```python
import configparser

con_parser = configparser.ConfigParser()
sin_parser = configparser.ConfigParser(interpolation=None)

TEXTO = """
[app]
nombre = mi app
puerto = 8080
debug = true
ratio = 0.75
"""

for clave, funcion in (("puerto", "getint"), ("debug", "getboolean"), ("ratio", "getfloat")):
    con_parser.read_string(TEXTO)
    print(clave, "->", getattr(con_parser, funcion)("app", clave))

con_ratio = TEXTO.replace("ratio = 0.75", "ratio = 75%")
con_parser.read_string(con_ratio)
try:
    con_parser.get("app", "ratio")
except Exception as e:
    print(type(e).__name__)

sin_parser.read_string(con_ratio)
print("sin interpolacion:", sin_parser.get("app", "ratio"))
```

La salida es esta:

```
puerto -> 8080
debug -> True
ratio -> 0.75
InterpolationSyntaxError
sin interpolacion: 75%
```

El orden de las dos últimas líneas es el resultado del ejercicio: el mismo texto que con el parser por omisión es un archivo inválido, con `interpolation=None` es un archivo perfectamente válido. Lo que se desactiva no es "el porcentaje", es la interpolación entera.

**Ejercicio 2.**

```python
import configparser

parser = configparser.ConfigParser()
parser.read_string("""
[DEFAULT]
timeout = 30
charset = utf-8

[web]
timeout = 5

[api]
""")

for seccion in ("web", "api"):
    print(seccion, "->", parser.options(seccion), "timeout:", parser.getint(seccion, "timeout"))

print("secciones:", parser.sections())
print("defaults:", dict(parser.defaults()))
```

```
web -> ['timeout', 'charset'] timeout: 5
api -> ['timeout', 'charset'] timeout: 30
secciones: ['web', 'api']
defaults: {'timeout': '30', 'charset': 'utf-8'}
```

`api` no escribe ni `timeout` ni `charset` y los tiene igual. Y fijate en el orden de `web`: primero `timeout`, que es suyo, y después `charset`, que es heredada. `DEFAULT` no está en `sections()` porque no es una sección: es el valor por omisión contra el que se resuelven todas.

**Ejercicio 3.**

```python
import configparser


def mezclar(rutas: list[str]) -> dict[str, dict[str, str]]:
    salida: dict[str, dict[str, str]] = {}
    for ruta in rutas:
        parser = configparser.ConfigParser()
        leidos = parser.read(ruta, encoding="utf-8")
        if not leidos:
            raise FileNotFoundError(f"falta {ruta}")
        for seccion in parser.sections():
            salida.setdefault(seccion, {}).update(dict(parser[seccion]))
    return salida
```

Con `base.ini` diciendo `host = del_base` y `usuario = comun` en `[db]`, y `produccion.ini` diciendo `host = del_produccion` en `[db]`, la mezcla es `{'db': {'host': 'del_produccion', 'usuario': 'comun'}}`. El `.update()` es el que hace el trabajo: la clave repetida se pisa, la que no se repite se conserva. Es exactamente el `{**a, **b}` de la sección 15, aplicado a varias fuentes y a la vez.

**Ejercicio 4.**

```python
import os

from dotenv import dotenv_values, load_dotenv

with open("demo.env", "w", encoding="utf-8") as f:
    f.write(
        "BASE=http://localhost\n"
        "RAIZ=${BASE}/api\n"
        "COMILLA=\"con # hash\"\n"
        "CON_COMENTARIO=antes # despues\n"
        "VACIA=\n"
    )

for clave, valor in dotenv_values("demo.env").items():
    print(clave, "->", repr(valor))

os.environ["RAIZ"] = "ya estaba"
print("cargado:", load_dotenv("demo.env"))
print("RAIZ:", os.getenv("RAIZ"))

load_dotenv("demo.env", override=True)
print("con override:", os.getenv("RAIZ"))
```

La salida:

```
BASE -> 'http://localhost'
RAIZ -> 'http://localhost/api'
COMILLA -> 'con # hash'
CON_COMENTARIO -> 'antes'
VACIA -> ''
cargado: True
RAIZ: ya estaba
con override: http://localhost/api
```

`dotenv_values` no toca el entorno, y por eso `RAIZ` muestra `http://localhost/api` aunque todavía no exista en `os.environ`. Las dos líneas del final son la demostración: el entorno gana, hasta que le digás lo contrario.

**Ejercicio 5.**

```python
import tomli

TEXTO = """
[servidor]
host = "localhost"
puerto = 8080

[servidor.pool]
minimo = 1
maximo = 10

[[servidor.lista]]
nombre = "uno"

[[servidor.lista]]
nombre = "dos"
"""

datos = tomli.loads(TEXTO)
print("pool:", datos["servidor"]["pool"])
print("lista:", datos["servidor"]["lista"])
print("host es", type(datos["servidor"]["host"]).__name__)

try:
    tomli.loads("[app]\nactivo = null\n")
except Exception as e:
    print(type(e).__name__)
```

```
pool: {'minimo': 1, 'maximo': 10}
lista: [{'nombre': 'uno'}, {'nombre': 'dos'}]
host es str
TOMLDecodeError
```

`[servidor.pool]` no necesita una clave `pool =` en `[servidor]`: la jerarquía se arma con los corchetes. Y `lista` es una lista de diccionarios, no un diccionario con claves que se pisan.

**Ejercicio 6.**

```python
import os

from pydantic_settings import BaseSettings, SettingsConfigDict


class Config(BaseSettings):
    model_config = SettingsConfigDict(env_file="app.env", env_prefix="APP_")

    nombre: str
    puerto: int = 8080
    etiquetas: list[str] = []

os.environ["APP_PUERTO"] = "1111"
print("2) entorno:", Config(nombre="x").puerto)
print("1) argumento:", Config(nombre="x", puerto=2222).puerto)
del os.environ["APP_PUERTO"]
print("3) solo el archivo:", Config(nombre="x").puerto)


class SinArchivo(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="APP_")

    puerto: int = 8080

print("4) valor por omision:", SinArchivo().puerto)

os.environ["APP_PUERTO"] = "ocho mil"
try:
    Config(nombre="x")
except Exception as e:
    print(str(e).splitlines()[2].strip())

del os.environ["APP_PUERTO"]
```

Con `APP_PUERTO=7000` en `app.env`, la salida es:

```
2) entorno: 1111
1) argumento: 2222
3) solo el archivo: 7000
4) valor por omision: 8080
Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='ocho mil', input_type=str]
```

Las cuatro primeras líneas son la pirámide de la sección 2, medida y escrita, y la quinta es lo que la convierte en un contrato: el valor imposible no llega al programa como un texto que alguien va a sumar más tarde, llega como un error que dice qué campo es y qué se esperaba.

Y con esto terminaste lo que empezaste en la sección 1. Un programa que antes tenía su puerto escrito adentro y su contraseña pegada en el código ahora recibe los dos de afuera, los convierte a los tipos correctos, y se niega a arrancar si algo no tiene sentido. Eso no es un detalle de la infraestructura: es la diferencia entre un programa que se puede instalar en dos máquinas y uno que solo funciona en la tuya.
