# DeepSeek-V4 Deep Dive · English Script

> Working English companion script for the original Chinese deck.
> This file is being built in parallel with the slide translation so the Chinese source stays unchanged.

---

## 00 · Cover

Today we are reading the DeepSeek-V4 technical report.

This is not a “world-best model” release. It is a release about making a 1M-context open Agent model affordable enough for normal developers to actually use.

---

## 01 · One-Pager

If you remember only four numbers, remember these:

- Codeforces: **3206**
- Putnam 2025: **120 / 120**
- Native context: **1M tokens**
- V4-Flash cached-input price: **0.2 RMB per million tokens**

My first reaction was simple: DeepSeek’s ambition here is not “be strongest at any cost,” but “make stronger systems cheaper for more people.”

---

## 02 · Positioning

The paper benchmarks against Opus 4.6 and GPT-5.4.

But by release time, the frontier had already moved again: Claude Opus 4.7 arrived on **April 16, 2026**, and GPT-5.5 arrived on **April 23, 2026**.

So is V4 the best model in the world? No.

That is not the point of the release. The point is that V4 turns **1M context + Agent capability** into something closer to a default configuration for ordinary users. One group competes at the ceiling; DeepSeek is trying to raise the floor.

---

## 03 · Four Theses

Four claims organize the whole analysis:

1. It is not world-best; it is mass-market 1M Agent access.
2. It inherits competition-programming DNA, so it is exceptional at tasks with clear right answers, especially math and coding.
3. It is still weaker on taste-heavy work, especially creative writing.
4. It is unusually honest about its own limits, pricing constraints, and next steps.

Every later section loops back to these four ideas.

---

## 04 · Roadmap

The 73-slide deck breaks into 11 acts.

The hardest architecture material is in the middle. The strongest positioning and “what it means” sections are later. If you only have limited time, Acts 0, 7, 9, and 10 carry the highest-level conclusions.

---

## 05 · Two Models

V4 ships in two sizes built on the same architecture.

`V4-Pro` has **1.6T total parameters**, **49B active parameters**, and **61 layers**. It is positioned against Opus 4.6 and GPT-5.4.

`V4-Flash` has **284B total parameters** and **13B active parameters**.

The key is the price. Flash offers cached input at **0.2 RMB per million tokens**. That pushes large-scale Agent usage out of the “enterprise toy” bracket and much closer to something an individual developer can actually run. Pro raises the ceiling; Flash lowers the floor.

---

## 06 · MoE Basics

Before discussing the architecture, you need the right mental model for MoE.

Mixture-of-Experts is not “everyone votes.” It is an expert meeting.

Each V4-Pro layer contains **384 routed experts**. For a given token, only **6 routed experts** are activated, plus **1 shared expert**. The team size is 1.6T. The people actually speaking on each step are closer to 49B.

---

## 07 · Sparse Economics

V4-Pro activates **49B** out of **1.6T** parameters on each step, which means an activation ratio of about **3.1%**.

That is even more aggressive than V3, which sat around **5.6%**.

So the rough economics are: you pay the storage cost of a 1.6T model, but you get inference closer to a 49B model. That is a huge part of why V4 can be so much cheaper than a dense model with similar knowledge coverage.

---

## 08 · Training Data

V4-Pro uses **33 trillion training tokens**.

What matters is not just the size, but how the data mix is built:

1. AI-generated content is filtered out to reduce collapse risk.
2. Agentic data is injected during mid-training, so tool-use behavior is introduced before post-training.
3. Long-tail multilingual data is deliberately expanded.
4. Scientific papers and technical reports are up-sampled to support scientific reasoning.

One implementation detail matters a lot: DeepSeek enables **sample-level attention masking** so separately packed samples do not contaminate one another inside the same sequence.

---

## 09 · Training Stability

This is one of the most interesting parts of the paper because the authors are unusually direct.

How do you stop a **1.6T** model from blowing up?

They mention two practical tricks:

1. **Anticipatory Routing**: when loss spikes, routing is recomputed using older parameters from several steps back.
2. **SwiGLU Clamping**: the linear term is simply clamped into a safe numeric range.

The striking part is the honesty. The paper effectively says: these tricks work in practice, but we still do not fully understand why.

That is one of the clearest signs that DeepSeek is writing like an engineering team, not a marketing team.

---

## 10 · Specs Table

This is a reference slide more than an argument slide.

The big takeaway is that the two models are structurally similar, and one detail is worth remembering for later: the attention stack transitions into sparse behavior during warmup around **64K**.

---

## 11 · mHC Problem

This section asks a simple question: if residual connections are so good, why do they start failing here?

The answer is depth. Residual paths are great at preserving signal, but when you keep stacking them past 60 layers, unconstrained accumulation becomes its own source of instability. The same path that helps optimization at moderate depth starts amplifying noise at extreme depth.

So V4’s move is not to discard residuals. It is to keep them, but put them under tighter mathematical control.

---

## 12 · mHC Intuition

The simplest mental model is a water network.

Each residual path can move signal forward, but the total inflow and total outflow are both conserved. That is what the doubly stochastic constraint means in practice. No layer can secretly create extra gain, and no path can swallow all the energy.

This is a very engineering-style insight: if the unconstrained system is unstable, do not hope optimization fixes it later. Put the guardrail directly into the parameterization.

---

## 13 · mHC Math

The math slide makes the intuition precise.

Rows sum to 1. Columns sum to 1. All entries stay nonnegative.

That gives two powerful consequences:

1. The spectral norm is bounded by 1, so the transform does not amplify signal.
2. The product of two doubly stochastic matrices is still doubly stochastic, so the constraint survives repeated composition.

That second point is why this is so useful for depth. You are not patching one layer at a time. You are choosing a structure that stays stable when multiplied 61 times.

---

## 14 · mHC Cost

This is not free.

DeepSeek uses Sinkhorn-Knopp projection to push each learned matrix back onto the doubly stochastic manifold, with about **20 iterations per layer per forward pass**. Even with fused kernels and recomputation tricks, the paper still reports a throughput penalty of roughly **5%**.

But the comparison is not “5% overhead versus zero overhead.” The real comparison is “5% overhead versus a 1.6T, 61-layer model that may not train stably at all.”

---

## 15 · mHC Effect

The payoff is visible in the loss curve.

Without mHC, the reconstructed baseline shows repeated spikes and instability. With mHC, training stays smooth across the full **33T-token** run.

That is the important lens for the whole section: mHC is not decorative theory. It is an infrastructure choice that buys enough stability to actually finish pretraining a model at this scale.

---

## 16 · Transition

At this point the argument shifts.

mHC solved depth. The next bottleneck is length. Once you move from ordinary context windows into the million-token regime, attention becomes the new systems problem.

---

## 17 · Attention Bottleneck

Long context creates two separate explosions:

1. Compute grows as **O(n²)** because every query still compares against every key.
2. KV cache grows as **O(n)** because every layer still stores keys and values for every token.

That means “1M context” is not just a model-quality claim. It is a brutal architecture and systems claim.

---

## 18 · Hybrid Idea

DeepSeek’s answer is to alternate between two attention styles.

CSA is the fine-grained path. It compresses first, then sparsifies, then reads only the most relevant blocks.

HCA is the coarse-grained path. It compresses much harder, but then reads all compressed blocks densely to preserve global coverage.

The human analogy is simple: first scan the table of contents, then closely read the few paragraphs that matter.

---

## 19 · CSA Explained

CSA is a two-stage reduction:

1. Compress the sequence in blocks of **m = 64**.
2. Use the Lightning Indexer to score compressed blocks and select only top-k for the expensive path.

That means it saves cost twice: once from compression, and again from sparse selection.

The clever part is that the scoring path is cheap enough to run broadly, but accurate enough to keep the useful blocks.

---

## 20 · HCA Explained

HCA takes the opposite trade.

Instead of using moderate compression plus sparsity, it uses very aggressive compression with **m′ ≫ m**, then keeps the attention dense over the compressed representation.

So HCA throws away fine detail, but keeps global document structure alive. That is why it complements CSA instead of duplicating it.

---

## 21 · Supporting Tricks

This slide matters because the main architecture alone is not enough.

Four small choices make the million-context stack practical:

1. **Shared KV / MQA** so all heads reuse the same compressed KV.
2. A **sliding window** so recent tokens stay uncompressed.
3. An **attention sink** so the model can effectively abstain instead of attending to garbage.
4. **Partial RoPE** so position handling remains stable at extreme length.

This is a recurring DeepSeek pattern: the headline idea gets attention, but the real system works because several quieter implementation details are aligned with it.

---

## 22 · Precision Stack

This slide shows another important design principle: precision is allocated by task, not uniformly.

The indexer only needs to rank relevance, so it can live at **FP4**. The KV cache sits at **FP8** because it is mainly a storage problem. RoPE-sensitive dimensions and final core compute stay at **BF16** because those are the places where numerical damage hurts most.

So V4 is not just “using lower precision.” It is using the right precision in the right place.

---

## 23 · Efficiency Comparison

This is the quantitative payoff slide.

At 1M context, compared with V3.2, V4-Pro is shown as:

1. **9.8× lower** in single-token inference FLOPs
2. **13.7× smaller** in accumulated KV cache

And the abstract states the same point another way: roughly **27% of the FLOPs** and **10% of the cache** relative to V3.2.

The main point is not the exact reconstruction. It is that the hybrid-attention stack changes the slope of long-context cost curves in a serious way.

---

## 24 · KV Cache Diet

This is the easiest headline in the whole section to remember:

Against a BF16 GQA-8 baseline, V4 gets the KV cache for 1M context down to about **2%**.

That result is not from one trick. It comes from the whole stack working together:

1. CSA/HCA compression
2. Shared-KV MQA
3. FP8 main storage
4. BF16 only where positional fidelity still matters

That is the pattern to keep in mind: DeepSeek is repeatedly taking a systems bottleneck and attacking it as a layered engineering problem instead of with one silver bullet.

---

## 25 · AdamW Bias Problem

The optimizer section starts by asking what is wrong with AdamW in the first place.

The complaint is not that it is obsolete or universally bad. The complaint is that it updates each parameter dimension independently, which can distort the relative geometry of a matrix update. Strong directions keep getting pushed; weak directions stay under-trained.

At small scale, that may be tolerable. At V4 scale, it becomes expensive.

---

## 26 · Muon Orthogonalization

Muon’s core move is simple to state:

Take the momentum matrix, strip away unequal singular values, and keep only the directional frame.

In SVD language, if the momentum is `M = UΣVᵀ`, Muon replaces it with `UVᵀ`. That means every singular direction gets the same step size. The update becomes geometrically balanced instead of stretched toward one dominant axis.

---

## 27 · Newton-Schulz

The obvious objection is cost: real SVD on every update would be too expensive.

That is why Muon uses Newton-Schulz iteration to approximate the orthogonal projection using only matrix multiplications. V4 further splits the 10-step process into two phases:

1. An aggressive phase to yank singular values toward 1 quickly.
2. A gentler refinement phase to settle them precisely.

That is a very DeepSeek-style compromise: mathematically principled, but engineered for throughput.

---

## 28 · RMS Rescale

Orthogonalization solves the shape problem, but introduces a scaling problem.

Muon updates do not naturally live on the same RMS scale as AdamW updates, which would normally force a full new hyperparameter sweep. V4 fixes that by explicitly rescaling the update magnitude so the old AdamW-tuned learning-rate regime can be reused.

This is a small-looking trick with enormous practical value. It turns optimizer replacement from a research rerun into a production-compatible swap.

---

## 29 · Muon vs AdamW

The result is the whole point of the section:

Muon is presented as both faster to converge and smoother to train.

The paper does not show the full raw training curve, so the slide is necessarily schematic. But the qualitative claim is clear: better-conditioned updates lead to lower loss earlier, with less instability along the way.

The final engineering nuance matters too: DeepSeek does not replace AdamW everywhere. Small modules like embeddings, the prediction head, bias terms, and RMSNorm stay on AdamW. The big weight matrices move to Muon.

---

## 30 · Infrastructure Overview

This section changes tone again.

The earlier chapters explain the architecture. This chapter explains what it takes to make that architecture actually run.

The clean summary is that DeepSeek is not relying on one magic systems trick. It is attacking six different infrastructure layers at once.

---

## 31 · MoE Comm Overlap

The first infrastructure win is classic but important: hide communication under compute.

DeepSeek breaks the MoE path into four stages and schedules them in waves, so dispatch/combine traffic is overlapped with the two linear layers instead of sitting in series.

This is not just a training detail. It is the sort of throughput work that changes whether a giant sparse model is operationally tolerable.

---

## 32 · TileLang

TileLang is really about refusing a false tradeoff.

Handwritten CUDA is powerful but slow to iterate on. Higher-level systems like Triton are productive but can become constraining. TileLang is DeepSeek’s attempt to hold onto both development velocity and low-level control.

That is also why the slide emphasizes host codegen, SMT-backed integer analysis, and explicit numerical control rather than only kernel speed.

---

## 33 · Determinism

This is one of the highest-signal infrastructure slides in the whole deck.

DeepSeek is aiming for bit-exact alignment between training and inference. That matters enormously for RL and post-training, because it means production anomalies can actually be replayed and debugged instead of vaguely approximated.

The three problem areas are:

1. Attention accumulation order
2. Batch-invariant matmul
3. Backward-pass nondeterminism from atomicAdd

And the theme is consistent: if non-determinism comes from execution order, then execution order has to be made explicit and controlled.

---

## 34 · FP4 QAT

This slide shows where DeepSeek spends its quantization budget most aggressively.

The two heavy targets are:

1. MoE expert weights
2. The CSA indexer’s QK path

The practical win is not only memory reduction. It is that FP4 can still plug back into the existing FP8 pipeline cleanly, which keeps the engineering cost of adoption manageable.

---

## 35 · Disk KV Cache

This is one of the most striking engineering moves in the deck.

Once CSA/HCA has already compressed KV enough, DeepSeek treats shared-prefix KV almost like a storage hierarchy problem: hot in memory, colder on disk, reusable across long agent-style requests.

That only becomes plausible after compression. Without the earlier compression work, disk bandwidth would have been a nonstarter.

---

## 36 · Infrastructure Philosophy

The chapter closes by making its worldview explicit.

Three values run through the whole systems section:

1. Build key infrastructure yourself instead of depending blindly on vendors.
2. Treat reproducibility as a product feature, not a luxury.
3. Waste as little time and bandwidth as possible at every layer.

That is why the infrastructure chapter matters. It tells you DeepSeek is not only chasing model quality. It is trying to own the entire operational stack needed to make that quality affordable.

---

## 37 · The Old Post-Training Pain

This chapter starts by arguing that the usual post-training recipe itself is the problem.

If you use one student model to run mixed SFT and mixed RLHF across math, code, agent, dialogue, and everything else, you inherit three predictable problems:

1. RL instability
2. Multi-task interference
3. Reward hacking

This is the setup V4 is trying to escape.

---

## 38 · Two-Stage Paradigm

The replacement is structurally simple:

1. Train domain specialists with RL.
2. Distill them into one final student with OPD.

That means RL is no longer the final stage for the universal model. RL becomes a specialist-training tool, while the general student only learns by distillation.

---

## 39 · Specialist Stage

Each expert runs its own local pipeline:

1. SFT to establish the base skill
2. GRPO for RL inside that domain
3. Save the resulting expert checkpoint

This is where DeepSeek also introduces two notable ideas: the generative reward model for fuzzy tasks, and adjustable reasoning depth modes like `Non-think`, `Think High`, and `Think Max`.

The whole point is reward clarity. Each expert sees one domain and one cleaner objective.

---

## 40 · OPD Stage

On-Policy Distillation is the student-training half of the new paradigm.

The important ingredients are:

1. **Reverse KL**, which pushes the student toward a high-probability mode instead of forcing it to average across all teacher modes.
2. **On-policy sampling**, so the student learns on the trajectories it actually produces.
3. **Full-vocab logits**, so the KL is computed exactly instead of approximated cheaply.

This is more expensive than lightweight distillation, but much more faithful.

---

## 41 · Reverse KL

This slide explains the cleverest part of the setup.

With multiple teachers, reverse KL gives the student an implicit routing behavior. The student’s own rollout distribution decides which teacher mode it is closest to, and alignment happens there.

So the system does not need a separate external router that says “listen to the math teacher now.” The on-policy distribution performs that role automatically.

---

## 42 · GRM and DSec

Two pieces of infrastructure sit underneath agent RL here:

1. A **Generative Reward Model**, where the judge is itself a model rather than a static scalar scorer.
2. **DSec**, a serious elastic sandbox stack with multiple execution tiers ranging from warm function calls to full virtual machines.

This is a strong signal that DeepSeek is not treating agent RL as a benchmark toy. It is treating it as a production systems problem.

---

## 43 · Paradigm Shift

This is the conclusion of the whole chapter:

DeepSeek is redefining what RL is for.

Instead of forcing one universal student to survive the entire RL mess directly, RL is used to train specialists on cleaner tasks. Then the universal model inherits those abilities through distillation.

That is why the chapter calls this a paradigm shift rather than a small optimization. It changes the role of RL in the training stack.

---

## 44 · Team DNA

The argument in this slide is blunt:

DeepSeek hires people with Olympiad, elite-school, and hard-science credentials, so the model inherits that exact bias profile.

That does not mean "smart people build smart models." It means the organization is structurally optimized for tasks that look like competition problems:

1. Well specified
2. Answerable
3. Rewarded for precision

That becomes an explanatory lens for everything that follows in the benchmark section.

---

## 45 · Codeforces 3206

This is one of the splashiest slides in the deck.

A rating of `3206` is not just a benchmark number. It is translated into a human ranking claim: about 23rd among global human contestants.

That matters because it makes the result legible outside ML. Instead of "a model got a score," the claim becomes "the model performs like an elite contest programmer."

The real strategic point is the paper’s framing: this is presented as the first time an open model clearly beats closed flagships on a major competition-coding metric.

---

## 46 · Putnam 120/120

This is the mathematical equivalent of the Codeforces slide, but arguably even more shocking.

The setup is not plain text math QA. It is hybrid formal-informal proving: reason in language, then compile the proof into Lean.

A perfect `120/120` on that pipeline means two things at once:

1. The reasoning is strong.
2. The translation into formal structure is reliable enough to survive theorem-checker constraints.

That is why the slide calls it the first AI system ever to get a perfect Putnam score in this setting.

---

## 47 · Math Benchmarks

This table widens the aperture.

The important takeaway is not that DeepSeek wins every math benchmark. It does not. The real point is that it is competitive everywhere and wins exactly where its style is strongest:

1. Short-answer, highly standardized math
2. Formal proving with crisp correctness signals

On fuzzier or broader research-grade math, Gemini or GPT can still lead. But DeepSeek remains firmly in the top tier across the board.

---

## 48 · Coding Benchmarks

This is one of the most honest slides in the whole deck.

DeepSeek dominates competitive coding, but it does not dominate engineering coding. Those are related skills, not identical ones.

The split is very clear:

1. Contest coding: DeepSeek leads.
2. Engineering bug-fixing and repo work: DeepSeek is strong, but not decisively ahead.

That distinction matters because many people casually collapse "coding ability" into one bucket. The deck argues that would be a category error.

---

## 49 · Versus the Latest SOTA

This slide updates the benchmark story beyond the paper itself.

The paper benchmarked against older closed models, but the deck adds newer releases like Opus 4.7 and GPT-5.5. Once that happens, the pattern sharpens:

1. DeepSeek still leads on standardized contest-like tasks.
2. Newer closed models pull ahead on messy engineering workloads.

So the strongest possible claim is not "DeepSeek is the best model overall." It is narrower and more defensible: "DeepSeek is best in the domains most aligned with exact, competition-style evaluation."

---

## 50 · Verdict

This slide compresses the whole chapter into one sentence:

On standardized tasks with clear right answers, DeepSeek is the open-source ceiling.

That is the right level of precision. It avoids overclaiming while still stating a real advantage.

The four proof points on the slide are chosen carefully:

1. Codeforces
2. LiveCodeBench
3. Apex Shortlist
4. Putnam-2025

Together they define the territory where DeepSeek looks strongest: exacting tasks, explicit scoring, and low ambiguity.

---

## 51 · Transition

This slide exists to reset the audience’s confidence.

After a full chapter showing where DeepSeek looks dominant, the deck deliberately slams on the brakes with one word: “But.”

The framing is important. The next section is not saying the previous chapter was fake. It is saying the strengths were conditional. Once the task leaves the safe zone of explicit answers, the model’s deeper biases become visible.

---

## 52 · HLE

Humanity’s Last Exam is the first benchmark in the deck that clearly resists the “competition problem” template.

It is still objective enough to score, but broad and interdisciplinary enough that raw specialist-style optimization helps less.

DeepSeek is still the best open model here, which matters. But it is no longer setting the pace overall. This is the first slide where the deck says, in effect: the advantage is narrowing.

---

## 53 · HLE with Tools

This is one of the sharpest diagnostic slides in the entire presentation.

Without tools, DeepSeek is mid-pack but respectable. Add tools, and instead of benefiting like everyone else, it falls to last.

That points to a very specific weakness:

1. Not pure reasoning
2. Not pure knowledge
3. Coordination across tool calls

So the problem is agent orchestration rather than isolated intelligence.

---

## 54 · Terminal Bench

Terminal Bench pushes the same conclusion into a more realistic setting.

This is long-horizon, stateful, tool-using work in a terminal, much closer to actual autonomous coding or operations behavior than a clean multiple-choice benchmark.

Against newer closed models, especially GPT-5.5, the gap widens materially. That matters because it shows the weakness is not only academic. It appears in practical agent workflows too.

---

## 55 · Creative Writing

This slide broadens the weakness story beyond tool use.

DeepSeek can outperform Gemini in more functional or instruction-shaped writing, which fits the broader thesis. But once the task becomes taste-heavy, constrained, and aesthetically judged, it loses to Claude Opus 4.5.

That is a different kind of failure mode:

1. Not coordination failure
2. Not knowledge failure
3. Judgment and aesthetic calibration failure

---

## 56 · Ground Truth vs Taste

This may be the single best conceptual slide in the deck.

It reduces the whole benchmark story to one axis: how much ground truth the task provides.

On the left side, where answers are explicit, RL and competition-style training shine. On the right side, where evaluation becomes subjective or vaguely continuous, that same training style loses traction.

This is the deck’s deepest claim about DeepSeek’s character as a model.

---

## 57 · What the Paper Admits

This section is not about benchmark wins or losses. It is about credibility.

The three admissions matter because they are concrete:

1. The architecture is too stacked, and ablations are incomplete.
2. Some effective techniques are empirically useful but theoretically underexplained.
3. The sparsity frontier is still not fully pushed.

That kind of specificity makes the paper feel more trustworthy, not less.

---

## 58 · Verdict

This chapter’s conclusion is the mirror image of chapter seven’s conclusion.

Where chapter seven said DeepSeek is strongest on standardized, clearly graded tasks, this chapter says it is visibly weaker on:

1. Taste-heavy work
2. Fuzzy judgment
3. Long tool-using chains

That symmetry is what makes the overall argument coherent. The model’s strengths and weaknesses are both downstream of the same organizational and training DNA.

---

## 59 · The Real Value of 1M

This chapter shifts from model quality to product meaning.

The key claim is that 1M context is not valuable because people constantly need to fill a million tokens. It is valuable because users can stop spending mental energy on compression and triage.

That is a product-design argument, not a benchmark argument.

---

## 60 · Agentic Prefill

This slide makes the same point numerically.

The paper’s own agentic search workload only uses about 13.6K prefill tokens, nowhere near the 1M ceiling. But that does not weaken the 1M story. It strengthens it.

The existence of slack is the feature. It means users can stop planning around scarcity.

---

## 61 · Official Pricing

This slide separates Pro from Flash very clearly.

Pro is the prestige model. Flash is the accessibility vehicle.

That distinction matters because many people hear “DeepSeek is cheap” and assume the flagship itself is the whole pricing story. The deck argues that would be imprecise. The real democratization engine is Flash.

---

## 62 · Pro vs Latest Flagships

This is the disciplined version of the pricing claim.

V4-Pro is dramatically cheaper than the newest closed flagships, especially on output pricing, but the slide is careful not to oversell. Relative to top closed models, it is cheap. In absolute terms, it is not absurdly free.

That kind of calibration is part of the broader honesty thesis.

---

## 63 · Flash Is the Price Killer

This is where the deck cashes out the accessibility argument.

If you want the concrete model that changes adoption behavior, it is Flash:

1. Huge context
2. Low enough cost to use casually
3. Good enough capability to matter in real workflows

That is why the deck treats Flash, not Pro, as the core enabler of mass-market use.

---

## 64 · Ecosystem Positioning

This is the strategic layer.

DeepSeek is not only trying to win as a destination product. It is trying to become infrastructure inside developer tools.

That is an important distinction:

1. Consumer chat products fight for direct attention.
2. Infrastructure models fight to be embedded.

The second battle can look quieter while being strategically deeper.

---

## 65 · Accessibility Verdict

This is the summary of thesis one:

The real release is not “the best model in the world.” It is “million-context agent capability becomes affordable enough to be normal.”

That is exactly why the slide uses the metaphor of moving from meter-running scarcity to an all-you-can-use monthly pass.

---

## 66 · Three Honest Admissions

This chapter reopens the honesty theme, but now at a higher level than benchmark weaknesses.

The core point is that DeepSeek does not only admit weaknesses in one hidden appendix line. It repeatedly uses the paper to surface unresolved issues in architecture, understanding, and future direction.

That is why the deck frames honesty as a product trait and organizational trait, not just a tone choice.

---

## 67 · V5 Roadmap

This slide does something many labs avoid: it turns limitations directly into roadmap signals.

The most interesting part is not the generic list like “better agents” or “multimodal.” It is the hint that future versions may move toward scalable lookup-style memory systems, which could mean a real architectural shift beyond ordinary attention scaling.

That makes the roadmap feel technically grounded rather than cosmetic.

---

## 68 · Ascend 950 Signal

This is a pricing slide, but it mainly exists to reinforce the honesty thesis.

The company openly says Pro is expensive now because high-end compute is constrained, and then ties future relief to a specific hardware supply event: Ascend 950 supernodes in the second half of the year.

Specificity is the point. Honest constraints plus a concrete dependency is much more credible than vague future optimism.

---

## 69 · Closing the Four-Thesis Loop

This slide is structural.

It closes the deck by showing that the four main theses are not disconnected observations:

1. Accessibility
2. Competition-team DNA
3. Structural weaknesses
4. Honesty and discipline

The last thesis is presented as the one that binds the other three together.

---

## 70 · Closing Line

This slide is rhetorical rather than analytical.

The Xunzi quote gives the deck a moral frame: not seduced by praise, not frightened by criticism, and committed to upright conduct.

That is used to interpret DeepSeek’s public posture as a deliberate ethos rather than mere PR style.

---

## 71 · Overall Positioning

This is the one-sentence conclusion of the whole presentation:

DeepSeek-V4 is the most honest and disciplined open-source flagship.

Notice how carefully that is phrased. It is not “the best model.” It is a claim about role, posture, and contribution to the ecosystem.

---

## 72 · Credits

The final slide mostly records sources and authorship, but it also quietly tells you something about the project:

This deck is not pretending to be a neutral benchmark spreadsheet. It is an interpretation built from the paper, the official launch post, and hands-on experience in the open-agent ecosystem.

That framing matters because it explains both the strengths of the analysis and its perspective.

---

## Note

This English script is intentionally parallel to the Chinese original rather than replacing it.

The next step is to export a dedicated English PDF from the finished English HTML deck.
