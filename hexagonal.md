# Arquitectura Hexagonal

Una de las importantes lecciones que un desarrollador de software puede aprender es familiarizarse con el concepto de arquitectura hexagonal, también conocida como arquitectura de puertos y adaptadores, propuesta inicialmente por Alistair Cockburn.

## Historia

La popular arquitectura de capas, conocida también como arquitectura de 3-capas o N-capas, que se hizo popular a comienzos de siglo con el advenimiento de las aplicaciones cliente/servidor, propone separar los componentes capas, tradicionalmente 3:

* Presentación
* Lógica de negocios
* Datos

&#8203;

       ,------------.     ,------------------.     ,--------------.  
       |Presentación| --> |Lógica de Negocios| --> |Acceso a Datos|    
       `------------'     `------------------'     `--------------'  

Por ejemplo, una aplicación de un servicio REST, tendría en su capa de *presentación* una clase para exponer el punto de contacto REST `GET /cuentas/{id}`.

En Java, un archivo `ControladorRESTCuentas.java` contiene la clase de la capa `presentación`

    package presentacion;
    
    import logica_de_negocios.ServicioCuentas;
    
    @RestController
    class ControladorRESTCuentas {
    
      ServicioCuentas servicio;
    
      @GetMapping("/cuentas/{id}")
      Cuenta obtenerPorId(@PathVariable Long id) {
          return servicio.obtenerPorId(id);
      }
    }

Lo importante de notar aquí es que la clase `ControladorRESTCuentas` tiene una instrucción:

    import logica_de_negocios.ServicioCuentas;

Este `import` es la dependencia representada por la flecha:

       ,-----------------------------.     ,---------------------. 
       |presentacion                 |     |logica_de_negocios   |
       |  ,------------------------. |     | ,-----------------. |
       |  | ControladorRESTCuentas | ------> | ServicioCuentas | |
       |  `------------------------' |     | `-----------------' |
       `-----------------------------'     `---------------------'

Es decir, para compilar correctamente, `presentacion.ControladorRESTCuentas` **necesita** de `logica_de_negocios.ServicioCuentas`, pero **no** vice versa.

Igualemente, una clase en la capa `logica_de_negocios` tendría una dependencia hacia una clase de la capa `acceso_a_datos`:

    package logica_de_negocios;
    
    import acceso_a_datos.MySqlCuentas;
    
    @Service
    class ServicioCuentas {
    
      MySqlCuentas repo;
    
      Cuenta obtenerPorId(Long id) {
          return repo.selectCuentaPorId(id);
      }
    }

Es una implementación un poco cruda, porque no hay lógica de negocios como tal, solo una llamada directa a la capa de `acceso_a_datos`, pero está bien, nos queremos concentrar en las dependencias. Otra vez, aquí la dependencia está marcada por:

    import acceso_a_datos.MySqlCuentas;

Y es la dependencia representada por la flecha:

       ,------------------------.     ,---------------------. 
       |logica_de_negocios      |     |acceso_a_datos       |
       |  ,-------------------. |     | ,-----------------. |
       |  | ServicioCuentas   | ------> | MySqlCuentas    | |
       |  `-------------------' |     | `-----------------' |
       `------------------------'     `---------------------'

## ¿Cúal es el problema?

Hasta ahora todo parece bien: Las capas lógicas nos permiten separar el código de presentación, el código aplicativo y el código de acceso a las bases de datos en diferentes paquetes. ¿Por qué no es suficiente este tipo de separación lógica?

Imaginemos que ahora necesitamos cambiar la implementación de la clase `acceso_a_datos.MySqlCuentas` por una clase `acceso_a_datos.MongoDbCuentas`, y ya no regrese una clase tipo `Cuenta` sino que regrese una clase `Map<String, Object>`. Ahora nos vemos obligados no solamente a introducir la nueva clase `MongoDbCuentas`, sino también a **cambiar n el código de la clase dependiente:**`ServicioCuentas`.

    package logica_de_negocios;
    
    import acceso_a_datos.MongoDbCuentas; // esta linea de código cambia
    
    @Service
    class ServicioCuentas {
    
      MongoDbCuentas repo; // Esta linea de código cambia
    
      Cuenta obtenerPorId(Long id) {
          // el cuerpo del método también cambia.
          Map<String, Object> rec = repo.selectCuentaPorId(id);
          return transformarRecToCuenta(rec);
      }
    }

Este cambio de código en la clase `ServicioCuentas` se ve representado por una nueva flecha hacia `MongoDbCuentas`.

       ,------------------------.     ,---------------------. 
       |logica_de_negocios      |     |acceso_a_datos       |
       |  ,-------------------. |     | ,-----------------. |
       |  | ServicioCuentas   | |     | | MySqlCuentas    | |
       |  `-------------------' \     | `-----------------' |
       `------------------------'\    | ,-----------------. |  
                                  `---> | MongoDbCuentas  | |
                                      | `-----------------' |
                                      `---------------------'

Parece un esfuerzo muy simple en este ejemplo, pero una aplicación real puede tener fácilmente docenas de clases en la capa de `logica_de_negocios` que nos veriamos obligados a cambiar.

## Interfaces al rescate

El concepto de *interfaz* o *contrato* existe para proteger las clases de sus dependencias: Consiste en definir las operaciones que se esperan de una dependencia (un *contrato*), y la dependencia se crea hacia ese *contrato* (o *interfaz*) en lugar de depender directamente de la clase.

¿Cómo sería entonces el cambio descrito en la sección anterior usando *interfaces*?

Digamos que en lugar de la clase `MySqlCuentas`, `ServicioCuentas` depende de una interfaz `RepositorioCuentas`:

    interface RepositorioCuentas {
        Cuenta obtenerCuenta(Long id);
    }

El código de la clase `ServicioCuentas` sería:

    package logica_de_negocios;
    
    import acceso_a_datos.RepositorioCuentas;
    
    @Service
    class ServicioCuentas {
    
      RepositorioCuentas repo;
    
      Cuenta obtenerPorId(Long id) {
          return repo.obtenerCuenta(id);
      }
    }

En este caso, es la responsabilidad de la clase `MySqlCuentas` de satifacer el contrato de `RepositorioCuentas`:

    package acceso_a_datos;
    
    @Service
    class MySqlCuentas implements RepositorioCuentas {
    
      Cuenta obtenerCuenta(Long id) {
          // Código implementando la intefaz
      }
    }

En importante notar el `import` que la clase `MySqlCuentas` debe hacer en este caso, al implementar la interfaz `RepositorioCuentas`, debe importarla.

Gráficamente, estas dependencias serían:

       ,------------------------.     ,------------------------. 
       |logica_de_negocios      |     |acceso_a_datos          |
       |  ,-------------------. |     | ,--------------------. |
       |  | ServicioCuentas   | ------> | RepositorioCuentas | |
       |  `-------------------' |     | `--------------------' |
       `------------------------'     |            ∆           |
                                      |            |           |
                                      |   ,----------------.   |
                                      |   |  MySqlCuentas  |   |
                                      |   `----------------'   |
                                      `------------------------'

El diagrama deja en evidencia el beneficio de utilizar la intefaz: Si cambiamos `MySqlCuentas` por `MongoDbCuentas` (quién también debe satifacer la intefaz `RepositorioCuentas`), el cambio del código queda limitado dentro del paquete `acceso_a_datos`:

       ,------------------------.     ,--------------------------------------------. 
       |logica_de_negocios      |     |acceso_a_datos                              |
       |  ,-------------------. |     | ,--------------------.                     |
       |  | ServicioCuentas   | ------> | RepositorioCuentas |                     |
       |  `-------------------' |     | `--------------------'                     |
       `------------------------'     |            ∆                               |
                                      |            `-------------------.           |
                                      |                                |           |
                                      |   ,----------------.   ,----------------.  |
                                      |   |  MySqlCuentas  |   |  MySqlCuentas  |  |
                                      |   `----------------'   `----------------'  |
                                      `--------------------------------------------'

**el código en la capa de** `logica_de_negocios` **queda exactamente igual** dado que la dependencia de `ServicioCuentas` es con `RepositorioCuentas` y no con `MySqlCuentas`. Las lineas que código que antes cambiaban cuando no teníamos una *interfaz* de por medio, ahora quedan iguales:

    package logica_de_negocios;
    
    // el import NO cambia
    import acceso_a_datos.RepositorioCuentas;
    
    @Service
    class ServicioCuentas {
    
      // la referencia a la intefaz NO cambia
      RepositorioCuentas repo;
    
      Cuenta obtenerPorId(Long id) {
          // el llamado a obtenerCuenta NO cambia
          return repo.obtenerCuenta(id);
      }
    }
    
    // ¡NADA cambia!

Este mismo patrón se puede aplicar para desacoplar la dependencia entre la capa `presentacion` y la capa `logica_de_negocios`:

    ,------------------.   ,---------------------. 
    |presentacion      |   |logica_de_negocios   |
    |  ,------ ------. |   | ,-----------------. |
    |  | RESTCuentas | ----> | GestionCuentas  | |
    |  `-------------' |   | `-----------------' |
    `------------------'   |          ∆          |     ,------------------------. 
                           |          |          |     |acceso_a_datos          |
                           | ,-----------------. |     | ,--------------------. |
                           | | ServicioCuentas | ------> | RepositorioCuentas | |
                           | `-----------------' |     | `--------------------' |
                           `---------------------'     |            ∆           |
                                                       |            |           |
                                                       |   ,----------------.   |
                                                       |   |  MySqlCuentas  |   |
                                                       |   `----------------'   |
                                                       `------------------------'

El diagrama anterior ilustra el fundamento principal en el cuál se base la arquitectura hexagonal: Desacoplamiento de componentes a través de contratos.

Una versión más generalizada del diagrama sería:

    ,---------.       ,---------. 
    | Cliente | ----> | Puerto  | 
    `---------'       `---------' 
                           ∆     
                           |     
                    ,------------.       ,-----------. 
                    | Componente | ----> | Adaptador | 
                    `------------'       `-----------' 
                                                ∆           
                                                |           
                                        ,-------------.   
                                        |  Proveedor  |   
                                        `-------------'                                                 

Y de ahí el nombre alternativo, arquitectura de *puertos* y *adaptadores*.

El nombre arquitectura *hexagonal* viene de un diagrama comúnmente utilizado para representar el mismo concepto:

    ,---------.           ,--------.
    | Cliente | ----> Puerto        \  
    `---------'         /            \  
                       (  Componente  )
                        \            /   
                         \        Adaptador
                          `--------'  ∆           
                                      |           
                               ,-------------.   
                               |  Proveedor  |   
                               `-------------'          
