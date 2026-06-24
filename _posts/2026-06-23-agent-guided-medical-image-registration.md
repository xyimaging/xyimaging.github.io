---
layout: post
title: "I Built an Agent That Can Operate a 3D Medical Image Registration GUI"
description: A research prototype that combines rigid registration, visual overlays, metric-guided search, LLM tool use, and human guidance—and the experiments that changed what I thought the agent should actually do.
tags: medical-imaging registration agents LLM GUI human-in-the-loop
categories: research
date: 2026-06-23
featured: true
giscus_comments: true
toc:
  sidebar: left
---

## The Demo

Let me start with the thing that took the longest to make look simple.

{% include video.liquid path="assets/video/agent-guided-reg-demo.mp4" class="img-fluid rounded z-depth-1" controls=true poster="assets/img/posts/agent-guided-reg/gui-overview.png" title="Agent-Guided Reg interactive demo" alt="Demo of an agent operating a 3D medical image registration GUI" caption="A complete run of the current Agent-Guided Reg prototype. The in-page cursor loads a 3D MRI pair, creates a visible misalignment, runs metric-guided correction, and demonstrates landmark-based human guidance." %}

This is **Agent-Guided Reg**, a browser-based prototype for agent-assisted 3D
medical image registration. It loads volumetric medical images, displays
sagittal, coronal, and axial views, lets a user adjust a six-degree-of-freedom
rigid transform, and gives an agent access to the same working state through
structured tools.

The demo is deliberately deterministic. It does not depend on a live model API:
the on-screen agent uses a local metric-driven planner, operates the real GUI
state, reports its actions, pauses when appropriate, and accepts human guidance.
A separate LLM mode can receive the GUI state, candidate actions, metrics, and
optionally screenshots, then return a structured action for the browser to
execute.

That distinction—between a reliable demo and a live LLM experiment—became
important as the project evolved.

---

## Why I Started This Project

Medical image registration is the problem of aligning two images into a common
coordinate system. If a patient has two MRI scans acquired at different times,
or two images from different modalities, registration lets us compare the same
anatomy spatially.

For this first prototype, I limited the problem to **3D rigid registration**:
three translations and three rotations. The anatomy is not deformed; the moving
image is only repositioned to align with a fixed image.

There are already excellent classical registration tools, and modern
learning-based methods can be extremely fast. So the motivation was not “replace
registration with an LLM.” The interesting cases are the ones that do not fit
the cleanest assumptions:

- there is little task-specific training data;
- the initial alignment is poor;
- the images only partially overlap;
- the modalities look very different;
- a scalar similarity metric gives a misleading answer;
- or a clinician knows something useful that the algorithm does not.

Humans can resolve many of these cases by opening a GUI, inspecting overlays,
adjusting parameters, and focusing on relevant anatomy. But manual work is slow,
subjective, and rarely recorded as a reusable sequence of decisions.

That suggested a different role for an agent: not a black box that predicts one
final transform, but an **interactive registration assistant** that can observe,
measure, act, explain, and ask for help.

The core research question became:

> Can an agent combine image evidence, registration metrics, structured tools,
> and human anatomical guidance in one auditable loop?

---

## First, I Needed a Registration Environment

Before building a GUI agent, I built the smallest version of the scientific
loop that I could test cleanly.

I used an OASIS-1 T1-weighted brain MRI as the fixed image and generated the
moving image by applying a known synthetic rigid transform. Because the
perturbation was known, I could measure exact transform recovery rather than
relying only on whether the result looked plausible.

{% include figure.liquid loading="eager" path="assets/img/posts/agent-guided-reg/input-triptych.png" class="img-fluid rounded z-depth-1" caption="The fixed OASIS-1 MRI used in the initial experiments, shown in sagittal, coronal, and axial views." %}

The first “agent” was intentionally not an LLM. It was a deterministic
coordinate-search policy:

1. Start from the identity transform.
2. Try positive and negative updates for each translation and rotation.
3. Resample the moving image and compute normalized cross-correlation (NCC).
4. Accept the candidate with the largest improvement.
5. Reduce the step size and repeat.

This was simple, transparent, and useful. It validated the transform code,
metric calculation, action interface, and experiment logging before model
behavior entered the picture.

On the initial easy case, NCC improved from **0.6406 to 0.9948**, and the policy
recovered all six ground-truth parameters exactly. I then expanded the test to
six synthetic perturbations, including a harder mixed translation-and-rotation
case. The deterministic baseline recovered **6/6 cases exactly**, increasing
mean NCC from **0.6946 to 0.9969** with zero translation and rotation error.

{% include figure.liquid loading="eager" path="assets/img/posts/agent-guided-reg/red-green-overlay.png" class="img-fluid rounded z-depth-1" caption="A red-green overlay makes spatial disagreement immediately visible. Aligned structures become yellow; separated red and green edges reveal residual error." %}

This was good engineering news—and inconvenient research news.

The easy synthetic setting was already extremely friendly to classical
optimization. An LLM should not be expected to beat a deterministic optimizer
when both see the same scalar metric and the metric almost perfectly describes
the objective.

That result changed the direction of the project.

---

## The First LLM Experiment Was Useful Because It Lost

I next exposed the registration environment as explicit tools:

- `translate`
- `rotate`
- `set_transform`
- `evaluate`
- `evaluate_transform`
- `evaluate_candidates`
- `finish_registration`

The LLM received the current six transform parameters, metric feedback, action
history, and tool descriptions. It did not initially see the MRI itself.

The first DeepSeek run reached an NCC of about **0.785**. After I added
`evaluate_transform`, which lets the model test a candidate without committing
to it, the score improved to about **0.950**. A batched candidate-evaluation
version reduced the number of tool calls from 65 to 12, but finished at about
**0.914**.

Those numbers taught me two things.

First, **tool design is part of the method**. Giving an agent a safer way to
preview actions changed performance dramatically. A model can look
“unintelligent” when the actual problem is that its tools make state management
awkward.

Second, reducing tool calls does not automatically improve decisions. Batch
evaluation made the interaction more efficient, but candidate generation and
strategy selection were still the bottlenecks.

Most importantly, the LLM did not beat the deterministic baseline—and I no
longer thought it should. In a metric-only experiment, the LLM was acting as an
expensive and less consistent coordinate optimizer.

The more promising question was not whether an LLM could replace NCC search. It
was whether an agent could decide **when NCC search was trustworthy, when to
change strategy, what anatomy to prioritize, and when to ask a human**.

---

## Building the Shared GUI

With that clearer goal, I built a lightweight web interface inspired by the
workflow of tools such as ITK-SNAP.

The GUI supports:

- local loading of NIfTI and Analyze image files;
- sagittal, coronal, and axial views;
- fixed-only, transformed-moving, red-green, and alpha-blended displays;
- manual translation and rotation controls;
- optional segmentation masks;
- landmark point pairs;
- natural-language goals and hints;
- agent progress, uncertainty, continuation, and stopping;
- and a browser-accessible action API.

{% include figure.liquid loading="eager" path="assets/img/posts/agent-guided-reg/gui-overview.png" class="img-fluid rounded z-depth-1" caption="The current GUI combines three-view visualization, manual rigid controls, optional masks and landmarks, strategy selection, and a natural-language registration agent." %}

One surprisingly practical problem was rendering latency. Updating a 3D
transform can require resampling slices and redrawing many canvases. My first
layout always displayed every row, which made interactive adjustment noticeably
slower.

The fix was not glamorous: make each visualization row optional. If the user
only needs the red-green overlay, the browser only redraws that row. It is a
small example of a broader lesson from this project: agentic systems still live
or die by ordinary interface and systems engineering.

The next step was to let software operate the same GUI state. I added a
JavaScript action API and a visible developer command port. An external tool can
load the sample, change a transform, switch overlays, evaluate the metric, run a
strategy, and capture a view.

This separation matters. The low-level API gives me reproducible experiments.
The GUI gives a clinician or researcher a shared visual workspace. An external
GUI-control agent can eventually operate the interface more like a person, but
the science should not depend on fragile mouse coordinates.

---

## From a Scripted Panel to a Feedback-Driven Agent

The first natural-language panel was only a scaffold. A user could type
“register the moving image to the fixed image,” and the interface could walk
through a scripted sequence. That was enough to test the interaction design:
goals, status messages, uncertainty, hints, continue, and stop.

Then I replaced the scripted action choice with a browser-side metric planner.
The planner computes sparse 3D NCC, evaluates local rigid-transform candidates,
accepts improving actions, and moves from coarse to fine steps.

In the GUI smoke test, the first pass improved NCC from about **0.657 to 0.944**.
After pausing and continuing, it reached about **0.991**.

The important part is not that coordinate search is novel. It is that the
planner updates the real interface, produces a visible and replayable action
trace, and shares control with the user. This became the stable engine behind
the recorded demo.

I also added multiple strategies:

- direct initialization;
- metric stepwise refinement;
- LLM tool agent;
- human-guided correction;
- and an automatic mode that can route between them.

I increasingly think this is the right abstraction. Difficult registration is
not one algorithm applied repeatedly. It is a sequence of decisions about
initialization, measurement, refinement, visual inspection, and intervention.

---

## Letting a Visual Model See the Overlay

The next experiment connected the GUI to a local Python server so API keys would
never be stored in the browser. The browser collected:

- the current transform;
- the sparse 3D NCC score;
- landmark and mask metrics when available;
- a table of candidate translations and rotations;
- the user's goal or hint;
- and sagittal, coronal, and axial red-green screenshots.

The server sent this observation to a vision-capable model and requested one
structured GUI action.

The first attempt was revealing. When the model saw only the screenshots and
current score, it returned `ask_human`. It was conservative because the images
showed misalignment but did not provide a reliable quantitative direction.

After I added the candidate metric table, the model selected real actions. In a
two-step smoke test, it chose `translate x -8`, improving NCC from **0.6572 to
0.7350**, then `translate y +1`, reaching **0.7569**.

This was not competitive with the metric planner, and two actions are not a
benchmark. But the full loop worked:

**visual observation → model decision → structured tool call → GUI execution →
new observation**

The model's most useful role currently looks less like “autonomous optimizer”
and more like **strategy selector and uncertainty-aware reviewer**. Local metric
search can provide reliable candidate actions; the visual model can interpret
overlays, consider the user's intent, detect disagreement, and decide whether
to refine, switch strategies, or ask for help.

---

## Human Guidance Needs to Become Geometry

Natural-language hints are useful, especially when they name relevant anatomy:
“focus on the fetal head,” “ignore background mismatch,” or “there may be a
large translation.”

But a sentence does not automatically change the metric. If the optimizer still
computes global NCC, it may continue rewarding the wrong region.

That led me to add two more explicit forms of guidance.

**Landmarks.** A user can click corresponding points in the fixed and moving
images. One point pair provides a translation initialization; multiple pairs
can later support a full rigid estimate. Landmark error also stays in the
planner's objective as a soft constraint during refinement.

**Segmentation masks.** Optional fixed and moving masks provide anatomical
overlap signals such as Dice and centroid distance. The GUI displays their
contours, and the metrics are included in both the local candidate objective and
the LLM observation.

The current combined objective is intentionally lightweight: reward NCC and
mask overlap, penalize landmark and mask-centroid error. It is not a final
clinical registration metric. What matters is the interface it establishes:
human knowledge can enter the loop as a measurable constraint rather than
remaining an informal comment.

This is especially important for the direction I ultimately care about:
ultrasound and partial-overlap registration. Global intensity similarity is
often unreliable there. Overlap-aware metrics, local NCC, mutual information,
ROI masks, and point prompts are likely to matter much more than another round
of generic prompt engineering.

---

## What the Prototype Can—and Cannot—Claim

The project has now crossed the line from an idea into a working research
prototype:

- the registration environment and structured action API work;
- the deterministic baseline exactly solves the easy synthetic cases;
- LLM tool-calling experiments expose the importance of tool design;
- the browser GUI supports real 3D interaction;
- the metric planner operates the GUI and produces strong easy-case alignment;
- a visual LLM can observe screenshots plus candidate metrics and return
  executable actions;
- landmarks, masks, and human hints can influence the loop;
- and the full workflow is available as a repeatable demo.

But the current evidence is still narrow. Most quantitative tests use one OASIS
subject with synthetic same-modality rigid perturbations. The GUI currently
uses voxel-based translations, and the browser and Python backends still need a
fully unified convention for orientation, rotation order, transform direction,
and physical coordinates.

So the right claim is not:

> An LLM is better than classical medical image registration.

It is:

> Agent-Guided Reg provides an auditable, interactive framework for combining
> classical optimization, visual reasoning, GUI tools, and human anatomical
> guidance.

That is a more modest statement, but I think it points toward the more useful
research problem.

---

## What I Learned

**A strong baseline can improve the research question.** The deterministic
optimizer did not make the agent idea irrelevant. It showed exactly where the
agent was unnecessary and forced me to identify where it might actually add
value.

**Tools are not neutral wrappers.** `evaluate_transform` changed the LLM result
more than another paragraph of prompting probably would have. The available
actions determine what kinds of reasoning are practical.

**Visual reasoning needs quantitative support.** A screenshot can show that
something is wrong without making the corrective direction obvious. Candidate
metrics turned vague visual uncertainty into an actionable choice.

**Human-in-the-loop should be more than a chat box.** Landmarks, masks, and ROIs
convert domain knowledge into geometry and measurable objectives.

**The agent should orchestrate strengths, not imitate every component.** A
classical optimizer is excellent at local metric search. A vision model may be
better at semantic inspection and uncertainty. A human may know which anatomy
matters. The agent's job is to make those capabilities cooperate.

---

## Where I Want to Go Next

The next milestone is not a more theatrical demo. It is a cleaner benchmark.

I want to:

1. unify transform conventions across the browser, Python backend, logs, and
   exported transforms;
2. save complete replayable trajectories with actions, metrics, screenshots,
   and uncertainty events;
3. evaluate multiple OASIS subjects at small, medium, and large perturbations;
4. compare deterministic search, GUI metric planning, visual LLM tool use, and
   human-assisted runs;
5. move to multimodal MRI and partial-overlap cases;
6. implement coarse-to-fine and overlap-aware search;
7. and test whether agent routing helps when metrics and visual evidence
   disagree.

The current prototype started with a simple ambition: let an agent move six
sliders in a registration GUI. The more interesting project that emerged is
about deciding **which evidence to trust, which tool to use, and when another
person should enter the loop**.

That feels much closer to how useful medical AI will actually be built.
