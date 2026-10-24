---
layout: "post"
title: "Computemaxxxing"
date: "2026-08-16 11:38:36 +0000"
description: "How OLMo 3 turns a 9.31T-token crawl into a 5.93T training mix, and what small labs can learn from it."
categories: ["writing"]
tags: ["machine-learning", "pretraining", "data-curation"]
original_url: "https://x.com/pranay5255/status/2088953568527347768"
reading_time: 5
math: true
image: "/assets/posts/computemaxxxing/cover.jpg"
---

Bookmark this if you are trying to maximise $ spent per unit of intelligence while pre-training

<strong>TLDR :-</strong>

- choose a size of the ablation model that meaningful produces signal.  If your long run model size is around 30B then abalations on 500M-3B make sense.
- run cheap deterministic filters before expensive learned ones to filter data to train the ablation models above
- compare every data intervention with an equal-compute baseline and
- inspect both full-data-mixture and held-out regressions before scaling.

![Pipeline to select Pre-training data mix]({{ "/assets/posts/computemaxxxing/2087933567636762624.jpg" | relative_url }})

Early in my career, I built a lead-scoring model and learned a hard lesson of data quality and calibration of classification models. The value of doing the painful, grunt-like EDA step so i could surface the signal or at least throw away the noise. Improving data quality by manually writing pandas, was cumbersome but rewarding. That frame's this article's question:

<strong>when AI researchers are compute bound, how much capability can better data curation buy per token?</strong>

---

OLMo 3 tech report is an especially useful case study because @allen_ai is one of the very few AI labs to open nearly the entire research stack: training infrastructure, code, data, checkpoints, logs, and model artifacts. That transparency let me inspect not just the outcome, which is a fairly compettitive model but also the decisions connecting a <strong>9.31T token pool to a 5.93T token training mix</strong>. This article extracts that practical decision process.

![Illustration from the original article]({{ "/assets/posts/computemaxxxing/2077424847534395392.png" | relative_url }})

## What is a good proxy metric for accuracy at small model scales?

At small model scales,  accuracy scores jump around so much that a real improvement is hard to decipher from noise. Olmo addressed this by grouping evaluation into capability clusters and mapping them to eval metrics. An important metric which helps us measure a proxy for accuracy is BPB. 
Bits-per-byte (BPB) measures how confidently the model predicts a known correct answer, adjusted for answer length. Small models may remain near chance or zero accuracy and two models can receive the same accuracy even when one assigns subtantially more probability to the correct answer.

Mathematically,

<div class="math" markdown="0">
\[
\operatorname{BPB}(y \mid x)
=
\frac{-\log_{2} p_{\theta}(y \mid x)}
{\left|\operatorname{UTF8}(y)\right|},
\]
</div>

where UTF8{y} denotes the number UTF-8 bytes in the gold answer.

Lets consider the question,

“What is the capital of Italy?”

The gold answer is  “Rome” which contains  four UTF-8 bytes.

Model A assigns <strong>p(“Rome”)=1/16</strong>:

<div class="math" markdown="0">
\[
\begin{aligned}
\operatorname{BPB}_A
&amp;=
\frac{-\log_2(1/16)}{4} \\
&amp;=
\frac{4}{4} \\
&amp;=
1.0.
\end{aligned}
\]
</div>

Model B assigns <strong>p(“Rome”)=1/4</strong>:

<div class="math" markdown="0">
\[
\begin{aligned}
\operatorname{BPB}_B
&amp;=
\frac{-\log_2(1/4)}{4} \\
&amp;=
\frac{2}{4} \\
&amp;=
0.5.
\end{aligned}
\]
</div>

Both models could still receive zero exact-match accuracy if another answer is their argmax, but BPB reveals that <strong>Model B is moving toward the correct answer.</strong>

<strong>Lower BPB is better.</strong>

- STEM/non STEM multiple-choice QA → QA BPB
- Math BPB
- Code BPB
- Code fill-in-the-middle

Once the evaluation signal is trustworthy, the next rule is to spend preprocessing compute from cheapest to most expensive. Commoncrawl data is very very dirty so OLMo 3 first applies lightweight heuristic filters, then removes exact duplicates with hashes before paying for fuzzy near duplicate detection and repeated-substring removal.

---

## Cheap filtering over expensive text classification

So what does cheap filtering look like, 
- <strong>Heuristic filters</strong> are simple rules like too much NSFW text, blocked URLs, too little alphabetic text or too many symbols or numbers.

- Compute a hash of each document’s text and keep one document per matching hash

- Fuzzy near-duplicate detection using MinHash to find likely matches then verify candidates with <strong>jaccard similarity</strong> and using <strong>suffix array based matching</strong> to find long passages repeated across otherwise different documents.

Exact matches shrink the corpus before costly similarity comparisons, while the slower language classifier runs only after cheaper commutative filters have discarded obvious noise. The goal is to remove accidental repetition first, so any <strong>later upsampling is a deliberate data-mixture decision.</strong>

![Illustration from the original article]({{ "/assets/posts/computemaxxxing/2077443119759736832.jpg" | relative_url }})

---

## Turn the corpus into 480 controllable experimental cells

OLMo 3 treats <strong>topic and quality as separate sampling decisions</strong>:

- Classify the topic using a cheap fastText classifier, distilled from WebOrganizer labels,  assigns each document to one of 24 topics. Score quality separately. Another fastText classifier gives every document a learned quality score.

Documents are ranked against others in the same topic and  divided into 20 five-percentile buckets.

<strong>This produces 24 × 20 = 480 controllable  subsets.</strong>

The separation provides two distinct controls:

- <strong>Topic weights determine which capabilities receive training tokens.</strong>
- <strong>Quality-aware sampling determines which documents within each topic receive repetition in the pre-training datamix.</strong>

So if you've been reading until now, this is where we make use of all the artifacts we built out,  Separating topic and quality creates a controllable pool, but it does not decide the final token allocation.

OLMo <strong>trained small 30M-parameter proxy models for 3B tokens</strong> on different mixtures, fitted and evaluated for each task, and searched for weights that respected token availability and repetition limits.

## What should a small Lab do?

A small lab should borrow the constrained comparison, test three to five mixtures from one checkpoint with identical token counts and hyperparameters, tracking one continuous development metric, a small capability eval, and one held-out metric.

<strong>Advance ONLY clear winners</strong>.

The hidden assumption is that small-run rankings transfer, so rerun the two best candidates at a second scale before committing the final compute budget.

<strong>This looks like a 5 step procedure --></strong>

<strong>Step 1</strong> -> Specify the starting checkpoint or initialization policy, model architecture, tokenizer, sequence length, optimizer, learning-rate schedule, token count, and evaluation suite.

<strong>Step 2 -></strong>Test three to five candidate mixtures or filtering policies. <strong>Do not change the model or training recipe at the same time</strong>.

<strong>Step 3</strong> -> Every candidate must see the same number of training tokens and updates. Report repeated-token counts separately so that aggressive upsampling is visible and measurable.

<strong>Step 4</strong> -> Track three levels of evidence = one continuous low-scale metric <strong>such as BPB</strong>; one capability-level aggregate such as Log likelihood; per-task or held-out regressions raw task accuracy.

<strong>Step 5</strong> ->Rerun only candidates whose gains exceed run to run loss gain. Validate the top two using another seed or a second model scale{slightly bigger model to make sure the run will be stable} before spending the final compute budget.

---

<strong>OLMo’s 4–7× repetition constraints, classifier choices, and final mixture ratios are  empirical settings, NOT universal laws.</strong>

---

What transfers is the experimental discipline  behind them. Turn each data decision into a controlled test, When compute is expensive, every token should have to <strong>EARN</strong> its training step.
