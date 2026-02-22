THREE ARCHITECTURES FOR A RESPONSIVE IDE

- how to make a snappy IDE, in three different ways
- only the highest-level architecture

Map Reduce
- The idea is to split analysis into relatively simple indexing phase, and a separate full analysis phase.
- The core constraint of indexing is that it runs on a per-file basis.

Leveraging Headers

Intermission: Laziness vs Incrementality

Query-based Compiler
