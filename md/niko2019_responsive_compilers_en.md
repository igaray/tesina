<!--
\addcontentsline{toc}{section}{Responsive Compilers}
\section*{Responsive compilers - Nicholas Matsakis - PLISS 2019}
\cite{niko2019responsive}

@online{niko2019responsive,
    author = "Nicholas Matsakis",
    title = "Youtube: Responsive compilers - Nicholas Matsakis - PLISS 2019",
    url = "https://www.youtube.com/watch?v=N6b44kMS6OM",
    year = {2019},
    month = {06},
    keywords = "rustc,incremental",
}
 -->

Many compiler textbooks and courses treat compilation as a "batch process", where the compiler takes in a bunch of input files, executes a suite of comiler passes, and ultimately produces object code as output.
Increasingly, though, users expect integration with IDEs like VSCode, which requires a different structure.
Moreover, many languages have recursive constructs where the correct processing order is difficult to determine statically.
Nicholas will discuss some of the work the Rust team has been doing on restructuring the compiler to support incremental compilation and IDE integration.

- [Slides](https://nikomatsakis.github.io/pliss-2019/responsive-compilers.html)

<!-- WHY --------------------------------------------------------------------->

# Pipelines and Passes

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

# The "responsive" compiler

  - Compiler as an actor
  - Editor sends diffs and requests completions, diagnostics
  - Compiler responds

  In this model instead of having the compiler be something that one runs by the command line it's more of an actor, the editor/IDE sends it messages with diffs of the files the user is compiling, and the compiler responds to them and sends back diagnostics.
  For example, the user might want to know the completions available at a certain point in the source code and the compiler responds with a vector of possible completions.

  The key point is that the compiler needs to be able to respond as quickly as possible.

  Thus one has to consider what the minimum information needed to answer those user requests, whether the compiler can process just that to respond as fasta as possible.

  Thus, instead of a type checker that walks all the sources and performs all steps for all functions, one could request the type checking of just one  statement and compute how much context is needed to do that.

  In this way a rather different structure for the compiler results.

# Demand driven

  What the rustc compiler team has been trying to do is move to a more demand driven architecture.

  - Start from goal
  - Figure out what is needed for that

  A given goal is implemented by some function that requires some subgoals, implemented by other functions, and works backwards, trying to keep this set to a minimum.

  After this reorganization one still has these traditional compiler passes, but they might not be executed completely nor in a fixed order.

# Why should you care about IDEs?

  There are some reasons for structuring a compiler using an IDE-friendly approach from the beginning which are not immediately obvious.

  - One is forced to write code in the language being implemented
  - The process informs the language design, as one becomes more aware of dependencies needed to compute information, which in turn might lead to making certain decisions or not.
  - Strict phase separation is impossible anyway

# Dependencies matter

  An example of something that would not have been allowed had the rustc compiler implemented support for IDEs earlier on is allowing arbitrary nesting of declarations inside functions, e.g. a function with a struct inside of it.
  This language feature is rather useful, as sometimes one wants some local data that's not needed outside the function.

  ```rust
  fn foo() {
    // Equivalent to a struct declared at the root of the
    // file, but only visible inside this function.
    struct Bar{}
    let x = Bar{};
  }
  ```

  One can also put methods on the struct:

  ```rust
  fn foo() {
    struct Bar{}
    impl Bar {
      pub fn method() { ... }
    }
    let x = Bar{};
  }
  ```

  However a side effect of this is that auto-completion requires looking inside many function bodies: a struct could be visible from outside the function, and have methods defined on it inside the function, and since methods are dispatched based on type, those methods can be called from outside the function, ultimately requiring the parsing of all function bodies in scope and computing whether there is a relevant `impl` block for completion to provide the full set of methods available for that struct type.

  ```rust
  struct Bar {}
  fn some_method() {
    let bar = Bar::new();
    bar. // <-- what methods should we offer as
         // auto-completion here?
  }
  ```

# Strict phase separation is impossible anyway

  Another reason that led the compiler team in this direction is that in most compilers, if there is a strict phase separation in which the compiler first resolves all symbols, then type checks all bodies, and so on, undesireable constraints appear and the source code must be processed in a difficult order.

  Rust, for example, allows this:

  ```rust
  const LEN: u8 = 1 + 1 + 1;
  const DATA: [u8; LEN] = [1, 1, 1]:
  ```

  This defines an interdependency: in order to know the full type of `DATA`, the `LEN` constant must be evaluated, and the expression defining it must be type checked and evaluated in some way to compute its value. This implies that constants cannot be type checked in a single order without considering the dependencies between them.

  `rustc`'s original approach was a horrible hack, involving essentially two implementions of some parts of the compiler, because a subset of the typechecker and evaluator that had enough features to evaluate symbols such as `LEN` and execute any part on demand was needed, and then the code that performed the full type checks came after.

  This duplication was a horrible pain, and in this more demand-based system this is not a problem because it can evaluate `LEN` on its own.

  Another example:

  ```rust
  const LEN: u8 = DATA[0] + DATA[1] + DATA[2];
  const DATA: [u8; LEN] = [1,1,1];
  ```

  In this example, `LEN` depends on `DATA` and vice versa.
  The compiler needs to detect cycles and this falls out of the framework involving phase separation.

  Other examples of features complicating phase separation:
  - Inferred types across function boundaries in languages such as ML.
  - In Java and its lazy class file loading, the dot operator performs many functions in a lazy way, such that the set of classes a given class can touch is determined as you walk the file being compiled.
  - The way Racket deals with phase separation and scheme macros; in many languages with metaprogramming facilities one finds one wants to evaluate some subset of the source and type check and be able to work with them without necesarily processing the whole source.
  - In rust, there are several things in the logic language related to traits, such as specialization, which requires solving some traits.
    - A trait tells the Rust compiler about functionality a particular type has and can share with other types. We can use traits to define shared behavior in an abstract way. We can use trait bounds to specify that a generic can be any type that has certain behavior.
    Note: Traits are similar to a feature often called interfaces in other languages, although with some differences.
    Trait resolution is the process of pairing up an impl with each reference to a trait.
    Trait specialization is ...
    TODO: ask Niko to clarify.

# Not a solved problem

  There are several approaches that can be taken to solve this situation:

  - Hand coded
  - Salsa
  - More formal techniques
    - Attribute grammars
    - Datalog or structured queries

  The hand coded approach has been used in practice in many languages and involves simply not having a framework but thinking very carefully about things, doing the same things as in the other approaches but in an open coded manner, such as figuring out if a symbol requires type checking, the dependencies needed, etc.
  This approach is of course very practical, but can be prone to bugs, incremental inconsistencies, etc if for example a dependency is omitted while computing some information.

  Another technique is using a more formal, higher level expression system such as attribute grammars or datalog queries.
  In this case one has to ensure that everything done by the compiler fits in this framework or extend the framework to amek it fit.

  rust-analyzer takes an approach based on a framework called salsa, which  enables you to still write your compiler in a general purpose programming language that feels familiar, in a kind of middle ground between the hand written and formal approaches.

<!-- SALSA ------------------------------------------------------------------->

# Salsa

  - high level idea:
    - inputs
    - derived queries

  The high level idea of this framework is that the compilation is separated into the inputs to the compilation process and a set of computations derived from the inputs.
  The derived queries are pure functions that can demand needed results from other queries, some of which may be inputs.
  When one of these functions is used, the data and any input functions it uses are tracked, and when a change to one of the inputs occurs it can be propagated through the dependencies in order to avoid re-executing some of the computation.

  There are a lot of systems in this space, for examples:
  - adapton
  - glimmer from the ember web framework
  - build systems a la carte

  Two of these are academic.
  Adapton is an approach by Mathew Hammer.
  Build Systems a la Carte is a paper by Simon Peyton Jones that explores the design space of incremental build systems and implements a flexible system in Haskell that has a similar basis to Salsa.
  The main difference between Salsa and the system described in Build Systems a la Carte is that the latter allows one to customize many aspects.
  Adapton as well is more flexible but also more complicated.

  Finally, an interesting approach is the Glimmer negine in the Ember web framework, which also does incremental updates.
  Many web frameworks based on virtual DOMs and computing deltas to minimize changes to the actual DOM such as React and Elm are also related.

  This problem applies to many areas, not just compilation, and the design is affected by the needs and constraints of each application area.

# Salsa core idea

  When writing a program in Salsa, it takes this approximate form consisting of a loop that sets some inputs and computes some derived values.
  The inputs are for example the diffs the program receives from the editor.
  The idea is that derived values are memoized, and inputs updated on change, such that whenever a derived values is queried, it will always be up to date.

  \begin{minted}{rust}
  let mut db = Database::new();
  loop {
    db.set_input_1( ... );
    db.set_input_2( ... );

    db.derived_value_1();
    db.derived_value_2();
  }
  \end{minted}

# Entity component System

  In writing a whole compiler in this model, one of the things that arises is this relationship to a design pattern from game programming known as Entity Component Systems (ECS), which is presented as an alternative to Object-Oriented Programming.
  In ECS, data and identity are separated.
  In OOP, classes are defined and instantiated in such a way that the data and operations are associated with this class identity at the moment it is created, whereas in ECCS you can create a new identity with nothing associated except its identity, and then the data is attached to it separately from creation.
  In games this is useful due to the dynamic nature of the data a game handles during runtime, allowing the data to be unstructured and easier to handle.

  - entity: unit of entity
  - component: data about an entity

  \begin{minted}{rust}
  fn move_left(entity: Entity, amount: usize) {
    let pos = DB.position(entity);
    pos.x -= amount;
    DB.set_position(entity, pos);
  }
  \end{minted}

# Entities in a compiler

  In the context of compilers, this lack of structure is not as important, although useful.
  The result of structuring data in a compiler in this fashion is a system in which entities correspond to things declared in the program language, on top of which different elements of data are layered, e.g. a symbol might have a type, a name, etc.

  The reason it is useful to have the data layered on in this way is that it allows the system to be demand driven, by decoupling when an entity is created and when its associated data is computed: the compiler can be queried regarding the type of a symbol and can answer without requiring all the other data that might eventually become associated with it.

  - often called symbols
  - things like
    - input files
    - struct declarations
    - fields
    - function declarattions
    - parameters or local variables
  - something "addressable" by other parts of the system

# Components in a compiler

  In the context of a compiler, there are a number of elements that might be represented as components, such as type, signature, errors, or the result of applying some analysis.

  things like
  - the type of an entity
  - the signature of a function

# Salsa queries

  Salsa's system is not quite an ECS in that it doesn't have a formal concept of entities.
  Instead, the basic structure of a system written with salsa consists of named queries which are similar to components.
  A query name might be something like 'type' or 'signature'.

  In addition, there is a set of keys that go into the query, often just a single key.

  When this query is executed it returns some result value.
  The keys and values have to be plain values in the sense that they can be copied in memory and compared for equality.

  Q(K0 .. Kn) -> V
  - Q is the query name (like AST)
  - the K0 .. Kn are the query keys
    - atomic values of any type
  - The V is the value associated with the query

# Example queries

  Some examples of the kinds of queries that might be present:
  - Base inputs, such as the input text / the source in a file.
  - Derived values such as the AST or signature of a function.
  - High level operations, such as a query returning the line and column position of the cursor in a file.
  - The set of strings to display to the user as possible completions at the cursor's position.

  ```
  input_text(FileName)
  ast(FileName) -> Ast
  signature(Entity) -> Signature
  completions(FileName, Line, Column) -> Vec<String>
  ```

# Query group

  The program is thus structured into groups of queries, which provide an interface.
  A database is declared which will provide the range of possible queries, some of which will be explciitly set as inputs and the rest as derived queries.

  For example, the AST associated with a given file could be returned by a query names 'ast' which will take the filename and return the AST for the code contained in said file. This query will use the input query 'input_text', which will provide the contents of the file and track changes to the contents.

  Note that queries may take different types of parameters, e.g. the signature of some entity in the system instead of a filename.

  ```rust
  #[salsa::query_group]
  trait CompilerDatabase {

    #[salsa::input]
    fn input_text(&self, filename: FileName) -> String;

    fn ast(&self, filename: FileName) -> Ast;

    fn signature(&self, entity: Entity) -> FnSignature;
  }
  ```

  Why are inputs separate from derived queries?
  As far as the interface is concerned, they both appear as functions that can be invoked.
  The actual implementation of Salsa uses a procedural macro to generate glue and memoization code, and for inputs different code is generated than for derived queries, as well as generating a 'set' function which will inform of changes and invoke invalidation.

# Input queries

  Input queries are essentially a field.
  The `#[salsa::input]` annotation generates accessors.
  Input queries essentially wrap a hashmap that stores the base data.
  The framework will automaticall generate functions that can be used to get and set the value.

  ```rust
  //  #[salsa::input]
  //  fn input_text(&self, filename: FileName) -> String;
  let text = db.input_text(filename);
  db.set_input_text(filename, text);
  ```

# Derived Queries

  The derived queries are a little bit different.

  ```rust
  // fn ast(&self, filename: FileName) -> Ast;
  fn ast(
    db: &impl CompilerDatabase,
    filename: FileName,
  ) -> Ast {
    let input_text = db.input_text(filename);
    ... /* implement parser here */ ...
  }
  ```

  Derived queries are defined as functions which given keys, return results.
  The first 'db' parameter is the database and provides access to other queries.
  This parameter is some unknown generic type which is constrained to implement the `CompilerDatabase` trait, such that the function only  has access to and can work with the methods exposed in that interface.
  The other arguments are the inputs the query keeps.

# Rustc History

  One subtle but important issue is that in rust, a top level function such as this one has access to nothing else.
  Global mutable data can be defined and used if truly necessary but is difficult to work with.

  This is relevant because the rustc compiler team went through three iterations of this incremental system which differed among other things in in how access to data was handled.

  The first attempty failed early.

  The second was implemented but did not work well, as it was not as strict in regard to access to global data.
  The initial difficulty of ensuring proper access to data was underestimated, leading to difficult to debug errors.
  Situations often arose caused by subtle leaks of information involving data that was thought to be usable but was in fact influenced by earlier phases of the compilation.
  This version was never user-facing.

  The system was rewritten a third time taking a much stricter approach such that feature implementations did not have access to anything that was not tracked.

  TODO: ask Niko about elaborating on these earlier version, and what he meant with "that is kind of like the equivalent of putting a type system onto your language"

# How salsa works

  Within a given revision, if the inputs or dependency queries do not change, the query system can be viewed as a memoization layer.
  When one of the database query methods is invoked to compute some value it is memoized, and on subsequent invocations the query checks whether the result has already been computed and if so returns the stored value, otherwise it executes the function.
  This process involves checking for dependency cycles.

  This is what occurs within one revision, but the query system also tracks when a query invokes another to establish dependencies across revisions, such that if a memoized result changes the queries whose memoized results need to be invalidated can be determined.

  The database contains:
  - a global revision counter
  - one map per query (e.g. ast)
  - this maps query key to:
    - Memoized result
    - Vector of dependencies ([input_text("foo.rs")])
    - Revision when last changed

  ```rust
  db.ast("foo.rs")
  ```

  The data used to do is a global revision counter and a hashmap for every query which maps the key to the query result value, a vector of dependencies and a revision that tracks in which version the value last changed.
  During the initial computation this revision is the current one, but as the query system runs it will not always be equal.

# Recomputation (simplified)

  When `db.ast("foo.rs")` is invoked:

  - If there is no entry yet, execute query and store result.
  - Otherwise, if any input dependency is out of date:
    - Re-execute the `ast` function, recording the new result and its dependencies
    - Update the "last changed" revision

  A simplified description of recomputation.

  When a query is invoked and no entry is present in the hashmap, it is executd and its result stored.
  Otherwise, the dependencies are walked and checked for changes, including their dependencies.
  If any have changed since the last revision, they are reexecuted.

  This is similar to what make does for its targets.
  Make as well establishes a dependency graph and determines whether the timestamp for any of the target files is newer.
  If so, it eagerly invalidates everything reachable from that target and re executes the commands, or in the case of the compiler, the queries.

  This is the approach taken in the first version that was never released, since it does present some problems.

  Example:

    Assume a query called signature, and that the current revision is R1.

    db.signature(..)
    Current revision: R1

    To compute the signature the AST is required, so `db.ast(...)` is invoked.
    To obtain the AST the input containing the function signature needs to be read and parsed, so `db.input_text(...)` is invoked.
    At this point the following call stack has been built and the input text is reached:

      db.signature(..)  -- changed in R1
      db.ast(..)        -- changed in R1
      db.input_text(..) -- changed in R1

    The system knows the revision number, since it is the previously set one, and this revision value is propagated up.

    Now assume a change is made and there is a new revision when the signature is once again requested.

      Current revision: R2

    The signature query once again checks its dependency on the ast query, which checks its dependency the input_text query, and since the input_text revision number is now R2, it is known that the ast is out of date since its revision number is older than its dependency, and must be recomputed.

      db.signature(..)  -- changed in R1
      db.ast(..)        -- changed in R1 ← out of date
      db.input_text(..) -- changed in R2

<!--
  QUESTION:
  the input.txt is effectively like the source in JavaScript
  you're saying like of an HTML Dom?
  yeah it's kind of like that
  you could think of it like that in this system I think
  the difference would be
  mmm difference might be this
  that the input text would be for the entire file usually
  and some of the other things you're computing might not be on the whole file
  so you might be getting the signature of some function
  but in order to get the signature of a particular function you would have to get the ast of a particular function
  in order to get the ast of a particular function you have to find the input text for the whole file that it's in
  which kind of you can parse
  well maybe you can parse parts of the file but usually you would parse the whole file and then you can extract out just the ast part of the ast that you need and propagate that back out
  and so the point being that you don't actually have the the source tokens
  at least not trivially for a particular function
  you have more of the source tokens for the whole file
  but you can usually you'll track position information so that if you needed them
  you could subset

  QUESTION

  no it returns the whole file is what I'm saying but you might later extract
  out a slice or something

 -->

# But suppose input change is not important

  A frequent case in this system is of course that the actual change in a dependency doesn't truly affect the value to be recomputed.
  A simple example is adding a comment to a function body.

  Before:

  ```rust
  // foo.rs
  fn foo() {
    do_something();
  }
  ```

  After:

  ```rust
  // foo.rs
  fn foo() {
    do_something(); // FIXME
  }
  ```

  The actual resulting AST is going to be the same, whether or not there is a comment, so recomputing it is not optimal.

# Recomputation (less simplified)

  The algorithm used has an additional detail: when a query function is executed, the new result resulting from re-execution is then compared to the previous one to verify it is in fact different.
  If it is the value is updated but otherwise the recomputation stops.

  Example:

  When `db.ast("foo.rs")` is invoked:

  - If no entry yet, execute query and store result.
  - If any input dependency is out of date:
  - Re-execute ast function, recording new result + dependencies
  - If the new result is different from old result:
    - Update the "last changed" revision

  This is important since when applying this, the system verifies that the AST is the same, and the revision in which it changed will not be updated.
  Thus, the AST is still considered to have changed in the first revision and its signature is not dirty and can be reused.
  In the case of the AST the impact is not that important but there are likely later computations such as type checks, etc that have a larger impact.
  The convenience lies in that a projection can be made in which the parts that are actually needed can be extracted and used to contrain changes from propagating too far.

<!-- SUBTLETIES -------------------------------------------------------------->

# Order matters

  A subtle point about this basic algorithm is that the order of computations has a great impact.
  When the compiler verifies that a datum is out of date, it has to be verified in the same order in which it was executed in the first place, or code that should never be executed might be triggered.

  Example:
  Given a function `A` that invokes another function `B` and then either `C` or `D` depending on the result.

  ```rust
  fn a(db: &impl Database) {
    if db.b() {
      db.c();
    } else {
      db.d();
    }
  }
  ```

  - The input dependencies of A are B, C, and D.
  - If B were true in R1, we execute D
  - In R2, if B is false, C should never be executed

  If B is true, then the list of dependencies in the first revision would be determined by the fact that A invoked B, and then invoked C because B was true.
  In the second revision, it is not desireable to check if C is up-to-date before checking B if B has changed, since it might be the case that C should never have been invoked in the first place.
  If B has already changed it must be re-executed since execution could go through some other path.

# Minimizing redundant checks

<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< MARK >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< MARK >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< MARK >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

  A second subtle point is that not only must the system track when a value last changed but also when
  
  the second part is we also track,
  we don't just check when did it last changed
  but we track
    when did we last check if it is
    when did we last update and check this value

  that basically ensures that we never recompute something more than once in a given revision
  so you know that at any point it's kind of linear over the set of things

  - For each memoized value, track:
    - "Last changed" revision
    - "Last checked" revision
  - Update "last checked" revision when value is updated
  - Ensures that we only execute a value once per revision


# Garbage collection

  - Memoized results from previous revisions may no longer be relevant
  - But GC can be quite efficient:
    - Execute "master query"
    - Sweep any value whose "last checked" revision was not updated
  - Key idea:
    - The master query doubles as the mark

  finally, you do have to worry about garbage collection because if you think back to this ABC example like the first round the function A wound up invoking the function C and we memorize that but if in some later execution that function may never get invoked and we still have this memorized value kind of hanging around that we might want to (because we're thinking maybe we'll want to reuse it later) so that that requires you to collect these old results at some point

  and it turns out you can do this in a kind of nice way
  because we're already tracking for all the memorized values the revision when we last computed them (when they were last checked if they were up-to-date or not)

  and so essentially what you can do is: (let's say your master query whatever it is like type check all the functions) and then at the end of that you can just sweep through the memoized  values and say "did that wind up being recomputed in the most recent revision or not" and if it didn't then you know it's something that's no longer needed or at least was not needed
  to do that master query and you can throw it away

  and the nice part of this of course is these are all you know fully functional pure things that were derived so at the end of the day if you throw something away that you might want later it doesn't really matter because you can always recompute it

  in fact we've been finding more and more that you know computation is cheap sometimes it's better to just throw away everything even if you know you're going to need it later because you might as well recompute it

  so that's an interesting point of like let's say a place where we have been playing around with what the right strategy is

# General idea

  A --> C --> E --> F --> G
              ^
              |
  B --> D ----+---> H

  Re-execute the "early steps"
  But cut off as quickly as you can

  when you're done you kind of get this picture where you have a graph of computation and what you really want to do is reexecute the early steps but cut it off as quickly as you can

<!-- LAYERING ---------------------------------------------------------------->

# Layering

  Common pattern:
      produce a base structure
      other queries "layer" structure on top
  Example:
      parser produces AST
      name resolution resolves names to entities
      type check adds types

  when you're doing this approach one of the things you have to do is separate out
  you know you don't want to have all the thing all the different bits of data that you're going to compute all packaged together into one data structure

  so I mentioned entity component systems and so on earlier, that's really relevant here because
  let's say you have a parser that produces an ast
  and a lot of compilers you might have like a class for the ast node and in that ast node it would have "oh when I named resolve this what did I resolve it to",  "what is the type of this ast node", sort of all stored as fields in the ast itself

  but if you do that in this incremental system that won't work so well because when we reparse the ast we of course don't have those values anymore and you you'd have to sort of port them you just can't combine it

  basically you can't reuse mix and match bits of data from different revisions that way,

# Represent layers with maps

  - Give each node in the AST a numeric id AstId
  - Name resolution produces Map<AstId, Entity>
  - Type-check produces Map<AstId, Type>

  so what we do instead is to sort of separate out the layer and you wind up with essentially a lot of maps that's what it comes down to
  so you could imagine for example that if you have the ast for a given function, you can give each node in the ast an ID

  map just an integer
  and then you can produce (for name resolution) you can have a map that says for the node with this ID here's what I resolved it - here's the symbol or entity
  and for type checking you might have a similar map it says here's the type for that AST alright

  and this works pretty well, it's reasonably efficient

# Rust compiler of yore

  - one big ast
  - nodes in the assigned a pre-index (NodeId)
  - this ID was used everywhere

  but it turns out that the way that you give these IDs actually matters a lot - and that was something (it's like a trick that keeps coming up)
  and in the old rust compiler before we made it incremental also used a lot of maps
  that's because it was written in a very functional style so it didn't want to be mutating things but it assigned the node IDs in a very simple way
  it just did a walk of the entire ast for your whole program and gave them numbers


  fn foo() { // node 0
    let x = 3; // node 1
  }
  fn bar() { // node 2
    let y = 4; // node 3
  }

  pre index of this walk so 0 1 2 3
  this had a sort of downside
  which is it was very simple but if I modify it
  let's say the function foo
  and add some more stuff into it
  then all the IDs for the function bar are going to be different after that
  so that there's this contamination
  and that obviously won't work with an incremental system
  or at least if you edit things early in the file you'll have to do a lot of work
  recomputing stuff later on in the file and that's probably not what you wanted

# Trees are your friends

  Entity = FileName
         | Entity "." Name [ Index ]

  so the the basic trick here is to use trees and this is one of those tricks where I feel like as we do the design we just keep coming across this this being a useful technique

  so what you do is instead of giving instead of your ID being just a simple integer it's some kind of path right

  Entity = FileName
         | Entity "." Index

  fn foo() { // node "foo.rs".0
    let x = 3; // node "foo.rs".0.0
  }
  fn bar() { // node "foo.rs".1
    let y = 4; // node "foo.rs".1.0
  }

  and this path it can be just in it it can just be indexes or it can be something richer with names it doesn't matter that much
  but so this would be the simplest simple scheme
  you might start it you might say the first step is a file name
  and then every that that's the base kind of entity that can be in your system is a whole file
  but then within that you can sort of nest right
  so the function foo might be like represented as dot 0
  and dot 1 would be the function bar and
  then within there we have further numbers right
  and now of course we have the advantage that changing the contents
  of foo doesn't affect the numbers of bar in any way

  and what we actually do in the compiler is like a little bit different
  so the main downside of this of this index scheme is if I add a new
  function like if I put a function in between foo and bar now the the index of
  bar has changed and so we'd have to recompute things about bar
  and maybe that's a problem maybe it's not
  like I said it turns out you know the computer is pretty fast like that might not be
  actually that big a deal
  but if you wanted to avoid it you can use names

  fn foo() { // node "foo.rs".foo[0]
    let x = 3; // node "foo.rs".foo[0].expr[0]
  }
  fn bar() { // node "foo.rs".bar[0]
    let y = 4; // node "foo.rs".bar[0].expr[0]
  }

  as long as we insert a new function in between, as long as it has a different name it's ok

  the problem with names of course is then you have to deal with incorrect programs and or you might have two things with the same name which might or might not be correct but you have to deal with that possibility

  so we actually use in the compilers we have this extra index
  so we use names but we give them an index and then when the same name appears more than once we increment the index and that's usually actually an error but we still need to keep going so that we can give you feedback but it lets us sort of have a unique ID
  and sometimes it's not an error because there are certain things that are
  anonymous for example
  that works pretty well in practice

  fn foo() {} // node "foo.rs".foo[0]
  fn foo() {} // node "foo.rs".foo[1]

<<<<<<<<<<<<<<<<<<<<<<< HALF WAY MARK >>>>>>>>>>>>>>>>>>>>>>

# Interning

  struct Entity {
    value: u32
  }

  so you have these big trees and that that's great but you have to actually pass them
  around and so forth
  and if you had like a garbage collected language I guess that's not such a big deal you can sort of allocate them
  but you probably do wind up creating a lot of the same tree over and over
  so what we do is we have also the last piece of this system is a kind of interning mechanism
  it's basically there to turn these big trees into little integers and go back and forth
  and so you can reference this integer, you basically intern
  the way you'd represented it in rust is that the whole path is represented as an integer (a new typed integer so it has a struct that wraps it around it so that we can give it a meaningful type)

  and then this is the actual data which is recursive but it goes through the interning system right

  enum EntityData {
    Root(FileName),
    Child(Entity),
  }

  so the recursive step references the previously interned value
  and then we have a special interning mechanism that can convert the data into a new one

  #[salsa::query_group]
  trait CompilerData {
    #[salsa::intern]
    fn intern_entity(&self, data: EntityData) -> Entity;
  }

  and we can actually track those dependencies too
  and thus if for example some function is renamed or parts of the system are different we'll figure it out but

# Tree-based entities also give context

  enum EntityData {
    Root(FileName),
    Child(Entity),
  }

  the other advantage of this this entity based approach or this this tree based identifier our approach is that you get some context
  because it turns out you you often really need this kind of context

  so if you think about some of the examples I gave earlier
  when you're computing the signature of a given entity I sort of hand-wove and said something like "to compute the signature of a function we have to look at the ast of that function" but then I said that there's only one ast per file
  the question would be how do I get from this function to that file,
  how do I know what file the function is in right in the first place

  and there are different ways to do it but one way that works really well if you have this tree based approach is that it's it's right there in the identifier basically
  you can walk up the ID and find the root of it is some file and you can you can get the file from there

# Signature

  Example from earlier:

      db.signature(entity)
      db.ast(file_name)
      db.input_text(file_name)

  How do we get the file_name from the entity?

  so the the other big technique that comes up a lot is the ability to tighten your queries
  so far what we have is this the system that lets you write a bunch of queries, it tracks their dependencies
  the nice part about is it's guaranteed to sort of be correct
  it'll only recompute or it will always give you a refreshed value
  but it might actually do a lot of recomputation if your queries are very broad

# Tightening queries with projection

  db.signature(entity) -- signature of a function
  db.ast(file_name) -- ast of the entire file

  and this particular query setup that I showed here is an example where it might
  be too broad in practice you have to try it and see
  because that's what you're gonna find wind up doing here is recomputing the signature of the function whenever anything in the file changes
  because this ast is for the entire file
  and that's maybe that's okay or maybe it's not alright
  and what what you can do if you find that your recomputing something too often is you can insert a sort of intermediate query to do a projection or a narrowing or some sort of transformation all right
  so maybe I have a query that says give me the ast just for this one function and all that does is find the function and pull it out from the bigger ast
  but the advantage of this is that if the bigger ast changes all I have to re-execute is the code that finds and extracts the one smaller piece of AST and I won't have to do any of the dependent operations

  db.signature(entity)
  db.entity_ast(entity) -- extract AST of a single entity
  db.ast(file_name)

  so that that kind of is basically just a handy thing where you can do while you're looking, while you're optimizing

  QUESTION how many queries are there in the rust compiler?

  I don't know there's a lot probably not thousands it's probably more that's probably hundreds

  one of the things the rust compiler is actually using a slightly different version of this system
  so this is like an idealized version of the rust compiler that has been extracted to a library and we are actually using it in a separate effort to build an IDE like an IDE first compiler for rust
  that we have to figure out how to bridge the two

  that's another story but
  that is using this framework and I'm not sure how many queries they have
  but they're not complete yet
  the rust framework is using a slightly different one
  but in any case I think it's has on the order of hundreds

  but the reason I mentioned that we're not using it
  is one of the problems we had in the rust compiler is that we didn't support any
  kind of modules and the number of queries did indeed grow very large and it's just annoying if nothing else but also hard to read the source because they're all in like one huge list

  and so part of the reason that everything is by interfaces and so on is exactly so that you can modularize the queries and say here's the stuff for the parser and here's the type checker and they depend on one another and so on

  but it does it does grow quite large I would say

# The "outer spine"

  so one of the questions I think what when I've been working with this system it seems very clear when you look at any individual function how it's supposed to work and it's kind of clear when you think about at the outer level okay I'm gonna make a query like get me the completions

  but getting from give me the completions at this point to those intermediate queries that actually know what entities there are and can think about the type checking and all that stuff it's kind of challenging

  it's like a quantum mechanics thing or something it all makes sense at the two extremes but the middle is confusing

  so I thought it would be useful to just sort of walk over this this spine how it all connects
  this is an example how you might do it

  Question: How do we get "all the errors in the project"?

  db.all_errors()
  db.filenames() -- get a list of all filenames
  db.entities(filename) -- for each filename, get all entities
      db.parse(filename) -- parse file to AST
  db.type_check(entity) -- for each entity, do type-check
      db.ast(entity) -- get the AST for this entity
          db.parse(filename) -- parse file to AST (memoized)

  so if you were say computing what are all the errors in the project
  that's that's like that's kind of a query you would likely have for your IDE
  you might start by saying give me all the file names in the project right
  and that's probably just a base input
  and then you can kind of iterate over each file and have something that says "give me all the entities" which would go through which would have to parse the ast and then walk the ast and extract, just the IDS and return them to you
  all right and then you can walk through and type check all the entities
  and by this point once you're in this this round of like getting the ast for a given entity and so on and it becomes again fairly clear this is like your standard code
  because now you have the identify as you need for all the things
  but the I left out some steps here for sure like type checking would probably have to get the ast but it would also need to get the name resolution results and things like that but they fit into this framework

  all right and so having this now we can sort of walk through and see what happens in this complete picture

  - What if the user edits a comment?
  - Reparse, but that's it.

  - What if the user edits a fn body?
  - Reparse the file.
  - Extract the AST.
  - Type-check the function that changed.

  if the user edits a comment
  and the answer might be as simple as "well we've done one revision so we have all this memoized data"
  and if the user edits a comment we can ReWalk it and we see that we have to rerun the parser but once we rerun the parser the ast that results is the same so we just stop
  and we can keep all the type checks intact
  but if the user edits a single function body then we would have to rerun the parser they would not be the same because they did actually edit a function body so the ast is different, so in that case we would have to as we're doing the type checking for each we would extract out one by one what is the AST for any given function and we'll find that only one of them has changed
  so we'll wind up we running the type checker but just for that one function
  and so we wind up we doing a pretty reasonably minimal amount of work overall

  there was a question of how many queries there are in the compiler and I mentioned memory use, so one of the things I would add is that in practice you may not actually want to keep all of this memoized data around for all of your queries because it can be quite a large graph
  but there's a lot of different tricks you can do
  one of the things we do and the compiler is we only keep the hash actually we don't keep the full value so we can still re-execute and we can still see if it has changed because it's a cryptographic hash so it's at least as good as sha-1 or whatever

  we know if it's changed but if we have to recompute it then we'll just redo the work because it's not worth it
  we only keep the values we sort of selectively keep what what data to keep and what not to keep and

  that kind of tuning is I think it's sort of annoying that it's necessary but if you have this framework it's or have a framework it's nice that you can you can do it relatively easily

<!-- ERRORS ------------------------------------------------------------------>

# Error handling

  now I want to talk about a few other sort of things that happen in practice one of them is error handling

  I mentioned somewhere along the line that we if you have errors you you know you might like to
  I'll give you have two functions with the same name we want to have to handle that case and I think that in general especially if you're in an IDE context you really want to be handling, you really don't want to stop compilation basically ever

  How not to handle an error in a compiler:

  throw new TypeError();

  a lot of early compilers I think will basically handle an error by just saying well okay something's wrong I give up I'm done and that's that's a reasonably easy approach, it's probably okay for for many projects

  but it's actually but it's it's really difficult to sort of recover once you've
  baked this strategy into your compiler it's much hard to get it out again

  and the rustc compiler had this strategy for sure for awhile and we've been slowly
  getting rid of it and it's been very difficult

  the next thing that I think you people often try to do which the Rust compiler also tries to do is to say well there's an error I'll report the error and then I'm gonna give back some value that's like reasonable, it has the right type, but for the compilers type but it's not the right value
  so it might be like I need to compute the type of this expression, this expression is bogus I'll just give back the type integer and say "yeah good enough"

  Another way not to handle errors:

  if (some kind of error) {
    return fake_but_otherwise_legal_value;
  }

  maybe they'll get some other errors down the line but the user can figure it out and that kind of works sometimes but it does lead to some really confusing errors to a user where suddenly the compiler is talking about the type integer and you have no idea why

# Recovery from day one

  and what you can do instead that's actually not a lot harder and much nicer

  so a better idea is to introduce some kind of Sentinel values that say "this is bogus" basically, this is an erroneous expression a bad parse whatever and then you can just return this value

  - Create a sentinel value that means "bad user code here"
  - Invariant:
      - If you see this sentinel value, errors have been reported
      - So you can feel free to suppress downstream errors
  - No such thing as a "fallible" compiler operation

  it's not that much more work because as long as you just have them there then it's like almost as easy as throwing to just return this sentinel value

  they're usually very easy to propagate because you know by that if you ever see that sentinel value and you know that an error has been reported so you can just kind of short-circuit all the other computation and just keep propagating bigger and bigger sentinels up the line
  and so you wind up with a notion where basically in compilers there is no such thing as a fallible operation
  it always succeeds but it might produce an error value

  this invariant is pretty important though
  I've seen subtle bugs introduced where people return the sentinel value but they haven't actually reported an error
  usually because they think they know that an error will be reported by some other phase in the compiler but then because they are (they're not wrong about that but they're wrong about the ordering) and the phase actually comes later and the phase winds up being skipped because we saw this error value and we thought that there already was an error so we don't want to report duplicate errors
  so you really want to keep this invariant very clear that you know you actually reported the error there then you can produce a sentinel value or you got one from somebody else and then it will work much better

# Example: error type

  so this might be an example of you know just basically whenever you make something that represents a type or whatever just include an error in there

  enum Type {
    Integer,
    Character,
    Error,
  }

  so now you can have integer character or error and propagated along

# Diminishing returns

  and that the thing that the main problem here is you get to this point of sort of diminishing returns of how much precision
  if you really want to take this notion of an error sentinel all the way I mean the goal is basically to never give users an error
  if you saw an error earlier on, you never want to put a second error related to that code because you really don't know what's happening
  however that can be difficult

  so if we have like this structure that stores the signature of a method it says here's the argument types

  struct MethodSignature {
    argument_types: Vec<Type>,
    return_type: Type,
  }

  Hidden assumption here?

  there's a list of types and a return type
  there's a hidden assumption here for example that we know the arity
  we know how many arguments there are in the function
  but maybe we couldn't really parse the function signature there was a missing comma or something and we're not really sure how many arguments the user even meant for there to be
  I don't know I've tried some in different versions of the compiler I've tried to be very propagate this all the way out and at some point it stops being worth the trouble so I would say in a case like this I would probably just say well you pick a best guess and if they get an error like I expected three arguments and you gave me two you know oh well
  so it's not exactly easy but I think if you know the right place to cut it off it works pretty well so yeah

  QUESTION

  that's a legacy of the older Rustc, I'm kind of blurring the lines as I said the this idea of never stopping is basically me saying don't do what we did in rusty

  when we've been reimplemented us see and we've not and we've used the approach I'm describing here it's been much much easier

  but what we do in rustc is basically that it's some hybrid of "we don't throw the error immediately" (we used to do that too) but we stopped doing that usually
  but we do have big phases where we sort of say "if there have been any errors thus far there's no point in going further because the data is so corrupt"
  that is something we're trying to remove
  but it is difficult because the assumptions are
  you don't realize what where you've implicitly made a dependency and you have to kind of sort that out

  QUESTION 58:30

  the goal is to have no point like that
  what instead would happen is that let's say that you have typing errors within a function you might not get borrow checkers for that function
  and maybe you will actually depends right on how we do it but I think the first phase would be to make it much more granular and then proceed from there
  I would actually like I mean I think we should be able to give you borrow checkers even in the face of type errors as long as we can sort of make some sense you know other parts of the function make sense so we can type those
  I don't see any reason that would be hard if or if we just do it basically

  I did want to mention one interesting thing so this actually is something we don't yet handle in salsa but that is related to that which is
  I've been describing to you this whole thing of out of order execution and so on
  but one of the problems comes up how do you actually report an error to the user in the first place like what what is an error kind of in this framework and I think what I would like to do is have some sort of side-effect channel for these queries
  so right now what we've done in the compiler is we built on this is that they return basically instead of saying they basically embed errors into the values they return
  so you'll get back a type the types of all the things in your function and in there is a list of errors and some other part of the framework then has to go through and sort of sweep over all the possible places errors might occur and collect them into one big vector and send them to the IDE
  that's error prone because you can overlook an error from from one phase or another
  you might not remember that you have to check not only the type checker but also the parser can produce errors and so on
  and so I'd like to have some sort of mechanism where they can
  it's also just so it's error-prone but it's annoying because mostly in your compiler you just want to ignore errors right
  the whole point of this whole Sentinel approach is that you can just treat code like it's well typed and and sort of ignore errors for the most part unless you happen to see a sentinel value and
  this winds up making them more visible
  so what I would like to do is have some way where you can say I signal an error and it gets accumulated and well basically handle all that for you in the framework itself
  but we haven't implemented it yet

  in rustc what we actually do is we just dump them to standard error
  that's good because you get very fast feedback
  it's bad because when you re- incrementally re- execute you want to see the errors
  that's kind of a tricky part
  you want to see the errors again, even if we didn't have to rerun the type checker you'd like to still see the errors that resulted from the first run
  so what we do is we buffer them up and we save them and we have a whole bunch of mechanisms here
  so basically if the flow of errors is itself something I glossed over but it's a kind of a pain

<!-- CYCLES ------------------------------------------------------------------>

# Handling cycles

  Salsa can't handle cycles:
      currently panics, extending to permit controlled errors
  If not careful, permitting cycles is an easy way to induce order dependence

  cycles is an interesting case
  I mentioned that way back when in the beginning that if you have constants in rust they're not allowed to have a cyclic relationship for example that's a simplifying rule for us
  you can imagine that it could work in some cases
  but in general this framework basically can't handle cycles
  if you ask to compute a given value and it winds up needing to compute itself that's a problem

  I mean of course it would be a problem in a regular program also it would recurse infinitely

  what we currently do is we we take a pretty hard line in the framework where we actually basically panic which means that in rust that's a exception but it can't be caught

  it's actually not a good enough answer because you don't want your compiler to ever panic and sometimes these cycles are kind of out of your control right it
  the user created the cycle essentially
  so we're extending it to allow you to permit controlled errors but it's still a pretty harsh mechanism
  essentially what happens is if you get a cycle we will we will force you to propagate back a return value that doesn't depend on the inputs it's just like a cyclic error value that you can intercept it at some point

  but and the reason we're were so mean about this is because we've noticed some problems when we were more flexible
  we used to have a way to say try to do this function and if it's already on the stack give me back you know and a signal a None or something an optional value, I'll recover from the cycle

  because there are a lot of times when you would you know like to walk an entire graph and if you you have a cycle you just want to ignore it or handle it
  but it turns out that that's actually a very easy way to get things wrong

# Example: inlining (take 1)

  let me give you an example
  actually this is code that's still in rustc today but not configured on because this is still an experimental code where we do it sort of the wrong way and we have to fix it at some point before we turn it on
  and it wasn't obvious to me how this would go wrong at first

  fn optimized_mir(func_id: FunctionId) {
    let mut mir = db.unoptimized_mir(func_id);
    for call_site in mir.call_sites() {
      let callee_mir = db.optimized_mir(call_site.callee);
      mir.inline_call(call_site, callee_mir);
    }
  }

  fn foo() {
    bar();
  }
  fn bar() {
    ....
  }

  so what's happening here is we're doing inlining
  and the way we do it in rustc is we have this thing called mir which is our middle IR and it's kind of a low-level IR for rust sort of sort of like JVM but a little higher level on that or alike bytecode
  and it goes through some some phases right and the first one is we produce the unoptimized mirror and then we produce the optimized mir
  in between of course we do some optimizations on the mir

  one of those optimizations is inlining
  taking one function body putting it into the call site
  and it has some code that looked looks roughly like this
  it says okay if we're gonna compute the optimized form of this function
  let's get the unoptimized ir
  walk overall the call sites
  and get the optimized version of the function we're calling

  because you don't want to inline the unoptimized one in
  you'd rather have it already be optimized before you you inline it
  and and then we'll actually do the inlining

  this works fine as long as there's no cycle in the call graph
  if there's a cycle of course then it would panic which is not good

# Example: inlining (take 2)

  fn optimized_mir(func_id: FunctionId) {
    let mut mir = db.unoptimized_mir(func_id);
    for call_site in mir.call_sites() {
      if let Ok(callee_mir) = db.try_optimized_mir(call_site.callee) {
        mir.inline_call(call_site, callee_mir);
      }
    }
  }

  Attempt and recover on cycle

  so we said oh well we can just we can just use this recovery operation
  and say well let's just check if it is not a cycle then we'll inline in
  otherwise we'll just ignore it
  because actually you know we don't have to do this perfectly
  when you think about it
  if things are in the same cyclic component (same strongly connected component) you know that's basically an edge case and it's good enough for our optimizer

  we're gonna give this to LLVM anyway
  it's good enough for us to just handle the trees up there
  we want the leaf functions to get inlined

  so we tried this version of the code and it does indeed work
  it will optimize the Leafs of a call graph into their callers
  it won't handle the cyclic case
  but the problem is it does actually handle the cyclic case, it just doesn't handle it fully

# Non-deterministic results

  fn foo() {
    bar();
  }
  fn bar() {
    foo();
  }

  - If I start from foo, then bar is inlined into foo
  - But if I start from bar, then foo is inlined into bar

  and what I mean by that is let's suppose I have foo and bar to call one another

  if I start by optimizing foo then I'm gonna try to optimize bar and then I'm gonna fail because of the cycle and I'm gonna return back
  and so what will happen is I will produce an optimized bar and I will inline it into foo but I won't do the reverse
  but if I start from bar I'm gonna produce and optimize through and inline into bar
  so I'm going to get non-deterministic compilation results
  depending on which order I did the processing in
  and that's not good
  when you're doing this on demand compilation and all this stuff one of the key constraints that we want to produce is that you basically can't have non-deterministic results
  the framework generally does I think ensure that although I have not tried to prove it maybe I'm wrong if one of you sees an edge case please come talk to me
  but other people have proven it for similar frameworks that's probably true but this was a case where we we kind of by being too simplistic we failed that
  so the question is well how can I handle this then
  what should I do in a case like this

# One better approach

  - Construct call graph, compute SCCs as one query
  - Process one SCC at a time

  and one approach the one I think we should do in Rustc at least
  is that you make a sort of master query that computes the graph
  and so this might for example compute the call graph for the entire thing you're compiling
  it would walk all the functions figure out who calls one another and detect the
  cycles and basically instead of having the walk be done through the framework you're moving the walk into into a single query right
  and you're going to compute back out here's the list of strongly connected opponents here's the order in which you should process them and this is what a lot of compilers do of course
  and if we did it that way we wouldn't have this problem but we have the downside is in order to optimize any function we have to recompute the whole call graph every time
  because we haven't broken it up into sub pieces so I don't actually have a good answer for that I think that's a tricky problem
  this is one of the cases where there's some advantages to moving from to a higher level representation of what you're doing because you might be able to do finer grained incremental results
  however the thing I would say is that what I've observed is that it's basically good enough in most the time because you don't need to do these sort of optimizations for people when they're pressing the dot to give them completions because you don't need to give the optimized code
  and when you're actually generating the code this isn't like computing the call graph is not a big percentage of your compilation time
  so you can just redo it it's okay
  but I think handling it for when you if you actually do want the incremental reuse that's a little more tricky

# Other cases involving cycles

  From Rust:
      Name resolution
      Trait resolution
  Each has their own specific requirements

  so there's a bunch of cases in rust
  the thing about cycles is that the semantics of them really depends
  part what makes it tricky it depends on exactly what you're doing
  what you want to do in the event of a cycle
  there's no general correct answer well
  like just within rust we have cycles that arise besides inlining
  name resolution and trait resolution and
  each has their own specific requirements
  for example in trait resolution sometimes it would be
  trait resolution is figuring out whether a type implements an interface or not
  and that can be kind of complicated if you say this type implements display but only if this other type implements display so like a vector is displayable on the screen only if its elements are displayable on the screen and they may have their own requirements and you have to sort of evaluate this
  and if there's a cycle that basically means normally that means no they do not implement display because you want it to be acyclic but sometimes it's okay
  it can be tricky

<!-- PARSING ----------------------------------------------------------------->

# Tracking location information ("spans")

  Various techniques:
      rustc appends all the input files to one big string
      stores 32-bit indices into that string
      compact, but hostile to incremental
  One alternative:
      in the AST node, just stores its offset from previous sibling + length
      in other nodes, track the AST node id
      recompute the starting offset when needed

  next thing
  so the another interesting problem that has arisen is how do you track the location of things in a file
  I mentioned a couple of times as my example
  if all I did was edit a comment that shouldn't cause anything to recompile right?
  that's sort of true but not exactly true
  because editing a comment does affect the line and column information for your data and if you want to show an error on the screen you need that information

  and so it might actually in it and you're probably embedding the line and column information into your AST so it probably does actually affect your result
  and one of the things
  there are various ways to get around this they kind of come back to the tree trick that I showed earlier
  so you know what rusty was doing and still is doing in order to track location information was a was a highly memory optimized representation where basically we took all the input data from all of your input files and put it in one huge string and then we tracked with a single 32-bit number you could sort of get a span of bytes within that string
  usually those spans have a very short length so we you know try to like use five bits for that I don't know what it is use some small number of bits for the length and use the rest of it for the location and it works pretty well and sometimes it's overflows and we have a fallback

  but that whole system like is terrible for incremental right because now if you change one file not only did you invalidate all the location information within the file but all the other files that are in the same big string are also changed so we've been moving to different systems

  one way is don't track the spans at all or keep them separate
  don't put them into AST but have a separate table that says for a given just for a given ast node ID here's the span right and

  that table may completely change and be recomputed each time you parse
  but all the intermediate values are just carrying around the ID of the ast node and that's much more stable

  another way is to you can store offsets from the previous sibling and lengths or things like that but and basically trying to compute you just need to essentially all these schemes boiled down to tracking enough that you can figure out the actual line and column later
  but trying to minimize what you're carrying around in the moment

  so that the idea so for example you could just carry how long how long is each ast node and then you can later walk the whole AST and figure out well I don't know that doesn't tell you where it began in the file it only tells you how how long it is but if you walk all the previous nodes and sum up their lengths that'll tell you the beginning point and that you can only do in the event of error so maybe you don't need to do it at all most of the time

  this is basically what it comes down to is tricks like this
  exactly which one is best I don't think we know yet

  QUESTION

  the question was how do you track the spans across desugarrings and transformations and so rust has also a macro system which is highly relevant here
  so the spans in in rust are not only a line of text but they also have a call stack essentially from for tracking macro expansions across them and we use that in a couple of different ways

  one of the things that's really useful for is exactly these desugarings
  so the problem there is if you're desugaring, for example we do sugar our for loops into while loops but we don't want the users errors to talk about while loops when they typed for loop that would be confusing
  so we use this stack to push on and say well this is the span in the file where this token appears but it came from a desugarring from a from a for loop and that way we can customize the error message by inspecting this stack

  you do need to represent it
  I think it's kind of this a similar problem so that's also part of our very compact 32-bit representation is tracking those stacks they luckily occur kind of infrequently so you can mostly not worry about them
  I mean you can they can be less efficient
  but that's the basic idea though is you have to track the stack up there

# What to put in your AST

  another interesting question is what to put in your ast

  - Including whitespace, comments is useful for refactoring
  - Rest of compiler doesn't care

  QUESTION

  one of the things I think traditional compilers like basically love to throw away information whenever they possibly can and I think that's a good a nice thing to do it makes your compiler simpler it's better for incremental reuse for example
  but yeah it's not always what you want and the prime example is comments but also whitespace
  what the rusts compiler does right
  now this is another case where we're people are there are different opinions about what is best and I'm not I don't have a firm one yet
  but what the Russ compiler does is it it has a traditional approach it throws away all the comments and all the white space but it keeps precise spans and so you can sort of recover them by going back to the original text which we also have, finding the line and column number and sort of seeing where their comments and you know whitespace around it and stuff like that
  I think that the rust formatter so the code that automatically reformats your your rusts code does use that technique so it will say let me go find in the space between these two items let me go find all the comments that were in there and parts them and repost them

  QUESTION

  so we distinguish between doc comments which are the ones that will actually show up in the formatted documentation and ordinary comments and the doc comments are part of the compiler, they're part of the ast in a formal way but so the downside of this of course is that you have to do this weird stuff to like recover the comments, you have to recompile

# Incremental re-parsing

  - Compiler receives "diffs" to apply to previous inputs
  - Useful to be able to keep old AST trees around
  - If they contain absolute spans or information, they must be rebuilt

  you can recover the comments and things after the fact but
  there's another approach which is I think what they use in Swift and what's we're trying in a different project where you actually keep all the information in your in your tree
  the problem is then you want to throw it away and it's kind of annoying for all the reasons now you have more information than you need

# Example: swift red and black trees

  so what Swift does for example as I understand it is they have these two layers they call red and black
  a black tree contains all the details of the the comments and all the white space and so on
  actually I think both our trees do but but the main thing about the black one is it doesn't have it's it's all relative to their current point
  it only has for example the lengths of ast nodes as I mentioned
  and so that means it can be incrementally reused

  so if you reparse and you know there's been no changes in this function you can just reuse it but then the the red tree layers on top and computes lazily all the other context you would need

# Alternative: "zooming out" or "zooming in"

  and that's one approach but the other one is to do this kind of zooming out on in and I think this is what Rustc does this is what I think the typescript compiler does so I understand though I haven't read that source and I'm not sure which is better I
  I sort of leaned towards this one myself but that is to say leaving it out but being able to recover it

  - Eliding data is good
      - especially when it can be recovered
  - On-demand system is a good fit for this

<!-- THREADING --------------------------------------------------------------->

# Threading

  - Compiler is effectively a server
  - Spawn off threads to handle requests

  the last thing I wanted to say is
  that there's also this need to handle threading
  when when your compiler is an actor taking messages back and forth
  it needs to be able to process them and and always be responsive to the editor as it makes changes
  so it's not okay to just sit there and take over the main thread and not answer questions

# Salsa's threading model

  - Effectively a read-write lock:
    - One master thread that applies edits
    - Any number of "helper threads"
        - while the helper threads are active, attempts to edit will block

  so what we do the salsa has a threading model basically
  where there's a master thread that is able to change the inputs
  and then there are helper threads that are only able to read and compute derive values
  of course they're actually making changes in the database but it's hidden from you
  and there's basically a readwrite lock here
  so if the master thread goes to set an input and there are still active helper threads out there it will block until they've completed

# Cancellation

  - Threads can check periodically for pending edits
  - Easiest way to recover is to panic and unwind the thread

  but this blocking as I just said is not so great because now you're not responding to the users requests so we have this notion of cancellation where essentially if there's a master thread wanting to change the input and you're off computing some derived value then you should panic
  that means they will propagate an error, it will unwind all the stuff your thread will die and while that happens we just don't make changes to the database basically
  and once all the helper threads have have cleaned themselves up the master thread can make its change and we can keep going
  this is what you basically want to do
  I think is this kind of cancellation the exact mechanism may vary
  but that's the basic idea
  so that you can essentially when people are typing and they press dot and then they press backspace and then they push dot again type a little more you can recover and handle all those things so

  QUESTION

  we don't cancel it, we don't like inject a pair wherever it happens to be but if it
  so they're in the middle of doing some recomputation if they happen to complete before we check for panic we will store that like any other incremintal but if they panic in the middle of the function then we just leave the state as it was
  and that way when you after you apply the new diff and then you will presumably restart those computations they will just re execute

  one other thing I didn't mention but I'll mention here which I think is relevant is when you have multiple threads we also support many threads at once doing different things so you can type check all your functions in parallel or whatever but they might all need to access the same value
  so they might say what is the signature of this function that they're both calling or something right
  and what we currently do this is this is one of those places where I think the best strategy also will depend
  what we currently do is they block so one of them wins it computes the value of the others block and wait for it and then we re execute

  this seems to work pretty well we've measured it

  but there are many alternatives
  like they could both go and do the computation because it's a pure functional computation as long as we handle the error propagation right we'll be fine

  what you want to do probably depends on how much work that is
  like computing the signature is very cheap usually but maybe doing the whole type check is not so you don't really want to do it twice

# Conclusion

  - Start with an on-demand style
  - Best practices in some areas still need to be codified
      - how to represent ASTs? location information?
      - how to handle cycles?

  so I think my conclusions
  as I said I thought I would when I when I agreed to give this talk
  I thought surely by now we'll have a really great working system
  and I'll I think exactly what I think you should do but I don't
  but what I do know is you should at least start with an on demand style in my opinion
  that proven to be a nice way to write a think to write the compiler
  it doesn't have to be
  just starting with basically an on-demand style using error sentinels and a few other I think those are the two big things from the beginning I think will sort of put you in the right ballpark for building a responsive IDE

  and the details of even how you represent your spans but you know definitely stuff like optimizing your memoization and does the thread
  how do you handle cancellation is less important at the end of the day
  and easier to add on after the fact

  but those two things are really quite painful
  so that's my lesson

  QUESTION

  yeah that's a very good question
  so I elided parser error recovery
  yes so the answer is the way you would do it is also a sentinel value
  so you have an ast node that is error
  and it includes you know some chunk of tokens and some information about the error usually
  and in practice there's a lot of
  this is another of those things where people treat parsing like a solve problem
  but actually you know this is tricky
  what I've seen in practice for error recovery though is that it's usually people do a pretty simple strategy and it works pretty well
  basically looking ahead for like a semicolon that's probably a strong signal or some other key word or something that really means like let's you reorient where you are and let the rest fall out

  but yeah it's a good question

  is the on demand as important for the backend

  probably not
  so I think what we do in the rust compiler of course
  LLVM does most of our back-end compilation
  so what we do is we we do use on-demand sort of up until
  we basically create the elevate my our on-demand but then we ship it off to LVM and let it do its thing and at that point you can just let it run

  it's also less vital for this incremental reuse I think and also the cycles and stuff gets much more complicated there

  QUESTION

  yeah if you can write a one-pass compiler like turbo pascal for your language then maybe you should just do that
  I think that's a good point

  it just often in practice doesn't turn out that simple these days
  so I think that's the bottom line

  QUESTION

  the back door we can bring in very
  expensive competition Liberty
  competition
  these masturbate is really needed
  because I mean I better just go and
  introduce many of these masters oh yeah

  I think that so I think that's correct

  you could make an on-demand program where sort of you could take the old model of do the phase on the whole program and just make a query per face with no inputs essentially and you wouldn't get very much benefit right
  so it's not that the system falls down but it's that your incremental performance is suboptimal

  QUESTION

  so setting up with some smallish number of very big queries and then rebuilding
  I think that that would work fine
  and we've been trying to do that also
  it is definitely easier if you do it from the beginning though then coming back to added later

  that is a reasonable approach in my opinion
  I think the places where you really need the bigger queries are mostly around this cycles and things like that

  either
  - the cycles or
  - the produce all the errors
  - compile the whole program
  - the sort of batch compilation end points

  those are the two places

  I think you'll find that if you're doing the finer grained stuff it's quite natural to make it finer grained

  I'm not sure if you did the approach of making one query per phase
  you might find it kind of hard later on to slice them up
  you could do it it just might be more work than you then it would have been to do it from the beginning
