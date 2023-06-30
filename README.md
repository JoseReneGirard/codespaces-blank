# Gestor de Tablas con Índices Lineales


## Objetivos
Los objetivos de este proyecto son:
* Practicar con programas que reciben su entrada desde los parámetros de línea de comando
* Trabajar con archivos de registros binarios:
  * De registros de longitud fija
  * Con índices primarios y secundarios lineales
  * Con "avail-list"
  * Con descripción en cabecera
* Explorar las operaciones REST

## Descripción

La idea del programa que desarrollarás es que el programa recibirá parámetros de línea de comando para ejecutar operaciones sobre un archivo binario de registros de longitud fija. Las operaciones que el programa realizará están descritas en la siguiente tabla.

| Operación          | Descripción                                                                     | Parámetros de Línea de Comando                         |
|--------------------|---------------------------------------------------------------------------------|--------------------------------------------------------|
| Crear Archivo      | Crear el archivo de datos con la estructura del archivo JSON provisto           | `-create structure.json`                               |
| Cargar Datos       | Carga los datos de un archivo CSV                                               | `-load data.csv`                                       |
| Compactar          | Compacta el archivo                                                             | `-compact`                                             |
| Reindexar          | Regenera los índices del archivo                                                | `-reindex`                                             |
| Describir          | Describe la estructura del archivo de registros e indica el número de registros | `-describe`                                            |
| Listar             | Lista todos los registros del archivo                                           | `-GET`                                                 |
| Buscar PK          | Muestra los registros filtrados por la condición provista (llave primaria)      | `-GET -pk -value=23`                                   |
| Buscar SK          | Muestra los registros filtrados por la condición provista (llave secundaria)    | `GET -sk=city -value=quito`                            |
| Agregar Registro   | Agrega un registro al archivo                                                   | `-POST -data={"field":"value", "other":"value"}`       |
| Modificar Registro | Modifica un registro del archivo                                                | `-PUT -pk=33 -data={"field":"value", "other":"value"}` |
| Borrar Registro    | Borra un registro del archivo                                                   | `-DELETE -pk=43`                                       |

Si quieres saber más acerca de algunas operaciones familiares puedes visitar este [sitio](https://www.restapitutorial.com/lessons/httpmethods.html).

En todos los casos el programa responderá con JSON la operación que se le ha solicitado.

### Crear Archivo

Esta operación tomará como parámetro la estructura de un archivo, y creará el mismo con el encabezado necesario para poder trabajar con él. La estructura del archivo será especificada por un archivo JSON como el que se muestra a continuación:

```json
{
  "fields": [
    {"name": "id", "type": "int", "length": 4},
    {"name": "name", "type": "char", "length": 20},
    {"name": "age", "type": "int", "length": 4},
    {"name": "sex", "type": "char", "length": 1},
    {"name": "city", "type": "char", "length": 20}
  ], 
  "primary-key": "id",
  "secondary-key": ["sex", "city"]
}
```
En este archivo se puede ver que tiene cinco campos. Estos campos son:
1. `id`, que tiene tipo `int` y una longitud de 4 bytes
2. `name`, que tiene tipo `char` y una longitud de 20 bytes
3. `age`, que es de tipo `int` y una longitud de 4 bytes
4. `sex`, que es de tipo `char` y una longitud de 1 byte
5. `city`, que es de tipo `char` y una longitud de 4 bytes

Consideraciones acerca de la descripción de los campos:
* Los nombres de los campos no deberán llevar espacios.
* Los tipos posibles son
  * `char`, la cantidad de caracteres será indicada por la longitud del campo
  * `int`, el número de bytes siempre es 4, ya que el archivo es binario
  * `float`, el número de bytes siempre es 4, ya que el archivo es binario

Después de la lista de campos se enumeran los índices del archivo:
* Un único índice primario, en este caso será el campo `id`. 
* Cero o más índices secundarios, en este caso un índice secundario para el campo `sex` y otro para el campo `city`.

> _Deberás validar que el campo que se especifique en los índices esté en la lista de campos_.

Una vez hayas validado que la estructura es válida puedes proceder a crear la estructura de tu encabezado (queda libre a tu discreción) y guardar el archivo. Recuerda que para todas las operaciones posteriores necesitarás la información del encabezado.

Ejemplos del uso de esta operación
* Suponiendo que no hay problemas y que se está usando el archivo JSON mostrado anteriormente en el ejemplo.
```
./build/data-manager -create people.json
{"result": "OK", "fields-count": 5, "file": "people.bin", "index": "people.idx", "secondary": ["people-sex.sdx", "people-city.sdx"]}
```
* Suponiendo hay errores
```
./build/data-manager -create people1.json
{"result": "ERROR", "error": "primary index field does not exist"}
./build/data-manager -create people2.json
{"result": "ERROR", "error": "secondary index field does not exist"}
./build/data-manager -create people3.json
{"result": "ERROR", "error": "invalid type"}
./build/data-manager -create no-existe.json
{"result": "ERROR", "error": "json file not found"}
./build/data-manager -create people1.json
{"result": "ERROR", "error": "Format not recognized"}

```

### Cargar Datos
Esta operación ayuda a cargar los datos de un archivo CSV al archivo binario que fue creado previamente. En el caso de la carga de archivos el número de campos y el nombre de los campos debe de coincidir con el encabezado del archivo. Si no coinciden entonces se reporta el error y no se cargan los datos. Adicionalmente, hay que validar si cuando se espera un campo de tipo `int` o `float` y no es eso lo que se encuentra en el archivo CSV ese registro será saltado y se procederá al siguiente. Habrá que reportar luego el número de registros que se cargaron y los registros que se saltaron.

Suponiendo la estructura mostrada anteriormente y dado el siguiente archivo de datos `amigos.csv`:
```
id,name,age,sex,city
334,juan,12,m,quito
321,ana,23,f,paris
98,pedro,47,m,tegucigalpa
71,cristina,50,f,quito
111,julio,48,m,paris
569,anne,23,f,paris
520,philip,38,m,seattle
52,richard,33,m,taipei
```

Ejecutar el programa para cargar esos datos sería:
```
./build/data-manager -file people.bin -load amigos.csv
{"result": "OK", "records": "8"}
```

Ahora bien el archivo CSV podría tener alguno de los siguientes problemas:
* Los nombres de los campos no coinciden con la estructura del archivo
* El número de campos no coincide con la estructura del archivo
* Algún campo no corresponde al tipo especificado en la estructura del archivo
* Faltan campos en algún registro
* Hay campos vacíos en algún registro

Lo que debes hacer en cada uno de estos casos:
* Si el nombre de los campos o el número de campos no coincide, no cargas ningún dato y tu programa responderá:
```
{"result": "ERROR", "error": "CSV fields do not match file structure"}
```
* Si algún registro no logra ser leído porque algún campo no tenía el tipo esperado, o porque le hacía falta algún campo, te saltarás ese registro (skip) y mantendrás cuenta de cuántos registros te saltaste para reportarlo luego. Por ejemplo, si al leer el archivo tu programa tuvo que saltar 4 registros, tu programa responderá:
```
{"result": "WARNING", "records": 10, "skipped": 4}
```
* Hay campos vacíos en algún registro. Si el tipo del campo es `char` coloca una cadena vacía, si es numérico coloca un cero. No reportas error por esto. Un registro con datos faltantes se vería as: `520,philip,,m,seattle`. Nota que hay dos comas seguidas, esto significa que el campo 3 está vacío. Campos que *siempre* deben de estar: la llave primaria y las llaves secundarias. No deberás agregar registros que no tengan estos campos.

### Compactación y Re-indexación
Estas operaciones solo necesitan saber el nombre del archivo. En el caso de compactación el programa barrerá el archivo buscando los "espacios" de registros borrados y recuperará el espacio. Esta operación deberá reindexar el archivo también. En el caso de re-indexación el programa leerá la estructura del archivo y volverá a crear el índice primario e índices secundarios. Efectivamente, esto borrará el archivo `idx` y los archivos `sdx` y generará nuevos. En nuestro proyecto los archivos SIEMPRE tendrán un índice primario y pueden tener cero o más índices secundarios.

Ejemplos:
```
./build/data-manager -file people.bin -compact
{"result":"OK", "records-reclaimed": 3}
./build/data-manager -file people.bin -reindex
{"result":"OK", "indices-processed": 3}

```


### Describir
Describe la estructura del archivo. Esta operación desplegará la estructura del archivo y el número de registros en el archivo. Esta descripción es *muy* similar a la estructura con la que el archivo fue creado. Suponiendo la estructura de amigos con los datos CSV mostrados anteriormente:
```
./build/data-manager -file people.bin -describe
{
  "fields": [
    {"name": "id", "type": "int", "length": 4},
    {"name": "name", "type": "char", "length": 20},
    {"name": "age", "type": "int", "length": 4},
    {"name": "sex", "type": "char", "length": 1},
    {"name": "city", "type": "char", "length": 20}
  ], 
  "primary-key": "id",
  "secondary-key": ["sex", "city"],
  "records": 8
}
```

### Listar y Buscar

Estas operaciones ayudarán al usuario a desplegar datos del archivo, a continuación se muestran ejemplos de cómo usar estas opciones.

Listar todos los registros, muestra los registros en formato JSON.
```
./build/data-manager -file people.bin -GET
[
 {
   "id": 334,
   "name": "juan",
   "age": 12,
   "sex": "m",
   "city": "quito"
 },
 {
   "id": 321,
   "name": "ana",
   "age": 23,
   "sex": "f",
   "city": "paris"
 },
 {
   "id": 98,
   "name": "pedro",
   "age": 47,
   "sex": "m",
   "city": "tegucigalpa"
 },
 {
   "id": 71,
   "name": "cristina",
   "age": 50,
   "sex": "f",
   "city": "quito"
 },
 {
   "id": 111,
   "name": "julio",
   "age": 48,
   "sex": "m",
   "city": "paris"
 },
 {
   "id": 569,
   "name": "anne",
   "age": 23,
   "sex": "f",
   "city": "paris"
 },
 {
   "id": 520,
   "name": "philip",
   "age": 38,
   "sex": "m",
   "city": "seattle"
 },
 {
   "id": 52,
   "name": "richard",
   "age": 33,
   "sex": "m",
   "city": "taipei"
 }
]
```

Buscando un registro por medio de llave primaria (Primary Key -pk). En el primer caso, se encuentra el registro con llave primaria 569, en el segundo caso no encuentra el registro.

```
./build/data-manager -file people.bin -GET -pk -value=569
 {
   "id": 569,
   "name": "anne",
   "age": 23,
   "sex": "f",
   "city": "paris"
 }
 
./build/data-manager -file people.bin -GET -pk -value=1
 {
   "result": "not found"
 }
 
```

Usando índices secundarios. 
```
./build/data-manager -file people.bin -GET -sk=city -value=paris
[
 {
   "id": 321,
   "name": "ana",
   "age": 23,
   "sex": "f",
   "city": "paris"
 },
 {
   "id": 111,
   "name": "julio",
   "age": 48,
   "sex": "m",
   "city": "paris"
 },
 {
   "id": 569,
   "name": "anne",
   "age": 23,
   "sex": "f",
   "city": "paris"
 }
 ]
```

> Nota: La búsqueda por índices primarios retorna 0 o 1 registro. En cambio la búsqueda por índice secundario retorna 0 o más registros. En el caso de que la búsqueda por índice secundario no resulte en ningún registro, solo imprima "[]" para indicar búsqueda vacía.

### Agregar
Para agregar registros nuevos se puede usar la bandera `-POST`. A continuación se muestran ejemplos.

Agregar un registro, el primero tiene todos los campos, el segundo colocará 0 en el campo `age`:
```
./build/data-manager -file people.bin -POST -data={"id":835, "name":"jen", "age":24, "sex":"f", "city":"miami"}
{"result":"OK"}

```
No deberás agregar campos con llaves primarias duplicadas, ni registros que no correspondan a la estructura del archivo.

¡Recuerda! Tomar en consideración todas las estructuras de datos que están involucradas: avail-list, índices, archivo.

### Modificar
Esta operación toma la llave primaria provista por el usuario, busca el registro con esa llave y en el caso de encontrarlo lo reescribe con la información provista. Antes de hacer la modificación deberá asegurarse que la estructura corresponda a la del archivo. Ejemplo:
```
./build/data-manager -file people.bin -PUT -pk=111 -data={"id": 111, "name": "julio",  "age": 48, "sex": "m", "city": "berlin"}
{"result":"OK"}
```
Mira que los datos a modificar incluyen todos los campos del registro, incluyendo la llave primaria. Si la llave primaria no se encuentra, o si la estructura de los campos no coincide con la estructura del archivo entonces reportas el error.

### Borrar
Borra el registro especificado por el valor de la llave primaria. Si la llave no se encuentra reporta el error. Recuerda que esta operación *no borra* el registro del archivo, en vez marca ese espacio como "libre" y agrega una entrada al avail-list.
Ejemplo:
```
./build/data-manager -file people.bin -DELETE -pk=111
{"result":"OK"}
./build/data-manager -file people.bin -DELETE -pk=111
{"result":"not found"}
```

## Evaluación

En este repositorio se incluyen algunos archivos que puedes utilizar para probar tu programa. Para la evaluación se podrían usar estos u otros archivos.

Archivos incluidos:

* Productos
  * `products.csv` contiene una lista de productos con categorías
  * `products.json` contiene la correcta descripción del archivo `products.csv`
  * `products.e1.json` contiene una descripción que no coincide con `products.csv`
* Amigos
  * `friends.csv` contiene una lista de amigos
  * `friends.json` contiene la correcta descripción de `friends.csv`
  * `friends.e1.json` contiene campos que no existen en `friends.csv`
  * `friends.e1.csv` contiene datos que no corresponden a la descripción de `friends.json`
  * `friends.e2.csv` contiene datos con llaves primarias repetidas
  * `friends.e3.csv` contiene datos con campos vacíos

Adicionalmente, se proveen: `customers.csv` y `employees.csv`, pero para estos tú tendrás que definir el archivo JSON. También puedes aprovechar para utilizar los datos de los archivos de _products_ o _customers_ del programa de creación de índices.

La evaluación del programa será de la siguiente manera:

### Crea, carga y lista. [20 puntos] 
El programa genera el archivo con la estructura proporcionada, y carga los datos que se le indican. Se probará con los siguientes comandos:
```
./build/data-manager -create data/friends.json
{"result": "OK", "fields-count": 5, "file": "friends.bin", "index": "friends.idx", "secondary": ["friends-sex.sdx", "friends-city.sdx"]}

./build/data-manager -file data/friends.bin -describe
{
  "fields": [
    {"name": "id", "type": "int", "length": 4},
    {"name": "name", "type": "char", "length": 20},
    {"name": "age", "type": "int", "length": 4},
    {"name": "sex", "type": "char", "length": 1},
    {"name": "city", "type": "char", "length": 20}
  ], 
  "primary-key": "id",
  "secondary-key": ["sex", "city"],
  "records": 0
}
./build/data-manager -file data/friends.bin -load data/friends.csv
{"result": "OK", "records": "8"}

./build/data-manager -file data/friends.bin -describe
{
  "fields": [
    {"name": "id", "type": "int", "length": 4},
    {"name": "name", "type": "char", "length": 20},
    {"name": "age", "type": "int", "length": 4},
    {"name": "sex", "type": "char", "length": 1},
    {"name": "city", "type": "char", "length": 20}
  ], 
  "primary-key": "id",
  "secondary-key": ["sex", "city"],
  "records": 8
}
./build/data-manager -file data/friends.bin -GET
[
 {
   "id": 334,
   "name": "juan",
   "age": 12,
   "sex": "m",
   "city": "quito"
 },
 {
   "id": 321,
   "name": "ana",
   "age": 23,
   "sex": "f",
   "city": "paris"
 },
 {
   "id": 98,
   "name": "pedro",
   "age": 47,
   "sex": "m",
   "city": "tegucigalpa"
 },
 {
   "id": 71,
   "name": "cristina",
   "age": 50,
   "sex": "f",
   "city": "quito"
 },
 {
   "id": 111,
   "name": "julio",
   "age": 48,
   "sex": "m",
   "city": "paris"
 },
 {
   "id": 569,
   "name": "anne",
   "age": 23,
   "sex": "f",
   "city": "paris"
 },
 {
   "id": 520,
   "name": "philip",
   "age": 38,
   "sex": "m",
   "city": "seattle"
 },
 {
   "id": 52,
   "name": "richard",
   "age": 33,
   "sex": "m",
   "city": "taipei"
 }
]
```
### Captura errores de creación y carga 10
### Busca por llave primaria 20
### Busca por llave secundaria 20
### Borra, lista y busca 10
### Compacta 10
### Modifica, lista y busca 10



# Si no hay problema tomaré el anterior con mejor diseño

```json
{
  "fields": {
    "number": 5,
    "names": ["id", "name", "age", "sex", "city"],
    "types": ["int", "char", "int", "char", "char"],
    "lengths": [4, 20, 4, 1, 20]
  },
  "primary-key": "id",
  "secondary-key": ["sex", "city"]
}
```
