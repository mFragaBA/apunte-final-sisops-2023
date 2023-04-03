# Design of a General Purpose Memory Allocator for the 4.3BSD UNIX Kernel

## Intro

Setea el contexto actual de los allocators de memoria en BSD. Hasta el
momento venían armando un allocator custom para cada tipo de uso.
Plantean que hay algunas cosas muy importantes que hay que priorizar:

- Tener un allocator de propósito general
- que preserve la misma interfaz de `malloc()` y `free()`
- el `malloc()` tiene que recibir sólamente el tamaño del cacho del bloque pedido.
- el `free()` sólo tiene que recibir el puntero al inicio del bloque a liberar

## Criterios para un allocator de memoria

Evalúan la efectividad del algoritmo midiendo la utilización porcentual. Esto es calcular la cantidad de memoria necesaria para cubrir un conjunto de reservas de memoria en cualquier momento. Está dado por:

\\[
\text{utilization} = \frac{requested}{required}
\\]

Donde \\( \text{requested} \\) es la suma de la cantidad memoria reservada y todavía no liberada. Por required se refiere a la cantidad de memoria reservada para responder a esos pedidos de memoria (siempre se requiere un poco más para sopesar los problemas de fragmentación y para tener un buffer listo para nuevos pedidos de memoria).

- En la práctica una utilización del 50% es considerada buena

También destaca algunos aspectos/requerimientos importantes de un buen algoritmo de reserva de memoria.

- Es más importante una buena utilización de memoria en el kernel vs en los procesos de usuario, principalmente porque la memoria de usuario es virtual y siempre podemos sacar páginas.
- El algoritmo tiene que ser **rápido**, debido a que es una operación utilizada frecuentemente. Un allocator lento degrada la performance de todo el OS y de lios procesos que corren en el mismo.
  - además un mal allocator va a incentivar a los desarrolladores a crear su propio allocator, reduciendo la efectividad del allocator de propíosito general.

## Implementación User-Level

El user-level memory allocator de 4.2BSD mantiene un conjunto de
listas ordenadas según cada potencia de 2 (O sea tengo listas de
bloques de 1 byte, 2 bytes, 4 bytes, 8 bytes, etc. Para satisfacer un
pedido de memoria, redondeo a la mínima potencia de 2 más grande que
el tamaño pedido y le devuelvo ese bloque de memoria. De esa forma
`free()` sólo tiene que devolver el bloque de memoria al arreglo
correspondiente. Sólo cuando la lista sea vacía, hace un pedido de
memoria posta (al kernel).

## Consideraciones de un Kernel-Level memory allocator

Se asumen algunas suposiciones:

- el tamaño máximo a reservar está acotado desde el momento del
  booteo.
  - de esa forma el kernel crea y mantiene un conjunto estático de
    estructuras para manejar la memoria dinámicamente reservada.
- el allocator puede manejar su propio espacio de memoria como le
  plazca.
  - a diferencia de los user-space que sólo pueden reservar/liberar
    memoria para hacer crecer/achicar el heap.
- otra suposición es la de conocer de antemano la distribución de los
  pedidos de memoria
  - según sus estimaciones, frecuentemente se pide poca memoria, y
    poco frecuentemente se pide mucha memoria. Entonces priorizo que
    sea rápido para porciones pequeñas de memoria.

## Implementación de un kernel level memory allocator

- para pedidos de poca memoria se usa la estrategia de potencias de 2.
  - cuando no hay espacio disponible recién ahí se llama al "allocator
    real"
- para asegurarse que los pedidos grandes llaman al allocator las
  listas de las potencias de 2 grandes siempre están vacías.
- idea del allocator posta:
  - se redondea al múltiplo de `PAGE_SIZE` más cercano. Usa first-fit
    para encontrar los bloques.
- otra pequeña optimización en pedidos chicos: me traigo una página
  entera aunque sobre, la divido y con los que sobran lleno la lista
  de potencias de 2 para acelerar pedidos futuros.
- Para bloques grandes se guardan en las páginas el tamaño del bloque (esto es necesario porque `free()` no recibe el tamaño del bloque a liberar).
