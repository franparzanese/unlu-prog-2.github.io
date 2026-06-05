---
title: Árboles B y B+
---

[← Volver a Programación II](../guia-prog2.md) · [← Apunte de Árboles](arboles-apunte.md)

# Árboles B y B+

Esta guía complementa la práctica de Árboles, en su parte de **árboles B** y **B+**. Son estructuras de búsqueda balanceadas pensadas para grandes volúmenes de datos almacenados en **disco**, donde lo caro no es comparar claves sino **acceder a memoria** (leer un bloque del disco).

> En la teoría, al **B+** se lo llama **«B Mejorado»**: es un árbol B con dos mejoras (una en la inserción y otra en la eliminación). No hay que confundirlo con otras variantes que se ven en bibliografía; acá seguimos el modelo de la cátedra.

Conviene leer primero el [apunte de Árboles](arboles-apunte.md) (ABB y AVL).

A lo largo de la guía usamos **página** como sinónimo de nodo (cada nodo es un bloque de disco) y **rama** para referirnos a cada hijo.

---

## Motivación

Un AVL es excelente en memoria, pero tiene un problema en disco: su altura es $O(\log_2 n)$ y cada nodo guarda **una sola clave**. Bajar un nivel del árbol implica leer un nodo nuevo, es decir, **un acceso a disco**. Con millones de claves, $\log_2 n$ accesos son demasiados.

La idea de los árboles B (Bayer y McCreight, 1970) es **achatar** el árbol: árboles de búsqueda **multicamino**, donde cada página guarda **muchas claves** y tiene **muchas ramas**. Así la altura crece como $\log_m n$ (con $m$ grande), y se necesitan muy pocos accesos a disco para llegar a cualquier clave. Cada página se dimensiona para coincidir con el tamaño de un **bloque** de disco.

---

## Árbol B

### Definición

Un **árbol B de orden $m$** es un árbol de búsqueda **100% balanceado** que cumple:

- Cada página almacena a lo sumo $m - 1$ claves y tiene a lo sumo $m$ ramas (si una página tiene $k$ claves y no es hoja, tiene $k + 1$ ramas).
- Cada página (salvo la raíz) está al menos **medio llena**: tiene al menos $\lceil m/2 \rceil$ ramas. La raíz puede tener menos.
- Las claves dentro de una página están **ordenadas**, y separan los rangos de sus ramas (igual que en un ABB, pero con varias claves por página): a la izquierda las menores, a la derecha las mayores.
- **Todas las hojas están en el mismo nivel**: el árbol siempre está perfectamente balanceado.

```text
Árbol B de orden 5 (hasta 4 claves por página):

                 [ 10 | 20 ]
                /     |     \
        [5|8]    [12|18]   [25|65|92|99]
```

> **Orden 5** ⇒ hasta **4 claves** y **5 ramas** por página; mínimo $\lceil 5/2 \rceil = 3$ ramas (2 claves) en las páginas que no son la raíz.

### Búsqueda

Se busca de forma similar a un ABB: se compara la clave buscada con las de la página actual para decidir por **qué rama** bajar; dentro de cada página la búsqueda es **secuencial ordenada**. Se repite hasta encontrar la clave o llegar a una hoja sin éxito. (Una optimización habitual: si la clave es mayor que la última de la página, se baja directamente por la rama más a la derecha.)

### Inserción

Siempre se inserta en una **hoja** (todas están al mismo nivel).

1. Buscar la hoja donde debería ir la clave.
2. Si la página **tiene espacio**, se agrega la clave ordenada y termina.
3. Si la página se **desborda** (supera $m - 1$ claves), se **divide (split)**: se crea una página nueva al mismo nivel y se reparten los elementos —la mitad menor queda en la página original, la mitad mayor en la nueva— y la **clave del medio sube** a la página padre, siguiendo la misma lógica de inserción. Si no hay padre, se crea una **nueva raíz** y el árbol **crece un nivel**.

El árbol crece "desde las hojas hacia la raíz", lo que mantiene todas las hojas al mismo nivel automáticamente.

### Eliminación

1. Buscar la clave. Si no existe, termina.
2. Si existe:
   - **a)** Si está en una **hoja** con **más** del mínimo de claves, se elimina directamente.
   - **b)** Si está en una **hoja con el mínimo** de claves, la página queda en *underflow* y hay que repararla pidiendo prestada una clave a una **rama hermana adyacente** (a través del padre): baja una clave del padre a la hoja y sube una clave del hermano al padre. Si el hermano también está en el mínimo, no puede prestar y se deben **unir (fusionar)** ambas páginas en una sola, bajando la clave que las separaba en el padre.
   - **c)** Si **no** está en una hoja, se busca la **clave menor de la rama derecha** (el sucesor in-orden), se la usa para reemplazar la clave a borrar, y luego se elimina ese sucesor de la hoja que lo contenía (cayendo en los casos a/b).
3. El proceso puede ser **recursivo hacia la raíz** y, eventualmente, hacer que el árbol **pierda un nivel**.

### Complejidad

$n$ = cantidad de claves; $m$ = orden del árbol.

| Operación | Complejidad     | Notas                                                       |
|-----------|-----------------|-------------------------------------------------------------|
| Buscar    | $O(\log_m n)$   | Altura del árbol; cada nivel = 1 acceso a disco.            |
| Insertar  | $O(\log_m n)$   | Descenso + posibles splits hacia la raíz.                   |
| Eliminar  | $O(\log_m n)$   | Descenso + posibles préstamos/fusiones hacia la raíz.       |

Lo importante en disco es **la cantidad de accesos** = altura del árbol. Como $m$ es grande, $\log_m n$ es muy chico (pocos niveles aun para millones de claves).

---

## Árbol B+ («B Mejorado»)

El **B+** introduce **dos mejoras** sobre el árbol B —una en la inserción y otra en la eliminación— con un mismo objetivo: **mantener las páginas más llenas**, para evitar divisiones y fusiones innecesarias y que el árbol crezca o decrezca de nivel lo menos posible. Ambas mejoras operan **solo a nivel de las hojas**.

### Mejora 1 — Inserción: redistribuir antes de dividir

Cuando una hoja se desborda, en lugar de dividirla de inmediato, primero se fija si una **hoja hermana** (la de la rama izquierda) **tiene lugar** para una clave más:

- Si lo tiene, se **redistribuye a través del padre** (una rotación): baja al hermano la clave que los separa en el padre y sube al padre una clave de la hoja llena. Así no se crea ninguna página nueva.
- Solo si el hermano tampoco tiene lugar se procede a dividir, como en el árbol B.

De esta forma se **llenan todas las hojas antes de producir una división**, lo que evita divisiones innecesarias y hace que el árbol no crezca un nivel hasta que todas las hojas estén llenas.

```text
Orden 5 — insertar 56 (la hoja [25|65|92|99] está llena):

        [ 10 | 20 ]                         [ 10 | 25 ]
       /    |     \           →            /    |      \
   [5|8] [12|18] [25|65|92|99]      [5|8] [12|18|20] [56|65|92|99]

El hermano izquierdo [12|18] tiene lugar: baja el 20 del padre a esa hoja
y sube el 25 al padre. Queda lugar para el 56, sin dividir ninguna página.
```

### Mejora 2 — Eliminación: balancear con ambos hermanos antes de unir

Cuando al eliminar una clave la hoja queda por debajo del mínimo, antes de **unir** páginas se intenta **balancear pidiendo prestada una clave**, pero ahora probando **tanto por la rama izquierda como por la derecha** (el árbol B básico solo prueba con un hermano). Solo si **ninguno** de los dos hermanos puede prestar se fusionan páginas.

> Esto es posible solamente a nivel de las hojas y siempre que la hoja sea una **hoja intermedia** (tenga hermanos a ambos lados).

```text
Orden 5 — eliminar 10 de la hoja [10|18]:

        [ 9 | 20 ]                          [ 9 | 65 ]
       /    |    \            →             /    |    \
   [5|8] [10|18] [65|92|99]          [5|8] [18|20] [92|99]

Al sacar el 10, la hoja queda con menos del mínimo. El hermano izquierdo
[5|8] está en el mínimo y NO puede prestar; el derecho [65|92|99] SÍ:
baja el 20 del padre a la hoja y sube el 65 al padre. No hace falta unir.
```

### ¿Qué se gana?

- **Páginas más llenas** → mejor aprovechamiento del espacio de disco.
- **Menos divisiones y fusiones** → menos reescrituras de páginas.
- El árbol **crece y decrece de nivel con menos frecuencia**, por lo que tiende a quedar **más bajo** que un árbol B equivalente.

Las complejidades asintóticas no cambian respecto del árbol B: búsqueda, inserción y eliminación siguen siendo $O(\log_m n)$. La mejora está en las **constantes** (menos accesos a disco por reorganización y mejor ocupación de las páginas).

---

## Resumen rápido

- Los árboles **B** achatan el árbol guardando **muchas claves por página**, minimizando los accesos a disco; están **100% balanceados** (todas las hojas al mismo nivel).
- En un **B de orden $m$**: hasta $m - 1$ claves y $m$ ramas por página; las páginas (salvo la raíz) están al menos medio llenas. Crece por **divisiones (splits)** y decrece por **fusiones**, que se propagan desde las hojas hacia la raíz.
- El **B+ («B Mejorado»)** agrega dos mejoras a nivel de hojas: en la **inserción**, redistribuye con un hermano antes de dividir; en la **eliminación**, intenta pedir prestado a **ambos** hermanos antes de unir.
- El efecto del B+ es mantener las páginas **más llenas**, con menos divisiones/fusiones y un árbol más bajo. La complejidad sigue siendo $O(\log_m n)$; lo que mejora son las constantes y el uso del disco.
