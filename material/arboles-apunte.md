---
title: TAD Árboles — Binario, de Búsqueda y AVL
---

[← Volver a Programación II](../guia-prog2.md)

# TAD Árboles — Binario, de Búsqueda y AVL

Esta guía complementa la práctica de Árboles. Cubre las tres estructuras que se ven en código —**árbol binario**, **árbol binario de búsqueda (ABB)** y **AVL**—, sus operaciones (API) y la complejidad de cada una. Las cuestiones genéricas (terminología, recorridos) se introducen en la sección de árbol binario y se reutilizan en el resto.

Al final se explica la **transformación de Knuth**, que permite representar árboles n-arios como árboles binarios.

> Los árboles **B** y **B+** se tratan en una guía aparte: [Árboles B y B+](arboles-b-apunte.md).

---

## 1. Árbol Binario

### Motivación

Un **árbol** es una estructura **jerárquica**: a diferencia de listas, pilas y colas (que son lineales), cada elemento puede tener varios sucesores. Un **árbol binario** es el caso más simple: cada nodo tiene a lo sumo **dos hijos**, el hijo izquierdo (`hi`) y el hijo derecho (`hd`).

Sirve para modelar relaciones de jerarquía o de decisión: expresiones aritméticas, estructuras de archivos, árboles de decisión, etc.

### Terminología

| Término             | Definición                                                                 |
|---------------------|----------------------------------------------------------------------------|
| **Nodo**            | Cada elemento del árbol. Guarda un dato y los enlaces a sus hijos.         |
| **Raíz**            | El nodo superior, único, sin padre.                                       |
| **Hoja (terminal)** | Nodo sin hijos.                                                            |
| **Nodo interno**    | Nodo con al menos un hijo.                                                 |
| **Padre / hijo**    | Relación directa entre un nodo y los que cuelgan de él.                    |
| **Hermanos**        | Nodos con el mismo padre.                                                  |
| **Nivel**           | Distancia desde la raíz (la raíz está en el nivel 0).                      |
| **Altura**          | Longitud del camino más largo desde un nodo hasta una hoja.               |
| **Subárbol**        | Cualquier nodo junto con todos sus descendientes es, a su vez, un árbol.  |

### Estructura interna (`nodo.h`)

```c
struct NodoArbolRep {
    TipoElemento datos;
    struct NodoArbolRep *hi;  // hijo izquierdo
    struct NodoArbolRep *hd;  // hijo derecho
    int fe;                   // factor de equilibrio (lo usa el AVL)
};
typedef struct NodoArbolRep *NodoArbol;
```

El árbol guarda la `raiz` y la `cantidad_elementos`; cada nodo se aloja con `malloc` cuando se conecta.

```text
        (raíz)
         8
       /   \
      3      10
     / \       \
    1   6       14
                ↑ hoja
```

### API del árbol binario (`arbol-binario.h`)

| Operación              | Firma                                                | Intención                                                              |
|------------------------|------------------------------------------------------|------------------------------------------------------------------------|
| Crear                  | `ArbolBinario a_crear()`                             | Devuelve un árbol vacío.                                               |
| ¿Vacío?                | `bool a_es_vacio(ArbolBinario)`                      | Indica si el árbol no tiene raíz.                                      |
| Cantidad               | `int a_cantidad_elementos(ArbolBinario)`             | Cantidad de nodos del árbol.                                          |
| Raíz                   | `NodoArbol a_raiz(ArbolBinario)`                     | Devuelve el nodo raíz (o `NULL` si está vacío).                       |
| ¿Rama nula?            | `bool a_es_rama_nula(NodoArbol)`                     | Indica si una posición/rama es nula (`NULL`).                         |
| Establecer raíz        | `NodoArbol a_establecer_raiz(ArbolBinario, TipoElemento)` | Crea la raíz de un árbol vacío.                                  |
| Conectar hijo izq.     | `NodoArbol a_conectar_hi(ArbolBinario, NodoArbol pa, TipoElemento)` | Cuelga un hijo izquierdo del nodo `pa`.            |
| Conectar hijo der.     | `NodoArbol a_conectar_hd(ArbolBinario, NodoArbol pa, TipoElemento)` | Cuelga un hijo derecho del nodo `pa`.             |

Operaciones sobre un nodo (`nodo.h`): `n_hijo_izquierdo`, `n_hijo_derecho`, `n_recuperar` (devuelve el `TipoElemento` del nodo).

### Recorridos

Recorrer un árbol es visitar todos sus nodos en algún orden. Hay cuatro recorridos clásicos; en esta cátedra se exponen mediante **iteradores** (`util_arboles.h`), reutilizando el TAD Iterador de listas.

| Recorrido        | Orden de visita                          | Uso típico                                  |
|------------------|------------------------------------------|---------------------------------------------|
| **Pre-orden**    | raíz → izquierdo → derecho               | Clonar el árbol, prefijo de expresiones.    |
| **In-orden**     | izquierdo → raíz → derecho               | En un ABB, devuelve las claves **ordenadas**. |
| **Post-orden**   | izquierdo → derecho → raíz               | Liberar memoria, postfijo de expresiones.   |
| **Por niveles (BFS)** | nivel por nivel, de izquierda a derecha | Recorrido en anchura; usa una **cola** auxiliar. |

Los recorridos en profundidad son naturalmente **recursivos**:

```c
void in_orden(NodoArbol nodo) {
    if (nodo != NULL) {
        in_orden(nodo->hi);
        // visitar nodo->datos
        in_orden(nodo->hd);
    }
}
```

El recorrido por niveles (BFS) no es recursivo: encola la raíz y, mientras la cola no esté vacía, desencola un nodo, lo visita y encola sus hijos.

### Complejidad

$n$ = cantidad de nodos.

| Operación                | Complejidad | Notas                                              |
|--------------------------|:-----------:|----------------------------------------------------|
| `a_crear`                |   $O(1)$    | Solo inicializa la cabecera.                       |
| `a_es_vacio`             |   $O(1)$    | Comparación simple.                                |
| `a_cantidad_elementos`   |   $O(1)$    | Contador mantenido.                                |
| `a_raiz`                 |   $O(1)$    | Devuelve un puntero.                               |
| `a_establecer_raiz`      |   $O(1)$    | Crea un nodo y lo enlaza.                          |
| `a_conectar_hi` / `a_conectar_hd` | $O(1)$ | Crea un nodo y lo cuelga del padre dado.        |
| Cualquier **recorrido**  |   $O(n)$    | Visita cada nodo exactamente una vez.              |

En un árbol binario "puro" no hay criterio de ubicación: las inserciones se hacen indicando explícitamente el nodo padre, por eso son O(1). La búsqueda de un valor cualquiera requiere recorrer y es O(n).

---

## 2. Árbol Binario de Búsqueda (ABB)

### Motivación

Un **árbol binario de búsqueda** es un árbol binario que mantiene un **invariante de orden**: para todo nodo, las claves de su subárbol izquierdo son **menores** y las de su subárbol derecho son **mayores**.

Ese invariante convierte la búsqueda en un descenso guiado: en cada nodo se compara la clave buscada y se baja a izquierda o a derecha, descartando la mitad restante. Es la versión "en árbol" de la búsqueda binaria, pero sobre una estructura dinámica que admite inserciones y borrados.

```text
        8          buscar 6:  8 → (6<8) izq → 3 → (6>3) der → 6 ✓
      /   \
     3      10
    / \
   1   6
```

### API del ABB (`arbol-binario-busqueda.h`)

| Operación   | Firma                                                  | Intención                                                  |
|-------------|--------------------------------------------------------|------------------------------------------------------------|
| Crear       | `ArbolBinarioBusqueda abb_crear()`                     | Devuelve un ABB vacío.                                     |
| ¿Vacío?     | `bool abb_es_vacio(ArbolBinarioBusqueda)`              | Indica si no tiene nodos.                                  |
| Raíz        | `NodoArbol abb_raiz(ArbolBinarioBusqueda)`             | Devuelve el nodo raíz.                                     |
| Cantidad    | `int abb_cantidad_elementos(ArbolBinarioBusqueda)`     | Cantidad de nodos.                                         |
| Insertar    | `bool abb_insertar(ArbolBinarioBusqueda, TipoElemento)` | Inserta respetando el orden. Devuelve `false` si ya existía. |
| Eliminar    | `bool abb_eliminar(ArbolBinarioBusqueda, int clave)`   | Elimina la clave. Devuelve `false` si no estaba.          |
| Buscar      | `TipoElemento abb_buscar(ArbolBinarioBusqueda, int clave)` | Devuelve el elemento con esa clave, o `NULL`.         |

### Cómo funcionan las operaciones

- **Insertar / buscar**: se desciende desde la raíz comparando claves, hasta encontrar la clave o llegar a una rama nula (donde se inserta el nodo nuevo).
- **Eliminar**: hay tres casos según los hijos del nodo a borrar:
  1. **Hoja**: se elimina directamente.
  2. **Un solo hijo**: el hijo ocupa el lugar del nodo borrado.
  3. **Dos hijos**: se reemplaza el dato por su **sucesor in-orden** (el mínimo del subárbol derecho) y luego se borra ese sucesor, que cae en el caso 1 o 2.

### Complejidad

$h$ = altura del árbol.

| Operación       | Complejidad | Notas                                             |
|-----------------|:-----------:|---------------------------------------------------|
| `abb_crear`     |   $O(1)$    |                                                   |
| `abb_es_vacio`  |   $O(1)$    |                                                   |
| `abb_buscar`    |   $O(h)$    | Un descenso desde la raíz.                        |
| `abb_insertar`  |   $O(h)$    | Busca el lugar y crea el nodo.                    |
| `abb_eliminar`  |   $O(h)$    | Busca la clave (+ sucesor en el caso de 2 hijos). |

La clave está en cuánto vale $h$:

- **Caso promedio / árbol equilibrado**: $h = O(\log n)$ → las operaciones son $O(\log n)$.
- **Peor caso (árbol degenerado)**: si se insertan claves ya ordenadas, el árbol se vuelve una "lista" y $h = O(n)$ → las operaciones son $O(n)$.

```text
inserto 1,2,3,4,5 en orden:   1
                               \
                                2
                                 \
                                  3      ← árbol degenerado, h = O(n)
                                   \
                                    4
                                     \
                                      5
```

Este problema es el que motiva a los árboles **AVL**.

---

## 3. Árbol AVL

### Motivación

Un **AVL** (Adelson-Velsky y Landis) es un ABB **autobalanceado**: tras cada inserción o borrado se reacomoda solo para garantizar que se mantenga equilibrado, evitando el peor caso degenerado del ABB.

El invariante AVL es: para **todo** nodo, las alturas de sus dos subárboles difieren a lo sumo en 1. Ese valor se llama **factor de equilibrio**:

$$FE = altura(subárbol\ izquierdo) - altura(subárbol\ derecho) \in \{-1, 0, +1\}$$

Mantener el FE acotado garantiza $h = O(\log n)$ **siempre**, no solo en promedio. (En la implementación de la cátedra, el campo `fe` del nodo almacena la altura del subárbol, y el balanceo se calcula como la diferencia de alturas de los hijos.)

### Rotaciones

Cuando una inserción o borrado deja un nodo con $FE = \pm 2$ (desbalanceado), se aplica una **rotación** que restaura el equilibrio sin romper el orden del ABB. Hay cuatro casos:

| Caso         | Situación                                   | Solución                          |
|--------------|---------------------------------------------|-----------------------------------|
| **Izq-Izq**  | desbalance a la izquierda, hijo cargado a izq. | rotación simple a la **derecha**  |
| **Der-Der**  | desbalance a la derecha, hijo cargado a der.   | rotación simple a la **izquierda** |
| **Izq-Der**  | desbalance a la izquierda, hijo cargado a der. | rotación **izquierda** del hijo, luego **derecha** |
| **Der-Izq**  | desbalance a la derecha, hijo cargado a izq.   | rotación **derecha** del hijo, luego **izquierda** |

Ejemplo de rotación simple a la izquierda (caso Der-Der):

```text
   a                     b
    \                   / \
     b      →          a   e
    / \                 \
   d   e                 d
```

Las rotaciones son operaciones **locales**: reacomodan unos pocos punteros, por lo que cuestan $O(1)$.

### API del AVL (`arbol-avl.h`)

La interfaz es **idéntica a la del ABB** (cambia solo el prefijo `avl_`); el balanceo es interno y transparente para quien lo usa.

| Operación   | Firma                                          |
|-------------|------------------------------------------------|
| Crear       | `ArbolAVL avl_crear()`                         |
| ¿Vacío?     | `bool avl_es_vacio(ArbolAVL)`                  |
| Raíz        | `NodoArbol avl_raiz(ArbolAVL)`                 |
| Cantidad    | `int avl_cantidad_elementos(ArbolAVL)`         |
| Insertar    | `bool avl_insertar(ArbolAVL, TipoElemento)`    |
| Eliminar    | `bool avl_eliminar(ArbolAVL, int clave)`       |
| Buscar      | `TipoElemento avl_buscar(ArbolAVL, int clave)` |

### Complejidad

| Operación      | Complejidad | Notas                                                       |
|----------------|:-----------:|-------------------------------------------------------------|
| `avl_crear`    |   $O(1)$    |                                                             |
| `avl_buscar`   | $O(\log n)$ | Igual que un ABB, pero con altura garantizada.              |
| `avl_insertar` | $O(\log n)$ | Descenso + a lo sumo una rotación ($O(1)$) al volver.       |
| `avl_eliminar` | $O(\log n)$ | Descenso + reequilibrado en el camino de vuelta.            |

La diferencia clave con el ABB: el AVL garantiza $O(\log n)$ **en el peor caso**, mientras que el ABB solo lo logra en promedio. El precio es el costo (constante) del rebalanceo en cada modificación.

### Cuadro comparativo

| Estructura      | Búsqueda (peor caso) | Inserción (peor caso) | Mantiene orden | Se autobalancea |
|-----------------|:--------------------:|:---------------------:|:--------------:|:---------------:|
| Árbol binario   |        $O(n)$        |        $O(1)$*        |       No       |       No        |
| ABB             |        $O(n)$        |        $O(n)$         |       Sí       |       No        |
| AVL             |     $O(\log n)$      |     $O(\log n)$       |       Sí       |       Sí        |

\* En el árbol binario la inserción es $O(1)$ porque se indica explícitamente el nodo padre; no hay búsqueda de posición.

---

## 4. Transformación de Knuth (n-ario → binario)

### Motivación

Un **árbol n-ario** (o general) permite que cada nodo tenga una cantidad arbitraria de hijos. Representarlo directamente es incómodo: no se sabe de antemano cuántos punteros reservar por nodo.

La **transformación de Knuth** (también llamada representación *hijo-izquierdo / hermano-derecho*) convierte cualquier árbol n-ario en un árbol **binario** equivalente, usando solo dos punteros por nodo. Así se puede manejar un árbol general con la misma estructura de un binario, sin conocer el "n".

### La regla

Para cada nodo del árbol n-ario:

- su **primer hijo** pasa a ser el **hijo izquierdo** en el árbol binario;
- su **hermano siguiente** pasa a ser el **hijo derecho** en el árbol binario.

Es decir: **hi = primer hijo**, **hd = siguiente hermano**.

### Ejemplo

Árbol n-ario:

```text
        A
      / | \
     B  C  D
    / \     \
   E   F     G
```

Aplicando la regla (primer hijo a la izquierda, hermanos encadenados a la derecha):

```text
   A
  /
 B
/ \
E  C
 \   \
  F   D
       \
        G
```

- `A` baja a su primer hijo `B` (izquierda).
- `B → C → D` eran hermanos: ahora se encadenan por la derecha.
- `B` baja a su primer hijo `E`; `E → F` (hermanos) se encadenan por la derecha.
- `D` baja a su único hijo `G`.

La práctica incluye un ejemplo mayor con diagramas en [TP6 — Árboles](../practicas/TP6_Arboles.md).

### Por qué sirve

- Un árbol general queda representado con la **misma estructura de nodo** que un árbol binario (`hi`, `hd`).
- Las operaciones sobre el n-ario (recorridos, altura, nivel, hermanos, etc.) se resuelven navegando esa representación binaria: "bajar a un hijo" es ir por `hi`, y "recorrer los hermanos" es seguir la cadena de `hd`.

---

## Resumen rápido

- Un **árbol** es una estructura jerárquica; el **binario** limita cada nodo a dos hijos (`hi`, `hd`).
- Los **recorridos** (pre/in/post-orden y por niveles) visitan todos los nodos en $O(n)$; el in-orden de un ABB devuelve las claves ordenadas.
- Un **ABB** mantiene el orden y permite buscar en $O(h)$, pero puede degenerar a $O(n)$.
- El **AVL** es un ABB autobalanceado: con rotaciones $O(1)$ garantiza altura $O(\log n)$ y, por lo tanto, operaciones $O(\log n)$ en el peor caso.
- La **transformación de Knuth** representa árboles n-arios como binarios con la regla *primer hijo → izquierda, hermano → derecha*.
