---
title: "How Binary Jumps and Local Connections Can Make Attention Faster"
date: 2026-10-10
lastmod: 2026-10-12
draft: false
author: "HaoyangMa"
description: "A popular-science introduction to sparse attention, binary information propagation, book and code experiments, and the gap between theoretical savings and GPU speed."
tags:
  - AI
  - attention
  - popular science
---

As we all know, an attention block in a neural network combines information from one word with information from other words. For example, in "I love machine learning", the word "learning" can pay attention to "machine" to understand what "learning" means here. That's quite simple and powerful.

And this pattern has been around for nearly 10 years. So the question today is: can we speed it up? With some kind of mathematical trick?

Of course we can.

## Attention as information transport

So before that, let's think again: what exactly is an attention block? In my opinion, we can just think of this complex mathematical structure as a simple idea: information transmission. The job of an attention block is to pass information from one word to the other words that can attend to it. And then, for the next word, we do it again.

## Can fewer connections do the job?

So, based on that assumption, can we just move the information with fewer computations? A full attention block needs n^2 pairwise comparisons. Even causal attention still needs roughly (n^2)/2. The computational complexity is still quadratic.

So many of these connections are computed over and over again. But my human instinct says: when I read words or sentences, I rarely go back over every word. For example, when I read "I love machine learning", I read "I love", then "machine learning", and put the two parts together. I don't go back and compare "I" with "machine", then "I" with "learning", then every other pair. So a bold idea was born. Maybe we don't need all those dense connections. Maybe we just need to make sure all the information gets connected at least once.

## The sparse-attention idea

So I went and looked into it. And yes, all kinds of "sparse attention" already exist. Other people's work was already pretty close to what I had in mind. And their results support the idea: a lot of those dense connections are redundant.

For example, the Sparse Transformer paper found that a full-attention model had already learned sparse patterns in many layers. Their fixed sparse version even scored slightly better: 0.99 bits per byte versus 1.00 for dense attention.

And PowerAttention used about 94 percent sparsity, while perplexity only went from 9.42 to 10.00.

And other studies like these support the same idea.

So the idea that a lot of that computation is redundant is already widely accepted.

But after looking around, I didn't find quite the combination I was looking for: spread information quickly, avoid spreading the same information over and over, and keep the rules regular enough to turn into efficient GPU kernels.

So, here's my attempt to fill that gap in sparse attention.

I'll call it Staged Binary-Local Diffusion Attention. A bit of a mouthful. But the idea is simple.

## Binary jumps and local connections

Here's how it works. Instead of connecting every word to every earlier word, I give each layer a simple jumping rule.

In the first layer, each word looks at itself and one position to the left. In the second layer, it looks at itself and two positions to the left. Then the long jump doubles again: four, eight, sixteen, and so on.

But wait. If I only jump one, two, four, eight... what about three? Did I just forget a word?

No. Three is one plus two. The information can travel one position in the first layer, then another two in the second. Seven? One plus two plus four. Every distance can be built from these binary jumps.

And the word at the other end isn't carrying just its own information anymore. It's also carrying information it picked up in earlier layers. So I don't need to ask every word directly. I can let them pass the information along.

That's the binary part. But passing information farther isn't enough. I also want to keep nearby words working together. Now here's the staged, local part.

The first two layers use just those two sources. From layer three onward, I also keep the two nearest words on the left. So each word looks at four sources: itself, its two left neighbors, and one long-distance source.

The nearby connections stay nearby. The long jump keeps doubling. I keep a small local route to organize the words, while the binary route spreads information farther and farther.

And I'm still using attention to choose how much information to take from each source. The connections are fixed. Their weights are learned.

So I'm not replacing attention with a plain average. I'm giving it a much smaller, more organized set of choices.

With the binary scaffold alone, the reachable range can double with each layer. Two positions, four, eight, sixteen... Ten layers can cover 1024 positions. Twenty can cover about a million. All without asking each position to directly compare itself with a million others.

And the rule is the same at every position in a layer. No searching for neighbors. No sorting words into groups. Just fixed offsets, which I can calculate directly.

That's the idea: a few regular connections, a rapidly expanding reach, and a small local route to keep the information organized. The local route keeps nearby information organized. The long-distance route keeps expanding.

In short, it's a diffusion step along the time axis.

## How many connections can we remove?

So, in theory, the longer the context, the bigger the savings. Let's compare my architecture with dense attention, using the same number of layers in both. At 1K, with ten layers, that's about 150 times fewer attention connections. At 8K, with thirteen layers, it's already over a thousand times fewer. And at 1M, with twenty layers? About 140,000 times fewer. That's over 99.999 percent of the pairwise attention computation gone.

But that's the attention interactions, not the whole model. The FFNs, projections, and normalization still have work to do. Those costs don't disappear when I remove connections. That's incredible!

## Turning the graph into GPU code

But when I tried to turn that theory into code, things started going wrong.

I started with plain PyTorch: gather the selected words, compute the scores, apply softmax, and combine the values. Simple enough. But each little operation comes with its own overhead. I had removed most of the math, not most of the time.

So I tried the built-in attention operator, SDPA. Then I wrote fused Triton kernels, combining those operations so the GPU wouldn't have to keep starting another tiny job.

And I kept pushing the CUDA execution: process several positions together, keep intermediate results inside the kernel, and reorganize the backward pass to avoid competing gradient writes. With grouped-query attention, I also stopped copying the same keys and values across multiple heads.

All that, just to make the GPU do less work by doing less work. Apparently, that needed explaining.

Kernel launch overhead and data movement were eating into the savings. The calculation was tiny. The cost of getting it started wasn't.

## What the speed test actually measures

But wait. Would that overhead still matter as much with a longer context?

I ran the speed test on an RTX 4090 with 24 gigabytes of VRAM. And yes, I checked the competition. The standard causal attention was already using a fused FlashAttention kernel. My version used fused Triton kernels. Both used twenty layers, batch size one, and one million positions.

First, just the attention interactions: scoring the connections, applying softmax, and mixing the values. Across all twenty layers, the measured GPU kernel time was about 40.9 seconds for standard attention. Mine? About 86 milliseconds. That's about 477 times faster for the attention interactions alone. No QKV projections. No output projections. No FFNs.

So, about 140,000 times fewer connections, but about 477 times faster attention. The GPU is noticing the missing math. Just not at the same scale.

Now include the FFNs, projections, normalization, and the rest of the model backbone. Standard: about 41.5 seconds. Mine: about 0.74 seconds. That's about 56 times faster overall. Those unchanged costs don't vanish. They become a much bigger part of the bill.

But this was forward only: no backward pass, no optimizer, and no vocabulary output layer. Twenty layers provide a route across the full million-position window. This test measured speed. It didn't prove that the model could use all that context.

## Does it learn well on books?

But speed is one question. Prediction quality is another. So I trained both versions on a small, fixed subset of PG-19: actual long books. Thirty books for training, five for validation, and five for testing, taken from the dataset's separate splits. No joining unrelated books to make a longer sample.

I used windows of up to 32,768 characters and fifteen attention layers, enough to cover each window. Short books and the last partial window stayed in the data too. Same model width, same parameter count—about 629,000 each—and the same twenty passes. And I repeated the comparison with three random seeds. For each run, I picked the checkpoint with the lowest validation loss, then measured it on the test books.

And the loss? On average, training cross-entropy was 1.44 with standard attention and 1.31 with mine. On the held-out test books, it was 1.56 versus 1.43. Lower is better.

Next-character accuracy on the test books? About 53.76 percent versus 57.68 percent. That's almost four percentage points higher. And the sparse version won with all three seeds.

So under the same training budget, fewer connections didn't hurt the result here. They helped. These are still small character-level models, and the standard version was still improving. This isn't a claim that both models have reached their limits. But it's a result worth following up on.

## The code experiment changes the picture

So let me get this straight. We spend thousands of dollars on a GPU, then ask it to calculate a mountain of connections that a much smaller graph might replace.

At one million positions, my graph uses about 0.0007 percent of the attention connections. Roughly one in 140,000. That's the size of the cut on paper. Put it another way: if I'm paying for 140,000 times as many connections, I'd like to see those connections earn their keep. Where's the spectacular improvement to match that spectacular amount of work? Looking at the loss, I don't see it. In the PG-19 training test, the sparse version even came out ahead with all three seeds.

But this architecture has problems too. Those results were on natural language. When I switched to code, the gap went the other way. And it wasn't small.

I used a Python subset of The Stack Smol, with separate repositories for training, validation, and testing. Same fifteen layers, about 693,000 parameters each, twenty passes, and three random seeds. Average training loss: 0.77 for standard attention, 1.23 for mine. Test loss: 0.84 versus 1.43. And test accuracy? About 78 percent for standard attention, but only 62 percent for mine. Sixteen percentage points lower. All three seeds told the same story.

So this wasn't just a problem with unseen code. Mine was worse at fitting the training code too. Maybe code needs more direct, precise connections: which variable, which bracket, which function? Passing that information through several other positions might blur it along the way. That's a possible explanation, not something I've proved yet.

Training loss was still slowly falling at twenty passes, so this isn't the final limit either. But the lesson is clear: fewer connections worked well on these books. That doesn't mean they work just as well on code.

But stepping back, this still tells me something. At least for some tasks, a lot of those dense connections are unnecessary. Yet today's GPUs, and the whole AI ecosystem around them, are still built around dense computation. The hardware, the software, the optimization tricks—all really good at doing more dense math. Not so friendly when I ask them to do less, sparse work instead. That's the frustrating part: removing the work doesn't automatically remove the cost.

So here are a few interesting questions. Where exactly can I cut connections, and what decides that? Why doesn't less computation automatically mean more speed? Could a whole new hardware architecture unlock the power of sparsity?

This article describes an exploratory idea and small, fixed-budget experiments. It aims to explain the mechanism and the measured tradeoffs; broader task coverage and longer training remain open questions.

## Manuscript and experiment records

[Download the original English script](<files/How Binary Jumps and Local Connections Can Make Attention Faster.txt>). The attachment preserves the source script, including its video cues.

The [v0.1.2 research release](https://github.com/HaoyangMaLabs/Staged-Binary-Local-Diffusion-Attention/releases/tag/v0.1.2) contains the English paper, code, experiment protocols and recorded results.
