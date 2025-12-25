Minimal Conceptual Hello World Flow

1) Declarative command: Create parameterized line with length L.
2) Tree synth: Range of L in [10, 20, 30]; map CreateLine(L) -> set of lines.
3) Commit: select line set and commit as TimelineStep "CreateLinesFromRange".
4) Timeline evaluation: Kernel creates geometry blobs, stores entities, outputs refs.
5) Change: Update range to [10, 15, 20]; tree recomputes; step invalidates and replays.
6) Explainability query: "Why does this line exist?" -> traced to range entry L=15 at SynthNode X and TimelineStep Y.
