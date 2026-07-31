# Large Language Models for Planning: How AI Agents Turn a Goal into Step-by-Step Actions

> **Beginner-friendly explainer** — read this to understand the paper before you build your slides.

**Paper:** Large Language Models for Planning: A Comprehensive and Systematic Survey  
**Authors:** Pengfei Cao, Tianyi Men, Wencan Liu, Jingwen Zhang, Xuzhao Li, Xixun Lin, Dianbo Sui, Yanan Cao, Kang Liu, Jun Zhao  
**Published:** 2025-05-26 (arXiv:2505.19683)  
**Link:** https://arxiv.org/abs/2505.19683  
**Security-focused:** No

---

## 1. The Big Picture

People are increasingly trying to use LLMs not just to chat, but to act as autonomous agents that get real jobs done, like booking travel, operating software, or controlling a robot. Every one of those jobs requires planning: figuring out which steps to take, in what order, to reach a goal. This survey matters because it is the first thing you should read to understand the whole zoo of methods people have invented to make LLMs plan well, instead of getting lost in hundreds of individual papers. It gives you a mental filing cabinet with three drawers, so any new planning method you meet can be quickly understood as a variation of one of three basic ideas.

## 2. Key Terms, Decoded

- **Planning** — The process of turning a high-level goal into an ordered sequence of actions that reaches that goal.
- **Large Language Model (LLM)** — An AI trained on huge amounts of text that predicts and generates language, and can also reason and follow instructions.
- **Agent** — A system that perceives its environment and takes actions to accomplish goals, often powered by an LLM.
- **Task decomposition** — Breaking one hard task into smaller, easier subtasks that can be solved step by step.
- **Classical planner and PDDL** — Traditional planning software plus a formal language for describing goals, actions, and world states precisely.
- **Trajectory** — A recorded sequence of steps an agent took to attempt a task, used as training data or examples.
- **Fine-tuning** — Further training an existing model on task-specific data so it gets better at that task.
- **Search (tree search or Monte Carlo)** — Systematically exploring many possible action sequences and picking the most promising one.

## 3. How It Works (Step by Step)

The survey is a map, not a single method, so understanding it means understanding how it organizes the field. Step 1: it defines what planning is and reviews classical planning, where a formal language like PDDL describes the world and a dedicated solver searches for a valid action sequence. Step 2: it explains why LLMs are attractive, namely that they take goals in plain language and bring broad world knowledge, but also why they are unreliable, since they can hallucinate invalid steps and forget context. Step 3: it introduces the first family, External Module Augmented Methods, where the LLM is paired with outside helpers, for example letting the LLM translate a request into PDDL and handing it to a classical solver, or giving it tools and memory. Step 4: it introduces the second family, Finetuning-based Methods, where the LLM is trained on planning trajectories and on feedback signals about success or failure, so good planning behavior is baked into the model weights. Step 5: it introduces the third family, Searching-based Methods, which decompose a complex task into subtasks and explore multiple candidate plans, using techniques like tree search or improved decoding to select the best path instead of trusting the first guess. Step 6: it surveys how all these methods are evaluated, listing benchmark environments and metrics. Step 7: it steps back to discuss why LLMs can plan and lists open research directions.

## 4. What They Found (Results)

Because this is a survey, its result is the organized map itself rather than a single experiment. Its main finding is that the large and messy body of LLM planning work fits cleanly into three families: adding external modules, fine-tuning the model, and adding search or decomposition. It also finds that there is no single best method, since each family trades off differently, for example external planners give guarantees but need formal descriptions, fine-tuning needs lots of trajectory data, and search costs extra computation. On the evaluation side, it consolidates the benchmarks the community uses, such as Blocksworld, ALFWorld, WebShop, TravelPlanner, and web-agent tasks, and the metrics that matter, mainly success rate, whether plans are executable, plan optimality or efficiency, and how well methods hold up on longer, harder tasks.

## 5. Why It Matters

If you want to build an AI agent that reliably does multi-step jobs, you need to choose a planning strategy, and this survey tells you what the options are and how they are tested. It also matters for safety and security: a planner that produces invalid or unintended steps can take harmful actions in the real world, so knowing which methods add checks, feedback, or formal guarantees helps you build agents that fail safely. For anyone entering the field, it is a shortcut to seeing the whole landscape and spotting where the unsolved problems, and therefore the research opportunities, are.

## 6. A Simple Analogy

Think of planning a dinner party. The External Module approach is like an experienced cook who calls a specialist, such as a recipe app or a calculator, to handle the tricky parts precisely. The Fine-tuning approach is like a cook who has personally hosted hundreds of parties and learned from every success and disaster, so good instincts are built in. The Searching approach is like a cook who sketches three different menus, mentally simulates how each evening would go, and only then picks the best one instead of committing to the first idea.

## 7. How to Present This

Open with a relatable agent goal, such as plan and book a weekend trip, and show why naive prompting fails, to motivate why planning is hard. Make the three families the backbone of the talk and give each family exactly one concrete, named example so the audience remembers it: LLM plus classical planner for External Module, training on trajectories for Fine-tuning, and tree-of-thoughts style search for Searching. Use a single diagram that shows the same goal flowing through the three families side by side, so the differences are visual. For a demo, contrast a plain LLM answer with a decomposed, step-by-step plan for the same task to make the improvement obvious. Save the last few minutes for benchmarks and open challenges, since a beginner audience finds it satisfying to end on where the field is going. Keep math out and lean on analogies.

## 8. Likely Questions & Answers

**Q: Is this survey proposing a new planning method?**

A: No. It is a survey, so its contribution is a clear taxonomy that organizes existing methods into three families plus a consolidated view of benchmarks, metrics, and open problems.

**Q: Why not just always use a classical planner if it is guaranteed correct?**

A: Classical planners need the world hand-written in a formal language like PDDL, which is slow, brittle, and impractical for messy real-world tasks, whereas LLMs accept plain-language goals directly.

**Q: How do we know one planning method is better than another?**

A: By running them on shared benchmarks such as Blocksworld, ALFWorld, or TravelPlanner and comparing metrics like success rate, whether the plan is executable, and its efficiency.

**Q: Can these three families be combined?**

A: Yes. Many strong systems mix them, for example fine-tuning a model and also giving it external tools and search, and the survey treats the families as building blocks rather than mutually exclusive choices.
