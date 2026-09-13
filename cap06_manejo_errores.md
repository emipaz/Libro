# Capítulo 6 — Manejo de errores y excepciones

Llegaste lejos: tus programas deciden, repiten, agrupan datos y hasta reconocen patrones. Ahora viene la parte incómoda pero inevitable: **cuando los datos no llegan como esperás, el programa falla**. Un usuario escribe letras donde pediste números, se borra un archivo, la API responde con otra estructura. Los errores no son un accidente del mal código: son parte del paisaje de cualquier programa real. Este capítulo te enseña a **anticiparlos y manejarlos con elegancia** — con `try`/`except`, `raise`, `assert` y tus propias excepciones. Cuando termines, vas a entender por qué el `else` del capítulo 4 tiene un cuarto hermano que casi nadie enseña.

---

## 1. Dos tipos de errores: sintaxis y excepciones

En Python hay **dos grandes categorías** de errores, y son tan distintas que hasta ocurren en momentos distintos:

- **Errores de sintaxis** (`SyntaxError`): el código ni siquiera se puede entender. Pasa antes de ejecutar, cuando Python intenta "leer" tu archivo.
- **Excepciones** (errores en tiempo de ejecución): el código es válido y se interpreta, pero **falla al ejecutarse**. Por ejemplo, dividir por cero, o convertir `"abc"` a entero.

La diferencia te interesa porque es la definición de lo que podés **capturar y controlar**: a una excepción la podés atrapar; a un `SyntaxError` no — tu programa no llega ni a correr.

```python
# esto nunca se ejecuta: Python no entiende el código
x = = 5          # SyntaxError: invalid syntax
```

> **Dato clave:** una forma rápida de saber si algo es excepción o error de sintaxis: si te aparece el mensaje `SyntaxError`, es de sintaxis. Todo lo demás (lo vas a ver en este capítulo) es una excepción que **se puede atrapar**.

La lista de nombres de excepciones va a aparecer constantemente — `ValueError`, `TypeError`, `ZeroDivisionError`… Por ahora no memorices: alcanza con saber que **cada tipo de error tiene su nombre** y que poder nombrarlo es el primer paso para manejarlo.

---

## 2. Las excepciones más comunes: tu vocabulario de errores

Estas son las que vas a ver en el 90% de tus primeros programas. Fijate que cada nombre describe el *tipo* de problema, no el caso concreto.

| Excepción           | Cuándo aparece                                       | Ejemplo                        |
|---------------------|------------------------------------------------------|--------------------------------|
| `ValueError`        | valor incorrecto para una operación                  | `int("abc")`, `float("hola")`  |
| `TypeError`         | tipo incorrecto en una operación                     | `"1" + 1`, `len(5)`            |
| `ZeroDivisionError` | división entre cero                                  | `10 / 0`                       |
| `IndexError`        | índice fuera de rango en una **secuencia**           | `[1, 2, 3][10]`                |
| `KeyError`          | clave que no existe en un **diccionario**            | `{"a": 1}["z"]`                |
| `AttributeError`    | atributo o método que no existe en un objeto         | `(3).upper()`                  |
| `FileNotFoundError` | archivo que no existe                                | `open("no_existe.txt")`        |
| `ImportError`       | módulo o nombre que no se puede importar             | `import modulo_inexistente`    |
| `NameError`         | nombre de variable que no está definido              | usar `edad` sin haberla creado |

```python
# NameError
print(edad_mal_escrita)   # NameError: name 'edad_mal_escrita' is not defined

# AttributeError
numeros = [1, 2, 3]
numeros.upper()           # AttributeError: 'list' object has no attribute 'upper'
```

Fijate en la pareja que ya conocés por el capítulo 5: `IndexError` es de las **secuencias** (indexás mal una lista o tupla) y `KeyError` es de los **diccionarios** (buscás una clave que no está). El nombre de la excepción te dice *qué tipo de dato* la provocó.

> **Importante:** los nombres de las excepciones forman una **jerarquía**, y la clave de todo el capítulo está en la punta: las excepciones comunes heredan de `Exception`, y `Exception` hereda de `BaseException`. Eso significa, entre otras cosas, que cuando capturás `Exception` capturás casi todas — pero no a `KeyboardInterrupt` (Ctrl + C) ni a `SystemExit`, que descienden directo de `BaseException`. Por eso la recomendación "nunca captures `BaseException`" va a aparecer más de una vez en código real.

---

## 3. `try`/`except`: el cinturón de seguridad

La estructura que captura una excepción es el bloque **`try`/`except`** — *try*, "intentar", y *except*, "excepto en caso de que…":

```python
try:
    # código que puede fallar
except TipoDeError:
    # qué hacer si falla
```

Python intenta ejecutar el bloque de `try`. Si en alguna línea salta una excepción, **detiene el `try` ahí mismo** y salta al bloque `except` que corresponda (si lo hay). Si no falla nada, el `except` se ignora.

```python
try:
    numero = int("abc")
except ValueError:
    print("no se pudo convertir")
```

> **Dato clave:** el `try` funciona sobre el **bloque completo**: si la línea 3 de un `try` de 10 líneas falla, Python salta directo al `except` y **las líneas 4 a 10 nunca se ejecutan**. No es un "intento en cada línea": es todo el bloque como unidad.

### Capturar el error en una variable con `as`

Con `as e` ligás el objeto excepción a una variable y podés leer su mensaje:

```python
try:
    numero = int("abc")
except ValueError as e:
    print(f"el mensaje del error es: {e}")
```

Eso se vuelve clave cuando la excepción trae información útil. Es la misma mentalidad que en el capítulo 5, donde `get()` te devolvía el dato o un default: acá `except ... as e` te da el error, y lo que hagas con él depende de vos.

### Múltiples `except`: capturar varios tipos

Un `try` puede tener varios `except`, uno por tipo de error, y se evalúan **en orden**:

```python
a, b = 10, 0

try:
    resultado = a / b
except ZeroDivisionError:
    print("no se puede dividir por cero")
except TypeError:
    print("los operandos deben ser numéricos")
# salida: no se puede dividir por cero
```

Y cuando dos errores distintos se resuelven de la misma forma, se agrupan en una **tupla**:

```python
texto = "abc"

try:
    numero = int(texto)
except (ValueError, TypeError) as e:
    print("no se pudo convertir:", e)
else:
    print("conversión correcta:", numero)
# salida: no se pudo convertir: invalid literal for int() with base 10: 'abc'
```

> **Dato clave:** los `except` se prueban **en orden y solo se ejecuta el primero** que coincida. Por eso el orden importa: un `except ValueError` antes de un `except Exception` es más específico y gana; al revés, `ValueError` quedaría opacado por el más general.

---

## 4. `else` y `finally`: los dos hermanos del `try`

Ahora el tema que prometió el capítulo 4. En la sección 6 viste el `else` de los bucles — "se ejecuta si el bucle terminó sin `break`". Acá hay una jugada parecida y **mucho** menos conocida:

- El **`else` del `try`** se ejecuta **solo si no hubo ninguna excepción**. Si hubo, se saltea.
- El **`finally`** se ejecuta **SIEMPRE**, haya excepción o no.

```python
# Caso 1: sin error
a, b = 10, 2

try:
    resultado = a / b
except ZeroDivisionError:
    print("error: división por cero")
except TypeError:
    print("error: operandos no numéricos")
else:
    print("resultado:", resultado)
finally:
    print("se ejecutó el finally")
# salida:
# resultado: 5.0
# se ejecutó el finally
```

```python
# Caso 2: con error (división por cero)
a, b = 10, 0

try:
    resultado = a / b
except ZeroDivisionError:
    print("error: división por cero")
except TypeError:
    print("error: operandos no numéricos")
else:
    print("resultado:", resultado)
finally:
    print("se ejecutó el finally")
# salida:
# error: división por cero
# se ejecutó el finally
```

La tabla de las combinaciones posibles:

| ¿Hubo excepción?                                  | `except`                              | `else`                      | `finally`    |
|---------------------------------------------------|---------------------------------------|-----------------------------|--------------|
| no                                                | se ignora                             | se ejecuta                  | se ejecuta   |
| sí, y hay un `except` que la captura              | se ejecuta                            | se ignora                   | se ejecuta   |
| sí, y ningún `except` la captura                  | se ignora                             | se ignora                   | se ejecuta   |

> **Importante:** `else` y `finally` **no** son intercambiables. El `else` corre solo cuando todo salió bien: es el lugar natural para el código que *usa el resultado* del `try`. El `finally` corre bajo todas las circunstancias: es el lugar para **liberar recursos** ("pase lo que pase, cerrá esto"). Con esto ya viste el `else` en todos los flujos que Python permite — `if`, `while`, `for`, `try` — y el `match`/`case` con su `case _` es el único que no lo tiene.

El caso de uso clásico del `finally` es cuando abrís un recurso y tenés que asegurarte de cerrarlo **pase lo que pase** (en serio lo vas a explotar en la Parte VIII, con archivos):

```python
f = open("ejemplo.txt")
try:
    contenido = f.read()
finally:
    f.close()       # se cierra siempre, aunque read() falle
```

> **Buenas prácticas:** a ese bloque hoy se lo prefiere en versión compacta, con el **gestor de contexto** `with` (*context manager*, "administrador de contexto"), que abre y **libera solo** al salir:

```python
with open("ejemplo.txt") as f:
    contenido = f.read()     # f se cierra automáticamente al salir del with
```

---

## 5. `raise`: lanzar errores a propósito

Capturar errores es la mitad del trabajo. La otra mitad es **lanzarlos** cuando detectás una situación inválida — no esperar a que el lenguaje falle, sino fallar **vos, con el mensaje que querés**. La palabra es `raise` ("lanzar, elevar"):

```python
n = 5

if n < 0:
    raise ValueError(f"{n} no es un número positivo")

print(n)     # 5
```

Ahora capturamos el error

```python
n = -5

try:
    if n < 0:
        raise ValueError(f"{n} no es un número positivo")
    else:
        print(n) 
except ValueError as e:
    print("capturado:", e)   # capturado: -5 no es un número positivo
```

- `raise` **detiene la ejecución del código ahí mismo**, como un stop abrupto.
- `raise` no devuelve un valor: **aborta** hacia el `try` más cercano. Si no hay ninguno, el programa se detiene y muestra el *traceback* (el "rastro" de llamadas que llevaron al error).
- La sintaxis es `raise` + una **instancia** de excepción (`raise ValueError("mensaje")`) o directamente la **clase** (`raise ValueError`), que arma la instancia sola.

> **Dato clave:** raise corta la ejecución en el acto y le pasa el problema a quien tenga un try esperándolo. La regla práctica: si el dato inválido es algo previsible (una entrada equivocada), raise y que el que quiera lo capture; si es un bug de lógica del programa, quizá lo que precisás es una assert (sección siguiente).

### Relanzar con un `raise` pelado

Dentro de un bloque `except`, un `raise` sin argumentos **relanza la misma excepción** que se está manejando. Es la forma de decir "miré el error, lo registré, y lo dejo subir":

```python
try:
    # simulamos un error de conexión
    raise ConnectionError("servidor caído")
except ConnectionError as e:
    print("registrado en un log:", e)
    raise                    # vuelve a lanzar la misma excepción
# el programa se detiene aquí con el error original
```

Ese patrón —capturar para registrar y luego relanzar— es de los que más se usan en sistemas reales.

### No levantar algo y dejar que pase

No todo error necesita `try`/`except`. El patrón que vas a ver en código profesional es: **detectar el error, lanzarlo con `raise`, y capturarlo donde tenga sentido**. Por ahora, ese lugar es el programa principal:

```python
edad_texto = "-5"

try:
    edad = int(edad_texto)          # si escribe letras -> ValueError
    if edad < 0:
        raise ValueError("la edad no puede ser negativa")
except ValueError as e:
    print("entrada rechazada:", e)
else:
    print("edad válida:", edad)
# salida: entrada rechazada: la edad no puede ser negativa
```

---

## 6. `assert`: pedir garantías en el código

La palabra **`assert`** ("afirmar, asegurar") es una forma de **declarar una condición que a tu juicio siempre debe cumplirse**. Es como escribirle una nota al código: "en este punto, estoy segura de que esto es así". Si la condición es `True`, no pasa nada y el programa sigue; si es `False`, Python lanza un `AssertionError`:

```python
x = 10
assert x > 0          # pasa: x es positivo
assert x < 0          # AssertionError: se corta el programa aquí
```

La sintaxis es `assert <condición>` y puede llevar un **mensaje**:

```python
dividendo, divisor = 10, 0

assert divisor != 0, "el divisor no puede ser cero"
```

Fijate que `assert` **corta el programa** si la condición falla — no es un `try` que captura: es una *verificación dura* (el famoso "falla rápido, *fail fast*").

### Los usos reales de `assert`

Los tres casos donde `assert` brilla:

**1. Precondiciones e invariantes internos.** "Esta lista debe estar ordenada", "la edad nunca puede ser negativa". Son condiciones del *código*, no del usuario:

```python
lista = [1, 2, 3, 4]
assert lista, "no se puede promediar una lista vacía"
resultado = sum(lista) / len(lista)
print("promedio:", resultado)
```

**2. Guardas en desarrollo.** Documentás expectativas que, si se rompen, te avisan *a vos* (el programador) que algo se desincronizó:

```python
precio = 100
assert precio >= 0, "el precio no puede ser negativo"
precio_con_descuento = precio * 0.9
print("con descuento:", precio_con_descuento)
```

**3. Tests y verificación de resultados.** Cuando escribís código, verificas que el resultado es el esperado:

```python
n = 2
resultado = n * 2
assert resultado == 4, "debería devolver 4"
print("test pasado")
```

> **Importante:** esta es la diferencia conceptual que tenés que evitar confundir. **`assert` no es para validar la entrada del usuario**: es para condiciones internas que *deberían* ser ciertas siempre. Para entrada del usuario se usa `raise` + `try`/`except` (sección 5). La pista: si la condición depende de lo que escribe una persona o lee de un archivo, no es una `assert`; si depende de tu propia lógica, sí.

### `assert` se puede apagar

`assert` es una *verificación liviana*: corre solo en el modo de ejecución normal. Si ejecutás Python con la opción `-O` (optimizar, de *"optimize"*), **todas las `assert` se ignoran** — Python elimina cada cheque por completo:
```bash
python programa.py    # las assert corren
python -O programa.py # las assert NO corren
```

Que no se ejecuten no es un accidente: es la justificación del párrafo anterior. Como son condiciones *internas* del código (que después de pruebas no deberían romperse), Python las considera "gasto que se puede recortar en producción". Pero fijate lo que implica: **si necesitás validar algo importante, no lo dejes a cargo de `assert`** — el `raise` y el `try`/`except` siempre están activos.

> **Dato clave:** la regla de oro: `assert` para *debuguear* (condiciones internas, desarrollo, tests), `raise` + `try`/`except` para *validar* (entrada de usuario, datos de archivos, producción). `assert` puede desaparecer con `-O`; `raise` no.

---

## 7. Excepciones personalizadas: nombrar tus propios errores

Hasta acá usaste excepciones que vienen con el lenguaje: `ValueError`, `TypeError`… Pero el código de un dominio concreto tiene sus propios fracasos. Un sistema de cuentas tiene el error "saldo insuficiente"; un juego, "movimiento inválido"; una agenda, "contacto inexistente". Inventar tu propia excepción le da **un nombre** a ese fracaso y hace que el código lo hable con las palabras de tu problema.

Para crear una excepción propia se define una **clase** que hereda de `Exception`. Vas a ver POO en detalle en la Parte VI, pero acá alcanza con esta plantilla mínima:

```python
class SaldoInsuficienteError(Exception):
    """Se lanza cuando se intenta retirar más de lo que hay."""
    pass
```

> **Importante:** la sintaxis `class ... (Exception):` define una nueva clase que **hereda** el comportamiento de `Exception`. El `pass` indica que el cuerpo de la clase está vacío — por ahora solo nos interesa el *nombre* de la excepción, no su comportamiento interno. En la Parte VI (programación orientada a objetos) vamos a profundizar en qué significa heredar, cómo funciona el `__init__` y cómo personalizar aún más las excepciones. Acá las vas a usar como "etiquetas nombradas".

Una excepción personalizada se **lanza** con `raise` y se **captura** como cualquier otra:

```python
class SaldoInsuficienteError(Exception):
    """Se lanza cuando se intenta retirar más de lo que hay."""
    pass

saldo = 100
monto = 150

try:
    if monto > saldo:
        raise SaldoInsuficienteError(
            f"saldo disponible {saldo}, intentó retirar {monto}"
        )
    saldo = saldo - monto
except SaldoInsuficienteError as e:
    print("operación rechazada:", e)
# salida: operación rechazada: saldo disponible 100, intentó retirar 150
```

### El flujo completo: dominio + frontera

El patrón de la sección 5 cobra otra dimensión con errores propios. Definís la excepción propia, la lanzás cuando detectás un problema, y la capturás donde tenga sentido:

```python
class EntradaClaveError(Exception):
    """Clave o usuario incorrectos."""
    pass

usuarios_validos = {"juan": "1234"}
usuario = "juan"
clave = "xxxx"

try:
    if usuario not in usuarios_validos:
        raise EntradaClaveError(f"no existe el usuario {usuario}")
    if usuarios_validos[usuario] != clave:
        raise EntradaClaveError("clave incorrecta")
    print(f"bienvenido, {usuario}")
except EntradaClaveError as e:
    print("no se pudo iniciar sesión:", e)
# salida: no se pudo iniciar sesión: clave incorrecta
```

> **Buenas prácticas:** mientras arrancás, basta con que tus excepciones hereden de `Exception` y tengan una **docstring** (el texto entre `"""` que documenta qué significa). Después vas a ver que se pueden agregar atributos, mensajes customizados en `__init__` y hasta métodos — todo eso es terreno de POO (Parte VI). El hábito que te llevás hoy: **una excepción por fracaso del dominio, con nombre descriptivo, y capturarla justo donde se decide**.

### ¿Cuándo conviene una excepción personalizada?

- Cuando el mismo fracaso ocurre en **varios lugares** de tu código (validar, autenticar, calcular) y querés capturarlo siempre igual.
- Cuando un `ValueError` genérico es ambiguo: `SaldoInsuficienteError` se explica sola, y su `as e` trae contexto.
- Cuando querés que el programa pruebe el **nombre** del error en un `except`: `except SaldoInsuficienteError` comunica intención mejor que `except ValueError`.

No te preocupes por crear errores propios *antes* de tiempo. La señal de que los necesitás aparece sola: cuando repetís `raise ValueError` con el mismo concepto en varios archivos, ya tenés candidato a excepción propia.

---

## 8. Buenas prácticas: manejar o propagar

Cerrar el capítulo con las costumbres que hacen a un código con errores *digerible*. Tres reglas y un anti-patrón:

### 1. Capturá solo lo que podés manejar

Cada `except` es una promesa: "este fracaso lo sé resolver". Capturar un `TypeError` y no saber qué hacer con él es peor que dejarlo subir.

```python
# aceptable: sé el problema y tengo plan
try:
    numero = int("42")
except ValueError:
    numero = 0          # plan concreto: default

# mal: capturo todo y no hago nada
try:
    numero = int("42")
except Exception:
    pass                # se traga el error y sigue como si nada
```

### 2. El anti-patrón del `except: pass` silencioso

El `pass` dentro de un `except` es el clásico *error tragado*: el programa sigue corriendo, pero vos nunca te enterás de que algo falló. Si no lo vas a atender, al menos **muestralo o registralo**:

```python
try:
    dato = int(input("edad: "))
except ValueError as e:
    print("no se pudo leer la edad — se usa 0 por defecto:", e)
    dato = 0           # explícito: sé que usé un fallback
```

> **Importante:** un `except: pass` sin rastro es la forma más fea de perder un error. En el 90% de los casos el error tiene información valiosa; tragartelo es como tapar la luz de aceite del auto con cinta. O lo manejás (y queda registro) o lo **propagás** con un `raise` pelado (sección 5).

### 3. Propagar cuando no sabés

Si no sabés resolver el problema, **no lo captures**: dejá que suba. Si capturás parcialmente, relanzá con `raise` para no perder el rastro:

```python
try:
    with open("config.json") as f:
        contenido = f.read()
except FileNotFoundError:
    # este error SÍ sé resolver
    print("no había config; sigo con defaults")
    contenido = "{}"
except Exception as e:
    # este error NO sé resolver: lo registro y lo dejo subir
    print("error inesperado:", e)
    raise
```

### El `with` como reflexión, no como reemplazo

Ya viste `with` en la sección 4. La idea que suma acá: `with` no captura errores — **garantiza liberación de recursos** pase lo que pase. Componen perfecto con `try`/`except`:

```python
try:
    with open("datos.txt") as f:
        contenido = f.read()         # f se cierra solo, incluso con error
except FileNotFoundError:
    print("el archivo no existe; registrando el problema...")
except Exception as e:
    print("error inesperado, lo registro y lo propago:", e)
    raise
```

---

## 9. Resumen y conceptos clave

Este capítulo te dio el par completo sobre errores: **capturarlos** (`try`/`except`) y **lanzarlos** (`raise`), con un espacio propio para cada matiz — el `else` que corre cuando todo salió bien, el `finally` que corre siempre, y la pareja `assert` vs `raise` que define cuándo una condición es "del código" y cuándo es "de los datos". Sobre esa base aprendiste a darle **nombre a los fracasos de tu dominio** con excepciones personalizadas, y a usar las buenas prácticas que separan un manejo de errores sólido de un `except: pass` que esconde problemas. Con esto, tus programas no solo hacen más cosas: saben *explicar* qué les pasó.

Repasa con esta lista y asegúrate de que cada punto te resulta familiar antes de continuar:

- [ ] Hay **dos tipos de errores**: de sintaxis (`SyntaxError`, irrecuperable) y **excepciones** en tiempo de ejecución (capturables).
- [ ] Vocabulario de errores: `ValueError`, `TypeError`, `ZeroDivisionError`, `IndexError`, `KeyError`, `AttributeError`, `FileNotFoundError`, `ImportError`, `NameError`.
- [ ] Las excepciones forman una **jerarquía**: todas las comunes heredan de `Exception`; `KeyboardInterrupt` y `SystemExit` cuelgan de `BaseException`.
- [ ] `try`/`except` captura: cuanto capturás con `as e`, leés el mensaje; el `try` es de **bloque completo**.
- [ ] Varios `except` se prueban **en orden**; errores que se resuelven igual se agrupan en tupla: `except (ValueError, TypeError)`.
- [ ] El **`else` del `try`** corre solo si no hubo excepción; el **`finally`** corre siempre (liberar recursos).
- [ ] `with` abre y **libera solo** al salir (gestor de contexto); no captura errores, pero compone con `try`.
- [ ] **`raise`** lanza un error con tu mensaje; `raise` pelado dentro de un `except` **relanza** la misma excepción.
- [ ] **`assert`** declara una condición interna que debe cumplirse; si es `False` → `AssertionError` y corta el programa.
- [ ] `assert` se **desactiva** con `python -O`; por eso valida interna (debug/tests), no entrada de usuario → ahí va `raise` + `try`.
- [ ] **Excepciones personalizadas**: `class MiError(Exception):` y `raise MiError("...")`. Los detalles de `class` se completan en la Parte VI (POO).
- [ ] Buenas prácticas: capturar solo lo que podés manejar, **nunca `except: pass` silencioso**, y **propagar** con `raise` lo que no es tu capa resolver.

## 10. Ejercicios

> **Nota importante:** algunos ejercicios usan funciones — cuando llegues a la **Parte V (Funciones)**, los completarás allá. Por ahora practica los que funcionan sin ellas.

1. **Conversión con default** (sin función): `int(texto)` dentro de `try`; si falla (`ValueError` o `TypeError`), usá `0`. Probá con `"42"`, `"abc"` y `None`.
2. **Contar intentos con excepción**: un `while` que intente `int(input())` y capture `ValueError` pidiendo de nuevo hasta lograrlo. (Si no querés leer entrada interactiva, usá una lista con valores prefijados y recorréla.)
3. **`assert` en una operación**: escribí un bloque que sume dos números, agregá un `assert` que garantice que el resultado no sea negativo, y otro que no sea `None`.
4. **Excepción personalizada**: creá `EdadNoValidaError(Exception)` con `pass`, y un bloque que lance la excepción si la edad es menor a 18. Capturala y mostrá el mensaje.
5. **`try`/`except` con múltiples errores**: escribí un bloque que intente convertir texto a entero, divida por ese número, y capture tanto `ValueError` como `ZeroDivisionError` de forma diferente.
6. **Relanzar con `raise` pelado**: crea un bloque que capture una excepción, la registre (con `print`), y la relance.
7. **Propagar selectivamente**: usa `try`/`except` para capturar `FileNotFoundError` con un plan (mostrar defaults), pero relanzá cualquier otro `Exception`.

```python
# Soluciones (no las mires antes de intentarlo)

# 1. Conversión con default
for texto in ["42", "abc", None]:
    try:
        numero = int(texto)
    except (ValueError, TypeError):
        numero = 0
    print(f"convertir({texto!r}) = {numero}")
# salida:
# convertir('42') = 42
# convertir('abc') = 0
# convertir(None) = 0

# 2. Contar intentos
valores = ["abc", "25", "100"]  # simulamos entrada
idx = 0
while True:
    try:
        numero = int(valores[idx])
        print("logrado:", numero)
        break
    except ValueError:
        print("eso no era un número, probá de nuevo")
        idx += 1

# 3. assert en una operación
a, b = 10, 20
resultado = a + b
assert resultado is not None, "el resultado no debería ser None"
assert resultado >= 0, "el resultado no debería ser negativo"
print("suma:", resultado)

# 4. Excepción personalizada
class EdadNoValidaError(Exception):
    """Se lanza cuando la edad no es válida."""
    pass

edad = 15
try:
    if edad < 18:
        raise EdadNoValidaError(f"{edad} es menor de 18")
    print("mayor de edad")
except EdadNoValidaError as e:
    print("capturado:", e)  # capturado: 15 es menor de 18

# 5. try/except con múltiples errores
texto = "0"
try:
    numero = int(texto)
    resultado = 10 / numero
    print("resultado:", resultado)
except ValueError:
    print("error: no es un número válido")
except ZeroDivisionError:
    print("error: no se puede dividir por cero")

# 6. Relanzar con raise pelado
try:
    raise ConnectionError("servidor caído")
except ConnectionError as e:
    print("registrado:", e)
    raise  # relanza la misma excepción
# el programa se detiene aquí

# 7. Propagar selectivamente
try:
    with open("config.json") as f:
        contenido = f.read()
except FileNotFoundError:
    print("no había config; sigo con defaults")
    contenido = "{}"
except Exception as e:
    print("error inesperado:", e)
    raise  # propago lo que no sé resolver
```

Con el manejo de errores ya tenés los ingredientes de un programa "de verdad": decidir, repetir, agrupar datos, reconocer patrones y ahora **fallar con elegancia**. El próximo paso natural es la **Parte V — Funciones**, donde vamos a usar `try`/`except`, `raise` y `assert` adentro de funciones para construir las que siguen. Una cosa más que vas a ver: el `raise` y el `try` se vuelven la columna vertebral de las excepciones personalizadas que vas a armar en POO (Parte VI).