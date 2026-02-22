
---
# Ask Me Anything with Felix Klock, hosted by Derek Dreyer
https://www.youtube.com/watch?v=joZbWqqfqgg

- how did you get involved in rust?
	- dave herman at mozilla got him to quit adobe
- how has your role in the rust team evolved over time?
- 7:30 do you have a sense why rust has been more successfull at addressing the problems it wants to solve?
	- uptake from ruby & python users driven by syntax sugar and docs, yehuda katz
- 10:45 use of unsafe
- 

SALSA

salsa is a project for incremental recomputation
if you have a set of basic inputs, and you want to do work to compute some derived value from them
the compiler is the motivating example and what it was written for but it is being used in other things too
in the compiler you have basic things like rust source files and the outputs would be binary code

the way that salsa works is that you write your program in terms of small pure functions that don't have side effects
these will derive intermediate values along the way of computing the final value 
and then when the inputs change we can trace which intermediate values did they flow into and which ones may have changed, and only re execute the ones that may have changed
at that point you kind of have make
but we can do better than that
we can remember, because they are pure functions, we can say "the inputs changed, let's re execute", and then see if the output changed
maybe the inputs changed but it didn't matter to the output, and if the output didn't change then we don't have to consider it as having changed anymore, because everything else is only dependent on the output, not the input

that is a huge improvement on make, you get ccache

salsa is being used in rustanalyzer

we have a bunch of ideas, called salsa 2.0 to make it cooler
it evolved out of the incremental system that rustc uses
in an attempt to extract it and make it into a library

---
it struck as a set of techniques not seen very often in other programs
the idea that the state of your system, you want it to be queryable and update it, but you want the queries to be extremely fast as you update the state

salsa a way of thinking of the global state of the system not as a database, but as optimized for mutation but also doing fast queries on the state, whatever it happens to be

the closest thing you see it in is react or ember, glimmers engine

---
then you get into differential dataflow
frank mcsherry 
more like propagatinf diffs, instead of recomputing values and reexecuting

---
there is an interesting interaction between salsa and librarification, the name given to the idea of breaking the compiler into libraries
those three things are interacting, chalk, salsa and librarification
because the first part is that for librarification, you need some subtrate for all of those libraries to interoperate with each other
how does the trait system hook into the compiler? how does it get information about the traits that are in the scope?
maybe it needs to do const evaluation, we haven't quite worked out the interplay between them, that's probably a separate library to do const evaluation
how do those parts talk to one another? 
and the asnwer is the query system, which is salsa

one of the advantages of salsa over rustc, because it was designed with this in mind, is that you can break up you queries into different crates and then compose them
so it should be possible to have chalk export a set of queries, have the const evaluator expose a different set of queries, and these get combined up at a higher level and connected to each other, and they are all incremental all the way down

what chalk needs from salsa, salsa can't yet deliver
it needs to ability to do recursive queries, because solving traits sometimes requires cycles and salsa plain out bans cycles right now
one of the things I would like to do is fix that
that would be a cool extension because chalk would then be able to cache at a very fine level the results of queries and incrementally recompute them
today we dont do incremental with trait solving very well
so when you recompile you could conceivably get it as good as , as long as you dont add an impl to that trait, were not going to have to do any of those traits lookups 

the root of the idea is that slsa becomes a core part of the engine, and other parts plug into it
salsa becomes the fabric that connects everything
the one caveat is that this is how niko originally thought of it and still thinks its corrects except for one thing
chalk doesnt directly export a salsa interface
it just uses traits
you should give me some database that has these methods
its the other level that says " i instantiate theat trait with a salsa database"
of rone thing that makes chalk simpler, chalk has fewer dependencies, it just talks what it cares about
but also it allows swapping incremental systems at the final layer

it turns out that rust analyzer and rustc dont want the same incremental system
rust analyzer wants to be all in memory
rustc probably wants to disk, it is batch compilation, the inputs never change within a single execution of the compiler, they change from a previous execution
what that means is it needs to know all the results from a previous computation, but as it generates new results from this one, those will never change again, and can be streamed out to disk right away, whereas in rust analyzer things are constantly changing and you have to keep that all up to date

the best system is one in which the different parts/libraries interact with traits, they dont really know how this fabric is implemtented, and then we have two differente incremenetal fabrics, one optimized for the live typing case and another for the batch compilation case.

---
compilation caching

if you think of how make works, something might be out of date, I better re do it, and take everything thats dependent, all the way down, its quite a bit of work

ccache says, ill hash the inputs and see if they really changed, its not exactly what salsa does but its kind of closer, 
it can let you say " i thoiught i had to re do this but none of my inputs actually changged"

salsa does that but inside the process and even better because it also hashes the output

this allows it to be very fine grained 

