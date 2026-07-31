# LLM Inference & NVIDIA Dynamo — The Fundamentals
### A ground-up primer so the basics are automatic
*A patient, plain-English companion to the Dynamo interview prep guide. Read it start to finish once; skim the glossary before you walk in.*

**How this is built:** every concept follows the same rhythm — a one-sentence definition, an everyday analogy, what actually happens under the hood, why it matters (and why interviewers ask), a worked example from your real HPE/Dynamo benchmark, and a one-line way to say it out loud. Read the sections in order; each one builds on the last.

---


## The Big Picture (read this first)

Here's the one story that everything else in this primer hangs on. When you send a prompt to a large language model, the model does its job in **two very different acts**. First it **reads your entire prompt at once** — this is called **prefill**, and it ends the moment the very first word of the answer appears. Then it **writes the answer one token at a time**, looping over and over, each new token feeding back in to help pick the next — this is **decode**. A token is just a chunk of text, about three-quarters of a word, and it's the real unit an LLM reads and writes.

Those two acts stress the GPU in **opposite ways**. Prefill has a mountain of math to do (thousands of prompt tokens processed together), so it's **compute-bound** — the chip's math units are the bottleneck. Decode has almost no math per step, but to produce each tiny token it must re-read the whole model plus its growing memory out of GPU memory, so it's **memory-bandwidth-bound** — the chip sits waiting on memory. Because the two jobs want different things, running them on the *same* GPU makes them fight: one long prefill can freeze everyone else's quick decode steps (that traffic jam has a name — head-of-line blocking).

The fix is to **split the two jobs across different GPUs** — prefill workers over here, decode workers over there. This is **disaggregated serving**, and it's the core idea of the whole stack. It stops the fighting and lets you scale each job on its own. But splitting has a price: the memory of everything prefill just read — the **KV cache** — now lives on the wrong GPU and must be **shipped** to the decode GPU. A fast courier called **NIXL** does exactly that, moving the cache GPU-to-GPU with almost no overhead.

Around this split sits a small **cast of helpers**. **Dynamo** is the air-traffic controller that routes requests and arranges the prefill/decode workers (it never runs the model itself). **vLLM** is the engine that actually runs the model and produces tokens. **KVBM** is the safety net that spills the KV cache to CPU/disk when GPU memory fills, so you get overflow room instead of a crash. And **LMCache** reuses the cache of repeated prompt prefixes so shared text isn't recomputed. Together they make the whole thing fast, roomy, and reliable.

So: **prompt → prefill (read) → first token → decode loop (write) → answer**, all powered by a reused KV cache, split across GPUs and stitched back together by a handful of helpers. As you read on, keep this map in mind — every detail below is just one piece of this single picture.
---
## 1. How an LLM Actually Generates Text (the foundation)

Everything else in the Dynamo stack — caching, routing, splitting work across GPUs — exists to make the loop in this section run faster and cheaper. So we start here. By the end you'll understand exactly what happens between "you type a prompt" and "words stream back," and you'll have crisp one-liners for each idea.

We'll use one running example throughout: the student's benchmark of **Meta-Llama-3.1-8B-Instruct** running on the **vLLM** engine (vLLM = the actual inference engine that does the math), on **8x NVIDIA A100-80GB** GPUs.

---

### (a) Tokens & Tokenization

**In one sentence:** A token is a small chunk of text — often about three-quarters of a word — and tokenization is the step that chops your text into these chunks before the model ever sees it.

**Analogy:** Think of LEGO. You don't build with "a house" as one piece; you build with standardized bricks. Tokens are the standardized bricks of language. Common words are one brick ("the"), rarer words are several bricks ("token" might be "tok" + "en"), and the model only ever handles bricks, never raw letters.

**What actually happens:** Before anything, a *tokenizer* maps your text to a list of integer IDs from a fixed vocabulary (Llama-3.1 has a vocabulary of ~128,000 tokens). The model doesn't see characters or words — it sees numbers like `[128000, 9906, 1917]`. Each number is looked up in an embedding table to get a vector, and that's the model's actual input. A rough rule of thumb in English: **1 token ≈ 0.75 words**, or **100 tokens ≈ 75 words**. Punctuation, spaces, and word-pieces all count.

**Why it matters (and why interviewers ask):** Almost everything in serving is *measured and billed in tokens*, not words or characters — context length limits, throughput, latency, and price. If you think in words, every number you quote will be wrong. Interviewers ask this to check that you know the model's true unit of work.

**From the project:** The benchmark's two workloads are defined purely in tokens. **ISL (Input Sequence Length)** is the prompt size; **OSL (Output Sequence Length)** is how much the model generates.

| Workload | Input (ISL) | Output (OSL) |
|----------|-------------|--------------|
| lowISL   | ~500 tokens | ~256 tokens  |
| highISL  | ~16,000 tokens | ~256 tokens |

A ~16,000-token prompt is roughly 12,000 English words — a small book chapter. Notice both workloads generate the same ~256 output tokens; only the *input* size changes. That single difference is what stresses the two phases we'll meet at the end.

**Say it in one line:** "Models don't read words or letters — they read tokens, and a token is about three-quarters of a word, so everything in serving is counted in tokens."

---

### (b) Autoregressive Generation

**In one sentence:** The model writes one token at a time, and each token it produces is fed back in as input to help produce the next one.

**Analogy:** It's like texting with autocomplete on steroids. You write a word, autocomplete suggests the next; you accept it, and now *that* word becomes part of the context for the next suggestion. The model is playing this "guess the next token" game with itself, over and over, hundreds of times.

**What actually happens:** "Autoregressive" just means *each output depends on all the outputs so far*. The loop is:

```
1. Look at all tokens so far (prompt + everything generated).
2. Predict the single most likely next token.
3. Append that token to the sequence.
4. Go back to step 1 — now with one more token.
```

It repeats until the model emits a special "stop" token or hits the output limit. Crucially, it **cannot** produce token #50 without first having produced tokens #1 through #49 — generation is inherently sequential. That sequential nature is the root reason serving LLMs is hard, and the reason the next idea (the KV cache) exists.

**Why it matters (and why interviewers ask):** This is *why* you can't just generate a 256-token answer in one shot — it takes 256 sequential steps. It explains why output tokens are slower and more expensive to produce than input tokens, and why latency is measured per-token. Interviewers ask it to see if you understand that generation is a loop, not a single function call.

**From the project:** For every request in the lowISL run, the model runs this loop ~256 times to produce the answer. The headline result — **2037.5 output tokens/second** at peak (lowISL, the winning 2P2D setup) — is literally a measure of how fast the system can crank this one-token-at-a-time loop across all concurrent requests.

**Say it in one line:** "Generation is autoregressive: the model predicts one token, feeds it back in, and repeats — so a 256-token reply is 256 sequential steps."

---

### (c) The Forward Pass

**In one sentence:** A forward pass is one trip of the current tokens through the neural network, and out the other end comes a probability score for every possible next token.

**Analogy:** Picture a giant pinball machine. You drop your tokens in the top; they cascade down through 32 layers of bumpers (Llama-3.1-8B has **32 layers**); and at the bottom you get a scoreboard listing all ~128,000 vocabulary tokens, each with a probability of being next. Pick from the scoreboard, and that's your token.

**What actually happens:** The input tokens (as vectors) flow through the network's layers. Each layer does two main things: **attention** (each token looks at other tokens to gather context) and a **feed-forward** step (a learned transformation). After the last layer, the model outputs a vector of *logits* — one score per vocabulary token — which is turned into probabilities. A sampling step then picks the next token. **One forward pass = one predicted next token.** So the autoregressive loop from (b) is really "run a forward pass, append the result, run another forward pass..."

**Why it matters (and why interviewers ask):** The forward pass is the unit of compute. Understanding that *one pass yields exactly one token* is the bridge to everything about performance: it's why prompt-processing and generation behave so differently, and why the KV cache (next) is such a big deal. Interviewers want to confirm you don't imagine the model "typing out a whole sentence" in one go.

**From the project:** Each of Llama-3.1-8B's 32 layers uses attention with **32 query heads** and **8 KV heads** (that 32-sharing-8 design is called **GQA — Grouped-Query Attention**), each head of dimension **128**, all in **BF16 (2 bytes per number)**. Hold onto those exact numbers — in the very next section they let us compute the cache size precisely.

**Say it in one line:** "A forward pass runs the tokens through all 32 layers once and produces a probability distribution over the next token — one pass, one token."

---

### (d) The KV Cache — the single most important idea

**In one sentence:** The KV cache saves the Key and Value vectors of every past token so the model can reuse them instead of recomputing them on every single step.

**Analogy:** Imagine writing a long research summary and, before adding each new sentence, re-reading the *entire* document from page one. That's what a naive model would do. The KV cache is like keeping detailed margin notes: once you've read a paragraph, you jot down its notes once and just glance at them later. You never re-read from scratch.

**What actually happens:** Inside attention, every token produces three vectors: a **Query (Q)**, a **Key (K)**, and a **Value (V)**.
- **Key (K)** = a *label / index* for a token — "here's what I'm about, come find me if you need this."
- **Value (V)** = the *content* the token contributes once it's been matched.
- **Query (Q)** = what the *current* token is looking for.

To generate the next token, the current token's Query is compared against the Keys of **all previous tokens**, and the matching Values are pulled in. Here's the key insight: the K and V of the past tokens **don't change** as generation continues. Recomputing them every step would cost O(n²) work over a sequence (each new token re-deriving K/V for all prior tokens). So instead the model computes each token's K and V **once**, stores them, and on every later step just **reuses** the cached K/V and **appends** the new token's K/V. Recompute becomes reuse.

```
Without KV cache (naive):        With KV cache:
step 5 recomputes K,V for        step 5 reuses stored K,V for
tokens 1,2,3,4,5   <- wasteful   tokens 1..4, computes only token 5
```

The trade-off: reuse costs *memory*. The cache grows with every token, every layer, and every request — and GPU memory is finite. That tension (fast reuse vs. memory pressure) is the reason most of the Dynamo stack exists (NIXL moves it, KVBM spills it to CPU/disk, LMCache reuses it across prompts — all covered in later clusters).

**Why it matters (and why interviewers ask):** The KV cache is *the* reason modern LLM serving is fast enough to be practical, and its memory footprint is the #1 constraint on how many requests you can run at once. If you understand only one thing about inference internals, make it this. It comes up in almost every LLM-systems interview.

**From the project:** The cache size is not hand-wavy — it's arithmetic you can do from Llama-3.1-8B's specs. Per token, the KV cache stores:

```
2 (K and V) x 32 layers x 8 KV heads x 128 head_dim x 2 bytes (BF16)
  = 131,072 bytes ≈ 128 KB per token
```

Multiply by the sequence length and you get the cache for one request. In the project, the **theoretical KV size of 2.13 GB matched the measured NIXL transfer of 2.147 GB** — a near-perfect match that confirms the model of how the cache is built. (Note GQA is doing real work here: 8 KV heads instead of 32 makes the cache 4x smaller than it would otherwise be.)

**Say it in one line:** "The KV cache stores each past token's Key (its label) and Value (its content) once and reuses them every step, turning O(n²) recompute into cheap reuse — at the cost of GPU memory."

---

### (e) The Two Phases: Prefill vs. Decode

**In one sentence:** Prefill processes the whole prompt in one shot to build the initial KV cache and emit the first token; decode then generates the rest of the tokens one at a time, reusing and extending that cache.

**Analogy:** Prefill is *reading the whole question* — you can take it in all at once, in parallel, before you start answering. Decode is *writing the answer word by word* — you can only produce the next word after the previous one is down on paper.

**What actually happens:** These two phases fall directly out of everything above.

- **Prefill:** You already have all the prompt tokens up front, so the model processes them *together in parallel* in effectively one big forward pass. This computes and stores K/V for every prompt token (filling the cache) and produces the **first** output token.
- **Decode:** Now you're in the autoregressive loop. Each step is one forward pass that reads the existing cache, computes K/V for just the **one** new token, appends it, and emits the next token. Repeat until done.

```
PREFILL                        DECODE (loops OSL-1 times)
all prompt tokens at once  ->   one token in, one token out, reuse cache
builds full KV cache            extends KV cache by one entry each step
emits 1st token                 emits tokens 2, 3, 4, ...
```

Because prefill crunches many tokens at once, it leans on raw compute; because decode does tiny one-token steps that mostly shuttle the growing cache around, it leans on memory bandwidth. That's why people say **prefill is "compute-heavy" and decode is "memory-heavy"** — but the *why* behind that, and how Dynamo splits prefill and decode across separate GPU workers, is Cluster 3's job. For now, just lock in that these are two genuinely different kinds of work.

**Why it matters (and why interviewers ask):** This prefill/decode split is the mental model the entire serving stack is organized around — disaggregation, worker ratios, and most optimizations only make sense once you see these as two phases. Interviewers ask it as the natural follow-up to the KV cache; naming both phases and their different resource profiles signals you actually understand inference.

**From the project:** The benchmark's two workloads isolate the two phases. **highISL** has a ~16,000-token prompt but only ~256 output tokens — a giant prefill, ordinary decode. **lowISL** has a ~500-token prompt with the same ~256 output — a small prefill, so decode dominates. The whole sweep of **worker ratios — 1P1D, 1P2D, 2P1D, 2P2D** (P = prefill workers, D = decode workers) — is just tuning how many GPU workers handle each phase. **2P2D won both workloads**, and the KV-cache-focused optimizations paid off dramatically: **KVBM cut TTFT (Time To First Token) by up to 95.3%**, and reusing cached prefixes with **LMCache** hit an **87.9% token hit rate on repeated prompts** (vs. 1.0% when every prompt was unique), pulling TTFT down to **418ms from 756ms**.

**Say it in one line:** "Inference has two phases: prefill reads the whole prompt at once to build the KV cache and give the first token (compute-heavy), then decode generates the rest one token at a time by reusing that cache (memory-heavy)."

---

### Putting it all together

```
   your prompt
   "~500 tokens"
        |
        v
  +-----------+     builds full KV cache,
  |  PREFILL  |     processes all prompt tokens in parallel
  +-----------+
        |
        v
   first token  ------------------+
        |                         |
        v                         |
  +-----------+   reuse + extend  |  each pass:
  |  DECODE   |   the KV cache     |  1 token in -> 1 token out
  |  (loop)   | <-----------------+   append to cache, repeat
  +-----------+
        |
        v
  tokens 2, 3, 4, ... 256   ->   the full answer
```

That loop — **prompt -> prefill -> first token -> decode loop -> more tokens**, all powered by a reused KV cache — is the beating heart of LLM serving. Every clever thing Dynamo, vLLM, NIXL, KVBM, and LMCache do is in service of running it faster, cheaper, and at higher concurrency.


---


## 2. The Vocabulary of Speed (the metrics interviewers quiz you on)

Before we talk about *how* to make an LLM (Large Language Model) fast, we need the words for *what "fast" even means*. Serving has its own dialect, and interviewers use these terms as a quick filter: if you say "latency" when you mean "throughput," they stop trusting you. This section makes the whole vocabulary automatic.

One idea to keep in your head the whole way through: every request has two phases. **Prefill** reads your prompt (all input tokens at once). **Decode** writes the answer (one token at a time). Almost every metric below is really asking "how good is prefill?" or "how good is decode?" (Those two phases get their own deep-dive in another cluster; here we only need the labels.)

```
   You hit "send"
        |
        v
  [ PREFILL ]  reads the whole prompt ......... controls TTFT
        |
        v  (first token pops out)
  [ DECODE ]   writes token, token, token ..... controls ITL
        |
        v
   Answer complete
```

---

### (a) ISL — Input Sequence Length

**In one sentence:** ISL is how many tokens are in the prompt you send in.

**Analogy:** It is the number of words in the question you hand to a tutor before they start answering.

**What actually happens:** Your text is chopped into tokens (roughly word-pieces; "tokenization" is covered elsewhere). ISL counts those input tokens. All of them get processed together during prefill, so ISL is the main driver of how much prefill work there is.

**Why it matters (and why interviewers ask):** Prefill cost grows with ISL, so ISL is the single biggest lever on TTFT. A short prompt feels instant; a giant prompt makes the user wait before *anything* appears. Interviewers use ISL to check you understand that "the prompt is not free."

**From the project:** The two workloads were defined by ISL. **lowISL ≈ 500 input tokens** and **highISL ≈ 16,000 input tokens** — a 32x difference in prompt size, same model, same hardware.

**Say it in one line:** "ISL is the prompt length in tokens, and it dominates prefill cost."

---

### (b) OSL — Output Sequence Length

**In one sentence:** OSL is how many tokens the model generates in its answer.

**Analogy:** It is the length of the tutor's reply — a one-word "Yes" versus a three-paragraph explanation.

**What actually happens:** Each output token is produced one at a time in the decode phase. Every new token requires one more forward pass through the model, so OSL directly sets how many decode steps you pay for.

**Why it matters (and why interviewers ask):** OSL drives the *decode* portion of latency the way ISL drives prefill. Two requests with identical prompts can have wildly different total times if one writes 10 tokens and the other writes 500. Interviewers want to see you separate "prompt size" (ISL) from "answer size" (OSL).

**From the project:** Both workloads fixed **OSL ≈ 256 output tokens**, on purpose. Holding OSL constant means any difference in results came from ISL — a clean experiment.

**Say it in one line:** "OSL is the answer length in tokens, and it sets how many decode steps you pay for."

---

### (c) Context length = ISL + OSL (and the context window)

**In one sentence:** Context length is the total number of tokens the model is juggling at once — the prompt plus everything generated so far.

**Analogy:** It is the total length of the conversation transcript on the desk, growing by one line every time the tutor speaks.

**What actually happens:** As decode proceeds, each new output token gets appended to the running sequence, so context length climbs from ISL up toward ISL + OSL. Every model has a hard **context window** — the maximum context length it can handle (for example, tens of thousands of tokens). Exceed it and the request errors or gets truncated. Longer context also means a bigger KV cache (the memory of past tokens; sized in another cluster).

**Why it matters (and why interviewers ask):** The context window is a real capacity limit you design around. If ISL is already 16,000, you have far less room left for a long answer, and your memory footprint is large. Interviewers ask this to see whether you know a model has a ceiling, not infinite memory.

**From the project:** highISL runs sat at roughly **16,000 (ISL) + 256 (OSL) ≈ 16,256 tokens** of context per request — comfortably inside Llama-3.1-8B's large window, but heavy enough that memory management (KVBM) started to matter.

**Say it in one line:** "Context length is ISL + OSL, and it must stay under the model's context window."

---

### (d) TTFT — Time To First Token

**In one sentence:** TTFT is how long you wait from hitting "send" until the very first token of the answer appears.

**Analogy:** It is the awkward pause after you ask a question before the other person opens their mouth — once they start talking, TTFT is over.

**What actually happens:** TTFT is basically the cost of prefill (plus a little queueing). The model has to read the entire prompt before it can predict token #1, so bigger ISL means bigger TTFT. This is the "responsiveness" metric — it decides whether an app feels snappy or frozen.

**Why it matters (and why interviewers ask):** TTFT is what users *feel* first. A chatbot with great throughput but a 10-second TTFT feels broken. It is also the metric most sensitive to ISL, so it is the classic interview probe: "What makes TTFT go up?" (Answer: long prompts, cold cache, queueing.)

**From the project:** This is the most dramatic number in the whole study. With a short prompt, **TTFT ≈ 7.2 s**. With the 16k-token prompt, prefill exploded to **TTFT ≈ 917.7 s** — over 15 minutes to the first token, because prefill work scales hard with ISL. That gap is *the* argument for prompt-length-aware serving.

**Say it in one line:** "TTFT is submit-to-first-token, it's dominated by prefill, and it's the responsiveness metric."

---

### (e) TTST — Time To Second Token

**In one sentence:** TTST is how long you wait for the *second* token after sending the request.

**Analogy:** After the tutor says their first word, TTST is the beat before the second word — the moment you learn whether they are going to speak fluently or stutter.

**What actually happens:** TTST ≈ TTFT + the time for one decode step. It is useful because it isolates the very first decode step, which sometimes behaves differently from later ones (the engine may transition out of prefill, warm up caches, or reshape the batch). Comparing TTST to TTFT shows how quickly the system shifts from "reading" to "writing."

**Why it matters (and why interviewers ask):** It is a diagnostic metric. If TTFT is fine but the gap to the second token is huge, your prefill-to-decode handoff is the bottleneck, not prefill itself. Interviewers mention it to see if you know latency is more than one number.

**From the project:** In the disaggregated setups (separate prefill and decode workers), the first token is produced on a prefill worker and later tokens on a decode worker, so the prefill→decode handoff — moving the KV cache over NIXL — lands right in the TTFT-to-TTST window. Watching TTST is how you check that handoff is not stalling.

**Say it in one line:** "TTST is time-to-second-token, and it exposes how cleanly the system hands off from prefill to decode."

---

### (f) ITL — Inter-Token Latency

**In one sentence:** ITL is the average time gap between two consecutive output tokens once the model is generating.

**Analogy:** It is the rhythm of speech — the pause between each spoken word. Small, even pauses feel fluent; long pauses feel like a bad phone connection.

**What actually happens:** During decode, each token needs one forward pass, and ITL is how long that pass takes (measured as the wall-clock gap between token N and token N+1). It is the "smoothness" metric: it controls how fast text streams onto the screen. ITL is set by decode speed, which depends on the model size, hardware, and how many other requests are batched alongside yours.

**Why it matters (and why interviewers ask):** For streaming UIs, ITL *is* the reading experience. If ITL is comfortably below human reading speed, the answer feels instant even for long outputs. Interviewers pair it with TTFT to check you can name both halves: TTFT = "how long until it starts," ITL = "how smoothly it continues."

**From the project:** Decode ran at roughly **ITL ≈ 20 ms per token**. That is about 50 tokens/second to a single user — faster than anyone reads — so even a 256-token answer streams smoothly once the first token arrives.

**Say it in one line:** "ITL is the average gap between output tokens during decode — the smoothness metric."

---

### (g) TPOT — Time Per Output Token (and how it differs from ITL)

**In one sentence:** TPOT is the average time spent per generated token, usually computed as total decode time divided by the number of output tokens.

**Analogy:** ITL is timing each individual footstep with a stopwatch; TPOT is measuring the whole walk and dividing by the number of steps. Same walk, two ways of reporting pace.

**What actually happens:** The two are close cousins and people often use them interchangeably, but the subtle difference is *how they're derived*:

| | How it's obtained | Nature |
|---|---|---|
| **ITL** | Measured gap between each pair of consecutive tokens | Per-token, can show jitter |
| **TPOT** | (decode time) ÷ (output tokens) | An average, smooths out jitter |

Because TPOT divides by output tokens, it excludes the first-token wait (TTFT) and reports the steady-state decode pace. ITL, being a direct measurement, can reveal *variance* — one slow token in a stream — that TPOT's averaging hides.

**Why it matters (and why interviewers ask):** Knowing they are near-synonyms *but not identical* is exactly the kind of nuance that separates someone who memorized a glossary from someone who understands it. If asked "TPOT vs ITL?", the winning answer is: "Same idea — per-token decode time — but TPOT is the averaged version, ITL is the measured gap, so ITL can surface jitter."

**From the project:** With decode steady around 20 ms, TPOT and ITL land in the same ballpark (~20 ms/token); they diverge only if a few tokens stall, which is precisely when you'd look at ITL instead of the TPOT average.

**Say it in one line:** "TPOT is decode-time-per-token as an average; ITL is the measured token-to-token gap — same pace, TPOT smooths, ITL shows jitter."

---

### (h) End-to-end latency

**In one sentence:** End-to-end latency is the total time for a request, from submit to the final token.

**Analogy:** It is the full round trip: the pause before the tutor speaks, plus the time to say every word of the answer.

**What actually happens:** You can approximate it by adding the startup cost to the streaming cost:

```
end-to-end  ≈  TTFT  +  (OSL - 1) x ITL
                |            |
            first token   the other output tokens,
             (prefill)    one ITL gap apart (decode)
```

TTFT covers getting token #1 out; the remaining OSL − 1 tokens each add roughly one ITL. This is an estimate — real systems have scheduling and batching wobble — but it captures the shape perfectly.

**Why it matters (and why interviewers ask):** It shows you can compose the metrics instead of reciting them. It also reveals *which* term dominates: for short answers TTFT rules; for long answers the ITL term takes over. Interviewers love asking you to derive this on the spot.

**From the project:** For a lowISL request: **≈ 7.2 s + (256 − 1) x 0.020 s ≈ 7.2 s + 5.1 s ≈ 12.3 s**. Notice both halves matter here. For highISL, TTFT alone (≈ 917.7 s) utterly dwarfs the ~5 s of decode — proof that with huge prompts, prefill *is* the latency.

**Say it in one line:** "End-to-end ≈ TTFT + (OSL − 1) x ITL — prefill startup plus per-token decode."

---

### (i) Throughput — tokens/sec, requests/sec, tokens/sec/GPU

**In one sentence:** Throughput is how much total work the system completes per unit time, across *all* users at once.

**Analogy:** Latency is how fast one car finishes the race; throughput is how many cars the highway moves per hour. A packed highway can move lots of cars while any single car crawls.

**What actually happens:** The same serving system is measured three common ways:

| Unit | Question it answers |
|---|---|
| **output tokens/sec** | How fast is text produced overall? |
| **requests/sec** | How many prompts finish per second? |
| **tokens/sec/GPU** | Throughput normalized per GPU (efficiency / cost) |

The trick that makes throughput high is **batching** — running many requests through the GPU together, so one pass over the model's weights serves everyone at once. That is why total tokens/sec can be far higher than any single user's ITL would suggest.

**Why it matters (and why interviewers ask):** Throughput is the cost/capacity metric — it decides how many users a GPU box can serve and therefore the bill. tokens/sec/GPU is what you actually compare across configs, because raw tokens/sec is unfair if one setup uses more GPUs. Interviewers check you never confuse this with latency.

**From the project:** The winning **2P2D** config (2 prefill + 2 decode workers) hit a **peak 2037.5 output tokens/sec** on lowISL at concurrency 100. On **8x A100-80GB**, dividing gives a rough efficiency figure of ~255 tokens/sec/GPU — the kind of normalized number you'd cite to compare hardware or configs fairly.

**Say it in one line:** "Throughput is total work per second — tokens/sec, requests/sec, or tokens/sec/GPU — and batching is what drives it up."

---

### (j) Latency vs throughput — the fundamental trade-off

**In one sentence:** Making the system serve more users at once (throughput) usually makes each individual user wait a bit longer (latency), and vice versa.

**Analogy:** A restaurant kitchen. Cook one dish at a time and that guest is served instantly, but the kitchen feeds few people per hour. Batch-cook for a full dining room and total output soars, but each plate waits for the batch. You cannot maximize both.

**What actually happens:** Bigger batches share the fixed cost of a forward pass across more requests, so tokens/sec/GPU climbs. But each request now shares the GPU, so its ITL and TTFT rise. Push the batch too far and latency blows past what users tolerate; keep batches tiny and the GPU sits half-idle. Good serving is picking the point on this curve that fits your latency budget.

**Why it matters (and why interviewers ask):** This is *the* core tension of LLM serving, and almost every design decision — batch size, concurrency, prefill/decode split — is a knob on this trade-off. If you internalize one idea from this whole section, make it this one. It is the most common conceptual interview question in the field.

**From the project:** Benchmarking at **concurrency 100** deliberately sat on the throughput end — packing 100 in-flight requests to push tokens/sec up (peaking at 2037.5), knowingly accepting higher per-request latency than a single lonely request would see. The whole study is an exploration of this curve.

**Say it in one line:** "Latency and throughput trade off — batching more requests raises total throughput but slows each one down."

---

### (k) Concurrency and batch size

**In one sentence:** Concurrency is how many requests are in flight at once; batch size is how many the GPU actually processes together in a single step — and turning either up trades latency for throughput.

**Analogy:** Concurrency is how many customers are inside the store; batch size is how many the single cashier rings up in one combined transaction. More people in the store, and bigger combined transactions, means more sales per hour but a longer wait for each person.

**What actually happens:** Concurrency is the *offered load* (requests arriving/waiting); batch size is the engine's *internal grouping* per forward pass. They're linked: high concurrency gives the scheduler enough waiting requests to form big batches. Bigger batches → better GPU utilization → higher throughput, but each request shares compute → higher TTFT and ITL. It's the concrete control knob for the latency-vs-throughput curve in (j).

**Why it matters (and why interviewers ask):** These are the dials you actually turn in production. "How would you raise throughput?" → raise concurrency/batch size and accept higher latency. "Why did latency spike?" → batches got too big under load. Interviewers want the cause-and-effect, not just definitions.

**From the project:** Every benchmark used **concurrency 100** — a high, throughput-oriented load. That's what let 2P2D form large enough batches to reach 2037.5 tokens/sec, at the cost of per-request latency being higher than a single isolated request.

**Say it in one line:** "Concurrency = requests in flight, batch size = requests processed together; raising either buys throughput at the cost of latency."

---

### (l) SLA, SLO, and goodput

**In one sentence:** An SLA is the promise you make to users about performance, an SLO is the stricter internal target you engineer to, and goodput is the throughput that actually *meets* that target.

**Analogy:** The SLA is the pizza shop's public sign: "delivered in 30 minutes or it's free." The SLO is the kitchen's private rule: "aim for 25 minutes," leaving a safety margin. Goodput is counting only the pizzas that arrived *on time* — the late ones don't count, even though you still made them.

**What actually happens:**

| Term | Full name | Who it's for | Example |
|---|---|---|---|
| **SLA** | Service Level Agreement | The customer (the promise) | "TTFT under 500 ms" |
| **SLO** | Service Level Objective | Your team (the internal goal) | "TTFT under 400 ms" |
| **Goodput** | — | Capacity planning | tokens/sec that met the SLO |

The key insight is **goodput ≤ throughput**. Raw throughput counts every token you produced; goodput counts only tokens from requests that satisfied the latency target. If you crank concurrency so high that half your requests blow the TTFT budget, throughput looks great but goodput collapses — you're producing output nobody accepts.

**Why it matters (and why interviewers ask):** Goodput is the metric that ties the whole section together: it forces latency and throughput into a *single* honest number. It's why you can't just chase peak tokens/sec. Interviewers ask about SLA/SLO to check you think about serving as a *commitment*, not just a benchmark, and goodput is an increasingly popular "do they really get it?" question.

**From the project:** The LMCache result shows goodput thinking directly. Against a TTFT target, reusing repeated prompt prefixes cut **TTFT from 756 ms to 418 ms** (87.9% token hit rate). Requests that would have missed a ~500 ms SLA now comfortably meet it — so the *goodput* (SLO-satisfying throughput) rises even if raw token counts look similar.

**Say it in one line:** "SLA is the promise, SLO is the stricter internal target, and goodput is the throughput that actually meets it — the number that keeps you honest."

---

### Master summary table

Every metric in this cluster, at a glance. "High is..." tells you whether a bigger number is good or bad.

| Metric | What it measures | Phase | High is... |
|---|---|---|---|
| **ISL** | Input/prompt tokens | Prefill (drives it) | neutral (bigger prompt = more prefill cost) |
| **OSL** | Generated output tokens | Decode (drives it) | neutral (bigger answer = more decode steps) |
| **Context length** | ISL + OSL in flight | Both | must stay under context window |
| **TTFT** | Submit → first token | Prefill | **bad** (less responsive) |
| **TTST** | Submit → second token | Prefill→decode handoff | **bad** (handoff stalling) |
| **ITL** | Gap between output tokens | Decode | **bad** (choppier stream) |
| **TPOT** | Avg decode time per token | Decode | **bad** (slower generation) |
| **End-to-end latency** | Submit → last token | Both | **bad** (slower overall) |
| **Throughput** | tokens/sec, req/sec, tok/sec/GPU | Both (system-wide) | **good** (more work done) |
| **Concurrency / batch size** | Requests in flight / grouped | Both | trade-off (↑throughput, ↑latency) |
| **SLA / SLO** | Promised / target performance | Policy | tighter = stricter promise |
| **Goodput** | Throughput meeting the SLO | Both | **good** (useful work done) |

**The one mental model to walk in with:** *Prefill sets TTFT, decode sets ITL, end-to-end is TTFT + (OSL−1)×ITL, throughput fights latency through batching/concurrency, and goodput is the only number that respects both.*


---


## 3. Why We Split Prefill and Decode (the core insight)

Every LLM (Large Language Model) request runs in two very different phases. First it reads your whole prompt (**prefill**). Then it writes the answer one token at a time (**decode**). These two phases stress the GPU in opposite ways. Understanding *why* is the single biggest idea behind the Dynamo serving stack — and the reason "disaggregated serving" exists at all.

First, three quick terms we will reuse:
- **Token:** a chunk of text (roughly ¾ of a word) — the unit LLMs read and write.
- **ISL / OSL (Input / Output Sequence Length):** how many tokens go in (the prompt) and come out (the answer). In this project, lowISL ≈ 500 input tokens, highISL ≈ 16000, both with OSL ≈ 256.
- **KV cache (Key-Value cache):** the model's "memory" of every token seen so far, saved in GPU memory so it doesn't re-read the prompt on every step. (Sized in detail in another cluster — here we just use it.)

---

### 3a. The key asymmetry: prefill is compute-bound, decode is memory-bandwidth-bound

**In one sentence:** Prefill is limited by how fast the GPU can do *math*, while decode is limited by how fast the GPU can *read from its own memory*.

**Analogy:** Prefill is cooking one big batch meal — you chop and stir a mountain of ingredients all at once, and your arms (the math units) are the bottleneck. Decode is fetching a single spice from an enormous pantry, walking back, then doing it again for every dish — the walking (memory reads), not the cooking, is what slows you down.

**What actually happens:**
- **Prefill** processes *all* input tokens in parallel. With ISL = 16000, the GPU multiplies huge matrices covering thousands of tokens at once. That is a mountain of arithmetic, and it saturates the GPU's math units (the tensor cores). The chip is busy *computing* — this is **compute-bound**.
- **Decode** generates just *one* token per step. There is almost no math to do. But to produce that single token, the GPU must re-read the *entire* set of model weights (8 billion parameters for Llama-3.1-8B) plus the growing KV cache, out of GPU memory, every single step. The math finishes in a flash; the chip sits waiting on memory. This is **memory-bandwidth-bound**.

```
PREFILL (compute-bound)              DECODE (memory-bound)
  all 16000 tokens at once             1 token per step, ~256 times
  ┌───────────────────────┐            step ─► read ALL 8B weights + KV
  │  BIG matrix multiply  │            step ─► read ALL 8B weights + KV
  │  math units MAXED OUT │            step ─► read ALL 8B weights + KV
  └───────────────────────┘            ...bottleneck = memory speed, not math
```

**Why it matters (and why interviewers ask):** This one asymmetry explains almost every design choice in modern serving. Throwing more raw compute at decode barely helps (it's waiting on memory); throwing more compute at prefill helps a lot. If you can say *why* the two phases have different bottlenecks, you've shown you understand LLM serving at the mechanical level, not just the buzzword level.

**From the project:** With highISL ≈ 16000 tokens, prefill is a genuinely heavy compute job — a 16000-token matrix multiply — which is exactly why extra prefill workers paid off (see 3f). Decode's per-step cost is dominated by re-reading weights + KV; that's why techniques that shrink KV memory traffic matter so much downstream.

**Say it in one line:** "Prefill saturates the GPU's math on the whole prompt at once; decode is one tiny token at a time, bottlenecked by re-reading the model and KV cache from memory."

---

### 3b. Aggregated serving: one worker does both phases

**In one sentence:** In aggregated serving, the same GPU worker runs *both* prefill and decode for a request, back to back.

**Analogy:** One chef takes your order, cooks the big batch, *and* plates every single dish one by one — no handoffs, but that one chef is doing two very different jobs on the same station.

**What actually happens:** A request arrives, the worker does the heavy prefill, then loops through decode steps until the answer is done — all on the same GPU. Nothing is transferred anywhere, because the KV cache built during prefill is already sitting in that GPU's memory, ready for decode. Simple and cheap to operate.

**Why it matters (and why interviewers ask):** Aggregated is the natural default and the baseline everything else is compared against. Its weakness is the whole point of this cluster: because one GPU juggles both phases, a big compute-bound prefill and many memory-bound decodes *fight over the same hardware* at the same time. Interviewers want to hear the trade-off: zero transfer cost, but the two phases interfere.

**From the project:** The 1P1D configuration (1 Prefill worker, 1 Decode worker) is the simplest split; a purely aggregated setup is even simpler, one worker per request doing everything. It works, but under concurrency 100 (100 requests in flight at once, via the AIPerf benchmark tool) the interference shows up fast — which motivates the next problem.

**Say it in one line:** "Aggregated = one GPU does prefill and decode with no transfer cost, but the two phases compete for the same chip."

---

### 3c. Head-of-line blocking: one long prefill stalls everyone's decode

**In one sentence:** When a long prefill hogs the GPU, all the other requests waiting to do their tiny decode steps get stuck behind it, spiking tail latency.

**Analogy:** One customer at the checkout with a 300-item cart (a long prefill) while ten people holding a single banana each (decode steps) wait behind them. The bananas would take two seconds — but they're blocked by the big cart at the *head of the line*.

**What actually happens:** The GPU processes work in batches. A 16000-token prefill is a long, compute-heavy chunk. While it runs, the in-flight requests that just need one quick decode token each cannot make progress. Their **TTFT (Time To First Token — how long until the user sees the first word)** and their between-token latency both jump. The average might look okay, but the **tail latency** (the slowest requests, e.g. the 99th percentile) gets ugly — and tail latency is what users actually feel.

**Why it matters (and why interviewers ask):** "Head-of-line blocking" is a classic systems term, and interviewers love that it reappears in LLM serving. It's the concrete pain that justifies disaggregation. If you can name it and explain that *a long compute-bound prefill starves many memory-bound decodes*, you've connected a general CS concept to this domain.

**From the project:** With inputs as long as 16000 tokens, a single prefill is genuinely large. Run 100 of those concurrently on shared workers and the long prefills repeatedly elbow ahead of everyone's decode steps — precisely the stall that the disaggregated design removes.

**Say it in one line:** "A long prefill blocks the queue like a giant shopping cart at checkout, freezing everyone else's quick decode steps and spiking tail latency."

---

### 3d. Disaggregated serving: prefill GPUs and decode GPUs, kept apart

**In one sentence:** Disaggregated serving puts prefill on one dedicated set of GPUs and decode on another, so each phase runs on hardware tuned and scaled just for it.

**Analogy:** A restaurant with a *prep kitchen* (heavy chopping and batch-cooking = prefill) that's separate from the *plating station* (assembling dishes one at a time = decode). Prep never blocks plating, and you can staff each station independently based on demand.

**What actually happens:** The **Frontend** (the OpenAI-compatible API gateway/router) sends a request to a **prefill worker**, which does the big matrix multiply and builds the KV cache. That KV cache is then handed to a **decode worker**, which generates the output tokens. Because prefill and decode live on different GPUs, a long prefill can no longer block anyone's decode — head-of-line blocking is designed away. You also get to scale them separately: more prefill workers for prompt-heavy traffic, more decode workers for answer-heavy traffic.

```
                 ┌── Prefill GPUs ──┐        ┌── Decode GPUs ──┐
request ─►Frontend│ big matmul over  │  KV    │ 1 token/step,   │─► tokens
                 │ all input tokens │ ─────► │ re-reads weights│   to user
                 └──────────────────┘ cache  └─────────────────┘
                    compute-bound   (shipped)   memory-bound
```

**Why it matters (and why interviewers ask):** This is *the* architecture Dynamo is built to enable, and the headline reason the project swept prefill:decode ratios at all. The interview payoff: disaggregation lets you independently tune two workloads with opposite bottlenecks, and it kills head-of-line blocking — at the price of moving the KV cache (next).

**From the project:** The project swept **1P1D, 1P2D, 2P1D, and 2P2D** (P = prefill workers, D = decode workers) on 8× A100-80GB GPUs. These are disaggregated layouts: separate workers own separate phases. **2P2D won both workloads**, peaking at **2037.5 output tokens/second on lowISL** — the concrete win from keeping the phases apart and giving each enough hardware.

**Say it in one line:** "Disaggregation runs prefill and decode on separate GPUs so neither blocks the other and each scales on its own."

---

### 3e. The cost of disaggregation: the KV cache must be shipped

**In one sentence:** Splitting the phases means the KV cache built on the prefill GPU has to be physically moved to the decode GPU before decoding can start.

**Analogy:** The prep kitchen and the plating station are now in different rooms, so every dish's half-finished components must be carried down the hall. That trip is real work you didn't have when one chef did everything.

**What actually happens:** During prefill, the model fills the KV cache with an entry for every input token. In aggregated serving that cache is already where decode needs it. In disaggregated serving it lives on the *wrong* GPU, so it must be transferred — over **NVLink** (NVIDIA's fast GPU-to-GPU interconnect) within a node, or across the network between nodes. In Dynamo this transfer is the job of **NIXL** (covered fully in cluster 4). The transfer isn't free, so it only pays off when the benefit — no blocking, independent scaling — outweighs the move.

How big is that move? We can size it from Llama-3.1-8B's KV facts. Per token, the KV cache is:

```
2 (K and V) × 32 layers × 8 KV heads × 128 head-dim × 2 bytes (BF16)
        = 131,072 bytes  ≈  128 KB per token
```

(Note the **GQA — Grouped-Query Attention** trick: 32 query heads *share* just 8 KV heads, which is why we multiply by 8, not 32. That alone shrinks the KV cache 4×.)

| Workload | Input tokens | KV per token | KV to ship (approx) |
|---|---|---|---|
| lowISL | ~500 | ~128 KB | ~0.06 GB |
| highISL | ~16000 | ~128 KB | ~2 GB |

**Why it matters (and why interviewers ask):** Every optimization has a cost, and the honest answer to "why not always disaggregate?" is *the KV transfer*. Interviewers want to see you name the trade-off and know it's a measurable amount of data, not hand-waving.

**From the project:** For highISL, the **theoretical KV size of 2.13 GB matched the measured NIXL transfer of 2.147 GB** — the theory and the real bytes-on-the-wire agree almost exactly. That's the concrete "cost of disaggregation" you can put a number on.

**Say it in one line:** "Disaggregation's price is moving the KV cache from prefill to decode GPUs — about 2.13 GB per highISL request, which NIXL actually ships."

---

### 3f. The P:D ratio: how many prefill vs decode workers

**In one sentence:** The P:D ratio is how you split your GPUs between prefill and decode workers, and the best split depends on whether your workload is prompt-heavy or answer-heavy.

**Analogy:** Staffing a kitchen. A catering job with huge ingredient prep (long prompts) needs more prep cooks (prefill). A tasting menu with tiny plates but many courses (long outputs) needs more platers (decode). You match your staff to the actual menu.

**What actually happens:** Long inputs make prefill the bottleneck — that giant compute-bound matrix multiply takes real time, so you want *more prefill workers* to chew through prompts in parallel. Long outputs make decode the bottleneck — more memory-bound steps per request — so you want *more decode workers*. There's no universal best ratio; it tracks the ISL:OSL shape of your traffic.

| Config | Prefill | Decode | Leans toward |
|---|---|---|---|
| 1P2D | 1 | 2 | short prompts, longer answers |
| 2P1D | 2 | 1 | long prompts, short answers |
| 2P2D | 2 | 2 | balanced power for both phases |

**Why it matters (and why interviewers ask):** This is where the theory becomes a tuning knob you actually turn. Interviewers ask because it proves you can reason from *workload shape* to *hardware layout* — the core skill of a serving engineer.

**From the project:** **2P2D won both lowISL and highISL.** The intuition for highISL is the punchline of this whole cluster: with ISL ≈ 16000, prefill is an enormous compute-bound job, so going from 1 to **2 prefill workers** is what unlocked throughput — two workers grind through those 16000-token prompts in parallel instead of one struggling alone. Two decode workers then kept the answer generation flowing. (On lowISL, prefill is cheap, so 2P2D's edge came more from decode headroom — but the balanced 2P2D still topped the charts at **2037.5 output tok/s**.)

**Say it in one line:** "The P:D ratio matches workers to workload shape — long inputs demand more prefill power, which is exactly why 2 prefill workers won at ISL ≈ 16000."

---

**Cluster recap — the core insight in one breath:** Prefill is compute-bound and decode is memory-bandwidth-bound; running both on one GPU (aggregated) is simple but lets long prefills head-of-line-block decodes; disaggregating onto separate prefill and decode GPUs fixes that and lets each scale independently, at the cost of shipping the KV cache (NIXL, ~2.13 GB per highISL request); and you tune the P:D ratio to the workload — which is why **2P2D**, with its extra prefill muscle, won both of this project's workloads.


---


## 4. The Dynamo Cast of Characters (who does what)

Before we meet each piece, here's the whole cast on one stage. When a request comes in to serve a Llama-3.1-8B answer, it flows through these players in order:

```
        Request (OpenAI-style API call)
                 |
                 v
        [ Frontend / Router ]   <- picks the best worker
                 |
                 v
   +------------ Dynamo (orchestration) ------------+
   |   coordinates workers, disaggregation, KV      |
   |                                                |
   |   [Prefill worker]  --NIXL-->  [Decode worker] |
   |     runs vLLM                     runs vLLM     |
   |        |                             |          |
   |     KVBM (spill to CPU/disk)   KVBM             |
   |     LMCache (reuse prefixes)                    |
   +------------------------------------------------+
```

Keep that picture in mind. Now each character, one at a time.

---

### (a) NVIDIA Dynamo — the orchestration layer

**In one sentence:** Dynamo is the coordinator that sits *above* the inference engines and manages many GPUs and workers across a datacenter — it does *not* run the model itself.

**Analogy:** Dynamo is the air-traffic controller, not the airplane. It decides which plane (worker) takes which passenger (request), when planes take off, and how cargo (KV cache) gets moved between them — but it never flies the plane.

**What actually happens:** Dynamo handles the "big picture" jobs that a single model-runner can't: routing requests to the right worker, splitting work into a *prefill* stage and a *decode* stage on different GPUs (called disaggregation), and moving the resulting KV cache between those GPUs. Underneath it, an actual inference engine does the math. In this project, Dynamo orchestrated 8x A100-80GB GPUs, arranging them into prefill and decode workers (the 2P2D layout that won both workloads).

**Why it matters (and why interviewers ask):** The single most common beginner mistake is thinking "Dynamo serves the model." It doesn't. It orchestrates the things that do. Interviewers ask this to check you understand the *layers* of a modern serving stack — that scaling to a datacenter is a different job from running one forward pass.

**From the project:** Dynamo is what let the student sweep worker layouts (1P1D, 1P2D, 2P1D, 2P2D) and pick a winner. The engine (vLLM) couldn't do that on its own — Dynamo arranged the workers and 2P2D peaked at **2037.5 output tok/s** on lowISL.

**Say it in one line:** "Dynamo is the orchestration layer above the engine — it routes, disaggregates, and moves KV, but it never runs the model itself."

---

### (b) The inference engine (vLLM) — the thing that actually runs the model

**In one sentence:** The inference engine is the component that actually executes the model's forward passes on a GPU to produce tokens.

**Analogy:** If Dynamo is air-traffic control, vLLM is the airplane with its engine — the thing that actually does the flying (the compute).

**What actually happens:** The engine loads the model weights onto the GPU, runs the matrix math for prefill and decode, and manages the KV cache in GPU memory. vLLM is one popular engine; TensorRT-LLM and SGLang are two other common choices Dynamo can wrap. Dynamo talks to whichever engine you plug in — the engine is swappable; the orchestration around it stays the same.

**Why it matters (and why interviewers ask):** This is the other half of the layering point. Dynamo *wraps* an engine; the engine does the tokens. Knowing the difference tells an interviewer you won't confuse "which GPU runs the model" with "which system decides where requests go."

**From the project:** The engine here was **vLLM v0.12.0**, running **Meta-Llama-3.1-8B-Instruct** in BF16. Every actual token generated in the benchmarks came out of vLLM; Dynamo just coordinated the copies of it.

**Say it in one line:** "vLLM is the inference engine — it runs the model's forward passes on the GPU, and Dynamo wraps it."

| Layer | Who | Job |
|-------|-----|-----|
| Orchestration | Dynamo | Route, disaggregate, move KV |
| Engine | vLLM | Actually run the model on the GPU |

---

### (c) The Frontend / Router — the API gateway

**In one sentence:** The Frontend is an OpenAI-compatible API gateway that receives incoming requests and routes each one to the best worker.

**Analogy:** It's the restaurant host stand. Requests walk in, and the host decides which waiter (worker) to seat them with — and smartly seats you at the table that *already has your order half-prepared*.

**What actually happens:** Your app sends a normal OpenAI-style request (same `/v1/chat/completions` shape you'd send to OpenAI), so no special client is needed. The Frontend then picks a worker. Its clever trick is **KV-aware routing**: if some worker already has the KV cache for the tokens your prompt starts with, the router sends you *there* so that work isn't redone. Plain "round-robin" routing would ignore that and waste compute.

**Why it matters (and why interviewers ask):** Two things impress here: (1) "OpenAI-compatible" means drop-in adoption, and (2) KV-aware routing is a real, concrete optimization — it turns caching from luck into a routing decision. Interviewers like candidates who can name a routing strategy beyond round-robin.

**From the project:** The Frontend is the front door the AIPerf load generator hit at **concurrency 100**. Its KV-aware routing is the *sibling* idea to LMCache below: routing sends you to a worker that has your cache; LMCache is what makes that cache reusable in the first place.

**Say it in one line:** "The Frontend is the OpenAI-compatible gateway that does KV-aware routing — it sends each request to the worker that already has its cached tokens."

---

### (d) NIXL (NVIDIA Inference Xfer Library) — the KV delivery service

**In one sentence:** NIXL is the high-speed transport that moves KV cache data directly from a prefill GPU to a decode GPU with almost no overhead.

**Analogy:** NIXL is a dedicated courier with a key to the other building. Instead of knocking, waiting for someone to answer, and handing over a package (a normal network copy), the courier walks straight to the right shelf and places it there — no interruptions, no repackaging.

**What actually happens:** In disaggregation, the prefill GPU computes the KV cache, and the decode GPU needs it to keep generating. A normal network copy is slow because it involves the CPU, multiple memory copies, and both sides coordinating. NIXL uses **one-sided RDMA (Remote Direct Memory Access)**: "one-sided" means the sending GPU writes straight into the other GPU's memory without the receiver's CPU getting involved, and "zero-copy" means the data isn't duplicated into staging buffers along the way. That's why a *special* tool exists — regular networking pays overhead that would kill decode latency.

**Why it matters (and why interviewers ask):** Disaggregation only pays off if moving the KV cache is cheap. If the transfer were slow, you'd lose everything you gained by splitting prefill and decode. Interviewers ask about NIXL/RDMA to see if you understand that "move the cache fast" is a first-class problem, not an afterthought.

**From the project:** The student validated NIXL end-to-end with math: KV cache theory predicted **2.13 GB** per transfer, and the measured NIXL transfer was **2.147 GB** — a near-perfect match. That's proof the data crossing the wire is exactly the KV cache, and nothing extra.

**Say it in one line:** "NIXL moves KV cache GPU-to-GPU using one-sided, zero-copy RDMA — it's the fast courier that makes disaggregation worth it."

---

### (e) KVBM (KV Block Manager) — the reactive safety net

**In one sentence:** KVBM is a tiered memory manager that, when GPU memory fills up, spills KV cache out to CPU RAM (or disk) instead of crashing.

**Analogy:** KVBM is an overflow parking garage. When the main lot (GPU memory) is full, instead of turning cars away (crashing with Out-Of-Memory), it sends the extra cars to nearby lots (CPU RAM, then disk) and fetches them back when needed.

**What actually happens:** KV cache lives in fast but limited GPU memory. Long prompts or many concurrent requests can exhaust it, and normally that means an **OOM (Out-Of-Memory)** crash. KVBM adds a memory hierarchy: GPU -> CPU RAM -> disk. When the GPU is full, blocks of KV cache get moved down a tier and pulled back up when the request needs them again. It's **reactive** — it kicks in to rescue you when you're running out of room.

**Why it matters (and why interviewers ask):** Capacity is the difference between "serves the request slowly" and "the server dies." KVBM buys you headroom, especially for very long inputs. And keeping more of the cache reachable (instead of recomputing it) can slash the wait for the first token. Interviewers ask to check you can separate a *capacity* problem from a *speed* problem.

**From the project:** With the long-input workload (**highISL, ~16000 input tokens**), KV cache pressure is huge. KVBM cut **TTFT (Time To First Token)** by up to **95.3%** — because instead of recomputing or dying, it kept the needed KV blocks available across the memory tiers.

**Say it in one line:** "KVBM is the reactive safety net — when GPU memory fills, it spills KV cache to CPU/disk so you get overflow room instead of an OOM crash."

---

### (f) LMCache — the proactive speedup

**In one sentence:** LMCache reuses the KV cache of repeated prompt *prefixes* (like a shared system prompt) so the model skips re-computing them.

**Analogy:** LMCache is a photocopier with a memory. If ten forms all start with the same boilerplate header, it doesn't re-type the header ten times — it keeps one copy and reuses it, so it only does the new part.

**What actually happens:** Many requests share an identical opening — the same long system prompt, the same instructions, the same document. Prefill normally recomputes that KV cache every single time. LMCache stores the KV for those repeated prefixes and hands it back on the next matching request, so the engine only computes the genuinely new tokens. It's **proactive** — it makes common cases faster on purpose, not a rescue when something breaks.

**Why it matters (and why interviewers ask):** Prefix reuse is one of the biggest real-world wins in LLM serving, because production traffic is full of repeated prompts. The payoff scales with how often prompts repeat — high reuse, big speedup; all-unique traffic, almost nothing. Interviewers want to hear that this is a *hit-rate-dependent* optimization.

**From the project:** With repeated prompts, LMCache hit a **token hit rate of 87.9%** and TTFT dropped to **418 ms**. With all-unique prompts, the hit rate was just **1.0%** and TTFT was **756 ms** — nothing to reuse, so almost no benefit. Same tool, opposite outcomes, driven entirely by reuse.

**Say it in one line:** "LMCache is the proactive speedup — it reuses the KV of repeated prompt prefixes so the model skips re-computing shared text."

---

### KVBM vs LMCache — don't mix these up

These two both touch KV cache, so beginners blur them. They solve *opposite* problems:

| | **KVBM** | **LMCache** |
|---|---|---|
| Problem it solves | Running out of GPU memory | Recomputing the same prefix |
| Nature | **Reactive** safety net | **Proactive** speedup |
| Trigger | GPU memory fills up | A prompt prefix repeats |
| Action | Spill KV to CPU RAM / disk | Reuse stored KV, skip recompute |
| Failure it prevents | OOM crash | Wasted prefill work |
| Project number | TTFT cut up to **95.3%** (highISL) | TTFT **418 ms** vs **756 ms** at **87.9%** vs **1.0%** hit rate |

**Say it in one line:** "KVBM is about *capacity* — don't crash when memory is full; LMCache is about *speed* — don't recompute what you've already seen."


---


## 5. Sizing the KV Cache & a Feel for the Hardware (intuition, kept gentle)

Before this section, a quick reminder of one idea from earlier clusters: when an LLM (Large Language Model) generates text, it saves the **KV cache** — the Key and Value vectors for every token it has already seen — so it never has to re-read the whole prompt for each new word. This section is about one simple, high-stakes question: *how big does that cache get, and can the hardware move it around fast enough?* We will build intuition, not do heavy math.

---

### 5a. Why the KV cache grows with sequence length

**In one sentence:** Every token the model processes leaves behind a little memory footprint (its K and V vectors) in every layer, so the cache gets bigger and bigger as the text gets longer.

**Analogy:** Think of the model reading a book and taking two sticky notes per page — one "Key" note, one "Value" note — and it does this on *every floor of a 32-story library* (one floor per layer). Ten pages is a handful of notes. Ten thousand pages is a mountain of notes, on every floor. The notes never get thrown away while you are still reading.

**What actually happens:** For each new token, the model computes a Key vector and a Value vector inside every one of its layers, and it keeps them. So the total memory is roughly:

```
memory  ~  (number of tokens)  x  (number of layers)  x  (size of each K/V)
```

Two of those three things are fixed by the model (layers, vector size). The one that *you* control by your workload is **the number of tokens**. Double the sequence length, and you roughly double the cache. It grows *linearly* with how much text is in play (prompt + everything generated so far).

**Why it matters (and why interviewers ask):** This is the single reason "long context is expensive." GPU memory is finite (80 GB on the cards in this project). If the cache grows past what fits, you get an OOM (Out Of Memory) crash, or you have to serve fewer requests at once. Interviewers ask because it separates people who think "context is free" from people who know context costs *memory*, and memory is the thing that runs out first.

**From the project:** The two workloads were **lowISL** (ISL, Input Sequence Length, ~500 tokens) and **highISL** (ISL ~16000 tokens), both producing OSL (Output Sequence Length) ~256 tokens. The highISL prompt is about **32x longer** than the lowISL prompt, so its per-request KV cache is roughly 32x heavier. That is exactly why the highISL runs are "memory-hungry" and why a tiered memory manager (KVBM, covered in another cluster) earns its keep there.

**Say it in one line:** "The KV cache grows linearly with sequence length because every token adds K and V vectors in every layer — so long contexts are memory-hungry."

---

### 5b. GQA — Grouped-Query Attention (sharing to save memory)

**In one sentence:** GQA (Grouped-Query Attention) lets several of the model's query "heads" share a single set of Key/Value heads, so the model stores far fewer K/V vectors.

**Analogy:** Imagine 32 detectives (query heads) all working a case. In the old way, each detective keeps their *own private copy* of the evidence file (K/V). With GQA, you form groups: 4 detectives share **one** evidence file. Same 32 detectives asking their own questions, but only **8** evidence files to store instead of 32. Four times less filing.

**What actually happens:** Attention has multiple "heads." Each **query head** asks its own questions, but the Keys and Values are what actually take up cache space. GQA keeps all the query heads (so the model stays expressive) but reduces the number of *K/V* heads that get stored. Query heads are bucketed into groups, and each group reads from one shared K/V head.

```
Multi-Head (old):  32 query heads  ->  32 K/V sets stored   (heavy)
GQA (this model):  32 query heads  ->   8 K/V sets stored   (4x lighter)
                    \___ 4 queries share 1 K/V ___/
```

**Why it matters (and why interviewers ask):** GQA is a big part of why modern models can handle long contexts at all. The memory formula (next) scales with the number of *KV heads*, not query heads — so cutting KV heads from 32 to 8 cuts the cache by ~4x for free-ish quality. Interviewers like this one because it shows you understand *where* the memory savings come from.

**From the project:** Meta-Llama-3.1-8B-Instruct has **32 query heads sharing 8 KV heads**. That 32:8 = 4:1 ratio is precisely the ~4x KV memory saving. If it had stored a full 32 KV heads, every cache number below would be 4x bigger.

**Say it in one line:** "GQA has several query heads share one KV head — Llama-3.1-8B has 32 query heads sharing 8 KV heads, which cuts KV cache memory about 4x."

---

### 5c. The KV size formula (gently), with the project's real plug-in

**In one sentence:** You can estimate the exact KV cache size by multiplying together six simple numbers, and for this project the estimate lands almost perfectly on the measured value.

**Analogy:** It is like sizing a parking garage. Multiply (2 vehicle types) x (floors) x (cars per floor) x ... — once you know each factor, the total is just a product. No magic, just multiplication.

**What actually happens:** The formula is:

```
bytes = 2  x  layers  x  sequence_length  x  KV_heads  x  head_dim  x  bytes_per_number
        ^                                                              ^
        K and V                                                        precision (see 5d)
```

Each factor answers one question:

| Factor | Meaning | Why it's there |
|---|---|---|
| **2** | Keys *and* Values | You store both |
| **layers** | how deep the model is | cache exists in every layer |
| **sequence_length** | tokens in play | the part that grows (5a) |
| **KV_heads** | shared K/V heads | shrunk by GQA (5b) |
| **head_dim** | size of each head's vector | fixed by the model |
| **bytes_per_number** | precision | fixed by data type (5d) |

**From the project:** Plug in Llama-3.1-8B's real numbers. The sequence length is **16256** = 16000 input + 256 output (the highISL request at full length):

```
2  x  32  x  16256  x  8  x  128  x  2
= 2,130,706,432 bytes
≈ 2.13 GB
```

That is the **theoretical** size for one request's KV cache. When Dynamo actually moved that cache between GPUs using NIXL (the KV-transfer layer, covered elsewhere), the **measured** transfer was **2.147 GB**. Theory 2.13 GB vs reality 2.147 GB — within about 1%. That is the satisfying "theory-meets-reality" moment: the formula is not hand-waving, it genuinely predicts what the hardware moves.

**Why it matters (and why interviewers ask):** Being able to estimate cache size on the back of a napkin tells you how many requests fit in 80 GB, whether you will OOM, and how much data has to travel when you disaggregate prefill and decode. Interviewers ask because it proves you can reason about capacity instead of guessing.

**Say it in one line:** "KV bytes = 2 x layers x seq_len x KV_heads x head_dim x bytes — for our 16256-token request that's 2 x 32 x 16256 x 8 x 128 x 2 ≈ 2.13 GB, matching the measured 2.147 GB NIXL transfer."

---

### 5d. Precision — how many bytes each number costs

**In one sentence:** Precision is how many bytes you spend to store each number in the cache, and using fewer bytes directly shrinks the memory.

**Analogy:** It is like saving a photo. A high-resolution photo is crisp but big; a compressed one is smaller and *almost* as good. Precision is choosing the resolution for every number the model stores.

**What actually happens:** The `bytes_per_number` factor in the formula is set by the data type:

| Precision | Bytes per number | Relative KV size |
|---|---|---|
| **BF16** (BFloat16) | 2 | baseline |
| **FP8** (8-bit float) | 1 | **half** |

Because that factor multiplies the *whole* formula, halving it halves the entire cache. FP8 roughly doubles how much context or how many concurrent requests you can fit — at the cost of a little numerical accuracy.

**From the project:** This project ran the KV cache in **BF16 = 2 bytes**, which is the final "x 2" in the 5c calculation. If it had used FP8 instead, that request's cache would be about **1.07 GB** instead of 2.13 GB — same tokens, half the footprint.

**Why it matters (and why interviewers ask):** Precision is the easiest lever for fitting bigger workloads on the same GPU. Interviewers ask to see if you connect "data type" to "memory and throughput," and whether you understand it is a quality-vs-capacity trade-off, not a free lunch.

**Say it in one line:** "BF16 costs 2 bytes per number and FP8 costs 1, so dropping to FP8 halves KV cache memory — we ran BF16."

---

### 5e. A feel for the hardware "pipes" (compute vs memory vs network)

**In one sentence:** Data inside a server travels through connections of very different speeds — think of them as pipes of different widths — and the KV cache has to flow through whichever pipe you use to move it.

**Analogy:** Moving water. A fire hose (on-GPU memory) gushes; a garden hose (GPU-to-GPU link) is good; a drinking straw (across the network) trickles. The same bucket of water (your 2.13 GB cache) takes very different time depending on which one you pour it through.

**What actually happens:** Roughly from widest to narrowest:

```
GPU memory bandwidth  ~2 TB/s   |==================|  fire hose  (inside one GPU)
NVLink (GPU <-> GPU)   fast      |=========|            good hose (GPUs in one box)
PCIe (GPU <-> CPU)     slower    |====|                 garden hose
Network / InfiniBand /
  HPE Slingshot         narrow   |=|                    straw     (box to box)
```

- **GPU memory bandwidth (~2 TB/s on the A100):** how fast a GPU reads its own memory. Decode (generating one token at a time) mostly re-reads the KV cache, so **decode speed is limited by this bandwidth**, not by raw math.
- **NVLink:** a fast, direct link between GPUs *inside the same box*. This project's 8x A100-80GB are on **NVLink, single node**, so moving KV between its GPUs stays on a wide pipe.
- **PCIe:** the slower path between GPU and CPU. Used when you spill the cache to CPU RAM (what KVBM does to avoid OOM) — cheaper storage, narrower pipe.
- **Network (InfiniBand / HPE Slingshot):** links GPUs in *different boxes*. The narrowest pipe here; crossing it is the most expensive way to move a cache.

The takeaway: three different things can be the bottleneck — **compute** (raw math), **memory bandwidth** (reading the cache), or **network** (moving the cache between GPUs/nodes). Which one bites depends on your workload.

**Why it matters (and why interviewers ask):** Disaggregated serving deliberately splits prefill and decode onto different workers, which means the KV cache must *travel*. If it travels over a narrow pipe, the transfer time can eat the gains. Knowing the pipe widths tells you why staying on NVLink is good and why crossing the network is something you budget for. Interviewers ask because "is this compute-, memory-, or network-bound?" is the first question in any real perf discussion.

**From the project:** The single node with NVLink is why moving a **2.147 GB** cache between workers was practical — NVLink is a wide pipe. On this hardware the 2P2D configuration (2 prefill + 2 decode workers) won both workloads, peaking at **2037.5 output tokens/second** on lowISL. Because that KV traffic stayed on NVLink inside one box, transfer cost never became the wall.

**Say it in one line:** "Think compute vs memory bandwidth vs network as pipes of different widths — decode is bandwidth-bound, NVLink keeps intra-node KV transfers fast, and crossing the network is the narrow, costly pipe."

---

**Cluster recap (one breath):** The KV cache grows linearly with tokens x layers; GQA (32 query heads sharing 8 KV heads) cuts it ~4x; the size is just `2 x layers x seq_len x KV_heads x head_dim x bytes`, which gave 2.13 GB and matched the measured 2.147 GB; precision sets the final byte-count (BF16 = 2, FP8 = 1); and whether you are limited by compute, memory bandwidth, or network decides how fast that cache can be used and moved.


---


## Master Glossary (every term, one line each)

**Core generation**
- **Token** — a small chunk of text (~¾ of a word); the unit an LLM actually reads and writes.
- **Autoregression** — generating one token at a time, feeding each output back in to help predict the next.
- **Forward pass** — one trip of the current tokens through the network that produces exactly one next token.
- **K / V (Key / Value)** — per token: the Key is its label ("find me if you need this"), the Value is the content it contributes.
- **KV cache** — the saved Key/Value vectors of every past token, reused each step so the model never re-reads the prompt from scratch.
- **Prefill** — the phase that reads the whole prompt in parallel, builds the KV cache, and emits the first token (compute-heavy).
- **Decode** — the phase that generates the rest of the answer one token at a time by reusing the cache (memory-heavy).

**Lengths & limits**
- **ISL (Input Sequence Length)** — number of tokens in the prompt; the main driver of prefill cost.
- **OSL (Output Sequence Length)** — number of tokens the model generates; sets how many decode steps you pay for.
- **Context length** — ISL + OSL, the total tokens in play at once (grows as decode proceeds).
- **Context window** — the maximum context length a model can handle; exceed it and the request errors or truncates.

**Latency metrics**
- **TTFT (Time To First Token)** — submit-to-first-token wait; dominated by prefill; the responsiveness metric.
- **TTST (Time To Second Token)** — submit-to-second-token; exposes how cleanly the system hands prefill off to decode.
- **ITL (Inter-Token Latency)** — the measured gap between consecutive output tokens; the streaming-smoothness metric.
- **TPOT (Time Per Output Token)** — decode time ÷ output tokens; the averaged version of ITL (smooths out jitter).
- **Latency** — how long one request takes (e.g. TTFT, end-to-end); about a single user's wait.

**Throughput & load**
- **Throughput** — total work done per unit time across all users at once (the capacity/cost metric).
- **Tokens/s** — a common throughput unit (also requests/s, or tokens/s/GPU for fair per-GPU comparison).
- **Concurrency** — how many requests are in flight at once (the offered load).
- **Batch size** — how many requests the GPU processes together in one forward pass.
- **SLA (Service Level Agreement)** — the performance promise made to the customer (e.g. "TTFT under 500 ms").
- **SLO (Service Level Objective)** — the stricter internal target you engineer to (e.g. "TTFT under 400 ms").
- **Goodput** — the throughput that actually meets the SLO; always ≤ raw throughput.

**Bottlenecks & serving shapes**
- **Compute-bound** — limited by how fast the GPU can do math (prefill's bottleneck).
- **Memory-bandwidth-bound** — limited by how fast the GPU can read its own memory (decode's bottleneck).
- **Head-of-line blocking** — a long prefill hogs the GPU and stalls everyone's quick decode steps, spiking tail latency.
- **Aggregated serving** — one GPU worker does both prefill and decode; simple, zero transfer, but the phases fight.
- **Disaggregated serving** — prefill and decode run on separate GPUs so neither blocks the other and each scales alone.
- **P:D ratio** — how you split GPUs between Prefill and Decode workers (e.g. 2P2D = 2 prefill + 2 decode).

**The Dynamo cast**
- **Dynamo** — the orchestration layer that routes requests, disaggregates phases, and moves KV; it never runs the model.
- **Inference engine** — the component that actually executes the model's forward passes on the GPU.
- **vLLM** — the specific inference engine used here (swappable; TensorRT-LLM and SGLang are alternatives).
- **Frontend / Router** — the OpenAI-compatible API gateway that receives requests and picks a worker.
- **KV-aware routing** — sending a request to the worker that already holds its cached tokens, instead of round-robin.
- **NIXL (NVIDIA Inference Xfer Library)** — the fast transport that ships KV cache GPU-to-GPU with near-zero overhead.
- **RDMA (Remote Direct Memory Access)** — one-sided writes straight into another GPU's memory, no receiver CPU involved.
- **Zero-copy** — moving data without duplicating it into staging buffers along the way.
- **KVBM (KV Block Manager)** — the reactive safety net that spills KV cache to CPU RAM/disk when GPU memory fills.
- **LMCache** — the proactive speedup that reuses the KV of repeated prompt prefixes so shared text isn't recomputed.
- **Prefix caching** — the general idea of reusing cached KV for a shared prompt opening (what LMCache does).

**Model internals**
- **GQA (Grouped-Query Attention)** — several query heads share one KV head (32 share 8 here), cutting KV cache ~4×.
- **Head dimension** — the size of each attention head's vector (128 in Llama-3.1-8B); a fixed factor in the KV formula.
- **BF16 / FP8** — precision (bytes per number): BF16 = 2 bytes, FP8 = 1 byte; dropping to FP8 halves KV memory.

**Hardware "pipes"**
- **HBM / memory bandwidth** — a GPU's on-chip memory and how fast it reads it (~2 TB/s on A100); decode's limiter.
- **NVLink** — NVIDIA's fast direct GPU-to-GPU link *inside one box* (the wide pipe for intra-node KV transfer).
- **PCIe** — the slower GPU↔CPU link, used when spilling cache to CPU RAM (the "garden hose").
- **InfiniBand** — a high-speed network linking GPUs in *different boxes* (crossing it is the costly, narrow pipe).
- **HPE Slingshot** — another datacenter interconnect for box-to-box traffic, in the same "narrow network pipe" category.

---

## Test Yourself (10 questions)

**1. What is a token, and why is everything in serving measured in tokens instead of words?**
A token is a small chunk of text, roughly three-quarters of a word — the actual unit the model reads and writes. Context limits, latency, throughput, and price are all counted in tokens, so if you think in words your numbers will be off (1 token ≈ 0.75 words).

**2. Why does generating a 256-token answer take 256 steps instead of one?**
Generation is autoregressive: each token is fed back in to help predict the next, and one forward pass produces exactly one token. You literally cannot produce token #50 before tokens #1–49, so the loop runs once per output token.

**3. What is TTFT, and what makes it go up?**
TTFT (Time To First Token) is the wait from hitting "send" to the first word appearing; it's the responsiveness metric and is basically the cost of prefill. It rises with long prompts (big ISL), a cold cache, and queueing — which is why a 16k-token prompt pushed TTFT into the hundreds of seconds.

**4. Why does ISL matter so much?**
ISL (Input Sequence Length) is the prompt size in tokens, and prefill processes all of it at once, so ISL is the single biggest lever on TTFT. In the project, going from ~500 to ~16,000 input tokens is what turned prefill into a genuinely heavy job.

**5. Why is decode memory-bandwidth-bound rather than compute-bound?**
Decode produces just one token per step, so there's almost no math to do — but to make that token the GPU must re-read all 8 billion model weights plus the growing KV cache out of memory every step. The chip finishes the math instantly and sits waiting on memory, so memory bandwidth is the bottleneck.

**6. What does NIXL do, and why is it needed?**
NIXL is the transport that ships the KV cache from a prefill GPU to a decode GPU using one-sided, zero-copy RDMA (writing straight into the other GPU's memory, no receiver CPU, no duplicate copies). Disaggregation only pays off if that move is cheap, and ordinary networking would be too slow — so a special fast courier exists.

**7. How are KVBM and LMCache different?**
KVBM is a reactive safety net for *capacity*: when GPU memory fills, it spills KV cache to CPU RAM or disk so you don't OOM-crash. LMCache is a proactive speedup for *speed*: it reuses the KV of repeated prompt prefixes so the model skips recomputing shared text. One prevents crashes; the other prevents wasted work.

**8. Why did 2P2D win, especially on the highISL workload?**
With ISL ≈ 16,000, prefill is an enormous compute-bound job, so going from 1 to 2 prefill workers let two GPUs grind through the giant prompts in parallel instead of one struggling alone, while 2 decode workers kept answers flowing. The balanced 2P2D layout topped both workloads, peaking at 2037.5 output tokens/second on lowISL.

**9. What do SLA, SLO, and goodput mean, and how do they relate?**
The SLA is the performance promise to the customer, the SLO is the stricter internal target you engineer to, and goodput is the throughput that actually meets the SLO. Goodput ≤ throughput, because if you crank concurrency so high that requests blow the latency target, you're producing output nobody accepts.

**10. Roughly how big is the KV cache for one highISL request, and why is GQA involved?**
Using `2 × 32 layers × 16,256 tokens × 8 KV heads × 128 head-dim × 2 bytes`, it comes to about 2.13 GB — which matched the measured 2.147 GB NIXL transfer within ~1%. GQA matters because 32 query heads share only 8 KV heads, so we multiply by 8 instead of 32, shrinking the cache about 4×.


---

*That's the whole foundation. Once these feel automatic, the main interview guide's deep dives will click into place.*
