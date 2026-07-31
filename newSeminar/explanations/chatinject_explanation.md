# ChatInject: How Forged Chat Formatting Tricks AI Agents into Obeying Attackers

> **Beginner-friendly explainer** — read this to understand the paper before you build your slides.

**Paper:** ChatInject: Abusing Chat Templates for Prompt Injection in LLM Agents  
**Authors:** Hwan Chang, Yonghyun Jun, Hwanhee Lee  
**Published:** 2025-09-26 (arXiv:2509.22830)  
**Link:** https://arxiv.org/abs/2509.22830  
**Security-focused:** Yes

---

## 1. The Big Picture

AI agents are LLMs that do not just chat but take actions: reading your inbox, searching the web, or running tools on your behalf. To be useful they must read data from the outside world, and that is exactly where danger enters, because attackers can plant instructions in that data. This paper shows a surprisingly simple and effective way to hijack such agents by faking the invisible formatting that chat models use to organize a conversation. Anyone building, deploying, or trusting an AI assistant should care, because the attack works across many models and slips past the usual defenses.

## 2. Key Terms, Decoded

- **LLM (Large Language Model)** — An AI system trained on huge amounts of text to understand and generate human language.
- **AI agent** — An LLM that can take actions such as calling tools, browsing, or sending messages, not just replying with text.
- **Prompt injection** — An attack that sneaks unauthorized instructions into an AI's input so it does what the attacker wants.
- **Indirect prompt injection** — Prompt injection where the malicious instructions are hidden inside external content the agent reads, like a web page or email.
- **Chat template** — The hidden formatting scheme, made of special markers, that labels each part of a conversation as system, user, or assistant.
- **Role markers / special tokens** — Symbols such as the start and end markers plus labels like system, user, and assistant that separate speaker turns for the model.
- **Attack success rate (ASR)** — The percentage of attempts in which the attack makes the agent perform the attacker's intended harmful action.
- **Multi-turn attack** — An attack spread across several conversation turns that slowly builds context until a harmful request seems acceptable.

## 3. How It Works (Step by Step)

Step 1: Understand the target. Chat models format every conversation with a hidden template that uses special markers to show where the system message, the user message, and the assistant reply each begin and end. Step 2: Find an injection point. In an agentic setting the agent reads outside content such as a tool result, email, or web page, and the attacker controls some of that text. Step 3: Forge the template. Instead of writing a plain command, the attacker writes text that copies these role markers, so the injected block looks like a brand-new user turn, or even a system instruction, rather than ordinary data. Step 4: Exploit instruction-following. Because the model is trained to obey whatever is framed as a legitimate user or system turn, it treats the forged block as a real instruction and carries out the attacker's action. Step 5: For the Multi-turn variant, the attacker fakes a short back-and-forth, including fake assistant replies that agree to help, gradually normalizing the harmful request so the real model continues the pattern and complies. Step 6: The same forged-template trick is reused across different models and even works on closed-source models whose exact template is unknown, because their formats are similar enough.

## 4. What They Found (Results)

Across many frontier models and two standard agent-security benchmarks, ChatInject was far more effective than ordinary plain-text injection. On AgentDojo the average attack success rate rose from about 5% to about 32%, and on InjecAgent it rose from about 15% to about 46%. The persuasion-based Multi-turn version did even better on InjecAgent, succeeding about 52% of the time on average. The attack also transferred well between models and stayed effective against closed-source models whose templates are secret. Finally, four common prompt-based defenses did little to stop it, and were especially weak against the Multi-turn variant.

## 5. Why It Matters

Agents are only safe if they can tell the difference between data they read and commands they should obey, and ChatInject shows that today's models often cannot, because the very formatting used to mark who is speaking can be forged. This matters for anyone connecting an LLM to email, the web, files, or tools, since a single poisoned document could redirect the agent. It also shows that popular prompt-based defenses can give a false sense of security. The finding pushes the field toward stronger, structural defenses that reliably separate untrusted content from trusted instructions, rather than just asking the model politely to ignore attacks.

## 6. A Simple Analogy

Imagine a receptionist who only acts on notes written on official company letterhead. An outsider mails in a letter but forges the company letterhead at the top, so the receptionist assumes it came from the boss and does what it says. ChatInject is that forged letterhead: it copies the official formatting the model trusts, so injected text gets treated as a genuine instruction instead of outside mail.

## 7. How to Present This

Open with a concrete example: show an agent reading an email that secretly contains forged role markers, then acting on the hidden command. Spend the most time on one clear diagram contrasting a normal chat template with a forged one, since that contrast is the heart of the paper. Put the three headline numbers on a single results slide (roughly 5% to 32% on AgentDojo, 15% to 46% on InjecAgent, and 52% for Multi-turn) so the jump is obvious. Emphasize three takeaways: the attack is simple, it transfers across models, and prompt defenses fail. Keep the math out and lean on the letterhead analogy. A strong demo is a side-by-side of the raw injected text with the visible fake tokens next to the agent's wrong action, then end on the defense implications to spark discussion.

## 8. Likely Questions & Answers

**Q: If the chat template is internal, how does the attacker know which markers to fake?**

A: Many models use similar, publicly documented formats with system, user, and assistant markers, and the paper shows the attack transfers even to closed-source models whose exact template is unknown.

**Q: Isn't this just the old 'ignore previous instructions' attack?**

A: No. Those are plain text that models learn to resist, while ChatInject forges the structural formatting the model uses to decide who is speaking, which is much harder to ignore.

**Q: Why does the Multi-turn version work better?**

A: It fakes a gradual conversation, including fake assistant agreement, so the harmful request looks like a natural continuation rather than a sudden suspicious command.

**Q: Can we just filter out the special tokens?**

A: Filtering helps but is brittle, because attackers can vary the wording or use look-alikes, so the paper argues for stronger structural separation of trusted and untrusted input.
