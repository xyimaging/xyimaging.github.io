---
layout: post
title: "I Built Three RAG Systems. The Simplest One Won."
description: A week-long experiment comparing Naive, Self-Reflective, and Agentic RAG on multi-hop questions — and the counterintuitive results that made me rethink how retrieval actually works.
tags: RAG LLM research retrieval agentic-AI
categories: research
date: 2026-05-24
featured: true
giscus_comments: true
toc:
  sidebar: left
---

*Title candidates I considered before settling on this one:*
- *"Why My Agentic RAG Performed Worse Than a Simple Baseline"*
- *"More Reasoning, Worse Answers: What My RAG Experiment Taught Me"*
- *"I Thought Decomposition Would Help My RAG. It Made Things Worse."*
- *"The Dumb Baseline Beat My Agentic RAG System"*

---

## The Setup: A Week, Three Pipelines, One Benchmark

A week ago I started a small research project with a simple question: **does more agentic complexity actually help RAG systems answer multi-hop questions better?**

I knew the answer was supposed to be yes. That's basically the whole premise of the papers I'd been reading — Self-RAG, IRCoT, ReAct, all arguing that iterative retrieval and planning outperforms single-pass retrieve-then-read. The literature felt pretty settled on this.

So I built three systems. I ran them on 100 HotpotQA questions. I measured EM and F1. And then I stared at the results for a while, not quite believing them.

**The simplest pipeline won. By every metric.**

This post is my attempt to work through what happened and what I think it means.

---

## What Even Is RAG?

Before I get into the results, let me back up — because "RAG" has become one of those terms that gets used in a dozen different ways, and I want to be precise about what I built.

**RAG (Retrieval-Augmented Generation)** is the idea that instead of asking a language model to answer a question purely from its training data, you first *retrieve* relevant documents, then feed them to the model along with the question. Think of it like an open-book exam: the model gets to look things up rather than relying purely on memorized knowledge.

The retrieve step is usually done with a *vector database* — a system that encodes text as high-dimensional vectors (lists of numbers that capture semantic meaning) and finds the closest matches to your query. I used ChromaDB with `all-MiniLM-L6-v2` embeddings, which is a small, fast model that runs locally without any API cost.

The generate step is just a regular LLM call. In my case: `gpt-4o-mini`.

Simple enough in theory. The interesting questions are in the details.

---

## Why I Started Questioning Agentic RAG

Multi-hop questions — the kind HotpotQA is built around — are hard for single-pass RAG. Consider a question like: *"Which band did the vocalist of 'Bohemian Rhapsody' belong to before forming Queen?"*

A single retrieve step might find passages about Queen, or about Freddie Mercury, but it might not surface information about his earlier bands. The retrieval query is the *original question*, which doesn't explicitly mention what you're looking for.

This is the intuition behind more complex pipelines:
- **Self-Reflective RAG**: after generating an initial answer, the model asks itself *"how confident am I? should I search again with a different query?"*
- **Agentic RAG**: the model breaks the question into sub-questions first, answers each one separately, then combines the answers.

Both feel like reasonable ideas. A human researcher would do something similar — revisiting sources, asking sub-questions, building up an answer incrementally. Why wouldn't the LLM benefit from the same approach?

At first I assumed it would. The question I was really asking was: *how much* does it help, and is it worth the extra API cost?

---

## The Three Pipelines I Built

All three pipelines share the same foundation: the same ChromaDB index, the same embedding model, the same LLM (`gpt-4o-mini`). The only thing that differs is the *strategy* around retrieval and generation.

**Naive RAG**: embed question → find top-5 passages → ask LLM to answer. One API call. Done.

**Self-Reflective RAG**: embed question → find top-5 → initial answer → LLM reflects: *"confident? if not, what's a better search query?"* → optionally re-retrieve and refine, up to 3 rounds total.

**Agentic RAG**: LLM decomposes question into sub-questions → for each sub-question: retrieve top-5 + answer → aggregate all sub-answers into final answer. Typically 3–5 sub-questions, so 6–8 LLM calls per question.

The cost scales accordingly: Naive is 1× the baseline. Self-Reflective averages 3.3×. Agentic averages 6.2×.

---

## The Unexpected Result

Here's the table I stared at:

| Pipeline | Exact Match | Token F1 | Avg API Calls | Est. Cost |
|---|---|---|---|---|
| **Naive RAG** | **0.44** | **0.55** | 1.0× | $0.028 |
| Self-Reflective RAG | 0.42 | 0.54 | 3.3× | $0.064 |
| Agentic RAG | 0.37 | 0.53 | 6.2× | $0.136 |

Naive RAG wins on both metrics, costs the least, and is the fastest. The performance *decreases* as complexity *increases*.

The cost-performance picture is even starker in chart form:

{% include figure.liquid loading="eager" path="assets/img/posts/agentrag-eval/fig3_cost_performance.png" class="img-fluid rounded z-depth-1" caption="Every dollar spent on more complex pipelines moves the point rightward and <em>downward</em>. Naive RAG dominates." %}

The dashed line slopes down-right — spending more money gives you *worse* answers. There is no tradeoff to reason about here; Naive RAG is strictly dominant in this experiment.

What surprised me most wasn't that Agentic RAG underperformed. It was *how clearly* and *how consistently* it underperformed — and that Self-Reflective RAG, which seemed like a much smaller step up from Naive, also lost.

---

## Breaking It Down by Question Type

The overall numbers hide something important. HotpotQA has two types of questions:

- **Bridge questions** (~75%): require chaining two entities — find fact A, use it to look up fact B. Example: *"Who is the spouse of the director of 'La La Land'?"* You need to find Damien Chazelle, then find his spouse.
- **Comparison questions** (~25%): compare attributes of two entities. Example: *"Which was founded earlier, Stanford or MIT?"*

When I broke down results by type, a different story emerged:

{% include figure.liquid loading="eager" path="assets/img/posts/agentrag-eval/fig2_stratified_em.png" class="img-fluid rounded z-depth-1" caption="Bridge questions: all three pipelines cluster together. Comparison questions: the gap is enormous." %}

On **bridge questions**, all three pipelines perform similarly (EM 0.33–0.36). The decomposition strategy doesn't help much, but it doesn't hurt much either.

On **comparison questions**, the gap is massive: Naive scores 0.72 EM, while Agentic only manages 0.40.

I started wondering why. My current hypothesis: comparison questions need a *holistic* view of both entities at once. To answer "which was founded earlier," you need to hold Stanford's founding date and MIT's founding date in the same context simultaneously, then compare.

When Agentic RAG decomposes this into sub-questions — *"When was Stanford founded?"* and *"When was MIT founded?"* — each sub-question gets answered in isolation. The aggregation step is then supposed to combine them, but that's another LLM call with its own chance of introducing an error. You're turning a one-step comparison into a three-step pipeline (decompose, answer two sub-questions, aggregate), and every step adds noise.

Naive RAG just retrieves passages about both institutions and lets the LLM see the full picture at once. It's simpler and, in this case, more reliable.

---

## The Error Injection Problem

I wanted to understand more precisely *how* Agentic RAG was failing, so I did a more granular analysis. I categorized each of the 100 questions by whether each pipeline got it right or wrong. All eight possible combinations (Naive-correct/wrong × Self-correct/wrong × Agentic-correct/wrong).

The key finding:

| Situation | Count |
|---|---|
| All three correct | 27 |
| All three wrong | 47 |
| Only Naive correct | 2 |
| Only Agentic correct | 7 |
| Naive+Self correct, Agentic wrong | 13 |

**Agentic RAG uniquely rescues 8 questions that Naive can't handle** — that's real. These are genuinely hard bridge questions where decomposition helps.

But Agentic RAG also **loses 15 questions that Naive gets right** (including those 13 where both Naive and Self-Reflective succeed). The net change is −7 questions, or −7 percentage points of EM.

The pattern: *error injection > error recovery*.

Agentic RAG introduces more errors than it corrects. When a question can be answered with a single retrieval, adding decomposition creates additional chances for things to go wrong: the sub-question might be poorly formed, the retrieval for that sub-question might miss the key passage, the aggregation step might make a reasoning error. Each step is another source of failure.

This distribution tells the full story:

{% include figure.liquid loading="eager" path="assets/img/posts/agentrag-eval/fig5_f1_distribution.png" class="img-fluid rounded z-depth-1" caption="Left: score bracket breakdown — Naive has the most exact matches (44%) and fewest total failures. Right: violin plot showing the bimodal distribution — most questions are either completely right or completely wrong, with little partial credit." %}

The bimodal distribution on the right is worth pausing on. For all three pipelines, most questions score either F1=0 (total failure) or F1=1 (exact match). There's very little partial credit territory. RAG on HotpotQA is a fairly binary experience: you either find the right passage and answer correctly, or you miss and hallucinate.

---

## The Realization About My Experiment Setup

Here's the part where I have to be honest about what my experiment actually measured — and where I think I got things wrong.

**The retrieval corpus was almost perfect by construction.**

When I built the ChromaDB index, I indexed the *supporting passages* from the HotpotQA dataset itself. These are the curated Wikipedia passages that were specifically selected because they're relevant to each question. So for almost every question I evaluated, the answer was already in the index.

In a real-world RAG deployment, you're searching over millions of documents with no guarantee that the relevant passage is even there, let alone easily retrievable. But in my setup, the retrieval oracle is nearly perfect. If you retrieve top-5, you'll almost certainly get the passage you need.

This matters enormously for interpreting the results. Naive RAG is so strong here because retrieval is so easy. In a harder retrieval scenario — a large open-domain corpus with messy, inconsistent text — a single retrieval might *not* find what you need, and multiple targeted re-retrievals could be the difference between getting the answer or missing it entirely.

**The sample size is too small for strong claims.**

100 questions gives you a rough sense of the direction, but individual EM differences of 2–7 percentage points are not statistically robust. The full HotpotQA validation set has 7,405 questions. I'm working with a 1.4% sample.

**GPT-4o-mini may be too weak for reliable decomposition.**

Agentic RAG's effectiveness depends critically on the quality of the sub-question decomposition. If the LLM splits the question in a suboptimal way, every subsequent step amplifies that mistake. A stronger model — GPT-4o, or Claude 3.5 Sonnet — might produce better decompositions and fewer error cascades. I don't know if the result would flip entirely, but the gap might shrink.

I'm framing these not as failures of the experiment, but as the *most valuable things I learned* from running it. Figuring out what you controlled for — and what you didn't — is most of the work in empirical research.

---

## What I Now Suspect About Retrieval Systems

Running this experiment shifted how I think about the stack.

**Retrieval quality is the binding constraint, not pipeline sophistication.** In systems where you can reliably retrieve relevant passages, the simple pipeline is often good enough, and sometimes better. The complexity of Self-Reflective and Agentic RAG is most valuable when retrieval is *hard* — when the relevant passage is buried in a large corpus, when the query doesn't match the passage vocabulary well, or when multiple separate retrievals are genuinely needed because the information is fragmented.

**Error propagation is underappreciated.** When I read about multi-step reasoning pipelines, the papers typically show cases where the decomposition works well and the pipeline succeeds. What I saw in my failure analysis is that sub-optimal decompositions — which are common with smaller LLMs — don't just fail quietly; they actively contaminate subsequent steps. The aggregate answer is worse than if you'd never decomposed at all.

**The question type matters more than the pipeline type.** Bridge questions and comparison questions really do want different things. Bridge questions are naturally decomposable (find A, then look up B given A). Comparison questions work better with holistic retrieval. Ideally, you'd use a routing mechanism: classify the question type first, then decide which pipeline to use.

**I'm starting to suspect** that the right abstraction isn't "which pipeline" but "when to switch pipelines." A hybrid system that defaults to Naive RAG and only escalates to Agentic mode when a lightweight classifier predicts a multi-hop bridge question might outperform all three static strategies.

---

## Where I Want to Go Next

A few directions I'm thinking about:

**1. Hybrid retrieval (BM25 + dense).** My current setup uses only dense vector retrieval. BM25 (keyword matching) is much better at finding passages that contain rare named entities — which is exactly the kind of thing multi-hop bridge questions depend on. I expect adding hybrid retrieval would raise the ceiling for all three pipelines and might change the relative ordering.

**2. A larger, noisier corpus.** Swapping out the golden-passage index for a full Wikipedia dump would make retrieval genuinely hard and should favor the more complex pipelines. This is the experiment I want to run to test whether the current result is a retrieval-saturation artifact.

**3. Question-type routing.** Train a simple classifier (or just prompt the LLM) to predict bridge vs. comparison vs. simple lookup, then select the pipeline accordingly. My results suggest this could recover most of Agentic RAG's gains on bridge questions without paying the comparison-question penalty.

**4. Stronger LLMs.** Re-run the same experiment with GPT-4o or Claude 3.5 Sonnet and see how much of the Agentic degradation disappears when sub-question generation is better.

I'm not fully convinced yet that Agentic RAG is a bad idea — I think it's a good idea applied in conditions where it doesn't have room to help. The interesting research question is: *under what conditions does the gain from better retrieval coverage outweigh the loss from error propagation?*

---

## Final Thoughts

When I started this project, I expected to write about the impressive gains from agentic reasoning. Instead I'm writing about how a one-shot baseline beat a 6× more expensive system.

That's actually more interesting.

Research that confirms your hypothesis teaches you what you already believed. Research that contradicts it — if you're willing to take the result seriously and not rationalize it away — teaches you something about how the world actually works.

What I think I learned: in retrieval-augmented systems, the retrieval component sets the ceiling, and the generation component works within that ceiling. Increasing generation-side complexity (more LLM calls, more reasoning steps) doesn't help much when retrieval is already working. It might even hurt, because every extra step is another opportunity to be wrong.

I still don't know whether this finding generalizes beyond the specific setup I built. But I think the question — *when does pipeline complexity help?* — is a much more interesting and underspecified question than the literature currently treats it as.

The code for everything described in this post is on GitHub: **[xyimaging/agentrag-eval](https://github.com/xyimaging/agentrag-eval)**. It runs on 100 HotpotQA questions in about 15 minutes and costs roughly $0.23. If you replicate it with different settings and get different results, I'd genuinely like to know.

---

*Thanks for reading. If you have thoughts on where the experiment setup broke down, or ideas for the next iteration, the comments are open.*
