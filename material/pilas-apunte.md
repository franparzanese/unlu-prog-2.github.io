---
title: TAD Pilas - Operaciones, implementaciones y complejidad
---

[← Volver a Programación II](../guia-prog2.md)

# TAD Pilas - Operaciones, implementaciones y complejidad

Esta guía complementa la práctica de Pilas. Cubre el TAD abstracto, las dos implementaciones de la cátedra (arreglos y punteros), y casos de uso con análisis de complejidad.

---

## El TAD Pila

Una **pila** es una estructura LIFO (*Last In, First Out*): el último elemento en entrar es el primero en salir.

Ejemplo intuitivo: una pila de platos. Solo se puede agregar y retirar por arriba.

El TAD define *qué* operaciones se pueden hacer, sin imponer *cómo* se guarda internamente.

### Operaciones abstractas

| Operación | Firma                               | Intención                                                                             |
|-----------|-------------------------------------|---------------------------------------------------------------------------------------|
| Crear     | `Pila p_crear()`                    | Devuelve una pila vacía lista para usar.                                              |
| Apilar    | `bool p_apilar(Pila, TipoElemento)` | Inserta un elemento en el tope. Devuelve `false` si la operación no puede realizarse. |
| Desapilar | `TipoElemento p_desapilar(Pila)`    | Quita y devuelve el elemento del tope, o `NULL` si está vacía.                        |
| Tope      | `TipoElemento p_tope(Pila)`         | Devuelve el elemento del tope sin eliminarlo, o `NULL` si está vacía.                 |
| ¿Vacía?   | `bool p_es_vacia(Pila)`             | Indica si la pila no contiene elementos.                                              |

---

## Implementación 1 - Arreglo (`pilas_arreglos.c`)

### Estructura interna

```c
struct PilaRep {
    TipoElemento *valores;
    unsigned int tope;
};
```

- `valores` apunta a un arreglo contiguo en heap de tamaño fijo `TAMANIO_MAXIMO`.
- `tope` no es un índice válido del último elemento, sino la **próxima posición libre**.

Si `tope = 0`, la pila está vacía.
Si `tope = k`, hay elementos válidos en `valores[0..k-1]`.

```text
índice:  [0]  [1]  [2]  [3]  ...
valores:  e1   e2   e3   --
                    ^
                 tope = 3
```

### Particularidades

- En `p_crear`, se reserva todo el arreglo con `calloc(TAMANIO_MAXIMO, ...)`.
- `p_apilar` escribe en `valores[tope]` y luego incrementa `tope`.
- `p_desapilar` decrementa `tope` y devuelve `valores[tope]`.
- `p_tope` accede a `valores[tope - 1]`.

### Ventajas

- Operaciones principales (`p_apilar`, `p_desapilar`, `p_tope`) en O(1).
- Muy buena localidad de caché.
- Implementación simple y directa.

### Desventajas

- Capacidad fija (`TAMANIO_MAXIMO = 1000`).
- Se reserva memoria para todo el arreglo aunque la pila tenga pocos elementos.

---

## Implementación 2 - Punteros (`pilas_punteros.c`)

### Estructura interna

```c
struct Nodo {
    TipoElemento datos;
    struct Nodo *siguiente;
};

struct PilaRep {
    struct Nodo *tope;
};
```

Cada elemento se guarda en un nodo independiente y `tope` apunta al primer nodo de la cadena.

```text
tope
  |
  v
[e3|*] -> [e2|*] -> [e1|NULL]
```

### Particularidades

- `p_apilar` crea un nodo con `malloc`, lo enlaza arriba y actualiza `tope`.
- `p_desapilar` toma el nodo de arriba, mueve `tope` al siguiente y libera el nodo.

### Ventajas

- No reserva toda la memoria de una vez; crece nodo a nodo.
- `p_desapilar` y `p_tope` son O(1).
- Modelo LIFO muy natural para listas enlazadas.

### Desventajas

- Overhead por nodo (`siguiente`).
- Peor localidad de caché que un arreglo.

---

## Cuadro comparativo de complejidad

$n$ = cantidad de elementos actuales de la pila.
$N$ = capacidad máxima fija (`TAMANIO_MAXIMO = 1000`).

| Operación     | Arreglo | Punteros | Notas                                                      |
|---------------|:-------:|:--------:|------------------------------------------------------------|
| `p_crear`     | $O(N)$  |  $O(1)$  | Arreglo reserva $N$ celdas; punteros solo cabecera.        |
| `p_es_vacia`  | $O(1)$  |  $O(1)$  | Comparación simple (`tope == 0` o `tope == NULL`).         |
| `p_apilar`    | $O(1)$  |  $O(1)$  | Inserta directamente en el tope.                           |
| `p_desapilar` | $O(1)$  |  $O(1)$  | Quita del tope.                                            |
| `p_tope`      | $O(1)$  |  $O(1)$  | Lectura del tope sin quitar.                               |

### Cuándo conviene cada una

- Si importa rendimiento constante en operaciones básicas y hay capacidad acotada conocida, **arreglos** es muy conveniente.
- Si se prioriza crecimiento dinámico y simplicidad conceptual de nodos enlazados, **punteros** es flexible.
- En la práctica, la diferencia suele pasar más por uso de memoria y localidad de caché que por el costo asintótico de las operaciones básicas.

---

## Patrones de uso

### Patrón base LIFO

```c
Pila p = p_crear();

p_apilar(p, te_crear(10));
p_apilar(p, te_crear(20));
p_apilar(p, te_crear(30));

TipoElemento a = p_desapilar(p); // 30
TipoElemento b = p_tope(p);      // 20 (queda en la pila)
```

---

## Casos de uso con análisis de complejidad

### Caso 1 - Vaciar una pila imprimiendo elementos

**Objetivo:** desapilar hasta que quede vacía.

```c
void vaciar_e_imprimir(Pila p) {
    while (!p_es_vacia(p)) {
        TipoElemento e = p_desapilar(p);
        printf("%d\n", e->clave);
    }
}
```

**Complejidad:**

| Implementación | Complejidad |
|----------------|:-----------:|
| Arreglo        |   $O(n)$    |
| Punteros       |   $O(n)$    |

Cada iteración hace operaciones O(1), y hay $n$ iteraciones.

---

### Caso 2 - Copiar una pila preservando la original

**Objetivo:** construir una copia sin perder el contenido ni el orden de la pila original.

Estrategia común: usar una pila auxiliar temporal.

```c
Pila copiar_pila(Pila origen) {
    Pila aux = p_crear();
    Pila copia = p_crear();

    while (!p_es_vacia(origen)) {
        TipoElemento e = p_desapilar(origen);
        p_apilar(aux, e);
    }

    while (!p_es_vacia(aux)) {
        TipoElemento e = p_desapilar(aux);
        p_apilar(origen, e);
        p_apilar(copia, e);
    }

    return copia;
}
```

**Complejidad:**

| Implementación | Complejidad |
|----------------|:-----------:|
| Arreglo        |   $O(n)$    |
| Punteros       |   $O(n)$    |

En ambas implementaciones se hacen una cantidad lineal de operaciones básicas sobre la pila.

---

### Caso 3 - Invertir una pila

**Objetivo:** obtener otra pila con los elementos en orden inverso.

```c
Pila invertir_pila(Pila origen) {
    Pila invertida = p_crear();
    while (!p_es_vacia(origen)) {
        p_apilar(invertida, p_desapilar(origen));
    }
    return invertida;
}
```

**Complejidad:**

| Implementación | Complejidad |
|----------------|:-----------:|
| Arreglo        |   $O(n)$    |
| Punteros       |   $O(n)$    |

Nuevamente, en ambos casos se realiza una secuencia lineal de operaciones sobre el tope.

---

## Buenas prácticas para trabajar con Pilas

- Verificar siempre precondiciones: no desapilar ni consultar tope si está vacía.
- Evitar recorrer la pila con operaciones que la modifiquen si no se restaura el estado.

---

## Resumen rápido

- El TAD Pila modela políticas LIFO con operaciones sobre el tope.
- En esta cátedra hay dos implementaciones: arreglo y punteros.
- Asintóticamente ambas comparten las operaciones básicas en O(1), como corresponde al comportamiento esperado de una pila.
- En código real, no solo importa la estructura elegida: también importan decisiones puntuales de implementación.
