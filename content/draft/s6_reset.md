---
layout: post
title: Associative Boundary Resetting for SSMs (Mamba)
date: 2024-02-21
description: How to enforce boundary resetting in SSMs for sequence packed training.
tags: ML
---

This post is about a technical detail of boundary resetting for Selective State Space
Models (S6) like [Mamba](https://arxiv.org/abs/2312.00752). I discuss boundary resetting
for Selective SSMs, and give an implementaiton that enables resetting at boundaries
without sacrificing the associativity of the binary scan operation, thereby maintaining
parallelizability.

This model architecture was introduced by Albert Gu, Tri Dao in the paper "Mamba:
Linear-Time Sequence Modeling with Selective State Spaces" and has gained attention as a
potential alternative architecture to transformers.

### Sequence Packing

When training large-scale sequence models like as transformer language models, hardware
efficiency is critical to acheiving strong results. A well-known trick to improve
harware utilization algorithms is sequence packing: concatenating multiple seuqence into
a single token array. Without sequence packing, efficiency suffers due to abundance of
`<pad>` tokens which contribute no information to the loss. It is typcial to acheive
`<pad>` token rates well below 1% with a bin-packing algorithm, prior to running LM,
training. 

The transformer architecture admits a simple mechanism to prevent bin-packed from
attenting, and there by interacting with eachother: block-diagonal attention mask.
However, for non-attention based sequence models such as SSMs, alternative approaches
are requried. Let's look at how they deal with packed sequence boundaries in the Mamba
paper:

Section 3.5.2 "Boundary Resetting" reads:

> In settings where multiple independent sequences are stitched together, Transformers
> can keep them separate by instantiating a particular attention mask, while LTI models
> will bleed information between the sequences. Selective SSMs can also reset their
> state at boundaries (e.g. ∆𝑡 → ∞ or Theorem 1 when 𝑔𝑡 → 1). These settings may occur
> artificially (e.g. packing documents together to improve hardware utilization) or
> naturally (e.g. episode boundaries in reinforcement learning (Lu et al. 2023)).

Recall that these two quantities (∆𝑡,  𝑔𝑡) are outptus from the model - this means that
boundary-resetting is a learned behavior. The idea is that the model should learn to
reset itself when it encounters tokens like `<eos>`.

Let's see how that works. When dT -> inf, ...

    TODO: HOW TF DOES THIS ACTUALLY RESET THE STATE

## Enforced Boundary Resetting

In this post I consider what it would mean to enforce boundary resetting in Selective
SSMs. To start, we'll first demonstrate the concept using a naive implementation with
`jax.lax.scan` which is easier to reason about, but extremely inefficient because it
runs sequentially over the input instead of using a parallel scan. Then we'll move to a
parallelized implementation using `jax.lax.associative_scan` and find an associative
operaiton that resets at boundaries.

```python
import jax
import jax.numpy as jnp
from jaxtyping import Float, Bool, Array

def ssm(
    x:  Float[Array, "L D  "],
    dt: Float[Array, "L D  "],
    A:  Float[Array, "  D N"],
    B:  Float[Array, "L   N"],
    C:  Float[Array, "L   N"],
    D:  Float[Array, "  D  "],
) -> Float[Array, "L D"]:
    """VERY SLOW! Sequential Scan operation."""
    l, d = x.shape
 
    # Discretize the continuous-time SSM (implementation omitted). See paper.
    dA, dB = discretize(A, B, dt)

    def scan_op(
        h: Float[Array, "D N"],
        params: tuple[
            Float[Array, "D  "],  # x
            Float[Array, "D N"],  # dA
            Float[Array, "D N"],  # dB
            Float[Array, "  N"],  # C
        ],
    ) -> tuple[Float[Array, "d n"], Float[Array, "d"]]:
        xi, dAi, dBi, Ci, reset = params
        h_ = dAi * h + dBi * xi[:, None]
        y = h_ @ Ci
        return h_, y

    h0 = jnp.zeros(shape=(d, n), dtype=x.dtype)
    _, y = jax.lax.scan(scan_op, h0, (x, dA, dB, C))
    return y + x * D
```

Now, we can enforce boundary resetting by hard resettting the hidden state `h` to zeros 
when it encounters a `<eos>` or `<pad>` token.

```python
def scan_op(
    h: Float[Array, "D N"],
    params: tuple[
        Float[Array, "D  "],  # x
        Float[Array, "D N"],  # dA
        Float[Array, "D N"],  # dB
        Float[Array, "  N"],  # C
        Bool[Array,  ""],     # reset
    ],
) -> tuple[Float[Array, "d n"], Float[Array, "d"]]:
    xi, dAi, dBi, Ci, reset = params
    h_: = jax.lax.cond(
        reset,
        lambda _: jnp.zeros_like(h),
        lambda _: dAi * h + dBi * xi[:, None],
        None,
    )
    y = h_ @ Ci
    return h_, y

h0 = jnp.zeros(shape=(d, n), dtype=x.dtype)
_, y = jax.lax.scan(scan_op, h0, (x, dA, dB, C))
return y + x * D
```

And then we'll pass a sequence of booleans  `resets` to `jax.lax.scan`  to indicate
where resets should happen.

## Associative Boundary Resetting

A critical aspect of Mamba is that its recurrence can be parameterized by an associative
binary operation, enabling efficient training and inference over long sequence by an
[parallel associate-scan](https://en.wikipedia.org/wiki/Prefix_sum#Parallel_algorithms)
operation. Here's what it looks like to translate the 

Note that this kind of enforcable boundary resetting has already been reported, in 
["Structured State Space Models for In-Context Reinforcement Learning"](https://arxiv.org/abs/2303.03982)
See section 3.1 "Resettable S5" in this paper: 

Here we'll be considering how to acheive this 

<!-- Credit to  -->
<!-- "S5: Simplified State Space Layers for Sequence Modeling" (https://arxiv.org/abs/2208.04933) -->
<!-- Credit to https://github.com/lindermanlab/S5 -->


```python
def ssm(
    x:  Float[Array, "L D  "],
    dt: Float[Array, "L D  "],
    A:  Float[Array, "  D N"],
    B:  Float[Array, "L   N"],
    C:  Float[Array, "L   N"],
    D:  Float[Array, "  D  "],
) -> Float[Array, "L D"]:
    """Selective Scan operation."""
    l, d = x.shape
 
    # Discretize the continuous-time SSM (implementation omitted). See paper.
    dA, dB = discretize(A, B, dt)

    def binop(
        ei: tuple[Float[Array, "d n"], Float[Array, "d n"]],
        ej: tuple[Float[Array, "d n"], Float[Array, "d n"]],
    ):
        """See appendix H of S4 paper for detailed review of associate scan for LTI."""
        Ai, Bxi = ei
        Aj, Bxj = ej
        return Ai * Aj, Aj * Bxi + Bxj

    dBx: Float[Array, "l d n"] = dB * x[:, :, None]
    A_, B_ = jax.lax.associative_scan(binop, (dA, dBx))

    h: Float[Array, "l d n"] = A_ + B_
    y = einops.einsum(h, C, "l d n, l n -> l d")
    return y + x * D
```

Note that this kind of enforcable boundary resetting has already been reported, in 
["Structured State Space Models for In-Context Reinforcement Learning"](https://arxiv.org/abs/2303.03982)
See section 3.1 "Resettable S5" in this paper: 

Here we'll be considering how to acheive this, we introduce a new variable into the
associative binary operation whcih has the meaning "is there a reset boundary anywhere
in this segment".

Care must be taken to retain the associativity of the binary operation when considering 
how to integrate boupndary resetting. With a naive implementaiton, we'll end up
violating the 

Consider this pseudo-code where we (naively) try to implement reset gates

```python
def f(
    ei: tuple[Float[Array, "n"], Float[Array, "n"], Bool],
    ej: tuple[Float[Array, "n"], Float[Array, "n"], Bool],
):
    """Naive Implementation! Breaks associativity!"""
    Ai, Bi, reset_i = ei
    Aj, Bj, reset_j = ej
    if reset_i:
        return Aj, Bj, rj
    if reset_j:
        return zeros_like(Aj), zeros_like(Bj), rj
    else:
        return Ai * Aj, Aj * Bi + Bj
```
To understand why this won't work, consider this example of scanning the sequence
`[X, r, Y]` where `X` and `Y` are inputs and `r` is a boundary-reset indicator.

If we associate on the left, we arrive at result `Y`:
`X, r, Y -> f(X, r), Y -> r, Y -> f(r, Y) -> Y`

However, if we associate on the right, we'll get `f(X, Y)`
`X, r, Y -> X, f(r, Y) -> X, Y -> f(X, Y)`

The inequality of these two results demonstrates naive `f`'s non-associativity.

### Associativity Fix

The key to fixing associativity is thinking about the operation like a cumulative sum-
which is the simplest operation that can be made efficient with an associatve scan.

If we keep track of the *cumulative total number of reset boundaries* that have been
encountered so far, we can decide whether to merge the two running SSM states.

Again, in pseudo-code:

```python
def f(
    ei: tuple[Float[Array, "n"], Float[Array, "n"], Bool, Integer],
    ej: tuple[Float[Array, "n"], Float[Array, "n"], Bool, Integer],
):
    Aj, Bj, rj, count_j = ej
    Ai, Bi, ri, count_i = ei
    total_count = count_i + count_j

    if ri:
        # Only increment when reset comes from left.
        return Aj, Bj, rj, total_count + 1

    if rj:
        return zeros_like(Aj), zeros_like(Bj), rj, total_count

    if count_i < count_j:
        return Aj, Bj, rj, count_j

    A = Ai * Aj
    B = Aj * Bi + Bj
    return A, B, rj, jnp.maximum(count_i, count_j)
```

Again, this is just pseudo-code that should be re-implemented with `jax.lax.cond` so
that the whole thing can be compiled with `jax.jit`:

Note that this kind of enforcable boundary resetting has already been reported, in 
["Structured State Space Models for In-Context Reinforcement Learning"](https://arxiv.org/abs/2303.03982)
See section 3.1 "Resettable S5" in this paper.

## Performance Implications

A main constribution of the Mamba work is the hardware-efficient algorithm
implementation in CUDA which carefully uses re-computation, and strategic loading of SSM
parameters into S-RAM to avoid materialization of the full hidden state into HBM.

Its possible that the authors didn't include enforced boundary resetting because they
didn't fully consider it, or because 

Its hard to know what the performance implication of this reset-boundary bookkeeping are
without modifying and benchmarking their CUDA implementaiton. Its possible that the
authors condidered this approach, but 

## Does it matter?

The authors clearly considered the problem of boundary resetting, since they cite Lu et
al which use an associative SSM scan with boundary resetting operation in the context of
Reinforcement Learning. I am guessing that they simply considered the learnable
state-reset mechanism to be superior and more generalizable.

When considering this, I wonder why the authors of Mamba paper didn't. I doubt its
because they didn't realize its possible. Rather, I am guessing that 

## Conclusion

In this post I've covered the learnable boundary-resetting mechanism of Mamba, and
considered an alternative implementation which enforces boundary resetting while
preserving associativity/parallelizability of the SSM scan op.

None of these ideas are new! (though I did independently think of how to do associative
boundary resetting before discovering the resettable S5 paper)!
I hope that my contribute here is

1) to make it clear how 
2) show an alternative design space


My next plans are 

1. implement hard boundary resetting in the Mamba CUDA code
2. benchmark performance impacts



