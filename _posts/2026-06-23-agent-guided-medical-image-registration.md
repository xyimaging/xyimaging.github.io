---
layout: post
title: "I Built an Agent That Can Operate a 3D Medical Image Registration GUI"
description: A research prototype that combines rigid registration, visual overlays, metric-guided search, LLM tool use, two-stage visual planning, and human guidance—and the experiments that changed what I thought the agent should actually do.
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

{% include video.liquid path="assets/video/demo3.mp4" class="img-fluid rounded z-depth-1" controls=true poster="assets/img/posts/agent-guided-reg/gui-overview.png" title="Agent-Guided Reg two-stage visual demo" alt="Demo of a two-stage visual agent operating a 3D medical image registration GUI" caption="A live demo of the current Agent-Guided Reg prototype. The in-page cursor loads a 3D MRI pair, creates a visible rigid misalignment, selects the two-stage visual planner, sends sagittal/coronal/axial overlays to a vision model, and records the agent's visual and metric reasoning." %}

This is **Agent-Guided Reg**, a browser-based prototype for agent-assisted 3D
medical image registration. It loads volumetric medical images, displays
sagittal, coronal, and axial views, lets a user adjust a six-degree-of-freedom
rigid transform, and gives an agent access to the same working state through
structured tools.

The current demo is no longer just a scripted slider tour. It shows a
**two-stage visual planner** running through the GUI:

1. a visual observer inspects the current red-green overlays;
2. an action planner compares that visual hypothesis with candidate metric
   evidence;
3. the browser executes one structured rigid-transform action;
4. the GUI redraws the result and repeats the loop.

That distinction matters. The goal is not to make a language model imitate a
classical optimizer. The goal is to build an auditable registration assistant
that can combine **what it sees**, **what the metric says**, and **what the user
knows**.

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
recovered all six ground-truth parameters exactly. I later expanded the
same-modality benchmark to **18 synthetic cases** from six OASIS subjects, with
small, medium, and large perturbations. The deterministic NCC coordinate search
recovered **18/18 cases exactly**, with mean NCC improving from **0.6739 to
0.9963** and zero translation/rotation error.

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
the first recorded demo.

I also added multiple strategies:

- direct initialization;
- metric stepwise refinement;
- LLM tool agent;
- visual LLM guided loop;
- two-stage visual planner;
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

The first useful result was not that the model became a better optimizer. It was
that the model could participate in the closed loop:

**visual observation → model decision → structured tool call → GUI execution →
new observation**

On the OASIS synthetic case, the final visual hybrid recovered the exact
synthetic transform over 17 turns, improving NCC from **0.6405 to 0.9949**. But
that result came with an important caveat: when candidate NCC scores were
available, the model usually followed the metric-best candidate. The visual
input made the loop richer and more interpretable, but the metric table still
dominated the easy same-modality case.

That is why I split visual reasoning and action selection into two explicit
stages.

---

## Why I Split the Agent Into Two Stages

In the original visual-agent call, the model returned two things at once:

1. a written visual assessment;
2. a structured action such as `translate x +1` or `rotate z -0.5`.

That made the result hard to interpret. Did the written assessment actually
influence the action? Or did the model simply read the candidate metric table,
pick the best NCC update, and write a plausible explanation afterward?

The **two-stage visual planner** makes this dependency more auditable.

{% include figure.liquid loading="eager" path="assets/img/posts/agent-guided-reg/two-stage-visual-architecture.png" class="img-fluid rounded z-depth-1" caption="The current single-call visual agent co-generates a visual assessment and a tool call. The two-stage planner freezes a screenshot-only visual hypothesis first, then asks a separate planner to compare that hypothesis with candidate metrics before choosing an action." %}

The loop is:

1. Render sagittal, coronal, and axial red-green overlays.
2. Send the overlays to a **Visual Observer**.
3. The observer does **not** receive NCC or the candidate-action table.
4. It returns a frozen visual hypothesis: per-view findings, residual direction,
   anatomical cues, uncertainty, and confidence.
5. Send that frozen hypothesis plus metric candidates to an **Action Planner**.
6. The planner records the visual recommendation, metric recommendation,
   agreement/conflict, confidence, and selected GUI tool.
7. Local code executes the tool and applies the NCC safety gate.

This is not just a software refactor. It changes what can be claimed. The model
can now say:

- “The sagittal view suggests posterior/anterior mismatch.”
- “The axial view is ambiguous.”
- “I see no reliable anatomical directional cue.”
- “The visual hypothesis conflicts with the metric candidate table.”
- “I will follow the metric because the image evidence is uncertain.”
- “I should ask the human before continuing.”

In the first calibrated two-stage smoke test, the Visual Observer abstained:
all three views were reported as having no reliable directional cue, uncertainty
was high, and confidence was 0.00. Stage 2 then transparently followed the
metric-best candidate, selected `translate x -8`, and improved NCC from
**0.6572 to 0.7350**. That was a small result, but it was exactly the kind of
trace I wanted: the visual module was allowed to say “I do not know,” and the
planner recorded why it trusted the metric instead.

In the live GUI demo shown above, the same two-stage mode ran for five turns
using `gpt-4o-mini`. Starting from a moderate synthetic misalignment, the loop
sent the three-view overlays to the visual server each turn and improved NCC
from **0.5634 to 0.6536**. This is a demo-scale result, not a benchmark result,
but it demonstrates the full interactive pathway: visual observation,
metric-aware planning, GUI action execution, and timeline export.

---

## What the Ablation Actually Showed

I then ran four matched conditions on the same OASIS synthetic rigid case:

| Condition | Visual input | Metric input | Outcome |
|---|---:|---:|---|
| Full | yes | yes | exact recovery, final NCC 0.9949 |
| Metrics only | no | yes | exact recovery, final NCC 0.9949 |
| Images only | yes | no | stalled, final NCC 0.7296, 15 rejected actions |
| Full + human permission | yes | yes | paused at turn 12 with `askHuman`, final NCC 0.9663 |

{% include figure.liquid loading="eager" path="assets/img/posts/agent-guided-reg/visual-ablation-four-condition-summary.png" class="img-fluid rounded z-depth-1" caption="Four-condition visual reasoning ablation. On this easy MRI case, candidate NCC dominated ordinary action selection: full and metrics-only recovered the transform, while images-only failed. The human-permission run is interesting because the model voluntarily paused when its visual interpretation became uncertain." %}

This was a useful negative result. It prevents overclaiming.

On this easy MRI case, visual input did **not** outperform candidate NCC. The
full and metrics-only runs chose the same improving trajectory and both exactly
recovered the synthetic transform. Images alone were not enough; the model
matched the best metric candidate only once and then repeatedly proposed actions
that the local safety gate rejected.

The interesting behavior appeared when human guidance was permitted. At turn
12, the model stopped with `askHuman`. Numerically, candidate actions were still
available. Visually, the model reported residual mismatch but could not reliably
decide whether the remaining correction should be translation or rotation. It
changed its evidence source toward the image and reduced confidence.

{% include figure.liquid loading="eager" path="assets/img/posts/agent-guided-reg/full-human-horizontal-timeline.png" class="img-fluid rounded z-depth-1" caption="A horizontal timeline from the human-permission visual run. The figure records NCC, actions, and sagittal/coronal/axial overlays across turns. The run pauses when the model requests human guidance instead of forcing another uncertain update." %}

This is the current evidence for the value of the visual agent: not that it is a
better local optimizer, but that it can expose uncertainty, compare visual and
metric evidence, and create a structured place for human input.

---

## Early Multi-Case Pilot

To avoid staring too long at a single friendly case, I also started a small
multi-case pilot. The current pilot includes three OASIS subjects, three
perturbation levels per subject, and three LLM-based modes with a short
three-turn budget:

- metrics-only LLM;
- single-call visual LLM;
- two-stage visual planner.

{% include figure.liquid loading="eager" path="assets/img/posts/agent-guided-reg/multisubject-pilot-summary.png" class="img-fluid rounded z-depth-1" caption="A small multi-subject pilot across OASIS subjects and perturbation levels. The short-turn LLM modes currently produce similar numerical outcomes, while the two-stage mode uniquely records visual/metric conflicts and exposes several tool-format failures that need to be fixed before larger evaluation." %}

The numerical results were intentionally modest. With only three turns, all
three LLM modes reached the same mean final NCC of about **0.8131**, with mean
translation error around **3.89 voxels** and mean rotation error around
**4.55 degrees**. In contrast, the deterministic NCC baseline solved the larger
18-case same-modality benchmark exactly.

The two-stage planner did not win numerically in this pilot. Its value was
instrumentation:

- it recorded **5 visual/metric conflicts** across 9 cases;
- it matched the metric-best candidate in 26/27 turns;
- it exposed **4 malformed tool-call fallbacks**, which are engineering issues
  to fix before scaling;
- and it produced a richer trace for analyzing why an action was chosen.

That is still progress. A research prototype should not only produce good
numbers; it should also reveal why the current method fails, where it is leaning
on the baseline, and which claims are not yet supported.

---

## Human Guidance Needs to Become Geometry

Natural-language hints are useful, especially when they name relevant anatomy:
“focus on the fetal head,” “ignore background mismatch,” or “there may be a
large translation.”

But a sentence does not automatically change the metric. If the optimizer still
computes global NCC, it may continue rewarding the wrong region.

That led me to add two more explicit forms of guidance.

**Landmarks.** A user can click corresponding points in the fixed and moving
images. One point pair provides a translation initialization; multiple pairs can
later support a full rigid estimate. Landmark error also stays in the planner's
objective as a soft constraint during refinement.

**Segmentation masks.** Optional fixed and moving masks provide anatomical
overlap signals such as Dice and centroid distance. The GUI displays their
contours, and the metrics are included in both the local candidate objective and
the LLM observation.

The current combined objective is intentionally lightweight: reward NCC and mask
overlap, penalize landmark and mask-centroid error. It is not a final clinical
registration metric. What matters is the interface it establishes: human
knowledge can enter the loop as a measurable constraint rather than remaining an
informal comment.

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
- the deterministic baseline exactly solves easy same-modality synthetic cases;
- LLM tool-calling experiments expose the importance of tool design;
- the browser GUI supports real 3D interaction;
- the metric planner operates the GUI and produces strong easy-case alignment;
- a visual LLM can observe screenshots plus candidate metrics and return
  executable actions;
- the two-stage visual planner separates screenshot-only observation from
  metric-aware action selection;
- visual/metric agreement, conflict, confidence, and uncertainty are now stored
  in the trace;
- landmarks, masks, and human hints can influence the loop;
- and the full workflow is available as a repeatable visual demo.

But the current evidence is still narrow. Most quantitative tests are
same-modality OASIS MRI cases with synthetic rigid perturbations. The
multi-subject visual pilot is still small and has a very short turn budget. The
GUI currently uses voxel-based translations, and the browser and Python backends
still need a fully unified convention for orientation, rotation order, transform
direction, and physical coordinates.

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

**Two-stage reasoning makes uncertainty inspectable.** Freezing a
screenshot-only hypothesis before showing metrics prevents the visual assessment
from becoming an after-the-fact explanation. It lets the system record whether
vision agrees with the metric, conflicts with it, or abstains.

**Human-in-the-loop should be more than a chat box.** Landmarks, masks, and ROIs
convert domain knowledge into geometry and measurable objectives.

**The agent should orchestrate strengths, not imitate every component.** A
classical optimizer is excellent at local metric search. A vision model may be
better at semantic inspection and uncertainty. A human may know which anatomy
matters. The agent's job is to make those capabilities cooperate.

---

## Where I Want to Go Next

The next milestone is not a more theatrical demo. It is a cleaner benchmark and
a more honest set of failure cases.

I want to:

1. unify transform conventions across the browser, Python backend, logs, and
   exported transforms;
2. save complete replayable trajectories with actions, metrics, screenshots,
   visual hypotheses, conflicts, and uncertainty events;
3. evaluate multiple OASIS subjects at small, medium, and large perturbations
   with longer turn budgets;
4. fix malformed tool-call fallbacks and harden the two-stage planner;
5. compare deterministic search, GUI metric planning, single-call visual LLM,
   two-stage visual planning, and human-assisted runs;
6. move to multimodal MRI and partial-overlap cases;
7. implement coarse-to-fine and overlap-aware search;
8. add mask- and landmark-aware visual reasoning;
9. and test whether agent routing helps when metrics and visual evidence
   disagree.

The current prototype started with a simple ambition: let an agent move six
sliders in a registration GUI. The more interesting project that emerged is
about deciding **which evidence to trust, which tool to use, and when another
person should enter the loop**.

That feels much closer to how useful medical AI will actually be built.
