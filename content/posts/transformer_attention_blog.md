---
title: "Demystifying Transformers: Attention, Positional Encoding, and Feed-Forward Networks"
date: 2025-11-19
draft: false
description: "A clear and intuitive guide to Transformer architecture, covering attention mechanisms, Q/K/V projections, multi-head attention, feed-forward networks, residuals, layer normalization, and sinusoidal positional encoding."
tags: ["Transformers", "Attention", "Machine Learning", "Deep Learning", "NLP", "GPT", "BERT"]
categories: ["Machine Learning", "Deep Learning", "Natural Language Processing"]
---


Transformers have become the foundation of modern language models like GPT, PaLM, and LLaMA. But their internal workings — attention heads, Q/K/V projections, sinusoidal positional encodings, feed-forward networks — can feel overwhelming. Ironically or probably aptly, I used ChatGPT which is based on the Transformers architecture to help explain to me the concepts and it did a great job.   
This post which is a summary of the sessions I had with ChatGPT explains these concepts clearly and intuitively.

---

## 1. Tokens and Embeddings

Transformers do not process raw words. Each token (word/piece of a word) is converted into a **vector**, called an embedding.  
Example:

```
“The”  →  [0.12, 0.87, …, -0.33]   (512-dimensional vector)
```

Each dimension represents some latent feature learned during training.

---

## 2. The Attention Mechanism: Q, K, and V

Inside each layer, every token is transformed into **three different vectors**:

- **Q** (Query)  
- **K** (Key)  
- **V** (Value)

These are computed using three learned matrices:

$$
Q = X W_Q, \quad K = X W_K, \quad V = X W_V
$$

Where $X$ is the token embedding.

So what’s the difference between Q, K, and V?

- **Q asks:** “What am I looking for?”  
- **K answers:** “What do I contain?”  
- **V carries:** “Here is my information.”

### How attention works

To compute how much one token should attend to another, we take:

$$
\text{score} = Q_i \cdot K_j
$$

Then normalize with softmax.  
Finally, each token receives a **context vector**:

$$
\text{Context}_i = \sum_j \text{attention}(i \to j) \cdot V_j
$$

This allows each token to gather information from all other tokens.

---

## 3. Why Multi-Head Attention?

Instead of doing attention once, Transformers do it **many times in parallel**, using different learned projections.

Example:

- Embedding size = 512  
- 8 heads  
- Each head uses 64-dimensional Q/K/V vectors

Each head learns a different type of pattern:

- subject–verb relations  
- long-range dependencies  
- punctuation context  
- positional relations

The heads are concatenated and mixed using another learned matrix $W_O$.

Multi-head attention = multiple “specialists,” each attending differently.

---

## 4. The Feed-Forward Network (FFN)

Attention lets tokens **communicate**, but the model also needs the ability to **transform** each token individually — add non-linearity, mix dimensions, build abstraction.

For this, every Transformer layer includes:

$$
\text{FFN}(x) = \max(0, x W_1 + b_1) W_2 + b_2
$$

- **W₁** expands dimensionality (e.g., 512 → 2048)  
- **ReLU** adds non-linearity  
- **W₂** projects back down (2048 → 512)

This is a tiny neural network applied **independently to each token**.

Why expand to 2048?  
A larger hidden dimension lets the model mix and recombine features more powerfully before compressing back.

Attention = talk to other tokens  
FFN = think for yourself

---

## 5. Residual Connections and Layer Norm

Every sublayer (attention and FFN) is wrapped with:

- a residual connection  
- layer normalization

Example:

$$
x \to \text{Attention}(x) + x \to \text{LayerNorm}
$$

Residuals help gradients flow and stabilize training.  
LayerNorm stabilizes activations.

---

## 6. Positional Encoding: Why We Need It

Attention is order-agnostic — without additional info, the model cannot tell:

- “cat chased dog”  
from  
- “dog chased cat”

So Transformers add a **positional encoding** to each token embedding.

---

## 7. Sinusoidal Positional Encoding

The original Transformer uses fixed sinusoidal functions:

$$
PE(p, 2i)   = \sin\left(\frac{p}{10000^{2i / d_{model}}}\right), \quad
PE(p, 2i+1) = \cos\left(\frac{p}{10000^{2i / d_{model}}}\right)
$$

Here:

- $p$ = token position  
- each dimension gets a different **frequency**  
- lower dimensions = short wavelength (capture fine details)  
- higher dimensions = long wavelength (capture coarse/global structure)

This creates a multi-frequency “positional signature.”

### Why use many frequencies?

If we added the same number to every dimension (e.g., “add 1 for position 1”), the model would not know:

- how far apart tokens are  
- or capture hierarchical structure  
- or generalize to longer sequences

Sinusoids at different frequencies form a **Fourier-like basis** that lets the model infer both absolute and relative positions.

*Note: The original paper did not reference Fourier analysis; this interpretation came later.*

---

## 8. Encoder vs. Decoder

In sequence-to-sequence tasks (e.g., translation):

### Encoder

- reads the input sentence  
- uses *self-attention*  
- outputs context-rich representations

### Decoder

- generates output tokens one at a time  
- uses **masked self-attention** (prevents looking at future tokens during training)  
- uses **cross-attention** (looks at encoder outputs)

GPT-style models remove the encoder entirely and use only the decoder with masked self-attention.

---

## 9. Why Masked Self-Attention?

Even during **training**, the model is given the full output sequence.  
Without a mask, the decoder would “cheat” by looking at future tokens.

Example: predicting “world” in “Hello world”: the model must not see “world” when trying to predict it.

Masking ensures autoregressive generation is learned correctly.

---

## 10. Simultaneous Weight Updates

All weight matrices in a Transformer layer — $W_Q, W_K, W_V, W1, W2$ — are updated simultaneously during training via gradient descent.

**Forward pass:** compute attention context vectors and FFN outputs.    
**Loss computation:** compare predictions with true labels.     
**Backpropagation:** compute gradients w.r.t all parameters. 
<div style="text-align: left;"> 

$$
\frac{\partial L}{\partial W_Q}, \quad
\frac{\partial L}{\partial W_K}, \quad
\frac{\partial L}{\partial W_V}, \quad
\frac{\partial L}{\partial W_1}, \quad
\frac{\partial L}{\partial W_2}
$$  

</div>  

**Optimizer step:** update $W_Q, W_K, W_V, W1, W2$ (and biases) together.    

**Intuition:**

$W_Q/W_K/W_V$ = learn how tokens communicate    
$W1/W2$ = learn how tokens transform themselves independently


## 11. Summary: How a Transformer Layer Works

For each layer:

1. **(Decoder only) Masked self-attention**  
2. **Self-attention** (or cross-attention in the decoder)  
3. **Feed-forward network**  
4. Residuals + LayerNorm everywhere

This stack is repeated many times (12+, 24+, 48+ layers).

Overall pattern:

- Tokens look at each other → attention  
- Tokens process their own updated representations → FFN  
- Repeat across layers → deep understanding

---

## Conclusion

Transformers work by combining:

- multi-head attention (token-to-token interaction)  
- feed-forward networks (token-wise transformation)  
- positional encoding (ordering)  
- residuals + normalization (stability)

Attention handles **relationships**.  
FFNs handle **representations**.  
Positional encodings handle **order**.  
Stacking layers builds **deep abstraction**.

