# Arquitectura Hexagonal

Los desarrolladores de software nos encuentramos repetidamente con la misma pregunta: **¿Cómo organizar los componentes y el código de un proyecto?**. 

[Alistair Cockburn contestó a esta pregunta en el 2005](https://alistair.cockburn.us/hexagonal-architecture/) con un patrón de arquitectura genérico: La _arquitectura hexagonal_, también conocida como _arquitectura de puertos y adaptadores_.

             ,-----------------. 
            / Infraestructura   \
           /    ,------------.   \
          /    / Aplicación   \   \
         /    /    ,------.    \   \
        /    /    /        \    \   \
       (    (    (  Dominio )    )   )
        \    \    \        /    /   /
         \    \    `------'    /   /
          \    \              /   /
           \    `-----------'    /
            \                   /
             `-----------------'

Esta arquitectura tiene como objetivo minimizar el acoplamiento entre componentes, reduciendo el impacto al remplazar componentes y facilitando el uso de _mocks_ en la automatización de pruebas.

## Historia

La popular _arquitectura de capas_, conocida también como _arquitectura de 3-capas_ o _N-capas_, que se hizo popular a comienzos de siglo con el advenimiento de las aplicaciones cliente/servidor, propone separar los componentes en capas, tradicionalmente en 3 capas:

![](hexagonal.arch.springboot.template-Page-7.drawio.png)

      ,------------. 
      |Presentación| 
      `------------'
            |
            V
    ,------------------.
    |Lógica de Negocios|
    `------------------'
            |
            V
      ,--------------.  
      |Acceso a Datos|   
      `--------------'  

Por ejemplo, una aplicación de un servicio REST, tendría en su capa de _presentación_ una clase para exponer el punto de contacto REST `GET /cuentas/:id`.

En Java, una implemtación posible sería una clase `ControladorRESTCuentas` en el paquete `presentación`:

    package presentacion;
    
    import logica_de_negocios.ServicioCuentas;
    
    class ControladorRESTCuentas {
    
      ServicioCuentas servicio;
    
      // Responde a un pedido HTTP GET /cuentas/:id
      Cuenta obtenerPorId(long id) {
          return servicio.obtenerPorId(id);
      }
    }

Lo importante de notar en esta clase `ControladorRESTCuentas` es la instrucción:

    import logica_de_negocios.ServicioCuentas;

Este `import` es la dependencia representada por la flecha:

       ,-----------------------------.
       |presentacion                 |
       |  ,------------------------. |
       |  | ControladorRESTCuentas | |
       |  `------------------------' |
       `----------------|------------'
                        |
    ,-------------------|-------------.
    |logica_de_negocios V             |
    |         ,-----------------.     |
    |         | ServicioCuentas |     |
    |         `-----------------'     |
    `---------------------------------'

Es decir, para compilar correctamente `presentacion.ControladorRESTCuentas` se **necesita** acceso a `logica_de_negocios.ServicioCuentas`, pero **no** vice versa.

Igualemente, una clase en la capa `logica_de_negocios` tendría una dependencia hacia una clase de la capa `acceso_a_datos`:

    package logica_de_negocios;
    
    import acceso_a_datos.MySqlCuentas;
    
    class ServicioCuentas {
    
      MySqlCuentas repo;
    
      Cuenta obtenerPorId(long id) {
          return repo.selectCuentaPorId(id);
      }
    }

Es una implementación un poco cruda, porque no hay lógica de negocios como tal, solo una llamada directa a la capa de `acceso_a_datos`, pero es suficiente para el propósito de este artículo, nos queremos concentrar en las dependencias. Otra vez, aquí la dependencia está marcada por:

    import acceso_a_datos.MySqlCuentas;

Y es la dependencia representada por la flecha:

       ,------------------------.
       |logica_de_negocios      |     
       |  ,-------------------. |    
       |  | ServicioCuentas   | |
       |  `-------------------' |   
       `--------------|---------'    
     ,----------------|---------. 
     |acceso_a_datos  V         |
     |      ,-----------------. |
     |      | MySqlCuentas    | |
     |      `-----------------' |
     `--------------------------'

## ¿Cúal es el problema?

Hasta ahora todo parece bien: Las capas lógicas nos permiten separar el código de presentación, el código aplicativo y el código de acceso a las bases de datos en diferentes paquetes. ¿Por qué no es suficiente este tipo de separación lógica?

Imaginemos que ahora necesitamos cambiar la implementación de la clase `acceso_a_datos.MySqlCuentas` por una clase `acceso_a_datos.MongoDbCuentas`, y ya no regrese una clase tipo `Cuenta` sino que regrese una clase `Map<String, Object>`. Ahora nos vemos obligados no solamente a introducir la nueva clase `MongoDbCuentas`, sino también a **cambiar el código de la clase dependiente:** `ServicioCuentas`.

    package logica_de_negocios;
    
    import acceso_a_datos.MongoDbCuentas; // esta linea de código cambia
    
    class ServicioCuentas {
    
      MongoDbCuentas repo; // Esta linea de código cambia
    
      Cuenta obtenerPorId(long id) {
          // el cuerpo del método también cambia.
          Map<String, Object> rec = repo.selectCuentaPorId(id);
          return transformarRecToCuenta(rec);
      }
    }

Este cambio de código en la clase `ServicioCuentas` se ve representado por una nueva flecha hacia `MongoDbCuentas`.

       ,------------------------.
       |logica_de_negocios      |
       |  ,-------------------. |
       |  | ServicioCuentas   | |
       |  `-------------------' |
       `-----------/-------\----'
                  x         \
    ,------------------------\---------.
    |acceso_a_datos           v        |
    | ,------------.  ,--------------. |
    | |MySqlCuentas|  |MongoDbCuentas| |
    | `------------'  `--------------' |
    `----------------------------------'
    
Parece un esfuerzo muy simple en este ejemplo, pero una aplicación real puede tener fácilmente docenas de clases en la capa de `logica_de_negocios` que nos veriamos obligados a cambiar.

## Interfaces al rescate

El concepto de *interfaz* o *contrato* existe para proteger las clases de sus dependencias: Consiste en definir las operaciones que se esperan de una dependencia (un *contrato*), y la dependencia se crea hacia ese *contrato* (o *interfaz*) en lugar de depender directamente de la clase.

¿Cómo sería entonces el cambio descrito en la sección anterior usando *interfaces*?

Digamos que en lugar de la clase `MySqlCuentas`, `ServicioCuentas` depende de una interfaz `RepositorioCuentas`:

    interface RepositorioCuentas {
        Cuenta obtenerCuenta(long id);
    }

El código de la clase `ServicioCuentas` sería:

    package logica_de_negocios;
    
    import acceso_a_datos.RepositorioCuentas;
    
    class ServicioCuentas {
    
      RepositorioCuentas repo;
    
      Cuenta obtenerPorId(long id) {
          return repo.obtenerCuenta(id);
      }
    }

En este caso, es la responsabilidad de la clase `MySqlCuentas` de satifacer el contrato de `RepositorioCuentas`:

    package acceso_a_datos;
    
    class MySqlCuentas implements RepositorioCuentas {
    
      Cuenta obtenerCuenta(long id) {
          // Código implementando la intefaz
      }
    }

En importante notar el `import` que la clase `MySqlCuentas` debe hacer en este caso, al implementar la interfaz `RepositorioCuentas`, debe importarla.

Gráficamente, estas dependencias serían:

       ,------------------------.   
       |logica_de_negocios      |   
       |  ,-------------------. |   
       |  | ServicioCuentas   | |
       |  `-------------------' |
       `-------------|----------' 
                     |
    ,----------------|------------. 
    |acceso_a_datos  V            |
    |    ,----------------------. |
    |    |  RepositorioCuentas  | |
    |    `----------------------' |
    |               ∆             |
    |               |             |
    |      ,------------------.   |
    |      |   MySqlCuentas   |   |
    |      `------------------'   |
    `-----------------------------'

El diagrama deja en evidencia el beneficio de utilizar la intefaz: Si cambiamos `MySqlCuentas` por `MongoDbCuentas` (quién también debe satifacer la intefaz `RepositorioCuentas`), el cambio del código queda limitado dentro del paquete `acceso_a_datos`:

        ,------------------------.
        |logica_de_negocios      |
        |  ,-------------------. |
        |  | ServicioCuentas   | |
        |  `-------------------' |
        `------------|-----------'
                     |
    ,----------------|---------------------. 
    |acceso_a_datos  V                     |
    |       ,--------------------.         |
    |       | RepositorioCuentas |         |
    |       `--------------------'         |
    |               ∆      ∆               |
    |           X---'      `-----.         |
    |                            |         |
    | ,--------------.  ,----------------. |
    | | MySqlCuentas |  | MongoDbCuentas | |
    | `--------------'  `----------------' |
    `--------------------------------------'

**el código en la capa de** `logica_de_negocios` **queda exactamente igual** dado que la dependencia de `ServicioCuentas` es con `RepositorioCuentas` y no con `MySqlCuentas`. Las lineas que código que antes cambiaban cuando no teníamos una *interfaz* de por medio, ahora quedan iguales:

    package logica_de_negocios;
    
    // el import NO cambia
    import acceso_a_datos.RepositorioCuentas;
    
    @Service
    class ServicioCuentas {
    
      // la referencia a la intefaz NO cambia
      RepositorioCuentas repo;
    
      Cuenta obtenerPorId(long id) {
          // el llamado a obtenerCuenta NO cambia
          return repo.obtenerCuenta(id);
      }
    }
    
    // ¡NADA cambia!

Este mismo patrón se puede aplicar para desacoplar la dependencia entre la capa `presentacion` y la capa `logica_de_negocios`:

            ,------------------.
            |presentacion      |
            |  ,------ ------. |
            |  | RESTCuentas | |
            |  `-------------' |
            `-----------|------'
    ,-------------------|-----------. 
    |logica_de_negocios V           |
    |           ,-----------------. |
    |           | GestionCuentas  | |
    |           `-----------------' |
    |                    ∆          |
    |                    |          |
    |         ,-----------------.   |
    |         | ServicioCuentas |   |
    |         `-----------------'   |
    `--------------------|----------'
        ,----------------|----------. 
        |acceso_a_datos  V          |
        |      ,------------------. |
        |      |RepositorioCuentas| |
        |      `------------------' |
        |                ∆          |
        |                |          |
        |       ,----------------.  |
        |       |  MySqlCuentas  |  |
        |       `----------------'  |
        `---------------------------'


El diagrama anterior ilustra el fundamento principal en el cuál se base la arquitectura hexagonal: Desacoplamiento de componentes a través de contratos.

Una versión más generalizada del diagrama sería:

    ,-------.   ,------. 
    |Cliente|-->|Puerto| 
    `-------'   `------' 
                    ∆
                    |
             ,----------.   ,-------.
             |Componente|-->|Puerto |
             `----------'   `-------'
                                ∆
                                |
                            ,---------.
                            |Proveedor|
                            `---------'

## Otra Perspectiva

El diagrama anterior es la pieza de Lego con la que se construye la *arquitectura hexagonal*.

La *arquitectura hexagonal* propone una separación en capas concentricas, dónde las dependecias van de afuera hacía adentro, es decír, **las flechas solamente deben apuntar hacía las capas más internas. Nunca una capa debe tener dependencias hacía una capa más externa. dónde las capas más externas**.

            ,-----------------. 
            / Infraestructura   \
           /    ,------------.   \
          /    / Aplicación   \   \
         /    /    ,------.    \   \
        /    /    /        \    \   \
       (    (    (  Dominio )    )   )
        \    \    \        /    /   /
         \    \    `------'    /   /
          \    \              /   /
           \    `-----------'    /
            \                   /
             `-----------------'

La capa más externa, llamda *capa de infraesctura* es la más cercanas a la tecnología y la más propensa a cambiar, mientras que las capas internas son las más cercanas a la lógica de negocios que no cambia de una tecnología a otra.


      ,----------------------------.
     / Infraestructura              \
    /                ,--------.      \
    ,---------.     /Aplicación\      \
    |Adaptador|-->Puerto        \      \
    `---------'   / ,----------. \      \
                 (  |Componente|  )      \
                  \ `----------' /        \
                   \          Puerto       )
                    `----------' ∆        /
                                 |       /
                           ,---------.  /
                           |Adaptador| /   
    \                      `---------'/
     \                               /
      `-----------------------------'

Es de notar que tanto los componentes de `presentación` como los componentes de `acceso a datos` se encuentran en la misma capa en la *arquitectura hexagonal*: en la capa de `infraestructura`. La capa intermedia: capa de `aplicación`, contienue los components con la lógica de negocios y finalmente, el núcleo: capa `dominio` contiene los componentes reutilizables de una aplicación a otra.

