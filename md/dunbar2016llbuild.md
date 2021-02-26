\addcontentsline{toc}{section}{A New Architecture for Building Software}
\section*{2016 LLVM Developers’ Meeting: D. Dunbar “A New Architecture for Building Software”}
\cite{dunbar2016}

## Overview

  Los tiempos de compilacion impactan a los desarrolladores.
  Clang fue diseñado desde el principio para ser un compilador rapido de C/C++.
  La estrategia inicial fue
  - tunear la implementacion de lex y parseo,
  - enfocarse en tener un flujo de ejecucion con -O0 con muy poca sobrecarga, sin optimizaciones innecesarias,
  - rediseñar la implementacion de PCH (archivos de encabezado pre-compildos), intentando obtener la cantidad minima necesaria de los archivos de encabezado,
  - integrar el ensamblador en el compilador, evitando el tiempo utilizado para emitir codigo ensamblador y luego cargarlo en el ensamblador.

  Esta estrategia fue exigtosa, y en sus inicios clang era 3x mas rapido que gcc, pero con el tiempo esa diferencia se fue achicando, en parte porque gcc mismo mejoro sus tiempos de compilacion pero sobre todo porque la performance es un atributo que sufre regresiones con la adicion de funcionalidad.

  A la hora de mejorar el tiempo de compilacion, hay varias opcionesÑ
  - compilacion distribuida
  - cacheo mejorado, idealmente distribuido y compartido
  - hacer menos trabajo

  En el caso de clang, hay cosas que se pueden implementar hoy sin grandes cambios, tales como reutilizar el frontend del compilador a lo largo de la compilacion de varios archivos que comparten los mismo flags de compilacion, permitiendo que el frontend cachee mejor, e.g. stat syscalls y otra metainformacion, o PCHs para archivos que se editan frecuentemente.

  Aunque esto funciona, el problema es que el compilador no tiene control sobre como es invocado, dado que esto depende del build system.

  Una propuesta es construir un servicio de compilador, pero esto presenta otros problemas.
  \cite{dunbar2016}

## Como se construye software hoy

  Para la mayoria de los lenguajes, se sigue un modelo de build tradicional basado en unix
  - el compilador corre como un proceso separado
  - mecanismos primitivos para comunicar dependencias entre el build system y el compilador:
    - el build system informa las entradas y salidas a traves de argumentos de linea de comando
    - ausencia de un buen mecanismo mediante el cual el compilador puede informar nuevas dependencias al build system
    - entrada/salida fija: existe un conjunto limitado de lugares en los cuales se puedeen leer y escribir datos; si el compilador quisera cachear algo durante la sesion de compilacion, no hay un lugar estandard o consensuado donde hacerlo.

  Desde cierto punto de vista, esto constituye un contrato API, pero es un API que no ha cambiado en decadas.
  \cite{dunbar2016}

## Como se podria construir software

  Que sucederia si estuviesemos dispuestos a rompar este API?
  Cosas que se podrian hacer:
  - tablas de lookup ad-hoc:
    en muchos momentos del ciclo de compilacion a traves del build, el compilador computa informacion que podria ser acelerado a traves de una tabla de lookup, y si esa tabla pudiese ser persistida a lo largo de varias sesiones de compilacion se podria acelerar el build.
    Hoy, esto involucraria serializar a disco y volver a leerlo, mitigando la reduccion en tiempo de compilacion.
  - salida temprana del proceso de compilacion a traves firmas de codigo:
    se podria implementar la capacidad de tomar la signatura del IR LLVM que emite el codegen, y si la firma es igual a la ultima vez que se compilo ese archivo, se saltea ese backend.
    Esto puede hacer una gran diferencia, en la mayoria de los compiladores el backend de generacion de codigo domina los tiempos de compilacion, si lo que se cambia es un comentario y el codigo objeto que obtendriamos seria exactamente igual.
  - resolucion de instanciaciones de templates/monomorfizacion redundantes:
    se pasa mucho tiempo en un build repitiendo trabajo al instanciar el mismo template, creandolos, verificando los tipos, generando codigo, y finalmente pasando todo el resultado al linker, solo para que elimine la mayoria.

  Para todos estos casos lo que se requiere es evolucionar el API entre el compilador y el build system.
  \cite{dunbar2016}

## Y el cache de modulos?

  Clang tiene un cache de modulos, y es un ejemplo de un sistema que cachea algo a lo largo de todo el build.
  - automaticamente hace el build de modulos cuando es necesario
  - comparte los resultados a lo largo de todo el build
  - no se requiere modificar el sistema de build

  Pero este no es un buen ejemplo porque involucra una complejidad de implementacion considerable.
  - utiliza lockeo de archivos POSIX para coordinar
  - utiliza su propio esquema de manejo de consistencia de cache, tiene pocas herramientas de debuggeo, y tiene que observar los archivos y entender cuando necesita rehacer el build de modulos
  - tiene su propio mecanismo de desalojo de cacheo
  - es opaco al scheduler del sistema de build, si uno comienza un conjunto de tareas de build y todas requieren del mismo modulo, desde el punto de vista del sistema de build, va a bloquear esperando a esos modulos, pero podria estar haciendo mas trabajo
  \cite{dunbar2016}

## El modelo ideal de construccion de software

  Lo que realmente se necesita es un API flexible entre el compilador y el sistema de build.

  Objetivos:

  - deberia ser facil compartir trabajo redundante, si hay un lugar en el compilador que realiza trabajo que es redundante a lo largo del ciclo de vida del build, deberia ser facil implementar eso
  - optimizar el compilador para el build entero: a la mayoria de los usuarios, exceptuando de los ingenieros de compiladores, no les importa los archivos objetos, les importa el producto final.
    esta es la situacion que deberia optimizarse, ser rapido para el build completo
  - conversamente, deberia ser posible optimizar el sistema de build a traves de un API de compilador rico.
    si el compilador se da cuenta que podria realizar trabajo en paralelo, seria ideal que se le pueda comunicar con sistema de build para poder ingresar la tarea en el schedule de manera efectiva
  - builds incrementales consistentes y con una arquitectura debuggeable: uno de los beneficios de la arquitectura actual es que se hacen grandes esfuerzos para mantener una garantia fuerte de que el mismo input producira exactamente el mismo archivo de salida, porque es la base de tener builds reproducibles
  - necesito poder integrar el compilador con el sistema de build

  Requiere:
  - un compilador basado en librerias, i.e. un compilador cuya architectura tiene en cuenta su reutilizacion
  - un sistema de build extensible, i.e. un sistema de build que ha sido diseñado con la idea de que las herramientas con las que interactua deberian poder conectar mas profundamente que solo a traves de un subproceso
    cuando se invoca make o ninja, esperan que se los invoque a traves de un subproceso, y hay pocas maneras de poder interactuar con ellos
  - plugins de compilador
  \cite{dunbar2016}

## Presentando llbuild

  llbuild es un ejemplo de un libreria que provee funcionalidad para construir build systems.
  - open source
  - libreria c++
  - parte de llvm, swift
  - contiene una implementacion de ninja

  Objetivos:
  - ignorar el requerimiento de un lenguaje de input/descripcion de build
    no hacer una sintaxis nueva o reemplazo de cmake
    la mayoria de los build systems tienen profundo adentro un pequeño motor que es capaz de evaluar un grafo de dependencias, y que usualmente es bastante simple
  - enfocarse en construir un motor poderoso
    - soporte para descubrir tareas a realizar dinamicamente
      pocos sistemas de build dan soporte para agregar mas trabajo descubierto durante la ejecucion de una tarea
    - escalar a millones de tareas
      dado que el objetivo es tomar el sistema de build actual y particionarlo en tareas mas chicas para obtener mejor comportamiento incremental, lo que realmente se necesita es poder escalar
    - soportar scheduling de tareas dinamico
      durante un build de e.g. llvm se tienen muchas tareas cpu-bound, como correr el compilador, y muchas taras io-bound, como correr el linker
    - soportar un api 'pluggable'
  \cite{dunbar2016}

### llbuild architecture

  - motor subyacente flexible
  - el equivalente a nivel computacion del IR de LLVM, el cual es un comun denominador entre las cosas de alto nivel, como sistemas de build, y las optimizaciones de mas bajo nivel
  - libreria para computacion incremental y persistente
  - fuertemente inspirado en un sistema de build en haskell llamado shake
  - de bajo nivel:
    - las entradas y las salidas son secuencias de bytes
      no hay archivos, solo bytes, y la expectativa que los clientes (de)codificaran sus datos
    - las funciones son abstractas
    - c++ como api entre tareas
  - sobre este nucleo se construyen sistemas de build de mas alto nivel (e.g. una implementacion de ninja, un manejador de paquetes)
  \cite{dunbar2016}

### llbuild engine

  Como seria este motor del modelo minimal y funcional?
  Cuatro conceptos basicos:
  - claves: un nombre no ambiguo para una computacion que se quiere ejecutar
  - valor: el resultado de una computacion
  - regla: como producir un valor, dado una clave
  - tarea: una instancia ejecutandose de una regla
    una tarea puede solicitar otras claves de entrada como parte de su trabajo

  El nucleo del motor puede ser usado directamente para propositos de computacion general
  \cite{dunbar2016}

### Ejemplo: ackerman

  Las funciones recursivas forman un grafo natural, dado que cada resultado depende de entradas recursivas
  Para construir ackerman, codificamos la invocacion de ackerman que queremos ejecutar como una clave, y codificamos el entero resultante como un valor.
  Tomamos esa clave y la mapeamos a una tarea utilizando las reglas, y las tareas mismas implementan la funcion de ackerman.

  Se implementa un wrapper struct que envuelve el par de enteros que son los argumentos a la funcion.
  Este struct implementa un par de funciones para convertir a y desde una representacion serializada.
  Asimismo con el valor.

  La forma de implementar las reglas en llbuild es con un delegate.
  El motor de llbuild puede soportar esta idea de trabajo siendo generado dinamicamente porque le pasas un delegate, que es una funcion que provee las reglas que necesita cuando la invoca.
  De esta manera, a diferencia de un makefile o ninja  file, en el cual todas las reglas estarian presentes de antemano, desde el punto de vista del motor solo tiene una funcion que invoca a medida que van entrando las tareas.

  En este caso, la funcion sencillamente decodifica la clave y crea una tarea.
  Tambien se encarga de deteccion de ciclos.

  Asi, las tareas es donde realmente se ejecuta el trabajo.
  El motor invoca metodos de las tareas a medida que se realiza el trabajo del build.

  Cuando se comienza una tarea, se recibe una notificacion.
  Cuando el motor ha computado un resultado que se solicito, se lo devuelve al codigo que lo solicito.
  Cuando todas las entradas que tu tarea solicito han sido computados, entonces completa la tarea inicial y termina.
  \cite{dunbar2016}

## Una arquitectura nueva

  Requiere:
  - un compilador basado en librerias
  - un sistema de build extensible
  - un sistema de plugins de compilador
  \cite{dunbar2016}

## Estado actual de llbuild

  llbuild es el motor del sistema de build de XCode 10.
  Pasar a "la nueva arquitectura" sigue siendo dificil, principalmente por la dificultad de refactorear compiladores existentes, y los beneficios se empiezan a ver recien cuando la mayor parte del trabajo se ha hecho.
  Es dificil deshacer 40 años de precedente de un dia para el otro.
  Desde su publicacion, la implementacion de ninja basada en llbuild tiene mejor performance en la mayoria de las situaciones de mundo real (e.g. build de clang/llvm en variedad de hardware distinto).
  \cite{dunbar2016}
