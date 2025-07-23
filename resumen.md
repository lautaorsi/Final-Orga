
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

#### RAT (Register Alias Table)
Del tamaño de la cantidad de registros que hay. Se divide en 3 partes, _Tag_, _Valor_, _Valido_. <br>
Cada vez que decodificamos una operacion tomamos de la RAT el _Valor_ si Valido = 1 o _Tag_ si Valido = 0. <br>
Al operando destino de cada instruccion decodificada se le pone **Valido = 0 y *Tag = Nro de registro de la RS*** en la que se almaceno la instruccion. <br>
Si aparece una nueva escritura y *Valido* ya esta en 0, se sobreescribe el *Tag* -> RAT siemrpe mantiene el ultimo *Tag* valido y elimina el riesgo de **WAW   **

#### RS (Reservation Station)
Se encarga de completar el Register Renaming y de implementar las otras funciones de tomasulo:
- Mantener las instrucciones en espera hasta que esten listas para ser ejecutadas.
- Indicar cuando los operandos de una instruccion estan _"Ready"_
- Disparar la instruccion ni bien sus operandos estan _"Ready"_

La Reservation Station va a contener opcode, tag, valor, valido (1 y 2) y Control (3). <br> Es particularmente util el RS si hay una cantidad mucho mayor de registros que los registros de la arquitectura. <br>
El indice dentro de la RS es lo que usara la operacion como _Tag_ en la RAT


#### ROB (ReOrder Buffer)
Sirve para cuando utilizamos resolucion especulativa (intentar resolver la operacion sin tener todos los operandos) sin almacenarlo en el operando destino. <br>
El resultado permanecera en el buffer hasta que se haga el _commit_ (sea almacenado en el op dest)
**Implementacion:**
- Ejecucion: Si faltan operandos monitorea el Bus de datos hasta que estos esten disponibles. Ejecuta las operaciones cuyos operandos estan en la RS.
- Write Result: Escribe el resultado en el slot del ROB, si alguna RS espera el resultado tambien lo escribe ahi. 
- Commit: Fase final, resultado almacenado en opDest. Si es con prediccion incorrecta flushea la entrada del ROB y vuelve a empezar usando la direcc correcta.









# Resoluciones
## Coherencia de Cache :
- Explicar cuándo empezamos a tener problemas con la coherencia y cuál es el problema de tener incoherentes la memoria con las caches. 

Nuestros problemas comienzan a surgir a la hora de tener sistemas SMP, que trabajan en simultaneo con sus respectivos caches (en partciular si usamos un metodo de escritura Write Back). El problema de la incoherencia entre memorias parte de que si distintos cores trabajan sobre un mismo dato puede ocurrir que el core X no haya actualizado el valor en memoria luego de realizar una operacion, teniendo entonces en el core Y un valor invalido (desactualizado) sobre el cual trabajara para seguir el programa. 

- Explicar las diferentes políticas de escritura, comparándolas según el uso de procesador y el uso del bus. ¿Cuál es más apta para un sistema Monoprocesador y cuál para un sistema SMP? Justificar.

Write Through y Write Back son las 2 politicas de escritura, en el primer caso a la hora de realizar una escritura esta se vera replicada en todos los niveles de memoria, nos asegura coherencia pero tiene un mayor costo, en particular si realizamos varias escrituras sobre un mismo dato seguido (vamos a escribirlo en todos los niveles N cantidad de veces, pero a lo mejor solo accedemos luego de esas N escrituras). Por otro lado Write Back no asegura coherencia (solo el nivel mas alto tendra el valor actualizado y se vera replicado en los inferiores cuando sea desalojada la linea que lo contiene en el cache Lvl 1), pero nos otorga una velocidad de escritura a la hora de realizar multiples de estas dado que solo vamos a escribirlo en el cache Lvl 1 y luego lo pasaremos a memoria cuando sea absolutamente necesario (algo asi como una escritura lazy). <br> Respecto al uso de procesador y bus entiendo que ambas el write back y lo mismo para cual es mas apta, el tema es que cuando usamos un sistema SMP se va a complicar un poco mas la cosa a nivel hardware, pero habiendo superado eso nos queda algo bastante mas eficaz que usando write through 

- Explicar cómo se podría utilizar Copy Back en un sistema SMP.

- ¿Qué entiende por snooping y con qué elementos se implementa?¿Cómo se complementa con el protocolo MESI?¿Qué cosas se tienen que agregar a nivel de HW para implementar MESI (snoop bus, Shared, RFO)?

El snooping es el concepto de tener cada cache local atenta al contenido enviado por otras caches mediante el bus de sistema, no hace falta agregar un bus dedicado ya que simplemente se fija la direccion que se usa y los permisos read/write del system bus. Se usa para identificar si algun otro Core va a utilizar un dato que otro cache tiene como modified/exclusive. El cache va a necesitar tener bits para definir el estado (MESI)

- En el protocolo MESI, ¿qué significa el estado Modified?

Primero, estado modified implica que ninguna otra cache tiene esa misma linea en estado valido y ademas sabemos que el dato fue modificado mas no aun replicado a los niveles inferiores de memoria.

- MESI, tenés una línea en estado Shared, ¿qué significa?¿Qué pasa si la querés escribir?¿Es impreciso?

Significa que el dato esta contenido en el cache de uno o mas cores (mas o menos, de ahi sale lo impreciso) y que en todos esos otros caches el dato no fue modificado, cuando querramos escribir sobre esa linea se va a producir un aviso a las otras caches para invalidar esa informacion, luego pasaremos al estado modified. <br> 
Es impreciso y esto refiere a que las caches no tienen la obligacion (ni lo hacen) de avisar cuando desalojan una linea que estaba shared, esto puede ocasionar que tengamos una cache con una linea en estado shared sin que ninguna otra cache tenga la linea, ejemplo: <br>
Cache 1 fetchea linea N <br>
Cache 2 fetchea linea N <br>
Cache 1 desaloja linea N para fetchear linea M <br>
Cache 2 tiene estado shared, Cache 1 no tiene linea N <br>  

- Si un procesador quiere leer una línea que él no tiene pero otro cache tiene en estado Modified, ¿qué secuencia de cosas pasan?
Supongamos que la cache que tiene el estado Modified es Cache M y la otra Cache N, Cache N tira el Read Miss (pues no esta en su registro), el bus de sistema tendra permiso de lectura y la direccion buscada, esto sera espiado por Cache M que va a dar aviso de que esa linea fue modificada, frenando el fetcheo y priorizando la secuencia de Write Back (pasando la informacion a los niveles inferiores para segurar coherencia). Luego retomara el fetcheo la cache N y recibira la informacion actualizada. Ambas caches pasaran a estado Shared (Cache M degreadandose de modified a shared luego del copy back y Cache N ascendiendo a Shared cuando la reciba, pues M dara aviso de shared).

- ¿Qué pasa si un procesador escribe en una linea con Modified?¿Cómo afecta a la performance si se usa un protocolo con Write/copy back comparado con Write through?

Lo explicado arriba, si es Write Back es altamente performante ya que solo modificamos el cache lvl 1 (locura total, flash de escrituras) mientras que Write Through debera replicar la informacion en todos los niveles cada vez que la linea sea modificada (muy lenteja, todo mal), respecto al estado de la linea se va a mantener en modified.


## Predicción de Saltos :

- ¿Cómo funciona un predictor de saltos de 2 bits? Motivación y funcionamiento. Incluir diagrama y transiciones de estado.

Tenemos 4 estados, Taken Fuerte/Debil, Non-Taken Fuerte/Debil. <br>
Nos permite ser mas precisos a la hora de predecir saltos de la siguiente forma: 
1. Taken Fuerte,        asumimos Taken       -> Si esta bien la prediccion se queda aca, si esta mal vamos a 2
2. Taken Debil,         asumimos Taken       -> Si esta bien la prediccion vamos a 1, si esta mal pasamos vamos a 3
3. Non-Taken Fuerte,    asumimos Non-Taken   -> Si esta bien la prediccion vamos a 3, si esta mal vamos a 2
4. Non-Taken Debil,     asumimos Non-Taken   -> Si esta bien nos quedamos, si esta mal vamos a 3

- ¿En qué situaciones funciona bien un predictor de saltos de 2 bits y mal uno de 1 bit?

Si tenemos saltos alternados vamos a estar teniendo miss todo el tiempo en 1 bit, en 2 bits vamos a pegarle la mitad de las veces.

- ¿Por qué usar un predictor de 2 bits y no uno de 1 bit, 3 bits, spec89, etc?

Se intentaron distintos metodos, este es el que mantiene una mayor consistencia y hit rate, no creo que tenga mas misterio que eso.

## Ejecución Fuera de Orden

- Concepto y funcionamiento general.¿Qué nuevas dependencias se introducen con la ejecución fuera de orden?



Ventajas respecto de un esquema superescalar con ejecución en orden. Considerar que ambos modelos tienen la misma cantidad de vías de ejecución.
Algoritmo de Tomasulo
¿Cuándo se debe stallear una instrucción?
Explicar cuáles son los bloques de hardware que se agregan a un procesador superescalar, qué riesgos resuelve, y cómo funciona cada uno.
¿Qué elementos tiene una Reservation Station?
¿Cómo se establece la relación consumidor/productor según Tomasulo?¿Dónde está el tag o a qué hace referencia?
Detallar secuencia de pasos para ejecutar una instrucción.
Reorder Buffer
¿Qué le faltó al algoritmo de Tomasulo para tener excepciones precisas?
¿Qué elementos tiene un reorder buffer?
Explicar la implementación de Intel del Algoritmo de Tomasulo en el Three Cores Engine, detallando cada parte involucrada.
