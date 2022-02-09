Muchos libros de cabecera y cursos sobre compiladores tratan a la compilacion como una tarea de procesamiento por lotes, en la cual el compilador toma archivos de entrada, ejecuta un conjunto de pasadas del compilador, y en ultima instancia producen codigo objeto como salida.
Cada vez mas, los usuarios esperan integracion con IDEs como VSCode, lo cual requiere de una estructura diferente.
Mas aun, muchos lenguajes tienen construcciones recursivas en las cuales el orden de procesamiento es dificil de determinar estaticamente.
Nicholas hablara sobre algo del trabajo que ha realizado el equipo de Rust para reestructurar el compilador para soportar la compilacion incremental e integracion con IDEs.

https://nikomatsakis.github.io/pliss-2019/responsive-compilers.html

<!-- WHY --------------------------------------------------------------------->

# Pipelines y Pasadas (niko2019respcomp)

  When I started writing compilers we used the Dragon Book as a reference, which teaches this classic structure of how to write a compiler in passes.
  The way compilers usually work is this batch compilation model, in which one runs the compiler, process the whole source and produce an output and maybe gets an error out of it.

  Traditional compiler model is a series of passes:
  - lex(source) -> tokens
  - parse(tokens) -> ast
  - semantic_analysis(ast), type_check(ast)
  - loop: apply optimizations
  - etc

  That is precisely how `rustc`, the Rust compiler, was written in the beginning, and still looks like today in some ways, since the effort to move away from this architecture is ongoing.

  The reason that there has been a change is that the way one interacts with compilers has changed.
  These days people work with IDEs and want a different way to interact with the source in this model.
  It's necessary to accept erroneous inputs and make sense of them, perform source code completions, and jump to definitions in an interactive way, and to do this one needs to process just enough to answer the user's query.

  What if the Dragon Book were written today?

  There seem to be more questions than answers, not everything needed to make the book has been written down, but the rustc implementation community has a lot of experiences of what they tried and the challenges that come with those experiences.

  The first thing to learn about this environment today is that there has been a big shift during the last couple of years in how IDEs are written. Microsoft introduced VS Code, which is an amazing editor, but among the many amazing things it introduced is the LSP (Language Server Protocol), which is an intermediate protocol for interfacing between the language that's being compiled and the editor that is interacting, so that neither have to be tied to each other.

  It used to be the case that when you wrote, e.g. an Eclipse plugin for your language, it just worked in Eclipse and if you wanted to provide the same extension for Netbeans, Emacs, Vim, etc you would have to rewrite most of the plugin. LSP lets you sidestep that.

  As an example in the Rust community we have a language service which works for Emacs, Vim, or any editor or IDE which supports the LSP.

# The "responsive" compiler (niko2019respcomp)

# Demand driven (niko2019respcomp)

# Why should you care about IDEs? (niko2019respcomp)

# Dependencies matter (niko2019respcomp)

# Strict phase separation is impossible anyway (niko2019respcomp)

# Not a solved problem (niko2019respcomp)

<!-- SALSA ------------------------------------------------------------------->

# Salsa (niko2019respcomp)

# Salsa core idea (niko2019respcomp)

# Entity component System (niko2019respcomp)

# Entities in a compiler (niko2019respcomp)

# Components in a compiler (niko2019respcomp)

# Salsa queries (niko2019respcomp)

# Example queries (niko2019respcomp)

# Query group (niko2019respcomp)

# Input queries (niko2019respcomp)

# Derived Queries (niko2019respcomp)

# How salsa works (niko2019respcomp)

# Recomputation (simplified) (niko2019respcomp)

# Recomputation (niko2019respcomp)

# But suppose input change is not important (niko2019respcomp)

# Recomputation (less simplified) (niko2019respcomp)

<!-- SUBTLETIES -------------------------------------------------------------->

# Order matters (niko2019respcomp)

# Minimizing redundant checks (niko2019respcomp)

# Garbage collection (niko2019respcomp)

# General idea (niko2019respcomp)

<!-- LAYERING ---------------------------------------------------------------->

# Layering (niko2019respcomp)

# Represent layers with maps (niko2019respcomp)

# Rust compiler of yore (niko2019respcomp)

# Trees are your friends (niko2019respcomp)

# Interning (niko2019respcomp)

# Tree-based entities also give context (niko2019respcomp)

# Signature (niko2019respcomp)

# Tightening queries with projection (niko2019respcomp)

# The "outer spine" (niko2019respcomp)

<!-- ERRORS ------------------------------------------------------------------>

# Error handling (niko2019respcomp)

# Recovery from day one (niko2019respcomp)

# Example: error type (niko2019respcomp)

# Diminishing returns (niko2019respcomp)

<!-- CYCLES ------------------------------------------------------------------>

# Handling cycles (niko2019respcomp)

# Example: inlining (take 1) (niko2019respcomp)

# Example: inlining (take 2) (niko2019respcomp)

# Non-deterministic results (niko2019respcomp)

# One better approach (niko2019respcomp)

# Other cases involving cycles (niko2019respcomp)

<!-- PARSING ----------------------------------------------------------------->

# Tracking location information ("spans") (niko2019respcomp)

# What to put in your AST (niko2019respcomp)

# Incremental re-parsing (niko2019respcomp)

# Example: swift red and black trees (niko2019respcomp)

# Alternative: "zooming out" or "zooming in" (niko2019respcomp)

<!-- THREADING --------------------------------------------------------------->

# Threading (niko2019respcomp)

# Salsa's threading model (niko2019respcomp)

# Cancellation (niko2019respcomp)

# Conclusion (niko2019respcomp)
