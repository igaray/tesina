# Salsa In More Depth (2019.01)

    #[salsa::query_group(InputsStorage)]
    pub trait MyQueryGroup {
      #[salsa::input] // `set_manifest` is auto-generated
      fn manifest(&self) -> Manifest;

      #[salsa::input] // `set_source_text` is auto-generated
      fn source_text(&self, name: String) -> String;

      fn ast(&self, name: String) -> Ast;

      fn whole_program_ast(&self) -> Ast;
    }

    fn ast(db: &dyn MyQueryGroup, name: String) -> Ast {
      let source_text: String = db.source_text(name);

      // do the actual parser on `source_text`

      return ast;
    }

    fn whole_program_ast(db: &dyn MyQueryGroup, name: String) -> Ast {
      let mut ast = Ast::default();
      for source_file in db.manifest() {
        let ast_source_file = db.ast(name);
        ast.extend(ast_source_file);
      }
      return ast;
    }



# Database storage
- Database has internally a shared storage with:
  - revision (R0)
  - one struct per query group
    - one map per query
- `db.set_input_file(``"``a.rs``"``,` `"``fn main() { }``"``)`
  - storing into the `input_file` map with the key `a.rs` and the value given
    - store the (new) revision R1
  - increment the revision (R1)
- `db.set_input_file("b.rs", "fn foo() { }")`
  - go to revision R2
  - setters are `&mut self`
- `db.type_check()` // `&self`


# Query storage
- Input query:
  - (Key) → (Value, Revision)


    db.source_text("a.rs"): // an input
      - record `source_text("a.rs")` as a dependency of the currently active query
      - look it in up the hash map and clone the result
        - and return the associated revision


- Derived query:
  - (Key) → (Value, Dependencies, verified_at: Revision, changed_at: Revision)


    db.ast("a.rs"):
      - check for cycle -- panic
      - record `ast("a.rs")` as a dependency of the currently active query
      - we will check to see if we have a memoized value already
        - check if the `verified_at` field is equal to the current revision
          - if so, we will clone that value and return it
        - for all dependency d we had before, is `changed(d) < verified_at`
          - if so, we can update `verified_at` to the current revision
      - if there is no memoized value:
        - push a fresh record onto the "Currently active query" stack
        - invoke the function `let v = ast(self, "a.rs".clone())`
          - track the dependencies as `ast` executes
        - pop off the record, extract the dependencies from it
          - dependencies (`Vec<DatabaseKey>`) would be `[source_text("a.rs")]`
          - track max changed revision
        - store `v` into the map with the key `a.rs`
          - value: v
          - dependencies
          - changed_at: R2
          - verified_at: R2



    impl<T> MyQueryGroup for T {
      fn ast(&self, a: str) {
        ... sometimes invoke the free fn `ast(self, key)` ..
      }
    }


    db.set_source_text("a.rs", "fn main() {}") // "a.rs" -> ("...", R1), now db is in R1
    db.set_source_text("b.rs", ..) // "b.rs" -> ("...", R2), now db is in R2
    db.ast("a.rs") // (verified_at: R2, changed_at: R1, deps: [source_text(a.rs)])
    db.set_source_text("b.rs", ..) // "b.rs" -> ("...", R3), now db is in R3
    db.ast("a.rs") // (verified_at: R3, changed_at: R1, deps: [source_text(a.rs)])
      // on entry:
      //   (verified_at: R2, changed_at: R1, deps: [source_text(a.rs)])
      // iterate over the dependencies:
      //   source_text(a.rs) -- the most recent version is R1
      // nothing changed that affects us, so we just update `verified_at` to R3
      //
      // (verified_at: R3, changed_at: R1, deps: [source_text(a.rs)])
    db.set_source_text("a.rs", "fn main() { }") // "a.rs" -> ("...", R4), now db is in R4
    db.ast("a.rs")
      // on entry:
      //   (verified_at: R3, changed_at: R1, deps: [source_text(a.rs)])
      // iterate over the dependencies:
      //   source_text(a.rs) -- the most recent version is R4 <-- did change since we last verified
      // let old_value = /* the old ast */
      // let new_value = re-execute the `ast` method
      // if old_value == new_value {
      //   update verified_at but leave changed_at alone
      //   (verified_at: R4, changed_at: R1, deps: [source_text(a.rs)])
      // } else {
      //   update both verified_at and changed_at to current revision
      // }



## second generation derived queries and backdating


    db.set_source_text("a.rs", "fn main() {}") // "a.rs" -> ("...", R1), now db is in R1
    db.set_source_text("b.rs", ..) // "b.rs" -> ("...", R2), now db is in R2
    db.whole_program_ast() // will execute (never invoked before):
      - db.manifest()
      - db.ast("a.rs")
      - db.ast("b.rs")
      - (verified_at: R2, changed_at: R2, deps: [manifest(), ast(a.rs), ast(b.rs)])

    db.set_source_text("a.rs", "fn main() { }") // "a.rs" -> ("...", R4), now db is in R4
    db.whole_program_ast() // can we reuse the result?
      - iterate over the deps list:
        - manifest(): this has not changed since R2, this is fine
        - ast(a.rs):
          - "have you changed since R2"?
          - look at its own input, determine they have changed
          - re-execute and produce a new ast
          - compare the old and new ast -- see that they are the same
          - "no, I have not changed" -- changed_at value is <= R2
        - ast(b.rs)
          - no input has changed since R2
          - trivially still valid
      - the value is still value, and we can just update `verified_at`


## power of backdating




## Strategies for re-use


    enum Ast {
      Module(Vec<Ast>)
    }

    enum Ast {
      Module(Vec<AstId>)
    }

    // AstId becomes effectively a "path"

    fn ast_for_id(AstId) -> Ast

    mod foo { // value for "foo" `vec![foo::Bar]`
      struct Bar { ... } // has id `foo::Bar` // value for foo::Bar` would be the fields
    }


    fn main() {
      let x = 22;
      println();
      x.foo();
    }
# Questions for the future


- How salsa works internally
  - ✅
- What makes a good key/value type
  - why we use `Seq` and `Text` instead of `Vec` and `String`
  - interning — what is that?
    - indexmap — indexable hashmap
    - gives us an integer that represents the larger structure
- Strategies for re-use
- On-demand thinking
- Parallel patterns
- Cancellation

---

So this is a video about salsa and in particular, going just a little more in depth than the introductory video on house also works.

The goal is to kind of give you an understanding of what salsas doing when you actually invoke a query so that you can use it more effectively and here to ask make sure that I keep me honest.

Let's say this Jonathan Turner and the idea is when I say really confusing things you can tell me that they I need to explain more, so I think he falls in that that point of a position right now, if a salsa user who doesn't know how the User moles work, so how does also work internally?

So let's say: let's set the scene.

Let'S imagine you have like a salsa query group, and so I'm trait, I'm just gon na go grab an example here from the prior paper.

Suppose we have this this parser trait and I guess we need the inputs too.

So, let's review it.

So what we've got is we have.

We have two Curie groups, but I'm going to put I'm just going to collapse this into one, because the distinction isn't important for house also works internally.

So we have two inputs a manifest in a source text and we have two derived queries: the ast and the whole program ast and for each of those we're going to have some function which defines how they work.

The AST function is going to start by invoking the source text to get a string out, and so this source text is a string, because that's how the query is defined right here and it's going to do some actual work on the source text and return it And the whole program ast is going to iterate and invoke the ast query each time for a given name and combine them in some way and is there something else?

Oh no! Oh.

I know this just just for reference.

This is what the database looks like: you'll you'll define a database and there's probably some loop here where I instantiate the database.

I set the inputs, I invoke type check and I set the text.

So let's start here and walk through just a little bit of detail.

Just on this first line in my database default, you know what's happening here so the way when the idea, when salsa executes it needs to have it needs to keep track of the values that resulted from all the previous, like queries that you did in the prior Execution, that's basically how we avoid redoing work is by remembering like what was the ast of this file in the first iteration and so forth, and all of that data winds up being stored inside this also run time, and that and it's that's what these storage names Here are basically doing that you attach to the database, is that's it's gon na keep those strux or keep values associated with those trucks that has like a hash map, essentially for every query that you have to execute all right.

So me good yep for the for each of those they see yeah, some infant storage, parser, storage and pet record storage.

Are those defined do we did we define those them earlier?

Yes, so each of these corresponds to one query, group and they're defined by the query group there.

This is an actual name of a struct, and this struct is generated by this decorator sort of attribute macro and what the struct has is for each of the queries.

In that query group it has the actual storage so by in directing through this struct, like you, don't name the individual queries here, you name the struct and inside the struct, or is it crazy?

So this the struct for type checker storage, is gonna have one hash map for the tech jet query, but the struct for parser is going to have two one for the ast and one for whole program, the ast and so forth and they're slightly different.

So for the inputs, there will also be a kind of hash map, but it's a slightly different wrapper around it in puts work a little differently, but basically per each query.

You have the storage, which is this is a map and so that all gets concatenated into the database and when you call something like set manifest, what's happening under the scenes, is that let's say we we call it with m1 or something what's happening under the scenes?

Is that that data is getting stored into the map and it's because of the name set underscore manifest right?

I think that that made my special in some way for each input for every salsa input that you define.

There'S a setter automatically generated.

Oh yeah, I see the comment now you know so that you can't call set ASD, because it's not input you can only ask for the ast and it gets computed um.

So right.

So look so! Okay, this isn't really relevant um.

These might be the topics for future salsa discussions, but the so that's already taken a few notes here so like the database storage.

So the database has, as I said internally a shared storage with kind of one struct per query group, which has any the structs have one field for a query.

One hash map, basically all the cost kata map, because it is a hash map, but it's also other stuff one map per query and when you do DB dot set something.

What you do is you basically, let's say on: let's make it more concrete.

If I set input file, you know a dot RS, I don't know a little little rest source text there, then, what I'm basically doing is storing into the input file map with the key.

A dot RS and the value given right, I'm also doing one other thing is the database has internally a revision which is really just a counter.

It starts out at r0.

You know when I do that.

Every time I call set I'm going to increment the revision.

So in this case I would go to r1 and we're gon na see later that we need this revision so to remember.

Basically, when did we compute for derive queries and things we want to remember um.

When did this last change essentially, and so actually, when I store into this, I don't just put their value given.

I also store the revision r1, the new revision when I set an input.

So I'm remembering now this value was last changed as we entered revision one right and we're gon na up the revision.

Each time we do a set.

So when we set the manifest or when we said, if we called set input file twice, we would go to revision twice.

Revision to just out of curiosity revisions makes me think of databases and databases, make me think of things like transactions, so each one of these calls is kind of a its own transaction.

You can group them into a single one.

Just that's right, that's right! So we guarantee that when you, when you execute a derived query like a type check, query which we'll get to in a second that's gon na, do a whole bunch of work but um.

While it's executing that's like a transaction, there can't be any sets that come in between and actually you you kind of get that guarantee to a certain extent for free through rusts system, because this is an an self method.

Shared so has a shared reference, which means that you can't get a mutable reference at the same time and these set methods, the setters, are a mute self, so they can't kind of overlap with vb.n et check, but that's also really important for our algorithm.

Basically, it would be really confusing if, in the middle of executing the revision was changing and you even have a sort of explicit mechanism which I didn't talk about in the other.

I didn't talk about it and I won't talk about in great depth, but given a DB handle, I think the method is called freeze.

I forget, you can get a second handle or I think I call this snapshot, and this second handle basically lets you do.

A number of you can use it just like a database, but you can't do any set operations and it's effectively as it keeps the database in a frozen state.

While it exists, and then it's really meant to be sent to another thread and processed in parallel.

And it's exactly this idea that, while that thread is executing, it does not want it wants to execute atomically with respect to the database and there's just me clear, there's no way of saying start transaction and do these five minutes and then stop transaction like you're.

Just assumed to do one edit, edit, that's right and edits can only happen on the main thread.

Anyway, you can't go, you can't clone the DB.

You only snapshot it which freezes it so there's sort of no need for a transaction, because there could be nothing that intervenes so all right.

So that's the database concept um.

Let'S look a little bit more at this query.

Storage.

So I mentioned the for an input.

Query effectively, you have a map like this um.

It says, given a key I'll, give you a value and a revision which is when the value was set for a derived query.

We have there's actually a bunch of options here, but I'm just going to explain like the main, a normal one and I'll probably forget something we may.

We may come back to this, but this is the basic idea.

What is this verified at and changed at?

So we are actually two revisions, so a derived value would be something that we compute by doing a by running a function right and uh.

I'Ll come back to this struct with like the details of this, but let's, let's first just sort of walk through.

If you called, let's say DB ast for some file, what's going to happen here at a high level, is we will check to see if we have a memorized value already and if so, we will clone that value and return.

It that's for people that haven't heard the term memo is, but what right so basically did we already execute this query and if so, we will stash a copy of the results.

That'S the memorized value.

It'S like you wrote a little memo to yourself with what the result was then, the next time you use it, you can just clone it, and you already, you can kind of see here something important, which is that the type really matters.

So I I define these queries as returning an ast and if that were some big struck, that's kind of expensive to clone, like maybe it has deep ownership of all of its.

So then, that's not so great, because every time we invoke DB ast we're gon na clone it we've got enough, we've always got.

We always have at least one clone because we're keeping the old value here.

So so what you really want there is to have - maybe maybe you want to put it in an arc or something like that?

I'M not gon na worry about that right now.

What similarly string and so forth might not be the best choices um.

You really want keys and values to be cheaply, cloneable so about any case, we'll check to see if those memoirs value, if there is no memorized value.

This is like the basic algorithm.

I guess, then we invoke the function DB or Ras tdv8 other.

So essentially, here we're gon na call the function that you defined, we'll see it in a second we'll give it the database we'll give it the keys.

Sometimes there's more keys right now, there's only one, and then we will take the return value.

So this would be like let's take the return, value or store V into the map, with note that here, when we do this, we're actually going to clone the keys too.

So that's why I say both should be cloneable well store V into the map and well well put the ignore I'm going to ignore the dependencies part for a second we're gon na put the value you know with the key a dot RS.

The value is, the value is V and the verify that or the change that is going to be the current revision, whatever that is, and so we'll have verify, and what that's basically saying is the last time we computed this was we remember what the revision was.

So let's say it's r1 for now or r2.

I guess because we said we called two setters um, so we went to a revision r2 here now.

We know that in r2 we computed this ast, and this was what it looked like and I think that's a really great high level overview of what's happening, but maybe we can talk a little bit about that is function like what is it returning from that?

That function, that's going into V that we're then snoring into the map later right, so the AST function.

That'S that's literally.

This function that the user defined here.

So it's returning whatever this function returns and because we're sort of generating all this code with macros and things going well or generics and so forth as necessary.

The types all line up, um and write so that once we call here, this is really the users code.

Yes, we don't you're, not do anything um, so that would call that would internally if we walk through what that's going to do.

In this case, it's gon na call DB source text so we've given it our database right here as one of its parameters.

Um, and actually I wrote here - I wrote database, but I'm gon na write itself, because this is effectively that this is like when this is the definition for the actual method like, in other words, we're kind of defining the name of this trait.

Oh, I was willing consistent, we'll just call it example.

Do it a very good name?

My query group we're basically defining this.

What we're saying here this algorithm.

This is essentially what what the database actually gon na do when you call AST oh, sometimes invoke so right.

So we go back here and here we see that in while we're running the users code, it actually calls source text right and source text is an input, so inputs behave a little differently when you, when you invoke an input, what happens it's much simpler?

Essentially, you just look it up in the hashmap, amateur and clone the result.

But again you see we're cloning, so it's important it might be too expensive, but so when we call DB dot source text, that's that's literally just a hash name, but look up with a little a little bit of stuff around it that we'll get to in a Second, but that's where we get the string from and that the hash map is again returning, not just the value, but the revision and the value William.

Well, yeah, that's right internally! It is returning the boat both of them, but the revision doesn't make it all the way to the user right.

We kind of intercept that um and right.

So, okay, so if you didn't have any revision tracking, this is actually more or less how the system works, and you see that now you can sort of see what the memo ice happens, because if we call DB ast twice the first time we invoke the function.

The second time we have a value, so we can just return it.

The only thing that I would add on to this is that's kind of important is actually before we do any of this.

We check for a cycle, and that means that while you're computing DB or the ast for a given file, you can't recursively ask for the ast for that file because we don't know what to give you essentially um, and in that case we panic so you're.

Really not supposed to to do that.

You have to kind of set up your query, so they don't cycle and in in Russy itself.

These panics get we kind of capture the stack, trace and print them out to the user as user errors, and your source code is messed up in some cases.

So right, but now we can kind of start to add the revision tracking.

I guess so the idea.

What the revision tracking is when we next call, if we, when we next call a setter, it's going to up the revision, and so now, when we look at this memorized value, we really want to check check if the verified at field is equal to the current Revision, if so, we can return it.

So now we said was this basically was this verified out is telling us is this value?

Do we know that it's correct give in this revision, given the state of the inputs in this revision right?

So, if that's the current revision, then we're done and that's still going to work the same for the memorizing, but if it's not if it's older, like that's it, that means that an input has changed.

Since this revision was done, then we want to see like basically figure out if we can reuse the value or not that's what the into a bit um to do that we have to go back one step.

While we were actually computing the value, we were doing a little bit more behind-the-scenes.

We were also tracking what queries you did and recording them in a step, basically recording them as the dependencies.

So the way we do, that tracking is well first of all, so like in this case.

What that would mean for ast is we would get back a result.

Movie yeah, we'll get back to things here, we'll get back the value and we'll get back the dependencies and the dependencies would be like a vector in this case the.

What did we call source text so we have this concept, which is you don't directly interact with it, but it's called the database key and basically what it is.

If the query key, if the key for a query is just this string, that arguments to the query basically and the database key is kind of the pair of the query name and all of its arguments, so it kind of uniquely identifies one bit of computation and So-So a dependencies, the dependencies list is just it's like a vector of database keys, and in this case we would have okay in the course of computing, the ast we accessed the source text and we'll store that, along with the value and this kind of works, because When we're invoking the query and that query invokes another query because of the magic of macros we can, we have some visibility into.

What is like that second query: we can see that that's fitting in both by the first one yeah.

So let me it's not really a magic of macros per se, but we have in the database.

We know what is the currently active query.

We actually know the whole stack, which is how we check for a cycle in the first place.

So what we can do is the first thing we do when you enter into any operation.

Is we record this as a dependency of the currently active query so like also here we would record so for this is let's say this is DB choice text.

We would record source text as a dependency and similarly, finally, here we have to kind of push on to the currently active query stack the push afresh sort of afresh record on to the currently active query, step um, and here we would pop it off.

That'S kind of actually how we get the dependencies out as we pop off the record extract the dependencies from it, and so really it's not returned to us.

It'S it's something we recorded while we've Wally st was executed and right, so that we have that we also track one other thing as we go, which is we track the the maximum revision at which any of the things we did changed.

So we mentioned that in the source text we're going to clone the result and return the associated revision so track the maximum changed position.

So, for example, when we call a ste well when we in this case we're only we're only doing one thing, so the maximum revision would just be whatever.

Basically, whatever revision the source text changed in we'll bring that back with us um, and we use that later.

So, okay, so now we have information.

Now we can come back to figuring out if we can reuse the value.

Now we have a list of dependencies that we - and this is basically all that we assume that you're derived query is a purely deterministic function and that's kind of on you to get correct.

But we assume that it's a function, that if you give it exactly the same inputs, then it will do exactly the same thing right, and that means, and by inputs here I don't mean just the inputs to the salsa database as a whole like source text.

I really mean all the queries that it invokes are kind of its inputs right, so here there's an input, dbi source text, but for the whole program, ast query manifest as an input, but so is sort of the result of this recursive call.

So we assume that it's deterministic and therefore we assume, if none of the inputs have changed in this revision, then the result must not have changed in this revision either and we don't need to re-execute it.

So what we can do is say something like for each dependency we had before.

Let'S call it D, did D change or like look at the is the changed at of D, which is the last provision where this thing changed greater than or equal to the current revision.

I guess it can't ever be greater than equal to the Cruz and, if so or yeah, it's so break.

I don't know that's kind of annoying.

Let'S look at it more like this, for all the dependencies is changed at of D, less than the current revision, more or less, not quite right, but that's the idea.

I probably mean the verified app and if so we can update verified at to the current revision.

I probably this is maybe a bit more detail.

I'M sure I'm getting some of the exact logic here a little bit wrong, but the basic idea is we look at those we know when those values changed and we can go over them and see.

Have they changed in the current revision or not and or since the law, how they change?

Since the last time we computed this value basically, and if they haven't, then we can just say well, the value is still good and we can change the verified at field to say well, at least in this revision.

It was up to date right and the next time through.

We don't have to redo this work again, but the change that we don't change because it didn't change its value.

It'S it never changed it in revision.

In the new revision, it's still the same value.

It was before so it's like an example.

I think that'll help a little bit.

So if we say like we do DB dot set source text or whatever I called it, I think I called it source text, something this puts us in r1.

Then let's say we set source text here at a beam.

It puts us in r2 and then we invoke the parser on a that's going to give us a record that says you know I was verified at r2 and I changed at r2 and of course my dependencies is just a dot RS and then suppose that I Set the source for B again and now, I'm in r3 - and I really in puts for this function - have changed in the new revision.

So when we ask I'll just go ahead and put the dependencies list when we ask when we go look at source text for a dot RS and we asked when did it change we're gon na get back our or yeah?

I'M sorry our one function we'll get back our one, because that's the last time a new value was set, and so we'll say: okay, we can just.

We can just change this to our three um.

We can leave changed at the way it was.

We leave the dependencies the way it was because sort of if we had we executed it, we would have gotten the same thing we got in or that's fine and I forget actually it may it may be that we actually store our one year, because I mentioned We compute the maximum of all of our inputs.

When did they change?

I forget exactly how we do that, but either one would be correct, at least for the algorithm.

I think it doesn't really matter as long as it's uh as long as it's you know recorded at this point, because we only care about the future.

But the point is, we know, go on jump in.

Maybe I can repeat back to you what I'm hearing you say and we can kind of double-check that I got it some part, so we set tour specs on a RS and source tags.

A RS is kind of that special key that special key.

That will allow us to know if we've run this before in the past.

Alright, so we remember that we've done that.

We'Ve set it at revision, one, that's that's the r1 or a Don RS mm-hmm.

Then we do sense, we're specs or B dot.  Rs.

That'S a different file and a different key as a result, so because that screen is different, we now have sourced XB RS in the database as well and that's an r2.

We don't touch.

We don't touch me the previous tenants that we made at that point.

So a non RS would still be a more one, even though now b, dot RS is still alert yep.

I guess to change that.

So when we do the lion, 3db, dot ASP.

So now we're going to do a query: we're not setting anything we're just queering in back out when we create a dot RS.

We know to go look for its dependencies to be able to answer that.

We get that because, as the query runs, where we're basically logging the steps and it's picking to answer that, yes, okay, so at this time through we watch it run because it's not cached yet so we actually run the SQ function we step through.

We see it call the source, Bex query and when it does, we set that into the dependencies.

It finishes running and gives us a value, and we know that we verify this at the most recent, which is our two, because everything just came out fresh and they changed is the most recent.

For that.

For the query, I guess the camera which we called Oh max chain provision, was what you call it so the match change revision would be our one in this case, because everything that a dot RS means is in the first provision mm-hmm.

Does that sound right?

So far, yep I'm trying to like note down some of the things you said yeah.

That sounds exactly right.

So right, oh yeah, so maybe we can keep going.

That sounds good so far, right, I'm pretty sure yeah exactly so change.

That r1 is basically that's the last time.

Any input changed right.

So therefore that must be.

If we went back in time into revision 1 and we execute it, we should get the same result because we don't get any other things that changed after that, and then we do a sense or specs.

On line 4, we do a sensory specs, with B dot.

Rs again, so this is the first time we're reusing one of our keys.

Yes, he was are too, but now that we're editing it again.

That'S at our 3/10 on line 5, you have a SD, a dot RS.

So now we're four rerunning that query.

We update the, I guess we update the verify because we're we're saying: okay, we're in r3, I'm checking everything again an r3 8.

This is where we're at everything's still r1.

We don't have to rerun any of the query.

None of the dependencies change at this point.

So we can just give you the cash value or the memorized values, yep and so to walk through this just to touch more detail on entry.

We will have this value in particular.

This is out of date because the current the current revision is r3, but we saw that this was last verified in r2, so it might, there might be a problem, we don't know yet, and then we can iterate over the dependencies and basically ask them right when They last changed, and in this case this is exactly one input dependency.

So it's very easy to tell when it last change.

We just look it up in the hash map and so the most recent version is r1, and so we conclude, then that nothing changed that affects us.

So we just update verify that the current revision, that's the last time we figured, we checked it yeah.

So yeah, that's the idea so far, there's one twist we haven't gotten to yet one of it.

Well, there's two things we didn't talk about that are relevant.

I think the first one is.

I only showed you when you have a direct dependency, which is an input.

That'S very easy in the case where the dependency is not an input, but rather a derived thing.

Then we sort of have to recursively do this procedure.

So let's say the whole program ast invokes that invokes one of its ast dependencies.

We want to find out if it's up to date in this revision and it's the same basic idea, but it's a slightly different variation on it so before we actually dive into that which sounds really interesting, there's um!  Maybe we can take this example and go a little bit further in this particular example.

So we can see the interaction between the the memorizing and the edits.

So right now we're we're editing, feed on RS and that's not affecting eight others.

We still get the same annoys version of eight on RS, but if we set the source specs of eight on RS, you know after this block yep yeah.

Actually I think I think that you are totally correct.

We should continue with this example, and the thing I was going to say you said sounds very interesting.

I think it actually isn't that interesting.

So we'll come back to it baby, it's basically a variation on a theme, but let's, let's look at this instead suppose suppose that I call now a set source text on a dot RS, but where I had function main with an empty body.

I now change it.

Oops I lost my.

Where am I Here? I am.

I now change it to have a space in the body right.

It'S not a particularly important edit.

It won't affect the parser at all, but we don't really know that.

So what we're gon na do is we're.

Gon na remap this to r4 because now we're in revision 4 and now, if I reinvest e, we have a problem, we'll we'll have a sort of a problem in a sense.

We'Re gon na see that indeed this should now be r3.

I suppose so we see that it was verified in r3 - that's not our 4! So it's out of date, so we have to check we iterate over our dependencies and we find that hey.

This actually did change yeah.

You know like, since we last checked so now we have a problem: we're going to re-execute the AST method, but we're gon na do one last twist that I didn't we didn't write about in the algorithm.

Yet what we're gon na do is we're gon na hold on to the old value that we had before the old ASD and we're gon na get now the new value by re-executing, and then we can do a check and we can say what happened it'd.

We actually, even though the inputs changed, did that result in a change in the return value or not, because if it didn't change the return value, then nobody who is invoking us really cares.

Essentially the yeah.

It'S not it's, not that's assuming that again, this deterministic nature.

So what we can do is say: Oh in that case, update, verified at but leave change that alone and if they did actually change, then we have to make a new record where update both verified at and changed after to current revisions.

So in this particular case, since we didn't change anything that will affect the parsed result, we actually wind up with verified at our for, but we leave change that as our one we kind of back dated our result.

Even though we read something which changed in our for, we can observe that the result is the same as it was in our one, so we can leave it alone um.

So what is this is helped us do if we kind of back it like that yeah.

It doesn't help us do anything in this example.

So far, I guess we still had to re-execute the ast method right, so we still did all the work.

However, if we go to the next level up - and we invoke what did I call - it - hold program - AST old program AST and let's assume that we actually, we also invoked it.

You know earlier like here um, and we got some st actually i'm gon na do is, for the sake of the historical record, i'm going to take this and copy it, because it's kind of a useful artifact, just as it is and then start injecting the whole Program ast into this pose that so this is what is this like second-generation derived queries, so suppose that we invoked db2 whole program ast there before this last edit right and we got some some results.

Some result W then now, when we invoke - and let's say we didn't invoke DBS, he directly Roxy, okay, let's just make this a little more realistic.

We set the source text, we invoked DB whole program ast, and this, in turn, invokes debate like internally invokes.

Eb is theta, RS and beat out of us.

It also invokes the manifest, as it happens, and all this other stuff happens that we said before and we'll basically wind up with now something like verified at our.

I guess changed at, I think r2 and we have our dependencies list, which is all the other queries.

So the dependencies list will the ast of a dell RS there's to be done.

Rs and o ordering is important here.

I don't want it I'll.

Just say that and leave it mysteriously unexplained, but we have to record them in the order that they occurred, because otherwise that, for reasons actually an interesting point, so the the manifest is ers ers.

We know from earlier in the conversation that ast also depends on V source text, but we don't we're not finding all the dependencies say: hey s need into this list right.

That'S right! It'S a shallow list!  That'S sort of a STRs is problem.

What its dependencies were.

Um yeah yeah, so now we change the source text again, but all we did was add a space and weary invoke all program ast and we have this question: can we reuse?

Can we reuse the results or not, and what we're gon na do is we're going to go over the dependencies list and we'll look so like for th

e man first query: this has not changed.

Well, actually, we never said it.

It also seen, I think that will actually cause salsa to panic.

If you read input you never set, but let's assume it was in r0 or something this the point is it hasn't changed since r2, so this is fine right.

This is like exactly the case.

We saw it with source text above, but now we get to Asda dot, RS and we're gon na recursively.

Ask it because it's a derived query we're gon na sort of ask it.

Have you changed since our to write and it's going to do the process?

We just talked about it, we'll look at look at its own inputs that will determine that they have to temp and it'll, determine that they have changed.

It will re, execute and produce a new ast.

It will compare the old and new ast and it will see that they are the same, so it's gon na leave.

Therefore it's going to be able to say no.

I have not changed because the changed at value is less than our less than or equal to R 2, because it was able to backdate if it hadn't, backdated, it would have had to say.

Yes, i may have changed right.

A more conservative result if we didn't have the old value to compare against, for example, and in the case of ASD Beadle RS.

No input has changed since r2 because we only set a dot RS, not B, and therefore this is trivially still valid trivial.

I don't know, but that's the base case kind of, and so in the end we determine the value is still valid and we can just update verify that and we never have to re execute and that's really nice, because actually building the whole program.

Asd is perhaps no significantly more expensive than just any one piece of it I mean, so we were able to keep the end result that we all that we really care about.

That'S that's basically, the whole algorithm in all of its sides - and the one thing I would mention is that you can tweak this and salsa offers some knobs for this.

I would like to more so you could imagine, for example, that some queries might not keep the old value, in which case they have to be more conservative because they don't have.

They can't do this back dating trick and similarly, some queries might keep just a hash of the old value and not the actual value itself, in which case they can do the back dating.

If you assume it's like a cryptographic hash that you trust, but you can't you can sort of backdate yourself.

But if someone directly invokes you, you don't actually have a value to return, so you still have to execute.

So it's kind of these in between points where you might be able to save like we might be able to reuse the whole program AST.

But if we find, if we work we're only keeping hashes of the ast, then you know if we do have two real executes the whole program.

You see, we have two reparse everything, but we might be able to reuse a final end result.

So there's a bunch of knobs, you can show yeah, that's that's kind of it.

Questions on this last part.

That sounds.

That sounds good, so this is kind of like how we we can make our edits and then we can, as we develop a more sophisticated.

So the dependence views we can be smarter about how we're caching whole whole parts of that graph or whole parts of that tree.

So we don't rerun a very deep set of dependencies and queries on top of queries.

That'S right.

I had a note here for for posterity.

Yep no are we done?

Maybe were done?

No, so I stopped you before you got to another thing, so we were talking about that original example and I said well, let's do some edits on or into another query on, say, V dot, RS and you were about to take us into a different direction about Different kinds of queries, I think I think I was going to talk about this procedure where one of your dependencies is itself a derived query and I kind of hand waved over it.

But basically I asked this question: have you changed?

There are sort of two fundamental things that a query has to be able to do.

It has to be able to give you a result that is up-to-date and it has to be able to tell you if it has changed and they're like similar, but slightly ever so slightly different and they're.

The reason that they're ever so slightly different is exactly that.

You might be able to figure out that you have not changed.

Even if you don't know what your value is, as I was kind of saying, because you can see that none of your inputs have changed, but so there's actually two kind of branches of the code.

For handling these two cases for every query, we generate two methods and in what, in some cases like here, actually, when you ask has it changed if it finds that an input has changed, it will actually invoke the other method, the we execute method, so they kind Of invoke each other back and forth because producing a value has to check if the inputs have changed and then checking if the inputs have changed, sometimes has to produce value but yeah.

That'S the idea.

Okay, so we've been talking about using strings as keys and strings as values may be helpful to talk about different kinds of data types that you can put into the database and query back out.

There are some good practices there.

Yeah all right got a little time we'll go through our list check.

Okay, so we did this right.

So what makes a good key value type, so the short version is - or I mentioned, that we have to do a lot of cloning, so really vexing strings and stuff are possibly not a good choice.

Unless you know that they're going to be small you what we use for example in mark, is this seek and text type switch.

This is supposed to be sequence and that's supposed to be text.

They are kind of ref counted versions of Veck and string, and you can do some sub slicing they're sort of very simple ropes.

I wouldn't really call them ropes.

I guess a rope would be a potentially a good choice or an immutable data structure.

I think there will probably be some experimentation around this to figure out the right choices.

Another kind of thing you can do is interning, which we also do in lark, which then sort of produces integers, which are very good keys, but there's that brings its own complications.

That I don't want to get into at the moment like how to how and if to garbage, collect the internals and so on, but just for just for people watching so have you did you explain and turning on the other video by chance, I did not.

We can just gon na.

Do it quick?

What is what is interning and training who's been?

Very that's sound, scary, yeah, so interning is taking a risk.

It'S basically when you have a canonical pool like a hashmap um.

So for a given value, you store it in the map and you associate it with some integer and then you can just pass the integer around and that's a very cheap thing to pass around it's kind of like a pointer in its own way.

But when you later want to read what the value is, you can use the integer to get it out from the from the hash map and a particularly good hash map, for this is the index map trait, which is sort of a combination of a hash map And a vector, so it can give you an index back out that you can then use to index directly in that's one version of interning anyway.

So we take a complicated structure, is stick, it say, and the database or into uh and the index map.

And then we get out just an integer value and we can pass around the integer value and we can reference.

We can reference this larger structure just by this vintage, your values, and we only need to pull it that big structure back out again when we actually need to have it in our hands.

Otherwise we can just kind of refer to it by this integer.

Most of the time, the flesh is fine, yeah and that actually bleeds a little bit into the strategies for reuse.

I think merits a bigger discussion, but I would just say that in general, when you're setting up these sorts of queries, like we've shown here, you you often will want to introduce some kind of indirection like let's say you have a module in your compiler, at least It has like a list of items in it.

You could make a module.

You could make a data structure, that's like, let's call it.

The enum AST you might have.

The module might have the vector of ast nodes directly embedded within it, but you might be better off if you can finding a way to make the module have a vector of ID's and having a separate query.

That'S like given an ID give me the ast and the reason that you would want to do.

That is that maybe there are changes inside the ast, but like the the list of ID's, doesn't change essentially, so the module level looks the same, even if some of its contents changed and that way you can get finer-grained reuse, because only those derived queries that actually Had to access the ast for a given ID care if the ast changed so often with recursive structures, you'll want to set up a set it up.

This way it's awesome so in the database you might have like set the ID to an updated ast and ID doesn't change, but the let's say you had new parts of the structure behind that the new ASP has has some more data in it.

Right people keep using that that ID and everything that uses that ID none of those queries become invalidated because that ID doesn't change right.

So so I would like to make this a concrete if we assume that the ID is some kind of path, let's say, and it could literally be a maybe even an arch path or something.

Then then, this the value for foo the module foo, would be like a vector of this path.

Foo bar right and the value for a foo bar would be the fields, and so now, if they changed, if you change the fields, a bar, you don't actually change the value for a foo at all, so yeah.

I I this stuff can get a little tricky and complicated.

That'S why I think it's.

This is the idea of what you want, but actually realizing like this.

So I mean - and it's worth saying I mean some of this - is that salsas so new?

You know we're still learning how to use it to its fullest potential.

Some of the tricks that we're trying out may not be good later and some will learn new tricks as we go.

This is just kind of like a snapshot of where we're at yeah.

Look.

That'S right, I think the only thing in this list that I feel like might we could talk about real briefly may be cancellation, so we talked about parallel patterns already actually but yeah, so the on-demand thinking.

I think it's good to drill into that a little bit, especially for people coming to salsa and thinking about salsa as a user.

Knowing that you should think about your query in these stages, just like Nico was showing earlier, where you have a query and that query calls other queries that allows you to kind of cache things and do them and steps to think about.

Rather than pulling large bits of data of the database for one shot to have it in that stage allows you to like there's my key staying on demand.

Thinking like pull on the data up to the point that you need and then stop yeah.

So I think one of the challenges that I found with the on-demand stuff is that at least saying compilers.

You often have something where it's like in order to type check something you there's a certain amount of context that you need like.

Maybe I I can.

I could, for example, pull out an expression.

You know.

Isolation like I have some function like this and they have let x equals 22 or something something here I could I could.

I could maybe pull out this expression in isolation, but I can't really do anything with until I know what the type of X is and that's determined by these other statements.

So what what you often end up with is this: is this the structure of queries where you start out very local and you kind of branch out to get your context.

You'Ll have some outer queries that they will parse the whole file, maybe and give you like?

The set of names that are defined or something they try to do the minimal amount of work there very shallow kind of like we show here they just leave pointers for how to continue.

If you care about the details of this thing enough that you can, then then you can drill in on just the parts that you actually need, so maybe you would find out.

Oh the name X.

I have a map that tells me where it's defined.

I can find its initializer and therefore I'm just dependent on that, but if there are other statements in here like a print'ln, I never wind up asking for the details about this, and so it then I'm insulated from changes that might affect it.

That'S the idea, but the practice like I said I think that's really getting into that and showing a good examples of it would would be a good topic for a future and there's bit of an art to it.

Breaking up your problem so that it's in these really good stages for your from what makes sense for your project right.

One thing I want to add - or I want to come because it's directly relevant to what we said is that the one of the nice things that back the back dating technique lets you do basically is be a bit sloppy here, so you can have.

We showed an example where we had the source file here and it's a very crude, very imprecise right.

It doesn't like break the source text into chunks or anything, so every edit is gon na change.

This key, which means every edit, is going to rebuild the ast, but that's maybe not so expensive right and then we find out that in fact, later parts of the program are insulated by this.

The fact that the ASC doesn't ASC doesn't actually change on every edit and you might have even more finer grained queries like item names, which are which reads the ast.

It produces a list of names of items in the list right or something, and now the key point is even if the ast changes, the names of the of the items to find probably for a lot of edits are not so adding a field to a struct.

Won'T affect the structure name, for example, maybe items it's not a good choice.

Let'S call it type names ISM, so basically defining these levels of information lets you make use of backdating so that, even though you will execute up to a certain point, you won't get to the expensive stuff.

All right, I'm ready to stop how about you.

Our listeners may be ready to stuff too.

I think this is good.

I think there this is from a user's perspective.

This is a great review of how how salsa is working with your project fund equal, going to talk about like how salsa itself works.

Maybe we can do that at a separate video, oh yeah.

I think we should wait on that, but I, like this format, we'll do more of these in true.
