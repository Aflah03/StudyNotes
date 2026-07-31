# How Do We Know an AI Agent Is Any Good? A Survey on Evaluating LLM-Based Agents

> **Beginner-friendly explainer** — read this to understand the paper before you build your slides.

**Paper:** Survey on Evaluation of LLM-based Agents  
**Authors:** Asaf Yehudai, Lilach Eden, Alan Li, Guy Uziel, Yilun Zhao, Roy Bar-Haim, Arman Cohan, Michal Shmueli-Scheuer  
**Published:** 2025-03-20 (arXiv:2503.16416)  
**Link:** https://arxiv.org/abs/2503.16416  
**Security-focused:** No

---

## 1. The Big Picture

AI is shifting from systems that just answer questions to systems that take actions: booking things, browsing websites, editing code, and chaining many steps together to reach a goal. These action-taking systems are called agents, and they are being deployed quickly. But if we cannot reliably measure whether an agent actually works, is safe, and is worth its cost, we are essentially flying blind. This survey matters because it steps back from the flood of individual benchmarks and gives a single organized map of how the whole field measures agent quality, which is exactly what someone new needs before building or trusting an agent.

## 2. Key Terms, Decoded

- **LLM (large language model)** — An AI trained on huge amounts of text that predicts and generates language, such as the model behind ChatGPT.
- **LLM-based agent** — An LLM given the ability to plan, use tools, and act step by step in an environment to accomplish a goal, not just chat.
- **Benchmark** — A standardized set of tasks and a scoring method used to compare how well different systems perform.
- **Tool use (function calling)** — An agent's ability to call external functions or software, like a search engine, calculator, or API, to get things done.
- **Planning** — Breaking a big goal into an ordered sequence of smaller steps before or while acting.
- **Memory** — An agent's ability to store and recall information across steps or sessions instead of forgetting everything each turn.
- **Self-reflection** — An agent reviewing its own previous actions or mistakes and adjusting its behavior to do better.
- **LLM-as-a-judge** — Using another LLM to automatically grade an agent's output or behavior, instead of relying only on humans or fixed rules.
- **Trajectory** — The full sequence of an agent's actions and decisions on a task, not just its final answer.

## 3. How It Works (Step by Step)

This paper is a survey, so its "method" is a careful organization of existing research rather than a new algorithm. Step 1: The authors collect and review a large body of work on evaluating LLM-based agents. Step 2: They define a taxonomy, meaning a structured set of categories, organized into five perspectives. Step 3: The first perspective covers core agent capabilities, and they group benchmarks that test planning, tool use, memory, and self-reflection separately, so you can see how each skill is measured. Step 4: The second perspective covers application-specific benchmarks, grouping tests by domain such as web agents, software-engineering (coding) agents, and conversational or customer-service agents. Step 5: The third perspective looks at generalist agents, which are evaluated on broad, mixed tasks rather than one narrow domain. Step 6: The fourth perspective analyzes the benchmarks themselves along shared dimensions, like what task domain they use, how they score results, which capabilities they stress, and how realistic and reproducible they are. Step 7: The fifth perspective surveys the practical frameworks and tools developers use to actually run and monitor these evaluations. Step 8: Finally, they step back to identify overall trends and the gaps the field still needs to fix.

## 4. What They Found (Results)

Because it is a survey, the main output is a map and a set of observations, not a single score. The authors show that agent evaluation has exploded into many specialized benchmarks, and they organize dozens of them, such as WebArena and Mind2Web for web agents, SWE-bench for coding agents, tau-bench for customer-service agents, and GAIA, AgentBench, and OSWorld for generalist agents. Their key finding about direction is that evaluation is shifting toward more realistic and challenging tasks, with continuously updated or "live" benchmarks that prevent agents from simply memorizing answers, and toward methods that judge the whole sequence of actions rather than only the final result. They also point out clear gaps: most benchmarks still do a poor job of measuring cost-efficiency (how much compute or money a task takes), safety, and robustness, and the field lacks fine-grained, scalable ways to evaluate agents automatically.

## 5. Why It Matters

If you are building an agent, this survey tells you which benchmark to pick for the skill or domain you care about, so you are not testing blindly. If you are securing or deploying an agent, it highlights that safety, robustness, and cost are exactly the things current tests under-measure, which is where real-world failures and surprises tend to hide. And for the whole field, better evaluation is the feedback loop that drives progress: we can only improve what we can measure, so mapping and strengthening evaluation directly shapes how trustworthy future agents become.

## 6. A Simple Analogy

Think of an LLM as a bright new hire and an agent as that hire doing a real job. A one-question quiz tells you little about job performance. Instead you use different assessments: a driving-style road test for web browsing, a take-home coding assignment for software tasks, a role-play for customer service, and an all-around trial project for a generalist. This paper is like an HR handbook that catalogs every kind of test, explains what each one actually measures, and notes that the best tests are surprise, real-world scenarios rather than memorized practice questions.

## 7. How to Present This

Open with the hook question, "How do you grade something that takes 30 actions to finish a task?", to make the problem concrete before any jargon. Then show the five-perspective map as your backbone slide and keep returning to it so the audience never gets lost. Spend most time on one relatable example per branch, for instance SWE-bench (fix a real GitHub bug) and WebArena (complete a task on a website), because concrete tasks land far better than abstract categories. Emphasize the big trend, the move to live and trajectory-based evaluation, and explain why memorization is the enemy of a good benchmark. A strong visual is a simple diagram of an agent loop (goal, plan, act with a tool, observe, reflect) with labels showing where each capability benchmark plugs in. If you can, demo or screen-record a tiny agent doing a web or coding task and pause to ask "what would we measure here?" Close on the open gaps (cost, safety, robustness) as an invitation for future work, which leaves the audience with something to think about.

## 8. Likely Questions & Answers

**Q: What is the difference between an LLM and an LLM-based agent?**

A: An LLM just produces text from a prompt. An agent wraps an LLM with the ability to plan steps, call tools, keep memory, and act in an environment to reach a goal.

**Q: Why not just check if the agent got the final answer right?**

A: Two agents can reach the same answer, but one may take a risky, slow, or costly path. Judging the whole trajectory reveals reliability, safety, and efficiency that a final-answer check misses.

**Q: What does a 'live' or continuously updated benchmark mean, and why does it matter?**

A: It is a test whose tasks keep changing or refreshing so agents cannot memorize fixed answers. This keeps scores honest and reflects real, unseen situations.

**Q: Does this paper propose a new benchmark or a new model?**

A: No. It is a survey. Its contribution is organizing and comparing existing evaluation methods into a clear five-part map and pointing out the field's gaps.
