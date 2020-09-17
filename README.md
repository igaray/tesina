# UNS: Tesina

---
- Anders Hejlsberg on Modern Compiler Construction
	- https://www.youtube.com/watch?v=wSdV1M7n4gQ
	- the compiler book
	- we dont even do it that way anymore
	- over the last decade or so, we've become very accustomed to very intelligent ides and development tools and features like statement completion, refactoring, goto to definition, code navigation, error lists that are automatically populated
	- all of that underneath the covers is powered by a compiler
	- but its a very different compiler from what they teach you classically to build in in school
	- and theyre diverging more and more
	- there are things that you need to do today when you build these these services that editors use that are actually compilers

	- how are we taught
	- where are the gaps starting to show
	- classic compiler
	- typically has a lexer, parser, type checker, code generator, emitter
	- frontend, backend
	- in comes source text, the lexer breaks it into tokens
	- then the parser builds what's called abstract syntax tree, a tree representation of your source code
	- typically then you'll build up symbol tables and then the type checker assigns types to variables and then checks that everything is okay
	- that information in turn then can guide a code generator, here you may have a different representation that you you go into where you build some byte code, then for example then you can run to peephole optimization
	- and then finally the emitter that's sort of your classic text to machine code compiler
	- there might be a linker
	- that's sort of the classic picture

	- these compilers, after you build all these abstract syntax trees, with all your different statements or whatever
	- then you may do a whole number of passes over them and maybe even rewrite them into different trees before the type checker and then the code generator turns this kind of representation to another representation

	- so there's lots of transformations and lots of work that happens in a long
	- pipeline and it takes a while to get from here to here
	- obviously I mean it can take I'm in it for ages right my god

	- lets say Im editing my source code in my editor, first of all I want syntax coloring
	- I want to know first of all I want syntax coloring right
	- thats something my lexer can do
	- a lexer just looks at a little piece of the code
	- it looks at right where youre typing
	- its easy to syntax colorize

	- if you look back at the history of ides that was one of the first things surfaced
	- but then you also want to have collapsable regions, and now you start to need trees, you need to understang how all these keywords combin
	- where constucts begin and end
	- you would also like to see red where you make mistakes
	- and when you press dot you'd like to see a list of what could go
	- you could just show the user all identifiers in the file, it's probably one of them  but maybe it isnt'
	- you want to show what semantically could go there
	- so you now you need information produced by the type checker

	- you see as you get more and more intelligent you need more and more expensive information
	- in a classic compiler it takes a while to type check a whole program, and the bigger the program the longer it takes

	- yet we know from user studies, if you dont deliver the dropdown in 100ms, it's too slow
	- even if you're editing a 100000 line program

	- how do you do that?

	- this feels easy up until the moment you start thinking about code that is not correct
	- with incorrect semantics or syntax, and you still want something to happen

	- one of the big differences between a compiler that works on a finiches program and a copiler that sits behind your ide
	- is that the compiler that works on the finished program essentially assumed that everything is correct

	- in an editor, the core operating assumption has to be the exact reverse
	- whenever the user is typing code, his code is probably broke, hes in the middle of coding, hes not done
	- so youre error recovery has to be really good
	- and you have to somewhat antipate whats coming next
	- and you have to recognize more syntax than is traditionally valid because you know that users make certain mistakes
	- so instead of you seaying "error invalid token" its better to parse out what you think the user is meaning, and then give a more meaningful error messeage if you see a certain pattern, of what people think they can do but cant actually do
	- so sometimes you parse more than the permitted syntax just to give better error messages

	-

---
- https://ollef.github.io/blog/posts/query-based-compilers.html
	- Compilers are no longer just black boxes that take a bunch of source files and produce assembly code. We expect them to:
		- Be incremental, meaning that if we recompile a project after having made a few changes we only recompile what is affected by the changes.
		- Provide editor tooling, e.g. through a language server, supporting functionality like going to definition, finding the type of the expression at a specific location, and showing error messages on the fly.
	- A traditional compiler pipeline might look a bit like this:
		```
		+-----------+            +-----+                +--------+               +--------+
		|           |            |     |                |        |               |        |
		|source text|---parse--->| AST |---typecheck-+->|core AST|---generate--->|assembly|
		|           |            |     |       ^        |        |               |        |
		+-----------+            +-----+       |        +--------+               +---------
		                                       |
		                                 read and write
		                                     types
		                                       |
		                                       v
		                                  +----------+
		                                  |          |
		                                  |type table|
		                                  |          |
		                                  +----------+
		```
	- Going from pipeline to queries
		- What does it take to get the type of a qualified name, such as "Data.List.map"?
		- In a pipeline-based architecture we would just look it up in the type table.
		- With queries, we have to think differently.
		- Instead of relying on having updated some piece of state, we do it as if it was done from scratch.

---
- https://en.wikipedia.org/wiki/Incremental_compiler
	- An incremental compiler is a kind of incremental computation applied to the field of compilation. Quite naturally, whereas ordinary compilers make so called clean build, that is, (re)build all program modules, incremental compiler recompiles only those portions of a program that have been modified.
	- In imperative programming and software development, an incremental compiler is one that when invoked, takes only the changes of a known set of source files and updates any corresponding output files (in the compiler's target language, often bytecode) that may already exist from previous compilations. By effectively building upon previously compiled output files, the incremental compiler avoids the wasteful recompiling of entire source files, where most of the code remains unchanged.
	- For most incremental compilers, compiling a program with small changes to its source code is usually near instantaneous. It can be said that an incremental compiler reduces the granularity of a language's traditional compiling units while maintaining the language's semantics, such that the compiler can append and replace smaller parts.
	- Many programming tools take advantage of incremental compilers to provide developers with a much more interactive programming environment. It is not unusual that an incremental compiler is invoked for every change of a source file, such that the developer is almost immediately informed about any compilation errors that would arise as a result of his changes to the code. This scheme, in contrast with traditional compilation, shortens a programmer's development cycle significantly, because they would no longer have to wait for a long compile process before being informed of errors.
	- One downside to this type of incremental compiler is that it cannot easily optimize the code that it compiles, due to locality and the limited scope of what is changed. This is usually not a problem, because for optimization is usually only carried out on release, an incremental compiler would be used throughout development, and a standard batch compiler would be used on release.

---
- https://doc.rust-lang.org/edition-guide/rust-2018/the-compiler/incremental-compilation-for-faster-compiles.html
- https://blog.rust-lang.org/2016/09/08/incremental.html
	- Improving compile times has actually been a major development focus after Rust reached 1.0 -- although, up to this point, much of the work towards this goal has gone into laying architectural foundations within the compiler and we are only slowly beginning to see actual results.
	- One of the projects that is building on these foundations, and that should help improve compile times a lot for typical workflows, is incremental compilation. Incremental compilation avoids redoing work when you recompile a crate, which will ultimately lead to a much faster edit-compile-debug cycle.
	- **Why Incremental Compilation in the First Place?**
		- Much of a programmer's time is spent in an edit-compile-debug workflow:
			- you make a small change (often in a single module or even function),
			- you let the compiler translate the code into a binary, and finally
			- you run the program or a bunch of unit tests in order to see results of the change.
		- After that it's back to step one, making the next small change informed by the knowledge gained in the previous iteration. This essential feedback loop is at the core of our daily work. We want the time being stalled while waiting for the compiler to produce an executable program to be as short as possible.
		- Incremental compilation is a way of exploiting the fact that little changes between compiles during the regular programming workflow: Many, if not most, of the changes done in between two compilation sessions only have local impact on the machine code in the output binary, while the rest of the program, same as at the source level, will end up exactly the same, bit for bit. Incremental compilation aims at retaining as much of those unchanged parts as possible while redoing only that amount of work that actually must be redone.
		- **How Do You Make Something "Incremental"?**
			- We have already heard that computing something incrementally means updating only those parts of the computation's output that need to be adapted in response to a given change in the computation's inputs.
			- One basic strategy we can employ to achieve this is to view one big computation (like compiling a program) as a composite of many smaller, interrelated computations that build up on each other.
			- Each of those smaller computations will yield an intermediate result that can be cached and hopefully re-used in a later iteration, sparing us the need to re-compute that particular intermediate result again.
		- **An Incremental Compiler**
			- The way we chose to implement incrementality in the Rust compiler is straightforward: An incremental compilation session follows exactly the same steps in the same order as a batch compilation session.
			- However, when control flow reaches a point where it is about to compute some non-trivial intermediate result, it will try to load that result from the incremental compilation cache on disk instead.
			- If there is a valid entry in the cache, the compiler can just skip computing that particular piece of data. Let's take a look at a (simplified) overview of the different compilation phases and the intermediate results they produce:
			- First the compiler will parse the source code into an abstract syntax tree (AST). The AST then goes through the analysis phase which produces type information and the MIR for each function. After that, if analysis did not find any errors, the codegen phase will transform the MIR version of the program into its machine code version, producing one object file per source-level module. In the last step all the object files get linked together into the final output binary which may be a library or an executable.
			- So, this seems pretty simple so far: Instead of computing something a second time, just load the value from the cache. Things get tricky though when we need to find out if it's actually valid to use a value from the cache or if we have to re-compute it because of some changed input.
		- **Dependency Graphs**
			- There is a formal method that can be used to model a computation's intermediate results and their individual "up-to-dateness" in a straightforward way: dependency graphs.
			- It looks like this: Each input and each intermediate result is represented as a node in a directed graph. The edges in the graph, on the other hand, represent which intermediate result or input can have an influence on some other intermediate result.
			- Note, by the way, that the above graph is a tree just because the computation it models has the form of a tree. In general, dependency graphs are directed acyclic graphs
			- What makes this data structure really useful is that we can ask it questions of the form "if X has changed, is Y still up-to-date?". We just take the node representing Y and collect all the inputs Y depends on by transitively following all edges originating from Y. If any of those inputs has changed, the value we have cached for Y cannot be relied on anymore.
		- **Dependency Tracking in the Compiler**
			- When compiling in incremental mode, we always build the dependency graph of the produced data: every time, some piece of data is written (like an object file), we record which other pieces of data we are accessing while doing so.
			- The emphasis is on recording here. At any point in time the compiler keeps track of which piece of data it is currently working on (it does so in the background in thread-local memory).
			- This is the currently active node of the dependency graph. Conversely, the data that needs to be read to compute the value of the active node is also tracked: it usually already resides in some kind container (e.g. a hash table) that requires invoking a lookup method to access a specific entry.
			- We make good use of this fact by making these lookup methods transparently create edges in the dependency graph: whenever an entry is accessed, we know that it is being read and we know what it is being read for (the currently active node).
			- This gives us both ends of the dependency edge and we can simply add it to the graph. At the end of the compilation sessions we have all our data nicely linked up, mostly automatically.
			- This dependency graph is then stored in the incremental compilation cache directory along with the cache entries it describes.
			- At the beginning of a subsequent compilation session, we detect which inputs (=AST nodes) have changed by comparing them to the previous version. Given the graph and the set of changed inputs, we can easily find all cache entries that are not up-to-date anymore and just remove them from the cache.
			- Anything that has survived this cache validation phase can safely be re-used during the current compilation session.
			- There are a few benefits to the automated dependency tracking approach we are employing. Since it is built into the compiler's internal APIs, it will stay up-to-date with changes to the compiler, and it is hard to accidentally forget about. And if one still forgets using it correctly (e.g. by not declaring the correct active node in some place) then the result is an overly conservative, but still "correct" dependency graph: It will negatively impact the re-use ratio but it will not lead to incorrectly re-using some outdated piece of data.
			- Another aspect is that the system does not try to predict or compute what the dependency graph is going to look like, it just keeps track. A large part of our (yet to be written) regression tests, on the other hand, will give a description of what the dependency graph for a given program ought to look like. This makes sure that the actual graph and the reference graph are arrived at via different methods, reducing the risk that both the compiler and the test case agree on the same, yet wrong, value.
		- **"Faster! Up to 15% or More."***
			- Let's take a look at some of the implications of what we've learned so far:
				- The dependency graph reflects the actual dependencies between parts of the source code and parts of the output binary.
				- If there is some input node that is reachable from many intermediate results, e.g. a central data type that is used in almost every function, then changing the definition of that data type will mean that everything has to be compiled from scratch, there's no way around it.
			- In other words, the effectiveness of incremental compilation is very sensitive to the structure of the program being compiled and the change being made. Changing a single character in the source code might very well invalidate the whole incremental compilation cache. Usually though, this kind of change is a rare case and most of the time only a small portion of the program has to be recompiled.
		- **The Current Status of the Implementation** (09/2019)
			- For the first spike implementation of incremental compilation, what we call the alpha version now, we chose to focus on caching object files.
			- Consequently, if this phase can be skipped at least for part of a code base, this is where the biggest impact on compile times can be achieved.
			- With that in mind, we can also give an upper bound on how much time this initial version of incremental compilation can save: If the compiler spends X seconds optimizing when compiling your crate, then incremental compilation will reduce compile times at most by those X seconds.
			- Another area that has a large influence on the actual effectiveness of the alpha version is dependency tracking granularity: It's up to us how fine-grained we make our dependency graphs, and the current implementation makes it rather coarse in places. For example, the dependency graph only knows a single node for all methods in an impl. As a consequence, the compiler will consider all methods of that impl as changed if just one of them is changed. This of course will mean that more code will be re-compiled than is strictly necessary.
		- **Future Plans** (09/2019)
			- The section on the current status already laid out the two major axes along which we will pursue increased efficiency:
				- Cache more intermediate results, like MIR and type information, which will allow the compiler to skip more and more steps.
				- Make dependency tracking more precise, so that the compiler encounters fewer false positives during cache invalidation.

---
- Rustc Dev Guide
	- [Overview of the Compiler](https://rustc-dev-guide.rust-lang.org/overview.html)
	- [High-level overview of the compiler source](https://rustc-dev-guide.rust-lang.org/compiler-src.html)
	- [Queries: demand-driven compilation](https://rustc-dev-guide.rust-lang.org/query.html)
		- [The Query Evaluation Model in Detail](https://rustc-dev-guide.rust-lang.org/queries/query-evaluation-model-in-detail.html)
		- [Incremental compilation](https://rustc-dev-guide.rust-lang.org/queries/incremental-compilation.html)
		- [Incremental Compilation In Detail](https://rustc-dev-guide.rust-lang.org/queries/incremental-compilation-in-detail.html)
		- [Debugging and Testing Dependencies](https://rustc-dev-guide.rust-lang.org/incrcomp-debugging.html)
		- [Profiling Queries](https://rustc-dev-guide.rust-lang.org/queries/profiling.html)
		- [How Salsa works](https://rustc-dev-guide.rust-lang.org/salsa.html)
- [Incremental Compilation Working Group](https://www.youtube.com/playlist?list=PL85XCvVPmGQh0P_VEPVM2ZIlBwl4MQMNY)

---
- [Responsive compilers - Nicholas Matsakis - PLISS 2019](https://www.youtube.com/watch?v=N6b44kMS6OM)
- [Things I Learned (TIL) - Nicholas Matsakis - PLISS 2019](https://www.youtube.com/watch?v=LIYkT3p5gTs)
- [How Salsa Works (2019.01)](https://www.youtube.com/watch?v=_muY4HjSqVw)
- [Salsa In More Depth (2019.01)](https://www.youtube.com/watch?v=i_IhACacPRY)
- [RLS 2.0, Salsa, and Name Resolution](https://www.youtube.com/watch?v=Xr-rBqLr-G4)
- [JuliaCon 2020 | Salsa.jl: A framework for on-demand, incremental computation | Nathan Daly](https://www.youtube.com/watch?v=0uzrH2Ee494)
- [Making Fast Incremental Compiler for Huge Codebase - Michał Bartkowiak - code::dive 2019](https://www.youtube.com/watch?v=S2dK5lLFD_0)
- [Jaseem Abid - An Incremental Approach to Compiler Construction](https://www.youtube.com/watch?v=WBWRkUuyuE0)
- [Incremental Compile Updates in Vivado 2015.3](https://www.youtube.com/watch?v=9R5f5sScJh4)
- [Incremental compiler taming scalac by Krzysztof Romanowski at Scalar Conf 2016](https://www.youtube.com/watch?v=7eQ-8f6lYOE)

---
- [2016 LLVM Developers’ Meeting: D. Dunbar “A New Architecture for Building Software”](https://www.youtube.com/watch?v=b_T-eCToX1I)
- [On the Architecture of a (Verifying) Compiler](https://www.youtube.com/watch?v=TO9F3RvA0LA)
- [Build Your Own WebAssembly Compiler](https://www.youtube.com/watch?v=OsGnMm59wb4)
- [2019 LLVM Developers’ Meeting: E. Christopher & J. Doerfert “Introduction to LLVM”](https://www.youtube.com/watch?v=J5xExRGaIIY)
- [Introduction to LLVM Building simple program analysis tools and instrumentation](https://www.youtube.com/watch?v=VKIv_Bkp4pk)
- [Creating, Coding and Compiling a Compiler with LLVM (/dev/world/2013)](https://www.youtube.com/watch?v=L6Gz1hJZECg)
- [Implementing Domain Specific Languages with LLVM](https://www.youtube.com/watch?v=1Hwnagof1Wo)

---
- [An approach to incremental compilation](https://dl.acm.org/doi/10.1145/502949.502889)
- [Incremental Algorithms for Approximate Compilation](http://www.cs.ucc.ie/ccsl/GP-papers/2008/AAAI08_Incremental_submit.pdf)
- [Symbolic Debugging Through Incremental Compilation in an Integrated Environment](http://www.softwarepreservation.org/projects/interactive_c/bib/Fritzson-1983.pdf)

	- **Rust Analyzer**
		- [Rust Analyzer](https://rust-analyzer.github.io/)
		- [Manual](https://rust-analyzer.github.io/manual.html)
		- [Blog](https://rust-analyzer.github.io/blog)
		- [rust-analyzer/tree/master/docs/dev](https://github.com/rust-analyzer/rust-analyzer/tree/master/docs/dev)
		- [Changelog](https://rust-analyzer.github.io/thisweek)
		- [Github](https://github.com/rust-analyzer/rust-analyzer)
		- [Guide](https://github.com/rust-analyzer/rust-analyzer/blob/e0d8c86563b72e5414cf10fe16da5e88201447e2/guide.md)
			- [Guide Video](https://youtu.be/ANKBNiSWyfc)
		- [Rust Analyzer in 2018 and 2019](https://ferrous-systems.com/blog/rust-analyzer-2019/)
		- [Status of rust-analyzer: Achievements and Open Collective](https://ferrous-systems.com/blog/rust-analyzer-status-opencollective/)
		- [Rust Analyzer Q&A](https://www.youtube.com/playlist?list=PLXajQV_H-DxLMBt0amcuxgTeOTj6L-YGl)
		- [2020 Intro to Rust Analyzer](https://blog.logrocket.com/intro-to-rust-analyzer/)
		- [2020 What I learned contributing to Rust-Analyzer ](https://dev.to/bnjjj/what-i-learned-contributing-to-rust-analyzer-4c7e)

---
- https://github.com/rust-lang/rfcs/blob/master/text/1298-incremental-compilation.md

- https://www.microsoft.com/en-us/research/publication/build-systems-la-carte/
- https://petevilter.me/post/datalog-typechecking/

- https://github.com/bytecodealliance/wasmtime/blob/ec87aee147cb0c8a8ae7b5db192daa2163207c8a/cranelift/docs/compare-llvm.md
- https://thume.ca/2019/04/18/writing-a-compiler-in-rust/
- https://thume.ca/2019/04/29/comparing-compilers-in-rust-haskell-c-and-python/
- https://www.cs.cmu.edu/~mleone/language-people.html

- https://en.wikipedia.org/wiki/Substructural_type_system
- https://en.wikipedia.org/wiki/Linear_logic
- https://en.wikipedia.org/wiki/Type_theory

- http://www.jonathanturner.org/2016/10/programming-language-and-compilers-reading-list.html
- https://lexi-lambda.github.io/blog/2020/08/13/types-as-axioms-or-playing-god-with-static-types/
- https://www.sanity.io/blog/why-we-wrote-yet-another-parser-compiler

- https://en.wikipedia.org/wiki/CompCert
- https://dl.acm.org/doi/10.1007/s10817-009-9155-4
- http://compcert.inria.fr/doc/index.html
- https://vst.cs.princeton.edu/
- https://en.wikipedia.org/wiki/Formal_verification
- https://arxiv.org/pdf/1603.03599.pdf
- https://cacm.acm.org/magazines/2015/4/184701-how-amazon-web-services-uses-formal-methods/fulltext
- https://lamport.azurewebsites.net/tla/formal-methods-amazon.pdf
- https://en.wikipedia.org/wiki/TLA%2B
- https://www.microsoft.com/en-us/research/uploads/prod/2016/12/The-PlusCal-Algorithm-Language.pdf
- https://github.com/pair-code/lit/