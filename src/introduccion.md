# Introducción

> Una forma de dividir a los sistemas informáticos es:
>
> - Hardware (lo que se puede patear).
> - Software específico (lo que sólo se puede putear).
> 
> Un **sistema operativo** es un intermediario entre ellos

## Historia

- 1950: primeras computadoras comerciales
  - el usuario armaba tarjetas perforadas en assembler o FORTRAN, un operador se encargaba de todo.
  - problema: mucho tiempo de procesamiento desperdiciado.
  - (más adelante) solución: sistemas batch. Hacías tarjeta -> cinta, cinta -> impresora.
    - seguía habiendo un operador haciendo de intermediario entre usuario y hardware.
- más adelante, surge el concepto de **multiprogramación**, tratamos de usar el tiempo ocioso de un trabajo `j_1` ejecutando otro `j_2`. `j_1` capaz tarda un poco más pero en total tardo menos.
  - también surge la idea de **contención** (dos programas queriendo acceder a un recurso a la vez)
- (después) surgen sistemas de **timesharing**, o sea conectar varias terminales a una misma computadora y darles un poco de tiempo de procesador a cada una.
  
## Un S.O.

> - Un SO es una pieza de software que hace de intermediario entre el HW y los programas de usuario.
>   - permite que el software específico no se preocupe con detalles de bajo nivel de HW
>   - permite que el usuario use correctamente el HW.
> - Tiene que manejar cuestiones como la contención y la concurrencia de tal manera que:
>   - sea eficiente
>   - se haga correctamente

