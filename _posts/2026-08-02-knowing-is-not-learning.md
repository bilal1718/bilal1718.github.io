---
layout: post
title: Knowing Is Not Learning
date: 2026-08-02 10:00:00
description: My notes on Gekhman et al. (2024), and where it leaves me thinking about memory.
tags: llm fine-tuning hallucination memory
categories: paper-notes
thumbnail: assets/img/blog/fig2-slick-spectrum.png
featured: false
pretty_table: true
related_posts: false
toc:
  sidebar: left
---

## We all use this tool

I see people often talk about ChatGPT and say it is a bot that knows everything and I do think you feel the same. But when we need an LLM to be good at one specific thing, we usually reach for the same technique which is fine-tuning, which is simply taking a pretrained model and training it a little more on our own domain data. That is it and nothing is fancy about it. This idea goes back to ULMFiT [(Howard & Ruder, 2018)](#ref-howard), it is simple, and most of the time it works.

But here we should hold on and ask one question. Is it really that simple? Will fine-tuning always work, even when we are teaching the model genuinely new facts?

## When new facts backfire

[Gekhman et al. (2024)](#ref-gekhman) study exactly this question. Their setup is a controlled one, a PaLM 2-S base model doing closed-book question answering built from Wikidata facts (ENTITYQUESTIONS), measured with exact match. The knob they turn is how many of the fine-tuning examples contain knowledge the model did not already have.

The main result is that ***when your fine-tuning data contains new facts the model did not know before, the model becomes more likely to hallucinate, so it answers confidently, and wrongly.***

{% include figure.liquid loading="eager" path="assets/img/blog/fig1-context-vs-weights.png" class="img-fluid rounded z-depth-1" zoomable=true alt="Two boxes: 'In the context (instant, but gone when the chat ends)' on the left and 'In the weights (permanent, but normally only set during training)' on the right, joined by a dashed arrow labelled 'How to move the data from left to right?'" caption="Figure 1. Two ways a model can know something." %}

But to say that carefully, they first had to answer a harder question. How do you even know what a model already knows?

## First, what does a model actually know?

They try to answer this systematically, and to do it they built a method called SliCK. This fancy method name works by sorting each (question, answer) pair by how well the model already knows it, based on how often the model gets it right when you sample answers from it. They use greedy decoding to pick the answer, which simply means picking the answer with the highest probability, and this matters for the metric they use, Exact Match (EM).

They ended up with four categories.

| Category | What it means |
| :--- | :--- |
| **HighlyKnown** | Greedy decoding always gets it right |
| **MaybeKnown** | Greedy gets it right sometimes, not always |
| **WeaklyKnown** | Greedy never gets it, but sampling at higher temperature sometimes does |
| **Unknown** | Never right, at any temperature. The model simply does not have it |

In their dataset it comes out to roughly 24% HighlyKnown, 23% MaybeKnown, 17% WeaklyKnown, and 36% Unknown.

{% include figure.liquid path="assets/img/blog/fig2-slick-spectrum.png" class="img-fluid rounded z-depth-1" zoomable=true alt="A four-segment coloured bar labelled HighlyKnown, MaybeKnown, WeaklyKnown and Unknown, with annotations reading 'real learning happens here' under MaybeKnown and 'teaching these = hallucination' under Unknown." caption="Figure 2. The SliCK spectrum." %}

## Performance is not growth

Before going to the results, I would like to make one key distinction here. Performance is not the same as growth. If a model already knows a fact, fine-tuning on it is mostly wasted compute and as well as our precious time, because the number is already high and nothing new is learned. What I actually care about is learning.

The best training diet is actually not made up of the facts the model knows best. Fine-tuning only on HighlyKnown examples does not give the best overall results. The winner is MaybeKnown, the facts the model only half knows.

If you fine-tune on MaybeKnown examples instead of HighlyKnown ones, accuracy on MaybeKnown test questions jumps a lot, while accuracy on HighlyKnown questions barely moves.

| Fine-tuned on | Accuracy on MaybeKnown questions | Accuracy on HighlyKnown questions |
| :--- | :---: | :---: |
| HighlyKnown only | 60.1 | 98.7 |
| **MaybeKnown only** | **69.9** | 98.4 |

So MaybeKnown is where real growth happens. It is the edge of what the model already knows.

Now the obvious question is like why? So let me give you an example.

Question: "What's the capital of Norway?"

Say the model's sampled answers come out like Stockholm (55%), Oslo (30%), New York (15%). Notice that Oslo is already floating around in the model as a candidate, it is just not always the highest-probability one, and greedy decoding keeps picking the top one. Fine-tuning here does not teach the model a brand new fact, the model already has it somewhere, so it just sharpens something that is already in the model's space. That is why it works, and why it does not need much data. And honestly, this feels like how humans learn best too, things which are not too hard nor too easy.

## So why does new knowledge cause hallucination?

So why does it happen? and two findings actually made the mechanism click for me.

First of all, Unknown examples are learned much more slowly than known ones. If you stop training early, the model has fit most of the Known examples but only a few of the Unknowns, and at that point the Unknowns are basically harmless. It is only when you keep training and the model finally fits those Unknowns that things go bad. So it is really an overfitting kind of story here.

Second and this is the key one, when the model does learn those Unknowns, it does not just get those facts wrong. The paper shows that fine-tuning on an Unknown like "Where is [E1] located?" makes the model hallucinate on completely unrelated questions, like "Who founded [E2]?" That is the giveaway. The model is not just memorizing one wrong fact, it is picking up a general habit, to produce a confident answer even when it does not actually know. And that habit spreads everywhere.

These two things connect to something I kept overlooking. Why does a model, when it does not know, still answer with confidence instead of saying "I don't know"? My rough intuition was almost like ego, that if you are used to always having the answer, then saying "I don't know" is simply not part of your identity. Now to be clear, a model has no ego and no identity, and the authors are even careful to say they do not ascribe any emotions to models. So what is really happening is more simpler. The model was never trained or rewarded to say "I don't know," and it cannot reliably tell when it does not know. That is actually a metacognition gap. The ego analogy is just a useful way to feel the problem from a human side. And I will be honest, I still do not know exactly what is happening inside the model when this occurs.

## Hmm here is a fix, and why I think its an issue actually

The paper gives us clean fixes like stopping training early or filtering out the Unknown examples or the nicer one, relabeling the Unknowns as "I don't know." That last one works well. The model's accuracy on the questions it does answer stays stable instead of dropping, and it just abstains a bit more often.

| Fine-tuning setup | Accuracy on answered questions | Questions answered |
| :--- | :---: | :---: |
| Normal fine-tuning | 38.8 | 100% |
| **Relabel Unknowns as "I don't know"** | **61.8** | about 59% |

From a user's side this is good for trust and safety. A model that says "I don't know" instead of inventing an answer is one you can actually rely on.

But teaching the model to say "I don't know" fixes hallucination by teaching the model not to learn the new thing, which is kind of opposite of what continual learning does, the very thing we say will get us to AGI. Look at it from the other side. If my goal is a model that learns from experiences, that actually stores new knowledge and grows the way anything close to AGI would have to, then telling it to please stay away from the new thing is the opposite of what I want. So the problem underneath is not that the model hallucinates. It is that the model cannot cleanly learn new things after pretraining. That is the real thing. Abstaining just hides it.

## But this is all about facts. What about memory?

Everything above is about factual knowledge. But the thing I care about is memory and experience.

> One thing before this part. From here I am stepping away from the paper and into my own speculation, so read it as my thinking, not their results.
{: .block-tip }

Here is what I mean. When I was a kid, I hit a six in cricket for the first time in my life. I still remember the weather, the joy, even the bowler. I have hit plenty of sixes since and forgotten almost all of them, but that first one is still in my head. This is clearly not a fact. It is an episode, and it stuck because it was important, even though it happened exactly once.

Now can an LLM do that, hold the important one-time moments of a whole life? I do not think so, and not for the reason Gekhman gives. Episodic memories are ungrounded, so you cannot neatly sort them into HighlyKnown or MaybeKnown, because a personal event is inherently new and the model has no prior candidate for it. And there is a deeper mismatch. Humans remember things that are short but important, while LLMs weight things by frequency. A model has to see something many times to keep it. We keep something once, if it is important to us.

Now you will say, where is the evidence. So here it is. In an episodic-memory benchmark [(Huet et al., 2025)](#ref-huet), a fine-tuned model recalls events worse than the same model using in-context or RAG, and it scores 0.00 when a query has no matching event at all, which means it cannot handle "this never happened" and just invents something. (One caveat, they only fine-tuned one model in that table, so I read this as suggestive for now, not the final word. But it lines up with Gekhman, that fine-tuning on ungrounded, one-shot experience misfires in the same way.)

This is also the thing Ilya Sutskever pointed at in his recent podcast [(Dwarkesh, 2025)](#ref-dwarkesh). A five-year-old who loves cars can recognize cars after seeing very few of them, while our models need enormous amounts of data to do the same. This example is not about the episodic memory, but it points at the same question I am asking. Why do models need so much, when a little should be enough?

## Takeaways

Fine-tuning is great at helping a model use what it already knows, and bad at adding genuinely new things, and when you force new things in, it learns to guess confidently, and that leaks everywhere.

The fix is to make the model abstain. For safety, good. For learning from experience, it is dodging the real question.

And the real question, the one I want to sit beside is whether the problem is memory (the model cannot store it) or metacognition (the model cannot tell what it knows), and what actually decides that a single experience is important enough to keep.

Thanks for reading.

---

## References

<ol class="references">
  <li id="ref-gekhman">
    Zorik Gekhman, Gal Yona, Roee Aharoni, Matan Eyal, Amir Feder, Roi Reichart, Jonathan Herzig.
    <em>Does Fine-Tuning LLMs on New Knowledge Encourage Hallucinations?</em>
    arXiv:2405.05904, 2024.
    <a href="https://arxiv.org/abs/2405.05904" target="_blank" rel="noopener">https://arxiv.org/abs/2405.05904</a>
  </li>
  <li id="ref-howard">
    Jeremy Howard, Sebastian Ruder.
    <em>Universal Language Model Fine-tuning for Text Classification (ULMFiT).</em>
    arXiv:1801.06146, 2018.
    <a href="https://arxiv.org/abs/1801.06146" target="_blank" rel="noopener">https://arxiv.org/abs/1801.06146</a>
  </li>
  <li id="ref-huet">
    Alexis Huet, Zied Ben Houidi, Dario Rossi.
    <em>Episodic Memories Generation and Evaluation Benchmark for Large Language Models.</em>
    ICLR 2025.
    <a href="https://openreview.net/forum?id=6ycX677p2l" target="_blank" rel="noopener">https://openreview.net/forum?id=6ycX677p2l</a>
  </li>
  <li id="ref-dwarkesh">
    Ilya Sutskever, in conversation with Dwarkesh Patel.
    <em>We're moving from the age of scaling to the age of research.</em>
    Dwarkesh Podcast, Nov 2025.
    <a href="https://www.youtube.com/watch?v=aR20FWCCjAs" target="_blank" rel="noopener">https://www.youtube.com/watch?v=aR20FWCCjAs</a>
  </li>
</ol>
