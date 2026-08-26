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