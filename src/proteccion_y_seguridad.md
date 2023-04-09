# Protección y Seguridad

## Ojo, son cosas distintas...

Por un lado, protección se trata de los mecanismos para que nadie pueda tocar
los datos de otro. O sea poder decidir qué usuario puede hacer cada cosa.

Por otro lado, tenemos seguridad que trata de asegurarse que los usuarios sean
quien dicen ser y de impedir la destrucción/adulteración de datos.

De todos modos nosotros medio que lo tomamos en cuenta como lo mismo. Una definición más moderna de seguridad (de la información. No confundir con seguridad informática) es la preservación de las siguientes características:

- Confidencialidad
- Integridad -> Datos no adulterados
- Disponibilidad

## Subsistema de seguridad de un SO

Los sistemas de seguridad suelen tener:

- Sujetos: usuarios, grupos de usuarios (roles), procesos (también podría contar como objeto)
- Objetos: archivos, dispositivos, pipes, etc.
- Acciones: leer, borrar, escribir, sobreescribir, propagar permisos

La idea de un sistema de seguridad es determinar qué sujetos pueden realizar qué acciones sobre qué objetos.

### Las tres A que garantiza el sistema de seguridad

- Autenticación (Authentication): ver que alguien sea quien dice ser. Se usan distintas técnicas por lo general combinadas con criptografía para lograr esto:
  - Contraseñas
  - Datos biométricos (Huella digital, Patrón de circulación de sangre)
- Autorización (Authorization): qué permisos tenés y qué te permiten hacer
- Auditoría (Accounting): dejar un registro de todo lo que sucedió / que hiciste

### Algo sobre criptografía

Tenemos dos ramas usualmente confundidas. En primer lugar, la **Criptografía** es
la rama de las matemáticas e informática que se encarga de hacer que la info no
sea accedible a terceros (encripción/cifrado y desenccripción/descifrado).

Por otro lado está el criptoanálisis que se encarga del estudio de los métodos
que rompen textos cifrados cuando no tengo la clave.

Hasta 1970, la técnica que predominaba era la del uso de mecanísmos simétricos,
o sea que tienen una única clave que se usa para encriptar y desencriptar. Por
ejemplo: Cifrado Caesar, DES, Blowfish, AES.

A partir del 1970, con el uso de **mecanismos asimétricos** como RSA esto
cambió. Los mecanismos asimétricos a diferencia de sus contrapartes usan
distintas claves: una para encriptar y otra para desencriptar. Una de las dos
es una clave pública que todos conocen y la otra es la privada que sólo se
supone que la conozca yo.

Hoy en día también contamos con **funciones de has one-way**, como son MD5,
SHA1, SHA-256, etc.

Una constante que se ve en casi todos los casos es que en cada lugar hay que
hacer un tradeoff entre qué tan seguro es el protocolo que aplico vs qué tan
rápido es.

## Autenticación

### Funciones de hash

Si bien ya conocemos las funciones de hash, en cripto se suelen usar funciones de hash especiales que aseguren:

- resistencia a la preimagen: es difícil encontrar alguna preimagen de un hash
  resultante.
- resistencia a la segunda preimagen. También tiene que ser difícil dado un
  valor, encontrar otro distinto cuyo hash sea el mismo (o sea encontrar una
  colisión)

Por lo general son útiles para almacenar contraseñas (es raro que se guarde la
contraseña en texto plano porque alguien que obtenga acceso obtiene las
contraseñas), aunque se puede ir más allá y usar SALT. El uso de un SALT mitiga
el uso de algunos ataques (hash tables y rainbow tables), y consiste en meter
aleatoreidad al input de la función de hash. De esa forma puede parecer que dos
inputs iguales tienen distinta función de hash. Por ejemplo, puedo generar un
string random y appendearlo a la contraseña antes de calcular el hash. También
puedo repetir el proceso de SALTeo varias veces para reducir la chance de que
el salt sea el mismo para dos contraseñas. El salt se guarda en el servidor
junto con el hash.

Para más info, [este post](https://auth0.com/blog/adding-salt-to-hashing-a-better-way-to-store-passwords/) de auth0 está muy bueno.

### Método RSA

- tomo dos números de muchos dígitos, uno va a ser la clave pública y el otro
  la clave privada. No sólo eso si no también queremos que sean primos.
- para encriptar un mensaje, interpreto cada letra como si fuera un número y
  hago una cuenta que involucra a la clave pública.
- para desencriptar, necesito la clave privada y nuevamente hacer una cuenta.

Veamos un poco de detalle:

- tenemos los números \\( p \\) y \\( q \\) mencionados anteriormente.
- definamos \\( n = pq \\) y \\( n' = (p-1)(q-1) \\)
- elijamos cualquier entero \\( e \\) entre \\( 2 \\) y \\( n' - 1 \\) que sea coprimo con \\( n' \\).
- calculamos \\( d \\) tal que \\( de = 1 \text{ mod }n' \\) (es fácil de hacer).
- encriptar(c): calculo el resto de dividir \\( c^e \\) por \\( n \\)
- desencriptar(c): calculo el resto de dividir \\( c^d \\) por \\( n \\)

El método funciona en la práctica porque factorizar es difícil (problema NP), y
no puedo encontrar el valor de la clave privada únicamente partiendo de la
pública.

Un detalle importante es que las operaciones de encripción y desencripción son
inversas entre sí. O sea \\(Enc(Dec(M)) = M = Dec(Enc(M))\\). Esto permite como
veremos más adelante hacer firmas digitales con mi clave privada.

Para una explicación más a fondo recomiendo [este
post](https://eli.thegreenplace.net/2019/rsa-theory-and-implementation/), que
tiene desde la teoría hasta detalles de implementación y una implementación
hecha en go.

### Usando RSA para firmas digitales

Situación: quiero enviar un archivo pero quiero que estén seguros de que fui yo y nadie más. Puedo usar RSA para eso:

1. calculo un hash del documento que quiero firmar
2. encripto con mi clave privada el hash
3. Entrego el documento + el hash encriptado
4. El receptor lo puede desencriptar con mi clave pública. Si efectivamente lo pudo desencriptar es porque yo fui el autor.
5. Por último chequea que el hash coincida con el documento.

Ventajas de usar firmas digitales:

- asegura integridad (lo que firmé no se adulteró en el proceso)
- asegura no repudio: es mi firma y no puedo decir que no lo es

## Autorización

### Monitor de referencias

El monitor de referencias es el encargado de mediar sobre las solicitudes de
acceso a objetos por parte de los usuarios. Se basan en algúna política de
acceso.

![](./img/reference_monitor.png#center)

### Representando permisos

La forma más sencilla de implementar la autorización es mediante una matriz de
control de accesos, es una matriz indexada en Sujetos y Objetos y el contenido
de las celdas tienen las acciones permitidas. Si algo no está en la celda no se
puede hacer, basándonos en el principio del mínimo privilegio. Y se suelen
tener definidos permisos por defecto para cuando se crea un objeto nuevo.
También por lo general ese default cambia según el tipo de objeto.

Se puede almacenar como una matriz centralizada, o separada por filas /
columnas. Esto tiene sentido si pensamos que por ejemplo los archivos suelen
guardar lo que el usuario puede hacer consigo mismos.

### DAC vs MAC

Hay dos esquemas posibles para tomar para nuestra política de acceso:

- Discretionary Access Control: se basa en que los atributos de seguridad se tienen que definir explícitamente. O sea que el dueño decide los permisos.
- Mandatory Access Control: 
  - En este caso cada sujeto tiene un grado.
  - Los objetos creados heredan el agrado del último sujeto que los modifica.
  - Un sujeto sólo puede acceder a objetos de grado menor o igual.
  - suele implementarse en entornos de manejo de info sensible.

### DAC en UNIX

El esquema usado en UNIX es el de DAC. Veamos un ejemplo de qué tipo de permisos existen:

```bash
# En particular sólo me interesa la columna de permisos
> ls -lh
drwx------@ - -  -   -  - 
drwxr-xr-x  - -  -   -  - 
drwx------+ - -  -   -  - 
drwx------+ - -  -   -  - 
drwx------@ - -  -   -  - 
drwx------@ - -  -   -  - 
-rw-r--r--  - -  -   -  - 
drwx------  - -  -   -  - 
drwx------+ - -  -   -  - 
drwx------+ - -  -   -  - 
drwxr-xr-x+ - -  -   -  - 
drwxr-xr-x  - -  -   -  - 
drwxr-xr-x  - -  -   -  - 
drwxr-xr-x  - -  -   -  - 
drwxr-xr-x  - -  -   -  - 
drwxr-xr-x  - -  -   -  - 
```

Qué significan esas letras y símbolos?

![](./img/basic_permissions_unix.png#center)

Uno de los ejemplos:

- permisos: `drwxr-xr-x`
- permisos de dueño: `drwx` -> es un directorio, el dueño puede leer, escribir y ejecutar
- permisos de grupo: `r-x` -> el grupo puede leer y ejecutar pero no escribir
- permisos de usuarios: `r-x` -> los usuarios pueden leer y ejecutar pero no escribir

También por abajo los permisos se representan como una tira de bits así que uno
podría decir que asignar 7 (`chmod 777`) para los permisos es asignar lectura,
escritura y ejecución (en el ejemplo asigno esos permisos a los 3: owner, group
y user).

### SETUID y SETGID

Son permisos de acceso que permiten a los usuarios del sistema ejecutar
binarios con un nivel de privilegio mayor (temporalmente y sólo para esa tarea
específica).

Me doy cuenta porque el archivo tiene activado el bit de `SETUID`. El permiso
que debería aparecer sería `-rws....`. Otro detalle es que el bit de execute
también tiene que estar seteado para que haga efecto el `SETUID`. Para `SETGID`
también se muestra la s pero sobre los permisos de grupo. Se pueden settear con
`chmod
<nro_para_setuid_segid_stickybit><nro_para_owner><nro_para_group><nro_para_user>`

### SUDO

Permite ejecutar comandos en nombre de otro. Está bueno porque puedo escalar
privilegios con auditabilidad de por medio, porque los comandos se loguean.

### Los procesos y permisos

Estoy un día ahí, tranqui y tiro un comando en la terminal. Eso va a spawnear
otro proceso para el ejecutable que quiero. Qué permisos tiene el proceso? En
general, heredan los permisos del usuario que lo corre. Pero como vimos,
también puede pasar que tenga el bit de `SETUID`. De esa forma uno puede correr
el comando para cambiar la contraseña sin tener permisos de administrador.

## Seguridad en el software

Podemos diferenciar 3 momentos en donde se hace presente el análisis de la seguridad en nuestro software:

- Arquitectura/Diseño: cuando pensamos la aplicación.
- Implementación: Cuando estamos escribiendo el código de la aplicación.
- Operación: Cuando la aplicación ya está en producción.

### Errores de implementación

Suelen ser errores más fáciles de entender y solucionar que los de diseño (Un
error de diseño cuando estás en prod puede ser carísimo). El grueso de los
errores además vienen de hacer suposiciones que no están garantizadas. Por
ejemplo, "*el usuario va a ingresar una cantidad acotada de caracteres*".
Dentro de la info "de entrada" que muchas veces no contemplamos están:

- Variables de ambiente.
- El input del programa.
- Software de terceros (?).

### Buffer Overflows

Hoy en día no son tan relevantes, pero es interesante aprenderlos para entender
que hay que tener cuidado al programar y para conocer algunas técnicas que los
OS proveen hoy en día para prevenir esto.

La idea es que a veces hay funciones con vulnerabilidades de seguridad. En
particular en el lenguaje C (podría ser algo de otro lenguaje pero nos vamos a
concentrar en C), existen funciones como por ejemplo `strcpy`. `strcpy` recibe
el puntero a donde hay que copiar la data y el origen de la data, no pide nunca
el tamaño de la misma. Lo que hace es leer del origen hasta encontrar el
caracter de fin de string y va copiándolo en el buffer destino. El problema de
todo esto es que las variables locales tienen su espacio reservado en la pila.
Si usamos una variable local como buffer para `strcpy` y el string tiene más
caracteres que el buffer, voy a estar escribiendo por fuera del espacio
reservado para la variable local y sobre la data que está en el stack... eso
sólo puede salir mal.

Veamos un ejemplo:

```c
void f(char, *origen) {
  char buffer[16];
  strcpy(buffer, origen);
}

void main(void) {
  char grande[18]; // también podría leer un input del usuario arbitrariamente grande.
  f(grande);
}
```

Antes del `strcpy`:

![](./img/buffer_overflow_1.png#center)

Después del `strcpy` (notar que se pisa el IP):

![](./img/buffer_overflow_2.png#center)

Si bien esta versión es stack based, existen otros ataques que son heap based y se basan en la misma idea. Existen para ambos algunas formas de detectarlo e intentar prevenirlo:

- memoria no ejecutable
- canary
- randomizar el espacio de memoria ([aslr](https://en.wikipedia.org/wiki/Address_space_layout_randomization))
- control de parámetros

### Control de parámetros

Esto aplica tanto para buffer overflows como otros tipos de ataque. El
principal problema es que asumimos como válido cualquier tipo de input y eso
nos puede llevar a ejecutar código que estaba contenido en el input. Por
ejemplo:

```bash
$direccion=argv[1]
echo Felix Cumple | mail -subject='Felicitaciones' $direccion
```

El programa espera una dirección de mail, pero ponele que usamos el parámetro
`asd@asd.uba.ar; rm -rf /`. Si el programa está corriendo en un servidor remoto
(y con privilegios) esto literal les borra todo.

Para este problema tenemos dos potenciales soluciones:

1. Usar siempre el mínimo privilegio posible para correr los programas
2. Validar los parámetros
3. Tainted data: se parece al anterior pero es un poco más flexible. Consiste
   en marcar los datos que provienen de fuentes externas como "manchados", y de
   la misma manera las cosas que se derivan de esa fuente, y las cosas que se
   derivan de esas cosas, etc. En el hipotético caso de que quiera ejecutar
   código con data manchada, recién ahí la tengo que desmanchar

~~~admonish note title="Esto pasa en bases de datos también" 
Este mismo concepto de no validar la data sucede frecuentemente con bases de
datos. El ataque es conocido como *SQL injection*. Por ejemplo, si no sanitizan
la entrada podría ocurrir que se ejecute lo siguiente (ej: en un buscador):

```c
db.execute("UPDATE alumnos SET aprobado=true WHERE lu='$1'", lu);
```

Y que el usuario ingrese `307/08'; DROP alumnos; SELECT '` tirándote toda la tabla de alumnos.
~~~

### Aislando a los usuarios

Los OS modernos suelen proveer distintas formas de aislar usuarios y procesos. Se las conocen como *sandboxes*:

- `chroot()`: te permite cambiar el directorio raíz. Por ejemplo si tengo:

```
- /
  - foo
    - file_1.txt
    - file_2.txt
  - bar
    - file_3.png
```

Si hago `chroot foo` y después `ls /` obtengo:

```
- file_1.txt
- file_2.txt
```

- `jail()`


## Principios generales

- Mínimo privilegio
- Simplicidad
- Validar todos los accesos a datos
- Separación de privilegios
- Minimizar nro. mecanismos compartidos
- Seguridad multicapa
- Facilitar el uso de las medidas de seguridad
