# Capítulo 12.0 — POO , sus pilares y el camino de Python

Este es un capítulo raro: **no hay una línea de sintaxis nueva que aprender**. Es, literalmente, el antes de del la Parte VI. que está dedicada a la Programación Orientada a Objetos (POO) hecha por Python, y para apreciar *cómo* la hace Python, hace falta saber primero *qué es* la POO y *de dónde viene*.

Porque acá hay una trampa histórica que casi nadie cuenta. Cuando se discute POO, el ejemplo de "lenguaje estricto" suele ser Java o C#: `private`, `interface`, clases abstractas que obligan, compilador policía. Y se da por sentado que Python "llegó después" y se desvió de la regla. Nada más lejos de la verdad. **Python nació antes que Java**, tenía clases y herencia desde su primera versión pública de 1991, y se construyó sobre un lenguaje educativo holandés llamado ABC. No es un lenguaje que se rebeló contra el manual: es un lenguaje que empezó con otra forma de pensar el molde.

En este capítulo vas a recorrer esa historia en dos paradas. Primero la línea de tiempo del paradigma — Simula, Smalltalk, C++, ABC, Python, Java — con los nombres y fechas que la armaron. Después, los **cuatro pilares clásicos de la POO** explicados con su cara más formal, para que cuando el rebelde los desarme en los capítulos 12 al 17, el contraste te pegue de lleno. Al cierre vas a entender por qué esta parte del libro se llama *"Python, el rebelde"* y qué significa, exactamente, eso de que acá las reglas existen **por convención, no por decreto**.

---

## 1. La historia: de Simula a Python

La Programación Orientada a Objetos no nació en un día ni en un solo lugar: fue una idea que maduró a lo largo de tres décadas, y que cada lenguaje adoptó con un sesgo distinto. Esta es la línea de tiempo completa, y después la desmenuzamos:

| Año | Evento | Qué aportó |
|-----|--------|-----------|
| 1967 | **Simula 67** (Ole-Johan Dahl y Kristen Nygaard, Noruega) | las primeras **clases y objetos** formales; nació para modelar sistemas de simulación |
| ~1972 | **Smalltalk** (Alan Kay, Xerox PARC) | "todo es objeto y todo se manda mensajes": herencia, polimorfismo y la idea estética del paradigma |
| 1979–1983 | **C++** (Bjarne Stroustrup, Bell Labs) | "C con Clases" → C++ en 1983: la POO *híbrida*, con el rendimiento de C, que la volvió industrial |
| 1983–86 | **ABC** (Lambert Meertens, Steven Pemberton y **Guido van Rossum**, CWI, Países Bajos) | lenguaje educativo de sintaxis limpia y tipos de alto nivel — el **abuelo directo de Python** |
| 1989 | **Guido va a Python** (CWI, vacaciones de Navidad) | proyecto **personal**: harto de que ABC no fuera extensible, arma el suyo |
| **1991** | **Python 0.9.0** (20 de febrero) | primer release público: ya trae `class`, **herencia** y manejo de excepciones. **Cuatro años antes de Java** |
| 1994–2000 | **Python 1.0 y 2.0** | la POO madura: `lambda`/`map`/`filter`, listas por comprensión, recolector de basura, Unicode; y con 2.2 las *new-style classes* (`property`, `super()`) |
| 1995 | **Java** (Sun Microsystems, James Gosling) | POO **por decreto corporativo**: `private`, `interface`, `abstract`, recolector de basura. El "manual" con el que se compara a Python |
| **2008** | **Python 3.0** | ruptura técnica (el `print` pasa a ser función, Unicode nativo, división real) — **NO es el nacimiento de la POO en Python: ya llevaba 17 años** |

Mirá la tabla con dos lecturas. La de fechas: cuando Java salió al público en 1995, Python ya tenía cuatro años con clases, herencia y excepciones en la mochila. La de orígenes: Python no fue un producto de una compañía que decidió *vender* un lenguaje — fue una idea de una persona que, en un instituto de investigación, tomó lo mejor de un lenguaje educativo y lo hizo extensible.

> **Dato clave — quién era quién:** Guido van Rossum no escribió Python "contra" Java: trabajó entre 1983 y 1986 en el grupo que construyó ABC en el CWI, y Python nació en 1989 como proyecto propio dentro de ese mismo instituto. El rebelde no reaccionó contra el manual: **llegó antes que el manual**. Su POO no es una desviación de Java, es una línea paralela que parte de un abuelo compartido muy distinto.

Y queda un detalle que redondea la historia: la palabra *"objeto"* que usás todos los días desde el capítulo 3 — `[1, 2, 3]`, `"hola"`, `str` — no llegó por decreto. Llegó porque un grupo de investigadores noruegos de los años 60 necesitaban modelar barcos en simulaciones, y descubrieron que describir *cosas que se comportan* era más natural que describir *datos que se procesan*. De esa intuición salió todo lo que viene.

## 2. El abuelo directo: ABC (sucesor, no modificación)

Antes de los pilares, una aclaración que evitamos en la tabla a propósito porque tiene su propia sección. A menudo se dice por ahí que "Python es una modificación de ABC". Es casi cierto — y el matiz vale oro.

Guido van Rossum trabajó durante la primera mitad de los años 80 en el grupo de ABC en el CWI, un lenguaje diseñado para que gente sin experiencia en programación — científicos, por ejemplo — pudiera escribir programas rápido. De ABC, Python heredó detalles que hoy parecen "esencia" de Python:

- la **indentación como estructura**: en ABC, el sangrado ya agrupaba bloques de código;
- los **tipos de alto nivel**: listas, diccionarios y cadenas "fuera de la caja";
- y una filosofía de **sintaxis limpia y legible**, casi en lenguaje natural.

Pero Guido tenía quejas irreparables, y la peor era la **extensibilidad**: ABC no se podía extender, y esa fue una de sus mayores fallas. Traducido a hoy: no podías agregarle piezas. Entonces no fue una cuestión de retocar ABC — era imposible. Fue una decisión de arrancar de cero con sintaxis *como la de ABC* pero con un diseño extensible, como lo relata él mismo: "un lenguaje de scripting con una sintaxis como la de ABC, pero capaz de acceder a las llamadas al sistema".

> **Dato clave:** por eso decimos **sucesor**, no **modificación**. ABC es el abuelo que le prestó su cara a Python; Python es el nieto que aprendió de su fracaso. La extensibilidad — que ABC no tenía y que Python lleva en el ADN desde el día uno — es la misma razón por la que tu código puede crear clases, excepciones propias y tipos nuevos: eso está en el núcleo de la rebeldía que vas a ver en toda la Parte VI.

Esa historia de fondo explica por qué Python se siente como se siente. No es un lenguaje que copió el manual y lo rompió: es un lenguaje que nació en un ambiente académico, del lado de la *claridad para el humano*, y que tomó de la POO lo que le servía — sin los candados que la industria agregó después.

## 3. Los cuatro pilares clásicos de la POO

Con la historia servida, pasemos al plano teórico: ¿qué dice el manual sobre la POO? A lo largo de los años se consolidó una lista de **cuatro pilares** que casi todo texto de orientación a objetos repite con variaciones menores. Acá los presento con su forma más clásica — y con su ejemplo en *pseudo-Java*, el lenguaje que más se usa como "registro oficial" del paradigma — para que después veas, uno por uno, cómo los deconstruye Python.

> **Antes de seguir:** los bloques que siguen son **conceptuales** — dejan la idea de cada pilar en una forma legible, sin pretender ser código ejecutable. La sintaxis *real* de Python para todo esto llega en los capítulos 12 al 17, uno por uno.

### 3.1 Encapsulamiento — juntar dato y comportamiento, y proteger el estado

La primera regla del manual: los datos y los métodos que los manipulan viven **juntos** en el mismo objeto, y el mundo exterior solo accede a través de puertas controladas. En el registro oficial de Java eso se traduce en marcar lo interno como `private` y exponer acceso a través de métodos:

```java
public class Cuenta {
    private double saldo;           // invisible desde afuera

    public void depositar(double monto) {
        if (monto > 0) {
            this.saldo += monto;
        }
    }

    public double verSaldo() {
        return this.saldo;
    }
}
```

En el manual, `private` no es un pedido: es **un decreto**. El compilador te *impide* leer `saldo` desde afuera; está prohibido por ley del lenguaje. La idea detrás es hermosa (proteger el estado de que quede en un valor absurdo), pero la implementación es policial.

### 3.2 Herencia — "es un", para reutilizar entre moldes

El segundo pilar: una clase puede *nacer de otra* y heredar sus atributos y métodos. El ejemplo clásico del registro:

```java
public class Perro extends Animal {
    // Perro hereda de Animal todo lo que Animal sabe hacer
    // y puede agregar o modificar lo suyo
}
```

El manual reserva esto para relaciones de "es un": `Perro` *es un* `Animal`. Y — matiz que te va a perseguir — el manual suele permitir **una sola** herencia: una clase hereda de una sola clase madre, ni más. La jerarquía es un árbol con una raíz, no una red.

### 3.3 Polimorfismo — la misma cara, muchos comportamientos

El tercer pilar es el más elegante y a la vez el más conflictivo. Una variable declarada con el tipo del padre puede guardar un objeto de cualquier hijo, y el comportamiento que se dispara depende del objeto real:

```java
Animal a = new Perro();
a.mover();    // "el perro camina"

Animal b = new Pato();
b.mover();    // "el pato vuela o nada"
```

La misma llamada `a.mover()`, con tipos declarados idénticos, produce resultados distintos. El manual lo logra con **interfaces y sobreescritura**: cada clase declara qué significa `mover()` para ella, y el runtime decide cuál usar. Es la promesa que hace que "los objetos se manden mensajes" y cada uno responda a su manera.

### 3.4 Abstracción — mostrar el contrato, esconder el andamiaje

El cuarto pilar es el que más se confunde con la herencia. La **abstracción** es separar *qué* hace una cosa de *cómo* lo hace: definir un contrato y dejar la implementación para después. En el manual se escribe con clases y métodos abstractos:

```java
public abstract class Figura {
    public abstract double area();   // promesa sin cuerpo
}

public class Cuadrado extends Figura {
    public double area() {           // la promesa se cumple acá
        return lado * lado;
    }
}
```

La clave del manual: una clase abstracta **no se puede instanciar**. Si `Figura` tiene métodos abstractos pendientes, el compilador te prohíbe crear una `Figura` pelada: tenés que honorar cada promesa. Es la forma más dura de control que tiene el paradigma — y, te adelanto, *justamente* la que Python convierte en opcional.

## 4. "Cómo lo hace Python": el contraste de frente

Puesto todo en la mesa, la gran comparación. Esta tabla resume cada pilar del manual, su herramienta en Java/C#, y la promesa de lo que vas a ver en los próximos capítulos:

| Pilar | El manual (Java/C#) | Python |
|-------|--------------------|--------|
| **Encapsulamiento** | `private`/`protected`/`public` — el compilador prohíbe tocar lo privado | el **`_`** y el **`__`** son convención, no candado; `@property` para vigilar (capítulo 14) |
| **Herencia** | `extends` con herencia **simple** | `class Perro(Animal)` — y hasta herencia **múltiple** (capítulo 15) |
| **Polimorfismo** | `interface` + sobreescritura declarada | **duck typing**: si camina y suena como pato, es un pato (capítulo 16) |
| **Abstracción** | `abstract class` obligatoria, no instanciable | `ABC` y `@abstractmethod` existen... pero **opcionales** (capítulo 16) |

Mirá la columna de la izquierda y la de la derecha: son *la misma arquitectura* — dato y comportamiento juntos, "es un", misma cara con muchos comportamientos, contrato separado del andamiaje. Python toma **los cuatro pilares**, pero cambia la herramienta de control. Donde Java grita `private` con un compilador que te corta la mano, Python te dice "somos adultos: no toques saldo, y si lo tocás, es problema tuyo".

> **Dato clave — el titular de esta tabla:** los pilares no desaparecen en Python. Lo que desaparece es la *policía*: los pilares pasan de ser **obligaciones** a ser **convenciones**. El capítulo 14 arranca con esa misma pregunta que te va a perseguir ("si no hay privados, ¿qué es `__`?"), y el capítulo 15 con el `class Perro(Animal)` que ya estás esperando.

## 5. El tono de Python: "somos adultos"

Llegamos al corazón de la parte. Todo lo que viste en las dos secciones anteriores — la historia y los cuatro pilares — tiene una utilidad doble: formar tu intuición y preparar el contraste.

La tesis de la Parte VI, la que vas a escuchar repetida hasta el cansancio, es esta: **Python hace POO sin rejas**. En Java o C# te *obligan*: `private` para esconder, `final` para congelar, `interface` para prometer, clases abstractas que prohíben instanciar. Python entrega las mismas herramientas — clases, herencia, métodos mágicos, propiedades, clases abstractas, protocolos — y te deja la responsabilidad al lado. Las reglas existen, pero son **reglas de la casa**: las respetás porque sabés por qué valen la pena, no porque un compilador te amenaza.

Eso no es descuido. Es una elección de diseño que baja por dos bisabuelos: la claridad de ABC (el lenguaje de los que *empezaban*) y la extensibilidad que Guido echó de menos. Cuando un lenguaje te trata como adulto, el poder de *hacer daño* también es tuyo — y por eso los próximos capítulos no solo te enseñan qué se puede hacer, sino cuándo conviene y cuándo no.

En el capítulo 12, la primera parada, vas a abrir la caja con *"la lista de los rebeldes"*: las once particularidades que hacen a Python distinto dentro del mundo orientado a objetos, y el mapa de dónde se descompone cada pilar del manual. Ahora que sabés de dónde viene el paradigma y cuáles son sus reglas clásicas, esa lista va a pegar donde tiene que pegar: no como rarezas sin motivo, sino como las decisiones de un lenguaje que eligió confiar en vos.

> **Dato clave — recordá la frase:** *por convención, no por decreto.* Es el aforismo de los seis capítulos que vienen. Cada vez que veas un `_`, un `__`, un decorador, un `ABC` o un mixin, vas a estar viendo una regla que Python prefirió *pedirte* antes que *imponerte*.

## 6. Resumen y conceptos clave

- [ ] La POO no nació con Python: arrancó con **Simula 67** (1967) y su idea de clases y objetos para modelar simulaciones.
- [ ] **Smalltalk** (Alan Kay, Xerox PARC) fijó la estética: todo es objeto, todo se manda mensajes.
- [ ] **C++** (1979–1983) la hizo industrial: la POO híbrida con el rendimiento de C.
- [ ] **Python nació antes que Java**: 0.9.0 en febrero de 1991, ya con `class`, herencia y excepciones — cuatro años antes del Java público de 1995.
- [ ] Python no nació por decreto corporativo: fue un **proyecto personal** de Guido van Rossum en el CWI (1989), heredero de **ABC**.
- [ ] Python es **sucesor de ABC, no una modificación**: tomó su sintaxis limpia y sus tipos de alto nivel, y creó algo nuevo porque ABC no era extensible.
- [ ] Python 3.0 (2008) fue una ruptura *técnica*, no el nacimiento de su POO: para entonces llevaba 17 años.
- [ ] Los **cuatro pilares clásicos**: encapsulamiento (dato+comportamiento, proteger estado), herencia ("es un"), polimorfismo (misma cara, muchos comportamientos) y abstracción (contrato separado del andamiaje).
- [ ] En el manual los pilares son **decretos** del compilador: `private`, `extends` simple, `interface`, `abstract` no instanciable.
- [ ] Python conserva **los cuatro pilares** y cambia la herramienta de control: convención (`_`/`__`), herencia múltiple, duck typing y `ABC` opcional.
- [ ] El aforismo de la Parte VI: **"por convención, no por decreto"** — el rebelde confía en vos.

---