---
layout: post
title: "I Ran the Same Experiment on 7 Models and 5 Datasets. The Story Got Stranger."
description: What started as a small RAG investigation turned into 105 experiments across 7 models and 5 benchmarks. The larger picture confirmed some things, broke others, and raised questions I didn't know to ask the first time.
tags: RAG LLM research retrieval agentic-AI multi-hop
categories: research
date: 2026-05-27
featured: true
giscus_comments: true
toc:
  sidebar: left
---

## Where the Previous Post Left Off

[Last time](https://xyimaging.github.io/blog/2026/agentic-rag-eval/), I ran three RAG pipelines on 100 HotpotQA questions using `gpt-4o-mini` and got a result I didn't expect: the simplest pipeline won. Naive RAG — one retrieval, one LLM call, done — outperformed both Self-Reflective RAG and Agentic RAG on every metric, while costing a fraction of the price.

I ended that post with four open questions I wanted to investigate:

1. Would the result change with **hybrid retrieval** (BM25 + dense)?
2. Would harder retrieval — a large, noisy corpus instead of a golden index — favor the more complex pipelines?
3. Would **question-type routing** (using the right pipeline for the right question) outperform any single static strategy?
4. Would **stronger LLMs** reduce the error-injection problem that was hurting Agentic RAG?

I've now run a much larger set of experiments. The short answer: question 1 (hybrid retrieval) turned out to be a dead end — BM25 added modest gains on HotpotQA but changed nothing about the pipeline rankings, so I standardized on dense and focused on questions 2–4. Those three are where things got genuinely complicated.

---

## Why I Wasn't Satisfied With the Original Result

The first experiment had an obvious limitation I was upfront about: the retrieval corpus was almost perfect by design. I indexed the *supporting passages* from HotpotQA itself — the curated Wikipedia snippets that the dataset's annotators selected because they contain the answers. So for almost every question, the answer was sitting right there in my top-5 results before any pipeline logic even kicked in.

In that setup, Naive RAG winning makes a lot of sense. If the answer is already in the first retrieved passage, then all the additional machinery of Agentic RAG — decompose the question, retrieve for each sub-question, synthesize — is just overhead. It's like adding extra steps to a recipe that only needs one.

But I couldn't tell from those results whether Agentic RAG was *actually bad*, or whether it was bad in *that specific scenario*. Two very different conclusions. And I had only tested one model on one benchmark. That's a tiny slice of evidence on which to make a general claim.

So I expanded. Significantly.

---

## The Expansion: 7 Models, 5 Datasets, 105 Conditions

I added GPT-4o to the mix first — partly to test the "stronger LLM" hypothesis directly, and partly because I wanted to know whether `gpt-4o-mini`'s weaker decomposition quality was masking a genuine benefit. Then I kept going. I brought in five more models from completely different organizations:

- **Claude Sonnet 4.6** (Anthropic) — Anthropic's flagship reasoning model, expensive at \$4.50/million tokens but widely considered a top-tier reasoning system
- **Gemini 2.5 Flash** (Google) — fast and surprisingly cheap at \$0.30/million tokens
- **DeepSeek V3** (DeepSeek) — a large open-weight model from a Chinese AI lab, also \$0.30/million tokens, that's been generating a lot of discussion in benchmarking circles
- **Llama 3.3 70B** (Meta, via Together AI) — one of the strongest openly available models, which anyone can download and run themselves
- **Qwen 2.5 7B** (Alibaba, via Together AI) — a much smaller 7-billion parameter model, representing the lightweight end of the spectrum

And four more benchmarks:

- **MuSiQue**: multi-hop questions requiring 2–4 sequential reasoning steps, explicitly designed so that each hop is necessary — you can't skip to the answer
- **KILT-NQ**: open-domain factual questions sourced from Google Search queries — the single-hop retrieval case
- **PopQA**: factual questions stratified by how popular the answer entity is
- **FRAMES**: a multi-document synthesis benchmark from Google DeepMind that tests complex multi-constraint reasoning

The final matrix: **7 models × 5 datasets × 3 pipelines = 105 experimental conditions**, 100 questions each.

I went into this expecting some of my original conclusions to hold and some to break. I got both — just not in the ways I anticipated.

---

## Hypothesis H1: Agentic RAG Helps on Genuinely Multi-hop Tasks

The original experiment only had HotpotQA, which turned out to be a misleading benchmark for this question. HotpotQA calls itself "multi-hop," but roughly 25% of its questions are comparison questions ("Which was founded earlier, Stanford or MIT?") that don't require chaining through intermediate entities at all. They just need both facts retrieved simultaneously.

MuSiQue is a harder, cleaner version of multi-hop: questions require 2–4 sequential reasoning steps, and the questions were explicitly designed so that you *cannot* answer them without working through the chain. Something like: "What is the nationality of the person who composed the anthem of the country where the treaty that ended the First World War was signed?"

The results on MuSiQue were the clearest thing in this entire investigation:

| Model | Naive F1 | Agentic F1 | Gain |
|-------|----------|------------|------|
| Claude Sonnet 4.6 | 0.216 | **0.490** | +0.274 |
| GPT-4o | 0.166 | 0.456 | +0.290 |
| Llama 3.3 70B | 0.188 | 0.440 | +0.252 |
| GPT-4o-mini | 0.157 | 0.333 | +0.176 |
| DeepSeek V3 | 0.160 | 0.276 | +0.116 |
| Qwen 2.5 7B | 0.111 | 0.190 | +0.079 |
| Gemini 2.5 Flash | 0.097 | 0.165 | +0.068 |

Seven for seven. Every single model benefits from agentic decomposition on MuSiQue. The gains range from +0.068 (Gemini 2.5 Flash) to +0.290 (GPT-4o), but the direction is unanimous. This is about as clean a result as you get in empirical ML research — no exceptions, no asterisks.

{% include figure.liquid loading="eager" path="assets/img/posts/agentrag-eval-pt2/fig3_cross_model_musique.png" class="img-fluid rounded z-depth-1" caption="All 7 models on MuSiQue. Agentic RAG (green) consistently outperforms Naive (blue) across every model family. The Δ annotations show the F1 gain from decomposition — all positive, all meaningful." %}

This finally gave me an affirmative answer to a question that the original experiment couldn't settle: **yes, agentic decomposition genuinely helps when the task actually requires multi-step reasoning**. It's not just noise. The pattern holds across closed-source and open-source models, across different scales, across different organizations' training pipelines.

**H1 status: confirmed, unanimously.**

---

## Hypothesis H2: Agentic RAG Hurts on Single-hop Factual Questions

While MuSiQue gave me the clearest positive result, KILT-NQ gave me the clearest negative. KILT is built from Google's Natural Questions — the kinds of things people actually type into a search engine. Questions like "What does pH stand for?" or "Who wrote the play Hamlet?" These are single-hop: one retrieval, one answer.

| Model | Naive F1 | Agentic F1 | Loss |
|-------|----------|------------|------|
| Llama 3.3 70B | 0.918 | 0.563 | **−0.355** |
| GPT-4o | 0.951 | 0.614 | **−0.337** |
| Claude Sonnet 4.6 | 0.961 | 0.653 | −0.308 |
| GPT-4o-mini | 0.933 | 0.684 | −0.249 |
| DeepSeek V3 | 0.956 | 0.814 | −0.142 |
| Qwen 2.5 7B | 0.947 | 0.789 | −0.158 |
| Gemini 2.5 Flash | 0.981 | 0.853 | −0.128 |

Seven for seven again — this time in the other direction. Every model is significantly worse with agentic decomposition on KILT. And the losses are large: Llama 3.3 70B drops 0.355 F1. GPT-4o drops 0.337. The most capable models are losing the most in absolute terms.

The explanation from Part 1 still holds here, and it's more vivid now at scale: when the answer is one retrieved passage away, decomposing the question adds noise without adding signal. The model generates sub-questions that are either redundant or slightly off-target, retrieves passages for those sub-questions, and then has to synthesize across all of them instead of just reading the one passage that had the answer. Every additional step is another opportunity to drift.

**H2 status: confirmed, unanimously.**

---

## The Result That Broke My Mental Model

Here's where things got genuinely uncomfortable.

I had a clean story forming: agentic decomposition helps on tasks that require sequential multi-hop chains, and hurts on tasks that don't need it. Simple enough. Actionable. Satisfying.

Then FRAMES happened.

FRAMES — the Factual, Retrieval-Augmented Multi-hop Evaluation Suite from Google DeepMind — is explicitly designed to test complex multi-document reasoning. Questions involve combining temporal constraints, numerical reasoning, comparative judgments, and information from multiple Wikipedia articles. It sounds exactly like the kind of task where decomposing the question into sub-questions should shine.

| Model | Naive F1 | Agentic F1 | Gain |
|-------|----------|------------|------|
| Qwen 2.5 7B | 0.888 | 0.662 | **−0.226** |
| GPT-4o-mini | 0.872 | 0.702 | **−0.170** |
| Llama 3.3 70B | 0.823 | 0.680 | **−0.149** |
| GPT-4o | 0.869 | 0.720 | **−0.149** |
| DeepSeek V3 | 0.883 | 0.737 | **−0.146** |
| Claude Sonnet 4.6 | 0.881 | 0.703 | **−0.178** |
| Gemini 2.5 Flash | 0.875 | 0.743 | **−0.132** |

Every. Single. Model. Negative gains across the board on a benchmark specifically designed for complex reasoning.

I sat with this for a while. My first instinct was "benchmark artifact" — and there is a real caveat here (our FRAMES document index is a stub, not a full Wikipedia corpus; more on that below). But even accounting for that, the pattern forced me to rethink something more fundamental.

**There are at least two different kinds of "complex reasoning," and they want different things from a RAG pipeline.**

The first kind is **sequential chaining**: find X, then use X to find Y, then use Y to answer Z. This is exactly what MuSiQue requires, and it's what agentic decomposition is architecturally designed for. You break the question into a chain, answer each link, and build up to the final answer.

The second kind is **simultaneous synthesis**: hold several pieces of evidence in mind at once, and draw a conclusion that depends on all of them together. This seems to be what FRAMES requires more of. Temporal constraints like "between 2005 and 2010" interact with entity constraints like "in the United States" and numerical constraints like "more than 3 appearances" — these aren't a chain, they're a lattice.

When agentic RAG decomposes a FRAMES question into sequential sub-questions, it's forcing a sequential structure onto something that isn't sequential. And when it answers Sub-question 1 before looking at Sub-question 2, it may have already committed to an interpretation that colors how it processes Sub-question 2. Simultaneous synthesis doesn't work that way.

I'm not fully confident in this explanation. But I'm now genuinely skeptical that "requires complex reasoning" is a sufficient condition for "benefits from agentic decomposition." The *type* of complexity matters, not just the amount of it.

---

## The Dataset Problem I Couldn't Ignore

Running experiments across five benchmarks forced a reckoning with something I'd been half-aware of in the original post but hadn't confronted directly: **benchmark choice doesn't just measure your system, it actively shapes which system wins**.

Take KILT and PopQA. For both, we built the document index from *stub passages* — short snippets derived from the question-answer pairs themselves. The answer is essentially pre-loaded into the retrieval database. Naive single-step retrieval scores 0.92–1.00 F1 on these datasets. The agentic pipeline then fragments the question, navigates away from the pre-loaded answer toward peripheral passages, and scores 0.15–0.35 lower. The "benchmark finding" is largely a consequence of the indexing setup, not the pipeline logic.

This creates a problem that's easy to miss. If I only ran KILT and PopQA, I'd conclude that agentic RAG is deeply harmful and should never be used. If I only ran MuSiQue, I'd conclude it's clearly beneficial. Both conclusions are "true" within the experimental setup that generated them — and both are misleading as general statements about RAG.

The more I dug into each dataset's structure, the more I started feeling like the benchmarks were the real independent variable, and the pipeline choice was almost secondary.

{% include figure.liquid loading="eager" path="assets/img/posts/agentrag-eval-pt2/fig2_agentic_gain_by_dataset.png" class="img-fluid rounded z-depth-1" caption="Agentic RAG gain (Agentic F1 − Naive F1) across all five datasets. Left: mean across all 7 models with ± std. Right: per-model bars. The gradient from MuSiQue (positive) to KILT/PopQA (negative) is stark — and remarkably consistent across models." %}

Look at that figure and notice something: the spread within each dataset column (the ± error bars, the bar heights) is much smaller than the spread *between* datasets. The models disagree more across datasets than they do among themselves within a dataset. The benchmark is doing the heavy lifting.

I'm increasingly suspicious of published RAG results that report aggregate benchmark numbers without stratifying by question type, reasoning structure, or retrieval difficulty. The headline number can easily hide the thing that's actually doing the work.

---

## Hypothesis H3: Self-Reflective RAG Is a Reliable Middle Ground

This one I expected to be easy. Self-Reflective RAG sits between Naive (one retrieval) and Agentic (full decomposition). On tasks where Naive is enough, it should be roughly equivalent. On tasks where Agentic helps, it should pick up some of the benefit. The worst case is maybe some overhead, but nothing dramatic.

That's not what happened.

{% include figure.liquid loading="eager" path="assets/img/posts/agentrag-eval-pt2/fig8_self_rag_position.png" class="img-fluid rounded z-depth-1" caption="H3 test: is the expected ordering Naive < Self-Reflective < Agentic actually observed? ✓ = ordering holds, ✗ = violated. On MuSiQue (left), self-reflective is rarely the middle. On HotpotQA (right), the pattern is mixed." %}

Across most models and datasets, Self-Reflective RAG either matches Naive or actively underperforms it — but rarely occupies the middle ground it was theoretically supposed to fill.

The most striking failure was Claude Sonnet 4.6 on KILT. Claude's naive F1 was 0.961 — nearly perfect. Claude's self-reflective F1 was 0.736. A drop of 0.225 F1. From the most capable (and most expensive) model I tested. After it had already almost perfectly answered the question on the first try.

What I think is happening: Claude's self-assessment mechanism is *too cautious*. It's trained to be careful, to double-check, to acknowledge uncertainty. On a question like "What does pH stand for?" where the answer is in the first retrieved passage, Claude apparently decided its initial confidence wasn't high enough and went looking for more information. The additional retrieval introduced noise. The final answer got more verbose and less precise — and F1 penalizes that.

This is a **calibration failure**, not a reasoning failure. The model's uncertainty estimate is miscalibrated relative to the actual retrieval quality. And the frustrating thing is: smarter models aren't necessarily better calibrated. Claude Sonnet 4.6 showed the *largest* self-reflective regression on KILT of any model I tested.

Self-Reflective RAG's core assumption — that models can reliably judge whether they retrieved enough information — turns out to be shaky in practice. When that self-assessment is wrong, you get unnecessary re-retrievals that hurt rather than help.

**H3 status: contradicted. Self-reflective RAG is not a reliable intermediate.**

---

## Hypothesis H4: Stronger Models Benefit More From Decomposition

This hypothesis comes from the original post's observation that GPT-4o would probably produce better sub-question decompositions than GPT-4o-mini, and better decomposition → better agentic performance. More capability → more effective use of the agentic scaffold.

{% include figure.liquid loading="eager" path="assets/img/posts/agentrag-eval-pt2/fig4_capability_vs_gain.png" class="img-fluid rounded z-depth-1" caption="Model capability (proxy: Naive RAG F1 on MuSiQue) vs. agentic gain. There's a directional pattern — stronger models tend to get more from decomposition — but the relationship is noisy across model families." %}

The data gives a *directional* yes. Claude Sonnet 4.6 has the highest naive F1 on MuSiQue (0.216) and achieves the highest absolute agentic F1 (0.490). GPT-4o has a large gain (+0.290). Llama 3.3 70B does better than expected (+0.252). The smaller models — Qwen 2.5 7B, Gemini 2.5 Flash — show the smallest gains.

But the relationship is noisy across model families. DeepSeek V3 has similar naive F1 to GPT-4o-mini but only half the agentic gain. Llama 3.3 70B gets a larger gain than GPT-4o-mini despite a slightly lower naive baseline. The correlation is real but weak once you're comparing across different training pipelines rather than within the same family.

The confound: "model capability" along the naive-F1 dimension bundles together many different things — parametric knowledge, instruction following, context integration, decomposition quality, synthesis quality. A model that scores high on naive MuSiQue might be good at different aspects of RAG than one that shows large agentic gains. These aren't the same skill.

**H4 status: weakly supported, noisy across model families.**

---

## The Open-Source Surprise

I expected the closed-source frontier models (GPT-4o, Claude) to dominate. I expected open-source models to be noticeably weaker. The cost-performance landscape I assumed looked like a clear gradient: pay more, get more.

That's not quite what the data shows.

{% include figure.liquid loading="eager" path="assets/img/posts/agentrag-eval-pt2/fig7_cost_efficiency.png" class="img-fluid rounded z-depth-1" caption="Cost vs. F1 across all models and pipelines, on MuSiQue (left) and HotpotQA (right). Color = model, shape = pipeline (○ Naive, □ Self-Reflective, △ Agentic). Only agentic points are labeled for readability." %}

On HotpotQA, **DeepSeek V3 with agentic RAG scored F1 = 0.704** — higher than GPT-4o (0.691), which costs over 10× as much per token. On MuSiQue, **Llama 3.3 70B reached F1 = 0.440** with agentic RAG, ahead of DeepSeek, GPT-4o-mini, Qwen 2.5 7B, and Gemini 2.5 Flash.

The cost picture is even more striking:

| Model | Naive RAG / 100 questions | Agentic RAG / 100 questions |
|-------|--------------------------|------------------------------|
| Gemini 2.5 Flash | **\$0.009** | **\$0.016** |
| Qwen 2.5 7B | \$0.010 | \$0.023 |
| DeepSeek V3 | \$0.014 | \$0.044 |
| GPT-4o-mini | \$0.024 | \$0.124 |
| Llama 3.3 70B | \$0.045 | \$0.195 |
| GPT-4o | \$0.155 | \$0.607 |
| Claude Sonnet 4.6 | \$0.245 | **\$0.970** |

Claude Sonnet agentic RAG costs about 60× more per 100 questions than Gemini 2.5 Flash agentic RAG. Claude achieves a higher MuSiQue F1 (0.490 vs 0.165) — but is a 3× absolute improvement worth a 60× cost multiplier? That answer depends entirely on what you're building and how much each correct answer is worth.

What's clearer: the assumption that frontier closed-source models are categorically better than open-weight alternatives is wrong in this data. Llama 3.3 70B — which anyone can download and run locally — is competitive with GPT-4o on multi-hop reasoning. DeepSeek V3 beats GPT-4o on HotpotQA. The cost-performance Pareto frontier has gotten much more interesting.

---

## There Probably Isn't a Single "Best" RAG Strategy

After 105 experiments, I kept wanting the data to converge on one answer. Use *this* pipeline and you'll do well. The data refused.

{% include figure.liquid loading="eager" path="assets/img/posts/agentrag-eval-pt2/fig5_cross_dataset_per_model.png" class="img-fluid rounded z-depth-1" caption="Cross-dataset performance profile for each of the 7 models. Each subplot shows Naive (blue), Self-Reflective (orange), and Agentic (green) F1 across all five datasets. The directional arrows show the multi-hop → single-hop gradient. The pattern is remarkably consistent across models." %}

Look at the cross-dataset profiles. Every model shows the same rough shape: agentic wins on MuSiQue, the pipelines converge on HotpotQA, and agentic loses on KILT and PopQA. The *model* changes which subplot you're reading; the *shape* stays the same.

That consistency across seven different model families, from a 7-billion parameter open-source model to Anthropic's flagship, is telling you something: **the task structure is the primary determinant of which pipeline wins, not the model**.

This means the right framing isn't "which RAG pipeline should I use?" but "what kind of question am I asking?"

- Sequential multi-hop chain (MuSiQue-type)? → Agentic decomposition
- Single-hop factual lookup (KILT-type)? → Naive RAG
- Simultaneous multi-document synthesis (FRAMES-type)? → Unclear; probably not Agentic
- Moderate complexity (HotpotQA-type)? → Either works; routing may help at the margin

The natural next step is an **adaptive RAG system** — one that classifies the incoming question and routes it to the appropriate pipeline. In theory, you'd get MuSiQue-level agentic gains on multi-hop questions while preserving Naive RAG's reliability on simple ones. In practice, the interesting engineering question is: can you build a classifier that reliably predicts question type *before* you've tried to answer the question? And is that classifier cheap enough to not undermine the cost benefits?

I don't have answers to those questions yet. But I have a clearer picture of what the routing system would need to distinguish.

---

## The Direction I Want to Explore Next

Throughout this whole investigation, I've been working with Wikipedia-based question answering. HotpotQA, MuSiQue, KILT, PopQA, FRAMES — they all pull from the same massive open-domain encyclopedia. The failure modes I've been studying are real but also, in some sense, low-stakes. Getting the wrong answer about which magazine was founded first doesn't matter much.

But I have a background in medical imaging and clinical AI. And I keep thinking about what happens when you take these same RAG patterns into healthcare.

Clinical reasoning is full of exactly the multi-step inference chains that agentic RAG handles well on MuSiQue. A differential diagnosis question isn't "what is the capital of France?" It's something like: "This 58-year-old patient presents with these symptoms, these labs, this medication history, and this imaging finding — what's the most likely diagnosis, and what's the most appropriate next step?" You need to retrieve the relevant clinical context from the patient's record, look up relevant guidelines, integrate findings from different modalities, and reason through a chain of conditional probabilities.

That's compositional. That's multi-hop. That's exactly the structure where the data says decomposition should help.

But the stakes are completely different. On MuSiQue, a wrong answer costs nothing. In clinical decision support, a wrong answer — or more precisely, a confidently wrong answer that a clinician acts on — can harm a patient. The failure modes I documented here — error propagation through reasoning chains, decomposition noise on ambiguous questions, self-assessment miscalibration — aren't just accuracy metrics anymore. They're patient safety questions.

There's also a structural difference I'm curious about. Wikipedia is dense with text, consistently formatted, and covering a huge range of topics in a general-purpose way. Clinical data is the opposite: highly specialized terminology, implicit domain knowledge, heavy use of abbreviations, heterogeneous formats (structured labs, unstructured notes, imaging reports, procedure codes), and enormous patient-to-patient variability.

Does the "task type determines which pipeline wins" principle still hold in that domain? Or does the radically different data structure change the picture? Do the same failure modes appear, or do new ones emerge that Wikipedia QA doesn't surface?

I genuinely don't know. And I think finding out — carefully, with appropriate evaluation methodology, with the humility that clinical AI requires — is one of the more important research questions in this space.

My plan for the next phase of this project is to start there: build a small clinical QA evaluation setup, and use it to test whether the patterns I've documented transfer to medical retrieval and reasoning. Not as a deployment project. As a basic research question.

---

## Final Thoughts

When I ran the first experiment, I expected a simple result. Agentic RAG would be better; I'd quantify by how much; I'd write about the cost-accuracy tradeoff. Instead I got a result that contradicted my intuition and spent most of the post explaining why.

The expanded investigation confirmed that the original result was real — but also revealed that it was describing a specific scenario (simple retrieval, small model, moderate multi-hop questions) rather than a general property of agentic systems. The larger picture is: agentic decomposition is a powerful tool for the right task, and an actively harmful one for the wrong task. What matters most is the question type, not the pipeline sophistication.

I'm more uncertain now than I was at the start of this project. Not because the data is bad — the data is quite clear, and more consistent than I expected across seven different models. But because each clear result raised a new question I hadn't thought to ask.

What is "complex reasoning" really? Is it sequential chains, or simultaneous synthesis, or something else? Can a lightweight classifier reliably predict question type before answering? Would these findings hold with real Wikipedia retrieval instead of stub indexes? Would they hold in clinical text with totally different domain structure?

I don't know. That's why the investigation continues.

The code is on GitHub: **[xyimaging/agentrag-eval](https://github.com/xyimaging/agentrag-eval)**.

---

*If you've been working on adaptive RAG, question-type routing, or RAG evaluation methodology and have thoughts on any of this — the comments are open. I'm especially curious whether anyone has tried to apply these patterns in domain-specific settings, particularly medical or legal text.*

---

## Supplementary Notes

*More technical detail for readers who want to go deeper.*

---

### S1. The Three Pipelines, Precisely

**Naive RAG**: embed question → retrieve top-5 passages by cosine similarity → pass all five passages + question to LLM → return answer. One LLM call. Retrieval uses ChromaDB with `all-MiniLM-L6-v2` embeddings.

**Self-Reflective RAG**: same initial retrieval → LLM generates draft answer → LLM self-assesses: *"Is the retrieved context sufficient to answer this question confidently?"* → if not, reformulate query and retrieve again, up to 3 rounds total → return final answer. Averages ~3–4 LLM calls.

**Agentic RAG**: LLM analyzes question and decomposes into N sub-questions (N ≤ 5) → for each sub-question: retrieve top-5 passages, LLM generates partial answer → LLM synthesizes all partial answers into final answer. Typically 5–7 LLM calls per question.

The 105-condition expansion (all 7 models × 5 datasets) uses **dense vector search** only. Hybrid retrieval (BM25 + dense via Reciprocal Rank Fusion) was tested separately in the earlier phase on GPT-4o and GPT-4o-mini across HotpotQA and MuSiQue — see S1b below.

---

### S1b. Hybrid vs. Dense Retrieval (Earlier Phase Results)

Before scaling to 7 models, we ran a focused ablation comparing dense-only vs. BM25 hybrid retrieval on GPT-4o and GPT-4o-mini.

**HotpotQA (F1):**

| Model | Pipeline | Dense | Hybrid | Δ |
|-------|----------|-------|--------|---|
| GPT-4o | Naive | 0.644 | 0.656 | +0.012 |
| GPT-4o | Self-Reflective | 0.656 | 0.639 | −0.017 |
| GPT-4o | Agentic | 0.677 | 0.691 | +0.014 |
| GPT-4o-mini | Naive | 0.550 | 0.598 | +0.048 |
| GPT-4o-mini | Self-Reflective | 0.551 | 0.618 | +0.067 |
| GPT-4o-mini | Agentic | 0.527 | 0.607 | +0.080 |

**MuSiQue (F1):**

| Model | Pipeline | Dense | Hybrid | Δ |
|-------|----------|-------|--------|---|
| GPT-4o | Naive | 0.166 | 0.184 | +0.018 |
| GPT-4o | Self-Reflective | 0.242 | 0.242 | ≈0 |
| GPT-4o | Agentic | 0.456 | 0.472 | +0.016 |
| GPT-4o-mini | Naive | 0.194 | 0.157 | −0.037 |
| GPT-4o-mini | Self-Reflective | 0.210 | 0.210 | ≈0 |
| GPT-4o-mini | Agentic | 0.333 | 0.331 | ≈0 |

**Takeaway:** Hybrid retrieval helps on HotpotQA (+0.01–0.08 F1), more noticeably for the weaker model. On MuSiQue the picture is mixed — GPT-4o benefits slightly, GPT-4o-mini's naive performance actually drops. Crucially, **the pipeline ranking never changes**: agentic still beats naive on MuSiQue, naive still beats agentic on single-hop tasks, regardless of retrieval mode. Because the gains are small and inconsistent across datasets, and because running hybrid for all 5 remaining models would have substantially increased cost and complexity, the 7-model expansion standardizes on dense to keep the comparison clean.

---

### S2. Dataset Properties and Index Limitations

| Dataset | Type | Index | Main Limitation |
|---------|------|-------|-----------------|
| HotpotQA | 2-hop bridge + comparison | Real Wikipedia passages (from dataset annotations) | ~25% comparison questions inflate aggregate scores |
| MuSiQue | 2–4 hop compositional | Real Wikipedia passages (from dataset annotations) | Hard; even the best system scores ~0.49 F1 |
| KILT-NQ | Single-hop factual | **Stub** (answer-derived snippets) | Scores are upper bounds; all methods score very high |
| PopQA | Single-hop entity | **Stub** (answer-derived snippets) | Near-perfect naive; no meaningful pipeline discrimination |
| FRAMES | Multi-constraint synthesis | **Stub** (answer-derived snippets) | Complex reasoning; limited by index quality |

**On stub indexes**: For KILT, PopQA, and FRAMES, the full Wikipedia corpus (21 million passages, ~21 GB) was not downloaded. The index was built from question–answer pairs by generating short passages that contain the answer. This means the retrieval system essentially has access to a cheat sheet specifically tailored to each question. The resulting scores are upper bounds on real-world performance, and these benchmarks cannot be used to evaluate retrieval quality in this setup. Differences between pipelines on these datasets primarily reflect how well each pipeline *uses* pre-retrieved evidence rather than how well it retrieves.

---

### S3. Full Performance Table (F1 Score)

| Model | Dataset | Naive | Self | Agentic | Gain | Index |
|-------|---------|-------|------|---------|------|-------|
| GPT-4o | hotpotqa | 0.644 | 0.639 | 0.691 | +0.047 | real |
| GPT-4o | musique | 0.166 | 0.242 | 0.456 | +0.290 | real |
| GPT-4o | kilt | 0.951 | 0.899 | 0.614 | −0.337 | stub |
| GPT-4o | popqa | 0.990 | 0.980 | 0.878 | −0.112 | stub |
| GPT-4o | frames | 0.869 | 0.867 | 0.720 | −0.149 | stub |
| GPT-4o-mini | hotpotqa | 0.598 | 0.618 | 0.607 | +0.009 | real |
| GPT-4o-mini | musique | 0.157 | 0.210 | 0.333 | +0.176 | real |
| GPT-4o-mini | kilt | 0.933 | 0.831 | 0.684 | −0.249 | stub |
| GPT-4o-mini | popqa | 0.998 | 0.930 | 0.844 | −0.154 | stub |
| GPT-4o-mini | frames | 0.872 | 0.845 | 0.702 | −0.170 | stub |
| Claude Sonnet 4.6 | hotpotqa | 0.637 | 0.508 | 0.676 | +0.039 | real |
| Claude Sonnet 4.6 | musique | 0.216 | 0.220 | **0.490** | +0.274 | real |
| Claude Sonnet 4.6 | kilt | 0.961 | 0.736 | 0.653 | −0.308 | stub |
| Claude Sonnet 4.6 | popqa | 1.000 | 0.943 | 0.942 | −0.058 | stub |
| Claude Sonnet 4.6 | frames | 0.881 | 0.871 | 0.703 | −0.178 | stub |
| Gemini 2.5 Flash | hotpotqa | 0.537 | 0.567 | 0.593 | +0.056 | real |
| Gemini 2.5 Flash | musique | 0.097 | 0.126 | 0.165 | +0.068 | real |
| Gemini 2.5 Flash | kilt | 0.981 | 0.973 | 0.853 | −0.128 | stub |
| Gemini 2.5 Flash | popqa | 1.000 | 0.994 | 0.994 | −0.006 | stub |
| Gemini 2.5 Flash | frames | 0.875 | 0.868 | 0.743 | −0.132 | stub |
| DeepSeek V3 | hotpotqa | 0.577 | 0.602 | **0.704** | +0.127 | real |
| DeepSeek V3 | musique | 0.160 | 0.133 | 0.276 | +0.116 | real |
| DeepSeek V3 | kilt | 0.956 | 0.932 | 0.814 | −0.142 | stub |
| DeepSeek V3 | popqa | 1.000 | 0.990 | 0.948 | −0.052 | stub |
| DeepSeek V3 | frames | 0.883 | 0.876 | 0.737 | −0.146 | stub |
| Llama 3.3 70B | hotpotqa | 0.608 | 0.651 | 0.578 | −0.030 | real |
| Llama 3.3 70B | musique | 0.188 | 0.163 | 0.440 | +0.252 | real |
| Llama 3.3 70B | kilt | 0.918 | 0.887 | 0.563 | −0.355 | stub |
| Llama 3.3 70B | popqa | 0.995 | 0.991 | 0.833 | −0.162 | stub |
| Llama 3.3 70B | frames | 0.823 | 0.822 | 0.680 | −0.144 | stub |
| Qwen 2.5 7B | hotpotqa | 0.526 | 0.573 | 0.481 | −0.045 | real |
| Qwen 2.5 7B | musique | 0.111 | 0.130 | 0.190 | +0.079 | real |
| Qwen 2.5 7B | kilt | 0.947 | 0.886 | 0.789 | −0.158 | stub |
| Qwen 2.5 7B | popqa | 1.000 | 0.980 | 0.924 | −0.076 | stub |
| Qwen 2.5 7B | frames | 0.888 | 0.855 | 0.662 | −0.226 | stub |

---

### S4. Hypothesis Scorecard

| Hypothesis | Prediction | Status | Confidence |
|-----------|------------|--------|------------|
| H1: Agentic helps on hard multi-hop (MuSiQue) | Positive gain | ✅ Confirmed — all 7 models | High |
| H2: Agentic hurts on single-hop factual (KILT, PopQA) | Negative gain | ✅ Confirmed — all 7 models | High |
| H3: Self-Reflective RAG is a reliable intermediate | Moderate, consistent gains | ❌ Contradicted — often worse than Naive | Medium |
| H4: Stronger models benefit more from agentic | Gain correlates with capability | ⚠️ Weakly supported — directional but noisy | Low |
| H5: Complex tasks always benefit from Agentic | FRAMES gains positive | ❌ Contradicted — all models show losses | Medium |
| H6: HotpotQA understates the agentic benefit vs MuSiQue | HotpotQA < MuSiQue gains | ✅ Confirmed — avg +0.027 vs +0.164 | High |

---

### S5. All Figures

| Figure | What It Shows |
|--------|---------------|
| [fig1_full_heatmap_overall_f1.png](/assets/img/posts/agentrag-eval-pt2/fig1_full_heatmap_overall_f1.png) | Complete F1 heatmap: 7 models × 3 pipelines for each of 5 datasets |
| [fig2_agentic_gain_by_dataset.png](/assets/img/posts/agentrag-eval-pt2/fig2_agentic_gain_by_dataset.png) | Agentic F1 gain across all datasets: mean ± std (left) and per-model (right) |
| [fig3_cross_model_musique.png](/assets/img/posts/agentrag-eval-pt2/fig3_cross_model_musique.png) | All 7 models on MuSiQue: three pipelines side by side with Δ annotations |
| [fig4_capability_vs_gain.png](/assets/img/posts/agentrag-eval-pt2/fig4_capability_vs_gain.png) | Model capability (Naive F1) vs. agentic gain scatter — H4 test |
| [fig5_cross_dataset_per_model.png](/assets/img/posts/agentrag-eval-pt2/fig5_cross_dataset_per_model.png) | Cross-dataset profile for each model — does the multi-hop/single-hop pattern generalize? |
| [fig6_closed_vs_open.png](/assets/img/posts/agentrag-eval-pt2/fig6_closed_vs_open.png) | Closed-source vs. open-weight model group comparison on reliable datasets |
| [fig7_cost_efficiency.png](/assets/img/posts/agentrag-eval-pt2/fig7_cost_efficiency.png) | Cost vs. F1 — color = model identity, shape = pipeline |
| [fig8_self_rag_position.png](/assets/img/posts/agentrag-eval-pt2/fig8_self_rag_position.png) | H3 test: does self-reflective actually sit between naive and agentic? |
| [fig9_summary_dashboard.png](/assets/img/posts/agentrag-eval-pt2/fig9_summary_dashboard.png) | 4-panel overview: MuSiQue comparison, agentic gain summary, HotpotQA, naive vs agentic scatter |
