---
title: TAD Colas - Operaciones, implementaciones y complejidad
---

[← Volver a Programación II](../guia-prog2.md)

# TAD Colas - Operaciones, implementaciones y complejidad

Esta guía complementa la práctica de Colas. Cubre el TAD abstracto, las tres implementaciones de la cátedra (arreglos, punteros y cola circular), y casos de uso con análisis de complejidad.

---

## El TAD Cola

Una **cola** es una estructura FIFO (*First In, First Out*): el primer elemento en entrar es el primero en salir.

Ejemplo intuitivo: una fila de personas en una ventanilla. Los nuevos elementos entran al final, y las salidas ocurren por el frente.

El TAD define *qué* operaciones se pueden hacer con una cola, sin imponer *cómo* se guarda internamente.

### Operaciones abstractas

| Operación  | Firma                                | Intención                                                                            |
|------------|--------------------------------------|--------------------------------------------------------------------------------------|
| Crear      | `Cola c_crear()`                     | Devuelve una cola vacía lista para usar.                                             |
| Encolar    | `bool c_encolar(Cola, TipoElemento)` | Inserta un elemento al final. Devuelve `false` si la operación no puede realizarse.  |
| Desencolar | `TipoElemento c_desencolar(Cola)`    | Quita y devuelve el elemento del frente, o `NULL` si la cola está vacía.             |
| Frente     | `TipoElemento c_frente(Cola)`        | Devuelve el elemento del frente sin eliminarlo, o `NULL` si la cola está vacía.      |
| ¿Vacía?    | `bool c_es_vacia(Cola)`              | Indica si la cola no contiene elementos.                                             |

---

## Implementación 1 - Arreglo (`colas_arreglos.c`)

### Estructura interna

```c
struct ColaRep {
    TipoElemento *valores;
    unsigned int cantidad;
};
```

- `valores` apunta a un arreglo contiguo en heap de tamaño fijo `TAMANIO_MAXIMO`.
- `cantidad` indica cuántos elementos válidos hay.
- El frente lógico de la cola está siempre en `valores[0]`.

```text
índice:   [0]  [1]  [2]  [3]  ...
valores:   e1   e2   e3   --
           ^
        frente
cantidad = 3
```

### Particularidades

- En `c_crear`, se reserva todo el arreglo con `calloc(TAMANIO_MAXIMO, ...)`.
- `c_encolar` escribe en `valores[cantidad]` y luego incrementa `cantidad`.
- `c_desencolar` devuelve `valores[0]` y luego desplaza todos los elementos una posición hacia la izquierda.
- `c_frente` accede a `valores[0]` sin modificar la cola.

### Ventajas

- `c_encolar` y `c_frente` son O(1).
- Muy buena localidad de caché.
- Implementación simple de visualizar.

### Desventajas

- Capacidad fija (`TAMANIO_MAXIMO = 1000`).
- `c_desencolar` es O(n) por el corrimiento de elementos.
- Se reserva memoria para todo el arreglo aunque la cola tenga pocos elementos.

---

## Implementación 2 - Punteros (`colas_punteros.c`)

### Estructura interna

```c
struct Nodo {
    TipoElemento datos;
    struct Nodo *siguiente;
};

struct ColaRep {
    struct Nodo *frente;
    struct Nodo *final;
};
```

Cada elemento se guarda en un nodo independiente. `frente` apunta al primer nodo y `final` al último.

```text
frente                              final
  |                                   |
  v                                   v
[e1|*] -> [e2|*] -> [e3|NULL]
```

### Particularidades

- `c_encolar` crea un nodo con `malloc`, lo enlaza después de `final` y actualiza `final`.
- `c_desencolar` toma el nodo del frente, mueve `frente` al siguiente y libera el nodo.
- Si la cola estaba vacía, al encolar hay que inicializar tanto `frente` como `final`.
- Si al desencolar se elimina el único elemento, la cola debe volver a un estado vacío consistente.
- `c_frente` devuelve los datos del primer nodo sin modificar la cola.

### Ventajas

- No reserva toda la memoria de una vez; crece nodo a nodo.
- `c_encolar`, `c_desencolar` y `c_frente` son O(1).
- Modelo FIFO natural usando punteros a frente y final.

### Desventajas

- Overhead por nodo (`siguiente`).
- Peor localidad de caché que un arreglo.

---

## Implementación 3 - Cola circular (`colas_arreglos_circular.c`)

### Estructura interna

```c
struct ColaRep {
    TipoElemento *valores;
    unsigned int frente;
    unsigned int final;
};
```

Es una implementación con **arreglo circular**. `frente` y `final` avanzan sobre el arreglo, y cuando llegan al final vuelven al comienzo.

```text
índice:   [1]  [2]  [3]  [4]  [5]  [6]
valores:   --   e2   e3   --   --   e1
                 ^                ^
              frente = 2       final = 6
```

### Particularidades

- Usa una función auxiliar para avanzar posiciones en forma circular.
- `c_encolar` hace avanzar `final` y escribe allí el nuevo elemento.
- `c_desencolar` devuelve `valores[frente]` y luego hace avanzar `frente`.
- La condición de vacía se detecta comparando `paso(final)` con `frente`.
- Se reserva una celda extra para distinguir correctamente entre cola vacía y cola llena.

### Ventajas

- `c_encolar`, `c_desencolar` y `c_frente` son O(1).
- Muy buena localidad de caché.
- Evita el corrimiento de elementos del arreglo lineal.

### Desventajas

- Capacidad fija (`TAMANIO_MAXIMO = 1000`).
- La lógica es más delicada que en el arreglo lineal por el manejo circular de índices.
- Requiere razonar con posiciones físicas y orden lógico.

---

## Cuadro comparativo de complejidad

$n$ = cantidad de elementos actuales de la cola.  
$N$ = capacidad máxima fija (`TAMANIO_MAXIMO = 1000`).

| Operación      | Arreglo | Punteros | Circular | Notas |
|----------------|:-------:|:--------:|:--------:|-------|
| `c_crear`      | $O(N)$  |  $O(1)$  |  $O(N)$  | Arreglo y circular reservan el arreglo; punteros solo cabecera. |
| `c_es_vacia`   | $O(1)$  |  $O(1)$  |  $O(1)$  | Compara `frente <= 0`, `frente == NULL` o `paso(final) == frente`. |
| `c_encolar`    | $O(1)$  |  $O(1)$  |  $O(1)$  | En punteros y circular se inserta directamente al final. |
| `c_desencolar` | $O(n)$  |  $O(1)$  |  $O(1)$  | En arreglo hay corrimiento de elementos. |
| `c_frente`     | $O(1)$  |  $O(1)$  |  $O(1)$  | Lectura del frente sin quitar. |

### Cuándo conviene cada una

- Si importa simplicidad conceptual y acceso contiguo en memoria, **arreglos** es fácil de entender, aunque desencolar sea costoso.
- Si se quiere una estructura dinámica con operaciones básicas O(1), **punteros** es una muy buena opción.
- Si se busca mantener las operaciones básicas en O(1) sin usar memoria dinámica por nodo, **cola circular** es la opción más eficiente.

---

## Patrones de uso

### Patrón base FIFO

```c
Cola c = c_crear();

c_encolar(c, te_crear(10));
c_encolar(c, te_crear(20));
c_encolar(c, te_crear(30));

TipoElemento a = c_desencolar(c); // 10
TipoElemento b = c_frente(c);     // 20 (queda en la cola)
```

---

## Casos de uso con análisis de complejidad

### Caso 1 - Vaciar una cola imprimiendo elementos

**Objetivo:** desencolar hasta que quede vacía.

```c
void vaciar_e_imprimir(Cola c) {
    while (!c_es_vacia(c)) {
        TipoElemento e = c_desencolar(c);
        printf("%d\n", e->clave);
    }
}
```

**Complejidad:**

| Implementación | Complejidad |
|----------------|:-----------:|
| Arreglo        |  $O(n^2)$   |
| Punteros       |   $O(n)$    |
| Circular       |   $O(n)$    |

En arreglo, cada `c_desencolar` desplaza elementos. En punteros y cola circular, cada iteración hace operaciones O(1).

---

### Caso 2 - Copiar una cola preservando la original

**Objetivo:** construir una copia sin perder el contenido ni el orden de la cola original.

Estrategia común: usar una cola auxiliar temporal.

```c
Cola copiar_cola(Cola origen) {
    Cola aux = c_crear();
    Cola copia = c_crear();

    while (!c_es_vacia(origen)) {
        TipoElemento e = c_desencolar(origen);
        c_encolar(aux, e);
        c_encolar(copia, e);
    }

    while (!c_es_vacia(aux)) {
        c_encolar(origen, c_desencolar(aux));
    }

    return copia;
}
```

**Complejidad:**

| Implementación | Complejidad |
|----------------|:-----------:|
| Arreglo        |  $O(n^2)$   |
| Punteros       |   $O(n)$    |
| Circular       |   $O(n)$    |

- En **arreglo**, el costo dominante aparece en los desencolados.
- En **punteros** y **cola circular**, todas las operaciones básicas usadas son O(1).

---

### Caso 3 - Invertir una cola sin destruir la original

**Objetivo:** obtener otra cola con los elementos en orden inverso.

Como una cola por sí sola conserva el orden FIFO, una forma simple de invertirla es apoyarse en una pila auxiliar.

```c
Cola invertir_cola(Cola origen) {
    Cola aux = c_crear();
    Cola invertida = c_crear();
    Pila pila = p_crear();

    while (!c_es_vacia(origen)) {
        TipoElemento e = c_desencolar(origen);
        c_encolar(aux, e);
        p_apilar(pila, e);
    }

    while (!c_es_vacia(aux)) {
        c_encolar(origen, c_desencolar(aux));
    }

    while (!p_es_vacia(pila)) {
        c_encolar(invertida, p_desapilar(pila));
    }

    return invertida;
}
```

**Complejidad:**

| Implementación | Complejidad |
|----------------|:-----------:|
| Arreglo        |  $O(n^2)$   |
| Punteros       |   $O(n)$    |
| Circular       |   $O(n)$    |

La pila auxiliar agrega operaciones O(1); la diferencia principal vuelve a estar en cómo cada cola resuelve `c_encolar` y `c_desencolar`.

---

## Buenas prácticas para trabajar con Colas

- Verificar siempre precondiciones: no desencolar ni consultar el frente si la cola está vacía.
- Si un algoritmo debe preservar la cola original, restaurarla explícitamente al finalizar.
- En implementaciones con punteros, cuidar especialmente los casos borde: cola vacía y cola con un único elemento.
- En cola circular, distinguir siempre entre el orden lógico FIFO y las posiciones físicas del arreglo.

---

## Resumen rápido

- El TAD Cola modela políticas FIFO con inserción al final y extracción por el frente.
- En esta cátedra hay tres implementaciones: arreglo, punteros y cola circular.
- Punteros y cola circular preservan las operaciones básicas en O(1); la diferencia principal pasa por memoria dinámica versus arreglo fijo.
- En código real, no solo importa la estructura elegida: también importan decisiones puntuales de implementación.
