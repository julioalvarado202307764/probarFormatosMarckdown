# Manual Técnico - Compilador BattleScript

## 1. Introducción y Objetivo
El presente manual técnico describe la arquitectura, herramientas y funcionamiento interno del IDE y Compilador para el lenguaje **BattleScript**. Este proyecto implementa un analizador léxico y un analizador sintáctico capaces de reconocer y validar estrategias de combate, configuración de partidas y ejecución de simulaciones, y además un **motor de ejecución** que interpreta el árbol generado por el parser y simula los combates por turnos descritos en el archivo `.btl`, aplicando reglas condicionales, cálculo de daño, sistema de recursos, puntuación y condiciones de victoria, con soporte para reproducibilidad determinista mediante semillas (`seed`). Todo esto se expone a través de reportes visuales detallados en la interfaz gráfica.

## 2. Tecnologías y Herramientas Empleadas
Para garantizar la robustez, portabilidad y facilidad de mantenimiento, el proyecto fue desarrollado utilizando el siguiente *stack* tecnológico:

*   **Lenguaje de Programación:** Java (JDK 8+). Elegido por su fuerte tipado, manejo de concurrencia nativo (útil para Swing) y su compatibilidad estándar con herramientas de análisis.
*   **Analizador Léxico:** JFlex (Generador de escáneres léxicos para Java).
*   **Analizador Sintáctico:** Java CUP (Generador de parsers LALR para Java).
*   **Interfaz Gráfica:** Java Swing y AWT (Librerías nativas para desarrollo de aplicaciones de escritorio).
*   **Entorno de Desarrollo (IDE):** Apache NetBeans.

## 3. Estructura General del Proyecto
El proyecto sigue un patrón de tres capas: análisis (Frontend del compilador), modelo de ejecución (Backend/Motor) e interfaz gráfica (GUI).

```text
📁 src/
 ├── 📁 proyecto1/                     (Paquete raíz)
 │    └── 📄 Proyecto1.java            (Punto de entrada de la aplicación)
 │
 ├── 📁 proyecto1.analizadores/        (Motor de Análisis)
 │    ├── 📄 Lexer.jflex               (Archivo fuente léxico)
 │    ├── 📄 Parser.cup                (Archivo fuente sintáctico, con acciones semánticas
 │    │                                 que construyen los objetos del paquete proyecto1.ast)
 │    ├── 📄 Generador.java            (Script automatizado de compilación)
 │    ├── 📄 TokenInfo.java            (Modelo de datos para Tokens)
 │    ├── 📄 ErrorInfo.java            (Modelo de datos para Errores)
 │    └── 📄 Lexer.java, Parser.java, sym.java (Clases autogeneradas)
 │
 ├── 📁 proyecto1.ast/                 (Modelo de dominio + Motor de Ejecución)
 │    ├── 📄 Accion.java               (Enum de las 10 acciones del lenguaje)
 │    ├── 📄 TipoJugador.java          (Enum MAGE / WARRIOR)
 │    ├── 📄 Estrategia.java           (Nombre, tipo, acción inicial, reglas, acción por defecto)
 │    ├── 📄 Regla.java                (Condición + acción de una regla if/then)
 │    ├── 📄 Partida.java              (Jugadores, rounds, scoring, bonuses de un match)
 │    ├── 📄 EjecucionRun.java         (Agrupa una lista de partidas con SU propia seed)
 │    ├── 📄 Combatiente.java          (Estado en tiempo de ejecución: vida, recurso, historial...)
 │    ├── 📄 ContextoEjecucion.java    (Snapshot de estado que reciben las condiciones al evaluarse)
 │    ├── 📄 Condicion.java            (Interfaz: todo nodo evaluable de una condición)
 │    ├── 📄 Expresion.java            (Interfaz: todo nodo evaluable de una expresión/operando)
 │    ├── 📄 OperacionLogica.java      (AND / OR / NOT, con corto-circuito real)
 │    ├── 📄 OperacionRelacional.java  (==, !=, >, <, >=, <=)
 │    ├── 📄 Variable.java             (self_health, opponent_score, random, etc.)
 │    ├── 📄 Literal.java              (Valores constantes: enteros, decimales, acciones, listas)
 │    ├── 📄 FuncionHistorial.java     (get_move, last_move, get_moves_count, get_last_n_moves)
 │    └── 📄 MotorSimulacion.java      (Orquesta la simulación completa de una partida)
 │
 └── 📁 proyecto1.interfaz/            (Interfaz Gráfica)
      └── 📄 VentanaPrincipal.java     (Lógica y maquetación visual del IDE)
```

## 4. Clases y Métodos Principales

### `VentanaPrincipal.java`
Gestor principal del entorno visual. Hereda de `JFrame`.

* `inicializarComponentes()`: Construye dinámicamente el árbol de componentes de Swing (Barra de menús, JTabbedPane, JTables, JScrollPanes) y configura los layouts responsivos.
* `ejecutarAnalisis()`: Método puente que extrae el texto del área de código, instancia los analizadores (léxico y sintáctico), atrapa excepciones generales para mantener viva la aplicación (tolerancia a fallos), y, si el análisis fue exitoso, itera sobre `parser.listaEjecuciones` instanciando un `MotorSimulacion` por cada bloque `run` del archivo (ver sección 5.3) y concatenando sus reportes en la consola de salida.
* `abrirArchivo()` / `guardarArchivo()`: Implementan `JFileChooser` manejando flujos de I/O mediante `BufferedReader` y `FileWriter`.

### Modelos de Datos (`TokenInfo` y `ErrorInfo`)
Clases tipo POJO (*Plain Old Java Object*) que actúan como estructuras de almacenamiento. Permiten desacoplar la generación de datos en los analizadores de su renderizado en las tablas dinámicas de Swing.

### Modelo de dominio (`proyecto1.ast`)

* **`Accion`**: enum con las 10 acciones del lenguaje (`ARCANE_BOLT`, `FIREBALL`, `MAGIC_BARRIER`, `HEALING_RUNE`, `MEDITATE`, `SLASH`, `HEAVY_STRIKE`, `SHIELD_BLOCK`, `WAR_CRY`, `REST`). Es el vocabulario compartido entre el parser, las estrategias y el motor.
* **`TipoJugador`**: enum `MAGE` / `WARRIOR`, usado para inicializar las estadísticas base de cada `Combatiente` según la tabla de la especificación (vida, recurso, ataque físico, poder mágico, armadura, resistencia mágica, velocidad).
* **`Estrategia`**: representa el bloque `mage`/`warrior` del `.btl` ya parseado: nombre, `TipoJugador`, acción inicial y una lista ordenada de `Regla` (donde la última siempre es el `else`).
* **`Regla`**: par (`Condicion`, `Accion`) que representa una línea `if <condición> then <acción>` (o la regla `else`, cuya condición es `null`/siempre verdadera).
* **`Partida`**: representa el bloque `match`: nombres de los dos jugadores, número de rondas, y los valores de `scoring` (`damage_point`, `healing_point`, `successful_defense`, `victory_bonus`, `failed_action_penalty`) y `bonuses` (combos de mago/guerrero y sus puntos, `low_health_victory`).
* **`EjecucionRun`**: representa **un solo** bloque `run [...] with { seed: N }`. Guarda la lista de nombres de partidas a ejecutar y la semilla que le corresponde **a ese bloque específico** — esto es lo que permite que distintos `run` dentro de un mismo `main` usen semillas independientes sin pisarse entre sí (ver sección 5.3).
* **`Combatiente`**: estado *runtime* de un jugador durante una partida concreta (no confundir con `Estrategia`, que es la definición estática). Contiene vida actual/máxima, recurso actual/máximo, ataque físico, poder mágico, armadura, resistencia mágica, velocidad, puntuación acumulada, buff pendiente de `WAR_CRY`, y el historial de acciones ejecutadas **exitosamente**.
* **`ContextoEjecucion`**: objeto inmutable que se le entrega a una `Condicion` para que pueda evaluarse — contiene referencias a `self` y `opponent` (como `Combatiente`), el número de ronda actual, el total de rondas, y el valor de `random` ya generado para ese jugador en esa ronda (ver sección 5.2).
* **`Condicion`** (interfaz) y sus implementaciones:
  * `OperacionLogica`: nodos `AND`/`OR`/`NOT`. Implementa corto-circuito real usando los operadores nativos `&&`/`||` de Java.
  * `OperacionRelacional`: nodos de comparación (`==`, `!=`, `>`, `<`, `>=`, `<=`). Si ambos operandos son numéricos, compara por valor; si no (por ejemplo, comparando dos `List<Accion>` o dos `Accion`), delega en `.equals()`.
* **`Expresion`** (interfaz) y sus implementaciones — representan el lado izquierdo/derecho de una comparación:
  * `Variable`: variables reservadas del lenguaje (`self_health`, `opponent_score`, `round_number`, `total_rounds`, `random`, etc.), resueltas contra el `ContextoEjecucion` en tiempo de evaluación.
  * `Literal`: valores constantes — enteros, decimales, una `Accion` (ej. `FIREBALL` suelto en una condición), o una `List<Accion>` (una secuencia literal entre corchetes, ej. `[SLASH, SLASH, HEAVY_STRIKE]`).
  * `FuncionHistorial`: implementa `get_move`, `last_move`, `get_moves_count` y `get_last_n_moves`, incluyendo la variante anidada `get_moves_count(get_last_n_moves(...), ACCION)`.
* **`MotorSimulacion`**: la clase central del motor. Recibe la lista de `Estrategia`, la lista de `Partida`, los nombres de las partidas a correr y una semilla, y expone `ejecutarSimulacion()`, que devuelve el reporte de texto completo de la simulación (ver sección 5).

## 5. Funcionamiento del Motor de Análisis

### 5.1 Analizador Léxico (JFlex)
El archivo `Lexer.jflex` se encarga de leer la cadena de caracteres de entrada y agruparlos en unidades con significado (Tokens).

* **Gestión de Estados:** Utiliza estados exclusivos como `<MULTILINE_COMMENT>` para manejar la ignorancia de bloques de texto que no deben ser analizados. Este estado incluye una regla `<<EOF>>` que detecta si el archivo termina sin que el comentario se haya cerrado, registrando un error léxico fatal y deteniendo el análisis.
* **Interceptación:** En lugar de retornar directamente los símbolos a CUP, utiliza un método propio `token(int tipoSym, String nombreTipo, Object valor)` que inyecta cada token válido en una lista dinámica (`listaTokens`) antes de enviarlo al parser.
* **Recuperación de Errores:** Posee una regla "catch-all" `[^]` al final del estado `<YYINITIAL>` que captura caracteres no reconocidos, agregándolos a `listaErrores` en lugar de detener la ejecución.

### 5.2 Analizador Sintáctico (CUP)
El archivo `Parser.cup` valida que la secuencia de tokens provista por el Lexer forme estructuras gramaticales válidas según el lenguaje BattleScript, y además **construye el árbol de objetos del paquete `proyecto1.ast`** que el motor de ejecución va a interpretar — es decir, las acciones semánticas (`{: ... :}`) de cada producción no solo validan sintaxis, sino que instancian `Estrategia`, `Regla`, `Condicion`, `Partida`, `EjecucionRun`, etc.

* **Resolución de Ambigüedades:** Emplea declaraciones de precedencia (`precedence left OR, AND...`) para garantizar la construcción correcta de expresiones lógicas y matemáticas.
* **Reglas Gramaticales:** Construidas en formato BNF subyacente. Define estructuras jerárquicas complejas como `estrategia`, `partida` y el bloque `main`.
* **Funciones de historial anidadas y secuencias literales:** la gramática admite tanto `get_moves_count(historial, ACCION)` como su forma anidada `get_moves_count(get_last_n_moves(historial, n), ACCION)` (dos producciones distintas para `funcion_historial`, distinguibles en LALR(1) porque no comparten token inicial), y admite listas literales de acciones (`[SLASH, SLASH, HEAVY_STRIKE]`) como un `operando` válido dentro de cualquier condición, no solo dentro de la definición de combos.
* **Múltiples `run` con semillas independientes:** cada `instruccion_run` del bloque `main` construye su propio objeto `EjecucionRun` (lista de partidas + semilla) y lo agrega a `listaEjecuciones`, en vez de acumular todo en una sola lista global — esto es lo que permite que dos `run` con distinta `seed` no se pisen entre sí.
* **Manejo de Pánico:** Se sobreescribieron los métodos nativos `syntax_error(Symbol s)` y `unrecovered_syntax_error(Symbol s)` para interceptar tokens inesperados y almacenarlos en `listaErroresSintacticos`, permitiendo reportarlos en la interfaz gráfica.

### 5.3 Motor de Ejecución (`proyecto1.ast.MotorSimulacion`)

Una vez que el análisis léxico y sintáctico terminan sin errores, `VentanaPrincipal` recorre `parser.listaEjecuciones` y, **por cada `EjecucionRun`**, crea una instancia nueva de `MotorSimulacion` con la semilla y la lista de partidas de ese bloque específico, llamando a `ejecutarSimulacion()`. Esto garantiza que cada `run [...] with { seed: N }` del archivo se ejecute de forma aislada, con su propia semilla, sin compartir estado con los demás.

Dentro de `ejecutarSimulacion()`, por cada partida a correr:

1. **Inicialización de combatientes:** se crean dos objetos `Combatiente` a partir de las `Estrategia` de los dos `players`, con las estadísticas base según su `TipoJugador`.
2. **Generadores de semilla independientes:** se instancian dos `java.util.Random`, uno con la semilla exacta (jugador 1) y otro con `semilla + 1` (jugador 2), tal como exige la especificación para garantizar reproducibilidad determinista.
3. **Bucle de rondas (`round_number` inicia en 0):** en cada ronda:
   * Se genera **un** valor `double` (rango `[0.0, 1.0)`) por jugador, usando el generador correspondiente, y se guarda en el `ContextoEjecucion` de esa ronda — se genera exactamente una vez por jugador por ronda, se use o no la variable `random` en las reglas, y toda referencia a `random` dentro de esa ronda reutiliza el mismo valor cacheado.
   * La ronda 0 siempre usa la `accion inicial` de cada estrategia (sin evaluar reglas); a partir de la ronda 1 se evalúan las reglas en cascada, usando el `else` si ninguna condición fue verdadera.
   * Se cobra el costo en recurso (maná/energía) de la acción elegida por cada jugador; si no alcanza, la acción falla: no produce ningún efecto, no se agrega al historial, y se aplica `failed_action_penalty` (con la puntuación nunca por debajo de cero).
   * **Orden de resolución por prioridad:** se determina qué jugador actúa primero según la tabla de prioridades de cada acción, con empate resuelto por velocidad y, si también empatan, por la posición en `players`. El actor que va primero resuelve completamente su acción (mejora, recuperación, curación, o ataque con el cálculo de daño real según sus propias estadísticas contra la armadura/resistencia del rival, incluyendo el bono de `WAR_CRY` si estaba pendiente, y aplicando la reducción del 50% si el rival se está defendiendo); si esa acción deja al rival con vida ≤ 0, el segundo jugador **no llega a actuar** esa ronda.
   * Se verifica si algún combatiente completó la secuencia de combo (`mage_combo`/`warrior_combo`) con sus últimas acciones exitosas, otorgando los puntos de combo correspondientes.
   * La puntuación por daño y por curación se calcula sobre el **valor real** aplicado (`daño_real × damage_point`, `vida_recuperada_real × healing_point`), no sobre un valor fijo — incluyendo el tope de curación al llegar a la vida máxima.
   * Si algún combatiente llega a vida ≤ 0, la partida termina de inmediato (derrota directa).
4. **Determinación del ganador:** si la partida terminó por derrota directa, gana automáticamente el sobreviviente, **sin importar la puntuación acumulada**. Si la partida llegó al límite de rondas con ambos combatientes vivos, se aplica la cadena de desempate: mayor puntuación → mayor vida restante → mayor recurso restante → empate técnico. Al ganador se le suma `victory_bonus`, y adicionalmente `low_health_victory` si su vida final quedó igual o por debajo del 25% de **su propia** vida máxima (25 para Mago, 35 para Guerrero).
5. **Reporte:** todo el detalle (acción elegida por ronda, mensajes de combo, advertencias de recurso insuficiente, resultado final) se acumula en un `StringBuilder` (bitácora) que se devuelve como texto y se muestra en la consola de salida del IDE.

### 5.4 Flujo de Ejecución Completo

1. **Lectura:** El código fuente en texto plano se envuelve en un `StringReader`.
2. **Tokenización:** El objeto `Lexer` procesa el texto bajo demanda de CUP, llenando silenciosamente las listas de Tokens e Errores Léxicos.
3. **Parseo + construcción del AST:** El objeto `Parser` solicita tokens al Lexer, arma el árbol de validación y, en el mismo proceso, construye las `Estrategia`, `Partida` y `EjecucionRun` que necesitará el motor; también atrapa fallos sintácticos en su lista de Errores Sintácticos.
4. **Simulación:** Si el parseo fue exitoso, se recorre `parser.listaEjecuciones` instanciando y ejecutando un `MotorSimulacion` por cada bloque `run`, en el mismo orden en que aparecen en el archivo.
5. **Reporte:** La interfaz itera sobre las listas de tokens y errores (independientemente de si el parser o el motor lanzaron un error fatal interno encapsulado en el `try-catch` del hilo de Swing) y puebla el modelo de las tablas visuales usando `DefaultTableModel.addRow()`, además de mostrar la bitácora de la simulación en la consola de salida.

## 6. Guía de Mantenimiento Futuro

Para agregar nuevas palabras reservadas, funciones u operadores al lenguaje:

1. **Modificar exclusivamente** los archivos fuente `Lexer.jflex` (para reconocer el nuevo token) y `Parser.cup` (para integrarlo a la gramática y, si aplica, a las acciones semánticas que construyen el AST en `proyecto1.ast`).
2. **Jamás** modificar manualmente los archivos `Lexer.java`, `Parser.java` o `sym.java`.
3. Ejecutar la clase `Generador.java` para que reescriba los analizadores.
4. Correr la aplicación desde `Proyecto1.java` para probar los cambios en el IDE.

Para agregar una **nueva acción de combate** (ej. una nueva habilidad):

1. Agregar el valor al enum `Accion.java`.
2. Agregar su costo de recurso y prioridad en las tablas correspondientes de `MotorSimulacion.java` (método de cobro de recurso y método de prioridad).
3. Si es una acción ofensiva o de curación, agregar su poder base y, si corresponde, su fórmula de daño (física o mágica) en el método de resolución de acciones individuales.
4. Actualizar el token correspondiente en `Lexer.jflex` si el nombre de la acción es nuevo.

Para agregar una **nueva variable o función reservada** consultable desde una condición (similar a `self_health` o `get_move`):

1. Agregar el token en `Lexer.jflex` y la producción en `Parser.cup` (dentro de `operando` o `funcion_historial`, según corresponda).
2. Agregar el caso correspondiente en `Variable.java` o `FuncionHistorial.java` para que sepa resolver su valor contra el `ContextoEjecucion`.
