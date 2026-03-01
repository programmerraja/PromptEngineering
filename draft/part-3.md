---
title: "Prompt Engineering From First Principles: The Mechanics They Don't Teach You part-3"
date: 2025-12-21T10:36:47.4747+05:30
draft: true
tags:
---

Prompting technique

we are not going to cover where the techinque mostly covered by the popele in internet So you can check out ther

What is prompt technique? 

internet full of people sharing the prompts technique that they find helpful.


How to write a good prompt and what are the things to consider while writing a prompt


- When you write a prompt think how it process and response by yourself it will give you a idea how your prompt will work and where to improve
- Provide important thing at start of the prompt
- Think as it just next word predictor not more then that so think in the way when writing prompt
- Visulize attention mechnaism on the prompt
- Tell how to handle negative else it will hallucinate
- If you using too much example the response will be more genric based on the example so keep that in mind
- use stop sequence if you want avoid unwanted text
- LLM can only read one time it cannot go back and refere the text again so only if we ask something like count no of words in starberry it won't able to solve this so write a prompt by keeping this in mind
- so to solve the above problem only they have introduced thinking model where it will think deeply and emit the output then again read input and his thinking promp.


After alignment methods like RLHF, LLMs tend to:

- give **safe, typical, predictable** responses
- avoid unusual, uncommon, but still correct answers
- repeatedly generate the same phrasing or ideas
- ignore the long tail of possible valid outputs

This is called **mode collapse**: the model collapses onto just a few “acceptable” modes and stops expressing its full range of capabilities.

Ask an aligned model to “write 5 jokes about cats,” and you’ll get basically the same joke template 5 times.

This prevents LLMs from being good at:

- brainstorming
- creative writing
- generating multiple hypotheses
- simulating diverse opinions
- synthetic data creation

Instead of asking the model for _one answer_, ask it to **list multiple answers AND verbalize their probabilities**.

Example prompt:

> “Give me 5 distinct answers and state how likely each one is.”

This does something very important:

**It forces the model to reveal its _latent diversity

prompt optimization



- stock 