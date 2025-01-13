
---
title: Hadamard Conjugation
<!-- author: Jon Deaton -->
date: '2024-07-27'
---

## Hadamard Conjugation

On the wikipedia page for the Hadamard-Walsh Transform, you'll find a near
incomprehensible section the transform's application in mollecular phylogenetics.


It turns out that whats going on here really isn't that complicated, but is merely being
obfuscated by unnecessary jargon (as biologists tend) like "Klein group".


The "Klein group" is really "two bit xor".

01 01 10
10 11 11
-- -- --
11 10 01



In a phylogenetic tree T, there are 

- labeled vertices v1, v2, ... vn
- unlabeled vertices, which are also called "branching vertices" 

Because its a tree, deleting any edge will split all vertices into two disjoint sets.

There are n * (n-1) / 2 possible edges but since its a tree there really can't be any
more than 2n - 3  edges while it remains a tree. We'll represent 

We can encode a tree with its "tree spectrum" which is a vector w of length 2^(n-1)
with positive values indicating the weight each edge of the tree. zero entries in the
tree spectrum represent edges that do not exist in the tree.



The hadamard transform is inherently related to trees. How?

h_ij in the hadamard matrix indicates whether the number of bits in both i and j is 
even (+1) or odd (-1). If we interpret each number as a bipartition, then its telling us
the parities (i.e. even/odd) of all the intersections of all subsets.

how is this related to trees?

Well, each edge in a tree defines a partition of the vertices.







