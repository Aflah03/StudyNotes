# Progent: Least-Privilege Security for AI Agents

> **Beginner-friendly explainer** — read this to understand the paper before you build your slides.

**Paper:** Progent: Securing AI Agents with Privilege Control  
**Authors:** Tianneng Shi, Jingxuan He, Zhun Wang, Hongwei Li, Linyu Wu, Wenbo Guo, Dawn Song  
**Published:** 2025-04-16 (arXiv:2504.11703)  
**Link:** https://arxiv.org/abs/2504.11703  
**Security-focused:** Yes

---

## 1. The Big Picture

AI agents are moving from answering questions to actually doing things on our behalf, like managing email, browsing the web, and running commands. The moment an agent can take actions in the real world, security stops being optional, because a single tricked action can leak data, delete files, or spend money. Progent tackles this by borrowing a classic idea from computer security, least privilege, and adapting it to agents so that an agent is only ever allowed to do the specific things its current task requires and nothing more.

## 2. Key Terms, Decoded

- **AI agent** — An LLM-powered system that can take actions by calling tools, not just produce text replies.
- **Tool call** — A specific action an agent invokes, such as send_email or delete_file, usually with arguments like a recipient or filename.
- **Indirect prompt injection** — An attack where malicious instructions are hidden in content the agent reads, tricking it into unwanted actions.
- **Privilege control (least privilege)** — The principle that a system should be granted only the minimum permissions it needs, and no more.
- **Security policy** — A set of rules that says which tool calls are allowed and which are blocked for the current task.
- **SMT solver** — An automated logic engine that can prove whether one set of rules is stricter than another; Progent uses one called Z3.
- **Monotonic confinement** — A guarantee that an agent's allowed actions can only shrink automatically and can never expand without human approval.
- **Utility** — How well the agent still completes its intended tasks; a good defense keeps utility high while blocking attacks.

## 3. How It Works (Step by Step)

Step 1: The user gives the agent a task, for example booking travel or answering an email. Step 2: Before the agent starts acting, an LLM automatically writes an initial security policy, which is a whitelist of the exact tool calls the task needs, expressed as rules over tool names and their arguments (Progent implements these rules using JSON Schema). Step 3: The agent runs normally, but every single tool call it wants to make is first checked against the policy by a simple, deterministic procedure. Step 4: If the tool call matches an allow rule, it proceeds; if it is not allowed, for example a hidden instruction telling it to transfer money, it is blocked before it can happen. Step 5: As the agent gathers new information mid-task, the LLM may propose updating the policy, and an SMT solver called Z3 classifies each update as either a narrowing (making rules stricter) or an expansion (loosening them). Step 6: Narrowing updates are applied automatically, but expansions require explicit human approval, so the agent's real power can only shrink on its own, never grow silently. This is the monotonic confinement guarantee that keeps attacks from quietly escalating the agent's permissions.

## 4. What They Found (Results)

Progent was tested on two established agent security benchmarks. On AgentDojo, it cut the attack success rate from 39.9 percent down to 1.0 percent. On ASB (Agent Security Bench), it cut the attack success rate from 70.3 percent down to 3.9 percent. In both cases the agent kept its utility high, meaning it still completed legitimate tasks well rather than being crippled by the defense. The authors also compared Progent against a range of other defenses (such as prompt-injection detectors, delimiting, and tool filtering) and showed it works inside real-world agent frameworks including LangChain and the OpenAI Agents SDK.

## 5. Why It Matters

As agents get permission to touch money, files, and private data, a single injected instruction could cause real harm, so we cannot rely only on the LLM to always behave. Progent matters because it puts a deterministic, non-LLM guardrail between the agent and its actions, so security does not depend on the model perfectly resisting every trick. Its design is practical: policies are generated automatically so developers do not hand-write them, and the shrink-only update rule means the safe default holds even under attack. Because it plugs into popular frameworks, it points toward a realistic way to ship safer agents today.

## 6. A Simple Analogy

Think of a valet key for a car. When you hand your car to a valet, you can give them a special key that starts the engine and opens the door but cannot open the trunk or glove box where valuables are kept. Progent is like automatically fitting each agent task with its own valet key: the agent gets exactly the powers it needs to do the job and nothing more, so even if someone whispers a bad instruction, the agent simply lacks the key to do damage.

## 7. How to Present This

Open with a concrete scare-story demo: show an agent reading a web page or email that secretly says transfer all money or delete files, and let the audience feel how easily the agent obeys. Then reframe the whole talk around one idea, least privilege, so the audience has a single anchor. A great visual is a two-panel diagram: without Progent the injected tool call goes straight through, while with Progent it hits a policy check and is blocked. Spend real time on the SMT solver step, using a simple picture of an allowed-action circle that can only shrink automatically and needs approval to grow, because monotonic confinement is the clever core. Present the two headline numbers (39.9 to 1.0 percent and 70.3 to 3.9 percent) on one slide next to a utility bar so people see security went up without utility crashing. Close by showing it runs in LangChain and the OpenAI Agents SDK to prove it is practical, not just theory.

## 8. Likely Questions & Answers

**Q: How is this different from just asking the LLM to detect and ignore malicious instructions?**

A: Detection depends on the model being right every time, which attackers keep defeating. Progent enforces rules with a deterministic, non-LLM check, so blocking does not rely on the model's judgment.

**Q: If an LLM writes the policy, can the attacker just trick it into writing a loose policy?**

A: That risk is limited by monotonic confinement: any policy update that loosens permissions needs explicit human approval, so an attacker cannot silently expand the agent's powers even if it influences the LLM.

**Q: Does locking the agent down hurt its ability to do normal tasks?**

A: The results show utility stayed high on both benchmarks while attacks dropped sharply, because the policy whitelists exactly the tool calls the task legitimately needs.

**Q: Can I use this on my own agent, or is it just a research prototype?**

A: The authors demonstrate Progent working inside real frameworks like LangChain and the OpenAI Agents SDK, so it is designed to integrate with practical agent systems.
