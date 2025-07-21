
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


    


