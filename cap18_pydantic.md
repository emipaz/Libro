# Capítulo 18 — Los hints que sí validan: pydantic

El capítulo 9 te dejó una cuenta pinchada: los hints **documentan, no controlan**. El capítulo 13 la guardó en el espejo con las dataclasses, y el 14 le puso el dedo en la llaga — `@dataclass` *no protege*: un campo público sigue aceptando cualquier cosa. Este es el capítulo donde esa cuenta se cierra. Vas a conocer a **pydantic**, la librería que toma la clase con anotaciones que ya sabés escribir y la convierte en un **contrato**: cuando un dato entra —venga de un formulario web, de un `json.load`, de una fila de una base de datos—, pydantic lo examina campo por campo; si pasa, queda convertido al tipo que declaraste; si no, el programa se detiene con un error que te dice *exactamente* qué pasó, dónde y por qué.

Y no es un capricho de aficionados: en los proyectos reales pydantic es el pasaporte de los datos. Cada vez que una API moderna —FastAPI, que conocerás en el capítulo 39— recibe o responde, delante hay un modelo de pydantic. Ya te cruzaste con su motor en el capítulo 1 (el núcleo en Rust, pydantic-core) y se te marcó `[OK]` en la verificación del ecosistema; ahora le toca el papel de protagonista. El recorrido tiene tres movimientos: **declarar** el contrato, **validar** contra él y **serializar** hacia el mundo exterior (archivos, JSON, APIs). Tres pasos que te van a dar la llave de un patrón que verás en todos lados.

---

## 1. El plano que no valida

Imaginate la escena: un cliente te manda por un formulario web un dict con los datos de un producto. Nada del otro mundo... salvo que **llega en texto**: `precio` vino como `"150.5"`, quizá con espacios, quizá con un `"."` donde debería haber una coma. Los hints de la clase no te ayudan acá: son un *plano* (anotaciones en frío) que el IDE lee, pero el runtime ignora. Alguien tiene que vigilar de verdad.

El reflejo natural es escribir el guardián a mano:

```python
def validar_producto_mano(datos):
    """Guardián casero: a cada campo hay que vigilarlo a mano."""
    if not isinstance(datos.get("nombre"), str) or not datos["nombre"].strip():
        raise ValueError("nombre inválido")
    try:
        precio = float(datos["precio"])
    except (TypeError, ValueError):
        raise ValueError("precio no numérico") from None
    if precio <= 0:
        raise ValueError("precio positivo")
    return {"nombre": datos["nombre"], "precio": precio}

print(validar_producto_mano({"nombre": "yerba", "precio": "150.5"}))
# {'nombre': 'yerba', 'precio': 150.5}

try:
    validar_producto_mano({"nombre": "yerba", "precio": "sin precio"})
except ValueError as e:
    print("ValueError:", e)
# ValueError: precio no numérico
```

> **Para curiosear:** la cola `from None` al final del `raise` controla el **encadenado de excepciones**. Cuando lanzás un error mientras estás manejando otro —como acá, que rebotás tu `ValueError` desde dentro del `except`—, Python por defecto te muestra los dos en el *traceback* (el resumen de errores del capítulo 6): el de `float()` y el tuyo, en dos pisos. Con `from None` le pedís que **silencie el original** y muestre solo tu mensaje. Es un pulido de presentación: sin él el ejemplo funcionaría igual, solo que el error llegaría acompañado del detalle interno de la conversión. En el capítulo 6 el `raise` era simple; lo vas a ver con esta cola en código profesional, siempre con la misma idea: el detalle no aporta, mostrá el mensaje limpio.

El guardián funciona, pero miralo con ojos de ingeniería: **cada regla es un `if` escrito a mano**, y cada campo nuevo suma otro. Precio positivo; longitud mínima del nombre; código con formato; email con arroba... en dos modelos la función ya no entra en la pantalla. Y si el dato viene anidado —un pedido con productos adentro—, la cosa explota en tamaño y en tentación de bugs.

La jugada de pydantic es **invertir el proceso**: no escribís los controles, **declarás el contrato** y el guardián nace solo. La materia prima del contrato ya la tenés: las anotaciones de tipos que aprendiste en el capítulo 9, escritas sobre una clase como las del capítulo 13. pydantic las lee **en tiempo de ejecución** —usando exactamente la introspección del capítulo 12— y las convierte en reglas vivas.

> **Dato clave:** pydantic sube a los hints de "decoración para el IDE" a "regla de verdad". La clase dice `precio: float`, y pydantic garantiza que `precio` sea un `float` o que el programa lo grite. Por eso lo del capítulo 9 era "el terreno de herramientas como pydantic que verás después de la POO": recién ahora tenés todo el vocabulario para entenderlas.

---

## 2. `BaseModel`: los datos con contrato

La receta para armar el contrato no podría ser más corta: una clase que hereda de **`BaseModel`**. Nada más — las anotaciones son las cláusulas.

```python
from pydantic import BaseModel, ValidationError

class Producto(BaseModel):
    nombre: str
    precio: float
```

Eso es todo el código. `Producto` ahora escudriña todo lo que se le pasa. Fijate el detalle que cambia la película: le das `precio="150.5"` (texto, como llega del formulario) y pydantic hace la **coerción** — convierte lo convertible al tipo declarado, como si dijera "bueno, esto es un precio, lo paso a número":

```python
liquidacion = Producto(nombre="yerba", precio="150.5")
print(type(liquidacion.precio).__name__, liquidacion.precio)
# float 150.5

print(liquidacion.precio * 2)
# 301.0
```

`type(...).__name__` confiesa la magia: adentro quedó un `float` de verdad, no un texto que parece número. Por eso la multiplicación funciona sin chistar. Esta es la filosofía completa en una línea: **convertir también es validar** — si pydantic logra convertirlo, el dato puede vivir como corresponde; si no, es la señal de que algo raro llegó.

Y cuando un dato es *irrecuperable*, pydantic no se calla. Levanta la excepción **`ValidationError`** con un informe forense:

```python
try:
    Producto(nombre="yerba", precio="sin precio")
except ValidationError as e:
    print(e)
# 1 validation error for Producto
# precio
#   Input should be a valid number, unable to parse string as a number [type=float_parsing, input_value='sin precio', input_type=str]
#     For further information visit https://errors.pydantic.dev/2.9/v/float_parsing

try:
    Producto(nombre="yerba")
except ValidationError as e:
    print(e)
# 1 validation error for Producto
# precio
#   Field required [type=missing, input_value={'nombre': 'yerba'}, input_type=dict]
#     For further information visit https://errors.pydantic.dev/2.9/v/missing
```

Leé el reporte como un expediente: el **modelo** (`Producto`), el **campo** que falló (`precio`), el **tipo de error** (`float_parsing`, `missing`) y el **valor que llegó** (`input_value`). El `[type=...]` es tu mapa: ahí el link te lleva a la explicación exacta de cada diagnosis, y en el programa podés recorrer `e.errors()` — una lista de dicts, uno por cada campo problemático. El segundo caso es oro puro: un campo sin valor por defecto es **obligatorio**, y si falta, `missing` te lo marca. Recién ahora la frase del capítulo 9 cobra sentido: los hints *sí* controlan, cuando alguien los hace valer.

Los datos de la vida real llegan como dict. La entrada del contrato por excelencia es **`model_validate`** — y fijate que el modelo atiende hasta los anidados que todavía no viste:

```python
class PedidoMini(BaseModel):
    cliente: str
    producto: Producto
    cantidad: int = 1
    urgente: bool = False

mini = PedidoMini.model_validate(
    {"cliente": "Ana", "producto": {"nombre": "mate", "precio": "40"}})
print(mini.producto.precio, mini.cantidad, mini.urgente)
# 40.0 1 False
```

`cantidad` y `urgente` no venían en el dict, y sin embargo existen: **un campo con valor por defecto es opcional**. El dict se convirtió en objetos anidados reales — `mini.producto` *es* un `Producto` validado, ya con `precio` como `float`.

> **Dato clave:** pydantic valida en la **frontera**: al construir (`Producto(...)`) o al cruzar la puerta (`model_validate(...)`). Lo que entra queda verificado; lo que sale después de la creación —`objeto.atributo = valor`— no se revalida por defecto. El contrato custodia la entrada, igual que los sets del capítulo 14 custodian la asignación: cada herramienta vigila su puerta.

---

## 3. Tipos que piensan solos

Hasta acá usaste los tipos del capítulo 9: `str`, `float`, `int`, `bool`. Pero el verdadero poder aparece cuando el contrato exige más que "que sea un número". El menú de pydantic está lleno de **tipos que piensan solos**: en lugar de que vos escribas la regla, el tipo la trae incorporada.

Volvés a encontrarte con el **`Enum`** del capítulo 17. Como la clase hereda de `str`, un texto se convierte solo en su miembro — y eso es exactamente la validación:

```python
from typing import Literal
from enum import Enum
from pydantic import Field

class EstadoPedido(str, Enum):
    PENDIENTE = "pendiente"
    EN_CAMINO = "en_camino"
    ENTREGADO = "entregado"
    DEVUELTO = "devuelto"

class Envio(BaseModel):
    estado: EstadoPedido
    tipo: Literal["retiro", "entrega", "punto"]

print(Envio(estado="en_camino", tipo="entrega").estado)
# EstadoPedido.EN_CAMINO

try:
    Envio(estado="perdido", tipo="entrega")
except ValidationError as e:
    print(e)
# 1 validation error for Envio
# estado
#   Input should be 'pendiente', 'en_camino', 'entregado' or 'devuelto' [type=enum, input_value='perdido', input_type=str]
#     For further information visit https://errors.pydantic.dev/2.9/v/enum
```

`estado="en_camino"` salió siendo `EstadoPedido.EN_CAMINO`: el dict entró en texto y el contrato entregó un miembro del Enum, listo para comparar con `==` y para guardarse. Y un valor que no existe en el menú —`"perdido"`— se rechaza con la lista honesta de los permitidos.

Al lado vive **`Literal[...]`**: acotar el campo a un puñado de valores escritos en el mismo lugar. Mirá la diferencia entre los dos en una tabla:

| `Enum` | `Literal[...]` |
|---|---|
| Miembro con nombre y `.value`; namespace reutilizable (capítulo 17) | Valores pelados, sin membresía |
| Se declara una vez y se usa en muchos modelos | Se escribe donde acota |
| Comparaciones, iteración, `estado.value` | Solo la restricción inline |

La regla práctica: `Enum` cuando el estado es un **concepto con vida propia** (lo vas a repetir, comparar, guardar); `Literal` cuando es un **menú puntual** del campo.

Ahora el kit de **`Field`**: las reglas numéricas y de texto en la declaración, sin funciones satélite:

```python
class Paquete(BaseModel):
    descripcion: str = Field(min_length=3, max_length=40)
    peso_kg: float = Field(gt=0, le=30)
    codigo: str = Field(pattern=r"^PKG-\d{4}$")

try:
    Paquete(descripcion="ab", peso_kg=45, codigo="kilo-1")
except ValidationError as e:
    print(e)
# 3 validation errors for Paquete
# descripcion
#   String should have at least 3 characters [type=string_too_short, input_value='ab', input_type=str]
#     For further information visit https://errors.pydantic.dev/2.9/v/string_too_short
# peso_kg
#   Input should be less than or equal to 30 [type=less_than_equal, input_value=45, input_type=int]
#     For further information visit https://errors.pydantic.dev/2.9/v/less_than_equal
# codigo
#   String should match pattern '^PKG-\d{4}$' [type=string_pattern_mismatch, input_value='kilo-1', input_type=str]
#     For further information visit https://errors.pydantic.dev/2.9/v/string_pattern_mismatch
```

`min_length`/`max_length` para textos, `gt` (mayor que, *greater than*) y `le` (menor o igual, *less or equal*) para números, y **`pattern`** — un *regex* (expresión regular) que exige el formato exacto del código: `PKG-` seguido de cuatro dígitos. Los tres disparos fallaron a la vez y el reporte lista los tres: pydantic **no abandona al primer error**, te entrega el inventario completo de lo que se rompió.

Te queda pendiente la familia de tipos especializados. El **opcional** es el mismo patrón que el capítulo 9: `X | None` con default `None`, para campos que pueden faltar legítimamente:

```python
class Domicilio(BaseModel):
    calle: str
    numero: int
    piso: str | None = None

print(Domicilio(calle="Av. Siempre Viva", numero=742).piso)
# None
```

Y los **validadores de formato**: `EmailStr` y `HttpUrl` saben reconocer lo que *parece* un mail o una URL:

```python
from pydantic import EmailStr, HttpUrl

class Contacto(BaseModel):
    email: EmailStr

class Sitio(BaseModel):
    url: HttpUrl

print(Contacto(email="ana@dominio.com.ar").email)
# ana@dominio.com.ar

print(Sitio(url="https://universidad.edu.ar").url)
# https://universidad.edu.ar/

try:
    Contacto(email="no-es-un-mail")
except ValidationError as e:
    print(e)
# 1 validation error for Contacto
# email
#   value is not a valid email address: An email address must have an @-sign. [type=value_error, input_value='no-es-un-mail', input_type=str]
```

`EmailStr` exige un paso previo: la dependencia extra `email-validator` se instala con `pip install "pydantic[email]"`. `HttpUrl`, por su parte, **normaliza**: notá que la salida agregó la barra final — porque valida la URL completa, no solo que "empiece con `http`". Ninguno de estos tipos es magia: cada uno trae enchufado su propio pequeño validador, igual que los que vas a escribir a mano en la próxima sección.

> **Buenas prácticas:** antes de escribir un control a mano, mirá el menú de tipos de pydantic. Cuando el problema ya tiene nombre propio (`EmailStr`, `HttpUrl`, un `Field` con `pattern`), conviene usarlo: viene testeado, con su mensaje de error y su documentación — ya no tenés que mantener vos la regla.

---

## 4. Validadores a medida

Los tipos vienen con reglas prefabricadas, pero la vida real siempre mete una regla que no estaba en el menú: "el código del paquete puede llegar con espacios", "la fecha viene en formato `2026-12-31`", "un paquete frágil debe declarar su valor". Para esas, pydantic te da dos herramientas: **`@field_validator`** (le da personalidad a un campo) y **`@model_validator`** (le da personalidad al conjunto).

El decorador del campo tiene dos *modos* (formas de actuar), y la distinción vale oro. Empezá con `mode="before"`: el validador atiende **antes de los controles**, como recepcionista que acomoda la materia prima antes de que el inspector la juzgue. Es el lugar para **normalizar** (ordenar el formato) y **parsear** (convertir de texto):

```python
from datetime import date
from pydantic import field_validator, model_validator

class Paquete(BaseModel):
    descripcion: str = Field(min_length=3, max_length=40)
    codigo: str = Field(pattern=r"^PKG-\d{4}$")
    fecha_envio: date | None = None

    @field_validator("codigo", mode="before")
    @classmethod
    def _normalizar_codigo(cls, v):
        return v.strip().upper() if isinstance(v, str) else v

    @field_validator("fecha_envio", mode="before")
    @classmethod
    def _parsear_fecha(cls, v):
        if isinstance(v, str):
            return date.fromisoformat(v)
        return v

q = Paquete(codigo="  pkg-0034 ", descripcion="Mate de calabaza", fecha_envio="2026-12-31")
print(q.codigo, "-", q.fecha_envio)
# PKG-0034 - 2026-12-31
```

`"  pkg-0034 "` (minúsculas, con espacios) llegó como `PKG-0034` y `"2026-12-31"` se convirtió en un `date` de verdad. ¿Y cómo sabe el validador cuál campo atiende cada uno? El decorador lo dice en la firma: `@field_validator("codigo", mode="before")`. El `@classmethod` es el pasaporte que aprendiste en el capítulo 14 — pydantic manda el campo, no un objeto.

Fijate el detalle sutil del `mode="before"`: el `pattern` del código corre **después** de que el recepcionista normalizó. Si hubiésemos puesto el validador en el modo por defecto ("después" de los controles), el `pattern` ya hubiera rechazado `"  pkg-0034 "` antes de que nadie lo limpiara. Por eso el orden es el alma de la elección.

> **Dato clave:** `mode="before"` **arregla el dato antes de los controles** (normalizar, parsear, unificar); el modo por defecto —podés escribirlo explícito como `mode="after"`— **juzga el dato ya validado** (rechazar, refinar, calcular). Primero la materia prima, después el producto terminado. Dependiendo de si tu regla es de limpieza o de censura, elegís la puerta de entrada.

El segundo decorador, **`@model_validator`**, ya no mira un campo: mira el **objeto entero**, con la vista de pájaro que permite las reglas cruzadas —"si A, entonces B". Cada una de las dos reglas siguientes va en un modelo chico, para que los errores se lean sin ruido:

```python
class Carga(BaseModel):
    fragil: bool = False
    valor_declarado: float = Field(default=0, ge=0)

    @model_validator(mode="after")
    def _seguro_fragil(self):
        if self.fragil and self.valor_declarado == 0:
            raise ValueError("un paquete frágil debe declarar un valor")
        return self

try:
    Carga(fragil=True)
except ValidationError as e:
    print(e)
# 1 validation error for Carga
#   Value error, un paquete frágil debe declarar un valor [type=value_error, input_value={'fragil': True}, input_type=dict]
#     For further information visit https://errors.pydantic.dev/2.9/v/value_error
```

La regla es una frase: *si es frágil, tiene que haber un valor declarado*. Cada campo por separado es irreprochable (True es un bool válido, 0 es un float válido), pero **el conjunto es un disparate** — y solo el validador del modelo puede verlo. Mirá la firma: recibe `self` (el objeto ya construido), y si todo está bien **devuelve `self`**, porque es el objeto final el que tiene que seguir el viaje. Levantás `ValueError` con tu propio mensaje, y pydantic lo viste de `ValidationError` con `[type=value_error]`.

```python
class Entrega(BaseModel):
    hora_desde: int = Field(ge=0, le=23)
    hora_hasta: int = Field(ge=1, le=24)

    @model_validator(mode="after")
    def _horario_coherente(self):
        if self.hora_hasta <= self.hora_desde:
            raise ValueError("hora_hasta debe ser posterior a hora_desde")
        return self

print(Entrega(hora_desde=9, hora_hasta=13).model_dump())
# {'hora_desde': 9, 'hora_hasta': 13}

try:
    Entrega(hora_desde=19, hora_hasta=18)
except ValidationError as e:
    print(e)
# 1 validation error for Entrega
#   Value error, hora_hasta debe ser posterior a hora_desde [type=value_error, input_value={'hora_desde': 19, 'hora_hasta': 18}, input_type=dict]
#     For further information visit https://errors.pydantic.dev/2.9/v/value_error
```

`19` a `18` es el clásico horario imposible: el validador del modelo lo detecta y hasta te muestra el dict de entrada completo en `input_value`. Esta regla cruzada, la de *comparar dos campos entre sí*, es la abuela de todas las reglas de negocio: fechas donde inicio <= fin, rangos válidos, tarifas coherentes. Vas a verla en los proyectos reales todo el tiempo.

> **Para curiosear:** pydantic v2 trae un tercer modo, `mode="wrap"`, que combina "antes" y "después" en un solo validador: recibe el dato crudo y una función `next` para llamar al siguiente paso. Y los validadores pueden **encadenarse** — pydantic los corre en orden de declaración. No lo vas a necesitar hoy; queda en el catálogo cuando te toque pulir reglas complejas.

---

## 5. La composición validada

Los datos reales no llegan chatos: un pedido *contiene* un cliente, y una lista de paquetes, cada uno con su fecha y su código. La buena noticia es que la composición que aprendiste en el capítulo 15 se aplica **adentro de los contratos**: un modelo puede tener como campo a otro modelo, y a listas de modelos. La validación baja en cascada.

```python
class Cliente(BaseModel):
    nombre: str = Field(min_length=1)
    email: EmailStr

class Pedido(BaseModel):
    folio: str = Field(pattern=r"^P-\d{3}$")
    cliente: Cliente
    paquetes: list[Paquete] = Field(min_length=1)
    estado: EstadoPedido = EstadoPedido.PENDIENTE
```

Un dict gigante, anidado hasta el infinito, vale como entrada. El `Paquete` de la sección anterior y el `Cliente` de acá se validan solos dentro del árbol:

```python
pedido = Pedido.model_validate({
    "folio": "P-042",
    "cliente": {"nombre": "Ana", "email": "ana@dominio.com.ar"},
    "paquetes": [
        {"codigo": "pkg-0034", "descripcion": "Mate de calabaza", "fecha_envio": "2026-12-31"},
        {"codigo": "pkg-0035", "descripcion": "Yerba"},
    ],
})
print(pedido.paquetes[0].codigo, "|", pedido.estado)
# PKG-0034 | EstadoPedido.PENDIENTE
```

El dict de entrada se convirtió en objetos. `pedido.cliente` *es* un `Cliente` validado, `pedido.paquetes` es una lista de `Paquete` (con sus códigos normalizados y fechas parseadas), y `pedido.estado` es un miembro del `Enum`. Nada quedó en texto; nada quedó sin control; el `list[Paquete]` con `min_length=1` garantiza, además, que ningún pedido viaje sin mercadería.

Y cuando algo entra mal en las profundidades del árbol, el expediente te da la **dirección completa**:

```python
try:
    Pedido.model_validate({
        "folio": "P-043",
        "cliente": {"nombre": "Bet", "email": "bet@dominio.com.ar"},
        "paquetes": [
            {"codigo": "pkg-0001", "descripcion": "Caja"},
            {"codigo": "kilo-1", "descripcion": "Botella"},
        ],
    })
except ValidationError as e:
    print(e)
# 1 validation error for Pedido
# paquetes.1.codigo
#   String should match pattern '^PKG-\d{4}$' [type=string_pattern_mismatch, input_value='KILO-1', input_type=str]
#     For further information visit https://errors.pydantic.dev/2.9/v/string_pattern_mismatch
```

`paquetes.1.codigo` es la ruta al problema dentro del árbol: listas por índice (`1`), modelos por nombre de campo (`codigo`). Y hay un detalle maestro enmascarado ahí: vos pasaste `"kilo-1"`, pero el reporte dice `input_value='KILO-1'`. Ese `KILO-1` en mayúsculas es la **huella del validador `before`**: el mode="before" de la sección 4 ya había limpiado el dato antes de que el `pattern` dijera que no. El error muestra el valor **en el momento en que falló** — y esa pista te confirma que el flujo fue "normalizar, luego controlar".

> **Dato clave:** la composición es recursiva hasta donde quieras: un modelo dentro de un modelo dentro de una lista. Cada nivel valida el suyo, y los `ValidationError` anidados se aplanan en rutas tipo `cliente.paquetes.1.codigo`. Es el mismo mecanismo que hace funcionar a los posters de las APIs reales: pedidos, usuarios, todo es árbol de contratos.

---

## 6. Serialización y puente

El contrato tiene dos puertas. Hasta acá viste la de la **entrada** (del caos hacia adentro: validar y convertir). Ahora la de la **salida** (de adentro hacia el mundo: **serializar**, volcar el objeto para que otro lo lea). Porque después de vivir su vida dentro del programa, los datos tienen que irse: a un archivo de texto (capítulo 27), a un `json` que otro servicio entienda, a una fila de base de datos.

El método se llama **`model_dump()`** (volcar, "dump"), y su argumento `mode` decide el disfraz de salida:

```python
print(pedido.model_dump())
# {'folio': 'P-042', 'cliente': {'nombre': 'Ana', 'email': 'ana@dominio.com.ar'},
#  'paquetes': [{'descripcion': 'Mate de calabaza', 'codigo': 'PKG-0034', 'fecha_envio': datetime.date(2026, 12, 31)},
#               {'descripcion': 'Yerba', 'codigo': 'PKG-0035', 'fecha_envio': None}],
#  'estado': <EstadoPedido.PENDIENTE: 'pendiente'>}

print(pedido.model_dump(mode="json"))
# {'folio': 'P-042', 'cliente': {'nombre': 'Ana', 'email': 'ana@dominio.com.ar'},
#  'paquetes': [{'descripcion': 'Mate de calabaza', 'codigo': 'PKG-0034', 'fecha_envio': '2026-12-31'},
#               {'descripcion': 'Yerba', 'codigo': 'PKG-0035', 'fecha_envio': None}],
#  'estado': 'pendiente'}

print(pedido.model_dump_json())
# {"folio":"P-042","cliente":{"nombre":"Ana","email":"ana@dominio.com.ar"},
#  "paquetes":[{"descripcion":"Mate de calabaza","codigo":"PKG-0034","fecha_envio":"2026-12-31"},
#              {"descripcion":"Yerba","codigo":"PKG-0035","fecha_envio":null}],
#  "estado":"pendiente"}
```

Tres voces, una historia. El `dump` pelado mantiene los **tipos nativos**: la fecha sigue siendo un `datetime.date` y el estado sigue siendo el miembro del `Enum`. El `mode="json"` traduce todo a lo que un `json` soporta de verdad: la fecha pasa a `"2026-12-31"` y el estado a su valor plano `"pendiente"`. Y `model_dump_json()` entrega el **texto listo** — string JSON, con comillas dobles y `null` — para escribir directo en un archivo o mandárselo a otro programa.

Lo lindo es que el viaje se hace en ambas direcciones sin perder una coma:

```python
crudo = pedido.model_dump_json()
vuelto = Pedido.model_validate_json(crudo)
print(vuelto.paquetes[0].codigo, "|", vuelto.estado)
# PKG-0034 | EstadoPedido.PENDIENTE
```

El string JSON salió del modelo y **volvió a ser modelo** con `model_validate_json()`: validación completa de nuevo en la entrada, esta vez a partir de texto. Es el bucle entero de la vida de los datos — *del documento al contrato, y del contrato al documento*.

Para los casos livianos, pydantic trae **`TypeAdapter`**: cuando querés validar un **valor suelto** sin armarle un modelo, tomás el tipo directo y lo revestís de validador:

```python
from pydantic import TypeAdapter, ConfigDict

ad = TypeAdapter(list[int])
print(ad.validate_python(["1", 2, 3]))
# [1, 2, 3]
```

`["1", 2, 3]` entró con texto y salió `[1, 2, 3]` puro int: la validación con persistencia, sin clase de por medio.

Y queda un último puente, quizá el más valioso para el mundo real: los objetos que **no son** de pydantic. Una fila de base de datos, un objeto de un ORM —en los proyectos reales te cruzarás con esto todos los días— tiene atributos, pero no tiene contrato. Con **`ConfigDict(from_attributes=True)`** y `model_validate()`, pydantic lee los atributos del objeto ajeno como si fueran un dict:

```python
class ClienteDB:
    def __init__(self, nombre, email):
        self.nombre = nombre
        self.email = email

class ClienteOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    nombre: str
    email: EmailStr

fila = ClienteDB("Luca", "luca@sitio.com")
print(ClienteOut.model_validate(fila).model_dump_json())
# {"nombre":"Luca","email":"luca@sitio.com"}
```

El `ClienteDB` de mentira (o la fila real de un ORM) no sabe nada de pydantic, y sin embargo el contrato lo escaneó: leyó sus atributos, validó su email y lo volcó a JSON. Ese es el patrón `In`/`Out` de los proyectos reales: un modelo para lo que entra, otro para lo que sale, y una frontera de pydantic entre dos mundos.

> **Dato clave:** `model_validate`/`model_validate_json` custodian la **entrada** (dict o texto), `model_dump`/`model_dump_json` preparan la **salida** (dict o texto). `from_attributes=True` extiende la entrada a cualquier objeto con atributos. Cuando montes la API del capítulo 32, vas a ver este mismo baile: el JSON que llega por pedido entra al contrato, y la respuesta sale validada por el contrato de la salida.

---

## 7. Resumen y conceptos clave

La cuenta del capítulo 9 quedó saldada con una venganza elegante. Los hints eran un plano en frío; **pydantic los hizo valer**: tomó la clase con anotaciones que ya sabías escribir y la convirtió en un contrato que **valida** (¿esto es lo que se prometió?) y **convierte** (perfecto, lo dejo listo: `"150.5"` a `150.5`, `"en_camino"` a `EstadoPedido.EN_CAMINO`). Viste la frontera por adentro —`ValidationError` con su expediente de campo, tipo y valor— y la composición en cascada (modelos dentro de modelos, listas de modelos, rutas como `paquetes.1.codigo`). Sumaste los tipos que piensan solos: `Enum`, `Literal`, los `Field` numéricos y de texto, `EmailStr`, `HttpUrl`. Y para las reglas que no vienen de fábrica, escribiste tus propios validadores — `@field_validator` con sus dos puertas (`before` para normalizar, `after` para juzgar) y `@model_validator` para las reglas de negocio que miran el objeto entero. Cerró el viaje de los datos: `model_dump()` y `model_dump_json()` hacia afuera, `model_validate_json()` hacia adentro, `TypeAdapter` para valores sueltos y `from_attributes` para puentear con el mundo que no conoce a pydantic.

Todo este material —declarar, validar, componer, serializar— es exactamente el rito de pasaje que te va a esperar en los capítulos de estructuras de datos y, más adelante, en el capítulo 39, cuando tus modelos sean la puerta de entrada de una API. Los hints, que en el capítulo 9 eran tinta sobre papel, ahora son ley.

Repasá el checklist antes de seguir:

- [ ] Los hints **documentan**; pydantic los hace **validar** en tiempo de ejecución (se cierra la cuenta del capítulo 9 y el puente del 13).
- [ ] **`BaseModel`**: la clase con anotaciones es el contrato; un campo sin valor por defecto es **obligatorio** (`missing`).
- [ ] **Coerción**: `"150.5"` → `150.5`, `"true"` → `True`, `"en_camino"` → `EstadoPedido.EN_CAMINO`. Convertir es también validar.
- [ ] **`ValidationError`**: capturála en la frontera (`try`/`except`) y leé el reporte: campo, `[type=...]` y `input_value`. `e.errors()` da la lista de dicts.
- [ ] **`Enum(str, ...)`** convierte texto en miembro y lo acota; **`Literal[...]`** acota con valores inline. Elegí según la tabla de la sección 3.
- [ ] **`Field`**: `min_length`, `max_length`, `gt`, `ge`, `le`, `pattern` (regex) declaran la regla sin escribir la guardia.
- [ ] `X | None` con default `None` = campo opcional (capítulo 9); `EmailStr`/`HttpUrl` validan formato (requieren `pip install "pydantic[email]"`).
- [ ] **`@field_validator`**: `mode="before"` normaliza/parsea **antes** de los controles; el modo por defecto (`after`) juzga después. El `pattern` corre tras `before`.
- [ ] **`@model_validator(mode="after")`**: reglas cruzadas sobre el objeto entero; `raise ValueError("mensaje")` = mensaje propio; devolvé `self`.
- [ ] **Composición**: modelos dentro de modelos y `list[Modelo]`; los errores se aplanan en rutas (`paquetes.1.codigo`).
- [ ] **`model_dump()`** (tipos nativos), **`model_dump(mode="json")`** (JSON-safe: fecha → texto, Enum → valor), **`model_dump_json()`** (string), **`model_validate_json()`** (de vuelta al modelo).
- [ ] **`TypeAdapter`** valida un valor sin modelo; **`ConfigDict(from_attributes=True)`** + `model_validate()` puentea con objetos ajenos (filas de BD, ORM).

---

## 8. Ejercicios

1. **El contrato del turno**: un modelo `Cita` con `paciente: str` (mínimo 3 caracteres), `hora: int` (entre 8 y 20) y `email: EmailStr`. Probá una cita válida, una con la hora fuera de rango y una con el mail roto; mostrá en la última qué dice `e.errors()[0]["type"]`.
2. **El estado con memoria**: un `EstadoVenta(str, Enum)` con los estados `pagada`, `enviada`, `completada` y `cancelada`; un modelo `Venta` con `total: float = Field(gt=0)` y `estado`. Alimentá el estado con el texto `"enviada"` y mostrá qué pasa al pasar `"perdida"`.
3. **El código que se acomoda**: un modelo `Codigo` con `codigo: str = Field(pattern=r"^EMP-\d{3}$")` y un `@field_validator(mode="before")` que borre los espacios de los costados y pase a mayúsculas. Validá `"  emp-041 "` y mostrá el resultado.
4. **La regla del horario**: un modelo `Turno` con `inicio: int` (0 a 23) y `fin: int` (1 a 24), más un `@model_validator` que exija `fin > inicio`. Mostrá un turno válido y después `Turno(inicio=9, fin=8)`.
5. **El pedido anidado**: un `Item` con `descripcion: str = Field(min_length=1)` y `precio: float = Field(gt=0)`; un `Compra` con `cliente: str` y `items: list[Item]`. Validá un dict con un item de precio negativo y observá la ruta del error en `e.errors()[0]["loc"]`.
6. **El ida y vuelta**: a partir del dict de un `Paquete`, validalo con `model_validate`, mandalo a JSON con `model_dump_json`, reconstruilo con `model_validate_json` y mostrá que el código quedó idéntico.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. El contrato del turno
from pydantic import BaseModel, EmailStr, Field, ValidationError

class Cita(BaseModel):
    paciente: str = Field(min_length=3)
    hora: int = Field(ge=8, le=20)
    email: EmailStr

cita = Cita(paciente="Ana", hora=9, email="ana@dominio.com.ar")
print(cita.paciente, "-", cita.hora)                     # Ana - 9

try:
    Cita(paciente="Ana", hora=21, email="ana@dominio.com.ar")
except ValidationError as e:
    print("fuera de rango:", e.errors()[0]["type"])      # fuera de rango: less_than_equal

try:
    Cita(paciente="Ana", hora=9, email="roto")
except ValidationError as e:
    print("mail roto:", e.errors()[0]["type"])           # mail roto: value_error

# 2. El estado con memoria
from enum import Enum

class EstadoVenta(str, Enum):
    PAGADA = "pagada"
    ENVIADA = "enviada"
    COMPLETADA = "completada"
    CANCELADA = "cancelada"

class Venta(BaseModel):
    total: float = Field(gt=0)
    estado: EstadoVenta

v = Venta(total="150.5", estado="enviada")
print(v.estado, type(v.estado).__name__)                 # EstadoVenta.ENVIADA EstadoVenta

try:
    Venta(total="150.5", estado="perdida")
except ValidationError as e:
    print("estado inválido:", e.errors()[0]["type"])     # estado inválido: enum

# 3. El código que se acomoda
from pydantic import field_validator

class Codigo(BaseModel):
    codigo: str = Field(pattern=r"^EMP-\d{3}$")

    @field_validator("codigo", mode="before")
    @classmethod
    def _limpiar(cls, v):
        return v.strip().upper() if isinstance(v, str) else v

print(Codigo(codigo="  emp-041 ").codigo)                # EMP-041

# 4. La regla del horario
from pydantic import model_validator

class Turno(BaseModel):
    inicio: int = Field(ge=0, le=23)
    fin: int = Field(ge=1, le=24)

    @model_validator(mode="after")
    def _regla(self):
        if self.fin <= self.inicio:
            raise ValueError("fin debe ser posterior a inicio")
        return self

print(Turno(inicio=9, fin=13).model_dump()["fin"])       # 13

try:
    Turno(inicio=9, fin=8)
except ValidationError as e:
    print(e.errors()[0]["msg"])                          # Value error, fin debe ser posterior a inicio

# 5. El pedido anidado
class Item(BaseModel):
    descripcion: str = Field(min_length=1)
    precio: float = Field(gt=0)

class Compra(BaseModel):
    cliente: str
    items: list[Item]

try:
    Compra.model_validate({
        "cliente": "Ana",
        "items": [{"descripcion": "mate", "precio": "12"},
                  {"descripcion": "yerba", "precio": "-3"}],
    })
except ValidationError as e:
    print(e.errors()[0]["loc"])                          # ('items', 1, 'precio')

# 6. El ida y vuelta
class Paquete(BaseModel):
    codigo: str = Field(pattern=r"^PKG-\d{4}$")
    descripcion: str = Field(min_length=3)

    @field_validator("codigo", mode="before")
    @classmethod
    def _limpiar(cls, v):
        return v.strip().upper() if isinstance(v, str) else v

original = {"codigo": "  pkg-0077 ", "descripcion": "Vela"}
paq = Paquete.model_validate(original)
crudo = paq.model_dump_json()
vuelto = Paquete.model_validate_json(crudo)
print(vuelto.codigo == "PKG-0077", vuelto.descripcion)   # True Vela
```