# Sistemas Distribuidos - parte II

## Modelo de fallas

Para trabajar con algoritmos distribuidos es importante definir el modelo de fallas, por ejemplo:

- Nadie falla
- Los procesos se pueden caer pero no levantarse
- Los procesos se pueden caer y levantar
- Los procesos se pueden caer y levantar pero sólo en determinados momentos
- La red se particiona
- Los procesos se pueden comportar de manera arbitraria (procesos "bizantinos"/ hay fallas bizantinas)

Cada una te fuerza a hacer algoritmos distintos (y como vimos antes, hay
problemas que bajo un modelo se pueden resolver pero bajo otro no). Nosotros
nos concentramos más que nada en los modelos sin fallas.

Los algoritmos distribuidos/paralelos están buenísimos pero tienen un problema,
o en realidad el problema es nuestro. Desde algo I armamos un marco teórico
para analizar la complejidad de los problemas, que en entornos
paralelos/distribuidos no tienen sentido. Para elegir algo que tiene más
sentido, tomamos lo que por lo general resulta el cuello de botella en
algoritmos distribuidos: el pasaje de mensajes (a través de la red, en cnpt).
En particular vamos a medir la complejidad según la cantidad de mensajes que se
envían a través de la red.

Algunas variantes que también se pueden analizar:

- el tamaño de la red
- la cantidad de procesos
- cómo ubicar a cada proceso

Los problemas que vamos a tratar pertenecen a 3 clases de problemas:

- Orden de ocurrencia de eventos
- Exclusión mutua
- Consenso

## Exclusión mutua distribuida

En su forma más sensilla, el problema también se conoce como **token passing**.
La idea es armar un anillo lógico entre procesos y poner a circular un token.
Cuando tengo el token, es como si entrara a la sección crítica. En un modelo en
donde no hay fallas esto te sirve porque no hay inanición. Por otro lado, en
redes grandes, es posible que tengas que esperar mucho hasta tu turno, aunque
nadie más necesite el token. También estoy generando mensajes aunque no haga
falta.

Algunas de las implementaciones:

- Fiber Distributed Data Interface (FDDI).
- Time-Division Multiple-Access (TDMA).
- TImed-Triggered Architecture (TTA).

### V2: pedidos en lugar de pasar el token

CUando quiero entrar a la sección crítica envío a todo el mundo (el mismo
proceso inclusive) el mensaje `solicitud(P_i, ts)`, siendo `ts` el timestamp.
Cada proceso puede responder inmediatamente o encolar la respuesta. Si todos
los procesos me responden, puedo entrar a la sección crítica. Si entro, al
salir respondo a todos los pedidos demorados.

Respondo `SI` cuando:

- no estoy intentando entrar a la sección crítica
- quiero entrar, todavía no lo hice, y el `ts` del pedido que recibo es menor que el mío (el otro tiene prio).

Si bien ahora requiero que todos conozcan a todos (antes sólo hace falta conocer a 2), no circulo mensajes si nadie quiere entrar a la sección crítica.

~~~admonish note title="Pseudocódigo" collapsible=true

```rust
fn procesar_solicitud(pid, ts) {
  if not in_seccion_critica or ts <= seccion_critica_ts {
    responder_solicitud(pid, "SI");
  }
}

fn procesar_respuesta(pid) {
  respuesta_procesos[pid] = true;
  if all(respuesta_procesos) {
    seccion_critica()
    set_all(respuesta_procesos, false);
    // el copy sería como sacar una foto.
    // De esa manera mientras proceso esos 
    // mensajes pueden entrar nuevos
    for msg in cola_mensajes.clone() {
      procesar_mensaje(msg);
    }
  }
}

fn procesar_mensaje(msg) {
  case msg.tipo {
    SOLICITUD => procesar_solicitud(msg.pid, msg.ts),
    RESPUESTA => procesar_respuesta(msg.pid)
  }
}
```
~~~


```admonish warning title="Ojo"
Notar que estamos asumiendo que:

- No se pierden mensajes
- Ningún proceso falla
```

### Locks Distribuidos

Anteriormente vimos que tener un coordinador de locks centralizado venía con
sus problemas (bottleneck en el coordinador, punto único de falla, etc.).
Veamos ahora una versión distribuida: protocolo de mayoría.

La idea es que queremos un lock para un objeto que está copiado en \\( n \\)
lugares. Para obtener un lock, nos lo tiene que otorgar al menos \\(
\frac{n}{2} + 1 \\) nodos. Cuando pedimos el lock nos pueden responder que nos
otorgan el lock o que no. Además, cada copia del objeto viene con su número de
versión. Si lo escribimos, tomamos el número más alto y lo incrementamos en 1.
Para las lecturas también basta con leer la copia que tenga el número de
versión más alto.

La elección de la cantidad de aprobaciones que necesitamos es necesaria para
asegurar que no se otorgan 2 locks a la vez. Sin embargo, pueden haber
deadlocks. Y por eso hay que usar algoritmos de detección y resolución de esos
deadlocks (un timeout por ejemplo).

Otra cosa que podríamos preguntarnos es si puede ocurrir que lea una copia
desactualizada. Para que eso ocurra, deberían existir \\( k \geq \frac{n}{2} +
1 \\) locks cuya marca sea \\( t \\), y existir otra copia con timestamp mayor.
Pero eso significa que el que escribió tenía menos de \\( \frac{n}{2} + 1 \\)
"permisos", lo cual no debería ocurrir.


## Elección de líder

Un conjunto de procesos debe elegir a uno como *líder* para algún tipo de tarea
(por ejemplo, si quiero tener control centralizado de locks, pero quiero que el
coordinador cambie de a ratos puedo usar esto donde "lider" = "coordinador de
locks"). En una red sin fallas, es sencillo.

Lo que hago es al igual que para el problema de la sección crítica organizar
los procesos lógicamente en un anillo. Todos los procesos hacen circular un
mensaje con su ID. Cuando un proceso recibe un mensaje, comparo con su ID y
hace circular al mayor de los dos. Cuando el mensaje dio toda la vuelta es
porque ya sabemos el lider, así que hago circular un mensaje de notificación
indicando quién resultó líder (particularmente le va a tocar al lider).

Si al que le toca el lider justo se cae no se entera, puede ser un problema.
Pero para eso podemos armar elecciones periódicas y te bancás caídas de nodos.
También otras cosas a considerar son posibles particiones en la red, o la
posibilidad de tener varias elecciones simultáneas.

### Análisis de complejidad

- Tiempo: \\( \mathcal{O}(n) \\)
- Comunicación: \\( \mathcal{O}(n^2) \\) (tenés n mensajes que "dan toda la vuelta")
  - Existe cota inferior de \\( \Omega(n log n) \\)

## Instantánea Global Consistente (sería un caso de ordenar eventos)

Tenemos un estado \\( E = \sum E_i \\) siendo \\( E_i \\) la parte del estado
que le corresponde a \\( P_i \\). El estado se modifica únicamente mediante los
mensajes que se mandan los procesos entre sí. Lo que quiero es poder en un
momento dado, saber cuánto valían los \\( E_i \\) y qué mensajes había
circulando en la red.

Una posible solución es la siguiente:

- Cuando se quiere una instantánea, un proceso se manda a sí mismo un mensaje
  de `marca`.
- Cuando un proceso recibe un mensaje de `marca` por primera vez, guarda una
  copia de su estado y envía un mensaje de marca a los otros procesos. A partir
  de ese momento el proceso empieza a registrar todos los mensajes (puede ser
  cualquier mensaje, no sólo de marca) que recibe de cada vecino hasta que
  recibe `marca` de todos sus vecinos (requiere conocer la red).
- Al recibir la segunda `marca`, queda conformada la secuencia `recibidos_i_j`
  de los mensajes que recibe un proceso de otro antes de tomar la instantánea.
- El estado global es es que cada proceso está en el estado de la copia tomada
  y los mensajes son los que están en `recibidos` (si junto todos los
  `recibidos_i_j`)

~~~admonish note title="Pseudocódigo" collapsible=true
```rust
enum TipoMensaje {
  Marca,
  Mensaje
}

fn iniciar_instantánea() {
  enviar_mensaje(pid, TipoMensaje::Marca)
}

fn recibir_mensaje(msg) {
  if msg.tipo == TipoMensaje::Marca {
    if registrando_mensajes {
      marca_registrada[msg.pid] = true;
      if all(marca_registrada) {
        // Tengo la data necesaria para la instantánea
        // Acá sólo broadcasteo pero habría que juntar toda la data
        broadcastear_mensaje({pid, copia_estado, recibidos})
      }
    } else {
      // PRIMER MENSAJE
      copia_estado = estado_actual;
      registrando_mensajes = true;
      marca_registrada[msg.pid] = true;
      for vecino in vecinos {
        enviar_mensaje(vecino.pid, TipoMensaje::Marca);
      }
    }
  } else {
    if registrando_mensajes {
      recibidos[msg.pid].append(msg)
    }
  }
}
```
~~~

### Posibles usos

Este algoritmo se puede usar para:

- Detección de propiedades estables (propiedades que una vez verdaderas lo siguen siendo)
- Detección de terminación
- Debugging Distribuido
- Detección de Deadlocks

## 2PC (2 Phase Commit)

La idea es realizar una transacción de forma atómica. Varios procesos se tienen
que poner de acuerdo en que se realizó la transacción o no. Esto tiene mucho
sentido por ejemplo en bases de datos distribuidas en donde varios nodos tienen
que decidir si hacer `commit` o `abort` de una transacción.

A grandes rasgos, el algoritmo funciona en dos fases: en la primera le
preguntamos a todos si están de acuerdo. Si ahí ya es no entonces abortamos. Si
no, anotamos los que dijeron que sí y vamos anotando los que dijeron sí. Además
tenemos un timeout que nos hace abortar si faltan respuestas. Si recibimos
todos los sí, recién ahí avisamos que quedó confirmada la transacción.

### Descripción "Formal" del problema COMMIT

- Valores: \\( V = \lbrace 0 \text{(abort)}, 1 \text{(commit)} \rbrace \\)
- Acuerdo: \\( \nexists i \neq j. \text{decide(i)} \neq \text{decide(j)} \\)
- Validez:
  1. \\( \exists i. \text{init}(i) = 0 \implies \nexists i. \text{decide}(i) = 1\\) (Si uno aborta todos abortan)
  2. \\( \forall i. \text{init}(i) = 1 \land \text{no fallas} \implies \nexists i. \text{decide}(i) = 0\\) (Si no hay fallas y todos querían commitear entonces se commitea)

Y tenemos dos versiones distintas del problema:

- con Terminación Débil: Si no hay fallas, todo proceso decide.
- con Terminación Fuerte: Todo proceso que no falla decide.

~~~admonish note title="Pseudocódigo 2PC" collapsible=true

```rust
fn fase_1() {
  match pid {
    1 => {
      votes = receive_messages();
      if all(votes) {
        decide = init;
      } else {
        decide = 0;
      }
    }
    _ => {
      enviar_mensaje(1, init);
      if init == 0 {
        decided = true;
        decide = 0
      }
    }
  }
}

fn fase_2 {
  match pid {
    1 => {
      broadcast(decide);
    }
    _ => {
      if not decided {
        decide = wait_for_message(1);
      }
    }
  }
}
```

![](./img/2pc.png#center)
~~~

```admonish info title="Correctitud"
**Teorema**: 2PC resuelve COMMIT con terminación débil.

Sin embargo, no satisface terminación fuerte. Para eso se usa el algoritmo 3PC (Three Phase Commit).
```

### Variantes del problema de consenso

- *k-agreement*: \\( \text{decide}(i) \in W, \text{ tal que } |W| = k \\)
- Aproximado: \\( \forall i \neq j. |\text{decide}(i) - \text{decide}(j)| \leq \epsilon \\)
- Probabilístico: \\( Pr[\exists i \neq j. \text{decide}(i) \neq \text{decide}(j)] \leq \epsilon \\)

## Miscelaneos

### 3 Phase Commit (3PC)

![](./img/3pc.png#halffloating)

La idea del algoritmo es similar. Veamos cada ronda:

- ronda 1
  - para los procesos != 1 todo es igual.
  - para el proceso 1, si todos votan 1 entonces **pasa al estado ready** pero no decide todavía a menos que alguno haya votado 0.
- ronda 2
  - si el proceso 1 decidió 0 entonces avisa al resto, si no broadcastea el **ready**. Luego el proceso 1 decide si todavía no lo hizo.
  - Los procesos que reciben `decide(0)` deciden 0, los otros (reciben `ready`) pasan al estado **ready**.
- ronda 3
  - si el proceso decidió 1 broadcastea `decide(1)`.
  - los procesos que reciben `decide(1)` deciden 1.

```admonish info title="Detalles"
En realidad esto no alcanza para garantizar *strong termination*, garantiza que
si el proceso 1 no falla entonces hay strong termination. Para garantizar
*strong termination* hay que agregar \\( 3 * (n - 1)\\) rondas más, en donde el
proceso 2 hace lo mismo que el 1, después el 3, 4, etc (sólo cambian de estado
los que todavía no decidieron).
```

