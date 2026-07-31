# Old Rules for New Agents: Applying Classic Security Principles to LLM Agents

> **Beginner-friendly explainer** — read this to understand the paper before you build your slides.

**Paper:** LLM Agents Should Employ Security Principles  
**Authors:** Kaiyuan Zhang, Zian Su, Pin-Yu Chen, Elisa Bertino, Xiangyu Zhang, Ninghui Li  
**Published:** 2025-05-29 (arXiv:2505.24019)  
**Link:** https://arxiv.org/abs/2505.24019  
**Security-focused:** Yes

---

## 1. The Big Picture

AI assistants are moving from just chatting to actually doing things for us, such as booking trips, managing calendars, and handling money. Once an assistant can take real actions and touch our private data, a mistake or a manipulation can cause real harm like leaked credit card numbers or fraudulent payments. Attackers have already found reliable ways to trick these agents through hidden text instructions. This paper says the safest path forward is not a pile of one-off tricks, but the same time-tested security principles that have protected operating systems, the web, and mobile apps for decades.

## 2. Key Terms, Decoded

- **LLM agent** — A program built on a large language model that can plan, reason, and take real actions like sending emails or making payments.
- **Prompt injection** — An attack where malicious instructions are hidden inside data the agent reads, tricking it into obeying the attacker instead of the user.
- **Privacy leakage** — When an agent reveals sensitive personal information, such as a social security number or credit card, to a party that should not receive it.
- **Least privilege** — The rule that any component should get only the minimum access and data it needs for its current task, and nothing more.
- **Defense-in-depth** — Using several independent layers of protection so that if one layer is bypassed, others still catch the attack.
- **Complete mediation** — Checking every single access request against the security policy every time, not just the first time.
- **Psychological acceptability** — The idea that security must be easy and unobtrusive, because burdensome protections tend to get disabled or ignored.
- **PII** — Personally Identifiable Information, such as your name, phone number, passport number, or credit card details.

## 3. How It Works (Step by Step)

The core idea is to redesign the agent as a team of specialized parts instead of one all-powerful agent, and to apply the four security principles across that team. Step 1: A Persistent Agent (PA) is the user's trusted personal agent. It holds the long-term profile and memory (like preferences and PII) but never talks to the outside world directly. Step 2: When a task arrives, the request goes to the Data Minimizer (DM), which applies access-control policies and passes along only the small slice of data actually needed for that task. Step 3: A fresh Ephemeral Agent (EA) is created just for that one task. It is a throwaway worker that talks to external services using only the minimized data, and it is destroyed when the task ends, so any poison it swallowed cannot linger. Step 4: Every message the EA sends out or receives passes through an I/O Firewall, a fixed rule-based layer that forces inputs and outputs to follow an expected schema and blocks suspicious instructions. Step 5: Before results go back to the PA, a Response Filter (RF) sanitizes and validates them to make sure no sensitive data is leaking out. Step 6: A reward modeling policy engine automatically tunes the sharing policies over time based on whether tasks succeed safely, so humans do not have to hand-write endless rules. Each principle maps onto this design: defense-in-depth comes from the separate layers and disposable agents, least privilege comes from the Data Minimizer handing out only essential data, complete mediation comes from the DM, RF, and I/O Firewall checking every request, and psychological acceptability comes from the automatic self-tuning policies.

## 4. What They Found (Results)

The authors tested AgentSandbox on AgentDojo, a benchmark of 97 realistic tasks across four areas: Banking, Slack, Travel, and Workspace, each paired with prompt injection attacks. Using GPT-4o as the base model, AgentSandbox reduced the average attack success rate to about 4.34 percent, compared with about 58.84 percent when no defense was used. At the same time it kept benign usefulness at about 82 percent, very close to the roughly 83.81 percent of the undefended agent, meaning the protection did not cripple normal functionality. Other defenses fell short on one side or the other: simple methods like delimiting and repeating the prompt kept usefulness high but still let roughly 27 percent of attacks through, while a strict prompt-injection detector cut attacks but often halted the agent, dropping its usefulness under attack below 20 percent. AgentSandbox achieved the best overall trade-off, reaching the lowest attack success rate in every task suite (for example 0.83 percent on Workspace and 3.81 percent on Slack).

## 5. Why It Matters

As companies rush to deploy agents that manage money, health, and business workflows, a single tricked agent could cause real financial and privacy damage, and current quick fixes are demonstrably weak. This paper matters because it offers a coherent, principled blueprint rather than another isolated patch, and it shows the approach can be strong on security without wrecking usefulness. The idea of disposable, least-privilege agents behind checking layers is something builders can adopt today, and the authors argue these principles should be baked into future agent standards like MCP and A2A. It also connects the fast-moving AI world back to fifty years of hard-won security wisdom, which is a reassuring and practical direction.

## 6. A Simple Analogy

Think of a company handling a sensitive project. The CEO (the Persistent Agent) knows all the secrets but never meets outside vendors directly. For each errand, the CEO hires a temp worker (an Ephemeral Agent) and, through an assistant (the Data Minimizer), gives that temp only the one document they need, nothing else. All mail in and out of the building passes through a security desk (the I/O Firewall) that rejects anything not on the approved form, and a reviewer (the Response Filter) checks the temp's report before it reaches the CEO. When the errand is done, the temp is let go, so if a vendor slipped them a bribe or bad instruction, it cannot spread. Many small, limited helpers behind checkpoints are far safer than one all-knowing employee who talks to everyone.

## 7. How to Present This

Open with a concrete scare story: the travel-booking example from the paper, where a compromised flight tool hides a note telling the agent to wire 500 dollars to an attacker and email the user's SSN. That instantly makes the threat real. Then state the thesis in one line: reuse fifty-year-old security principles instead of inventing new patches. Introduce the four principles briefly, then reveal the AgentSandbox architecture with a single clear diagram showing the five components and the numbered flow from user to Persistent Agent to Data Minimizer to Ephemeral Agent to I/O Firewall and back through the Response Filter. Emphasize the two biggest ideas: disposable ephemeral agents and giving each one only minimal data. For the payoff, show one results slide with the headline numbers (about 59 percent attack success with no defense versus about 4.3 percent with AgentSandbox, at nearly the same usefulness). A great demo is animating the same travel example twice, once undefended where the attack succeeds and once through AgentSandbox where each layer blocks it. Close by noting it is a position paper with a preliminary evaluation, so it is a call to action, not a finished product.

## 8. Likely Questions & Answers

**Q: Is AgentSandbox a finished product I can download and use?**

A: No. It is a conceptual framework in a position paper with a preliminary evaluation. It demonstrates the principles and reports promising numbers, but it is meant to guide design, not ship as a library.

**Q: If it splits the agent into five parts and creates a new agent per task, isn't it slow and expensive?**

A: Adding layers and spinning up ephemeral agents does add overhead and extra model calls. The paper focuses on the security-versus-usefulness trade-off and does not claim to eliminate this cost, which is a fair limitation to raise.

**Q: Why not just use a prompt injection detector that blocks bad inputs?**

A: The paper tested that. It did lower attacks, but it often halted the agent entirely on any suspicion, dropping usefulness under attack below 20 percent, so users likely could not rely on it in practice.

**Q: What actually stops the attack in their example?**

A: Several layers. The Data Minimizer never gives the ephemeral agent the SSN, so it cannot leak it, and the I/O Firewall rejects the out-of-schema instruction to wire money. That layering is defense-in-depth in action.
