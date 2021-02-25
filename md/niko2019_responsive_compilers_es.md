Muchos libros de cabecera y cursos sobre compiladores tratan a la compilacion como una tarea de procesamiento por lotes, en la cual el compilador toma archivos de entrada, ejecuta un conjunto de pasadas del compilador, y en ultima instancia producen codigo objeto como salida.
Cada vez mas, los usuarios esperan integracion con IDEs como VSCode, lo cual requiere de una estructura diferente.
Mas aun, muchos lenguajes tienen construcciones recursivas en las cuales el orden de procesamiento es dificil de determinar estaticamente.
Nicholas hablara sobre algo del trabajo que ha realizado el equipo de Rust para reestructurar el compilador para soportar la compilacion incremental e integracion con IDEs.

https://nikomatsakis.github.io/pliss-2019/responsive-compilers.html

<!-- WHY --------------------------------------------------------------------->

# Pipelines y Pasadas