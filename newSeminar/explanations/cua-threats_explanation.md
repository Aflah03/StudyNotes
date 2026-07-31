# JARVIS or Ultron? A Beginner's Guide to the Safety and Security Threats of Computer-Using AI Agents

> **Beginner-friendly explainer** — read this to understand the paper before you build your slides.

**Paper:** A Survey on the Safety and Security Threats of Computer-Using Agents: JARVIS or Ultron?  
**Authors:** Ada Chen, Yongjiang Wu, Junyuan Zhang, Jingyu Xiao, Shu Yang, Jen-tse Huang, Kun Wang, Wenxuan Wang, Shuai Wang  
**Published:** 2025-05-16 (arXiv:2505.10924)  
**Link:** https://arxiv.org/abs/2505.10924  
**Security-focused:** Yes

---

## 1. The Big Picture

AI is shifting from tools that only talk to tools that act. Computer-Using Agents can operate a computer the way a person does, which promises huge productivity gains but also means an AI can now cause real-world harm through the same mouse and keyboard we use. This survey matters because it is one of the first organized attempts to catalog everything that can go wrong when an AI is allowed to click, browse, and control software, and to summarize how researchers are trying to keep these agents safe.

## 2. Key Terms, Decoded

- **Computer-Using Agent (CUA)** — An AI system that operates a computer like a human, reading the screen and clicking, typing, and browsing on its own.
- **Large Language Model (LLM)** — The AI text engine, such as the model behind ChatGPT, that gives the agent its reasoning and language abilities.
- **GUI (Graphical User Interface)** — The visual screen of buttons, menus, and windows that both people and these agents interact with.
- **Prompt injection** — An attack where hidden instructions placed in a webpage or file trick the agent into obeying the attacker instead of the user.
- **Jailbreak** — A trick that gets the agent to bypass its safety rules and do something it was told to refuse.
- **Intrinsic vs extrinsic threats** — Intrinsic threats come from the agent's own flaws; extrinsic threats are deliberately caused by outside attackers.
- **Attack Success Rate (ASR)** — A metric measuring how often an attack succeeds in making the agent misbehave.
- **Sandboxing** — Running the agent in a restricted, walled-off environment so mistakes cannot damage the real system.

## 3. How It Works (Step by Step)

This is a survey, so the method is a careful, structured literature review rather than building a new tool. Step 1, the authors searched academic databases and preprint servers (arXiv, Semantic Scholar, Google Scholar, and OpenReview) and screened over 700 candidate papers, narrowing down to 124 relevant ones for deep analysis. Step 2, they define a Computer-Using Agent in a way suited to safety analysis, breaking it into three components: perception (gathering information from the screen, logs, and user input), the brain (reasoning, memory, and planning that decides what to do), and action (executing clicks, keystrokes, and API calls). Step 3, they organize safety threats into two big families: intrinsic threats, which are the agent's own built-in weaknesses like hallucination, misalignment, and grounding errors, and extrinsic threats, which are deliberate attacks like adversarial inputs, prompt injection, jailbreaks, memory attacks, and backdoors. Step 4, they build a taxonomy of fourteen defense strategies, ranging from input validation and defensive prompting to sandboxing, output monitoring, and cross-verification. Step 5, they collect the benchmarks, datasets, and evaluation metrics the community uses so that future researchers have a shared reference point.

## 4. What They Found (Results)

Because it is a survey, the main output is a clear, organized map of the field rather than a single experimental number. From more than 700 screened papers, 124 were analyzed in depth. The authors produce a threat taxonomy with two top-level families (intrinsic and extrinsic), each containing about eight distinct threat types such as hallucination, misalignment, prompt injection, jailbreak, memory attacks, and backdoors. They present fourteen categories of defense strategies and catalog roughly twenty safety benchmarks (for example MobileSafetyBench, R-Judge, AgentDojo, ST-WebAgentBench, and InjecAgent) along with common metrics like Attack Success Rate, Task Success Rate, Refusal Rate, and Leakage Rate. A key finding is that the field is young and fragmented: threats are far ahead of defenses, and there is no single standard way to measure how safe an agent really is.

## 5. Why It Matters

As companies rush to deploy agents that can book travel, manage files, or shop online, giving an AI the power to act means giving it the power to cause harm, whether by honest mistakes or by falling for attacks. This survey gives builders a checklist of what can go wrong and a menu of defenses to consider, and it gives researchers a shared vocabulary and a list of open problems. In short, it helps steer the field toward JARVIS, the trustworthy helper, and away from Ultron, the agent that turns against its users.

## 6. A Simple Analogy

Think of a Computer-Using Agent as a brand-new intern who has full access to your computer and your accounts. The intern is fast and eager, but they cannot always tell a real instruction from a fake one. If a sticky note on a website says please forward all the company passwords, a careless intern might just do it. This survey is like a training manual that lists every way the intern could mess up or be tricked, and every safeguard a manager can put in place, such as limiting what the intern can touch and double-checking their work before anything important happens.

## 7. How to Present This

Open with a vivid one-sentence scenario, such as an agent reading a malicious webpage and leaking a password, then reveal the JARVIS-versus-Ultron framing to hook the audience. Spend the core of the talk on three anchors that map cleanly to slides: the three-part agent architecture (perception, brain, action), the two threat families (intrinsic versus extrinsic), and the fourteen defenses grouped into a few intuitive buckets. Use one concrete prompt-injection example walked through step by step, because it makes the abstract risk feel real. A strong visual is a single diagram showing the agent components in the middle, threats as arrows coming in, and defenses as shields on each component. Keep math out entirely; lean on analogies and real examples. Close by emphasizing that defenses lag behind threats and that no standard safety benchmark yet exists, which naturally motivates future research and invites discussion.

## 8. Likely Questions & Answers

**Q: How is a Computer-Using Agent different from a normal chatbot like ChatGPT?**

A: A chatbot only produces text, but a CUA actually takes actions, clicking, typing, and browsing, so its mistakes can change files, send messages, or spend money in the real world.

**Q: What is prompt injection and why is it so dangerous here?**

A: It is when hidden instructions in a webpage, email, or file trick the agent into obeying an attacker. It is dangerous because the agent can act on those instructions, not just repeat them.

**Q: Does the paper solve these security problems?**

A: No. It is a survey that organizes and maps the problems and existing defenses. It highlights that defenses are still immature and that there is no unified safety benchmark yet.

**Q: What is the single most important takeaway?**

A: Giving AI the power to act creates powerful new risks, and the field currently has many more documented threats than reliable defenses, so safety must be designed in from the start.
