# How Salsa Works (2019.01)

# How Salsa works (from the RDG)

<!-- toc -->

This chapter is based on the explanation given by Niko Matsakis in this
[video](https://www.youtube.com/watch?v=_muY4HjSqVw) about
[Salsa](https://github.com/salsa-rs/salsa). To find out more you may
want to watch [Salsa In More
Depth](https://www.youtube.com/watch?v=i_IhACacPRY), also by Niko
Matsakis.

> As of January 2021 <!-- date: 2021-01 -->, although Salsa is inspired by
> (among other things) rustc's query system, it is not used directly in rustc.
> It _is_ used in chalk and extensively in `rust-analyzer`, but there are no
> medium or long-term concrete plans to integrate it into the compiler.

## What is Salsa?

Salsa is a library for incremental recomputation. This means it allows reusing
computations that were already done in the past to increase the efficiency
of future computations.

The objectives of Salsa are:
 * Provide that functionality in an automatic way, so reusing old computations
   is done automatically by the library
 * Doing so in a "sound", or "correct", way, therefore leading to the same
   results as if it had been done from scratch

Salsa's actual model is much richer, allowing many kinds of inputs and many
different outputs.
For example, integrating Salsa with an IDE could mean that the inputs could be
the manifest (`Cargo.toml`), entire source files (`foo.rs`), snippets and so
on; the outputs of such an integration could range from a binary executable, to
lints, types (for example, if a user selects a certain variable and wishes to
see its type), completions, etc.

## How does it work?

The first thing that Salsa has to do is identify the "base inputs" that
are not something computed but given as input.

Then Salsa has to also identify intermediate, "derived" values, which are
something that the library produces, but, for each derived value there's a
"pure" function that computes the derived value.

For example, there might be a function `ast(x: Path) -> AST`. The produced
`AST` isn't a final value, it's an intermidiate value that the library would
use for the computation.

This means that when you try to compute with the library, Salsa is going to
compute various derived values, and eventually read the input and produce the
result for the asked computation.

In the course of computing, Salsa tracks which inputs were accessed and which
values are derived. This information is used to determine what's going to
happen when the inputs change: are the derived values still valid?

This doesn't necessarily mean that each computation downstream from the input
is going to be checked, which could be costly. Salsa only needs to check each
downstream computation until it finds one that isn't changed. At that point, it
won't check other derived computations since they wouldn't need to change.

It's is helpful to think about this as a graph with nodes. Each derived value
has a dependency on other values, which could themselves be either base or
derived. Base values don't have a dependency.

```ignore
I <- A <- C ...
          |
J <- B <--+
```

When an input `I` changes, the derived value `A` could change. The derived
value `B` , which does not depend on `I`, `A`, or any value derived from `A` or
`I`, is not subject to change.  Therefore, Salsa can reuse the computation done
for `B` in the past, without having to compute it again.

The computation could also terminate early. Keeping the same graph as before,
say that input `I` has changed in some way (and input `J` hasn't) but, when
computing `A` again, it's found that `A` hasn't changed from the previous
computation. This leads to an "early termination", because there's no need to
check if `C` needs to change, since both `C` direct inputs, `A` and `B`,
haven't changed.

## Key Salsa concepts

### Query

A query is some value that Salsa can access in the course of computation.  Each
query can have a number of keys (from 0 to many), and all queries have a
result, akin to functions.  0-key queries are called "input" queries.

### Database

The database is basically the context for the entire computation, it's meant to
store Salsa's internal state, all intermediate values for each query, and
anything else that the computation might need.  The database must know all the
queries that the library is going to do before it can be built, but they don't
need to be specified in the same place.

After the database is formed, it can be accessed with queries that are very
similar to functions.  Since each query's result is stored in the database,
when a query is invoked N times, it will return N **cloned** results, without
having to recompute the query (unless the input has changed in such a way that
it warrants recomputation).

For each input query (0-key), a "set" method is generated, allowing the user to
change the output of such query, and trigger previous memoized values to be
potentially invalidated.

### Query Groups

A query group is a set of queries which have been defined together as a unit.
The database is formed by combining query groups. Query groups are akin to
"Salsa modules".

A set of queries in a query group are just a set of methods in a trait.

To create a query group a trait annotated with a specific attribute
(`#[salsa::query_group(...)]`) has to be created.

An argument must also be provided to said attribute as it will be used by Salsa
to create a struct to be used later when the database is created.

Example input query group:

```rust,ignore
/// This attribute will process this tree, produce this tree as output, and produce
/// a bunch of intermidiate stuff that Salsa also uses.  One of these things is a
/// "StorageStruct", whose name we have specified in the attribute.
///
/// This query group is a bunch of **input** queries, that do not rely on any
/// derived input.
#[salsa::query_group(InputsStorage)]
pub trait Inputs {
    /// This attribute (`#[salsa::input]`) indicates that this query is a base
    /// input, therefore `set_manifest` is going to be auto-generated
    #[salsa::input]
    fn manifest(&self) -> Manifest;

    #[salsa::input]
    fn source_text(&self, name: String) -> String;
}
```

To create a **derived** query group, one must specify which other query groups
this one depends on by specifying them as supertraits, as seen in the following
example:

```rust,ignore
/// This query group is going to contain queries that depend on derived values a
/// query group can access another query group's queries by specifying the
/// dependency as a super trait query groups can be stacked as much as needed using
/// that pattern.
#[salsa::query_group(ParserStorage)]
pub trait Parser: Inputs {
    /// This query `ast` is not an input query, it's a derived query this means
    /// that a definition is necessary.
    fn ast(&self, name: String) -> String;
}
```

When creating a derived query the implementation of said query must be defined
outside the trait.  The definition must take a database parameter as an `impl
Trait` (or `dyn Trait`), where `Trait` is the query group that the definition
belongs to, in addition to the other keys.

```rust,ignore
///This is going to be the definition of the `ast` query in the `Parser` trait.
///So, when the query `ast` is invoked, and it needs to be recomputed, Salsa is going to call this function
///and it's is going to give it the database as `impl Parser`.
///The function doesn't need to be aware of all the queries of all the query groups
fn ast(db: &impl Parser, name: String) -> String {
    //! Note, `impl Parser` is used here but `dyn Parser` works just as well
    /* code */
    ///By passing an `impl Parser`, this is allowed
    let source_text = db.input_file(name);
    /* do the actual parsing */
    return ast;
}
```

Eventually, after all the query groups have been defined, the database can be
created by declaring a struct.

To specify which query groups are going to be part of the database an attribute
(`#[salsa::database(...)]`) must be added. The argument of said attribute is a
list of identifiers, specifying the query groups **storages**.

```rust,ignore
///This attribute specifies which query groups are going to be in the database
#[salsa::database(InputsStorage, ParserStorage)]
#[derive(Default)] //optional!
struct MyDatabase {
    ///You also need this one field
    runtime : salsa::Runtime<MyDatabase>,
}
///And this trait has to be implemented
impl salsa::Databse for MyDatabase {
    fn salsa_runtime(&self) -> &salsa::Runtime<MyDatabase> {
        &self.runtime
    }
}
```

Example usage:

```rust,ignore
fn main() {
    let db = MyDatabase::default();
    db.set_manifest(...);
    db.set_source_text(...);
    loop {
        db.ast(...); //will reuse results
        db.set_source_text(...);
    }
}
```


---

# What is Salsa?
- Incremental recomputation


    fn foo(input1: A, input2: B) -> Output


    let x1 = foo(a, b1) // reuse some of the work we did
    let x2 = foo(a, b2)


- “Automatic”
- “Sound / Correct” — you get the same result as if we did not re-use at all


# Salsa’s actual model is richer


- IDE:
  - Inputs:
    - manifest (`Cargo.toml`)
    - source files (`foo.rs =>` `"``…``"`)
  - Outputs:
    - Binary executable
    - Completions at this point?
    - What should I show the user when the mouse is hovering on this line?


# How does it work?


- Identify the “base inputs” to your computation
- Identify intermediate “derived” values
  - deterministic, “pure” function that computes them
- Inputs:
  - manifest `manifest() → Vec<Path>`
  - source text `source_text(x: Path) → String`
- Derived values:
  - for each source file `X`, we might have a derived value `ast(x: Path) → AST`
  - `completion(path: Path, line: usize, column: usize)`
- Track:
  - in the course of computing (say) `ast(Path)`,
    - which inputs did we access?
    - which derived values?
- When an input changes:
  - these derived values are still valid (because none of the inputs that they touch changed)
  - these derived values may be invalid (because some of the inputs that they touch changed)


    manifest() <------------------------ whole_program_ast() <-- type-checking
                                               |
    source_text("a.rs") <- ast("a.rs") <-------+
                                               |
    source_text("b.rs") <- ast("b.rs") <-------+


# Key Salsa concepts
## Query


- `manifest() → Manifest` — (input) query
- `whole_program_ast() → Ast` — (derived) query
- `source_text(p: Path) → String` — (derived) query
- `ast(p: Path) → Ast` — (derived) query
- `my_query(a: u32, b: u32) → f32`  `(u32, u32)` as the key)


## Database


- Store all of salsa’s internal state
- Has to know all the queries that you will do
  - but you don’t list them all in one place
- `db.manifest()` — I get a `Manifest` returned to me
  - `db.ast(``"``foo.rs``"``)` — ast returned to me for `foo.rs`
    - if I invoke twice, I just get the result cloned (don’t recompute)
    - even once the inputs change, I may just get the result cloned
- `db.set_manifest(Manifest { .. })` —
  - triggers previous memoized values to be potentially invalidated


## Query groups


- “Salsa modules”


    #[salsa::query_group(InputsStorage)]
    pub trait Inputs {
      #[salsa::input] // `set_manifest` is auto-generated
      fn manifest(&self) -> Manifest;

      #[salsa::input] // `set_source_text` is auto-generated
      fn source_text(&self, name: String) -> String;
    }


    #[salsa::query_group(ParserStorage)]
    pub trait Parser: Inputs {
      fn ast(&self, name: String) -> Ast;

      fn whole_program_ast(&self) -> Ast;
    }

    fn ast(db: &dyn Parser, name: String) -> Ast {
      let source_text = db.source_text(name);

      // do the actual parser on `source_text`

      return ast;
    }

    fn whole_program_ast(db: &dyn Parser, name: String) -> Ast {
      let mut ast = Ast::default();
      for source_file in db.manifest() {
        let ast_source_file = db.ast(name);
        ast.extend(ast_source_file);
      }
      return ast;
    }


    #[salsa::query_group(TypeCheckerStorage)]
    pub trait TypeChecker: Parser {
      fn type_check(&self) -> Vec<Error>;
    }


    #[salsa::database(InputsStorage, ParserStorage, TypeCheckerStorage)]
    #[derive(Default)]
    struct MyDatabase {
      runtime: salsa::Runtime<MyDatabase>,
    }

    impl salsa::Database for MyDatabase {
      fn salsa_runtime(&self) -> &salsa::Runtime<MyDatabase> {
        &self.runtime
      }
    }

    fn main() {
      let mut db = MyDatabase::default();
      db.set_manifest(..);
      db.set_source_text(...);
      loop {
        db.type_check(); // will reuse results
        db.set_source_text(...); // edit
      }
    }

---

Hello, so I'd like to welcome you to this little video, we're gon na be talking about how salsa works and I'm gonna be doing this by working in this Dropbox paper document the URL for which I'll post with the video you can come later kind of Look at it if you like.

So, let's start with the first question: what is salsa anyway?

It's also the libraries for incremental recomputation and what I mean by that is basically imagine you have some function like we'll call it through various, and it has that's like two inputs and it produces an output.

These types they might be, and then the idea is well in some part of your code.

You invoke this function with one input and then later you invoke it again, but with slightly different set of inputs and ei.

Is this thing the PS change?

We would like to make this more efficient, we'd like to reuse some of the intermediate values that you did right in the first of all, and that's the basic idea of salsa is how to make this kind of automatic, and what I mean by that is, you Will reuse automatically?

I don't necessarily mean that you'll get really good news.

That requires really fishy to me, but we will be use automatically and sound or correct.

By that I mean you get the same result essentially as if we didn't do any, who used at all how's that so cool, that's what sells there is.

No, so salsas actual model doesn't isn't like a random function like through it's actually some easting and all much more complicated cases.

Right so imagine you have like an IDE.

In that case, you might have the inputs might be something like a manifest.

You can think of my card with optimal file.

The data inside that may be as well as the sources the source files in their text right will not be like.

Okay, food LRS has contents food on our essence and the outputs.

Well, there's a lot of possible outputs, but one might be the actual like binary executable between Mantovani, but you might also have things like what are the completions at this point or what is the type or what what should I show the user when the mouse is Hovering on this line that sort of thing so there's a kind of rich set of inputs and outputs, not a single function and that's that's kind of more the model that software is is working in right.

So, let's give it a kind of high level idea.

So how does it work?

Is that kind of high level view from house also works well you're going to identify, but the first thing is you: have you have to identify for yourself those inputs and outputs that you want right and when we well, rather, you identify the inputs later, like that, It'S probably the base inputs to your computation, and the main thing here is that they are not something which you compute.

There are something that you kind of gets at from the outside, you get given them, and then you have these things with a kind of derived values and derived values are basically some deterministic you're gon na give a deterministic and a pure function from the from the Inputs to that computes them right and this this function will take we'll start with some of the inputs and produce the outputs.

So these divided values they might be.

You know at the very limit they'll, be things like what are the completions at this point, but there might be intermediate steps, though.

Let me give you so if we come back to our up to our IDE example, if the inputs are things like the source text, the manifest what's the source cuts and some derived values might be like for each source file X, you might have a derived value.

The ast for X, right and so actually we're gon na stress the manifest here is like think of these, like a label and maybe some some inputs right, so the source fax might be like kind of giving the source text for some path X and the manifest Just to keep it simple, maybe the manifest actually gives you back like a vector of of paths and the source text would be like path X.

I look give you back a string, that's they that's kind of how these are gon na.

Look for me when she get to meet with us, so our ast might be something like X, the same path and it's going to give us an ast whatever that is, and so an ast is not like a final output.

That'S the meaning!

It'S an intermediate value that we could use right to do our computation and eventually, we'll have something like the completion at line number, and this would be like this would be like if you what what the IDE should show and so what's going to happen, is us When you go to execute, say what is the completion? We'Re gon na see that it's going to compute variants derived values as if those intermediate values and eventually read from the inputs to produce the result and also the framework is going to track.

But in the course of computing say the ast for a given path will track, which inputs did we access, which derived values and we'll use that later, to figure out what should happen when the inputs change right.

So when an input changes, you can say these values.

These derived values are still valid because the inputs that they touch changed.

Well, maybe these drive values may be invalid because some of their inputs change and we'll see that actually we're a little bit smarter than this might suggest.

So it doesn't just figure out everything.

That'S sort of downstream from a change.

What does that would usually be quite a lot of your computation? We can actually do somewhat better than that um, no, but but basically that the model you should.

I think mono useful to have in your head is there's the kind of graph, so we might have.

We have our inputs on one side right.

So let's say something like this and then these are.

These are like.

We call these queries.

These even know it's in the back and then when you have a derived query, so something like AST edges this way, these become also nodes in there.

So what we're saying here is when I'm computing the ast for a given path.

I need to know the I have to use the source text to do it and then on both the parser I may be computing.

The ast is just running the parser on one file.

It doesn't need any other input than the source tags um, but when, but how do we know so the whole? This is all in the context of some compilation like so somewhere.

There'Ll be some sort of some root thing that we're trying to do, and this is kind of that function.

Foo.

I was button in this compilation.

Maybe this is going to read from the manifest to start, and it's also going to read like you know, from each of these once it reads from the manifest it's going to go access.

We are gonna go, read the ast es of all the inputs right and so forth.

Actually, personally, a lot of other values between these two is to say that the role compilation is just a purse.

Maybe we can rename this to them.

First everything.

So now we have kind of our base inputs, the manifests the source text, so I stack.

These are our basic words.

We have some intermediate derived values and then we have this goal here: parse everything, but maybe we'll call it whole program ast.

This was fun and we'll say the whole program has to do all the assets for each file compacted and in order to conclude that we had to access the name Fest, we had to access the SD for each individual file and then we gave out of the Salt, well maybe this winds up being computed part.

I was like typing or something.

Actually, this wouldn't be a very good structure, and so now what? If we have this graph? If we see a change like the source x, then we can easily see okay.

If you change a dot RS, that's going to potentially affect the ISTE that will, in turn, potentially affect the whole program, which would mean type checking.

All these results are invalidated, but the AAS teams from the other files, they're just fun and one of the magic is actually stop with propagation a little bit early in something.

So imagine that, although we see that, like this roll path is potentially effective, this will pass here.

We might also see that after me run the part, so you end up with the same ASD we got the time before and so maybe all they did was add.

Some spaces and that doesn't affect the ASD, but then all those sorts of extremes, the ast scale is C.

So then we can wind up keeping these values because we say their direct influence did not change, even though some of their like indirect influence happened, but it didn't make any difference.

Well come back here, that's a pretty important thing.

So all right, let's go just a little bit.

I'M going to talk now about the idea of I'm sure.

I'M gon na show you how these salsa concepts for time factors so there's a theme te he contracted to keep in mind.

The first thing is something called a query and the query: we've actually been looking at queries at all times.

So when I write like manifest, this is an input there and when I write whole program ast.

This is a derived where both of these queries have a query.

Basically, some value that we access in the course about, maybe something you compute for a drive-through or something we didn't agree and queries in general have some number of keys.

Both of these have zero keys, but something like sois text that had one that had one gene, which was the path and all queries right, initially have a result site right, so they're kind of a function.

So this would be like a manifest.

This might be a st source text.

I think you said it's a string and yes, that's just for completeness, we'll throw in a sta also the AST.

These are either drive phrases, one input and one output.

You always have one output, you can have any number of engines, and so you don't have any good examples here, but you can have a query.

That'S like that takes two integers, and now it kind of has two keys or you can think of.

It is yes, one tuple as the key and the output here, let's say after we do.

Okay, that's what a query is, and the next key concept is something called the database go.

Salsa database is going to be a strut.

It'S basically the context for your entire computation.

It'S going to store all of salsas internal stings.

You can also store other things.

Whatever you want basically - and they should be a bit careful because you can mess something in the no result - sure in the goal statement.

But but it's going to store our salsa to internal state and it has essentially all the intermediate values for these different queries that we might wind up reusing when we compute all those intermediate values are things that we that are stored somewhere in this database.

But so, basically, the database is kind of like all the kind of the heap for your salsa composition, all the in all the data that you need now.

The only thing is just like you don't build your whole program in one giant file or function.

You don't build your database and of all at once, but it's gonna have to currently has all the internal.

It has to know all the queries that you do, but you don't have to miss them sort of all in one place.

Instead, what we do must be group queries what I'm calling query groups, they're kind of like salsa, modulus they're, basically a set of queries which which are defined together as a unit, and then you combine very groups before you do this welcome back to tricks, but I Want to just point out Monday's in the way you're going to interact and salsa.

When you have these queries, the database basically feels like a function of the business.

So, for example, if I have a database - and I invoke TB doesn't manifest - I got a manifest return to write and then the idea is - or if I broke your DBA s2 something then I'm gon na get the ast returned to me for a food RS and The idea of incremental recomputation is that these values are memorized.

If I invoke this place, I just get the result.

Cloned cannot be computed, at least by default.

The idea is even once the inputs change.

I may just hope this is a result.

It did not was not affected by the change didn't work.

I can just get it cloned, they don't have to so how do inputs change? Well, unlike when you have an input query, you also have a method, a set method that you can use where you kind of say: okay, I'm gon na set set the value of this input, and this triggers other memorized or previous memorized values, [ Music ], whose Needs okay, so let's come back to three keys, so we saw now how you're actually going to use the database.

But how are we going to define the database do that using prayer groups? The idea would be, let's use an example, we'll go back to our off to our IDE query, sir.

You might want to have, for example, a prayer group that is basically the for inputs will, despite evils that I've identified and to do that.

We just basically write a trade, what exactly starting point and we're going to apply the we're going to apply I'm gon na use these types we know he's actually would be spending a second and we're gon na apply this salsa.

Actually that growth was very good once again, this meaning second, and what this will do is it will it will kind of a decorator or I can you network it will process this tree.

It will produce that tree as output, but it will also produce a bunch of intermediate stuff.

That'S also uses, and one of those things is something called the storage struck, the name of which is given here, and this will see later we're going to use that later.

When we create the database to put all the pretty girls together and so that, basically the set of queries - and we reboot are just a set of methods in the street, all the methods have been done before and much like this one cell, and you can mark Them with special attributes, like salsa, employed to indicate that, in this case were saying, these are the base inputs.

Don'T you have for every input we're also going to have a setter as a member area trade, which is all of you so now we defined it very good.

I'M the key point of this.

This is like a set of standalone queries in this case are all integers, but that we can define and we could define another query group perhaps for the parsing lets them and it's gon na look much like the one you did before.

But here you might say function we said that were defining this ast query.

That'S it okay, but I would define this ast query and if you recall, I showed you before the ast query is gon na wind up when it executes that it's given the names of the past apart.

So it needs to get the source text for that.

In order to parse it, so you can make one query: you have access to another, just like we being traits important here.

I had a trait inputs, and now I say well: the trade parser extends inputs and this query ast is not in English ready to derive 3.

So we have to give the definition, and you can do that, just by writing a function.

You write it outside of compute engine and it's something like this.

This is just ordinary sort of it's just ordinary code, but so this is the function that when we do need to ask you for a file, we're gon na call this function and we're going to give it the database.

Now we don't know all the query groups that are part of this database, so what we get is actually some unknown type infiltrate to kind of say.

Well, this is some database that implements the price rate because it implements the parson trait.

We know it has an ast query.

We also know that it implements the input stream, which means it has a manifest and an input file where you can find as well.

So now we can do stuff, like you, can get the source text for an even input file, and actually i sorry i change means you can get the source text we're getting file by phone, he begot source site, and now we might like the actual pressure on Sorry, Spence um, so that's how you define query.

It'S awesome and also I want to point out this is using impulse rate in quick used in train, doesn't actually like using better because, as new copies of this function, we will generate the cord okay.

Now we have this source and we can keep building these layers as high as you want like, so maybe hmm, maybe we want to put this whole program.

This whole program AST, perhaps belongs with the parser kind of feels like the parser Benefield's, like a parser activity.

So you can put what it like so and then it might say, sort of or source file in DB down in efest.

Let'S say something up here: whatever so that you here would be we iterate from the manifest we create.

We get a get each source file.

We get the est of the source file by invoking the ast query, which we'll call this code, and then we want to put that into this master.

He has to be the female, and this is the whole protein.

That'S.

This is kind of how you define movie plays.

There are ways to take these functions if you don't want them to be in the same module the traders to find inside, but now.

Finally, then we can layer this up even further, so maybe our type checker.

We have this type chicken that might be kind of your rank, checker storage, so that that will build on the oppressor.

This returns a vector of errors.

That'S the basic idea! Now, when you're all done with this, eventually, you do have to define.

So this is how you define all the little modules.

So then you have to put them all together to meet your final thickness and the way that looks is that you make a start, and in that struct you annotate it with stuffing.

Is this invokes another grade and what you're gon na list here? This query takes as input all the storage for each very good.

So for every crater that you to put into this business, you have to have it smooth, and so you put them here and you have thousands one field, which is this also run time and you have to implement it's also a long time.

I'M sorry.

It'S also database treat for your database types.

Well, it might be the bids and implementing the straight is actually during Paul, has been far in terms of nine different methods is a single, simple method which returns to this.

Basically, how the so what happens here is inside this one time we're going to generate the storage for all of these query groups internally in the engine and your you own, that storage in the end, and you have to be able to give us a call.

There'S nothing um and that's how you defined your database.

Usually what I do personally is, I also do by default, so that then you might have a program which says like, but if me is my database invoked VD dot set manifest blah blah blah.

You got set device.

Ah so you kind of setup your inputs, and now I could invoke this fight, Chuck, Berry, alright, and then I could have a little loop.

That would like perhaps apply some edits and mean both type check a few times and each time we're going to reuse.

Whatever results you can from the previous execution ability.

Okay, so that's the basic overview awesome thanks for listening, I would say if you want to learn more possible, I doing some other overviews, but also you can complete the look.

Here'S where this also RS salsa there's some stuff, there's some instructions and, in particular, there's the kind of shows all the things I talked about.

But in someone looking to all the comments and tweets that IO very much