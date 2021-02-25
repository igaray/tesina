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

	- where does the divergence come in?
	- is it the case that you have two compilers?
		one for build, another for editing?
		thats a good question, the first version of the c#, before roslyn, was written in c++
		by the time we finished it, it was a lot of code
		then we had the ide version
		and they started diverging, and you had twice the work, you had to implement things twice
		and that was the origin of the roslyn project
		1 we cant keep implementing everything twice
		2 we want people to have access to the insight the compiler has into your code, to be able to write tools
		so we decided to build the compiler as an api
		in a manner that was engineereed to be usable from an ide
		now its the same thing, the cli is just a driver
	- in the typescript we had a similar architecture
	-

---
- https://ollef.github.io/blog/posts/query-based-compilers.html

---
- https://en.wikipedia.org/wiki/Incremental_compiler

---
- https://doc.rust-lang.org/edition-guide/rust-2018/the-compiler/incremental-compilation-for-faster-compiles.html
- https://blog.rust-lang.org/2016/09/08/incremental.html

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