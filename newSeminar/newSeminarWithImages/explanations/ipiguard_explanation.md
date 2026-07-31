# IPIGuard: Planning Tool Use Up Front to Stop Prompt Injection in AI Agents

> **Beginner-friendly explainer** — read this to understand the paper before you build your slides.

**Paper:** IPIGuard: A Novel Tool Dependency Graph-Based Defense Against Indirect Prompt Injection in LLM Agents  
**Authors:** Hengyu An, Jinghuai Zhang, Tianyu Du, Chunyi Zhou, Qingming Li, Tao Lin, Shouling Ji  
**Published:** 2025-08-21 (arXiv:2508.15310)  
**Link:** https://arxiv.org/abs/2508.15310  
**Security-focused:** Yes

---

## 1. The Big Picture

Companies are racing to deploy AI agents that act on our behalf, such as booking trips, managing inboxes, or handling payments. To be useful, these agents must read information from the open internet and other sources they do not control. That same openness is a security hole, because whatever the agent reads could secretly contain commands. IPIGuard is one of the first defenses to fix this at the level of how the agent runs, not just what it is told, which is why it points toward a safer general way to build agents.

## 2. Key Terms, Decoded

- **LLM agent** — An AI system that uses a language model to decide which software tools to call in order to complete a task.
- **Tool (or tool call)** — An external function the agent can run, such as reading a file, fetching a web page, or sending money.
- **Prompt injection** — Sneaking instructions into text so the language model treats them as if they were real commands.
- **Indirect prompt injection (IPI)** — Prompt injection hidden inside data the agent fetches during a task, rather than typed by the user.
- **Tool Dependency Graph (TDG)** — A map built before execution that lists every planned tool call and shows which calls depend on others.
- **Directed acyclic graph (DAG)** — A set of nodes joined by one-way arrows with no loops, used here to order tool calls.
- **Attack Success Rate (ASR)** — The fraction of test cases in which the attacker's hidden goal was actually achieved.
- **Utility** — How often the agent still completes the user's real, intended task correctly.

## 3. How It Works (Step by Step)

Step 1: Collect only trusted inputs. IPIGuard gathers the user's instruction, the list of available tools, and the user's profile, but does not read any external data yet. Step 2: Plan first. Using only that trusted information, the LLM writes out a full plan as a Tool Dependency Graph, where each node is one tool call and arrows show which call needs another call's output. Step 3: Mark unknowns. Nodes whose arguments are already known are called deterministic; nodes with values that must come from other tool responses are called pending and are left with placeholders. Step 4: Execute in order. The agent walks the graph from start to finish and is only allowed to run tools that already appear in the plan. Step 5: Fill in the blanks, called argument estimation. For a pending node, the agent looks at the outputs of the earlier tools it depends on and fills in the missing arguments, turning it into a resolved node. Step 6: Allow safe extra lookups, called node expansion. If the agent needs more information mid-task, it may add only read-only query tools, never action-taking command tools that change the world. Step 7: Neutralize overlap attacks, called fake tool invocation. If an injected instruction tries to reuse a tool the plan already contains, such as transferring money, the agent runs a harmless fake version to absorb the injected command, so the real tool call stays aligned with the user's original intent.

## 4. What They Found (Results)

Tested on the AgentDojo benchmark, which has 97 tasks in four domains and 629 attack test cases, across six LLMs and four attack types, IPIGuard was consistently the strongest defense. On GPT-4o-mini it lowered the average attack success rate from 13.16 percent with no defense to 0.69 percent, and it never exceeded 1 percent on any single attack. At the same time it kept the user's real task working: average utility under attack was 58.77 percent, higher than every other defense, and its benign no-attack success rate of about 67 percent almost matched the 68 percent you get with no defense at all. Competing defenses either let too many attacks through, such as Spotlight allowing 22.26 percent on the strongest attack, or badly hurt utility, such as the Detector dropping utility to 26.50 percent. The main cost is roughly double the tokens and time compared with no defense, which is still far cheaper than the Sandwich baseline.

## 5. Why It Matters

As agents gain the power to spend money and touch real systems, a single injected instruction could cause real harm. IPIGuard shows that adding a hard structural rule, deciding all actions before reading untrusted data, blocks most attacks at their source instead of hoping the model resists them. This execution-centric idea is reusable, because any agent framework could adopt plan-then-execute with a locked tool list. It also demonstrates that you can gain strong security without throwing away usefulness, which is exactly the trade-off that usually stops safety features from being adopted.

## 6. A Simple Analogy

Think of a careful shopper who writes a complete shopping list at home before walking into a store full of tempting signs. Because the list is fixed in advance, no in-store advertisement can make them add a big unplanned purchase. They can still glance at a price tag or compare two items, which are harmless read-only checks, but they will never buy something new just because a sign told them to.

## 7. How to Present This

Open with a concrete scare story, such as a hidden note in a web page making an agent email out private data, so the audience feels the threat before any theory. Then contrast the two paradigms side by side: the traditional agent that decides and reads in a loop versus IPIGuard that plans first and locks the plan; the paper's Figure 2 is ideal to redraw as a before-and-after diagram. Spend most of your time on the Tool Dependency Graph using a single worked example, the pay-the-bill case with a hidden money-transfer instruction, and animate how the fake tool invocation absorbs the attack. Show one results slide with the headline numbers, 0.69 percent attack success versus 13.16 percent with utility roughly unchanged, rather than the full table. Close with the trade-off of about double the cost and the limitations so the talk feels balanced; a live or recorded demo of an attack failing under IPIGuard makes the strongest impression.

## 8. Likely Questions & Answers

**Q: How is this different from just telling the model to ignore suspicious instructions?**

A: Prompting relies on the model choosing to obey, while IPIGuard adds a hard rule that unplanned tools simply cannot run, so it does not depend on the model resisting temptation.

**Q: If the plan is fixed, how does the agent handle values it cannot know in advance, like a price from a website?**

A: Those become pending nodes with placeholders, and the argument estimation step fills them in later using the outputs of the tools they depend on.

**Q: Does locking the plan make the agent less capable?**

A: Mostly no, because node expansion still lets it add read-only lookups and measured utility stayed near the no-defense level, though very dynamic tasks that need brand-new actions mid-task are restricted.

**Q: What are the main costs or weaknesses?**

A: It roughly doubles token use and time, it needs a model with decent planning ability, and it targets tool-based attacks rather than attacks that only manipulate the agent's text output.
