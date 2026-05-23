---
title: TAD Listas — Operaciones, implementaciones y complejidad
---

[← Volver a Programación II](../guia-prog2.md)

# TAD Listas — Operaciones, implementaciones y complejidad

Esta guía complementa la práctica de Listas. Cubre el TAD abstracto, las tres implementaciones de la cátedra, el TAD Iterador y casos de uso con análisis de complejidad.

---

## El TAD Lista

Una **lista** es una secuencia ordenada de elementos donde el orden importa: cada elemento ocupa una posición (ordinal), y las operaciones respetan esa posición.

El TAD define *qué* se puede hacer con una lista, sin decir *cómo* está guardada internamente.

### Operaciones abstractas

| Operación | Firma                                           | Intención                                                                                                                          |
|-----------|-------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Crear     | `Lista l_crear()`                               | Devuelve una lista vacía lista para usar.                                                                                          |
| ¿Vacía?   | `bool l_es_vacia(Lista)`                        | Indica si la lista no contiene ningún elemento.                                                                                    |
| Longitud  | `int l_longitud(Lista)`                         | Devuelve la cantidad de elementos en la lista.                                                                                     |
| Agregar   | `bool l_agregar(Lista, TipoElemento)`           | Inserta un elemento al final de la lista. Devuelve `false` si la operación no puede realizarse.                                    |
| Insertar  | `bool l_insertar(Lista, TipoElemento, int pos)` | Inserta un elemento en la posición `pos`, desplazando hacia adelante los siguientes. Si `pos` supera la longitud, agrega al final. |
| Borrar    | `bool l_borrar(Lista, int clave)`               | Elimina **todos** los elementos con la clave indicada. Devuelve `false` si no encontró ninguno.                                    |
| Eliminar  | `bool l_eliminar(Lista, int pos)`               | Elimina el elemento en la posición `pos`.                                                                                          |
| Buscar    | `TipoElemento l_buscar(Lista, int clave)`       | Devuelve el primer elemento cuya clave coincide, o `NULL` si no existe.                                                            |
| Recuperar | `TipoElemento l_recuperar(Lista, int pos)`      | Devuelve el elemento en la posición `pos` sin eliminarlo, o `NULL` si `pos` es inválida.                                           |

### Sobre posiciones ordinales

Las posiciones en esta cátedra empiezan en **1** (no en 0). El primer elemento está en la posición 1, el último en la posición `l_longitud(lista)`.

---

## Implementación 1 — Arreglo (`listas_arreglos.c`)

### Estructura interna

```c
struct ListaRep {
    TipoElemento *valores;  // arreglo contiguo en heap
    int cantidad;           // cantidad de elementos actuales
};
```

Los elementos se almacenan en celdas contiguas de memoria. El campo `cantidad` actúa como centinela: todo lo que está en `valores[0..cantidad-1]` es válido.

```
índice:  [0]  [1]  [2]  [3]  ...  [999]
valores:  e1   e2   e3   --         --
          ↑                ↑
        inicio           cantidad = 3
```

### Particularidades

- El arreglo se aloja con `calloc(TAMANIO_MAXIMO, ...)` al crear la lista; el tamaño es **fijo**.
- El acceso por posición es directo: `valores[pos - 1]`.
- Insertar o eliminar en una posición arbitraria requiere **desplazar** elementos.
- Agregar al final es barato porque la posición libre siempre se conoce (`valores[cantidad]`).

### Ventajas

- `l_recuperar` y `l_agregar` son O(1): acceso directo sin recorrer nada.
- Localidad de caché excelente: los datos son contiguos en memoria.
- Implementación sencilla.

### Desventajas

- Tamaño máximo fijo: si se necesitan más de `TAMANIO_MAXIMO` elementos, la lista no crece.
- Insertar o eliminar en el medio es O(n) por el desplazamiento de elementos.
- La memoria se reserva completa desde el inicio, aunque la lista esté casi vacía.

---

## Implementación 2 — Punteros (`listas_punteros.c`)

### Estructura interna

```c
struct Nodo {
    TipoElemento datos;
    struct Nodo *siguiente;  // puntero al nodo siguiente
};

struct ListaRep {
    struct Nodo *inicio;  // puntero al primer nodo
    int cantidad;
};
```

Cada elemento vive en un nodo independiente en el heap. Los nodos se enlazan mediante punteros.

```
inicio
  ↓
[e1 | →] → [e2 | →] → [e3 | NULL]
```

### Particularidades

- Cada nodo se crea con `malloc` al momento de insertar; se libera con `free` al borrar.
- No hay tamaño máximo real (sólo el límite artificial `TAMANIO_MAXIMO` por consistencia con las otras implementaciones).
- Para llegar a la posición `pos` siempre hay que recorrer desde el inicio.
- La implementación distingue el caso de insertar/eliminar en la primera posición (no hay nodo previo).

### Ventajas

- Inserciones y eliminaciones en el **inicio** son O(1).
- Usa sólo la memoria necesaria: cada nodo se aloja cuando se necesita.
- No requiere desplazamiento de elementos; sólo actualizar punteros.

### Desventajas

- `l_recuperar`, `l_insertar` y `l_eliminar` son O(n): hay que recorrer hasta la posición.
- `l_agregar` también es O(n): hay que llegar al último nodo para enlazar.
- Peor localidad de caché: los nodos pueden estar dispersos en el heap.
- El overhead de cada nodo incluye el puntero `siguiente`.

---

## Implementación 3 — Cursores (`listas_cursores.c`)

### Estructura interna

```c
struct Nodo {
    TipoElemento datos;
    int siguiente;  // índice del siguiente nodo (NULO = -1)
};

struct ListaRep {
    struct Nodo *cursor;  // arreglo estático de nodos
    int inicio;           // índice del primer nodo lógico
    int libre;            // índice del primer nodo disponible
    int cantidad;
};
```

La implementación usa **un único arreglo** `cursor` de `struct Nodo`. Cada nodo tiene dos campos: el `TipoElemento` y el índice `siguiente`.

El arreglo `cursor` simula el heap: en lugar de punteros reales se usan **índices enteros**. Sobre ese mismo arreglo conviven dos encadenamientos lógicos:

- la lista activa (arrancando en `inicio`), con los nodos que hoy pertenecen a la lista;
- la lista de libres (arrancando en `libre`), con los nodos disponibles.

```
cursor[] (un solo arreglo de nodos):
    índice:   0            1            2            3            4     ...
    nodo:   [e1, sig=2]  [--, sig=4]  [e2, sig=3]  [e3, sig=-1] [--, ...]

inicio = 0
libre  = 1
```

Al crear la lista, todos los nodos se encadenan en la lista de libres (`0 -> 1 -> 2 -> ... -> NULO`) y `inicio` queda en `NULO`.

```
libre = 0 → 1 → 2 → 3 → ... → 999 → NULO
```

### Particularidades

- No hay dos arreglos separados (uno de datos y otro de enlaces): el enlace está dentro de cada nodo como `int siguiente`.
- Simula la dinámica de punteros usando índices sobre un arreglo estático; no usa `malloc`/`free` por elemento.
- Tomar un nodo libre es O(1): se saca la cabeza de la lista de libres.
- Devolver un nodo es O(1): se reinsertan en la lista de libres.
- El comportamiento lógico es el mismo que punteros; la diferencia es que los "punteros" son enteros.
- Útil en entornos donde la asignación dinámica no está disponible o es costosa.

### Ventajas

- Sin llamadas a `malloc`/`free` por elemento: asignación y liberación en O(1).
- Comportamiento de lista enlazada sin overhead de gestión dinámica de memoria.
- Portable a entornos embebidos o con memoria estática.

### Desventajas

- El arreglo completo se aloja al crear la lista, igual que en arreglos.
- `l_recuperar`, `l_insertar` y `l_eliminar` son O(n) como en punteros.
- `l_agregar` es O(n): debe recorrer hasta el último nodo para enlazar.
- La inicialización de la lista de libres en `l_crear` es O(TAMANIO_MAXIMO).
- Mayor complejidad conceptual que los arreglos.

---

## Cuadro comparativo de complejidad

$n$ = cantidad de elementos en la lista. El análisis asume **peor caso**.

| Operación     | Arreglo | Punteros | Cursores | Notas                                                                                      |
|---------------|:-------:|:--------:|:--------:|--------------------------------------------------------------------------------------------|
| `l_crear`     | $O(N)$  |  $O(1)$  |  $O(N)$  | Arreglos y cursores alocan $N$ celdas; punteros sólo la cabecera.                          |
| `l_es_vacia`  | $O(1)$  |  $O(1)$  |  $O(1)$  | Compara `cantidad == 0`.                                                                   |
| `l_longitud`  | $O(1)$  |  $O(1)$  |  $O(1)$  | El contador `cantidad` se mantiene actualizado.                                            |
| `l_agregar`   | $O(1)$  |  $O(n)$  |  $O(n)$  | Arreglos: inserta en `valores[cantidad]`. Punteros/cursores: recorre hasta el último nodo. |
| `l_borrar`    | $O(n)$  |  $O(n)$  |  $O(n)$  | Recorre toda la lista para eliminar todas las ocurrencias; desplaza o reenlaza.            |
| `l_buscar`    | $O(n)$  |  $O(n)$  |  $O(n)$  | Búsqueda lineal por clave.                                                                 |
| `l_insertar`  | $O(n)$  |  $O(n)$  |  $O(n)$  | Arreglos: desplaza $n - pos$ elementos. Punteros/cursores: recorre hasta `pos - 1`.        |
| `l_eliminar`  | $O(n)$  |  $O(n)$  |  $O(n)$  | Arreglos: desplaza $n - pos$ elementos. Punteros/cursores: recorre hasta `pos - 1`.        |
| `l_recuperar` | $O(1)$  |  $O(n)$  |  $O(n)$  | Arreglos: acceso directo `valores[pos-1]`. Punteros/cursores: recorre hasta `pos`.         |

> **Nota sobre $N$:** en los casos de `l_crear`, $N$ es el tamaño máximo fijo del arreglo (`TAMANIO_MAXIMO = 1000`), no la cantidad de elementos actuales.

### ¿Cuándo importa la diferencia?

- Si el acceso por posición es frecuente, **arreglos** gana con `l_recuperar` en O(1).
- Si las inserciones/eliminaciones al **inicio** son frecuentes, **punteros** es más eficiente conceptualmente (aunque la implementación actual no aprovecha esto para `l_agregar`).
- Si no se puede usar memoria dinámica, **cursores** ofrece comportamiento de lista enlazada con asignación estática.
- Para la mayoría de las operaciones las tres implementaciones tienen la misma complejidad asintótica: la elección depende del perfil de uso y las restricciones del entorno.

---

## TAD Iterador

### Motivación

Recorrer una lista elemento por elemento es una necesidad muy frecuente. Sin el iterador, la opción más directa sería:

```c
for (int i = 1; i <= l_longitud(lista); i++) {
    TipoElemento e = l_recuperar(lista, i);
    // procesar e
}
```

Esto funciona, pero tiene un problema de complejidad:

- En **arreglos**, `l_recuperar` es O(1), así que el bucle completo es O(n). Correcto.
- En **punteros** y **cursores**, `l_recuperar` es O(n), así que el bucle completo es **O(n²)**. Muy ineficiente.

El **Iterador** resuelve esto encapsulando el recorrido de forma eficiente: recuerda en qué punto de la lista está y avanza un paso en O(1), sin importar la implementación.

### Interfaz del Iterador

```c
Iterador iterador(Lista lista);     // crea un iterador apuntando al primer elemento
bool hay_siguiente(Iterador iter);  // true si quedan elementos por visitar
TipoElemento siguiente(Iterador iter); // devuelve el elemento actual y avanza
```

### Patrón de uso

```c
Iterador iter = iterador(lista);
while (hay_siguiente(iter)) {
    TipoElemento e = siguiente(iter);
    // procesar e
}
free(iter);
```

El bucle completo es **O(n)** para las tres implementaciones.

### Implementación según tipo de lista

#### Iterador en arreglos

```c
struct IteradorRep {
    Lista lista;
    int posicionActual;  // índice en el arreglo (empieza en 0)
};

Iterador iterador(Lista lista) {
    Iterador iter = malloc(sizeof(struct IteradorRep));
    iter->lista = lista;
    iter->posicionActual = 0;
    return iter;
}

bool hay_siguiente(Iterador iter) {
    return iter->posicionActual < iter->lista->cantidad;
}

TipoElemento siguiente(Iterador iter) {
    return iter->lista->valores[iter->posicionActual++];
}
```

El estado del iterador es simplemente un entero que avanza de 0 a `cantidad - 1`.

#### Iterador en punteros

```c
struct IteradorRep {
    struct Nodo *posicionActual;  // puntero al nodo actual
};

Iterador iterador(Lista lista) {
    Iterador iter = malloc(sizeof(struct IteradorRep));
    iter->posicionActual = lista->inicio;
    return iter;
}

bool hay_siguiente(Iterador iter) {
    return iter->posicionActual != NULL;
}

TipoElemento siguiente(Iterador iter) {
    TipoElemento actual = iter->posicionActual->datos;
    iter->posicionActual = iter->posicionActual->siguiente;
    return actual;
}
```

El estado del iterador es un puntero que sigue la cadena de nodos.

Ejemplo rápido de avance del iterador (punteros):

```
inicio -> [A | * ] -> [B | * ] -> [C | NULL]
```

1. Al crear el iterador: `posicionActual = inicio` (apunta al nodo con `A`).
2. Primera llamada a `siguiente`: devuelve `A` y avanza al nodo con `B`.
3. Segunda llamada a `siguiente`: devuelve `B` y avanza al nodo con `C`.
4. Tercera llamada a `siguiente`: devuelve `C` y avanza a `NULL`.
5. Ahora `hay_siguiente` devuelve `false` porque `posicionActual == NULL`.

#### Iterador en cursores

```c
struct IteradorRep {
    Lista lista;
    int posicionActual;  // índice en el arreglo cursor
};

Iterador iterador(Lista lista) {
    Iterador iter = malloc(sizeof(struct IteradorRep));
    iter->lista = lista;
    iter->posicionActual = lista->inicio;  // índice del primer nodo lógico
    return iter;
}

bool hay_siguiente(Iterador iter) {
    return iter->posicionActual != NULO;  // NULO = -1
}

TipoElemento siguiente(Iterador iter) {
    TipoElemento actual = iter->lista->cursor[iter->posicionActual].datos;
    iter->posicionActual = iter->lista->cursor[iter->posicionActual].siguiente;
    return actual;
}
```

El estado del iterador es un entero que sigue la cadena de índices, igual que el puntero en la versión de punteros pero usando índices en lugar de direcciones.

Ejemplo rápido de avance del iterador (cursores):

```
inicio = 5
cursor[5] = [dato=A, siguiente=9]
cursor[9] = [dato=B, siguiente=2]
cursor[2] = [dato=C, siguiente=-1]
```

1. Al crear el iterador: `posicionActual = 5`.
2. Primera llamada a `siguiente`: devuelve `A` y deja `posicionActual = 9`.
3. Segunda llamada a `siguiente`: devuelve `B` y deja `posicionActual = 2`.
4. Tercera llamada a `siguiente`: devuelve `C` y deja `posicionActual = -1`.
5. Ahora `hay_siguiente` devuelve `false` porque `-1` representa `NULO`.

### Complejidad del iterador

| Operación             | Arreglo | Punteros | Cursores |
|-----------------------|:-------:|:--------:|:--------:|
| `iterador(lista)`     | $O(1)$  |  $O(1)$  |  $O(1)$  |
| `hay_siguiente(iter)` | $O(1)$  |  $O(1)$  |  $O(1)$  |
| `siguiente(iter)`     | $O(1)$  |  $O(1)$  |  $O(1)$  |
| Recorrido completo    | $O(n)$  |  $O(n)$  |  $O(n)$  |

---

## Casos de uso con análisis de complejidad

### Caso 1 — Imprimir todos los elementos

**Objetivo:** recorrer la lista e imprimir cada elemento.

```c
void imprimir_lista(Lista lista) {
    Iterador iter = iterador(lista);
    while (hay_siguiente(iter)) {
        TipoElemento e = siguiente(iter);
        printf("%d\n", e->clave);
    }
    free(iter);
}
```

**Complejidad:**

| Implementación | Complejidad |
|----------------|:-----------:|
| Arreglo        |   $O(n)$    |
| Punteros       |   $O(n)$    |
| Cursores       |   $O(n)$    |

El iterador garantiza O(n) en todos los casos. Sin iterador, usando `l_recuperar` en un for, punteros y cursores serían O(n²).

---

### Caso 2 — Encontrar el máximo

**Objetivo:** encontrar el elemento con mayor clave en la lista.

```c
TipoElemento maximo(Lista lista) {
    if (l_es_vacia(lista)) return NULL;

    Iterador iter = iterador(lista);
    TipoElemento max = siguiente(iter);

    while (hay_siguiente(iter)) {
        TipoElemento e = siguiente(iter);
        if (e->clave > max->clave) {
            max = e;
        }
    }
    free(iter);
    return max;
}
```

**Complejidad:**

| Implementación | Complejidad |
|----------------|:-----------:|
| Arreglo        |   $O(n)$    |
| Punteros       |   $O(n)$    |
| Cursores       |   $O(n)$    |

Búsqueda lineal: hay que visitar todos los elementos en el peor caso.

---

### Caso 3 — Contar ocurrencias de una clave

**Objetivo:** contar cuántas veces aparece una clave determinada.

```c
int contar_ocurrencias(Lista lista, int clave) {
    int contador = 0;
    Iterador iter = iterador(lista);
    while (hay_siguiente(iter)) {
        TipoElemento e = siguiente(iter);
        if (e->clave == clave) contador++;
    }
    free(iter);
    return contador;
}
```

**Complejidad:**

| Implementación | Complejidad |
|----------------|:-----------:|
| Arreglo        |   $O(n)$    |
| Punteros       |   $O(n)$    |
| Cursores       |   $O(n)$    |

Siempre O(n): hay que recorrer toda la lista porque puede haber duplicados al final.

---

### Caso 4 — Insertar manteniendo orden ascendente

**Objetivo:** insertar un elemento en la posición correcta para que la lista quede ordenada.

```c
void insertar_ordenado(Lista lista, TipoElemento nuevo) {
    int pos = 1;
    for (int i = 1; i <= l_longitud(lista); i++) {
        TipoElemento e = l_recuperar(lista, i);
        if (e->clave <= nuevo->clave) {
            pos = i + 1;
        } else {
            break;
        }
    }
    l_insertar(lista, nuevo, pos);
}
```

**Complejidad:**

| Implementación | Búsqueda de posición | Inserción |  Total   |
|----------------|:--------------------:|:---------:|:--------:|
| Arreglo        |        $O(n)$        |  $O(n)$   |  $O(n)$  |
| Punteros       |       $O(n^2)$       |  $O(n)$   | $O(n^2)$ |
| Cursores       |       $O(n^2)$       |  $O(n)$   | $O(n^2)$ |

En arreglos `l_recuperar` es O(1), por lo que el bucle de búsqueda es O(n). En punteros y cursores, cada llamada a `l_recuperar` es O(n), lo que hace que el bucle sea O(n²).

**Versión optimizada con iterador** (O(n) en todas las implementaciones):

```c
void insertar_ordenado_v2(Lista lista, TipoElemento nuevo) {
    int pos = 1;
    Iterador iter = iterador(lista);
    while (hay_siguiente(iter)) {
        TipoElemento e = siguiente(iter);
        if (e->clave <= nuevo->clave) pos++;
        else break;
    }
    free(iter);
    l_insertar(lista, nuevo, pos);
}
```

| Implementación | Complejidad |
|----------------|:-----------:|
| Arreglo        |   $O(n)$    |
| Punteros       |   $O(n)$    |
| Cursores       |   $O(n)$    |

La búsqueda de posición pasa a ser O(n) en todos los casos al usar el iterador. La inserción sigue siendo O(n) por el desplazamiento o recorrido hasta la posición.

---

### Caso 5 — Construir una lista invertida

**Objetivo:** dada una lista, construir una nueva con los mismos elementos en orden inverso.

```c
Lista invertir_lista(Lista original) {
    Lista invertida = l_crear();
    Iterador iter = iterador(original);
    while (hay_siguiente(iter)) {
        TipoElemento e = siguiente(iter);
        l_insertar(invertida, e, 1);  // inserta siempre al principio
    }
    free(iter);
    return invertida;
}
```

**Análisis:** el bucle ejecuta `l_longitud(original)` veces. Dentro del bucle, `l_insertar` en posición 1 tiene distinta complejidad según implementación:

- **Arreglo**: insertar en posición 1 desplaza todos los elementos → O(n). El bucle completo es $O(1 + 2 + \ldots + n) = O(n^2)$.
- **Punteros**: insertar en posición 1 es O(1) (sólo actualizar punteros al inicio). El bucle completo es $O(n)$.
- **Cursores**: insertar en posición 1 es O(1) (toma del libre y actualiza índice de inicio). El bucle completo es $O(n)$.

**Complejidad:**

| Implementación | Complejidad |
|----------------|:-----------:|
| Arreglo        |  $O(n^2)$   |
| Punteros       |   $O(n)$    |
| Cursores       |   $O(n)$    |

Este caso ilustra una situación donde **punteros y cursores superan claramente a arreglos**: las inserciones repetidas al inicio tienen un costo muy diferente.
