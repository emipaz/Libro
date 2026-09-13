# Capítulo 2 — Instalación con uv y Herramientas Modernas

Antes de escribir tu primer programa necesitas un entorno donde puedas experimentar sin romper nada. Históricamente, esto ha sido complicado: instalar Python, crear un entorno virtual, gestionar dependencias... eran varios pasos manuales, frágiles y fáciles de hacer mal. 

Hoy existe una forma mejor. En este capítulo aprenderás a configurar tu entorno de desarrollo con **uv**, una herramienta moderna que unifica todo el proceso en comandos simples. Al terminar tendrás un entorno listo para el resto del libro, entenderás qué sucede "por debajo" cuando trabajas con entornos virtuales, y sabrás usar las mismas herramientas que verás en el mundo profesional.

## 1. El problema: Múltiples herramientas, un mismo objetivo

Para que entiendas por qué uv es importante, vale la pena mirar el "antes". Configurar un proyecto de Python solía requerir una cadena de pasos manuales:

1. Instalar Python desde el sitio oficial, o usar **pyenv** para gestionar versiones.
2. Crear un entorno virtual con **virtualenv** o `python -m venv`.
3. Activar el entorno manualmente cada vez que querías trabajar.
4. Instalar paquetes con **pip** y crear un `requirements.txt`.
5. (Opcional) Usar **pip-tools** para fijar versiones exactas y garantizar reproducibilidad.

Eran cinco conceptos y cuatro herramientas distintas. En cada paso había una oportunidad para equivocarse: olvidar activar el entorno, instalar en la versión equivocada de Python, crear entornos incompatibles con otros desarrolladores...

**uv** unifica todo esto en un solo binario ultrarrápido, escrito en Rust por el equipo de [Astral](https://astral.sh). Una herramienta, una sintaxis clara, un objetivo: que configurar un proyecto de Python sea rápido y seguro.

| Función | Antes (herramientas clásicas) | Ahora (uv) |
|---------|------|------|
| Instalar Python | pyenv / sitio oficial | `uv python install` |
| Crear entorno virtual | virtualenv / venv | `uv venv` |
| Instalar paquetes | pip | `uv pip install` |
| Fijar versiones exactas | pip-compile | `uv pip compile` |
| Ejecutar scripts | python (+ activar) | `uv run` |

Fíjate en la última fila: `uv run` ejecuta tu script **usando automáticamente el entorno del proyecto**, sin que tengas que activarlo. Esos pequeños ahorros de fricción, multiplicados a lo largo de un proyecto, cambian completamente la experiencia diaria.

## 2. ¿Qué es un entorno virtual?

Antes de instalar uv, necesitas entender un concepto fundamental que usaremos durante todo el libro: el **entorno virtual**.

Un entorno virtual es un directorio aislado que contiene su **propia** instalación de Python y sus **propios** paquetes. ¿Por qué es esto importante? Imagina que trabajas en dos proyectos:

- **Proyecto A** necesita pandas 2.0
- **Proyecto B** necesita pandas 1.5 (una versión más vieja)

Si instalas ambas versiones en el Python del sistema, hay conflicto. ¿Cuál usa Python cuando ejecutas código? Depende del orden de instalación y es un caos.

Los entornos virtuales resuelven esto: cada proyecto tiene su propio "Python" aislado con sus propias dependencias. Proyecto A corre con pandas 2.0 en su entorno; Proyecto B corre con pandas 1.5 en el suyo. Ninguno interfiere con el otro.

> **Dato clave:** la regla es **"un entorno virtual por proyecto"**, no uno global. Así, cada proyecto queda "congelado" con las versiones exactas con las que funciona, y puedes borrar y recrear un entorno cuando quieras sin tocar nada más del sistema.

### Estructura interna de un entorno virtual

La estructura típica de un entorno virtual (creado con `uv venv`) se ve así:

```
.venv/
├── bin/                    (macOS/Linux) o Scripts/ (Windows)
│   ├── python             o python.exe
│   ├── pip                o pip.exe
│   └── activate           o activate.ps1
├── lib/                    (o Lib/ en Windows)
│   └── python3.13/
│       └── site-packages/   ← aquí se instalan los paquetes
└── pyvenv.cfg             Configuración del entorno
```

La carpeta **`site-packages`** es la clave: ahí viven todos los paquetes que instalas en este entorno, completamente aislados del resto del sistema.

## 3. Instalación de uv

Pasemos a lo práctico. El proceso de instalar uv varía según tu sistema operativo, pero en ambos casos es un solo comando.

### En Windows (PowerShell)

Abre PowerShell y ejecuta:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### En macOS o Linux

Abre una terminal y ejecuta:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Verificar la instalación

Una vez instalado, verifica que quedó correctamente ejecutando este código desde una terminal nueva:

```powewrshell
PS D:\> uv --version
uv 0.9.26 (ee4f00362 2026-01-15)
```

o si ya tenias python instalado y queres comprobarlo desde tu python global

```bash
import shutil

uv_path = shutil.which("uv")

if uv_path:
    print(f"✓ uv encontrado en: {uv_path}")
else:
    print("✗ uv no está en el PATH. Ejecuta los comandos de instalación arriba.")
```

Si ves un mensaje con ✓, estás listo para continuar.

## 4. Comandos esenciales de uv

Ahora que uv está instalado, veamos los comandos que usarás con mayor frecuencia. **Importante: ejecuta estos comandos en tu terminal.**

### Gestión de versiones de Python

```bash
# Instalar una versión específica de Python
uv python install 3.13

# Listar todas las versiones disponibles e instaladas
uv python list

# Fijar la versión de Python para este proyecto
# (crea un archivo .python-version)
uv python pin 3.13
```

**¿Qué diferencia hay entre `install` y `pin`?**
- `install` descarga e instala una versión de Python en tu máquina
- `pin` registra qué versión debe usar *este proyecto específico* en un archivo `.python-version`

### Crear y usar entornos virtuales

```bash
# Crear un entorno virtual en el directorio .venv/
uv venv

# Activar el entorno (Windows PowerShell)
.venv\Scripts\Activate.ps1

# Activar el entorno (macOS/Linux)
source .venv/bin/activate
```

### Instalar paquetes y ejecutar scripts

```bash
# Instalar paquetes en el entorno actual (previa activación)
uv pip install pandas numpy matplotlib

# Ejecutar un script usando el entorno del proyecto
# (sin necesidad de activarlo manualmente)
uv run python script.py

# Sincronizar todas las dependencias desde un archivo pyproject.toml
uv sync
```

La belleza de `uv run` es que **maneja todo automáticamente**: si no existe `.venv`, lo crea; si faltan dependencias, las instala. Ejecuta tu código en el contexto correcto sin pasos intermedios.

> **Dato clave:** el **equivalente** de `uv run` en el mundo clásico era: activar el entorno manualmente y luego escribir `python script.py`. Con `uv run` te ahorras la activación: el comando sabe en qué proyecto estás (por el `.venv/` o el `pyproject.toml` de la carpeta) y usa ese contexto solo.

## 5. La cadena de gestión de paquetes en Python

Ya tienes uv instalado, pero el ecosistema de gestión de paquetes en Python tiene varios componentes que conviene conocer. No es necesario dominarlos ahora, pero entender cómo encajan te ayudará a resolver problemas cuando surjan.

### pip: El gestor clásico

**pip** es el instalador de paquetes oficial de Python. Busca paquetes en [PyPI](https://pypi.org) (Python Package Index), el repositorio central de librerías de Python, y los instala resolviendo automáticamente sus dependencias.

Su limitación histórica: por defecto **no fija versiones exactas**. Si ejecutas `pip install pandas` dos veces en momentos diferentes, podrías obtener versiones distintas (pandas 2.0 hoy, pandas 2.1 mañana). Eso rompe la reproducibilidad: el código que funciona hoy puede no funcionar mañana.

### pip-tools y la reproducibilidad

**pip-tools** ataca ese problema con dos comandos:

- **`pip-compile`**: toma un archivo `requirements.in` (tus dependencias directas, con versiones flexibles) y genera un `requirements.txt` con las versiones **exactas** de todo el árbol de dependencias.
- **`pip-sync`**: instala o desinstala paquetes para que tu entorno coincida exactamente con lo que dice el `requirements.txt`.

**Ejemplo de flujo:**

Archivo `requirements.in`:
```
pandas >= 2.0
numpy >= 1.26
matplotlib >= 3.8
```

Ejecutas:
```bash
pip-compile requirements.in -o requirements.txt
```

Archivo `requirements.txt` generado (versiones exactas):
```
pandas == 2.2.1
numpy == 1.26.4
matplotlib == 3.8.3
cffi == 1.16.0        # dependencia de matplotlib
pycparser == 2.21     # dependencia de cffi
...
```

Ahora, cualquier persona que ejecute `pip-sync requirements.txt` obtendrá exactamente las mismas versiones.

### uv: la alternativa moderna

uv ofrece exactamente los mismos conceptos, pero **a la velocidad de Rust**:

```bash
# Compilar dependencias exactas (equivalente a pip-compile)
uv pip compile requirements.in -o requirements.txt

# Sincronizar (equivalente a pip-sync)
uv pip sync requirements.txt
```

Es lo mismo, pero 10-100x más rápido. Para proyectos con cientos de dependencias, la diferencia es notable.

### pip-audit: Auditoría de vulnerabilidades

Otra buena práctica que te ahorrará disgustos en producción: **`pip-audit`** revisa tus dependencias instaladas contra una base de datos de vulnerabilidades conocidas (CVE). Te recomiendo ejecutarlo regularmente como un "chequeo médico" de tu proyecto.

```bash
pip-audit
```

Te mostrará algo como:

```
Found 2 known security vulnerabilities in your environment:
 name: package-name  version: 1.0.0  vulnerability: CVE-YYYY-XXXX
```

### pyproject.toml: El estándar moderno

Finalmente, el **`pyproject.toml`** es hoy el archivo de configuración estándar para cualquier proyecto Python. Define:

- Metadatos del proyecto (nombre, autor, descripción)
- Dependencias (qué paquetes necesitas)
- Herramientas y sus configuraciones (black, pytest, etc.)

Un ejemplo simple:

```toml
[project]
name = "mi-proyecto"
version = "0.1.0"
dependencies = [
    "pandas >= 2.0",
    "numpy >= 1.26",
]

[tool.uv]
python-version = "3.13"
```

Si trabajas en un paquete que estás desarrollando, puedes instalarlo en **modo editable** con:

```bash
uv pip install --editable .
```

Eso te permite modificar el código sin tener que reinstalar cada vez.

## 6. Verificación del entorno

Ahora vamos a verificar que tu entorno está correctamente configurado. Los siguientes bloques de código te mostrarán:

1. La versión de Python que tienes
2. Los paquetes clave instalados
3. La presencia de herramientas de auditoría
4. La configuración de `.gitignore`

Ejecuta cada celda y lee atentamente lo que dice. Si algo falta, los mensajes te dirán exactamente cómo resolverlo.

### Verificar la versión de Python

```python
import sys
import platform

print(f"Python:      {sys.version}")
print(f"Plataforma:  {platform.platform()}")
print(f"Ejecutable:  {sys.executable}")
print()
```

**Resultado esperado:** Python 3.13 o posterior, en tu máquina (Windows, macOS o Linux).

### Verificar paquetes clave

```python
import importlib.metadata

paquetes_clave = [
    "jupyter-book",
    "ipykernel",
    "pandas",
    "numpy",
    "matplotlib",
    "pydantic",
]

print(f"{'Paquete':20s} {'Versión':>12s}  Estado")
print("=" * 50)

for paq in paquetes_clave:
    try:
        ver = importlib.metadata.version(paq)
        print(f"{paq:20s} {ver:>12s}  [OK]")
    except importlib.metadata.PackageNotFoundError:
        print(f"{paq:20s} {'—':>12s}  [NO INSTALADO]")
```

**Resultado esperado:** Al menos `ipykernel`, `pandas` y `numpy` instalados. Si falta alguno, instálalo con `uv pip install <nombre>`.

### Verificar pip-audit

```python
import shutil
import subprocess

pip_audit_path = shutil.which("pip-audit")

if pip_audit_path:
    print(f"✓ pip-audit encontrado en: {pip_audit_path}")
    result = subprocess.run(
        ["pip-audit", "--version"],
        capture_output=True, text=True, timeout=15
    )
    print(f"  Versión: {result.stdout.strip()}")
else:
    print("✗ pip-audit no está instalado.")
    print("  Instálalo con: uv pip install pip-audit")
```

**Resultado esperado:** pip-audit debe estar disponible. Si no, instálalo con el comando que se sugiere.

## 7. Configurar el kernel de Jupyter

### ¿Qué es Jupyter?

**Jupyter** es un entorno interactivo que te permite escribir código Python y documentación (texto, ecuaciones, gráficos) en un mismo lugar, llamado **cuaderno** (*notebook*). A diferencia de un archivo `.py` tradicional que ejecutas de principio a fin, en un cuaderno de Jupyter puedes:

- Escribir código en **celdas** que ejecutas una por una
- Ver resultados inmediatamente después de cada celda
- Mezclar código, explicaciones en texto, y visualizaciones
- Experimentar interactivamente sin escribir un script completo

Es especialmente útil para aprender programación, hacer análisis de datos, y documentar tu trabajo de manera clara.

### ¿Qué es Jupyter Lab?

**Jupyter Lab** es la interfaz moderna y mejorada de Jupyter. Si **Jupyter Notebook** es un cuaderno simple, **Jupyter Lab** es un entorno completo de trabajo con:

- Editor de código avanzado
- Explorador de archivos integrado
- Visor de variables en tiempo real
- Soporte para múltiples lenguajes
- Terminal incorporada
- Mejor disposición de paneles

Cuando instales y abras Jupyter Lab, verás una interfaz visual en el navegador donde puedes crear y editar cuadernos.

### ¿Qué es un kernel?

Un **kernel** es el *intérprete de Python* que ejecuta tu código. Cuando escribes código en una celda de Jupyter Lab y presionas "ejecutar", ese código se envía al kernel, que lo corre y devuelve el resultado.

El truco importante: **tu máquina puede tener múltiples kernels registrados**. Por ejemplo:

- Un kernel que usa el Python del sistema (versión 3.11)
- Un kernel que usa el Python de tu entorno virtual `.venv` (versión 3.13)
- Un kernel de otro proyecto completamente distinto

Jupyter Lab te deja elegir cuál usar cuando abres un cuaderno. **Si no registras un kernel para tu proyecto, Jupyter Lab usará el Python del sistema**, y no tendrá acceso a los paquetes que instalaste en tu `.venv` (como pandas, numpy, etc.).

### Por qué necesitas registrar un kernel

Imagina este escenario:

1. Instalas `pandas` en tu `.venv` con `uv pip install pandas`
2. Abres Jupyter Lab y creas un cuaderno
3. Escribes `import pandas` sin registrar el kernel
4. Obtienes un error: `ModuleNotFoundError: No module named 'pandas'`

¿Por qué? Porque Jupyter Lab está usando el Python del sistema, no el de tu `.venv`. El `pandas` que instalaste está en ``.venv/lib/python3.13/site-packages/``, pero el kernel del sistema no lo ve.

**La solución:** Registrar un kernel que apunte específicamente al Python de tu `.venv`. Así, cuando abras Jupyter Lab y selecciones ese kernel, tendrá acceso a todos tus paquetes instalados.

> **Dato clave:** el kernel y el entorno virtual **no son lo mismo**. El entorno virtual es el conjunto de paquetes; el kernel es el "puente" que Jupyter usa para ejecutar código con un Python concreto. Si instalas paquetes en tu `.venv` pero Jupyter sigue usando el Python del sistema, los paquetes "no existen" para Jupyter. Registrar el kernel conecta ambos.

### El comando de registro

Abre una terminal **fuera de este cuaderno** (no en Jupyter) y ejecuta:

en la carpeta de tu projecto 

**Windows PowerShell:**
```powershell
.venv\Scripts\python.exe -m ipykernel install --user --name=curso-python --display-name "Python 3.13 (curso)"
```

**macOS/Linux:**
```bash
.venv/bin/python -m ipykernel install --user --name=curso-python --display-name "Python 3.13 (curso)"
```

### ¿Qué hace este comando?

| Parámetro | Función |
|-----------|---------|
| `-m ipykernel install` | Ejecuta el módulo `ipykernel` para registrar un kernel nuevo |
| `--user` | Instala el kernel para tu usuario (sin permisos de administrador) |
| `--name=curso-python` | Nombre interno del kernel (usado por Jupyter internamente) |
| `--display-name "..."` | Nombre visible en el menú de kernels de Jupyter |

Una vez registrado, verás **"Python 3.13 (curso)"** en el menú de kernels al crear o abrir cuadernos. Selecciónalo y estarás ejecutando tu código dentro del entorno del proyecto, con todos sus paquetes, sin contaminar el resto del sistema.

### Verificar que ipykernel está listo

```python
import importlib.metadata

try:
    ver = importlib.metadata.version("ipykernel")
    print(f"✓ ipykernel v{ver} está instalado.")
    print("  Puedes registrar el kernel con el comando de la sección anterior.")
except importlib.metadata.PackageNotFoundError:
    print("✗ ipykernel no está instalado.")
    print("  Instálalo con: uv pip install ipykernel")
```

## 8. Buenas prácticas de gestión de dependencias

Para cerrar, aquí hay reglas que te ahorrarán problemas cuando trabajes en proyectos reales. No son teoría: son hábitos que todos los equipos profesionales siguen.

### Git y GitHub, en pocas palabras

Antes de hablar de qué archivos subir o no a tu proyecto, necesitas conocer dos herramientas que verás mencionadas en casi todo trabajo profesional. **Git** es un *control de versiones*, es decir, una herramienta que toma una "foto" de tu código cada vez que se lo pides y guarda ese momento en un historial. Con ese historial puedes volver a cualquier estado anterior de tu proyecto, comparar qué cambió y deshacer errores. Piensa en él como un deshacer ilimitado.

**GitHub** es una plataforma en la nube donde se alojan proyectos de Git. Mientras Git vive en tu computadora, GitHub te permite respaldar ese historial en internet y compartirlo con otras personas. En un equipo, es el lugar donde el código "vive" para todos.

Tres palabras que usarás de aquí en adelante:

- **commit** — el acto de guardar una de esas fotos en el historial.
- **repositorio** (o *repo*) — la carpeta del proyecto junto con su historial.
- **.gitignore** — la lista de archivos que le pides a Git que ignore, porque no quieres que entren en el historial.

No profundizamos más por ahora: al final del libro hay un anexo completo dedicado a Git y GitHub. Con estas tres ideas basta para entender la regla de oro que viene.

### El flujo: requirements.in → requirements.txt

El enfoque recomendado es trabajar con **dos archivos**:

**Archivo 1: `requirements.in`** — tus dependencias **directas**, sin versión exacta:
```
pandas >= 2.0
numpy >= 1.26
matplotlib >= 3.8
```

Este archivo lo escribes tú y lo mantienes a mano. Es legible, simple, y claro sobre qué necesita tu proyecto.

**Archivo 2: `requirements.txt`** — generado automáticamente con `pip-compile` o `uv pip compile`, con versiones **exactas** fijadas:
```
pandas == 2.2.1
numpy == 1.26.4
matplotlib == 3.8.3
cffi == 1.16.0
pycparser == 2.21
...
```

Este archivo lo genera la máquina, no lo editas. Garantiza reproducibilidad total.

**El flujo:**

```bash
# Editas requirements.in manualmente
nano requirements.in

# Compilas a versiones exactas
uv pip compile requirements.in -o requirements.txt

# Sincronizas tu entorno
uv pip sync requirements.txt
```

Este flujo garantiza que cualquier persona (o tu "yo" del futuro) que clone el proyecto obtendrá exactamente las mismas versiones. Es crítico para equipos y para despliegues en producción.

### Lock files

Herramientas como `uv` y `pip-tools` generan archivos de bloqueo (*lock files*) que registran la configuración exacta del entorno. Piensa en ellos como una "foto congelada" del estado de tu proyecto en un momento específico. Son esenciales para reproducibilidad.

**Regla de oro:** commit el `requirements.txt` (o `uv.lock` si usas uv) a Git. Así otros desarrolladores obtienen exactamente lo que tú tenías.

### El .venv nunca se versiona en Git

El directorio `.venv/` **nunca** debe subirse a un repositorio Git. Es:
- **Pesado** (cientos de MB)
- **Regenerable** (puedes recrearlo en cualquier momento con `uv venv`)
- **No portable** (cambia según el sistema operativo)

> **Dato clave:** lo que sí se versiona (lo que garantiza la reproducibilidad) es el **`requirements.txt`** (o `uv.lock`), no el `.venv`. Cualquier persona que clone tu proyecto crea su propio `.venv` local y ejecuta `uv pip sync requirements.txt` para obtener exactamente tu mismo entorno. El `.venv` se ve igual, pero es individual de cada desarrollador.

En su lugar, crea un `.gitignore` en la raíz del proyecto:

```gitignore
# Entornos virtuales
.venv/
venv/
env/

# Caché de Python
__pycache__/
*.pyc
*.pyo
*.egg-info/

# IDE
.vscode/
.idea/
*.swp
```

### Verificar tu .gitignore

```python
from pathlib import Path

gitignore = Path(".gitignore")

if gitignore.exists():
    contenido = gitignore.read_text(encoding="utf-8")
    print("✓ .gitignore encontrado:")
    print("-" * 50)
    print(contenido)
else:
    print("✗ No se encontró .gitignore en el directorio actual.")
    print("  Crea uno con las entradas de la sección anterior.")
```

## 9. Resumen y conceptos clave

Has configurado tu primer entorno Python profesional. Aprendiste por qué las herramientas viejas eran engorrosas, entendiste qué es un entorno virtual y por qué los necesitas, instalaste uv, aprendiste sus comandos esenciales, comprendiste la cadena de gestión de paquetes, verificaste tu entorno, registraste un kernel de Jupyter y aprendiste las buenas prácticas que los equipos reales usan.

**La idea clave: uv no es solo un instalador más rápido. Es una forma más simple, segura y profesional de trabajar.**

Contrasta con esta lista. Asegúrate de que cada punto es claro antes de continuar:

- [ ] **uv** es un gestor moderno de Python y paquetes, escrito en Rust, que reemplaza pip + pyenv + virtualenv.
- [ ] Un **entorno virtual** es un directorio aislado con su propia instalación de Python y sus propias dependencias.
- [ ] `uv python install` descarga versiones de Python; `uv python pin` registra qué versión usa este proyecto.
- [ ] `uv venv` crea un entorno virtual en `.venv/` que puedes activar manualmente.
- [ ] `uv pip install` instala paquetes; `uv run` ejecuta scripts usando automáticamente el entorno del proyecto.
- [ ] Los entornos virtuales aíslan dependencias **por proyecto**, evitando conflictos entre versiones.
- [ ] `pip-compile` / `uv pip compile` generan `requirements.txt` con versiones exactas (*pinned*) a partir de `requirements.in`.
- [ ] `pip-audit` verifica vulnerabilidades conocidas en las dependencias instaladas.
- [ ] El kernel de Jupyter se registra con `ipykernel install` para usar el intérprete de tu `.venv`.
- [ ] `.venv/` **nunca** se versiona en Git; siempre añádelo a `.gitignore`.
- [ ] **`pyproject.toml`** es el estándar moderno para configurar proyectos Python.

Con tu entorno listo y funcionando, estás en condiciones de pasar a lo más interesante: escribir código Python. En el próximo capítulo conocerás los fundamentos del lenguaje, y ahí es donde finalmente vas a ensuciarte las manos.
