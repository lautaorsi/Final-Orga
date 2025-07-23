
# Memoria

## Cache

### Line replacement

1. Random: Selecciona al azar la linea a ser desalojada y reemplazada por aquella que no encontro en el cache
2. Least Recently Used: Desaloja la que fue usada por ultima vez hace mas tiempo
3. First In First Out: Desaloja la ultima que fue accedida 

### Write policies
1. Write Through: Se escribe en la memoria cache y los niveles inferiores. <br>
   - Ventajas: Implementacion sencilla. Asegura coherencia.
   - Desventaja: Escritura lenta.
2. Write Back: Se escribe solo en la memoria cache, cuando vaya a ser desalojada recien ahi se escribe en los niveles inferiores. <br>
    - Ventajas: Se escribe a maxima velocidad. Poco consumo de energia.
    - Desventaja: No asegura coherencia. 
 <br> **Importante: notar que para esto necesitaremos marcar si una linea esta _Dirty_ o _clean_** 

### Write miss 
1. Write allocate: Se lee la linea cache y luego se escribe.
2. No-Write allocate: Se escribe en el nivel inferior inmediato, se subirá recien cuando se lea.

### Protocolos de coherencia

#### MESI: 

- (M)odified: Linea presente solamente en este cache y que fue modificada respecto a la memoria del sistema (Write Back).

- (E)xclusive: Linea presente solamente en este cache y que no fue modificada.

- (S)hared: Linea presente y **puede** estar presente en cache de otros procesadores.

- (I)nvalid: Linea invalida. <br>

Necesitaremos una forma de invalidar la linea en todos los otros caches, logrado con el protocolo snoopy que hace que todos los caches reciban actualizaciones y sigan una serie de acciones en base a esto. <br>


Lineamientos fundamentales de mesi: 
- pedidos de lectura se resuelven en cache, salvo que sea linea invalida
- cada linea invalida en el cache se busca en la jerarquia inferior inmediata
- lineas shared o exclusive pueden pasar a ser invalidas en cualquier momento
- modified tambien, pero requiere write back primero
- modified sobreescrita por su cpu se queda modified
- exclusive sobreescrita pasa a modified
- shared sobreescrita requiere lineas invalidas en todos los otros caches 

#### MESIF:

Entre todas las lineas compartidas por mas de un cache (en estado S) tiene que haber una con estado F.
Basicamente, la linea que tiene el F actua como forwarder para enviar el dato al cache que lo solicita, como es un estado impreciso y no se notifica puede ocurrir que no quede ninguna linea F pero si varias shared, en ese caso cuando haya un read miss va a tener que buscarlo en niveles inferiores.

#### MOESI:

Cuando hay un read miss en una linea que un cache tiene modified, en vez de hacer write back y degradarse a shared va a tomar el control de la linea como Ownership, pasando a encargarse de mantener la coherenciam, marcando como dirty la LLC/memoria y enviando el dato a la cache que produjo el read miss.



# Paralelismo de instrucciones 

## Pipeline y sus obstaculos

Obviamente el pipeline no puede funcionar de una forma "lineal", esto puede traer serios problemas a la hora de encontrarnos con obstaculos categorizados en:
- Obstaculos estructurales (dos operaciones requieren acceder a memoria al mismo tiempo, no se puede y debemos retrasar lo necesario una de las dos) <br> **Posible solucion: ensanchar buses, partir cache L1 en datos e instrucciones Desventaja: costoso**
- Obstaculos de datos (se requiere usar un dato en un orden distinto al logico del programa, tipo x = 2 + 3 ; z = 2 + x pero primero llegamos a la 2da operacion, tenemos un stall)<br>
 **Posible solucion: forwarding del resultado. Desventaja: solo funciona back-to-back** 
- Obstaculos de control (saltos condicionales que nos hacen perder todas las instrucciones que habiamos hecho de forma preventiva) <br>
**Posible solucion: predicciones de saltos**

En cualquier caso, estos obstaculos causan lo que llamamos **_pipeline stall_**, cosa que queremos evitar.

## Predicciones de saltos

Tenemos distintos tipos de predictores de saltos:
#### Compilador-dependientes
- Predict taken, asumimos que se va a tomar el salto siempre. **Es particularmente util para loops**
- Predict non-taken, asumimos que **no** se va a tomar el salto siempre. 
- Predict delayed, calculamos el resultado de tomar la branch y lo almacenamos, pero solo la aplicamos si efectivamente se toma el salto.

#### Hardware-dependientes (prediccion dinamica)

- Branch prediction buffer, una tabla que indica si se tomo o no el salto la ultima vez y predecimos que va a ocurrir eso mismo. 
   - 1 bit: 1 taken | 0 non-taken
   - 2 bit: 11 taken-strong | 10 taken-weak | 01 non-taken-weak | 00 non-taken-strong

- Branch Target Buffer, una tabla con la direccion de instruccion de salto y la direccion de la primera instruccion de la branch tomada por ultima vez. Si esta vacia se asume taken, va a ir cambiando segun que fue el ultimo salto tomado.

## Superescalar 
Incrementa las chances de tener pipeline stalls (por los obstaculos). Falas en las predicciones de saltos son incluso mas graves porque tenemos que limpiar ambos pipelines

## Ejecucion fuera de orden

Podemos implementar la ejecución fuera de orden para intentar evitar los pipeline stalls, obviamente esto conlleva riesgos como:
- Write After Read (WAR), realizamos una operacion adelantada que sobreescribe un dato que se va a usar antes por otra operacion, leyendo un dato incorrecto.
- Write After Write (WAW), realizamos una escritura adelantada y luego es pisada por una instruccion anterior, anulando la ultima.
- Read After Write (RAW), realizamos una lectura adelantada pero teniamos una escritura antes, usando un dato viejo. 

Todo esto es implementado mediante hardware, dado que usar el compilador podria resultar en mas saltos que complican la situación.


## Tomasulo
El algoritmo de Tomasulo busca principalmente: 
- Minimizar RAW.
- Solucionar WAR y WAW mediante _Register Renaming_. 

Implementa dos nuevas tablas a continuacion de la _Unidad de Decodificacion_:
- Register Alias Table (RAT), que implementa el link productor-consumidor.
- Reservation Station (RS), complementa el link de la RAT y realiza los pasos necesarios para Tomasulo.
- Common Data Bus (CBD), que une las unidades mencionadas para enviar cada resultado y el tag que reemplaza

#### RAT
Del tamaño de la cantidad de registros que hay. Se divide en 3 partes, _Tag_, _Valor_, _Valido_. <br>
Cada vez que decodificamos una operacion tomamos de la RAT el _Valor_ si Valido = 1 o _Tag_ si Valido = 0. <br>
Al operando destino de cada instruccion decodificada se le pone **Valido = 0 y *Tag = Nro de registro de la RS*** en la que se almaceno la instruccion. <br>
Si aparece una nueva escritura y *Valido* ya esta en 0, se sobreescribe el *Tag* -> RAT siemrpe mantiene el ultimo *Tag* valido y elimina el riesgo de **WAW**