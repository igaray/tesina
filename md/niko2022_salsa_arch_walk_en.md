 so what i wanted to do
uh was yeah talk a little bit about
salsa and especially dive into this
um
this pr that i have that kind of reworks
or is it a pr or is it just a branch
maybe it's just a branch i thought i
made a pr from it but i don't see a pr
from me so
okay
um
that reworks how the
code is implemented
because
if you're interested in helping hack on
salsa there's no reason to learn the old
code
so i think the new one is better
um and it's not this is not
so just to put it in context i guess
before i start
um
let me let me pull up a hack and be here
first of all
um
and i'll send the link here
uh
um
so let me just step back a second and
give the big picture
uh
so
i think
where i would like to get us to
is
that
we have this newer entity system that
some of you may have seen
or been
or seen the work we've been doing there
um
and that we have a sort of
a run time that allows you to easily run
work in parallel
sort of rayon style um
maybe i don't know ryan ish uh
and
and
well that's it
those are the big ones uh and it's
efficient in everything
uh oh no there's one more thing um and
handling fixed point cycles
more or handling cycles better
including fixed point i'll explain that
in a second um
i think
in terms of where i what i hope is that
i hope salsa can become used
by not only rust analyzer but by rust c
and other
uh
compiler like applications where it
makes sense or um or just in general
applications um
but i think in order to get used by
rusty especially
that may not actually make sense
because rusty is very different from
rust analyzer it's like a batch compiler
has different
optimization strategies it needs to
persist to disk um
that may want to be
its own crate but hopefully one that's
compatible with this general salsa
philosophy
or it may be that we can extend cells in
place to do all those things but that's
kind of the big picture where i'd like
to see it go
uh
is highly usable
convenient
and uh
supporting parallel and and kind of more
opinionated than it is now like
so that it's more this is the way you do
it and if you do it this way it works
out nicely versus the current api is
was aimed for flexibility in a couple of
ways that i think are not necessary
that makes sense
um
so let me talk a little bit about fixed
point cycles uh
mixed point cycles the idea is today
when you have a cycle
it's just a it's just an error although
we have this recovery mechanism right so
if you have a query that
depends on another query that kind of
circles back to itself
we allow you to
recover from that with some default
error value essentially
but that's about it
and in practice especially in rusty and
other compilers that's not often good
enough
often you need to have
more of a the ability to say well here's
a starting value
that you can use and then i'd
for the value result of that cycle and
then i'd like to run again and again
until until i kind of reach a fixed
point so an example would be like
well an example might be if you were
searching through
a tree
to see whether
um something is send or not like in like
all the fields reachable from a data
structure
you can wind up reaching yourself again
like you have a linked list and it has a
link to
another node and then comes back to
itself
and
that's okay we should be able to handle
that so if we want to make queries for
each of the parts of that search so that
we can get better incremental reuse
throughout the search we need to
tolerate cycles um i'm not going to go
into that in detail i just wanted to
kind of explain what it was and that's i
think
probably one of the next set of pr's or
things that i'm trying to work on is
extending the cycle handling to cover
that
but i did recently land a big pr that
refactored the cycle handling
in anticipation thereof
which i didn't plan to talk about but
we could uh if i can find it
um
somewhere
anyway
what i did expect to talk about today
is another little piece let me share my
screen
which is really like
the core
algorithm of salsa essentially
um
let me just get a poll how many people
i realized i have no idea how familiar
you all are with
house also works
so how many people feel like they
understand
the idea of salsa if not the way it's
implemented exactly
uh pretty used to using salsa as
consumers of the library uh
though we haven't hacked much on the
internals
okay
cool
so i will just go over that real briefly
uh
i assume that everyone's kind of fault
and you can tell me if you don't follow
along but right you have these queries
they invoked on a set of keys
and they produce a value
and
then you have inputs
and
inputs are set
on the outside
um and so you can build up a dynamic
graph
uh
where
you have like
oh i could even do this
let's see if i can remember how it works
uh oh oh forget it
try that
i can never remember the syntax sweet
ones where you have like input
input um
and then maybe there's some what's the
usual example let's call it there's the
source text
for the compiler and then there's like
the parser
which consumes the source text
and then the
ast
creation
um
or something right and then the type
trigger um
and
when there's a change in say the source
text here uh
that'll rerun as much of this as it has
to until the result is the same right
and then it kind of cuts off the
re-execution
so
um
that's how it feels as a user and what i
wanted to go over today was
the way that's implemented and the
algorithm and all the little annoying
uh aspects to it like among other things
we support multi-threading so
there could be multiple threads trying
to run a given query at a given time
and
um how do we manage that and that the
the making sure that
they coordinate with one another and
don't mess things up and how do we
determine when something has changed
might need to be re-executed
so
i did a really
diagram that i'm super proud of
let me find it uh
that i'm gonna work through here there
it is
come on it's super nice uh
it's too too small to even hope that
you're gonna read it that's okay um
but
it's there for
for later
digestion uh
but the basic
algorithm or the basic well first of all
any questions on on this part
and
feel free to just
interrupt me if you have a question
um
so
i'm realizing i haven't really planned
how i'm going to explain this so let me
let me start from this the basic data
structure
that
we have in salsa is this
derived
queue
derived structure and the idea is that
that stands for a derived query meaning
a query whose results are derived from
inputs the pure function of its inputs
as opposed to an input query which is
different
or a um
memoization query or not in a member uh
interning query can ignore those for now
we're just going to look at these
kind of these are the workhorse so all
of these queries parser asd patient
they'd all be derived queries
um
and that's
what this illustrates here is the
flow
when you actually execute a derived
query
of what happens
but
let me take you into the source
this is as i said i'm just going to walk
through the new source i'm not even
going to try to compare it to what's
done today because i don't
think most of you are familiar with
what's done today anyways it wouldn't be
helpful
um
and it's here
let's throw this link in here um
so
the derived storage is
saying
what is the storage we need in the
database
for this query
and
it's defined by this queue which is like
a type that implements the query
function trait and this memoization
policy which isn't that important um
and you see that it has various fields
right uh
the group index
well the database is going to be like
your actual database struct
um has
essentially
one well just has a big bunch of fields
one for each query in your system uh
that are equal to these derived storage
so this group index is kind of part of
this routing system which i don't i'm
not going to go into right now but
we'll get to it uh but that's the way we
when you execute when you have
dependencies between queries as we'll
see they each get
they're kind of we kind of store an
integer that has a group index and a
query index and that we can route back
to the right table um
come back to that
the lru that's used to track
lru storage i'm also going to
skip over that except to say that that's
basically a mechanism for
making sure that you have it most n
entries in a table to avoid
wasting too much memory
um
and the key map and the memo map are
uh more interesting
so
what the key map says
is
we have to map from
the arguments right each of these
things actually has some some arguments
oops that's not going to work
uh
and we need to map
from those
to a unique integer
because integers make our lives easier
um
and that's what this that's the role of
this key map
is
it's going to map from the arguments
the actual values in the arguments to a
unique integer and what i will say is in
the newer salsa that i'm trying to move
us towards this entity system
the arguments to queries are almost
always
themselves a unique integer
so this map is just an identity function
uh
and
the query arguments are entity ids which
are unique integers and so this is an
identity function but in this version of
salsa as it is in this branch
uh
it doesn't have any optimizations
around that so this is really an actual
hashmap
so the idea is that we're going to see
when you look
when you
start
sort of saying
let's see
so you've got your start node
your start node is like
somebody did query
with key one key2 let's say right two
arguments or something
uh
the first thing we're gonna do
is um
maybe i should not do this in graphis
that's probably gonna get old real fast
let's do it like this
uh when we start with doing a query the
very first thing we're going to do is
we're going to
map
key one key two
to an integer
with the key map
no no
so from now on
which we'll just call k so from now on
whenever we after this point
uh we just have this integer and we can
we can work with that and that's what we
call a key
one thing you'll note is when do you
remove things from this map
the answer right now in this version of
the code is we never do
so
if you had a constantly growing set of
keys that would be a problem for you um
so far it hasn't seemed to bite people
too much in practice but that is
something that i think works better in
the entity system later on because we
actually have a sort of an answer for
that uh
so far
that's the key in turning part right
sorry the key and turning part
yeah yeah the key in turning so it's
just the keys
they persist forever
um
because it's basically we we have these
we don't know otherwise when to get rid
of these indices um
okay
the memo map
um
and the sync map are the next two things
so the memo map is the most interesting
part
uh
that stores well
i don't know if it's most interesting
it's one of the most interesting it
stores a map from the key
to a memoized value
so it's saying
a completed something executed we
completed this query in the past
and we got a value out of it
so this identifies
the value
the result
and
what dependencies we needed to compute
that result
along with durability and some other
stuff which i think i will gloss over
at first so let's take a look at that
actually
um we can jump back
to opens
the memo
rs file um so here's the memo map
so derived key index
that's this integer i was talking about
um that's the unique integer and
you can see it's a dash map which is
like a
concurrent read write lock map um
from that
to a memo
arc swap
we'll talk about our swap in a second a
memo map is
i'm sorry a memo
has the value
and
um
which revision it was verified at
and
this is the dependency information
um
so there's a couple things interesting
about this structure
i want to point out
um
which is that the memo
if you look here you'll see uh
well what arc swap is it's basically
like a cell that wraps an arc so it
allows you to change
once we get
a handle on an arc swap we can kind of
get and set the arc that's inside but
the important reason i'm bringing that
up is that this these memos so every
time we complete a result we store it in
an arc and you'll note that like
most of these fields are not mutable
right the value field and the revisions
field
are immutable
like the only one that can be changed
after the arc is created is the verified
out which has an atomic cell so the idea
is kind of that
we've got this cache
when you start to do a new thing you
start running
and at the end you stick it in the cache
when you get the result
and then when the next person comes they
see it in the cache they can just use it
and
um
it's that that value is going to be
anytime you see something in the cache
you know that it's
uh not changing under you so you can
make borrows and iterate over that
that value v and so forth in convenient
ways um
if it should happen that
what will happen next is in the next
revision
there's been some changes to the inputs
we might need to recompute the memo in
that case we'll re-execute allocate a
new memo and replace the old one
with a new value
right we don't update it in place
um so they're kind of like a
persistent
data structure
um
that's a little different than the old
architecture we used to update them in
place and things got really hairy um
so basically memos are
a record
of a value
that was produced along with its
dependencies
they are not changed
once they are first created
we just make new ones um
and the last thing is the sync map
so
the sync map
is meant to if there are multiple
threads
oops
um
if there's multiple threads the role of
the sync map is to coordinate between
them
so that only one of them
is executing
uh
the query at a time
um
that is actually something i plan to
change as part of the fixed point and i
think could just be useful in general
like it's not obvious why you have to
have only one query
or only one thread execute the query at
a time because
queries are
item potent or i mean like you can run
them many times they'll just produce the
same value
as one another hopefully they should or
else there's something funky going on
but a lot of times it is useful because
it's just a waste of effort and if it's
expensive query like running the type
checker or something you might not want
to do it more than once
so you sort of want the option to
synchronize but you don't necessarily
want to always have to do it and for
fixed point cycles and stuff
i think it at least i don't know how to
make it work it gets a lot more
complicated if you have to spread them
across threads it's much easier if
they're if you can keep them within a
thread so i would like
i although this current version of the
branch always synchronizes it's designed
to make that optional
uh in the future
which is partly why this is the
synchronization map is separated out
so now we have all the pieces the high
level phases is like first
we get this integer k
then we look in
or
we check the synchronization map
to see if
somebody is already executing this query
if so
we block
for their answer
and
um
if not
we check the memo map to see if we have
a cached result
or a cached memo m
if so we check if m
is up to date
and the fastest way to check is we say
well we'll get we'll get back i still
want to give you the high level view so
we check if m is up to date if it is we
return the cache value
otherwise
we we execute the
function
create a memo
insert it into the memo map
and return
right that's the high level and then
we're going to sort of dive into these
in a little more detail um
so far so good
so
uh
how do we check if the memo is up to
date let's look at that
so each memo
has
a verified at field
and a set of dependencies
we saw them here
um
or we call it i guess the revisions
so what verified at
oh first of all
tangent
revisions maybe you already know this
but just in case
the way that we think about changes is
there's a single integer associated with
the database that we call the revision
every time you change an input in any
way
we create a new revision
so it's always linear so they can sort
of say okay we're it's like a timestamp
right we're in we start out in revision
one or zero or something and then make a
change now you're in revision one now
you're in revision two three four so
so we always have
always uh each change to inputs triggers
a new revision
always have a current revision
r
um
and the only thing i'll say is we use
the and mute self
guarantees to ensure this right so in
order to make a new revision you have to
have an unmute hold on the database
which means there can't be multiple
threads
going off and starting new revisions and
parallels there's always like a
well-defined notion of
one revision what revision you're at
um
so the verified at field stores
a revision
uh
and it's an atomic cell so it's mutable
which is useful um
and the idea here is this is basically
just a to save a lot of work
when we first create
when a memo is first created verified at
is set to the current revision
so that's essentially at that point it's
just saying this is when it was computed
and so of course it's up to date now if
you never make any more changes to the
input
you never have to check anything because
uh it couldn't possibly have changed so
so the first thing we're going to do
actually here i'll pop over to my
my crazy diagram and try to show you in
here
um
the first thing we do is we load the
memo map
i guess that's interesting i was a
little bit wrong i guess i guess we
claimed the sync map later oh yeah i
think that was on purpose i remember
that now
uh
so this isn't exactly right
the first thing we do when you do a
query is we check to see if we have a
cache memo
and this was exactly because
why synchronize if the result is already
there and already memoized
we don't want different threads to have
to block on one another that's like an
unnecessary luck i'm i'm really shooting
for my real hope and goal is that we can
get this path this hot path
to have no locks at all
no rights
uh
that's not exactly true today because
the dash map has some internal locking
and and so forth but if you ignore that
it is true i think um and there are data
structures like concurrent hash maps and
things that don't do locks on a
successful read but i don't know if
their acts actually faster i think the
reason dash map uses locks is it turns
out
it's pretty good in practice um
anyway so the very first thing we do is
we look and see if we have a memo and if
we do
then
we can check is it
is it the current revision or not and if
it is we can just return it right so
this is like the ultra fast path
um
and
in this newer
version of salsa we do something
interesting that isn't exposed to the
user yet
but we do leverage it
in the entity version when we try to
find it
so the code corresponding to that read
is right here
oh i guess it's not in this
well
anyway
uh
so this function fetch is the one that
gets called
when you actually
try to do a read
um
so it's really called fetch
dot fetch you can think of it
and
i guess it doesn't do it in this branch
but
what we can do in the memoization code
is actually have an ampersand here so
we're returning a pointer into that memo
um
so and that way you don't even have to
clone the value on a successful path you
just hold a reference to it and the way
that that works
is
a little bit of unsafe code which i
won't talk about
the first but the first on safe code
installs this it's a little
uh sad but i think it's worth it um
so the idea is essentially that's part
of when i made this whole big deal about
memos being
uh immutable once they're created it
kind of leverages that so as long as we
can guarantee that the memo isn't freed
until the next revision
it's safe to have a pointer into that
memo
uh
that lasts as long as the anself that
started it and you can't make a new
revision until you drop that and self on
the database because you need an ad mute
on the database to do it so it all works
out
um which is kind of cool
uh but for now we clone it i think it
doesn't matter
so if the pro so that's that's the easy
path it was created in the current
revision or at least we've already we'll
see it could also mean we've already
checked it in the current revision
that's easy but otherwise
we have to go
um
and do a more detailed look
uh and and actually this is like a bit
of an optimization
so i'm going to skip over this
durability thing for the moment
but
what the durability is basically a way
for us to say
rather than iterate through all the
dependencies one by one if we recognize
that some inputs change less frequently
than others we can assign them a higher
durability the user does that tells us
these are less likely to change
and so
then we might be able to skip if we can
say oh
like the the high level idea is like
rust analyzer says
i doubt the rust standard library is
changing anytime soon so that has a very
high durability
and then
if you have some query that all its
inputs came from the rest standard
library even if the user changed their
code we don't need to recheck its inputs
because
that would be a waste of time uh so
that's what this is meant to do so this
is another fast check
um
but i'm gonna ignore it for now and say
okay we gotta do the slow check we have
to go and actually figure out
if
any of our inputs have actually changed
and then we would have to re-execute and
at this point we have to start
synchronizing
um
so
we
uh
claim the synchronization map and by the
way this diagram like each of these
things corresponds to a piece of the
code right so
so this this quick check i talked about
i called it shallow verify
lives
somewhere uh
maybe in fetch
[Music]
no i don't know where it lives um
i can't remember here
shallow verify
yeah
uh and you can kind of see it
corresponds pretty closely so it checks
if
the revision it loads the revision out
of the cell checks if it's equal to the
current revision otherwise checks the
durability or returns false
so this does the shallow
this is the hot this together is the hot
path or at least what i hope is the hot
path or else you've got problems uh and
then goes over here to actually claim
the lock
um
and for now let's ignore this path and
just say assume we're able to claim the
lock and
the lock is not a uh
when i say claim the lock i don't mean
a
like mutex in the traditional way i mean
there's a map
from your key to your state right and if
there's an entry for your key
then somebody has the lock
and if there's not an entry for your key
then nobody has the lock on that key
um so it's more like a file system lock
or something like that
and
so claiming basically
calls dot entry
and if it's vacant nobody has the lock
we can insert a state saying hey this is
my id
i'm working on this key right now
um and we have a little flag that people
can set to true
if they
want the result so when others are
blocking on us we find out um so this
this relies on
this part here
is building on dash map like you don't
see the lock but that's because it's
built in to dash map there's a kind of
lock that makes all of this happen
atomically
um
and
at the point where we call
entry dot insert is where the lock gets
released so we acquire the lock when we
call entry and then when we call entry
insert kind of this is so if there are
multiple threads trying to claim at the
same time they're going to get
linearized in some order
that make sense
um
so right that's the synchronization map
uh
and then
this is the part where we i left out so
now it's
if there was a memo iterate over the
inputs
so the next thing we have to do
to go back to our
oh what's the right
well we'll just use this one uh there's
a nicer example that i can't remember
off top of my head right now that i
usually use to explain this but
right if we found basically what might
have happened is we changed the source
text
so in revision one
we set the source text and we executed
so all of these nodes
um let's do this
i'm gonna walk through an execution now
we're gonna we're gonna start adding in
a little more of the state with what
we've seen so far
um so in the very beginning
we set the source text
and we didn't have any other stuff
right
um
so we set the source text to be
whatever the first version of the
program was
and that that created revision one
and then we invoked the type checker
the type checker said i need an ast so
that invoked the ast query the ast query
said i need to parse it so that invoked
the parser the parser read the source
text created whatever the parser does
which then was used to create the ast
which then
was used to to run the type checker
right and i guess
that's i'm just gonna you know i'm gonna
cut this note out because
let's say the parser makes the asd
why not
um i don't think it's important for for
anything
uh
so
now that's the data structure we have at
only like that's sort of a set of memos
but each one of these is going to have a
little extra data right
i'm not going to get too fancy
um
so this like this parser actually has
also verified at
okay i'm gonna get fancy
uh
i might regret this
but it's gonna look so cool
uh
you have to look at the rendered one
because this is going to be unreadable
um
so this is the parser
okay
so we got the
we've got the verified at field for this
and then the type checker
also has its own verified outfield right
and we're all in revision one
and everybody's happy and i guess i left
out one other thing which is the source
text we didn't talk about this but
of course the inputs have their own data
and
um
they have their own sort of well it's
called change that because we don't have
to verify it we just know when the input
changed when it didn't but they have
their own little field right
um
oops what is this nonsense
okay
so
now
that's revision one and then a change
happens
and so change that is
here um now we're in revision two
right
and then we're going to call the type
checker
um
so first thing we're going to notice
the verified at is out of date
uh it's not the current revision anymore
and the durability check fails so we get
the synchronization lock
um
and we
uh
start walking the inputs
each of these edges which we're going to
see are stored in a list
and what we'll re we'll invoke the
the same function on the parser
um
which will in turn
uh go and check its its input and see
that it's changed
right and so this this check is going to
return that the parser has has
altered um so let's kind of walk through
the code that does that i guess and then
it will re-execute the parser does this
make sense
one thing i realized might be a little
confusing is i'm drawing nodes here
but the way that it's stored
is like for a given query the hashmap
each of these nodes is one entry
in one of those hashmaps
[Music]
okay
so let's see right we were going to walk
through the the code that
that does this and this is looking
really messy i have to adjust what's in
front i think but um
the code is here
so
it's going to be maybe changed after
cushion oh yeah what is the granularity
of the dependency information that's
stored on each query is he is it like
query plus key so you know
this query depends on that query called
with these arguments
or
okay
it's exactly that um
so
right this
where is this function uh
i think it's called there it is um yeah
we can talk about that so
what's
what's in these
let's let's look at the structure of the
dependency information
um
so the memo
stores a
field called query revisions
and what is that we haven't actually
looked at that field yet that's found in
this other
crate for some reason or this other
module
um
there which is here
so
what does this have
this has a kind of summary of
what is the
of all the inputs when was the when did
when is the most recent revision and
which one of them changed what was their
minimum durability that's not so
important
uh
but also
what is the full list of inputs
which is down here um
and it can be one of three things i
think i simplified this in my
uh
some other branch where i got rid of
this one this was just a mini
optimization for saying when you have an
empty list
don't
don't allocate an arc for every empty
list that's kind of wasteful um but
basically you either have a list of
known
nodes
and yes it's both a combination we'll
look at this in a second but this is the
combination of a query and a key
or
you have
an unknown set of nodes
and the reason for this is sometimes you
have these untracked reads like volatile
data and stuff like that and you just
kind of have to assume that it may have
changed in that case because you don't
you don't know the full set of inputs
um
so that that can be useful for
integrating with
like sources of randomness or other
stuff that actually does potentially
change on every revision um
so
where's the durability storage that you
were talking about
it's up here oh sorry yeah cool
yeah
um
so this database key index
first of all let me copy a few links
here
so this is how we store dependency
information
uh
there's
um
so there's the query revisions struct
that has
summary information
and then
the
query inputs
that has the detailed list um
and each thing in there is called a
database key index this is a pretty
useful
thing that we might it's probably now is
probably the right time to cover
so a database key index what is that
that is a 64-bit integer
with three parts
so
the group
the query and the key
[Music]
if you've written salsa programs you
know that you make these query groups
that are like a trait and
they have inside of them a bunch of
methods which are the individual queries
and those methods have arguments which
are the keys
each of those things gets an index we
already saw how the key gets their index
there's a big hash map
that
maps from the
like values you gave to some fresh index
um the group ones are assigned
by
the order when you make the database you
have to list all your groups so we give
each of them
an index as we do that
and then inside any particular group
we give them we give each of the
functions an index based on the order in
which they appear
the macro that's generating them does
that right
the procedural macro and so this way you
can kind of route with this 64-bit
integer
we can route from
uh
we can route all the way to an exact
entry in the table first we look at the
group index
and the database has basically a
vector of groups so the data the
challenge here is that if you look
carefully at the code the place that
generates the code for the database
doesn't know the full set of queries
like by design it just knows the names
of the groups right the structs and
things that represent the group but it
doesn't know what queries are in those
groups and that way you don't have to
change anything when you add new queries
in a group
which is good but so it has basically a
vector of groups and it uses this group
index to find the right group and then
it hands the rest of the index to the
group and then that group knows about
its queries
so it can use the query index to find
the exact derived storage or whatever
that you're looking for and then it
hands the key index to the derived
storage which
can use that to look up things in its
map um
so
uh
this this lets us make
links between nodes the other nice thing
about it is you can make links between
nodes without knowing
their types
right so
my query may read data of all kinds of
different types
that don't appear if i had to
sort of make a link to the actual type
it would be a memo and the memo is
generic over the key and value
so i'd have link i'd have to somehow
deal with the fact that every kind of
key and value that i use
inside my running code
would
i'd have to give it a type it would be
very difficult but now i can just call
them all by integer and
uh let they they know their own types i
don't need to know them for them if that
makes sense
does that mean that in principle you
could
dynamically generate new queries
for a given group yes and i want to
support that
i didn't mention that but yeah
i think we should it's sort of been
designed with that in mind also that you
should be able to link in extra groups
or extra queries
and
the use case i have in mind is like
procedural macros actually so if rusty
were
let's say
we're linking in
new
a new procedural macro that macro might
itself be based on salsa and be able to
integrate with our incremental
compilation um and have its own group
index and query index and stuff i
haven't really worked out how that would
actually work but
in principle it should work
so
right so that's so that's basically
at the end of the day
it's assuming it's not one of these sort
of well what the heck was i i guess i
was here
i'm actually getting some messages let
me just take a second to see if there's
something urgent
uh nope
so
at the end of the day
the inputs are basically you know these
inputs that we saw
is
essentially an arced array of these
indices
so that we can easily copy them around
so when we
in this deep verify code so the deep
verify is after we've gotten the
synchronization lock and we've decided
that it might indeed be out of date and
our simple checks didn't work
then
we have to do the expensive checks um
and the first thing we do is check the
simple ones again because in the time
between us acquiring the lock and
or us doing the early check and
acquiring the lock somebody else might
have come in and actually done the check
for us so it's always a good idea to
double check um but then
then we basically just iterate
over each of those inputs
and we call this maybe changed after
function
and so what that is
so basically we say
so deep verify
basically says for each input i in the
memo
if may be changed after
i
lv so
let the lv is the last
or let's call it lr
um
the last revision i was verified at
right so what we know is we we read the
verified at field and we know that
that field only gets updated
whenever this memo is known to be up to
date
so as of that revision whatever it was
the memo was up to date
and the question is is it still up to
date
and the answer is uh
it is
so long as no input has changed since
that revision when we last verified it
um so we're going to ask
did i did this input i
maybe change after
the revision lr
and then we would say
if so
this entry is dirty
right and
if no inputs have possibly
changed
then we can update
the verified at revision
to the current revision
and return in
memo is clean
makes sense
i hope
um i'm a little bit confused
um but don't struggle
to understand that last point i'm
struggling to describe
my question
so
i don't understand how there would be a
new revision
if no inputs have possibly changed
yeah that's a good that's a really good
question let's keep let's walk through
our example a little more and i'll show
you how that could happen
um
so remember we were here and we had
modified the source text
to be something different
right and so we entered revision2 and
then we have these potentially outdated
memos hanging around
that came from the older revision
you're going to have to unmute and
because i want you to check back with me
make sure that makes sense
but so if we're at parser and parser
last verse or last revision i was
verified that is one right
so
i still don't understand it says if no
inputs have possibly changed
yeah i see the revisions are per query
well
let's walk through it from here
so
we start at the type checker or
right
actually and the type checker says
well my verified ad is out of date
so we can kind of add that in here first
it says
if it does the shallow verify and the
verif the shallow verify says
is my verified ad up to date
and in this case the answer is no
and so we go down
to the deep version
and we start iterating over our inputs
one by one
in this case there's only one
and then we ask the parser did it maybe
change
and it's going to recursively do the
same thing
so it's going to say no i'm not up to
date let me iterate over my inputs i
have exactly one
it's going to ask the source text
did you maybe change since revision one
so far so good
nobody's answered yet right um we just
asked these questions
uh so we got a little stack going on
and
the
in this case it's a base input
right and so when a base input gets
asked did you maybe change
it can answer definitively
by just looking at its record of when it
last
changed so it says yes i did change
right
um
and so it returns yes i may be changed
and now we're here we're in the parser
we're iterating over our inputs we see
that some input may be changed and so we
consider the parser to be dirty
and remember we still haven't answered
the type checker yet hypertech are still
waiting to find out if the parser
changed or not
um so we're just looking at the parser
right now
and what's going to happen i haven't
shown you that code yet but is the
parser is going to re-execute
so part of when you ask did you maybe
change after this date
it's actually maybe a deceptive question
it's not just saying do you have any
inputs that changed
it will also re-execute the parser
re-parse and look at the output
and it might be that all we changed in
the source text was like adding a
comment and it has no effect on the
result of the parser at all
and in that case the output is the same
and if the output is the same
we can actually say no we didn't change
we can we can move our verified ad up to
this current revision
and say no we have not changed
uh
even though
um
some of our inputs changed because our
output didn't and that's all that
that the type checker cares about
okay yeah that that that makes sense i
just kind of was forgetting the case
that something could output the same
value based on a different input
yeah that's kind of the whole magic like
it's sort of the trick that
lets also be smarter than make or
something um
uh
whether it's whether it's worth it in
the end is an interesting question
because sometimes it might be faster it
might be fast enough to just re-execute
anyway uh but
i'm gonna i think it's cool so we still
do it um
but that's the idea
and
uh
yeah so so then you can actually have
the verified ad
get
well i think what we do actually
is that we create a new memo
i didn't quite cover that part yet
there's a field in the memo that says
when you last changed and so if we
re-execute we will still make a new memo
because it might have different
dependencies and other things like it
might have gotten the same value but
through a different route route this
time
the computation might have been
different but the value is the same
and then we'll we'll back date so we
call it we'll take the change that and
move it to an older revision saying
you know uh this memo was produced in
this revision but the value hasn't
changed since the older ones so i'm
going to i'm going to back date um but
can ignore that for the moment so
um and this if no inputs have possibly
changed basically means if this for loop
completes without
finding anything
then
it's not like an extra check it's just
if the for loop
exits normally so
if for loop terminates then
no inputs have possibly
changed um
so that's right here right here's the
for loop
it says if i may be changed after then i
can return false i failed to verify
otherwise i call mark as verified which
just sets the field and i think does a
salsa event or something
and then return true
okay
any more questions on that so far
you've like almost covered the whole
thing
but i know i'm zooming through so
i i want questions because it suggests
some of you understand
all right
one question one thing i didn't quite
understand um
so you mentioned the
you could actually
you have to update the memo
even if the value didn't change
how does that work exactly how could the
dependencies have changed if the value
didn't
or was i misunderstanding no no uh you
were understanding i mean that's like
we're kind of about to walk through that
part of the code so maybe it'll be
clearer if i just walk through it but
but let me start with the high level
idea leaving aside like the mechanics
how the data is stored
imagine i mean
if the parser
has the job of parsing the input
it also throws away some of the input
right like the source text might have a
bunch of white space and maybe they
change the indentation on a line but
that doesn't affect the parser which
produces
the same ast either way
so its inputs can change because it has
a different input string it's got more
white space than it used to but its
output might still be the same
another way to look at it is like the
plus you know if one of your nodes is is
summing up a bunch of things in a list
and the individual values in the list
might change but their their total might
not
and so anything that only looks at the
total doesn't need to re-execute
right so so if the source text contains
some some new information the parser
looked at it potentially
applied a query
to it
decided that it wasn't important the
value didn't change but if
the same query
that it used to decide that it wasn't
important if that would potentially
change in the future it needs to be
updated
i guess yeah but in this case the way
i'm thinking of it is the type checker
just looks at the ast
the ast just looks at the source text
so the idea is does the type checker
you've you've made some edits to the
file do we need to rerun the type
checker
or can we skip it
we're probably we're always going to
rerun the parser because you made an
edit the source text has changed so the
parser is going to rerun no matter what
but it might produce the same ast it did
before in which case we would skip
re-running the type checker
um
and that that step that allows this to
re-run without
making its
down its dependencies dirty
or it's um the things that depend on it
dirty
and forcing them to rerun that's this
back dating thing that i'm talking about
that makes sense okay
so how does that work
let's dig into it and i think
look at
i hope it will get clearer um
so jumping back to fetch this is the
biggest this is like the high level
entry point
uh
oh i guess compute value is what i want
yeah okay well so fetch is where it
starts it calls
somewhere here calls compute value
compute value tries first the hot path
this is the one that just checks if the
revision information is all to date and
if so it just does a clone
also calls the cold path the cold path
grabs the synchronization lock
pushes the query onto a stack of active
queries i skipped over that i'm not
going to go into it right now but that's
part of cycle handling
and then
grabs the memo grabs the memoized value
from the map
calls deep verify right and if if deep
verify which is the thing we've just
been talking about
iterates over all of the
dependencies and finds them all to be to
either be up to date or have produced
the same output since last time
then you just clone the existing value
but
if
deep verify returns false
or there is no memoized value yet
then we have to execute the function
um so that's where we where we are now
and in the case where we execute the
function
but but either way does it then create a
new revision is that
uh the revision was already created when
you change the input
yeah okay okay but it's just the the
value associated with it can either just
be
one that isn't recomputed or yeah okay i
think i get it
yeah so if there may be an existing memo
and we're gonna
if we re-execute we're going to make a
new memo
but we still have the old one around and
we can compare them
tweak their uh
revision information um
if that makes sense yeah it does it does
yeah um so execute is here you'll notice
it takes the old memo if there was one
um as an as an option
uh
but mostly what does it do
um
let me just
there's some cycle handling that i'm
going to ignore continue to ignore but
it calls q colon colon execute
that is
an associated function defined by
so the queue type is
defined by the procedural macro
uh and it's like unique to your code
and so this executes whatever code you
wrote
for this query so this is really calling
the query now
and running its body
um and that let's just assume it's
successful that's going to produce a
result
um this is of course a recursive call it
might go make other queries it can do
all kinds of things
if it makes other queries
uh that stack i talked about
we put we're pushing this query
somewhere where do we do it
uh
i don't know where we do it i'm confused
but um i guess i overlooked it already
somewhere
uh in any case we pushed this query on
the stack
that stack entry sort of accumulates
um
the set of things you looked at in
thread local storage basically uh oh i
see it's already been pushed before we
came into this function
so the active query guard is the type
that's saying
hey
uh there is an active query on the stack
um
so we're going to run this code it's
gonna it's going to return a value
and then we call activequery.pop
that's right
and this will take it off the stack
um
this this takes the query off of the so
there there is a stack that's
i'm now realizing there's a part i
didn't talk about which is for every
active thread there's a run time the
runtime has a stack
it's available from thread local storage
um
this takes that query off that stack and
gives back to you
the list of all the things all the all
the edges
that it created all the dependencies
so if if it were the parser
that we've re-executed the parts are
read from the source text that would
have inserted an edge into this set
saying
here's here's what you read you read
this space input
um
in that case there would just be one
thing in it
and then we run along and we say okay
was there an old memoized value
did it still have a valid
value associated with it if there was
we can compare
with this memoized value eq and see if
the old value is equal to the new value
this is the back dating we've been
talking about the kind of complex bit
so
we've we've rerun the query we've got a
new value we're going to compare against
the old value and if the two are equal
we're going to update the changed
revision so instead of saying that it
changed
in the current revision
we're going to back date it to whatever
revision we had before
because
apparently it hasn't changed
it still produced the same value
um
so we still have a new memo struct with
a potentially new set of dependencies
and which is
verified in the current revision
and which has
but it has the same value or at least an
eq value you know a value that is eq to
the old value in it and so we can
push the changed at to be
uh equal to the old value
then we insert this into the map
um
this is at that point
we're done uh we re-executed it
we've stored the value
and so we can just return it this is
just a clone of the value or something
um
and
now if somebody
comes later they'll find it in the map
and they'll start this whole process
over again but they'll find it but the
new memo that we inserted has a rivet
you know
it's last verified at this current
revision so as long as no inputs
you know if someone calls this again in
the same revision it'll always be a hit
back date
um
so if we
just to walk through
this one
what would actually happen
let's complete this walkthrough because
it's a little bit interesting
what will actually happen
is
um
we start at the type checker
we've discovered that it might be dirty
we go back to the parser
we discover that it might be dirty
you go back to the source text we
discover that it is dirty
we re-execute the parser
and at that point
it's going to produce
um
a new memo i left out this field let's
add this field in
so in addition to verified at there's a
change that field in the memo that tells
you the revision
in which this last changed
so
so in the new revision we're going to
create a new memo
it has the verify initially it has
verified at 2 and changed at 2. so when
you first make the memo it always starts
with the current revision and everything
but then we go ahead
i guess i'll add this in here too
it's also got the value
and
let's say it's like
you know whatever
something
uh
type checker maybe its value is just
okay or a list of errors or something uh
so
initially after we execute we rerun and
and it turns out these have the same
the same ast and so we compare them we
see that they're equal
and we set
the change that back to one
to copy
uh
what it used to be
and then we replace
this memo with the new one in the table
um
by doing that insert call
and so
that effectively this part i realized
might not be entirely obvious um
effectively that creates this edge
and sort of deletes these old edges
uh like that
um
and we were
at that point we returned to the type
checker and we say
i have verified the parser
it has not changed since revision one
and this data is still out of date but
ignore that for a second does this
so far make sense uh conceptually
and did this thing i said about the edge
make sense i'm guessing not because i
didn't really explain
that like the edge between
between the memos there or
conceptually okay
so the thing the thing is that there is
no edge in the memory like
there's a conceptual edge
but actually this just stores the index
of this memo
in some table right so to actually like
there's no direct link between the two
bits of memory
um
this has a this has an index and there's
a database and the database has a map
from the index to this so when we
replace
when we replace the memo in the hashmap
at that index we've essentially
redirected the edge now
without mutating this structure at all
um
so it's
the edges are
i guess there's a bit of a mismatch
between like this diagram which is kind
of at the conceptual level and then you
have to map it to
how it gets stored which is totally
different
uh
it might be useful if i
were to draw it in but i'm
that'll take me too long
um
does that make sense
so essentially every
query plus key has
an act like the latest memo or
and that is the id yeah yeah
i'm thinking
well maybe i can do it
anyway uh
i might try to do it in a second but let
me just finish out before we let's not
get over rotated on this edge bit
because i want to finish the last thing
which is
now we return false in a sense or we
return that we have not changed
to the type checker
the type checker says oh okay
i've been through all my inputs none of
them have changed
all i have to do
is bump my revision and that's the thing
that's in a cell so i can do that
without
creating a new memo i bump into the
current revision and i just return the
existing value and i never re-execute
um
and that's how it works end-to-end
fiend i don't know i don't know what
else to say uh that's how it works um
there's
i guess the next
there's like a little time left i think
i bracketed in 90 minutes
uh
so
obviously feel free to ask questions but
we can also talk about
where we go from here
so
i guess the question is
what do we do next like i said i want to
grow up a set of maintainers and you all
seem excited and interested and i'm
happy about that
uh hopefully i haven't scared you off i
my hunch is
this
you know i went through a lot in this 90
minutes and
i'm guessing that if it were me at least
in your shoes i would have to sit down
with the code and actually debug a bug
or make some changes before i would
claim to to really understand it um so
don't
don't feel discouraged if you think
that's the case uh
i hope that one could compare what i
said with this flowgraph and try to map
that to the code and it will all make
sense um
eventually
uh
but
the
i think
like i have to turn this in terms of the
next steps i have to turn this into a pr
that's step number one i would like to
lay on this code because it's much
different than the current architecture
and i think much simpler and
kind of takes us to the next level i
have only one hesitation which is that i
haven't benchmarked it at all
or done anything to really test it in
production
um
so
i think what one thing i would love is
if somebody who is familiar with rust
analyzer which is or actually
it doesn't matter i would love if people
would try it
on their own projects rust analyzer
among them but i think some of you work
on other things that use salsa this in
theory i think is api compatible with
the existing salsa so you should be able
to just
grab you know the get get revision and
build it
and see whether
it works
first of all
and secondly if it performs well for you
um
uh this is the staging branch that
you're talking about yeah so i'll open a
pr
but
i think i would love people to try it
see if it works
and how the performance is uh
i did try it on rust analyzer
on some of the existing benchmarks
and i did not see a good result
i i'm not exactly sure what i saw but it
looked like it was doing a lot more
executions than it used to
which either means there's a bug
or
means that
or i mean something else i couldn't
quite tell because the way that the test
was set up i think it was running rust
analyzer on itself
and i wasn't entirely sure if the reason
that it was getting a lot more
re-execution was actually legitimate or
not uh
so someone who knows west allies are
better would probably be able to answer
that um
all i'll say is the test looked a tiny
bit suspicious to me because it was like
no i can't remember why but i felt like
maybe
uh
there were there actually were more
changes
like it was comparing against uh
a code base that was it wasn't it was
not the same in both cases um
but anyway someone should look at that
that would be awesome uh and if you all
have if you're not working at
wrestlemania but you have other projects
same thing um if we had two or three
people try it and and things are working
well then i would feel pretty good about
landing in and we'll just
go from there
if we find bugs then
that's a great to do item tracking down
the bug and trying to fix it uh
and i'd be happy to
coordinate that and either we could do
it some like live debug it or
which might be useful
pair debug it so to speak
or people can try on their own
we should be able to try it on on our
system uh
we should be able to get a pretty good
performance estimate
uh as well as
if there was any uh output change
cool are you using the parallel
capabilities of salsa
uh yeah yeah we are okay because i'm
also curious to know how that performs
like
i believe it should be better
so
i mean it's doing
much finer grain locking and things than
it used to thanks to dash map uh and a
relatively simplified architecture
um is it still using the same parallel
database api yeah so you make a snapshot
and okay yeah
i think it's api
equivalent as far as i know
it should be pretty easy to test them
i guess if it's not api equivalent
and i've forgotten something i'd like to
know that too um
but
okay
cool
um
there was a um an old pr somewhere i
can't remember what it was for making um
allowing some of the queries to be async
so giving them features would that be
easier to do now as well since it's a
stack of things that um
or have you thought about this at all
because i know it's not a super common
scenario for particularly compilers but
i mean even in our case i mean there is
some
context where
being able to make async queries or like
could be useful
um
yeah i'm just trying to picture it
i have not really thought about it and i
feel pretty bad because that person as
marwaz i think right they they did a lot
of excellent work and i
uh i haven't devoted a lot of energy to
it now i'm coming here with the big prs
that are totally different uh which
feels like bad os open source maintainer
etiquette so sorry marwaz uh but um
the
i haven't really thought about it i
think i've kind of
been
a little skeptical about
facing support because
it just seemed like
it wasn't clearly a win
and it would make
the async support and rust is still not
totally there yet so things like traits
and stuff don't work so well and it
would just make everything more
complicated to try and support it
but
i am open to it i guess i would rather i
think what the order in which i'd like
to do things is to first get it
working the way we want it
with synchronous support with the new
entity system and stuff like that and
then try to adapt it
um
rather than
adapting it now and then having to like
keep that
complexity you know what i mean
as we go
um
that said i think the newer system
um
i'll have to think about it the the
newer entity system in particular has a
lot less traits than the old one
yeah
uh because it doesn't like in this
version everything every every um
query you do goes through the database
trait
but in the newer entity system they're
actually direct calls to functions just
like any other rest function
uh
and
it's just that the function reaches into
the database to find its storage so it
sort of wrapped i kind of wrapped your
function with a decorator but
uh
which might actually change the ball
game a lot um i still think we're going
to want good async and trade support to
make it really work i guess maybe we can
use the async to create or something
fair enough
but it might be it might be good to
uh
at least make a mention of it uh
yeah at least i should i should read
ros's pr
i owe that to him uh
and
see what approach he took maybe he had
some clever idea that uh
i haven't thought about
um the only other thing i can think
i guess what i was just realizing is in
addition to assessing this pr
uh i did just land
recently als um
like what what's on master has the new
the new cycle handling
and
that was also a pretty big rewrite from
the old one so
maybe what makes sense is to first
assess master
and see if that's working
um
and if so we could even issue a new
release with that
and then land this and assess that
and if we wanted we could do another
call like this to walk through how the
cycle handling works because it's
interesting
to me
but awesome
so that cycle handling is um
without the
without the like fixed point
stuff or does that include some of the
new changes for better fixed point
support
no it doesn't do anything to point um
what it does do is
it fixes the recovery which is pretty
broken uh
and broken both in the sense of it would
ice sometimes for the equivalent of ice
it would panic
and uh
and that it just didn't make sense what
it did
and so now now it does um the model now
is basically you can assign a recovery
value
to any
query
and if none of the queries that are
participating in the cycle have recovery
then it will panic
but
if
um
any of them do
then they'll get it'll sort of break the
cycle by giving them their
every function that has recovery gets
that recovery value and the functions
that don't get their value based on you
know as if the recovery value had been
normally returned so that sort of breaks
the cycle uh
which is much cleaner unless you do some
cool stuff
plus the code is
um easier to understand because it it's
been refactored uh but
it doesn't yet do fixed point but i
think it's in a better place to be able
to do a fixed point when
which is sort of what i plan to do next
all right so let me do that and then
we'll schedule another meeting
to go over that cycle code
um thanks to all of you for
joining this is super fun i hope we got
a great salsa group going
um
i will post this recording to youtube
so if you want to watch it again you can
um also the hackmd's there and i tried
to do a good job of putting links
um but let me know
i'm i'm really keen to have salsa well
documented
so
another thing i would appreciate is any
if people take a look at the
look at what's there look at the docs
and the the
the flow map and all that stuff and see
if it's helpful and if not ask questions
so we can realize how to improve it
oh
one other thing
i'm also planning this is sort of
independent but i'm i'm planning this
algorithm as you may have noticed is
fairly complicated and we
it feels like not not in a while but it
feels like every time i would do a deep
dive i would find some bug
uh
so i have like about 90 confidence in
its correctness um
and so one of the things i'm also
planning on doing is
i've been talking to various people in
the formal verification community and
tinkering with like modeling the
algorithm and trying to prove properties
about it
so far i haven't figured out how to do
that useful way but if i do
um
if any of you have experience in that
i'd love to talk about it and if i do
succeed i'll
try to show you
but
that sounds sounds really cool i have uh
i'm being incredibly generous uh some
experience but
more more just interesting
all right well thanks everybody
and uh just have a good day
thanks
thank you thank you thanks
this was great 