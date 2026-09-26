# Capítulo 28 — Serialización: mandar un objeto por el mundo

El capítulo 27 te enseñó a leer y escribir archivos, y lo hizo con una observación que ahora va a pagar: todo lo que pasó por `open()` en modo texto fueron **letras**. Un archivo de texto es una secuencia de caracteres, y los caracteres son alfombras de ideas que existen en todos los lenguajes. Te faltaba el otro lado: la mayoría de los datos que querés guardar **no son letras**. Son números con unidades, fechas, precios exactos, clases que definiste vos. Y esos datos no entran en un archivo de texto sin que algo se pierda en el traslado.

Este capítulo es el de esa pérdida. Vamos a tomar un objeto de Python y hacerlo viajar — a un archivo, a un mensaje, a otro programa, a otro día — y vamos a ver, dato por dato, **qué sobrevive del viaje y qué no**. Al final vas a tener una respuesta que mucha gente tarda años en conseguir y que se reduce a una pregunta: *¿esto lo va a leer otro programa, otro humano, o un yo del futuro?* Según quién lea, se serializa de una manera o de otra.

Los tres caminos que vas a ver son, en orden: **JSON**, que es el idioma que habla todo el mundo; **YAML**, que es el mismo idioma con corbetas y menos ruido; y **pickle**, que no es un idioma sino un trozo de Python portable. Y al final, dos contenedores de la biblioteca estándar que no aparecen en ningún tutorial: `shelve`, que es una base clave-valor de la stdlib, y el despachador por extensión que usa cualquier programa real que acepta configuración.

---

## 1. El problema: un objeto no entra en un archivo

Empecemos por el problema concreto, porque es más fácil que el abstracto. Tenés un diccionario con información de un cliente y querés mandárselo a alguien por un canal de texto —un mensaje, un correo, un ticket—:

```python
from datetime import datetime

cliente = {
    "nombre": "Acero SRL",
    "alta": datetime(2024, 3, 15, 10, 30),
    "cursos": ["python", "sql"],
    "ubicacion": ("cordoba", 33, -70),
}

print(type(cliente))
print(cliente)
```

Lo que tenés es un `dict` de Python. Lo que el canal de texto acepta es un `str`. Hay que traducir. La traducción más obvia —`str()`— funciona y está disponible sin importar nada:

```python
mensaje = str(cliente)
print(mensaje)
print(len(mensaje))
```

Funciona, no tira error, y parece resuelta el problema. No lo resolvió. Mandás esa línea y quien la recibe tiene **una cadena de texto**, no un diccionario. Y tiene que volver a hacer, a mano, el trabajo de adivinar dónde están las comas, dónde los acentos y dónde los límites de cada dato. Ese parseo manual es una fragilidad pura: mañana alguien agrega un campo con una coma adentro y tu mensaje deja de poder leerse.

Y hay un problema más profundo, que solo aparece cuando lo mirás de cerca. `str()` imprime los **valores** y se olvidó por completo de los **tipos**. Mirá lo que pasó con la fecha y con la tupla:

```python
print(type(cliente["alta"]))
print(cliente["alta"])
print(type(cliente["ubicacion"]))
print(cliente["ubicacion"])
```

La fecha era un `datetime`. Ahora es un `str` que dice `"2024-03-15 10:30:00"`. Parece la misma cosa y no lo es: ya no podés sumarle días, ni comparar dos fechas con `<`, ni pedirle `.year`. La tupla era una `tuple` y ahora es un texto que empieza con un paréntesis. Los dos datos viajaron **como forma de texto**, y la forma se perdió.

> **Dato clave:** un `str` conserva la **apariencia** de un dato, no su **significado**. La diferencia entre los dos es exactamente la diferencia entre un número escrito en un papel y el número que tenés en la calculadora: se leen igual, solo uno sirve para calcular. Serializar bien es elegir, para cada dato, si querés mandar la apariencia o el significado.

La pregunta del capítulo, entonces, no es *"cómo escribo esto"*. Es: **¿qué tengo que perder para que quepa en un texto?** Y hay que perder algo, siempre. Un archivo de texto es una secuencia de caracteres; cualquier tipo que quieras guardar es una estructura de datos. La pregunta es si el formato que uses sabe reconstruir esa estructura, y de qué le permitís que se pierda en el camino.

Los tres formatos que vas a ver dan tres respuestas distintas a esa pregunta, y por eso existen los tres. JSON decide que el significado no viaja y te devuelve todo como `dict`, `list`, `str`, `int`, `float`, `bool` y `None`. YAML hace lo mismo pero escrito para que un humano lo lea cómodo. Y pickle decide que el significado **sí** viaja, y reconstruye el objeto tal cual era, con su `datetime`, su `tuple`, su clase y hasta sus funciones. Los tres tienen razón en su contexto. Elegir mal es el error.

---

## 2. La matriz de lo que viaja

Antes de usar nada, conviene tener el mapa. Este cuadro es el que tenés que tener en la cabeza cuando estés decidiendo: son los tipos que la biblioteca estándar de Python sabe convertir a texto y volver, sin ayuda.

| Tipo de Python | ¿Viaja en JSON? | ¿Vuelve como qué? | Nota |
|---|---|---|---|
| `str` | sí | `str` | el viaje redondo |
| `int` | sí | `int` | ojo con los números muy grandes, abajo |
| `float` | sí | `float` | y ojo con la coma decimal, más abajo |
| `bool` | sí | `bool` | sale como `true` / `false`, en minúscula |
| `None` | sí | `None` | sale como `null` |
| `list` | sí | `list` | el viaje redondo |
| `dict` | sí | `dict` | las claves tienen que ser `str` |
| `tuple` | sí | **`list`** | la trampa silenciosa, la vemos en la sección 5 |
| `set` | **no** | — | `TypeError`, falla ruidosamente |
| `bytes` | **no** | — | hay que convertirlo a texto a mano |
| `datetime`, `date` | **no** | — | hay que convertirlo a texto a mano |
| `Decimal` | **no** | — | se vuelve `float` si lo dejás pasar |
| clase propia | **no** | — | hay que describirla a mano |

Leé las filas de abajo otra vez. Cinco de los doce tipos más comunes de tu vida diaria **no viajan**. Y no es un defecto del formato: es que un archivo de texto no tiene dónde anotar que esto era una fecha y no una frase. La información de que algo es una fecha **no está en los bytes**: está en tu cabeza, y es lo único que permite que `json` la vuelva a construir. Por eso, cuando le decís "esto es una fecha", el programa tiene que guardarlo de alguna forma —como un texto con un formato reconocible— y después adivinar de vuelta.

Probemos la diferencia entre las dos noticias, la que falla y la que avisa:

```python
import json

fácil = {"cliente": "Acero SRL", "monto": 15200.5, "urgente": True, "vence": None}
print(json.dumps(fácil))
```

Eso sale bien. Ahora el mismo diccionario con una fecha adentro:

```python
con_fecha = {"cliente": "Acero SRL", "alta": datetime(2024, 3, 15)}
try:
    print(json.dumps(con_fecha))
except TypeError as e:
    print(type(e).__name__, "|", e)
```

`TypeError: Object of type date is not JSON serializable`. Es un error **claro y con nombre**, y esa es la buena noticia: la biblioteca te dice exactamente qué objeto se le atravesó. La mala noticia es que un `dict` con veinte campos puede tener el `TypeError` en el primero que no conozca, y vas a tener que adivinar cuál es. Eso se arregla con una sola función, y la vemos en dos secciones.

> **Buenas prácticas:** antes de serializar algo, hacé la pregunta de la matriz en voz alta — *¿qué tipos tengo adentro?* — y buscá los que están en negrita de la tabla. Es más barato que descubrirlo con un `TypeError` en producción, y más barato todavía que descubrirlo con un `TypeError` en el servidor de un cliente. Los tipos que no viajan son los que te van a obligar a pensar qué representás con texto.

---

## 3. JSON: las cuatro funciones

`json` viene en la biblioteca estándar, sin instalar nada, y tiene una API de cuatro funciones que parecen muchas pero son dos operaciones repetidas: **convertir a texto** y **convertir desde texto**, cada una en dos versiones según quieras o no un archivo.

| Función | Qué hace | Devuelve |
|---|---|---|
| `json.dumps(obj)` | objeto → texto | `str` |
| `json.loads(texto)` | texto → objeto | lo que corresponda |
| `json.dump(obj, archivo)` | objeto → archivo | escribe en el archivo |
| `json.load(archivo)` | archivo → objeto | lo que corresponda |

La diferencia entre el par con `s` y el par sin `s` es exactamente la que ya conocés del capítulo 27: `dumps`/`loads` trabajan con **cadenas**, `dump`/`load` trabajan con **archivos**. La `s` viene de *string*. No hay nada más que aprender de la API; todo lo demás son opciones.

Empecemos por el viaje de ida completo, con las dos opciones que vas a usar todos los días:

```python
import json

factura = {"numero": 7, "cliente": "Acero SRL", "monto": 99.5}

print(json.dumps(factura))
print(json.dumps(factura, indent=2))
print(json.dumps(factura, sort_keys=True))
```

`indent=2` es lo que hace que un archivo de configuración sea legible por un humano. Sin eso sale todo en una línea, y es técnicamente correcto y humanamente inservible. `sort_keys=True` ordena las claves alfabéticamente: útil cuando querés que dos archivos se comparen bien en un control de versiones, porque el orden de las claves en un `dict` de Python es el orden en que las insertaste, y eso cambia entre versiones del programa.

Dos opciones más, que parecen detalles y no lo son. La primera es `ensure_ascii`, y tiene un costo medible:

```python
con_acento = {"ciudad": "Córdoba", "pais": "Argentina"}

print(json.dumps(con_acento))
print(json.dumps(con_acento, ensure_ascii=False))
print(len(json.dumps(con_acento).encode("utf-8")))
print(len(json.dumps(con_acento, ensure_ascii=False).encode("utf-8")))
```

Por defecto, `json` **escapa todo lo que no sea ASCII**: la `ó` se escribe `ó`, seis caracteres en vez de dos. Con `ensure_ascii=False` sale la `ó` de verdad, y el archivo pesa menos. La razón de la existencia del parámetro es histórica —los lectores de JSON del 2000 sabían solo leer ASCII— pero hoy casi siempre querés `False`, sobre todo si el archivo lo va a leer una persona. Lo vas a ver en el capítulo 29, aplicado a archivos de configuración, donde la legibilidad es el objetivo.

Y la segunda es la forma de escribir a archivo, que es la que se parece a lo que hiciste con `csv` en el capítulo 27:

```python
from pathlib import Path

ruta = Path("datos/factura.json")
ruta.parent.mkdir(parents=True, exist_ok=True)
with open(ruta, "w", encoding="utf-8") as f:
    json.dump(factura, f, indent=2, ensure_ascii=False)

print(ruta.read_text(encoding="utf-8"))
```

Fijate en la última línea: el `open` es modo texto con `encoding` explícito, tal como el capítulo 27 te hizo prometer. `json.dump` escribe sobre un archivo que vos abriste, y hereda todas las reglas del capítulo 27 —codificación, saltos de línea, cierre con `with`. `json` no te exime de nada de lo que aprendiste; solo te da una función que escribe la parte chata.

> **Dato clave:** `json.dump` y `json.load` no abren ni cierran archivos. Reciben un archivo **ya abierto**. Es una decisión de diseño muy buena y es la razón por la que podés meter un `json.dump` dentro de un `with` que también escribe otras cosas, o en un archivo que compres con `gzip`, o en un objeto de red. Cuando una librería te da `dump`/`load` además de `dumps`/`loads`, siempre gana la versión con archivo: es la que se integra con lo que ya sabés hacer.

---

## 4. Lo que JSON no sabe: el parámetro `default`

Volvamos a la fila problemática de la matriz. `json` no sabe qué es una fecha. Pero vos sí: la fecha es `2024-03-15`, un texto con un formato reconocible. Si le decís a `json` cómo se escribe una fecha, puede reconstruirla leyendo el texto. Eso se hace con el parámetro `default`, que es una **función**: la llamás vos cuando `json` se topó con algo que no reconoce, y vos devolvés algo que sí reconoce.

```python
import json
from datetime import date, datetime
from decimal import Decimal

class Factura:
    def __init__(self, numero, monto):
        self.numero, self.monto = numero, monto

def a_json(objeto):
    """Le dice a json cómo convertir los tipos que no conoce."""
    if isinstance(objeto, (date, datetime)):
        return objeto.isoformat()          # 'YYYY-MM-DD' o 'YYYY-MM-DDTHH:MM:SS'
    if isinstance(objeto, Decimal):
        return str(objeto)                 # el precio exacto, no aproximado
    if isinstance(objeto, set):
        return sorted(objeto)              # el set no tiene orden: lo imponemos
    if isinstance(objeto, Factura):
        return {"numero": objeto.numero, "monto": objeto.monto}
    raise TypeError(f"no sé convertir {type(objeto).__name__}")

datos = {
    "cliente": "Acero SRL",
    "alta": date(2024, 3, 15),
    "tags": {"urgente", "dolar"},
    "saldo": Decimal("10.25"),
    "factura": Factura(7, 99.5),
}

print(json.dumps(datos, default=a_json))
```

`isoformat()` no es un detalle: es el método que produce el formato de fecha que el estándar ISO 8601 ordena, que es el que JSON y casi todo el resto del mundo usan. Si en vez de `isoformat()` usaras `str(objeto)`, en este caso te daría el mismo resultado, pero no siempre: `str(date(2024, 3, 5))` da `"2024-03-05"` y `str(date(2024, 3, 15))` da `"2024-03-15"`, pero para algunos formatos las reglas se rompen. `isoformat()` no se rompe nunca, porque su único trabajo es ese.

Y hay un atajo que parece tempting y es un error. `json` acepta `default=str`, que convierte **todo** lo desconocido a texto:

```python
print(json.dumps(datos, default=str))
```

No tira error, sale algo, y el programa sigue. Pero mirá lo que pasó con el `set`: se convirtió en una cadena con las llaves y las comillas del `repr` de Python adentro, algo como `"{'urgente', 'dolar'}"` — y el orden de esas dos palabras cambia de una ejecución a la otra, porque un `set` no tiene orden. Del otro lado del viaje vas a tener un `str` con unos `{` de adorno, no una lista. El tipo **parece** resuelto y en realidad se rompió en silencio. `default=str` es la manera más eficiente de convertir un error visible en un bug invisible.

> **Buenas prácticas:** `default=` es la costura donde tu programa le habla al formato. Poné la función en un solo lugar del proyecto, con un nombre que diga qué hace, y usá `raise TypeError` como último recurso. Con eso, un tipo que se te escape no se convierte en texto basura: se convierte en un error que dice el nombre del tipo. Es la diferencia entre un programa que falla ruidosamente y uno que guarda datos que nadie va a poder leer después.

La otra mitad del problema es simétrico y se resuelve en el sentido inverso. Si vos le dijiste que una fecha es un texto con un formato ISO, el que lee tiene que hacer la operación contraria, y `json` no lo hace solo:

```python
texto = '{"numero": 7, "alta": "2024-03-15"}'
leido = json.loads(texto)
print(leido["alta"], type(leido["alta"]))
```

Vuelve `str`, no `date`. Te falta la vuelta. Y esto **no es un defecto de `json`**: es que el archivo no dice *"esto es una fecha"*, dice *"esto es un texto que parece una fecha"*. La información se perdió en la ida, y el que la tiene que recuperar sos vos. Volveremos a esto en la sección 14, con pydantic, que hace las dos mitades por vos.

---

## 5. El viaje de vuelta: `is` contra `==`, y la trampa de la `tuple`

Hay un comportamiento de todos los formatos de serialización que conviene entender bien, porque da errores muy confusos: **el viaje de vuelta siempre crea un objeto nuevo**. Nunca recuperás la misma instancia que tenías antes.

```python
import json

original = {"lista": [1, 2, 3], "nombre": "Acero SRL"}
texto = json.dumps(original)
vuelta = json.loads(texto)

print(vuelta == original)
print(vuelta is original)
print(vuelta["lista"] is original["lista"])
print(vuelta["lista"] == original["lista"])
```

Leé los cuatro resultados con calma, porque la combinación es la que importa: `vuelta == original` es `True` (el contenido se conservó) y `vuelta is original` es `False` (pero son dos objetos distintos). Y algo que sorprende más: **la lista interior tampoco se comparte**. No es que el `dict` de afuera sea nuevo y el de adentro el mismo; los dos son nuevos. Lo mismo pasa con `pickle`: cada `loads` construye el objeto entero de nuevo.

Esto no es un detalle, es una **decisión de diseño** de todos estos formatos, y es la correcta. Si el objeto recuperado fuera el mismo, tendrías dos nombres apuntando a la misma cosa, y cualquier cambio en uno cambiaría el otro sin que nadie lo pidiera. Los formatos de serialización te devuelven una **copia**, siempre. Cuando el flujo de datos necesita compartir, eso se resuelve con referencias explícitas, no con un efecto escondido.

Ahora, la trampa silenciosa. Mirá qué le pasa a una `tuple` en el viaje:

```python
print(json.dumps((1, 2, 3)))
print(json.loads(json.dumps((1, 2, 3))))
print(type(json.loads(json.dumps((1, 2, 3))).__class__))
```

Y el `set`, en cambio:

```python
try:
    json.dumps({1, 2})
except TypeError as e:
    print(type(e).__name__, "|", e)
```

La `tuple` se convirtió en `list`, **sin avisar**. Nadie te dio un error, no hubo un warning, y sin embargo un tipo se perdió en el viaje. El `set` en cambio falla con un `TypeError` claro. Es al revés de lo que uno esperaría: el tipo que "perdía algo" es el que pasa desapercibido.

Y sí: hay un lugar donde ese comportamiento vuelve a morder. El `json` también usa `list` como su único tipo de arreglo, así que es imposible distinguir una `tuple` de una `list` mirando el archivo. Un programa que escribe, otro programa que lee, y entre los dos un cambio de `tuple` a `list` en el primero: el segundo no se entera nunca.

> **Buenas prácticas:** si un `tuple` es parte de tu contrato de datos y te importa que vuelva a ser `tuple`, no lo mandes a JSON como `tuple`. Mandalo como `list` a propósito y convertilo vos al leer, con una línea: `tuple(datos["ubicacion"])`. Convertir a mano un solo tipo es barato; detectar después, en tres meses, que un campo público cambió de `list` a `tuple` en un cliente, es caro. Y si el tipo **no** lo vas a necesitar del otro lado, ni lo mires: la pérdida de la `tuple` a `list` no es un error, es un detalle de implementación de `json` que a tu programa no le afecta.

Ese `str` de la sección 4 era otro caso del mismo fenómeno, y el `datetime` es el ejemplo grande: parece un `str` y es otra cosa. Después de esta sección ya tenés las dos caras del mismo problema: `json` es **fiel** cuando los tipos son los suyos, y **mentiroso** cuando tiene que adivinar. Saber cuándo estás en cada una de las dos situaciones es la habilidad que separa a quien usa `json` de quien pelea con `json`.


---

## 6. YAML: la misma información, con menos ruido

JSON es un formato excelente y va a estar con vos siempre. Pero mirá cómo se ve un archivo de configuración en JSON y cómo se vería el mismo archivo como *algo que una persona va a leer y editar*:

```python
import json

config = {
    "servidor": {
        "host": "localhost",
        "puerto": 8080,
        "tls": {"activo": True, "certificado": "/etc/ssl/cert.pem"},
    },
    "reintentos": 3,
}

print(json.dumps(config, indent=2))
```

Es perfectamente legible, si sabés leer JSON. El problema es que cada nivel de anidamiento cuesta dos líneas —una de apertura, una de cierre— y cada clave está entre comillas. Para un archivo de diez claves está perfecto. Para uno de doscientas, o para un archivo que alguien edita a mano una vez por mes, se vuelve un muro de ruido donde un error de tipeo en una llave es casi imposible de ver.

YAML es exactamente la misma información sin ese ruido:

```python
import yaml

print(yaml.dump(config, allow_unicode=True, sort_keys=False))
```

Menos de la mitad de líneas, sin comillas, y la estructura del `dict` se lee directo. Eso es todo lo que YAML es, en el fondo: **JSON con un formato más barato de leer y de escribir**. No es un lenguaje de programación, no agrega lógica, no calcula. Su única promesa es la misma que la de los archivos `.ini` que ya usaste sin saber que existían: *que un humano pueda editar el archivo sin romperlo*.

Para leerlo hay dos funciones, y una de ellas es obligatoria:

```python
texto = "host: localhost\npuerto: 8080\n"
print(yaml.safe_load(texto))
```

`safe_load` es la única forma de cargar un archivo que venga de afuera. Y no es una recomendación de estilo, es una consecuencia de cómo está escrito `yaml` hoy. Fijate qué pasa si no lo usás:

```python
try:
    yaml.load(texto)
except TypeError as e:
    print(type(e).__name__, "|", e)
```

`TypeError: load() missing 1 required positional argument: 'Loader'`. Hace años, `yaml.load(texto)` construía **cualquier objeto que el archivo pidiera**, incluidos objetos que ejecutan código al construirse. Hoy la biblioteca te obliga a **nombrar** qué cargador estás usando, y si no lo nombrás, te dice que no. Ese `TypeError` no es una molestia: es la barrera de seguridad poniéndose en el lugar.

Si la nombrás, el riesgo vuelve:

```python
print(yaml.load(texto, Loader=yaml.Loader))
```

Funciona. Sale el mismo `dict`. Pero ahora el archivo no es solo datos: es un **programa** que dice "construí esto". Con `safe_load` solo puede construir `dict`, `list`, `str`, números, booleanos y `None`. Con `Loader` puede construir lo que le pidan. La palabra `safe` del nombre no es propaganda: es la lista de tipos que el cargador se permite construir.

> **Buenas prácticas:** `yaml.safe_load` y `yaml.safe_dump`, siempre, sin excepción, aunque el archivo lo hayas escrito vos. La tentación de usar el `Loader` rápido es real, pero el argumento "este archivo es mío" se vuelve falso el día que alguien te pasa un archivo de configuración que se descargó de algún lado. El costo de `safe_load` sobre `Loader` es de microsegundos; el costo de construir un objeto arbitrario es otro.

Y una diferencia práctica que te va a morder el mismo día: **YAML no acepta tabuladores para indentar**. Solo espacios.

```python
Path("con-tab.yaml").write_text("cliente:\n\tnombre: Acero SRL\n", encoding="utf-8")
try:
    yaml.safe_load(Path("con-tab.yaml").read_text(encoding="utf-8"))
except yaml.YAMLError as e:
    print(type(e).__name__, "|", str(e).splitlines()[1].strip())
```

`found character '\t' that cannot start any token`, con el caret señalando la línea exacta. La regla es simple y conviene dejarla escrita en la cabeza: **en YAML, un espacio es un carácter con significado y un tabulador es un error**. Es la diferencia entre un lenguaje donde el espacio es decorativo —como Python— y uno donde el espacio es parte de la gramática. En Python el indentado es obligatorio y por eso todo el mundo lo entiende; en YAML el indentado también, pero many programmers llegan con la costumbre de configurar el editor con tabs, y el error que sale es bastante críptico.

---

## 7. YAML adivina los tipos, y a veces adivina mal

Acá está el dato que separa a los que usan YAML de los que se pelean con YAML, y no aparece en ningún tutorial: **YAML infiere el tipo de cada valor**. `1.10` no es un texto, es un número. `yes` no es un texto, es un booleano. Y hay una consecuencia que ya te debe sonar familiar, porque la viste recién con las tuplas de `json`.

```python
print(yaml.safe_load("version: 1.10"))
print(yaml.safe_load("ok: yes"))
print(yaml.safe_load("nada: null"))
```

Sale `{'version': 1.1}`, `{'ok': True}`, `{'nada': None}`. Prestá atención al primero: escribiste `1.10` y recibiste el flotante `1.1`. El cero final no viaja, porque un número flotante no puede saber si era un `1.1` o un `1.10`. Es el mismo tipo de pérdida que la `tuple` que se volvió `list`, pero más sutil, porque el valor *parece* el mismo.

Y el caso que de verdad duele, porque aparece en datos reales y no se parece a nada:

```python
print(yaml.safe_load("codigo_postal: 0123"))
print(yaml.safe_load("extension: 00456"))
print(yaml.safe_load("codigo_postal: '0123'"))
```

El primero te da `83`. El segundo te da `302`. Y el tercero, con las comillas, te da `'0123'`, que es lo que querías. Te está diciendo tres cosas: el `0` inicial **no es decorativo**, es la marca de que el número es **octal**, y `0o123` en base ocho son ochenta y tres.

Esto viene de la versión 1.1 de la especificación de YAML, que es la que implementa PyYAML, y que esvieja: define `y`, `yes`, `on`, `off`, `n`, `no` como booleanos además de `true` y `false`. Un archivo escrito por una persona que quiere decir "sí, acepto el pedido" con la palabra `yes` se convierte en el booleano `True`, y si ese valor lo compara con algo esperando texto, se rompe.

Mirá el tamaño del problema en una sola línea:

```python
print(yaml.safe_load("stock: 010"), yaml.safe_load("tel: +54 11 1234"))
```

`stock: 010` te da `8`, porque `010` es octal y `0o10` son ocho. En cambio `tel: +54 11 1234` se queda como texto, porque el `+` del principio lo saca de la lista de números. Dos líneas de un archivo de configuración, y una ya está mal. Un código postal, un número de teléfono con zéro adelante, un código de barra, un ID con padding, una versión: todos tienen el mismo problema, y ninguno de ellos se parece a un número en la vida real.

> **Dato clave:** en YAML, **entrecomillar un valor es una decisión semántica, no un detalle de estilo**. `codigo: 0123` y `codigo: '0123'` son dos datos distintos: uno es el número 83 y el otro es la cadena de cuatro caracteres. Si un valor tiene ceros a la izquierda, un punto, un `sí`/`no` o una `~`, entrecomillalo. Es el mismo cuidado que tenés al escribir un número de teléfono en un formulario: el cero a la izquierda no es decoración, es parte del dato.

Hay unaIronía útil en esto. `json` **no** tiene este problema: los valores de `json` siempre son texto con comillas, números, `true`/`false` o `null`, y nada más. No hay forma de escribir `codigo: 0123` en JSON porque tenés que poner las comillas. Es menos cómodo de escribir a mano, pero es **imposible de malinterpretar**. YAML gana en la escritura y pierde en la lectura automática, y esa es exactamente la transacción que estás haciendo cuando elegís uno u otro.

Y hay un último detalle de `yaml.dump` que te va a hacer ruido cuando veas un archivo YAML generado por un programa: **no pone las comillas donde las pondrías un humano**.

```python
print(yaml.dump({"codigo_postal": "0123", "ok": "yes"}, allow_unicode=True))
```

Ni comillas ni comillas: PyYAML sabía que `"0123"` es un `str` y que el `str` `"0123"` se puede escribir sin comillas sin ambigüedad, así que no las puso. El problema es que el archivo resultante, si un humano lo edita y saca o pone una coma, cambia de tipo sin que nadie lo note. `default_flow_style` y `explicit_start` son las opciones que controlan esto, y dejarlas como están es la decisión correcta para archivos que genera un programa.

> **Buenas prácticas:** si un archivo YAML lo **edita una persona**, revisá los valores que empiezan con cero, los que son `sí`/`no`/`on`/`off`, y los que parecen versiones. Si el archivo lo **genera un programa**, `yaml.dump` con `sort_keys=False` y `allow_unicode=True` es todo lo que necesitás. La library no hace el trabajo de pensar qué es un código postal y qué es un entero: eso es tuyo.

---

## 8. Multidocumento: cuando un archivo es varios

Hasta acá leímos un archivo y obtuvimos un dato. YAML tiene una capacidad que JSON no tiene: **un mismo archivo puede contener varios documentos separados por `---`**. Es lo que usan herramientas como Docker Compose y los sistemas de automatización, donde un archivo de texto largo con muchos bloques es más cómodo de versionar que un directorio lleno de archivos.

La forma de escribirla también tiene su propia función:

```python
Path("varios.yaml").write_text(
    "cliente: Acero SRL\nmonto: 10\n---\ncliente: Norte Textil\nmonto: 20\n",
    encoding="utf-8",
)
print(repr(yaml.dump_all([{"cliente": "Acero SRL", "monto": 10},
                          {"cliente": "Norte Textil", "monto": 20}])))
```

Y acá está la trampa, que es de las mejores que vas a ver en el libro. Si leés un archivo multidocumento con la función de siempre:

```python
try:
    with open("varios.yaml", encoding="utf-8") as f:
        print(yaml.safe_load(f))
except yaml.YAMLError as e:
    print(type(e).__name__, "|", str(e).splitlines()[0].strip())
```

`ComposerError: expected a single document in the stream`. El error está perfecto: te dice qué pasó, en qué archivo, en qué línea, y que encontró otro documento donde esperaba el fin. Lo que **no** dice es cuál de las dos funciones tenías que usar, y eso es información que un error de biblioteca debería darte. La función para un documento solo es `safe_load`; para varios es `safe_load_all`:

```python
with open("varios.yaml", encoding="utf-8") as f:
    documentos = yaml.safe_load_all(f)
    print(type(documentos).__name__)
    for documento in documentos:
        print(documento)
```

Y prestá atención a la primera línea de la salida: `generator`. `safe_load_all` **no devuelve una lista**, devuelve un generador — el mismo objeto que conocés del capítulo 10 y que te enseñó que se consume mientras iterás. Eso significa que el archivo se va leyendo a medida que avanzás, no entero en memoria primero. Para dos documentos no lo vas a notar; para un archivo de cien mil, la diferencia es la que separó `read()` de `for linea in f` en el capítulo 27.

Y ese detalle del generador no es un detalle: es la razón por la que el error de arriba es tan fácil de entender. `safe_load` lee **un** documento y se detiene. Cuando el archivo tiene un `---` de más, el parser llega al final del primer documento, seibly espera y encuentra que hay otro, y ahí se queja. No lo cargó dos veces: siguió al segundo y dijo "no sé qué hacer con esto".

> **Buenas prácticas:** cuando no estés seguro de la forma del archivo, no adivines la función: leé el archivo y preguntá. `if "\n---" in texto:` es una comprobación fea, pero es honesta; el patrón correcto es conocer la convención del formato que estás leyendo. `Docker Compose`, `Ansible` y los pipelines de integración continua usan todos `---` para separar documentos, y por eso `safe_load_all` es la función que vas a necesitar el día que toques cualquiera de ellos.

---

## 9. pickle: el idioma propio de Python

Cambio de marcha. JSON y YAML son idiomas que hablan muchos lenguajes; la razón por la que existen es que **un archivo tiene que poder ser leído por algo que no es Python**. Un navegador, un servicio en Java, una app de teléfono. Por eso son tan rigurosos: solo pueden representar los tipos que todas las máquinas saben representar.

`pickle` no se importa de esa discusión. `pickle` habla **el idioma de Python**, y existe para una pregunta mucho más simple: *¿cómo hago que este objeto mío sobreviva a que termine el programa?* Un modelo entrenado, un árbol que tardó tres horas en armarse, un contador acumulado: cosas que son carísimas de recalcular.

La diferencia fundamental se ve en la primera línea de uso:

```python
import pickle
from datetime import datetime

datos = {
    "nombre": "Acero SRL",
    "alta": datetime(2024, 3, 15, 10, 30),
    "cursos": ["python", "sql"],
    "ubicacion": ("cordoba", 33, -70),
}

salmuera = pickle.dumps(datos)
print(type(salmuera).__name__, len(salmuera), "bytes")

vuelta = pickle.loads(salmuera)
print(vuelta["alta"], type(vuelta["alta"]).__name__)
print(vuelta["ubicacion"], type(vuelta["ubicacion"]).__name__)
```

Ahí está. Con `json`, un `datetime` dentro del `dict` te daba un `TypeError`. Con `pickle`, vuelve siendo un `datetime`. Y la `tuple` también: vuelve siendo `tuple`. Pickle no necesita que le digas cómo convertir nada, porque no convierte a un idioma universal: **escribe la estructura misma**, con el tipo embebido.

Y funciona con objetos que definiste vos, que es la mitad de la razón de ser:

```python
class Factura:
    def __init__(self, numero, monto):
        self.numero, self.monto = numero, monto

    def __repr__(self):
        return f"Factura({self.numero}, {self.monto})"

f = Factura(7, 99.5)
vuelta = pickle.loads(pickle.dumps(f))
print(type(vuelta).__name__, "|", vuelta)
```

El objeto vuelve siendo un `Factura`, con sus atributos, y `__repr__` funciona porque la clase estaba definida cuando lo cargaste. Ese es el detalle que hay que entender: **pickle no guarda el código de tu clase, guarda el nombre de tu clase**. Cuando lo cargás, Python busca ese nombre en el programa que está corriendo. Si lo encuentra, construye el objeto; si no, revienta.

```python
otro = pickle.loads(pickle.dumps(f))
otro.numero = 99
print(f.numero, "|", otro.numero)
```

El original quedó en `7`. Pickle no compartió memoria con el original: construyó uno nuevo, como hacía `json`. Es la misma regla de la sección 5, y acá importa todavía más, porque un objeto compartido por error entre dos partes del programa es un bug de los difíciles.

> **Dato clave:** `pickle` serializa **el estado**, no la definición. "Estado" son los valores de los atributos. "Definición" es el código de la clase: el `__init__`, los métodos, los valores por defecto. Pickle se lleva lo primero y deja lo segundo donde estaba, y por eso el otro programa tiene que tener la clase definida. Es la misma promesa del capítulo 18 con pydantic: los hints viajan, la clase tiene que estar. Un `.pkl` es un archivo **incompleto** por diseño: le falta el programa que lo va a leer.

Como el caso de uso real de pickle es "esto me costó carísimo", y la forma más común de carísimo es un modelo entrenado, el formato tiene una consecuencia práctica que conviene ver:

```python
from pathlib import Path

datos_gordos = [{"id": i, "nombre": f"registro-{i}", "valores": list(range(20))} for i in range(500)]

import json
Path("grande.json").write_text(json.dumps(datos_gordos, indent=2), encoding="utf-8")
Path("grande.pkl").write_bytes(pickle.dumps(datos_gordos))

for nombre in ("grande.json", "grande.pkl"):
    p = Path(nombre)
    print(f"{nombre:<12} {p.stat().st_size:>9} bytes")
```

Con `indent=2` el JSON se infla, pero incluso sin `indent` el `pickle` gana en la representación de estructuras puras de Python. Donde `pickle` brilla de verdad es con **tipos que JSON ni intenta**: una `datetime`, un `Decimal`, un `set`, un `frozenset`, un objeto con referencias circulares. Ahí no hay competencia, porque los otros formatos ni pueden expresar esas cosas. Y esa es la razón de su existencia, y también la razón de sus dos problemas, que son las dos secciones siguientes.

---

## 10. pickle por dentro: qué hay en esos bytes

Vale la pena mirar una vez qué es realmente un archivo de `pickle`, porque casi todos los libros te lo muestran como una caja negra. Abrimos el archivo con `pickletools`, que viene en la biblioteca estándar:

```python
import pickletools

bruto = pickle.dumps(datos)
for opcode, argumento, posicion in pickletools.genops(bruto):
    print(f"{posicion:>3}  {opcode.name:<18} {'' if argumento is None else repr(argumento)}")
```

La primera fila dice `PROTO 4`. Ese es el **número de versión del formato**, y siempre va primero, para que una versión de Python que no lo conozca pueda decir "no sé leer esto" en vez de inventarse algo. Es la diferencia con `json`, que no tiene versión: un archivo JSON de 2015 y uno de 2025 son indistinguibles porque el formato no cambió, y esa es justamente su fuerza.

Después viene el resto, y hay que leerlo con paciencia. Las filas que importan son:

- `EMPTY_DICT` y `SHORT_BINUNICODE 'nombre'`: empieza un diccionario, y la primera clave es un texto corto. Cada `SHORT_BINUNICODE` es un pedacito de tu dato, con su largo escrito adelante. No hay que adivinar dónde termina un texto: el archivo **te lo dice**. Esa es la diferencia estructural con JSON, donde un texto sin comillas de escapado es ambiguo.
- `STACK_GLOBAL` y `REDUCE`: acá está el corazón del asunto, y lo vamos a mirar de nuevo en la sección siguiente.
- `STOP`: fin del archivo.

Y el formato de protocolo no es un detalle de versión, es una decisión de tamaño. `pickle` puede escribir el mismo dato de cuatro maneras distintas:

```python
for protocolo in (0, 2, 4, pickle.HIGHEST_PROTOCOL):
    bruto_p = pickle.dumps(datos, protocol=protocolo)
    print(f"protocolo {protocolo}: {len(bruto_p):>4} bytes | primeros dos: {bruto_p[:2].hex()}")
```

El protocolo 0 es el original, de los tiempos en que `pickle` significaba "serializar para Perl", y escribe todo en **texto plano legible**: abrís el archivo con un editor y lo entendés. Por eso era el protocolo por defecto en los años 90, y por eso hoy el 4 es el que se usa, y por eso el 5 es el más reciente.

Y el detalle de los dos primeros bytes merece su comentario: **el protocolo 0 no tiene esos dos bytes**, y los protocolos 2 en adelante empiezan con `80` seguido del número. Ese primer byte `80` es un valor reservado, y la razón de que exista es un detalle de historia de la biblioteca: se reservó para poder distinguir "esto es un pickle de texto" de "esto es un pickle binario" sin ambigüedad.

> **Buenas prácticas:** usá siempre el protocolo por defecto y no te metas con esto salvo que tengas una razón medida. El caso en el que sí tenés razón es la **compatibilidad**: si tu archivo lo va a leer un Python más viejo que el tuyo, es posible que necesites un protocolo más bajo para que lo entienda. Pero verificá, no supongas: el protocolo 2 es el más antiguo que `pickle` soporta en casi todas las versiones. Lo que no se puede hacer es escribir con un protocolo alto y esperar que un Python viejo lo lea, porque el archivo no tiene forma de avisarle al lector de qué necesita.

---

## 11. El precio de `pickle`: ejecuta código

Todo lo de arriba es cierto, y hay un hecho que cancela la mitad de las ventajas. Y hay otro que cancela casi todas las demás.

Voy a mostrarte el mecanismo, no el peligro. `pickle` es un formato que, cuando lee, **ejecuta instrucciones**. No es una metáfora: el archivo contiene una secuencia de operaciones, y el lector las va haciendo una por una. La fila `STACK_GLOBAL` de la sección anterior significa "buscá esta función en este módulo", y la fila `REDUCE` significa "**llamala** con estos argumentos".

Se puede ver con un ejemplo que no hace daño:

```python
def saludar(texto):
    print("   *** se ejecutó código al cargar:", texto, "***")
    return {"ok": True}

class CargaUtil:
    def __reduce__(self):
        return (saludar, ("hola desde el archivo",))

with open("carga.pkl", "wb") as f:
    pickle.dump(CargaUtil(), f)

print("el archivo pesa", Path("carga.pkl").stat().st_size, "bytes")
print("ahora lo cargo, como haría cualquier programa:")
recuperado = pickle.loads(Path("carga.pkl").read_bytes())
print("lo que devolvió:", recuperado)
```

El `print` de adentro se ejecutó, aunque vos solo hayas llamado a `loads`. No hay ninguna magia: `__reduce__` devuelve una tupla con el nombre de una función y sus argumentos, y `pickle` reconstruye el objeto **llamando esa función**. El archivo que acabás de cargar no era un dato, era un **programa**.

Y prestá atención a lo que dice la documentación de `pickle`, en la primera línea: *"Serializing arbitrary Python objects is insecure*"*. No dice "puede fallar". Dice **inseguro**. La cadena de confianza no es un detalle operativo: es la diferencia entre un archivo y un programa.

La forma más común de que esto te toque es la más inocente: un archivo `.pkl` que alguien te pasó.

```python
comando = "echo hola > marca.txt"
mensaje = comando.encode("utf-8")

# estos son los bytes reales del ataque, escritos a mano, sin usar pickle.dumps
PROTO = b"\x80\x04"                          # protocolo 4
MODULO = b"\x8c\x02os"                       # texto corto: "os"
NOMBRE = b"\x8c\x06system"                   # texto corto: "system"
STACK_GLOBAL = b"\x93"                        # junta "os" y "system"
ARGUMENTO = b"\x8c" + bytes([len(mensaje)]) + mensaje
TUPLE1 = b"\x85"                              # un solo argumento
REDUCE = b"R"                                 # llama os.system(mensaje)
STOP = b"."

oscuro = PROTO + MODULO + NOMBRE + STACK_GLOBAL + ARGUMENTO + TUPLE1 + REDUCE + STOP

print("el archivo pesa", len(oscuro), "bytes")
pickletools.dis(oscuro)
```

Y ahora el que lo carga, que es el que no tiene idea de lo que viene:

```python
Path("marca.txt").unlink(missing_ok=True)
print("antes de cargar, existe:", Path("marca.txt").exists())
resultado = pickle.loads(oscuro)
print("despues de cargar, existe:", Path("marca.txt").exists())
print("y el archivo dice:", Path("marca.txt").read_text(encoding="utf-8").strip())

# antes de cargar, existe: False
# despues de cargar, existe: True
# y el archivo dice: hola
```

Los 41 bytes de arriba dicen, en texto plano, `os`, `system` y `echo hola > marca.txt`. `pickle` los lee, importa `os`, llama a `os.system` con ese argumento y lo ejecuta: el archivo `marca.txt` aparece en el disco. Vos solo llamaste a `loads`. Nada de esto requiere ser un hacker: requiere que hayas cargado un archivo que no era tuyo.

Y hay una versión más sigilosa del mismo ataque, que no necesita que hagas nada sospechoso. Un atacante puede **reemplazar la clase** que tu programa espera, manteniendo el mismo nombre y los mismos atributos:

```python
# clase legitima, en tu programa
class Factura:
    def __init__(self, numero, monto):
        self.numero, self.monto = numero, monto

# el atacante escribe un archivo con una clase IGUAL llamada Factura,
# pero cuyo __init__ hace otra cosa. Tu pickle.load la encuentra primero
# y construye la version del atacante.
```

Esto se llama **sustitución de clase** y es el motivo por el que la regla no es "cargá pickles de fuentes conocidas" sino "**no cargues pickles ajenos**". La defensa no es técnica, es de proceso: si el archivo no lo creó tu propio programa en esta misma máquina, no lo cargues.

> **Dato clave:** la serialización **nunca es cifrado**. `pickle` no protege nada: no tiene clave, ni firma, ni checksum. Un `pickle` de un número de tarjeta es un archivo de texto que cualquiera puede abrir y leer, y si el dato es binario, cualquier editor hex te lo muestra igual. Confundir "serializar" con "proteger" es el error más caro de esta lista, porque un archivo cifrado que creés seguro se puede leer con un comando. Si necesitás que nadie más lo vea, eso es cifrado, y es un capítulo completamente distinto.

Y si necesitás las dos cosas — que el dato no se vea **y** que no sea un archivo de texto plano —, el orden correcto es cifrar después de serializar, no al revés: `pickle` genera bytes, y esos bytes son los que van adentro del contenedor cifrado. Un `pickle` de un objeto que vos creaste para serializar, **nunca** de algo que vino de afuera. Ese es el patrón: adentro, tu `pickle`; afuera, tu cifrado.

Con esta sección ya tenés el criterio completo sobre `pickle`: es el único formato de esta lista que **conserva los tipos** y el único que puede **ejecutar código**. Las dos cosas son la misma capacidad, vista desde dos ángulos. `pickle` no es "inseguro a pesar de ser útil": es útil **porque** funciona como un intérprete de un mini-lenguaje, y eso es exactamente lo que lo vuelve peligroso.

---

## 12. El otro precio de `pickle`: el tiempo y las versiones

El problema más común de `pickle` en el mundo real no es de seguridad, es de mantenimiento. Y tiene un solo protagonista: **el archivo que guardaste hace seis meses con un programa que ya no existe**.

Recall el detalle de la sección 9: pickle guarda el **nombre** de la clase, no su código. El programa que lo lee tiene que tener la clase definida con ese nombre exacto. Si cambiaste el nombre, si moviste la clase a otro módulo, si la borraste, o si la convertiste en una función: el archivo se vuelve ilegible.

```python
# El archivo tiene guardada la clase "Factura".
# Si en el programa nuevo la renombraste a "FacturaV2", al cargar:
#     AttributeError: Can't get attribute 'Factura' on <module '__main__'>
```

Y el caso de `datetime` que vimos en la sección 9 funciona al revés de lo que uno esperaría: `pickle` sabe serializar un `datetime` sin que le digas nada, porque `datetime` es un tipo **de la biblioteca estándar** y pickle tiene una regla interna para reconstruirlos. Si esa clase interna cambia entre versiones —y cambia, `datetime` es un tipo en C— un archivo viejo puede fallar al cargarse.

El otro problema es la **incompatibilidad entre versiones de Python**, y es real. El número de protocolo va al principio del archivo justamente para esto: si guardás con el protocolo por defecto de Python 3.10 (que es el 4) y lo cargás en un Python más nuevo, funciona. Si lo cargás en uno más viejo, puede que no.

```python
print("protocolo por defecto en este Python:", pickle.DEFAULT_PROTOCOL)
print("el más alto que existe:", pickle.HIGHEST_PROTOCOL)
print("el número va al principio del archivo, para poder avisar que no se entiende")
```

`pickle` avisa con un error si el archivo usa un protocolo más nuevo del que soporta, y ahí sí estás a salvo. El problema no es que avise: el problema es que **el archivo sigue ahí**. Guardaste, cerraste el programa, y seis meses después el entorno cambió. La variable de la que dependés —el archivo de hoy compatible con el Python de mañana— ya no la controlás.

Y hay un tercer problema, más sutil: **pickle no guarda el tipo, guarda el valor, y algunos valores no sobreviven**. Un `open` sin cerrar, un socket, una conexión a una base: son objetos que `pickle` no puede significativamente serializar, y si lo intentás te da un error críptico. Un `threading.Lock` tampoco. Son objetos que tienen estado del sistema operativo adentro, y el sistema operativo no está en tu archivo.

> **Buenas prácticas:** dos reglas para `pickle`, y las dos son sobre el ciclo de vida del archivo. Primera: **guardá siempre la versión del protocolo junto con el archivo** —en el nombre (`modelo-v4.pkl`) o en un `README`— porque el archivo por sí solo no lo dice. Segunda: **nunca uses `pickle` para algo que otro programa o otra persona tiene que leer**. Para eso están `json` y `yaml`, que son texto, que se debuguean, que se versionan y que no ejecutan nada. `pickle` es para el caso de uso interno y privado: tu caché, tu sesión, tu modelo entrenado que solo tu programa va a cargar. Si la respuesta a "¿quién va a leer esto?" es "otro equipo, o el yo del futuro dentro de seis meses", el formato no es `pickle`.


---

## 13. shelve: la base clave-valor que viene con Python

Después de `json`, `yaml` y `pickle`, queda una herramienta de la biblioteca estándar que no aparece en ningún tutorial y que resuelve un problema muy concreto: guardar **muchos datos con nombre**, actualizados uno por uno.

La forma de pensar en un `shelve` es: un diccionario que **está en disco**. Le asignás una clave y un valor, y queda escrito. Cerrás el programa. Mañana lo abrís de nuevo y está todo. Como un `dict`, pero que sobrevive.

```python
import shelve

with shelve.open("agenda") as db:
    db["ana"] = {"tel": "11-1234", "alta": "2024-01-10"}
    db["beto"] = {"tel": "11-9999", "alta": "2024-02-20"}
    db["carla"] = {"tel": "11-5555", "alta": "2024-03-05"}

print("archivos creados:", sorted(p.name for p in Path(".").glob("agenda*")))
```

Fijate en la primera línea de la salida: `shelve` no crea un archivo, crea **tres**. `agenda.dat`, que tiene los datos, `agenda.dir`, que es el índice de las claves, y `agenda.bak`, que es un respaldo que escribe por las dudas. El prefijo que le pasaste a `shelve.open` no es un nombre de archivo sino un **prefijo**: `shelve` le agrega la extensión que necesita. Es un detalle de la interfaz, y un detalle que confunde la primera vez.

Y el bloque anterior tiene algo que vale la pena mirar: `with shelve.open("agenda") as db`. El `with` es **obligatorio**, y no por estilo. Un `shelve` abre los archivos de verdad y los deja abiertos; sin `with`, los datos de las escrituras más recientes pueden quedar sin guardar hasta que el archivo se cierre. Es el mismo `finally` del capítulo 6, aplicado a un objeto que parece un `dict` pero tiene archivos adentro. Un `dict` no necesita `with`. Esto sí.

Ahora la demostración de que es de verdad un diccionario persistente:

```python
with shelve.open("agenda") as db:
    print("claves:", sorted(db.keys()))
    print("ana:", db["ana"])
    print("borrando a beto...")
    del db["beto"]
    print("claves ahora:", sorted(db.keys()))

# el bloque anterior se cerró; esto es como si el programa terminara y arrancara de nuevo
with shelve.open("agenda") as db:
    print("al reabrir:", sorted(db.keys()))
    print("el borrado sobrevivió:", "beto" not in db)
```

La segunda apertura está en el mismo programa, pero para los efectos de `shelve` es **otro programa**: no hay memoria compartida, no hay variables, no hay contexto. Todo lo que hay que recordar es el nombre `"agenda"`. Esa es toda la promesa, y es una promesa útil: una caché, un progreso, un índice de lo que ya calculaste.

Y ahora, la parte que casi nadie menciona y que importa mucho: **detrás de `shelve` hay `pickle`**. No es una metáfora, es la implementación.

```python
print("agenda.dat empieza con:", Path("agenda.dat").read_bytes()[:12])
print("agenda.dir, el índice:", Path("agenda.dir").read_text(encoding="utf-8"))
```

`agenda.dat` empieza con `b'\x80\x04\x95'`, que es un `pickle` de protocolo 4, exactamente el mismo formato de la sección 10. Y `agenda.dir` es un archivo de texto con las claves y en qué posición del `.dat` está cada una. O sea: `shelve` es `pickle` **más un índice**, y por eso hereda todas las ventajas y todos los problemas de `pickle`. Los datos se cargan rápido, y cargar datos de un archivo que no es tuyo tiene el mismo problema de siempre.

> **Buenas prácticas:** usá `shelve` para **cachés locales y descartables**: resultados caros que podés volver a calcular, un índice que tardó una hora, un mapeo que no querés perder. No lo uses como base de datos: no tiene transacciones, no soporta consultas, no maneja concurrencia, y cargar todo en memoria para buscar una clave es exactamente lo contrario de lo que hace una base de datos. Para un `dict` que tiene que sobrevivir, guardá un solo `pickle` de una vez. Para muchos datos con nombre, y solo si son tuyos, `shelve` es cómodo.

---

## 14. En producción: un solo punto de entrada para todo

Juntemos lo que viene. En un programa real no elegís formato: lo detectás. Y la forma de detectarlo es la extensión del archivo, que para eso está. El programa que todo el mundo escribe termina siendo más o menos esto:

```python
import json
import pickle

import yaml

CARGADORES = {
    ".json": json.load,
    ".yaml": yaml.safe_load,
    ".yml": yaml.safe_load,
    ".pkl": pickle.load,
}

def cargar(ruta):
    """Carga un archivo de configuración o de datos según su extensión."""
    p = Path(ruta)
    cargador = CARGADORES.get(p.suffix.lower())
    if cargador is None:
        raise ValueError(f"formato no soportado: {p.suffix!r}")
    modo = "rb" if p.suffix == ".pkl" else "r"
    with open(p, modo, encoding=None if modo == "rb" else "utf-8") as f:
        return cargador(f)
```

Y funciona, y es exactamente lo que tenías en mente:

```python
Path("demo.json").write_text(json.dumps({"cliente": "Acero SRL", "monto": 99.5}),
                             encoding="utf-8")
Path("demo.yaml").write_text(yaml.dump({"cliente": "Acero SRL", "monto": 99.5},
                                       allow_unicode=True, sort_keys=False),
                             encoding="utf-8")
with open("demo.pkl", "wb") as f:
    pickle.dump({"cliente": "Acero SRL", "monto": 99.5}, f)

for nombre in ("demo.json", "demo.yaml", "demo.pkl"):
    print(nombre, "->", cargar(nombre))
```

Mirá lo que devuelve cada uno, porque es el resumen del capítulo en tres líneas: los dos primeros devuelven un `dict` con las claves que escribiste, y el `.pkl` **también** devuelve un `dict`. Pero si en vez de un `dict` hubiera guardado un `datetime` o un objeto tuyo, el `.pkl` lo devolvería con su tipo y los otros dos no podrían ni haberlo escrito. El formato que elegís determina qué sobrevive.

Y los dos errores que un programa de verdad tiene que manejar, que son los que se ven en la práctica:

```python
try:
    cargar("demo.txt")
except ValueError as e:
    print(type(e).__name__, "|", e)

try:
    cargar("no-existe.json")
except FileNotFoundError as e:
    print(type(e).__name__, "|", e)
```

El `ValueError` para la extensión desconocida y el `FileNotFoundError` que viene de la biblioteca estándar. Dos errores distintos para dos problemas distintos, y los dos salen del lugar correcto: el primero de tu código, el segundo de `open`. Es el mismo contrato que aprendiste en el capítulo 6: cada error dice qué pasó.

Repasemos las cuatro decisiones que hacen que esto sea un programa y no un script.

La primera, **`CARGADORES` como diccionario en vez de una cadena de `if`**. Es la tabla de la sección 2, usada como código: un diccionario de extensión a función. Agregar un formato es agregar una línea, no tocar la lógica. Es la versión chica de la idea que ya viste en el capítulo 17 con las funciones de primer orden: los datos como tabla, no la lógica repetida.

La segunda, **`p.suffix.lower()`**. Las extensiones en mayúscula existen (`CONFIG.JSON` en Windows es perfectamente válido) y si no normalizás, el programa falla con un `ValueError` en un archivo que el usuarioClearly tenía razón en nombrar. Es el mismo cuidado que poner `encoding` en todo `open`, del capítulo 27: **normalizá la entrada, una vez, en el borde**.

La tercera, **el modo de apertura según el formato**. `pickle` necesita `"rb"`, los otros necesitan `"r"` con `encoding`. Un solo `open` para todos no funciona, y la tentación es usar `"rb"` para todos ynah. No: el `encoding` explícito es una regla del capítulo 27 y no se negocia, aunque el `pickle` se lo lleve.

La cuarta, **`ValueError` para lo que el programa no sabe hacer**. No un `print` y seguir. Si a tu función de carga le mandan un `.docx`, tiene que decir "no sé leer esto" y dejar que quien la llamó decida, en vez de devolver `None` y dejar que el error aparezca ocho líneas más abajo, con la forma de un `AttributeError` sobre `None`. Un programa real **informa y deja decidir**, y esa es exactamente la lección del capítulo 27, aplicada a un problema de otra clase.

> **Importante:** si vas a ofrecer esta función a alguien más, agregale dos cosas que este ejemplo no tiene y que el capítulo 27 ya te enseñó: validar que la ruta exista **antes** de abrir (o capturar el `FileNotFoundError` y re-lanzarlo con un mensaje propio) y devolver los **errores** en una lista en vez de imprimirlos. Un cargador de archivos es el punto de entrada ideal para una ataque de tipo *path traversal* —`../../etc/passwd`— así que en un programa que no es un ejercicio, normalizá la ruta con `Path(ruta).resolve()` y verificá que esté donde tiene que estar. Y si el archivo puede venir de otro lado, `pickle` no se carga nunca: es un formato que ejecuta código, y un cargador de archivos que acepta `.pkl` de origen desconocido es una puerta abierta.

---

## 15. Resumen y conceptos clave

Este capítulo tomó un objeto de Python y lo puso a viajar, y el viaje resultó tener un costo medible en dos cosas: el **tipo** y la **seguridad**. Todo lo demás son detalles de formato.

**El tipo.** Un archivo de texto es una secuencia de caracteres, así que serializar es, siempre, traducir a un idioma que solo sabe de caracteres. `json` e `YAML` saben seis tipos —`dict`, `list`, `str`, números, booleanos, `None`— y el resto tiene que viajar como texto que después alguien tiene que volver a interpretar. `pickle` no traduce: **escribe la estructura con el tipo embebido**, y por eso conserva `datetime`, `tuple`, `set` y clases propias sin que le pidas nada. El viaje de vuelta siempre crea objetos nuevos: `vuelta == original` es `True` pero `vuelta is original` es `False`, y las colecciones internas tampoco se comparten. Y hay pérdidas silenciosas que conviene reconocer: la `tuple` se vuelve `list` en `json` sin avisar, y en `YAML` un `0123` se vuelve el entero `83` porque el cero inicial significa "esto es octal". Por eso `default=` en `json` existe, y por eso las comillas en `YAML` son una decisión semántica.

**La seguridad.** `json` y `YAML` (con `safe_load`) solo construyen datos. `pickle` construye lo que el archivo le pida, porque su formato es una secuencia de instrucciones que el lector ejecuta: `STACK_GLOBAL` busca una función, `REDUCE` la llama. Por eso un `pickle` ajeno no es un archivo con datos, es un programa, y la regla no es "de fuentes conocidas" sino **"nunca de otro programa"**. Y la serialización nunca es cifrado: no tiene clave ni firma. Si necesitás las dos cosas, el orden es serializar y cifrar después.

**La forma.** Los dos formatos de texto se detectan por extensión y se cargan con una tabla. Los binarios se detectan igual, pero se abren en modo `"rb"`. `shelve` es el caso intermedio: parece un `dict`, escribe en disco, y por debajo es `pickle` con un índice al lado. Su `with` no es estilo, es la garantía de que lo que escribiste se guardó.

Las cuatro herramientas, una fila cada una, con las tres preguntas que importan:

| Formato | Es texto | Conserva el tipo | Ejecuta código al cargar | Se elige cuando |
|---|---|---|---|---|
| `json` | sí | no: la `tuple` se vuelve `list` | no | lo lee otro programa, o lo manda otro lenguaje |
| `YAML` | sí | a medias: adivina, y a veces mal | no, con `safe_load` | lo edita una persona |
| `pickle` | no | sí: `datetime`, `tuple`, `set`, clases | **sí, y esto no es un detalle** | lo carga tu propio programa, nunca otro |
| `shelve` | no | sí, es `pickle` con un índice al lado | **sí** | una caché local con claves y valores |

Leí la fila de la derecha a la izquierda: la pregunta no es "cuál es mejor", sino "quién va a leer esto". Si la respuesta es "otro programa que no es el mío", la fila es `json`. Si es "una persona", la fila es `YAML`. Y si la respuesta es "nadie, solo mi programa, y necesito el tipo exacto", entonces `pickle` — y solo entonces.

Repasá el checklist antes de seguir:

- [ ] Serializar es traducir a texto. La pregunta es **qué tipo pierdo** y **quién lee el archivo**.
- [ ] `json` para interoperabilidad, `YAML` para que un humano lea y edite, `pickle` para guardar **tu propio** objeto sin perder el tipo.
- [ ] `dumps`/`loads` con cadenas, `dump`/`load` con archivos ya abiertos. La `s` es de *string*.
- [ ] `indent` para que un humano lea; `ensure_ascii=False` para que los acentos no ocupen seis caracteres.
- [ ] `json` con `date`, `Decimal` o una clase tuya necesita `default=`. **`default=str` es un bug silencioso**, no una solución.
- [ ] El viaje de vuelta crea objetos nuevos: `==` es `True`, `is` es `False`, y las colecciones internas tampoco se comparten.
- [ ] La `tuple` se vuelve `list` al pasar por `json`, sin error. Si te importa, convertila vos al volver.
- [ ] En `YAML`: solo espacios, nunca tabs. `safe_load` siempre, nunca `yaml.load(texto)`.
- [ ] En `YAML`, entrecomillá lo que tiene ceros a la izquierda, `sí`/`no`/`on`/`off`, o un punto que es una versión. `0123` se lee como octal.
- [ ] Multidocumento: `safe_load_all` devuelve un **generador**, y solo sirve si el archivo tiene `---`.
- [ ] `pickle` no es cifrado y **ejecuta código al cargar**. Nunca cargues un `.pkl` que no haya creado tu programa.
- [ ] `pickle` guarda el **nombre** de la clase, no su código. Si la renombrás, el archivo viejo se rompe con `AttributeError`.
- [ ] El protocolo va al principio del archivo. Guardá la versión con el archivo, porque el archivo solo no lo dice.
- [ ] `shelve` es `dict` en disco, son **tres archivos**, `with` obligatorio, y por debajo es `pickle`. Para cachés locales, no para datos compartidos.
- [ ] Un cargador de archivos: tabla de extensiones, `suffix.lower()`, modo según el formato, y `ValueError` para lo que no sabe hacer.

Y la pregunta con la que cerramos, que es la que te llevás al capítulo 29: **¿esto lo va a leer otro programa, o una persona?** Si es un programa, `json`. Si es una persona, `YAML` — y entonces el archivo no es un dato, es una **interfaz**, y cada coma mal puesta es un error de esa persona. El capítulo 29 es exactamente sobre archivos de configuración: el lugar donde esa pregunta se responde todos los días.

---

## 16. Ejercicios

1. **La matriz, de verdad.** Escribí una función `puede_viajar(objeto, profundo=False)` que recorra un objeto de Python (un `dict`, una `list`, o cualquier cosa que contenga otras) y devuelva `True` si **todo** lo que encuentra adentro es un tipo que `json` sabe serializar sin `default`, y `False` si hay algo que no. Después usala con un `dict` bien armado y con uno que tenga un `date` adentro, y comprobá que la segunda devuelve `False`. El parametro `profundo` es para que entiendas la diferencia: con `profundo=False` solo mirá el objeto de primer nivel.

2. **La trampa del octal.** Escribí una función `cargar_config(ruta)` que use `yaml.safe_load` y que, si el archivo tiene una clave con un valor que empezó con `0` seguido de dígitos (un código postal, un ID con padding), te avise. Ojo: el `0` ya se perdió al cargar, así que tenés que mirar el **texto** del archivo, no el `dict` que volvió. Mostrá el caso que falla sin comillas y el que funciona con ellas.

3. **Un serializador propio.** Escribí una función `serializar(objeto)` que convierta un `dict` de Python a una cadena JSON y, en el camino, convierta los `date` a texto ISO y las tuplas a listas. Usá `default=`. Después probá que podés ir y volver sin perder información **de los tipos que decidiste perder a propósito**, y escribí en un comentario qué perdiste y por qué.

4. **El registro de archivos.** Escribí un programa que, para cada archivo de una carpeta, guarde en un `shelve` un registro con la ruta (como clave), el tamaño en bytes, y la fecha de modificación. Después mostrá los tres archivos más grandes. Después de cerrarlo y volverlo a abrir, verificá que sigue todo. Ojo con el `with`.

5. **El pickle y las versiones.** Escribí una función `guardar(objeto, ruta)` y otra `cargar(ruta)` que usen `pickle` con archivos. Después **cambiá el nombre de la clase** que estás guardando (o borrala) y tratá de cargar un archivo que habías guardado antes: mostrá el `AttributeError` real y explicá en un comentario por qué pasó. Esta es la sección 12, hecha con las manos.

6. **Un cargador con criterio.** Escribí una función `cargar(ruta)` que acepte `.json`, `.yaml` y `.pkl`, que lance `ValueError` si la extensión no está en la lista, y que **rechace `.pkl`** con un mensaje que explique que no es seguro, salvo que le pases `permitir_pickle=True`. Después usala con los tres formatos y mostrá que el mismo dato vuelve como `dict` en los dos primeros y como el objeto original en el tercero.

Y con esto la Parte IX tiene su primer capítulo y su segundo. El 27 te dio los archivos, el 28 te dio los formatos. El 29 va a ser el lugar donde todo eso se junta: configuración que se lee al arrancar un programa, con capas de prioridad entre variables de entorno, archivos y valores por defecto, y un contrato de pydantic que verifique que lo que llegó esté bien. Es el cierre de la parte, y es el más de fondo de los tres.


---

## 17. Soluciones

Las seis soluciones siguientes están escritas para que puedas ejecutarlas tal cual están. Cada una empieza con lo que ya importaste en el capítulo, y los `print` comentados muestran **exactamente** lo que sale en Python 3.10.

### 1. La matriz, de verdad

```python
TIPOS_JSON = (dict, list, str, int, float, bool, type(None))

def puede_viajar(objeto, profundo=False):
    if not isinstance(objeto, TIPOS_JSON):
        return False
    if not profundo:
        return True
    if isinstance(objeto, dict):
        return all(isinstance(k, str) and puede_viajar(v, True) for k, v in objeto.items())
    if isinstance(objeto, list):
        return all(puede_viajar(v, True) for v in objeto)
    return True

limpio = {"cliente": "Acero SRL", "monto": 99.5, "tags": ["a", "b"]}
con_fecha = {"cliente": "Acero SRL", "alta": date(2024, 3, 15)}

print(puede_viajar(limpio), puede_viajar(con_fecha))
# True True

print(puede_viajar(con_fecha, profundo=True))
# False

print(puede_viajar({"a": {"b": {1, 2}}}, profundo=True))
# False
```

La primera línea es la parte que sorprende y vale la pena mirar dos veces: `True True`. Los dos diccionarios **sí pueden viajar** en el nivel superior, porque un `dict` es un tipo que `json` maneja. La fecha está adentro, pero a un nivel de profundidad. Recién con `profundo=True` la encuentra. Ese es el punto del parámetro: la diferencia entre "es serializable" y "todo lo que contiene es serializable", que son dos preguntas distintas y de las cuales la segunda es la que importa.

Y el `set` de la última línea falla por el motivo de siempre: `{1, 2}` no es un tipo que `json` sepa escribir, y ningún `default` lo arregla, porque el `set` **no tiene orden**. Ahí el error no es de formato, es de información: un `set` de dos elementos no se puede escribir en JSON sin decidir sin qué criterio va ordenado.

### 2. La trampa del octal

```python
import re

PATRON_OCTAL = re.compile(r"^\s*[A-Za-z_][\w]*\s*:\s*0[0-7]+(\s|$)")

def cargar_config(ruta):
    texto = Path(ruta).read_text(encoding="utf-8")
    datos = yaml.safe_load(texto)
    crudos = [m.group(0).strip() for m in PATRON_OCTAL.finditer(texto)]
    if crudos:
        raise ValueError(f"posible octal, entrecomillá: {crudos}")
    return datos

Path("cfg_malo.yaml").write_text("codigo_postal: 0123\n", encoding="utf-8")
Path("cfg_bueno.yaml").write_text("codigo_postal: '0123'\n", encoding="utf-8")

for nombre in ("cfg_malo.yaml", "cfg_bueno.yaml"):
    try:
        print(nombre, "->", cargar_config(nombre))
    except ValueError as e:
        print(nombre, "->", type(e).__name__, "|", e)

# cfg_malo.yaml -> ValueError | posible octal, entrecomillá: ['codigo_postal: 0123']
# cfg_bueno.yaml -> {'codigo_postal': '0123'}

print("el valor real del malo:", yaml.safe_load("codigo_postal: 0123"))
# el valor real del malo: {'codigo_postal': 83}
```

Lo importante de esta solución es una limitación y el rodeo que resuelve. El `0` que se comió la conversión octal **ya no está en el `dict`**: cuando `cargar_config` recibe el dato, llegó como `83`, un entero como cualquier otro, y un entero `83` es perfectamente válido. Ningún análisis del resultado puede detectar el problema. Por eso el chequeo tiene que hacerse sobre `texto`, que es lo único que conserva lo que la persona escribió. Esa es una clase entera de errores: **la transformación destruye la evidencia**.

La última línea existe para que veas el daño antes del aviso, y para que el `raise` no parezca una paranoia. `83` a simple vista parece un número de puerta o un código postal mal leído, y en un archivo de producción nadie lo nota.

### 3. Un serializador propio

```python
def _a_json(objeto):
    if isinstance(objeto, date):
        return objeto.isoformat()
    if isinstance(objeto, tuple):
        return list(objeto)
    raise TypeError(f"no se: {type(objeto).__name__}")

def serializar(objeto):
    return json.dumps(objeto, default=_a_json, ensure_ascii=False)

original = {"alta": date(2024, 3, 15), "ubicacion": ("cordoba", 33)}
texto = serializar(original)
print(texto)
# {"alta": "2024-03-15", "ubicacion": ["cordoba", 33]}

vuelta = json.loads(texto)
print(vuelta, type(vuelta["ubicacion"]).__name__)
# {'alta': '2024-03-15', 'ubicacion': ['cordoba', 33]} list
```

`_a_json` es el `default` con forma de función, y `raise TypeError` en la última línea es lo que hace que el sistema sea útil: si mañana metés un `Decimal` y no lo contemplaste, el error lo dice, en lugar de guardarlo con la `repr` y descubrirlo tres meses después. Es exactamente lo contrario de `default=str`, que nunca falla y por eso nunca te avisa.

Lo que perdiste a propósito, para dejarlo escrito como pedía el enunciado: **la tupla se volvió lista**. La fecha sobrevive como texto ISO, que es reversible con `date.fromisoformat`, pero la tupla no se puede recuperar: de `"ubicacion": ["cordoba", 33]` no hay forma de saber si en el original era una tupla o una lista. Ese dato se perdió en la frontera y no hay ningún formato de texto universal que lo conserve sin agregar una marca explícita. Que es, otra vez, la razón de que `pickle` exista.

### 4. El registro de archivos

```python
import shelve

CARPETA = Path("carpeta_prueba")
CARPETA.mkdir(exist_ok=True)
for nombre, tamano in (("a.txt", 1200), ("b.bin", 3400), ("c.log", 700), ("d.dat", 9000)):
    (CARPETA / nombre).write_bytes(b"x" * tamano)

with shelve.open("registro") as db:
    for p in sorted(CARPETA.iterdir()):
        st = p.stat()
        db[p.name] = {"bytes": st.st_size, "mtime": st.st_mtime}

with shelve.open("registro") as db:
    print("claves:", sorted(db.keys()))
    for nombre, info in sorted(db.items(), key=lambda kv: -kv[1]["bytes"])[:3]:
        print(nombre, info["bytes"], "bytes")

# claves: ['a.txt', 'b.bin', 'c.log', 'd.dat']
# d.dat 9000 bytes
# b.bin 3400 bytes
# a.txt 1200 bytes

with shelve.open("registro") as db:
    print("reabre y sigue igual:", len(db), "entradas")
# reabre y sigue igual: 4 entradas
```

Los tres bloques con `with` son tres "apagones y encendidos" del programa, y esa es la demostración. El último no comparte nada con el primero: no hay variables, no hay memoria, no hay un objeto `db` que sobreviva. Lo único que viaja es el nombre `"registro"`, y si ese nombre cambia, se pierden todos los datos sin avisar. Es el mismo contrato del `open` del capítulo 27 aplicado a un `dict` que parece no necesitarlo.

El `[:3]` del final es una expresión con la que ya te tropezaste: una lista ordenada por tamaño descendente, recortada a los tres primeros. Y `sorted(db.items(), key=lambda kv: -kv[1]["bytes"])` es `sorted` con una clave propia, que es la forma de que el orden no dependa del orden en que `shelve` te devuelva las claves. `shelve` no garantiza ningún orden, así que sin esa `lambda` el resultado sería distinto en cada ejecución.

### 5. El pickle y las versiones

```python
import pickletools

class Factura:
    def __init__(self, numero, monto):
        self.numero, self.monto = numero, monto

    def __repr__(self):
        return f"Factura({self.numero}, {self.monto})"

def guardar(objeto, ruta):
    with open(ruta, "wb") as f:
        pickle.dump(objeto, f)

def cargar(ruta):
    with open(ruta, "rb") as f:
        return pickle.load(f)

guardar(Factura(7, 99.5), "factura.pkl")
print("vuelve bien:", cargar("factura.pkl"))
# vuelve bien: Factura(7, 99.5)

print("primeros bytes:", Path("factura.pkl").read_bytes()[:2].hex())
# primeros bytes: 8004

del Factura
try:
    cargar("factura.pkl")
except AttributeError as e:
    print(type(e).__name__, "|", str(e).split(" from ")[0])

# AttributeError | Can't get attribute 'Factura' on <module '__main__'
```

`8004` son los dos primeros bytes en hexadecimal, y son el protocolo 4 escrito little-endian, tal como lo anunciaba la sección 10. El archivo no dice "soy un pickle versión 4" en texto: lo dice **en los dos primeros bytes**, y por eso `pickle` puede rechazarlo con un mensaje claro si el lector es viejo, en lugar de producir basura.

El `del Factura` es el experimento completo de la sección 12, y la línea que lo hace posible es `str(e).split(" from ")[0]`. `pickle` construye un mensaje de error a medio camino entre "`Can't get attribute 'Factura' on <module '__main__'>`" y la ruta del archivo donde lo buscas, que en un entorno real es un `C:\...` de doscientas caracteres. Recortarlo es lo que hace que el error se pueda **pegar en un issue** y se entienda. Los mensajes de error también son parte del programa: si son ilegibles, no cumplen su función.

### 6. Un cargador con criterio

```python
CARGADORES = {".json": json.load, ".yaml": yaml.safe_load, ".yml": yaml.safe_load}

def cargar(ruta, permitir_pickle=False):
    p = Path(ruta)
    sufijo = p.suffix.lower()
    if sufijo == ".pkl":
        if not permitir_pickle:
            raise ValueError("'.pkl' ejecuta codigo al cargar: no se carga sin permiso explicito")
        with open(p, "rb") as f:
            return pickle.load(f)
    cargador = CARGADORES.get(sufijo)
    if cargador is None:
        raise ValueError(f"formato no soportado: {sufijo!r}")
    with open(p, encoding="utf-8") as f:
        return cargador(f)

Path("d.json").write_text(json.dumps({"cliente": "Acero SRL"}), encoding="utf-8")
Path("d.yaml").write_text(yaml.dump({"cliente": "Acero SRL"}, allow_unicode=True), encoding="utf-8")
with open("d.pkl", "wb") as f:
    pickle.dump({"alta": datetime(2024, 3, 15), "tags": {"a", "b"}}, f)

for nombre in ("d.json", "d.yaml", "d.pkl"):
    try:
        dato = cargar(nombre)
        print(nombre, "->", type(dato).__name__, dato)
    except ValueError as e:
        print(nombre, "->", type(e).__name__, "|", e)

# d.json -> dict {'cliente': 'Acero SRL'}
# d.yaml -> dict {'cliente': 'Acero SRL'}
# d.pkl -> ValueError | '.pkl' ejecuta codigo al cargar: no se carga sin permiso explicito

con_permiso = cargar("d.pkl", permitir_pickle=True)
print("tipo de la fecha:", type(con_permiso["alta"]).__name__)
print("tipo de las etiquetas:", type(con_permiso["tags"]).__name__, sorted(con_permiso["tags"]))

# tipo de la fecha: datetime
# tipo de las etiquetas: set ['a', 'b']

try:
    cargar("d.txt")
except ValueError as e:
    print(type(e).__name__, "|", e)

# ValueError | formato no soportado: '.txt'
```

La línea con permiso es la que justifica todo el ejercicio. El mismo archivo que dos formatos de texto no pueden ni representar —un `set` y un `datetime` — vuelve **con su tipo intacto**: un `datetime` de verdad y un `set` de verdad, no dos textos que hay que volver a interpretar. Y prestá atención al `sorted` del `print`: sin él, el `set` mostraría sus elementos en un orden distinto en cada ejecución, y un ejemplo cuya salida cambia es un ejemplo que nadie puede verificar. `json` habría lanzado un `TypeError` al escribirlo y `YAML` te habría devuelto un texto que tendrías que volver a interpretar a mano. Eso es `pickle` pagándose a sí mismo: comfort y riesgo, del mismo lugar.

Y notá que el `ValueError` del `.pkl` sale **antes** de abrir el archivo. La función aplica la regla del límite de confianza: la regla del capítulo 6, convertida en una decisión de diseño de la API.
