--- 
layout: post 
title:  "Applying AI to Biological Data: Challenges and Lessons"
date: 2023-10-05
description: Lessons and challenges from training AI on biological datasets.
tags: AI, Biology 
---

My work in the past few years has focused on translation of methods underlying the
remarkable advances in AI and NLP to biological sequence design problems.

Many of the most successful algorithms in NLP can be translated with little modification
to operate on biological sequences rather than natural language. Take for example the
influential Evolutionary Scale Model (ESM) which is basically just the BERT masked-token
prediction task applied to protein sequences.

However, this line of work comes with unique challenges which include the limitation of
direclty being able to inspect the data, lead times in testing biological constructs,
and some fundamental gaps between natural language and biological sequneces.

Despite this, I find working in this field incredibly rewarding and I have a positive
outlook on its impact.

Imagine how much harder it would have been to train the GPT line of models if individual
reseachers performing experiments did not have the ability to prompt one of the models
and visually inspect its generated natural language?

In this post, I'll give a picture of the experience and challenges of training
transformer language models on large biologicla sequence datasets.

## Staring at the Data

It has been said that great ML researchers spend a surprising amount of time directly
inspecting raw data. If you have ever looked at biological data, its obvious that its
impossible for a human to comprehend the  meaning of a biological sequence in the same
way we understand of natural language, images or audio.

<!-- Maybe make this a hover-over text? --> <!-- Eric Lander in 2004 famously said about
the Human Genome Project "Bought the --> <!-- book. Hard to read." -->

<!-- One particularly challenging aspect of working with biological sequence data is -->
<!-- that you cannot comprehend it simply by looking it as you could for natural -->
<!-- language, images or audio data types. -->

Directly looking at the raw biological data (e.g. nucleotide/amino acid sequences,
multiple sequence alignments, etc) is always a good idea and I've been surprised by
things you can tell by doing this. However, the  level of comprehension with this
approach is always going to be far below what is possible in other domains and you'll
need to resort to other methods to tell whats going on.

    ```
    CGGAATTCAAGAAGCCCGAGGTGCATGTCGAGGTGCGGTTTGCCTCGTAAAAAAGCCGCA
    ATTTAAAGTAATCGCAAACGACGATAACTACTCTCTAGCAGCTTAGGCTGGCTAGCGCTC
    CTTCCATGTATTCTTGTGGACTGG_TTTTGGAGTGTCACCCTAACACCTGATCGCGACGG
    AAACCCTGGCCGGGGTTGAAGCGTTAAAACTAAGCGGCCTCGCCTTTATCTACCGTGTTT
    GTCCGGGATTTAAAGGTTAATTAAATGACAATACTAAACATGTAGTACCGACGGTCGAGG
    CTTTTCGGACGGGG
    ```

A [transfer-messenger RNA](https://en.wikipedia.org/wiki/Transfer-messenger_RNA)
sequence from [RNAcentral](https://rnacentral.org), a database of ~30 million non-coding
RNA sequences. The 145'th nucleotide is masked. Can you guess what it is? The RNA BERT
language model [1] can get this query correctly for ncRNA sequences ??% of the time.
TODO: check its MLM accuracy.

What are the consequences of this? One consequence is that you could go an alarmingly
long time working with data that you are unaware is complete garbage.

``` the the the the the the the the a a a the the a a a the the. ``` An example of some
bad natural language data. If you were a NLP researcher you would immediately recognize
this as something to be purged from your dataset (unless maybe appeared in a blog post
about ML).

Its possible for a team to bang its head against a wall for years trying to model
biological data from an assay before realizing the assay's data is useless.

TODO: this doesn't flow here

However, consider the following: I've spoken with ML researchers developing generative
AI for natural language applications. A common practice in model pre-training or
fine-tuning is to periodically generate and display the generated output from the LM in
your training dashboard. This way you can see at what point in training does the model
start generating sensible outputs.

Can we do this same thing when training bio-LMs? Not directly but we can do similar
things if we get more creative. If you just directly look at the generated outputs from
a model you'll basically be able to tell only one thing: whether or not its repeating
itself.

But if you think carefully about your problem domain, you can craft more elaborate
autoamtical evaluations. These might be things like probing for secondary or tertiary
structue in the representaiton of the model like was done to genrate the plots in Figure
6 of the ESM.

This means that as a reseracher, your interaction with the data is primarily mediated by
algorithms and software such as sequence alignment and database search tools. Of course,
you can always

So are we completely out of luck in bio/ML applications? Of course not. Any
self-respecting computational biologist will tell you that there are myriand ways of
analyzing biological sequence data, and you could and should use these methods. My main
point here is that all of these things are going to take careful thought about your
particular problem domain. Where an NLP researcher could just prompt the model and look
at the output you'll have to string togehter some bash bioinformatic toosl, maybe even
fit a small ML model in order to figure out whats going on in your model. Its harder,
but fulfilling when you get it right.

We can do this but we have to get more clever and this is where it'll depend entirely on
your particular application type. Fo


## Recognizing Bad Data

Recognizing bad data is probably the single most valuable skill a researcher can have
working on applied ML in Bio.

Here's an example of what it would look like to train a language model on RNACentral
versus training on a biological dataset with almost no structure. If you assay generates
data which falls into that category, you would see something like this:

<plots of loss / mlm accuracy curve> for RNA cental vs a bad dataset.

Another way that we can slice this is to look at data compression efficiency.

RNACentral v22 (late 2022) uncompressed is 25.65GB of sequence but when compressed with
`gzip -9` is 6.48GB - a 4x reduction in size. This means that `gzip` by noticing
repeating patterns in the data... By contrast, a completely unstructured biological
sequence dataset of the same size could not be compressed much below the original 25GB.
The compression factor of your biological data might be a first indication of how much
structure. If gzip fails to compress your sequences by any significant margin - consider
testing the hypothesis that the data you generated is just random strings.

Of course, compression efficiency or language modeling loss is never sufficient to tell
you if your model has learned something useful model is - it will tell you roughly how
much structure (or how much repetition) is in your dataset. Whether that structure is
useful or not can only be probed by - you guessed it a real biological assay.

I see this challenge as incredibly frustrating at times, but simultaneously, this is the
main reason why I find applying AI in this domain to be

This type of work requires extensive use of analogy. For instance, we need to


Fundamental Gaps
1. left-to-right causal is a gap. There is no temporal or causal relationship between
   part of a biological sequence (protein or nucleic acid) that comes upstream

Ilya Sutskever claims that next token prediction is an extremely powerful training
signal, and that "the hardest problems in next token prediction is harder than the
hardest problem in bidirectional masked token prediction (i.e. BERT)" [2]

This raises the question: is the same true of a modality like biological sequences where
there is no unidirectional causal relationship between residues in the beginning vs end
of the sequence? It is unclear to me.


## Outlook

My oulook on the use of generative AI for biology is positive. In some ways I see it as
the most promising way of making sense of the complexities of biological sequences.


References
1. RNA BERT
2. "An observation on Generalization" Ilya Sutskever (OpenAI)
3. "Borges and AI" Léon Bottou, Bernhardt Schölkopf (https://arxiv.org/abs/2310.01425)



