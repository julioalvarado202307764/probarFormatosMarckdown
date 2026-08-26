# Manual Técnico - Compilador BattleScript

## 1. Introducción y Objetivo
El presente manual técnico describe la arquitectura, herramientas y funcionamiento interno del IDE y Compilador para el lenguaje **BattleScript**. Este proyecto consiste en la implementación de un analizador léxico y un analizador sintáctico capaces de reconocer y validar estrategias de combate, configuración de partidas y ejecución de simulaciones, brindando reportes visuales detallados.

## 2. Tecnologías y Herramientas Empleadas
Para garantizar la robustez, portabilidad y facilidad de mantenimiento, el proyecto fue desarrollado utilizando el siguiente *stack* tecnológico:

*   **Lenguaje de Programación:** Java (JDK 8+). Elegido por su fuerte tipado, manejo de concurrencia nativo (útil para Swing) y su compatibilidad estándar con herramientas de análisis.
*   **Analizador Léxico:** JFlex (Generador de escáneres léxicos para Java).
*   **Analizador Sintáctico:** Java CUP (Generador de parsers LALR para Java).
*   **Interfaz Gráfica:** Java Swing y AWT (Librerías nativas para desarrollo de aplicaciones de escritorio).
*   **Entorno de Desarrollo (IDE):** Apache NetBeans.

## 3. Estructura General del Proyecto
El proyecto sigue el patrón de diseño arquitectónico que separa la interfaz gráfica de usuario (Frontend/GUI) de la lógica de análisis computacional (Backend/Motor).

```text
📁 src/
 ├── 📁 proyecto1/                     (Paquete raíz)
 │    └── 📄 Proyecto1.java            (Punto de entrada de la aplicación)
 │
 ├── 📁 proyecto1.analizadores/        (Motor de Análisis)
 │    ├── 📄 Lexer.jflex               (Archivo fuente léxico)
 │    ├── 📄 Parser.cup                (Archivo fuente sintáctico)
 │    ├── 📄 Generador.java            (Script automatizado de compilación)
 │    ├── 📄 TokenInfo.java            (Modelo de datos para Tokens)
 │    ├── 📄 ErrorInfo.java            (Modelo de datos para Errores)
 │    └── 📄 Lexer.java, Parser.java, sym.java (Clases autogeneradas)
 │
 └── 📁 proyecto1.interfaz/            (Interfaz Gráfica)
      └── 📄 VentanaPrincipal.java     (Lógica y maquetación visual del IDE)
```
## 4. Clases y Métodos Principales

### `VentanaPrincipal.java`
Gestor principal del entorno visual. Hereda de `JFrame`.

* `inicializarComponentes()`: Construye dinámicamente el árbol de componentes de Swing (Barra de menús, JTabbedPane, JTables, JScrollPanes) y configura los layouts responsivos.
* `ejecutarAnalisis()`: Método puente que extrae el texto del área de código, instancia los analizadores y atrapa excepciones generales para mantener viva la aplicación (tolerancia a fallos).
* `abrirArchivo()` / `guardarArchivo()`: Implementan `JFileChooser` manejando flujos de I/O mediante `BufferedReader` y `FileWriter`.

### Modelos de Datos (`TokenInfo` y `ErrorInfo`)
Clases tipo POJO (*Plain Old Java Object*) que actúan como estructuras de almacenamiento. Permiten desacoplar la generación de datos en los analizadores de su renderizado en las tablas dinámicas de Swing.

## 5. Funcionamiento del Motor de Análisis

### Analizador Léxico (JFlex)
El archivo `Lexer.jflex` se encarga de leer la cadena de caracteres de entrada y agruparlos en unidades con significado (Tokens).

* **Gestión de Estados:** Utiliza estados exclusivos como `<MULTILINE_COMMENT>` para manejar la ignorancia de bloques de texto que no deben ser analizados.
* **Interceptación:** En lugar de retornar directamente los símbolos a CUP, utiliza un método propio `token(int tipoSym, String nombreTipo, Object valor)` que inyecta cada token válido en una lista dinámica (`listaTokens`) antes de enviarlo al parser.
* **Recuperación de Errores:** Posee una regla "catch-all" `[^]` al final del estado `<YYINITIAL>` que captura caracteres no reconocidos, agregándolos a `listaErrores` en lugar de detener la ejecución.

---

### Analizador Sintáctico (CUP)
El archivo `Parser.cup` valida que la secuencia de tokens provista por el Lexer forme estructuras gramaticales válidas según el lenguaje BattleScript.

* **Resolución de Ambigüedades:** Emplea declaraciones de precedencia (`precedence left OR, AND...`) para garantizar la construcción correcta de expresiones lógicas y matemáticas.
* **Reglas Gramaticales:** Construidas en formato BNF subyacente. Define estructuras jerárquicas complejas como `estrategia`, `partida` y el bloque `main`.
* **Manejo de Pánico:** Se sobreescribieron los métodos nativos `syntax_error(Symbol s)` y `unrecovered_syntax_error(Symbol s)` para interceptar tokens inesperados y almacenarlos en `listaErroresSintacticos`, permitiendo reportarlos en la interfaz gráfica.

---

### Flujo de Ejecución del Motor

1. **Lectura:** El código fuente en texto plano se envuelve en un `StringReader`.
2. **Tokenización:** El objeto `Lexer` procesa el texto bajo demanda de CUP, llenando silenciosamente las listas de Tokens e Errores Léxicos.
3. **Parseo:** El objeto `Parser` solicita tokens al Lexer, arma el árbol de validación y atrapa fallos sintácticos en su lista de Errores Sintácticos.
4. **Reporte:** La interfaz iterará sobre estas listas (independientemente de si el parser lanzó un error fatal interno encapsulado en el `try-catch` del hilo de Swing) y poblará el modelo de las tablas visuales usando `DefaultTableModel.addRow()`.

## 6. Guía de Mantenimiento Futuro

Para agregar nuevas palabras reservadas, funciones u operadores al lenguaje:

1. **Modificar exclusivamente** los archivos fuente `Lexer.jflex` (para reconocer el nuevo token) y `Parser.cup` (para integrarlo a la gramática).
2. **Jamás** modificar manualmente los archivos `Lexer.java`, `Parser.java` o `sym.java`.
3. Ejecutar la clase `Generador.java` para que reescriba los analizadores.
4. Correr la aplicación desde `Proyecto1.java` para probar los cambios en el IDE.












