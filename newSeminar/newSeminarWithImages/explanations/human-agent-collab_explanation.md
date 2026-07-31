# Humans in the Loop: How People and LLM Agents Work Better Together

> **Beginner-friendly explainer** — read this to understand the paper before you build your slides.

**Paper:** LLM-Based Human-Agent Collaboration and Interaction Systems: A Survey  
**Authors:** Henry Peng Zou, Wei-Chieh Huang, Yaozu Wu, Jizhou Guo, Yankai Chen, Chunyu Miao, Hoang Nguyen, et al.  
**Published:** 2025-05-01 (arXiv:2505.00753)  
**Link:** https://arxiv.org/abs/2505.00753  
**Security-focused:** No

---

## 1. The Big Picture

AI agents built on large language models are moving from just answering questions to actually doing things — writing and running code, browsing websites, controlling robots, or managing workflows. The exciting but scary part is letting them act without supervision, because today's agents still make confident mistakes. This paper argues that the smarter near-term path is collaboration: design clear ways for humans and agents to work as a team, so people supply judgment, correction, and accountability while agents supply speed and scale. Understanding how to structure that teamwork matters to anyone who will build, use, or secure AI systems in the coming years.

## 2. Key Terms, Decoded

- **LLM (Large Language Model)** — An AI trained on huge amounts of text that can understand and generate human-like language.
- **Agent** — An LLM wrapped with the ability to plan, use tools, and take multi-step actions toward a goal, not just chat.
- **LLM-HAS (LLM-based Human-Agent System)** — A system where humans and LLM agents work together, with people providing input, feedback, or control.
- **Hallucination** — When an AI states false information confidently as if it were true.
- **Human-in-the-loop** — A design where a person reviews, guides, or approves an agent's actions during the task.
- **Orchestration** — The rules for who acts when — whether humans and agents take turns or work at the same time.
- **Human Agency Scale** — The paper's five-level rating (A1 to A5) of how much a task depends on the human versus the agent.
- **Benchmark** — A standardized test or dataset used to measure and compare how well systems perform.

## 3. How It Works (Step by Step)

Because this is a survey, its "method" is a way of organizing the field into clear building blocks. Step 1: it defines the environment and profiling — the setting (real world or simulation) and who takes part (one or many humans, one or many agents), plus whether users are "lazy" (giving little input) or "informative" (giving rich guidance). Step 2: it categorizes human feedback into four kinds — evaluative (rating or ranking outputs), corrective (directly fixing them), guidance (instructions or demonstrations given up front), and implicit (inferred from what the user does). Step 3: it classifies interaction types — collaboration (delegation, supervision, cooperation, coordination), competition, and a mix called coopetition. Step 4: it describes orchestration — whether people and agents act one-by-one or simultaneously, and synchronously (live) or asynchronously (delayed). Step 5: it maps communication — the structure (centralized, decentralized, or hierarchical) and the mode (conversation, observation, or a shared message pool). Step 6: it introduces the five-level Human Agency Scale from full automation to fully human-driven, and reviews how systems are built (prompting, fine-tuning, or reinforcement learning), plus applications, benchmarks, and open challenges.

## 4. What They Found (Results)

As a survey, its main "result" is the organized picture it produces. It finds that fully autonomous agents remain unreliable, so most successful real systems keep humans involved in structured ways. It shows that human feedback consistently falls into four types and that human involvement can be rated on a five-level scale. It observes that most current systems treat humans mainly as evaluators rather than active partners, that communication is often channeled through a central coordinator or a shared message pool, and that today's evaluation over-focuses on agent accuracy while ignoring how much effort the human spends. It also gathers concrete example systems (such as MetaGPT, Collaborative Gym, and COWPILOT) and benchmarks (such as MINT, tau-bench, and PARTNR) that the field relies on.

## 5. Why It Matters

If you are building AI agents, this gives you a menu of design choices — what feedback to collect, when humans should step in, and how components should talk — instead of reinventing them. If you care about safety, it highlights that humans act as a guardrail against hallucinations and unsafe actions, and it flags under-studied risks like privacy and error-containment during live collaboration. The shared vocabulary also makes it easier to compare systems and to spot gaps, such as the lack of testing in high-stakes areas like healthcare and finance.

## 6. A Simple Analogy

Think of an LLM agent like a brilliant but overconfident new intern. Turning it loose on a critical project alone is risky — it works fast but sometimes makes things up. The survey is essentially a management handbook for that intern: it lays out how to give feedback (a quick thumbs-up, a hands-on edit, or upfront instructions), when to check in (every step or only at milestones), and how the team should communicate (through a manager, peer-to-peer, or a shared notice board) so the human-plus-intern team outperforms either one alone.

## 7. How to Present This

Open with the core tension: autonomous agents are impressive but unreliable, so keeping humans in the loop is the practical fix — this hooks a general audience immediately. Spend most of the talk on the five building blocks (environment/profiling, feedback, interaction, orchestration, communication) using one clear diagram a viewer can absorb in ten seconds. Make the four feedback types and the five-level Human Agency Scale your two memorable takeaways, since they are concrete and easy to quiz on. Use one relatable example, like a coding assistant that suggests then waits for approval (COWPILOT's "Suggest-then-Execute" idea), to ground the abstractions. Close with the open challenges — human variability, measuring human effort, and safety — to invite discussion. A good demo is a live or mocked "suggest, human approves, agent executes" loop; a good diagram is a labeled human-agent loop showing feedback flowing back to the agent.

## 8. Likely Questions & Answers

**Q: Isn't the goal to make agents fully autonomous so we don't need humans?**

A: Eventually maybe, but the paper argues today's agents are too unreliable, so structured human involvement gives better, safer results right now.

**Q: What's the difference between a human 'in the loop' and just chatting with an AI?**

A: Chatting is one form, but the survey covers richer setups — humans correcting mid-task, supervising, or working in parallel — and formalizes when and how that help is given.

**Q: Since this is a survey, does it actually build anything new?**

A: It doesn't build a new model; its contribution is the first structured framework and vocabulary for the field, plus a five-level agency scale and a catalog of systems, benchmarks, and open problems.

**Q: How do you measure whether human-agent collaboration is actually working?**

A: The paper notes this is an open problem: most benchmarks only measure agent accuracy and ignore the human's workload, so it calls for human-centered evaluation.
