# Trustworthy LLM Agents: Mapping the Threats and Defenses of AI Agents

> **Beginner-friendly explainer** — read this to understand the paper before you build your slides.

**Paper:** A Survey on Trustworthy LLM Agents: Threats and Countermeasures  
**Authors:** Miao Yu, Fanci Meng, Xinyun Zhou, Shilong Wang, Junyuan Mao, Linsey Pang, Tianlong Chen, Kun Wang, Xinfeng Li, Yongfeng Zhang, Bo An, Qingsong Wen  
**Published:** 2025-03-12 (arXiv:2503.09648)  
**Link:** https://arxiv.org/abs/2503.09648  
**Security-focused:** Yes

---

## 1. The Big Picture

AI agents are the next step beyond chatbots. Instead of only answering, they take actions: searching the web, running code, sending messages, and coordinating with other agents to reach a goal. Companies are racing to deploy them for coding, customer support, research, and personal assistants. But every new ability is also a new opening for misuse or failure, and a mistake by an acting agent can cause real harm, not just a wrong sentence. This survey steps back and organizes the whole safety picture so builders can see where the weak points are.

## 2. Key Terms, Decoded

- **Large Language Model (LLM)** — An AI trained on huge amounts of text that predicts and generates human-like language.
- **LLM Agent** — A system that wraps an LLM with memory, tools, and planning so it can act, not just chat.
- **Multi-Agent System (MAS)** — Several agents that communicate and cooperate to solve a task together.
- **Jailbreak** — Crafting inputs that trick a model into ignoring its built-in safety rules.
- **Prompt Injection** — Hiding malicious instructions inside data the agent reads so it obeys the attacker instead of the user.
- **Memory Poisoning** — Planting false or harmful information in an agent's stored memory so it recalls bad data later.
- **Backdoor** — A hidden trigger secretly planted during training that makes the agent misbehave on command.
- **Trustworthiness** — Whether an agent is safe, private, truthful, fair, and robust in practice.

## 3. How It Works (Step by Step)

Step 1: The authors break an AI agent down into its building blocks so that risks can be located precisely. Step 2: They separate these into intrinsic parts that live inside the agent, namely the brain (the core LLM that reasons and plans), the memory (what it stores and recalls), and the tools (external functions or APIs it calls). Step 3: They add extrinsic parts that describe how the agent connects to the outside world, namely the user it serves, other agents it talks to, and the environment it operates in. Step 4: For each of these six parts they gather the attacks that have been published, such as jailbreaks on the brain, poisoning of memory, or hijacking of tools. Step 5: They pair each attack area with the defenses researchers have proposed, like input filters, guardrail agents, and debate-based cross-checking. Step 6: They collect the benchmarks and datasets used to measure each risk. Step 7: They view everything through five trustworthiness properties (safety, privacy, truthfulness, fairness, and robustness) and finish by pointing out where defenses and tests are still missing.

## 4. What They Found (Results)

In plain terms, the survey maps the current state of the field rather than running one experiment. It finds that attacks are running ahead of defenses: work on attacking agents is plentiful, but defenses for tools are notably scarce and there are almost no reliable benchmarks for memory attacks or agent-to-agent safety. It catalogs concrete evaluation resources, such as AgentHarm with 110 malicious agent tasks, R-Judge with 27 risk scenarios, and S-Eval spanning 8 risk dimensions across 4 levels. It also observes that more complex setups (multi-agent systems) tend to expose a larger attack surface than single agents or bare LLMs. Finally, it shows that its coverage is broader than earlier surveys because it brings memory, tools, multi-agent interaction, and a modular taxonomy together in one place.

## 5. Why It Matters

Once an AI can act on the world, a security slip is no longer just a bad paragraph. It can leak private data, run harmful code, or spread misinformation to other agents. Builders need a shared map of where agents are vulnerable and which defenses actually exist, and this survey provides exactly that. For anyone deploying agents, it works as a checklist of threats to guard against and of gaps that still need research.

## 6. A Simple Analogy

Think of an LLM agent like a new employee. The brain is their judgment, the memory is their notebook, and the tools are the office equipment and accounts they can use. A jailbreak is smooth-talking them into breaking company rules; prompt injection is slipping a fake instruction into a document they read; memory poisoning is writing lies into their notebook so they act on them later. The survey is like a security audit that checks every one of these weak spots and lists which locks are already installed and which are still missing.

## 7. How to Present This

Open with the shift from chatbot to acting agent, because that framing motivates everything else. Put the TrustAgent map on one slide (a central agent with three inner boxes for brain, memory, and tool, and three arrows out to user, agent, and environment) and return to it as your anchor throughout. Spend most of your time on two or three vivid attack examples, such as prompt injection and memory poisoning, then immediately show the matching defense so the audience sees the attack-defense pairing. A good live demo is a simple prompt injection: paste a webpage snippet containing a hidden instruction and show the agent obeying it. Close with the gaps (scarce tool defenses, weak evaluation) so the audience leaves with clear open research directions. Keep the math out and lean on the diagram and concrete stories.

## 8. Likely Questions & Answers

**Q: Is this paper proposing a new defense system?**

A: No, it is a survey. Its contribution is the TrustAgent framework that organizes existing attacks, defenses, and benchmarks into one clear map, plus its analysis of the gaps.

**Q: What is the difference between attacking the LLM and attacking the agent?**

A: An LLM only handles text, so attacks target its outputs. An agent adds memory, tools, and other agents, so attackers can also poison memory, hijack tools, or spread attacks between agents.

**Q: What is the difference between a jailbreak and prompt injection?**

A: A jailbreak directly persuades the model to break its own rules. Prompt injection hides malicious instructions inside data the agent reads, so it follows the attacker while thinking it is doing its job.

**Q: Which area is least protected today?**

A: The survey flags tool-use defenses as notably scarce, and evaluation for memory and multi-agent safety as immature, so those are among the biggest open problems.
