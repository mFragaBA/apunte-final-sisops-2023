# Sistemas de Archivos

Para nosotros (y sistemas basados en UNIX) un archivo es una secuencia de
bytes, sin estructura. Se los identifica con un nombre y ese nombre puede
incluir una extensión que puede servir para distinguir el contenido.

Para ordenar los archivos en disco, los sistemas operativos tienen un módulo
dentro del kernel llamado **sistema de archivos o file system**. Existen tanto
sistemas de archivos "locales", como lo son FAT, NTFS, etc. O distribuidos como
lo son NFS, DFS, SMBFS, entre otros.

Una de las responsabilidades elementales de un filesystem es determinar una
organización lógica de los archivos, o sea:

- interna: cómo estructurar la info dentro del archivo (ej: linux usa secuencia
  de bytes y el usuario tiene la responsabilidad del formato).
- externa: cómo se ordenan los archivos (uso directorios, algo más tipo object storage?)

La mayoría hoy en día además soporta alguna noción de **link o alias** a un
archivo, o sea tener varios nombres para un mismo archivo (o por ejemplo poder
acceder a un archivo desde dos rutas distintas).

El filesystem también determina **cómo se nombran los archivos**, esto incluye:

- Determinar Caracteres de separación de directorio
- Si los archivos llevan o no extensión
- Restricciones a la longitud y caracteres permitidos
- Si es case sensitive o insensitive
- Punto de montaje (la ruta absoluta al dispositivo correspondiente al archivo)
  - ej: `/disk1/bla/dir_1_de_lo_montado/etc` tiene como punto de montaje `/disk1/bla` por ejemplo

Por último y casi lo principal, el filesystem también determina la
representación del archivo y dos preguntas relacionadas a este:

- cómo gestiono el espacio libre?
- qué hago para la metadata (permisos, atributos, etc.)

## Representación de archivos

A los ojos del FS (filesystem), un archivo no es más que una lista de bloques +
metadata. Sin embargo, una representación así no ayuda a la administración de
metadata y gestión del espacio libre. Dada la naturaleza por lo general
dinámica de los archivos, sólo poner los bloques contiguos en disco (esto hace
muy rápidas las lecturas though) puede ser mala idea porque el archivo puede
aumentaro o reducir su tamaño. Podemos reservar "de más" o utilizar menos
espacio que el asignado pero eso naturalmente nos lleva a problemas de
fragmentación. Es por esto que un esquema así no es usado en ningún FS que
permita tanto lectura como escritura.

### Versión 1 (Lista Enlazada)

![](./img/linked_list_fs.png#floating)

Claramente el problema de modificar un archivo se puede resolver mediante una
lizta en lazada de bloques. Ahora el archivo puede aumentar/disminuir en tamaño
y alcanza con buscar bloques y modificar la lista enlazada. Peeero:

- Lectura consecutiva = GOOD, **lectura aleatoria = VERY BAD**
- Tampoco puedo leerme tooodo el archivo de una porque no sé donde está un bloque hasta leer los anteriores.
- Cada bloque desperdicia espacio indicando dónde está el siguiente bloque

### Versión 2 (FAT)


<div style="display: flex">
<div style="flex-grow: 1; margin-right: 5%">

| Bloque   | Siguiente    |
|--------------- | --------------- |
| 0   | vacío   |
| 1   | 2   |
| 2   | 5   |
| 3   | 8   |
| 4   | 3   |
| 5   | 7   |
| 6   | vacío   |
| 7   | 9   |
| 8   | -1   |
| 9   | -1   |
| 10   | vacío   |
| 11   | vacío   |
| 12   | vacío   |
| 13   | vacío   |

</div>


<div style="flex-grow: 1">
  <p>
  Tomando lo que ya tenemos de la V1, FAT le da una vuelta de tuerca. En lugar de
  guardar los punteros al siguiente bloque en el mismo bloque, me guardo una
  tabla que tenga el bloque que le sigue a otro. Veamos un ejemplo:

  - El archivo A está en los bloques 1, 2, 5, 7 y 9 
  - El archivo B en los bloques 4, 3 y 8.

  FAT usa este método con algunas variantes, en particular hay algunas
  direcciones que están reservadas (para marcar un bloque libre, uno reservado,
  uno con un sector defectuoso, último bloque del archivo). 

  - Soluciona los problemas previos porque no desperdicio espacio del bloque y puedo leerlos fuera de orden, y taaaaan ineficiente no es la lectura no secuencial
  - Pero tengo que tener toda la tabla en memoria, lo cual puede ser mucho para discos muy grandes.
  - También tengo una única tabla... mucha contención del recurso.
  - Tampoco maneja seguridad
  - Y necesito tener la tabla en memoria, no es robusto (sorprendentemente esto se usaba en windows hasta hace poco por default)
  </p> 
</div>

</div>

## Versión 3 - FS en sistemas UNIX e **Inodos**

Los **Inodos** son la estructura fundamental. Cada archivo cuenta con al menos un inodo, y se estructuran de la siguiente manera:

| Inodo  |
| :---------------: |
| **Atributos** <br>(Tamaño, permisos, etc.)   |
| **Direcciones a unos pocos bloques** <br> (puedo acceder directo, para archivos pequeños)   |
| **Single Indirect Block <br>(Puntero)**   |
| **Double Indirect Block <br>(Puntero)**   |
| **Triple Indirect Block <br>(Puntero)**   |

Donde:

- El Single Indirect Block es un bloque con punteros a bloques de datos (cubre archivos de hasta 16MB)
- El Double Indirect Block apunta a una tabla de Single Indirect Blocks (cubre archivos de hasta 32MB)
- El Triple Indirect Block apunta a un bloque de Double Indirect Blocks (cubre archivos de hasta 70TB)

![](./img/unix_inode.png#floating)

Esta estructura agrega complejidad pero resuelve algunos de los problemas de los modelos anteriores:

- Permite cargar en memoria las tablas correspondientes a los archivos abiertos
- Una tabla por archivo significa que hay mucha menos contención (tiene sentido
  que haya contención si quiero modificar el mismo archivo desde dos procesos
  distintos)
- Además el modelo es más consistente: sólo tengo cargada en memoria las cosas correspondientes a archivos abiertos.

```admonish note title="Observación"
El uso de inodos introduce una nueva noción de tamaño, cosa que con FAT no pasaba. Ahora tenemos:

- el tamaño de el archivo respecto a **cuántos datos tiene**
- el tamaño del archivo respecto a **cuánto espacio en disco ocupa**
```

### Metadata

Cuando hablamos de metadata estamos incluyendo los inodos (o las estructuras que use el OS en su lugar) pero además otra info, por ejemplo:

- Permisos
- Tamaños
- Propietarios/s.
- Fechas de creación, modificación, acceso.
- Bit de archivado.
- Tipo de archivo (regular, dispositivo virtual, pipe, etc.)
- Flags.
- Conteo de referencias.
- CRC o similar para chequeo/arreglo de errores.

### Implementando directorios con Inodos

Un directorio también es un archivo. A esta altura nos podríamos preguntar si
está pasando un archivo-ception donde tódo es un archivo. La respuesta es sí.

Retomando, al ser un archivo, un directorio tiene que tener su propio inodo.
Dentro del bloque se guarda la lista de pares de inodo y nombre del
archivo/directorio (probar hacer `vim <ruta a un directorio>`,  qué nos
muestra?)

```admonish note collapsible=true
También se puede implementar con una tabla de hash. Cuando computo el hash
sobre el nombre del archivo obtengo la entrada correspondiente al archivo. De
esa forma se pueden optimizar los tiempos de búsqueda para archivos.

El mayor problema de la tabla de hash es la dependencia de la función de hash y
el tamaño fijo de la misma. Para eso se puede optar también por la alternativa
de tabla de hash que usa chaining, que si bien menos eficiente mejora los
tiempos de búsqueda en comparación a la alternativa inicial. 
```

### Dónde están los inodos en el disco?

![](./img/hdd_inodes.png#center)

### Links

Como mencionamos antes, es normal que hoy en día los sistemas operativos
permitan armar links entre archivos. Tenemos 2 tipos de loinks que podemos
armar:

- link (a secas): accedo al archivo posta
- link simbólico: referencia al archivo posta (en lugar de guardar `#{nombre}
  #{inode_id}` guardo `#{nombre} #{path_to_file}`)

Agregar links ciertamente introduce complejidad a la estructura del sistema de archivos. Es por eso que hay algunos problemas a considerar:

1. Un archivo con links (dependiendo del tipo) puede tener muchas rutas
   absolutas. Si estamos recorriendo el sistema para contar estadísticas,
   deberíamos tener eso en cuenta para no contar por duplicado. Más aún, en el
   caso de directorios no queremos recorrer 2 veces el mismo directorio (o
   loopear infinitamente), por lo que es necesario saber si ya pasé por un
   directorio o no.
2. Otro problema que surge es el del borrado
  - Si borro un enlace simbólico no debería no debería afectar al original,
    sólo borro el link.
  - Si borro el archivo original hay que considerar qué hacer. EN el caso de
    UNIX, los enlaces simbólicos se mantienen (no se borran), y es
    responsabilidad del usuario darse cuenta de que hay "referencias huérfanas"
    (al momento de acceder al archivo el OS se puede dar cuenta de que no
    existe más el original".
  - UNIX cuenta con un tipo de link más, **hard links** que lo que hacen es
    mantener en el inodo la cuenta de cuántos links tiene. Si borro el archivo
    original o un link, decremento el contador. Sólo libero el bloque si el
    count llega a 0 (es una forma de hacer reference counting). No hace falta
    mantener la lista de links que apuntan al inodo.

## Manejo del espacio libre

Otro problema que le concierne al filesystem es el del manejo del espacio
libre. Cómo trackeo las partes que están libres y las que no?

- Una forma es usar un mapa de bits (empaquetado), pero eso requiere tener el vector en memoria (malo).
- Al igual que con la memoria, podríamos tener una lista enlazada de bloques libres
- En general se **clusteriza*. O sea si un bloque de disco puede tener *n* punteros a otros bloques, los primeros n - 1 indican bloques libres y el último es el puntero al siguiente nodo de la lista.
  - Se puede mejorar haciendo que cada nodo indique cuántos bloques libres consecutivos le siguen.

## Uso de caché

Una forma de mejorar el rendimiento del FS es usando una caché. Puedo usarla
tanto para cachear inodos como datos. Lo que hacemos es copiar en memoria
algunos bloques del disco. Se maneja de manera similar a las páginas (al punto
de que se suele usar un caché unificado para ambas).

## Consistencia

Para situaciones en las que la computadora deja de funcionar antes de bajar los
cambios a disco, como no queremos perder datos el SO tiene que proveer algún
mecanismo para no perder dichos datos. Dentro de las herramientas que el sistema provee están:

- puedo hacer write through (que escriba a disco directamente), pero tiene un impacto grande de perf.
- syscall `fsync()`, que hace que se graben en disco todo lo que haga falta
  (las páginas "dirty" del caché)
- usar `fsck`, que es un programa que restaura la consistencia del FS. Recorre
  todo el disco y por cada bloque cuenta cuántos inodos le apuntan y cuántas
  veces aparece referenciado como libre. Dependiendo de esos valores y si es
  posible, se toman acciones correctivas.
- otro técnica, consiste en usar un bit que denota si se apagó normalmente el sistema:
  - Cuando inicia el sistema y ese bit está apagado, el apagado no fue normal. En ese caso se corre `fsck`

### Journaling

Otra técnica más para asegurar consistencia es **journaling**: el OS lleva un
log o journal que tiene los cambios que hay que hacer en disco. Cuando se baja
la caché a disco se marca que esos cambios están listos. Después de X datos
escritos (se detecta porque se llena un buffer) también se baja a disco.

Como principal ventaja está que es mucho menos trabajo que correr `fsck`.
Cuando el sistema inicia, se aplican los cambios del buffer que no se hayan
logrado bajar a disco. Además, las escrituras a disco son en bloques de
operaciones, mucho mejor que hacer escrituras secuenciales.

Esto no viene sin su cuota de impacto en performance, pero a pesar de eso es
una técnica usada en la mayoría de sistemas operativos.

## Características avanzadas de OS

- cuotas de disco
- encripción
- snapshots
- manejo de raid por software (más control, menos perf, se puede obtener más redundancia)
- compresión

## Network File System (NFS)

El nfs es un protocolo que permite acceder a FS remotos como si fueran locales
mediante RPC. La idea es que "montamos" el filesystem remoto en algún punto del
sistema local y todo lo que acceda ahí no sabe si es algo local o remoto. Para
soportar esto, los OS tienen una capa llamada **Virtual File System (VFS)**.

La VFS tiene nodos virtuales por cada archivo abierto. Si es algo local son
inodos, si es remoto se usa otra estructura. Dependiendo de si un pedido es
local/remoto el VFS despacha el pedido al FS real o a un cliente de NFS que
conoce el protocolo de red a utilizar.

Del lado del cliente es necesario un módulo del kernel específico pero del
servidor basta con un programa común y corriente.

## Ext2

Ext2 es un filesystem utilizado en los sistemas linux (hoy en día existen Ext3
y Ext4, que incorporan journaling). Veamos cómo se guarda la info en disco. El espacio se separa en **block groups**. Cada block group tiene:

- el superblock, tamaño 1 bloque: tiene metadata crítica para el FS. Algunos de los datos que tiene son:
  - block group number
  - block size
  - blocks per group
- group descriptors, tamaño *n* bloques
  - tiene para cada block group la info de:
    - nro de bloque del block bitmap
    - nro de bloque del inode bitmap
    - nro de bloque del inode table
    - nro de bloques libres, inodos libres
- data block bitmap, tamaño 1 bloque
- inode bitmap, tamaño 1 bloque
- inode table, tamaño *n* bloques
- data blocks, tamaño *n* bloques

![](./img/ext_2_fs.png#center)

```admonish info
Por lo general, sólo se lee el superbloque del block group 0, pero el resto de
los block group tienen una copia para casos de corrupción de memoria. Ocurre lo
mismo para los group descriptors.
```
