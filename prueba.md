# 📖 MANUAL TÉCNICO - CINEMA USAC (Fase 1)

**Proyecto:** Sistema de Gestión de Cine - Fase 1

**Lenguaje:** C++ (Estándar C++11 o superior)

**Framework GUI:** Qt / Qt Creator

**Generación de Reportes:** Graphviz (Lenguaje DOT)


## 1. Introducción

El presente manual documenta la arquitectura interna y las decisiones de diseño del sistema **Cinema USAC (Fase 1)**. El sistema fue desarrollado utilizando un paradigma de Programación Orientada a Objetos (POO) en C++ y gestionando la memoria dinámica de forma manual mediante punteros, absteniéndose estrictamente del uso de librerías de contenedores estándar (como `std::vector` o `std::list`) para cumplir con las restricciones del proyecto.

La interfaz de usuario fue construida utilizando el framework **Qt**, empleando un diseño de ventana única iterativa mediante `QStackedWidget` y `QTabWidget` para optimizar el rendimiento y la administración de los punteros en memoria.


## 2. Implementación de Estructuras de Datos

El sistema se fundamenta en 4 estructuras de datos dinámicas principales, cada una resolviendo un problema específico del dominio del cine.

### 2.1. Árbol Binario de Búsqueda (Cartelera de Películas)

**Clases principales:** `NodoBST`, `Pelicula`, `ArbolPeliculas`.

*   **Descripción:** Estructura jerárquica no lineal donde cada nodo tiene un máximo de dos hijos. El hijo izquierdo contiene un valor menor que el padre, y el derecho un valor mayor.
*   **Uso en el sistema:** Almacena la cartelera de películas cargada desde el archivo `.csv`. El criterio de ordenamiento es el **Código de la Película** (alfanumérico).
*   **Decisión de Diseño:** Se eligió un ABB (Árbol Binario de Búsqueda) porque reduce el tiempo de búsqueda de $O(n)$ a $O(\log n)$ en casos promedio. Esto es crucial cuando el cliente busca una película por su código desde la taquilla. Se implementó un recorrido **InOrden** recursivo para poblar las tablas visuales de la interfaz visual (`QTableWidget`), garantizando que la cartelera se muestre siempre ordenada de forma ascendente.

### 2.2. Matriz Dispersa Ortogonal (Gestión de Asientos y Reservas)

**Clases principales:** `NodoMatriz`, `MatrizDispersa`.

*   **Descripción:** Estructura multidimensional de listas enlazadas ortogonales (punteros en 4 direcciones: arriba, abajo, izquierda, derecha) con nodos de cabecera independientes para filas y columnas.
*   **Uso en el sistema:** Representa la sala de cine y la disponibilidad de asientos. Cada asiento reservado es un nodo activo en la memoria.
*   **Decisión de Diseño:** Representar una sala de cine de, por ejemplo, 100x100 usando una matriz bidimensional tradicional desperdiciaría memoria si solo hay 5 reservas. La Matriz Dispersa resuelve esto instanciando *únicamente* los nodos que representan asientos ocupados y sus respectivas cabeceras de fila/columna.
*   **Algoritmo de Inserción:** 1. Se valida si la cabecera (Fila o Columna) existe; de lo contrario, se crea y se ordena.
    2. Se inserta el nuevo nodo y se entrelaza ortogonalmente con los nodos adyacentes más cercanos validando colisiones (asientos ya ocupados).

### 2.3. Lista Circular Doblemente Enlazada (Solicitudes Especiales)

**Clases principales:** `NodoSolicitud`, `ListaSolicitudes`.

*   **Descripción:** Secuencia lineal de nodos donde cada uno apunta a su antecesor y sucesor, con la particularidad de que el último nodo se enlaza nuevamente al primero, formando un ciclo cerrado.
*   **Uso en el sistema:** Gestiona la cola de quejas, sugerencias y solicitudes de los clientes.
*   **Decisión de Diseño:** El doble enlace permite al administrador tener mayor flexibilidad algorítmica para desenlazar (eliminar/rechazar) un nodo específico en $O(1)$ una vez encontrado, al tener acceso directo al nodo anterior sin necesidad de reiniciar la búsqueda. La naturaleza circular permite que el sistema siempre encuentre el inicio desde cualquier punto de la estructura.

### 2.4 Lista Circular Simple de Listas Dobles (Promociones y Beneficios)

**Clases principales:** `NodoPromocion`, `NodoBeneficio`, `ListaPromociones`.

*   **Descripción:** Una estructura compuesta. El nivel principal (Promociones) es una Lista Circular Simple, y cada nodo de este nivel contiene punteros que inician y administran una sub-lista interna doblemente enlazada (Beneficios).
*   **Uso en el sistema:** El administrador crea una promoción general y le "cuelga" una serie de beneficios específicos.
*   **Decisión de Diseño:** Modelar la relación 1 a N (1 Promoción -> N Beneficios) utilizando estructuras anidadas proporciona un encapsulamiento perfecto. En lugar de crear un árbol n-ario, se optó por la lista de listas ya que las promociones siguen un flujo secuencial (se leen una tras otra). En la GUI, esto se tradujo directamente a un `QTreeWidget` para facilitar su visualización jerárquica al usuario.

## 3. Generación de Reportes (Graphviz)

Todos los mapas y estructuras lógicas se grafican dinámicamente.

*   **Lógica de implementación:** Cada clase de estructura contiene un método `generarReporteDOT()`. Este método utiliza la clase `QFile` para escribir código de sintaxis DOT (`digraph`) en tiempo de ejecución.
*   **Ejecución externa:** Se utiliza `QProcess` para invocar la herramienta del sistema `dot -Tpng archivo.dot -o imagen.png`, automatizando la renderización y permitiendo visualizar los reportes sin requerir comandos manuales en la terminal por parte del usuario.
*   **Alineamiento en Matriz Dispersa:** Para evitar distorsiones en el grafo de la matriz, se utilizó el atributo `{ rank=same; }` para forzar la horizontalidad de las filas, y el atributo `group` para alinear las columnas verticalmente.


## 4. Arquitectura de la Interfaz Gráfica (Frontend)

Para el desarrollo en Qt, se tomaron las siguientes decisiones de arquitectura visual:

| Componente | Justificación Técnica |
| :--- | :--- |
| `QStackedWidget` | Se utilizó para concentrar todos los paneles (Login, Administrador, Usuario) en una única ventana (`MainWindow`). Esto elimina la complejidad de instanciar múltiples ventanas `QDialog` o `QMainWindow` que podrían generar fugas de memoria o problemas de scope con los punteros globales de las estructuras. |
| `QTabWidget` | Implementado dentro de los roles para subdividir módulos (Ej: Cartelera, Solicitudes, Promociones). Mejora la experiencia de usuario y aisla los componentes de entrada (como `QLineEdit` múltiples) previniendo lecturas cruzadas. |
| **Población en Demanda** | Las tablas y listas de la interfaz se "dibujan" en memoria (`poblarTablaUI`) únicamente durante eventos críticos (Ej: clic en login, clic en aprobar solicitud), garantizando que el usuario siempre vea los datos respaldados en el backend C++. |

