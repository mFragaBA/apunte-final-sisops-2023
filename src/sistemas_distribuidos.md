# Sistemas Distribuidos

A los ojos de la materia, un sistema distribuido es un conjunto de recursos conectados que interactúan. Esto puede ser:

- Varias máquinas conectadas por red
- Un procesador con varias memorias
- Varios procesadores que comparten una (o más) memoria(s).

Una extensión a esa definición que a mi me gusta es que el sistema distribuido **hacia afuera se comporta como si fuese "un único recurso"**

Qué ventajas nos puede traer un sistema distribuido vs. el esquema que veníamos usando?

- Paralelismo: ojo, esto no siempre ocurre. Ej: cualquier sistema de cómputo
  paralelo. Mando un algoritmo paralelo a correr en 4 máquinas y que en una "se
  junte" el resultado.
- Replicación: esta está muy bien. Como es el caso de bitcoin, que se caiga un
  nodo, dos nodos, veinte nodos no me tira el sistema.
- Descentralización: No quiero un único punto de falla. O tampoco quiero que el
  sistema lo rija una única entidad.

Qué desventajas puede tener un sistema distribuido?

- La sincronización es difícil
- Mantener coherencia es difícil
- muy rara vez se comparte clock lo cual nos restringe a no necesariamente
  poder confiar en timestamps o heartbeats. A menos que tengas toda la papa y
  seas como google que compra relojes atómicos.
- Muchas veces cada "participante" tiene una visión parcial de todo lo que ocurre

## Sistemas distribuidos mediante memoria compartida

Por HW:

- UMA
- NUMA
- Híbrida

Por SW:

- Estructurada
  - Memoria asociativa
  - Arrays distribuidos
- No estructurada
  - Memoria virtual global.
  - Memoria virtual particionada por localidad

## Cuando no hay memoria compartida

Como dijimos, no tener clocks hace las cosas difíciles. Pero **no tener clock y
memoria compartida** hace las cosas muuuucho muy difícil. De todos modos hay
algunas alternativas:

- Telnet (sólo para conectarse a otro equipo)

![](./img/rpc.png#floating)

- RPC
  - permite a los programas hacer **procedure calls* de manera remota, también enviar datos.
  - existen bibliotecas que ocultan al programador los detalles de la comunicación
  - importante: es un mecanismo **sincrónico**

Para generalizar un poco, estos métodos tienen en común la forma de cooperación
de solicitar un servicio a otro. El otro servicio no tiene un rol activo además
de proveer el servicio. Estas arquitecturas son las que se conocen como
**cliente/servidor**.

## Mecanismos asincrónicos

Hasta ahora vimos:

- con memoria compartida
- con cliente/servidor (sincrónico)

Ahora veamos las formas de comunicación asincrónica:

- RPC asincrónico en sus distintos colores:
  - Promises
  - Futures
  - Otros(?)
- Pasaje de mensajes (`send` / `receive`)

Pasaje de mensajes es el mecanismo más general, ya que no asume nada más allá
de tener un canal de comunicación. Este modelo si bien simple, viene con una
serie de problemas a considerar:

- encoding/decoding de los datos
- la comunicación puede ser muy lenta
- se pueden perder mensajes (TCP/IP atenúa mucho la posibilidad de que esto suceda)
- enviar mensajes puede tener un costo económico
- Los nodos pueden morir
- La red se puede partir

Los últimos dos problemas los vamos a ignorar pero son mucho muy reales llevado a la práctica.

Aparece también la noción de **complejidad medida sobre la cantidad de mensajes que intercambian**.

```admonish info title="Conjetura de Brewer"
En un entorno distribuido no se puede tener a la vez consistencia,
disponibilidad y tolerancia a fallas todas al mismo tiempo. A lo sumo 2 de las
3.

Esto de hecho fue demostrado en 2002 por Seth Gilbert y Nancy Lynch con lo cual
es un teorema más que una conjetura, pero como teorema se lo conoce como el CAP
theorem.
```

## Locks en entornos distribuidos

En entornos distribuidos no tenemos un `TestAndSet` atómico. Es por eso que tenemos que buscar alternativas. Podemos distinguir a grandes rasgos dos enfoques:

- Un enfoque centralizado, en donde tenemos un nodo que hace de coordinador entre los recursos.
  - Poco resiliente, hay un punto único de falla
  - El coordinador se transforma en un cuello de botella del procesamiento y la capacidad de la red (no siempre)
  - Tengo que recurrir al coordinador (que puede estar lejos), para acceder a un recurso que puede estar al lado mio.
- Un enfoque distribuido en donde los procesos "negocian" recursos

## Locks descentralizados

La analogía que usamos para esto es el "canto guerra pri". El tema es que en un
entorno distribuido, saber quién cantó pri es difícil. Quién gana? El que
primero mandó el mensaje? El que logró que la mayoría reciba su mensaje? Si hay
empate, elijo del te timestamp más chico? Cómo comparo tiempstamps si es
difícil coordinar clocks?

Esto último nos lo responde nuestro queridísimo amigo Leslie Lamport, que dice
que nos tendría que... something something un huevo something sincronizar
relojes. Lo importante es saber si un evento ocurre antes que otro o no. Para
eso se define el siguiente orden parcial entre eventos:

- Si dentro de un proceso, \\( A \\) sucede antes que \\( B \\), entonces \\( A \rightarrow B \\).
- Si \\( E \\) es el envío de un mensaje y \\( R \\) su recepción, \\( E \rightarrow R \\), aunque sucedan en distintos procesos.
- Es transitiva (si \\( A \rightarrow B \\) y \\( B \rightarrow C \\), entonces \\( A \rightarrow C \\)).
- Si no vale ni \\( A \rightarrow B \\) ni \\( B \rightarrow A \\), entonces \\( A \\) y \\( B \\) son concurrentes.

### Implementación

- Cada procesador tiene un reloj (lo único importante es que sea monótono
  creciente)
- Cada mensaje lleva el ts del reloj
- Como la recepción siempre es posterior al envío, cuando se recibe un mensaje
  en tiempo `t` mayor al tiempo de nuestro reloj, actualizo nuestro reloj al
  tiempo `t + 1`.
- Lo único que falta resolver son los empates, que al ser concurrentes la
  resolución puede ser arbitraria (por ejemplo, decido por PID)

## Acuerdo Bizantino

Es el perfecto ejemplo de que cuando puede haber pérdida de mensajes es difícil ponerse de acuerdo.

> La idea del problema bizantino es que dos generales en campamentos de
> distintas ubicaciones se quieren poner de acuerdo para atacar una ciudad.
> Para eso, uno manda un mensajero (pero ojo, puede ser interceptado por la
> gente de la ciudad y ser asesinado) que le dice la hora de ataque al otro
> general. El otro general va a responderle, o bien estando de acuerdo con la
> hora o proponiendo otra. El problema está que cuando uno responde no tiene
> forma de saber que la respuesta llegó a menos que el otro le responda, pero
> en ese caso el otro no puede estar seguro de que éste recibió el mensaje.

### Formalización

Dados:

- Potenciales **fallas en la comunicación**
- Valores: \\( V = \lbrace 0, 1 \rbrace \\)
- Inicio: Todo proceso \\( i \\) arranca con algún \\( \text{init}(i) \in V \\)

Se busca:

- Acuerdo: Para todo \\( i \neq j \\), \\( \text{decide}(i) = \text{decide}(j) \\)
- Validez: Existe algún \\( i \\) tal que \\( \text{decide}(i) = \text{init}(i) \\)
- Terminación: Todo \\( i \\) termina en un número finito de transiciones (WAIT-FREEDOM)

```admonish info title="Teorema"
Teorema: No existe ningún algoritmo para resolver el problema de consenso en este escenario
```

### Variante \#1

No hay errores de comunicación, pero los procesos pueden dejar de funcionar. En este caso pedimos que los procesos que no fallen terminen en un número finito de transiciones. En ese caso consenso se puede resolver con \\( \mathcal{O}((k + 1) * N^2) \\) mensajes.

### Variante \#2

En este caso lo que suponemos es que los procesos no son confiables (es como si reemplazaran al mensajero por uno de la ciudad). Se puede resolver consenso bizantino para \\( n \\) procesos y \\( k \\) fallas si y sólo si \\( n > 3 * k \\) (falla menos de un tercio de los nodos) y la conectividad es mayor que \\( 2 * k \\).

## Scheduling en sistemas distribuidos

- local: (lo que ya vimos). Le doy el procesador a un proceso listo
- global: asigno un proceso a algún procesador/recurso. Ej: un load balancer ponele recibiendo requests

Cuando es global, comparto la carga entre procesadores/equipos:

- cuando es estática, determino el procesador al crear el proceso (o recibir el request)
- cuando es dinámico asigno durante la ejecución el procesador (puede requerir migrar, lo cual es caro)
  - puedo triggerear la migración cuando un procesador está muy cargado (sender initiated) o cuando está muy libro (work stealing)

La política de scheduling se va a encargar de:

- Transferencia: cuándo hay que migrar un proceso
- Selección: elegir qué proceso migrar
- Ubicación: saber dónde migrar un proceso
- Info: cómo se difunde el estado del scheduler (los procesadores, las tareas, etc.)
