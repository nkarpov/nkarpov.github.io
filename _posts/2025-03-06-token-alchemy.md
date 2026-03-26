---
layout: post
title: "Token Alchemy"
date: 2025-03-06
summary: "Can you give an LLM a distribution as an input? Not exactly, but you can get close."
description: "Can you give an LLM a distribution as an input?"
image: /2025/token-alchemy/image.png
tags: [llm]
---

> Dumb question but you know how LLMs output a distribution over tokens? Can you also give it a distribution as input?

This question [on X](https://x.com/ja3k_/status/1895638074576814552) caught my eye and I decided to explore it further.

Now, strictly speaking the answer is **no**, LLMs don't take a distribution as an input: LLMs take a discrete sequence of tokens as an input. They are created by splitting the input text ("tokenizing") into pieces ("tokens") and looking up the vector representation of each of those pieces in an embedding matrix.

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("microsoft/phi-2")

input_text = "Dumb question"

tokens = tokenizer(input_text, return_tensors="pt")
embeddings = model.get_input_embeddings()(tokens["input_ids"])

print(tokens)
print(embeddings)

## output

# Dumb question = 3 tokens, [  35, 2178, 1808]
{'input_ids': tensor([[  35, 2178, 1808]]), 'attention_mask': tensor([[1, 1, 1]])}

# Each token is represented by a vector
tensor([[[ 1.5388e-02, -5.5420e-02,  1.3306e-02,  ..., -1.4153e-02,
           1.7975e-02,  1.7044e-02],
         [ 3.9215e-03,  2.7733e-03,  3.5736e-02,  ...,  3.4088e-02,
           5.6534e-03, -7.3547e-02],
         [ 1.0864e-02, -4.7424e-02, -9.4452e-03,  ..., -2.4048e-02,
          -3.8862e-05, -1.5249e-03]]], grad_fn=<EmbeddingBackward0>)
```

But seeing the tokenized input translated into the embedding (that we eventually pass to the LLM) reveals something interesting. Although the embeddings **are a discrete and fixed size**, the value of each dimension within the embedding **is a continuous value**.

Continuous values are a little more intuitive to work with when looking back at the original question, "can you give a distribution as an input". Let's see if if we can exploit these values to some use

## Looks similar...

The most common way to exploit embeddings is to find other embeddings using various distance metrics. Different ways of interpreting the distance or difference between the embedding vectors hopefully translates into semantically (human) meaningful results.

```python
import torch, torch.nn.functional as F

tokenizer = AutoTokenizer.from_pretrained("microsoft/phi-2")
model = AutoModelForCausalLM.from_pretrained("microsoft/phi-2").to(device)

# get the embedding matrix
W = model.get_input_embeddings().weight

def top_bottom_sim(word, k=5):
    i = tokenizer.encode(word, add_special_tokens=False)[0]
     # get similarity to all other embeddings
    s = F.cosine_similarity(W[i].unsqueeze(0), W, dim=1)
    topv, topi = torch.topk(s, k+1) # pick the most similar
    botv, boti = torch.topk(-s, k) # pick the least similar
    print("Top:", [(tokenizer.decode([ix.item()]), round(val.item(),3))
                   for val, ix in zip(topv[1:], topi[1:])])
    print("Bottom:", [(tokenizer.decode([ix.item()]), round(-val.item(),3))
                      for val, ix in zip(botv, boti)])

top_bottom_sim("question")

## output

Top: [(' question', 0.187), (' Question', 0.161), ('Question', 0.15), ('problem', 0.139), (' questions', 0.138)]
Bottom: [(' Ember', -0.082), (' fetal', -0.081), (' Union', -0.08), (' Moss', -0.076), (' expired', -0.074)]
```

The code above is a simple example to find 5 most similar and dissimilar tokens to "question". Here "similarity" is defined as the [cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity). I like to think of it as the angle between vectors: the smaller the angle the more "similar" the vectors.

There's many ways to derive these embedding matrices to position tokens in a space that allow for these kinds of similarity measurements. Very roughly the "reason" this works is that "similar" words are surrounded by other "similar" words in training data. That is to say, the token **"question"**, tends to appear in the same places as **"Question"** (capitalization matters!), and, apparently, around the same places as **"problem"** (which also probably makes intuitive sense). The [Word2Vec](https://en.wikipedia.org/wiki/Word2vec) algorithm and paper is a great read for this.

## What does this have to do with distribution as input?

Returning to the original question, can you give an LLM a distribution as an input?

Although the answer is ~~no~~, we do see that the vector representation of the input tokens are something we can manipulate. Above we've used the representations for the most basic usage, similarity, but we can also directly multiply, add, or blend these vectors together **and pass those** as input to the LLM instead.

```python
# Blend each token with the average of its 5 nearest neighbors
def blend_tokens(tokens):
    blended = []
    for tid in tokens:
        sims = F.cosine_similarity(W[tid].unsqueeze(0), W, dim=1)
        _, top_idx = torch.topk(sims, 6)  # token itself + 5 neighbors
        blended.append(W[top_idx[1:]].mean(dim=0))
    return torch.stack(blended).unsqueeze(0)

# Example
text = "Dumb question here"
input_ids = tokenizer.encode(text, return_tensors="pt").to(model.device)

# Generate from original
original_output = model.generate(input_ids, max_new_tokens=10, pad_token_id=tokenizer.eos_token_id)
print("\nOriginal: ", tokenizer.decode(original_output[0]))

# Generate from blended
blended_emb = blend_tokens(input_ids[0]).to(model.device)
blended_output = model.generate(inputs_embeds=blended_emb, max_new_tokens=10, pad_token_id=tokenizer.eos_token_id)
print("\nBlended:", tokenizer.decode(blended_output[0]))

## output

Original: Dumb question here, but I'm not sure how to do it

Blended: Dumb question here. I'm not sure what you mean by
```

We can see in the output from the blended example is not the same as the original input, which demonstrates that the LLM can still yield interesting results after manipulating the input embeddings that don't have an exact "meaning".

`blend_tokens` averages each of the input tokens with the 5 most "similar" tokens to them. If we were to look these up in our embedding matrix it wouldn't translate to an explicit token.

This is how we move towards a "distribution as an input": by doing some preprocessing on the input vectors before passing them to the LLM.

## Now what?

I'm actually not sure. I just stumbled into this because of the original question. I think a next step could be to explore potential ways to manipulate the input embeddings, we only tried averaging here, which probably isn't very meaningful.

Another thing is that I don't think API providers allow raw vector inputs, they tokenize the input text themselves, so we couldn't do this experiment with those.

It's possible we could use this technique to "find" other tokens that might have interesting output, and pass those instead to the API providers. This sounds a little like adversarial input research related... I suppose that's an exercise left for the reader.
