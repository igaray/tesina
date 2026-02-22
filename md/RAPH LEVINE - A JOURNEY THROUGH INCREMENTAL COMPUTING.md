RAPH LEVINE - A JOURNEY THROUGH INCREMENTAL COMPUTATION

Overview
- Why incremental computation?
  - Focus of this talk is sequence data types: vector and big string
- A complicated apporach: propagating deltas
- A simpler approach: diffing
- Immutable data structures
- Theme: Rust has wide dynamic range
  - can express algorithms at a high level, yet plumb down to platform FFI
- Theme: long journeys
  - xi-editor started 5 years ago
  - still learning how to express things in Rust

a major theme of the xy editor project was capturing the incremental nature of text editing
a small edit should be a quick computation
incremental computation in general is when you only recompute the outputs that depended on what changed 
one example is a build system where you don't want to recompile everything when you change one file
another is a spreadsheet 

incremental compilation in general is a large and complex area

i'll be focusing in on sequences which includes vectors and strings

there's a bunch of different ways to do incremental computation
one that's very general but also somewhat complicated is to explicitly work with deltas

another is to do diffing of structures which is simpler but often slower

i'm going to introduce today the use of immutable data structures
to kind of give you the best of both worlds

the main reason i'm interested in incremental computation is to make highly performant user interfaces 
especially creative tools like font editors or text editors where you're dealing with potentially large documents

most ui toolkits at their heart have some kind of incremental computation engine to reflect the fact that most of the ui is not changing
a lot of the work that's been done can be reused

a recent example of that that's gotten a lot of attention is swift ui which has a complex and somewhat opaque attribute graph implementation under the hood 
another example that's probably more familiar to viewers of this talk will be the vdom reconciliation engine in react

incremental computation is also used in other domains besides ui
especially in compilers to do incremental compilation faster
and in rust it's used in rust analyzer 

incremental computation in general covers a lot of ground and much of the literature deals with graphs 
that's certainly important in the compiler case 

this talk will focus specifically on sequence data types
where you do some modification to the sequence like insert delete or update 
and then you want that modification to be reflected in a graphical user interface or some other derived computation

there are two use cases that i'm going to be looking at which have some similarities but also some differences

one of those case studies is going to be text layout 
this takes a string and some other parameters a font a size width and creates a text layout object
a large part of that task is breaking the string into lines and then on each line there's an arrangement of glyphs to represent that string
text layout is stateless 
that means you always get the same result whether you rerun the entire process as a batch or you make incremental changes to it
if we're infinitely fast we wouldn't care about making it incremental but unfortunately all of the font measurement and shaping can be pretty slow 

when we think about how to represent the computation it would be possible to go even finer grain but for the purpose of this talk i'm going to talk about essentially a map on paragraphs 
each paragraph can have text layout independent of the other paragraphs in the string

even with plain text strings text layout is a pretty challenging task
but ultimately we want to take this towards representing rich text as well

the data structures will get more complex but the incremental nature of the computation will remain the same

the second major use case is a list view
in a user interface whether where
there's a list of items in the app data
then each one of those items is expanded
into a subtree in the widget graph
as items are added removed and modified
the gui needs to update in response
unlike text layout updates to a list
view are stateful
in two different ways first your changes
such as inserting and deleting might
drive an animation so you might show
the the item being inserted or deleted
second while i didn't really show that
in this to do example
it is possible for the widgets
themselves to be stateful
for example any embedded text editing
widgets in
in in that gui can have their own cursor
and focus state
you don't want to lose that state while
you're modifying the other items so you
need to preserve the identity
that's a major reason why you want to be
working with deltas
if you did a top to bottom batch
recomputation of the whole thing
you would lose that state the list view
state also requires more coordination
between the app logic and the ui toolkit
to
define the widgets that you use to to
create each item
and also to wire up the events like if
there's a button in there
what happens if you click and things get
more complicated
when you scale up even further like if
you need virtualized scrolling
but the techniques that we're going to
talk about in this talk hopefully will
work up to pretty big scale link maybe
tens of thousands or even higher of
items but
not into the millions where you would
certainly need to to be thinking about
virtualization
so while it might seem fairly simple to
wire up sequence
mutations to a view in a user interface
things get trickier when the ui is
changing dynamically
a particularly important example is when
you have a user interface composed of
tabs
and so you might switch from one tab to
another and then the tab that isn't
being shown
is hidden for a while and so what
happens to the updates when you do that
do you keep sending the updates and
updating the state of those widgets
which might be wasteful or do you
[Music]
pause the updates and then of course
when you re-show the tab what happens
then are you going to recompute from
scratch
are you going to apply the updates that
got queued these are
these are complex decisions and a lot of
the approaches that you could do are
either
slow they're either doing wasted work or
they're wrong
in some way you could you can end up
with your state not quite in sync
there is a general purpose solution to
this class of problems which is to adopt
an
incremental computation framework
there's a
lot of literature on this topic there
are kind of two
major academic
examples one of which is adapton and the
other
is the self-adjusting computation work
of umet akar
which is adapted into the incremental
framework by jane street which was
which was originally written in ocamel
and both of these have
translations into rust there's adapton
rs
and anchors is an in-progress port of
incremental to rust and i think there
might be
another one in that category as well
the salsa incremental computation
framework
is really motivated it really came from
work on rust analyzer and trying to make
analysis of rust programs incremental
but it's
a general-purpose computation framework
and certainly you could use that today
as well
another interesting framework to look at
is moxie which is really intended for
user interfaces
specifically and it has a lot of it's
in many ways an adaptation of react but
it also has some inspiration from salsa
and can be used as a general purpose
incremental computation framework i'm
also going to include in this list xy
editor
xy editor is really focused on strings
but it is trying to be fairly
general purpose in its approach to
incremental computation
and then it has another sort of advanced
feature
which is the ability to synchronize
changes across multiple processes or
across multiple computers
so each one of these incremental
computation frameworks has
its own kind of way of doing things but
there are some features in common
most of them have some sense of
storing the source document or the
source data structure in a database
that's not
generally going to be a sql database but
it has that nature
of taking transactions like every time
you make a change to the data it's a
transaction
and then when you commit a transaction
that commit is usually associated with a
revision id
and then when you want to capture the
incremental nature like if you want to
update the ui based on that
then you do a query into that database
you ask for a delta between two revision
ids you say here's my old revision id
the last state that i
displayed here's the new revision id the
current state of that document
what's the delta between that and then
these incremental computation frameworks
will give you that
there's a lot of other things they can
do for example some of them give you
undo an ability to go back to older
revisions
a lot of them have a fairly
sophisticated way of attaching
computation
to the query so you have some sort of
demand graph that says
don't just give me the data of what
changed but do some derived computation
based on that in an incremental way
and of course as i mentioned xy editor
has this
conflict-free replicated data type this
crdt
which allows collaborative editing one
of the features
that these incremental computation
frameworks tend to have
in common is a lot of complexity
it's not that easy to integrate with
them
so a simpler approach that a lot of
people take to incremental computation
is diffic when you
use the diffing approach you're going to
take the old version of the data
structure
and the new version of the data
structure and you're going to walk
through both of them
finding out which parts are the same and
which parts are different
so the parts that are different you're
going to create a diff based on that
and then you're going to use that diff
to drive your ui process or whatever
other computation and this is a fairly
simple approach it's not
the data structures are quite
straightforward
but it has a tendency to be pretty slow
because in general you need to be
touching each element of those sequences
in order to produce that diff
but because of its simplicity you see
this used a lot especially when the
scale is low when you're not dealing
with super large documents or where
performance is not as critical
one of the downsides to this is that
your diff
is not unique so if you have you know a
sequence and it's already got an a in it
and then you insert another a it's not
clear just from the
from the two sequences whether you
inserted before or after
and if you're driving an animation in a
list view you might actually care so
that's something to be
that's something to be careful about the
main concept of an immutable data
structure
is to represent that source data
structure whether it's a sequence or
there are immutable maps and other
things you use a tree
to represent that and so if you're
storing a string
usually that data structure is called a
rope and i first adopted the rope
in xy editor as the main data storage
technique for strings and even though i
think
there is a lot of decisions that were
not awesome in xy editor i think the
choice of the rope
for the main editable text storage
really was
the right thing to do that was the right
choice
so the idea of the rope is that you're
storing the sequence you're storing the
characters of a string as
leaves in a tree and then you have nodes
and those nodes are annotated with
additional information in this case the
length of the string that's below them
in the tree
and that and that annotation lets you do
efficient random access
i should also mention that in order to
make the visual simpler i'm showing this
as a binary tree but
the actual implementation of immutable
data structures
usually uses some kind of b tree or some
kind of variant on that as it's it's
going to be more efficient than a binary
tree
so here's how you do an update in an
immutable data structure
let's say we want to change just element
3 of this sequence to a new value we
want to go from
its current value which is a z to a new
value which is a d
so in in an immutable data structure
you retain the old copy of that sequence
you're not mutating it in place like you
would if it were a vec or a string or
some other mutable rust data type
so you're going to keep the old data
data structure
but you're also going to have a new one
that's going to represent the new
sequence
and that new sequence is not a copy of
the entire thing
hopefully most of the substructure of
of that sequence is actually going to be
shared with the existing sequence that
you have
in general you're going to build a path
from the root
to the leaf and along that path you're
going to be allocating new nodes
but the other nodes in the tree are
going to be shared between the old and
the new version
so one of the nice things about these
immutable data structures is that the
performance is
very consistent it's always o log
n because you keep your trees balanced
even in the worst case
and that's important for interactive
applications by contrast
if you look at a gap buffer which is
another popular data structure for
representing edible
editable text in a text editor it's fast
most of the time
but if you have to move the gap then it
can go to on
so the worst case is quite a bit worse
than
than the expected case immutable data
data structures are really big in the
functional programming community they
really
come from haskell and closure
and these kind of pure functional
programming languages
there's a lot of literature over the
years there's some
very highly tuned implementations a lot
of
computer science analysis for example
finger trees
have this amortized o1 complexity if
you're just
appending or deleting to the end of the
tree so there's a lot of literature to
draw on
now even though these come from sort of
the pure functional programming language
community
these data structures adapt pretty well
to rust as well
and there are several really excellent
implementations of immutable data
structures
in the rust crate ecosystem probably the
best general purpose
implementation of immutable data
structures is the im create
and that has vectors immutable vectors
immutable maps
a few different flavors of each of those
for my work i'm adapting zyrope and
xyrope was really
originally designed for strings but it
does have a general purpose structure to
it
where you can do other data types or you
could even
add monoids to those computations and
compute other derived data
which has some interesting applications
and then another
entrant in that kind of rope flavor that
specifically
you know optimized for strings is ropey
and that's another high quality
implementation
of the immutable data type concept for
strings
so we don't want to do
just efficient updates although that's
nice we also want to solve this diffing
problem
hopefully in a more efficient way than
if you use just a standard vac or some
other standard data structure for this
and so we want to find out
is there some way to use the fact that
this
that these um sub trees
are shared to answer this query in a
more efficient way
what has actually changed is there a way
to kind of skip over
these shared subtrees so that we zoom in
on just the part that's changed and
as you can imagine the answer is yes we
store the nodes
of an immutable data type in a reference
counted container
so in rust that would usually be the arc
atomic reference counter and arc has
a pointer eq method which tells us when
two references point to the same value
and we can use that
to detect common subsequences what we do
is run the algorithm from both ends so
first we run it from the start of the
string to the part that's changed
and then we also run it from the end of
the string back towards the middle
and we take one of the concepts
that you find in this rope world is a
cursor
so that's just really a pointer to a
specific place in the tree
and you have a cursor on both of the
sequences that you're trying to diff
then what you do is that you start at
the bottom of the tree you start at the
leaf
and those are probably equal points are
equal you can tell they're point or
equal without having to go into them
and you keep walking up the tree
until you find the first node that is
not point or equal or you keep
walking as long as you are point or
equal and so now you know a common
subtree that is
that is common between those two
sequences and you look at the length of
that
and you advance both cursors by that
length and that takes you to new leaves
so you keep doing that until you find a
pair of leaves that are not point or
equal and at that point
that's all that this common shared
sequence can tell you and you fall back
to kind of more standard differing
techniques to give you the actual
difference between the
the sequences at that point so
in the case of a simple edit if you're
just inserting a character or deleting
or updating or something like that or if
your edits are
you know even a cluster of edits that
are all local then that's going to give
you
this common subsequence in log n time
which is really really quite good
now this technique is not a silver
bullet and that's probably one reason
why you don't find it as much in the
literature
that if you have completely random
scattered edits like for example if you
have
multiple cursors in a text editor and
you have one cursor at the beginning
one cursor at the end and you do some
some edit on both
then it's going to tell you there's no
common subsequence even though there
might be in the middle
that that's a harder problem and so you
know using this simple technique of just
working from the ends
is not going to give you as highly
optimized a result in the completely
general purpose case
but we think that it will work quite
well in practice for the kinds of
workloads that you that you see
in text editing and in user interfaces
so what about these tricky cases
what about these cases where you're not
just doing a simple pipe from the
sequence to the user interface
but you might be hiding and showing tabs
you might have multiple
views of the same model of the same
source document and it turns out that
this sequence diffing approach actually
works very well for these tricky cases
as well
instead of having an explicit revision
number you encode that
in your arc reference so the simplest
possible case
is that your new data is equal to the
old data it hasn't changed at all
and in that case the arc reference is
just point or equal to the old one and
you can skip it completely
now if you suspend updates to a widget
and then re-show it
then this diffing technique really just
kind of automatically does the right
thing
all the updates are automatically
batched without having to do
any additional tracking work any
explicit tracking work in the middle
you just do the diff between the last
live state that the widget had before
being suspended
and the new state when you're when
you're um bringing it back
and that diff tells you what you need in
order to update that widget
same thing for multiple views each view
is going to have its own
local copy of that state and it's going
to just do
its diff of the new state against its
own local copy of that state
one of the really nice advantages
of this technique of using arc as the
fundamental
data structure to hold these immutable
data types
is the automatic cleanup when you're
done that when your
document editing process drops its
reference and when your ui
drops its reference like if you close
the window then
the reference count on arc goes to zero
and you can just drop the data
when you have these sort of explicit
databases then
having to keep track of when you can
actually clean up the data do a
garbage collection turns out to be a
little bit of a trickier problem
and one of the other nice features of
this of using arc
in particular is that it works across
threads that if you want to have some
background computation in some other
thread like for example in the editor
case
you might want to do syntax highlighting
in a different thread you can just pass
that value to that thread
do the computation and then when you're
done apply that back to the
to the main thread the ui thread so the
conclusion is that you get a lot of the
basic functionality that you would
expect out of a more complex incremental
computation engine
but you're able to use much simpler
types
you don't have to have a database
context to connect to the database
you don't need to be talking about
revision ids explicitly
and you don't have to do anything
special to deal with the life cycle
of these objects
so this is an example
of rust's wide dynamic range
that you can express high-level concepts
the same kinds of things that you would
expect to express in haskell or
ml you know kind of higher order
composition
really dealing with abstract concepts
without having to worry too much about
the
low level details of how that maps to
the machine
but when you need to think about the low
level when you need to think about
exactly when is this getting allocated
and
when you need low level control to bind
with platform capabilities
then rust lets you do that in a very low
friction way as well
and of course rust gives you these
compile time guarantees that that the
composition
is safe even when you're doing fairly
advanced things
with um we're gonna we're gonna see some
examples
uh that would be kind of scary in a
language like c plus plus but in rust
you get
very strong safety guarantees from the
language
so let's look in a little bit more
detail
at the way arc works in rust and
i think this is a great example of how
rust the language helps span
the low-level and level worlds so
you're going to use arc as kind of the
core piece to implement these immutable
data structures
and internally that's represented as
a reference count and also the actual
data
and so when you clone the arc what
that's going to do is
increment the reference count so you're
not copying the data when you click you
know clone
sometimes sounds like you clone a big
giant complicated data structure
but in the case of arc you're just
incrementing that that
that reference count so arc
also has a more exotic capability
you can get a mutable reference to the
data that it holds
so when the collar of the this make mute
method on arc
holds the only reference to that arc
then it can just get a mutable reference
to the inner data
now in the general case that wouldn't be
safe if you have
more than one reference to that data
it's going to violate one of rust's core
invariants which is that a reference can
either be shared or immutable but not
both
that's that's kind of the core of why
rust's approach to mutable references is
always safe so if you just gave
immutable reference to the
to the data that's held by that arc then
it would violate that and you would get
unsafe behavior so what meek mute does
in this case when the reference count is
greater than one that it creates a clone
of the value
and it gives immutable reference to that
clone
and it also updates the the reference to
the
to the arc itself so before you had sort
of one reference
now you have two references
so this functionality gives you a nice
performance post on
immutable data structures it lets you
mutate them in place
as opposed to having this allocate each
you know to do that traversal from the
root to the leaf
and do an allocation of a new sub tree
at each level
of of that traversal so um
it's also transparent like you don't
have to do anything special
uh it can automatically you know the
implementation of your immutable data
structure can
automatically apply that optimization
when there is only one mutable reference
to the data
and it of course it maintains rust's
safety guarantees
and i think it's it's important to note
that this
really wouldn't work in a garbage
collected run time in a garbage
collected language
in a language like that when you create
an additional reference
it's it's this lightweight operation
it's not tracked in any way
so when you want to find out the answer
to this question is there another
reference
to this to this data somewhere
there's no real way to answer that
question
you could add explicit tracking but that
is also a pretty big problem because if
you missed something there
then you could end up mutating data even
though another thread is holding a
reference to it and that and
when there's multiple threads involved
then that really is a safety problem not
just a correctness this year
so the performance boost is nice that's
a good motivation to apply these
techniques
but there's another case where we care
even more deeply
and that is when we are integrating with
the platform
and a really good example of that is
text
layout creation so when we're on windows
we use direct
write to create these text layouts and
the methods for manipulating text
layouts tend to be
mutations tend to be mutable methods
that set some property on that text
layout
direct write does not have immutable
methods for creating new text layouts
that are
similar to previous text layouts you
mutate that text layout
and so this is particularly important
for example when you're
resizing a window or moving a splitter
or some other reason why you need to do
a layout change where the text is the
same
but the the the width is going to change
so you're going to redo the line
breaking on it
um so what we'd like to do
is wrap these platform methods in a
clean immutable wrapper
so you can share those among threads you
can use them for example as a cache and
a widget implementation or something
else
and if it holds that reference it can be
confident that it's not going to change
out from under it or you know much worse
concurrently in a different thread
while a different thread is updating it
but we also don't want to sacrifice a
performance we
we want to be able to use those platform
methods because they can be
probably about an order of magnitude
faster
to reuse the layout but just change the
width than to recompute the entire
layout itself
and so in order to do this we're going
to use a close cousin
of get of make mute we're going to use
get mute
and this just this just optionally gives
you
immutable reference if their reference
count is one
otherwise it it gives you none and so
when getmute gives you immutable
reference you can call the platforms
set text width method on that underlying
text layout object safely
and when it gives you back none then
well you have to rebuild the text layout
but
you know that that
you would have to do that in order in
order to maintain safety
so i think that rust is basically unique
among all programming languages in
allowing you to express these high-level
concepts
safely while also giving you
the low level access so you can bind
those platform capabilities
and be able to use them without having
to worry about these details of
am i mutating something that i shouldn't
be mutating do i you know am i thread
safe the
the rust types that you get when you do
this wrapping properly
encode all of that information all that
context
so i'm just gonna walk real quickly
through
a summary of the methods on arc you can
really learn a lot just by looking at
the type signatures we've already talked
about clone which bumps the reference
count
drop is kind of the reverse it decreases
the reference count and when it goes to
zero it it frees the
it frees the underlying object uh and
then you know i'll
go into a little bit more detail here
that arc has a
dref method that gives you an immutable
uh reference to its contents but it
doesn't have a dref mute
method box does so arc and box are not
the same here
box has a unique reference to the data
but arc has a shared reference to the
data so you can get immutable references
but not in the general case mutable
references
so arc has this pointer equality method
on it so you can see if two references
are actually pointing to the same data
and then it has these uh copy on write
semantics methods it has
make mute which will make a clone when
needed and it has get mute which will
give you the immutable acts the mutable
reference when it is safe to do so
and i'll also point out make mute
updates
with this reference to the new clone
when it makes the clone
get mute doesn't do that but it still
needs an and mute reference
in order to guarantee at the language
level that the reference
you get is in fact unique arc has a ton
more methods it forwards all your kind
of
standard traits like eq
and ord and hash and so on and so forth
has some unsafe methods
if you need even lower level control you
can do that
and it has a whole mechanism to give you
weak references
as well i should also point out i'm
focusing on arc here because it gives
you this thread safety
but if you're in a single threaded
context you can use rc instead it's
really very similar
but it's it's not thread safe and so
it's a little faster because it just
uses ordinary
uh arithmetic rather than atomics to
update those reference counts
so i want to talk a little bit about the
journey
that i've been on and my my journey with
russ really goes back
25 years and here's a paper from 1995
where my research group was trying to
really
solve this problem of an ml
language or an ml like language in this
case this is ml kit which is a
ml variant where we can use static
memory management where it's not garbage
collected but it frees memory
when it knows that the that it can be
deallocated
and at the time i think we had some good
ideas that we were using explicit
lifetimes that if you look at this paper
that the types have these row
annotations on them the greek letter rho
which refers to regions and those are
those are familiar to like tick a tick b
lifetimes in rust
but they're a little different they use
effects rust rust uses
some techniques that we did not have
back then rust uses move semantics
afine types and is able to to really
crack this problem in a way that we were
trying to do 25 years ago
and just didn't quite have the tools
that we needed
so when russ happened i was very very
excited i was like
here's a language that actually solves
the problems that we were trying to
solve 25 years ago
so
with rust itself i've been programming a
little bit more than five years before
just before 1.0 came out and one of the
bigger and more ambitious projects that
i've
taken on is xy editor which is
started in spring of 2016 and the goal
was to make a
fully featured high performance test
editor which is
pretty pretty difficult pretty ambitious
project
and at the time it seemed to me that it
just
wasn't practical to build the user
interface
in rust and so i explored an
architecture where there was a core
in rust and then you would write a front
end you would write the user interface
in some other language you
whatever language is most convenient for
driving the user interface on that
platform
and the two pieces would communicate
over async channels
and that architecture has some
disadvantages
uh i wrote a retrospective on
kind of what went wrong the ultimately
the
biggest problem is that the whole thing
was too complex
now out of this project one of the
explorations that i was doing was
the windows user interface and
so you know that we had a mac front end
that was fairly well developed that was
always kind of the reference front end
but on windows it's like a playground i
can explore and try
new different things and so i wanted to
see is it possible to write
a user interface in rust and then bind
those
platform capabilities for text layout
and
menus and all those other kind of
capabilities that the platform gives you
so originally it was very windows
specific and
the the original like the results from
that exploration were extremely positive
it's like wow this this actually
might be a really good way to build ui
so since then a lot of the work has gone
into
taking those window specific things and
making them cross-platform instead
so you know one of the goals has been
that when you type cargo build
it should just work type cargo run a
window should just pop up
and so that work has evolved into the
druid toolkit and a lot of what i've
been talking about today
with these immutable data structures is
revisiting
the text editing piece how can you make
text editing and text layout performance
even when you're dealing with very very
large scale
um so so it's really been
kind of a pleasure to
revisit some of those things that were
the original motivation for the project
and this journey continues um
there is so much more work to be done
uh to really to see this vision through
one of the things that i'm very excited
about is extending the incremental
computation ideas
all the way to the gpu so that you're
retaining fragments of the scene graph
in gpu memory and when you change
something in the ui
on the cpu you're only touching a tiny
fraction of the entire state
just once changed another thing
that is very difficult and also very
exciting
is rich text so we already have in druid
a prototype
markdown viewer and we're we're i'm
excited about where that's going to go
now these ideas with these immutable
data structures you know sequence
is uh you know this is pretty basic and
i think that there's a lot of
interesting future direction to take
that into kind of scalable data grids
and another direction that is
interesting to explore
is making reactive architectures in
general easier like
i think that's one of the kind of themes
of this work in particular is you can do
really impressive computer science and
if it's too hard for people to use
that's a barrier what you really want to
do if you can
is is do kind of sophisticated
uh computer science techniques but then
package that up
in a simple way with simple types
so that you can use that in some other
project
and and take advantage of those advanced
capabilities
so i hope you've enjoyed hearing a
little bit about my journey
and i wish you luck in your journey with
rust i hope that
you get some of the same fulfillment
that that i've been getting and i hope
that the
that the journey lasts for many years
for you as well
um so i'll take questions now i'm going
to switch from this recording to the
to question and answer format and i
am also going to give you um these
resources that you can follow if you're
interested there's the druid tool kit
there's the zyrope uh crate which is
actually
in the xy editor repository and if you
want to discuss any of these concepts
then please stop by our zulipchat it's
very friendly it's open to anybody with
a github account
and uh you know we have a lot of people
certainly not just me we have a very
active community
of people on that zulip who are all
working on druid
and related projects runebender the font
editor
and are generally very friendly
and welcoming of new people so with that
i will now turn it over to live q a
thank you
[Music]
all right everyone well welcome back i
hope you enjoyed that video
uh we are now ready to go live with rafe
uh to answer some questions we saw some
questions in the q a
yes the question mark uh dial so let me
go ahead
and turn him on and we can get started
hi hi ryan wonderful well welcome back
and first of all thank you so much for
your contribution to
restlab um do you have anything you want
to say
about your participation in this event
um just thank you very much for inviting
me it's an honor i'm really happy to be
able to uh
let's see the audio is echoey a little
oh is it my idea audio
i think it's probably on my end it's
probably the speaker and the mic but it
sounds good now so
yes uh thanks thanks for inviting me and
i'm very happy to uh
address any questions that people yeah
yeah we have a few in chat the
wonderful questions as well so yeah
let's um let's get started with them
uh the first question came around um i
think you were
talking about diffing and we have a
question
from uh yahshua
and yahshua says does that mean that
general purpose engines
don't have to keep the original data
around or did i misinterpret the point
on diffing
right so good question and so the
when you use a general purpose engine
like adapton or something like that
then the what really happens there is
that
all revisions are stored in a database
so the if you get a mutable data
structure like avec or something like
that
um you don't
there's a little there's there's kind of
two parts to this question and so one of
them is if you just use a vec
then that's a mutable data type and so
when you do a mutation of that you don't
keep the original version
but when you use adapton you're not
using a vec you're using an api
where you have to do kind of an explicit
delta
on that data structure and so the
original version
is kept but it's kept in the database
it's not kept in a data structure that's
accessible directly to the program
so i hope that clears that up thank you
for for explaining that a bit more
we have a few more questions from arif
the first one was
what is the difference between the clone
and the copy trait
that's a good question this is a concept
a distinction that really is unique to
rust other languages don't usually make
that distinction between
like a clonable and a copyable data type
the
copy trait says that that data type can
be cloned
just by copying the bytes so if it's an
integer
or some simple data it doesn't even have
to be simple it could be a struct it
could be an enum those
can be copy but it has to be cloneable
just by copying the bytes
of that data structure so an example of
a data structure that is clone
but not copy is a reference counted
container like arc
because the clone operation if you just
copied the bytes
you would get a violation you'd get your
the point of arc is that you're trying
to not
copy the contents and so copying the
bytes would would do the wrong thing
so the clone operation is implemented
by incrementing the reference count so
clone gives you richer opportunities
it lets you do things that just copy
can't but
copy is is most efficient when you have
those kind of
simpler data types yeah that's a great
point
for sure uh let's see we have another
question from arief as well
talking about arf he says you said that
box gives
a unique sorry it gives a shared ref and
art gives a unique reference and
nico matsakis has talked about wishing
mute were named unique and could you
explain why you prefer the name paradigm
of unique shared over mutable and
non-mutable
it means excuse me sure so first of all
it is backwards that box is a unique
reference boxes like unique pointer and
c
plus plus and arc is a shared reference
it's like shared pointer in c
plus but the question is valid and the
reason
why um unique and shared is sort of a
more precise distinction than mutable
immutable
is that there is this other concept
called interior
mutability i didn't go into that in this
talk because you don't really need
interior mutability these immutable data
structures work
without that concept but you can have it
and so
mutex is a thread safe immutable oh
sorry
mutex is a thread safe interior
immutability
ref cell is the single thread version of
that
and that lets you mutate if you have a
unique reference or a you know what's
often called an
immutable reference if there's interior
immutability
you can still reach inside it and you
mutate things
so unique and shared are really more
precise
characterization that covers that in
that
interior immutability case as well
whereas talking about mute and
you know just sort of an anu kind of
immutable versus mutable references
that's that's an accurate distinction
when you don't have immutable
i'm sorry when you don't have interior
mutability
in your data types uh so that that's the
reason for that
for that uh preference wonderful all
right
i think we have just one last question
let's uh let's go this one
is from steph jones and steph says uh
i am interested to know if you have any
thoughts on gui in video games
specifically or perhaps how the concepts
of the talk apply to
entity component systems so i have a
talk
on using data oriented techniques in
in graphical user interfaces from i
think a couple years ago now and i can
refer you to that talk i can
fish out the the link and paste it to
you it's a very broad question
i think that for the most part
the techniques in this so the techniques
in this talk are definitely oriented
towards
graphical user interfaces and i think
probably
not when we're talking about the apis
there's probably not a huge distinction
between a gui that you'd build for a
video game for embedding in a video game
or a more
general purpose documented oriented gui
which is the main thing that we're doing
but i do think that there's a
distinction between the kind of things
we're trying to do here
and the kind of things that would be in
an entity component system
architecture that the
point of this is to give you these
simple data types that don't reference
external databases like an entity
component system
architecture is actually similar in some
ways to these
uh incremental frameworks that there is
a database which is
often called the world and then when i
talk about
using references or revision numbers to
talk about the incremental part
the revision numbers are similar in some
ways to entity references
as well and so i think the ideas of this
talk
are a way to simplify that to say let's
just talk about the sequence let's just
talk about the data that's held in that
sequence
let you do things like mutate it and
then be able to query that later in a
way that's useful to ui
but not have this explicit entity
reference
so i think maybe there's future work
in how to apply those concepts more
in video game worlds and guise that
you'd build in a in a video game but um
uh in general i think it's easier to
build gui
using um these simpler types than
uh i think the entity component system
architecture is
interesting but i'm not convinced that
it is the
simplest and most ergonomic way to
control a gui from your app logic
all right well thank you so much for
that explanation um
let me check really quickly i think that
was our last
question here yeah in the q a it sounds
like
that's about it and steph says thank you
as well um
okay wonderful i guess that's it okay so
that was the last question
and so we're just about finished with
this uh this
session right now i want to say thank
you all so much ray for coming here
and hopefully we'll have you also in the
next edition
of rest lab so thank you so much i hope
uh the virtual format was convenient but
uh hopefully next time in person
yeah yeah we would love that wonderful
well
all right all right well uh stick around
everyone we have another
talk coming up very soon um our next one
is getting started with mongodb
and rust with mark smith that starts at
12
40 p.m so we'll go ahead and
uh shuffle on into the next one for
those of you who are
uh are coming and thanks again rafe
sure thing thank you all right
and uh let's see i want to just check if
there are any announcements here
nope that's about it so we'll go to the
next uh room for the next session
we'll hopefully see you all there take
care bye
you