---
title: 'Attention in Transformers'
date: 2025-07-26
permalink: /posts/2012/08/Attention in Transformers/
tags:
  - Artificial Intelligence
  - Machine Learning
  - Transformers
---

[Attention is all you need](https://arxiv.org/pdf/1706.03762) introduced the attention mechanism in 2017 specifically for language translation, Today, every major language model you hear about BERT, GPT, Llama builds on this foundation.

But what attention? And why did it change how machines understand language?

The Transformer is a deep learning architecture designed to handle sequential data like text. It processes all words in a sentence in parallel and uses attention mechanisms to learn which words matter more when understanding a given word.

Instead of moving word-by-word through a sentence, Transformers look at the entire sentence at once—allowing them to capture long-range dependencies and complex relationships with ease.

Why does Attention matter?

“The cat sat on the mat because it was tired.”

To understand what "it" refers to, you instinctively look back at "cat."  

Your brain assigns more attention to “cat” than to “mat” when resolving the pronoun "it".

This is exactly what attention mechanisms do in Transformers they allow the model to dynamically decide which words to focus on while processing a sentence.

## The Big Picture

![Transformer Architectire](https://cdn-images-1.medium.com/max/900/0*eypQiRGVaHAVY4PY.png)

To understand the role of attention, it is important to first understand where it fits in transformer architecture

The transformer consists of two main parts:

- The Encoder stack

- The Decoder stack

The Encoder stack is to process each input sequence and to generate contextualize representation of each word

- Multi-Head Self-Attention

- Feed-Forward Neural Network (FFN)

Additional components:

- Norm layers: A residual connection followed by layer normalization wraps around both the attention and the FFN layers.

The Decoder generates the output sequence, one word at a time, while also attending to the encoder's outputs.

- Masked Multi-Head Self-Attention: Prevents a word from attending to future words (ensures causality in autoregressive generation).

- Multi-Head Cross-Attention: Allows the decoder to attend to the encoder outputs.

- Feed-Forward Neural Network

As in the encoder, each of these blocks is wrapped in Norm layers.

Where Attention Fits In

- Self-Attention (Encoder & Decoder): Helps the model understand relationships within a sequence.

- Cross-Attention (Decoder only): Helps the decoder decide which parts of the input to focus on while generating each word in the output.

In other words:

- Encoder self-attention: “How does each word relate to other words in the input?”

- Decoder self-attention: “How should I build the output so far?”

- Cross-attention: “Which parts of the input should I look at while generating this word?”

## The Core Idea

Digging deeper…

In transformer, attention helps each word focus on relevant parts of the sentence no matter the position.

The core components that make this possible are:

- Query (Q): What we are asking about. (pretty easy)

- key (K): What we are scanning. (The other words)
 
- Value (V): What we will extract if the key is important

Each word (token) becomes a query, a key, and a value.

Let’s look at some quick maths from the “Attention is all you need” paper.

![attention](https://cdn-images-1.medium.com/max/900/0*3uDxU_CVSzMpfAoY.png)

- Q @ K.T: Calculates the similarity between words

- sqrt(d_k): Scales down dot products.

- softmax(…): Converts Scores to probabilities 

- @V: Applies Attention to values

Let’s say:

You ask a question (Query)

Your friends each have opinions (Keys + Values)

1. You compare your question (Q) to their answers (K) → similarity

2. You discount extremely strong opinions (sqrt(d_k))

3. You weigh their advice fairly (softmax)

4. You make a final decision by combining what they say (V) based on those weights.

To illustrate further let’s see some code:

![attention code](https://cdn-images-1.medium.com/max/900/0*MMBWrtL2CLTwHz1e.png)

Comments made code self-explanatory :)

Output:

```bash
Attention to 'She': 0.15
Attention to 'went': 0.20
Attention to 'to': 0.21
Attention to 'the': 0.19
Attention to 'market': 0.25

Contextual embedding for 'went':
 [0.75 0.77 0.25 0.49]
```

The question we are asking here is

When trying to understand the word “went”, which other words in the sentence matter the most?

The output of the code describes the individual attention given to the word “went”

Then the final contextualized output “went” given below the attention output, that is the new vector representation of “went”, computed as weighted sum of all the value vectors (V), based on how much attention was given to each token.

If each word were a student in a discussion group, and "went" was trying to understand its own role:

- It listens a little to itself (0.20)

- It pays close attention to “market” (0.25), that’s where it’s going!

- It also listens to the preposition “to” (0.21), which links it to the destination.

This is the heart of the self-attention mechanism: Every word learns where to look (via attention scores) and what to take (via value vectors) — allowing it to adapt its meaning dynamically depending on context.

## Multi-head Attention

But one head self-attention is not enough, take a look at this … 

Let’s say we’re analyzing the sentence: "She saw the man with the telescope."

Depending on how you attend, “with the telescope” could describe “saw” or “man”. One attention head might focus on syntax (structure), another on semantics(meaning) and many more (depending on scenario). Multi-head attention gives the model the ability to look at the sentence from different "angles" — syntactic, semantic, positional, etc.

Instead of using a single attention operation, we use multiple heads. Each head:

- Has its own Query, Key, and Value matrices.

- Performs attention separately.

- The outputs are concatenated and projected.

This allows the model to attend to different types of relationships simultaneously