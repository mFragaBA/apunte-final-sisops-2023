# Entrada/Salida

Nos vamos a concentrar en dispositivos de almacenamiento. Dentro de estos se incluyen:

- Discos Rígidos (hoy también discos sólidos).
- Unidades de cinta: principalmente usados para backup
- Discos removibles: cd, dvd, disquette, usb, etc.
- discos virtuales (también llamados NAS: Network Attached Storage): NFS, CIFS,
  DFS, AFS, Coda
- Storage Area Network (SAN): similar a Nas, tengo el almacenamiento en red pero los protocolos que usa son específicos para este tipo de datos.

## Subsistema de I/O y Modelo

Conceptualmente un dispositivo de I/O consta de dos partes:

- El dispositivo físico
- Un **controlador del dispositivo** (ojo, no son los drivers de software si no
  de hw) que interactúa con el SO mediante algún bus o registro.

Al igual que con otros subsistemas, el sistema operativo mediante **drivers** logra abstraer detalles propios del dispositivo al usuario:

![layout subsistema I/O](./img/io_subsystem.png#floating)

- Los drivers conocen las particularidades del HW con el que se comunican.
- Por lo general los drivers son provistos por el mismo fabricante de HW que el
  dispositivo.
- Distintos modelos de un mismo fabricante pueden usar distintos drivers.
- Por ejemplo, algo a considerar por un driver puede ser qué bit hay que leer
  para marcar el final de una operación.
- Tienen un impacto muy grande sobre el rendimiento del sistema:
  - corren en máximo privilegio (a.k.a te pueden hacer bosta todo el sistema)
  - De ellos depende el rendimiento del I/O que como vimos es frecuentemente
    usado y fundamental para el rendimiento general del sistema.

## Formas de I/O

Vemos 3 formas de implementar I/O (por lo general en la práctica están disponibles las 3):

|    | Polling    | Interrupciones (o push)    | DMA (acceso directo a memoria)    |
|---------------- | --------------- | --------------- | --------------- |
| 📄 Desc. | El driver periódicamente verifica si el dispositivo se comunicó    | El dispositivo avisa mediante interrupciones    | La CPU no interviene (por lo general para transferir grandes volúmenes de info)    |
| ✅ Ventajas | Cambios de contexto predecibles   | Eventos asincrónicos poco frecuentes   |  Cuando el controlador de DMA finaliza, interrupe a la CPU (1 vs varios para comunicarse en interrupciones)  |
| 💀 Desventajas | Alto consumo de CPU   | Cambios de contexto impredecibles   | Necesitás el componente específico de HW (controlador de DMA)  |

## API de I/O

Tenemos las syscalls:

- `open()`/ `close()`
- `read()` / `write()`
- `seek()`

Ocultan bastante de la complejidad, aunque hay algunos detalles que se exponen (ej: si se obtiene acceso exclusivo al dispositivo o no)

## Tipos de dispositivos

Los dispositivos pueden separarse en 2 grupos:

- **char device**: 
  - la info se transmite byte a byte, debido a eso no tienen acceso aleatorio y utilizan caches para mejorar la performance.
  - ej: mouse, teclado, terminales, puerto serie.
- **block device**:
  - se transmite info en bloque, permite acceso aleatorio y en general usan un buffer.
  - ej: disco rígido, memoria flash, cd rom

Si bien esta es una clasificación general, la comunicación con dispositivos tiene otras variables:

- Si es lectura, escritura o lecto-escritura
- Si es compartido o dedicado
- Si es sincrónico o asincrónico
- La velocidad de respuesta del dispositivo

Parte del objetivo del SO y la api de I/O es ocultar la mayor cantidad de
detalles posibles y a la vez brindar acceso consistente a toda la fauna de
dispositivos.

~~~admonish info title="Dispositivos en Linux" collapsible=true
Los dispoistivos en linux están representados por archivos y se ubican en el directorio `/dev`. En el siguiente ejemplo, podemos ver que el archivo contiene información del tipo de archivo (c para char device, b para block device):

```bash
ls -lh /dev
crw-rw-rw-  1 root    wheel        0x9000001 Mar  5 12:06 cu.wlan-debug
brw-r-----  1 root    operator     0x1000000 Mar  5 12:06 disk0
brw-r-----  1 root    operator     0x1000001 Mar  5 12:06 disk0s1
brw-r-----  1 root    operator     0x1000002 Mar  5 12:06 disk0s2
brw-r-----  1 root    operator     0x1000003 Mar  5 12:06 disk0s3
```
~~~

~~~admonish info title="API de I/O en linux"
En el caso de linux, bajo la premisa de que *"todo es un archivo"* se proveen funciones de alto nivel para acceso a archivos:

- `fopen()`, `fclose()`
- `fread()`, `fwrite()`: modo bloque
- `fgetc()`, `fputc()`: modo char
- `fgets()`, `fputs()`: modo char stream (ej: con esto puedo escribir en consola)
- `fscanf()`, `fprintf()`: modo char con formato
~~~

## Planificación de I/O

En el caso del disco, una de las claves para obtener un buen rendimiento es
optimizar los accesos al mismo. Esto es porque el disco consiste de una cabeza
que se mueve, y eso lleva tiempo. Si junto varios pedidos y los ordeno de
manera tal que minimice la cantidad de movimientos voy a mejorar mucho la
performance.

La planificación de disco entronces se trata de cómo manejar la cola de pedidos
de I/O para lograr el mejor rendimiento posible. Es un equilibrio entre ancho
de banda, latencia rotacional y **tiempo de búsqueda (seek time)**, que es el
tiempo necesario para que la cabeza se ubique sobre el cilindro que tiene el
sector deseado.

### Políticas de scheduling de I/O a disco

Tenemos algunos esquemas:

- FIFO
  - problema: estoy moviendo la cabeza de un lado a otro, a menos que los
    pedidos "se porten bien".
- Shortest Seek Time First (SSTF)
  - idea: atiendo como próximo al pedido más cercano a la posición actual de la
    cabeza (o sea es un algoritmo goloso)
  - si bien mejora el tiempo de respuesta, puede producit inanición
- **Algoritmo scan o del ascensor**
  - idea: ir primero en un sentido, atendiendo los pedidos en el camino. Luego
    vuelvo y hago lo mismo.
  - es una mejora, aunque si me piden un sector por donde acaba de pasar la
    cabeza tardo muuuuucho.
  - además no es tan uniforme (predecible) el tiempo de espera
- **C-Scan**: igual a scan pero al llegar al final vuelve al principio sin
  atender otros pedidos (asume que el disco es una lista circular)

```admonish info title="Nota"
En la práctica ninguno de estos algoritmos se usan al 100%, suele ser una mezcla entre estos, prioridades, caches, etc.
```

## SSD

Hoy en día prolifera el uso de discos de estado sólido (SSD). Si bien tienen notables ventajas (más resistentes, menor consumo, más silenciosos, mejor performance de lectura), también presenta sus propios problemas:

- durabilidad
- write amplification

<!-- TODO: expandir esto -->

## Gestión del disco

### Formateo

Consiste en poner por cada sector códigos para detección de errores,
puntualmente un prefijo y un postfijo. Si al leer un sector, el prefijo y
postfijo no tienen el valor esperado es porque el sector está dañado.

### Booteo

Las computadoras suelen tener un programa en ROM que carga algunos sectores del
principio del disco, y los comienza a ejecutar (bootloader?). Dicho programa es
muy pequeño y solamente sirve para cargar el OS, no es un OS en sí mismo.

### Manejo de bloques dañados

Hay distintas formas de atajarnos:

- por software, si hacemos que el sistema de archivos se encarge de la gestión
  de bloques dañados
- por hardware:
  - hay discos que vienen con sectores extra para reemplazar a los defectuosos.
    A veces algunos discos traen sectores extra en todos los cilindros
  - cuando la controladora detecta un bloque dañado puede actualizar una tabla
    interna de remapeo para usar un sector distinto (esto puede interferir con
    las optimizaciones del scheduler de I/O)

## Spooling (Simultaneous Peripheral Operation On-Line)

Spooling permite interactuar con dispositivos que requieren acceso dedicado sin
bloquear al proceso que los necesita. El ejemplo más distintivo es el de la
impresora. Cuando uno imprime, en lugar de bloquearse el proceso hasta tener
acceso, se encola el trabajo en una cola específica y se tiene un proceso que
se encarga de desencolarla a medida que la impresora (o el dispositivo en otros
casos) se libera.

```admonish info title="Nota"
En el caso de spooling, el usuario es consciente de que ocurre pero el kernel no.
```

## Protección de la información

Vamos a ver algunas técnicas que podemos emplear para proteger nuestros datos.
Muchas veces dependiendo del tipo de dato, del valor que le asignemos y el
costo de mantenerlo vamos a optar por una u otra manera. Con proteger nuestros
datos nos referimos a por ejemplo que el disco falle y se descomponga algún
sector (o el disco entero), a que tengo una aplicación que está corriendo 24/7
y tengo que tener tolerancia a fallas, etc. Las dos técnicas que vamos a ver son:

- copias de seguridad
- redundancia

```admonish note
En particular **la redundancia no nos protege de la modificación/eliminación
accidental de la información**. Es por eso que se suelen combinar ambas técnicas.
Hay casos sin embargo en donde ninguna de las técnicas nos salva (en algún
momento corrompemos datos/archivos y nos damos cuenta tarde con varios backups
encima). En esos casos los sistemas de archivos nos deberían brindar algunas
protecciones para que eso no pase (o al menos no pase seguido).
```

### Copias de seguridad

Hacer una **copia de seguridad (backup)** consiste en resguardar los datos
(obviamente solo los importantes) en otro lado. Para grandes volúmenes de info,
se suele hacer en cinta o discos duros (Si es algo hogareño podés hacerlo con
el dispositivo de almacenamiento externo que te plazca). De vuelta, en caso de
sistemas con mucha data, se suelen programar los backups para que se hagan de
noche.

Sin embargo, copiar **todos** los datos es muy caro. Por eso podemos implementar distintas estrategias:

- hacer una **copia total** con cierta frecuencia (cada semana, mes, bimestre, etc.)
- hacer una **copia incremental** (por ejemplo cada noche): sólo guarda los archivos modificados **desde la última copia incremental**
- hacer una **copia diferencial**: sólo guarda los archivos modificados desde la última copia total (obviamente combino esto con hacer una copia total cada tanto).

Cuando llega la hora de restaurar:

- si sólo hago copias totales restauro la que quiero y listo

\\[
\text{Hoy} = \text{último total}
\\]

- si hago copias diferenciales, agarro la última copia total y aplico la última
  copia diferencial

\\[
\text{Hoy} = \text{último total} + \text{último diferencial}
\\]

- si hago copias incrementales, agarro la última copia total y aplico desde la
  primera incremental después de la total hasta la última (en ese orden)

\\[
\text{Hoy} = \text{último total} + \sum_i  \text{incremental}_i
\\]

### Redundancia

El uso de redundancia es clave en sistemas que tienen que estar up 24/7, de
forma tal que las fallas en discos no te tiren el sistema (por supuesto, puede
caer un meteorito en cada datacenter del planeta y ahí cagaste)

Un método muy usado es **RAID**: **R**edundant **A**rray of **I**nexpensive **D**isks. La técnica fue evolucionando por lo que tenemos RAID en sus distintas versiones:

Podemos resumir en la siguiente tabla las distintas versiones (las más usadas en la práctica) de RAID:

<!-- TODO: armar tabla -->

![](./img/raid_0.png#floating)

- **RAID 0 (stripping)**
  - necesito 2 discos
  - Parto el archivo (o data) en pedacitos y mando algunos pedacitos a un disco y otros pedacitos a otro
  - en realidad no aporta redundancia, pero mejora el rendimiento (mejor ancho de banda y permito escrituras en paralelo)
<!-- needed for images not to overlap -->
<br><br><br><br>

![](./img/raid_1.png#floating)

- **RAID 1 (mirroring)**
  - 2 discos
  - espejo la data, si se cae un disco entero todavía tengo la copia en el otro
  - mejora el rendimiento de las lecturas pero en peor caso tardo el doble en escribir (muy caro).
<!-- needed for images not to overlap -->
<br><br><br><br><br>

- **RAID 0+1**

![](./img/raid_0_plus_1.png#center)

-
  - 4 discos
  - combina los dos anteriores: cada archivo está espejado, pero al leer leo un bloque de cada disco.
  - leo como si usara stripping en lugar de sólo mirroring, pero al escribir hay que escribir en cada bloque de ambos.

- **RAID 2 y 3**
  - La idea es guardar por bloque info suficiente para determinar si se dañó o no, y a veces puedo corregir errores con esa info.
  - sigo distribuyendo los bloques entre todos los discos participantes
  - nro. discos
    - RAID 2: 3 de paridad por cada 4 de datos
    - RAID 3: 1 disco de paridad por cada 4 de datos
  - todos los discos participan de todo I/O, por lo que es más lento que incluso RAID 1
  - Requiere mucho cómputo recalcular redundancias
  - si se implementa es por HW

- **RAID 4**
  - como RAID 3 pero usa stripping por bloque (cada bloque va a un único disco)
  - el bottleneck sigue siendo el disco dedicado a paridad porque todo write tiene que tocarlo

- **RAID 5**

![](./img/raid_5.png#halfcenter)

-
  - Usa datos redundantes pero los distribuye en N + 1 discos (cada write consecutivo va a discos distintos)
  - para cada bloque, un disco tiene la data y otro tiene info de paridad (hay info de paridad en todos los discos)
  - ya no tengo cuello de botella para los writes, pero mantener la paridad no es fácil
  - cae el rendimiento en la reconstrucción de info

- **RAID 6**

![](./img/raid_6.png#halfcenter)

-
  - como RAID 5 pero agrega un segundo bloque de paridad también distribuido, puede soportar la rotura de hasta 2 discos
  - No hay diferencia sustancial vs RAID 5 respecto del espacio desperdiciado (en la práctica se suele usar RAID 5 + hot spare)

## Miscelaneos

### RAM como dispositivo

Es posible crear **RAM Drives**, que consisten en separar una porción de la
DRAM del sistema y presentarlo al resto del sistema como si fuera otro
dispositivo de almacenamiento. Por qué tendría sentido hacer esto? Si bien el
SO usa caches y buffers para por ejemplo optimizar operaciones de I/O, esto le
permite **al usuario** guardar data en la memoria usando operaciones con
archivos. Si bien los dispositivos NVM (ej: SSD) son rápidos, la ram sigue
siendo mucho más rápido.

### Flujo de un pedido de lectura/escritura a disco

1. Cuando un proceso necesita **I/O** llama a una syscall del sistema
   operativo. La syscall tiene varios parámetros:

- tipo de operación (Input o Output)
- el file descriptor indicando el archivo sobre el que se opera
- la dirección de memoria de donde transferir
- la cantidad de data a transferir

2. Si el disco y su controlador están disponibles, el pedido se atiende
   inmediatamente. Si no, los pedidos se guardan en una cola de pendientes para
   ese disco. Como vimos antes, el scheduler de I/O puede ordenar esa cola con
   fines de mejorar la performance.

```admonish info
En el pasado, era necesario especificar el track y qué cabeza del HDD usar, y
los algoritmos de scheduling de disco eran más complejos. Hoy en día mediante
el uso de LBAs (Logical Block Addresses) incluso el SO es abstraido de lo que
en realidad pasa en la memoria. Sin embargo, se sigue asumiendo que LBAs
cercanas significan accesos cercanos (físicamente hablando). Por lo tanto sigue
teniendo sentido tener consideraciones como:

- lecturas/escrituras en bloque
- fairness
- timeliness
```

### El **deadline scheduler** de linux

Los algoritmos que vimos de scheduling de I/O tienen una desventaja muy grande,
y es que pueden producir inanición. Linux para resolver esto implementó el
**deadline scheduler**. 

Este scheduler mantiene 4 colas, dos de lectura y dos de escritura. La primera
cola es aquella ordenada por LBA (implementando en cierta forma un C-Scan), y
la segunda es una FIFO a la que se mandan todos aquellos pedidos que superan el
tiempo límite configurado (500ms por default).
