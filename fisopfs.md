# fisop-fs

# Correr el programa y Tests

Para correr el filesystem, hay que crear la carpeta `./mount` y correr `make run`. Tambien se puede crear con otro nombre y correrla con `./fisopfs -f ./nombre`

Creamos test unitarias para las funcionalidades en python. Nos pareció la mejor opción ya que es un lenguaje de scripting y fue muy comodo realizar las pruebas en el.

Para las test hay que crear la carpeta `./mount`. Correr make run en un terminal y correr `python test.py` en otra.

# Documentación de diseño

Filesystem basado en VSFS. Es un sistema de archivos basado en inodos, los cuales representan a una única entidad de archivo.

Guardamos los inodos en un array de memoria estática, llamado `INODES`. Cada inodo guarda metadatos acerca del archivo, como fechas, modos, tamaño del archivo, etc. Pero lo más importante es que guarda punteros (indices) a bloques, donde se guardan los datos:
```c
typedef struct inode {
	mode_t mode;
	int size_blocks;
	time_t fecha_de_acceso;
	time_t fecha_de_cambio;
	time_t fecha_de_modificacion;
	size_t n_links;
	uid_t uid;
	gid_t gid;
	size_t len;
	int blocks[BLOCKS_PER_INODE];
} inode_t;
```

Cada inodo presenta un array de indices a bloque. Todos los bloques los guardamos en otro array estático, cuyo tipo son chars (bytes).
```c
static char BLOCKS[MAX_BLOCKS][BLOCK_SIZE];
```

Un inodo puede no tener bloques continuos, por ejemplo, puede tener su archivo guardado en los bloques 5, 20 y 2.

Para saber que bloques e inodos en sus respectivos array están libres, utilizamos una estructura bitmap. El bitmap se guarda en el heap, y tenemos una referencia en memoria estática.
* Un bitmap es una estructura bastante sencilla en la que se mapea 0 al bit $n$ si el objeto $n$ está libre.

## Directorios

Para poder aprovechar el hecho de que el inodo represente tanto a archivos como a directorios, representamos la estructura del directorio adentro de bloques de chars. Es decir que a priori se lo puede ver como una cadena de chars, pero cuando queremos agregar/remover una entrada al directorio lo casteamos a un struct que creamos:

```c
typedef struct dirent {
	char name[50];
	int inode_number;
} dirent_t;
```

Este struct mappea un nombre de archivo/directorio dentro de ese padre directorio, a su respectivo inodo, que puede ser de directorio como archivo.

Como lo hicimos de esta manera, la anidación de múltiples directorios es 'natural' y se soporta.

Por lo que para **acceder a un archivo desde su path**, empezamos desde el nodo raiz (lo creamos por default en la función init y tiene indice de inodo 0) y vamos recorriendo cada entrada buscando el directorio hasta llegar al inodo final.

Para borrar una entrada de un directorio, como tenemos un array de entradas (o dirents) y queremos que sea continuo, simplemente borramos la entrada reemplazando con la ultima entrada, ya que el orden no importa. Ejemplo:

Si tenemos las siguientes entradas:
```
('..', 4)
('.', 5)
('hola.txt', 51)
('main.py', 2)
('chau.txt', 34)
```

y queremos borrar `'hola.txt'`, reemplazamos la primera entrada con la última, y quedaría asi:
```
('..', 4)
('.', 5)
('chau.txt', 34)
('main.py', 2)
```

# Persistencia en disco

La persistencia en disco fue simplemente guardar el array de NODOS y BLOQUES en un archivo cuando se destruye el filesystem, junto con sus bitmaps.

Luego en la función init abrimos el archivo y lo leemos en su respectivo lugar.
