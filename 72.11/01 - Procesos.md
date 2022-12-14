# Procesos

Segun UNIX, un proceso es un programa en ejecucion. Cada proceso tiene almacenado el codigo que va a ejecutar, su area de datos, su program counter, sus registros e informacion adicional. Los sistemas operativos son quienes manejan a estos procesos, pudiendo crearlos, destruirlos, suspenderlos y retomarlos, ademas de mecanismos de concurrencia.

El **contexto** de un proceso es el conjunto de informacion dinamica que representa el estado de ejecucion de un proceso.

Los procesos se pueden crear directamente desde codigo, en el caso de UNIX esto se hace con la instruccion `fork()` que crea un proceso hijo identico que luego va a continuar ejecutando el programa junto al proceso padre. La funcion `fork()` devuelve `0` al proceso hijo y devuevlve el `pid` del hijo al proceso padre, de ocurrir un error devuelve `-1`.



### Funciones Relacionadas

Existen funciones especiales para manejar la interaccion de un proceso hijo y su padre. La funcion `wait()`, ejecutada en el proceso padre, pausa la ejecucion del proceso hasta que el proceso hijo no se cierre. De forma similar, la funcion `exit()` termina la ejecucion del proceso. De ser terminado el proceso padre antes que el hijo, entonces este se convierte en un **proceso huerfano** y el proceso indetificador del padre coincide con el de la terminal (esto causa que el proceso hijo corra en el background). Si, por el contrario, se termina de ejecutar el proceso hijo antes de que el proceso padre llegue a la funcion `wait`, entonces creamos un **proceso zombie**. Exite una variacion de la funcion `wait` que se llama `waitpid` que suspende la ejecucion hasta que un proceso en especial termine.

Ambas funciones (`wait` y `waitpid`) toman un puntero a un `int`, que al ejecutarse la funcion devuelven el valor de estado del proceso hijo. Se puede averiguar mas informacion del proceso hijo si utilizamos una de las siguentes macros con el estado del hijo:

- `WIFEXECUTED(status)`: Es distinto de cero si el hijo termino normalmente.
- `WEXITSTATUS(status)`: Evalua los 8 bits menos significativos del codigo de retorno del hijo. Esta macro solo se puede llamar si la macro anterior no devolvio 0
- `WIFSIGNALED(status)`: Devuelve verdadero si el proceso hijo termino a causa de una se??al no capturada.
- `WTERMSIG(status)`: Devuelve el numero de la se??al que provoco la muestre del proceso hijo. Esta macro solo se puede usar si la macro anterior devuelve un valor distinto de `0`
- `WIFSTOPPED(status)`: Devuelve verdadero si el proceso hijo que provoco el retorno esta parado. Esta macro solo se puede utilizar si la llamada se hizo utilizando `WUNTRACED`.
- `WSTOPSIG(status)`: devuelve el numero de la se??al que provoco la parada del hijo. Solo se puede usar si la macro anterior devolvio un valor distinto de `0`.

#### `exec()`

Las funciones `exec` son aquellas que nos permiten ejecutar un programa desde nuestro codigo. Este tipo de funciones no deberian regresar, de hacerlo sera `-1` y sera por un error. 

Hay dos conjuntos distintos de estas funciones `execl` toma una lista de parametros concida, mientras que `execv` toma un arreglo con puntero a char, es decir que no sabemos la cantidad de argumentos. 

#### `atexit()`

Esta funcion sirve para instalar un manejador de terminacion de usuario. Se puede registar mas de una funcion y las funciones registradas se llaman en orden inverso a su registro a la hora de terminar el proceso (cuando se ejecuta `exit()`). De cargar bien la funcion devuelve `0`, sino devuelve `-1`.

## Pipes

Los pipes son el mecanismo mas antiguo de IPC y lo proveen los sistemas Linux. Este sistema provee un canal unidireccional para flujos de datos entre procesos y solo puede ser utilizados entre procesos on un antecesor comun que crea el pipe.

Los pipes se crean con la funcion:

```c
int pipe(int fildes[2]);
```

Donde el parametro es una tabla con dos descriptores de ficheros, siendo el primero de lectura y el segundo de escritura. El valor de retorno de esta funcion es 0 si tiene exito y -1 si hay un error.

Ejemplo:

```c
int main(void){
    int fd[2];
    pipe(fd);
    
    if(fort()!= 0){
        close(fd[0]); // El padre no lee
        write(fd[1], "Hola\n");  // El padre escribe
    }else{
        close(fd[1]); // El hijo no escribe
        char str[100];
        int n = read(fd[0], str,100);
        write(STDOUT_FILENO,str,n);
    }
}
```

Hay mas funciones que participan en los `pipes`. Por ejemplo:

```c
int dup(int oldfd);
```

Es una funcion que permite duplicar un file descriptor ya abierto y te devuelve un nuevo file descriptor. En caso de error devuelve `-1`.

```c
int dup2(int oldfd, int newfd);
```

Esta funcion es similar a la anterior pero le podes indicar que file descriptor queres que te asigne. Si el file descriptor deseado esta ocupado, lo cierra y luego lo duplica.

## Gestion de Procesos

Un proceso puede terminar por distintas razones:

- Su funcion main llama a `return`:

  Fuerza la salida inmediata de la funcion y provoca que salte al punto en el que se hizo la llamada a la funcion. Si hay un valor asociado con en `return`, se trata del valor de devolucion de la funcion.Si el `return` es de la funcion main se cierra el programa.

- Se llama a `exit()` de la libreria `<stdlib.h>`:

  Produce la terminacion normal del programa y devuelve el estado al proceso padre. Antes de finalizar se ejecutan todas las funciones registradas con `atexit()`.

- Se llama a system call `_exit` de  `<unistd.h>`:

  Termina inmediatamente la ejecucion del proceso invocador y no se ejecuta ninguna funcion cargada con `atexit()`.

- Se llama a `abort:`

  Causa una terminacion anormal y graba una imagen en memoria. Cierra todos los canales abiertos y los borra. Genera la se??al `SIGABRT`.

- Es finalizado por una se??al

### Se??ales

Son un suceso que tiene lugar en cualquier momento durante la ejecuci??n de un proceso. Cada se??al tiene un nombre y las funciones que las manejan se encuentran en la librer??a `<include/signal.h>`. Hay distintas acciones que se pueden realizar al recibir una se??al:

- Ignorar la se??al
- Permitir que ocurra una acci??n predeterminada.
- Capturar a la se??al, ejecutar a la rutina de tratamiento y volver al estado anterior.

Existen las siguientes se??ales definidas en el sistema:

|   Se??al    |  Valor   | Acci??n |                           Detalle                            |
| :--------: | :------: | :----: | :----------------------------------------------------------: |
|  `SIGHUP`  |    1     |  `A`   | Cuelgue detectado en la terminal o muerte del proceso de control. |
|  `SIGINT`  |    2     |  `A`   |             Interrupci??n del teclado (`ctrl+c`)              |
| `SIGQUIT`  |    3     |  `C`   |        Terminaci??n procedente del teclado (`ctrl+4`)         |
|  `SIGILL`  |    4     |  `C`   |                      Instrucci??n Ilegal                      |
| `SIGABRT`  |    6     |  `C`   |                       Se??al de aborto                        |
|  `SIGFPE`  |    8     |  `C`   |                  Excepci??n de coma flotante                  |
| `SIGKILL`  |    9     | `AEF`  |                        Se??al de matar                        |
| `SIGSEGV`  |    11    |  `C`   |                Referencia invalida a memoria                 |
| `SIGPIPE`  |    13    |  `A`   |             Tuber??a rota: Escritura sin lectores             |
| `SIGALARM` |    14    |  `A`   |                       Se??al de alarma                        |
| `SIGTERM`  |    15    |  `A`   |                     Se??al de terminaci??n                     |
| `SIGUSR1`  | 30,10,16 |  `A`   |               Se;al definida por el usuario 1                |
| `SIGUSR2`  | 31,12,17 |  `A`   |               Se;al definida por el usuario 2                |
| `SIGCHLD`  | 20,17,18 |  `B`   |             Proceso del hijo terminado o parado              |
| `SIGCONT`  | 19,18,25 |        |                  Continuar si estaba parado                  |
| `SIGSTOP`  | 17,19,23 | `DEF`  |                       Parar el proceso                       |
| `SIGTSTP`  | 18,20,14 |  `D`   |                   Parada escrita de la tty                   |
| `SIGTTIN`  | 21,21,26 |  `D`   |            E. de la tty para un proceso de fondo             |
| `SIGTTOU`  | 22,22,27 |  `D`   |            S. de la tty para un proceso de fondo             |

De los cuales la acci??n puede ser:

- `A`: La acci??n por defecto es terminar el proceso
- `B`: La acci??n por defecto es ignorar la se??al 
- `C`: La acci??n por defecto es terminar el proceso y hacer core dump
- `D`: La acci??n por defecto es parar la ejecuci??n del proceso
- `E`: La se??al no puede ser capturada por el programa
- `F`: La se??al no puede ser ignorada.

Cuando queremos que un proceso espere a que llegue una se??al , utilizamos a la funci??n:

```c
int pause();
```

Esta funci??n hace que el proceso que recibe la se??al duerma hasta que llegue, y luego devuelve `-1`.

Tambi??n podemos encargarnos de las se??ales de una mejor forma, mediante callbacks:

```c
sighandler_t signal(int signum, sighandler_t handler);
```

Estos par??metros son el numero de la se??al que queremos enviar y un puntero a la funci??n que se encargara de tratar a la se??al indicada. De fallar, la funci??n devuelve `SIG_ERR`.

### Otras funciones relacionadas

Otras funciones relacionadas con la gesti??n de procesos son:

```c
unsigned int alarm(unsigned int seconds);
```

Genera un timer en el procesador que env??a al proceso que la ejecuto la se??al `SIGALRM` cada una dada cantidad de segundos.

Por otro lado tenemos a la funci??n sleep:

```c
int sleep(unsigned int seconds);
```

Esta funci??n suspende el proceso por una dada cantidad de segundos.



## Shared Memory

Es un m??todo de comunicaci??n entre procesos por el cual mas de un proceso tienen acceso a una secci??n de memoria al mismo tiempo y todos tienen permisos de leer y escribir. Queda a disposici??n del programador para establecer como se va a realizar la comunicaci??n.

**Zona critica**: fragmento de c??digo que accede a memoria compartida.

Hay dos implementaciones de este sistema, una en POSIX y otra que se llama SYSTEM V. La implementaci??n de POSIX esta mas al d??a y es la que vamos a usar. Para utilizarla hay que usar las siguientes funciones:

#### `shm_open(const char * name, int oflag, mode_t mode)`

Crea o abre el archivo de memoria compartida en la ubicaci??n `/dev/shm/<name>` con el nombre indicado. El archivo es creado pero esta vac??o, es decir que tiene tama??o 0.

Tiene los siguientes par??metros:

- `oflag`: Es una mascara que indica las propiedades del archivo creado. Se le pueden asignar los siguientes valores:
  - `O_RDONLY`: Solo se puede leer el archivo.
  - `O_RDWR`: El archivo tiene permiso de escritura y lectura.
  - `O_CREAT`: Crea el archivo si no existe. Si el archivo que se quiere crear ya existe va a tirar error.
  - `O_EXCL`: Se usa en conjunto con `O_CREAT`. Esta flag valida autom??ticamente de que el nombre del archivo no exista antes de crear la memoria compartida.
  - `O_TRUNC`: Si al abrir el archivo ya existe, lo vac??a y cambia el tama??o a 0.
- `mode`: ??

La funci??n devuelve un file descriptor si tuvo existo y `-1` en caso de que haya un error.

#### `int ftruncate(int fd, off_t lenght)`

Esta funci??n le asigna el tama??o a un archivo que se le pasa mediante su file descriptor. De ser mas achicarse el archivo se va a perder informaci??n y al agrandarse se va a expandir al archivo.

El file descriptor tiene que haber sido abierto con permisos de escritura para usar a esta funci??n.

Devuelve `0` si todo sale bien, `-1` si hay error.

#### `void * mmap(void * addr, size_t lenght, int prot, int flags, int fd, off_t offset)`

 Esta funci??n mapea un file descriptor a una posici??n virtual de memoria en nuestro proceso.

Los argumentos que acepta son:

- `addr`: Es la direcci??n virtual a la cual se va a mapear el archivo. De ser `null` el sistema operativo elije donde ubicarlo.
- `lenght`: Indica el tama??o del mapeo.
- `prot`: Indica las protecciones que se le van a asignar a las paginas de memoria virtual, estas no deben contradecir a la del archivo. Las protecciones son:
  - `PROT_EXEC`: Pueden ser ejecutadas.
  - `PROT_READ`: Pueden ser le??das.
  - `PROT_WRITE`: Pueden ser escritas.
  - `PROT_NONE`: No pueden ser accedidas
- `flags`: Indican quienes pueden ver las actualizaciones del archivo. Puede tener los siguientes valores:
  - `MAP_SHARED`: Las actualizaciones del archivo pueden ser vistas por todos los procesos que est??n en la misma region.
  - `MAP_SHARED_VALIDATE`: Hace lo mismo que el flag anterior pero adem??s valida si los flags enviados son validos.
  - `MAP_PRIVATE`: Al crear el mapeo en memoria virtual copia la informaci??n en vez de compartirla, por lo que no puede ver ni enviar cambios al archivo, todos los cambios son locales.

Si no se encuentra ning??n error devuelve un puntero al ??rea mapeada, sino devuelve `(void *)-1`.

#### `int munmap(void * addr)`

Esta funci??n remueve el mapeo a memoria hecho por `mmap()`. El valor de retorno es `-1` si hay alg??n error y `0` si no lo hay.

#### `int shm_unlink(const char * name)`

Esta funci??n remueve y destruye al archivo de memoria compartida. Devuelve `0` si no hay error y `-1` si lo hay.

#### `close()`

cierra el file descriptor

## Message Passing

Otra forma de comunicar procesos. En este caso se utilizan mensajes de forma asincr??nica de un proceso a otro y permite que la comunicaci??n sea de uno a muchos.

